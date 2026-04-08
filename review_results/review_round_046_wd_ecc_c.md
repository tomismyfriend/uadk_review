# UADK代码检视报告 - 第46轮

## 检视信息
- **检视文件**: wd_ecc.c (根目录)
- **检视时间**: 2026-04-08
- **文件行数**: 2496行
- **检视规则**: cpp-memory-safety, cpp-resource-management, cpp-input-validation

## 检视摘要
| 等级 | 数量 |
|------|------|
| HIGH | 3 |
| MEDIUM | 5 |
| LOW | 3 |
| INFO | 1 |
| POSITIVE | 3 |

---

## 问题详情

### 1. [req->cb未检查NULL] - wd_ecc_poll_ctx函数 - [Severity: HIGH]

**Location**: `wd_ecc.c:2410-2413`

**Dangerous Code**:
```c
msg->req.dst_bytes = recv_msg.req.dst_bytes;
msg->req.status = recv_msg.result;
req = &msg->req;
req->cb(req);  // req->cb未检查NULL
wd_put_msg_to_pool(&wd_ecc_setting.pool, idx, recv_msg.tag);
```

**Risk Analysis**:
与wd_rsa.c、wd_dh.c中的相同问题模式。异步poll函数中直接调用req->cb，如果回调函数为NULL会导致崩溃。

**Fix Suggestion**:
```c
req = &msg->req;
if (!req->cb) {
    WD_ERR("ecc req callback is NULL!\n");
    wd_put_msg_to_pool(&wd_ecc_setting.pool, idx, recv_msg.tag);
    return -WD_EINVAL;
}
req->cb(req);
wd_put_msg_to_pool(&wd_ecc_setting.pool, idx, recv_msg.tag);
```

---

### 2. [driver->send/recv未检查NULL] - wd_do_ecc_sync函数 - [Severity: HIGH]

**Location**: `wd_ecc.c:1651-1652`

**Dangerous Code**:
```c
msg_handle.send = wd_ecc_setting.driver->send;
msg_handle.recv = wd_ecc_setting.driver->recv;

pthread_spin_lock(&ctx->lock);
ret = wd_handle_msg_sync(wd_ecc_setting.driver, &msg_handle, ctx->ctx, &msg,
             &balance, wd_ecc_setting.config.epoll_en);
```

**Risk Analysis**:
driver->send和driver->recv可能为NULL，wd_handle_msg_sync内部会调用这些函数指针，如果为NULL会导致崩溃。

**Fix Suggestion**:
```c
if (!wd_ecc_setting.driver ||
    !wd_ecc_setting.driver->send ||
    !wd_ecc_setting.driver->recv) {
    WD_ERR("ecc driver or driver ops is NULL!\n");
    return -WD_EINVAL;
}
msg_handle.send = wd_ecc_setting.driver->send;
msg_handle.recv = wd_ecc_setting.driver->recv;
```

---

### 3. [rand_t->cb未检查NULL] - generate_random函数 - [Severity: HIGH]

**Location**: `wd_ecc.c:1723`

**Dangerous Code**:
```c
ret = rand_t->cb(k->data, k->dsize, rand_t->usr);
if (ret)
    WD_ERR("failed to do rand cb, ret = %d!\n", ret);
```

**Risk Analysis**:
rand_t->cb回调函数可能为NULL，直接调用会崩溃。generate_random被new_sign_in、wd_sm2_new_enc_in等函数调用。

**Fix Suggestion**:
```c
if (!rand_t->cb) {
    WD_ERR("rand callback is NULL!\n");
    return -WD_EINVAL;
}
ret = rand_t->cb(k->data, k->dsize, rand_t->usr);
```

---

### 4. [hash->cb未检查NULL] - sm2_compute_za_hash函数 - [Severity: MEDIUM]

**Location**: `wd_ecc.c:1778-1779`

**Dangerous Code**:
```c
hash_bytes = get_hash_bytes(hash->type);
*len = hash_bytes;
ret = hash->cb((const char *)p_in, in_len,
        (void *)za, hash_bytes, hash->usr);
```

**Risk Analysis**:
hash->cb回调函数可能为NULL。

**Fix Suggestion**:
```c
if (!hash->cb) {
    WD_ERR("hash callback is NULL!\n");
    free(p_in);
    return -WD_EINVAL;
}
ret = hash->cb((const char *)p_in, in_len, (void *)za, hash_bytes, hash->usr);
```

