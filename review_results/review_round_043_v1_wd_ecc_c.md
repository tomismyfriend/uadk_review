# UADK代码检视报告 - 第43轮

## 检视信息
- **检视文件**: v1/wd_ecc.c
- **检视时间**: 2026-04-08
- **文件行数**: 2510行
- **检视规则**: cpp-memory-safety, cpp-resource-management, cpp-input-validation

## 检视摘要
| 等级 | 数量 |
|------|------|
| HIGH | 2 |
| MEDIUM | 5 |
| LOW | 3 |
| INFO | 1 |
| POSITIVE | 2 |

---

## 问题详情

### 1. [空指针解引用风险] - ecc_poll函数 - [Severity: HIGH]

**Location**: `v1/wd_ecc.c:1655-1658`

**Dangerous Code**:
```c
tag = (void *)(uintptr_t)resp->usr_data;
ctx = tag->ctx;
ctx->setup.cb(resp, tag->tag);
wd_put_cookies(&ctx->pool, (void **)&tag, 1);
```

**Risk Analysis**:
v1版本所有模块都有此问题。ecc_poll被wcrypto_ecxdh_poll、wcrypto_ecdsa_poll、wcrypto_sm2_poll调用。

**Fix Suggestion**:
```c
tag = (void *)(uintptr_t)resp->usr_data;
if (!tag) {
    WD_ERR("resp->usr_data is NULL!\n");
    return -WD_EINVAL;
}
ctx = tag->ctx;
if (!ctx || !ctx->setup.cb) {
    WD_ERR("ecc ctx or cb is NULL!\n");
    return -WD_EINVAL;
}
```

---

### 2. [resp未检查NULL] - ecc_sync_recv函数 - [Severity: HIGH]

**Location**: `v1/wd_ecc.c:1563, 1584-1586`

**Dangerous Code**:
```c
resp = (void *)(uintptr_t)ctx->ctx_id;

do {
    ret = wd_recv(ctx->q, (void **)&resp);
    if (ret > 0) {
        break;
    }
    ...
} while (true);

balance = rx_cnt;
opdata->out = resp->out;  // resp未检查NULL
opdata->out_bytes = resp->out_bytes;
opdata->status = resp->result;
```

---

### 3. [cookie未检查NULL] - do_ecc函数 - [Severity: MEDIUM]

**Location**: `v1/wd_ecc.c:1604, 1614`

**Dangerous Code**:
```c
ret = wd_get_cookies(&ctxt->pool, (void **)&cookie, 1);
if (ret)
    return ret;

if (tag) {
    if (unlikely(!ctxt->setup.cb)) {
        ...
    }
    cookie->tag.tag = tag;  // cookie未检查NULL
```

---

### 4. [整数截断风险] - ecc_request_init函数 - [Severity: MEDIUM]

**Location**: `v1/wd_ecc.c:1477`

**Dangerous Code**:
```c
req->in_bytes = (__u16)op->in_bytes;  // 强制转换为16位
```

**Risk Analysis**:
ECC操作数据量可能超过65535字节。

---

### 5. [prikey->data未检查NULL] - release_ecc_prikey函数 - [Severity: MEDIUM]

**Location**: `v1/wd_ecc.c:302-303`

**Dangerous Code**:
```c
wd_memset_zero(prikey->data, prikey->size);
br_free(br, prikey->data);
```

**Risk Analysis**:
prikey->data可能为NULL。

**Fix Suggestion**:
```c
if (prikey->data) {
    wd_memset_zero(prikey->data, prikey->size);
    br_free(br, prikey->data);
}
```

---

### 6. [重复删除资源泄漏] - wcrypto_del_ecc_ctx函数 - [Severity: MEDIUM]

**Location**: `v1/wd_ecc.c:1206-1209`

**Dangerous Code**:
```c
if (unlikely(qinfo->ctx_num <= 0)) {
    WD_ERR("error:repeat del ecc ctx ctx!\n");
    wd_unspinlock(&qinfo->qlock);
    return;  // ctx内存未释放
}
```

---

### 7. [rand_mt->cb未检查NULL] - generate_random函数 - [Severity: MEDIUM]

**Location**: `v1/wd_ecc.c:1806`

**Dangerous Code**:
```c
ret = rand_mt->cb(k->data, k->dsize, rand_mt->usr);  // rand_mt->cb未检查NULL
```

