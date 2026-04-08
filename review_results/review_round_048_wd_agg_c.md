# UADK代码检视报告 - 第48轮

## 检视信息
- **检视文件**: wd_agg.c (根目录)
- **检视时间**: 2026-04-08
- **文件行数**: 1587行
- **检视规则**: cpp-memory-safety, cpp-resource-management, cpp-input-validation

## 检视摘要
| 等级 | 数量 |
|------|------|
| HIGH | 2 |
| MEDIUM | 3 |
| LOW | 2 |
| INFO | 2 |
| POSITIVE | 5 |

---

## 问题详情

### 1. [req->cb未检查NULL] - wd_agg_poll_ctx函数 - [Severity: HIGH]

**Location**: `wd_agg.c:1564-1566`

**Dangerous Code**:
```c
msg->req.output_done = resp_msg.output_done;
req = &msg->req;

req->cb(req, req->cb_param);  // req->cb未检查NULL
/* Free msg cache to msg_pool */
wd_put_msg_to_pool(&wd_agg_setting.pool, idx, resp_msg.tag);
```

**Risk Analysis**:
与其他模块（wd_rsa.c, wd_dh.c, wd_ecc.c）的poll_ctx函数相同的问题模式。异步回调直接调用，如果req->cb为NULL会导致崩溃。

**Fix Suggestion**:
```c
req = &msg->req;
if (!req->cb) {
    WD_ERR("agg req callback is NULL!\n");
    wd_put_msg_to_pool(&wd_agg_setting.pool, idx, resp_msg.tag);
    return -WD_EINVAL;
}
req->cb(req, req->cb_param);
```

---

### 2. [driver->send/recv未检查NULL] - wd_agg_sync_job函数 - [Severity: HIGH]

**Location**: `wd_agg.c:1100-1101`

**Dangerous Code**:
```c
msg_handle.send = wd_agg_setting.driver->send;
msg_handle.recv = wd_agg_setting.driver->recv;

pthread_spin_lock(&ctx->lock);
ret = wd_handle_msg_sync(wd_agg_setting.driver, &msg_handle, ctx->ctx,
             msg, NULL, config->epoll_en);
```

**Risk Analysis**:
driver->send和driver->recv可能为NULL，直接调用会崩溃。

**Fix Suggestion**:
```c
if (!wd_agg_setting.driver ||
    !wd_agg_setting.driver->send ||
    !wd_agg_setting.driver->recv) {
    WD_ERR("agg driver or driver ops is NULL!\n");
    return -WD_EINVAL;
}
msg_handle.send = wd_agg_setting.driver->send;
msg_handle.recv = wd_agg_setting.driver->recv;
```

---

### 3. [sched_init未检查NULL] - wd_agg_alloc_sess函数 - [Severity: MEDIUM]

**Location**: `wd_agg.c:411-412`

**Dangerous Code**:
```c
sess->sched_key = (void *)wd_agg_setting.sched.sched_init(
    wd_agg_setting.sched.h_sched_ctx, setup->sched_param);
```

**Risk Analysis**:
sched_init可能为NULL，直接调用会崩溃。

---

### 4. [wd_agg_setting.driver未检查NULL] - wd_agg_async_job函数 - [Severity: MEDIUM]

**Location**: `wd_agg.c:1206`

**Dangerous Code**:
```c
ret = wd_alg_driver_send(wd_agg_setting.driver, ctx->ctx, msg);
```

**Risk Analysis**:
wd_agg_setting.driver可能为NULL。

---

### 5. [ops回调函数未检查driver] - wd_agg_init_sess_priv函数 - [Severity: MEDIUM]

**Location**: `wd_agg.c:364, 372`

**Dangerous Code**:
```c
if (sess->ops.sess_init) {
    if (!sess->ops.sess_uninit) {
        WD_ERR("failed to get session uninit ops!\n");
        return -WD_EINVAL;
    }
    ret = sess->ops.sess_init(wd_agg_setting.driver, setup, &sess->priv);
    ...
}

if (sess->ops.get_row_size) {
    ret = sess->ops.get_row_size(wd_agg_setting.driver, sess->priv);
```

**Risk Analysis**:
虽然检查了ops函数是否存在，但未检查wd_agg_setting.driver是否为NULL。

---

### 6. [整数除法边界] - wd_agg_rehash_sync函数 - [Severity: LOW]

**Location**: `wd_agg.c:1500`

**Code**:
```c
max_cnt = MAX_HASH_TABLE_ROW_NUM / req->out_row_count;
```

**Risk Analysis**:
如果req->out_row_count为0会导致除零错误。但前面的wd_agg_check_rehash_params函数会检查，所以这里是安全的。建议添加注释说明依赖关系。

---

### 7. [wd_agg_setting.driver未检查NULL] - 多个函数 - [Severity: LOW]

**Location**: `wd_agg.c:404, 418`

**Code**:
```c
ret = wd_drv_alg_support(sess->alg_name, wd_agg_setting.driver);
...
if (wd_agg_setting.driver->get_extend_ops) {
```

**Risk Analysis**:
wd_agg_setting.driver可能在初始化失败时为NULL。

---

### 8. [信息性发现] - 会话状态机 - [Severity: INFO]

**Location**: `wd_agg.c:21-28`

**Code**:
```c
enum wd_agg_sess_state {
    WD_AGG_SESS_UNINIT, /* Uninit session */
    WD_AGG_SESS_INIT, /* Hash table has been set */
    WD_AGG_SESS_INPUT, /* Input stage has started */
    WD_AGG_SESS_RESET, /* Hash table has been reset */
    WD_AGG_SESS_REHASH, /* Rehash stage has started */
    WD_AGG_SESS_OUTPUT, /* Output stage has started */
};
```

**Analysis**:
使用原子操作实现状态机转换：
- wd_agg_check_sess_state: 状态检查和转换
- wd_agg_input_try_init: 初始化输入状态
- wd_agg_output_try_init: 初始化输出状态
- wd_agg_rehash_try_init: 初始化rehash状态

这是良好的并发控制设计。

---

### 9. [信息性发现] - Hash Aggregation操作 - [Severity: INFO]

**Location**: `wd_agg.c:71`

**Code**:
```c
static const char *wd_agg_alg_name = "hashagg";
```

**Analysis**:
文件实现了数据库风格的哈希聚合操作：
1. 设置哈希表 (wd_agg_set_hash_table)
2. 添加输入数据 (wd_agg_add_input_sync/async)
3. 获取输出结果 (wd_agg_get_output_sync/async)
4. 重新哈希 (wd_agg_rehash_sync)

支持同步和异步两种操作模式。

---

### 10. [正面发现] - 完整的参数验证 - [Severity: POSITIVE]

**Location**: `wd_agg.c:192-297`

**Code**:
```c
static int wd_agg_check_sess_params(struct wd_agg_sess_setup *setup, __u32 *out_agg_cols_num)
{
    if (!setup) {
        WD_ERR("invalid: agg sess setup is NULL!\n");
        return -WD_EINVAL;
    }

    if (!setup->key_cols_num || !setup->key_cols_info) {
        WD_ERR("invalid: agg key cols is NULL, num: %u\n", setup->key_cols_num);
        return -WD_EINVAL;
    }
    ...
}
```

**Analysis**:
完整的会话参数验证，包括：
- check_key_cols_info: 检查键列信息
- check_agg_cols_info: 检查聚合列信息
- wd_agg_check_input_req: 检查输入请求
- wd_agg_check_output_req: 检查输出请求

---

### 11. [正面发现] - wd_agg_check_common_params回调检查 - [Severity: POSITIVE]

**Location**: `wd_agg.c:794-797`

**Code**:
```c
if (unlikely(mode == CTX_MODE_ASYNC && !req->cb)) {
    WD_ERR("invalid: agg req cb is NULL!\n");
    return -WD_EINVAL;
}
```

**Analysis**:
异步模式下正确检查req->cb是否为NULL。这与poll_ctx中的问题形成对比——异步入口检查了，但poll回调时没检查。

---

### 12. [正面发现] - 列地址完整性检查 - [Severity: POSITIVE]

**Location**: `wd_agg.c:802-870`

**Code**:
```c
static int check_in_col_addr(struct wd_dae_col_addr *col, __u32 row_count,
                 enum wd_dae_data_type type, __u64 data_size)
{
    if (unlikely(!col->empty || col->empty_size != row_count * sizeof(col->empty[0]))) {
        ...
    }
    ...
}
```

**Analysis**:
详细的列地址验证，检查empty、value、offset数组的边界。

---

### 13. [正面发现] - 整数溢出保护 - [Severity: POSITIVE]

**Location**: `wd_agg.c:306-307, 318-320`

**Code**:
```c
key_size = setup->key_cols_num * sizeof(struct wd_key_col_info);
agg_size = setup->agg_cols_num * sizeof(struct wd_agg_col_info);
```

**Analysis**:
使用size_t类型进行大小计算，避免了整数溢出问题。

---

### 14. [正面发现] - 错误处理路径完整 - [Severity: POSITIVE]

**Location**: `wd_agg.c:438-445`

**Code**:
```c
uninit_priv:
    if (sess->ops.sess_uninit)
        sess->ops.sess_uninit(wd_agg_setting.driver, sess->priv);
free_key:
    free(sess->sched_key);
free_sess:
    free(sess);
    return (handle_t)0;
```

**Analysis**:
错误处理路径正确释放资源。

---

## 总结

### 问题分布
| 问题类型 | 数量 |
|----------|------|
| 空指针检查缺失 | 5 |
| 整数运算 | 1 |

### 问题模式分析

wd_agg.c与之前检视的算法模块（wd_rsa.c, wd_dh.c, wd_ecc.c）存在相同的问题模式：

| 问题模式 | wd_agg.c | wd_rsa.c | wd_dh.c | wd_ecc.c |
|---------|----------|----------|---------|----------|
| poll_ctx req->cb未检查NULL | ✓ | ✓ | ✓ | ✓ |
| driver->send/recv未检查NULL | ✓ | ✓ | ✓ | ✓ |
| sched_init未检查NULL | ✓ | ✓ | ✓ | ✓ |
| 异步入口检查req->cb | ✓ | ✓ | - | ✓ |

这表明这些问题模式是系统性的，需要统一修复。

### 与其他模块对比

wd_agg.c是数据库聚合操作模块，特点：
1. 使用状态机管理会话生命周期
2. 支持多种数据类型（INT, LONG, DECIMAL, VARCHAR等）
3. 支持同步和异步两种模式
4. 参数验证非常完整

### 建议优先修复顺序
1. wd_agg_poll_ctx回调函数检查（HIGH）
2. driver->send/recv NULL检查（HIGH）
3. sched_init NULL检查（MEDIUM）
4. wd_agg_setting.driver NULL检查（MEDIUM）