# UADK代码检视报告 - 第25轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第25轮
- **规则来源**: E:/work_space/加速器用户态学习/代码检视/review_rules
- **使用Skills**: cpp-memory-safety, cpp-input-validation

---

## 本轮检视文件

### 已检视文件
1-24. 前序文件 (见review_progress.md)
25. include/wd_digest.h (本轮) - 摘要算法头文件，291行

### 待检视文件
- include/目录下其他头文件
- lib/crypto/目录下的文件
- v1/目录下的文件(不含test)
- sample/目录下的文件

---

## include/wd_digest.h 检视结果

### 1. 枚举数量不一致问题 - wd_digest_mac_full_len [HIGH]

**位置**: wd_digest.h:21-65

**问题类型**: 数组越界风险 (cpp-memory-safety)

**代码**:
```c
enum wd_digest_type {
    WD_DIGEST_SM3,              // 0
    WD_DIGEST_MD5,              // 1
    WD_DIGEST_SHA1,             // 2
    WD_DIGEST_SHA256,           // 3
    WD_DIGEST_SHA224,           // 4
    WD_DIGEST_SHA384,           // 5
    WD_DIGEST_SHA512,           // 6
    WD_DIGEST_SHA512_224,       // 7
    WD_DIGEST_SHA512_256,       // 8
    WD_DIGEST_AES_XCBC_MAC_96,  // 9
    WD_DIGEST_AES_XCBC_PRF_128, // 10
    WD_DIGEST_AES_CMAC,         // 11
    WD_DIGEST_AES_GMAC,         // 12
    WD_DIGEST_TYPE_MAX,         // 13
};

/* User need to input full mac len in first and middle hash */
enum wd_digest_mac_full_len {
    WD_DIGEST_SM3_FULL_LEN      = 32,  // 0
    WD_DIGEST_MD5_FULL_LEN      = 16,  // 1
    WD_DIGEST_SHA1_FULL_LEN     = 20,  // 2
    WD_DIGEST_SHA256_FULL_LEN   = 32,  // 3
    WD_DIGEST_SHA224_FULL_LEN   = 32,  // 4
    WD_DIGEST_SHA384_FULL_LEN   = 64,  // 5
    WD_DIGEST_SHA512_FULL_LEN   = 64,  // 6
    WD_DIGEST_SHA512_224_FULL_LEN = 64, // 7
    WD_DIGEST_SHA512_256_FULL_LEN = 64, // 8
    // 缺少 AES_XCBC_MAC_96, AES_XCBC_PRF_128, AES_CMAC, AES_GMAC
};
```

**分析**:
- wd_digest_type定义了14种类型(包括MAX)
- wd_digest_mac_full_len只定义了9种长度
- 这会导致在wd_digest.c中g_digest_mac_full_len数组只有9个元素
- 但代码可能使用WD_DIGEST_AES_XCBC_MAC_96等作为索引访问，导致数组越界

**风险**:
- 与第7轮wd_digest.c检视发现的问题相同
- 访问索引9-12会导致数组越界

**修复建议**:
```c
enum wd_digest_mac_full_len {
    WD_DIGEST_SM3_FULL_LEN          = 32,
    WD_DIGEST_MD5_FULL_LEN          = 16,
    WD_DIGEST_SHA1_FULL_LEN         = 20,
    WD_DIGEST_SHA256_FULL_LEN       = 32,
    WD_DIGEST_SHA224_FULL_LEN       = 32,
    WD_DIGEST_SHA384_FULL_LEN       = 64,
    WD_DIGEST_SHA512_FULL_LEN       = 64,
    WD_DIGEST_SHA512_224_FULL_LEN   = 64,
    WD_DIGEST_SHA512_256_FULL_LEN   = 64,
    WD_DIGEST_AES_XCBC_MAC_96_FULL_LEN   = 12,   // 添加
    WD_DIGEST_AES_XCBC_PRF_128_FULL_LEN = 16,    // 添加
    WD_DIGEST_AES_CMAC_FULL_LEN     = 16,        // 添加
    WD_DIGEST_AES_GMAC_FULL_LEN     = 16,        // 添加
};
```

