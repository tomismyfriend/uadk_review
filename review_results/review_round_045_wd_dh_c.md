# UADK代码检视报告 - 第45轮

## 检视信息
- **检视文件**: wd_dh.c (根目录)
- **检视时间**: 2026-04-08
- **文件行数**: 725行
- **检视规则**: cpp-memory-safety, cpp-resource-management, cpp-input-validation

## 检视摘要
| 等级 | 数量 |
|------|------|
| HIGH | 2 |
| MEDIUM | 3 |
| LOW | 2 |
| INFO | 0 |
| POSITIVE | 2 |

---

## 问题详情

### 1. [req->cb未检查NULL] - wd_dh_poll_ctx函数 - [Severity: HIGH]

**Location**: `wd_dh.c:511-512`

**Dangerous Code**:
```c
msg = wd_find_msg_in_pool(&wd_dh_setting.pool, idx, rcv_msg.tag);
if (!msg) {
    WD_ERR("failed to find msg!\n");
    return -WD_EINVAL;
}

msg->req.pri_bytes = rcv_msg.req.pri_bytes;
msg->req.status = rcv_msg.result;
req = &msg->req;
req->cb(req);  // req->cb未检查NULL
```

**Risk Analysis**:
与wd_rsa.c中的相同问题模式。

**Fix Suggestion**:
```c
req = &msg->req;
if (!req->cb) {
    WD_ERR("dh req callback is NULL!\n");
    wd_put_msg_to_pool(&wd_dh_setting.pool, idx, rcv_msg.tag);
    return -WD_EINVAL;
}
req->cb(req);
```

---

### 2. [driver->send/recv未检查NULL] - wd_do_dh_sync函数 - [Severity: HIGH]

**Location**: `wd_dh.c:389-393`

**Dangerous Code**:
```c
msg_handle.send = wd_dh_setting.driver->send;
msg_handle.recv = wd_dh_setting.driver->recv;

pthread_spin_lock(&ctx->lock);
ret = wd_handle_msg_sync(wd_dh_setting.driver, &msg_handle, ctx->ctx,
             &msg, &balance, wd_dh_setting.config.epoll_en);
```

**Risk Analysis**:
与wd_rsa.c相同问题，driver->send/recv可能为NULL。

---

### 3. [mm_ops->alloc未检查NULL] - wd_dh_alloc_sess函数 - [Severity: MEDIUM]

**Location**: `wd_dh.c:622`

**Dangerous Code**:
```c
sess->g.data = sess->mm_ops.alloc(sess->mm_ops.usr, sess->key_size);
```

**Risk Analysis**:
mm_ops->alloc可能为NULL。

---

### 4. [sched->sched_init未检查NULL] - wd_dh_alloc_sess函数 - [Severity: MEDIUM]

**Location**: `wd_dh.c:629-630`

**Dangerous Code**:
```c
sess->sched_key = (void *)wd_dh_setting.sched.sched_init(
         wd_dh_setting.sched.h_sched_ctx, setup->sched_param);
```

**Risk Analysis**:
sched_init可能为NULL。

---

### 5. [mm_ops->free未检查NULL] - wd_dh_free_sess函数 - [Severity: MEDIUM]

**Location**: `wd_dh.c:654-655`

**Dangerous Code**:
```c
if (sess_t->g.data)
    sess_t->mm_ops.free(sess_t->mm_ops.usr, sess_t->g.data);
```

**Risk Analysis**:
mm_ops->free可能为NULL。

**Fix Suggestion**:
```c
if (sess_t->g.data && sess_t->mm_ops.free)
    sess_t->mm_ops.free(sess_t->mm_ops.usr, sess_t->g.data);
```

---

### 6. [sess->g.data未检查NULL] - wd_dh_set_g函数 - [Severity: LOW]

**Location**: `wd_dh.c:564-565`

**Dangerous Code**:
```c
if (g->dsize && g->data && g->dsize <= sess_t->g.bsize) {
    memset(sess_t->g.data, 0, sess_t->g.bsize);  // sess_t->g.data未检查
```

**Risk Analysis**:
虽然sess_t->g.data在alloc_sess中分配，但如果分配失败，sess_t->g.data为NULL。但这里已经检查了g->dsize && g->data，所以只有当sess_t->g.data为NULL且g->dsize为非零时才会有问题。

---

### 7. [wd_dh_setting.driver未检查NULL] - wd_do_dh_async函数 - [Severity: LOW]

**Location**: `wd_dh.c:440`

**Dangerous Code**:
```c
ret = wd_alg_driver_send(wd_dh_setting.driver, ctx->ctx, msg);
```

---

### 8. [正面发现] - 密钥长度验证 - [Severity: POSITIVE]

**Location**: `wd_dh.c:596-604`

**Code**:
```c
if (setup->key_bits != 768 &&
    setup->key_bits != 1024 &&
    setup->key_bits != 1536 &&
    setup->key_bits != 2048 &&
    setup->key_bits != 3072 &&
    setup->key_bits != 4096) {
    WD_ERR("invalid: alloc dh sess key_bit %u is err!\n", setup->key_bits);
    return (handle_t)0;
}
```

**Analysis**: 完整的DH密钥长度验证。

---

### 9. [正面发现] - wd_dh_set_g参数验证 - [Severity: POSITIVE]

**Location**: `wd_dh.c:563-570`

**Code**:
```c
if (g->dsize && g->data && g->dsize <= sess_t->g.bsize) {
    memset(sess_t->g.data, 0, sess_t->g.bsize);
    memcpy(sess_t->g.data, g->data, g->dsize);
    sess_t->g.dsize = g->dsize;
    if (*g->data != WD_DH_G2 && sess_t->setup.is_g2)
        return -WD_EINVAL;
    return WD_SUCCESS;
}

return -WD_EINVAL;
```

**Analysis**: 良好的参数验证和G2模式检查。

---

## 总结

### 问题分布
| 问题类型 | 数量 |
|----------|------|
| 空指针检查缺失 | 5 |
| 安全实践 | 2 |

### 与v1版本对比

| 对比项 | wd_dh.c (根目录) | v1/wd_dh.c |
|--------|-------------------|-------------|
| 文件行数 | 725 | 480 |
| 架构风格 | 新版框架 | 旧版框架 |
| poll函数问题 | req->cb未检查NULL | tag->ctx未检查NULL |
| 整数截断 | 无 | 有(__u16转换) |

**根目录版本相对更完善**：
1. 无整数截断问题
2. 架构更现代化
3. 但driver函数指针未检查NULL的问题仍然存在

### 根目录版本问题模式

与v1版本类似，根目录的wd_rsa.c和wd_dh.c存在以下共同问题：
1. poll_ctx函数req->cb未检查NULL
2. driver->send/recv未检查NULL
3. mm_ops函数指针未检查NULL
4. sched函数指针未检查NULL

### 建议优先修复顺序
1. wd_dh_poll_ctx回调函数检查（HIGH）
2. driver->send/recv NULL检查（HIGH）
3. mm_ops函数指针检查（MEDIUM）