---

### 5. [mm_ops->alloc未检查NULL] - 多个create函数 - [Severity: MEDIUM]

**Location**: `wd_ecc.c:527, 556, 590, 618, 658, 702, 750`

**Dangerous Code**:
```c
// create_ecc_prikey line 527
data = sess->mm_ops.alloc(sess->mm_ops.usr, dsz);

// create_ecc_pubkey line 556
data = sess->mm_ops.alloc(sess->mm_ops.usr, dsz);

// create_ecc_in line 590
in = sess->mm_ops.alloc(sess->mm_ops.usr, len);
```

**Risk Analysis**:
mm_ops->alloc可能为NULL，直接调用会崩溃。虽然create函数本身检查了返回值，但调用alloc前未检查函数指针。

---

### 6. [mm_ops->free未检查NULL] - release_ecc_prikey/pubkey函数 - [Severity: MEDIUM]

**Location**: `wd_ecc.c:499, 508, 575, 1537, 1557`

**Dangerous Code**:
```c
// release_ecc_prikey line 499
sess->mm_ops.free(sess->mm_ops.usr, prikey->data);

// release_ecc_pubkey line 508
sess->mm_ops.free(sess->mm_ops.usr, pubkey->data);

// wd_ecc_del_in line 1537
sess_t->mm_ops.free(sess_t->mm_ops.usr, in);
```

**Risk Analysis**:
mm_ops->free可能为NULL。

**Fix Suggestion**:
```c
if (sess->mm_ops.free)
    sess->mm_ops.free(sess->mm_ops.usr, prikey->data);
```

---

### 7. [sched_init未检查NULL] - wd_ecc_alloc_sess函数 - [Severity: MEDIUM]

**Location**: `wd_ecc.c:1264-1265`

**Dangerous Code**:
```c
sess->sched_key = (void *)wd_ecc_setting.sched.sched_init(
         wd_ecc_setting.sched.h_sched_ctx, setup->sched_param);
```

**Risk Analysis**:
sched_init可能为NULL。

---

### 8. [get_extend_ops未检查NULL] - wd_ecc_alloc_sess函数 - [Severity: MEDIUM]

**Location**: `wd_ecc.c:1241-1242`

**Dangerous Code**:
```c
if (wd_ecc_setting.driver->get_extend_ops) {
    ret = wd_ecc_setting.driver->get_extend_ops(&sess->eops);
```

**Risk Analysis**:
虽然检查了driver->get_extend_ops是否存在，但如果driver本身为NULL，这里会崩溃。

---

### 9. [prikey->data未检查NULL] - release_ecc_prikey函数 - [Severity: LOW]

**Location**: `wd_ecc.c:498-499`

**Dangerous Code**:
```c
wd_memset_zero(prikey->data, prikey->size);
sess->mm_ops.free(sess->mm_ops.usr, prikey->data);
```

**Risk Analysis**:
prikey->data可能为NULL，wd_memset_zero会出问题。

**Fix Suggestion**:
```c
if (prikey->data) {
    wd_memset_zero(prikey->data, prikey->size);
    if (sess->mm_ops.free)
        sess->mm_ops.free(sess->mm_ops.usr, prikey->data);
}
```

---

### 10. [wd_ecc_setting.driver未检查NULL] - wd_do_ecc_async函数 - [Severity: LOW]

**Location**: `wd_ecc.c:2342`

**Dangerous Code**:
```c
ret = wd_alg_driver_send(wd_ecc_setting.driver, ctx->ctx, msg);
```

**Risk Analysis**:
wd_ecc_setting.driver可能为NULL。

---

### 11. [wd_ecc_sess_eops_init回调检查不完整] - [Severity: LOW]

**Location**: `wd_ecc.c:1184`

**Dangerous Code**:
```c
ret = sess->eops.sess_init(wd_ecc_setting.driver, &sess->eops.params);
```

**Risk Analysis**:
虽然前面检查了sess->eops.sess_init存在，但没有检查wd_ecc_setting.driver是否为NULL。

---

### 12. [信息性发现] - ECC曲线支持表 - [Severity: INFO]

**Location**: `wd_ecc.c:79-93`