**Fix Suggestion**:
```c
if (!rand_mt->cb) {
    WD_ERR("rand callback is NULL!\n");
    return -WD_EINVAL;
}
ret = rand_mt->cb(k->data, k->dsize, rand_mt->usr);
```

---

### 8. [hash->cb未检查NULL] - sm2_compute_za_hash函数 - [Severity: LOW]

**Location**: `v1/wd_ecc.c:1862`

**Dangerous Code**:
```c
ret = hash->cb((const char *)p_in, in_len,
        (void *)za, hash_bytes, hash->usr);  // hash->cb未检查NULL
```

---

### 9. [ctx->q未检查NULL] - init_ctx_cookies函数 - [Severity: LOW]

**Location**: `v1/wd_ecc.c:1071, 1073`

**Dangerous Code**:
```c
struct q_info *qinfo = ctx->q->qinfo;
...
__u32 flags = ctx->q->capa.flags;
```

---

### 10. [信息性发现] - ECC曲线参数表 - [Severity: INFO]

**Location**: `v1/wd_ecc.c:86-133`

**Code**:
```c
static const struct wcrypto_ecc_curve_list g_curve_list[] = {
    { .id = CURVE_X25519, .name = "x25519", .key_bits = 256, ... },
    { .id = CURVE_X448, .name = "x448", .key_bits = 448, ... },
    { .id = WCRYPTO_SECP128R1, .name = "secp128r1", .key_bits = 128, ... },
    ...
    { .id = CURVE_SM2P256, .name = "sm2", .key_bits = 256, ... }
};
```

**Analysis**:
支持多种标准ECC曲线：
- X25519, X448（Diffie-Hellman）
- secp128r1, secp192k1, secp256k1, secp521r1（NIST/SECG曲线）
- brainpoolP320r1, brainpoolP384r1（Brainpool曲线）
- SM2（国密曲线）

---

### 11. [正面发现] - 密钥宽度验证 - [Severity: POSITIVE]

**Location**: `v1/wd_ecc.c:993-1006`

**Code**:
```c
static bool is_key_width_support(__u32 key_bits)
{
    if (unlikely(key_bits != 128 &&
        key_bits != 192 &&
        key_bits != 256 &&
        key_bits != 320 &&
        key_bits != 384 &&
        key_bits != 448 &&
        key_bits != 521))
        return false;
    return true;
}
```

**Analysis**: 完整的ECC密钥宽度验证。

---

### 12. [正面发现] - K参数安全检查 - [Severity: POSITIVE]

**Location**: `v1/wd_ecc.c:1740-1767`

**Code**:
```c
static bool check_k_param(struct wd_dtb *k, struct wcrypto_ecc_ctx *ctx)
{
    ...
    if (unlikely(!less_than_latter(k, &cv->n))) {
        WD_ERR("invalid: k >= n!\n");
        return false;
    }

    if (unlikely(is_all_zero(k))) {
        WD_ERR("invalid: k all zero!\n");
        return false;
    }
    return true;
}
```

**Analysis**: 对签名随机数k的安全检查：
1. 检查k < n（阶）
2. 检查k不全为零
这是ECC签名安全的关键检查。

---

## 总结

### 问题分布
| 问题类型 | 数量 |
|----------|------|
| 空指针检查缺失 | 4 |
| 整数截断 | 1 |
| 资源管理 | 1 |
| 回调函数检查 | 2 |
| 安全实践 | 2 |

### v1版本问题模式汇总表（最终）

| 问题模式 | cipher | digest | aead | comp | rsa | dh | ecc |
|---------|--------|--------|------|------|-----|-----|-----|
| poll函数空指针解引用 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| do_xxx resp未检查NULL | - | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| 整数截断风险 | - | - | - | - | ✓ | ✓ | ✓ |
| cookie未检查NULL | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| del_ctx重复删除资源泄漏 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| 回调函数未检查NULL | - | - | - | - | - | - | ✓ |

### v1版本模块总结

v1版本的7个主要模块（wd_cipher.c, wd_digest.c, wd_aead.c, wd_comp.c, wd_rsa.c, wd_dh.c, wd_ecc.c）存在高度一致的问题模式：

1. **poll函数空指针解引用** - 100%存在（7/7）
2. **del_ctx重复删除资源泄漏** - 100%存在（7/7）
3. **cookie未检查NULL** - 100%存在（7/7）
4. **整数截断风险** - 43%存在（3/7）

这表明v1版本代码可能是基于相同模板生成的，需要统一修复这些共性问题。