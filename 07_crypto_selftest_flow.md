# Crypto API 自测试流程分析

## 一、整体流程图

```
驱动加载 (hisi_zip.ko)
    ↓
hisi_zip_probe()
    ↓
hisi_zip_register_to_crypto()
    ↓
crypto_register_acomp(&hisi_zip_acomp_deflate)
    ↓
crypto_register_alg() [algapi.c]
    ↓
创建 larval (临时占位算法)
    ↓
CRYPTO_MSG_ALG_REGISTER 通知
    ↓
cryptomgr_schedule_test() [algboss.c]
    ↓
创建 cryptomgr_test 内核线程
    ↓
alg_test() [testmgr.c]
    ↓
alg_test_comp()
    ↓
crypto_alloc_acomp() → 调用 hisi_zip_acomp_init()
    ↓
test_acomp() → 压缩/解压测试
    ↓
crypto_alg_tested() → 设置 CRYPTO_ALG_TESTED 标志
    ↓
算法可用，用户可正常使用
```

## 二、代码路径详解

### 2.1 驱动注册算法

**文件: zip_crypto.c:650**
```c
static int hisi_zip_register_deflate(struct hisi_qm *qm)
{
    ret = crypto_register_acomp(&hisi_zip_acomp_deflate);
    // ...
}
```

**文件: acompress.c:196**
```c
int crypto_register_acomp(struct acomp_alg *alg)
{
    struct crypto_alg *base = &alg->calg.base;

    comp_prepare_alg(&alg->calg);

    base->cra_type = &crypto_acomp_type;
    base->cra_flags |= CRYPTO_ALG_TYPE_ACOMPRESS;

    return crypto_register_alg(base);  // ← 调用底层注册
}
```

### 2.2 创建 Larval 并触发测试

**文件: algapi.c:350-361**
```c
// crypto_register_alg 的核心逻辑
larval = crypto_larval_add(alg->cra_name, alg->cra_flags, alg->cra_refcnt);
if (IS_ERR(larval))
    goto out;

list_add(&alg->cra_list, &crypto_alg_list);

if (larval) {
    /* No cheating! */
    alg->cra_flags &= ~CRYPTO_ALG_TESTED;  // ← 清除测试标志，必须通过测试

    list_add(&larval->alg.cra_list, &crypto_alg_list);
}
```

关键概念：**larval（幼虫）** 是一个临时占位对象，表示算法正在等待测试。

**文件: algapi.c:298**
```c
// 注册时发送通知
crypto_notifier.call_chain(CRYPTO_MSG_ALG_REGISTER, larval);
```

### 2.3 测试线程启动

**文件: algboss.c:188-219**
```c
static int cryptomgr_schedule_test(struct crypto_alg *alg)
{
    struct task_struct *thread;
    struct crypto_test_param *param;

    param = kzalloc(sizeof(*param), GFP_KERNEL);

    // 复制算法名称
    memcpy(param->driver, alg->cra_driver_name, sizeof(param->driver));
    memcpy(param->alg, alg->cra_name, sizeof(param->alg));
    param->type = alg->cra_flags;

    // 创建测试线程
    thread = kthread_run(cryptomgr_test, param, "cryptomgr_test");
    // ...
}
```

**文件: algboss.c:174-186**
```c
static int cryptomgr_test(void *data)
{
    struct crypto_test_param *param = data;
    u32 type = param->type;
    int err;

    // 执行测试！
    err = alg_test(param->driver, param->alg, type, CRYPTO_ALG_TESTED);

    // 测试完成后回调
    crypto_alg_tested(param->driver, err);

    kfree(param);
    module_put_and_kthread_exit(0);
}
```

### 2.4 alg_test 入口

**文件: testmgr.c:63-66 (测试禁用版本)**
```c
#ifdef CONFIG_CRYPTO_MANAGER_DISABLE_TESTS
int alg_test(const char *driver, const char *alg, u32 type, u32 mask)
{
    return 0;  // 直接返回，不做测试
}
#else
```

