# UADK代码检视报告 - 第38轮

## 检视信息
- **检视文件**: v1/wd_digest.c
- **检视时间**: 2026-04-08
- **文件行数**: 624行
- **检视规则**: cpp-memory-safety, cpp-resource-management, cpp-input-validation, cpp-concurrency-userspace

## 检视摘要
| 等级 | 数量 |
|------|------|
| HIGH | 2 |
| MEDIUM | 4 |
| LOW | 2 |
| INFO | 1 |
| POSITIVE | 2 |

---

## 问题详情

### 1. [数组越界] - g_digest_mac_full_len数组 - [Severity: HIGH]

**Location**: `v1/wd_digest.c:59-65, 390`

**Dangerous Code**:
```c
static __u32 g_digest_mac_full_len[WCRYPTO_MAX_DIGEST_TYPE] = {
    WCRYPTO_DIGEST_SM3_FULL_LEN, WCRYPTO_DIGEST_MD5_FULL_LEN,
    WCRYPTO_DIGEST_SHA1_FULL_LEN, WCRYPTO_DIGEST_SHA256_FULL_LEN,
    WCRYPTO_DIGEST_SHA224_FULL_LEN, WCRYPTO_DIGEST_SHA384_FULL_LEN,
    WCRYPTO_DIGEST_SHA512_FULL_LEN, WCRYPTO_DIGEST_SHA512_224_FULL_LEN,
    WCRYPTO_DIGEST_SHA512_256_FULL_LEN
};  // 只有9个元素

// 第390行使用
if (unlikely(d_opdata->out_bytes < g_digest_mac_full_len[alg])) {
```

**Risk Analysis**:
1. 数组声明大小为WCRYPTO_MAX_DIGEST_TYPE，但初始化只有9个元素
2. 根据g_digest_mac_len数组（第50-57行），WCRYPTO_MAX_DIGEST_TYPE至少为13
3. 当alg >= WCRYPTO_AES_XCBC_MAC_96时，访问g_digest_mac_full_len[alg]会越界
4. 数组越界可能返回垃圾值或导致崩溃

**Evidence Chain**:
1. 第59行：数组声明大小为WCRYPTO_MAX_DIGEST_TYPE
2. 第60-64行：只有9个初始化值
3. g_digest_mac_len数组有13个元素，暗示WCRYPTO_MAX_DIGEST_TYPE >= 13
4. 第390行：使用g_digest_mac_full_len[alg]，alg可能是WCRYPTO_AES_XCBC_MAC_96等值
5. 第108-111行：create_ctx_para_check允许alg < WCRYPTO_MAX_DIGEST_TYPE
6. 第113-117行：NORMAL模式限制alg < WCRYPTO_AES_XCBC_MAC_96，但HMAC模式无此限制

**Fix Suggestion**:
```c
static __u32 g_digest_mac_full_len[WCRYPTO_MAX_DIGEST_TYPE] = {
    WCRYPTO_DIGEST_SM3_FULL_LEN, WCRYPTO_DIGEST_MD5_FULL_LEN,
    WCRYPTO_DIGEST_SHA1_FULL_LEN, WCRYPTO_DIGEST_SHA256_FULL_LEN,
    WCRYPTO_DIGEST_SHA224_FULL_LEN, WCRYPTO_DIGEST_SHA384_FULL_LEN,
    WCRYPTO_DIGEST_SHA512_FULL_LEN, WCRYPTO_DIGEST_SHA512_224_FULL_LEN,
    WCRYPTO_DIGEST_SHA512_256_FULL_LEN,
    /* AES_XCBC_MAC_96, AES_XCBC_PRF_128, AES_CMAC, AES_GMAC 无full_len */
    0, 0, 0, 0
};

// 或在使用时增加范围检查
if (unlikely(alg >= WCRYPTO_AES_XCBC_MAC_96)) {
    WD_ERR("stream mode not support alg %u!\n", alg);
    return -WD_EINVAL;
}
if (unlikely(d_opdata->out_bytes < g_digest_mac_full_len[alg])) {
    ...
}
```

