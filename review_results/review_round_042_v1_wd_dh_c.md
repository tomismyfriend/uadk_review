# UADK代码检视报告 - 第42轮

## 检视信息
- **检视文件**: v1/wd_dh.c
- **检视时间**: 2026-04-08
- **文件行数**: 480行
- **检视规则**: cpp-memory-safety, cpp-resource-management, cpp-input-validation

## 检视摘要
| 等级 | 数量 |
|------|------|
| HIGH | 2 |
| MEDIUM | 3 |
| LOW | 2 |
| INFO | 0 |
| POSITIVE | 1 |

---

## 问题详情

### 1. [空指针解引用风险] - wcrypto_dh_poll函数 - [Severity: HIGH]

**Location**: `v1/wd_dh.c:435-438`

**Dangerous Code**:
```c
tag = (void *)(uintptr_t)resp->usr_data;
ctx = tag->ctx;
ctx->setup.cb(resp, tag->tag);
wd_put_cookies(&ctx->pool, (void **)&tag, 1);
```

**Risk Analysis**:
v1版本所有模块（cipher/digest/aead/comp/rsa/dh）都有此问题。

**Fix Suggestion**:
添加tag、ctx、cb的NULL检查。

---

### 2. [resp未检查NULL] - wcrypto_do_dh函数 - [Severity: HIGH]

**Location**: `v1/wd_dh.c:379, 401-404`

**Dangerous Code**:
```c
resp = (void *)(uintptr_t)ctxt->ctx_id;
do {
    ret = wd_recv(ctxt->q, (void **)&resp);
    ...
} while (true);

balance = rx_cnt;
opdata->pri = (void *)resp->out;  // resp未检查NULL
opdata->pri_bytes = resp->out_bytes;
opdata->status = resp->result;
```

**Fix Suggestion**:
在循环后添加resp的NULL检查。

---

### 3. [整数截断风险] - dh_request_init函数 - [Severity: MEDIUM]

**Location**: `v1/wd_dh.c:272-273`

**Dangerous Code**:
```c
req->xbytes = (__u16)op->xbytes;
req->pbytes = (__u16)op->pbytes;
```

**Risk Analysis**:
强制转换为__u16可能导致数据截断。

**Fix Suggestion**:
```c
if (op->xbytes > UINT16_MAX || op->pbytes > UINT16_MAX) {
    WD_ERR("xbytes or pbytes overflow!\n");
    return -WD_EINVAL;
}
```

---

### 4. [cookie未检查NULL] - do_dh_prepare函数 - [Severity: MEDIUM]

**Location**: `v1/wd_dh.c:314, 318-321`

**Dangerous Code**:
```c
ret = wd_get_cookies(&ctxt->pool, (void **)&cookie, 1);
if (ret)
    return ret;

if (tag)
    cookie->tag.tag = tag;  // cookie未检查NULL
```

---

### 5. [重复删除资源泄漏] - wcrypto_del_dh_ctx函数 - [Severity: MEDIUM]

**Location**: `v1/wd_dh.c:462-466`

**Dangerous Code**:
```c
if (qinfo->ctx_num <= 0) {
    wd_unspinlock(&qinfo->qlock);
    WD_ERR("error: repeat del dh ctx!\n");
    return;  // ctx内存未释放
}
```

---

### 6. [ctx->q未检查NULL] - wcrypto_init_dh_cookie函数 - [Severity: LOW]

**Location**: `v1/wd_dh.c:94`

**Dangerous Code**:
```c
__u32 flags = ctx->q->capa.flags;  // ctx->q未检查NULL
```

---

### 7. [g->data检查后未检查g->dsize] - wcrypto_set_dh_g函数 - [Severity: LOW]

**Location**: `v1/wd_dh.c:246`

**Dangerous Code**:
```c
if (g->dsize && g->data && g->dsize <= cx->g.bsize) {
    memset(cx->g.data, 0, cx->g.bsize);
    memcpy(cx->g.data, g->data, g->dsize);  // 如果g->dsize为0但g->data非NULL，memcpy不执行
    cx->g.dsize = g->dsize;
```

**Analysis**: 此函数检查较完整，是良好实践。

---

### 8. [正面发现] - DH密钥大小验证 - [Severity: POSITIVE]

**Location**: `v1/wd_dh.c:76-87`

**Code**:
```c
switch (setup->key_bits) {
case DH_KEYSIZE_768:
case DH_KEYSIZE_1024:
case DH_KEYSIZE_1536:
case DH_KEYSIZE_2048:
case DH_KEYSIZE_3072:
case DH_KEYSIZE_4096:
    return WD_SUCCESS;
default:
    WD_ERR("invalid: dh key_bits %u is error!\n", setup->key_bits);
    return -WD_EINVAL;
}
```

**Analysis**: 完整的DH密钥长度验证。

---

## 总结

### 问题分布
| 问题类型 | 数量 |
|----------|------|
| 空指针检查缺失 | 3 |
| 整数截断 | 1 |
| 资源管理 | 1 |
| 代码质量 | 2 |

### v1版本问题模式汇总表（更新）

| 问题模式 | cipher | digest | aead | comp | rsa | dh |
|---------|--------|--------|------|------|-----|-----|
| poll函数空指针解引用 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| do_xxx resp未检查NULL | - | ✓ | ✓ | ✓ | ✓ | ✓ |
| 整数截断风险 | - | - | - | - | ✓ | ✓ |
| cookie未检查NULL | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| del_ctx重复删除资源泄漏 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |