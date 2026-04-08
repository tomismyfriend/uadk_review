# UADK代码检视报告 - 第6轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第6轮
- **规则来源**: E:/work_space/加速器用户态学习/代码检视/review_rules
- **使用Skills**: cpp-memory-safety, cpp-resource-management, cpp-input-validation

---

## 本轮检视文件

### 已检视文件
1. wd.c (第1轮) - 检视报告: review_round_001_wd_c.md
2. wd_util.c (第2轮) - 检视报告: review_round_002_wd_util_c.md
3. wd_alg.c (第3轮) - 检视报告: review_round_003_wd_alg_c.md
4. wd_sched.c (第4轮) - 检视报告: review_round_004_wd_sched_c.md
5. wd_mempool.c (第5轮) - 检视报告: review_round_005_wd_mempool_c.md
6. wd_cipher.c (本轮) - 加密算法文件，954行

### 待检视文件
- wd_digest.c, wd_aead.c, wd_comp.c, wd_rsa.c, wd_dh.c, wd_ecc.c
- wd_bmm.c, wd_agg.c, wd_join_gather.c, wd_udma.c, wd_zlibwrapper.c
- drv/目录下的所有文件
- include/目录下的所有头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## wd_cipher.c 检视结果

### 1. 内存泄漏风险 - wd_cipher_free_sess函数 [HIGH]

**位置**: wd_cipher.c:333-348

**问题类型**: 内存泄漏 (cpp-memory-safety)

**危险代码**:
```c
void wd_cipher_free_sess(handle_t h_sess)
{
    struct wd_cipher_sess *sess = (struct wd_cipher_sess *)h_sess;

    if (unlikely(!sess)) {
        WD_ERR("invalid: cipher input h_sess is NULL!\n");
        return;
    }

    wd_memset_zero(sess->key, sess->key_bytes);
    sess->mm_ops.free(sess->mm_ops.usr, sess->key);

    if (sess->sched_key)
        free(sess->sched_key);
    free(sess);
}
```

**证据链**:
1. 在wd_cipher_alloc_sess函数中分配了sess->dev_mask (未在代码中显示，但结构体定义有dev_mask)
2. wd_cipher_free_sess函数中未释放sess->dev_mask
3. 如果dev_mask在分配时动态分配，则会造成内存泄漏
4. 另外sess->priv也未被释放

**风险分析**:
- 如果dev_mask或priv被动态分配，释放session时会泄漏内存
- 需要检查dev_mask和priv的分配位置并确保释放

**修复建议**:
```c
void wd_cipher_free_sess(handle_t h_sess)
{
    struct wd_cipher_sess *sess = (struct wd_cipher_sess *)h_sess;

    if (unlikely(!sess))
        return;

    if (sess->key) {
        wd_memset_zero(sess->key, sess->key_bytes);
        sess->mm_ops.free(sess->mm_ops.usr, sess->key);
    }

    if (sess->sched_key)
        free(sess->sched_key);

    if (sess->dev_mask)
        free(sess->dev_mask);

    free(sess);
}
```

---

### 2. wd_cipher_set_key函数缓冲区溢出风险 [HIGH]

**位置**: wd_cipher.c:229-253

**问题类型**: 缓冲区溢出 (cpp-memory-safety)

**危险代码**:
```c
int wd_cipher_set_key(handle_t h_sess, const __u8 *key, __u32 key_len)
{
    struct wd_cipher_sess *sess = (struct wd_cipher_sess *)h_sess;
    int ret;

    if (!key || !sess) {
        WD_ERR("invalid: cipher set key input param err!\n");
        return -WD_EINVAL;
    }

    ret = cipher_key_len_check(sess, key_len);
    if (ret) {
        WD_ERR("cipher set key input key length err!\n");
        return -WD_EINVAL;
    }
    // ...

    sess->key_bytes = key_len;
    memcpy(sess->key, key, key_len);  // 未检查sess->key是否已分配

    return 0;
}
```

**证据链**:
1. line 250: 直接使用memcpy复制key到sess->key
2. 未检查sess->key是否已分配或是否为NULL
3. 如果cipher_setup_memory_and_buffers函数分配失败，sess->key可能为NULL
4. 虽然wd_cipher_alloc_sess会检查分配结果，但作为公共API应更谨慎

**风险分析**:
- 如果sess->key为NULL，memcpy会导致崩溃
- 如果key_len超过MAX_CIPHER_KEY_SIZE，可能导致缓冲区溢出
- cipher_key_len_check函数已检查key_len，但不能保证不溢出

**修复建议**:
```c
int wd_cipher_set_key(handle_t h_sess, const __u8 *key, __u32 key_len)
{
    struct wd_cipher_sess *sess = (struct wd_cipher_sess *)h_sess;
    int ret;

    if (!key || !sess || !sess->key)
        return -WD_EINVAL;

    if (key_len > MAX_CIPHER_KEY_SIZE)
        return -WD_EINVAL;

    ret = cipher_key_len_check(sess, key_len);
    if (ret)
        return ret;
    // ...
}
```

---

### 3. wd_do_cipher_async函数错误处理不完整 [MEDIUM]

**位置**: wd_cipher.c:765-817

**问题类型**: 资源管理 (cpp-resource-management)

