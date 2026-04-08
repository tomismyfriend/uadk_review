# UADK代码检视报告 - 第16轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第16轮
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
16. drv/hisi_udma.c (本轮) - UDMA驱动文件，600行

### 待检视文件
- drv/hisi_dae.c, drv/hisi_dae_common.c, drv/hisi_dae_join_gather.c
- drv/hisi_comp_huf.c, drv/hash_mb/hash_mb.c
- include/目录下的所有头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## drv/hisi_udma.c 检视结果

### 1. 空指针解引用风险 - udma_send函数 [HIGH]

**位置**: hisi_udma.c:293-337

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static int udma_send(struct wd_alg_driver *drv, handle_t ctx, void *udma_msg)
{
    handle_t h_qp = (handle_t)wd_ctx_get_priv(ctx);
    struct hisi_qp *qp = (struct hisi_qp *)h_qp;
    struct udma_internal_addr *inter_addr = qp->priv;  // 未检查qp是否为NULL
    struct udma_addr_array *addr_array = NULL;
    struct wd_udma_msg *msg = udma_msg;
    struct udma_sqe sqe = {0};
    __u16 send_cnt = 0;
    int idx = 0;
    int ret;

    ret = check_udma_param(msg);
    if (unlikely(ret))
        return ret;

    if (msg->addr_num > 1 || msg->dst->data_size > UDMA_S_MAX_ADDR_SIZE) {
        idx = get_free_inter_addr(inter_addr);  // 如果inter_addr为NULL，崩溃
        // ...
    }
    // ...
    hisi_set_msg_id(h_qp, &msg->tag);  // 如果h_qp为NULL，崩溃
    // ...
}
```

**证据链**:
1. wd_ctx_get_priv(ctx)可能返回NULL
2. 直接将返回值转换为qp，未检查是否为NULL
3. 立即访问qp->priv获取inter_addr
4. 调用hisi_set_msg_id时传入h_qp(可能为NULL)

**风险分析**:
- 与hisi_sec.c、hisi_hpre.c、hisi_comp.c中的问题模式相同
- 驱动send函数是关键路径，缺少防御性编程

**修复建议**:
```c
static int udma_send(struct wd_alg_driver *drv, handle_t ctx, void *udma_msg)
{
    struct hisi_qp *qp = wd_ctx_get_priv(ctx);
    struct udma_internal_addr *inter_addr;
    handle_t h_qp;
    // ...

    if (!qp) {
        WD_ERR("invalid: qp is NULL!\n");
        return -WD_EINVAL;
    }
    h_qp = (handle_t)qp;
    inter_addr = qp->priv;
    // ...
}
```

---

### 2. 空指针解引用风险 - udma_recv函数 [HIGH]

**位置**: hisi_udma.c:345-392

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static int udma_recv(struct wd_alg_driver *drv, handle_t ctx, void *udma_msg)
{
    handle_t h_qp = (handle_t)wd_ctx_get_priv(ctx);
    struct hisi_qp *qp = (struct hisi_qp *)h_qp;
    struct udma_internal_addr *inter_addr = qp->priv;  // 未检查qp是否为NULL
    struct wd_udma_msg *msg = udma_msg;
    struct wd_udma_msg *temp_msg = msg;
    struct udma_sqe sqe = {0};
    __u16 recv_cnt = 0;
    int ret;

    ret = hisi_qm_recv(h_qp, &sqe, 1, &recv_cnt);  // 如果h_qp为NULL，崩溃
    if (ret)
        return ret;

    ret = hisi_check_bd_id(h_qp, msg->tag, sqe.low_tag);  // 同样风险
    // ...
    if (qp->q_info.qp_mode == CTX_MODE_ASYNC) {  // 直接访问qp成员
        // ...
    }
}
```

**分析**:
- 与udma_send相同的问题模式
- send和recv函数都缺少qp的NULL检查

---

### 3. 空指针解引用风险 - udma_init_qp_priv函数 [MEDIUM]

**位置**: hisi_udma.c:412-443

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
static int udma_init_qp_priv(handle_t h_qp)
{
    struct hisi_qp *qp = (struct hisi_qp *)h_qp;
    __u16 sq_depth = qp->q_info.sq_depth;  // 未检查qp是否为NULL
    struct udma_internal_addr *inter_addr;
    int ret = -WD_ENOMEM;

    inter_addr = calloc(1, sizeof(struct udma_internal_addr));
    // ...
}
```

**分析**:
- 未检查qp是否为NULL
- 直接访问qp->q_info.sq_depth

---

### 4. 空指针解引用风险 - get_free_inter_addr函数 [MEDIUM]

**位置**: hisi_udma.c:106-127

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
static int get_free_inter_addr(struct udma_internal_addr *inter_addr)
{
    __u16 addr_count = inter_addr->addr_count;  // 未检查inter_addr是否为NULL
    __u16 idx = inter_addr->tail;
    __u16 cnt = 0;

    if (unlikely(!addr_count)) {
        WD_ERR("invalid: internal addr count is 0!\n");
        return -WD_EINVAL;
    }

    while (__atomic_test_and_set(&inter_addr->addr_status[idx], __ATOMIC_ACQUIRE)) {
        idx = (idx + 1) % addr_count;
        cnt++;
        if (cnt == addr_count)
            return -WD_EBUSY;
    }

    inter_addr->tail = (idx + 1) % addr_count;

    return idx;
}
```

**分析**:
- 检查了addr_count是否为0，但未检查inter_addr是否为NULL
- 如果inter_addr为NULL会导致崩溃
- 建议在函数开头添加NULL检查

---

