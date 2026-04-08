# UADK代码检视报告 - 第7轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第7轮
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
6. wd_cipher.c (第6轮) - 检视报告: review_round_006_wd_cipher_c.md
7. wd_digest.c (本轮) - 摘要算法文件，909行

### 待检视文件
- wd_aead.c, wd_comp.c, wd_rsa.c, wd_dh.c, wd_ecc.c
- wd_bmm.c, wd_agg.c, wd_join_gather.c, wd_udma.c, wd_zlibwrapper.c
- drv/目录下的所有文件
- include/目录下的所有头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## wd_digest.c 检视结果

### 1. 数组越界风险 - g_digest_mac_full_len访问 [HIGH]

**位置**: wd_digest.c:25-30, 540

**问题类型**: 数组越界 (cpp-memory-safety)

**危险代码**:
```c
// 数组定义
static __u32 g_digest_mac_full_len[WD_DIGEST_TYPE_MAX] = {
    WD_DIGEST_SM3_FULL_LEN, WD_DIGEST_MD5_LEN, WD_DIGEST_SHA1_FULL_LEN,
    WD_DIGEST_SHA256_FULL_LEN, WD_DIGEST_SHA224_FULL_LEN,
    WD_DIGEST_SHA384_FULL_LEN, WD_DIGEST_SHA512_FULL_LEN,
    WD_DIGEST_SHA512_224_FULL_LEN, WD_DIGEST_SHA512_256_FULL_LEN
};  // 只有9个元素

// 使用位置
static int wd_mac_length_check(struct wd_digest_sess *sess,
                               struct wd_digest_req *req)
{
    // ...
    if (unlikely(req->out_bytes != g_digest_mac_full_len[sess->alg])) {
        // sess->alg可能是WD_DIGEST_AES_XCBC_MAC_96等，超出数组范围
```

**证据链**:
1. g_digest_mac_full_len数组只有9个元素（对应SM3到SHA512-256）
2. WD_DIGEST_TYPE_MAX包含13种算法类型（包括AES_XCBC_MAC、CMAC、GMAC等）
3. line 540使用sess->alg作为索引访问该数组
4. 如果sess->alg >= 9，会导致数组越界读取

**风险分析**:
- 当算法类型为AES_XCBC_MAC_96、AES_XCBC_PRF_128、AES_CMAC、AES_GMAC时
- 访问g_digest_mac_full_len会越界，读取未定义的内存值
- 可能导致错误的验证逻辑或程序崩溃

**修复建议**:
```c
static int wd_mac_length_check(struct wd_digest_sess *sess,
                               struct wd_digest_req *req)
{
    // 只有前9种算法有full_len
    if (sess->alg >= ARRAY_SIZE(g_digest_mac_full_len)) {
        // 对于AES_XCBC_MAC等算法，使用g_digest_mac_len
        if (req->out_bytes != g_digest_mac_len[sess->alg])
            return -WD_EINVAL;
        return 0;
    }

    if (req->out_bytes != g_digest_mac_full_len[sess->alg])
        return -WD_EINVAL;
    // ...
}
```

---

### 2. wd_digest_set_key函数缓冲区溢出风险 [HIGH]

**位置**: wd_digest.c:159-190

**问题类型**: 缓冲区溢出 (cpp-memory-safety)

**危险代码**:
```c
int wd_digest_set_key(handle_t h_sess, const __u8 *key, __u32 key_len)
{
    struct wd_digest_sess *sess = (struct wd_digest_sess *)h_sess;
    int ret;

    if (!sess || (key_len && !key)) {
        WD_ERR("invalid: digest session or key is NULL!\n");
        return -WD_EINVAL;
    }

    // ... key_len检查 ...

    sess->key_bytes = key_len;
    if (key_len)
        memcpy(sess->key, key, key_len);  // 未检查sess->key是否有效

    return 0;
}
```

**证据链**:
1. line 187: 直接使用memcpy复制key到sess->key
2. 未检查sess->key是否已分配或是否为NULL
3. 如果digest_setup_memory_and_buffers分配失败，sess->key可能为NULL
4. 与wd_cipher.c中的问题类似

**风险分析**:
- 如果sess->key为NULL，memcpy会导致崩溃
- key_len检查已存在，但sess->key的有效性未检查

**修复建议**:
```c
int wd_digest_set_key(handle_t h_sess, const __u8 *key, __u32 key_len)
{
    struct wd_digest_sess *sess = (struct wd_digest_sess *)h_sess;

    if (!sess || !sess->key)
        return -WD_EINVAL;

    if (key_len && !key)
        return -WD_EINVAL;
    // ...
}
```

---

