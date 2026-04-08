# UADK代码检视报告 - 第44轮

## 检视信息
- **检视文件**: wd_rsa.c (根目录)
- **检视时间**: 2026-04-08
- **文件行数**: 1337行
- **检视规则**: cpp-memory-safety, cpp-resource-management, cpp-input-validation

## 检视摘要
| 等级 | 数量 |
|------|------|
| HIGH | 2 |
| MEDIUM | 4 |
| LOW | 2 |
| INFO | 1 |
| POSITIVE | 3 |

---

## 问题详情

### 1. [msg未检查NULL] - wd_rsa_poll_ctx函数 - [Severity: HIGH]

**Location**: `wd_rsa.c:561-571`

**Dangerous Code**:
```c
msg = wd_find_msg_in_pool(&wd_rsa_setting.pool, idx,
              recv_msg.tag);
if (!msg) {
    WD_ERR("failed to find msg from pool!\n");
    return -WD_EINVAL;
}

msg->req.dst_bytes = recv_msg.req.dst_bytes;
msg->req.status = recv_msg.result;
req = &msg->req;
req->cb(req);  // req->cb未检查NULL
```

**Risk Analysis**:
1. msg找到后，直接调用req->cb
2. 如果req->cb为NULL，会导致崩溃
3. 异步回调是关键入口，崩溃影响整个流程

**Fix Suggestion**:
```c
req = &msg->req;
if (!req->cb) {
    WD_ERR("rsa req callback is NULL!\n");
    wd_put_msg_to_pool(&wd_rsa_setting.pool, idx, recv_msg.tag);
    return -WD_EINVAL;
}
req->cb(req);
```

---

### 2. [driver->send/recv未检查NULL] - wd_do_rsa_sync函数 - [Severity: HIGH]

**Location**: `wd_rsa.c:450-454`

**Dangerous Code**:
```c
msg_handle.send = wd_rsa_setting.driver->send;
msg_handle.recv = wd_rsa_setting.driver->recv;

pthread_spin_lock(&ctx->lock);
ret = wd_handle_msg_sync(wd_rsa_setting.driver, &msg_handle, ctx->ctx, &msg,
             &balance, wd_rsa_setting.config.epoll_en);
```

**Risk Analysis**:
1. driver->send和driver->recv可能为NULL
2. wd_handle_msg_sync内部会调用这些函数指针
3. 如果为NULL会导致崩溃

**Fix Suggestion**:
```c
if (!wd_rsa_setting.driver || 
    !wd_rsa_setting.driver->send || 
    !wd_rsa_setting.driver->recv) {
    WD_ERR("rsa driver or driver ops is NULL!\n");
    return -WD_EINVAL;
}
msg_handle.send = wd_rsa_setting.driver->send;
msg_handle.recv = wd_rsa_setting.driver->recv;
```

---

### 3. [sess->prikey/prikey->data未检查NULL] - del_sess_key函数 - [Severity: MEDIUM]

**Location**: `wd_rsa.c:919-929`

**Dangerous Code**:
```c
static void del_sess_key(struct wd_rsa_sess *sess)
{
    struct wd_rsa_prikey *prk = sess->prikey;
    struct wd_rsa_pubkey *pub = sess->pubkey;

    if (!prk || !pub) {
        WD_ERR("invalid: del sess key error, prk or pub NULL!\n");
        return;
    }

    if (sess->setup.is_crt)
        wd_memset_zero(prk->pkey.pkey2.data, CRT_PARAMS_SZ(sess->key_size));
    else
        wd_memset_zero(prk->pkey.pkey1.data, GEN_PARAMS_SZ(sess->key_size));
    ...
}
```

**Risk Analysis**:
1. prk->pkey.pkey2.data或prk->pkey.pkey1.data可能为NULL
2. wd_memset_zero直接使用可能崩溃

**Fix Suggestion**:
```c
if (sess->setup.is_crt) {
    if (prk->pkey.pkey2.data)
        wd_memset_zero(prk->pkey.pkey2.data, CRT_PARAMS_SZ(sess->key_size));
} else {
    if (prk->pkey.pkey1.data)
        wd_memset_zero(prk->pkey.pkey1.data, GEN_PARAMS_SZ(sess->key_size));
}
```

---

### 4. [mm_ops->alloc/free未检查NULL] - wd_rsa_new_kg_in函数 - [Severity: MEDIUM]

**Location**: `wd_rsa.c:648`

**Dangerous Code**:
```c
kg_in = c->mm_ops.alloc(c->mm_ops.usr, kg_in_size + sizeof(*kg_in));
```

**Risk Analysis**:
mm_ops->alloc可能为NULL，直接调用会崩溃。

**Fix Suggestion**:
```c
if (!c->mm_ops.alloc) {
    WD_ERR("mm_ops alloc is NULL!\n");
    return NULL;
}
kg_in = c->mm_ops.alloc(c->mm_ops.usr, kg_in_size + sizeof(*kg_in));
```

---

### 5. [sched->sched_init返回值未检查] - wd_rsa_alloc_sess函数 - [Severity: MEDIUM]

**Location**: `wd_rsa.c:980-985`