**文件: testmgr.c (正常版本)**
```c
int alg_test(const char *driver, const char *alg, u32 type, u32 mask)
{
    // 根据算法名查找测试描述
    const struct alg_test_desc *test_desc;

    for (test_desc = alg_test_descs; test_desc->alg; test_desc++) {
        if (!strcmp(test_desc->alg, alg)) {
            // 执行对应测试函数
            return test_desc->test(test_desc, driver, type, mask);
        }
    }
    // ...
}
```

### 2.5 alg_test_desc 表

**文件: testmgr.c:4307**
```c
static const struct alg_test_desc alg_test_descs[] = {
    // deflate 算法的测试描述
    {
        .alg = "deflate",
        .test = alg_test_comp,  // ← 压缩算法测试函数
        .suite = {
            .comp = {
                .comp = {
                    .vecs = deflate_comp_tv_template,  // 压缩测试向量
                    .count = DEFLATE_COMP_TEST_VECTORS
                },
                .decomp = {
                    .vecs = deflate_decomp_tv_template,  // 解压测试向量
                    .count = DEFLATE_DECOMP_TEST_VECTORS
                }
            }
        }
    },
    // ... 其他算法
};
```

### 2.6 alg_test_comp 函数

**文件: testmgr.c:3686-3722**
```c
static int alg_test_comp(const struct alg_test_desc *desc,
                          const char *driver, u32 type, u32 mask)
{
    struct crypto_acomp *acomp;
    int err;
    u32 algo_type = type & CRYPTO_ALG_TYPE_ACOMPRESS_MASK;

    if (algo_type == CRYPTO_ALG_TYPE_ACOMPRESS) {
        // 分配异步压缩算法实例
        acomp = crypto_alloc_acomp(driver, type, mask);
        if (IS_ERR(acomp)) {
            pr_err("alg: acomp: Failed to load transform for %s: %ld\n",
                   driver, PTR_ERR(acomp));
            return PTR_ERR(acomp);
        }

        // 执行测试！
        err = test_acomp(acomp,
                         desc->suite.comp.comp.vecs,   // 压缩向量
                         desc->suite.comp.decomp.vecs, // 解压向量
                         desc->suite.comp.comp.count,
                         desc->suite.comp.decomp.count);

        crypto_free_acomp(acomp);
    }
    return err;
}
```

### 2.7 test_acomp 实际测试逻辑

**文件: testmgr.c:3404-3592**
```c
static int test_acomp(struct crypto_acomp *tfm,
                      const struct comp_testvec *ctemplate,  // 压缩测试向量
                      const struct comp_testvec *dtemplate,  // 解压测试向量
                      int ctcount, int dtcount)
{
    const char *algo = crypto_tfm_alg_driver_name(crypto_acomp_tfm(tfm));
    struct acomp_req *req;
    struct crypto_wait wait;

    // ========== 压缩测试循环 ==========
    for (i = 0; i < ctcount; i++) {
        int ilen = ctemplate[i].inlen;
        unsigned int dlen = COMP_BUF_SIZE;

        // 准备输入数据
        input_vec = kmemdup(ctemplate[i].input, ilen, GFP_KERNEL);

        sg_init_one(&src, input_vec, ilen);
        sg_init_one(&dst, output, dlen);

        // 分配请求对象
        req = acomp_request_alloc(tfm);

        acomp_request_set_params(req, &src, &dst, ilen, dlen);
        acomp_request_set_callback(req, CRYPTO_TFM_REQ_MAY_BACKLOG,
                                   crypto_req_done, &wait);

        // ★ 执行压缩！调用 hisi_zip_acompress → hisi_zip_do_work → hisi_qp_send
        ret = crypto_wait_req(crypto_acomp_compress(req), &wait);
        if (ret) {
            pr_err("alg: acomp: compression failed on test %d for %s: ret=%d\n",
                   i + 1, algo, -ret);
            goto out;
        }

        compressed_len = req->dlen;

        // ========== 解压验证 ==========
        sg_init_one(&src, output, compressed_len);
        sg_init_one(&dst, decomp_out, COMP_BUF_SIZE);
        acomp_request_set_params(req, &src, &dst, compressed_len, COMP_BUF_SIZE);

        // ★ 执行解压！调用 hisi_zip_adecompress
        ret = crypto_wait_req(crypto_acomp_decompress(req), &wait);
        if (ret) {
            pr_err("alg: acomp: decompression failed on test %d for %s: ret=%d\n",
                   i + 1, algo, -ret);
            goto out;
        }

        // ========== 数据完整性验证 ==========
        if (req->dlen != ctemplate[i].inlen) {
            pr_err("alg: acomp: Compression test %d failed for %s: output len = %d\n",
                   i + 1, algo, req->dlen);
            goto out;
        }

        // 比较解压后数据是否与原始数据一致
        if (memcmp(input_vec, decomp_out, req->dlen)) {
            pr_err("alg: acomp: Compression test %d failed for %s\n",
                   i + 1, algo);
            goto out;
        }

        acomp_request_free(req);
    }

    // ========== 解压测试循环 ==========
    for (i = 0; i < dtcount; i++) {
        // 类似逻辑，测试解压功能
        // ...
    }

    return 0;
}
```

