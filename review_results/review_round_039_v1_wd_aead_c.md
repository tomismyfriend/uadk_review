# UADK代码检视报告 - 第39轮

## 检视信息
- **检视文件**: v1/wd_aead.c
- **检视时间**: 2026-04-08
- **文件行数**: 780行
- **检视规则**: cpp-memory-safety, cpp-resource-management, cpp-input-validation

## 检视摘要
| 等级 | 数量 |
|------|------|
| HIGH | 2 |
| MEDIUM | 5 |
| LOW | 2 |
| INFO | 1 |
| POSITIVE | 2 |

---

## 问题详情

### 1. [数组越界] - g_aead_mac_len数组 - [Severity: HIGH]

**Location**: `v1/wd_aead.c:34-39, 305, 347`

**Dangerous Code**:
```c
static int g_aead_mac_len[WCRYPTO_MAX_DIGEST_TYPE] = {
    WCRYPTO_SM3_LEN, WCRYPTO_MD5_LEN, WCRYPTO_SHA1_LEN,
    WCRYPTO_SHA256_LEN, WCRYPTO_SHA224_LEN,
    WCRYPTO_SHA384_LEN, WCRYPTO_SHA512_LEN,
    WCRYPTO_SHA512_224_LEN, WCRYPTO_SHA512_256_LEN
};  // 只有9个元素，但声明大小为WCRYPTO_MAX_DIGEST_TYPE

// 第305行使用
authsize > g_aead_mac_len[ctxt->setup.dalg]

// 第347行使用
return g_aead_mac_len[ctxt->setup.dalg];
```

**Risk Analysis**:
1. 数组声明大小为WCRYPTO_MAX_DIGEST_TYPE（至少13），但初始化只有9个元素
2. 当dalg >= WCRYPTO_AES_XCBC_MAC_96时，访问会越界
3. 未初始化的元素可能是0或垃圾值，导致authsize检查逻辑错误
4. 在wcrypto_aead_setauthsize中，错误的检查可能允许无效的authsize
5. 在wcrypto_aead_get_maxauthsize中，可能返回错误的值

**Evidence Chain**:
1. 第34行：数组声明大小为WCRYPTO_MAX_DIGEST_TYPE
2. 第35-38行：只有9个初始化值
3. 第305行：wcrypto_aead_setauthsize中使用
4. 第347行：wcrypto_aead_get_maxauthsize中使用
5. 第304行：检查dalg < WCRYPTO_MAX_DIGEST_TYPE，但允许dalg在[9,12]范围内

**Fix Suggestion**:
```c
static int g_aead_mac_len[WCRYPTO_MAX_DIGEST_TYPE] = {
    WCRYPTO_SM3_LEN, WCRYPTO_MD5_LEN, WCRYPTO_SHA1_LEN,
    WCRYPTO_SHA256_LEN, WCRYPTO_SHA224_LEN,
    WCRYPTO_SHA384_LEN, WCRYPTO_SHA512_LEN,
    WCRYPTO_SHA512_224_LEN, WCRYPTO_SHA512_256_LEN,
    /* AES_XCBC_MAC_96, AES_XCBC_PRF_128, AES_CMAC, AES_GMAC */
    WCRYPTO_AES_XCBC_MAC_96_LEN, WCRYPTO_AES_XCBC_PRF_128_LEN,
    WCRYPTO_AES_CMAC_LEN, WCRYPTO_AES_GMAC_LEN
};
```

**Explanation**: 补充缺失的数组元素，确保数组完整初始化。

---

### 2. [空指针解引用风险] - wcrypto_aead_poll函数 - [Severity: HIGH]

**Location**: `v1/wd_aead.c:744-747`

**Dangerous Code**:
```c
tag = (void *)(uintptr_t)aead_resp->usr_data;
ctx = tag->wcrypto_tag.ctx;
ctx->setup.cb(aead_resp, tag->wcrypto_tag.tag);
aead_requests_uninit(&aead_resp, ctx, 1);
wd_put_cookies(&ctx->pool, (void **)&tag, 1);
```

**Risk Analysis**:
1. aead_resp->usr_data可能为0，导致tag为NULL
2. 直接访问tag->wcrypto_tag.ctx会导致崩溃
3. ctx->setup.cb调用，如果cb为NULL也会崩溃
4. 与v1/wd_cipher.c和v1/wd_digest.c中的相同问题模式

**Evidence Chain**:
1. 第730行：wd_recv接收响应
2. 第733-742行：处理各种返回值情况
3. 第744行：从usr_data获取tag，未验证有效性
4. 第745行：直接访问tag->wcrypto_tag.ctx
5. 第746行：调用ctx->setup.cb