### 5. 空指针解引用风险 - fill_single_addr_info函数 [MEDIUM]

**位置**: hisi_udma.c:260-266

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
static void fill_single_addr_info(struct udma_sqe *sqe, struct wd_udma_msg *msg)
{
    if (msg->op_type == WD_UDMA_MEMCPY)
        sqe->addr_array = (__u64)(uintptr_t)msg->src->addr;  // 未检查msg->src是否为NULL
    sqe->dst_addr = (__u64)(uintptr_t)msg->dst->addr;  // 未检查msg->dst是否为NULL
    sqe->data_size = msg->dst->data_size;
}
```

**分析**:
- check_udma_param检查了msg->dst->data_size的大小
- 但未检查msg->src和msg->dst指针是否为NULL
- 如果这些指针为NULL会导致空指针解引用

---

### 6. 空指针解引用风险 - udma_get_usage函数 [MEDIUM]

**位置**: hisi_udma.c:522-558

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
static int udma_get_usage(void *param)
{
    struct hisi_dev_usage *udma_usage = (struct hisi_dev_usage *)param;
    struct wd_alg_driver *drv = udma_usage->drv;  // 未检查udma_usage是否为NULL
    struct wd_ctx_config_internal *config;
    struct hisi_udma_ctx *priv;
    char *ctx_dev_name;
    handle_t ctx = 0;
    handle_t qp = 0;
    __u32 i;

    if (udma_usage->alg_op_type >= drv->op_type_num) {  // 直接访问成员
        // ...
    }

    priv = (struct hisi_udma_ctx *)drv->priv;  // 未检查drv是否为NULL
    // ...
}
```

**分析**:
- 未检查param是否为NULL
- 直接将param转换为udma_usage并访问成员
- 未检查drv是否为NULL

---

### 7. 数组越界风险 - fill_long_size_memcpy_info函数 [LOW]

**位置**: hisi_udma.c:174-193

**问题类型**: 数组越界 (cpp-memory-safety)

**代码**:
```c
static void fill_long_size_memcpy_info(struct udma_sqe *sqe, struct wd_udma_msg *msg,
                       struct udma_addr_array *addr_array)
{
    __u32 addr_num = 0;
    __u64 count;

    for (count = 0; count < msg->src->data_size; count += UDMA_S_MAX_ADDR_SIZE) {
        addr_array->src_addr[addr_num].addr = (__u64)(uintptr_t)msg->src->addr + count;
        addr_array->dst_addr[addr_num].addr = (__u64)(uintptr_t)msg->dst->addr + count;
        // ...
        addr_num++;  // 无边界检查
    }
    sqe->dw1 |= (addr_num - 1) << UDMA_ADDR_NUM_SHIFT;
}
```

**分析**:
- addr_num在循环中递增，但未检查是否超过UDMA_MAX_ADDR_NUM(64)
- check_udma_param在第154-158行检查了msg->dst->data_size <= UDMA_M_MAX_ADDR_SIZE
- UDMA_M_MAX_ADDR_SIZE = 1073741760，除以UDMA_S_MAX_ADDR_SIZE(16777215)约64个地址
- 实际上addr_num最多约64，刚好等于UDMA_MAX_ADDR_NUM
- 建议添加显式的边界检查确保安全

---

### 8. fill_long_size_memset_info函数同样问题 [LOW]

**位置**: hisi_udma.c:195-211

**问题类型**: 数组越界 (cpp-memory-safety)

**代码**:
```c
static void fill_long_size_memset_info(struct udma_sqe *sqe, struct wd_udma_msg *msg,
                       struct udma_addr_array *addr_array)
{
    __u32 addr_num = 0;
    __u64 count;

    for (count = 0; count < msg->dst->data_size; count += UDMA_S_MAX_ADDR_SIZE) {
        addr_array->dst_addr[addr_num].addr = (__u64)(uintptr_t)msg->dst->addr + count;
        // ...
        addr_num++;  // 无边界检查
    }
    sqe->dw1 |= (addr_num - 1) << UDMA_ADDR_NUM_SHIFT;
}
```

**分析**:
- 与fill_long_size_memcpy_info相同的问题
- 虽然check_udma_param限制了data_size，但建议添加显式检查

---

### 9. put_inter_addr函数参数检查 [LOW]

**位置**: hisi_udma.c:129-132

**问题类型**: 参数检查 (cpp-input-validation)

**代码**:
```c
static void put_inter_addr(struct udma_internal_addr *inter_addr, int idx)
{
    __atomic_clear(&inter_addr->addr_status[idx], __ATOMIC_RELEASE);  // 未检查参数
}
```

**分析**:
- 未检查inter_addr是否为NULL
- 未检查idx是否在合法范围内
- 建议添加防御性检查

---

## drv/hisi_udma.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 2 |
| MEDIUM | 4 |
| LOW | 3 |

**重点问题**:
- udma_send/recv函数空指针检查缺失(HIGH)
- 与其他驱动文件(hisi_sec/hisi_hpre/hisi_comp)相同的问题模式

**建议修复顺序**:
1. 修复udma_send和udma_recv函数的qp NULL检查
2. 完善udma_init_qp_priv函数的参数检查
3. 添加get_free_inter_addr等辅助函数的防御性检查
4. 完善fill_single_addr_info函数的指针检查

---

## 检视进度更新

**已完成**: 16/94 文件 (17.0%)

**下次检视**: drv/hisi_dae.c (DAE驱动文件)

---

## 累计问题统计更新

| 等级 | 数量 |
|------|------|
| HIGH | 29 (+2) |
| MEDIUM | 47 (+4) |
| LOW | 23 (+3) |
| **总计** | **99** |