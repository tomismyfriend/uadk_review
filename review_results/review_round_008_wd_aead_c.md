# UADK代码检视报告 - 第8轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第8轮
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
7. wd_digest.c (第7轮) - 检视报告: review_round_007_wd_digest_c.md
8. wd_aead.c (本轮) - 认证加密算法文件，1108行

### 待检视文件
- wd_comp.c, wd_rsa.c, wd_dh.c, wd_ecc.c
- wd_bmm.c, wd_agg.c, wd_join_gather.c, wd_udma.c, wd_zlibwrapper.c
- drv/目录下的所有文件
- include/目录下的所有头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## wd_aead.c 检视结果

### 1. 数组越界风险 - g_aead_mac_len访问 [HIGH]

**位置**: wd_aead.c:16-21, 269

**问题类型**: 数组越界 (cpp-memory-safety)

**危险代码**:
```c
// 数组定义 - 只有9个元素
static int g_aead_mac_len[WD_DIGEST_TYPE_MAX] = {
    WD_DIGEST_SM3_LEN, WD_DIGEST_MD5_LEN, WD_DIGEST_SHA1_LEN,
    WD_DIGEST_SHA256_LEN, WD_DIGEST_SHA224_LEN,
    WD_DIGEST_SHA384_LEN, WD_DIGEST_SHA512_LEN,
    WD_DIGEST_SHA512_224_LEN, WD_DIGEST_SHA512_256_LEN
};

// 使用位置
int wd_aead_set_authsize(handle_t h_sess, __u16 authsize)
{
    // ...
    if (sess->dalg >= WD_DIGEST_TYPE_MAX || !authsize ||
        authsize > g_aead_mac_len[sess->dalg]) {  // 可能越界
```

**证据链**:
1. g_aead_mac_len数组只有9个元素（与wd_digest.c相同的问题）
2. WD_DIGEST_TYPE_MAX包含13种算法类型
3. line 269: 使用sess->dalg作为索引访问数组
4. 如果sess->dalg >= 9，会导致数组越界读取

**风险分析**:
- 与wd_digest.c中的g_digest_mac_full_len问题相同
- 数组元素数量与WD_DIGEST_TYPE_MAX不匹配
- 可能读取未初始化的内存值

**修复建议**:
```c
int wd_aead_set_authsize(handle_t h_sess, __u16 authsize)
{
    // ...
    if (sess->dalg >= WD_DIGEST_TYPE_MAX || !authsize)
        return -WD_EINVAL;

    // 只有前9种算法有mac_len定义
    if (sess->dalg < ARRAY_SIZE(g_aead_mac_len) &&
        authsize > g_aead_mac_len[sess->dalg])
        return -WD_EINVAL;
    // ...
}
```

---

### 2. wd_aead_set_ckey/set_akey函数缓冲区溢出风险 [HIGH]

**位置**: wd_aead.c:189-241

**问题类型**: 缓冲区溢出 (cpp-memory-safety)

**危险代码**:
```c
int wd_aead_set_ckey(handle_t h_sess, const __u8 *key, __u16 key_len)
{
    struct wd_aead_sess *sess = (struct wd_aead_sess *)h_sess;

    if (unlikely(!key || !sess)) {
        WD_ERR("failed to check cipher key input param!\n");
        return -WD_EINVAL;
    }

    // ... key_len检查 ...

    sess->ckey_bytes = key_len;
    memcpy(sess->ckey, key, key_len);  // 未检查sess->ckey是否有效

    return 0;
}

int wd_aead_set_akey(handle_t h_sess, const __u8 *key, __u16 key_len)
{
    // ...
    sess->akey_bytes = key_len;
    if (key_len)
        memcpy(sess->akey, key, key_len);  // 未检查sess->akey是否有效
    // ...
}
```

**证据链**:
1. 直接使用memcpy复制key到sess->ckey/sess->akey
2. 未检查ckey/akey是否已分配或是否为NULL
3. 如果aead_setup_memory_and_buffers分配失败，这些指针可能为NULL
4. 与wd_cipher.c和wd_digest.c中的问题类似

**修复建议**:
```c
int wd_aead_set_ckey(handle_t h_sess, const __u8 *key, __u16 key_len)
{
    struct wd_aead_sess *sess = (struct wd_aead_sess *)h_sess;

    if (!sess || !sess->ckey || !key)
        return -WD_EINVAL;
    // ...
}
```

