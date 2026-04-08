# UADK代码检视报告 - 第14轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第14轮
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
14. drv/isa_ce_sm3.c (本轮) - SM3指令集实现文件，409行

### 待检视文件
- drv/isa_ce_sm4.c, drv/hisi_udma.c
- drv/hisi_dae.c等drv目录其他文件
- include/目录下的所有头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## drv/isa_ce_sm3.c 检视结果

### 1. 空指针解引用风险 - do_sm3_ce函数 [MEDIUM]

**位置**: isa_ce_sm3.c:173-224

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static int do_sm3_ce(struct wd_digest_msg *msg, __u8 *out_digest)
{
    enum hash_block_type block_type;
    struct sm3_ce_ctx sctx = {0};
    size_t data_len, iv_len;
    __u8 *data, *iv;

    block_type = get_hash_block_type(msg);
    data_len = msg->in_bytes;
    data = msg->in;   // 未检查msg->in是否为NULL
    iv_len = SM3_DIGEST_SIZE;
    iv = msg->out;    // 未检查msg->out是否为NULL

    switch(block_type) {
    case HASH_MIDDLE_BLOCK:
        sm3_ce_init_ex(&sctx, iv, iv_len);  // 如果iv为NULL，可能崩溃
        // ...
    }
    // ...
    memcpy(msg->out, out_digest, ...);  // 如果msg->out为NULL，崩溃
}
```

**证据链**:
1. 未检查msg->in是否为NULL
2. 未检查msg->out是否为NULL
3. 调用sm3_ce_init_ex时传入iv(即msg->out)
4. 最后memcpy到msg->out，如果为NULL会崩溃

**风险分析**:
- 如果调用方未正确初始化msg->in或msg->out，会导致空指针崩溃
- 需要添加参数检查

---

### 2. 空指针解引用风险 - do_hmac_sm3_ce函数 [MEDIUM]

**位置**: isa_ce_sm3.c:284-336

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static int do_hmac_sm3_ce(struct wd_digest_msg *msg, __u8 *out_hmac)
{
    size_t data_len, key_len, iv_len;
    enum hash_block_type block_type;
    struct hmac_sm3_ctx hctx = {0};
    __u8 *data, *key, *iv;

    data_len = msg->in_bytes;
    data = msg->in;     // 未检查是否为NULL
    key = msg->key;     // 未检查是否为NULL
    key_len = msg->key_bytes;
    iv_len = SM3_DIGEST_SIZE;
    iv = msg->out;      // 未检查是否为NULL

    switch(block_type) {
    case HASH_SINGLE_BLOCK:
        sm3_ce_hmac_init(&hctx, key, key_len);  // 如果key为NULL，可能崩溃
        // ...
    }
    // ...
    memcpy(msg->out, out_hmac, ...);  // 如果msg->out为NULL，崩溃
}
```

**分析**:
- 与do_sm3_ce相同的问题
- 额外未检查msg->key是否为NULL

---

### 3. sm3_ce_init_ex函数参数检查 [MEDIUM]

**位置**: isa_ce_sm3.c:89-100

**问题类型**: 参数检查 (cpp-input-validation)

**代码**:
```c
static void sm3_ce_init_ex(struct sm3_ce_ctx *sctx, __u8 *iv, __u16 iv_bytes)
{
    size_t i;

    if (iv_bytes != SM3_DIGEST_SIZE) {
        WD_ERR("invalid iv size: %u\n", iv_bytes);
        return;
    }

    for (i = 0; i < SM3_STATE_WORDS; i++)
        PUTU8_TO_U32(sctx->word_reg[i], iv + i * WORD_TO_CHAR_OFFSET);  // 未检查iv是否为NULL
}
```

**分析**:
- 检查了iv_bytes大小，但未检查iv是否为NULL
- 如果iv为NULL，PUTU8_TO_U32宏会访问NULL指针

---

### 4. sm3_hmac_key_padding函数参数检查 [LOW]

**位置**: isa_ce_sm3.c:226-246

**问题类型**: 参数检查 (cpp-input-validation)

**代码**:
```c
static void sm3_hmac_key_padding(struct hmac_sm3_ctx *hctx,
                     const __u8 *key, size_t key_len)
{
    size_t i;

    if (key_len <= SM3_BLOCK_SIZE) {
        memcpy(hctx->key, key, key_len);  // 未检查hctx和key是否为NULL
        // ...
    } else {
        sm3_ce_init(&hctx->sctx);
        sm3_ce_update(&hctx->sctx, key, key_len, sm3_ce_block_compress);  // 同样风险
        // ...
    }
    // ...
}
```

**分析**:
- 未检查hctx是否为NULL
- 未检查key是否为NULL
- 建议添加防御性检查

---

### 5. sm3_ce_drv_recv函数无操作 [LOW]

**位置**: isa_ce_sm3.c:373-376

**问题类型**: 代码逻辑 (cpp-coding-standards)

**代码**:
```c
static int sm3_ce_drv_recv(struct wd_alg_driver *drv, handle_t ctx, void *digest_msg)
{
    return WD_SUCCESS;
}
```

**分析**:
- recv函数直接返回成功，不做任何处理
- 这是因为SM3 CE驱动是同步执行的
- 但如果调用方期望recv做某些处理，可能导致逻辑错误
- 建议添加注释说明

---

### 6. sm3_ce_update函数边界检查 [LOW]

**位置**: isa_ce_sm3.c:102-138

**问题类型**: 边界检查 (cpp-memory-safety)

**代码**:
```c
static void sm3_ce_update(struct sm3_ce_ctx *sctx, const __u8 *data,
              size_t data_len, sm3_ce_block_fn *block_fn)
{
    size_t remain_data_len, blk_num;

    /* Get the data num that need compute currently */
    sctx->num &= (SM3_BLOCK_SIZE - 1);

    if (sctx->num) {
        remain_data_len = SM3_BLOCK_SIZE - sctx->num;
        /* If data_len does not enough a block size, then leave it to final */
        if (data_len < remain_data_len) {
            memcpy(sctx->block + sctx->num, data, data_len);  // 未检查data是否为NULL
            // ...
        }
        // ...
    }
    // ...
}
```

**分析**:
- 未检查data是否为NULL
- 未检查block_fn是否为NULL

---

## drv/isa_ce_sm3.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 0 |
| MEDIUM | 3 |
| LOW | 3 |

**重点问题**:
- do_sm3_ce/do_hmac_sm3_ce函数参数检查不完整(MEDIUM)
- 多个内部函数缺少NULL检查

**建议修复顺序**:
1. 在do_sm3_ce和do_hmac_sm3_ce中添加msg->in/msg->out/msg->key的NULL检查
2. 完善sm3_ce_init_ex函数的iv参数检查
3. 添加其他内部函数的防御性检查

---

## 检视进度更新

**已完成**: 14/94 文件 (14.9%)

**下次检视**: drv/isa_ce_sm4.c (SM4指令集实现)