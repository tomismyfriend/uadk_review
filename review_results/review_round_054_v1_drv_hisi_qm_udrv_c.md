# UADK代码检视报告 - 第54轮

## 检视信息
- **检视文件**: v1/drv/hisi_qm_udrv.c
- **检视时间**: 2026-04-08
- **文件行数**: 851行
- **检视规则**: cpp-memory-safety, cpp-resource-management, cpp-input-validation

## 检视摘要
| 等级 | 数量 |
|------|------|
| HIGH | 3 |
| MEDIUM | 5 |
| LOW | 3 |
| INFO | 2 |
| POSITIVE | 3 |

---

## 问题详情

### 1. [br->iova_map未检查NULL] - qm_hw_sgl_sge_init函数 - [Severity: HIGH]

**Location**: `v1/drv/hisi_qm_udrv.c:72-73`

**Dangerous Code**:
```c
static int qm_hw_sgl_sge_init(struct wd_sgl *sgl, struct hisi_sgl *hisi_sgl,
			      struct wd_mm_br *br, int num, __u32 buf_sz)
{
	void *buf;

	buf = wd_get_sge_buf(sgl, num);
	if (!buf)
		return -WD_EINVAL;

	hisi_sgl->sge_entries[num - 1].buf = (uintptr_t)br->iova_map(br->usr,
					buf, buf_sz);
```

**Risk Analysis**:
1. br指针未检查是否为NULL
2. br->iova_map函数指针未检查是否为NULL
3. 直接调用可能导致崩溃

**Fix Suggestion**:
```c
if (!br || !br->iova_map) {
	WD_ERR("br or br->iova_map is NULL!\n");
	return -WD_EINVAL;
}
hisi_sgl->sge_entries[num - 1].buf = (uintptr_t)br->iova_map(br->usr, buf, buf_sz);
```

---

### 2. [info->sqe_fill[qinfo->atype]未检查NULL] - qm_send函数 - [Severity: HIGH]

**Location**: `v1/drv/hisi_qm_udrv.c:633-634`

**Dangerous Code**:
```c
int qm_send(struct wd_queue *q, void **req, __u32 num)
{
	...
	for (i = 0; i < num; i++) {
		ret = info->sqe_fill[qinfo->atype](req[i], qinfo->priv,
				info->sq_tail_index);
```

**Risk Analysis**:
1. q未检查NULL，直接访问q->qinfo
2. info->sqe_fill是函数指针数组，qinfo->atype作为索引
3. sqe_fill函数指针可能未初始化或为NULL
4. 在hpre_alg_info_init/sec_alg_info_init/zip_alg_info_init中只设置了特定atype的回调，其他atype可能为NULL

**Fix Suggestion**:
```c
if (!q || !q->qinfo || !q->qinfo->priv)
	return -WD_EINVAL;
if (!info->sqe_fill || !info->sqe_fill[qinfo->atype])
	return -WD_EINVAL;
```

---

### 3. [info->sqe_parse[qinfo->atype]未检查NULL] - qm_recv函数 - [Severity: HIGH]

**Location**: `v1/drv/hisi_qm_udrv.c:739-741`

**Dangerous Code**:
```c
int qm_recv(struct wd_queue *q, void **resp, __u32 num)
{
	...
	ret = info->sqe_parse[qinfo->atype](sqe,
			(const struct qm_queue_info *)info,
			info->cq_head_index, (__u16)(uintptr_t)resp[i]);
```

**Risk Analysis**:
同qm_send函数，sqe_parse函数指针可能为NULL。

---

### 4. [br->alloc/free未检查NULL] - qm_hw_sgl_init/uninit函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_qm_udrv.c:121-124, 154, 178, 189`

**Dangerous Code**:
```c
// qm_hw_sgl_init
br = drv_get_br(pool);

hisi_sgl = br->alloc(br->usr, sizeof(struct hisi_sgl) +
		     sizeof(struct hisi_sge) * buf_num);
...
br->free(br->usr, hisi_sgl);

// qm_hw_sgl_uninit
br = drv_get_br(pool);
...
br->iova_unmap(br->usr, (void *)hisi_sgl->sge_entries[i].buf, buf, buf_sz);
...
br->free(br->usr, hisi_sgl);
```

**Risk Analysis**:
drv_get_br返回值未检查NULL，后续br->alloc/free/iova_unmap调用可能崩溃。

---

