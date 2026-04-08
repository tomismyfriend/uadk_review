# UADK代码检视报告 - 第2轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第2轮
- **规则来源**: E:/work_space/加速器用户态学习/代码检视/review_rules
- **使用Skills**: cpp-memory-safety, cpp-resource-management, cpp-concurrency-userspace

---

## 本轮检视文件

### 已检视文件
1. wd.c (第1轮) - 检视报告: review_round_001_wd_c.md
2. wd_util.c (本轮) - 核心工具函数文件，约2400行

### 待检视文件
- wd_alg.c
- wd_sched.c
- wd_mempool.c
- wd_cipher.c
- wd_digest.c
- wd_aead.c
- wd_comp.c
- wd_rsa.c
- wd_dh.c
- wd_ecc.c
- wd_bmm.c
- wd_agg.c
- wd_join_gather.c
- wd_udma.c
- wd_zlibwrapper.c
- drv/目录下的所有文件
- include/目录下的所有头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## wd_util.c 检视结果

### 1. 空指针解引用风险 - wd_parse_dev_id函数 [HIGH]

**位置**: wd_util.c:188-211

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static int wd_parse_dev_id(handle_t h_ctx)
{
    struct wd_ctx_h *ctx = (struct wd_ctx_h *)h_ctx;
    char *dev_path = ctx->dev_path;  // 直接访问ctx成员
    char *last_str = NULL;
    char *endptr;
    int dev_id;

    if (!dev_path)
        return -WD_EINVAL;
    // ...
}
```

**证据链**:
1. 函数参数h_ctx可能为无效值(如0或NULL)
2. line 190: ctx直接从h_ctx转换，未进行NULL检查
3. line 191: 直接访问ctx->dev_path，若ctx为NULL将导致崩溃
4. 仅检查dev_path是否为NULL，未检查ctx本身

**风险分析**:
- 传入无效handle_t会导致空指针解引用崩溃
- 该函数被多处调用，需要防御性编程

**修复建议**:
```c
static int wd_parse_dev_id(handle_t h_ctx)
{
    struct wd_ctx_h *ctx = (struct wd_ctx_h *)h_ctx;
    char *dev_path;
    char *last_str = NULL;
    char *endptr;
    int dev_id;

    if (!ctx)
        return -WD_EINVAL;

    dev_path = ctx->dev_path;
    if (!dev_path)
        return -WD_EINVAL;
    // ...
}
```

---

### 2. 内存泄漏 - wd_env_set_ctx_nums函数 [HIGH]

**位置**: wd_util.c:2087-2126

**问题类型**: 内存泄漏 (cpp-memory-safety)

**危险代码**:
```c
static int wd_env_set_ctx_nums(const char *alg_name, const char *name, const char *var_s,
                               struct wd_ctx_params *ctx_params, __u32 op_type_num)
{
    char alg_type[CRYPTO_MAX_ALG_NAME];
    char *left, *section, *start;
    struct uacce_dev_list *list;
    int is_comp;
    int ret;

    // ...

    start = strdup(var_s);
    if (!start)
        return -WD_ENOMEM;

    ret = wd_get_alg_type(alg_name, alg_type);
    if (ret)
        return ret;  // 内存泄漏！start未释放

    list = wd_get_accel_list(alg_type);
    // ...
}
```

**证据链**:
1. line 2101: strdup分配内存给start
2. line 2106: wd_get_alg_type调用可能失败并返回错误
3. line 2107: 直接return ret，未释放start内存
4. 导致每次调用失败都泄漏一个字符串内存

**风险分析**:
- 每次调用wd_get_alg_type失败都会泄漏内存
- 长时间运行的程序可能累积大量内存泄漏
- 影响系统稳定性

**修复建议**:
```c
static int wd_env_set_ctx_nums(...)
{
    // ...
    start = strdup(var_s);
    if (!start)
        return -WD_ENOMEM;

    ret = wd_get_alg_type(alg_name, alg_type);
    if (ret) {
        free(start);  // 添加释放
        return ret;
    }
    // ...
}
```

---

### 3. 空指针解引用风险 - wd_get_msg_from_pool函数 [HIGH]

**位置**: wd_util.c:499-523

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
int wd_get_msg_from_pool(struct wd_async_msg_pool *pool,
                         int ctx_idx, void **msg)
{
    struct msg_pool *p = &pool->pools[ctx_idx];  // 直接访问pool成员
    __u32 msg_num = p->msg_num;
    // ...
}
```

