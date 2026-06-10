# 研究文档：华为鲲鹏（Kunpeng）平台 SME/SVE/NEON 可用性 + P3 口径决议 + P3-2 草稿

- **日期**: 2026-06-09
- **任务**: 三部分重构计划的 Task 1 —— 解决 R1（平台 SME 可用性未查实）+ 为 P3-2（"SME/SVE/NEON 是什么、为什么对 agent 特别"）提供研究底稿与口径决议。
- **驱动 spec**: `docs/superpowers/specs/2026-06-09-cpu-agent-3part-restructure-design.md`（§1.2 统一主线、§5 P3 页表、§11 R1）。
- **引用铁律**: 每个 ISA 特性声明与每个性能数字均带内联 `[来源: …]`；性能对比标注 确认✅ / 数量级⚠️。

---

## 0. 结论先行（TL;DR）

| 问题 | 结论 |
|------|------|
| 团队现役主力平台是否有 SME/SVE？ | **否**。现役 **鲲鹏 920（TaiShan v110, ARMv8.2-A）只有 NEON（固定 128-bit SIMD）**，无 SVE / SVE2 / SME。 |
| 鲲鹏 930 是否已发布且带 SVE2/SME？ | **公开信息无法确认**。鲲鹏 930 **从未以独立型号正式发布**；2024 Q1 展出的"920 新型号"被展台工作人员口称为"930"，但其 ISA / 是否含 SVE2/SME **官方未公布**。 |
| **P3 口径决议** | **NEON-now + SVE/SME-roadmap**（详见 §2）。 |
| 何时解锁 SVE2/SME？ | 公开路线图显示 **鲲鹏 950（计划 2026 Q4）/ 鲲鹏 960（计划 2028 Q1）**；SVE2/SME 的确切搭载型号与时间**尚未官方确认**，应作为"前瞻/路线图"处理。 |

---

## 1. 平台特性表（Kunpeng 920 / 930 → NEON / SVE 宽度 / SVE2 / SME）

| 平台 / 微架构 | ISA 基线 | NEON（128-bit SIMD） | SVE 向量宽度 | SVE2 (FEAT_SVE2) | FEAT_SME / FEAT_SME2 |
|---|---|---|---|---|---|
| **鲲鹏 920 / TaiShan v110** | ARMv8.2-A `[来源: HiSilicon 官方产品页, https://www.hisilicon.com/en/products/kunpeng/huawei-kunpeng/huawei-kunpeng-920 ; IEEE Micro 2021 "Kunpeng 920: The First 7-nm Chiplet-Based 64-Core ARM SoC"]` | **有**（NEON-basic，128-bit 固定宽度）`[来源: IEEE Micro 2021 原文 "TaiShan V110's SIMD architecture is compatible with NEON-basic instructions and also supports part of the extension instructions of ARM-v8.2"；VLDB Journal 2022 "the Kunpeng only supports ARM's SIMD extension called NEON with 128-bit", https://link.springer.com/article/10.1007/s00778-022-00744-2 ；Stony Brook Ookami 文档列出 Kunpeng 920 = 128-bit NEON, https://www.stonybrook.edu/commcms/ookami/support/_docs/Arm_SVE_reduced.pdf]` | **无**（不实现 SVE）`[来源: 同上 Stony Brook / VLDB —— 二者均将 Kunpeng 920 列为"NEON-only / 非 SVE"对照组；社区亦确认 SVE 难以移植到此代，Reddit r/hardware "ARM CPU cores stuck at 128-bit SIMD"]` | **无**（SVE2 是 ARMv9-A 特性，920 为 ARMv8.2-A）`[来源: ARM Developer "Introducing SVE2", SVE2 引入于 ARMv9-A, https://developer.arm.com/documentation/102340/0100/Introducing-SVE2]` | **无**（SME 为 ARMv9-A 可选扩展，920 为 ARMv8.2-A）`[来源: ARM Developer "Introducing the Scalable Matrix Extension for the Armv9-A Architecture", SME 是 Armv9-A 扩展, https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/scalable-matrix-extension-armv9-a-architecture]` |
| **鲲鹏 930**（**未正式发布**；公开信息有限） | **官方未确认**。原计划 ~2021 因制裁推迟；2024 Q1 展出"920 新型号"被口称"930"，华为官方未以"鲲鹏930"名义单独发布 `[来源: 同花顺 2026 "华为鲲鹏950处理器将商用" 转述展台信息 "2024年一季度推出了鲲鹏920新型号（展台工作人员称其是鲲鹏930）", https://news.10jqka.com.cn/20260320/c675452560.shtml ；知乎专栏 "华为鲲鹏930归来" 讨论其推迟与状态, https://zhuanlan.zhihu.com/p/675438893]` | 假定有（NEON 是 ARMv8/v9 基线 SIMD）`[来源: ARM ARM, NEON/Advanced SIMD 为 ARMv8-A 起的基线特性]` | **公开未确认**。无权威来源证实 930 实现 SVE 或其向量宽度 | **公开未确认**。华为已获 **ARMv9 永久授权**（自 ARMv9 2021 发布起），但 930 是否落地 ARMv9/SVE2 **未官方公布** `[来源: 行业报道 华为 2013 获 ARMv8 永久授权、2021 后获 ARMv9 永久授权]` | **公开未确认**。SME 为 ARMv9-A 可选扩展，即便 930 走 ARMv9 路线，SME 是否搭载仍**无官方证据**。SME 通常出现在 Cortex-X3+ / Neoverse V2+ 级别核心 `[来源: ARM Developer SME 博客; Chips and Cheese "Hot Chips 2023: Arm's Neoverse V2", https://chipsandcheese.com/p/hot-chips-2023-arms-neoverse-v2]` |
| **(参考) 鲲鹏 950 / 960 路线图** | 计划上市：**950 ~ 2026 Q4、960 ~ 2028 Q1** `[来源: 同花顺 2026 鲲鹏路线图报道, https://news.10jqka.com.cn/20260320/c675452560.shtml]` ⚠️ 数量级（路线图为厂商计划，非已交付） | — | — | — | — |

