# UADK代码检视报告 - 第40轮

## 检视信息
- **检视文件**: v1/wd_comp.c
- **检视时间**: 2026-04-08
- **文件行数**: 406行
- **检视规则**: cpp-memory-safety, cpp-resource-management, cpp-input-validation

## 检视摘要
| 等级 | 数量 |
|------|------|
| HIGH | 1 |
| MEDIUM | 4 |
| LOW | 2 |
| INFO | 1 |
| POSITIVE | 2 |

---

## 问题详情

### 1. [空指针解引用风险] - wcrypto_comp_poll函数 - [Severity: HIGH]

**Location**: `v1/wd_comp.c:359-362`

**Dangerous Code**:
```c
tag = (void *)(uintptr_t)resp->udata;
ctx = tag->wcrypto_tag.ctx;
ctx->cb(resp, tag->wcrypto_tag.tag);
wd_put_cookies(&ctx->pool, (void **)&tag, 1);
```

**Risk Analysis**:
1. resp->udata可能为0，导致tag为NULL
2. 直接访问tag->wcrypto_tag.ctx会导致崩溃
3. ctx->cb调用，如果cb为NULL也会崩溃
4. 与v1版本其他模块（cipher/digest/aead）的相同问题模式

**Evidence Chain**:
1. 第344行：wd_recv接收响应
2. 第347-356行：处理各种返回值情况
3. 第359行：从udata获取tag，未验证有效性
4. 第360行：直接访问tag->wcrypto_tag.ctx
5. 第361行：调用ctx->cb

**Fix Suggestion**:
```c
tag = (void *)(uintptr_t)resp->udata;
if (!tag) {
    WD_ERR("resp->udata is NULL!\n");
    return -WD_EINVAL;
}
ctx = tag->wcrypto_tag.ctx;
if (!ctx || !ctx->cb) {
    WD_ERR("comp ctx or cb is NULL!\n");
    return -WD_EINVAL;
}
ctx->cb(resp, tag->wcrypto_tag.tag);
wd_put_cookies(&ctx->pool, (void **)&tag, 1);
```

---

### 2. [resp未检查NULL] - wcrypto_do_comp函数 - [Severity: MEDIUM]

**Location**: `v1/wd_comp.c:293, 311-316`

**Dangerous Code**:
```c
resp = (void *)(uintptr_t)cctx->ctx_id;  // 初始化为临时值

do {
    ret = wd_recv(cctx->q, (void **)&resp);
    if (ret > 0) {
        break;  // 跳出循环，但ret可能是其他值
    } else if (!ret) {
        ...
    } else {
        ...
        goto err_put_cookie;  // 错误路径
    }
} while (true);

opdata->consumed = resp->in_cons;  // resp可能为NULL
```

**Risk Analysis**:
1. 循环在ret > 0时跳出，但跳出后resp未检查
2. 如果wd_recv返回特殊值导致非正常跳出，resp可能为NULL
3. 直接访问resp->in_cons会导致崩溃

**Evidence Chain**:
1. 第293行：resp初始化为ctx_id临时值
2. 第296-309行：wd_recv循环接收
3. 第297行：ret > 0时跳出循环
4. 第311行：直接访问resp->in_cons，未检查resp

**Fix Suggestion**:
```c
} while (true);

if (!resp) {
    WD_ERR("resp is NULL after wd_recv!\n");
    ret = -WD_EINVAL;
    goto err_put_cookie;
}

opdata->consumed = resp->in_cons;
```

---

### 3. [ctx参数未检查NULL] - fill_comp_msg函数 - [Severity: MEDIUM]

**Location**: `v1/wd_comp.c:43-61`

**Dangerous Code**:
```c
static void fill_comp_msg(struct wcrypto_comp_ctx *ctx,
                          struct wcrypto_comp_msg *msg,
                          struct wcrypto_comp_op_data *opdata)
{
    msg->avail_out = opdata->avail_out;  // msg未检查NULL
    msg->src = opdata->in;
    ...
    msg->tag = ctx->ctx_id;  // ctx未检查NULL
```