**Explanation**: 补充缺失的数组元素，或在使用前检查alg范围。

---

### 2. [空指针解引用风险] - wcrypto_digest_poll函数 - [Severity: HIGH]

**Location**: `v1/wd_digest.c:589-592`

**Dangerous Code**:
```c
tag = (void *)(uintptr_t)digest_resp->usr_data;
ctx = tag->wcrypto_tag.ctx;
ctx->setup.cb(digest_resp, tag->wcrypto_tag.tag);
wd_put_cookies(&ctx->pool, (void **)&tag, 1);
```

**Risk Analysis**:
1. digest_resp->usr_data可能为0，导致tag为NULL
2. 直接访问tag->wcrypto_tag.ctx会导致空指针解引用崩溃
3. 异步模式下poll函数是关键入口，崩溃会导致整个digest流程失败
4. 与v1/wd_cipher.c中的相同问题模式（第37轮检视发现）

**Evidence Chain**:
1. 第574行：wd_recv接收响应
2. 第577-586行：处理各种返回值情况
3. 第589行：从usr_data获取tag，未验证有效性
4. 第590行：直接访问tag->wcrypto_tag.ctx
5. 第591行：调用ctx->setup.cb，未检查cb是否为NULL

**Fix Suggestion**:
```c
tag = (void *)(uintptr_t)digest_resp->usr_data;
if (!tag) {
    WD_ERR("digest_resp->usr_data is NULL!\n");
    return -WD_EINVAL;
}
ctx = tag->wcrypto_tag.ctx;
if (!ctx || !ctx->setup.cb) {
    WD_ERR("digest ctx or cb is NULL!\n");
    return -WD_EINVAL;
}
ctx->setup.cb(digest_resp, tag->wcrypto_tag.tag);
wd_put_cookies(&ctx->pool, (void **)&tag, 1);
```

**Explanation**: 添加对tag、ctx和cb的NULL检查。

---

### 3. [空指针解引用] - append_tag_restore_status函数 - [Severity: MEDIUM]

**Location**: `v1/wd_digest.c:480`

**Dangerous Code**:
```c
ctxt->io_bytes = *(__u64 *)opdata[ind]->priv;
```

**Risk Analysis**:
1. opdata[ind]->priv可能为NULL
2. 直接将其转换为__u64*并解引用会导致崩溃
3. priv指针用于存储io_bytes的临时值，但在某些路径可能未设置

**Evidence Chain**:
1. 第480行：直接解引用opdata[ind]->priv
2. 第482行：之后设置opdata[ind]->priv = NULL
3. wcrypto_burst_digest第507行调用此函数，当has_next > WCRYPTO_DIGEST_DOING
4. priv的设置在第512行：`cookies[i]->tag.priv = opdata[i]->priv;`
5. 但opdata[i]->priv可能原本就是NULL

**Fix Suggestion**:
```c
if (!opdata[ind]->priv) {
    WD_ERR("opdata[%u]->priv is NULL!\n", ind);
    return -WD_EINVAL;
}
ctxt->io_bytes = *(__u64 *)opdata[ind]->priv;
opdata[ind]->priv = NULL;
```

**Explanation**: 添加priv的NULL检查，防止空指针解引用。

---

### 4. [数组元素未检查NULL] - digest_requests_init函数 - [Severity: MEDIUM]

**Location**: `v1/wd_digest.c:254-264`

**Dangerous Code**:
```c
for (i = 0; i < num; i++) {
    req[i]->has_next = op[i]->has_next;  // req[i]可能为NULL
    req[i]->key = c->key;
    ...
    c->io_bytes += op[i]->in_bytes;  // op[i]可能为NULL
}
```

**Risk Analysis**:
1. req和op数组的元素可能为NULL
2. req[i]来自wcrypto_burst_digest中的`req[i] = &cookies[i]->msg`
3. 如果cookies[i]为NULL，req[i]访问会崩溃
4. op[i]在param_check中已检查，但req[i]未检查

