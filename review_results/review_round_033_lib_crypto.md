# UADK代码检视报告 - 第33轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第33轮
- **规则来源**: E:/work_space/加速器用户态学习/代码检视/review_rules
- **使用Skills**: cpp-memory-safety, cpp-input-validation, cpp-coding-standards

---

## 本轮检视文件

### 已检视文件
1-32. 前序文件 (见review_progress.md)
33. lib/crypto/aes.c (本轮) - AES加密算法软件实现，410行
34. lib/crypto/galois.c (本轮) - Galois域乘法运算，95行
35. lib/crypto/sm4.c (本轮) - SM4国密算法实现，177行

### 待检视文件
- include/drv/目录下的头文件
- include/crypto/目录下的头文件
- v1/目录下其他源文件(不含test)

---

## lib/crypto/aes.c 检视结果

### 1. aes_encrypt函数参数未检查NULL [MEDIUM]

**位置**: lib/crypto/aes.c:399-409

**问题类型**: 参数验证 (cpp-input-validation)

**代码**:
```c
void aes_encrypt(__u8 *key, __u32 key_len, __u8 *src, __u8 *dst)
{
    struct aes_key local_key;
    int ret;

    ret = aes_set_encrypt_key(key, key_len << 0x3, &local_key);
    if (ret)
        return;

    aes_encrypt_(src, dst, &local_key);
}
```

**分析**:
- 公开API函数未检查key、src、dst是否为NULL
- 内部函数aes_set_encrypt_key正确检查了userkey和key参数(第372行)
- 但aes_encrypt直接将key传给aes_set_encrypt_key，如果key为NULL会在那里返回
- src和dst为NULL时会导致aes_encrypt_内部memcpy崩溃

---

### 2. aes_encrypt函数key_len未验证范围 [MEDIUM]

**位置**: lib/crypto/aes.c:404

**问题类型**: 参数验证 (cpp-input-validation)

**代码**:
```c
ret = aes_set_encrypt_key(key, key_len << 0x3, &local_key);
```

**分析**:
- key_len是字节长度，转换为比特后应只接受16/24/32(对应128/192/256位)
- aes_set_encrypt_key内部验证bits范围(第374行)
- 但建议在aes_encrypt入口处也添加早期验证

---

### 3. 正面发现 - aes_set_encrypt_key正确检查参数 [POSITIVE]

**位置**: lib/crypto/aes.c:372-375

**正面代码**:
```c
if (!userkey || !key)
    return -1;
if (bits != AES_128_BIT && bits != AES_192_BIT && bits != AES_256_BIT)
    return -1;
```

**分析**:
- 正确检查了NULL参数
- 正确验证了bits参数范围
- 这是良好的安全实践

---

## lib/crypto/galois.c 检视结果

### 4. galois_compute函数参数未检查NULL [MEDIUM]

**位置**: lib/crypto/galois.c:71-94

**问题类型**: 参数验证 (cpp-input-validation)

**代码**:
```c
void galois_compute(__u8 *S, __u8 *H, __u8 *g, __u32 len)
{
    unsigned int SL[GALOIS_UL_COUNT] = {0};
    unsigned int HL[GALOIS_UL_COUNT] = {0};
    unsigned int G[GALOIS_UL_COUNT] = {0};
    __u8 i, j;

    for (i = 0; i < GALOIS_UL_COUNT; i++) {
        j = i * GALOIS_UL_COUNT;
        SL[i] = GALOIS_PARA_TRANS_S(S, j);   // S可能为NULL
        j = GALOIS_UL_COUNT_H - j;
        HL[i] = GALOIS_PARA_TRANS_H(H, j);   // H可能为NULL
    }
```

**分析**:
- S、H、g参数未检查是否为NULL
- 如果这些参数为NULL会导致崩溃
- 这是GCM认证加密模式的关键组件，应有严格参数验证

---

### 5. galois_compute函数len参数边界问题 [MEDIUM]

**位置**: lib/crypto/galois.c:88

**问题类型**: 边界检查 (cpp-input-validation)

**代码**:
```c
j = len - GALOIS_UL_COUNT;
for (i = 0; i < GALOIS_UL_COUNT; i++) {
    GALOIS_UINT_TRANS_CHAR(&G[i]);
    memcpy(&g[j], &G[i], sizeof(unsigned int));  // 如果len<4，j可能为负
    j -= GALOIS_UL_COUNT;
}
```

**分析**:
- GALOIS_UL_COUNT定义为4
- 如果len小于4，j = len - 4会变成负数(__u8为无符号，会变成大正数)
- 导致memcpy写入到错误位置或越界

---

### 6. 栈上大数组使用 [INFO]

**位置**: lib/crypto/galois.c:53

**问题类型**: 内存使用 (cpp-coding-standards)

**代码**:
```c
static void galois_multi(unsigned int *input_a, unsigned int *input_b,
                         unsigned int *mul, unsigned int array_size)
{
    /* 4 * (unsigned int), 4 * 32bit = 128bit */
    unsigned int gf_V[GALOIS_BITS][GALOIS_UL_COUNT] = {0};  // 128*4=512个unsigned int
```