**Risk Analysis**:
1. 函数参数ctx、msg、opdata未检查是否为NULL
2. 如果任一参数为NULL，会导致崩溃
3. 调用点wcrypto_do_comp第283行未检查cookie是否为NULL

**Fix Suggestion**:
```c
static int fill_comp_msg(struct wcrypto_comp_ctx *ctx,
                         struct wcrypto_comp_msg *msg,
                         struct wcrypto_comp_op_data *opdata)
{
    if (!ctx || !msg || !opdata)
        return -WD_EINVAL;

    msg->avail_out = opdata->avail_out;
    ...
    return WD_SUCCESS;
}
```

---

### 4. [cookie未检查NULL] - wcrypto_do_comp函数 - [Severity: MEDIUM]

**Location**: `v1/wd_comp.c:267, 271`

**Dangerous Code**:
```c
ret = wd_get_cookies(&cctx->pool, (void **)&cookie, 1);
if (ret)
    return ret;

msg = &cookie->msg;  // cookie可能为NULL（即使ret==0）
```

**Risk Analysis**:
1. wd_get_cookies返回0不代表cookie一定有效
2. cookie可能为NULL，访问cookie->msg会崩溃

**Fix Suggestion**:
```c
ret = wd_get_cookies(&cctx->pool, (void **)&cookie, 1);
if (ret)
    return ret;

if (!cookie) {
    WD_ERR("cookie is NULL!\n");
    return -WD_EINVAL;
}

msg = &cookie->msg;
```

---

### 5. [ctx未检查NULL] - init_comp_ctx函数 - [Severity: MEDIUM]

**Location**: `v1/wd_comp.c:141`

**Dangerous Code**:
```c
static int init_comp_ctx(struct wcrypto_comp_ctx *ctx, int ctx_id,
                         struct wcrypto_comp_ctx_setup *setup)
{
    int cache_num = ctx->pool.cookies_num;  // ctx未检查NULL
```

**Risk Analysis**:
1. ctx参数未检查是否为NULL
2. 调用点wcrypto_create_comp_ctx第228行，ctx已分配，但缺乏防御性检查

**Fix Suggestion**:
```c
static int init_comp_ctx(struct wcrypto_comp_ctx *ctx, int ctx_id,
                         struct wcrypto_comp_ctx_setup *setup)
{
    if (!ctx || !setup)
        return -WD_EINVAL;

    int cache_num = ctx->pool.cookies_num;
```

---

### 6. [重复删除资源泄漏] - wcrypto_del_comp_ctx函数 - [Severity: LOW]

**Location**: `v1/wd_comp.c:391-395`

**Dangerous Code**:
```c
if (qinfo->ctx_num <= 0) {
    wd_unspinlock(&qinfo->qlock);
    WD_ERR("error: repeat delete compress ctx!\n");
    return;  // ctx内存未释放
}
```

**Risk Analysis**:
1. ctx_num <= 0（重复删除）时直接返回
2. cctx内存未释放，可能导致内存泄漏
3. 与其他模块相同的问题模式

**Fix Suggestion**:
```c
if (qinfo->ctx_num <= 0) {
    wd_unspinlock(&qinfo->qlock);
    WD_ERR("error: repeat delete compress ctx!\n");
    free(cctx);  // 释放ctx内存
    return;
}
```

---

### 7. [指针检查冗余] - wcrypto_del_comp_ctx函数 - [Severity: LOW]

**Location**: `v1/wd_comp.c:386`

**Dangerous Code**:
```c
br = &qinfo->br;
if (br && br->free && cctx->ctx_buf)
```

**Risk Analysis**:
br来自`&qinfo->br`，不可能为NULL。

**Fix Suggestion**:
```c
if (br->free && cctx->ctx_buf)
```

---

### 8. [信息性发现] - ctx_buf清零 - [Severity: INFO]

**Location**: `v1/wd_comp.c:58-60`

