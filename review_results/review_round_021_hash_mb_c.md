# UADK代码检视报告 - 第21轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第21轮
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
17. drv/hisi_dae.c (第17轮) - 检视报告: review_round_017_hisi_dae_c.md
18. drv/hisi_dae_common.c (第18轮) - 检视报告: review_round_018_hisi_dae_common_c.md
19. drv/hisi_dae_join_gather.c (第19轮) - 检视报告: review_round_019_hisi_dae_join_gather_c.md
20. drv/hisi_comp_huf.c (第20轮) - 检视报告: review_round_020_hisi_comp_huf_c.md
21. drv/hash_mb/hash_mb.c (本轮) - 多哈希缓冲区实现文件，853行

### 待检视文件
- include/目录下的所有头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## drv/hash_mb/hash_mb.c 检视结果

### 1. 空指针解引用风险 - hash_mb_send函数 [HIGH]

**位置**: hash_mb.c:558-618

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static int hash_mb_send(struct wd_alg_driver *drv, handle_t ctx, void *drv_msg)
{
    struct wd_soft_ctx *s_ctx = (struct wd_soft_ctx *)ctx;
    struct hash_mb_queue *mb_queue = s_ctx->priv;  // 未检查s_ctx是否为NULL
    struct wd_digest_msg *d_msg = drv_msg;
    struct hash_mb_poll_queue *poll_queue;
    struct hash_job hash_sync_job;
    struct hash_job *hash_job;
    int ret;

    ret = hash_mb_check_param(mb_queue, d_msg);
    if (ret)
        return ret;
    // ...
}
```

**证据链**:
1. ctx直接转换为s_ctx，未检查是否为NULL
2. 立即访问s_ctx->priv获取mb_queue
3. 如果ctx为NULL会导致空指针解引用崩溃

**风险分析**:
- 这是软件实现的哈希驱动send函数
- 如果调用方传入无效ctx会导致崩溃

**修复建议**:
```c
static int hash_mb_send(struct wd_alg_driver *drv, handle_t ctx, void *drv_msg)
{
    struct wd_soft_ctx *s_ctx = (struct wd_soft_ctx *)ctx;
    struct hash_mb_queue *mb_queue;

    if (!s_ctx)
        return -WD_EINVAL;

    mb_queue = s_ctx->priv;
    if (!mb_queue)
        return -WD_EINVAL;
    // ...
}
```

---

### 2. 空指针解引用风险 - hash_mb_recv函数 [HIGH]

**位置**: hash_mb.c:779-800

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static int hash_mb_recv(struct wd_alg_driver *drv, handle_t ctx, void *drv_msg)
{
    struct wd_soft_ctx *s_ctx = (struct wd_soft_ctx *)ctx;
    struct hash_mb_queue *mb_queue = s_ctx->priv;  // 未检查s_ctx是否为NULL
    struct wd_digest_msg *msg = drv_msg;
    int ret, i = 0;

    if (mb_queue->ctx_mode == CTX_MODE_SYNC)  // 直接访问mb_queue成员
        return WD_SUCCESS;
    // ...
}
```

**分析**:
- 与hash_mb_send相同的问题模式
- 未检查ctx和s_ctx->priv是否为NULL

---

### 3. 空指针解引用风险 - 回调函数未检查 [MEDIUM]

**位置**: hash_mb.c:290, 368, 454, 467, 480, 485, 497, 501, 511, 750, 752, 756

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码示例**:
```c
// hash_middle_block_process函数
poll_queue->ops->asimd_x1(job, 1);  // 未检查ops和asimd_x1是否为NULL

// hash_mb_init_iv函数
poll_queue->ops->asimd_x1(job, 1);  // 多处类似调用

// hash_do_sync函数
poll_queue->ops->asimd_x1(job, job->len);

// hash_mb_do_jobs函数
poll_queue->ops->sve(len, j, job_vecs);      // 未检查sve是否为NULL
poll_queue->ops->asimd_x4(job_vecs[0], ...); // 未检查asimd_x4是否为NULL
poll_queue->ops->asimd_x1(job_vecs[i++], len);
```

**分析**:
- 多处调用poll_queue->ops的回调函数
- 未检查poll_queue->ops是否为NULL
- 未检查各个回调函数指针(asimd_x1、asimd_x4、sve)是否为NULL
- 如果这些函数指针为NULL会导致崩溃

---

### 4. 空指针解引用风险 - hash_mb_check_param函数 [MEDIUM]

**位置**: hash_mb.c:543-556

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
static int hash_mb_check_param(struct hash_mb_queue *mb_queue, struct wd_digest_msg *d_msg)
{
    if (unlikely(mb_queue->ctx_mode == CTX_MODE_ASYNC && d_msg->has_next)) {  // 未检查参数是否为NULL
        WD_ERR("invalid: async mode not supports long hash!\n");
        return -WD_EINVAL;
    }

    if (unlikely(d_msg->data_fmt != WD_FLAT_BUF)) {
        WD_ERR("invalid: hash multibuffer not supports sgl mode!\n");
        return -WD_EINVAL;
    }

    return WD_SUCCESS;
}
```

**分析**:
- 未检查mb_queue是否为NULL
- 未检查d_msg是否为NULL
- 直接访问这些指针的成员

---

### 5. 空指针解引用风险 - hash_mb_do_jobs函数 [MEDIUM]

**位置**: hash_mb.c:720-777

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
static int hash_mb_do_jobs(struct hash_mb_queue *mb_queue)
{
    struct hash_mb_poll_queue *poll_queue = hash_get_poll_queue(mb_queue);
    struct hash_job *job_vecs[HASH_MAX_LANES];
    __u64 len = 0;
    int maxjobs;
    int j = 0;
    int i = 0;

    if (!poll_queue)
        return -WD_EAGAIN;

    maxjobs = poll_queue->ops->max_lanes();  // 未检查ops和max_lanes是否为NULL
    maxjobs = MIN(maxjobs, poll_queue->ops->max_jobs);
    // ...

    if (j > HASH_NENO_PROCESS_JOBS) {
        poll_queue->ops->sve(len, j, job_vecs);  // 未检查sve是否为NULL
    } else if (j == HASH_NENO_PROCESS_JOBS) {
        poll_queue->ops->asimd_x4(job_vecs[0], job_vecs[1],
                      job_vecs[2], job_vecs[3], len);
    } else {
        while (i < j)
            poll_queue->ops->asimd_x1(job_vecs[i++], len);
    }
    // ...
}
```