**分析**:
- gf_V数组大小为128*4=512个unsigned int = 2048字节
- 全部初始化为0，可能影响栈空间
- 对于嵌入式环境可能需要考虑栈大小限制

---

### 7. 正面发现 - 基于NIST标准的实现 [POSITIVE]

**位置**: lib/crypto/galois.c:36

**正面代码**:
```c
/* Based the NIST Special Publication 800-38D */
for (i = 0; i < GALOIS_UL_COUNT; i++) {
    tmp = src[i] >> 0x1;
```

**分析**:
- 实现基于NIST SP 800-38D标准
- 注释说明了算法来源
- 这是规范的加密实现实践

---

## lib/crypto/sm4.c 检视结果

### 8. sm4_encrypt函数参数未检查NULL [MEDIUM]

**位置**: lib/crypto/sm4.c:135-176

**问题类型**: 参数验证 (cpp-input-validation)

**代码**:
```c
void sm4_encrypt(__u8 *key, __u32 key_len, __u8 *input, __u8 *output)
{
    __u32 tmp_keys[U32_BYTES] = { 0 };
    __u32 keys[TOTAL_ROUND] = { 0 };
    ...
    if (key_len < AES_BLOCK_SIZE)
        return;

    for (i = 0; i < U32_BYTES; i++) {
        get_u32(key + U32_BYTES * i, &tmp_keys[i]);  // key可能为NULL
```

**分析**:
- key、input、output参数未检查是否为NULL
- 只检查了key_len的最小值，未检查其他参数有效性

---

### 9. sm4_encrypt函数key_len检查不完整 [LOW]

**位置**: lib/crypto/sm4.c:143

**问题类型**: 参数验证 (cpp-input-validation)

**代码**:
```c
if (key_len < AES_BLOCK_SIZE)
    return;
```

**分析**:
- 只检查了key_len不能小于16字节
- 未检查key_len的上限或有效范围
- SM4密钥应该是16字节，建议检查key_len == AES_BLOCK_SIZE

---

### 10. sm4_encrypt函数逻辑分析 [INFO]

**位置**: lib/crypto/sm4.c:155-163

**代码**:
```c
in_len = AES_BLOCK_SIZE;
for (i = 0; i < in_len; i++)
    *(p + i) = *(input + i);

k = AES_BLOCK_SIZE - (in_len % AES_BLOCK_SIZE);  // k = 16 - (16 % 16) = 0
for (i = 0; i < k; i++)
    *(p + in_len + i) = 0;
```

**分析**:
- in_len固定设置为AES_BLOCK_SIZE(16)
- k计算结果始终为0，padding循环不执行
- 这段代码可能是遗留的或为未来扩展准备的

---

### 11. 正面发现 - 使用国密标准S盒 [POSITIVE]

**位置**: lib/crypto/sm4.c:38-71

**正面代码**:
```c
const __u8 TBL_SBOX[256] = {
    0xd6, 0x90, 0xe9, 0xfe, 0xcc, 0xe1, ...
};
```

**分析**:
- S盒数据正确符合SM4国密标准
- 使用const确保数据不会被修改
- 系统参数和固定参数也正确定义

---

### 12. 正面发现 - 常量定义清晰 [POSITIVE]

**位置**: lib/crypto/sm4.c:23-36

**正面代码**:
```c
const __u32 TBL_SYS_PARAMS[4] = {
    0xa3b1bac6, 0x56aa3350, 0x677d9197, 0xb27022dc
};

const __u32 TBL_FIX_PARAMS[32] = {
    0x00070e15, 0x1c232a31, ...
};
```

**分析**:
- 系统参数FK和固定参数CK正确定义
- 符合SM4算法规范
- 使用const保证常量不变

---

## lib/crypto目录检视总结

| 文件 | 问题等级分布 |
|------|-------------|
| aes.c | MEDIUM 2, POSITIVE 1 |
| galois.c | MEDIUM 2, INFO 1, POSITIVE 1 |
| sm4.c | MEDIUM 1, LOW 1, INFO 1, POSITIVE 2 |
| **总计** | **MEDIUM 5, LOW 1, INFO 2, POSITIVE 4** |

**重点问题**:
- 所有公开API函数缺少参数NULL检查(MEDIUM)
- galois_compute的len参数可能导致数组越界(MEDIUM)

**加密代码特殊关注**:
- 密钥处理和内存清除
- 常数时间执行(未发现明显问题)
- 基于标准实现(正面发现)

**建议修复**:
1. aes_encrypt: 添加key、src、dst参数NULL检查
2. galois_compute: 添加S、H、g参数NULL检查，验证len >= GALOIS_UL_COUNT
3. sm4_encrypt: 添加key、input、output参数NULL检查，验证key_len == 16

---

## 检视进度更新

**已完成**: 35/94 文件 (37.2%)

**下次检视**: include/crypto/目录下的头文件