# HiSilicon ZIP 驱动架构分析

## 一、崩溃现场分析

### 1.1 关键寄存器信息

```
崩溃地址: ffff0800007f0028
崩溃指令: hisi_zip_acomp_cb+0x38
错误类型: level 1 translation fault (页面未映射)

关键寄存器:
  x20 = ffff0800007f0000   ← req 指针 (从 SQE tag 恢复)
  x0  = ffff08015fa4c880   ← qp 指针
  
计算: ffff0800007f0000 + 0x28 = ffff0800007f0028
      这正是 req->qp_ctx 的访问地址！
```

### 1.2 崩溃点定位

```c
// zip_crypto.c: hisi_zip_acomp_cb
static void hisi_zip_acomp_cb(struct hisi_qp *qp, void *data)
{
    struct hisi_zip_sqe *sqe = data;
    struct hisi_zip_req *req = (struct hisi_zip_req *)GET_REQ_FROM_SQE(sqe);
//  ↑ +0x10: 获取 req 指针

    struct hisi_zip_qp_ctx *qp_ctx = req->qp_ctx;
//  ↑ +0x38: 访问 req->qp_ctx ← 崩溃点！
//           偏移 0x28 (req 结构体中 qp_ctx 的偏移)
//           指令: ldr x?, [x20, #0x28]
}
```

---

## 二、整体架构

### 2.1 分层架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户空间                                        │
│                      (应用程序: 压缩/解压请求)                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ crypto API (acomp_register/acomp_request)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Crypto 子系统层                                    │
│                     (crypto/acompress.h, crypto API)                        │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ struct acomp_alg {                                                  │   │
│   │     .init      = hisi_zip_acomp_init,    // 打开设备时调用          │   │
│   │     .exit      = hisi_zip_acomp_exit,    // 关闭设备时调用          │   │
│   │     .compress  = hisi_zip_acompress,     // 压缩请求                │   │
│   │     .decompress= hisi_zip_adecompress,   // 解压请求                │   │
│   │ }                                                                   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           hisi_zip 驱动层                                    │
│                    (drivers/crypto/hisilicon/zip/)                         │
│                                                                             │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐            │
│   │   zip_main.c    │  │  zip_crypto.c   │  │     zip.h       │            │
│   │                 │  │                 │  │                 │            │
│   │ - 模块初始化    │  │ - 算法实现      │  │ - 数据结构定义  │            │
│   │ - PCI probe     │  │ - 请求处理      │  │ - 常量定义      │            │
│   │ - QM 启动/停止  │  │ - 回调处理      │  │                 │            │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘            │
│                                                                             │
│   核心数据结构:                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ hisi_zip_ctx (算法上下文)                                           │   │
│   │   ├── qp_ctx[0]: 压缩 QP 上下文                                     │   │
│   │   ├── qp_ctx[1]: 解压 QP 上下文                                     │   │
│   │   └── ops: SQE 操作函数表                                           │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           QM 硬件管理层                                      │
│                    (drivers/crypto/hisilicon/qm.c)                         │
│                                                                             │
│   功能:                                                                     │
│   - 队列对(QP) 管理: 创建、启动、停止、销毁                                  │
│   - 中断处理: EQ(事件队列)、AEQ(异步事件队列)                                │
│   - DMA 内存管理: SQE/CQE 内存分配                                          │
│   - 硬件寄存器访问: doorbell、mailbox                                        │
│   - 硬件状态管理: 启动、停止、复位                                           │
│                                                                             │
│   核心数据结构:                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ hisi_qm (设备级管理结构)                                            │   │
│   │   ├── qp_array[]: 所有 QP 的数组                                    │   │
│   │   ├── sqc/cqc: 队列上下文表 (DMA)                                   │   │
│   │   ├── eqe/aeqe: 事件队列 (DMA)                                      │   │
│   │   └── 中断、锁、工作队列等                                           │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ hisi_qp (队列对结构 - 一个硬件队列)                                  │   │
│   │   ├── sqe: 提交队列入口数组 (DMA)                                   │   │
│   │   ├── cqe: 完成队列入口数组 (DMA)                                   │   │
│   │   ├── qp_status: 状态 (sq_tail, cq_head, cqc_phase, used)          │   │
│   │   └── req_cb: 请求完成回调函数                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ MMIO (内存映射 I/O)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ZIP 硬件                                          │
│                    (PCIe 设备: 0000:1d:00.0)                               │
│                                                                             │
│   功能:                                                                     │
│   - 压缩/解压加速                                                           │
│   - 支持多种算法: deflate, zlib, gzip, lz77, lz4                           │
│   - 多队列并行处理                                                          │
│                                                                             │
│   硬件队列机制:                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │   SQ (提交队列) ──► 硬件处理 ──► CQ (完成队列) ──► 中断             │   │
│   │        │                              │                             │   │
│   │        │                              │                             │   │
│   │        ▼                              ▼                             │   │
│   │   [SQE] [SQE] [SQE] ...         [CQE] [CQE] [CQE] ...              │   │
│   │   包含请求参数                    包含完成状态                       │   │
│   │   - 数据地址                      - sq_head (对应的 SQE 索引)        │   │
│   │   - 数据长度                      - phase (相位标志)                 │   │
│   │   - tag (请求标识)                - 状态码                           │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 三、核心数据结构详解

