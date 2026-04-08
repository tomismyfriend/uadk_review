# UADK代码检视报告 - 第55轮

## 检视信息
- **检视文件**: v1/drv/hisi_hpre_udrv.c
- **检视时间**: 2026-04-08
- **文件行数**: 2481行
- **检视规则**: cpp-memory-safety, cpp-resource-management, cpp-input-validation

## 检视摘要
| 等级 | 数量 |
|------|------|
| HIGH | 2 |
| MEDIUM | 6 |
| LOW | 4 |
| INFO | 3 |
| POSITIVE | 4 |

---

## 问题详情

### 1. [br->alloc/br->free未检查NULL] - create_req/free_req函数 - [Severity: HIGH]

**Location**: `v1/drv/hisi_hpre_udrv.c:1780, 1814, 1833, 1848, 1850`

**Dangerous Code**:
```c
static struct wcrypto_ecc_msg *create_req(...)
{
	...
	prikey->data = br->alloc(br->usr, ECC_PRIKEY_SZ(src->key_bytes));
	...
}

static void free_req(struct qm_queue_info *info, struct wcrypto_ecc_msg *req)
{
	...
	br->free(br->usr, key->prikey->data);  // br->free未检查NULL
	free(req->key);
	br->free(br->usr, req->out);  // br->free未检查NULL
	...
}
```

**Risk Analysis**:
1. br指针来自`qinfo->br`，未检查是否为NULL
2. br->alloc和br->free函数指针未检查是否为NULL
3. 如果br或其回调为NULL，会导致崩溃

**Fix Suggestion**:
```c
if (!br || !br->alloc || !br->free) {
	WD_ERR("br or callbacks is NULL!\n");
	return NULL;
}
```

---

### 2. [drv_iova_map返回值检查不一致] - 多个函数 - [Severity: HIGH]

**Location**: `v1/drv/hisi_hpre_udrv.c:436, 443, 581, 627, 1042, 1497`

**Dangerous Code**:
```c
// qm_rsa_prepare_iot line 436
phy = (uintptr_t)drv_iova_map(q, msg->in, msg->key_bytes);
if (!phy) {  // 正确检查
	...
}

// fill_dh_g_param line 581
phy = (uintptr_t)drv_iova_map(q, (void *)msg->g, msg->key_bytes);
if (unlikely(!phy)) {  // 正确检查
	...
}
```

**Risk Analysis**:
虽然大部分drv_iova_map调用检查了返回值，但：
1. drv_iova_map函数本身可能未检查q参数是否为NULL
2. drv_iova_map内部可能调用drv_get_mm_br获取br，如果br->iova_map为NULL会崩溃
3. 需要在更底层统一检查

---

### 3. [qinfo->br未检查NULL] - 多个函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_hpre_udrv.c:1772, 1798, 1845, 1944`

**Dangerous Code**:
```c
// init_req line 1772
struct wd_mm_br *br = &qinfo->br;
...
dst->out = br->alloc(br->usr, ECDH_OUT_PARAMS_SZ(ksz));

// create_req line 1798
struct wd_mm_br *br = &qinfo->br;
...
prikey->data = br->alloc(br->usr, ECC_PRIKEY_SZ(src->key_bytes));
```

**Risk Analysis**:
qinfo直接从`info->q->qinfo`获取，未检查中间指针是否为NULL。如果info->q或qinfo为NULL，会崩溃。

---

### 4. [info->q未检查NULL] - 多个函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_hpre_udrv.c:473, 518, 680, 731, 1644, 2007`

**Dangerous Code**:
```c
int qm_fill_rsa_sqe(void *message, struct qm_queue_info *info, __u16 i)
{
	...
	struct wd_queue *q = info->q;
	...
}

int qm_parse_rsa_sqe(void *msg, const struct qm_queue_info *info, ...)
{
	...
	struct wd_queue *q = info->q;
	...
}
```

**Risk Analysis**:
多个fill/parse函数直接访问info->q，未检查info或info->q是否为NULL。

---

### 5. [hash->cb调用点检查不一致] - fill_sm2_enc_sqe/fill_sm2_dec_sqe - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_hpre_udrv.c:1891, 1955, 2218, 2283`

**Code**:
```c
// fill_sm2_enc_sqe line 1891
if (unlikely(!hash->cb || hash->type >= WCRYPTO_HASH_MAX)) {
	WD_ERR("hash parameter error, type = %u\n", hash->type);
	return -WD_EINVAL;
}

// sm2_kdf line 2218 (在while循环中调用)
ret = hash->cb(p_in, in_len, t_out, h_bytes, hash->usr);
```

**Risk Analysis**:
1. fill_sm2_enc_sqe和fill_sm2_dec_sqe正确检查了hash->cb
2. 但sm2_kdf和sm2_hash函数中直接调用hash->cb，依赖调用者检查
3. 如果其他函数直接调用sm2_kdf/sm2_hash，可能绕过检查

