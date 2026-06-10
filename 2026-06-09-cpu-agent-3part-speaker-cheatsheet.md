# 演讲速查卡（Speaker Cheat-Sheet）

**主题**：CPU 底层软件团队：Agent 时代研究方向（三部分重构版）
**时长**：30 分钟 ｜ **正文页**：~21 ｜ **来源**：`2026-06-09-cpu-agent-3part-ppt-detailed.md`（citation-rigorous + 对抗式数据校验）
**引用铁律**：确认✅可直接引用；数量级⚠️必须带"约/数量级"+语境；GPU 特定项必须标"GPU 类比"；early-access 平台项标"新一代鲲鹏 early-access，暂无公开来源"。

---

## 1. 一句话主线（spine，逐字来自源文档统一主线）

> **Agent 运行时 = 一段不断在「计算」与「通信」之间来回切换的长循环。** 从单 Agent 的"工具调用往返"，到多 Agent 的"A2A 协作"，**通信占比随规模上升、逐渐压过计算成为主导成本**。而我们的 ARM 平台——**NEON（鲲鹏 920 现役）+ SVE/SVE2/SME（新一代鲲鹏 early-access 现已可得）是同一融合方向——武器已齐备，团队可从今天起在自有 SVE/SME 硅上实测验证，无需再等未来平台**——它不只是算子加速器，更是能把"通信的数据搬运/序列化"与"计算的向量化/矩阵化"放进**同一组执行单元**的融合点。**通信计算融合**，就是我们在 Agent 时代的主场。

---

## 2. 五个主线触点（🎯 标记页，全场三次点题贯穿）

| PAGE | 触点名 | 一句话点题 |
|------|--------|-----------|
| **5（P1-4）** | 🎯 主线第一次点题 | loop = 计算↔通信的反复交织；API 等待期间本地 CPU 利用率~0%（串行流水线的灰色空闲）就是优化核心靶点。 |
| **12（P2-6）** | 🎯 diagnosis 落地 | 从单 Agent 到多 Agent，通信占比随规模上升、渐成主导——单 agent → 小 swarm → 大 swarm 一张图收成一句话论断。 |
| **13（P2-7）** | 🎯 主线第二次点题（handoff） | 通信变贵了，谁来付？（只问不答，制造张力，把球交给 P3） |
| **16（P3-3）** | 🎯 thesis 展开 | 通信计算融合 = 三件事：①重叠 ②数据搬运即计算 ③kernel+原语协同。现役 NEON 今天就能做。 |
| **19（P3-6）** | 🎯 prescription climax（第三次点题） | 同一组单元、同一份布局、无边界拷贝；融合论点不依赖 SME，现役 NEON 上融合已在发生。 |

---

## 3. 时间分配（每段一句"emphasize"）

| 段落 | 时长 | emphasize（强调一句） |
|------|------|----------------------|
| **开场** | 1 min | 强调**边界**——不聊训模型/单 token 推理，聊"工具执行占 35%-61% 的系统层"，这是我们的主场。 |
| **P1 基础** | 8 min | 强调**loop 本质是 compute↔comm 乒乓**，API 等待 CPU~0% 是灰色空闲靶点（PASTE 推测执行 -48.5% 反证）。 |
| **P2 洞察** | 10 min | 强调**口径**——"2-5×"是 token/成本经验法则（非总时间），"58%"是调度气泡（非工具时间）；四组数据全指向"通信成主导"。 |
| **P3 融合** | 9 min | 强调**武器已齐备**：NEON（鲲鹏 920 现役）今天就能做融合 + SVE/SME（新一代鲲鹏 early-access）现已可得、可即起实测；同一组单元、同一份布局。 |
| **总结** | 2 min | 强调**主场收束 + 行动**：三个讨论问题 = 三个可立即在鲲鹏 920 上启动的调研任务（simdjson / iceoryx2 / 小-GEMM micro-bench）。 |

---

## 4. 必背关键数字（top 10，带限定词 + 口播说法）

> 全部来自源文档「关键数据速览」与附录。**逐字照搬限定词，绝不在口播里把"约"或"⚠️"抹掉。**

