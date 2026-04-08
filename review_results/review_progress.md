# UADK代码检视进度跟踪

## 检视规则来源
- 规则目录: E:/work_space/加速器用户态学习/代码检视/review_rules
- 使用Skills: cpp-memory-safety, cpp-general-security, cpp-api-misuse等

## 待检视文件列表 (排除test和uadk_tool)

### 核心源文件 (根目录)
1. wd.c
2. wd_aead.c
3. wd_agg.c
4. wd_alg.c
5. wd_bmm.c
6. wd_cipher.c
7. wd_comp.c
8. wd_dh.c
9. wd_digest.c
10. wd_ecc.c
11. wd_join_gather.c
12. wd_mempool.c
13. wd_rsa.c
14. wd_sched.c
15. wd_udma.c
16. wd_util.c
17. wd_zlibwrapper.c

### 驱动文件 (drv目录)
18. drv/hash_mb/hash_mb.c
19. drv/hash_mb/hash_mb.h
20. drv/hisi_comp.c
21. drv/hisi_comp_huf.c
22. drv/hisi_comp_huf.h
23. drv/hisi_dae.c
24. drv/hisi_dae.h
25. drv/hisi_dae_common.c
26. drv/hisi_dae_join_gather.c
27. drv/hisi_hpre.c
28. drv/hisi_qm_udrv.c
29. drv/hisi_qm_udrv.h
30. drv/hisi_sec.c
31. drv/hisi_udma.c
32. drv/isa_ce_sm3.c
33. drv/isa_ce_sm3.h
34. drv/isa_ce_sm4.c
35. drv/isa_ce_sm4.h

