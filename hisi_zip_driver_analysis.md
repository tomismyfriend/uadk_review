# HiSilicon ZIP 驱动业务流程详解

## 一、整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户空间                                  │
│                    (应用程序/测试工具)                            │
└─────────────────────────────────────────────────────────────────┘
                              │ crypto API
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   hisi_zip 驱动层                                │
│  zip_main.c (模块初始化)  zip_crypto.c (算法实现)               │
└─────────────────────────────────────────────────────────────────┘
                              │ QM API
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   QM 硬件管理层 (qm.c)                          │
│        队列管理、中断处理、DMA、硬件寄存器访问                    │
└─────────────────────────────────────────────────────────────────┘
                              │ MMIO
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   ZIP 硬件 (PCIe设备)                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、核心数据结构

### 2.1 层次关系

```
hisi_zip (设备级)
    │
    ├── qm (hisi_qm) ──────────────────────────────┐
    │       ├── qp_array[] (所有QP)                │
    │       ├── sqc/cqc (队列上下文表)              │
    │       ├── eqe/aeqe (事件队列)                 │
    │       └── 中断、锁等                          │
    │                                               │
    └── ctrl (PF控制结构)                           │
                                                    │
hisi_zip_ctx (算法上下文，每个tfm一个)              │
    │                                               │
    ├── qp_ctx[0] (压缩QP上下文)                    │
    │       ├── qp ─────────────────────────────────┘
    │       │      └── 指向 qm->qp_array[] 中的某个QP
    │       ├── req_q (请求队列，预分配的hisiz_zip_req数组)
    │       │       ├── q[]: hisi_zip_req数组
    │       │       ├── req_bitmap: 已用请求位图
    │       │       └── size: 队列深度
    │       └── sgl_pool (DMA SGL内存池)
    │
    ├── qp_ctx[1] (解压QP上下文)
    │
    └── ops (SQE操作函数表)
```

### 2.2 关键结构体

#### hisi_zip_req (请求结构)
```c
struct hisi_zip_req {
    struct acomp_req *req;        // 用户的压缩请求
    struct hisi_acc_hw_sgl *hw_src; // 源数据DMA地址
    struct hisi_acc_hw_sgl *hw_dst; // 目的数据DMA地址
    dma_addr_t dma_src;           // 源DMA物理地址
    dma_addr_t dma_dst;           // 目的DMA物理地址
    struct hisi_zip_qp_ctx *qp_ctx; // 所属QP上下文 ← 关键！
    u16 req_id;                   // 请求ID
};
```

#### hisi_qp (硬件队列对)
```c
struct hisi_qp {
    u32 qp_id;
    u16 sq_depth;                 // SQ深度
    u16 cq_depth;                 // CQ深度

    struct qm_dma qdma;           // DMA内存(SQE+CQE)
    void *sqe;                    // SQE数组指针
    struct qm_cqe *cqe;           // CQE数组指针

    struct hisi_qp_status qp_status;
        // atomic_t used;          // 正在处理的请求数
        // u16 sq_tail;            // SQ写指针
        // u16 cq_head;            // CQ读指针
        // bool cqc_phase;         // CQE相位标志

    void (*req_cb)(struct hisi_qp *qp, void *data); // 回调函数
};
```

#### hisi_zip_sqe (SQE格式)
```c
struct hisi_zip_sqe {
    u32 consumed;                 // 已消费字节数
    u32 produced;                 // 已生产字节数
    u32 dw3;                      // 状态 (bits 0-7)
    u32 input_data_length;        // 输入数据长度
    ...
    u32 source_addr_l;            // 源地址低32位
    u32 source_addr_h;            // 源地址高32位
    u32 dest_addr_l;              // 目的地址低32位
    u32 dest_addr_h;              // 目的地址高32位
    ...
    u32 dw26;                     // TAG低32位 ← 存放req指针
    u32 dw27;                     // TAG高32位 ← 存放req指针
};
```

---

## 三、模块加载流程 (insmod hisi_zip.ko)

### 3.1 流程图

