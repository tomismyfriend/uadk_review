# UADK代码检视报告 - 第56轮

## 检视信息
- **检视文件**: v1/drv/hisi_sec_udrv.c
- **检视时间**: 2026-04-08
- **文件行数**: 3507行
- **检视规则**: cpp-memory-safety, cpp-resource-management, cpp-input-validation

## 检视摘要
| 等级 | 数量 |
|------|------|
| HIGH | 2 |
| MEDIUM | 6 |
| LOW | 4 |
| INFO | 3 |
| POSITIVE | 5 |

---

## 问题详情

### 1. [tag未检查NULL] - qm_fill_cipher_sqe函数 - [Severity: HIGH]

**Location**: `v1/drv/hisi_sec_udrv.c:1090-1092`

**Dangerous Code**:
```c
int qm_fill_cipher_sqe(void *message, struct qm_queue_info *info, __u16 i)
{
	struct wcrypto_cipher_msg *msg = message;
	struct wcrypto_cipher_tag *tag = (void *)(uintptr_t)msg->usr_data;
	struct wd_sec_udata *udata = tag->priv;  // tag可能为NULL
	struct wd_queue *q = info->q;
	...
}
```

**Risk Analysis**:
1. msg->usr_data转换为tag指针，未检查是否为NULL
2. 直接访问tag->priv，如果tag为NULL会崩溃
3. info->q也未检查是否为NULL

**Fix Suggestion**:
```c
struct wcrypto_cipher_tag *tag = (void *)(uintptr_t)msg->usr_data;
if (unlikely(!tag)) {
	WD_ERR("tag is NULL!\n");
	return -WD_EINVAL;
}
struct wd_sec_udata *udata = tag->priv;
```

---

### 2. [drv_get_sgl_pri返回值未检查NULL] - map_addr函数 - [Severity: HIGH]

**Location**: `v1/drv/hisi_sec_udrv.c:507-508, 2332-2333`

**Dangerous Code**:
```c
static int map_addr(struct wd_queue *q, __u8 *key, __u16 len,
			 __u32 *addr_l, __u32 *addr_h, __u8 data_fmt)
{
	...
	if (data_fmt == WD_SGL_BUF) {
		if (unlikely(!key))
			return -WD_ENOMEM;
		p = drv_get_sgl_pri((struct wd_sgl *)key);
		phy = ((struct hisi_sgl *)p)->sge_entries[0].buf;  // p未检查NULL
	}
	...
}
```

**Risk Analysis**:
1. drv_get_sgl_pri返回void指针，未检查是否为NULL
2. 直接访问((struct hisi_sgl *)p)->sge_entries会导致崩溃
3. sge_entries数组索引0也未检查

**Fix Suggestion**:
```c
p = drv_get_sgl_pri((struct wd_sgl *)key);
if (unlikely(!p || !((struct hisi_sgl *)p)->sge_entries)) {
	WD_ERR("failed to get sgl private data!\n");
	return -WD_ENOMEM;
}
phy = ((struct hisi_sgl *)p)->sge_entries[0].buf;
```

---

### 3. [g_digest_a_alg/g_hmac_a_alg数组越界风险] - 多个函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_sec_udrv.c:62-72, 1230, 1232, 1293, 1295, 2151`

**Code**:
```c
static int g_digest_a_alg[WCRYPTO_MAX_DIGEST_TYPE] = {
	A_ALG_SM3, A_ALG_MD5, A_ALG_SHA1, ...
};
static int g_hmac_a_alg[WCRYPTO_MAX_DIGEST_TYPE] = {
	A_ALG_HMAC_SM3, A_ALG_HMAC_MD5, ...
};

// fill_digest_bd2_alg line 1230-1232
if (msg->mode == WCRYPTO_DIGEST_NORMAL)
	sqe->type2.a_alg = g_digest_a_alg[msg->alg];
else if (msg->mode == WCRYPTO_DIGEST_HMAC)
	sqe->type2.a_alg = g_hmac_a_alg[msg->alg];
```

**Risk Analysis**:
1. digest_param_check函数检查了`msg->alg >= WCRYPTO_MAX_DIGEST_TYPE`
2. 但部分调用路径可能绕过检查
3. 需要确保所有访问这些数组的代码都进行了边界检查