**表中关键事实的口径标注**：
- "920 = NEON-only / 无 SVE" 是 **确认✅**（多源交叉印证：IEEE Micro 官方论文 + VLDB Journal + Stony Brook）。
- "930 的 SVE2/SME 状态" 是 **公开未确认**（不是"否"，而是"无权威公开数据"）——本表对此如实标注，不臆断。

---

## 2. P3 口径决议（Decision）

### 决议

> **P3 口径 = NEON-now + SVE/SME-roadmap。**

### 三选一映射（对照 spec §11 R1 的两分支 + 本任务三选项）

| 候选口径 | 触发条件 | 是否命中 |
|---|---|---|
| have-SME | 平台有 FEAT_SME | ❌（920 无 SME；930 SME 未确认） |
| SVE-first pivot | 平台有 SVE/SVE2 但无 SME | ❌（**920 连 SVE 都没有**，pivot 无落点） |
| **NEON-now + SVE/SME-roadmap** | 平台仅 NEON（固定 128-bit，无 predicate、无 gather/scatter、无矩阵引擎） | ✅ **命中** |

### 一段式 rationale（供 controller sanity-check）

**采用 NEON-now + SVE/SME-roadmap 口径**：团队现役主力 **鲲鹏 920（TaiShan v110, ARMv8.2-A）经多源确认只具备 NEON（固定 128-bit Advanced SIMD），不实现 SVE / SVE2 / SME**（IEEE Micro 2021 官方芯片论文明确其 SIMD 为"NEON-basic + 部分 ARMv8.2 扩展"；VLDB Journal 2022 与 Stony Brook Ookami 文档均将 920 列为"128-bit NEON / 非 SVE"对照组）。下一代 **鲲鹏 930 至今未以独立型号正式发布**，其是否采用 ARMv9、是否搭载 SVE2/SME **官方均未公布**（华为虽已获 ARMv9 永久授权，但无证据表明 930 已量产 SVE2/SME 核心）。因此 P3 不能宣称"已有 SME"或"SVE-first"，而应落定 **NEON-now + SVE/SME-roadmap** 口径：P3 论点 = "在 NEON 上现在就能做的融合（向量化序列化 / pack / 基础 SIMD）+ SVE2/SME 在鲲鹏 930+/950+ 上的前瞻解锁（predicate / gather-scatter / 矩阵引擎才是真正大的融合机会）"；P3-6 punchline 围绕"同一组 SIMD 单元（NEON now）+ 同一份布局"重构，并将 SVE/SME 的解锁作为"同一融合方向的下一站"；P3-4（SME 小批量 kernel）口径为**前瞻/路线图**。**此口径反而强化 P3 叙事**："在现有 NEON 上今天就能做融合"是真落地，"predicate/gather-scatter/矩阵引擎随 SVE2/SME 解锁"是真增量——前者给可信度、后者给想象力，二者共享同一融合主线。

