# UADK代码检视报告 - 第41轮

## 检视信息
- **检视文件**: v1/wd_rsa.c
- **检视时间**: 2026-04-08
- **文件行数**: 1144行
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

### 1. [空指针解引用风险] - wcrypto_rsa_poll函数 - [Severity: HIGH]

**Location**: `v1/wd_rsa.c:1102-1105`

**Dangerous Code**:
```c
tag = (void *)(uintptr_t)resp->usr_data;
ctx = tag->ctx;
ctx->setup.cb(resp, tag->tag);
wd_put_cookies(&ctx->pool, (void **)&tag, 1);
```

**Risk Analysis**:
1. resp->usr_data可能为0，导致tag为NULL
2. 直接访问tag->ctx会导致崩溃
3. ctx->setup.cb调用，如果cb为NULL也会崩溃
4. v1版本所有模块（cipher/digest/aead/comp/rsa）都有此问题

**Evidence Chain**:
1. 第1094行：wd_recv接收响应
2. 第1095-1100行：处理返回值
3. 第1102行：从usr_data获取tag，未验证有效性
4. 第1103行：直接访问tag->ctx
5. 第1104行：调用ctx->setup.cb

**Fix Suggestion**:
```c
tag = (void *)(uintptr_t)resp->usr_data;
if (!tag) {
    WD_ERR("resp->usr_data is NULL!\n");
    return -WD_EINVAL;
}
ctx = tag->ctx;
if (!ctx || !ctx->setup.cb) {
    WD_ERR("rsa ctx or cb is NULL!\n");
    return -WD_EINVAL;
}
ctx->setup.cb(resp, tag->tag);
wd_put_cookies(&ctx->pool, (void **)&tag, 1);
```

---

### 2. [resp未检查NULL] - wcrypto_do_rsa函数 - [Severity: HIGH]

**Location**: `v1/wd_rsa.c:1048, 1069-1072`

**Dangerous Code**:
```c
resp = (void *)(uintptr_t)ctxt->ctx_id;  // 初始化为临时值

do {
    ret = wd_recv(ctxt->q, (void **)&resp);
    if (ret > 0) {
        break;  // ret > 0跳出，但resp可能为NULL
    } else if (!ret) {
        ...
    } else {
        ...
        goto fail_with_cookie;
    }
} while (true);

balance = rx_cnt;
opdata->out = (void *)resp->out;  // resp可能为NULL
opdata->out_bytes = resp->out_bytes;
opdata->status = resp->result;
```

**Risk Analysis**:
1. 循环在ret > 0时跳出，但跳出后resp未检查
2. 如果wd_recv返回特殊情况导致resp为NULL，访问resp->out会崩溃
3. 与v1/wd_comp.c中的相同问题模式

**Fix Suggestion**:
```c
} while (true);

if (!resp) {
    WD_ERR("resp is NULL after wd_recv!\n");
    ret = -WD_EINVAL;
    goto fail_with_cookie;
}

balance = rx_cnt;
opdata->out = (void *)resp->out;
```

---

### 3. [kout参数未检查NULL] - wcrypto_set_rsa_kg_out_crt_psz函数 - [Severity: MEDIUM]

**Location**: `v1/wd_rsa.c:378-386`

**Dangerous Code**:
```c
void wcrypto_set_rsa_kg_out_crt_psz(struct wcrypto_rsa_kg_out *kout,
                    size_t qinv_sz,
                    size_t dq_sz,
                    size_t dp_sz)
{
    kout->qinvbytes = qinv_sz;  // kout未检查NULL
    kout->dqbytes = dq_sz;
    kout->dpbytes = dp_sz;
}
```

**Risk Analysis**:
1. kout参数未检查是否为NULL
2. 如果kout为NULL，访问kout->qinvbytes会崩溃

**Fix Suggestion**:
```c
void wcrypto_set_rsa_kg_out_crt_psz(struct wcrypto_rsa_kg_out *kout,
                    size_t qinv_sz,
                    size_t dq_sz,
                    size_t dp_sz)
{
    if (!kout) {
        WD_ERR("kout is NULL!\n");
        return;
    }
    kout->qinvbytes = qinv_sz;
    ...
}
```

---

### 4. [kout参数未检查NULL] - wcrypto_set_rsa_kg_out_psz函数 - [Severity: MEDIUM]

**Location**: `v1/wd_rsa.c:388-394`

**Dangerous Code**:
```c
void wcrypto_set_rsa_kg_out_psz(struct wcrypto_rsa_kg_out *kout,
                size_t d_sz,
                size_t n_sz)
{
    kout->dbytes = d_sz;  // kout未检查NULL
    kout->nbytes = n_sz;
}
```

**Risk Analysis**:
与上述问题相同，kout未检查NULL。

