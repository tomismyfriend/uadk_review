# hisi_zip 驱动 Kernel Panic 根因分析

## 一、问题概述

### 1.1 故障现象

```
[271533.597375][ T4297] Unable to handle kernel paging request at virtual address ffff0800007f0028
[271533.633127][ T4297] FSC = 0x05: level 1 translation fault
[271533.761703][ T4297] pc : hisi_zip_acomp_cb+0x38/0x1a8 [hisi_zip]
[271533.775918][ T4297] lr : qm_work_process+0x13c/0x2d8 [hisi_qm]
```

### 1.2 关键证据

| 项目 | 值 | 说明 |
|------|-----|------|
| 崩溃地址 | `ffff0800007f0028` | 无效的内核虚拟地址 |
| x20 (req) | `ffff0800007f0000` | 从 SQE tag 恢复的 req 指针 |
| pud | `0x0000000000000000` | 页表项为空，地址未映射 |
| FSC | `0x05` | Level 1 translation fault |

---

## 二、代码架构分析

### 2.1 分层架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户空间                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │ crypto API
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        hisi_zip 驱动层 (业务逻辑)                            │
│                        drivers/crypto/hisilicon/zip/                        │
│                                                                             │
│   职责:                                                                      │
│   - 算法注册与请求处理                                                       │
│   - req 池管理                                                              │
│   - SQE 填充与 tag 写入                                                     │
│   - 回调处理                                                                │
│                                                                             │
│   ★ 本层代码逻辑正确，问题不在这一层 ★                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │ QM API
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          QM 层 (队列管理)                                    │
│                        drivers/crypto/hisilicon/qm.c                        │
│                                                                             │
│   职责:                                                                      │
│   - QP (队列对) 创建与管理                                                   │
│   - DMA 内存分配 (SQE/CQE)                                                  │
│   - 中断处理与回调分发                                                       │
│                                                                             │
│   ★ 问题点: qm_create_qp_nolock() 只清零 CQE，不清零 SQE ★                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │ MMIO
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            ZIP 硬件                                          │
│                                                                             │
│   ★ 问题点: 可能在请求发送前产生异常中断 ★                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 三、hisi_zip 层代码正确性证明

### 3.1 Req 池创建流程

**代码位置:** `zip_crypto.c:451-491`

```c
static int hisi_zip_create_req_q(struct hisi_zip_ctx *ctx)
{
    u16 q_depth = ctx->qp_ctx[0].qp->sq_depth;
    struct hisi_zip_req_q *req_q;
    int i, ret;

    for (i = 0; i < HZIP_CTX_Q_NUM; i++) {
        req_q = &ctx->qp_ctx[i].req_q;
        req_q->size = q_depth;

        // 1. 分配位图 (用于跟踪 req 使用状态)
        req_q->req_bitmap = bitmap_zalloc(req_q->size, GFP_KERNEL);

        // 2. 初始化自旋锁
        spin_lock_init(&req_q->req_lock);

        // 3. 分配 req 数组
        // ★ kcalloc 会清零内存，所有字段初始为 0/NULL ★
        req_q->q = kcalloc(req_q->size, sizeof(struct hisi_zip_req),
                           GFP_KERNEL);
    }
    return 0;
}
```

**正确性分析:**

| 检查项 | 状态 | 说明 |
|--------|------|------|
| 内存分配 | ✓ | 使用 `kcalloc()`，内存会被清零 |
| 并发保护 | ✓ | 使用自旋锁保护 bitmap 操作 |
| 大小计算 | ✓ | 与 QP 深度一致 |

### 3.2 Req 获取流程

**代码位置:** `zip_crypto.c:143-169`

