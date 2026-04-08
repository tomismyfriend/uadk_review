# UADK代码检视报告 - 第32轮

## 检视摘要

- **检视日期**: 2026-04-07
- **检视轮次**: 第32轮
- **规则来源**: E:/work_space/加速器用户态学习/代码检视/review_rules
- **使用Skills**: cpp-memory-safety, cpp-resource-management, cpp-input-validation, cpp-api-misuse

---

## 本轮检视文件

### 已检视文件
1-31. 前序文件 (见review_progress.md)
32. sample/uadk_comp.c (本轮) - 压缩示例程序，506行

### 待检视文件
- v1/目录下其他源文件(不含test)
- lib/crypto/目录下的文件
- include/drv/和include/crypto/目录下的头文件

---

## sample/uadk_comp.c 检视结果

### 1. get_dev_list函数参数未检查NULL [MEDIUM]

**位置**: sample/uadk_comp.c:77-86

**问题类型**: 参数验证 (cpp-input-validation)

**代码**:
```c
static struct uacce_dev_list* get_dev_list(char *alg_name)
{
    struct uacce_dev_list *list, *p, *head = NULL, *prev = NULL;
    int ctx_set_num = CTX_SET_NUM;
    int max_ctx_num;
    int i;

    for (i = 0; i < ARRAY_SIZE(alg_options); i++)
        if (!strcmp(alg_name, alg_options[i].name))  // alg_name可能为NULL
            config.alg = alg_options[i].alg;
```

**分析**:
- 未检查alg_name参数是否为NULL
- 如果alg_name为NULL，strcmp会崩溃
- 建议在函数开头添加NULL检查

---

### 2. wd_request_ctx调用前未检查config.list->dev [MEDIUM]

**位置**: sample/uadk_comp.c:200

**问题类型**: 空指针检查 (cpp-memory-safety)

**代码**:
```c
ctx->ctxs[j].ctx = wd_request_ctx(config.list->dev);
if (!ctx->ctxs[j].ctx) {
    fprintf(stderr, "%s fail to request context #%d.\n",
        __func__, i);
    ret = -WD_EINVAL;
    goto out_free_ctx;
}
```

**分析**:
- 虽然检查了wd_request_ctx返回值
- 但未检查config.list->dev是否为NULL
- 如果config.list为NULL或config.list->dev为NULL会导致问题
- config.list在main函数的get_dev_list调用中设置，但dev成员未验证

---

### 3. 整数溢出风险 - dst_len计算 [MEDIUM]

**位置**: sample/uadk_comp.c:353

**问题类型**: 整数溢出 (cpp-integer-overflow)

**代码**:
```c
req->src_len = fs.st_size;
req->dst_len = fs.st_size * 4;
```

**分析**:
- fs.st_size是off_t类型，可能很大
- fs.st_size * 4可能溢出int范围
- req->dst_len是__u32类型，如果fs.st_size超过1GB，乘4会溢出
- 建议添加文件大小限制检查

---

### 4. strcpy缓冲区溢出风险 [HIGH]

**位置**: sample/uadk_comp.c:476

**问题类型**: 缓冲区溢出 (cpp-memory-safety)

**代码**:
```c
strcpy(config.algname, optarg);
```

**分析**:
- config.algname是MAX_ALG_LEN(32)字节的数组
- optarg长度未限制，可能超过32字节
- 应使用strncpy或检查optarg长度
- 这是典型的缓冲区溢出漏洞

---

### 5. strtol使用缺少errno初始化 [LOW]

**位置**: sample/uadk_comp.c:480, 483, 486

**问题类型**: 错误处理 (cpp-input-validation)

**代码**:
```c
case 2:
    config.complv = strtol(optarg, NULL, 0);
    break;
case 3:
    config.optype = strtol(optarg, NULL, 0);
    break;
case 4:
    config.winsize = strtol(optarg, NULL, 0);
    break;
```

**分析**:
- strtol调用前未设置errno = 0
- 无法正确检测溢出错误
- strtol返回值未做范围验证
- config.complv、config.optype、config.winsize是枚举类型，应验证范围

---

### 6. fwrite返回值类型不匹配 [LOW]

**位置**: sample/uadk_comp.c:378-379

**问题类型**: 类型不匹配 (cpp-coding-standards)

**代码**:
```c
size = fwrite(data.req.dst, 1, data.req.dst_len, dest);
if (size < 0)
    return size;
```

**分析**:
- fwrite返回size_t类型，不是int
- size_t是无符号类型，永远不会小于0
- 应检查size != data.req.dst_len来判断写入是否完整

---

### 7. uadk_comp_sess_init错误处理路径问题 [MEDIUM]

**位置**: sample/uadk_comp.c:268-282

**问题类型**: 错误处理逻辑 (cpp-resource-management)

**代码**:
```c
h_sess = wd_comp_alloc_sess(&setup);
if (!h_sess) {
    fprintf(stderr, "%s fail to alloc comp sess.\n", __func__);
    ret = -WD_EINVAL;
    goto out_free_sess;
}
data.h_sess = h_sess;

return 0;

out_free_sess:
wd_comp_free_sess(data.h_sess);  // data.h_sess此时可能未初始化
```

