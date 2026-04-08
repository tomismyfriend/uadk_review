# UADK代码检视报告 - 第37轮

## 检视信息
- **检视文件**: v1/wd_cipher.c
- **检视时间**: 2026-04-08
- **文件行数**: 587行
- **检视规则**: cpp-memory-safety, cpp-resource-management, cpp-input-validation, cpp-concurrency-userspace, cpp-return-value-checking

## 检视摘要
| 等级 | 数量 |
|------|------|
| HIGH | 1 |
| MEDIUM | 4 |
| LOW | 3 |
| INFO | 1 |
| POSITIVE | 2 |

---

## 问题详情

### 1. [空指针解引用风险] - wcrypto_cipher_poll函数 - [Severity: HIGH]

**Location**: `v1/wd_cipher.c:551-553`

**Dangerous Code**:
```c
tag = (void *)(uintptr_t)cipher_resp->usr_data;
ctx = tag->wcrypto_tag.ctx;
ctx->setup.cb(cipher_resp, tag->wcrypto_tag.tag);
```

**Risk Analysis**:
1. cipher_resp->usr_data可能为0（未初始化或恶意设置），导致tag为NULL或无效指针
2. 直接访问tag->wcrypto_tag.ctx会导致空指针解引用崩溃
3. ctx->setup.cb调用，如果cb为NULL也会崩溃
4. 异步模式下，poll函数是关键的回调处理入口，崩溃会导致整个加密流程失败

**Evidence Chain**:
1. 第537行：`ret = wd_recv(q, (void **)&cipher_resp)`接收响应
2. 第541-544行：处理cipher_resp为NULL的情况（但仅针对-WD_HW_EACCESS错误）
3. 第551行：从cipher_resp->usr_data获取tag，未验证usr_data有效性
4. 第552行：直接访问tag->wcrypto_tag.ctx，未检查tag是否为NULL
5. 第553行：调用ctx->setup.cb，未检查cb是否为NULL

**Fix Suggestion**:
```c
tag = (void *)(uintptr_t)cipher_resp->usr_data;
if (!tag) {
    WD_ERR("cipher_resp->usr_data is NULL!\n");
    wd_put_cookies(&ctx->pool, (void **)&tag, 1);
    return -WD_EINVAL;
}
ctx = tag->wcrypto_tag.ctx;
if (!ctx || !ctx->setup.cb) {
    WD_ERR("cipher ctx or cb is NULL!\n");
    wd_put_cookies(&ctx->pool, (void **)&tag, 1);
    return -WD_EINVAL;
}
ctx->setup.cb(cipher_resp, tag->wcrypto_tag.tag);
```

**Explanation**: 添加对tag、ctx和cb的NULL检查，防止空指针解引用。

---

### 2. [数组元素未检查NULL] - cipher_requests_init函数 - [Severity: MEDIUM]

**Location**: `v1/wd_cipher.c:346-370`

**Dangerous Code**:
```c
for (i = 0; i < num; i++) {
    req[i]->alg = c->setup.alg;  // req[i]可能为NULL
    req[i]->mode = c->setup.mode;
    ...
    req[i]->in = op[i]->in;
```

**Risk Analysis**:
1. req数组中的元素可能为NULL
2. req[i]来自wcrypto_burst_cipher中的`req[i] = &cookies[i]->msg`
3. 如果wd_get_cookies返回成功但cookies[i]为NULL，会导致崩溃
4. op数组在param_check中已检查元素是否为NULL，但req数组未检查

**Evidence Chain**:
1. 第346行：直接访问req[i]->alg，未检查req[i]是否为NULL
2. wcrypto_burst_cipher第478行：`req[i] = &cookies[i]->msg`
3. 如果cookies[i]为NULL，则req[i]也为NULL
4. wd_get_cookies函数返回成功不代表所有cookies元素都有效

**Fix Suggestion**:
```c
for (i = 0; i < num; i++) {
    if (!req[i]) {
        WD_ERR("req[%u] is NULL!\n", i);
        return -WD_EINVAL;
    }
    req[i]->alg = c->setup.alg;
    ...
}
```

**Explanation**: 在循环开始时添加req[i]的NULL检查。

---

### 3. [cookies数组元素未检查NULL] - wcrypto_burst_cipher函数 - [Severity: MEDIUM]

**Location**: `v1/wd_cipher.c:476-481`

**Dangerous Code**:
```c
for (i = 0; i < num; i++) {
    cookies[i]->tag.priv = c_opdata[i]->priv;  // cookies[i]可能为NULL
    req[i] = &cookies[i]->msg;
    if (tag)
        cookies[i]->tag.wcrypto_tag.tag = tag[i];
}
```

**Risk Analysis**:
1. wd_get_cookies返回成功不代表cookies数组所有元素都有效
2. 某些实现可能在cookie池耗尽时返回部分NULL元素
3. 直接访问cookies[i]->tag会导致空指针解引用

