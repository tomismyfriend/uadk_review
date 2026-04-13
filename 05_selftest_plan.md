# hisi_zip 自测试方案

## 一、测试目标

针对 SQE 未清零导致的 kernel panic 问题，设计以下自测试：

| 测试项 | 目的 | 验证内容 |
|-------|------|---------|
| SQE 初始化测试 | 验证 SQE 创建时是否清零 | dw26/dw27 必须为 0 |
| rmmod/insmod 稳定性测试 | 验证驱动重复加载稳定性 | 100次循环不崩溃 |
| 回调时机测试 | 验证回调只在有效请求时调用 | 无请求时回调不被触发 |
| 正常业务测试 | 验证压缩解压功能正常 | 数据完整性验证 |

## 二、内核态自测试

### 2.1 SQE 初始化自测（驱动层）

在 QP 创建时添加自测检查：

```c
/* 文件: drivers/crypto/hisilicon/qm.c */
static struct hisi_qp *qm_create_qp_nolock(struct hisi_qm *qm, u8 alg_type,
                                            bool is_in_kernel)
{
    qp = &qm->qp_array[qp_id];
    hisi_qm_unset_hw_reset(qp);

    /* 自测试：验证 SQE 初始化后 tag 字段为 0 */
    {
        int slot;
        u64 tag;
        int selftest_passed = 1;

        for (slot = 0; slot < qp->sq_depth; slot++) {
            struct hisi_zip_sqe *sqe = qp->sqe + slot * qm->sqe_size;
            tag = (u64)sqe->dw26 | ((u64)sqe->dw27 << 32);

            if (tag != 0) {
                dev_err(&qm->pdev->dev,
                    "[SELFTEST FAIL] QP%u SQE[%d] tag=0x%llx (should be 0)\n",
                    qp_id, slot, tag);
                selftest_passed = 0;
            }
        }

        if (selftest_passed)
            dev_info(&qm->pdev->dev,
                "[SELFTEST PASS] QP%u all SQE slots cleared\n", qp_id);
    }

    memset(qp->cqe, 0, sizeof(struct qm_cqe) * qp->cq_depth);
    /* ... */
}
```

### 2.2 回调安全自测

在回调函数中添加 tag 验证：

```c
/* 文件: drivers/crypto/hisilicon/zip/zip_crypto.c */
static void hisi_zip_acomp_cb(struct hisi_qp *qp, void *data)
{
    struct hisi_zip_sqe *sqe = data;
    struct hisi_zip_req *req;
    u64 tag;

    /* 自测试：验证 tag 在合理范围内 */
    tag = (u64)sqe->dw26 | ((u64)sqe->dw27 << 32);

    if (tag == 0) {
        dev_err(&qp->qm->pdev->dev,
            "[SELFTEST FAIL] callback called with tag=0\n");
        return;  /* 安全退出，不崩溃 */
    }

    /* 验证 tag 指针在内核地址范围内 */
    if (tag < 0xffff000000000000ULL || tag > 0xffffffffffffffffULL) {
        dev_err(&qp->qm->pdev->dev,
            "[SELFTEST FAIL] tag=0x%llx outside kernel range\n", tag);
        return;
    }

    req = (struct hisi_zip_req *)tag;

    /* 验证 req 结构完整性 */
    if (!req->qp_ctx || req->qp_ctx->qp != qp) {
        dev_err(&qp->qm->pdev->dev,
            "[SELFTEST FAIL] req->qp_ctx mismatch\n");
        return;
    }

    /* 正常处理 */
    struct hisi_zip_qp_ctx *qp_ctx = req->qp_ctx;
    /* ... */
}
```

### 2.3 模块加载自测

添加模块参数触发自测：