### 3.1 hisi_zip_ctx - 算法上下文

```c
// 每个 crypto_tfm 对应一个 hisi_zip_ctx
// 在 hisi_zip_acomp_init() 中初始化

struct hisi_zip_ctx {
    struct hisi_zip_qp_ctx qp_ctx[HZIP_CTX_Q_NUM];  // 2个QP上下文
                                                    // [0] = 压缩
                                                    // [1] = 解压
    const struct hisi_zip_sqe_ops *ops;             // SQE 操作函数表
    struct crypto_comp *soft_tfm;                   // 软件回退 tfm
    bool fallback;                                   // 是否使用软件回退
};
```

### 3.2 hisi_zip_qp_ctx - QP 上下文

```c
struct hisi_zip_qp_ctx {
    struct hisi_qp *qp;                    // 指向 QM 层的 QP
    struct hisi_zip_req_q req_q;           // 请求队列 (预分配的请求数组)
    struct hisi_acc_sgl_pool *sgl_pool;    // DMA SGL 内存池
    struct hisi_zip *zip_dev;              // 设备结构
    struct hisi_zip_ctx *ctx;              // 所属的算法上下文
    u8 req_type;                           // 请求类型
};
```

### 3.3 hisi_zip_req - 请求结构 (关键!)

```c
// 每个 pending 请求对应一个 hisi_zip_req
// 这是崩溃时访问的结构！

struct hisi_zip_req {
    struct acomp_req *req;           // +0x00: 用户的压缩请求
    struct hisi_acc_hw_sgl *hw_src;  // +0x08: 源数据 DMA 地址
    struct hisi_acc_hw_sgl *hw_dst;  // +0x10: 目的数据 DMA 地址
    dma_addr_t dma_src;              // +0x18: 源 DMA 物理地址
    dma_addr_t dma_dst;              // +0x20: 目的 DMA 物理地址
    struct hisi_zip_qp_ctx *qp_ctx;  // +0x28: 所属 QP 上下文 ← 崩溃访问位置!
    u16 req_id;                      // +0x30: 请求 ID
};
```

### 3.4 hisi_zip_req_q - 请求队列

```c
struct hisi_zip_req_q {
    struct hisi_zip_req *q;          // 预分配的请求数组
    unsigned long *req_bitmap;       // 已用请求位图
    spinlock_t req_lock;             // 自旋锁
    u16 size;                        // 队列深度
};
```

### 3.5 hisi_qp - 硬件队列对 (QM 层)

```c
struct hisi_qp {
    u32 qp_id;                       // QP ID
    u16 sq_depth;                    // SQ 深度
    u16 cq_depth;                    // CQ 深度
    u8 alg_type;                     // 算法类型

    struct qm_dma qdma;              // DMA 内存
    void *sqe;                       // SQE 数组 (虚拟地址)
    struct qm_cqe *cqe;              // CQE 数组 (虚拟地址)
    dma_addr_t sqe_dma;              // SQE 物理地址
    dma_addr_t cqe_dma;              // CQE 物理地址

    struct hisi_qp_status qp_status;
        // atomic_t used;            // 正在处理的请求数
        // u16 sq_tail;              // SQ 写指针
        // u16 cq_head;              // CQ 读指针
        // bool cqc_phase;           // CQE 相位标志

    void (*req_cb)(struct hisi_qp *qp, void *data);  // 请求完成回调

    struct hisi_qm *qm;              // 所属 QM
    bool is_resetting;
    bool is_in_kernel;
    u32 ref_count;
};
```

### 3.6 hisi_zip_sqe - SQE 格式 (关键!)

