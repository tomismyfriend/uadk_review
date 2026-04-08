# UADK代码检视报告 - 第17轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第17轮
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
12. drv/hisi_comp.c (第12轮) - 检视报告: review_round_012_hisi_comp_c.md
13. drv/hisi_qm_udrv.c (第13轮) - 检视报告: review_round_013_hisi_qm_udrv_c.md
14. drv/isa_ce_sm3.c (第14轮) - 检视报告: review_round_014_isa_ce_sm3_c.md
15. drv/isa_ce_sm4.c (第15轮) - 检视报告: review_round_015_isa_ce_sm4_c.md
16. drv/hisi_udma.c (第16轮) - 检视报告: review_round_016_hisi_udma_c.md
17. drv/hisi_dae.c (本轮) - DAE(数据分析引擎)驱动文件，1336行

### 待检视文件
- drv/hisi_dae_common.c, drv/hisi_dae_join_gather.c
- drv/hisi_comp_huf.c, drv/hash_mb/hash_mb.c
- include/目录下的所有头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## drv/hisi_dae.c 检视结果

### 1. 空指针解引用风险 - hashagg_send函数 [HIGH]

**位置**: hisi_dae.c:429-473

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static int hashagg_send(struct wd_alg_driver *drv, handle_t ctx, void *hashagg_msg)
{
    handle_t h_qp = (handle_t)wd_ctx_get_priv(ctx);
    struct hisi_qp *qp = (struct hisi_qp *)h_qp;
    struct dae_extend_addr *ext_addr = qp->priv;  // 未检查qp是否为NULL
    struct wd_agg_msg *msg = hashagg_msg;
    struct dae_addr_list *addr_list;
    struct dae_ext_sqe *ext_sqe;
    struct dae_sqe sqe = {0};
    __u16 send_cnt = 0;
    int ret, idx;

    ret = check_hashagg_param(msg);
    if (ret)
        return ret;

    if (qp->q_info.hw_type >= HISI_QM_API_VER5_BASE &&  // 直接访问qp成员
        qp->q_info.qp_mode == CTX_MODE_SYNC && msg->pos == WD_AGG_REHASH_INPUT)
        return WD_SUCCESS;

    fill_hashagg_task_type(msg, &sqe, qp->q_info.hw_type);  // 同样风险
    // ...
    hisi_set_msg_id(h_qp, &msg->tag);  // 如果h_qp为NULL，崩溃
    // ...
}
```

**证据链**:
1. wd_ctx_get_priv(ctx)可能返回NULL
2. 直接将返回值转换为qp，未检查是否为NULL
3. 立即访问qp->priv获取ext_addr
4. 立即访问qp->q_info.hw_type和qp->q_info.qp_mode

**风险分析**:
- 与其他驱动文件(hisi_sec/hisi_hpre/hisi_comp/hisi_udma)问题模式相同
- send函数是关键路径，缺少防御性编程

**修复建议**:
```c
static int hashagg_send(struct wd_alg_driver *drv, handle_t ctx, void *hashagg_msg)
{
    struct hisi_qp *qp = wd_ctx_get_priv(ctx);
    struct dae_extend_addr *ext_addr;
    handle_t h_qp;
    // ...

    if (!qp) {
        WD_ERR("invalid: qp is NULL!\n");
        return -WD_EINVAL;
    }
    h_qp = (handle_t)qp;
    ext_addr = qp->priv;
    // ...
}
```

---

### 2. 空指针解引用风险 - hashagg_recv函数 [HIGH]

**位置**: hisi_dae.c:566-622

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static int hashagg_recv(struct wd_alg_driver *drv, handle_t ctx, void *hashagg_msg)
{
    handle_t h_qp = (handle_t)wd_ctx_get_priv(ctx);
    struct hisi_qp *qp = (struct hisi_qp *)h_qp;
    struct dae_extend_addr *ext_addr = qp->priv;  // 未检查qp是否为NULL
    struct wd_agg_msg *msg = hashagg_msg;
    struct wd_agg_msg *temp_msg = msg;
    struct hashagg_ctx *agg_ctx;
    struct dae_sqe sqe = {0};
    __u16 recv_cnt = 0;
    int ret;

    if (qp->q_info.hw_type >= HISI_QM_API_VER5_BASE &&  // 直接访问qp成员
        qp->q_info.qp_mode == CTX_MODE_SYNC && msg->pos == WD_AGG_REHASH_INPUT) {
        // ...
    }

    ret = hisi_qm_recv(h_qp, &sqe, 1, &recv_cnt);  // 如果h_qp为NULL，崩溃
    // ...
    ret = hisi_check_bd_id(h_qp, msg->tag, sqe.low_tag);  // 同样风险
    // ...
    if (qp->q_info.qp_mode == CTX_MODE_ASYNC) {  // 直接访问
        temp_msg = wd_agg_get_msg(qp->q_info.idx, msg->tag);
        // ...
    }
}
```

