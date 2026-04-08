# UADK代码检视报告 - 第10轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第10轮
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
10. drv/hisi_sec.c (本轮) - 安全引擎驱动文件，约3900行

### 待检视文件
- drv/hisi_hpre.c, drv/hisi_comp.c, drv/hisi_qm_udrv.c
- drv/isa_ce_sm3.c, drv/isa_ce_sm4.c, drv/hisi_udma.c
- drv/hisi_dae.c等drv目录其他文件
- include/目录下的所有头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## drv/hisi_sec.c 检视结果

### 1. 数组越界风险 - g_digest_a_alg数组 [HIGH]

**位置**: hisi_sec.c:510-513, 1853, 2290

**问题类型**: 数组越界 (cpp-memory-safety)

**危险代码**:
```c
// 数组定义 - 只有9个元素
static __u32 g_digest_a_alg[WD_DIGEST_TYPE_MAX] = {
    A_ALG_SM3, A_ALG_MD5, A_ALG_SHA1, A_ALG_SHA256, A_ALG_SHA224,
    A_ALG_SHA384, A_ALG_SHA512, A_ALG_SHA512_224, A_ALG_SHA512_256
};

// 使用位置
if (msg->mode == WD_DIGEST_NORMAL)
    sqe->type2.mac_key_alg |=
    g_digest_a_alg[msg->alg] << AUTH_ALG_OFFSET;  // 可能越界
```

**证据链**:
1. g_digest_a_alg数组只定义了9个元素（SM3到SHA512-256）
2. WD_DIGEST_TYPE_MAX包含13种算法类型
3. line 1831检查了msg->alg >= WD_DIGEST_TYPE_MAX，但数组只有9个元素
4. 如果msg->alg >= 9且 < WD_DIGEST_TYPE_MAX，会越界读取

**风险分析**:
- 当算法类型为AES_XCBC_MAC_96、AES_XCBC_PRF_128、AES_CMAC、AES_GMAC时
- 数组访问会越界，读取未定义的内存值
- 可能导致硬件配置错误或安全问题

**修复建议**:
```c
// 方法1: 扩展数组到完整大小
static __u32 g_digest_a_alg[WD_DIGEST_TYPE_MAX] = {
    A_ALG_SM3, A_ALG_MD5, A_ALG_SHA1, A_ALG_SHA256, A_ALG_SHA224,
    A_ALG_SHA384, A_ALG_SHA512, A_ALG_SHA512_224, A_ALG_SHA512_256,
    A_ALG_AES_XCBC_MAC_96, A_ALG_AES_XCBC_PRF_128, A_ALG_AES_CMAC,
    A_ALG_AES_GMAC
};

// 方法2: 添加边界检查
if (msg->mode == WD_DIGEST_NORMAL) {
    if (msg->alg < ARRAY_SIZE(g_digest_a_alg))
        sqe->type2.mac_key_alg |= g_digest_a_alg[msg->alg] << AUTH_ALG_OFFSET;
    else
        return -WD_EINVAL;
}
```

---

### 2. 数组越界风险 - g_sec_hmac_full_len数组 [HIGH]

**位置**: hisi_sec.c:523-527, 1846, 1849, 2281, 2285

**问题类型**: 数组越界 (cpp-memory-safety)

**危险代码**:
```c
// 数组定义 - 只有9个元素
static __u32 g_sec_hmac_full_len[WD_DIGEST_TYPE_MAX] = {
    SEC_HMAC_SM3_MAC_LEN, SEC_HMAC_MD5_MAC_LEN, SEC_HMAC_SHA1_MAC_LEN,
    SEC_HMAC_SHA256_MAC_LEN, SEC_HMAC_SHA224_MAC_LEN, SEC_HMAC_SHA384_MAC_LEN,
    SEC_HMAC_SHA512_MAC_LEN, SEC_HMAC_SHA512_224_MAC_LEN, SEC_HMAC_SHA512_256_MAC_LEN
};

// 使用位置
if (msg->has_next && msg->mode == WD_DIGEST_NORMAL &&
    msg->alg == WD_DIGEST_SM3)
    sqe->type2.mac_key_alg = g_sec_hmac_full_len[msg->alg];  // 可能越界
```

**证据链**:
1. g_sec_hmac_full_len数组只定义了9个元素
2. 使用msg->alg作为索引访问数组
3. 如果msg->alg >= 9，会导致越界读取

**风险分析**:
- 与g_digest_a_alg相同的问题
- 长hash模式下可能访问越界内存
- 可能导致错误的MAC长度配置

