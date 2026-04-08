# UADK代码检视报告 - 第4轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第4轮
- **规则来源**: E:/work_space/加速器用户态学习/代码检视/review_rules
- **使用Skills**: cpp-memory-safety, cpp-resource-management, cpp-concurrency-userspace

---

## 本轮检视文件

### 已检视文件
1. wd.c (第1轮) - 检视报告: review_round_001_wd_c.md
2. wd_util.c (第2轮) - 检视报告: review_round_002_wd_util_c.md
3. wd_alg.c (第3轮) - 检视报告: review_round_003_wd_alg_c.md
4. wd_sched.c (本轮) - 调度器文件，909行

### 待检视文件
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

## wd_sched.c 检视结果

### 1. 空指针解引用风险 - sched_key_valid函数 [HIGH]

**位置**: wd_sched.c:92-102

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static bool sched_key_valid(struct wd_sched_ctx *sched_ctx, const struct sched_key *key)
{
    if (key->numa_id >= sched_ctx->numa_num || key->mode >= SCHED_MODE_BUTT ||
        key->type >= sched_ctx->type_num) {
        WD_ERR("invalid: sched key's numa: %d, mode: %u, type: %u!\n",
               key->numa_id, key->mode, key->type);
        return false;
    }

    return true;
}
```

**证据链**:
1. 函数参数sched_ctx和key可能为NULL
2. 直接访问key->numa_id和sched_ctx->numa_num
3. 若key或sched_ctx为NULL将导致崩溃
4. 调用方session_sched_init_ctx (line 164) 未事先检查参数

**风险分析**:
- 作为内部函数，虽然调用方可能有检查，但缺乏防御性编程
- 若上层调用链中存在漏洞，可能触发空指针崩溃
- 建议添加参数检查

**修复建议**:
```c
static bool sched_key_valid(struct wd_sched_ctx *sched_ctx, const struct sched_key *key)
{
    if (!sched_ctx || !key)
        return false;

    if (key->numa_id >= sched_ctx->numa_num || key->mode >= SCHED_MODE_BUTT ||
        key->type >= sched_ctx->type_num) {
        WD_ERR("invalid: sched key's numa: %d, mode: %u, type: %u!\n",
               key->numa_id, key->mode, key->type);
        return false;
    }

    return true;
}
```

---

### 2. 数组越界风险 - session_poll_policy_rr函数 [HIGH]

**位置**: wd_sched.c:284-304

**问题类型**: 数组越界 (cpp-memory-safety)

**危险代码**:
```c
static int session_poll_policy_rr(struct wd_sched_ctx *sched_ctx, int numa_id,
                                  __u32 expect, __u32 *count)
{
    struct sched_ctx_region **region = sched_ctx->sched_info[numa_id].ctx_region;
    __u32 begin, end;
    __u32 i;
    int ret;

    for (i = 0; i < sched_ctx->type_num; i++) {
        // ...
    }
    // ...
}
```

**证据链**:
1. numa_id参数直接用于索引sched_ctx->sched_info数组
2. 未检查numa_id是否在有效范围内
3. 如果numa_id >= sched_ctx->numa_num，会导致数组越界访问
4. 调用方session_sched_poll_policy (line 355) 传入的i值来自循环

**风险分析**:
- 调用方循环中i的范围是region_mum
- region_mum来自sched_ctx->numa_num，理论上安全
- 但如果numa_num被错误设置，会导致越界
- 建议添加边界检查

**修复建议**:
```c
static int session_poll_policy_rr(struct wd_sched_ctx *sched_ctx, int numa_id,
                                  __u32 expect, __u32 *count)
{
    struct sched_ctx_region **region;
    __u32 begin, end;
    __u32 i;
    int ret;

    if (!sched_ctx || numa_id < 0 || numa_id >= sched_ctx->numa_num)
        return -WD_EINVAL;

    region = sched_ctx->sched_info[numa_id].ctx_region;
    // ...
}
```

---

### 3. 空指针解引用风险 - sched_get_ctx_range函数 [MEDIUM]

**位置**: wd_sched.c:107-125

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static struct sched_ctx_region *sched_get_ctx_range(struct wd_sched_ctx *sched_ctx,
                                                    const struct sched_key *key)
{
    struct wd_sched_info *sched_info;
    int numa_id;

    sched_info = sched_ctx->sched_info;
    if (key->numa_id >= 0 &&
        sched_info[key->numa_id].ctx_region[key->mode][key->type].valid)
        return &sched_info[key->numa_id].ctx_region[key->mode][key->type];
    // ...
}
```