**分析**:
- 如果wd_comp_alloc_sess失败，跳转到out_free_sess
- 此时data.h_sess未被设置为新分配的h_sess
- wd_comp_free_sess(data.h_sess)可能释放旧的/无效的句柄
- 应改为wd_comp_free_sess(h_sess)或确保data.h_sess初始化为NULL

---

### 8. 枚举值未验证范围 [LOW]

**位置**: sample/uadk_comp.c:480-486

**问题类型**: 输入验证 (cpp-input-validation)

**代码**:
```c
config.complv = strtol(optarg, NULL, 0);
config.optype = strtol(optarg, NULL, 0);
config.winsize = strtol(optarg, NULL, 0);
```

**分析**:
- complv是enum wd_comp_level类型
- optype是enum wd_comp_op_type类型
- winsize是enum wd_comp_winsz_type类型
- 用户输入的值未验证是否在有效枚举范围内

---

### 9. 正面发现 - 内存分配正确检查 [POSITIVE]

**位置**: sample/uadk_comp.c:192-196, 296-300, 302-307

**正面代码**:
```c
ctx->ctxs = calloc(ctx_set_num * CTX_SET_SIZE, sizeof(struct wd_ctx));
if (!ctx->ctxs) {
    fprintf(stderr, "%s fail to allocate contexts.\n", __func__);
    return -WD_ENOMEM;
}

src = malloc(src_len);
if (!src) {
    fprintf(stderr, "%s fail to alloc src.\n", __func__);
    return -WD_ENOMEM;
}
```

**分析**:
- 所有malloc/calloc调用后都正确检查了返回值
- 内存分配失败时有正确的错误处理和资源释放

---

### 10. 正面发现 - 资源释放顺序正确 [POSITIVE]

**位置**: sample/uadk_comp.c:385-426

**正面代码**:
```c
static int operation(FILE *source, FILE *dest)
{
    int ret;

    ret = uadk_comp_ctx_init();
    if (ret) {
        fprintf(stderr, "%s fail to init ctx!\n", __func__);
        return ret;
    }

    ret = uadk_comp_sess_init();
    if (ret) {
        fprintf(stderr, "%s fail to init sess!\n", __func__);
        goto out_ctx_uninit;
    }
    // ...
out_sess_uninit:
    uadk_comp_sess_uninit();

out_ctx_uninit:
    uadk_comp_ctx_uninit();

    return ret;
}
```

**分析**:
- 使用goto实现正确的资源释放顺序
- 初始化顺序：ctx -> sess -> request
- 释放顺序：request -> sess -> ctx
- 遵循了RAII风格的错误处理

---

### 11. 正面发现 - 文件操作错误处理 [POSITIVE]

**位置**: sample/uadk_comp.c:342-347, 309-314

**正面代码**:
```c
fd = fileno(source);
ret = fstat(fd, &fs);
if (ret < 0) {
    fprintf(stderr, "%s fstat error.\n", __func__);
    return ret;
}

ret = fread(src, 1, src_len, source);
if (ret != src_len) {
    fprintf(stderr, "%s fail to read stdin.\n", __func__);
    ret = -WD_ENOMEM;
    goto out_free_dst;
}
```

**分析**:
- fstat和fread都有错误检查
- fread检查了实际读取长度与期望长度是否一致

---

### 12. cowfail函数创意错误提示 [INFO]

**位置**: sample/uadk_comp.c:63-75

**代码**:
```c
static void cowfail(char *s)
{
    fprintf(stderr, ""
        "__________________________________\n\n"
        "%s"
        "\n----------------------------------\n"
        "\t        \\   ^__^\n"
        "\t         \\  (oo)\\_______\n"
        "\t            (__)\\       )\\\\\n"
        "\t                ||----w |\n"
        "\t                ||     ||\n"
        "\n", s);
}
```

**分析**:
- 使用ASCII艺术显示错误信息
- 提供了友好的用户体验
- 但s参数未检查NULL

---

## sample/uadk_comp.c 检视总结

| 问题等级 | 数量 |
|---------|------|
| HIGH | 1 |
| MEDIUM | 4 |
| LOW | 3 |
| INFO | 1 |
| POSITIVE | 3 |

**重点问题**:
- strcpy缓冲区溢出风险(HIGH) - optarg长度未限制
- 多处参数NULL检查缺失(MEDIUM)
- 整数溢出风险 - 文件大小乘4可能溢出(MEDIUM)

**建议修复**:
1. 将strcpy替换为strncpy并添加长度检查
2. 在get_dev_list添加alg_name NULL检查
3. 在uadk_comp_ctx_init添加config.list->dev检查
4. 修复uadk_comp_sess_init错误处理路径
5. 添加文件大小限制防止整数溢出
6. strtol调用前设置errno=0并验证枚举范围

---

## 检视进度更新

**已完成**: 32/94 文件 (34.0%)

**下次检视**: lib/crypto目录下的文件