**分析**:
- 与hashagg_send相同的问题模式
- 多处直接访问qp成员，缺少NULL检查

---

### 3. 空指针解引用风险 - fill_hashagg_output_order函数 [MEDIUM]

**位置**: hisi_dae.c:132-155

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
static void fill_hashagg_output_order(struct dae_sqe *sqe, struct dae_ext_sqe *ext_sqe,
                      struct wd_agg_msg *msg)
{
    struct hashagg_ctx *agg_ctx = msg->priv;  // 未检查msg->priv是否为NULL
    struct hashagg_col_data *cols_data = &agg_ctx->cols_data;
    struct hashagg_output_src *output_src = cols_data->normal_output;
    __u32 out_cols_num = cols_data->output_num;
    __u32 offset = 0;
    __u32 i;

    if (msg->pos == WD_AGG_REHASH_INPUT)
        output_src = cols_data->rehash_output;
    // ...
}
```

**分析**:
- 未检查msg->priv是否为NULL
- 直接访问agg_ctx的成员
- 如果msg->priv为NULL会导致崩溃

---

### 4. 空指针解引用风险 - fill_hashagg_table_data函数 [MEDIUM]

**位置**: hisi_dae.c:180-213

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
static void fill_hashagg_table_data(struct dae_sqe *sqe, struct dae_addr_list *addr_list,
                    struct wd_agg_msg *msg)
{
    struct hashagg_ctx *agg_ctx = (struct hashagg_ctx *)msg->priv;  // 未检查
    struct hash_table_data *table_data = &agg_ctx->table_data;
    struct dae_table_addr *hw_table = &addr_list->src_table;
    // ...
}
```

**分析**:
- 与fill_hashagg_output_order相同的问题

---

### 5. 空指针解引用风险 - fill_hashagg_key_data函数 [MEDIUM]

**位置**: hisi_dae.c:240-284

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
static void fill_hashagg_key_data(struct dae_sqe *sqe, struct dae_ext_sqe *ext_sqe,
                  struct dae_addr_list *addr_list, struct wd_agg_msg *msg)
{
    struct hashagg_ctx *agg_ctx = msg->priv;  // 未检查msg->priv
    struct hw_agg_data *key_data = agg_ctx->cols_data.key_data;
    struct wd_dae_col_addr *usr_key_addr;
    struct dae_col_addr *hw_key_addr;
    // ...

    if (msg->pos == WD_AGG_STREAM_INPUT || msg->pos == WD_AGG_REHASH_INPUT) {
        usr_key_addr = msg->req.key_cols;  // 未检查msg->req.key_cols是否为NULL
        hw_key_addr = addr_list->input_addr;
    } else {
        usr_key_addr = msg->req.out_key_cols;  // 未检查是否为NULL
        hw_key_addr = addr_list->output_addr;
    }