---

### 4. [info->q未检查NULL] - 多个fill/parse函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_sec_udrv.c:1092, 1133, 1543, 1777, 2777, 3381`

**Code**:
```c
int qm_fill_cipher_sqe(void *message, struct qm_queue_info *info, __u16 i)
{
	...
	struct wd_queue *q = info->q;  // info和info->q未检查NULL
	...
}
```

**Risk Analysis**:
多个fill和parse函数直接访问info->q，未检查info或info->q是否为NULL。

---

### 5. [wd_get_first_sge_buf返回值检查不一致] - 多个函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_sec_udrv.c:240, 260, 2228-2233`

**Code**:
```c
// ctr_iv_inc line 240 - 有检查
counter1 = wd_get_first_sge_buf((struct wd_sgl *)counter);
if (unlikely(!counter1))
	return;

// update_iv_from_res line 260 - 有检查
dst1 = wd_get_first_sge_buf((struct wd_sgl *)dst);
if (unlikely(!dst1))
	return;

// set_aead_auth_iv line 2228-2233 - 无检查
if (msg->data_fmt == WD_SGL_BUF) {
	iv = wd_get_first_sge_buf((struct wd_sgl *)msg->iv);
	aiv = wd_get_first_sge_buf((struct wd_sgl *)msg->aiv);
} else {
	...
}
```

**Risk Analysis**:
set_aead_auth_iv函数中调用wd_get_first_sge_buf后未检查返回值，可能导致后续操作崩溃。

---

### 6. [enc_func函数指针未检查NULL] - gcm_do_soft_mac函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_sec_udrv.c:2537-2544`

**Code**:
```c
static int gcm_do_soft_mac(struct wcrypto_aead_msg *msg)
{
	typedef void (*enc_ops)(__u8 *, __u32, __u8 *, __u8 *);
	enc_ops enc_func;
	...
	if (msg->calg == WCRYPTO_CIPHER_AES)
		enc_func = aes_encrypt;
	else if (msg->calg == WCRYPTO_CIPHER_SM4)
		enc_func = sm4_encrypt;
	else
		return -WD_EINVAL;

	enc_func(msg->ckey, msg->ckey_bytes, data, H);  // enc_func调用点
```

**Risk Analysis**:
虽然switch语句覆盖了有效情况，但如果aes_encrypt或sm4_encrypt函数指针为NULL（可能因为库未正确初始化），调用会崩溃。

---

### 7. [msg->iv/aiv可能为NULL] - set_aead_auth_iv函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_sec_udrv.c:2226-2262`

**Code**:
```c
static void set_aead_auth_iv(struct wcrypto_aead_msg *msg)
{
	...
	if (msg->data_fmt == WD_SGL_BUF) {
		iv = wd_get_first_sge_buf((struct wd_sgl *)msg->iv);
		aiv = wd_get_first_sge_buf((struct wd_sgl *)msg->aiv);
	} else {
		iv = msg->iv;
		aiv = msg->aiv;
	}

	memcpy(aiv, iv, msg->iv_bytes);  // iv和aiv可能为NULL
	...
}
```

**Risk Analysis**:
msg->iv和msg->aiv可能为NULL，需要在使用前检查。

---

### 8. [栈上大数组] - 多个函数 - [Severity: LOW]

**Location**: `v1/drv/hisi_sec_udrv.c:2320, 2523-2531, 2817, 3110, 3421`

**Code**:
```c
// aead_bd3_map_iv_mac line 2320
__u8 mac[AEAD_IV_MAX_BYTES] = { 0 };  // 64 bytes

// gcm_do_soft_mac line 2523-2531
__u8 ctr_r[AES_BLOCK_SIZE] = {0};
__u8 data[AES_BLOCK_SIZE] = {0};
__u8 H[AES_BLOCK_SIZE] = {0};
__u8 K[AES_BLOCK_SIZE] = {0};
__u8 S[AES_BLOCK_SIZE] = {0};
__u8 g[AES_BLOCK_SIZE] = {0};
__u8 G[AES_BLOCK_SIZE] = {0};  // 共112 bytes

// parse_aead_bd2 line 3421
__u8 mac[64] = { 0 };  // 64 bytes
```