**Evidence Chain**:
1. 第254行：循环开始，直接访问req[i]->has_next
2. wcrypto_burst_digest第504行：`req[i] = &cookies[i]->msg`
3. 第263行：访问op[i]->in_bytes并累加

**Fix Suggestion**:
```c
for (i = 0; i < num; i++) {
    if (!req[i] || !op[i]) {
        WD_ERR("req[%u] or op[%u] is NULL!\n", i, i);
        return -WD_EINVAL;
    }
    req[i]->has_next = op[i]->has_next;
    ...
}
```

**Explanation**: 添加req[i]和op[i]的NULL检查。

---

### 5. [resp数组元素未检查NULL] - digest_recv_sync函数 - [Severity: MEDIUM]

**Location**: `v1/wd_digest.c:365-368`

**Dangerous Code**:
```c
for (i = 0; i < recv_count; i++) {
    d_opdata[i]->out = (void *)resp[i]->out;  // resp[i]可能为NULL
    d_opdata[i]->out_bytes = resp[i]->out_bytes;
    d_opdata[i]->status = resp[i]->result;
}
```

**Risk Analysis**:
1. wd_burst_recv返回的响应数组可能有NULL元素
2. 直接访问resp[i]->out会导致崩溃
3. 与cipher版本相同的问题模式

**Fix Suggestion**:
```c
for (i = 0; i < recv_count; i++) {
    if (!resp[i]) {
        WD_ERR("resp[%u] is NULL!\n", i);
        d_opdata[i]->status = WD_EINVAL;
        continue;
    }
    d_opdata[i]->out = (void *)resp[i]->out;
    ...
}
```

---

### 6. [cookies数组元素未检查NULL] - wcrypto_burst_digest函数 - [Severity: MEDIUM]

**Location**: `v1/wd_digest.c:503-514`

**Dangerous Code**:
```c
for (i = 0; i < num; i++) {
    req[i] = &cookies[i]->msg;  // cookies[i]可能为NULL

    if (opdata[i]->has_next > WCRYPTO_DIGEST_DOING) {
        ret = append_tag_restore_status(ctxt, opdata, req, i);
        ...
    }

    cookies[i]->tag.priv = opdata[i]->priv;  // cookies[i]可能为NULL
```

**Risk Analysis**:
1. wd_get_cookies返回成功不代表所有元素都有效
2. 直接访问cookies[i]->msg或cookies[i]->tag会崩溃

**Fix Suggestion**:
```c
for (i = 0; i < num; i++) {
    if (!cookies[i]) {
        WD_ERR("cookies[%u] is NULL!\n", i);
        ret = -WD_EINVAL;
        goto fail_with_cookies;
    }
    req[i] = &cookies[i]->msg;
    ...
}
```

---

### 7. [指针检查冗余] - del_ctx_key函数 - [Severity: LOW]

**Location**: `v1/wd_digest.c:84`

**Dangerous Code**:
```c
if (br && br->free && ctx->key)
    br->free(br->usr, ctx->key);
```

**Risk Analysis**:
1. br来自`&(ctx->setup.br)`，不可能为NULL
2. 冗余检查不影响正确性，但降低代码可读性

**Fix Suggestion**:
```c
if (br->free && ctx->key)
    br->free(br->usr, ctx->key);
```

---

### 8. [ctx->q未显式检查] - init_digest_cookie函数 - [Severity: LOW]

**Location**: `v1/wd_digest.c:126`

**Dangerous Code**:
```c
__u32 flags = ctx->q->capa.flags;
```

**Risk Analysis**:
1. ctx->q在wcrypto_create_digest_ctx第213行已设置
2. 缺乏防御性检查

**Fix Suggestion**:
```c
if (!ctx || !ctx->q)
    return -WD_EINVAL;
__u32 flags = ctx->q->capa.flags;
```

---

### 9. [信息性发现] - g_digest_mac_len数组完整性 - [Severity: INFO]

**Location**: `v1/wd_digest.c:50-57`