```c
static struct hisi_zip_req *hisi_zip_create_req(struct hisi_zip_qp_ctx *qp_ctx,
                                                 struct acomp_req *req)
{
    struct hisi_zip_req_q *req_q = &qp_ctx->req_q;
    struct hisi_zip_req *q = req_q->q;
    struct hisi_zip_req *req_cache;
    int req_id;

    spin_lock(&req_q->req_lock);

    // 1. 查找空闲 slot
    req_id = find_first_zero_bit(req_q->req_bitmap, req_q->size);
    if (req_id >= req_q->size) {
        spin_unlock(&req_q->req_lock);
        return ERR_PTR(-EAGAIN);
    }

    // 2. 标记为已使用
    set_bit(req_id, req_q->req_bitmap);

    spin_unlock(&req_q->req_lock);

    // 3. 计算 req 地址
    // ★ req_cache = 数组基地址 + req_id ★
    // ★ 地址一定在 kcalloc 分配的范围内 ★
    req_cache = q + req_id;

    // 4. 初始化关键字段
    req_cache->req_id = req_id;
    req_cache->req = req;
    req_cache->qp_ctx = qp_ctx;  // ← 设置 qp_ctx 指针

    return req_cache;
}
```

**正确性分析:**

| 检查项 | 状态 | 说明 |
|--------|------|------|
| 空闲查找 | ✓ | 使用 bitmap 原子操作 |
| 地址计算 | ✓ | `q + req_id` 保证在数组范围内 |
| 字段初始化 | ✓ | 正确设置 req_id, req, qp_ctx |
| 返回值 | ✓ | 返回有效的内核地址 |

### 3.3 Tag 写入流程

**代码位置:** `zip_crypto.c:215-219`

```c
static void hisi_zip_fill_tag(struct hisi_zip_sqe *sqe, struct hisi_zip_req *req)
{
    // 将 req 指针拆分为高低 32 位存入 SQE
    sqe->dw26 = lower_32_bits((u64)req);
    sqe->dw27 = upper_32_bits((u64)req);
}
```

**代码位置:** `zip_crypto.c:230-243`

```c
static void hisi_zip_fill_sqe(struct hisi_zip_ctx *ctx, struct hisi_zip_sqe *sqe,
                               u8 req_type, struct hisi_zip_req *req)
{
    const struct hisi_zip_sqe_ops *ops = ctx->ops;

    // ★ 关键: 先清零整个 SQE ★
    memset(sqe, 0, sizeof(struct hisi_zip_sqe));

    // 然后填充各个字段
    ops->fill_addr(sqe, req);
    ops->fill_buf_size(sqe, req);
    ops->fill_buf_type(sqe, HZIP_SGL);
    ops->fill_req_type(sqe, req_type);
    ops->fill_tag(sqe, req);      // ← 写入 tag
    ops->fill_sqe_type(sqe, ops->sqe_type);
}
```

**正确性分析:**

| 检查项 | 状态 | 说明 |
|--------|------|------|
| SQE 清零 | ✓ | `memset(sqe, 0, ...)` 在填充前执行 |
| tag 写入 | ✓ | 正确拆分 64 位指针 |
| 调用顺序 | ✓ | 先清零后填充 |

### 3.4 请求发送流程

**代码位置:** `zip_crypto.c:245-291`

```c
static int hisi_zip_do_work(struct hisi_zip_qp_ctx *qp_ctx,
                             struct hisi_zip_req *req)
{
    // ... DMA 映射 ...

    // 1. 在栈上填充 SQE
    hisi_zip_fill_sqe(qp_ctx->ctx, &zip_sqe, qp_ctx->req_type, req);

    // 2. 发送到硬件
    // ★ hisi_qp_send 会 memcpy 到 QP 的 DMA 内存 ★
    ret = hisi_qp_send(qp, &zip_sqe);

    return -EINPROGRESS;
}
```

**正确性分析:**

| 检查项 | 状态 | 说明 |
|--------|------|------|
| SQE 来源 | ✓ | 栈上局部变量，已清零 |
| 发送时机 | ✓ | 只有在用户请求时才发送 |
| 数据拷贝 | ✓ | memcpy 到 QP 内存 |

---

## 四、QM 层问题分析

### 4.1 QP 创建时 SQE 未清零

**代码位置:** `qm.c:2133-2168`