**Risk Analysis**:
多个函数在栈上定义固定大小数组，虽然不大（最大约112字节），但在深度调用链中可能增加栈压力。

---

### 9. [整数运算可能溢出] - 多个函数 - [Severity: LOW]

**Location**: `v1/drv/hisi_sec_udrv.c:2728-2729, 2858, 2915-2916`

**Code**:
```c
// aead_comb_param_check line 2728-2729
total = msg->in_bytes + msg->assoc_bytes;
if (unlikely(total > MAX_CIPHER_LENGTH)) {

// fill_aead_bd3 line 2858
sqe->a_len = msg->in_bytes + msg->assoc_bytes;
```

**Risk Analysis**:
两个__u32相加可能溢出，导致检查失效。应使用__u64或进行溢出检查。

---

### 10. [parse函数返回值检查不完整] - 多个parse函数 - [Severity: LOW]

**Location**: `v1/drv/hisi_sec_udrv.c:2839-2846, 3445-3454`

**Code**:
```c
// parse_aead_bd3 line 2839-2846
ret = wd_sgl_cp_to_pbuf(...);
if (ret)
	return;  // 只返回，不设置错误码
ret = wd_sgl_cp_from_pbuf(...);
if (ret)
	return;  // 只返回，不设置错误码
```

**Risk Analysis**:
wd_sgl_cp_to_pbuf和wd_sgl_cp_from_pbuf的错误处理只返回，不设置msg->result，可能导致上层无法感知错误。

---

### 11. [信息性发现] - BD类型架构 - [Severity: INFO]

**Location**: `v1/drv/hisi_sec_udrv.c:35-60`

**Analysis**:
文件实现了三种BD（Buffer Description）类型：
- **BD_TYPE1**: 存储场景，支持DIF（Data Integrity Field）
- **BD_TYPE2**: IPSEC场景，标准加密认证
- **BD_TYPE3**: 流式场景，支持CCM/GCM等AEAD模式

BD类型决定硬件处理流程和数据格式。

---

### 12. [信息性发现] - GCM软件计算 - [Severity: INFO]

**Location**: `v1/drv/hisi_sec_udrv.c:2505-2595`

**Code**:
```c
static int gcm_do_soft_mac(struct wcrypto_aead_msg *msg)
{
	...
	if (msg->calg == WCRYPTO_CIPHER_AES)
		enc_func = aes_encrypt;
	else if (msg->calg == WCRYPTO_CIPHER_SM4)
		enc_func = sm4_encrypt;
	...
	galois_compute(G, H, msg->aiv + GCM_STREAM_MAC_OFFSET, AES_BLOCK_SIZE);
	...
}
```

**Analysis**:
实现了GCM模式的软件fallback：
1. 当硬件不支持某些流式操作时使用
2. 使用AES或SM4加密函数
3. 使用Galois域乘法计算认证标签

---

### 13. [信息性发现] - AEAD流式处理 - [Severity: INFO]

**Location**: `v1/drv/hisi_sec_udrv.c:2616-2674`

**Code**:
```c
static int fill_aead_stream_bd3(...)
{
	switch (msg->msg_state) {
	case WCRYPTO_AEAD_MSG_FIRST:
		fill_gcm_first_bd3(msg, sqe);
		break;
	case WCRYPTO_AEAD_MSG_MIDDLE:
		fill_gcm_middle_bd3(q, msg, sqe);
		break;
	case WCRYPTO_AEAD_MSG_END:
		gcm_auth_ivin(msg);
		ret = gcm_do_soft_mac(msg);
		break;
	...
}
```

**Analysis**:
支持流式AEAD处理：
- FIRST/MIDDLE BD：硬件处理
- END BD：软件计算最终MAC标签

---

### 14. [正面发现] - cipher_param_check完整验证 - [Severity: POSITIVE]

**Location**: `v1/drv/hisi_sec_udrv.c:739-776`