**证据链**:
1. 直接访问sched_ctx->sched_info，未检查sched_ctx
2. 直接访问key->numa_id，未检查key
3. sched_key_valid在调用方(line 164)已做部分检查
4. 但函数本身缺乏防御性检查

**风险分析**:
- 如果调用链中检查不完整，可能导致崩溃
- 建议添加参数检查

---

### 4. 资源释放后内存泄漏风险 [MEDIUM]

**位置**: wd_sched.c:770-808

**问题类型**: 资源管理 (cpp-resource-management)

**代码分析**:
```c
void wd_sched_rr_release(struct wd_sched *sched)
{
    // ...
    for (i = 0; i < region_num; i++) {
        for (j = 0; j < SCHED_MODE_BUTT; j++) {
            if (sched_info[i].ctx_region[j]) {
                free(sched_info[i].ctx_region[j]);
                sched_info[i].ctx_region[j] = NULL;
            }
        }
    }

info_out:
    free(sched_ctx);
ctx_out:
    free(sched);

    return;
}
```

**分析**:
- 释放逻辑基本正确
- 但pthread_mutex_destroy未被调用
- 每个ctx_region在初始化时(line 702, 765)调用了pthread_mutex_init
- 释放时应调用pthread_mutex_destroy避免资源泄漏

**修复建议**:
```c
void wd_sched_rr_release(struct wd_sched *sched)
{
    // ...
    for (i = 0; i < region_num; i++) {
        for (j = 0; j < SCHED_MODE_BUTT; j++) {
            if (sched_info[i].ctx_region[j]) {
                for (k = 0; k < sched_ctx->type_num; k++)
                    pthread_mutex_destroy(&sched_info[i].ctx_region[j][k].lock);
                free(sched_info[i].ctx_region[j]);
                sched_info[i].ctx_region[j] = NULL;
            }
        }
    }
    // ...
}
```

---

### 5. wd_sched_get_nearby_numa_id函数参数检查 [LOW]

**位置**: wd_sched.c:617-634

**问题类型**: 参数检查 (cpp-input-validation)

**代码**:
```c
static int wd_sched_get_nearby_numa_id(struct wd_sched_info *sched_info, int node, int numa_num)
{
    int dis = INT32_MAX;
    int valid_id = -1;
    int i, tmp;

    for (i = 0; i < numa_num; i++) {
        if (sched_info[i].valid) {
            tmp = numa_distance(node, i);
            // ...
        }
    }

    return valid_id;
}
```

**分析**:
- 未检查sched_info是否为NULL
- 未检查numa_num是否为有效值
- 由于是static内部函数，风险较低

---

### 6. 错误处理路径检查 [LOW]

**位置**: wd_sched.c:905-907

**问题类型**: 错误处理 (cpp-resource-management)

**代码**:
```c
err_out:
    wd_sched_rr_release(sched);
    return NULL;
```

**分析**:
- wd_sched_rr_release会释放sched及其内部资源
- 错误处理路径正确，会清理已分配的资源
- 设计良好

---

## wd_sched.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 2 |
| MEDIUM | 2 |
| LOW | 2 |

**重点问题**: 
- sched_key_valid函数缺少参数检查(HIGH)
- session_poll_policy_rr函数缺少数组边界检查(HIGH)
- pthread_mutex_destroy未在释放时调用(MEDIUM)

**建议修复顺序**:
1. 修复空指针检查问题
2. 添加数组边界检查
3. 在释放函数中添加pthread_mutex_destroy调用

---

## 检视进度更新

**已完成**: 4/94 文件 (4.3%)

**下次检视**: wd_mempool.c (内存池文件)