### 2.8 测试向量定义

**文件: testmgr.h:35685**
```c
static const struct comp_testvec deflate_comp_tv_template[] = {
    {
        .inlen  = 70,
        .outlen = 38,
        .input  = "Join us now and share the software "
                  "Join us now and share the software ",
        .output = "\xf3\xca\xcf\xcc\x53\x28\x2d\x56"
                  "\xc8\xcb\x2f\x57\x48\xcc\x4b\x51"
                  // ... 预期的压缩输出
    },
    {
        .inlen  = 191,
        .outlen = 122,
        .input  = "This document describes a compression method...",
        .output = "\x5d\x8d\x31\x0e\xc2\x30\x10\x04"
                  // ...
    },
};
```

### 2.9 测试完成回调

**文件: algapi.c:373-429**
```c
void crypto_alg_tested(const char *name, int err)
{
    struct crypto_larval *test;
    struct crypto_alg *alg;

    // 找到对应的 larval
    list_for_each_entry(q, &crypto_alg_list, cra_list) {
        if (!strcmp(q->cra_driver_name, name))
            goto found;
    }

found:
    alg = test->adult;  // 真正的算法

    if (err) {
        // 测试失败，算法标记为 DEAD
        goto complete;
    }

    // ★ 测试成功！设置 TESTED 标志
    alg->cra_flags |= CRYPTO_ALG_TESTED;

    // 完成注册，唤醒等待的用户
    crypto_alg_finish_registration(alg, best, &list);
}
```

## 三、时序图

```
时间线:

T1: insmod hisi_zip.ko
    │
T2: hisi_zip_probe()
    │   → 初始化硬件
    │   → hisi_zip_register_to_crypto()
    │
T3: crypto_register_acomp()
    │   → crypto_register_alg()
    │   → 创建 larval (占位符)
    │   → alg->cra_flags &= ~CRYPTO_ALG_TESTED  ← 此时算法未测试
    │   → 发送 CRYPTO_MSG_ALG_REGISTER 通知
    │
T4: cryptomgr_schedule_test()
    │   → kthread_run(cryptomgr_test)  ← 创建测试线程
    │
T5: cryptomgr_test 线程执行
    │   → alg_test("hisi-deflate-acomp", "deflate", ...)
    │   → alg_test_comp()
    │
T6: crypto_alloc_acomp()
    │   → hisi_zip_acomp_init()  ← ★ 你的代码在这里执行！
    │       → hisi_zip_create_req_q()
    │       → hisi_zip_set_acomp_cb(ctx, hisi_zip_acomp_cb)
    │       → QP 创建，回调设置，SQE 未清零 ← ★ 问题点！
    │
T7: test_acomp()
    │   → crypto_acomp_compress(req)  ← 第一次压缩请求
    │       → hisi_zip_acompress()
    │       → hisi_zip_do_work()
    │       → hisi_zip_fill_sqe() → memset(sqe, 0) ← SQE 第一次被清零！
    │       → hisi_zip_fill_tag() ← 正确的 tag 写入
    │       → hisi_qp_send()
    │   → 等待完成...
    │   → hisi_zip_acomp_cb() ← 回调执行，此时 SQE 是正确的
    │
T8: 测试成功/失败
    │   → crypto_alg_tested()
    │   → if success: alg->cra_flags |= CRYPTO_ALG_TESTED
    │   → 算法可用
    │
T9: 用户可以正常使用 crypto API
```

