# 研究文档：华为鲲鹏（Kunpeng）平台 SME/SVE/NEON 可用性 + P3 口径决议 + P3-2 草稿

- **日期**: 2026-06-09
- **任务**: 三部分重构计划的 Task 1 —— 解决 R1（平台 SME 可用性未查实）+ 为 P3-2（"SME/SVE/NEON 是什么、为什么对 agent 特别"）提供研究底稿与口径决议。
- **驱动 spec**: `docs/superpowers/specs/2026-06-09-cpu-agent-3part-restructure-design.md`（§1.2 统一主线、§5 P3 页表、§11 R1）。
- **引用铁律**: 每个 ISA 特性声明与每个性能数字均带内联 `[来源: …]`；性能对比标注 确认✅ / 数量级⚠️。

---

## 0. 结论先行（TL;DR）

| 问题 | 结论 |
|------|------|
| 团队现役主力平台是否有 SME/SVE？ | **鲲鹏 920：否**。**鲲鹏 920（TaiShan v110, ARMv8.2-A）只有 NEON（固定 128-bit SIMD）**，无 SVE / SVE2 / SME。**但新一代鲲鹏（团队 early-access / 内部可得）= full SVE + SVE2 + SME（⚠️ 团队已可得；具体型号与公开可引用来源暂缺）。** |
| 新一代鲲鹏是否带 SVE2/SME？ | **是（团队 early-access 已可得）**。新一代鲲鹏平台**团队已可得（⚠️ 暂无公开来源；不指定 SKU 名）**，实现 full SVE + SVE2 + SME。**不**在鲲鹏 920 上（920 = NEON-only）。 |
| **P3 口径决议** | **NEON（鲲鹏 920 现役）+ SVE/SVE2/SME（新一代鲲鹏 early-access 现已可得）同一融合方向——武器已齐备，团队可从今天起在自有 SVE/SME 硅上实测验证，无需再等未来平台**（详见 §2）。 |
| 公开路线图（仅供参考） | 公开路线图显示 **鲲鹏 950（计划 2026 Q4）/ 鲲鹏 960（计划 2028 Q1）**（⚠️ 数量级，厂商计划）；但团队**已不依赖**该路线图——early-access SVE/SME 硅现已可得。 |

---

## 1. 平台特性表（Kunpeng 920 / 930 → NEON / SVE 宽度 / SVE2 / SME）

| 平台 / 微架构 | ISA 基线 | NEON（128-bit SIMD） | SVE 向量宽度 | SVE2 (FEAT_SVE2) | FEAT_SME / FEAT_SME2 |
|---|---|---|---|---|---|
| **鲲鹏 920 / TaiShan v110** | ARMv8.2-A `[来源: HiSilicon 官方产品页, https://www.hisilicon.com/en/products/kunpeng/huawei-kunpeng/huawei-kunpeng-920 ; IEEE Micro 2021 "Kunpeng 920: The First 7-nm Chiplet-Based 64-Core ARM SoC"]` | **有**（NEON-basic，128-bit 固定宽度）`[来源: IEEE Micro 2021 原文 "TaiShan V110's SIMD architecture is compatible with NEON-basic instructions and also supports part of the extension instructions of ARM-v8.2"；VLDB Journal 2022 "the Kunpeng only supports ARM's SIMD extension called NEON with 128-bit", https://link.springer.com/article/10.1007/s00778-022-00744-2 ；Stony Brook Ookami 文档列出 Kunpeng 920 = 128-bit NEON, https://www.stonybrook.edu/commcms/ookami/support/_docs/Arm_SVE_reduced.pdf]` | **无**（不实现 SVE）`[来源: 同上 Stony Brook / VLDB —— 二者均将 Kunpeng 920 列为"NEON-only / 非 SVE"对照组；社区亦确认 SVE 难以移植到此代，Reddit r/hardware "ARM CPU cores stuck at 128-bit SIMD"]` | **无**（SVE2 是 ARMv9-A 特性，920 为 ARMv8.2-A）`[来源: ARM Developer "Introducing SVE2", SVE2 引入于 ARMv9-A, https://developer.arm.com/documentation/102340/0100/Introducing-SVE2]` | **无**（SME 为 ARMv9-A 可选扩展，920 为 ARMv8.2-A）`[来源: ARM Developer "Introducing the Scalable Matrix Extension for the Armv9-A Architecture", SME 是 Armv9-A 扩展, https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/scalable-matrix-extension-armv9-a-architecture]` |
| **新一代鲲鹏**（团队 early-access / 内部可得；⚠️ 暂无公开来源，不指定 SKU 名） | **ARMv9-A**（⚠️ 团队 early-access 已可得；具体型号与公开可引用来源暂缺）`[来源: 团队 early-access 硅（⚠️ 暂无公开来源）；ARMv9-A 为 SVE2/SME 引入基线, https://developer.arm.com/documentation/102340/0100/Introducing-SVE2]` | **有**（NEON/Advanced SIMD 仍为 ARMv8/v9 基线 SIMD）`[来源: ARM ARM, NEON/Advanced SIMD 为 ARMv8-A 起的基线特性]` | **有（full SVE）**（⚠️ 团队 early-access 已可得；具体宽度与公开来源暂缺） | **有（FEAT_SVE2）**（⚠️ 团队 early-access 已可得；公开来源暂缺）`[来源: ARM Developer "Introducing SVE2", SVE2 引入于 ARMv9-A, https://developer.arm.com/documentation/102340/0100/Introducing-SVE2]` | **有（FEAT_SME / FEAT_SME2）**（⚠️ 团队 early-access 已可得；公开来源暂缺）`[来源: ARM Developer "Introducing the Scalable Matrix Extension for the Armv9-A Architecture", SME 是 Armv9-A 扩展, https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/scalable-matrix-extension-armv9-a-architecture]` |
| **(参考，仅供路线图背景) 鲲鹏 930 / 950 / 960 公开路线图** | 计划上市：**950 ~ 2026 Q4、960 ~ 2028 Q1**；930 公开状态长期未确认 `[来源: 同花顺 2026 鲲鹏路线图报道, https://news.10jqka.com.cn/20260320/c675452560.shtml]` ⚠️ 数量级（厂商计划，非已交付；团队**已不依赖**此路线图——early-access SVE/SME 硅现已可得） | — | — | — | — |