---

### 3. wd_aead_poll_ctx函数回调检查缺失 [HIGH]

**位置**: wd_aead.c:1020-1023

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
msg->tag = resp_msg.tag;
msg->req.state = resp_msg.result;
req = &msg->req;
req->cb(req, req->cb_param);  // 直接调用cb，未检查是否为NULL
```

**证据链**:
1. line 1023: 直接调用req->cb
2. 未检查req->cb是否为NULL
3. 虽然wd_do_aead_async检查了cb，但poll时无法保证
4. 如果cb为NULL会导致空指针调用崩溃

**修复建议**:
```c
if (req->cb)
    req->cb(req, req->cb_param);
else
    WD_ERR("callback function is NULL!\n");
```

---

### 4. wd_aead_free_sess函数资源释放检查 [MEDIUM]

**位置**: wd_aead.c:493-509

**问题类型**: 资源管理 (cpp-resource-management)

**代码分析**:
```c
void wd_aead_free_sess(handle_t h_sess)
{
    struct wd_aead_sess *sess = (struct wd_aead_sess *)h_sess;

    if (unlikely(!sess)) {
        WD_ERR("invalid: aead input sess is NULL!\n");
        return;
    }

    wd_memset_zero(sess->ckey, sess->ckey_bytes);  // 未检查ckey是否为NULL
    wd_memset_zero(sess->akey, sess->akey_bytes);  // 未检查akey是否为NULL

    if (sess->sched_key)
        free(sess->sched_key);
    wd_aead_sess_eops_uninit(sess);
    cleanup_session(sess);
}
```

**分析**:
- wd_memset_zero在ckey/akey为NULL时可能出问题
- cleanup_session会释放这些指针，但未检查是否已释放
- 建议添加NULL检查

---

### 5. cleanup_session函数检查 [MEDIUM]

**位置**: wd_aead.c:409-418

**问题类型**: 资源管理 (cpp-resource-management)

**代码**:
```c
static void cleanup_session(struct wd_aead_sess *sess)
{
    sess->mm_ops.free(sess->mm_ops.usr, sess->mac_bak);
    sess->mm_ops.free(sess->mm_ops.usr, sess->iv);
    sess->mm_ops.free(sess->mm_ops.usr, sess->ckey);
    sess->mm_ops.free(sess->mm_ops.usr, sess->akey);

    if (sess)  // 这个检查应该放在前面
        free(sess);
}
```

**分析**:
- 先使用了sess，再检查是否为NULL，顺序错误
- 应该先检查sess是否为NULL
- mm_ops.free未检查是否有效

**修复建议**:
```c
static void cleanup_session(struct wd_aead_sess *sess)
{
    if (!sess)
        return;

    if (sess->mm_ops.free) {
        sess->mm_ops.free(sess->mm_ops.usr, sess->mac_bak);
        sess->mm_ops.free(sess->mm_ops.usr, sess->iv);
        sess->mm_ops.free(sess->mm_ops.usr, sess->ckey);
        sess->mm_ops.free(sess->mm_ops.usr, sess->akey);
    }
    free(sess);
}
```

---

### 6. wd_aead_init2_函数循环逻辑 [LOW]

**位置**: wd_aead.c:725-765

**问题类型**: 代码逻辑 (cpp-coding-standards)

**代码**:
```c
while (ret != 0) {
    // ... 初始化操作 ...
}
```

**分析**:
- 与之前文件相同的循环模式
- 循环可能因continue无法退出，但实际通过goto退出
- 代码可读性较差

---

## wd_aead.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 3 |
| MEDIUM | 2 |
| LOW | 1 |

**重点问题**: 
- g_aead_mac_len数组越界访问(HIGH)
- wd_aead_set_ckey/set_akey缓冲区溢出风险(HIGH)
- wd_aead_poll_ctx回调函数未检查(HIGH)

**建议修复顺序**:
1. 修复g_aead_mac_len数组越界问题
2. 添加key设置函数的NULL检查
3. 改进wd_aead_poll_ctx的回调函数检查

---

## 检视进度更新

**已完成**: 8/94 文件 (8.5%)

**下次检视**: wd_comp.c (压缩算法文件)