---

### 6. [wcrypto_get_*_params返回值未检查] - 多个函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_hpre_udrv.c:123, 172, 200, 229, 890, 964`

**Code**:
```c
// qm_fill_rsa_crt_prikey2 line 123
wcrypto_get_rsa_crt_prikey_params(prikey, &wd_dq, &wd_dp,
			&wd_qinv, &wd_q, &wd_p);
if (unlikely(!wd_dq || !wd_dp || !wd_qinv || !wd_q || !wd_p)) {  // 检查了
	...
}

// ecc_prepare_prikey line 890
wcrypto_get_ecc_prikey_params((void *)key, &p, &a, &b, &n, &g, &d);
// 未检查返回的指针是否为NULL
```

**Risk Analysis**:
wcrypto_get_*系列函数返回指针参数，部分地方检查了返回值，部分未检查。

---

### 7. [msg->key未检查NULL] - 多个函数 - [Severity: MEDIUM]

**Location**: `v1/drv/hisi_hpre_udrv.c:329, 345, 366, 398, 1031, 1037`

**Code**:
```c
// rsa_key_unmap line 329
struct wcrypto_rsa_kg_out *key = (void *)msg->key;
// msg->key未检查

// rsa_prepare_sign_key line 345
ret = qm_fill_rsa_crt_prikey2((void *)msg->key, &data);
// msg->key未检查
```

**Risk Analysis**:
多个prepare函数直接使用msg->key，未检查是否为NULL。

---

### 8. [栈上大数组可能溢出] - sm2_kdf/sm2_hash函数 - [Severity: LOW]

**Location**: `v1/drv/hisi_hpre_udrv.c:2182, 2266, 2353`

**Code**:
```c
static int sm2_kdf(struct wd_dtb *out, ...)
{
	char p_out[MAX_HASH_LENS] = {0};  // MAX_HASH_LENS = BITS_TO_BYTES(521) = 66
	...
}

static int sm2_hash(...)
{
	char hash_out[MAX_HASH_LENS] = {0};  // 栈上66字节数组
	...
}
```

**Risk Analysis**:
虽然MAX_HASH_LENS只有66字节，不是很大，但如果在深度递归或多线程环境下可能增加栈压力。

---

### 9. [整数乘法可能溢出] - 多个函数 - [Severity: LOW]

**Location**: `v1/drv/hisi_hpre_udrv.c:162-163, 190-191, 218-219, 2264`

**Code**:
```c
// qm_fill_rsa_crt_prikey2 line 162-163
return (int)(wd_dq->bsize + wd_qinv->bsize + wd_p->bsize +
		wd_q->bsize + wd_dp->bsize);

// sm2_hash line 2264
__u64 lens = (__u64)msg->dsize + 2 * (__u64)x2y2->x.dsize;
```

**Risk Analysis**:
多个bsize/dsize相加可能溢出。虽然返回类型是int，但如果中间值超过INT_MAX会产生问题。

---

### 10. [parse_second_sqe无限循环风险] - [Severity: LOW]

**Location**: `v1/drv/hisi_hpre_udrv.c:2096-2125`

**Code**:
```c
static int parse_second_sqe(...)
{
	...
	while (1) {
		...
		if (cnt++ > MAX_WAIT_CNT)  // MAX_WAIT_CNT = 10000000
			return 0;
		usleep(1);
	}
	...
}
```

**Risk Analysis**:
虽然有MAX_WAIT_CNT限制，但1000万次循环加usleep(1)可能导致约10秒的等待时间。返回0表示超时，但调用者可能处理不当。

---

### 11. [信息性发现] - SM2加密/解密流程 - [Severity: INFO]

**Location**: `v1/drv/hisi_hpre_udrv.c:1872-2467`

**Analysis**:
文件实现了SM2公钥加密算法：
1. **SM2加密**: 分两步计算 k*G 和 k*Pb，然后使用KDF和哈希计算C1/C2/C3
2. **SM2解密**: 计算 d*C1，然后使用KDF和哈希验证C3并恢复明文
3. 支持大数据量（超过硬件限制）的软件计算fallback

---

### 12. [信息性发现] - ECC算法支持矩阵 - [Severity: INFO]

**Location**: `v1/drv/hisi_hpre_udrv.c:772-814`

**Code**:
```c
static int qm_ecc_prepare_alg(struct hisi_hpre_sqe *hw_msg,
			      struct wcrypto_ecc_msg *msg)
{
	switch (msg->op_type) {
	case WCRYPTO_ECXDH_GEN_KEY:
	case WCRYPTO_ECXDH_COMPUTE_KEY:
		...
	case WCRYPTO_ECDSA_SIGN:
		hw_msg->alg = HPRE_ALG_ECDSA_SIGN;
		break;
	case WCRYPTO_ECDSA_VERIFY:
		hw_msg->alg = HPRE_ALG_ECDSA_VERF;
		break;
	case WCRYPTO_SM2_ENCRYPT:
		hw_msg->alg = HPRE_ALG_SM2_ENC;
		break;
	...
	}
}
```

