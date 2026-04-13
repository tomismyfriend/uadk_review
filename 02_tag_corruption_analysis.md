# 第二步：tag/dw26/dw27 损坏路径分析

## 一、问题核心回顾

崩溃发生时：
```
crash address: ffff0800007f0028
x20 = ffff0800007f0000 (req pointer from tag)
访问 req->qp_ctx (offset 0x28) 时崩溃
```

req 指针从 SQE 的 dw26/dw27 (TAG 字段) 读出，值为无效地址。

## 二、关键发现：SQE 从未被清零！

### 2.1 QP 创建时只清 CQE，不清 SQE

**代码位置：qm.c:2158**
```c
static struct hisi_qp *qm_create_qp_nolock(struct hisi_qm *qm, u8 alg_type,
                                            bool is_in_kernel)
{
    // ...
    qp = &qm->qp_array[qp_id];
    hisi_qm_unset_hw_reset(qp);

    /* 只清零 CQE，SQE 未清零！这是关键问题！ */
    memset(qp->cqe, 0, sizeof(struct qm_cqe) * qp->cq_depth);

    qp->event_cb = NULL;
    qp->req_cb = NULL;
    // ...
}
```

### 2.2 QP 释放时也不清 SQE

**代码位置：qm.c:2202-2214**
```c
static void hisi_qm_release_qp(struct hisi_qp *qp)
{
    struct hisi_qm *qm = qp->qm;

    down_write(&qm->qps_lock);
    qm->qp_in_used--;
    idr_remove(&qm->qp_idr, qp->qp_id);
    up_write(&qm->qps_lock);
    qm_pm_put_sync(qm);

    /* 完全没有清零 SQE 的操作！ */
}
```

### 2.3 DMA 内存分配不保证清零

**代码位置：qm.c:3123-3128**
```c
qp->qdma.va = dma_alloc_coherent(dev, dma_size, &qp->qdma.dma,
                                  GFP_KERNEL);
if (!qp->qdma.va) {
    kfree(qp->msg);
    return -ENOMEM;
}

qp->sqe = qp->qdma.va;  /* SQE 指向 DMA 内存 */
```

虽然 dma_alloc_coherent 通常返回清零的内存，但如果这块内存之前被其他 QP 使用过并释放，
再次分配时可能包含残留数据。

## 三、tag 写入的完整路径分析

### 3.1 正常写入路径

```
用户请求 → hisi_zip_compress() → hisi_zip_do_work()
                                     ↓
                               hisi_zip_fill_sqe()
                                     ↓
                               memset(sqe, 0, ...)  // 清零本地 SQE
                                     ↓
                               hisi_zip_fill_tag()
                                     ↓
                               sqe->dw26 = lower_32_bits(req)
                               sqe->dw27 = upper_32_bits(req)
                                     ↓
                               hisi_qp_send()
                                     ↓
                               memcpy(qp->sqe + offset, &zip_sqe, size)
                                     ↓
                               硬件处理请求 → 生成 CQE → 中断处理
                                     ↓
                               qm_poll_req_cb()
                                     ↓
                               sqe = qp->sqe + sqe_size * cqe->sq_head
                                     ↓
                               req = GET_REQ_FROM_SQE(sqe)  // 从 tag 读 req
                                     ↓
                               hisi_zip_acomp_cb(qp, sqe)
```

**代码位置：zip_crypto.c:230-243**
```c
static void hisi_zip_fill_sqe(struct hisi_zip_ctx *ctx,
                               struct hisi_zip_sqe *sqe,
                               u8 req_type, struct hisi_zip_req *req)
{
    const struct hisi_zip_sqe_ops *ops = ctx->ops;

    memset(sqe, 0, sizeof(struct hisi_zip_sqe));  // 清零！

    ops->fill_addr(sqe, req);
    ops->fill_buf_size(sqe, req);
    ops->fill_buf_type(sqe, HZIP_SGL);
    ops->fill_req_type(sqe, req_type);
    ops->fill_tag(sqe, req);          // 写入 tag
    ops->fill_sqe_type(sqe, ops->sqe_type);
}
```

**代码位置：zip_crypto.c:215-219**
```c
static void hisi_zip_fill_tag(struct hisi_zip_sqe *sqe,
                               struct hisi_zip_req *req)
{
    sqe->dw26 = lower_32_bits((u64)req);
    sqe->dw27 = upper_32_bits((u64)req);
}
```

### 3.2 关键：正常流程中 SQE 必定被清零

在正常业务流程中，发送请求时：
1. 首先调用 `memset(sqe, 0, sizeof(struct hisi_zip_sqe))` 清零整个 SQE
2. 然后才写入正确的 tag 值

这意味着：**如果回调被调用时 SQE 内有残留数据，说明这个 SQE slot 从未被正常发送请求时使用过！**

## 四、可能的损坏场景分析

### 场景 A：硬件在回调设置后、首次请求前产生中断