### 3. wd_digest_poll_ctx函数回调检查不完整 [MEDIUM]

**位置**: wd_digest.c:820-823

**问题类型**: 空指针检查 (cpp-input-validation)

**危险代码**:
```c
msg->req.state = recv_msg.result;
req = &msg->req;
if (likely(req))  // 这个检查无意义，req不可能为NULL
    req->cb(req);  // cb可能为NULL
```

**证据链**:
1. req = &msg->req，这是结构体成员地址，不可能为NULL
2. if (likely(req))检查无意义
3. 直接调用req->cb(req)，未检查cb是否为NULL
4. 与wd_cipher.c中类似问题

**风险分析**:
- 如果用户未设置回调函数，会导致空指针调用崩溃
- 应检查req->cb是否有效

**修复建议**:
```c
msg->req.state = recv_msg.result;
req = &msg->req;
if (req->cb)
    req->cb(req);
else
    WD_ERR("callback function is NULL!\n");
```

---

### 4. wd_digest_free_sess函数检查不完整 [MEDIUM]

**位置**: wd_digest.c:267-281

**问题类型**: 资源管理 (cpp-resource-management)

**代码分析**:
```c
void wd_digest_free_sess(handle_t h_sess)
{
    struct wd_digest_sess *sess = (struct wd_digest_sess *)h_sess;

    if (unlikely(!sess)) {
        WD_ERR("failed to check free sess param!\n");
        return;
    }

    wd_memset_zero(sess->key, sess->key_bytes);
    sess->mm_ops.free(sess->mm_ops.usr, sess->key);  // 未检查mm_ops.free
    if (sess->sched_key)
        free(sess->sched_key);
    free(sess);
}
```

**分析**:
- 未检查sess->key是否为NULL就调用wd_memset_zero
- 未检查sess->mm_ops.free是否有效
- 如果key分配失败后调用free_sess，可能导致问题

**修复建议**:
```c
void wd_digest_free_sess(handle_t h_sess)
{
    struct wd_digest_sess *sess = (struct wd_digest_sess *)h_sess;

    if (!sess)
        return;

    if (sess->key && sess->mm_ops.free) {
        wd_memset_zero(sess->key, sess->key_bytes);
        sess->mm_ops.free(sess->mm_ops.usr, sess->key);
    }

    if (sess->sched_key)
        free(sess->sched_key);
    free(sess);
}
```

---

### 5. wd_do_digest_async函数错误处理 [MEDIUM]

**位置**: wd_digest.c:753-770

**问题类型**: 资源管理 (cpp-resource-management)

**代码**:
```c
ret = wd_alg_driver_send(wd_digest_setting.driver, ctx->ctx, msg);
if (unlikely(ret < 0)) {
    if (ret != -WD_EBUSY)
        WD_ERR("failed to send BD, hw is err!\n");

    goto fail_with_msg;
}

wd_dfx_msg_cnt(config, WD_CTX_CNT_NUM, idx);
ret = wd_add_task_to_async_queue(&wd_digest_env_config, idx);
if (ret)
    goto fail_with_msg;  // 消息已发送，但释放了msg
```

**分析**:
- 与wd_cipher.c相同的问题
- 如果wd_add_task_to_async_queue失败，消息已发送但被释放
- 可能导致异步请求丢失或回调无法执行

---

### 6. wd_digest_init2_函数循环逻辑 [LOW]

**位置**: wd_digest.c:430-469

**问题类型**: 代码逻辑 (cpp-coding-standards)

**代码**:
```c
while (ret != 0) {
    // ... 初始化操作 ...
    ret = wd_alg_attrs_init(&wd_digest_init_attrs);
    if (ret) {
        if (ret == -WD_ENODEV) {
            // 继续尝试下一个驱动
            continue;
        }
        WD_ERR("failed to init alg attrs.\n");
        goto out_params_uninit;  // 其他错误直接退出
    }
}
```

**分析**:
- 与wd_cipher.c相同的设计模式
- 循环可能因continue无法退出，但实际通过goto退出
- 代码可读性较差

---

## wd_digest.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 2 |
| MEDIUM | 3 |
| LOW | 1 |

**重点问题**: 
- g_digest_mac_full_len数组越界访问(HIGH)
- wd_digest_set_key函数缓冲区溢出风险(HIGH)

**建议修复顺序**:
1. 修复g_digest_mac_full_len数组越界问题
2. 添加wd_digest_set_key的NULL检查
3. 改进wd_digest_poll_ctx的回调函数检查

---

## 检视进度更新

**已完成**: 7/94 文件 (7.4%)

**下次检视**: wd_aead.c (认证加密算法文件)