**Analysis**:
支持多种ECC操作：
- ECDH/X25519/X448密钥生成和计算
- ECDSA签名和验证
- SM2加密、解密、签名、验证、密钥生成

---

### 13. [信息性发现] - X25519/X448特殊处理 - [Severity: INFO]

**Location**: `v1/drv/hisi_hpre_udrv.c:920-927`

**Code**:
```c
if (id == WCRYPTO_X25519) {
	dat[31] &= 248;
	dat[0] &= 127;
	dat[0] |= 64;
} else if (id == WCRYPTO_X448) {
	dat[55 + bsize - dsize] &= 252;
	dat[0 + bsize - dsize] |= 128;
}
```

**Analysis**:
按照RFC 7748规范对X25519/X448的私钥进行特殊处理，设置特定位以满足安全要求。

---

### 14. [正面发现] - 参数验证 - [Severity: POSITIVE]

**Location**: `v1/drv/hisi_hpre_udrv.c:62-70, 92-95`

**Code**:
```c
static int qm_crypto_bin_to_hpre_bin(char *dst, const char *src,
				     int b_size, int d_size, const char *p_name)
{
	if (unlikely(!dst || !src || b_size <= 0 || d_size <= 0)) {
		WD_ERR("%s trans to hpre bin: parameters err!\n", p_name);
		return -WD_EINVAL;
	}
	...
}
```

**Analysis**:
核心数据转换函数正确检查所有参数。

---

### 15. [正面发现] - drv_iova_map返回值检查 - [Severity: POSITIVE]

**Location**: `v1/drv/hisi_hpre_udrv.c:411-415, 436-440`

**Code**:
```c
phy  = (uintptr_t)drv_iova_map(q, msg->key, ret);
if (unlikely(!phy)) {
	WD_ERR("Dma map rsa key fail!\n");
	return -WD_ENOMEM;
}
```

**Analysis**:
所有DMA映射调用都正确检查了返回值。

---

### 16. [正面发现] - req_cache NULL检查 - [Severity: POSITIVE]

**Location**: `v1/drv/hisi_hpre_udrv.c:525-528, 734-737, 2013-2016`

**Code**:
```c
if (unlikely(!rsa_msg)) {
	WD_ERR("info->req_cache is null at index:%hu\n", i);
	return 0;
}
```

**Analysis**:
parse函数正确检查req_cache索引处的指针。

---

### 17. [正面发现] - 硬件错误处理 - [Severity: POSITIVE]

**Location**: `v1/drv/hisi_hpre_udrv.c:534-541, 741-750`

**Code**:
```c
if (hw_msg->done != HPRE_HW_TASK_DONE || hw_msg->etype) {
	WD_ERR("HPRE do %s fail!done=0x%x, etype=0x%x\n", "rsa",
		(__u32)hw_msg->done, (__u32)hw_msg->etype);
	if (hw_msg->done == HPRE_HW_TASK_INIT) {
		rsa_msg->result = WD_EINVAL;
	} else {
		rsa_msg->result = WD_IN_EPARA;
	}
	...
}
```

**Analysis**:
正确处理硬件执行结果，区分初始化错误和参数错误。

---

## 总结

### 问题分布
| 问题类型 | 数量 |
|----------|------|
| 空指针检查缺失 | 4 |
| 回调函数未检查NULL | 2 |
| 整数运算风险 | 1 |
| 栈溢出风险 | 1 |
| 参数验证不完整 | 2 |

### 模块特点

v1/drv/hisi_hpre_udrv.c实现了HPRE（高性能RSA引擎）驱动：
1. 支持RSA签名/验证/密钥生成
2. 支持DH密钥交换
3. 支持ECC（ECDSA/ECDH/SM2/X25519/X448）
4. 实现了SM2加密/解密的复杂流程

### 主要风险

**最大风险**：br->alloc/br->free未检查NULL
- create_req和free_req函数直接调用br->alloc/br->free
- 如果br或其回调为NULL，会导致崩溃
- 影响SM2加密/解密流程

### 与根目录drv/hisi_hpre.c对比

| 对比项 | v1/drv/hisi_hpre_udrv.c | drv/hisi_hpre.c |
|--------|------------------------|-----------------|
| 文件行数 | 2481 | 约2000行 |
| RSA支持 | 完整 | 完整 |
| ECC支持 | 更丰富 | 标准 |
| SM2实现 | 详细流程 | 简化版本 |

### 建议优先修复顺序
1. br->alloc/br->free NULL检查（HIGH）
2. info->q/qinfo NULL检查（MEDIUM）
3. hash->cb调用一致性检查（MEDIUM）
4. msg->key NULL检查（MEDIUM）