```
hisi_zip_init()
    │
    ├─► hisi_qm_init_list(&zip_devices)  // 初始化设备链表
    │
    ├─► hisi_zip_register_debugfs()      // 创建debugfs目录
    │
    └─► pci_register_driver()            // 注册PCI驱动
              │
              ▼ (PCI设备匹配时)
         hisi_zip_probe()
              │
              ├─► devm_kzalloc()         // 分配hisi_zip结构
              │
              ├─► hisi_zip_qm_init()     // 初始化QM
              │       │
              │       ├─► 设置qm参数 (pdev, mode, sqe_size等)
              │       │
              │       └─► hisi_qm_init()
              │               │
              │               ├─► hisi_qm_pci_init()    // 启用PCI设备
              │               │
              │               ├─► qm_irqs_register()    // 注册中断
              │               │
              │               ├─► qm_dev_mem_reset()    // PF: 复位设备内存
              │               │
              │               ├─► qm_alloc_uacce()      // uacce模式: 注册uacce
              │               │
              │               └─► hisi_qm_memory_init() // 分配QM内存
              │                       │
              │                       ├─► 分配 qdma (EQ/AEQ)
              │                       ├─► 分配 sqc/cqc 表
              │                       └─► hisi_qp_alloc_memory()
              │                               │
              │                               └─► 为每个QP分配:
              │                                   - sqe/cqe 内存 (qdma)
              │                                   - msg 数组
              │
              ├─► hisi_zip_probe_init()      // PF初始化
              │       │
              │       └─► hisi_zip_pf_probe_init()
              │               ├─► hisi_zip_set_user_domain_and_cache()
              │               ├─► hisi_zip_debug_regs_clear()
              │               └─► hisi_zip_show_last_regs_init()
              │
              ├─► hisi_qm_start()           // 启动QM
              │       │
              │       ├─► qm_eq_aeq_ctx_cfg()  // 配置EQ/AEQ
              │       │       │
              │       │       ├─► memset(qm->qdma.va, 0, ...)  // 清零EQ/AEQ内存
              │       │       ├─► qm_eq_ctx_cfg()
              │       │       └─► qm_aeq_ctx_cfg()
              │       │
              │       ├─► hisi_qm_mb_write(SQC_BT) // 写入SQ上下文基址
              │       ├─► hisi_qm_mb_write(CQC_BT) // 写入CQ上下文基址
              │       │
              │       └─► qm_enable_eq_aeq_interrupts() // 使能中断
              │
              ├─► hisi_qm_alg_register()     // 注册到crypto子系统
              │       │
              │       └─► hisi_zip_register_to_crypto()
              │               │
              │               └─► crypto_register_acomp(&hisi_zip_acomp_deflate)
              │
              └─► qm_register_uacce()        // uacce模式注册
```

### 3.2 关键点说明

**注意：QM启动时，只清零了EQ/AEQ内存，没有清零各QP的SQE/CQE内存！**

```c
// qm.c: __hisi_qm_start()
ret = qm_eq_aeq_ctx_cfg(qm);  // 这里只清零 qm->qdma.va (EQ/AEQ)

// QP的内存是在 hisi_qp_alloc_memory() 中分配的，分配时没有清零！
```

---

## 四、算法上下文初始化 (crypto tfm init)

当用户打开压缩设备时，调用 `hisi_zip_acomp_init`:

```
hisi_zip_acomp_init(crypto_acomp *tfm)
    │
    ├─► hisi_zip_ctx_init()
    │       │
    │       ├─► zip_create_qps()  // 创建2个QP (压缩+解压)
    │       │       │
    │       │       └─► hisi_qm_alloc_qps_node()
    │       │               │
    │       │               └─► qm_get_and_start_qp()
    │       │                       │
    │       │                       ├─► qm_create_qp_nolock()  // 获取空闲QP
    │       │                       │       │
    │       │                       │       ├─► idr_alloc_cyclic()  // 分配QP ID
    │       │                       │       │
    │       │                       │       ├─► hisi_qm_unset_hw_reset()
    │       │                       │       │
    │       │                       │       ├─► memset(qp->cqe, 0, ...)  // 清零CQE
    │       │                       │       │   // ⚠️ 注意：没有清零SQE！
    │       │                       │       │
    │       │                       │       ├─► qp->req_cb = NULL
    │       │                       │       └─► qp->ref_count = 1
    │       │                       │
    │       │                       └─► qm_start_qp_nolock()  // 启动QP
    │       │                               │
    │       │                               ├─► qm_init_qp_status()  // 初始化状态
    │       │                               │       sq_tail = 0
    │       │                               │       cq_head = 0
    │       │                               │       cqc_phase = true (1)
    │       │                               │       used = 0
    │       │                               │
    │       │                               ├─► qm_sq_ctx_cfg()  // 配置SQ
    │       │                               └─► qm_cq_ctx_cfg()  // 配置CQ
    │       │                                       cqc.dw6 = 1 << QM_CQ_PHASE_SHIFT  // phase=1
    │       │
    │       └─► 设置 qp_ctx 字段
    │               qp_ctx->qp = qps[i]
    │               qp_ctx->ctx = hisi_zip_ctx
    │               qp_ctx->zip_dev = hisi_zip
    │
    ├─► hisi_zip_create_req_q()  // 创建请求队列
    │       │
    │       ├─► bitmap_alloc()  // 请求位图
    │       └─► kcalloc()       // hisi_zip_req 数组
    │
    ├─► hisi_zip_create_sgl_pool()  // 创建SGL内存池
    │
    └─► hisi_zip_set_acomp_cb(ctx, hisi_zip_acomp_cb)
            │
            └─► qp->req_cb = hisi_zip_acomp_cb  // 设置回调！
```

### 4.1 关键点：QP内存状态

```
                      ┌─────────────────────────────────────┐
                      │         QP内存 (qdma.va)            │
                      │                                     │
   分配时:            │  ┌─────────┬─────────┬─────────┐   │
   (未初始化)         │  │ SQE[0]  │ SQE[1]  │ ...     │   │
                      │  │ 随机值!  │ 随机值! │         │   │
                      │  └─────────┴─────────┴─────────┘   │
                      │  ┌─────────┬─────────┬─────────┐   │
                      │  │ CQE[0]  │ CQE[1]  │ ...     │   │
                      │  │ 随机值!  │ 随机值! │         │   │
                      │  └─────────┴─────────┴─────────┘   │
                      └─────────────────────────────────────┘

                      ┌─────────────────────────────────────┐
                      │         QP内存 (qdma.va)            │
                      │                                     │
   create_qp后:       │  ┌─────────┬─────────┬─────────┐   │
                      │  │ SQE[0]  │ SQE[1]  │ ...     │   │
                      │  │ 残留值!  │ 残留值! │         │   │ ← ⚠️ 未清零
                      │  │dw26/27  │         │         │   │
                      │  │有旧tag! │         │         │   │
                      │  └─────────┴─────────┴─────────┘   │
                      │  ┌─────────┬─────────┬─────────┐   │
                      │  │ CQE[0]  │ CQE[1]  │ ...     │   │
                      │  │  0x00   │  0x00   │  0x00   │   │ ← ✓ 已清零
                      │  └─────────┴─────────┴─────────┘   │
                      └─────────────────────────────────────┘

   qp_status:         sq_tail=0, cq_head=0, cqc_phase=1, used=0
```

---

## 五、压缩请求处理流程

### 5.1 提交请求

```
用户调用 crypto_acomp_compress()
    │
    ▼
hisi_zip_acompress(acomp_req)
    │
    ├─► 获取 ctx 和 qp_ctx
    │
    ├─► hisi_zip_create_req(qp_ctx, acomp_req)  // 分配请求
    │       │
    │       ├─► find_first_zero_bit()  // 找空闲req_id
    │       ├─► set_bit()              // 标记已用
    │       │
    │       └─► 设置 req_cache:
    │               req_cache->req_id = req_id
    │               req_cache->req = acomp_req       // 用户请求
    │               req_cache->qp_ctx = qp_ctx       // ← 关键！设置qp_ctx
    │
    ├─► hisi_zip_do_work(qp_ctx, req)
    │       │
    │       ├─► hisi_acc_sg_buf_map_to_hw_sgl()  // 映射源地址
    │       ├─► hisi_acc_sg_buf_map_to_hw_sgl()  // 映射目的地址
    │       │
    │       ├─► hisi_zip_fill_sqe()  // 填充SQE
    │       │       │
    │       │       ├─► fill_addr(sqe, req)      // DMA地址
    │       │       ├─► fill_buf_size(sqe, req)  // 数据长度
    │       │       ├─► fill_tag(sqe, req)       // ← 关键！
    │       │       │       sqe->dw26 = lower_32_bits(req)
    │       │       │       sqe->dw27 = upper_32_bits(req)
    │       │       │       // 把req指针存入SQE的tag字段
    │       │       │
    │       │       └─► fill_sqe_type()
    │       │
    │       └─► hisi_qp_send(qp, &zip_sqe)  // 发送到硬件
    │               │
    │               ├─► 检查队列状态
    │               ├─► memcpy(sq_tail位置, sqe, size)  // 写入SQE
    │               ├─► sq_tail++
    │               ├─► used++  // 增加使用计数
    │               │
    │               └─► 写doorbell通知硬件
    │
    └─► return -EINPROGRESS  // 异步返回
```

