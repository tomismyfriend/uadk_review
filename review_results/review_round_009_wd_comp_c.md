# UADK代码检视报告 - 第9轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第9轮
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
8. wd_aead.c (第8轮) - 检视报告: review_round_008_wd_aead_c.md
9. wd_comp.c (本轮) - 压缩算法文件，1066行

### 待检视文件
- wd_rsa.c, wd_dh.c, wd_ecc.c, wd_bmm.c, wd_agg.c, wd_join_gather.c, wd_udma.c, wd_zlibwrapper.c
- drv/目录下的所有文件
- include/目录下的所有头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## wd_comp.c 检视结果

### 1. wd_comp_poll_ctx函数回调检查缺失 [HIGH]

**位置**: wd_comp.c:401-404

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
req = &msg->req;
req->src_len = msg->in_cons;
req->dst_len = msg->produced;
req->cb(req, req->cb_param);  // 直接调用cb，未检查是否为NULL
```

**证据链**:
1. line 404: 直接调用req->cb
2. 未检查req->cb是否为NULL
3. 虽然wd_do_comp_async检查了cb，但poll时无法保证
4. 与之前检视的cipher/digest/aead文件相同的问题

**风险分析**:
- 如果用户未设置回调函数，会导致空指针调用崩溃
- 这是所有poll函数的通用问题

**修复建议**:
```c
req = &msg->req;
req->src_len = msg->in_cons;
req->dst_len = msg->produced;
if (req->cb)
    req->cb(req, req->cb_param);
else
    WD_ERR("callback function is NULL!\n");
```

---

### 2. wd_alloc_ctx_buf函数参数检查 [MEDIUM]

**位置**: wd_comp.c:442-452

**问题类型**: 参数检查 (cpp-input-validation)

**危险代码**:
```c
static int wd_alloc_ctx_buf(struct wd_mm_ops *mm_ops, struct wd_comp_sess *sess)
{

    sess->ctx_buf = mm_ops->alloc(mm_ops->usr, HW_CTX_SIZE);
    if (!sess->ctx_buf)
        return -WD_ENOMEM;

    memset(sess->ctx_buf, 0, HW_CTX_SIZE);

    return WD_SUCCESS;
}
```

**证据链**:
1. 未检查mm_ops是否为NULL
2. 未检查mm_ops->alloc是否为NULL
3. 如果mm_ops为NULL，会导致空指针解引用

**风险分析**:
- 虽然调用方可能有检查，但内部函数应具备防御性编程
- 建议添加参数检查

**修复建议**:
```c
static int wd_alloc_ctx_buf(struct wd_mm_ops *mm_ops, struct wd_comp_sess *sess)
{
    if (!mm_ops || !mm_ops->alloc || !sess)
        return -WD_EINVAL;

    sess->ctx_buf = mm_ops->alloc(mm_ops->usr, HW_CTX_SIZE);
    // ...
}
```

---

### 3. wd_free_ctx_buf函数参数检查 [MEDIUM]

**位置**: wd_comp.c:454-458

**问题类型**: 参数检查 (cpp-input-validation)

**危险代码**:
```c
static void wd_free_ctx_buf(struct wd_mm_ops *mm_ops, struct wd_comp_sess *sess)
{
    mm_ops->free(mm_ops->usr, sess->ctx_buf);  // 未检查mm_ops
    sess->ctx_buf = NULL;
}
```

**分析**:
- 未检查mm_ops是否为NULL
- 未检查mm_ops->free是否为NULL
- 未检查sess->ctx_buf是否为NULL（虽然free(NULL)是安全的）

**修复建议**:
```c
static void wd_free_ctx_buf(struct wd_mm_ops *mm_ops, struct wd_comp_sess *sess)
{
    if (!mm_ops || !mm_ops->free || !sess)
        return;

    mm_ops->free(mm_ops->usr, sess->ctx_buf);
    sess->ctx_buf = NULL;
}
```

---

### 4. wd_do_comp_sync2函数整数溢出风险 [MEDIUM]

**位置**: wd_comp.c:774-779

**问题类型**: 整数溢出 (cpp-integer-overflow)

**危险代码**:
```c
req->dst_len += strm_req.dst_len;
strm_req.dst += strm_req.dst_len;
total_avail_out -= strm_req.dst_len;