**证据链**:
1. 函数未检查pool参数是否为NULL
2. line 502: 直接访问pool->pools[ctx_idx]
3. 若pool为NULL将导致崩溃
4. wd_find_msg_in_pool函数(line 477)有类似问题

**风险分析**:
- pool参数可能为NULL(如上层初始化失败)
- 直接访问会导致空指针解引用崩溃
- 多个异步消息池相关函数都有此风险

**修复建议**:
```c
int wd_get_msg_from_pool(struct wd_async_msg_pool *pool,
                         int ctx_idx, void **msg)
{
    struct msg_pool *p;
    __u32 msg_num;

    if (!pool || !msg)
        return -WD_EINVAL;

    if ((__u32)ctx_idx >= pool->pool_num) {
        WD_ERR("invalid: ctx_idx %d >= pool_num %u!\n", ctx_idx, pool->pool_num);
        return -WD_EINVAL;
    }

    p = &pool->pools[ctx_idx];
    // ...
}
```

---

### 4. 并发安全 - 信号量错误处理不完整 [MEDIUM]

**位置**: wd_util.c:1447-1471

**问题类型**: 并发安全 (cpp-concurrency-userspace)

**危险代码**:
```c
int wd_add_task_to_async_queue(struct wd_env_config *config, __u32 idx)
{
    // ...
    ret = sem_wait(&task_queue->empty_sem);
    if (ret) {
        WD_ERR("failed to wait empty_sem!\n");
        return ret;
    }

    pthread_mutex_lock(&task_queue->lock);
    // ... 操作 ...
    pthread_mutex_unlock(&task_queue->lock);

    ret = sem_post(&task_queue->full_sem);
    if (ret) {
        WD_ERR("failed to post full_sem!\n");
        goto err_out;
    }
    // ...
}
```

**证据链**:
1. sem_wait可能因EINTR中断而返回错误
2. pthread_mutex_lock/unlock操作与信号量混合使用
3. err_out路径中需要恢复状态，但未完全处理所有情况
4. 多线程环境下可能导致死锁或状态不一致

**风险分析**:
- sem_wait可能因信号中断返回-1且errno=EINTR
- 应区分EINTR和其他真正的错误
- 错误恢复路径可能存在竞态条件

**修复建议**:
```c
int wd_add_task_to_async_queue(struct wd_env_config *config, __u32 idx)
{
    // ...
    do {
        ret = sem_wait(&task_queue->empty_sem);
    } while (ret == -1 && errno == EINTR);

    if (ret) {
        WD_ERR("failed to wait empty_sem!\n");
        return -errno;
    }
    // ...
}
```

---

### 5. strtol返回值检查不完整 [MEDIUM]

**位置**: wd_util.c:776

**问题类型**: 返回值检查 (cpp-return-value-checking)

**危险代码**:
```c
static int str_to_bool(const char *s, bool *target)
{
    int tmp;

    if (!is_number(s))
        return -WD_EINVAL;

    tmp = strtol(s, NULL, 10);
    if (tmp != 0 && tmp != 1)
        return -WD_EINVAL;

    *target = tmp;

    return 0;
}
```

**证据链**:
1. strtol可能返回LONG_MAX/LONG_MIN并设置errno=ERANGE
2. 仅检查值是否为0或1，未检查溢出情况
3. 如果字符串表示的数字超出int范围，行为未定义
4. is_number函数仅检查是否为数字字符串，不检查范围