**Fix Suggestion**:
```c
tag = (void *)(uintptr_t)aead_resp->usr_data;
if (!tag) {
    WD_ERR("aead_resp->usr_data is NULL!\n");
    return -WD_EINVAL;
}
ctx = tag->wcrypto_tag.ctx;
if (!ctx || !ctx->setup.cb) {
    WD_ERR("aead ctx or cb is NULL!\n");
    return -WD_EINVAL;
}
ctx->setup.cb(aead_resp, tag->wcrypto_tag.tag);
```

---

### 3. [数组元素未检查NULL] - aead_requests_init函数 - [Severity: MEDIUM]

**Location**: `v1/wd_aead.c:531`

**Dangerous Code**:
```c
for (i = 0; i < num; i++) {
    ret = check_op_data(op, ctx, i);  // op[i]已检查
    if (ret)
        goto err_uninit_requests;

    req[i]->calg = ctx->setup.calg;  // req[i]可能为NULL
```

**Risk Analysis**:
1. req[i]来自wcrypto_burst_aead中的`req[i] = &cookies[i]->msg`
2. 如果cookies[i]为NULL，req[i]访问会崩溃
3. op数组在param_check中已检查，但req数组未检查

**Fix Suggestion**:
```c
for (i = 0; i < num; i++) {
    if (!req[i]) {
        WD_ERR("req[%u] is NULL!\n", i);
        ret = -WD_EINVAL;
        goto err_uninit_requests;
    }
    ret = check_op_data(op, ctx, i);
    ...
}
```

---

### 4. [req参数未检查NULL] - aead_requests_uninit函数 - [Severity: MEDIUM]

**Location**: `v1/wd_aead.c:447`

**Dangerous Code**:
```c
for (i = 0; i < num; i++) {
    if (req[i]->aiv)  // req[i]可能为NULL
        ctx->setup.br.free(ctx->setup.br.usr, req[i]->aiv);
}
```

**Risk Analysis**:
1. 如果req[i]为NULL，访问req[i]->aiv会崩溃
2. 此函数在错误处理路径中调用

**Fix Suggestion**:
```c
for (i = 0; i < num; i++) {
    if (req[i] && req[i]->aiv)
        ctx->setup.br.free(ctx->setup.br.usr, req[i]->aiv);
}
```

---

### 5. [req参数未检查NULL] - fill_stream_msg函数 - [Severity: MEDIUM]

**Location**: `v1/wd_aead.c:486, 558`

**Dangerous Code**:
```c
static void fill_stream_msg(struct wcrypto_aead_msg *req, ...)
{
    req->msg_state = op->state;  // req可能为NULL
    req->mac = ctx->mac;
    ...
}

// 调用点
fill_stream_msg(req[0], op[0], ctx);  // req[0]可能为NULL
```

**Risk Analysis**:
1. req参数可能为NULL（如果cookies[0]为NULL）
2. 直接访问req->msg_state会崩溃
3. 此函数无返回值，无法报告错误

**Fix Suggestion**:
```c
static int fill_stream_msg(struct wcrypto_aead_msg *req, ...)
{
    if (!req || !op)
        return -WD_EINVAL;

    req->msg_state = op->state;
    ...
    return WD_SUCCESS;
}

// 调用时检查返回值
ret = fill_stream_msg(req[0], op[0], ctx);
if (ret)
    goto err_uninit_requests;
```

---

### 6. [resp数组元素未检查NULL] - aead_recv_sync函数 - [Severity: MEDIUM]

**Location**: `v1/wd_aead.c:600-603`

**Dangerous Code**:
```c
for (i = 0; i < recv_count; i++) {
    a_opdata[i]->out = (void *)resp[i]->out;  // resp[i]可能为NULL
    a_opdata[i]->out_bytes = resp[i]->out_bytes;
    a_opdata[i]->status = resp[i]->result;
}
```

**Risk Analysis**:
与cipher/digest版本相同的问题模式。

**Fix Suggestion**:
```c
for (i = 0; i < recv_count; i++) {
    if (!resp[i]) {
        a_opdata[i]->status = WD_EINVAL;
        continue;
    }
    ...
}
```

---

### 7. [cookies数组元素未检查NULL] - wcrypto_burst_aead函数 - [Severity: MEDIUM]

**Location**: `v1/wd_aead.c:669-670`

**Dangerous Code**:
```c
for (i = 0; i < num; i++) {
    cookies[i]->tag.priv = opdata[i]->priv;  // cookies[i]可能为NULL
    req[i] = &cookies[i]->msg;
```

**Risk Analysis**:
与cipher/digest版本相同的问题模式。

**Fix Suggestion**:
```c
for (i = 0; i < num; i++) {
    if (!cookies[i]) {
        WD_ERR("cookies[%u] is NULL!\n", i);
        ret = -WD_EINVAL;
        goto fail_with_cookies;
    }
    ...
}
```

---

### 8. [指针检查冗余] - del_ctx_key函数 - [Severity: LOW]

**Location**: `v1/wd_aead.c:86`

**Dangerous Code**:
```c
if (br && br->free) {
```