```c
// SQE (Submission Queue Entry) - 提交队列入口
// 大小: 128 字节

struct hisi_zip_sqe {
    u32 consumed;                    // +0x00: 已消费字节数
    u32 produced;                    // +0x04: 已生产字节数
    u32 dw3;                         // +0x08: 状态 (bits 0-7)
    u32 input_data_length;           // +0x0C: 输入数据长度
    
    // ... 其他字段 ...
    
    u32 source_addr_l;               // +0x48: 源地址低32位
    u32 source_addr_h;               // +0x4C: 源地址高32位
    u32 dest_addr_l;                 // +0x50: 目的地址低32位
    u32 dest_addr_h;                 // +0x54: 目的地址高32位
    
    // ... 其他字段 ...
    
    u32 dw26;                        // +0x68: TAG 低32位 ← 存放 hisi_zip_req 指针!
    u32 dw27;                        // +0x6C: TAG 高32位 ← 存放 hisi_zip_req 指针!
};
```

---

## 四、关键宏定义

### 4.1 GET_REQ_FROM_SQE (崩溃根源!)

```c
// 从 SQE 的 dw26/dw27 恢复 hisi_zip_req 指针
#define GET_REQ_FROM_SQE(sqe)  ((u64)(sqe)->dw26 | (u64)(sqe)->dw27 << 32)

// 问题: 如果 dw26/dw27 包含无效/残留数据，恢复出的指针就是无效的！
```

### 4.2 CQE Phase 机制

```c
// CQE phase 用于区分新的完成项
#define QM_CQE_PHASE(cqe)  (le16_to_cpu((cqe)->w7) & 0x1)

// 处理逻辑:
// while (QM_CQE_PHASE(cqe) == qp->cqc_phase) {
//     处理 CQE...
//     cq_head++;
//     if (cq_head == depth)
//         cqc_phase = !cqc_phase;  // 回绕时翻转
// }
```

---

## 五、请求处理流程

### 5.1 发送请求流程

```
用户调用: crypto_acomp_compress(req)
    │
    ▼
hisi_zip_acompress(acomp_req)
    │
    ├─► 获取 hisi_zip_ctx 和 qp_ctx
    │
    ├─► hisi_zip_create_req(qp_ctx, acomp_req)
    │       │
    │       ├─► find_first_zero_bit() → 找空闲 req_id
    │       ├─► set_bit() → 标记已用
    │       │
    │       └─► 初始化 hisi_zip_req:
    │               req_cache->req_id = req_id
    │               req_cache->req = acomp_req
    │               req_cache->qp_ctx = qp_ctx  ← 关键: 设置 qp_ctx
    │
    ├─► hisi_zip_do_work(qp_ctx, req)
    │       │
    │       ├─► hisi_acc_sg_buf_map_to_hw_sgl() → 映射源地址
    │       ├─► hisi_acc_sg_buf_map_to_hw_sgl() → 映射目的地址
    │       │
    │       ├─► hisi_zip_fill_sqe()
    │       │       │
    │       │       ├─► memset(&zip_sqe, 0, ...) → 清零栈上 SQE
    │       │       ├─► fill_addr() → 填充 DMA 地址
    │       │       ├─► fill_buf_size() → 填充数据长度
    │       │       ├─► fill_tag() → 填充 tag (req 指针)
    │       │       │       sqe->dw26 = lower_32_bits(req)
    │       │       │       sqe->dw27 = upper_32_bits(req)
    │       │       │
    │       │       └─► fill_sqe_type()
    │       │
    │       └─► hisi_qp_send(qp, &zip_sqe)
    │               │
    │               ├─► memcpy(qp->sqe + sq_tail, &zip_sqe, size) → 拷贝到 QP 内存
    │               ├─► sq_tail++
    │               ├─► used++  ← 增加使用计数
    │               │
    │               └─► 写 doorbell 通知硬件
    │
    └─► return -EINPROGRESS
```

### 5.2 中断处理流程