**Code**:
```c
if (msg->stream_mode == WCRYPTO_COMP_STATEFUL &&
    opdata->stream_pos == WCRYPTO_COMP_STREAM_NEW && ctx->ctx_buf)
    memset(ctx->ctx_buf, 0, MAX_CTX_RSV_SIZE);
```

**Analysis**:
有状态压缩模式下的ctx_buf管理：
1. 在新的流开始时清零ctx_buf
2. 检查了ctx_buf是否为NULL
3. 这是压缩算法有状态模式的正确处理方式

---

### 9. [正面发现] - ctx_params_check完整验证 - [Severity: POSITIVE]

**Location**: `v1/wd_comp.c:63-117`

**Code**:
```c
if (!q || !q->qinfo || !setup) {
    WD_ERR("%s: input param err!\n", __func__);
    return -WD_EINVAL;
}

if (setup->alg_type >= WCRYPTO_COMP_MAX_ALG) {
    WD_ERR("err: alg_type is invalid!\n");
    return -WD_EINVAL;
}

if (setup->comp_lv > WCRYPTO_COMP_L9) {
    ...
}

if (setup->op_type > WCRYPTO_INFLATE) {
    ...
}

if (setup->stream_mode > WCRYPTO_COMP_STATEFUL) {
    ...
}

if (setup->win_size > WCRYPTO_COMP_WS_32K) {
    ...
}

if (setup->data_fmt > WD_SGL_BUF) {
    ...
}
```

**Analysis**:
完整的参数验证：
1. 检查核心参数q、qinfo、setup是否为NULL
2. 检查alg_type、comp_lv、op_type、stream_mode、win_size、data_fmt的范围
3. 验证算法名称匹配（zlib、gzip、deflate、lz77_zstd）
4. 检查ctx_num是否超过限制

---

### 10. [正面发现] - 错误处理路径完整 - [Severity: POSITIVE]

**Location**: `v1/wd_comp.c:237-246`

**Code**:
```c
free_ctx_buf:
    free(ctx);
free_ctx_id:
    wd_free_id(qinfo->ctx_id, WD_MAX_CTX_NUM, ctx_id, WD_MAX_CTX_NUM);
    qinfo->ctx_num--;
    wd_spinlock(&qinfo->qlock);
unlock:
    wd_unspinlock(&qinfo->qlock);
    return NULL;
```

**Analysis**:
错误处理路径设计：
1. 使用goto链式错误处理
2. 按分配顺序的反序释放资源
3. 正确处理锁状态

---

## 总结

### 问题分布
| 问题类型 | 数量 |
|----------|------|
| 空指针检查缺失 | 5 |
| 资源管理 | 1 |
| 代码质量 | 1 |
| 安全实践 | 2 |

### 主要风险
1. **wcrypto_comp_poll空指针解引用**：v1版本所有模块（cipher/digest/aead/comp）都有此问题

### 与之前检视的关联
1. 第37-39轮检视的v1/wd_cipher.c、wd_digest.c、wd_aead.c发现poll函数空指针解引用，v1/wd_comp.c有相同问题模式
2. v1版本的四个主要模块（cipher/digest/aead/comp）代码结构相似，问题模式高度一致

### v1版本问题模式汇总表（更新）

| 问题模式 | cipher | digest | aead | comp |
|---------|--------|--------|------|------|
| poll函数空指针解引用 | ✓ | ✓ | ✓ | ✓ |
| 数组越界（静态数组） | - | ✓ | ✓ | - |
| do_xxx resp未检查NULL | - | ✓ | ✓ | ✓ |
| fill_xxx_msg参数未检查NULL | - | - | - | ✓ |
| init_xxx_ctx参数未检查NULL | - | - | - | ✓ |
| cookie未检查NULL | ✓ | ✓ | ✓ | ✓ |
| del_ctx重复删除资源泄漏 | ✓ | ✓ | ✓ | ✓ |
| del_ctx br检查冗余 | ✓ | ✓ | ✓ | ✓ |

### 建议优先修复顺序
1. wcrypto_comp_poll空指针检查（HIGH）
2. wcrypto_do_comp resp检查（MEDIUM）
3. 各函数的参数NULL检查（MEDIUM）