**Risk Analysis**:
br来自`&(ctx->setup.br)`，不可能为NULL。

**Fix Suggestion**:
```c
if (br->free) {
```

---

### 9. [ctx->q未显式检查] - init_aead_cookie函数 - [Severity: LOW]

**Location**: `v1/wd_aead.c:143`

**Dangerous Code**:
```c
__u32 flags = ctx->q->capa.flags;
```

**Risk Analysis**:
缺乏防御性检查。

---

### 10. [信息性发现] - 资源分配完整性 - [Severity: INFO]

**Location**: `v1/wd_aead.c:232-251, 260-267`

**Code**:
```c
ctx->ckey = setup->br.alloc(setup->br.usr, MAX_CIPHER_KEY_SIZE);
if (!ctx->ckey) {
    ...
    goto free_ctx;
}
ctx->akey = setup->br.alloc(setup->br.usr, MAX_AEAD_KEY_SIZE);
if (!ctx->akey) {
    ...
    goto free_ctx_ckey;
}
...
// 错误处理路径完整
free_ctx_mac:
    setup->br.free(setup->br.usr, ctx->mac);
free_ctx_civ:
    setup->br.free(setup->br.usr, ctx->civ);
free_ctx_akey:
    setup->br.free(setup->br.usr, ctx->akey);
free_ctx_ckey:
    setup->br.free(setup->br.usr, ctx->ckey);
```

**Analysis**:
良好的资源分配和错误处理模式：
1. 按顺序分配4个资源（ckey, akey, civ, mac）
2. 每次分配后立即检查返回值
3. 错误路径正确释放已分配的资源
4. 使用goto链式错误处理，确保所有资源被释放

---

### 11. [正面发现] - authsize验证完整性 - [Severity: POSITIVE]

**Location**: `v1/wd_aead.c:279-315`

**Code**:
```c
if (ctxt->setup.cmode == WCRYPTO_CIPHER_CCM) {
    if (authsize < AEAD_CCM_GCM_MIN ||
        authsize > AEAD_CCM_GCM_MAX ||
        authsize % (AEAD_CCM_GCM_MIN >> 1))
        return -WD_EINVAL;
} else if (ctxt->setup.cmode == WCRYPTO_CIPHER_GCM) {
    if (authsize < AEAD_CCM_GCM_MIN << 1 ||
        authsize > AEAD_CCM_GCM_MAX)
        return -WD_EINVAL;
} else {
    if (ctxt->setup.dalg >= WCRYPTO_MAX_DIGEST_TYPE || !authsize ||
        authsize > g_aead_mac_len[ctxt->setup.dalg])
        return -WD_EINVAL;
}
```

**Analysis**:
完整的authsize验证：
1. CCM模式：检查范围4-16且必须是偶数
2. GCM模式：检查范围8-16
3. 其他模式：检查dalg范围和authsize上限
4. 验证逻辑符合各AEAD模式的标准要求

---

## 总结

### 问题分布
| 问题类型 | 数量 |
|----------|------|
| 数组越界 | 1 |
| 空指针检查缺失 | 6 |
| 代码质量 | 2 |
| 安全实践 | 2 |

### 主要风险
1. **g_aead_mac_len数组越界**：与v1/wd_digest.c中的g_digest_mac_full_len问题类似
2. **wcrypto_aead_poll空指针解引用**：v1版本三个模块（cipher/digest/aead）都有此问题

### 与之前检视的关联
1. 第8轮检视的核心wd_aead.c发现g_aead_mac_len数组越界问题，v1版本存在相同问题
2. 第37-38轮检视的v1/wd_cipher.c和v1/wd_digest.c中发现poll函数空指针解引用，v1/wd_aead.c有相同问题模式
3. v1版本的三个加密模块（cipher/digest/aead）代码结构高度相似，问题模式也高度一致

### 建议优先修复顺序
1. g_aead_mac_len数组越界（HIGH）
2. wcrypto_aead_poll空指针检查（HIGH）
3. 各函数的数组元素NULL检查（MEDIUM）

### v1版本问题模式总结
通过第37-39轮检视，发现v1版本的wd_cipher.c、wd_digest.c、wd_aead.c存在以下共同问题模式：
| 问题模式 | cipher | digest | aead |
|---------|--------|--------|------|
| poll函数空指针解引用 | ✓ | ✓ | ✓ |
| 数组越界（静态数组初始化不完整） | - | ✓ | ✓ |
| requests_init数组元素未检查NULL | ✓ | ✓ | ✓ |
| recv_sync resp元素未检查NULL | ✓ | ✓ | ✓ |
| burst_xxx cookies元素未检查NULL | ✓ | ✓ | ✓ |
| del_ctx资源泄漏（重复删除） | ✓ | ✓ | ✓ |
| del_ctx_key br检查冗余 | ✓ | ✓ | ✓ |
| init_xxx_cookie ctx->q未检查 | ✓ | ✓ | ✓ |