---

## 3. P3-2 页草稿："SME/SVE/NEON 是什么、为什么对 agent 特别"

> 口径对齐：**NEON = 现在就有（now）；SVE/SVE2 与 SME = 鲲鹏 930+/950+ 前瞻解锁（roadmap）**。本页既讲"是什么"，也讲"为什么 agent workload 特别受益"——为 P3-3 融合三件事铺武器。

### P3-2 标题：NEON（现在）/ SVE（前瞻）/ SME（前瞻）——ARM 的三类向量/矩阵武器，为什么对 agent 特别

**一句话论点**：agent workload 是"变长 / 不规则 / 小批量"的，NEON 已能做基础融合；SVE 的 **predicate + gather/scatter** 与 SME 的 **2D tile + outer-product 矩阵引擎** 正是为这种不规则负载设计——它们是融合的"下一站"。

#### 3.1 三者各是什么（按"现役 → 前瞻"顺序）

**(A) NEON —— 现役（鲲鹏 920 现在就有）**
- 定义：ARM 的 **Advanced SIMD**，**固定 128-bit 向量宽度**（16 × 8-bit / 8 × 16-bit / 4 × 32-bit / 2 × 64-bit lane）`[来源: ARM ARM, Advanced SIMD / NEON；IEEE Micro 2021 "NEON-basic instructions"]`。
- 特性：**固定宽度**、**整向量操作**、**无 per-lane 谓词**、**无原生 gather/scatter**（需用 `LD1`/`TBL` 等组合实现非连续访问）`[来源: ARM ARM]`。
- 位置：**鲲鹏 920 现役可立即用** `[来源: IEEE Micro 2021 + VLDB Journal 2022, 见 §1 表]`。

**(B) SVE / SVE2 —— 前瞻（鲲鹏 930+/950+ 解锁；属 ARMv9-A）**
- SVE（Scalable Vector Extension）三件套特性：
  1. **向量长度可扩展（vector-length agnostic）**：实现可选 **128–2048 bit**，软件与具体宽度解耦 `[来源: ARM SVE 论文 Stephens et al., IEEE Micro 2017 / Alastair Reid, https://alastairreid.github.io/papers/sve-ieee-micro-2017.pdf ；Stony Brook "implementations 128 to 2048 bits"]`。
  2. **Predicate 寄存器**（P0–P15，共 16 个）：**逐 lane 谓词**，对每个元素独立启用/禁用 `[来源: ARM SVE 论文；Microsoft DevBlogs "predicate registers", https://devblogs.microsoft.com/dotnet/engineering-sve-in-dotnet/]`。
  3. **原生 gather-load / scatter-store**：向量化**非连续**内存访问 `[来源: ARM SVE 论文；wasilzafar "SVE & SVE2" 三大特征, https://www.wasilzafar.com/pages/series/arm-assembly/arm-assembly-09-sve-sve2.html]`。
- SVE2（FEAT_SVE2）：SVE 的指令集扩展版，**引入于 ARMv9-A**，2022 年起 Arm 自研的 ARMv9-A 核心（Cortex-X3 / Neoverse V2 等）普遍实现 `[来源: ARM Developer "Introducing SVE2", https://developer.arm.com/documentation/102340/0100/Introducing-SVE2 ；Chips and Cheese "Neoverse V2", https://chipsandcheese.com/p/hot-chips-2023-arms-neoverse-v2]`。
- ⚠️ 注意：SVE2 在 ARMv9-A 中以特性 `FEAT_SVE2` 形式存在，社区对其"强制 vs 可选"有讨论，但**实践上 ARMv9-A 自研核心普遍带 SVE2** `[来源: AnandTech 论坛讨论；HN "all ARM cores designed by Arm and launched since 2022 support SVE2"]`。

**(C) SME / SME2 —— 前瞻（鲲鹏 930+/950+ 解锁；属 ARMv9-A 可选扩展）**
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