**表中关键事实的口径标注**：
- "920 = NEON-only / 无 SVE" 是 **确认✅**（多源交叉印证：IEEE Micro 官方论文 + VLDB Journal + Stony Brook）——**保持不变**。
- "新一代鲲鹏 = full SVE + SVE2 + SME" 是 **⚠️ early-access 已可得**（团队内部可得；⚠️ 暂无公开可引用来源，不指定 SKU 名）。**该 SVE/SME 硅不在 920 上**——他平台（A64FX / Graviton / LX2 / Apple M4 等）的 SVE/SME 性能数字需在本团队 early-access 硅上 re-measure 后方可作为本团队可引用数字。
- **关键变化（2026-06）**：团队**已不依赖**公开路线图（930/950/960）来确认 SVE/SME 可得性——新一代鲲鹏 early-access 硅现已可得。公开路线图行仅作背景保留。

---

## 2. P3 口径决议（Decision）

### 决议

> **P3 口径 = NEON（鲲鹏 920 现役）+ SVE/SVE2/SME（新一代鲲鹏 early-access 现已可得）同一融合方向——武器已齐备，团队可从今天起在自有 SVE/SME 硅上实测验证，无需再等未来平台。**
>
> **R1 重解（2026-06）**：原 "NEON-now + SVE/SME-roadmap"（等 930+/950+ 解锁）口径**作废**。原因——新一代鲲鹏（团队 early-access / 内部可得）现已实现 full SVE + SVE2 + SME（⚠️ 团队已可得；暂无公开可引用来源，不指定 SKU 名），团队不再等待未来平台。

### 三选一映射（对照 spec §11 R1 的两分支 + 本任务三选项）—— 重解

| 候选口径 | 触发条件 | 是否命中（重解后） |
|---|---|---|
| **have-SME** | 平台有 FEAT_SME | ⚠️ **命中（新一代鲲鹏 early-access 已可得 full SME，暂无公开来源）** |
| SVE-first pivot | 平台有 SVE/SVE2 但无 SME | 不再单独适用——新一代鲲鹏**同时**有 SVE/SVE2 与 SME |
| ~~NEON-now + SVE/SME-roadmap~~ | （原口径，已作废） | ❌ **作废**——SVE/SME 不再是"前瞻/路线图"，而是 early-access 已可得 |

> **重解后口径**：不再是 "NEON-now + SVE/SME-roadmap"，而是 **"NEON（920 现役）+ SVE/SVE2/SME（新一代鲲鹏 early-access 现已可得）= 同一融合方向的两段硅，武器已齐备"**。

### 一段式 rationale（供 controller sanity-check）

