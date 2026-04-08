# UADK代码检视报告 - 第12轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第12轮
- **规则来源**: E:/work_space/加速器用户态学习/代码检视/review_rules
- **使用Skills**: cpp-memory-safety, cpp-resource-management, cpp-input-validation

---

## 本轮检视文件

### 已检视文件
1. wd.c (第1轮) - 检视报告: review_round_001_wd_c.md
2. wd_util.c (第2轮) - 检视报告: review_round_002_wd_util_c.md
3. wd_alg.c (第3轮) - 检视报告: review_round_003_wd_alg_c.md
4. wd_sched.c (第4轮) - 检视报告: review_round_004_wd_sched_c.md
5. wd_mempool.c (第5轮) - 检视报告: review_round_005_wd_mempool_c.md
6. wd_cipher.c (第6轮) - 检视报告: review_round_006_wd_cipher_c.md
7. wd_digest.c (第7轮) - 检视报告: review_round_007_wd_digest_c.md
8. wd_aead.c (第8轮) - 检视报告: review_round_008_wd_aead_c.md
9. wd_comp.c (第9轮) - 检视报告: review_round_009_wd_comp_c.md
10. drv/hisi_sec.c (第10轮) - 检视报告: review_round_010_hisi_sec_c.md
11. drv/hisi_hpre.c (第11轮) - 检视报告: review_round_011_hisi_hpre_c.md
12. drv/hisi_comp.c (本轮) - 压缩驱动文件，1857行

### 待检视文件
- drv/hisi_qm_udrv.c, drv/hisi_udma.c
- drv/isa_ce_sm3.c, drv/isa_ce_sm4.c
- drv/hisi_dae.c等drv目录其他文件
- include/目录下的所有头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## drv/hisi_comp.c 检视结果

### 1. 空指针解引用风险 - hisi_zip_comp_send函数 [HIGH]

**位置**: hisi_comp.c:1537-1569

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static int hisi_zip_comp_send(struct wd_alg_driver *drv, handle_t ctx, void *comp_msg)
{
    struct hisi_qp *qp = wd_ctx_get_priv(ctx);  // 未检查qp是否为NULL
    struct wd_comp_msg *msg = comp_msg;
    handle_t h_qp = (handle_t)qp;
    struct hisi_zip_sqe sqe = {0};
    __u16 count = 0;
    int ret;

    /* Skip hardware, if the store buffer need to be copied to output */
    ret = check_store_buf(msg);
    if (ret)
        return 0;

    hisi_set_msg_id(h_qp, &msg->tag);  // 如果qp为NULL，会崩溃
    ret = fill_zip_comp_sqe(qp, msg, &sqe);  // 同样风险
    // ...
}
```

**证据链**:
1. wd_ctx_get_priv(ctx)可能返回NULL
2. 直接将qp转换为h_qp使用
3. 调用hisi_set_msg_id、fill_zip_comp_sqe等函数前未检查qp是否为NULL
4. 与hisi_sec.c和hisi_hpre.c中的问题相同

**风险分析**:
- 如果ctx无效或未正确初始化，会导致空指针解引用崩溃
- 驱动send函数是关键路径，需要防御性编程

**修复建议**:
```c
static int hisi_zip_comp_send(struct wd_alg_driver *drv, handle_t ctx, void *comp_msg)
{
    struct hisi_qp *qp = wd_ctx_get_priv(ctx);
    struct wd_comp_msg *msg = comp_msg;
    handle_t h_qp;
    // ...

    if (!qp) {
        WD_ERR("invalid: qp is NULL!\n");
        return -WD_EINVAL;
    }
    h_qp = (handle_t)qp;
    // ...
}
```

---

### 2. 空指针解引用风险 - hisi_zip_comp_recv函数 [HIGH]

**位置**: hisi_comp.c:1725-1760

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static int hisi_zip_comp_recv(struct wd_alg_driver *drv, handle_t ctx, void *comp_msg)
{
    struct hisi_qp *qp = wd_ctx_get_priv(ctx);  // 未检查qp是否为NULL
    struct wd_comp_msg *recv_msg = comp_msg;
    struct hisi_comp_buf *buf = NULL;
    handle_t h_qp = (handle_t)qp;
    struct hisi_zip_sqe sqe = {0};
    __u16 count = 0;
    int ret;

    // ...
    ret = hisi_qm_recv(h_qp, &sqe, 1, &count);  // 如果h_qp为NULL，会崩溃
    // ...
    ret = parse_zip_sqe(qp, &sqe, recv_msg);  // 同样风险
    // ...
}
```

**分析**:
- 与hisi_zip_comp_send相同的问题
- send和recv函数都存在空指针检查缺失

---

### 3. zip_mem_map函数参数检查 [MEDIUM]

**位置**: hisi_comp.c:434-502

**问题类型**: 参数检查 (cpp-input-validation)

**代码**:
```c
static int zip_mem_map(struct wd_mm_ops *mm_ops, struct hisi_zip_sqe *sqe,
                       struct hisi_comp_sqe_addr *addr)
{
    void *phy_dst, *phy_ctx, *mempool;
    void *phy_lit = NULL;
    void *phy_src = NULL;

    mempool = mm_ops->usr;  // 未检查mm_ops是否为NULL
    /* When the src len is 0, map is not required */
    if (sqe->input_data_length) {
        phy_src = mm_ops->iova_map(mempool, addr->src_addr, sqe->input_data_length);
        // 未检查mm_ops->iova_map是否为NULL
        // ...
    }
    // ...
}
```