## 四、关键发现：为什么自测可能掩盖问题

### 4.1 自测流程中的 SQE 清零时机

在自测流程中：

```
crypto_alloc_acomp()
    → hisi_zip_acomp_init()        // QP创建，SQE未清零
    → hisi_zip_set_acomp_cb()      // 回调设置，此时SQE有残留数据！

test_acomp()
    → crypto_acomp_compress()      // 第一次请求
        → hisi_zip_fill_sqe()
            → memset(sqe, 0)       // ★ SQE 在这里才被清零！
```

**问题：在 T6 (hisi_zip_acomp_init) 和 T7 (第一次请求) 之间，如果硬件产生中断，回调会被调用，读取残留 tag → 崩溃！**

### 4.2 自测掩盖问题的原因

自测会立即发送压缩请求，SQE 很快被清零，掩盖了残留数据的问题。

但实际场景中：
```
rmmod hisi_zip    → DMA内存释放，数据残留
insmod hisi_zip   → DMA内存可能复用，SQE有残留
                   → hisi_zip_acomp_init() 回调设置
                   → ★ 等待用户首次请求...
                   → 如果此时硬件产生虚假中断 → 崩溃！
                   → 用户发送请求 → SQE清零（太晚了）
```

### 4.3 为什么几百次测试都没复现

因为自测流程太快了：
- crypto_alloc_acomp() → hisi_zip_acomp_init() → 回调设置
- 立即 test_acomp() → 第一次请求 → SQE清零

间隔可能只有几毫秒，硬件来不及产生虚假中断。

而真实崩溃场景需要：
- 回调设置后，等待足够长时间
- 硬件恰好在此期间产生中断
- 这个概率极低，但确实存在

## 五、修复建议

### 方案：在 QP 创建时清零 SQE

```c
/* 文件: qm.c */
static struct hisi_qp *qm_create_qp_nolock(struct hisi_qm *qm, u8 alg_type,
                                            bool is_in_kernel)
{
    qp = &qm->qp_array[qp_id];
    hisi_qm_unset_hw_reset(qp);

    /* ★ 修复：清零 SQE，防止残留 tag 导致崩溃 */
    memset(qp->sqe, 0, qm->sqe_size * qp->sq_depth);

    memset(qp->cqe, 0, sizeof(struct qm_cqe) * qp->cq_depth);
    // ...
}
```

这样在 hisi_zip_acomp_init() → 回调设置 时，SQE 已经是干净的，即使硬件产生虚假中断，tag=0，回调可以安全检测并退出。

## 六、调试建议

### 添加自测中的 SQE 检查

可以在 hisi_zip_acomp_init() 中添加检查：

```c
static int hisi_zip_acomp_init(struct crypto_acomp *tfm)
{
    // ... QP创建 ...

    // ★ 在回调设置前检查 SQE
    struct hisi_zip_sqe *sqe = ctx->qp_ctx[0].qp->sqe;
    if (sqe->dw26 != 0 || sqe->dw27 != 0) {
        dev_err(dev, "SELFTEST: SQE has residual tag before first request!\n");
        dev_err(dev, "SELFTEST: tag=0x%llx, this would crash on interrupt!\n",
                (u64)sqe->dw26 | ((u64)sqe->dw27 << 32));
    }

    hisi_zip_set_acomp_cb(ctx, hisi_zip_acomp_cb);
    // ...
}
```

这个检查会在自测期间执行，帮助发现残留数据问题。