**采用 NEON（920 现役）+ SVE/SVE2/SME（新一代鲲鹏 early-access 现已可得）口径**：团队现役主力 **鲲鹏 920（TaiShan v110, ARMv8.2-A）经多源确认只具备 NEON（固定 128-bit Advanced SIMD），不实现 SVE / SVE2 / SME**（IEEE Micro 2021 官方芯片论文明确其 SIMD 为"NEON-basic + 部分 ARMv8.2 扩展"；VLDB Journal 2022 与 Stony Brook Ookami 文档均将 920 列为"128-bit NEON / 非 SVE"对照组）——**此事实与 ✅ 标注保持不变**。**但新一代鲲鹏（团队 early-access / 内部可得）现已实现 full SVE + SVE2 + SME（⚠️ 团队已可得；具体型号与公开可引用来源暂缺，不指定 930/940/950 等 SKU 名）**。因此 P3 **不再需要** "等 930+/950+ 解锁" 的前瞻框架：P3 论点 = "NEON（鲲鹏 920 现役）+ SVE/SVE2/SME（新一代鲲鹏 early-access）是同一融合方向——武器已齐备，团队可从今天起在自有 SVE/SME 硅上实测验证，无需再等未来平台"；P3-6 punchline 围绕"同一组向量/SIMD 单元（NEON 920 + SVE/SME 新一代鲲鹏）+ 同一份布局"重构；P3-4（SME 小批量 kernel）口径从"前瞻/路线图"改为"early-access 已可得、可在本团队硅上实测"。**他平台（A64FX / Graviton / LX2 / Apple M4 等）的 SVE/SME 性能数字仍为他平台实测，需在本团队 early-access SVE/SME 硅上 re-measure 后方可作为本团队可引用数字**（数据值与其 ⚠️/✅ 限定符不变）。**此重解强化 P3 叙事**："NEON 现在就能做融合"是真落地、"SVE/SME 在自有 early-access 硅上从今天起可实测验证"是真增量，二者共享同一融合主线。

---

## 3. P3-2 页草稿："SME/SVE/NEON 是什么、为什么对 agent 特别"

> 口径对齐：**NEON（鲲鹏 920 现役）+ SVE/SVE2/SME（新一代鲲鹏 early-access 现已可得）= 同一融合方向——武器已齐备**。本页既讲"是什么"，也讲"为什么 agent workload 特别受益"——为 P3-3 融合三件事铺武器。

### P3-2 标题：NEON（鲲鹏 920 现役）/ SVE·SVE2（新一代鲲鹏 early-access）/ SME·SME2（新一代鲲鹏 early-access）——ARM 的三类向量/矩阵武器，为什么对 agent 特别

**一句话论点**：agent workload 是"变长 / 不规则 / 小批量"的，NEON 已能做基础融合；SVE 的 **predicate + gather/scatter** 与 SME 的 **2D tile + outer-product 矩阵引擎** 正是为这种不规则负载设计——它们已在团队 early-access 硅上可得，从今天起可实测验证。

#### 3.1 三者各是什么（按"现役 → early-access 已可得"顺序）

**(A) NEON —— 现役（鲲鹏 920 现在就有）**
- 定义：ARM 的 **Advanced SIMD**，**固定 128-bit 向量宽度**（16 × 8-bit / 8 × 16-bit / 4 × 32-bit / 2 × 64-bit lane）`[来源: ARM ARM, Advanced SIMD / NEON；IEEE Micro 2021 "NEON-basic instructions"]`。
- 特性：**固定宽度**、**整向量操作**、**无 per-lane 谓词**、**无原生 gather/scatter**（需用 `LD1`/`TBL` 等组合实现非连续访问）`[来源: ARM ARM]`。
- 位置：**鲲鹏 920 现役可立即用** `[来源: IEEE Micro 2021 + VLDB Journal 2022, 见 §1 表]`。

**(B) SVE / SVE2 —— 新一代鲲鹏 early-access 已可得（⚠️ 暂无公开来源）；属 ARMv9-A**
- SVE（Scalable Vector Extension）三件套特性：
  1. **向量长度可扩展（vector-length agnostic）**：实现可选 **128–2048 bit**，软件与具体宽度解耦 `[来源: ARM SVE 论文 Stephens et al., IEEE Micro 2017 / Alastair Reid, https://alastairreid.github.io/papers/sve-ieee-micro-2017.pdf ；Stony Brook "implementations 128 to 2048 bits"]`。
  2. **Predicate 寄存器**（P0–P15，共 16 个）：**逐 lane 谓词**，对每个元素独立启用/禁用 `[来源: ARM SVE 论文；Microsoft DevBlogs "predicate registers", https://devblogs.microsoft.com/dotnet/engineering-sve-in-dotnet/]`。
  3. **原生 gather-load / scatter-store**：向量化**非连续**内存访问 `[来源: ARM SVE 论文；wasilzafar "SVE & SVE2" 三大特征, https://www.wasilzafar.com/pages/series/arm-assembly/arm-assembly-09-sve-sve2.html]`。