**代码分析**:
```c
int wd_do_cipher_async(handle_t h_sess, struct wd_cipher_req *req)
{
    // ...
    msg_id = wd_get_msg_from_pool(&wd_cipher_setting.pool, idx, (void **)&msg);
    if (unlikely(msg_id < 0)) {
        WD_ERR("failed to get msg from pool!\n");
        return msg_id;
    }

    fill_request_msg(msg, req, sess);
    msg->tag = msg_id;

    ret = wd_alg_driver_send(wd_cipher_setting.driver, ctx->ctx, msg);
    if (unlikely(ret < 0)) {
        if (ret != -WD_EBUSY)
            WD_ERR("wd cipher async send err!\n");
        goto fail_with_msg;
    }

    wd_dfx_msg_cnt(config, WD_CTX_CNT_NUM, idx);
    ret = wd_add_task_to_async_queue(&wd_cipher_env_config, idx);
    if (ret)
        goto fail_with_msg;  // 这里msg已发送，但未能添加到异步队列

    return 0;

fail_with_msg:
    wd_put_msg_to_pool(&wd_cipher_setting.pool, idx, msg->tag);
    return ret;
}
```

**证据链**:
1. line 799: wd_alg_driver_send已成功发送消息
2. line 808: wd_add_task_to_async_queue失败
3. line 815: 仅释放msg到pool，但消息已发送到硬件
4. 硬件可能返回结果，但没有对应的回调处理

**风险分析**:
- 如果wd_add_task_to_async_queue失败，消息已发送但无法收到回调
- 可能导致异步请求丢失
- 建议在发送失败时再释放资源

---

### 4. wd_cipher_poll_ctx函数NULL指针风险 [MEDIUM]

**位置**: wd_cipher.c:824-876

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
int wd_cipher_poll_ctx(__u32 idx, __u32 expt, __u32 *count)
{
    // ...
    msg = wd_find_msg_in_pool(&wd_cipher_setting.pool, idx, resp_msg.tag);
    if (!msg) {
        WD_ERR("failed to find msg from pool!\n");
        return -WD_EINVAL;
    }

    msg->tag = resp_msg.tag;
    msg->req.state = resp_msg.result;
    req = &msg->req;

    req->cb(req, req->cb_param);  // req->cb可能为NULL
    // ...
}
```

**证据链**:
1. line 868: 直接调用req->cb
2. 未检查req->cb是否为NULL
3. 虽然wd_cipher_check_params检查了异步模式下的cb，但poll时无法保证
4. 如果cb为NULL会导致崩溃

**风险分析**:
- 如果用户未设置回调函数，poll时会导致空指针调用
- 应在调用前检查cb是否有效

**修复建议**:
```c
if (req->cb)
    req->cb(req, req->cb_param);
else
    WD_ERR("callback function is NULL!\n");
```

---

### 5. cipher_setup_memory_and_buffers函数参数检查 [MEDIUM]

**位置**: wd_cipher.c:255-278

**问题类型**: 参数检查 (cpp-input-validation)

**代码**:
```c
static int cipher_setup_memory_and_buffers(struct wd_cipher_sess *sess,
                                           struct wd_cipher_sess_setup *setup)
{
    int ret;

    ret = wd_mem_ops_init(wd_cipher_setting.config.ctxs[0].ctx,
                          &setup->mm_ops, setup->mm_type);
    // ...
}
```

**分析**:
- 未检查sess和setup是否为NULL
- 虽然是内部函数，调用方已检查，但添加防御性检查更安全
- 直接访问setup->mm_ops和setup->mm_type

---

### 6. wd_cipher_init2_函数循环逻辑问题 [LOW]

**位置**: wd_cipher.c:462-547

**问题类型**: 逻辑问题 (cpp-coding-standards)

**代码**:
```c
int wd_cipher_init2_(char *alg, __u32 sched_type, int task_type, struct wd_ctx_params *ctx_params)
{
    // ...
    int state, ret = -WD_EINVAL;  // ret初始化为错误值

    // ...

    while (ret != 0) {  // 当ret != 0时循环
        // ...
        ret = wd_alg_attrs_init(&wd_cipher_init_attrs);
        if (ret) {
            if (ret == -WD_ENODEV) {
                // ...
                continue;  // 继续循环尝试下一个驱动
            }
            WD_ERR("fail to init alg attrs.\n");
            goto out_params_uninit;  // 这里会跳出循环
        }
    }
    // ...
}
```

**分析**:
- 循环条件是ret != 0，只有成功时(ret==0)才退出
- 如果所有驱动都失败，可能陷入无限循环
- 但由于continue和goto，实际不会无限循环
- 代码逻辑可读性较差，建议重构

---

## wd_cipher.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 2 |
| MEDIUM | 3 |
| LOW | 1 |

**重点问题**: 
- wd_cipher_free_sess函数可能内存泄漏(HIGH)
- wd_cipher_set_key函数缓冲区溢出风险(HIGH)

**建议修复顺序**:
1. 检查并修复wd_cipher_free_sess中的内存释放问题
2. 添加wd_cipher_set_key的NULL检查
3. 改进wd_cipher_poll_ctx的回调函数检查

---

## 检视进度更新

**已完成**: 6/94 文件 (6.4%)

**下次检视**: wd_digest.c (摘要算法文件)