```c
static struct hisi_qp *qm_create_qp_nolock(struct hisi_qm *qm, u8 alg_type,
                                            bool is_in_kernel)
{
    // ...

    qp = &qm->qp_array[qp_id];
    hisi_qm_unset_hw_reset(qp);

    // ★ 问题: 只清零 CQE，不清零 SQE！★
    memset(qp->cqe, 0, sizeof(struct qm_cqe) * qp->cq_depth);
    // 缺少: memset(qp->sqe, 0, qm->sqe_size * qp->sq_depth);

    qp->event_cb = NULL;
    qp->req_cb = NULL;
    // ...
}
```

**问题分析:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         QP 创建时的内存状态                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  DMA 内存布局:                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  SQE 区域                      │  CQE 区域                          │   │
│  │  (未清零，可能有残留数据)       │  (已清零)                          │   │
│  │                                │                                    │   │
│  │  [SQE0] [SQE1] ... [SQEn]      │  [CQE0] [CQE1] ... [CQEn]         │   │
│  │  ↑                             │  ↑                                 │   │
│  │  dw26/dw27 可能有残留 tag!     │  全部为 0                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  残留数据来源:                                                              │
│  1. 上次 rmmod 时 DMA 内存未清零                                           │
│  2. 再次 insmod 时 dma_alloc_coherent 可能分配到同一块内存                  │
│  3. SQE 中保留着上次的 tag 值                                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 回调触发时机问题

**代码位置:** `qm.c:1052-1063`

```c
static void qm_poll_req_cb(struct hisi_qp *qp)
{
    struct qm_cqe *cqe = qp->cqe + qp->qp_status.cq_head;
    struct hisi_qm *qm = qp->qm;

    // ★ 问题: 如果 CQE phase 匹配，就会调用回调 ★
    // ★ 但此时可能还没有发送过任何请求！★
    while (QM_CQE_PHASE(cqe) == qp->qp_status.cqc_phase) {
        dma_rmb();
        qp->req_cb(qp, qp->sqe + qm->sqe_size *
                   le16_to_cpu(cqe->sq_head));  // ← 读取 SQE
        // ...
    }
}
```

**问题分析:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         问题触发时序                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  正常时序:                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  T1: QP 创建 (SQE 有残留数据)                                       │   │
│  │  T2: 回调设置 (req_cb = hisi_zip_acomp_cb)                          │   │
│  │  T3: 用户发送请求 (SQE 被清零并写入正确 tag)                        │   │
│  │  T4: 硬件处理请求                                                   │   │
│  │  T5: 中断触发，回调读取正确的 tag                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  问题时序:                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  T1: QP 创建 (SQE 有残留数据)                                       │   │
│  │  T2: 回调设置 (req_cb = hisi_zip_acomp_cb)                          │   │
│  │  T3: ★ 硬件异常产生中断 ★                                          │   │
│  │  T4: 回调读取残留 tag → 无效指针 → 崩溃！                           │   │
│  │  (用户请求还未发送)                                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 五、证据链总结

### 5.1 崩溃地址分析

```
崩溃地址: ffff0800007f0028
req 指针: ffff0800007f0000 (从 SQE tag 恢复)

正常 req 地址范围: ffff8000xxxxxxxx (kcalloc 分配)
异常 req 地址:     ffff0800xxxxxxxx (不在有效范围内)

结论: req 指针不是 hisi_zip_create_req() 返回的地址！
```

### 5.2 代码正确性证明

| 层级 | 模块 | 正确性 | 说明 |
|------|------|--------|------|
| hisi_zip | req 池创建 | ✓ | kcalloc 清零内存 |
| hisi_zip | req 获取 | ✓ | 正确的地址计算 |
| hisi_zip | tag 写入 | ✓ | 先清零后填充 |
| hisi_zip | 请求发送 | ✓ | 正确的发送时机 |
| **QM** | **QP 创建** | **✗** | **SQE 未清零** |
| **QM** | **回调触发** | **✗** | **无请求有效性检查** |