- SVE2（FEAT_SVE2）：SVE 的指令集扩展版，**引入于 ARMv9-A**，2022 年起 Arm 自研的 ARMv9-A 核心（Cortex-X3 / Neoverse V2 等）普遍实现 `[来源: ARM Developer "Introducing SVE2", https://developer.arm.com/documentation/102340/0100/Introducing-SVE2 ；Chips and Cheese "Neoverse V2", https://chipsandcheese.com/p/hot-chips-2023-arms-neoverse-v2]`。
- ⚠️ 注意：SVE2 在 ARMv9-A 中以特性 `FEAT_SVE2` 形式存在，社区对其"强制 vs 可选"有讨论，但**实践上 ARMv9-A 自研核心普遍带 SVE2** `[来源: AnandTech 论坛讨论；HN "all ARM cores designed by Arm and launched since 2022 support SVE2"]`。

**(C) SME / SME2 —— 新一代鲲鹏 early-access 已可得（⚠️ 暂无公开来源）；属 ARMv9-A 可选扩展**
- FEAT_SME（Scalable Matrix Extension）：在 SVE 之上的**矩阵扩展**，三要素：
  1. **ZA tile 寄存器**（`VL × VL` 的 **2D 数组**，可切分为命名 tile 如 `ZA0.B`）`[来源: ARM Developer "SME Introduction", matrix tile storage, https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/arm-scalable-matrix-extension-introduction]`。
  2. **Outer-product 引擎**：对两个 SVE 向量做外积 `C += A ⊗ B`，累加进 ZA tile；**外积之和 = 矩阵乘** `[来源: ARM Developer "SME Instructions Part 2", Zn/Zm 外积累加进 ZA；GWDG "ARM-SME-Introduction.pdf", "matrix product = sum of vector outer products"]`。
  3. **Streaming-mode SVE**：进入 SME 时切换到 streaming SVE 模式（可有不同向量长度）`[来源: ARM Developer SME 博客]`。
  - 核心指令：`MOPA`（Matrix Outer Product Add）/ `MOPS` `[来源: DATE 2026 "KirbyMM", MOPA 指令, https://date26.date-conference.com/proceedings-archive/2026/DATA/316.pdf]`。
- FEAT_SME2：在 SME 上加 **inner-product 指令**（如 `BFMMLA`/`SMMLA`）与 **multi-vector 操作**，加速 BLAS-2 `[来源: UoB HPC SimEng poster "SME2 adds inner-product and multi-vector instructions"]`。
- SME 为 ARMv9-A **可选**扩展，通常出现在 Cortex-X3+ / Neoverse V2+ 级核心 `[来源: ARM Developer SME 博客]`。

#### 3.2 为什么这些特性对 agent workload 特别（口径落地）

**agent runtime 的负载形态**（承袭 P1 已校验内容 + 主线 spine）：
- **变长**：agent 的 token / 消息 / 工具输出长度不规则 `[来源: 承袭 SWE-bench/SWE-agent 数据，spec §3 P1-3/P1-5]`。
- **不规则内存访问**：消息体（JSON / Protobuf / 嵌套结构）字段非连续；向量检索的 gather 式访问 `[来源: 承袭 P2-4 序列化开销 + P3-5 comm 侧]`。
- **小批量**：单 agent / 小 swarm 的 embedding / rerank / attention 是"瘦高"矩阵，非训练大 batch `[来源: 承袭 P3-4 论点]`。

**逐特性映射**：

