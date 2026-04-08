# UADK代码检视报告 - 第36轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第36轮
- **规则来源**: E:/work_space/加速器用户态学习/代码检视/review_rules
- **使用Skills**: cpp-memory-safety, cpp-concurrency-userspace, cpp-input-validation

---

## 本轮检视文件

### 已检视文件
1-35. 前序文件 (见review_progress.md)
36. v1/wd_util.c (本轮) - 旧版本工具函数，232行
37. v1/wd_adapter.c (本轮) - 旧版本驱动适配器，279行

### 待检视文件
- v1/目录下其他源文件(wd_cipher.c, wd_digest.c, wd_aead.c等)

---

## v1/wd_util.c 检视结果

### 1. wd_spinlock函数参数未检查NULL [LOW]

**位置**: v1/wd_util.c:27-41

**问题类型**: 参数验证 (cpp-input-validation)

**代码**:
```c
void wd_spinlock(struct wd_lock *lock)
{
    int val = 0;

    if (__atomic_compare_exchange_n(&lock->lock, &val, 1, 1,
                          __ATOMIC_ACQUIRE, __ATOMIC_RELAXED))
        return;

    do {
        do {
            val = __atomic_load_n(&lock->lock, __ATOMIC_RELAXED);
        } while (val != 0);
    } while (!__atomic_compare_exchange_n(&lock->lock, &val, 1, 1,
                          __ATOMIC_ACQUIRE, __ATOMIC_RELAXED));
}
```

**分析**:
- lock参数未检查是否为NULL
- 如果lock为NULL会导致崩溃
- 建议添加防御性检查或使用断言

---

### 2. drv_iova_map函数参数未检查NULL [MEDIUM]

**位置**: v1/wd_util.c:77-85

**问题类型**: 参数验证 (cpp-input-validation)

**代码**:
```c
void *drv_iova_map(struct wd_queue *q, void *va, size_t sz)
{
    struct q_info *qinfo = q->qinfo;

    if (qinfo->br.iova_map)
        return (void *)qinfo->br.iova_map(qinfo->br.usr, va, sz);
    else
        return wd_iova_map(q, va, sz);
}
```

**分析**:
- q参数未检查是否为NULL
- va参数未检查是否为NULL
- 如果q为NULL，q->qinfo会崩溃
- 如果qinfo为NULL，qinfo->br.iova_map访问会崩溃

---

### 3. wd_alloc_id函数潜在的整数溢出 [LOW]

**位置**: v1/wd_util.c:97-113

**问题类型**: 边界检查 (cpp-integer-overflow)

**代码**:
```c
int wd_alloc_id(__u8 *buf, __u32 size, __u32 *id, __u32 last_id, __u32 id_max)
{
    __u32 idx = last_id;
    __u32 cnt = 0;

    while (__atomic_test_and_set(&buf[idx], __ATOMIC_ACQUIRE)) {
        idx++;
        cnt++;
        if (idx == id_max)
            idx = 0;
        if (cnt == id_max)
            return -WD_EBUSY;
    }

    *id = idx;
    return 0;
}
```

**分析**:
- buf参数未检查是否为NULL
- 未验证id参数是否为NULL
- 未验证last_id < id_max

---

### 4. wd_init_cookie_pool整数乘法溢出 [MEDIUM]

**位置**: v1/wd_util.c:125-142

**问题类型**: 整数溢出 (cpp-integer-overflow)

**代码**:
```c
int wd_init_cookie_pool(struct wd_cookie_pool *pool,
            __u32 cookies_size, __u32 cookies_num)
{
    __u64 total_size = cookies_size * cookies_num;

    pool->cookies = calloc(1, total_size + cookies_num);
```

**分析**:
- cookies_size * cookies_num在赋值给total_size前可能溢出
- 虽然__u64可以容纳较大值，但两个__u32相乘结果仍为__u32
- 应该写为`(__u64)cookies_size * cookies_num`
- total_size + cookies_num也可能溢出

---

### 5. wd_get_cookies资源释放正确 [POSITIVE]

**位置**: v1/wd_util.c:183-198

**正面代码**:
```c
int wd_get_cookies(struct wd_cookie_pool *pool, void **cookies, __u32 num)
{
    __u32 i;

    for (i = 0; i < num; i++) {
        cookies[i] = get_cookie(pool);
        if (!cookies[i])
            goto put_cookies;
    }

    return 0;

put_cookies:
    wd_put_cookies(pool, cookies, i);
    return -WD_EBUSY;
}
```

**分析**:
- 分配失败时正确释放已分配的资源
- 使用goto实现正确的错误处理路径
- 遵循了良好的资源管理模式

---

### 6. wd_check_src_dst_ptr函数防御性检查 [POSITIVE]

**位置**: v1/wd_util.c:226-231

**正面代码**:
```c
int wd_check_src_dst_ptr(void *src, __u32 in_bytes, void *dst, __u32 out_bytes)
{
    if (unlikely((in_bytes && !src) || (out_bytes && !dst)))
        return -WD_EINVAL;
    return 0;
}
```