| agent 负载痛点 | NEON（现役）能做什么 | SVE（前瞻）增量 | SME（前瞻）增量 |
|---|---|---|---|
| 变长 / 尾部不足一整个向量 | 需软件处理边界、mask 模拟 | **Predicate** 原生逐 lane 谓词，变长天然适配 `[来源: ARM SVE 论文 predicate registers]` | — |
| 不规则 / 非连续内存（JSON、嵌套、检索 gather） | 用 `LD1`/`TBL` 组合拼凑 | **原生 gather/scatter**，向量化非连续访问 `[来源: ARM SVE 论文 gather/scatter]` | — |
| 序列化 / pack / 转置（comm 数据通路） | 128-bit SIMD 加速基础 pack/转置（**现役可做**） | 向量长度可扩展 → 更高吞吐；predicate 处理不规则字段 | **ZA tile** 做 on-the-fly 转置/重排 `[来源: ARM SME "load/store/insert/extract including transposition"]` |
| 小批量 GEMM / embedding / attention | 128-bit 整向量 GEMM（小矩阵占满度低） | predicate 处理非整 tile 的"瘦高"尾边 | **outer-product + ZA tile** 天然适配小批量"瘦高"矩阵（区别于训练大 batch）`[来源: ARM SME 外积之和=矩阵乘；承袭 P3-4 论点]` |

**一句话收束**：NEON 让**今天**就能在 SIMD 上做融合（向量化序列化 / pack / 基础 pack/转置）；SVE 的 predicate+gather/scatter 与 SME 的矩阵引擎，是"不规则、变长、小批量"agent 负载的**结构性匹配**——它们随鲲鹏 930+/950+ 解锁时，融合会从"基础 SIMD 加速"升级到"predicate/gather-scatter/矩阵引擎级融合"。**前后不是两个方向，而是同一融合主线的现在与下一站。**

---

## 4. 对下游 P3 页的口径传导（给 Task 6 用）

| 页 | 原 spec 措辞 | NEON-now 口径下的调整 |
|---|---|---|
| P3-3（融合三件事） | "SME/SVE 单元" | "**NEON（现役）+ SVE/SME（前瞻）同一组向量/SIMD 单元**"；三件事（重叠 / 数据搬运即计算 / kernel+原语协同）**在 NEON 上现在就能立论**，SVE/SME 是增量增强。 |
| P3-4（算子侧） | "已具备的算子机会 on SME" | **改为"前瞻/路线图"口径**：NEON 现役做小批量 embedding/rerank 的基础向量化；SME outer-product+tile 是 930+/950+ 解锁的"瘦高"小批量 GEMM/attention 真增量。⚠️ 若无 SME 实测数据，标"数量级 + 可自测 micro-benchmark"（R2）。 |
| P3-5（通信侧） | "SVE 加速 comm 数据通路" | **NEON 现役落地 + SVE 前瞻**：现役用 NEON 做 simdjson 式 JSON/序列化、零拷贝 IPC 搬运；SVE 的 gather/scatter 是 930+ 解锁的非连续消息搬运增强。承袭 simdjson 数据。 |
| P3-6（punchline） | "同一组单元=SME+SVE" | **重述为"同一组 SIMD 单元（NEON now）+ 同一份布局；SVE/SME 是同一融合方向的下一站"**（去对 SME 的硬依赖）。 |
| P3-7（路线图） | 三层机会 | 明确把 **SVE2/SME 解锁节点（930+/950+）写进时间线**；短期全部用 NEON 可立即做。 |

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

**鲲鹏 930 / 路线图（⚠️ 厂商计划 / 展台口述，非正式发布数据）**
- 同花顺 2026, "华为鲲鹏950处理器将商用"（含 930 推迟 / 展台口称 / 950-960 路线图）：https://news.10jqka.com.cn/20260320/c675452560.shtml
- 知乎专栏, "华为鲲鹏930归来，ARM成为服务器趋势"：https://zhuanlan.zhihu.com/p/675438893
- 电子工程专辑, "华为首款服务器处理器鲲鹏930曝光"：https://www.eet-china.com/mp/a432942.html

**口径标注**：920 = NEON-only 无 SVE → **确认✅**（多源交叉）。930 的 SVE2/SME 状态 → **公开未确认**（如实标注，不臆断）。950/960 时间线 → **数量级⚠️**（厂商计划）。