| agent 负载痛点 | NEON（鲲鹏 920 现役）能做什么 | SVE（新一代鲲鹏 early-access）增量 | SME（新一代鲲鹏 early-access）增量 |
|---|---|---|---|
| 变长 / 尾部不足一整个向量 | 需软件处理边界、mask 模拟 | **Predicate** 原生逐 lane 谓词，变长天然适配 `[来源: ARM SVE 论文 predicate registers]` | — |
| 不规则 / 非连续内存（JSON、嵌套、检索 gather） | 用 `LD1`/`TBL` 组合拼凑 | **原生 gather/scatter**，向量化非连续访问 `[来源: ARM SVE 论文 gather/scatter]` | — |
| 序列化 / pack / 转置（comm 数据通路） | 128-bit SIMD 加速基础 pack/转置（**鲲鹏 920 现役可做**） | 向量长度可扩展 → 更高吞吐；predicate 处理不规则字段 | **ZA tile** 做 on-the-fly 转置/重排 `[来源: ARM SME "load/store/insert/extract including transposition"]` |
| 小批量 GEMM / embedding / attention | 128-bit 整向量 GEMM（小矩阵占满度低） | predicate 处理非整 tile 的"瘦高"尾边 | **outer-product + ZA tile** 天然适配小批量"瘦高"矩阵（区别于训练大 batch）`[来源: ARM SME 外积之和=矩阵乘；承袭 P3-4 论点]` |

> ⚠️ 上表 SVE/SME 增量已在新一代鲲鹏 early-access 硅上可得（⚠️ 暂无公开来源）；其具体加速倍数仍为他平台（A64FX / Graviton / LX2 / Apple M4 等）实测，**需在本团队 early-access SVE/SME 硅上 re-measure 后方可作为本团队可引用数字**（数据值与限定符不变）。

**一句话收束**：NEON（鲲鹏 920 现役）让**今天**就能在 SIMD 上做融合（向量化序列化 / pack / 基础 pack/转置）；SVE 的 predicate+gather/scatter 与 SME 的矩阵引擎，是"不规则、变长、小批量"agent 负载的**结构性匹配**——它们已在团队 early-access 硅上可得，从今天起可实测验证，融合会从"基础 SIMD 加速"升级到"predicate/gather-scatter/矩阵引擎级融合"。**前后不是两个方向，而是同一融合主线的两段硅，武器已齐备。**

---

## 4. 对下游 P3 页的口径传导（给 Task 6 用）

| 页 | 原 spec 措辞 | 重解后口径下的调整（NEON 920 现役 + SVE/SVE2/SME 新一代鲲鹏 early-access 已可得） |
|---|---|---|
| P3-3（融合三件事） | "SME/SVE 单元" | "**NEON（鲲鹏 920 现役）+ SVE/SME（新一代鲲鹏 early-access）同一组向量/SIMD 单元**"；三件事（重叠 / 数据搬运即计算 / kernel+原语协同）**在 NEON 上现在就能立论**，SVE/SME 是 early-access 已可得的增量增强（**非前瞻**）。 |
| P3-4（算子侧） | "已具备的算子机会 on SME" | **改为"early-access 已可得、可实测"口径**（不再是前瞻/路线图）：NEON 现役做小批量 embedding/rerank 的基础向量化；SME outer-product+tile 是新一代鲲鹏 early-access 上"瘦高"小批量 GEMM/attention 真增量，**可从今天起在本团队硅上实测**。⚠️ 具体加速倍数仍为他平台实测，需 re-measure 后引用；暂标"数量级 + 可自测 micro-benchmark"（R2）。 |
| P3-5（通信侧） | "SVE 加速 comm 数据通路" | **NEON（920 现役）落地 + SVE（新一代鲲鹏 early-access）已可得**：现役用 NEON 做 simdjson 式 JSON/序列化、零拷贝 IPC 搬运；SVE 的 gather/scatter 是新一代鲲鹏 early-access 上已可得的非连续消息搬运增强，**可实测**。承袭 simdjson 数据。 |
| P3-6（punchline） | "同一组单元=SME+SVE" | **重述为"同一组 SIMD 单元（NEON 920 + SVE/SME 新一代鲲鹏）+ 同一份布局——武器已齐备，团队可从今天起在自有 SVE/SME 硅上实测验证，无需再等未来平台"**（去对 SME 的硬依赖，也去掉"下一站"措辞）。 |
| P3-7（路线图） | 三层机会 | **去掉"等 930+/950+ 解锁"gate**。短期（现在–6 月）：NEON（鲲鹏 920 现役）立即可做 + SVE/SME（新一代鲲鹏 early-access）实测启动（simdjson 解析、iceoryx2 零拷贝、小-GEMM kernel、SVE gather/scatter 序列化、SME outer-product micro-bench）；中期（6–18 月）：SVE/SME 生产化/库化 + Agent-native 通信原语；长期（18–36 月）：Agent Workload Benchmark + 反哺 ARM。⚠️ 新一代鲲鹏暂无公开来源。 |