**分析**:
- 提供了通用的源/目标指针检查函数
- 使用unlikely宏优化分支预测
- 正确处理了有字节数但指针为NULL的情况

---

## v1/wd_adapter.c 检视结果

### 7. drv_open函数未检查q和qinfo是否为NULL [MEDIUM]

**位置**: v1/wd_adapter.c:94-121

**问题类型**: 参数验证 (cpp-input-validation)

**代码**:
```c
int drv_open(struct wd_queue *q)
{
    struct q_info *qinfo = q->qinfo;
    __u32 type_size = MAX_HW_TYPE;
    char *type;
    __u32 i;

    /* try to find another device if the user driver is not available */
    for (i = 0; i < type_size; i++) {
        if (!strcmp(qinfo->hw_type,
            hw_dio_tbl[i].hw_type)) {
            qinfo->hw_type_id = i;
            return hw_dio_tbl[qinfo->hw_type_id].open(q);
        }
    }
```

**分析**:
- q参数未检查是否为NULL
- qinfo未检查是否为NULL
- 直接使用qinfo->hw_type进行strcmp可能崩溃

---

### 8. drv_send/recv函数未检查参数 [MEDIUM]

**位置**: v1/wd_adapter.c:130-142

**问题类型**: 参数验证 (cpp-input-validation)

**代码**:
```c
int drv_send(struct wd_queue *q, void **req, __u32 num)
{
    struct q_info *qinfo = q->qinfo;

    return hw_dio_tbl[qinfo->hw_type_id].send(q, req, num);
}

int drv_recv(struct wd_queue *q, void **req, __u32 num)
{
    struct q_info *qinfo = q->qinfo;

    return hw_dio_tbl[qinfo->hw_type_id].recv(q, req, num);
}
```

**分析**:
- q、qinfo参数未检查NULL
- req参数未检查NULL
- hw_type_id可能越界访问hw_dio_tbl数组

---

### 9. drv_reserve_mem函数错误处理正确 [POSITIVE]

**位置**: v1/wd_adapter.c:187-244

**正面代码**:
```c
void *drv_reserve_mem(struct wd_queue *q, size_t size)
{
    ...
    ptr = wd_drv_mmap_qfr(q, WD_UACCE_QFRT_SS, tmp);
    if (ptr == MAP_FAILED) {
        WD_ERR("wd drv mmap fail!(err = %d)\n", errno);
        return NULL;
    }
    ...
    while (ret > 0) {
        ...
        rgn = malloc(sizeof(*rgn));
        if (!rgn) {
            WD_ERR("alloc ss region fail!\n");
            goto err_out;
        }
        ...
    }

    return ptr;

err_out:
    drv_free_slice(q);
    drv_unmap_reserve_mem(q, ptr, tmp);

    return NULL;
}
```

**分析**:
- 错误处理路径完整
- malloc失败后正确清理资源
- 使用goto实现统一的错误处理

---

### 10. hw_dio_tbl数组访问边界 [INFO]

**位置**: v1/wd_adapter.c:92, 106

**代码**:
```c
#define MAX_HW_TYPE (sizeof(hw_dio_tbl) / sizeof(hw_dio_tbl[0]))

int drv_open(struct wd_queue *q)
{
    ...
    for (i = 0; i < type_size; i++) {
        ...
        qinfo->hw_type_id = i;
        return hw_dio_tbl[qinfo->hw_type_id].open(q);
    }
    ...
    return hw_dio_tbl[qinfo->hw_type_id].open(q);  // 可能越界
}
```

**分析**:
- 如果循环正常结束，i == MAX_HW_TYPE
- 然后执行type检查逻辑，设置qinfo->hw_type_id
- 但如果type检查后仍然超出范围，可能导致数组越界

---

## v1/wd_util.c & wd_adapter.c 检视总结

| 文件 | 行数 | 问题等级分布 |
|------|------|-------------|
| v1/wd_util.c | 232 | MEDIUM 2, LOW 2, POSITIVE 2 |
| v1/wd_adapter.c | 279 | MEDIUM 2, INFO 1, POSITIVE 1 |
| **总计** | **511** | **MEDIUM 4, LOW 2, INFO 1, POSITIVE 3** |

**重点问题**:
- drv_iova_map/unmap函数缺少参数NULL检查(MEDIUM)
- drv_open/send/recv函数缺少参数NULL检查(MEDIUM)
- wd_init_cookie_pool整数乘法溢出风险(MEDIUM)

**建议修复**:
1. 为所有drv_*函数添加q和qinfo参数NULL检查
2. 修复wd_init_cookie_pool中的整数乘法溢出
3. 为wd_alloc_id添加buf和id参数检查

---

## 检视进度更新

**已完成**: 37/94 文件 (39.4%)

**下次检视**: v1/目录下其他源文件(wd_cipher.c, wd_digest.c等)