**分析**:
- 检查了poll_queue是否为NULL，这很好
- 但未检查poll_queue->ops是否为NULL
- 未检查max_lanes、max_jobs、sve、asimd_x4、asimd_x1是否为NULL

---

### 6. 空指针解引用风险 - 辅助处理函数 [MEDIUM]

**位置**: hash_mb.c:274-312, 314-331, 333-380, 382-407, 409-436

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码示例** (hash_middle_block_process):
```c
static int hash_middle_block_process(struct hash_mb_poll_queue *poll_queue,
                     struct wd_digest_msg *d_msg,
                     struct hash_job *job)
{
    __u8 *buffer = d_msg->partial_block + d_msg->partial_bytes;  // 未检查d_msg
    __u64 length = (__u64)d_msg->partial_bytes + d_msg->in_bytes;

    if (length < HASH_BLOCK_SIZE) {
        memcpy(buffer, d_msg->in, d_msg->in_bytes);  // 未检查d_msg->in
        d_msg->partial_bytes = length;
        return -WD_EAGAIN;
    }
    // ...
}
```

**分析**:
- hash_middle_block_process、hash_signle_block_process、hash_final_block_process
- hash_first_block_process、hash_do_partial等函数
- 都未检查poll_queue、d_msg、job是否为NULL
- 未检查d_msg->in、d_msg->out等指针是否为NULL

---

### 7. 正确实现 - hash_mb_init函数 [POSITIVE]

**位置**: hash_mb.c:189-216

**正面代码**:
```c
static int hash_mb_init(struct wd_alg_driver *drv, void *conf)
{
    struct wd_ctx_config_internal *config = conf;
    struct hash_mb_ctx *priv;
    int ret;

    /* Fallback init is NULL */
    if (!drv || !conf)  // 正确检查drv和conf
        return 0;

    priv = malloc(sizeof(struct hash_mb_ctx));
    if (!priv)
        return -WD_ENOMEM;
    // ...

    ret = hash_mb_queue_init(config);
    if (ret) {
        free(priv);
        return ret;
    }

    drv->priv = priv;
    return WD_SUCCESS;
}
```

**分析**:
- 正确检查了drv和conf是否为NULL
- 内存分配失败有错误处理
- 资源管理正确

---

### 8. 正确实现 - hash_mb_exit函数 [POSITIVE]

**位置**: hash_mb.c:218-229

**正面代码**:
```c
static void hash_mb_exit(struct wd_alg_driver *drv)
{
    struct hash_mb_ctx *priv;

    if (!drv || !drv->priv)  // 正确检查drv和drv->priv
        return;

    priv = (struct hash_mb_ctx *)drv->priv;
    hash_mb_queue_uninit(&priv->config, priv->config.ctx_num);
    free(priv);
    drv->priv = NULL;
}
```

**分析**:
- 正确检查了drv和drv->priv是否为NULL
- 正确释放资源并置NULL

---

### 9. 整数运算风险 - hash_mb_pad_data函数 [LOW]

**位置**: hash_mb.c:231-260

**问题类型**: 整数运算 (cpp-integer-overflow)

**代码**:
```c
static void hash_mb_pad_data(struct hash_pad *hash_pad, __u8 *in, __u32 partial,
             __u64 total_len, bool transfer)
{
    __u64 size = total_len << BYTES_TO_BITS_OFFSET;  // 移位操作
    __u8 *buffer = hash_pad->pad;

    if (partial)
        memcpy(buffer, in, partial);

    buffer[partial++] = 0x80;
    if (partial <= HASH_PADLENGTHFIELD_SIZE) {
        // ...
    } else {
        memset(buffer + partial, 0, HASH_PADDING_SIZE - partial);
        // ...
    }
}
```

**分析**:
- total_len左移3位可能溢出，但由于是__u64类型，溢出风险较低
- partial递增后用于数组索引，但没有检查是否越界
- 建议添加边界检查

---

## drv/hash_mb/hash_mb.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 2 |
| MEDIUM | 5 |
| LOW | 1 |
| POSITIVE | 2 |

**重点问题**:
- hash_mb_send/recv函数缺少ctx NULL检查(HIGH)
- 多处回调函数调用未检查函数指针是否为NULL(MEDIUM)

**正面发现**:
- hash_mb_init/exit函数正确实现了参数检查

**建议修复顺序**:
1. 修复hash_mb_send和hash_mb_recv函数的ctx NULL检查
2. 为poll_queue->ops回调函数添加NULL检查
3. 完善hash_mb_check_param等函数的参数检查

---

## 检视进度更新

**已完成**: 21/94 文件 (22.3%)

**下次检视**: include目录下的头文件

---

## 累计问题统计更新

| 等级 | 数量 |
|------|------|
| HIGH | 36 (+2) |
| MEDIUM | 69 (+5) |
| LOW | 30 (+1) |
| **总计** | **135** |