---

## 5. 来源清单（Sources）

**ARM 官方 / ARM ARM**
- ARM Developer, "Introducing SVE2": https://developer.arm.com/documentation/102340/0100/Introducing-SVE2
- ARM Developer, "SME Introduction": https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/arm-scalable-matrix-extension-introduction
- ARM Developer, "SME Instructions Part 2"（Zn/Zm 外积累加进 ZA）
- ARM Developer, "Introducing the Scalable Matrix Extension for the Armv9-A Architecture": https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/scalable-matrix-extension-armv9-a-architecture
- ARM SVE 论文, Stephens et al., IEEE Micro 2017（vector-length agnostic / predicate / gather-scatter）：https://alastairreid.github.io/papers/sve-ieee-micro-2017.pdf

**鲲鹏 / Huawei / HiSilicon 官方**
- HiSilicon Kunpeng 920 产品页：https://www.hisilicon.com/en/products/kunpeng/huawei-kunpeng/huawei-kunpeng-920
- IEEE Micro 2021, "Kunpeng 920: The First 7-nm Chiplet-Based 64-Core ARM SoC for Cloud Services"：https://www.computer.org/csdl/magazine/mi/2021/05/09444893/1u51AAgQwcU

**第三方权威（确认 920 = NEON-only）**
- VLDB Journal 2022, "To share or not to share vector registers?"（"the Kunpeng only supports ARM's SIMD extension called NEON with 128-bit"）：https://link.springer.com/article/10.1007/s00778-022-00744-2
- Stony Brook Ookami, "Arm SVE"（列 Kunpeng 920 = 128-bit NEON，作非-SVE 对照）：https://www.stonybrook.edu/commcms/ookami/support/_docs/Arm_SVE_reduced.pdf
- Chips and Cheese, "Huawei's Kunpeng 920 and TaiShan v110 CPU Architecture"：https://chipsandcheese.com/p/huaweis-kunpeng-920-and-taishan-v110
- ACM, "A Comparative Analysis of Kunpeng 920 on HPC Workloads"：https://dl.acm.org/doi/fullHtml/10.1145/3675018.3675768
- Chips and Cheese, "Hot Chips 2023: Arm's Neoverse V2"（SME/SVE2 出现的核心级别参考）：https://chipsandcheese.com/p/hot-chips-2023-arms-neoverse-v2

**SME/SVE 技术参考（论文 / 教程）**
- DATE 2026, "KirbyMM: Outer-Product Based Matrix Multiplication on ARMv9"（MOPA 指令）：https://date26.date-conference.com/proceedings-archive/2026/DATA/316.pdf
- arXiv, "Demystifying ARM SME to Optimize General Matrix Multiplications"：https://arxiv.org/html/2512.21473v1
- UoB HPC SimEng poster（SME2 inner-product / multi-vector）

**新一代鲲鹏 SVE/SME（⚠️ 团队 early-access 已可得；暂无公开可引用来源，不指定 SKU 名）**
- 团队 early-access 硅：full SVE + SVE2 + SME（⚠️ 内部可得，**无公开来源**；不写"930"/"940"/"950"等 SKU 名）。
- ARMv9-A / SVE2 / SME 的 ISA 定义来源（用于解释 early-access 平台的特性基线）：见上 "ARM 官方 / ARM ARM" 段。

**鲲鹏 930 / 950 / 960 公开路线图（⚠️ 厂商计划 / 展台口述，非正式发布数据；团队已不依赖此路线图确认 SVE/SME 可得性——early-access 硅现已可得）**
- 同花顺 2026, "华为鲲鹏950处理器将商用"（含 930 推迟 / 展台口称 / 950-960 路线图）：https://news.10jqka.com.cn/20260320/c675452560.shtml
- 知乎专栏, "华为鲲鹏930归来，ARM成为服务器趋势"：https://zhuanlan.zhihu.com/p/675438893
- 电子工程专辑, "华为首款服务器处理器鲲鹏930曝光"：https://www.eet-china.com/mp/a432942.html

**口径标注**：920 = NEON-only 无 SVE → **确认✅**（多源交叉，保持不变）。新一代鲲鹏 = full SVE + SVE2 + SME → **⚠️ early-access 已可得**（团队内部可得；暂无公开可引用来源，不指定 SKU 名；他平台数字需在本团队硅 re-measure）。950/960 公开时间线 → **数量级⚠️**（厂商计划；团队已不依赖）。