### 5. [drv_get_sgl_pri返回值未检查NULL] - qm_hw_sgl_merge函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_qm_udrv.c:194-195`

**Dangerous Code**:
```c
static int qm_hw_sgl_merge(void *pool, struct wd_sgl *dst_sgl, struct wd_sgl *src_sgl)
{
	struct hisi_sgl *d = drv_get_sgl_pri(dst_sgl);
	struct hisi_sgl *s = drv_get_sgl_pri(src_sgl);
```

**Risk Analysis**:
drv_get_sgl_pri返回void指针，未检查是否为NULL。如果sgl未初始化或priv为NULL，直接解引用会崩溃。

---

### 6. [strtoul未设置errno且未正确检查错误] - qm_set_queue_alg_info函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_qm_udrv.c:426-429`

**Dangerous Code**:
```c
ver = strtoul(qinfo->hw_type + HISI_VERSION_ID_SHIFT, NULL, 10);
if (!ver || ver == ULONG_MAX) {
	WD_ERR("failed to strtoul, ver = %lu!\n", ver);
	return ret;
}
```

**Risk Analysis**:
1. 调用strtoul前未设置errno=0
2. ULONG_MAX检查不足以判断错误，因为errno未检查
3. ver为0可能是合法值

**Fix Suggestion**:
```c
errno = 0;
ver = strtoul(qinfo->hw_type + HISI_VERSION_ID_SHIFT, NULL, 10);
if (errno || ver == ULONG_MAX || ver == 0) {
	WD_ERR("failed to strtoul, ver = %lu, errno = %d!\n", ver, errno);
	return ret;
}
```

---

### 7. [q->qinfo未检查NULL] - 多个函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_qm_udrv.c:245, 469, 512, 561, 617, 710`

**Dangerous Code**:
```c
// qm_set_queue_regions line 245
struct q_info *qinfo = q->qinfo;
struct qm_queue_info *info = qinfo->priv;

// qm_init_queue line 561
struct q_info *qinfo = q->qinfo;

// qm_send line 617
struct q_info *qinfo = q->qinfo;

// qm_recv line 710
struct q_info *qinfo = q->qinfo;
```

**Risk Analysis**:
多个函数直接访问q->qinfo而不检查q是否为NULL或q->qinfo是否为NULL。

---

### 8. [q->doorbell_base未检查有效性] - qm_db_v1/qm_db_v2函数 - [Severity: LOW]

**Location**: `v1/drv/hisi_qm_udrv.c:215, 231`

**Code**:
```c
static int qm_db_v1(struct qm_queue_info *q, __u8 cmd,
	       __u16 idx, __u8 priority)
{
	void *base = q->doorbell_base;
	...
	*((__u64 *)base) = doorbell;

static int qm_db_v2(struct qm_queue_info *q, __u8 cmd,
		      __u16 idx, __u8 priority)
{
	void *base = q->doorbell_base;
	...
	*((__u64 *)base) = doorbell;
```

**Risk Analysis**:
doorbell_base未检查是否为NULL或MAP_FAILED，直接写入可能导致崩溃。

---

### 9. [hisi_qm_inject_op_register参数检查顺序问题] - [Severity: LOW]

**Location**: `v1/drv/hisi_qm_udrv.c:786-802`

**Code**:
```c
int hisi_qm_inject_op_register(struct wd_queue *q, struct hisi_qm_inject_op *op)
{
	struct qm_queue_info *info;
	struct q_info *qinfo;

	if (!q || !q->qinfo || !op || !op->sqe_fill_priv ||
	    !op->sqe_parse_priv) {
		...
	}
	qinfo = q->qinfo;
	info = qinfo->priv;  // qinfo->priv可能为NULL
```

**Risk Analysis**:
检查了q和q->qinfo，但后续直接访问qinfo->priv未检查是否为NULL。

---

### 10. [整数乘法可能溢出] - qm_hw_sgl_init函数 - [Severity: LOW]

**Location**: `v1/drv/hisi_qm_udrv.c:123-124`

**Code**:
```c
hisi_sgl = br->alloc(br->usr, sizeof(struct hisi_sgl) +
		     sizeof(struct hisi_sge) * buf_num);
```

**Risk Analysis**:
buf_num最大为HISI_SGL_SGE_NUM_MAX(255)，sizeof(struct hisi_sge) * 255可能溢出（取决于struct hisi_sge大小）。

