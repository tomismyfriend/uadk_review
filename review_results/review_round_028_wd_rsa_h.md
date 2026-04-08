# UADK代码检视报告 - 第28轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第28轮
- **规则来源**: E:/work_space/加速器用户态学习/代码检视/review_rules
- **使用Skills**: cpp-memory-safety, cpp-input-validation

---

## 本轮检视文件

### 已检视文件
1-27. 前序文件 (见review_progress.md)
28. include/wd_rsa.h (本轮) - RSA算法头文件，252行

### 待检视文件
- include/目录下其他头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## include/wd_rsa.h 检视结果

### 1. 枚举缺少MAX值 - wd_rsa_op_type [LOW]

**位置**: wd_rsa.h:43-48

**问题类型**: 枚举设计 (cpp-coding-standards)

**代码**:
```c
/* RSA operational types */
enum wd_rsa_op_type  {
    WD_RSA_INVALID, /* invalid rsa operation */
    WD_RSA_SIGN, /* RSA sign */
    WD_RSA_VERIFY, /* RSA verify */
    WD_RSA_GENKEY, /* RSA key generation */
};
```

**分析**:
- 枚举定义了4种RSA操作类型
- 缺少WD_RSA_OP_TYPE_MAX值用于边界检查
- 与其他有MAX值的枚举不一致

---

### 2. 枚举缺少MAX值 - wd_rsa_key_type [LOW]

**位置**: wd_rsa.h:51-56

**问题类型**: 枚举设计 (cpp-coding-standards)

**代码**:
```c
/* RSA key types */
enum wd_rsa_key_type {
    WD_RSA_INVALID_KEY, /* invalid rsa key type */
    WD_RSA_PUBKEY, /* rsa publick key type */
    WD_RSA_PRIKEY1, /* invalid rsa private common key type */
    WD_RSA_PRIKEY2, /* invalid rsa private CRT key type */
};
```

**分析**:
- 枚举定义了4种RSA密钥类型
- 缺少WD_RSA_KEY_TYPE_MAX值用于边界检查
- 注释中"invalid rsa private common key type"应该是"valid"的笔误

---

### 3. 宏定义整数运算风险 [LOW]

**位置**: wd_rsa.h:18-21

**问题类型**: 整数运算 (cpp-integer-overflow)

**代码**:
```c
#define CRT_PARAMS_SZ(key_size)     ((5 * (key_size)) >> 1)
#define CRT_GEN_PARAMS_SZ(key_size) ((7 * (key_size)) >> 1)
#define GEN_PARAMS_SZ(key_size)     ((key_size) << 1)
#define CRT_PARAM_SZ(key_size)      ((key_size) >> 1)
```

**分析**:
- 宏定义使用了乘法和移位运算
- 如果key_size很大，5*key_size或7*key_size可能溢出
- key_size应该是有效的密钥大小(如1024, 2048, 4096等)
- 建议在文档中说明key_size的有效范围

---

### 4. 函数输出参数NULL检查 [INFO]

**位置**: wd_rsa.h:67-111

**代码**:
```c
void wd_rsa_get_pubkey(handle_t sess, struct wd_rsa_pubkey **pubkey);
void wd_rsa_get_prikey(handle_t sess, struct wd_rsa_prikey **prikey);
void wd_rsa_get_pubkey_params(struct wd_rsa_pubkey *pbk,
            struct wd_dtb **e, struct wd_dtb **n);
void wd_rsa_get_prikey_params(struct wd_rsa_prikey *pvk, struct wd_dtb **d,
            struct wd_dtb **n);
// ...
```

**分析**:
- 多个函数通过输出参数返回指针
- 调用者需要检查返回的指针是否为NULL
- 函数返回void，无法通过返回值判断成功/失败
- 实现代码中需要处理sess为无效句柄的情况

---

### 5. wd_rsa_req结构体设计 [INFO]

**位置**: wd_rsa.h:25-35

**代码**:
```c
struct wd_rsa_req {
    void *src; /* rsa operation input address */
    void *dst; /* rsa operation output address */
    __u32 src_bytes; /* rsa operation input bytes */
    __u32 dst_bytes; /* rsa operation output bytes */
    wd_rsa_cb_t cb;
    void *cb_param;
    int status; /* rsa operation status */
    __u8 data_fmt; /* data format denoted by enum wd_buff_type */
    __u8 op_type; /* rsa operation type */
};
```

**分析**:
- 结构体没有使用联合体，直接使用void* src/dst
- 这与wd_cipher_req等结构体的设计不同
- 对于RSA运算，通常不需要分散/聚集列表，设计合理

---

### 6. 正面发现 - 前向声明结构体 [POSITIVE]

**位置**: wd_rsa.h:37-40

**正面代码**:
```c
struct wd_rsa_kg_in; /* rsa key generation input parameters */
struct wd_rsa_kg_out; /* rsa key generation output parameters */
struct wd_rsa_pubkey; /* rsa public key */
struct wd_rsa_prikey; /* rsa private key */
```

**分析**:
- 使用前向声明隐藏实现细节
- 提供了良好的封装性
- 用户不需要了解内部结构定义

---

### 7. 正面发现 - API注释完善 [POSITIVE]

**位置**: wd_rsa.h:113-246

**正面代码示例**:
```c
/**
 * wd_rsa_init2_() - A simplify interface to initializate rsa.
 * This interface keeps most functions of
 * wd_rsa_init(). Users just need to descripe the deployment of
 * business scenarios. Then the initialization will request appropriate
 * resources to support the business scenarios.
 * To make the initializate simpler, ctx_params support set NULL.
 * And then the function will set them as default.
 * Please do not use this interface with wd_rsa_init() together, or
 * some resources may be leak.
 *
 * @alg: The algorithm users want to use.
 * @sched_type: The scheduling type users want to use.
 * @task_type: Reserved.
 * @ctx_params: The ctxs resources users want to use. Include per operation
 * type ctx numbers and business process run numa.
 *
 * Return 0 if succeed and others if fail.
 */
```

**分析**:
- API注释详细完整
- 包含参数说明和返回值
- 特别提醒了混用可能导致资源泄漏

---

## include/wd_rsa.h 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 0 |
| MEDIUM | 0 |
| LOW | 3 |
| INFO | 2 |
| POSITIVE | 2 |

**重点问题**:
- wd_rsa_op_type和wd_rsa_key_type枚举缺少MAX值(LOW)
- 宏定义可能存在整数溢出风险(LOW)

**建议改进**:
1. 为wd_rsa_op_type添加WD_RSA_OP_TYPE_MAX
2. 为wd_rsa_key_type添加WD_RSA_KEY_TYPE_MAX
3. 在文档中说明key_size的有效范围

---

## 检视进度更新

**已完成**: 28/94 文件 (29.8%)

**下次检视**: include/wd_dh.h (DH算法头文件)