```
硬件完成 → 触发 EQ 中断
    │
    ▼
qm_eq_irq()  // EQ 中断处理
    │
    └─► qm_get_complete_eqe_num()
            │
            ├─► 从 EQ 读取完成事件 (eqe)
            │   eqe 包含: cqn (完成队列号 = qp_id)
            │
            └─► queue_work(qm->wq, &poll_data->work)

                        ▼
                qm_work_process()
                        │
                        └─► qm_poll_req_cb(qp)
                                │
                                ├─► cqe = qp->cqe + cq_head
                                │
                                ├─► 检查 phase:
                                │   while (QM_CQE_PHASE(cqe) == qp->cqc_phase)
                                │
                                ├─► sq_head = le16_to_cpu(cqe->sq_head)
                                │
                                ├─► 调用回调:
                                │   qp->req_cb(qp, qp->sqe + sqe_size * sq_head)
                                │
                                │       ▼
                                │   hisi_zip_acomp_cb(qp, sqe)
                                │       │
                                │       ├─► req = GET_REQ_FROM_SQE(sqe)
                                │       │       // 从 dw26/dw27 恢复 req 指针
                                │       │
                                │       ├─► qp_ctx = req->qp_ctx
                                │       │       // ← 崩溃点！
                                │       │       // 如果 req 指针无效
                                │       │
                                │       ├─► unmap DMA
                                │       ├─► 调用用户回调
                                │       └─► hisi_zip_remove_req()
                                │
                                ├─► cq_head++
                                ├─► used--  ← 减少使用计数
                                │
                                └─► 写 doorbell 更新 CQ
```

---

## 六、内存布局

### 6.1 QP DMA 内存布局

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        QP DMA 内存 (qdma.va)                                │
│                        大小: sqe_size * sq_depth + cqe_size * cq_depth      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   偏移 0                                                                    │
│   ▼                                                                         │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         SQE 区域                                     │   │
│   │                                                                     │   │
│   │   ┌──────────┬──────────┬──────────┬──────────┬─────┐              │   │
│   │   │  SQE[0]  │  SQE[1]  │  SQE[2]  │  SQE[3]  │ ... │              │   │
│   │   │  128 字节 │  128 字节 │  128 字节 │  128 字节 │     │              │   │
│   │   │          │          │          │          │     │              │   │
│   │   │ dw26/27  │          │          │          │     │              │   │
│   │   │ = tag    │          │          │          │     │              │   │
│   │   └──────────┴──────────┴──────────┴──────────┴─────┘              │   │
│   │                                                                     │   │
│   │   大小: sqe_size * sq_depth                                         │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   偏移: sqe_size * sq_depth                                                 │
│   ▼                                                                         │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         CQE 区域                                     │   │
│   │                                                                     │   │
│   │   ┌──────────┬──────────┬──────────┬──────────┬─────┐              │   │
│   │   │  CQE[0]  │  CQE[1]  │  CQE[2]  │  CQE[3]  │ ... │              │   │
│   │   │  32 字节  │  32 字节  │  32 字节  │  32 字节  │     │              │   │
│   │   │          │          │          │          │     │              │   │
│   │   │ w7:phase │          │          │          │     │              │   │
│   │   └──────────┴──────────┴──────────┴──────────┴─────┘              │   │
│   │                                                                     │   │
│   │   大小: cqe_size * cq_depth                                         │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 数据结构关系图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           hisi_zip 模块层                                    │
│                                                                             │
│   ┌───────────────────────────────────────────────────────────────────┐     │
│   │                     hisi_zip_ctx                                   │     │
│   │                                                                   │     │
│   │   qp_ctx[0]                    qp_ctx[1]                          │     │
│   │   ┌─────────────────────┐      ┌─────────────────────┐            │     │
│   │   │ hisi_zip_qp_ctx     │      │ hisi_zip_qp_ctx     │            │     │
│   │   │                     │      │                     │            │     │
│   │   │  qp ────────────────────────► qp (共享或独立)     │            │     │
│   │   │  req_q ─────────────┐      │  req_q              │            │     │
│   │   │  sgl_pool           │      │  sgl_pool           │            │     │
│   │   │  ctx ───────────────┼──────► ctx (指向自己)      │            │     │
│   │   └─────────────────────┘      └─────────────────────┘            │     │
│   │           │                                                       │     │
│   └───────────┼───────────────────────────────────────────────────────┘     │
│               │                                                             │
│               ▼                                                             │
│   ┌───────────────────────────────────────────────────────────────────┐     │
│   │                   hisi_zip_req_q                                   │     │
│   │                                                                   │     │
│   │   ┌───────────────────────────────────────────────────────────┐   │     │
│   │   │                    hisi_zip_req[]                          │   │     │
│   │   │                                                           │   │     │
│   │   │  [0]        [1]        [2]        [3]           [n]       │   │     │
│   │   │  ┌────┐    ┌────┐    ┌────┐    ┌────┐         ┌────┐     │   │     │
│   │   │  │req │    │req │    │req │    │req │   ...   │req │     │   │     │
│   │   │  │    │    │    │    │    │    │    │         │    │     │   │     │
│   │   │  │qp_ │    │    │    │    │    │    │         │    │     │   │     │
│   │   │  │ctx │    │    │    │    │    │    │         │    │     │   │     │
│   │   │  └────┘    └────┘    └────┘    └────┘         └────┘     │   │     │
│   │   │                                                           │   │     │
│   │   │  req_bitmap: 标记哪些 req 已使用                          │   │     │
│   │   └───────────────────────────────────────────────────────────┘   │     │
│   │                                                                   │     │
│   └───────────────────────────────────────────────────────────────────┘     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            QM 层                                             │
│                                                                             │
│   ┌───────────────────────────────────────────────────────────────────┐     │
│   │                      hisi_qp                                       │     │
│   │                                                                   │     │
│   │   ┌─────────────────────────────────────────────────────────┐     │     │
│   │   │                    qp_status                              │     │     │
│   │   │  sq_tail = 0    cq_head = 0    cqc_phase = 1             │     │     │
│   │   │  used = 0                                                  │     │     │
│   │   └─────────────────────────────────────────────────────────┘     │     │
│   │                                                                   │     │
│   │   ┌─────────────────────────────────────────────────────────┐     │     │
│   │   │              qdma (DMA 内存)                              │     │     │
│   │   │                                                         │     │     │
│   │   │   sqe ──────────────────────────────► SQE 数组          │     │     │
│   │   │   cqe ──────────────────────────────► CQE 数组          │     │     │
│   │   │                                                         │     │     │
│   │   │   分配时: dma_alloc_coherent()                           │     │     │
│   │   │   内容: 可能有残留数据!                                   │     │     │
│   │   └─────────────────────────────────────────────────────────┘     │     │
│   │                                                                   │     │
│   │   req_cb ─────────────────► hisi_zip_acomp_cb                   │     │
│   │                                                                   │     │
│   └───────────────────────────────────────────────────────────────────┘     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 七、关键代码路径