### 5.3 根因定位

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              根因链                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. QM 层: qm_create_qp_nolock() 不清零 SQE                                │
│     ↓                                                                       │
│  2. DMA 内存复用: 上次使用的 tag 残留在 SQE 中                              │
│     ↓                                                                       │
│  3. 硬件异常: 在用户请求前产生中断                                          │
│     ↓                                                                       │
│  4. QM 层: qm_poll_req_cb() 无条件调用回调                                  │
│     ↓                                                                       │
│  5. hisi_zip: hisi_zip_acomp_cb() 读取残留 tag                             │
│     ↓                                                                       │
│  6. 访问无效地址 ffff0800007f0000 + 0x28                                    │
│     ↓                                                                       │
│  7. Kernel Panic                                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 六、结论

### 6.1 hisi_zip 层代码无问题

1. **Req 池管理正确**: 使用 `kcalloc()` 分配并清零内存
2. **Tag 写入正确**: 先 `memset()` 清零 SQE，再填充字段
3. **发送时机正确**: 只有用户请求时才发送

### 6.2 问题在 QM 层

1. **SQE 未清零**: `qm_create_qp_nolock()` 只清零 CQE
2. **无有效性检查**: 回调触发时未验证 SQE 是否有效

### 6.3 建议修复方案

**方案 1: 在 QM 层清零 SQE (推荐)**

```c
// qm.c: qm_create_qp_nolock()
static struct hisi_qp *qm_create_qp_nolock(...)
{
    // ...
    memset(qp->sqe, 0, qm->sqe_size * qp->sq_depth);  // 添加此行
    memset(qp->cqe, 0, sizeof(struct qm_cqe) * qp->cq_depth);
    // ...
}
```

**方案 2: 在回调中增加有效性检查**

```c
// zip_crypto.c: hisi_zip_acomp_cb()
static void hisi_zip_acomp_cb(struct hisi_qp *qp, void *data)
{
    struct hisi_zip_sqe *sqe = data;
    struct hisi_zip_req *req = (struct hisi_zip_req *)GET_REQ_FROM_SQE(sqe);

    // 添加有效性检查
    if (!req || IS_ERR_VALUE((unsigned long)req)) {
        dev_err(&qp->qm->pdev->dev, "Invalid req pointer from SQE tag!\n");
        return;
    }

    // ...
}
```

---

## 七、附录：关键代码路径

### 7.1 完整调用链

```
用户请求: crypto_acomp_compress()
    │
    ▼
hisi_zip_acompress()
    │
    ├──── hisi_zip_create_req()          // 获取 req
    │         └──── req = req_q->q + req_id
    │
    └──── hisi_zip_do_work()
              │
              ├──── hisi_zip_fill_sqe()   // 填充 SQE
              │         ├──── memset(sqe, 0, ...)  // 清零
              │         └──── fill_tag()           // 写 tag
              │
              └──── hisi_qp_send()         // 发送
                        └──── memcpy(qp->sqe, ...)  // 拷贝到 DMA

中断处理:
    │
    ▼
qm_eq_irq()
    │
    ▼
qm_work_process()
    │
    ▼
qm_poll_req_cb()
    │
    └──── qp->req_cb(qp, sqe)   // 调用回调
              │
              ▼
        hisi_zip_acomp_cb()
              │
              ├──── req = GET_REQ_FROM_SQE(sqe)  // 读取 tag
              │
              └──── qp_ctx = req->qp_ctx         // ← 崩溃点
```

### 7.2 内存布局

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              内存布局                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  内核内存 (kmalloc/kcalloc):                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  hisi_zip_ctx                                                       │   │
│  │  hisi_zip_qp_ctx[0]                                                 │   │
│  │  hisi_zip_qp_ctx[1]                                                 │   │
│  │  hisi_zip_req_q.q ──────► [req0][req1][req2]...[reqN]              │   │
│  │                            ↑                                        │   │
│  │                            地址: ffff8000xxxxxxxx (有效)            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DMA 内存 (dma_alloc_coherent):                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  QP DMA 区域                                                        │   │
│  │  ├── SQE 区域: [SQE0][SQE1]...[SQEn]                               │   │
│  │  │   ↑                                                              │   │
│  │  │   dw26/dw27 可能有残留数据 (未清零)                              │   │
│  │  │                                                                  │   │
│  │  └── CQE 区域: [CQE0][CQE1]...[CQEn]                               │   │
│  │      ↑                                                              │   │
│  │      已清零                                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```