| # | 数字（含限定词） | 口播说法（how to say it aloud） |
|---|------------------|-------------------------------|
| 1 | 工具执行占 agent 总请求时间 **35%-61%**（编码任务高达 60%）[来源: PASTE, arXiv 2603.18897] ✅ | "一个 coding agent 的请求里，真正花在跑工具——编译、测试、读写文件——上的时间，占了 35% 到 61%，编码任务里高达 60%。一大半延迟根本不在 LLM 身上。" |
| 2 | SWE-bench Verified 单实例上限 **250 步 / 3 美元** [来源: arXiv 2511.13646] ✅ | "SWE-bench Verified 给每个实例的标准预算上限是 250 步、3 美元。一个真实 coding agent 任务要发生几十到几百次'LLM 推理加工具调用'的往返。" |
| 3 | PASTE 推测执行：任务完成时间 **-48.5%**、工具吞吐 **+1.8×**、工具等待 **-67%** [来源: arXiv 2603.18897] ✅ | "PASTE 论文用推测执行反证了'等待是瓶颈'——在 LLM 思考时并行执行下一个工具，任务时间砍掉 48.5%，工具等待减少 67%。" |
| 4 | API 调用延迟：GPT-4o ~**570ms**、Claude 3.5 ~**671ms**、DeepSeek ~**23.64s/请求**（⚠️ 数量级估算，公开监控数据，随负载/时间变化）[来源: AILatency/Artificial Analysis/Medium, 2024-2025] | "API 延迟是数量级估算——GPT-4o 约 570 毫秒，Claude 约 671 毫秒，DeepSeek 官方 API 平均约 23 秒一次。这些是公开监控数据，随负载变化。" |
| 5 | API 等待期间本地 CPU 利用率 **~0%**（⚠️ 数量级估算，串行阻塞场景）[来源: 团队早期版本] | "在等 API 的 500 毫秒到 5 秒里，本地 CPU 利用率接近 0%——注意这是数量级估算，前提是串行阻塞场景。" |
| 6 | 多 Agent 比单 Agent 贵约 **2-5×** token/成本（⚠️ 经验法则/数量级估算，指 token-or-cost 而非 total-time，受任务/AI 数/轮数影响）[来源: 团队早期版本，综合 UIUC arXiv:2505.18286] | "多 Agent 比单 Agent 贵约 2 到 5 倍——但这是 token/成本的经验法则，不是总时间测量，随任务、agent 数、轮数变化。" |
| 7 | Continuum 多 Agent 调度气泡可达 **~58%**（⚠️ 调度气泡/任务间隙，**非工具调用时间**）[来源: Continuum, arXiv:2511.02230] ✅ | "Continuum 测得多 Agent 调度气泡可达约 58%——这是工具返回后等 KV-cache 重新驻留的空闲间隙，不是工具调用本身的执行时间。" |
| 8 | 锁竞争高竞争退化 **10-100×**（✅ 已确认；实际倍数取决于临界区/线程核数比/锁类型/工作负载）[来源: ACCU Overload 2020 / EMSE 2017] ✅ | "锁竞争是已确认的数字——多 Agent 共享状态时，高竞争下性能退化 10 到 100 倍。这是数量级，实际倍数取决于临界区和锁类型。" |
| 9 | FlashAttention **2-4×** 加速 [来源: arXiv 2205.14135] ✅ | "FlashAttention 把 HBM 读写从 O(N²) 降到 O(N)，带来 2 到 4 倍加速——这是确认的，但属于'我们不碰'的单次推理内核优化。" |
| 10 | 现役 NEON（鲲鹏 920）今天就能做融合；SVE/SVE2/SME（新一代鲲鹏 early-access，⚠️ 暂无公开来源）现已可得、可即起实测[来源: 平台文档 §1] ✅ | "现役 NEON——鲲鹏 920 今天就有——已经能做基础融合。新一代鲲鹏的 SVE/SVE2/SME 团队已 early-access 可得，能从今天起在自有硅上实测；他平台数字（A64FX/Graviton/LX2/M4）仍需在本团队 SVE/SME 硅上 re-measure 后引用。" |

---

## 5. 口径护栏（guardrails——什么 NOT to say）

> 逐字来自源文档「数据校验报告」的纠正项。**不要说 X，要说 Y。**