**Evidence Chain**:
1. 第472行：wd_get_cookies调用
2. 第476行：直接访问cookies[i]->tag.priv，未检查cookies[i]
3. 第478行：`req[i] = &cookies[i]->msg`，如果cookies[i]为NULL，req[i]为无效地址

**Fix Suggestion**:
```c
for (i = 0; i < num; i++) {
    if (!cookies[i]) {
        WD_ERR("cookies[%u] is NULL!\n", i);
        ret = -WD_EINVAL;
        goto fail_with_cookies;
    }
    cookies[i]->tag.priv = c_opdata[i]->priv;
    ...
}
```

**Explanation**: 添加cookies[i]的NULL检查，防止空指针解引用。

---

### 4. [返回值处理不完整] - cipher_recv_sync函数 - [Severity: MEDIUM]

**Location**: `v1/wd_cipher.c:409-411`

**Dangerous Code**:
```c
for (i = 0; i < recv_count; i++) {
    c_opdata[i]->out = (void *)resp[i]->out;  // resp[i]可能为NULL
    c_opdata[i]->out_bytes = resp[i]->out_bytes;
    c_opdata[i]->status = resp[i]->result;
}
```

**Risk Analysis**:
1. wd_burst_recv返回的响应数组中可能有NULL元素
2. 如果resp[i]为NULL，访问resp[i]->out会导致崩溃
3. 循环中未检查resp[i]的有效性

**Evidence Chain**:
1. 第384-385行：初始化resp数组为ctx_id临时值
2. 第388-389行：wd_burst_recv填充响应，可能返回NULL元素
3. 第409行：直接访问resp[i]->out，未检查resp[i]

**Fix Suggestion**:
```c
for (i = 0; i < recv_count; i++) {
    if (!resp[i]) {
        WD_ERR("resp[%u] is NULL!\n", i);
        c_opdata[i]->status = WD_EINVAL;
        continue;
    }
    c_opdata[i]->out = (void *)resp[i]->out;
    ...
}
```

**Explanation**: 添加resp[i]的NULL检查，防止空指针解引用。

---

### 5. [锁资源管理风险] - wcrypto_del_cipher_ctx函数 - [Severity: MEDIUM]

**Location**: `v1/wd_cipher.c:573-577`

**Dangerous Code**:
```c
wd_uninit_cookie_pool(&c_ctx->pool);
wd_spinlock(&qinfo->qlock);
if (qinfo->ctx_num <= 0) {
    wd_unspinlock(&qinfo->qlock);
    WD_ERR("error:repeat del cipher ctx!\n");
    return;  // 直接返回，未释放ctx
}
```

**Risk Analysis**:
1. 如果ctx_num <= 0（重复删除），函数直接返回
2. 此时ctx内存未释放，可能导致内存泄漏
3. 虽然有错误日志，但资源未正确清理

**Evidence Chain**:
1. 第571行：wd_uninit_cookie_pool调用
2. 第572行：获取锁
3. 第573行：检查ctx_num <= 0
4. 第574-577行：释放锁并返回，但ctx未释放

**Fix Suggestion**:
```c
if (qinfo->ctx_num <= 0) {
    wd_unspinlock(&qinfo->qlock);
    WD_ERR("error:repeat del cipher ctx!\n");
    free(ctx);  // 仍然释放ctx内存
    return;
}
```

**Explanation**: 即使是重复删除错误，也应该释放ctx内存，避免内存泄漏。

---

### 6. [指针检查冗余] - del_ctx_key函数 - [Severity: LOW]

**Location**: `v1/wd_cipher.c:69`

**Dangerous Code**:
```c
if (br && br->free && ctx->key)
    br->free(br->usr, ctx->key);
```

**Risk Analysis**:
1. br来自`&(ctx->setup.br)`，是一个结构体成员的地址
2. 结构体成员地址不可能为NULL
3. `if (br &&`检查是冗余的，不会影响正确性

**Evidence Chain**:
1. 第54行：`struct wd_mm_br *br = &(ctx->setup.br);`
2. &(ctx->setup.br)永远不为NULL
3. 第69行：`if (br &&`检查冗余

**Fix Suggestion**:
```c
if (br->free && ctx->key)
    br->free(br->usr, ctx->key);
```

**Explanation**: 移除冗余的br检查，代码更简洁。

---

### 7. [ctx->q未显式检查] - init_cipher_cookie函数 - [Severity: LOW]

**Location**: `v1/wd_cipher.c:127`

**Dangerous Code**:
```c
__u32 flags = ctx->q->capa.flags;
```

**Risk Analysis**:
1. ctx->q在调用此函数前已设置（wcrypto_create_cipher_ctx第213行）
2. 虽然实际不会为NULL，但缺乏显式检查不符合防御性编程原则
3. 如果代码重构可能导致问题

**Evidence Chain**:
1. 第127行：直接访问ctx->q->capa.flags
2. wcrypto_create_cipher_ctx第213行：ctx->q = q已设置
3. 缺乏显式NULL检查