strm_req.src += strm_req.src_len;
total_avail_in -= strm_req.src_len;
```

**证据链**:
1. line 774: req->dst_len累加，可能溢出
2. line 775: 指针增加，如果dst_len很大可能导致越界
3. line 776: 减法操作，如果strm_req.dst_len > total_avail_out，会下溢
4. 循环中持续修改这些值，缺乏边界检查

**风险分析**:
- 如果输入数据被恶意构造，可能导致整数溢出
- 指针计算可能导致缓冲区越界访问
- 建议添加边界检查

**修复建议**:
```c
if (strm_req.dst_len > total_avail_out) {
    WD_ERR("dst_len overflow!\n");
    return -WD_EINVAL;
}
req->dst_len += strm_req.dst_len;
total_avail_out -= strm_req.dst_len;
```

---

### 5. append_store_block函数边界检查 [MEDIUM]

**位置**: wd_comp.c:811-848

**问题类型**: 缓冲区溢出 (cpp-memory-safety)

**危险代码**:
```c
static int append_store_block(struct wd_comp_sess *sess,
                              struct wd_comp_req *req)
{
    unsigned char store_block[5] = {0x1, 0x00, 0x00, 0xff, 0xff};
    // ...

    if (sess->alg_type == WD_ZLIB) {
        if (unlikely(req->dst_len < blocksize + sizeof(checksum)))
            return -WD_EINVAL;
        memcpy(req->dst, store_block, blocksize);  // 未检查req->dst是否为NULL
        // ...
    }
```

**证据链**:
1. 检查了req->dst_len大小，但未检查req->dst是否为NULL
2. memcpy直接使用req->dst，可能为NULL
3. 如果req->dst为NULL，会导致崩溃

**修复建议**:
```c
if (sess->alg_type == WD_ZLIB) {
    if (!req->dst || req->dst_len < blocksize + sizeof(checksum))
        return -WD_EINVAL;
    memcpy(req->dst, store_block, blocksize);
    // ...
}
```

---

### 6. wd_comp_check_buffer函数检查 [LOW]

**位置**: wd_comp.c:602-617

**问题类型**: 参数检查 (cpp-input-validation)

**代码**:
```c
static int wd_comp_check_buffer(struct wd_comp_req *req, struct wd_comp_sess *sess)
{
    if (req->data_fmt == WD_FLAT_BUF) {
        if (unlikely(!req->src || !req->dst)) {
            WD_ERR("invalid: src or dst is NULL!\n");
            return -WD_EINVAL;
        }
    } else if (req->data_fmt == WD_SGL_BUF) {
        if (unlikely(!req->list_src || !req->list_dst)) {
            WD_ERR("invalid: list_src or list_dst is NULL!\n");
            return -WD_EINVAL;
        }
    }
    // ...
}
```

**分析**:
- 检查了FLAT_BUF和SGL_BUF两种情况
- 但如果data_fmt既不是FLAT_BUF也不是SGL_BUF，函数会继续执行
- 应该添加对非法data_fmt的处理

---

### 7. wd_comp_init2_函数循环逻辑 [LOW]

**位置**: wd_comp.c:284-323

**问题类型**: 代码逻辑 (cpp-coding-standards)

**代码**:
```c
while (ret != 0) {
    // ... 初始化操作 ...
}
```

**分析**:
- 与之前文件相同的循环模式
- 代码可读性较差，但功能正确

---

## wd_comp.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 1 |
| MEDIUM | 4 |
| LOW | 2 |

**重点问题**: 
- wd_comp_poll_ctx回调函数未检查(HIGH)
- wd_do_comp_sync2整数溢出风险(MEDIUM)

**建议修复顺序**:
1. 修复poll函数的回调检查问题
2. 添加整数溢出边界检查
3. 完善内存操作函数的参数检查

---

## 检视进度更新

**已完成**: 9/94 文件 (9.6%)

**下次检视**: drv/hisi_sec.c (驱动文件)