---

### 3. 空指针解引用风险 - cipher/digest/aead send/recv函数 [HIGH]

**位置**: hisi_sec.c:547-599

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static int cipher_send(struct wd_alg_driver *drv, handle_t ctx, void *msg)
{
    struct hisi_qp *qp = (struct hisi_qp *)wd_ctx_get_priv(ctx);

    if (qp->q_info.hw_type == HISI_QM_API_VER2_BASE)  // 未检查qp是否为NULL
        return hisi_sec_cipher_send(drv, ctx, msg);
    return hisi_sec_cipher_send_v3(drv, ctx, msg);
}

static int cipher_recv(struct wd_alg_driver *drv, handle_t ctx, void *msg)
{
    struct hisi_qp *qp = (struct hisi_qp *)wd_ctx_get_priv(ctx);

    if (qp->q_info.hw_type == HISI_QM_API_VER2_BASE)  // 未检查qp是否为NULL
        return hisi_sec_cipher_recv(drv, ctx, msg);
    return hisi_sec_cipher_recv_v3(drv, ctx, msg);
}

// 同样的问题存在于:
// digest_send(), digest_recv()
// aead_send(), aead_recv()
```

**证据链**:
1. wd_ctx_get_priv(ctx)可能返回NULL
2. 直接访问qp->q_info.hw_type，未检查qp是否为NULL
3. 6个函数（cipher/digest/aead的send/recv）都存在此问题

**风险分析**:
- 如果ctx无效或未正确初始化，qp可能为NULL
- 访问qp->q_info会导致空指针解引用崩溃

**修复建议**:
```c
static int cipher_send(struct wd_alg_driver *drv, handle_t ctx, void *msg)
{
    struct hisi_qp *qp = (struct hisi_qp *)wd_ctx_get_priv(ctx);

    if (!qp) {
        WD_ERR("invalid: qp is NULL!\n");
        return -WD_EINVAL;
    }

    if (qp->q_info.hw_type == HISI_QM_API_VER2_BASE)
        return hisi_sec_cipher_send(drv, ctx, msg);
    return hisi_sec_cipher_send_v3(drv, ctx, msg);
}
```

---

### 4. fill_digest_bd2_alg函数数组访问越界 [MEDIUM]

**位置**: hisi_sec.c:1846, 1849, 1853, 1863

**问题类型**: 数组越界 (cpp-memory-safety)

**危险代码**:
```c
static int fill_digest_bd2_alg(struct wd_digest_msg *msg,
        struct hisi_sec_sqe *sqe)
{
    if (msg->alg >= WD_DIGEST_TYPE_MAX) {
        WD_ERR("failed to check digest alg type, alg = %u\n", msg->alg);
        return -WD_EINVAL;
    }

    // ...

    if (msg->has_next && msg->mode == WD_DIGEST_NORMAL &&
        msg->alg == WD_DIGEST_SM3)
        sqe->type2.mac_key_alg = g_sec_hmac_full_len[msg->alg];  // 可能越界

    if (msg->has_next && msg->mode == WD_DIGEST_HMAC)
        sqe->type2.mac_key_alg = g_sec_hmac_full_len[msg->alg];  // 可能越界

    if (msg->mode == WD_DIGEST_NORMAL)
        sqe->type2.mac_key_alg |=
        g_digest_a_alg[msg->alg] << AUTH_ALG_OFFSET;  // 可能越界
    // ...
}
```

**分析**:
- 虽然检查了msg->alg >= WD_DIGEST_TYPE_MAX
- 但数组g_digest_a_alg和g_sec_hmac_full_len只有9个元素
- 如果9 <= msg->alg < WD_DIGEST_TYPE_MAX，仍会越界

---

### 5. fill_digest_bd3_alg函数数组访问越界 [MEDIUM]

**位置**: hisi_sec.c:2281, 2285, 2290, 2297

**问题类型**: 数组越界 (cpp-memory-safety)

**代码**:
```c
static int fill_digest_bd3_alg(struct wd_digest_msg *msg,
                               struct hisi_sec_sqe3 *sqe)
{
    if (msg->alg >= WD_DIGEST_TYPE_MAX) {
        WD_ERR("failed to check digest type, alg = %u\n", msg->alg);
        return -WD_EINVAL;
    }

    if (msg->has_next && msg->mode == WD_DIGEST_NORMAL &&
        msg->alg == WD_DIGEST_SM3)
        sqe->auth_mac_key |= g_sec_hmac_full_len[msg->alg] <<
                SEC_MAC_OFFSET_V3;  // 可能越界

    if (msg->has_next && msg->mode == WD_DIGEST_HMAC)
        sqe->auth_mac_key |= g_sec_hmac_full_len[msg->alg] <<
                SEC_MAC_OFFSET_V3;  // 可能越界

    if (msg->mode == WD_DIGEST_NORMAL) {
        sqe->auth_mac_key |=
        g_digest_a_alg[msg->alg] << SEC_AUTH_ALG_OFFSET_V3;  // 可能越界
    }
    // ...
}
```

**分析**:
- 与fill_digest_bd2_alg相同的问题
- 需要增加数组边界检查

---

### 6. ctr_iv_inc函数潜在整数溢出 [MEDIUM]

**位置**: hisi_sec.c:910-921

**问题类型**: 整数溢出 (cpp-integer-overflow)

**代码**:
```c
static void ctr_iv_inc(__u8 *counter, __u32 len)
{
    __u32 n = CTR_128BIT_COUNTER;
    __u32 c = len;

    do {
        --n;
        c += counter[n];
        counter[n] = (__u8)c;
        c >>= BYTE_BITS;
    } while (n);
}
```

**分析**:
- 如果counter为NULL，会导致空指针解引用
- 如果len值非常大，c累加可能溢出
- 建议添加参数检查

**修复建议**:
```c
static void ctr_iv_inc(__u8 *counter, __u32 len)
{
    __u32 n = CTR_128BIT_COUNTER;
    __u32 c = len;

    if (!counter)
        return;

    do {
        --n;
        c += counter[n];
        counter[n] = (__u8)c;
        c >>= BYTE_BITS;
    } while (n);
}
```

---

### 7. destroy_cipher_bd2_addr函数参数检查 [MEDIUM]

**位置**: hisi_sec.c:1103-1133

**问题类型**: 参数检查 (cpp-input-validation)

**代码**:
```c
static void destroy_cipher_bd2_addr(struct wd_cipher_msg *msg, struct hisi_sec_sqe *sqe)
{
    struct wd_mm_ops *mm_ops = msg->mm_ops;
    void *mempool;

    /* SVA mode and skip */
    if (!mm_ops || mm_ops->sva_mode)
        return;

    if (!mm_ops->usr) {
        WD_ERR("cipher failed to check memory pool!\n");
        return;
    }

    mempool = mm_ops->usr;
    // 直接使用mm_ops->iova_unmap，未检查是否为NULL
    if (sqe->type2.data_src_addr)
        mm_ops->iova_unmap(mempool, msg->in, ...);
}
```

**分析**:
- 检查了mm_ops->usr，但未检查mm_ops->iova_unmap是否为NULL
- 如果iova_unmap为NULL会导致崩溃

---

### 8. aead_sess_eops_init函数资源泄漏风险 [MEDIUM]

**位置**: hisi_sec.c:654-718

**问题类型**: 资源管理 (cpp-resource-management)

**代码**:
```c
static int aead_sess_eops_init(struct wd_alg_driver *drv,
                   struct wd_mm_ops *mm_ops, void **params)
{
    // ...
    aiv_addr = calloc(1, sizeof(struct wd_aead_aiv_addr));
    if (!aiv_addr) {
        WD_ERR("aead failed to alloc aiv_addr memory!\n");
        return -WD_ENOMEM;
    }

    sec_ctx = (struct hisi_sec_ctx *)drv->priv;  // 未检查sec_ctx是否为NULL
    qp = (struct hisi_qp *)wd_ctx_get_priv(sec_ctx->config.ctxs[0].ctx);  // 未检查qp
    sq_depth = qp->q_info.sq_depth;  // 可能空指针解引用
    // ...
}
```

**分析**:
- 未检查sec_ctx是否为NULL
- 未检查qp是否为NULL
- 未检查sec_ctx->config.ctxs[0].ctx是否有效

---

## drv/hisi_sec.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 3 |
| MEDIUM | 5 |

**重点问题**:
- g_digest_a_alg/g_sec_hmac_full_len数组越界访问(HIGH)
- cipher/digest/aead send/recv函数空指针检查缺失(HIGH)

**建议修复顺序**:
1. 修复数组越界问题，扩展数组或添加边界检查
2. 添加send/recv函数的qp NULL检查
3. 完善fill_digest_bd*_alg函数的边界检查

---

## 检视进度更新

**已完成**: 10/94 文件 (10.6%)

**下次检视**: drv/hisi_hpre.c (HPRE驱动文件)