```c
/* 文件: drivers/crypto/hisilicon/zip/zip_main.c */

static int selftest_on_load = 0;
module_param(selftest_on_load, int, 0444);
MODULE_PARM_DESC(selftest_on_load, "Run selftest on module load (0=off, 1=on)");

static int hisi_zip_selftest_run(struct hisi_qm *qm)
{
    struct crypto_acomp *acomp;
    struct acomp_req *req;
    char src_buf[64] = "test data for compression";
    char dst_buf[256];
    struct scatterlist src, dst;
    int ret;

    dev_info(&qm->pdev->dev, "[SELFTEST] Starting hisi_zip selftest...\n");

    /* 测试压缩功能 */
    acomp = crypto_alloc_acomp("deflate", 0, 0);
    if (IS_ERR(acomp)) {
        dev_err(&qm->pdev->dev, "[SELFTEST FAIL] alloc acomp failed\n");
        return PTR_ERR(acomp);
    }

    sg_init_one(&src, src_buf, strlen(src_buf));
    sg_init_one(&dst, dst_buf, sizeof(dst_buf));

    req = acomp_request_alloc(acomp);
    if (!req) {
        dev_err(&qm->pdev->dev, "[SELFTEST FAIL] alloc req failed\n");
        crypto_free_acomp(acomp);
        return -ENOMEM;
    }

    acomp_request_set_params(req, &src, &dst, strlen(src_buf), sizeof(dst_buf));

    ret = crypto_acomp_compress(req);
    if (ret) {
        dev_err(&qm->pdev->dev, "[SELFTEST FAIL] compress failed: %d\n", ret);
    } else {
        dev_info(&qm->pdev->dev,
            "[SELFTEST PASS] compress succeeded, outlen=%u\n", req->dlen);
    }

    acomp_request_free(req);
    crypto_free_acomp(acomp);

    return ret;
}
```

## 三、用户态测试脚本

### 3.1 rmmod/insmod 循环测试脚本

```bash
#!/bin/bash
# 文件: test_rmmod_insmod_loop.sh
# 目的: 验证驱动重复加载稳定性

LOOPS=100
PASS=0
FAIL=0

echo "=== hisi_zip rmmod/insmod stability test ==="
echo "Testing $LOOPS cycles..."

for i in $(seq 1 $LOOPS); do
    echo "Cycle $i/$LOOPS..."

    # 卸载驱动
    rmmod hisi_zip 2>/dev/null
    sleep 0.5

    # 加载驱动
    insmod hisi_zip.ko uacce_mode=0
    if [ $? -ne 0 ]; then
        echo "[FAIL] insmod failed at cycle $i"
        FAIL=$((FAIL + 1))
        break
    fi

    # 简单压缩测试
    echo "test data" | gzip > /dev/null
    if [ $? -ne 0 ]; then
        echo "[FAIL] compression test failed at cycle $i"
        FAIL=$((FAIL + 1))
        break
    fi

    PASS=$((PASS + 1))
    echo "[PASS] cycle $i completed"
done

echo "=== Test Summary ==="
echo "PASS: $PASS"
echo "FAIL: $FAIL"

if [ $FAIL -eq 0 ]; then
    echo "All tests passed!"
    exit 0
else
    echo "Test failed!"
    exit 1
fi
```

### 3.2 压缩功能完整性测试

```bash
#!/bin/bash
# 文件: test_compression_integrity.sh
# 目的: 验证压缩解压数据完整性

TEST_FILE="/tmp/test_data.bin"
COMPRESSED="/tmp/test_data.gz"
DECOMPRESSED="/tmp/test_data_restored.bin"

echo "=== hisi_zip compression integrity test ==="

# 生成随机测试数据
dd if=/dev/urandom of=$TEST_FILE bs=1K count=100 2>/dev/null
echo "Generated 100KB random test data"

# 压缩
gzip -k $TEST_FILE
echo "Compressed test data"

# 解压
gunzip -c $COMPRESSED > $DECOMPRESSED
echo "Decompressed test data"

# 比较数据完整性
if diff $TEST_FILE $DECOMPRESSED > /dev/null; then
    echo "[PASS] Data integrity verified - original matches decompressed"
    rm -f $TEST_FILE $COMPRESSED $DECOMPRESSED
    exit 0
else
    echo "[FAIL] Data integrity failed - decompressed data differs from original"
    rm -f $TEST_FILE $COMPRESSED $DECOMPRESSED
    exit 1
fi
```

### 3.3 并发压力测试

```bash
#!/bin/bash
# 文件: test_concurrent_stress.sh
# 目起: 验证并发场景下的稳定性

THREADS=10
ITERATIONS=100

echo "=== hisi_zip concurrent stress test ==="
echo "Threads: $THREADS, Iterations per thread: $ITERATIONS"

run_test() {
    local id=$1
    local test_file="/tmp/stress_test_$id.bin"

    for i in $(seq 1 $ITERATIONS); do
        dd if=/dev/urandom of=$test_file bs=1K count=10 2>/dev/null
        gzip $test_file
        gunzip ${test_file}.gz
        rm -f ${test_file} ${test_file}.gz
    done
}

for t in $(seq 1 $THREADS); do
    run_test $t &
done

wait

echo "[PASS] Concurrent stress test completed"
```