    for (i = 0; i < msg->key_cols_num; i++) {
        // ...
        hw_key_addr[i].empty_addr = (__u64)(uintptr_t)usr_key_addr[usr_col_idx].empty;
        // 如果usr_key_addr为NULL，崩溃
        // ...
    }
}
```

**分析**:
- 未检查msg->priv是否为NULL
- 未检查msg->req.key_cols/out_key_cols是否为NULL
- 如果这些指针为NULL会导致崩溃

---

### 6. 空指针解引用风险 - fill_hashagg_input_data函数 [MEDIUM]

**位置**: hisi_dae.c:315-369

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
static void fill_hashagg_input_data(struct dae_sqe *sqe, struct dae_ext_sqe *ext_sqe,
                    struct dae_addr_list *addr_list, struct wd_agg_msg *msg)
{
    struct hashagg_ctx *agg_ctx = msg->priv;  // 未检查msg->priv
    struct hashagg_col_data *cols_data = &agg_ctx->cols_data;
    struct wd_dae_col_addr *usr_agg_addr;
    // ...
    switch (msg->pos) {
    case WD_AGG_STREAM_INPUT:
        // ...
        usr_agg_addr = msg->req.agg_cols;  // 未检查是否为NULL
        // ...
    case WD_AGG_REHASH_INPUT:
        // ...
        usr_agg_addr = msg->req.agg_cols;  // 未检查是否为NULL
        // ...
    case WD_AGG_STREAM_OUTPUT:
    case WD_AGG_REHASH_OUTPUT:
        // ...
        usr_agg_addr = msg->req.out_agg_cols;  // 未检查是否为NULL
        // ...
    }

    for (i = 0; i < agg_col_num; i++) {
        usr_col_idx = agg_data[i].usr_col_idx;
        hw_agg_addr[i].empty_addr = (__u64)(uintptr_t)usr_agg_addr[usr_col_idx].empty;
        // 如果usr_agg_addr为NULL，崩溃
        // ...
    }
}
```

**分析**:
- 未检查msg->priv是否为NULL
- 未检查msg->req.agg_cols/out_agg_cols是否为NULL

---

### 7. 空指针解引用风险 - fill_sum_overflow_cols函数 [MEDIUM]

**位置**: hisi_dae.c:475-504

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
static void fill_sum_overflow_cols(struct dae_sqe *sqe, struct wd_agg_msg *msg,
                   struct hashagg_ctx *agg_ctx)
{
    __u32 i, output_num, usr_col_idx;
    struct hw_agg_data *agg_data;

    pthread_spin_lock(&agg_ctx->lock);  // 未检查agg_ctx是否为NULL
    agg_ctx->sum_overflow_cols |= sqe->sum_overflow_cols;
    if (!agg_ctx->sum_overflow_cols) {
        pthread_spin_unlock(&agg_ctx->lock);
        return;
    }
    pthread_spin_unlock(&agg_ctx->lock);
    // ...
    if (!msg->req.sum_overflow_cols)
        return;