**Code**:
```c
static int cipher_param_check(struct wcrypto_cipher_msg *msg)
{
	if (unlikely(msg->in_bytes > MAX_CIPHER_LENGTH ||
		     !msg->in_bytes || msg->in_bytes != msg->out_bytes)) {
		...
	}
	// 检查不同算法和模式的参数
	...
}
```

**Analysis**:
完整的参数验证，包括长度限制、算法特定要求。

---

### 15. [正面发现] - drv_iova_map返回值检查 - [Severity: POSITIVE]

**Location**: `v1/drv/hisi_sec_udrv.c:510-515, 617-621, 645-649`

**Code**:
```c
phy = (uintptr_t)drv_iova_map(q, key, len);
if (unlikely(!phy)) {
	WD_ERR("Get key dma address fail!\n");
	return -WD_ENOMEM;
}
```

**Analysis**:
所有DMA映射调用都正确检查了返回值。

---

### 16. [正面发现] - req_cache NULL检查 - [Severity: POSITIVE]

**Location**: `v1/drv/hisi_sec_udrv.c:1962-1965, 1995-1998, 2095-2098`

**Code**:
```c
if (unlikely(!cipher_msg)) {
	WD_ERR("info->req_cache is null at index:%hu\n", i);
	return 0;
}
```

**Analysis**:
parse函数正确检查req_cache索引处的指针。

---

### 17. [正面发现] - 硬件错误处理 - [Severity: POSITIVE]

**Location**: `v1/drv/hisi_sec_udrv.c:1813-1822, 1884-1889`

**Code**:
```c
if (sqe->type2.done != SEC_HW_TASK_DONE || sqe->type2.error_type) {
	WD_ERR("SEC BD2 %s fail!done=0x%x, etype=0x%x\n", "cipher",
	(__u32)sqe->type2.done, (__u32)sqe->type2.error_type);
	cipher_msg->result = WD_IN_EPARA;
} else {
	cipher_msg->result = WD_SUCCESS;
}
```

**Analysis**:
正确处理硬件执行结果，记录错误信息。

---

### 18. [正面发现] - fill_bd_addr_type封装 - [Severity: POSITIVE]

**Location**: `v1/drv/hisi_sec_udrv.c:210-219`

**Code**:
```c
static void fill_bd_addr_type(__u8 data_fmt, struct hisi_sec_sqe *sqe)
{
	if (data_fmt == WD_SGL_BUF) {
		sqe->src_addr_type = HISI_SGL_BUF;
		sqe->dst_addr_type = HISI_SGL_BUF;
	} else {
		sqe->src_addr_type = HISI_FLAT_BUF;
		sqe->dst_addr_type = HISI_FLAT_BUF;
	}
}
```

**Analysis**:
良好的封装，统一处理地址类型设置。

---

## 总结

### 问题分布
| 问题类型 | 数量 |
|----------|------|
| 空指针检查缺失 | 3 |
| 数组越界风险 | 1 |
| 函数指针未检查NULL | 1 |
| 参数验证不完整 | 2 |
| 整数运算风险 | 1 |

### 模块特点

v1/drv/hisi_sec_udrv.c实现了SEC（安全引擎）驱动：
1. 支持Cipher/Digest/AEAD三种算法类型
2. 支持BD1/BD2/BD3三种硬件描述符格式
3. 支持流式和块式处理
4. 实现GCM/CCM的软件fallback

### 主要风险

**最大风险**：tag未检查NULL
- 多个fill函数直接访问tag->priv
- 如果tag为NULL会崩溃
- 影响所有cipher/digest/aead操作

### 与根目录drv/hisi_sec.c对比

| 对比项 | v1/drv/hisi_sec_udrv.c | drv/hisi_sec.c |
|--------|------------------------|-----------------|
| 文件行数 | 3507 | 约2500行 |
| BD类型 | BD1/BD2/BD3 | 主版本 |
| 流式支持 | 完整 | 简化版本 |
| GCM软件计算 | 有 | 无 |

### 建议优先修复顺序
1. tag NULL检查（HIGH）
2. drv_get_sgl_pri返回值检查（HIGH）
3. info->q NULL检查（MEDIUM）
4. set_aead_auth_iv参数检查（MEDIUM）