1. **INT8 外积指令**：不要说 **FMOPA**，要说 **SMOPA**（INT8→INT32 整数外积；FMOPA 是浮点助记符）。
2. **FlatBuffers wire size**：不要说 **348B**，要说 **344B**。
3. **MPI overlap 论文作者**：不要说 **Brightwell**，要说 **Bernholdt**（Bernholdt/Shet 等，OSTI 944757, 2008）。
4. **iceoryx2**：不要说 **iceoryx2 比 raw shared memory 快**——同一来源显示它单字节反而慢约 600ns。要说"常数级 sub-µs、与 payload 无关"（定性/数量级）。
5. **LibXSMM**：不要说"小 GEMM **一律 10× 加速**"。要说"**>10× 是可达/上限，典型 MKL 对比多为 2-4×**，取决于 CPU 与应用形态"。
6. **鲲鹏 920**：不要说**鲲鹏 920 是 Arm Neoverse V2**——它用华为自研 TaiShan 核（ARMv8.2-A，NEON-only）。要说"**鲲鹏 920 = NEON-only，无 SVE/SVE2/SME**"（多源确认：IEEE Micro 2021 + VLDB Journal 2022 + Stony Brook Ookami）。
7. **鲲鹏 920 vs SVE/SME**：不要说"**鲲鹏 920 支持 SVE/SVE2/SME**"——它**不支持**（ARMv8.2-A NEON-only）。NEON = 鲲鹏 920 现役；**SVE/SVE2/SME 在新一代鲲鹏（团队 early-access，⚠️ 暂无公开来源）上可得，不在 920 上**。他平台数字（A64FX/Graviton/LX2/M4）需在本团队硅 re-measure 后引用。
8. **ARM 服务器核**：不要说"**鲲鹏 = V2**"。要说"**Arm V 系列服务器核（Neoverse V1/V2）**"，不暗示鲲鹏是 V2。
9. **多 Agent "2-5×"**：不要说"**多 Agent 总时间是单 Agent 的 2-5 倍**"。要说"**约 2-5× token/成本（经验法则，非总时间）**"。
10. **"58%"**：不要说"**工具调用时间占 58%**"。要说"**调度气泡可达 ~58%（任务间隙，非工具调用时间）**"。
11. **GPU 特定数据**：不要把 **Georgia Tech 重叠减速 18.9%/40%、FlashDecoding++ 补零 50%** 当 CPU/NEON 实测数字。要标"**GPU 实测，CPU 定性类比**"。
12. **SMOPA/MOPA 性能**：不要把 **0.512 TFLOPS/核** 当实测稠密 GEMM 吞吐。要说"**峰值 0.512（peak）；实测稠密 GEMM 仅约 0.33**"，且**在 LX2（非当前 920）上测**。
13. **iceoryx2 "100ns 改进"**：不要当产品加速引用。那是 benchmark 方法论脚注（warmup 降频）。

---

## 6. 如果被问到（likely Q&A）

**Q1：你说"通信成主导成本"，可现役鲲鹏 920 没有 SVE/SME，融合怎么落地？**
答：融合论点**不依赖 SME 才能成立**——现役 NEON（鲲鹏 920 今天就有）上，序列化/pack/转置本身已经是 128-bit SIMD 操作，simdjson 式 GB/s 解析、vzip/vuzp/vtrn 转置、小-GEMM kernel（LibXSMM/LibShaLom 风格）、iceoryx2 零拷贝 IPC 都是"今天就能做"的真落地。而 SME 相关部分也已是团队 early-access 可得（不再是未来平台的事）：团队已拿到新一代鲲鹏的 SVE/SME early-access 硅，可直接在本团队硬件上验证 SME outer-product、SVE gather/scatter 序列化等增量——⚠️ 仍是 early-access（暂无公开可引用来源），且他平台数字（A64FX/Graviton/LX2/M4）需在本团队硅 re-measure 后再引用。

**Q2："多 Agent 贵 2-5×" 这个数到底怎么来的？可信吗？**
答：这是**经验法则/数量级估算，指 token-or-cost 而非总时间**，随任务、agent 数、轮数变化，不可当精确乘数。要具体数：UIUC（arXiv:2505.18286）实测多 Agent prefill token 约 4×-220×（排除最难数据集 AIME 后典型 ~5-35×）；中心化 supervisor-worker MAS 约 285% 额外 token（≈3.85×）——但那是**拓扑特定，非通用乘数**。

**Q3：你强调 ARM 主场，但学界这些都是 GPU/A64FX/Apple M4 上的数据，鲲鹏上能复现吗？**
答：诚实地说，KirbyMM/MpGEMM/Open MPI SVE 这些**都不是在当前鲲鹏 920 上测的**（920 = NEON-only）。它们的口径必须标清楚：KirbyMM 在 LX2（920 的继任者核 ARMv9/SME）、Open MPI SVE 在 A64FX、MpGEMM 在 Apple M4 Pro——都是"SME 家族行为，需在目标硅上实测验证"。短期我们能在鲲鹏 920 上自测 NEON 现役项（simdjson GB/s、小-GEMM micro-benchmark、iceoryx2）；**同时团队已拿到新一代鲲鹏 SVE/SME early-access 硅**，可同步在本团队硬件上 re-measure 上述 SVE/SME 增量（⚠️ early-access，暂无公开来源；他平台数字需在本团队硅上实测后再引用）。

**Q4：Agent 真的需要强一致共识吗？那套 etcd/Raft 的 ms 级延迟墙不是太重？**
答：判断是——**Agent 系统通常可容忍最终一致性**，不需要银行级强一致。无状态 + 消息传递（Actor/事件驱动）模式延迟仅约 1-5μs（⚠️ 数量级），无锁队列低线程数下比 std::mutex 队列吞吐高 75%-150%。而 MPI 的集合通信（Allreduce/Broadcast）正是无锁、去中心化 ring/tree 算法的工业级实现——与 Actor 模型同源，正是我们 MPI/UCX 能力直接映射的主场。

---

*每条数字均带 `[来源: ...]` 与 ✅/⚠️ 限定词，与详细版「数据校验报告」一致；引用前回查源文档附录1（绝对延迟）/附录2（加速比）/附录3（平台 ISA）。*