---

### 2. wd_digest_req结构体联合体使用风险 [LOW]

**位置**: wd_digest.h:129-146

**问题类型**: 结构体设计 (cpp-memory-safety)

**代码**:
```c
struct wd_digest_req {
    union {
        void *in;
        struct wd_datalist *list_in;
    };
    void        *out;
    __u32       in_bytes;
    __u32       out_bytes;
    __u32       out_buf_bytes;
    __u8        *iv;
    __u32       iv_bytes;
    __u16       state;
    __u16       has_next;
    __u8        data_fmt;
    wd_digest_cb_t  *cb;
    void        *cb_param;
    __u64       long_data_len;
};
```

**分析**:
- 使用联合体表示in/list_in
- 需要根据data_fmt正确选择使用哪个成员
- 与wd_cipher_req结构体相同的设计模式

---

### 3. 回调函数定义简化 [INFO]

**位置**: wd_digest.h:107

**代码**:
```c
typedef void *wd_digest_cb_t(void *cb_param);
```

**分析**:
- 回调函数定义比wd_cipher的简单
- 只接收cb_param参数
- 需要在实现中检查cb是否为NULL

---

### 4. 正面发现 - wd_digest_mac_len定义完整 [POSITIVE]

**位置**: wd_digest.h:38-52

**正面代码**:
```c
enum wd_digest_mac_len {
    WD_DIGEST_SM3_LEN               = 32,
    WD_DIGEST_MD5_LEN               = 16,
    WD_DIGEST_SHA1_LEN              = 20,
    WD_DIGEST_SHA256_LEN            = 32,
    WD_DIGEST_SHA224_LEN            = 28,
    WD_DIGEST_SHA384_LEN            = 48,
    WD_DIGEST_SHA512_LEN            = 64,
    WD_DIGEST_SHA512_224_LEN        = 28,
    WD_DIGEST_SHA512_256_LEN        = 32,
    WD_DIGEST_AES_XCBC_MAC_96_LEN   = 12,
    WD_DIGEST_AES_XCBC_PRF_128_LEN  = 16,
    WD_DIGEST_AES_CMAC_LEN          = 16,
    WD_DIGEST_AES_GMAC_LEN          = 16,
};
```

**分析**:
- wd_digest_mac_len枚举定义了所有13种摘要类型的长度
- 与wd_digest_type一一对应
- 这是正确的设计

---

### 5. 正面发现 - 消息状态定义清晰 [POSITIVE]

**位置**: wd_digest.h:79-92

**正面代码**:
```c
/**
 * wd_digest_msg_state - Message state of digest
 * @WD_DIGEST_END: Final message or single block message
 * @WD_DIGEST_DOING: First message or middle message
 * @WD_DIGEST_STREAM_END: Final message carrying long data information
 * @WD_DIGEST_STREAM_DOING: Middle message carrying long data information
 */
enum wd_digest_msg_state {
    WD_DIGEST_END = 0x0,
    WD_DIGEST_DOING,
    WD_DIGEST_STREAM_END,
    WD_DIGEST_STREAM_DOING,
    WD_DIGEST_MSG_STATE_MAX,
};
```

**分析**:
- 消息状态定义清晰，有详细注释
- 包含MAX值用于边界检查

---

## include/wd_digest.h 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 1 |
| MEDIUM | 0 |
| LOW | 1 |
| INFO | 1 |
| POSITIVE | 2 |

**重点问题**:
- wd_digest_mac_full_len枚举缺少AES相关摘要类型的定义(HIGH)
- 这与wd_digest.c中发现的数组越界问题直接相关

**建议修复**:
1. 在wd_digest_mac_full_len中添加AES_XCBC、AES_CMAC、AES_GMAC的定义
2. 或者重新设计长度获取方式，使用函数而非数组索引

---

## 检视进度更新

**已完成**: 25/94 文件 (26.6%)

**下次检视**: include/wd_aead.h (认证加密头文件)