### 5.2 硬件处理

```
┌──────────────────────────────────────────────────────────────┐
│                        ZIP 硬件                               │
│                                                              │
│  1. 从SQ读取SQE                                              │
│  2. 解析SQE获取数据地址                                       │
│  3. 执行压缩/解压                                            │
│  4. 写入结果到CQE (包含sq_head索引)                           │
│  5. 触发完成中断 → EQ                                        │
└──────────────────────────────────────────────────────────────┘
```

### 5.3 中断处理和回调

```
硬件完成 → 触发EQ中断
    │
    ▼
qm_eq_irq()  // EQ中断处理
    │
    └─► qm_get_complete_eqe_num()
            │
            ├─► 从EQ读取完成事件
            │   eqe包含: cqn (完成队列号 = qp_id)
            │
            └─► queue_work(qm->wq, &poll_data->work)
                        │
                        ▼
                qm_work_process()
                        │
                        └─► qm_poll_req_cb(qp)
                                │
                                ├─► 获取 cqe = qp->cqe + cq_head
                                │
                                ├─► 检查 phase:
                                │   while (QM_CQE_PHASE(cqe) == qp->cqc_phase)
                                │
                                │   // CQE phase 与软件期望的phase比较
                                │   // 相等表示有新的完成项
                                │
                                ├─► 调用回调:
                                │   qp->req_cb(qp, qp->sqe + sqe_size * sq_head)
                                │       │
                                │       ▼
                                │   hisi_zip_acomp_cb(qp, sqe)
                                │       │
                                │       ├─► req = GET_REQ_FROM_SQE(sqe)
                                │       │       // 从SQE的dw26/dw27恢复req指针
                                │       │
                                │       ├─► qp_ctx = req->qp_ctx
                                │       │       // ← 这里可能崩溃！
                                │       │       // 如果SQE有残留数据，req可能是NULL或无效指针
                                │       │
                                │       ├─► unmap DMA
                                │       ├─► 调用用户回调
                                │       └─► hisi_zip_remove_req()
                                │
                                ├─► cq_head++
                                ├─► used--  // 减少使用计数
                                │
                                └─► 写doorbell更新CQ
```

---

## 六、CQE Phase 机制详解

### 6.1 Phase 的作用

CQE使用 **相位翻转** 机制来区分新旧完成项：

```
初始化时:
  cqc_phase = 1  (软件期望的phase)
  CQE中的phase = 1 (硬件写入)

CQE格式:
  ┌──────────────────────────────────────────────────┐
  │ w7: ... | phase (bit 0) | ...                    │
  └──────────────────────────────────────────────────┘

处理逻辑:
  while (QM_CQE_PHASE(cqe) == qp->cqc_phase) {
      // phase匹配，说明有新完成项
      process_cqe();
      cq_head++;
      if (cq_head == depth)
          cqc_phase = !cqc_phase;  // 队列回绕时翻转phase
  }
```

### 6.2 Phase 匹配问题

**问题场景**:
```
1. QP初始化:
   - cqc_phase = 1 (软件)
   - CQE内存清零后 phase = 0

2. 正常情况:
   - 硬件写入CQE时设置 phase = 1
   - 与软件 cqc_phase = 1 匹配
   - 正常处理

3. 异常情况 (虚假中断):
   - 硬件产生中断但CQE实际是旧的/无效的
   - 如果CQE的phase恰好是1
   - 与软件 cqc_phase = 1 匹配
   - 触发回调处理 ← 可能处理错误的SQE!
```