**Fix Suggestion**:
```c
if (!ctx || !ctx->q)
    return -WD_EINVAL;
__u32 flags = ctx->q->capa.flags;
```

**Explanation**: 添加防御性检查，提高代码健壮性。

---

### 8. [整数溢出风险] - cipher_requests_init函数 - [Severity: LOW]

**Location**: `v1/wd_cipher.c:366`

**Dangerous Code**:
```c
if (unlikely(op[i]->iv_bytes != c->iv_blk_size)) {
    WD_ERR("fail to check IV length %u!\n", i);
    return -WD_EINVAL;
}
```

**Risk Analysis**:
1. iv_bytes和iv_blk_size的比较，如果iv_bytes过大可能导致后续处理溢出
2. 但这里仅检查相等性，未检查范围上限
3. 对于某些加密模式，IV长度有严格限制

**Evidence Chain**:
1. 第366行：仅检查iv_bytes是否等于iv_blk_size
2. 未检查iv_bytes的上限值
3. 可能存在边界情况未覆盖

**Fix Suggestion**:
```c
if (unlikely(op[i]->iv_bytes > MAX_IV_SIZE ||
             op[i]->iv_bytes != c->iv_blk_size)) {
    WD_ERR("fail to check IV length %u, iv_bytes=%u!\n", i, op[i]->iv_bytes);
    return -WD_EINVAL;
}
```

**Explanation**: 添加IV长度上限检查，防止潜在的溢出风险。

---

### 9. [信息性发现] - XTS密钥处理逻辑 - [Severity: INFO]

**Location**: `v1/wd_cipher.c:270-281`

**Code**:
```c
if (setup->mode == WCRYPTO_CIPHER_XTS || setup->mode == WCRYPTO_CIPHER_XTS_GB) {
    if (length & XTS_MODE_KEY_LEN_MASK) {
        WD_ERR("invalid: unsupported XTS key length, length = %u!\n", length);
        return -WD_EINVAL;
    }
    key_len = length >> XTS_MODE_KEY_SHIFT;
    ...
}
```

**Analysis**:
XTS模式的密钥长度检查逻辑正确：
1. 检查密钥长度是否为偶数（XTS需要两个等长密钥）
2. 将总长度除以2得到单个密钥长度
3. 检查单个密钥长度是否为128或256位

这是正确的XTS密钥处理方式。

---

### 10. [正面发现] - create_ctx_para_check完整验证 - [Severity: POSITIVE]

**Location**: `v1/wd_cipher.c:100-121`

**Code**:
```c
if (!q || !q->qinfo || !setup) {
    WD_ERR("%s: input param err!\n", __func__);
    return -WD_EINVAL;
}

if (!setup->br.alloc || !setup->br.free ||
    !setup->br.iova_map || !setup->br.iova_unmap) {
    WD_ERR("create cipher ctx user mm br err!\n");
    return -WD_EINVAL;
}
```

**Analysis**:
良好的输入验证实践：
1. 检查核心参数q、qinfo、setup是否为NULL
2. 检查内存管理回调函数是否全部设置
3. 验证算法名称匹配

---

### 11. [正面发现] - DES弱密钥检测 - [Severity: POSITIVE]

**Location**: `v1/wd_cipher.c:241-250, 323-327`

**Code**:
```c
static int is_des_weak_key(const __u64 *key, __u16 keylen)
{
    int i;

    for (i = 0; i < DES_WEAK_KEY_NUM; i++)
        if (*key == des_weak_key[i])
            return 1;
    return 0;
}
// 在wcrypto_set_cipher_key中调用
if (ctxt->setup.alg == WCRYPTO_CIPHER_DES &&
    is_des_weak_key((__u64 *)key, key_len)) {
    WD_ERR("%s: des weak key!\n", __func__);
    return -WD_EINVAL;
}
```

**Analysis**:
良好的安全实践：
1. DES有4个弱密钥，可能导致加密强度降低
2. 代码正确检测并拒绝弱密钥
3. 符合加密算法最佳实践

---

## 总结

### 问题分布
| 问题类型 | 数量 |
|----------|------|
| 空指针检查缺失 | 4 |
| 资源管理 | 1 |
| 代码质量 | 2 |
| 安全实践 | 2 |

### 主要风险
1. **wcrypto_cipher_poll函数**：异步回调处理入口，空指针解引用会导致整个加密流程崩溃
2. **wcrypto_burst_cipher/cipher_requests_init**：批量操作中的数组元素未检查NULL

### 与之前检视的关联
1. 第36轮检视的v1/wd_util.c和v1/wd_adapter.c中也发现了类似的数组元素未检查NULL问题
2. 核心wd_cipher.c（第6轮检视）中的类似问题模式延续到v1版本

### 建议优先修复顺序
1. wcrypto_cipher_poll函数的空指针检查（HIGH）
2. cipher_requests_init和wcrypto_burst_cipher的数组元素检查（MEDIUM）
3. wcrypto_del_cipher_ctx的资源释放完善（MEDIUM）