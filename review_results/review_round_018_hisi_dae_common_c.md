# UADK代码检视报告 - 第18轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第18轮
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
9. wd_comp.c (第9轮) - 检视报告: review_round_009_wd_comp_c.md
10. drv/hisi_sec.c (第10轮) - 检视报告: review_round_010_hisi_sec_c.md
11. drv/hisi_hpre.c (第11轮) - 检视报告: review_round_011_hisi_hpre_c.md
12. drv/hisi_comp.c (第12轮) - 检视报告: review_round_012_hisi_comp_c.md
13. drv/hisi_qm_udrv.c (第13轮) - 检视报告: review_round_013_hisi_qm_udrv_c.md
14. drv/isa_ce_sm3.c (第14轮) - 检视报告: review_round_014_isa_ce_sm3_c.md
15. drv/isa_ce_sm4.c (第15轮) - 检视报告: review_round_015_isa_ce_sm4_c.md
16. drv/hisi_udma.c (第16轮) - 检视报告: review_round_016_hisi_udma_c.md
17. drv/hisi_dae.c (第17轮) - 检视报告: review_round_017_hisi_dae_c.md
18. drv/hisi_dae_common.c (本轮) - DAE通用函数文件，430行

### 待检视文件
- drv/hisi_dae_join_gather.c, drv/hisi_comp_huf.c
- drv/hash_mb/hash_mb.c
- include/目录下的所有头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## drv/hisi_dae_common.c 检视结果

### 1. 空指针解引用风险 - get_free_ext_addr函数 [MEDIUM]

**位置**: hisi_dae_common.c:56-73

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
/* The caller ensures that the address pointer or num is not null. */
int get_free_ext_addr(struct dae_extend_addr *ext_addr)
{
    __u16 addr_num = ext_addr->addr_num;  // 未检查ext_addr是否为NULL
    __u16 idx = ext_addr->tail;
    __u16 cnt = 0;

    while (__atomic_test_and_set(&ext_addr->addr_status[idx], __ATOMIC_ACQUIRE)) {
        idx = (idx + 1) % addr_num;
        cnt++;
        if (cnt == addr_num)
            return -WD_EBUSY;
    }

    ext_addr->tail = (idx + 1) % addr_num;

    return idx;
}
```

**分析**:
- 注释说明"调用者保证指针不为空"，这是不安全的防御性编程实践
- 未检查ext_addr是否为NULL
- 未检查ext_addr->addr_num是否为0(可能导致除零或无限循环)
- 此函数被hisi_dae.c的hashagg_send调用

**风险分析**:
- 依赖调用者保证是不安全的做法
- 如果调用者忘记检查，会导致崩溃
- 建议在函数内部也添加防御性检查

---

### 2. 空指针解引用风险 - put_ext_addr函数 [MEDIUM]

**位置**: hisi_dae_common.c:75-78

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
void put_ext_addr(struct dae_extend_addr *ext_addr, int idx)
{
    __atomic_clear(&ext_addr->addr_status[idx], __ATOMIC_RELEASE);  // 未检查参数
}
```

**分析**:
- 未检查ext_addr是否为NULL
- 未检查idx是否在合法范围内
- 如果ext_addr为NULL会导致崩溃
- 此函数被hisi_dae.c的hashagg_recv和hashagg_send错误处理调用

---

### 3. 空指针解引用风险 - dae_uninit_qp_priv函数 [MEDIUM]

**位置**: hisi_dae_common.c:80-90

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
static void dae_uninit_qp_priv(handle_t h_qp)
{
    struct hisi_qp *qp = (struct hisi_qp *)h_qp;
    struct dae_extend_addr *ext_addr = (struct dae_extend_addr *)qp->priv;  // 未检查qp

    free(ext_addr->addr_list);
    free(ext_addr->addr_status);
    free(ext_addr->ext_sqe);
    free(ext_addr);
    qp->priv = NULL;
}
```

**分析**:
- 未检查qp是否为NULL
- 未检查qp->priv是否为NULL
- 直接访问ext_addr成员并释放内存
- 如果qp或qp->priv为NULL会导致崩溃

**修复建议**:
```c
static void dae_uninit_qp_priv(handle_t h_qp)
{
    struct hisi_qp *qp = (struct hisi_qp *)h_qp;
    struct dae_extend_addr *ext_addr;

    if (!qp)
        return;

    ext_addr = (struct dae_extend_addr *)qp->priv;
    if (!ext_addr)
        return;

    free(ext_addr->addr_list);
    free(ext_addr->addr_status);
    free(ext_addr->ext_sqe);
    free(ext_addr);
    qp->priv = NULL;
}
```

---

### 4. 空指针解引用风险 - dae_init_qp_priv函数 [MEDIUM]

**位置**: hisi_dae_common.c:92-129

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
static int dae_init_qp_priv(handle_t h_qp)
{
    struct hisi_qp *qp = (struct hisi_qp *)h_qp;
    __u16 sq_depth = qp->q_info.sq_depth;  // 未检查qp是否为NULL
    struct dae_extend_addr *ext_addr;
    int ret = -WD_ENOMEM;

    ext_addr = calloc(1, sizeof(struct dae_extend_addr));
    // ...
}
```

**分析**:
- 未检查qp是否为NULL
- 直接访问qp->q_info.sq_depth
- 如果h_qp无效会导致崩溃

---

### 5. 空指针解引用风险 - dae_get_usage函数 [MEDIUM]

**位置**: hisi_dae_common.c:393-429

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
int dae_get_usage(void *param)
{
    struct hisi_dev_usage *dae_usage = (struct hisi_dev_usage *)param;
    struct wd_alg_driver *drv = dae_usage->drv;  // 未检查dae_usage是否为NULL
    struct wd_ctx_config_internal *config;
    struct hisi_dae_ctx *priv;
    char *ctx_dev_name;
    handle_t ctx = 0;
    handle_t qp = 0;
    __u32 i;

    if (dae_usage->alg_op_type >= drv->op_type_num) {  // 未检查drv是否为NULL
        WD_ERR("invalid: alg_op_type %u is error!\n", dae_usage->alg_op_type);
        return -WD_EINVAL;
    }

    priv = (struct hisi_dae_ctx *)drv->priv;
    if (!priv)
        return -WD_EACCES;
    // ...
}
```

**分析**:
- 未检查param是否为NULL
- 直接将param转换为dae_usage并访问成员
- 未检查drv是否为NULL
- 之后检查了priv是否为NULL，这是正确的

---

### 6. 整数运算风险 - dae_ext_table_rownum函数 [LOW]

**位置**: hisi_dae_common.c:131-186

**问题类型**: 整数运算 (cpp-integer-overflow)

**代码**:
```c
static __u32 dae_ext_table_rownum(void **ext_table, struct wd_dae_hash_table *hash_table,
                  __u32 row_size)
{
    __u64 tlb_size, tmp_size, row_num;
    void *tmp_table;

    tmp_table = PTR_ALIGN(hash_table->ext_table, DAE_TABLE_ALIGN_SIZE);
    tlb_size = (__u64)hash_table->table_row_size * hash_table->ext_table_row_num;
    tmp_size = (__u64)(uintptr_t)tmp_table - (__u64)(uintptr_t)hash_table->ext_table;  // 指针减法
    if (tmp_size >= tlb_size)
        return 0;

    row_num = (tlb_size - tmp_size) / row_size;
    if (row_size == ROW_SIZE32) {
        if (tmp_size >= row_size) {
            tmp_table = (__u8 *)tmp_table - row_size;
            row_num += 1;
        } else {
            if (row_num > HASH_TABLE_OFFSET_3ROW) {
                tmp_table = (__u8 *)tmp_table + HASH_TABLE_OFFSET_3ROW * row_size;
                row_num -= HASH_TABLE_OFFSET_3ROW;
            } else {
                return 0;
            }
        }
    } else if (row_size == ROW_SIZE64) {
        // ...
    }

    *ext_table = tmp_table;

    return row_num;
}
```

**分析**:
- 包含复杂的指针运算和整数计算
- 指针减法转换为__u64，理论上不会溢出，但逻辑复杂
- 边界检查逻辑复杂，容易出错
- 建议添加更多中间检查以确保安全

---

### 7. dae_hash_table_init函数参数检查 [LOW]

**位置**: hisi_dae_common.c:270-309

**问题类型**: 参数检查 (cpp-input-validation)

**代码**:
```c
int dae_hash_table_init(struct hash_table_data *hw_table,
            struct hash_table_data *rehash_table,
            struct wd_dae_hash_table *hash_table,
            __u32 row_size)
{
    bool is_rehash = false;
    int ret;

    if (!row_size || row_size > hash_table->table_row_size) {
        WD_ERR("invalid: row size %u is error, device need %u!\n",
            hash_table->table_row_size, row_size);
        return -WD_EINVAL;
    }

    /* hash_std_table is checked by caller */
    if (!hash_table->ext_table || !hash_table->ext_table_row_num) {
        WD_ERR("invalid: hash extend table is null!\n");
        return -WD_EINVAL;
    }
    // ...
}
```

**分析**:
- 检查了row_size、hash_table->ext_table等
- 但未检查hw_table和rehash_table是否为NULL
- 未检查hash_table本身是否为NULL
- 注释说明"hash_std_table is checked by caller"，但应添加防御性检查

---

### 8. 正确实现示例 - dae_init函数 [POSITIVE]

**位置**: hisi_dae_common.c:311-367

**正面代码**:
```c
int dae_init(struct wd_alg_driver *drv, void *conf)
{
    struct wd_ctx_config_internal *config = conf;
    // ...

    if (!config || !config->ctx_num) {
        WD_ERR("invalid: dae init config is null or ctx num is 0!\n");
        return -WD_EINVAL;
    }

    priv = malloc(sizeof(struct hisi_dae_ctx));
    // ...

    // 完善的错误处理和资源释放
    for (j = 0; j < i; j++) {
        h_qp = (handle_t)wd_ctx_get_priv(config->ctxs[j].ctx);
        if (h_qp) {  // 检查h_qp是否为NULL
            dae_uninit_qp_priv(h_qp);
            hisi_qm_free_qp(h_qp);
        }
    }
    // ...
}
```

**分析**:
- 正确检查了config和config->ctx_num
- 错误处理路径完善，有资源释放
- 检查了h_qp是否为NULL才调用释放函数

---

### 9. 正确实现示例 - dae_exit函数 [POSITIVE]

**位置**: hisi_dae_common.c:369-391

**正面代码**:
```c
void dae_exit(struct wd_alg_driver *drv)
{
    struct wd_ctx_config_internal *config;
    struct hisi_dae_ctx *priv;
    handle_t h_qp;
    __u32 i;

    if (!drv || !drv->priv)  // 正确检查drv和drv->priv
        return;

    priv = (struct hisi_dae_ctx *)drv->priv;
    config = &priv->config;
    for (i = 0; i < config->ctx_num; i++) {
        h_qp = (handle_t)wd_ctx_get_priv(config->ctxs[i].ctx);
        if (h_qp) {  // 检查h_qp是否为NULL
            dae_uninit_qp_priv(h_qp);
            hisi_qm_free_qp(h_qp);
        }
    }
    // ...
}
```

**分析**:
- 正确检查了drv和drv->priv是否为NULL
- 在释放前检查h_qp是否为NULL
- 正确的资源管理流程

---

## drv/hisi_dae_common.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 0 |
| MEDIUM | 5 |
| LOW | 2 |
| POSITIVE | 2 |

**重点问题**:
- get_free_ext_addr/put_ext_addr函数缺少NULL检查(MEDIUM)
- dae_uninit_qp_priv/dae_init_qp_priv函数缺少qp检查(MEDIUM)
- dae_get_usage函数缺少参数检查(MEDIUM)

**正面发现**:
- dae_init/dae_exit函数正确实现了参数检查和资源管理

**建议修复顺序**:
1. 为get_free_ext_addr和put_ext_addr添加防御性NULL检查
2. 修复dae_uninit_qp_priv和dae_init_qp_priv的qp NULL检查
3. 完善dae_get_usage函数的参数检查
4. 为dae_hash_table_init添加hw_table/rehash_table检查

---

## 检视进度更新

**已完成**: 18/94 文件 (19.1%)

**下次检视**: drv/hisi_dae_join_gather.c (DAE join/gather驱动)

---

## 累计问题统计更新

| 等级 | 数量 |
|------|------|
| HIGH | 31 |
| MEDIUM | 58 (+5) |
| LOW | 26 (+2) |
| **总计** | **115** |