**Code**:
```c
static const struct wd_ecc_curve_list curve_list[] = {
    { WD_X25519, "x25519", 256, X25519_256_PARAM },
    { WD_X448, "x448", 448, X448_448_PARAM },
    { WD_SECP128R1, "secp128r1", 128, SECG_P128_R1_PARAM },
    { WD_SECP192K1, "secp192k1", 192, SECG_P192_K1_PARAM },
    ...
    { WD_SM2P256, "sm2", 256, SM2_P256_V1_PARAM },
    { WD_SECP256R1, "secp256r1", 256, SECG_P256_R1_PARAM },
};
```

**Analysis**:
根目录版本支持12种ECC曲线，与v1版本类似。

---

### 13. [正面发现] - 密钥宽度验证 - [Severity: POSITIVE]

**Location**: `wd_ecc.c:1116-1126`

**Code**:
```c
static bool is_key_width_support(__u32 key_bits)
{
    /* key bit width check */
    if (unlikely(key_bits != 128 && key_bits != 192 &&
        key_bits != 224 && key_bits != 256 &&
        key_bits != 320 && key_bits != 384 &&
        key_bits != 448 && key_bits != 521))
        return false;

    return true;
}
```

**Analysis**: 完整的ECC密钥宽度验证，包含224和320（比v1版本更完整）。

---

### 14. [正面发现] - 敏感数据清零 - [Severity: POSITIVE]

**Location**: `wd_ecc.c:498, 834, 1536, 1556`

**Code**:
```c
wd_memset_zero(prikey->data, prikey->size);  // 私钥清零
wd_memset_zero(sess->key.d + 1, sess->key_size);  // d参数清零
wd_memset_zero(in->data, bsz);  // 输入清零
wd_memset_zero(out->data, bsz);  // 输出清零
```

**Analysis**: 正确的敏感数据清理实践。

---

### 15. [正面发现] - wd_do_ecc_async参数检查 - [Severity: POSITIVE]

**Location**: `wd_ecc.c:2317-2320`

**Code**:
```c
if (unlikely(!req || !sess || !req->cb)) {
    WD_ERR("invalid: input parameter NULL!\n");
    return -WD_EINVAL;
}
```

**Analysis**: 异步函数正确检查了req->cb是否为NULL（与同步poll函数不同）。

---

## 总结

### 问题分布
| 问题类型 | 数量 |
|----------|------|
| 空指针检查缺失 | 8 |
| 回调函数检查 | 4 |
| 安全实践 | 3 |

### 与v1版本对比

| 对比项 | wd_ecc.c (根目录) | v1/wd_ecc.c |
|--------|-------------------|-------------|
| 文件行数 | 2496 | 2510 |
| 架构风格 | 新版框架driver抽象 | 旧版框架 |
| poll函数问题 | req->cb未检查NULL | tag->ctx未检查NULL |
| 回调检查 | rand_t->cb未检查 | rand_mt->cb未检查 |
| hash->cb检查 | 未检查NULL | 未检查NULL |
| 整数截断 | 无 | 有(__u16转换) |
| mm_ops检查 | alloc/free未检查 | br_alloc/free未检查 |

**根目录版本相对更完善**：
1. 无整数截断问题（v1版本有）
2. wd_do_ecc_async正确检查req->cb（但poll_ctx未检查）
3. 密钥宽度验证更完整（包含224和320）
4. 架构更现代化

**但仍有共同问题**：
1. poll_ctx中req->cb未检查NULL
2. driver->send/recv未检查NULL
3. rand_t->cb未检查NULL
4. hash->cb未检查NULL
5. mm_ops函数指针未检查NULL

### 根目录RSA/DH/ECC问题模式汇总

| 问题模式 | wd_rsa.c | wd_dh.c | wd_ecc.c |
|---------|----------|---------|----------|
| poll_ctx req->cb未检查NULL | ✓ | ✓ | ✓ |
| driver->send/recv未检查NULL | ✓ | ✓ | ✓ |
| mm_ops->alloc未检查NULL | ✓ | ✓ | ✓ |
| mm_ops->free未检查NULL | ✓ | ✓ | ✓ |
| sched_init未检查NULL | ✓ | ✓ | ✓ |
| 额外回调未检查 | - | - | rand_t->cb, hash->cb |

### 建议优先修复顺序
1. wd_ecc_poll_ctx回调函数检查（HIGH）
2. driver->send/recv NULL检查（HIGH）
3. generate_random rand_t->cb检查（HIGH）
4. sm2_compute_za_hash hash->cb检查（MEDIUM）
5. mm_ops函数指针检查（MEDIUM）