**时机问题：**
```
时间线:
T1: QP 创建 (SQE 有残留数据，未清零)
T2: 回调设置 (qp->req_cb = hisi_zip_acomp_cb)
T3: ❌ 硬件产生中断（异常！）
T4: 中断处理调用回调，读取残留 tag → 崩溃
T5: 用户首次发送请求 (SQE 被清零并写入正确 tag)
```

**问题原因：**
- 硬件可能在 QP 创建后、用户请求前就产生完成中断
- 可能是硬件状态残留、硬件 bug、或中断风暴等原因

### 场景 B：DMA 内存被复用后残留数据

**内存复用链：**
```
第一次驱动加载:
1. dma_alloc_coherent → 分配内存块 A
2. QP 使用内存块 A，写入 SQE 数据
3. rmmod → dma_free_coherent → 内存块 A 释放但数据仍存在

第二次驱动加载:
4. dma_alloc_coherent → 可能再次分配内存块 A
5. 内存块 A 内仍有上次的 SQE 数据（包括 tag）
6. QP 创建 → SQE 未清零 → 残留 tag 值
7. 如果硬件此时产生中断 → 崩溃
```

### 场景 C：CQE sq_head 指向错误的 SQE 位置

**代码位置：qm.c:1057-1060**
```c
static void qm_poll_req_cb(struct hisi_qp *qp)
{
    struct qm_cqe *cqe = qp->cqe + qp->qp_status.cq_head;
    struct hisi_qm *qm = qp->qm;

    while (QM_CQE_PHASE(cqe) == qp->qp_status.cqc_phase) {
        dma_rmb();
        /* 关键：根据 CQE 的 sq_head 定位 SQE！ */
        qp->req_cb(qp, qp->sqe + qm->sqe_size *
                   le16_to_cpu(cqe->sq_head));
        // ...
    }
}
```

**问题：**
如果 CQE 中 `sq_head` 字段值为：
- 指向一个从未发送过请求的 SQE slot（残留数据）
- 或者硬件 bug 导致 sq_head 值错误

回调就会读取错误的 SQE，tag 为残留值 → 崩溃

### 场景 D：硬件状态残留（寄存器/队列状态）

**可能的硬件问题：**
1. 硬件 SQ/CQ 寄存器在上次 rmmod 后未正确复位
2. 硬件内部队列仍有上次的请求状态
3. 硬件中断状态残留

这可能导致硬件在驱动加载后立即产生"虚假"完成中断。

## 五、qp_stop_fail_cb 路径（次要）

**代码位置：qm.c:2334-2350**
```c
static void qp_stop_fail_cb(struct hisi_qp *qp)
{
    int qp_used = atomic_read(&qp->qp_status.used);
    u16 cur_tail = qp->qp_status.sq_tail;
    u16 sq_depth = qp->sq_depth;
    u16 cur_head = (cur_tail + sq_depth - qp_used) % sq_depth;
    struct hisi_qm *qm = qp->qm;
    u16 pos;
    int i;

    for (i = 0; i < qp_used; i++) {
        pos = (i + cur_head) % sq_depth;
        qp->req_cb(qp, qp->sqe + (u32)(qm->sqe_size * pos));
        // ...
    }
}
```

这个函数在 QP 停止时调用，处理未完成的请求。
但如果 qp_status.used 的值是错误的（硬件状态残留），可能遍历到未使用的 SQE slot。

## 六、关键结论

### 确认的问题点

| 问题点 | 代码位置 | 影响 |
|--------|----------|------|
| SQE 创建时未清零 | qm.c:2158 | QP 创建后 SQE 可能有残留数据 |
| SQE 释放时未清零 | qm.c:2202 | 内存释放后数据仍残留 |
| DMA 内存复用 | qm.c:3123 | 再次分配可能获得残留数据的内存 |
| 回调依据 CQE sq_head 定位 SQE | qm.c:1059 | sq_head 错误指向残留 SQE |
| tag 读取无验证 | zip_crypto.c:43 | 直接将 tag 值当作指针使用 |

### 最可能的崩溃原因

**核心问题：** QP 创建后，SQE 内可能有残留的 tag 数据（来自之前的使用），如果硬件在用户发送首个请求前就产生完成中断（由于硬件状态残留或其他原因），回调读取残留 tag → 无效指针 → 崩溃。

### 证据链

1. **calltrace 显示 req = ffff0800007f0000** - 这不是正常的 req 指针值
2. **崩溃发生在 hisi_zip_acomp_cb+0x38** - 即 req->qp_ctx 访问处
3. **rmmod 后 insmod 才崩溃** - 说明残留数据来自上次驱动使用
4. **正常流程 SQE 会被清零** - 所以残留数据一定是在"请求发送前"被读取

## 七、下一步：复现方案

需要验证的假设：
1. SQE 内存是否包含残留数据？
2. 硬件是否在回调设置后立即产生中断？
3. CQE sq_head 是否指向未使用的 SQE slot？

复现思路：
1. 在 QP 创建时打印 SQE 内容（验证残留数据）
2. 在回调被调用时打印 sq_head 值（验证定位逻辑）
3. 模拟硬件产生中断的时机（强制调用回调）