### 头文件 (include目录)
36. include/uacce.h
37. include/wd.h
38. include/wd_aead.h
39. include/wd_agg.h
40. include/wd_alg.h
41. include/wd_alg_common.h
42. include/wd_bmm.h
43. include/wd_cipher.h
44. include/wd_comp.h
45. include/wd_dae.h
46. include/wd_dh.h
47. include/wd_digest.h
48. include/wd_ecc.h
49. include/wd_ecc_curve.h
50. include/wd_internal.h
51. include/wd_join_gather.h
52. include/wd_rsa.h
53. include/wd_sched.h
54. include/wd_udma.h
55. include/wd_util.h
56. include/wd_zlibwrapper.h
57-67. include/drv/*.h (驱动头文件)
68-70. include/crypto/*.h (加密头文件)

### lib目录
71-73. lib/crypto/*.c

### v1目录 (旧版本)
74-93. v1目录下的文件 (不含test)

### sample目录
94. sample/uadk_comp.c

## 已检视文件
1. **wd.c** (2026-04-07 第1轮) - 检视报告: review_round_001_wd_c.md
   - 发现问题: HIGH 1个, MEDIUM 3个, LOW 2个

2. **wd_util.c** (2026-04-07 第2轮) - 检视报告: review_round_002_wd_util_c.md
   - 发现问题: HIGH 3个, MEDIUM 3个, LOW 1个

3. **wd_alg.c** (2026-04-07 第3轮) - 检视报告: review_round_003_wd_alg_c.md
   - 发现问题: HIGH 1个, MEDIUM 2个, LOW 2个

4. **wd_sched.c** (2026-04-07 第4轮) - 检视报告: review_round_004_wd_sched_c.md
   - 发现问题: HIGH 2个, MEDIUM 2个, LOW 2个

5. **wd_mempool.c** (2026-04-07 第5轮) - 检视报告: review_round_005_wd_mempool_c.md
   - 发现问题: HIGH 2个, MEDIUM 2个, LOW 1个

6. **wd_cipher.c** (2026-04-07 第6轮) - 检视报告: review_round_006_wd_cipher_c.md
   - 发现问题: HIGH 2个, MEDIUM 3个, LOW 1个

7. **wd_digest.c** (2026-04-07 第7轮) - 检视报告: review_round_007_wd_digest_c.md
   - 发现问题: HIGH 2个, MEDIUM 3个, LOW 1个

8. **wd_aead.c** (2026-04-07 第8轮) - 检视报告: review_round_008_wd_aead_c.md
   - 发现问题: HIGH 3个, MEDIUM 2个, LOW 1个

9. **wd_comp.c** (2026-04-07 第9轮) - 检视报告: review_round_009_wd_comp_c.md
   - 发现问题: HIGH 1个, MEDIUM 4个, LOW 2个

10. **drv/hisi_sec.c** (2026-04-07 第10轮) - 检视报告: review_round_010_hisi_sec_c.md
    - 发现问题: HIGH 3个, MEDIUM 5个

11. **drv/hisi_hpre.c** (2026-04-07 第11轮) - 检视报告: review_round_011_hisi_hpre_c.md
    - 发现问题: HIGH 3个, MEDIUM 4个, LOW 1个

12. **drv/hisi_comp.c** (2026-04-07 第12轮) - 检视报告: review_round_012_hisi_comp_c.md
    - 发现问题: HIGH 2个, MEDIUM 4个, LOW 1个

13. **drv/hisi_qm_udrv.c** (2026-04-07 第13轮) - 检视报告: review_round_013_hisi_qm_udrv_c.md
    - 发现问题: HIGH 2个, MEDIUM 3个, LOW 2个

14. **drv/isa_ce_sm3.c** (2026-04-07 第14轮) - 检视报告: review_round_014_isa_ce_sm3_c.md
    - 发现问题: MEDIUM 3个, LOW 3个

15. **drv/isa_ce_sm4.c** (2026-04-07 第15轮) - 检视报告: review_round_015_isa_ce_sm4_c.md
    - 发现问题: MEDIUM 4个, LOW 2个

16. **drv/hisi_udma.c** (2026-04-07 第16轮) - 检视报告: review_round_016_hisi_udma_c.md
    - 发现问题: HIGH 2个, MEDIUM 4个, LOW 3个

17. **drv/hisi_dae.c** (2026-04-07 第17轮) - 检视报告: review_round_017_hisi_dae_c.md
    - 发现问题: HIGH 2个, MEDIUM 6个, LOW 1个

18. **drv/hisi_dae_common.c** (2026-04-07 第18轮) - 检视报告: review_round_018_hisi_dae_common_c.md
    - 发现问题: MEDIUM 5个, LOW 2个

19. **drv/hisi_dae_join_gather.c** (2026-04-07 第19轮) - 检视报告: review_round_019_hisi_dae_join_gather_c.md
    - 发现问题: HIGH 2个, MEDIUM 5个, LOW 1个

20. **drv/hisi_comp_huf.c** (2026-04-07 第20轮) - 检视报告: review_round_020_hisi_comp_huf_c.md
    - 发现问题: HIGH 1个, MEDIUM 1个, LOW 2个

21. **drv/hash_mb/hash_mb.c** (2026-04-07 第21轮) - 检视报告: review_round_021_hash_mb_c.md
    - 发现问题: HIGH 2个, MEDIUM 5个, LOW 1个

22. **include/wd.h** (2026-04-07 第22轮) - 检视报告: review_round_022_wd_h.md
    - 发现问题: MEDIUM 2个, LOW 2个

23. **include/wd_util.h** (2026-04-07 第23轮) - 检视报告: review_round_023_wd_util_h.md
    - 发现问题: MEDIUM 2个, LOW 2个

24. **include/wd_cipher.h** (2026-04-07 第24轮) - 检视报告: review_round_024_wd_cipher_h.md
    - 发现问题: LOW 1个

25. **include/wd_digest.h** (2026-04-07 第25轮) - 检视报告: review_round_025_wd_digest_h.md
    - 发现问题: HIGH 1个, LOW 1个

29. **include/wd_dh.h, wd_ecc.h** (2026-04-07 第29轮) - 检视报告: review_round_029_wd_dh_ecc_h.md
    - 发现问题: LOW 4个

30. **include/uacce.h** (2026-04-07 第30轮) - 检视报告: review_round_030_uacce_h.md
    - 发现问题: 无问题，设计规范

31. **v1/wd.c** (2026-04-07 第31轮) - 检视报告: review_round_031_v1_wd_c.md
    - 发现问题: MEDIUM 2个, LOW 3个, INFO 1个, POSITIVE 1个

32. **sample/uadk_comp.c** (2026-04-07 第32轮) - 检视报告: review_round_032_sample_uadk_comp_c.md
    - 发现问题: HIGH 1个, MEDIUM 4个, LOW 3个, INFO 1个, POSITIVE 3个

33. **lib/crypto/aes.c, galois.c, sm4.c** (2026-04-07 第33轮) - 检视报告: review_round_033_lib_crypto.md
    - 发现问题: MEDIUM 5个, LOW 1个, INFO 2个, POSITIVE 4个

34. **include/crypto/*.h** (2026-04-07 第34轮) - 检视报告: review_round_034_include_crypto_h.md
    - 发现问题: LOW 3个, INFO 1个, POSITIVE 3个

35. **include/drv/*.h** (2026-04-07 第35轮) - 检视报告: review_round_035_include_drv_h.md
    - 发现问题: LOW 2个, INFO 7个, POSITIVE 3个 (11个头文件，共974行)

36. **v1/wd_util.c, wd_adapter.c** (2026-04-07 第36轮) - 检视报告: review_round_036_v1_util_adapter.md
    - 发现问题: MEDIUM 4个, LOW 2个, INFO 1个, POSITIVE 3个

37. **v1/wd_cipher.c** (2026-04-08 第37轮) - 检视报告: review_round_037_v1_wd_cipher_c.md
    - 发现问题: HIGH 1个, MEDIUM 4个, LOW 3个, INFO 1个, POSITIVE 2个

38. **v1/wd_digest.c** (2026-04-08 第38轮) - 检视报告: review_round_038_v1_wd_digest_c.md
    - 发现问题: HIGH 2个, MEDIUM 4个, LOW 2个, INFO 1个, POSITIVE 2个

39. **v1/wd_aead.c** (2026-04-08 第39轮) - 检视报告: review_round_039_v1_wd_aead_c.md
    - 发现问题: HIGH 2个, MEDIUM 5个, LOW 2个, INFO 1个, POSITIVE 2个

40. **v1/wd_comp.c** (2026-04-08 第40轮) - 检视报告: review_round_040_v1_wd_comp_c.md
    - 发现问题: HIGH 1个, MEDIUM 4个, LOW 2个, INFO 1个, POSITIVE 2个

41. **v1/wd_rsa.c** (2026-04-08 第41轮) - 检视报告: review_round_041_v1_wd_rsa_c.md
    - 发现问题: HIGH 2个, MEDIUM 5个, LOW 3个, INFO 1个, POSITIVE 2个

42. **v1/wd_dh.c** (2026-04-08 第42轮) - 检视报告: review_round_042_v1_wd_dh_c.md
    - 发现问题: HIGH 2个, MEDIUM 3个, LOW 2个, POSITIVE 1个

43. **v1/wd_ecc.c** (2026-04-08 第43轮) - 检视报告: review_round_043_v1_wd_ecc_c.md
    - 发现问题: HIGH 2个, MEDIUM 5个, LOW 3个, INFO 1个, POSITIVE 2个

44. **wd_rsa.c** (2026-04-08 第44轮) - 检视报告: review_round_044_wd_rsa_c.md
    - 发现问题: HIGH 2个, MEDIUM 4个, LOW 2个, INFO 1个, POSITIVE 3个

45. **wd_dh.c** (2026-04-08 第45轮) - 检视报告: review_round_045_wd_dh_c.md
    - 发现问题: HIGH 2个, MEDIUM 3个, LOW 2个, POSITIVE 2个

46. **wd_ecc.c** (2026-04-08 第46轮) - 检视报告: review_round_046_wd_ecc_c.md
    - 发现问题: HIGH 3个, MEDIUM 5个, LOW 3个, INFO 1个, POSITIVE 3个

47. **wd_bmm.c** (2026-04-08 第47轮) - 检视报告: review_round_047_wd_bmm_c.md
    - 发现问题: HIGH 1个, MEDIUM 4个, LOW 2个, INFO 2个, POSITIVE 4个

48. **wd_agg.c** (2026-04-08 第48轮) - 检视报告: review_round_048_wd_agg_c.md
    - 发现问题: HIGH 2个, MEDIUM 3个, LOW 2个, INFO 2个, POSITIVE 5个

49. **wd_udma.c** (2026-04-08 第49轮) - 检视报告: review_round_049_wd_udma_c.md
    - 发现问题: HIGH 2个, MEDIUM 2个, LOW 1个, INFO 1个, POSITIVE 4个

50. **wd_join_gather.c** (2026-04-08 第50轮) - 检视报告: review_round_050_wd_join_gather_c.md
    - 发现问题: HIGH 2个, MEDIUM 3个, LOW 2个, INFO 2个, POSITIVE 5个

51. **wd_zlibwrapper.c** (2026-04-08 第51轮) - 检视报告: review_round_051_wd_zlibwrapper_c.md
    - 发现问题: MEDIUM 2个, LOW 2个, INFO 2个, POSITIVE 4个

52. **v1/wd_bmm.c** (2026-04-08 第52轮) - 检视报告: review_round_052_v1_wd_bmm_c.md
    - 发现问题: MEDIUM 3个, LOW 2个, INFO 2个, POSITIVE 4个

53. **v1/wd_sgl.c** (2026-04-08 第53轮) - 检视报告: review_round_053_v1_wd_sgl_c.md
    - 发现问题: HIGH 1个, MEDIUM 4个, LOW 3个, INFO 2个, POSITIVE 5个

54. **v1/drv/hisi_qm_udrv.c** (2026-04-08 第54轮) - 检视报告: review_round_054_v1_drv_hisi_qm_udrv_c.md
    - 发现问题: HIGH 3个, MEDIUM 5个, LOW 3个, INFO 2个, POSITIVE 3个

55. **v1/drv/hisi_hpre_udrv.c** (2026-04-08 第55轮) - 检视报告: review_round_055_v1_drv_hisi_hpre_udrv_c.md
    - 发现问题: HIGH 2个, MEDIUM 6个, LOW 4个, INFO 3个, POSITIVE 4个

56. **v1/drv/hisi_sec_udrv.c** (2026-04-08 第56轮) - 检视报告: review_round_056_v1_drv_hisi_sec_udrv_c.md
    - 发现问题: HIGH 2个, MEDIUM 6个, LOW 4个, INFO 3个, POSITIVE 5个

57. **v1/drv/hisi_zip_udrv.c** (2026-04-08 第57轮) - 检视报告: review_round_057_v1_drv_hisi_zip_udrv_c.md
    - 发现问题: HIGH 2个, MEDIUM 5个, LOW 3个, INFO 2个, POSITIVE 4个

58. **v1/drv/hisi_zip_huf.c** (2026-04-08 第58轮) - 检视报告: review_round_058_v1_drv_hisi_zip_huf_c.md
    - 发现问题: HIGH 1个, MEDIUM 2个, LOW 2个, INFO 1个, POSITIVE 3个

59. **v1/drv/*.h + v1/internal/wd_ecc_curve.h** (2026-04-08 第59轮) - 检视报告: review_round_059_v1_drv_headers.md
    - 检视文件: hisi_qm_udrv.h, hisi_hpre_udrv.h, hisi_sec_udrv.h, hisi_zip_udrv.h, hisi_zip_huf.h, wd_drv.h, wd_ecc_curve.h
    - 发现问题: MEDIUM 3个, LOW 4个, INFO 5个, POSITIVE 4个

60. **v1/wd.h, wd_util.h, wd_cipher.h, wd_digest.h, wd_aead.h, wd_rsa.h, wd_dh.h, wd_ecc.h** (2026-04-08 第60轮) - 检视报告: review_round_060_v1_headers.md
    - 发现问题: HIGH 1个, MEDIUM 4个, LOW 5个, INFO 4个, POSITIVE 5个

61. **v1/wd_comp.h, wd_bmm.h, wd_sgl.h, wd_adapter.h** (2026-04-08 第61轮) - 检视报告: review_round_061_v1_comp_bmm_sgl_adapter.md
    - 发现问题: MEDIUM 3个, LOW 4个, INFO 4个, POSITIVE 3个

62. **include剩余头文件** (2026-04-08 第62轮) - 检视报告: review_round_062_include_remaining_headers.md
    - 检视文件: wd_alg.h, wd_alg_common.h, wd_bmm.h, wd_comp.h, wd_dae.h, wd_join_gather.h, wd_rsa.h, wd_sched.h, wd_udma.h, wd_internal.h, wd_agg.h, wd_zlibwrapper.h, wd_ecc_curve.h (共13个文件, 约2085行)
    - 发现问题: MEDIUM 6个, LOW 9个, INFO 5个, POSITIVE 2个

63. **v1/uacce.h** (2026-04-08 第63轮) - 检视报告: review_round_063_v1_uacce_h.md
    - 发现问题: INFO 1个, POSITIVE 1个

64. **drv目录头文件 + include/drv目录驱动接口头文件** (2026-04-08 第64轮) - 检视报告: review_round_064_drv_headers.md
    - 检视文件: hash_mb.h, hisi_dae.h, hisi_comp_huf.h, hisi_qm_udrv.h, isa_ce_sm3.h, isa_ce_sm4.h, wd_cipher_drv.h, wd_digest_drv.h, wd_aead_drv.h, wd_comp_drv.h, wd_rsa_drv.h, wd_dh_drv.h, wd_ecc_drv.h, wd_agg_drv.h, wd_udma_drv.h, wd_join_gather_drv.h (共16个文件, 约1356行)
    - 发现问题: MEDIUM 12个, LOW 12个, INFO 12个

65. **include/crypto目录头文件 + include/drv/arm_arch_ce.h** (2026-04-08 第65轮) - 检视报告: review_round_065_crypto_headers.md
    - 检视文件: aes.h, galois.h, sm4.h, arm_arch_ce.h (共4个文件, 约272行)
    - 发现问题: MEDIUM 3个, LOW 3个, INFO 2个, POSITIVE 1个

## 检视结果保存位置
- E:/work_space/加速器用户态学习/uadk/review_results/
- 检视顺序: review_round_001 → 002 → ... → 030

## 累计问题统计
| 等级 | 数量 |
|------|------|
| HIGH | 76 |
| MEDIUM | 208 |
| LOW | 149 |
| INFO | 78 |
| POSITIVE | 105 |
| **总计** | **616** |

## 🎉 检视完成总结 (2026-04-08 更新)

### 检视范围
本次检视覆盖了UADK项目的所有核心代码文件，排除了test目录和uadk_tool目录。

### 检视轮次统计
- **总轮次**: 65轮
- **检视文件数**: 约80个核心文件
- **总代码行数**: 约18000+行

### 各目录检视统计
| 目录 | 文件数 | 问题数 |
|------|--------|--------|
| 根目录源文件 (wd_*.c) | 17 | 131 |
| drv目录源文件 | 12 | 87 |
| drv目录头文件 | 6 | 24 |
| include目录头文件 | 21 | 32 |
| include/drv目录头文件 | 11 | 24 |
| include/crypto目录头文件 | 4 | 7 |
| lib/crypto目录源文件 | 3 | 6 |
| sample目录 | 1 | 12 |
| v1目录核心文件 | 22 | 92 |
| v1目录头文件 | 9 | 21 |
| **总计** | **~106** | **436+** |

### 主要问题类型分布
| 问题类型 | 数量 | 占比 |
|----------|------|------|
| 空指针解引用/未检查NULL | 50+ | 11%+ |
| 数组越界 | 8 | 2% |
| 回调函数未检查NULL | 40+ | 9%+ |
| 资源管理问题 | 20+ | 5%+ |
| 整数运算问题 | 15+ | 3%+ |
| 枚举缺少MAX值 | 12 | 3% |
| 文档缺失 | 30+ | 7% |
| 其他 | 余量 | - |

### 高优先级问题清单 (HIGH级别，共76个)
1. **数组越界问题**:
   - wd_digest.c: g_digest_mac_full_len数组越界
   - wd_aead.c: g_aead_mac_len数组越界
   - hisi_sec.c: g_digest_a_alg/g_sec_hmac_full_len数组越界

2. **空指针解引用**:
   - wd_util.c: strtol返回值检查问题
   - hisi_dae.c: check_bfinal_complete_block addr未检查NULL
   - 各驱动send/recv函数qp未检查NULL

3. **回调函数未检查NULL**:
   - 各算法模块poll_ctx回调未检查NULL
   - wd_sched.c sched_init未检查NULL

4. **资源管理问题**:
   - mm_ops回调函数未检查NULL

### MEDIUM级别问题分布 (共208个)
- 指针参数NULL检查文档缺失: 约60个
- 回调函数指针NULL检查缺失: 约40个
- 枚举缺少MAX值: 约12个
- 结构体设计问题: 约30个
- 其他: 约66个

### 检视价值总结
1. **发现系统性问题**: poll_ctx回调未检查NULL问题存在于100%的算法模块
2. **识别安全风险**: 数组越界问题可能导致内存访问异常
3. **改进建议**: 枚举统一添加MAX值、回调函数统一添加NULL检查
4. **POSITIVE发现**: ECC曲线参数设计规范、zlib兼容层设计良好、ARM架构检测完善

### 后续建议
1. 优先修复HIGH级别的76个问题
2. 建议添加统一的回调函数NULL检查宏或辅助函数
3. 建议枚举定义统一添加MAX边界值
4. 建议添加单元测试覆盖边界场景
5. 建议完善函数参数和返回值的文档说明

## 🎉 检视完成总结 (2026-04-08)

### 检视范围
本次检视覆盖了UADK项目的所有核心代码文件，排除了test目录和uadk_tool目录。

### 检视文件统计
| 目录 | 文件数 | 总行数 | 问题数 |
|------|--------|--------|--------|
| 根目录源文件 | 17 | ~13261 | 131 |
| drv目录 | 12 | ~N/A | 87 |
| include目录 | 21 | ~3080 | 32 |
| lib/crypto目录 | 3 | ~N/A | 6 |
| sample目录 | 1 | ~N/A | 12 |
| v1目录 | 22 | ~N/A | 92 |
| **总计** | **76** | **~16000+** | **368** |

### 主要问题类型分布
| 问题类型 | 数量 | 占比 |
|----------|------|------|
| 空指针解引用/未检查NULL | 45+ | 12%+ |
| 数组越界 | 8 | 2% |
| 回调函数未检查NULL | 35+ | 10%+ |
| 资源管理问题 | 20+ | 5%+ |
| 整数运算问题 | 15+ | 4%+ |
| 枚举缺少MAX值 | 12 | 3% |
| 其他 | 余量 | - |

### 高优先级问题清单 (HIGH级别)
1. **wd_digest.c数组越界**: g_digest_mac_full_len数组只有9元素但WD_DIGEST_TYPE_MAX=13
2. **wd_aead.c数组越界**: g_aead_mac_len数组同样问题
3. **hisi_sec.c数组越界**: g_digest_a_alg/g_sec_hmac_full_len数组越界
4. **wd_util.c strtol问题**: 未正确检查strtol返回值和errno
5. **hisi_dae.c check_bfinal_complete_block**: addr参数未检查NULL
6. **各算法模块poll_ctx回调未检查NULL**: wd_cipher, wd_digest, wd_aead, wd_comp, wd_rsa, wd_dh, wd_ecc, wd_agg, wd_udma, wd_join_gather
7. **驱动函数qp指针未检查NULL**: hisi_sec.c, hisi_hpre.c, hisi_comp.c, hisi_udma.c, hisi_dae.c
8. **mm_ops回调函数未检查NULL**: RSA/DH/ECC模块

### 检视价值总结
1. **发现系统性问题模式**: poll_ctx回调未检查NULL问题存在于100%的算法模块
2. **识别安全风险**: 数组越界问题可能导致内存访问异常
3. **改进建议**: 枚举统一添加MAX值、回调函数统一添加NULL检查
4. **POSITIVE发现**: ECC曲线参数设计规范、zlib兼容层设计良好、状态机设计清晰

### 后续建议
1. 优先修复HIGH级别的76个问题
2. 建议添加统一的回调函数NULL检查宏或辅助函数
3. 建议枚举定义统一添加MAX边界值
4. 建议添加单元测试覆盖边界场景

## include目录主要头文件检视总结

已完成include目录下10个主要头文件的检视：

| 文件 | 行数 | 问题数 | HIGH | MEDIUM | LOW | 主要发现 |
|------|------|--------|------|--------|-----|----------|
| wd.h | 642 | 4 | 0 | 2 | 2 | wd_ioread/write内联函数NULL检查 |
| wd_util.h | 563 | 4 | 0 | 2 | 2 | wd_dfx_msg_cnt等内联函数NULL检查 |
| wd_cipher.h | 231 | 1 | 0 | 0 | 1 | 联合体使用说明 |
| wd_digest.h | 291 | 2 | 1 | 0 | 1 | wd_digest_mac_full_len枚举缺失 |
| wd_aead.h | 271 | 3 | 0 | 0 | 3 | 枚举缺少MAX值 |
| wd_comp.h | 268 | 3 | 0 | 0 | 3 | 枚举缺少MAX值 |
| wd_rsa.h | 252 | 3 | 0 | 0 | 3 | 枚举缺少MAX值 |
| wd_dh.h | 108 | 1 | 0 | 0 | 1 | 枚举缺少MAX值 |
| wd_ecc.h | 200+ | 3 | 0 | 0 | 3 | 枚举缺少MAX值 |
| uacce.h | 51 | 0 | 0 | 0 | 0 | 设计规范 |
| **总计** | **2877+** | **24** | **1** | **4** | **19** | |

**头文件检视主要发现**：
1. **wd_digest_mac_full_len枚举缺失(HIGH)** - 与wd_digest.c数组越界问题直接相关
2. **多个枚举缺少MAX值** - 影响边界检查
3. **内联函数缺少NULL检查** - wd_ioread/write, wd_dfx_msg_cnt等

## drv目录检视完成总结

已完成drv目录下12个源文件的检视：

| 文件 | 问题总数 | HIGH | MEDIUM | LOW |
|------|---------|------|--------|-----|
| hisi_sec.c | 8 | 3 | 5 | 0 |
| hisi_hpre.c | 8 | 3 | 4 | 1 |
| hisi_comp.c | 7 | 2 | 4 | 1 |
| hisi_qm_udrv.c | 7 | 2 | 3 | 2 |
| isa_ce_sm3.c | 6 | 0 | 3 | 3 |
| isa_ce_sm4.c | 6 | 0 | 4 | 2 |
| hisi_udma.c | 9 | 2 | 4 | 3 |
| hisi_dae.c | 9 | 2 | 6 | 1 |
| hisi_dae_common.c | 7 | 0 | 5 | 2 |
| hisi_dae_join_gather.c | 8 | 2 | 5 | 1 |
| hisi_comp_huf.c | 4 | 1 | 1 | 2 |
| hash_mb/hash_mb.c | 8 | 2 | 5 | 1 |
| **drv目录小计** | **87** | **19** | **49** | **19** |

**drv目录发现的主要问题模式**：
1. 所有驱动的send/recv函数普遍缺少qp/ctx NULL检查
2. 多处回调函数调用未检查函数指针是否为NULL
3. mm_ops->iova_map/unmap调用前未检查
4. 与hisi_qm_udrv.c中的hisi_set_msg_id/hisi_check_bd_id函数相关联

## 根目录源文件检视完成总结

已完成根目录下16个源文件的检视：

| 文件 | 行数 | 问题总数 | HIGH | MEDIUM | LOW | 主要发现 |
|------|------|---------|------|--------|-----|----------|
| wd.c | 542 | 6 | 1 | 3 | 2 | wd_alg_init_notifier回调检查 |
| wd_util.c | 1077 | 7 | 3 | 3 | 1 | strtol使用问题 |
| wd_alg.c | 384 | 5 | 1 | 2 | 2 | 参数检查 |
| wd_sched.c | 342 | 6 | 2 | 2 | 2 | sched回调检查 |
| wd_mempool.c | 719 | 5 | 2 | 2 | 1 | 内存池管理 |
| wd_cipher.c | 574 | 6 | 2 | 3 | 1 | poll_ctx回调检查 |
| wd_digest.c | 569 | 6 | 2 | 3 | 1 | 数组越界问题 |
| wd_aead.c | 761 | 6 | 3 | 2 | 1 | 数组越界问题 |
| wd_comp.c | 421 | 7 | 1 | 4 | 2 | poll_ctx回调检查 |
| wd_rsa.c | 1337 | 12 | 2 | 4 | 2 | poll_ctx/driver检查 |
| wd_dh.c | 725 | 7 | 2 | 3 | 2 | poll_ctx/driver检查 |
| wd_ecc.c | 2496 | 15 | 3 | 5 | 3 | poll_ctx/driver/回调检查 |
| wd_bmm.c | 1180 | 9 | 1 | 4 | 2 | ctx->dev检查 |
| wd_agg.c | 1587 | 12 | 2 | 3 | 2 | 状态机设计良好 |
| wd_udma.c | 513 | 8 | 2 | 2 | 1 | 参数验证完整 |
| wd_join_gather.c | 1825 | 12 | 2 | 3 | 2 | 三种操作模式 |
| wd_zlibwrapper.c | 309 | 6 | 0 | 2 | 2 | zlib兼容层 |
| **根目录小计** | **13261** | **131** | **30** | **52** | **28** | |

**根目录文件发现的系统性问题模式**：
1. **poll_ctx回调未检查NULL** - 100%存在于算法模块（wd_cipher, wd_digest, wd_aead, wd_comp, wd_rsa, wd_dh, wd_ecc, wd_agg, wd_udma, wd_join_gather）
2. **driver->send/recv未检查NULL** - 100%存在于算法模块
3. **sched_init未检查NULL** - 100%存在于会话分配函数
4. **mm_ops函数指针未检查NULL** - RSA/DH/ECC模块

## 发现的主要问题类型（更新）
1. **空指针解引用/未检查NULL** - 最常见问题
   - 多个函数缺少参数NULL检查
   - 回调函数未检查就调用
   - 驱动函数qp指针未检查NULL (hisi_sec.c, hisi_hpre.c, hisi_comp.c, hisi_udma.c, hisi_dae.c等)
   - check_bfinal_complete_block函数addr未检查NULL

2. **数组越界** - 严重问题
   - g_digest_mac_full_len数组只有9元素但WD_DIGEST_TYPE_MAX=13
   - g_aead_mac_len数组同样问题
   - g_digest_a_alg/g_sec_hmac_full_len数组在drv/hisi_sec.c中同样存在越界

3. **资源管理问题**
   - 内存泄漏、缓冲区溢出
   - free/uninit函数检查不完整
   - mm_ops回调函数未检查NULL

4. **整数运算问题**
   - read_bits函数移位操作可能有未定义行为
   - 整数溢出风险

5. **通用问题模式**
   - memcpy前未检查目标指针
   - poll函数未检查回调函数
   - 整数溢出风险
   - iova_map/unmap函数调用前未检查

## 发现的主要问题类型（更新）
1. **空指针解引用/未检查NULL** - 最常见问题
   - 多个函数缺少参数NULL检查
   - 回调函数未检查就调用
   - 驱动函数qp指针未检查NULL (hisi_sec.c, hisi_hpre.c, hisi_comp.c)

2. **数组越界** - 严重问题
   - g_digest_mac_full_len数组只有9元素但WD_DIGEST_TYPE_MAX=13
   - g_aead_mac_len数组同样问题
   - g_digest_a_alg/g_sec_hmac_full_len数组在drv/hisi_sec.c中同样存在越界

3. **资源管理问题**
   - 内存泄漏、缓冲区溢出
   - free/uninit函数检查不完整
   - mm_ops回调函数未检查NULL

4. **通用问题模式**
   - memcpy前未检查目标指针
   - poll函数未检查回调函数
   - 整数溢出风险
   - iova_map/unmap函数调用前未检查