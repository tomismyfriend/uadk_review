# UADK代码检视报告 - 第57轮

## 检视信息
- **检视文件**: v1/drv/hisi_zip_udrv.c
- **检视时间**: 2026-04-08
- **文件行数**: 1107行
- **检视规则**: cpp-memory-safety, cpp-resource-management, cpp-input-validation

## 检视摘要
| 等级 | 数量 |
|------|------|
| HIGH | 2 |
| MEDIUM | 5 |
| LOW | 3 |
| INFO | 2 |
| POSITIVE | 4 |

---

## 问题详情

### 1. [tag未检查NULL] - fill_zip_sqe_hw_info_lz77_zstd函数 - [Severity: HIGH]

**Location**: `v1/drv/hisi_zip_udrv.c:693-694`

**Dangerous Code**:
```c
static void fill_zip_sqe_hw_info_lz77_zstd(void *ssqe, struct wcrypto_comp_msg *msg)
{
	struct wcrypto_comp_tag *tag = (void *)(uintptr_t)msg->udata;
	struct wcrypto_lz77_zstd_format *format = tag->priv;  // tag未检查NULL
	...
}
```

**Risk Analysis**:
1. msg->udata转换为tag指针，未检查是否为NULL
2. 直接访问tag->priv，如果tag为NULL会崩溃
3. format也未检查NULL，后续直接使用

**Fix Suggestion**:
```c
struct wcrypto_comp_tag *tag = (void *)(uintptr_t)msg->udata;
if (!tag || !tag->priv)
	return;
struct wcrypto_lz77_zstd_format *format = tag->priv;
```

---

### 2. [tag未检查NULL] - fill_priv_lz77_zstd函数 - [Severity: HIGH]

**Location**: `v1/drv/hisi_zip_udrv.c:819-820`

**Dangerous Code**:
```c
static void fill_priv_lz77_zstd(void *ssqe, struct wcrypto_comp_msg *recv_msg)
{
	struct wcrypto_comp_tag *tag = (void *)(uintptr_t)recv_msg->udata;
	struct wcrypto_lz77_zstd_format *format = tag->priv;  // tag未检查NULL
	...
}
```

**Risk Analysis**:
同上，tag未检查NULL直接访问tag->priv。

---

### 3. [info->q未检查NULL] - 多个fill/parse函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_zip_udrv.c:301, 398, 755, 862, 1030, 1079`

**Code**:
```c
int qm_fill_zip_sqe(void *smsg, struct qm_queue_info *info, __u16 i)
{
	...
	struct wd_queue *q = info->q;  // info和info->q未检查NULL
	...
}
```

**Risk Analysis**:
多个fill和parse函数直接访问info->q，未检查info或info->q是否为NULL。

---

### 4. [ops数组索引检查不完整] - qm_fill_zip_sqe_v3函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_zip_udrv.c:762-800`

**Code**:
```c
if (unlikely(msg->alg_type >= get_arrsize(ops))) {
	WD_ERR("The algorithm is invalid!\n");
	return -WD_EINVAL;
}

ret = ops[msg->alg_type].fill_sqe_alg(sqe, msg);  // 函数指针未检查NULL
...
ret = ops[msg->alg_type].fill_sqe_window_size(sqe, msg);
...
ret = ops[msg->alg_type].fill_sqe_addr(sqe, msg, q);
...
ret = ops[msg->alg_type].fill_sqe_buffer_size(sqe, msg);
```

**Risk Analysis**:
1. 检查了alg_type是否越界
2. 但未检查ops[msg->alg_type]中的函数指针是否为NULL
3. ops数组中某些元素可能某些函数指针为NULL

---

### 5. [check_huffman_block_integrity返回值处理不当] - qm_parse_zip_sqe_v3函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_zip_udrv.c:931-937`

**Code**:
```c
ret = check_huffman_block_integrity(cache_data, bit_cnt);
if (ret < 0) {
	WD_ERR("invalid: unable to parse data!\n");
	recv_msg->status = WD_IN_EPARA;
} else if (ret) {
	recv_msg->status = WCRYPTO_DECOMP_BLK_NOSTART;
}
```

**Risk Analysis**:
check_huffman_block_integrity函数可能返回错误，虽然检查了ret < 0，但如果该函数内部有崩溃风险（如访问无效内存），可能导致整个程序崩溃。

---

### 6. [msg->ctx_buf偏移访问风险] - 多个函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_zip_udrv.c:347-351, 419-423, 682-685`

**Code**:
```c
if (msg->ctx_buf) {
	sqe->ctx_dw0 = *(__u32 *)msg->ctx_buf;
	sqe->ctx_dw1 = *(__u32 *)(msg->ctx_buf + CTX_PRIV1_OFFSET);  // 偏移4
	sqe->ctx_dw2 = *(__u32 *)(msg->ctx_buf + CTX_PRIV2_OFFSET);  // 偏移8
}
```

**Risk Analysis**:
1. 检查了ctx_buf是否为NULL
2. 但未检查ctx_buf指向的缓冲区是否足够大
3. 如果ctx_buf分配的内存小于CTX_PRIV2_OFFSET + 4（12字节），会导致越界访问

---

### 7. [整数运算可能溢出] - qm_fill_zip_cipher_sqe_with_priv函数 - [Severity: LOW]

**Location**: `v1/drv/hisi_zip_udrv.c:984-986`

**Code**:
```c
sqe->input_data_length =
	(udata->block_size + dif_size + pad_size) * udata->gran_num;
```

**Risk Analysis**:
三个__u32相加后乘以gran_num可能溢出32位整数。

---

### 8. [wd_queue指针解引用风险] - qm_fill_zip_sqe_get_phy_address函数 - [Severity: LOW]

