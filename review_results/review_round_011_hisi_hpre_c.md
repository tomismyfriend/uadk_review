# UADK代码检视报告 - 第11轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第11轮
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
11. drv/hisi_hpre.c (本轮) - HPRE驱动文件，约2900行

### 待检视文件
- drv/hisi_comp.c, drv/hisi_qm_udrv.c, drv/hisi_udma.c
- drv/isa_ce_sm3.c, drv/isa_ce_sm4.c
- drv/hisi_dae.c等drv目录其他文件
- include/目录下的所有头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## drv/hisi_hpre.c 检视结果

### 1. 空指针解引用风险 - rsa_send/rsa_recv函数 [HIGH]

**位置**: hisi_hpre.c:762, 834

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static int rsa_send(struct wd_alg_driver *drv, handle_t ctx, void *rsa_msg)
{
    handle_t h_qp = (handle_t)wd_ctx_get_priv(ctx);
    struct wd_rsa_msg *msg = rsa_msg;
    // ...

    hisi_set_msg_id(h_qp, &msg->tag);  // 未检查h_qp是否为NULL
    // ...
    ret = hisi_qm_send(h_qp, &hw_msg, 1, &send_cnt);  // 可能崩溃
}

static int rsa_recv(struct wd_alg_driver *drv, handle_t ctx, void *rsa_msg)
{
    handle_t h_qp = (handle_t)wd_ctx_get_priv(ctx);
    struct hisi_qp *qp = (struct hisi_qp *)h_qp;
    // ...

    ret = hisi_qm_recv(h_qp, &hw_msg, 1, &recv_cnt);  // 未检查qp
```

**证据链**:
1. wd_ctx_get_priv(ctx)可能返回NULL
2. 直接使用h_qp调用hisi_set_msg_id、hisi_qm_send等函数
3. 未进行NULL检查

**风险分析**:
- 如果ctx无效或未正确初始化，会导致空指针解引用
- 与hisi_sec.c中的问题相同

**修复建议**:
```c
static int rsa_send(struct wd_alg_driver *drv, handle_t ctx, void *rsa_msg)
{
    handle_t h_qp = (handle_t)wd_ctx_get_priv(ctx);
    struct wd_rsa_msg *msg = rsa_msg;
    // ...

    if (!h_qp) {
        WD_ERR("invalid: qp is NULL!\n");
        return -WD_EINVAL;
    }
    // ...
}
```

---

### 2. 空指针解引用风险 - dh_send/dh_recv函数 [HIGH]

**位置**: hisi_hpre.c:954, 1023

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static int dh_send(struct wd_alg_driver *drv, handle_t ctx, void *dh_msg)
{
    handle_t h_qp = (handle_t)wd_ctx_get_priv(ctx);
    // ...
    hisi_set_msg_id(h_qp, &msg->tag);  // 未检查h_qp是否为NULL
}

static int dh_recv(struct wd_alg_driver *drv, handle_t ctx, void *dh_msg)
{
    handle_t h_qp = (handle_t)wd_ctx_get_priv(ctx);
    struct hisi_qp *qp = (struct hisi_qp *)h_qp;
    // ...
    ret = hisi_qm_recv(h_qp, &hw_msg, 1, &recv_cnt);  // 未检查qp
```

**分析**:
- 与rsa_send/rsa_recv相同的问题
- 6个send/recv函数都存在此问题

---

### 3. 空指针解引用风险 - ecc_send/ecc_general_send函数 [HIGH]

**位置**: hisi_hpre.c:2021, 2152

**问题类型**: 空指针解引用 (cpp-memory-safety)

**危险代码**:
```c
static int ecc_general_send(handle_t ctx, struct wd_ecc_msg *msg,
                             struct map_info_cache *cache)
{
    handle_t h_qp = (handle_t)wd_ctx_get_priv(ctx);
    // ...
    return hisi_qm_send(h_qp, &hw_msg, 1, &send_cnt);  // 未检查h_qp
}

static int ecc_send(struct wd_alg_driver *drv, handle_t ctx, void *ecc_msg)
{
    handle_t h_qp = (handle_t)wd_ctx_get_priv(ctx);
    // ...
    hisi_set_msg_id(h_qp, &msg->tag);  // 未检查h_qp
}
```

**分析**:
- ecc_send、ecc_general_send、sm2_enc_send、sm2_dec_send都存在相同问题
- 需要统一添加NULL检查

---

### 4. unmap_addr_in_cache函数参数检查 [MEDIUM]

**位置**: hisi_hpre.c:151-162

**问题类型**: 参数检查 (cpp-input-validation)

**代码**:
```c
static void unmap_addr_in_cache(struct wd_mm_ops *mm_ops, struct map_info_cache *cache)
{
    size_t i;

    if (mm_ops->sva_mode)
        return;

    /* The cnt is guaranteed not to exceed MAP_PAIR_NUM_MAX within hpre. */
    for (i = 0; i < cache->cnt; i++)
        mm_ops->iova_unmap(mm_ops->usr, cache->pairs[i].addr,
                   (void *)cache->pairs[i].pa, cache->pairs[i].size);
}
```

**分析**:
- 未检查mm_ops是否为NULL
- 未检查mm_ops->iova_unmap是否为NULL
- 如果mm_ops为NULL，访问mm_ops->sva_mode会崩溃

**修复建议**:
```c
static void unmap_addr_in_cache(struct wd_mm_ops *mm_ops, struct map_info_cache *cache)
{
    size_t i;

    if (!mm_ops || mm_ops->sva_mode)
        return;

    if (!mm_ops->iova_unmap)
        return;

    for (i = 0; i < cache->cnt; i++)
        mm_ops->iova_unmap(mm_ops->usr, cache->pairs[i].addr,
                   (void *)cache->pairs[i].pa, cache->pairs[i].size);
}
```

---

### 5. select_addr_by_sva_mode函数参数检查 [MEDIUM]

**位置**: hisi_hpre.c:192-209

**问题类型**: 参数检查 (cpp-input-validation)

**代码**:
```c
static uintptr_t select_addr_by_sva_mode(struct wd_mm_ops *mm_ops, void *data,
                        size_t data_sz, struct map_info_cache *cache)
{
    uintptr_t addr;

    if (!mm_ops->sva_mode) {
        addr = (uintptr_t)mm_ops->iova_map(mm_ops->usr, data, data_sz);
        if (!addr) {
            WD_ERR("Failed to get mappped DMA address for hardware.\n");
            return 0;
        }
        add_map_info(cache, data, addr, data_sz);
    } else {
        addr = (uintptr_t)data;
    }

    return addr;
}
```

**分析**:
- 未检查mm_ops是否为NULL
- 未检查mm_ops->iova_map是否为NULL
- 如果mm_ops为NULL，访问mm_ops->sva_mode会崩溃

---

### 6. add_map_info函数边界检查缺失 [MEDIUM]

**位置**: hisi_hpre.c:142-149

**问题类型**: 缓冲区溢出 (cpp-memory-safety)

**代码**:
```c
static void add_map_info(struct map_info_cache *cache, void *addr, uintptr_t dma, size_t size)
{
    /* The cnt is guaranteed not to exceed MAP_PAIR_NUM_MAX within hpre. */
    cache->pairs[cache->cnt].addr = addr;
    cache->pairs[cache->cnt].pa = dma;
    cache->pairs[cache->cnt].size = size;
    cache->cnt++;
}
```

**分析**:
- 注释说明cnt不会超过MAP_PAIR_NUM_MAX(6)
- 但没有实际边界检查
- 如果调用方传入过多请求，可能导致越界写入

**修复建议**:
```c
static void add_map_info(struct map_info_cache *cache, void *addr, uintptr_t dma, size_t size)
{
    if (cache->cnt >= MAP_PAIR_NUM_MAX) {
        WD_ERR("map_info_cache is full!\n");
        return;
    }
    cache->pairs[cache->cnt].addr = addr;
    cache->pairs[cache->cnt].pa = dma;
    cache->pairs[cache->cnt].size = size;
    cache->cnt++;
}
```

---

### 7. sm2_kdf函数整数溢出风险 [MEDIUM]

**位置**: hisi_hpre.c:2363-2423

**问题类型**: 整数溢出 (cpp-integer-overflow)

**代码**:
```c
static int sm2_kdf(struct wd_dtb *out, struct wd_ecc_point *x2y2,
           __u64 mt_len, struct wd_hash_mt *hash)
{
    // ...
    x2y2_len = x2y2->x.dsize + x2y2->y.dsize;
    lens = x2y2_len + sizeof(ctr);
    // ...
    while (1) {
        // ...
        i++;
    }
    // ...
}
```

**分析**:
- i在循环中递增，可能导致整数溢出
- 如果mt_len很大，循环次数可能超过预期
- 建议添加循环次数限制

---

### 8. free_req函数资源检查 [LOW]

**位置**: hisi_hpre.c:1946-1954

**问题类型**: 资源管理 (cpp-resource-management)

**代码**:
```c
static void free_req(struct wd_ecc_msg *msg)
{
    struct wd_ecc_key *key = (void *)msg->key;

    msg->mm_ops->free(msg->mm_ops->usr, key->prikey->data);
    free(key);
    msg->mm_ops->free(msg->mm_ops->usr, msg->req.dst);
    free(msg);
}
```

**分析**:
- 未检查key是否为NULL
- 未检查key->prikey是否为NULL
- 未检查msg->mm_ops和msg->mm_ops->free是否为NULL

---

## drv/hisi_hpre.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 3 |
| MEDIUM | 4 |
| LOW | 1 |

**重点问题**:
- rsa_send/rsa_recv、dh_send/dh_recv、ecc_send等函数空指针检查缺失(HIGH)
- mm_ops回调函数未检查NULL(MEDIUM)

**建议修复顺序**:
1. 修复所有send/recv函数的qp NULL检查
2. 完善mm_ops回调函数的NULL检查
3. 添加边界检查和整数溢出保护

---

## 检视进度更新

**已完成**: 11/94 文件 (11.7%)

**下次检视**: drv/hisi_comp.c (压缩驱动文件)