---

### 5. [cookie未检查NULL] - do_rsa_prepare函数 - [Severity: MEDIUM]

**Location**: `v1/wd_rsa.c:983, 990`

**Dangerous Code**:
```c
ret = wd_get_cookies(&ctxt->pool, (void **)&cookie, 1);
if (ret)
    return ret;

if (tag)
    cookie->tag.tag = tag;  // cookie可能为NULL

req = &cookie->msg;
```

**Risk Analysis**:
1. wd_get_cookies返回0不代表cookie一定有效
2. cookie可能为NULL，访问cookie->tag会崩溃

**Fix Suggestion**:
```c
ret = wd_get_cookies(&ctxt->pool, (void **)&cookie, 1);
if (ret)
    return ret;

if (!cookie) {
    WD_ERR("cookie is NULL!\n");
    return -WD_EINVAL;
}
```

---

### 6. [整数截断风险] - rsa_request_init函数 - [Severity: MEDIUM]

**Location**: `v1/wd_rsa.c:918, 920`

**Dangerous Code**:
```c
req->in_bytes = (__u16)op->in_bytes;   // 强制转换为16位
req->out_bytes = (__u16)op->out_bytes;
```

**Risk Analysis**:
1. op->in_bytes和op->out_bytes可能是32位或64位值
2. 强制转换为__u16（16位）可能导致数据截断
3. RSA操作数据量可能超过65535字节

**Evidence Chain**:
1. 第918行：强制转换为__u16
2. RSA密钥长度可达4096位（512字节），但数据量可能更大

**Fix Suggestion**:
```c
if (op->in_bytes > UINT16_MAX || op->out_bytes > UINT16_MAX) {
    WD_ERR("in_bytes or out_bytes overflow!\n");
    return -WD_EINVAL;
}
req->in_bytes = (__u16)op->in_bytes;
req->out_bytes = (__u16)op->out_bytes;
```

---

### 7. [未检查setup->cb] - wcrypto_do_rsa函数 - [Severity: MEDIUM]

**Location**: `v1/wd_rsa.c:1045-1046`

**Dangerous Code**:
```c
if (tag)
    return ret;  // 异步模式，但未检查cb是否设置
```

**Risk Analysis**:
1. do_rsa_prepare函数第978-981行检查了tag和cb的关系
2. 但如果tag为NULL且cb为NULL，同步模式可以工作
3. 如果tag非NULL且cb为NULL，会报错，这已经处理

**Note**: 此问题已在do_rsa_prepare中处理。

---

### 8. [重复删除资源泄漏] - wcrypto_del_rsa_ctx函数 - [Severity: LOW]

**Location**: `v1/wd_rsa.c:1129-1133`

**Dangerous Code**:
```c
if (qinfo->ctx_num <= 0) {
    wd_unspinlock(&qinfo->qlock);
    WD_ERR("error: repeat del rsa ctx!\n");
    return;  // ctx内存未释放
}
```

**Risk Analysis**:
1. ctx_num <= 0（重复删除）时直接返回
2. cx内存未释放，可能导致内存泄漏
3. 与其他模块相同的问题模式

**Fix Suggestion**:
```c
if (qinfo->ctx_num <= 0) {
    wd_unspinlock(&qinfo->qlock);
    WD_ERR("error: repeat del rsa ctx!\n");
    del_ctx_key(st, cx);
    del_ctx(cx);  // 释放ctx内存
    return;
}
```

---

### 9. [指针检查冗余] - del_ctx_key函数 - [Severity: LOW]

**Location**: `v1/wd_rsa.c:495`

**Dangerous Code**:
```c
if (br && br->free) {
```

**Risk Analysis**:
br来自`&setup->br`，不可能为NULL。

**Fix Suggestion**:
```c
if (br->free) {
```

---

### 10. [结构体指针检查不一致] - wcrypto_get_rsa_kg_out_params函数 - [Severity: LOW]

**Location**: `v1/wd_rsa.c:329-348`

**Dangerous Code**:
```c
void wcrypto_get_rsa_kg_out_params(struct wcrypto_rsa_kg_out *kout, ...)
{
    if (!kout) {
        WD_ERR("input null at get key gen parameters!\n");
        return;
    }

    if (d && kout->d) {  // 检查了d和kout->d
        ...
    }

    if (n && kout->n) {  // 检查了n和kout->n
        ...
    }
}
```

**Risk Analysis**:
1. 此函数正确检查了参数
2. 但wcrypto_set_rsa_kg_out_psz和wcrypto_set_rsa_kg_out_crt_psz未检查kout
3. 同一组API的检查不一致

---

### 11. [信息性发现] - RSA密钥大小验证 - [Severity: INFO]