---

### 11. [信息性发现] - 队列管理架构 - [Severity: INFO]

**Location**: `v1/drv/hisi_qm_udrv.c:40-44`

**Code**:
```c
enum {
	HISI_QM_V1 = 1,
	HISI_QM_V2 = 2,
	HISI_QM_V3 = 3,
};
```

**Analysis**:
支持三个版本的硬件队列管理器：
1. **V1**: 使用qm_db_v1门铃函数，基础版本
2. **V2**: 使用qm_db_v2门铃函数，增加mask处理
3. **V3**: 支持bd3_sqe系列回调函数（SEC/ZIP）

---

### 12. [信息性发现] - 多算法支持架构 - [Severity: INFO]

**Location**: `v1/drv/hisi_qm_udrv.c:278-410`

**Analysis**:
文件实现了三个加速器的统一队列管理：
- **HPRE**: RSA, DH, ECDH, X25519, X448, ECDSA, SM2
- **SEC**: Cipher, Digest, AEAD (V2/V3版本)
- **ZIP**: Zlib, Gzip, Deflate, lz77_zstd (V2/V3版本)

使用sqe_fill/sqe_parse函数指针数组实现算法分发。

---

### 13. [正面发现] - hisi_qm_inject_op_register参数检查 - [Severity: POSITIVE]

**Location**: `v1/drv/hisi_qm_udrv.c:786-790`

**Code**:
```c
if (!q || !q->qinfo || !op || !op->sqe_fill_priv ||
    !op->sqe_parse_priv) {
	WD_ERR("inject option is invalid!\n");
	return -WD_EINVAL;
}
```

**Analysis**:
正确检查q、q->qinfo、op及其回调函数指针。

---

### 14. [正面发现] - qm_hw_sgl_init参数检查 - [Severity: POSITIVE]

**Location**: `v1/drv/hisi_qm_udrv.c:108-111`

**Code**:
```c
if (!pool || buf_num < 0 || sge_num < 0) {
	WD_ERR("hw_sgl_init init param err!\n");
	return -WD_EINVAL;
}
```

**Analysis**:
正确检查pool参数和buf_num/sge_num范围。

---

### 15. [正面发现] - 队列深度检查 - [Severity: POSITIVE]

**Location**: `v1/drv/hisi_qm_udrv.c:732-736`

**Code**:
```c
sq_head = CQE_SQ_HEAD_INDEX(cqe);
if (unlikely(sq_head >= info->sq_depth)) {
	wd_fair_unlock(&info->rc_lock);
	WD_ERR("CQE_SQ_HEAD_INDEX(%u) error\n", sq_head);
	return -WD_EIO;
}
```

**Analysis**:
正确检查sq_head索引是否超出队列深度范围。

---

## 总结

### 问题分布
| 问题类型 | 数量 |
|----------|------|
| 空指针检查缺失 | 5 |
| 回调函数未检查NULL | 3 |
| strtol使用问题 | 1 |
| 整数溢出风险 | 1 |
| 参数验证不完整 | 2 |

### 模块特点

v1/drv/hisi_qm_udrv.c实现了统一的队列管理驱动：
1. 支持HPRE/SEC/ZIP三种加速器
2. 支持V1/V2/V3三个硬件版本
3. 使用函数指针数组实现算法分发
4. 实现硬件SGL管理

### 主要风险

**最大风险**：sqe_fill/sqe_parse回调未检查NULL
- qm_send和qm_recv直接调用回调函数
- 如果未正确初始化或atype错误，会导致崩溃
- 影响所有使用队列发送/接收的操作

### 与根目录drv/hisi_qm_udrv.c对比

| 对比项 | v1/drv/hisi_qm_udrv.c | drv/hisi_qm_udrv.c |
|--------|----------------------|-------------------|
| 文件行数 | 851 | 约500行 |
| 版本支持 | V1/V2/V3 | 主版本 |
| 回调检查 | 缺失NULL检查 | 同样缺失 |
| 参数检查 | 部分缺失 | 较完整 |

### 建议优先修复顺序
1. sqe_fill/sqe_parse NULL检查（HIGH）
2. br->iova_map/alloc/free NULL检查（HIGH/MEDIUM）
3. q/q->qinfo NULL检查（MEDIUM）
4. drv_get_sgl_pri返回值检查（MEDIUM）
5. strtoul errno设置（MEDIUM）