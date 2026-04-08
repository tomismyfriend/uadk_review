# UADK代码检视报告 - 第13轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第13轮
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
13. drv/hisi_qm_udrv.c (本轮) - QM用户态驱动文件，1168行

### 待检视文件
- drv/hisi_udma.c, drv/isa_ce_sm3.c, drv/isa_ce_sm4.c
- drv/hisi_dae.c等drv目录其他文件
- include/目录下的所有头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## drv/hisi_qm_udrv.c 检视结果

### 1. 空指针解引用风险 - hisi_set_msg_id函数 [HIGH]

**位置**: hisi_qm_udrv.c:665-685

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
void hisi_set_msg_id(handle_t h_qp, __u32 *tag)
{
    static __thread __u64 rand_seed = 0x330eabcd;
    struct hisi_qp *qp = (struct hisi_qp *)h_qp;
    __u8 mode = qp->q_info.qp_mode;  // 未检查qp是否为NULL
    __u16 seeds[3] = {0};
    __u64 id;

    // ...
}
```

**证据链**:
1. 直接将h_qp转换为qp，未检查是否为NULL
2. 立即访问qp->q_info.qp_mode
3. 如果h_qp为NULL，会导致空指针解引用崩溃
4. 此函数被多个驱动send函数调用

**风险分析**:
- 这是一个底层辅助函数，被hisi_sec/hisi_hpre/hisi_comp等驱动调用
- 如果调用方传入无效句柄，会导致崩溃

**修复建议**:
```c
void hisi_set_msg_id(handle_t h_qp, __u32 *tag)
{
    static __thread __u64 rand_seed = 0x330eabcd;
    struct hisi_qp *qp = (struct hisi_qp *)h_qp;
    __u8 mode;
    __u16 seeds[3] = {0};
    __u64 id;

    if (!qp || !tag)
        return;

    mode = qp->q_info.qp_mode;
    // ...
}
```

---

### 2. 空指针解引用风险 - hisi_check_bd_id函数 [HIGH]

**位置**: hisi_qm_udrv.c:651-663

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
int hisi_check_bd_id(handle_t h_qp, __u32 mid, __u32 bid)
{
    struct hisi_qp *qp = (struct hisi_qp *)h_qp;
    __u8 mode = qp->q_info.qp_mode;  // 未检查qp是否为NULL

    if (mode == CTX_MODE_SYNC && mid != bid) {
        WD_DEV_ERR(qp->h_ctx, "failed to recv self bd, send id: %u, recv id: %u\n",
                    mid, bid);
        return -WD_EINVAL;
    }

    return 0;
}
```

**证据链**:
1. 直接将h_qp转换为qp，未检查是否为NULL
2. 立即访问qp->q_info.qp_mode
3. 如果h_qp为NULL，会导致崩溃

**风险分析**:
- 此函数被所有驱动recv函数调用
- 是关键的错误检查函数

---

### 3. 空指针解引用风险 - hisi_qm_get_free_sqe_num函数 [MEDIUM]

**位置**: hisi_qm_udrv.c:434-439

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
int hisi_qm_get_free_sqe_num(handle_t h_qp)
{
    struct hisi_qp *qp = (struct hisi_qp *)h_qp;

    return get_free_num(&qp->q_info);  // 未检查qp是否为NULL
}
```

**分析**:
- 未检查qp是否为NULL
- 如果h_qp无效，会导致崩溃

---

### 4. 空指针解引用风险 - hisi_qm_get_sglpool函数 [MEDIUM]

**位置**: hisi_qm_udrv.c:1031-1045

**问题类型**: 空指针解引用 (cpp-memory-safety)

**代码**:
```c
handle_t hisi_qm_get_sglpool(handle_t h_qp, struct wd_mm_ops *mm_ops)
{
    struct hisi_qp *qp = (struct hisi_qp *)h_qp;

    if (mm_ops && !mm_ops->sva_mode) {
        pthread_spin_lock(&qp->q_info.sgl_lock);  // 未检查qp是否为NULL
        // ...
    }

    return qp->h_sgl_pool;  // 同样风险
}
```

**分析**:
- 未检查qp是否为NULL
- 直接访问qp->q_info和qp->h_sgl_pool

---

### 5. hisi_qm_sgl_copy函数参数检查不完整 [MEDIUM]

**位置**: hisi_qm_udrv.c:1114-1154

**问题类型**: 参数检查 (cpp-input-validation)

**代码**:
```c
void hisi_qm_sgl_copy(void *pbuff, void *hw_sgl, __u32 offset, __u32 size,
              __u8 direct)
{
    struct hisi_sgl *tmp = hw_sgl;
    int begin_sge = 0, i;
    __u32 sge_offset = 0;
    __u64 len = 0;

    if (!pbuff || !size || !tmp)
        return;
    // ...
}
```

**分析**:
- 检查了pbuff、size、tmp(hw_sgl)
- 但未检查direct参数是否合法(COPY_SGL_TO_PBUFF或其他值)
- 如果direct既不是COPY_SGL_TO_PBUFF也不是其他合法值，可能导致未定义行为

---

### 6. hisi_qm_create_sgl函数参数检查 [LOW]

**位置**: hisi_qm_udrv.c:687-703

**问题类型**: 参数检查 (cpp-input-validation)

**代码**:
```c
static void *hisi_qm_create_sgl(__u32 sge_num, struct wd_mm_ops *mm_ops)
{
    void *sgl;
    int size;

    size = sizeof(struct hisi_sgl) +
            sge_num * (sizeof(struct hisi_sge)) + HISI_SGL_ALIGE;
    if (mm_ops)
        sgl = mm_ops->alloc(mm_ops->usr, size);  // 未检查mm_ops->alloc是否为NULL
    else
        sgl = calloc(1, size);

    if (!sgl)
        WD_ERR("failed to alloc memory for the hisi qm sgl!\n");

    return sgl;
}
```

**分析**:
- 未检查mm_ops->alloc是否为NULL
- 如果mm_ops非空但mm_ops->alloc为空，会崩溃

---

### 7. hisi_qm_free_sglpool函数资源检查 [LOW]

**位置**: hisi_qm_udrv.c:731-754

**问题类型**: 资源管理 (cpp-resource-management)

**代码**:
```c
static void hisi_qm_free_sglpool(struct hisi_sgl_pool *pool)
{
    __u32 i;

    if (pool->sgl) {
        if (pool->mm_ops && !pool->mm_ops->sva_mode) {
            for (i = 0; i < pool->sgl_num; i++)
                pool->mm_ops->free(pool->mm_ops->usr, pool->sgl[i]);  // 未检查free
        } else {
            for (i = 0; i < pool->sgl_num; i++)
                free(pool->sgl[i]);
        }

        free(pool->sgl);
    }
    // ...
}
```

**分析**:
- 未检查pool是否为NULL
- 未检查pool->mm_ops->free是否为NULL

---

## drv/hisi_qm_udrv.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 2 |
| MEDIUM | 3 |
| LOW | 2 |

**重点问题**:
- hisi_set_msg_id/hisi_check_bd_id函数空指针检查缺失(HIGH)
- 这些函数被所有驱动send/recv函数调用，影响范围广

**建议修复顺序**:
1. 修复hisi_set_msg_id和hisi_check_bd_id的NULL检查
2. 完善hisi_qm_get_free_sqe_num和hisi_qm_get_sglpool的参数检查
3. 添加sgl相关函数的完整参数验证

---

## 检视进度更新

**已完成**: 13/94 文件 (13.8%)

**下次检视**: drv/isa_ce_sm3.c (SM3指令集实现)