**Location**: `v1/drv/hisi_zip_udrv.c:247, 265, 272`

**Code**:
```c
phy_ctxbuf = (uintptr_t)drv_iova_map(q, msg->ctx_buf, MAX_CTX_RSV_SIZE);
```

**Risk Analysis**:
q指针来自info->q，未在此函数内检查NULL。如果drv_iova_map内部也不检查，会导致崩溃。

---

### 9. [信号处理安全性] - zip_err_print_time_start函数 - [Severity: LOW]

**Location**: `v1/drv/hisi_zip_udrv.c:145-150`

**Code**:
```c
static void zip_err_print_time_start(void)
{
	g_err_print_enable = 0;
	signal(SIGALRM, zip_err_print_alarm_end);
	alarm(PRINT_TIME_INTERVAL);
}
```

**Risk Analysis**:
1. 在多线程环境中使用signal()可能不安全
2. 应该使用sigaction()代替signal()
3. g_err_print_enable是全局变量，多线程访问可能有竞争条件

---

### 10. [信息性发现] - ZIP压缩算法支持 - [Severity: INFO]

**Location**: `v1/drv/hisi_zip_udrv.c:719-748`

**Code**:
```c
static struct zip_fill_sqe_ops ops[] = { {
		.alg_type = "zlib",
		...
	}, {
		.alg_type = "gzip",
		...
	}, {
		.alg_type = "raw_deflate",
		...
	}, {
		.alg_type = "lz77_zstd",
		...
	},
};
```

**Analysis**:
支持四种压缩算法：
1. **zlib**: 标准zlib格式
2. **gzip**: GZIP格式
3. **raw_deflate**: 原始DEFLATE流
4. **lz77_zstd**: LZ77预处理 + ZSTD压缩

---

### 11. [信息性发现] - 内部存储缓冲区机制 - [Severity: INFO]

**Location**: `v1/drv/hisi_zip_udrv.c:106-117, 164-218`

**Analysis**:
实现了内部存储缓冲区机制：
1. 当用户输出缓冲区小于200字节时使用
2. 硬件输出到内部缓冲区，然后软件拷贝到用户缓冲区
3. 支持pending_out跳过硬件处理
4. 解决硬件最小输出缓冲区要求问题

---

### 12. [正面发现] - alg_type边界检查 - [Severity: POSITIVE]

**Location**: `v1/drv/hisi_zip_udrv.c:762-765`

**Code**:
```c
if (unlikely(msg->alg_type >= get_arrsize(ops))) {
	WD_ERR("The algorithm is invalid!\n");
	return -WD_EINVAL;
}
```

**Analysis**:
正确检查alg_type是否超出ops数组范围。

---

### 13. [正面发现] - drv_iova_map返回值检查 - [Severity: POSITIVE]

**Location**: `v1/drv/hisi_zip_udrv.c:249-251, 266-269, 273-276`

**Code**:
```c
phy_in = (uintptr_t)drv_iova_map(q, msg->src, msg->in_size);
if (!phy_in) {
	WD_ERR("Get zip in buf dma address fail!\n");
	goto unmap_phy_ctx;
}
```

**Analysis**:
所有DMA映射调用都正确检查了返回值。

---

### 14. [正面发现] - req_cache NULL检查 - [Severity: POSITIVE]

**Location**: `v1/drv/hisi_zip_udrv.c:401-404, 868-871, 1084-1087`

**Code**:
```c
if (unlikely(!recv_msg)) {
	WD_ERR("info->req_cache is null at index:%hu\n", i);
	return 0;
}
```

**Analysis**:
parse函数正确检查req_cache索引处的指针。

---

### 15. [正面发现] - 错误处理路径资源释放 - [Severity: POSITIVE]

**Location**: `v1/drv/hisi_zip_udrv.c:285-292`

**Code**:
```c
unmap_phy_in:
	drv_iova_unmap(q, msg->src, (void *)phy_in, msg->in_size);
unmap_phy_ctx:
	if (msg->stream_mode == WCRYPTO_COMP_STATEFUL)
		drv_iova_unmap(q, msg->ctx_buf, (void *)phy_ctxbuf,
			       MAX_CTX_RSV_SIZE);
	return -WD_ENOMEM;
```

**Analysis**:
错误路径正确释放已映射的资源。

---

## 总结

### 问题分布
| 问题类型 | 数量 |
|----------|------|
| 空指针检查缺失 | 2 |
| 数组/偏移访问风险 | 2 |
| 函数指针未检查NULL | 1 |
| 参数验证不完整 | 2 |
| 整数运算风险 | 1 |

### 模块特点

v1/drv/hisi_zip_udrv.c实现了ZIP压缩驱动：
1. 支持zlib/gzip/raw_deflate/lz77_zstd四种算法
2. 实现内部存储缓冲区解决硬件限制
3. 支持流式压缩和状态管理
4. 支持ZIP引擎的XTS加密

### 主要风险

**最大风险**：tag未检查NULL
- fill_zip_sqe_hw_info_lz77_zstd和fill_priv_lz77_zstd函数直接访问tag->priv
- 如果tag为NULL会崩溃
- 影响lz77_zstd算法操作

### 与根目录drv/hisi_comp.c对比

| 对比项 | v1/drv/hisi_zip_udrv.c | drv/hisi_comp.c |
|--------|------------------------|-----------------|
| 文件行数 | 1107 | 约1500行 |
| 算法支持 | 4种 | 更多 |
| 内部缓冲区 | 有 | 无 |
| V3 SQE | 支持 | 支持 |

### 建议优先修复顺序
1. tag NULL检查（HIGH）
2. info->q NULL检查（MEDIUM）
3. ops函数指针NULL检查（MEDIUM）
4. ctx_buf偏移访问边界检查（MEDIUM）