**分析**:
- 未检查mm_ops是否为NULL
- 未检查mm_ops->iova_map是否为NULL
- 如果mm_ops为NULL会导致崩溃

**修复建议**:
```c
static int zip_mem_map(struct wd_mm_ops *mm_ops, struct hisi_zip_sqe *sqe,
                       struct hisi_comp_sqe_addr *addr)
{
    void *phy_dst, *phy_ctx, *mempool;
    // ...

    if (!mm_ops || !mm_ops->iova_map) {
        WD_ERR("invalid: mm_ops or iova_map is NULL!\n");
        return -WD_EINVAL;
    }
    // ...
}
```

---

### 4. zip_mem_unmap函数参数检查 [MEDIUM]

**位置**: hisi_comp.c:504-543

**问题类型**: 参数检查 (cpp-input-validation)

**代码**:
```c
static void zip_mem_unmap(struct wd_comp_msg *msg, struct hisi_zip_sqe *sqe)
{
    void *dma_addr, *src_addr, *seq_addr, *lit_addr;
    struct wd_mm_ops *mm_ops = msg->mm_ops;  // 未检查msg是否为NULL
    struct wd_comp_req *req = &msg->req;
    void *mempool = mm_ops->usr;  // 未检查mm_ops是否为NULL

    // ...
    mm_ops->iova_unmap(mempool, src_addr, dma_addr, sqe->input_data_length);  // 未检查iova_unmap
    // ...
}
```

**分析**:
- 未检查msg是否为NULL
- 未检查mm_ops是否为NULL
- 未检查mm_ops->iova_unmap是否为NULL

---

### 5. fill_buf_deflate_generic函数整数溢出风险 [MEDIUM]

**位置**: hisi_comp.c:570-634

**问题类型**: 整数溢出 (cpp-integer-overflow)

**代码**:
```c
static int fill_buf_deflate_generic(struct hisi_zip_sqe *sqe,
                    struct wd_comp_msg *msg,
                    const char *head, int head_size)
{
    // ...
    if (msg->stream_pos == WD_COMP_STREAM_NEW && head != NULL) {
        if (msg->req.op_type == WD_DIR_COMPRESS) {
            memcpy(dst, head, head_size);
            dst += head_size;
            out_size -= head_size;  // 可能下溢
        } else {
            src += head_size;
            in_size -= head_size;  // 可能下溢
        }
    }
    // ...
}
```

**分析**:
- out_size和in_size减去head_size时可能发生下溢
- 如果head_size > out_size/in_size，结果会是负数或很大的正数
- 建议添加边界检查

---

### 6. get_sgl_from_pool函数空指针检查 [MEDIUM]

**位置**: hisi_comp.c:372-408

**问题类型**: 空指针检查 (cpp-input-validation)

**代码**:
```c
static int get_sgl_from_pool(handle_t h_qp, struct comp_sgl *c_sgl, struct wd_mm_ops *mm_ops)
{
    handle_t h_sgl_pool;
    int ret;

    h_sgl_pool = hisi_qm_get_sglpool(h_qp, mm_ops);
    if (unlikely(!h_sgl_pool)) {
        WD_ERR("failed to get sglpool!\n");
        return -WD_EINVAL;
    }

    c_sgl->in = hisi_qm_get_hw_sgl(h_sgl_pool, c_sgl->list_src);
    // 未检查c_sgl是否为NULL
    // 未检查c_sgl->list_src是否为NULL
    // ...
}
```

**分析**:
- 检查了h_sgl_pool是否为NULL
- 但未检查c_sgl和mm_ops是否为NULL
- 建议添加完整的参数检查

---

### 7. copy_to_out函数边界检查 [LOW]

**位置**: hisi_comp.c:268-295

**问题类型**: 边界检查 (cpp-memory-safety)

**代码**:
```c
static __u32 copy_to_out(struct wd_comp_msg *msg, struct hisi_comp_buf *buf, __u32 total_len)
{
    struct wd_comp_req *req = &msg->req;
    struct wd_datalist *node = req->list_dst;
    // ...

    copy_len = total_len > req->dst_len ?
           req->dst_len : total_len;
    // ...

    if (req->data_fmt == WD_FLAT_BUF) {
        memcpy(req->dst, buf->dst + buf->output_offset, copy_len);
        // 未检查req->dst是否为NULL
        return copy_len;
    }
    // ...
}
```

**分析**:
- 未检查req->dst和req->list_dst是否为NULL
- 未检查buf->dst是否为NULL

---

## drv/hisi_comp.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 2 |
| MEDIUM | 4 |
| LOW | 1 |

**重点问题**:
- hisi_zip_comp_send/recv函数空指针检查缺失(HIGH)
- zip_mem_map/unmap函数参数检查不完整(MEDIUM)

**建议修复顺序**:
1. 修复send/recv函数的qp NULL检查
2. 完善zip_mem_map/unmap函数的参数检查
3. 添加整数溢出边界检查

---

## 检视进度更新

**已完成**: 12/94 文件 (12.8%)

**下次检视**: drv/hisi_qm_udrv.c (QM用户态驱动文件)