    agg_data = agg_ctx->cols_data.output_data;
    output_num = agg_ctx->cols_data.output_num;
    for (i = 0; i < output_num; i++) {
        usr_col_idx = agg_data[i].usr_col_idx;
        if (agg_ctx->sum_overflow_cols & BIT(i))
            msg->req.sum_overflow_cols[usr_col_idx] = 1;
        else
            msg->req.sum_overflow_cols[usr_col_idx] = 0;
    }
}
```

**分析**:
- 未检查agg_ctx是否为NULL
- 直接调用pthread_spin_lock(&agg_ctx->lock)
- 如果agg_ctx为NULL会导致崩溃

---

### 8. 空指针解引用风险 - fill_hashagg_merge系列函数 [MEDIUM]

**位置**: hisi_dae.c:157-178, 215-238, 286-299, 371-378

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码示例** (fill_hashagg_merge_key_data):
```c
static void fill_hashagg_merge_key_data(struct dae_sqe *sqe, struct dae_ext_sqe *ext_sqe,
                    struct dae_addr_list *addr_list, struct wd_agg_msg *msg)
{
    struct hashagg_ctx *agg_ctx = msg->priv;  // 未检查msg->priv
    struct hw_agg_data *key_data = agg_ctx->cols_data.key_data;
    // ...
}
```

**分析**:
- fill_hashagg_merge_output_order、fill_hashagg_merge_table_data
- fill_hashagg_merge_key_data、fill_hashagg_merge_input_data
- 所有这些函数都有相同的问题，未检查msg->priv是否为NULL

---

### 9. 正确实现示例 - sess相关函数 [POSITIVE]

**位置**: hisi_dae.c:1204-1215, 1217-1256, 1258-1266, 1268-1278, 1280-1293

**正面代码**:
```c
// hashagg_sess_priv_uninit
static void hashagg_sess_priv_uninit(struct wd_alg_driver *drv, void *priv)
{
    struct hashagg_ctx *agg_ctx = priv;

    if (!agg_ctx) {
        WD_ERR("invalid: dae sess uninit priv is NULL!\n");
        return;
    }
    // ...
}

// hashagg_sess_priv_init
static int hashagg_sess_priv_init(struct wd_alg_driver *drv,
                  struct wd_agg_sess_setup *setup, void **priv)
{
    // ...
    if (!drv || !drv->priv) {
        WD_ERR("invalid: dae drv is NULL!\n");
        return -WD_EINVAL;
    }

    if (!setup || !priv) {
        WD_ERR("invalid: dae sess priv is NULL!\n");
        return -WD_EINVAL;
    }
    // ...
}
```

**分析**:
- 这些函数正确检查了参数是否为NULL
- 应作为其他函数的参考实现

---

### 10. fill_hashagg_msg_task_err函数边界检查 [LOW]

**位置**: hisi_dae.c:521-564

**问题类型**: 边界检查 (cpp-memory-safety)

**代码**:
```c
static void fill_hashagg_msg_task_err(struct dae_sqe *sqe, struct wd_agg_msg *msg,
                      struct wd_agg_msg *temp_msg, struct hashagg_ctx *agg_ctx)
{
    switch (sqe->err_type) {
    case DAE_TASK_BD_ERROR_MIN ... DAE_TASK_BD_ERROR_MAX:
        // ...
    case DAE_HASH_TABLE_NEED_REHASH:
        // ...
    case DAE_HASH_TABLE_INVALID:
        // ...
    // ...
    default:
        WD_ERR("failed to do hashagg task! done_flag=0x%x, etype=0x%x, ext_type = 0x%x!\n",
            (__u32)sqe->done_flag, (__u32)sqe->err_type, (__u32)sqe->ext_err_type);
        msg->result = WD_AGG_PARSE_ERROR;
        break;
    }
    // ...
}
```

**分析**:
- 使用了switch-case覆盖所有错误类型
- default分支处理未知错误类型
- 实现较为完善

---

## drv/hisi_dae.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 2 |
| MEDIUM | 6 |
| LOW | 1 |
| POSITIVE | 1 |

**重点问题**:
- hashagg_send/recv函数空指针检查缺失(HIGH)
- fill_hashagg系列辅助函数缺少msg->priv检查(MEDIUM)

**正面发现**:
- sess_priv_init/uninit等函数正确实现了参数检查，可作为参考

**建议修复顺序**:
1. 修复hashagg_send和hashagg_recv函数的qp NULL检查
2. 为所有fill_hashagg系列函数添加msg->priv检查
3. 添加msg->req.key_cols/agg_cols等指针的NULL检查

---

## 检视进度更新

**已完成**: 17/94 文件 (18.1%)

**下次检视**: drv/hisi_dae_common.c (DAE通用函数)

---

## 累计问题统计更新

| 等级 | 数量 |
|------|------|
| HIGH | 31 (+2) |
| MEDIUM | 53 (+6) |
| LOW | 24 (+1) |
| **总计** | **108** |