**Dangerous Code**:
```c
sess->sched_key = (void *)wd_rsa_setting.sched.sched_init(
         wd_rsa_setting.sched.h_sched_ctx, setup->sched_param);
if (WD_IS_ERR(sess->sched_key)) {
    WD_ERR("failed to init session schedule key!\n");
    goto sched_err;
}
```

**Risk Analysis**:
1. 使用WD_IS_ERR检查返回值是正确的
2. 但sched_init可能为NULL，直接调用会崩溃

**Fix Suggestion**:
```c
if (!wd_rsa_setting.sched.sched_init) {
    WD_ERR("sched_init is NULL!\n");
    goto sess_err;
}
sess->sched_key = (void *)wd_rsa_setting.sched.sched_init(...);
```

---

### 6. [wd_rsa_setting.driver未检查NULL] - wd_do_rsa_async函数 - [Severity: MEDIUM]

**Location**: `wd_rsa.c:501`

**Dangerous Code**:
```c
ret = wd_alg_driver_send(wd_rsa_setting.driver, ctx->ctx, msg);
```

**Risk Analysis**:
wd_rsa_setting.driver可能为NULL。

**Fix Suggestion**:
在函数开始处添加driver检查。

---

### 7. [sess->pubkey未检查NULL] - wd_rsa_get_pubkey函数 - [Severity: LOW]

**Location**: `wd_rsa.c:1261`

**Dangerous Code**:
```c
*pubkey = ((struct wd_rsa_sess *)sess)->pubkey;
```

**Risk Analysis**:
sess->pubkey可能为NULL，但调用者通常会检查返回值。

---

### 8. [sess->prikey未检查NULL] - wd_rsa_get_prikey函数 - [Severity: LOW]

**Location**: `wd_rsa.c:1271`

**Dangerous Code**:
```c
*prikey = ((struct wd_rsa_sess *)sess)->prikey;
```

---

### 9. [信息性发现] - 密钥长度验证 - [Severity: INFO]

**Location**: `wd_rsa.c:949-955`

**Code**:
```c
if (setup->key_bits != 1024 &&
    setup->key_bits != 2048 &&
    setup->key_bits != 3072 &&
    setup->key_bits != 4096) {
    WD_ERR("invalid: alloc rsa sess key_bit %u err!\n", setup->key_bits);
    return (handle_t)0;
}
```

**Analysis**:
良好的密钥长度验证，只支持标准RSA密钥长度。

---

### 10. [正面发现] - 完整的参数验证 - [Severity: POSITIVE]

**Location**: `wd_rsa.c:622-645`

**Code**:
```c
if (!c || !e || !p || !q) {
    WD_ERR("invalid: sess malloc kg_in memory params err!\n");
    return NULL;
}

if (!c->key_size || c->key_size > RSA_MAX_KEY_SIZE) {
    WD_ERR("invalid: key size err at create kg in!\n");
    return NULL;
}

if (!e->dsize || e->dsize > c->key_size || !e->data) {
    WD_ERR("invalid: e para err at create kg in!\n");
    return NULL;
}
...
```

**Analysis**: wd_rsa_new_kg_in函数有完整的参数验证。

---

### 11. [正面发现] - 敏感数据清零 - [Severity: POSITIVE]

**Location**: `wd_rsa.c:752, 925-927`

**Code**:
```c
wd_memset_zero(kout->data, kout->size);  // 清零密钥生成输出
wd_memset_zero(prk->pkey.pkey2.data, CRT_PARAMS_SZ(sess->key_size));  // 清零私钥
```

**Analysis**: 正确的敏感数据清理实践。

---

### 12. [正面发现] - wd_rsa_set_kg_out_psz/crt_psz参数检查 - [Severity: POSITIVE]

**Location**: `wd_rsa.c:810-830`

**Code**:
```c
void wd_rsa_set_kg_out_crt_psz(struct wd_rsa_kg_out *kout, ...)
{
    if (!kout) {
        WD_ERR("invalid: input null when set kg out crt psz!\n");
        return;
    }
    ...
}
```

**Analysis**: 与v1版本不同，根目录版本的这些函数正确检查了kout参数。

---

## 总结

### 问题分布
| 问题类型 | 数量 |
|----------|------|
| 空指针检查缺失 | 6 |
| 安全实践 | 3 |

### 与v1版本对比

| 对比项 | wd_rsa.c (根目录) | v1/wd_rsa.c |
|--------|-------------------|-------------|
| 文件行数 | 1337 | 1144 |
| 架构风格 | 新版框架 | 旧版框架 |
| poll函数问题 | req->cb未检查NULL | tag->ctx未检查NULL |
| 参数验证 | 更完整 | 部分缺失 |
| wd_rsa_set_kg_out_xxx | 有NULL检查 | 无NULL检查 |

**根目录版本相对更完善**：
1. 架构更现代化，使用driver抽象
2. 部分函数有更完整的参数验证
3. 但仍然存在driver函数指针未检查NULL的问题

### 建议优先修复顺序
1. wd_rsa_poll_ctx回调函数检查（HIGH）
2. driver->send/recv NULL检查（HIGH）
3. mm_ops函数指针检查（MEDIUM）
4. del_sess_key数据指针检查（MEDIUM）