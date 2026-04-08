# UADK代码检视报告 - 第15轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第15轮
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
15. drv/isa_ce_sm4.c (本轮) - SM4指令集实现文件，468行

### 待检视文件
- drv/hisi_udma.c等drv目录其他文件
- include/目录下的所有头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## drv/isa_ce_sm4.c 检视结果

### 1. 空指针解引用风险 - 加密函数参数检查 [MEDIUM]

**位置**: isa_ce_sm4.c:133-232

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static void sm4_ctr_encrypt(struct wd_cipher_msg *msg, const struct SM4_KEY *rkey_enc)
{
    sm4_v8_ctr32_encrypt(msg->in, msg->out, msg->in_bytes, rkey_enc, msg->iv);  // 未检查各参数
}

static void sm4_cbc_encrypt(struct wd_cipher_msg *msg, const struct SM4_KEY *rkey_enc)
{
    sm4_v8_cbc_encrypt(msg->in, msg->out, msg->in_bytes, rkey_enc, msg->iv, SM4_ENCRYPT);
}

static void sm4_ecb_encrypt(struct wd_cipher_msg *msg, const struct SM4_KEY *rkey_enc)
{
    sm4_v8_ecb_encrypt(msg->in, msg->out, msg->in_bytes, rkey_enc, SM4_ENCRYPT);
}
```

**证据链**:
1. 所有加密函数都直接使用msg->in、msg->out、msg->iv
2. 未检查这些指针是否为NULL
3. 未检查rkey_enc/rkey_dec是否为NULL

**风险分析**:
- 如果msg->in或msg->out为NULL，会导致空指针解引用
- 与isa_ce_sm3.c相同的问题模式

---

### 2. 空指针解引用风险 - sm4_v8_ctr32_encrypt函数 [MEDIUM]

**位置**: isa_ce_sm4.c:80-131

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
static void sm4_v8_ctr32_encrypt(__u8 *in, __u8 *out,
                 __u64 len, const struct SM4_KEY *key, __u8 *iv)
{
    __u8 ecount_buf[SM4_BLOCK_SIZE] = {0};
    __u64 blocks, offset;
    __u32 ctr32;
    __u32 n = 0;

    ctr32 = GETU32(iv + INCREASE_BYTES);  // 未检查iv是否为NULL
    while (len >= SM4_BLOCK_SIZE) {
        // ...
        sm4_v8_ctr32_encrypt_blocks(in, out, blocks, key, iv);  // 未检查各参数
        // ...
    }
    if (len) {
        sm4_v8_ctr32_encrypt_blocks(ecount_buf, ecount_buf, 1, key, iv);
        // ...
        while (len--) {
            out[n] = in[n] ^ ecount_buf[n];  // 未检查in/out是否为NULL
            ++n;
        }
    }
}
```

**分析**:
- 未检查in、out、key、iv是否为NULL
- 直接访问这些指针可能导致崩溃

---

### 3. sm4_xts_encrypt/decrypt函数参数检查 [MEDIUM]

**位置**: isa_ce_sm4.c:301-335

**问题类型**: 空指针检查 (cpp-input-validation)

**代码**:
```c
static int sm4_xts_encrypt(struct wd_cipher_msg *msg, const struct SM4_KEY *rkey)
{
    struct SM4_KEY rkey2;

    if (msg->in_bytes < SM4_BLOCK_SIZE) {
        WD_ERR("invalid: cipher input length is wrong!\n");
        return -WD_EINVAL;
    }

    /* set key for tweak */
    sm4_set_encrypt_key(msg->key + SM4_KEY_SIZE, &rkey2);  // 未检查msg->key是否为NULL

    sm4_v8_xts_encrypt(msg->in, msg->out, msg->in_bytes, rkey, msg->iv, &rkey2);

    return 0;
}
```

**分析**:
- 检查了in_bytes，很好
- 但未检查msg->key是否为NULL
- 未检查msg->in、msg->out、msg->iv是否为NULL

---

### 4. ctr96_inc函数参数检查 [MEDIUM]

**位置**: isa_ce_sm4.c:67-78

**问题类型**: 参数检查 (cpp-input-validation)

**代码**:
```c
static void ctr96_inc(__u8 *counter)
{
    __u32 n = INCREASE_BYTES;
    __u32 c = 1;

    do {
        --n;
        c += counter[n];  // 未检查counter是否为NULL
        counter[n] = (__u8)c;
        c >>= CTR96_SHIFT_BITS;
    } while (n);
}
```

**分析**:
- 未检查counter是否为NULL
- 如果counter为NULL会导致空指针解引用

---

### 5. sm4_set_encrypt_key/decrypt_key函数参数检查 [LOW]

**位置**: isa_ce_sm4.c:234-242

**问题类型**: 参数检查 (cpp-input-validation)

**代码**:
```c
static void sm4_set_encrypt_key(const __u8 *userKey, struct SM4_KEY *key)
{
    sm4_v8_set_encrypt_key(userKey, key);  // 未检查参数
}

static void sm4_set_decrypt_key(const __u8 *userKey, struct SM4_KEY *key)
{
    sm4_v8_set_decrypt_key(userKey, key);  // 未检查参数
}
```

**分析**:
- 未检查userKey和key是否为NULL
- 建议添加防御性检查

---

### 6. sm4_cfb_crypt函数边界检查 [LOW]

**位置**: isa_ce_sm4.c:244-289

**问题类型**: 边界检查 (cpp-memory-safety)

**代码**:
```c
static void sm4_cfb_crypt(struct wd_cipher_msg *msg, const struct SM4_KEY *rkey, const int enc)
{
    unsigned char keydata[SM4_BLOCK_SIZE];
    const unsigned char *src = msg->in;  // 未检查是否为NULL
    unsigned char *dst = msg->out;       // 未检查是否为NULL
    __u32 nbytes = msg->in_bytes;
    // ...

    /* store new IV  */
    if (enc == SM4_ENCRYPT) {
        if (msg->out_bytes >= msg->iv_bytes)
            memcpy(msg->iv, msg->out + msg->out_bytes - msg->iv_bytes, msg->iv_bytes);
        else
            memcpy(msg->iv, msg->out, msg->out_bytes);  // 未检查msg->iv是否为NULL
    }
    // ...
}
```

**分析**:
- 未检查msg->in、msg->out、msg->iv是否为NULL
- 边界检查存在，但指针检查缺失

---

## drv/isa_ce_sm4.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 0 |
| MEDIUM | 4 |
| LOW | 2 |

**重点问题**:
- 加密函数参数检查不完整(MEDIUM)
- sm4_v8_ctr32_encrypt等底层函数缺少NULL检查

**建议修复顺序**:
1. 在各加密函数中添加msg->in/out/iv/key的NULL检查
2. 完善sm4_v8_ctr32_encrypt函数的参数检查
3. 添加ctr96_inc等辅助函数的防御性检查

---

## 检视进度更新

**已完成**: 15/94 文件 (16.0%)

**下次检视**: drv/hisi_udma.c (UDMA驱动文件)