### 7.1 QP 创建路径 (SQE 未清零!)

```c
// qm.c: qm_create_qp_nolock()

static struct hisi_qp *qm_create_qp_nolock(struct hisi_qm *qm, u8 alg_type, bool is_in_kernel)
{
    // ...
    
    qp = &qm->qp_array[qp_id];
    hisi_qm_unset_hw_reset(qp);
    
    // ★ 只清零 CQE，不清零 SQE！★
    memset(qp->cqe, 0, sizeof(struct qm_cqe) * qp->cq_depth);
    // memset(qp->sqe, 0, qm->sqe_size * qp->sq_depth);  // 这行不存在！
    
    qp->event_cb = NULL;
    qp->req_cb = NULL;  // 回调设为 NULL
    // ...
}
```

### 7.2 回调设置路径

```c
// zip_crypto.c: hisi_zip_set_acomp_cb()

static void hisi_zip_set_acomp_cb(struct hisi_zip_ctx *ctx,
                                  void (*fn)(struct hisi_qp *, void *))
{
    int i;
    
    for (i = 0; i < HZIP_CTX_Q_NUM; i++)
        ctx->qp_ctx[i].qp->req_cb = fn;  // 设置回调
}
```

### 7.3 崩溃路径

```c
// zip_crypto.c: hisi_zip_acomp_cb()

static void hisi_zip_acomp_cb(struct hisi_qp *qp, void *data)
{
    struct hisi_zip_sqe *sqe = data;
    struct hisi_zip_req *req = (struct hisi_zip_req *)GET_REQ_FROM_SQE(sqe);
    //                               ↑ 从 dw26/dw27 恢复 req 指针
    
    struct hisi_zip_qp_ctx *qp_ctx = req->qp_ctx;
    //                               ↑ 访问 req->qp_ctx
    //                               ★ 如果 req 指针无效，这里崩溃！★
    // ...
}
```

---

## 八、问题总结

### 8.1 问题条件

崩溃需要同时满足以下条件：

1. **SQE 有残留 tag 数据** - rmmod 时 dw26/dw27 有有效值
2. **内存复用** - insmod 时 dma_alloc_coherent 分配到同一块内存
3. **QP 创建** - 用户打开设备触发 `hisi_zip_acomp_init()`
4. **回调已设置** - `hisi_zip_set_acomp_cb()` 已执行
5. **硬件产生中断** - 在请求发送前产生完成中断
6. **Phase 匹配** - CQE phase 与 cqc_phase 匹配

### 8.2 为什么复现困难？

条件 5 (硬件产生中断) 是最不可控的因素。正常情况下，硬件不会在没有发送请求的情况下产生完成中断。

---

## 九、下一步

第二部分: 分析哪些代码路径可能改写 dw26/dw27
第三部分: 设计复现方案