## 四、内核 KUnit 测试（可选）

如果项目支持 KUnit，可以编写单元测试：

```c
/* 文件: drivers/crypto/hisilicon/zip/zip_test.c */

#include <kunit/test.h>
#include <crypto/acompress.h>

static void test_sqe_tag_cleared(struct kunit *test)
{
    struct hisi_qm *qm;
    struct hisi_qp *qp;
    int slot;

    /* 创建 QP */
    qp = hisi_qm_create_qp(qm, 0);
    KUNIT_ASSERT_NOT_ERR_OR_NULL(test, qp);

    /* 验证所有 SQE slot 的 tag 为 0 */
    for (slot = 0; slot < qp->sq_depth; slot++) {
        struct hisi_zip_sqe *sqe = qp->sqe + slot * qm->sqe_size;
        u64 tag = (u64)sqe->dw26 | ((u64)sqe->dw27 << 32);

        KUNIT_EXPECT_EQ(test, tag, 0ULL);
    }

    hisi_qm_release_qp(qp);
}

static void test_acomp_compress_decompress(struct kunit *test)
{
    struct crypto_acomp *acomp;
    struct acomp_req *req;
    char src[] = "hello world test data";
    char dst[256];
    struct scatterlist src_sg, dst_sg;
    int ret;

    acomp = crypto_alloc_acomp("deflate", 0, 0);
    KUNIT_ASSERT_NOT_ERR_OR_NULL(test, acomp);

    sg_init_one(&src_sg, src, strlen(src));
    sg_init_one(&dst_sg, dst, sizeof(dst));

    req = acomp_request_alloc(acomp);
    KUNIT_ASSERT_NOT_ERR_OR_NULL(test, req);

    acomp_request_set_params(req, &src_sg, &dst_sg, strlen(src), sizeof(dst));

    ret = crypto_acomp_compress(req);
    KUNIT_EXPECT_EQ(test, ret, 0);

    acomp_request_free(req);
    crypto_free_acomp(acomp);
}

static struct kunit_case zip_test_cases[] = {
    KUNIT_CASE(test_sqe_tag_cleared),
    KUNIT_CASE(test_acomp_compress_decompress),
    {}
};

static struct kunit_suite zip_test_suite = {
    .name = "hisi_zip",
    .test_cases = zip_test_cases,
};

kunit_test_suite(zip_test_suite);
```

## 五、测试执行流程

```
完整测试流程:

1. 应用修复补丁 (04_fix_clear_sqe.patch)
   ↓
2. 编译内核模块
   make drivers/crypto/hisilicon/zip/
   ↓
3. 加载带自测的驱动
   insmod hisi_zip.ko uacce_mode=0 selftest_on_load=1
   ↓
4. 观察内核日志
   dmesg | grep SELFTEST
   预期: [SELFTEST PASS] QP* all SQE slots cleared
   ↓
5. 执行用户态测试脚本
   ./test_rmmod_insmod_loop.sh
   ./test_compression_integrity.sh
   ./test_concurrent_stress.sh
   ↓
6. 检查测试结果
   所有测试 PASS 则修复有效
```

## 六、预期测试结果

| 测试项 | 预期结果 | 验证意义 |
|-------|---------|---------|
| SQE 初始化自测 | PASS: all SQE slots cleared | 修复有效 |
| 回调安全自测 | 无 FAIL 输出 | 无残留 tag 导致回调 |
| rmmod/insmod 100次 | PASS: 100 | 稳定性验证 |
| 压缩完整性 | PASS: data matches | 功能正常 |
| 并发压力 | PASS: all threads done | 并发安全 |

## 七、测试失败处理

如果测试 FAIL，检查：

1. **SQE 初始化测试 FAIL**
   - 确认修复补丁是否正确应用
   - 检查 memset 是否执行

2. **回调安全测试 FAIL**
   - 分析 tag 值来源
   - 检查硬件状态残留

3. **功能测试 FAIL**
   - 检查硬件是否正常工作
   - 验证数据 DMA 映射是否正确