**Code**:
```c
static __u32 g_digest_mac_len[WCRYPTO_MAX_DIGEST_TYPE] = {
    WCRYPTO_DIGEST_SM3_LEN, WCRYPTO_DIGEST_MD5_LEN, WCRYPTO_DIGEST_SHA1_LEN,
    WCRYPTO_DIGEST_SHA256_LEN, WCRYPTO_DIGEST_SHA224_LEN,
    WCRYPTO_DIGEST_SHA384_LEN, WCRYPTO_DIGEST_SHA512_LEN,
    WCRYPTO_DIGEST_SHA512_224_LEN, WCRYPTO_DIGEST_SHA512_256_LEN,
    WCRYPTO_AES_XCBC_MAC_96_LEN, WCRYPTO_AES_XCBC_PRF_128_LEN,
    WCRYPTO_AES_CMAC_LEN, WCRYPTO_AES_GMAC_LEN
};
```

**Analysis**:
此数组初始化完整，覆盖所有WCRYPTO_MAX_DIGEST_TYPE类型的长度值。与g_digest_mac_full_len数组形成对比，后者缺少后4种类型的值。

---

### 10. [正面发现] - create_ctx_para_check完整验证 - [Severity: POSITIVE]

**Location**: `v1/wd_digest.c:88-120`

**Code**:
```c
if (setup->alg >= WCRYPTO_MAX_DIGEST_TYPE) {
    WD_ERR("invalid: the alg %u does not support!\n", setup->alg);
    return -WD_EINVAL;
}

if (setup->mode == WCRYPTO_DIGEST_NORMAL &&
    setup->alg >= WCRYPTO_AES_XCBC_MAC_96) {
    WD_ERR("invalid: the alg %u does not support normal mode!\n", setup->alg);
    return -WD_EINVAL;
}
```

**Analysis**:
良好的边界检查：
1. 检查alg是否在有效范围内
2. 检查特定算法与模式的兼容性

---

### 11. [正面发现] - HMAC密钥长度验证 - [Severity: POSITIVE]

**Location**: `v1/wd_digest.c:267-304`

**Code**:
```c
static int digest_hmac_key_check(enum wcrypto_digest_alg alg, __u16 key_len)
{
    switch (alg) {
    case WCRYPTO_SM3 ... WCRYPTO_SHA224:
        if (key_len > (MAX_HMAC_KEY_SIZE >> 1))
            return -WD_EINVAL;
        break;
    case WCRYPTO_SHA384 ... WCRYPTO_SHA512_256:
        if (key_len > MAX_HMAC_KEY_SIZE)
            return -WD_EINVAL;
        break;
    ...
    }
}
```

**Analysis**:
完善的HMAC密钥长度检查：
1. 不同算法有不同的密钥长度限制
2. 使用枚举范围语法简化代码
3. AES CMAC/XCBC/GMAC有特定的密钥长度要求

---

## 总结

### 问题分布
| 问题类型 | 数量 |
|----------|------|
| 数组越界 | 1 |
| 空指针检查缺失 | 5 |
| 代码质量 | 2 |
| 安全实践 | 2 |

### 主要风险
1. **g_digest_mac_full_len数组越界**：严重的内存安全问题，可能导致崩溃或数据泄露
2. **wcrypto_digest_poll空指针解引用**：异步回调处理入口，崩溃会导致整个digest流程失败

### 与之前检视的关联
1. 第7轮检视的核心wd_digest.c发现g_digest_mac_full_len数组越界问题，此处v1版本存在相同问题
2. 第37轮检视的v1/wd_cipher.c中发现poll函数空指针解引用，v1/wd_digest.c有相同问题模式
3. v1版本代码与核心版本代码结构高度相似，问题模式也高度相似

### 建议优先修复顺序
1. g_digest_mac_full_len数组越界（HIGH）- 与核心版本wd_digest.c的问题一致
2. wcrypto_digest_poll空指针检查（HIGH）
3. append_tag_restore_status空指针检查（MEDIUM）
4. 各函数的数组元素NULL检查（MEDIUM）