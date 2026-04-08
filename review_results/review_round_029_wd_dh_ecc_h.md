# UADK代码检视报告 - 第29轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第29轮
- **规则来源**: E:/work_space/加速器用户态学习/代码检视/review_rules
- **使用Skills**: cpp-memory-safety, cpp-input-validation

---

## 本轮检视文件

### 已检视文件
1-28. 前序文件 (见review_progress.md)
29. include/wd_dh.h (本轮) - DH算法头文件，108行
30. include/wd_ecc.h (本轮) - ECC椭圆曲线算法头文件

### 待检视文件
- include/目录下其他头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## include/wd_dh.h 检视结果

### 1. 枚举缺少MAX值 - wd_dh_op_type [LOW]

**位置**: wd_dh.h:20-24

**问题类型**: 枚举设计 (cpp-coding-standards)

**代码**:
```c
enum wd_dh_op_type {
    WD_DH_INVALID, /* invalid DH operation */
    WD_DH_PHASE1, /* Phase1 DH key generate */
    WD_DH_PHASE2 /* Phase2 DH key compute */
};
```

**分析**:
- 枚举定义了3种DH操作类型
- 缺少WD_DH_OP_TYPE_MAX值用于边界检查

---

### 2. wd_dh_req结构体字段使用说明 [INFO]

**位置**: wd_dh.h:34-52

**代码**:
```c
struct wd_dh_req {
    void *x_p; /* x and p */

    /* it is g, but it is PV at phase 2 */
    void *pv;

    /* phase 1&&2 output */
    void *pri;
    __u16 pri_bytes; /* output bytes */

    __u16 pbytes; /* p bytes */
    __u16 xbytes; /* x bytes */
    __u16 pvbytes; /* pv bytes */
    wd_dh_cb_t cb;
    void *cb_param;
    int status; /* output status */
    __u8 op_type; /* operational type */
    __u8 data_fmt; /* data format denoted by enum wd_buff_type */
};
```

**分析**:
- x_p和pv字段在不同阶段有不同含义
- 注释说明了"it is g, but it is PV at phase 2"
- 建议在文档中更详细说明各阶段字段使用方式

---

## include/wd_ecc.h 检视结果

### 1. 枚举缺少MAX值 - wd_ecc_curve_id [LOW]

**位置**: wd_ecc.h:49-58

**问题类型**: 枚举设计 (cpp-coding-standards)

**代码**:
```c
enum wd_ecc_curve_id {
    WD_SECP128R1 = 0x10,
    WD_SECP192K1 = 0x11,
    WD_SECP224R1 = 0x12,
    WD_SECP256K1 = 0x13,
    WD_BRAINPOOLP320R1 = 0x14,
    WD_BRAINPOOLP384R1 = 0x15,
    WD_SECP384R1 = 0x16,
    WD_SECP521R1 = 0x17,
};
```

**分析**:
- 枚举定义了8种椭圆曲线
- 缺少WD_ECC_CURVE_ID_MAX值用于边界检查
- 使用显式值(0x10开始)，可能与硬件协议相关

---

### 2. 枚举缺少MAX值 - wd_ecc_curve_cfg_type [LOW]

**位置**: wd_ecc.h:81-84

**问题类型**: 枚举设计 (cpp-coding-standards)

**代码**:
```c
enum wd_ecc_curve_cfg_type {
    WD_CV_CFG_ID, /* set curve param by denote curve ID */
    WD_CV_CFG_PARAM /* set curve param by denote curve param */
};
```

**分析**:
- 枚举定义了2种曲线配置类型
- 缺少MAX值用于边界检查

---

### 3. wd_ecc_curve_cfg联合体使用 [LOW]

**位置**: wd_ecc.h:86-93

**问题类型**: 结构体设计 (cpp-memory-safety)

**代码**:
```c
struct wd_ecc_curve_cfg {
    __u32 type; /* denoted by enum wd_ecc_curve_cfg_type */
    union {
        enum wd_ecc_curve_id id; /* if WD_CV_CFG_ID */
        struct wd_ecc_curve *pparam; /* if WD_CV_CFG_PARAM */
    } cfg;
    __u8 resv[4]; /* reserve */
};
```

**分析**:
- 使用联合体表示id或pparam
- 需要根据type字段正确选择使用哪个成员
- 设计合理，但需要文档说明

---

### 4. 正面发现 - wd_ecc_op_type有MAX值 [POSITIVE]

**位置**: wd_ecc.h:34-46

**正面代码**:
```c
enum wd_ecc_op_type {
    WD_EC_OP_INVALID,
    WD_ECXDH_GEN_KEY,
    WD_ECXDH_COMPUTE_KEY,
    WD_ECDSA_SIGN,
    WD_ECDSA_VERIFY,
    WD_SM2_SIGN,
    WD_SM2_VERIFY,
    WD_SM2_ENCRYPT,
    WD_SM2_DECRYPT,
    WD_SM2_KG,
    WD_EC_OP_MAX
};
```

**分析**:
- wd_ecc_op_type正确包含WD_EC_OP_MAX
- 可以用于边界检查，设计正确

---

### 5. 正面发现 - wd_ecc_hash_type有MAX值 [POSITIVE]

**位置**: wd_ecc.h:61-71

**正面代码**:
```c
enum wd_ecc_hash_type {
    WD_HASH_SM3,
    WD_HASH_SHA1,
    WD_HASH_SHA224,
    WD_HASH_SHA256,
    WD_HASH_SHA384,
    WD_HASH_SHA512,
    WD_HASH_MD4,
    WD_HASH_MD5,
    WD_HASH_MAX
};
```

**分析**:
- wd_ecc_hash_type正确包含WD_HASH_MAX
- 设计一致

---

### 6. 回调函数定义 [INFO]

**位置**: wd_ecc.h:24-26

**代码**:
```c
typedef int (*wd_rand)(char *out, size_t out_len, void *usr);
typedef int (*wd_hash)(const char *in, size_t in_len,
               char *out, size_t out_len, void *usr);
```

**分析**:
- 定义了随机数和哈希回调函数类型
- 在wd_ecc_sess_setup中使用
- 使用时需要检查回调函数是否为NULL

---

## include/wd_dh.h & wd_ecc.h 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 0 |
| MEDIUM | 0 |
| LOW | 4 |
| INFO | 3 |
| POSITIVE | 2 |

**重点问题**:
- 多个枚举缺少MAX值(wd_dh_op_type, wd_ecc_curve_id, wd_ecc_curve_cfg_type)

**建议改进**:
1. 为wd_dh_op_type添加WD_DH_OP_TYPE_MAX
2. 为wd_ecc_curve_id添加WD_ECC_CURVE_ID_MAX
3. 为wd_ecc_curve_cfg_type添加MAX值

---

## 检视进度更新

**已完成**: 30/94 文件 (31.9%)

**下次检视**: include/uacce.h (UACCE用户态接口头文件)