**风险分析**:
- 极大数值可能导致strtol溢出
- 返回值可能不准确
- 但由于is_number检查，实际风险较低

**修复建议**:
```c
static int str_to_bool(const char *s, bool *target)
{
    long tmp;
    char *endptr;

    if (!is_number(s))
        return -WD_EINVAL;

    errno = 0;
    tmp = strtol(s, &endptr, 10);
    if (errno == ERANGE || tmp < INT_MIN || tmp > INT_MAX)
        return -WD_EINVAL;

    if (tmp != 0 && tmp != 1)
        return -WD_EINVAL;

    *target = (bool)tmp;

    return 0;
}
```

---

### 6. wd_init_sched函数strdup失败处理 [MEDIUM]

**位置**: wd_util.c:321-336

**问题类型**: 资源管理 (cpp-resource-management)

**代码分析**:
```c
int wd_init_sched(struct wd_sched *in, struct wd_sched *from)
{
    if (!from->name || !from->sched_init ||
        !from->pick_next_ctx || !from->poll_policy) {
        WD_ERR("invalid: member of wd_sched is NULL!\n");
        return -WD_EINVAL;
    }

    in->h_sched_ctx = from->h_sched_ctx;
    in->name = strdup(from->name);  // 可能失败返回NULL
    in->sched_init = from->sched_init;
    in->pick_next_ctx = from->pick_next_ctx;
    in->poll_policy = from->poll_policy;

    return 0;  // 未检查strdup返回值
}
```

**证据链**:
1. strdup可能因内存不足返回NULL
2. 未检查in->name是否为NULL
3. 后续使用in->name可能出问题
4. wd_clear_sched函数会释放name，可能导致free(NULL)或free(野指针)

**风险分析**:
- strdup失败后in->name为NULL
- wd_clear_sched调用free(name)是安全的(free(NULL)合法)
- 但后续使用name指针可能导致崩溃

**修复建议**:
```c
int wd_init_sched(struct wd_sched *in, struct wd_sched *from)
{
    // ...
    in->name = strdup(from->name);
    if (!in->name)
        return -WD_ENOMEM;

    in->sched_init = from->sched_init;
    // ...
}
```

---

### 7. wd_check_ctx函数数组边界检查 [LOW]

**位置**: wd_util.c:1840-1856

**问题类型**: 数组边界检查 (cpp-memory-safety)

**代码**:
```c
int wd_check_ctx(struct wd_ctx_config_internal *config, __u8 mode, __u32 idx)
{
    struct wd_ctx_internal *ctx;

    if (unlikely(idx >= config->ctx_num)) {
        WD_ERR("failed to pick a proper ctx: idx %u!\n", idx);
        return -WD_EINVAL;
    }

    ctx = config->ctxs + idx;  // config->ctxs可能为NULL
    // ...
}
```

**分析**:
- 检查idx是否超出ctx_num，但未检查config->ctxs是否为NULL
- 如果ctxs为NULL，config->ctxs + idx会导致空指针偏移
- 实际风险较低，因为ctx_num为0时idx检查会失败

---

## wd_util.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 3 |
| MEDIUM | 3 |
| LOW | 1 |

**重点问题**: 
- wd_parse_dev_id函数缺少NULL检查(HIGH)
- wd_env_set_ctx_nums函数内存泄漏(HIGH)
- wd_get_msg_from_pool函数缺少NULL检查(HIGH)

**建议修复顺序**:
1. 优先修复内存泄漏问题(wd_env_set_ctx_nums)
2. 其次处理空指针解引用问题(wd_parse_dev_id, wd_get_msg_from_pool)
3. 改进strdup返回值检查和strtol边界检查

---

## 检视进度更新

**已完成**: 2/94 文件 (2.1%)

**下次检视**: wd_alg.c (算法管理文件)