---

## 七、问题根因分析

### 7.1 问题场景重现

```
时间线:
───────────────────────────────────────────────────────────────►

T1: 首次加载模块
    insmod hisi_zip.ko
    │
    ├─► QP内存分配
    │   sqe/cqe 可能包含随机值
    │
    └─► 首次使用后卸载
        rmmod hisi_zip
        QP内存释放，但可能残留数据

T2: 再次加载模块
    insmod hisi_zip.ko
    │
    ├─► QP内存重新分配
    │   可能获得同一块内存
    │   SQE中可能有残留的 dw26/dw27 值
    │
    ├─► hisi_zip_acomp_init()
    │   │
    │   ├─► qm_create_qp_nolock()
    │   │   ├─► memset(cqe, 0, ...)  // CQE清零
    │   │   └─► SQE未清零！← 问题点
    │   │
    │   └─► hisi_zip_set_acomp_cb()
    │       qp->req_cb = hisi_zip_acomp_cb
    │
    └─► 硬件可能产生虚假/残留中断
        │
        └─► qm_poll_req_cb()
            │
            ├─► used == 0 (没有发送过请求)
            │   但 CQE phase == 1 == cqc_phase
            │   // 匹配！进入处理
            │
            └─► 调用 hisi_zip_acomp_cb(qp, sqe)
                    │
                    ├─► req = GET_REQ_FROM_SQE(sqe)
                    │   // 从残留的 dw26/dw27 恢复地址
                    │   // 得到无效/NULL指针
                    │
                    └─► qp_ctx = req->qp_ctx
                        // ← KERNEL PANIC!
                        // req是NULL或无效指针
```

### 7.2 代码证据

**1. SQE未清零**:
```c
// qm.c: qm_create_qp_nolock()
qp = &qm->qp_array[qp_id];
hisi_qm_unset_hw_reset(qp);
memset(qp->cqe, 0, sizeof(struct qm_cqe) * qp->cq_depth);  // 只清零CQE
// ⚠️ 没有清零SQE！

qp->event_cb = NULL;
qp->req_cb = NULL;  // 回调设为NULL
```

**2. 回调可能在设置后立即被调用**:
```c
// zip_crypto.c: hisi_zip_acomp_init()
hisi_zip_set_acomp_cb(ctx, hisi_zip_acomp_cb);  // 设置回调
// 回调设置后，如果硬件有残留中断，可能立即被调用
```

**3. GET_REQ_FROM_SQE 从SQE恢复指针**:
```c
// zip_crypto.c
#define GET_REQ_FROM_SQE(sqe) ((u64)(sqe)->dw26 | (u64)(sqe)->dw27 << 32)

static void hisi_zip_acomp_cb(struct hisi_qp *qp, void *data)
{
    struct hisi_zip_req *req = (struct hisi_zip_req *)GET_REQ_FROM_SQE(sqe);
    struct hisi_zip_qp_ctx *qp_ctx = req->qp_ctx;  // ← 如果req无效，这里崩溃
}
```

---

## 八、修复方案

### 方案1: 清零SQE内存 (推荐)

```c
// 在 qm_create_qp_nolock() 中添加:
memset(qp->sqe, 0, qm->sqe_size * qp->sq_depth);
```

### 方案2: 检测虚假中断

```c
// 在 qm_poll_req_cb() 中添加:
if (atomic_read(&qp->qp_status.used) == 0) {
    // 没有发送过请求，不应该有完成项
    dev_warn(..., "Spurious interrupt detected\n");
    return;  // 跳过处理
}
```

### 方案3: 回调中验证指针

```c
// 在 hisi_zip_acomp_cb() 中添加:
if (!req || IS_ERR_VALUE((unsigned long)req) || !req->qp_ctx) {
    dev_err(..., "Invalid request pointer from SQE\n");
    return;
}
```

---

## 九、调试建议

1. **检查SQE残留数据**: 在 `qm_create_qp_nolock` 中打印 SQE 的 dw26/dw27
2. **检测虚假中断**: 在 `qm_poll_req_cb` 中检查 `used` 计数
3. **复现条件**: 重复 insmod/rmmod，或硬件复位后立即加载