**Location**: `v1/wd_rsa.c:573-582`

**Code**:
```c
switch (setup->key_bits) {
case RSA_KEYSIZE_1024:
case RSA_KEYSIZE_2048:
case RSA_KEYSIZE_3072:
case RSA_KEYSIZE_4096:
    return WD_SUCCESS;
default:
    WD_ERR("invalid: rsa key_bits %u is error!\n", setup->key_bits);
    return -WD_EINVAL;
}
```

**Analysis**:
良好的密钥长度验证：
1. 只支持标准的RSA密钥长度：1024、2048、3072、4096位
2. 不支持非标准密钥长度，这是正确的安全实践
3. 符合RSA算法规范

---

### 12. [正面发现] - 完整的参数验证 - [Severity: POSITIVE]

**Location**: `v1/wd_rsa.c:146-193`

**Code**:
```c
static int kg_in_param_check(void *ctx, struct wd_dtb *e,
                 struct wd_dtb *p, struct wd_dtb *q)
{
    ...
    if (unlikely(!c || !e || !p || !q)) {
        WD_ERR("ctx br->alloc kg_in memory fail!\n");
        return -WD_EINVAL;
    }

    if (unlikely(!c->key_size || c->key_size > RSA_MAX_KEY_SIZE)) {
        WD_ERR("key size err at create kg in!\n");
        return -WD_EINVAL;
    }

    if (unlikely(!e->dsize || e->dsize > c->key_size || !e->data)) {
        WD_ERR("e para err at create kg in!\n");
        return -WD_EINVAL;
    }
    ...
    if (unlikely(br->get_bufsize &&
        br->get_bufsize(br->usr) < (kg_in_size + sizeof(*kg_in)))) {
        WD_ERR("Blk_size < need_size<0x%lx>.\n", ...);
        return -WD_EINVAL;
    }
}
```

**Analysis**:
完整的参数验证：
1. 检查核心参数是否为NULL
2. 检查key_size范围
3. 检查各参数的大小范围和数据指针
4. 检查缓冲区大小是否足够

---

### 13. [正面发现] - 错误处理路径完整 - [Severity: POSITIVE]

**Location**: `v1/wd_rsa.c:431-488`

**Code**:
```c
static int create_ctx_key(struct wcrypto_rsa_ctx_setup *setup,
              struct wcrypto_rsa_ctx *ctx)
{
    ...
    ctx->prikey = br->alloc(br->usr, len);
    if (!ctx->prikey) {
        WD_ERR("alloc prikey2 fail!\n");
        return -WD_ENOMEM;
    }
    ...
    ctx->pubkey = br->alloc(br->usr, len);
    if (!ctx->pubkey) {
        br->free(br->usr, ctx->prikey);  // 释放已分配的prikey
        WD_ERR("alloc pubkey fail!\n");
        return -WD_ENOMEM;
    }
    ...
}
```

**Analysis**:
良好的资源管理和错误处理：
1. 每次分配后立即检查返回值
2. 后续分配失败时，正确释放之前分配的资源
3. 使用goto链式错误处理

---

## 总结

### 问题分布
| 问题类型 | 数量 |
|----------|------|
| 空指针检查缺失 | 4 |
| 整数截断 | 1 |
| 资源管理 | 1 |
| 代码质量 | 2 |
| 安全实践 | 2 |

### 主要风险
1. **wcrypto_rsa_poll空指针解引用**：v1版本所有模块都有此问题
2. **wcrypto_do_rsa resp未检查**：与comp模块相同问题模式

### 与之前检视的关联
1. 第37-40轮检视的v1版本模块都有poll函数空指针解引用问题
2. v1/wd_rsa.c代码结构与其他模块相似，问题模式也高度一致

### v1版本问题模式汇总表（更新）

| 问题模式 | cipher | digest | aead | comp | rsa |
|---------|--------|--------|------|------|-----|
| poll函数空指针解引用 | ✓ | ✓ | ✓ | ✓ | ✓ |
| do_xxx resp未检查NULL | - | ✓ | ✓ | ✓ | ✓ |
| set_xxx参数未检查NULL | - | - | - | - | ✓ |
| cookie未检查NULL | ✓ | ✓ | ✓ | ✓ | ✓ |
| del_ctx重复删除资源泄漏 | ✓ | ✓ | ✓ | ✓ | ✓ |
| del_ctx br检查冗余 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 整数截断风险 | - | - | - | - | ✓ |

### 建议优先修复顺序
1. wcrypto_rsa_poll空指针检查（HIGH）
2. wcrypto_do_rsa resp检查（HIGH）
3. wcrypto_set_rsa_kg_out_xxx参数检查（MEDIUM）
4. 整数截断检查（MEDIUM）