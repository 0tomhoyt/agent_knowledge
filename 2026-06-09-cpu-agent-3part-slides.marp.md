---
marp: true
theme: gaia
paginate: true
size: 16:9
header: 'CPU底层软件团队 · Agent时代研究方向'
footer: '2026-06 · 三部分重构版'
title: 'CPU底层软件团队 · Agent时代研究方向'
author: 'CPU底层软件团队'
---

## Agent 运行时 = 一段在「计算」与「通信」之间反复切换的长循环

- Agent 不是"更大的模型"，而是**带工具的长循环**：几十到上百次"LLM 推理 + 工具调用"往返
- 工具执行（编译/测试/IO）吃掉 **35%-61%** 请求时间，编码任务高达 **60%** [来源: PASTE, arXiv 2603.18897] ✅
- 单→多 Agent 转折：通信占比随规模上升，渐成主导成本
- 多 Agent 比 Agent ~**2-5×** token/成本（⚠️ 数量级估算，受任务/AI 数量/轮数影响）[来源: 团队早期版本]
- ARM 主场：NEON（鲲鹏 920 现役）+ SVE/SVE2/SME（新一代鲲鹏 early-access 现已可得）是同一融合方向——武器已齐备，团队可从今天起在自有 SVE/SME 硅上实测验证，无需再等未来平台

<!-- 开场先划边界：不聊训模型、不聊单 token 推理优化。一个 coding agent 请求里，跑工具占了 35%-61%，一大半延迟不在 LLM 身上，而在进程/IPC/序列化/并发——我们的主场。从单到多 Agent，通信随规模压过计算；ARM 上 NEON（鲲鹏 920 现役）+ SVE/SME（新一代鲲鹏 early-access 现已可得）是同一融合方向，武器已齐备，无需再等未来平台。 -->

---

## 三部分议程：problem setup → diagnosis → prescription

- **P1 基础（8min）**：Agent 执行流程 = 计算↔通信的反复交织
- **P2 洞察（10min）**：单→多 Agent，通信随规模上升、渐成主导（现状 + 数据论证）
- **P3 融合（9min）**：ARM 平台（NEON 鲲鹏 920 现役 + SVE/SVE2/SME 新一代鲲鹏 early-access）的通信计算融合 = 主场机会
- 全场一句话 spine：**通信占比随规模上升、逐渐压过计算成为主导成本**
- 统一融合点：把"通信的数据搬运/序列化"与"计算的向量化/矩阵化"放进**同一组执行单元**

<!-- 三部分：第一部分看 agent loop 的本质；第二部分看单到多 Agent 成本结构怎么变；第三部分看 ARM 融合机会。先从第一部分开始。 -->

---

## Agent 运行时 = loop 里被 LLM 光环掩盖的系统层

- **不碰**：pre-training/fine-tuning、GPU 利用率、单次推理内核优化
  - FlashAttention 约 **2-4×** 加速（HBM 读写 O(N²)→O(N)）[来源: arXiv 2205.14135] ✅
  - PagedAttention 让 vLLM 吞吐 **2-4×** [来源: arXiv 2309.06180] ✅
- **聚焦**：感知→推理→规划→工具调用→记忆更新长循环里的"非 LLM"部分
  - CPU 调度（进程启动/prefork/池化）、内存访问（向量检索 gather/距离）
  - 工具通信（IPC：HTTP/UDS/共享内存）、序列化（JSON/Protobuf/FlatBuffers）
  - 资源瓶颈（锁竞争、负载不均、fail-slow）
- 主场映射：**GEMM/Attention + MPI/UCX/SDMA** 能力直接对应

<!-- 边界划清楚：今天不聊训模型、不聊 GPU 单 token 推理。我们聊当 LLM 变成"循环"后整个系统成本结构怎么变。循环里被 LLM 光环掩盖的系统层——进程/IPC/序列化/并发——恰是经典系统工程，也是 GEMM/MPI/UCX/SDMA 能力映射的地方。 -->

---

## Agent 不是"一次 API"，而是混合/突发/长尾的长循环

- 标准循环（ReAct）：**感知→推理→规划→工具调用→记忆更新→ 回到感知** [来源: ReAct, arXiv 2210.03629]
- 长循环权威数据：SWE-bench Verified 单实例上限 **250 步 / 3 美元** [来源: arXiv 2511.13646] ✅
- SWE-bench Pro 每任务 **50 次工具调用**（"相当标准"）[来源: Scale AI, OpenReview 9R2iUHhVfr] ✅
- 每步不同计算形态：感知(CPU 密集)、推理(远程→CPU 空闲)、规划(轻逻辑)、工具(CPU/IO 混合)、记忆(向量检索)
- **一个真实 turn = 计算↔通信乒乓**：`感知 → [call_llm ~570ms] → read_file(fork 6.5ms) → [call_llm] → run_tests(35-61%) → answer`（250 步上限 = 几十~上百次这样的 turn）
- 新 workload 本质：**混合 / 突发 / 长尾 / 依赖复杂**——传统调度假设失效

<!-- 把 agent 工作流画出来：五个字一个圈。两个特征决定它和普通推理服务不一样：一是长循环（SWE-bench 上限 250 步/$3，几十到上百次往返）；二是每步不同计算形态，导致混合/突发/长尾/依赖复杂的负载。传统面向单一计算的调度器处理不了。 -->

---

## 循环的每一步，是不同的计算形态、不同的资源需求

| 步骤 | 计算形态 | 资源需求 | 典型延迟 |
|------|---------|---------|---------|
| 感知 | 文本处理/tokenization | CPU 密集、内存带宽 | JSON 解析 GB/s 级 |
| 推理 | LLM API 调用 | 网络 IO，本地 CPU 空闲 | GPT-4o ~570ms、Claude ~671ms |
| 规划 | 轻量推理/规则匹配 | 计算轻、逻辑复杂 | µs-ms 级 |
| 工具调用 | 代码分析/profiler + IO | CPU/IO 混合、可能 fork | 编译秒-分钟级、fork 1GB **6.5ms** |
| 记忆更新 | 向量检索 + rerank | 内存随机访问、小批量 | MiniLM **4.2ms/句**、rerank 5-50ms/doc |

- 工具执行占请求时间 **35%-61%**（编码 60%）[来源: PASTE, arXiv 2603.18897] ✅
- fork 1GB 进程 **6.5ms**，50GB **253.9ms** [来源: On-demand-fork, EuroSys 2021, Purdue] ✅

**手搓最简 agent：整个循环就是几十行 Python**

```python
for step in range(250):              # SWE-bench 上限 250 步/$3 ✅
    resp = call_llm(ctx)             # ← 远程通信 ~570ms，等待期 CPU~0% ⚠️
    if "answer" in resp: break
    result = TOOLS[resp["action"]["name"]](**resp["action"]["args"])  # ← 本地计算，占 35-61% ✅
    ctx += [{"role":"assistant","content":json.dumps(resp)}, {"role":"user","content":result}]  # 序列化税
# 整个 agent = for 循环 + 远程调用 + 本地工具 + ctx 累积 = 计算↔通信乒乓
```

<!-- 每步拆开：感知 CPU 密集（simdjson GB/s 级）；推理远程高延迟，等待期本地 CPU 接近 0%；规划轻逻辑；工具 CPU/IO 混合还可能 fork，fork 1GB 要 6.5ms、50GB 要 253ms；记忆更新向量检索 MiniLM 4.2ms/句、reranker 100 候选 500ms-5s。每步不同计算形态——异构负载根源。 -->

---

## 🎯 主线第一次点题：loop = 计算↔通信的反复交织

- 把五步重画成 **"本地计算 ↔ 跨边界通信"** 的乒乓
  - **本地计算**：感知、规划、记忆更新、工具前后处理
  - **跨边界通信**：推理(LLM API)、工具调用(远程/IPC/文件网络 IO)
- **关键现象**：API 等待期（**500ms-5s+**，⚠️ 数量级估算）本地 CPU 利用率 **~0%**（⚠️ 串行阻塞场景）
- API 延迟：GPT-4o ~**570ms**、Claude ~**671ms**、GPT-4o Mini **200-400ms**、DeepSeek ~**23.64s** [来源: AILatency/SitePoint/Medium, 2024-2025] ⚠️ 数量级估算
- **灰色空闲 = 优化核心靶点**：PASTE 推测执行使任务时间 **-48.5%**、工具吞吐 **+1.8×**、工具等待 **-67%** [来源: arXiv 2603.18897] ✅
- IPC 延迟：共享内存 P99 ~**850ns**、UDS ~**130µs**、TCP loopback ~**334µs**、HTTP localhost **0.1-1ms** [来源: HowTech/NodeVibe/Medium] ⚠️ 数量级估算

<!-- 🎯 这一页是主线第一次点题，请记住这幅乒乓图。Agent 循环本质就是"本地计算"和"跨边界通信"来回切换的长循环。等待 API 的 500ms-5s 里本地 CPU 接近 0%——串行流水线的灰色空闲，优化核心靶点。PASTE 用推测执行证明：等待本身就是瓶颈，任务时间砍 48.5%。"Agent 运行时 = 一段在计算与通信间反复切换的长循环"——全场主线，记住它。 -->

---

## Agent 是混合/突发/长尾/依赖复杂的新 workload

- **混合**：compute + comm 剧烈交替，工具执行占 **35%-61%**（编码 60%）[来源: PASTE, arXiv 2603.18897] ✅
- **突发**：SWE-agent 成功 **12 步/$1.21** vs 全部（含失败）**21 步/$2.52** [来源: arXiv 2405.15793] ✅
- **长尾**：Devin **72%** 成功任务耗时 **>10 分钟** [来源: Cognition SWE-bench Technical Report] ⚠️ 数量级估算（vendor self-report）
- **依赖复杂**：**55%** 文件编辑后立即跟随终端调用（Edit-Verify 模式）[来源: PASTE, arXiv 2603.18897, §2.3.1] ✅
- 传统调度假设（单一计算/可预测时长/独立任务）**全部被打破** → 为 P2 多 Agent 协同铺路

<!-- 为什么是"新"workload？四特征：混合（工具占 35%-61%）；突发（成功 12 步 vs 全部 21 步，失败近 2 倍=fail-slow）；长尾（Devin 72% 任务 >10 分钟）；依赖复杂（55% 编辑后立即跟终端调用，编辑-验证循环极频繁）。这打破传统调度器所有假设——为下一部分多 Agent 协同铺路。 -->

---

## 诊断问题：从 1 个 agent 到 N 个 agent，成本结构怎么变？

- 单 Agent 瓶颈在 loop **内部**效率；多 Agent 瓶颈变成 **多个 loop 之间的协调开销——即"通信"**
- 新变量："通信" = Agent 间消息传递、状态同步、角色调度、共识、负载均衡
- **本章诊断论断：通信占比随规模上升、逐渐压过计算成为主导成本**
- 三类证据：学术基准（UIUC / Continuum）+ 框架拓扑（AutoGen/MetaGPT/CrewAI/LangGraph）+ 协议趋势（Google A2A）

| 指标 | 数值 | 口径 |
|---|---|---|
| 单 Agent 工具执行占 agent 总请求时间 | **35%-61%**（编码 60%）| ✅ [来源: PASTE, arXiv 2603.18897] |
| 多 Agent 比 Agent token/成本 | ~**2-5×** | ⚠️ 经验法则 [来源: 综合 UIUC arXiv:2505.18286] |
| Continuum 调度气泡 | ~**58%** | ⚠️ 调度气泡/任务间隙 [来源: Continuum, arXiv:2511.02230] |

<!-- 第二部分开篇。先回顾单 Agent 瓶颈在 loop 内部效率（工具占 35%-61%），再抛诊断问题：从 1 个 agent 到 N 个 agent，成本结构怎么变？多 Agent 引入单 Agent 没有的新变量——多个 loop 之间的协调开销，即"通信"。本章论证：通信占比随规模上升、渐成主导。有学术基准、框架拓扑、协议趋势三类证据。先从现状盘点开始。-->

---

## A2A 生态分三层：框架拓扑各异、A2A 协议标准化、学术基准量化成本

- **框架拓扑各自为政** [来源: AutoGen/MetaGPT/CrewAI/LangGraph 官方文档] ✅
  - AutoGen：广播 / GroupChat pub-sub + GroupChatManager 选下一发言者
  - MetaGPT：全局共享消息池 + **5 个顺序角色**（PM → Architect → PM → Engineer → QA）[arXiv:2308.00352, ICLR 2024] ✅
  - CrewAI：层级任务委托；LangGraph：有状态有向图 + checkpointer
- **Google A2A 协议**：**2025-04-09** 宣布、**50+** 合作伙伴、HTTP + JSON-RPC 2.0 + SSE [来源: Google Developers Blog] ✅
  - Agent Card 在 `/.well-known/agent-card.json`；Task ~**8 个**状态；方法 **SendMessage/SendStreamingMessage/GetTask**
- **🎯 CPU 团队 load-bearing 洞察**：A2A 是纯 Web 传输——CPU 处理 **TLS/JSON**，无需特殊硬件
- 真正痛点在**服务层 prefill 膨胀** + **缓存重调度气泡** [来源: UIUC arXiv:2505.18286 + Continuum arXiv:2511.02230] ✅
- UIUC：多 agent 消耗 **4×-220×** 更多 prefill token（⚠️ 数量级；排除 AIME 后典型 ~5-35×）[来源: arXiv:2505.18286] ✅

<!-- 现状盘点，建立 A2A 共同认知。三层：框架拓扑各异；Google A2A 协议 2025-04 发布、50+ 合作伙伴、纯 HTTP+JSON-RPC+SSE；学术基准量化成本（UIUC 测得 4-220× prefill）。🎯 关键洞察：A2A 是纯 Web 传输，CPU 处理 TLS 和 JSON，不需要特殊硬件。真正痛点在服务层 prefill 膨胀和缓存重调度气泡——正是我们的主场。下一页看具体数据。-->

---

## 这一页的任务：把三个数字的口径标清楚

- **~2-5× token/成本增量**（⚠️ 经验法则/数量级估算，指 token-or-cost **而非 total-time**，受任务/AI 数/轮数影响）[来源: 综合 UIUC arXiv:2505.18286]
  - 拓扑特定印证（⚠️ 绝非通用乘数）：中心化 supervisor-worker ≈ **+285%**（≈3.85× 总量）；去中心化 +263%、混合 +515% [来源: arXiv:2512.08296 / Medium] ⚠️ 数量级估算
- **~58% 调度气泡**（⚠️ 调度气泡/任务间隙，**非工具调用时间**——工具返回后等 KV-cache 重驻留的空闲）[来源: Continuum, arXiv:2511.02230] ✅
- **木桶效应**：最慢 agent 决定端到端延迟——放大 fail-slow（SWE-agent 成功 **12 步/$1.21** vs 全部 **21 步/$2.52**）[来源: SWE-agent, arXiv 2405.15793] ✅
- 主因：**通信与协调开销**（序列化、同步、负载不均），非 agent 逻辑本身变贵

<!-- 整页任务是标口径（数据校验报告明确警告过）。第一，多 Agent 比单 Agent 贵约 2-5 倍——这是 token/成本经验法则，不是总时间测量；中心化 supervisor-worker 约 +285%，但只是拓扑特定，绝不通用。第二，58% 是调度气泡——工具返回后等 KV-cache 重驻留的空闲间隙，不是工具执行时间。第三，木桶效应：最慢 agent 决定整体延迟。主因都是通信与协调开销。-->

---

## A2A 数据怎么流：四种模式 + 序列化税 + O(N²) 爆炸

- **四种通信模式**：点对点(1-1) / 广播(1-N，如 AutoGen) / 层级编排(树状，CrewAI) / 共享黑板(N-1 共享状态，MetaGPT)
- **序列化是隐藏的税**：
  - JSON 比 Protobuf 慢约 **5-10×**（⚠️ 数量级估算，依赖语言/运行时/数据形态，严谨基准多 3-6×）[来源: Zuplo/inomera, 2023-2024] ⚠️
  - JSON 体积约 Protobuf 的 **6×**（FlatBuffers：1475B vs 228B vs **FlatBuffers 344B**）[来源: FlatBuffers 官方 Benchmarks] ⚠️
  - FlatBuffers 零拷贝：100 万次 Decode+Traverse+Dealloc **0.08s** vs Protobuf LITE 302s vs RapidJSON 583s [来源: FlatBuffers] ✅
  - simdjson 单核 **6 GB/s**（Minify）、UTF-8 校验 **13 GB/s**，比 RapidJSON 快 **4×** 以上 [来源: arXiv:1902.08318] ✅
- **O(N²) 通道爆炸**（全连接 mesh）：**10 agents=45 条**、**100 agents=4950 条** ✅
- Agent 消息：**频繁、小 payload、低延迟敏感** → per-message 开销被高频放大

<!-- 看数据怎么流。四种模式：点对点、广播、层级编排、共享黑板。重点是隐藏的税——序列化：JSON 比 Protobuf 慢约 5-10 倍、体积大 6 倍；FlatBuffers 零拷贝 100 万次解码 0.08s，JSON 解析器要 583s；simdjson 把 JSON 拉到 GB/s 级。第二成本点 O(N²)：10 agent 45 条通道，100 agent 4950 条。第三成本点：消息频繁、小 payload、低延迟敏感，per-message 开销被高频放大。通信成本藏在这三处。-->

---

## 被锁与不均吃掉的延迟：锁竞争 + 共识墙 + 负载不均

- **锁竞争退化 10-100×**（✅ 已确认；高竞争下临界区 = Amdahl 串行部分）[来源: ACCU Overload 2020 / EMSE 2017] ✅
  - 5-10 agents 频繁写约 **10-30%** 时间等锁；10+ agents 高并发写 **50%+** 等锁（⚠️ 数量级估算，随工作负载）
- **分布式共识延迟墙**（⚠️ 数量级估算，随拓扑/存储介质）：
  - etcd (Raft) LAN 单次提交约 **1-10ms**（fdatasync：HDD ~10ms、SSD <1ms）[来源: etcd 官方, 2017] ✅
  - Raft/Paxos 10 节点典型延迟 **10-50ms** 量级 [来源: 团队早期版本] ⚠️
- **负载不均（木桶效应）**：编译 agent CPU 100% vs 产品 agent 10% → 端到端被编译 agent 决定；MetaGPT 1 agent $0.915 → 4 agents $1.385 [来源: MetaGPT ICLR 2024 Table 3] ✅
- **推荐方向**：最终一致性 + 无状态消息传递延迟仅 **1-5μs**（⚠️ 数量级估算）[来源: 团队早期版本] ⚠️

<!-- 通信成本第二处：锁和负载不均。第一，锁竞争：高竞争退化 10-100 倍——已确认；5-10 个 agent 频繁写 10-30% 等锁，10 个以上 50%+ 等锁。第二，共识延迟墙：etcd Raft 单次提交 1-10ms，10 节点 10-50ms 量级。第三，负载不均：编译 agent 100% CPU、产品 agent 10%，端到端被编译 agent 决定。解药：Agent 通常可容忍最终一致性，无状态消息传递延迟仅 1-5μs，无锁队列吞吐高 75%-150%。这些都指向 MPI/UCX 无锁思想——P3 通信侧靶点。-->

---

## 🎯 诊断落地：通信占比随规模上升、渐成主导成本

**单 agent → 小 swarm → 大 swarm，通信从配角变成主角**

| 规模 | 通信表现 | 数据 |
|---|---|---|
| 单 agent | 占比小，瓶颈在 loop 内部 | 工具执行 **35%-61%** ✅ |
| 小 swarm | 通信上升，序列化税/O(N²) 显现 | 成本增量 ~**2-5×** ⚠️ 经验法则 |
| 大 swarm | **通信占比主导** | 气泡 ~**58%** / 锁 **10-100×** / 共识 **10-50ms** |

- 四组独立证据方向一致，且都随规模放大——**通信是上升中的主导成本**
- 优化靶点从"agent 逻辑/模型"精确移到"通信/系统层"
- 优化通信 = 优化 agent 运行时成本结构的**主杠杆**——GEMM/Attention + MPI/UCX/SDMA + SIMD 的经典领域

<!-- 🎯 诊断落地，请在规模演进图上停一下。单 agent 通信占比小，瓶颈在 loop 内部。小 swarm 通信上升——序列化税、O(N²)、2-5 倍成本增量。大 swarm 通信占比主导——58% 气泡、10-100 倍锁竞争、毫秒级共识墙全面爆发。四组独立证据方向完全一致：从单 Agent 到多 Agent，通信占比随规模上升、压过计算成主导成本。既然通信是主导，优化通信就是优化 agent 成本结构的主杠杆——正是我们的主场。下一页把问题抛给 P3。-->

---

## 🎯 通信变贵了，谁来付？（handoff——只问不答）

> **"如果通信是上升中的主导成本，而我们 ARM 平台（NEON 鲲鹏 920 现役 + SVE/SME 新一代鲲鹏 early-access 现已可得）能把通信和计算放进同一组 SIMD 单元……"**

- **传统架构**：comm 的"数据搬运/序列化" 与 compute 的"向量化/矩阵化" = **两套单元、两次拷贝**，边界开销无法消除
- **融合架构**：序列化/pack/gather/转置本身就是 SIMD/矩阵操作——**同一组单元、同一份布局、无边界拷贝**
- ARM 物理基础：**NEON（鲲鹏 920 现役）+ SVE/SVE2/SME（新一代鲲鹏 early-access，⚠️ 暂无公开来源）** [来源: docs/superpowers/research/2026-06-09-platform-sme-sve.md] ✅（920 ISA 已校验）/ ⚠️（新一代鲲鹏 early-access，无公开来源）
  - NEON 现在就能做基础融合；SVE predicate + gather/scatter；SME 矩阵引擎
- 在 ARM 平台上，"通信计算融合"不只是潜在机会，而是**有论证的主场答案**——但论证由 P3 给出

<!-- 🎯 主线第二次点题——只问不答，把张力交给 P3。诊断是通信占比随规模上升、渐成主导。那么问题：通信变贵了，谁来付？怎么付？传统架构里通信的数据搬运/序列化和计算的向量化/矩阵化是两套单元、两次拷贝，边界开销无法消除。融合架构把它们放进同一组 NEON/SVE/SME 单元、同一份数据布局。我们 ARM 平台 NEON（鲲鹏 920 现役）今天就能做基础融合，SVE/SME（新一代鲲鹏 early-access）现已可得，团队可在自有硅上实测。这个答案由第三部分给出。让我们进入第三部分。-->

---

## 通信计算融合 = 主场（handoff 落地）

承接 P2-7 的张力——"通信变贵了，谁来付？"——答案是**融合**。

- 不再把通信与计算当成两个独立问题、两套单元、两次拷贝
- 合并成**同一类操作、同一组单元、同一份布局**
- **现役 NEON 已能做基础融合**（向量化序列化 / pack / 基础 SIMD）— 鲲鹏 920 立即可做
- **SVE predicate+gather/scatter 与 SME 矩阵引擎**在新一代鲲鹏 early-access 现已可得（⚠️ 暂无公开来源）；NEON 现役与 SVE/SME 是同一融合方向的现在与未来
- 前后是**同一融合主线**的现在与未来，不是另起炉灶

> [来源: docs/superpowers/research/2026-06-09-platform-sme-sve.md] ✅（平台 ISA 状态，已校验）

<!--
本页是 P3 开场：回答 P2-7"谁来付"——融合。NEON（鲲鹏 920 现役）今天就能做基础融合；SVE/SME 在新一代鲲鹏 early-access 现已可得，团队可在自有硅上实测。叙事弧：武器 → 融合论点 → 算子侧 → 通信侧 → 融合点 → 路线图 → 总结。注意口径：NEON（鲲鹏 920 现役）/ SVE-SME（新一代鲲鹏 early-access，⚠️ 暂无公开来源），永不混。
-->

---

## P3-2 三类武器：NEON / SVE / SME — 为什么对 Agent 特别

Agent 负载是**变长 / 不规则 / 小批量**——三类单元的结构性匹配。

| 单元 | 状态 | 对 Agent 的匹配 |
|------|------|----------------|
| **NEON** | 🟢 **现役**（鲲鹏 920） | 固定 **128-bit** SIMD；无 per-lane 谓词、无原生 gather/scatter |
| **SVE / SVE2** | 🟡 **新一代鲲鹏 early-access 已可得**（⚠️ 暂无公开来源） | VLA **128-2048 bit** + **predicate** + 原生 **gather/scatter** |
| **SME / SME2** | 🟡 **新一代鲲鹏 early-access 已可得**（⚠️ 暂无公开来源） | **ZA tile + outer-product** 矩阵引擎，核心指令 **MOPA** |

- **变长** → SVE **predicate**（逐 lane）；NEON 需软件处理边界
- **不规则访存** → SVE **原生 gather/scatter**；NEON 无此指令
- **小批量**（batch **1-8** vs 训练 **32-1024**）→ SME **outer-product + ZA tile**

⚠️ 新一代鲲鹏 SVE2/SME 团队 early-access 已可得（⚠️ 暂无公开来源）；鲲鹏用自研 TaiShan 核（非 Neoverse）；他平台数字（A64FX/Graviton/LX2/M4 等）需在本团队 SVE/SME 硅上 re-measure 后引用

<!--
三类武器：NEON 现役 128-bit 整向量、无谓词无 gather-scatter；SVE 新一代鲲鹏 early-access 已可得 predicate+gather-scatter+VLA；SME early-access 已可得 ZA tile+outer-product+MOPA。对 Agent 特别的原因：负载变长/不规则/小批量，正好匹配三特性。强调新一代鲲鹏 SVE2/SME 团队已可得但暂无公开来源；鲲鹏非 Neoverse；他平台数字需在本团队硅 re-measure。

⚠️ 口径脚注（来自来源：ARM ARM / IEEE Micro 2021 / Stephens et al. arXiv:1803.06185 / DATE 2026 KirbyMM，✅）：新一代鲲鹏 SVE2/SME 团队 early-access 已可得（⚠️ 暂无公开来源）；鲲鹏用自研 TaiShan 核（非 Arm Neoverse）；他平台数字需在本团队硅 re-measure。
-->

---

## 🎯 P3-3 通信计算融合 = 三件事（thesis 展开）

融合不是一句口号，是**三件具体的事**——今天 NEON（鲲鹏 920 现役）就能立论，SVE/SME（新一代鲲鹏 early-access）现已可得。

**① 重叠** — 通信期间做计算（延迟隐藏）
- 经典 HPC：MPI 非阻塞 **Isend/Irecv**，计算与传输并发 [来源: Bernholdt 等, OSTI 944757, 2008] ✅
- API 等待 **500ms-5s+** 期间本地 CPU 不必空闲，可 SIMD 做向量检索/rerank/模板渲染
- ⚠️ 重叠不免费：GPU 实测计算减速**平均 ~18.9%（最高 40%）**；纯顺序更慢**平均 10.2%**（⚠️ GPU 特定，CPU 仅定性类比）[来源: Georgia Tech, ISPASS 2025, arXiv:2507.03114] ✅

**② 数据搬运即计算** — 序列化/pack/转置本身就是向量化 kernel
- NEON 现役：simdjson 式 **GB/s** 解析（**今天就能做**）[来源: arXiv:1902.08318] ✅
- SVE（A64FX 512-bit）：pack/unpack 比非-gather 基线快 **2×**；local reduce 比逐元素快 **4×**（✅ A64FX 特定，非鲲鹏）[来源: IEEE CCGrid 2020] ✅
- ⚠️ 高度分散 gather/scatter **"相当慢"**——访存连续性才是决定性因素 [来源: ACM Lattice QCD on A64FX] ✅

<!--
🎯 thesis 展开（第 1/2 页）。三件事的前两件：重叠（通信期间做计算，HPC 经典，GPU 上重叠减速约 19% 但净为正，CPU 定性类比）；数据搬运即计算（NEON simdjson GB/s，SVE pack 快 2×、reduce 快 4× 都是 A64FX；分散 gather-scatter 反而慢，连续性才是决定因素）。NEON（鲲鹏 920 现役）今天就能立论，SVE/SME（新一代鲲鹏 early-access）现已可得。第三件事 kernel+原语协同翻页。
-->

---

## P3-3 ③ kernel + 原语协同：同一组单元、同一份布局（→ 融合点）

**③ kernel + 原语协同** — comm 原语与算子**共享同一组 SIMD 单元 + 同一份布局**，消除边界拷贝；iceoryx2 零拷贝 IPC = "不搬运载荷"的极端端点（⚠️ sub-µs、与 payload 无关；**勿暗示快于 raw shmem**）[来源: iceoryx2 Discussion #435] ⚠️

<!--
🎯 thesis 展开（第 2/2 页）。第三件事 kernel+原语协同：同一组单元同一份布局，消除边界拷贝。iceoryx2 是零拷贝极端端点但勿暗示快于 raw shmem。这一页桥接进 climax（P3-6 融合点）。
-->

---

## P3-4 算子侧 · NEON 现役：专用小-GEMM kernel

Agent 算子（embedding / rerank / 小批量 GEMM / attention）是**"瘦高"矩阵**，batch **1-8** vs 训练 **32-1024**——结构性不同于大方阵批量 GEMM [来源: 承袭 P3-4 校验] ⚠️。

**NEON 现役 — 基础向量化 + 专用小-GEMM kernel**
- **LibXSMM**（SC15）：小 GEMM 加速 **>10×（可达/上限）**，典型 MKL 对比 **2-4×**（"取决于 CPU 与应用"，无鲲鹏 920 专用数字）[来源: SC15] ✅
- **LibShaLom**（SC'21）：**target ARMv8 多核含鲲鹏 920**；小 M 时 packing 约占运行时 **~50%**，运行时选择性 packing + 微内核内 packing 与 FMA 重叠（mr=7, nr=12）[来源: LibShaLom SC'21] ✅（agent 类比⚠️/方向性投影）

<!--
算子侧（第 1/2 页，NEON 现役）。Agent 算子是瘦高矩阵 batch 1-8。NEON 现役机会在专用小-GEMM kernel：LibXSMM >10× 是上限、典型 2-4×，LibShaLom 在 ARMv8 含鲲鹏 920 靠降低 packing 开销取胜（小 M 占约一半）。SME 在新一代鲲鹏 early-access 已可得翻页。
-->

---

## P3-4 算子侧 · SME early-access：outer-product + ZA tile（⚠️ 非万能药）

**SME（新一代鲲鹏 early-access，⚠️ 暂无公开来源）— outer-product + ZA tile（⚠️ 不是万能药，需混合策略）**
- **KirbyMM**（DATE 2026）：SME MOPA **峰值 0.512 TFLOPS FP32/核**（稠密 GEMM 实测仅约 0.33）vs SVE MLA 0.128 = **4×**；但稠密 GEMM 因此**内存带宽受限**
- KirbyMM 边缘 case：专用 SME+SVE 混合平均比 KBLAS 快 **3.59×**（LX2 ARMv9/SME，5 个瘦 case FP32，**非当前 920**）[来源: DATE 2026] ✅
- **MpGEMM**：SME INT8 外积指令是 **SMOPA**（整数外积，**非浮点助记符**）；Apple M4 Pro 峰值约 **4010 GOPS** vs 其他精度约 2006（约 2×）（⚠️ "SME 家族行为，需目标硅实测验证"）[来源: arXiv:2512.21473] ✅

> **FlashDecoding++**（MLSys 2024）：GPU 瘦 GEMM 补零约 **50%** 性能损失（⚠️ GPU 特定，CPU 仅定性类比，勿当 NEON-SME 实测）[来源: MLSys 2024] ✅

<!--
算子侧（第 2/2 页，SME early-access）。SME 不是万能药：峰值 0.512 TFLOPS 是 4× 但稠密 GEMM 内存受限，瘦形状需 SME+SVE 混合，KirbyMM 平均比 KBLAS 快 3.59×（LX2 非当前 920，需在本团队新一代鲲鹏硅上 re-measure）。INT8 指令是 SMOPA 整数外积。FlashDecoding++ 的 50% 是 GPU 数据 CPU 仅定性。
-->

---

## P3-5 通信侧 · 序列化加速：NEON 现役 simdjson 式 GB/s 解析

A2A / 工具调用流量**绝大多数是 JSON 形状**——序列化/验证数据通路是现实瓶颈，远早于 LLM 推理成本。

**序列化加速 — NEON 现役 simdjson 式 GB/s 解析**
- simdjson：首个单商用核 **GB/s 级**完全校验 JSON 解析器；指令数仅 RapidJSON 的 **1/4 或更少**；约 **4×** 快于 RapidJSON、约 **25×** 快于 nlohmann/json [来源: Langdale & Lemire, VLDB Journal 28(6) 2019, arXiv:1902.08318] ✅
- ARM 服务器核实测（单核）：Graviton 3（Neoverse V1）约 **3.6 GB/s**、Graviton 4（Neoverse V2）约 **4.6 GB/s**（⚠️ 用"Arm V 系列服务器核"，**勿暗示鲲鹏 = V2**）[来源: Lemire 博客] ✅
- SVE2 **SVMATCH**：Neoverse-N2 2 cycle/指令；Sonic JSON 吞吐提升**最高约 24%**（范围 **-0.3%~24.3%**，⚠️ 微基准、高度依赖语料，勿暗示均匀）[来源: Arm blog, 2024-09] ✅
- ⚠️ **不存在 SVE 版 JSON 解析器公开 GB/s 数字**（simdjson issue #2683 开放）— **不为 SVE JSON 附任何吞吐数字** [来源: P3-5 校验] ✅

<!--
通信侧（第 1/2 页，序列化加速）。JSON 是现实瓶颈。NEON（鲲鹏 920 现役）simdjson 式 GB/s 解析——首个单核 GB/s 校验解析器，4× 于 RapidJSON、25× 于 nlohmann。Arm 服务器核 Graviton 4 单核约 4.6 GB/s，但用 V 系列服务器核，勿暗示鲲鹏等于 V2。SVE2 SVMATCH 最高约 24% 但范围大、微基准。无 SVE JSON 公开 GB/s 数字。零拷贝 IPC + 消息搬运翻页。口径：NEON（鲲鹏 920 现役）做 simdjson，SVE/SME（新一代鲲鹏 early-access）是增量。
-->

---

## P3-5 通信侧 · 消息搬运：零拷贝 IPC + AoS→SoA 转置

**零拷贝 IPC — iceoryx2（⚠️ 数量级，定性）**
- 常数级 **sub-µs**，与 payload 大小无关 [来源: iceoryx2 Discussion #435] ⚠️
- ⚠️ **勿暗示快于 raw shmem**（同来源：单字节反而慢约 600ns）

**消息搬运 — NEON AoS→SoA 转置（鲲鹏 920 现役）/ SVE gather-scatter（新一代鲲鹏 early-access）**
- **NEON 无 gather/scatter**（仅 LD1-LD4 连续/交错）；非连续 AoS 字段需 **vzip/vuzp/vtrn** 寄存器级转置 [来源: ARM ARM] ✅
- gather/scatter 性能**远劣于 unit-stride**（⚠️ 定性，无精确乘数）[来源: ryg blog / Intel BKM] ✅；SVE 原生 gather/scatter 团队 early-access 已可得（⚠️ 暂无公开来源），需在本团队新一代鲲鹏硅上 re-measure

<!--
通信侧（第 2/2 页，消息搬运）。iceoryx2 sub-µs 与 payload 无关但勿暗示快于 raw shmem（单字节反而慢约 600ns）。NEON 无 gather-scatter，用 vzip-vuzp-vtrn 转置；gather-scatter 远劣于连续加载，SVE 原生 gather-scatter 团队 early-access 已可得（暂无公开来源），需在本团队硅 re-measure。口径：NEON（鲲鹏 920 现役）做转置，SVE/SME（新一代鲲鹏 early-access）是增量。
-->

---

## 🎯 P3-6 融合点：同一组单元、同一份布局（prescription climax）

**传统架构 vs 融合架构**——这是整场主线第三次点题、climax。

| | 传统架构 | 融合架构 |
|---|----------|----------|
| **单元** | 两套：comm 栈（HTTP/JSON/拷贝）+ compute 栈（向量化/矩阵） | **同一组**（NEON 鲲鹏 920 现役） |
| **布局** | 两份独立 → 边界 AoS↔SoA 转置拷贝 | **同一份**（SoA + 连续 `ld1/st1`） |
| **边界** | 拷贝/序列化/布局转换**结构性存在、无法消除** | **无边界拷贝** |
| **依赖** | — | **去 SME 硬依赖**：融合论点**不依赖 SME** |

- 序列化 / pack / gather / 转置**本身就是 SIMD 操作**——comm 数据搬运 = compute 向量化操作
- **NEON（鲲鹏 920 现役）今天就能做**（向量化序列化 = 128-bit SIMD，向量化 GEMM 也是 128-bit SIMD）——融合**已经在今天发生**
- **SVE/SME 是同一方向在自有硅上的延续**（新一代鲲鹏 early-access 已可得，更宽向量、predicate、矩阵引擎，⚠️ 暂无公开来源），不是另起炉灶

> 🎯 **主线三次点题**：P1-4（loop = compute↔comm 交织）→ P2-7（通信主导，谁来付）→ **本页 climax：同一组单元、同一份布局**。
>
> **通信计算融合，就是我们在 Agent 时代的主场。**

<!--
🎯 prescription climax，必须落得硬。传统架构两套单元两次拷贝，边界开销结构性无法消除。融合架构同一组单元（NEON 鲲鹏 920 现役）、同一份布局（SoA + 连续 ld1-st1）、无边界拷贝。关键：去 SME 硬依赖——融合论点不依赖 SME，今天 NEON 上序列化就是 SIMD 操作，融合已经在发生；SVE/SME（新一代鲲鹏 early-access）是同一方向在自有硅上的延续。主线三次点题在此 climax：同一组单元、同一份布局。落点一句：通信计算融合就是我们在 Agent 时代的主场。
-->

---

## P3-7 怎么开始：三层机会 + 路线图

时间线明确，**短期 NEON（鲲鹏 920 现役部署）立即可做 + SVE/SME（新一代鲲鹏 early-access）实测启动，无需等任何未来平台**。

| 阶段 | 时间 | 平台 / 口径 | 动作 |
|------|------|------------|------|
| **短期** | 现在–6 月 | 🟢 NEON 现役（鲲鹏 920 立即）+ 🟡 SVE/SME（新一代鲲鹏 early-access 实测启动，⚠️ 暂无公开来源） | NEON：simdjson 式解析 + iceoryx2 零拷贝 IPC 原型 + vzip/vuzp/vtrn pack + 小-GEMM kernel；SVE/SME 实测：SVE gather/scatter 序列化 + SME outer-product micro-bench + 小-GEMM kernel re-measure |
| **中期** | 6–18 月 | 🟡 SVE/SME 生产化/库化（新一代鲲鹏 early-access，⚠️ 暂无公开来源） | SVE predicate+gather/scatter 序列化 + SME 小批量 Agent kernel 库（KirbyMM 风格混合）+ Agent-native 通信原语标准化 |
| **长期** | 18–36 月 | — | Agent Workload Benchmark + CPU-Agent 协同设计反哺 ARM |

**能力映射**（三类能力全映射上，无需另起炉灶）
- **GEMM/Attention** → 小批量 Agent kernel（NEON 小-GEMM + SME 混合）
- **MPI/UCX/SDMA** → Agent-native 通信原语（零拷贝 IPC + 向量化序列化）
- **性能方法论** → Agent Workload Benchmark + 反馈 ARM 闭环

> ⚠️ 新一代鲲鹏 SVE2/SME **团队 early-access 已可得，但暂无公开来源**；他平台数字（A64FX/Graviton/LX2/M4 等）需在本团队新一代鲲鹏硅上 re-measure 后引用 [来源: docs/superpowers/research/2026-06-09-platform-sme-sve.md §1] ⚠️

<!--
怎么开始：三层机会。短期 NEON（鲲鹏 920 现役部署）立即可做 + SVE/SME（新一代鲲鹏 early-access）实测启动，不用等未来平台——NEON 上 simdjson 解析、iceoryx2 零拷贝 IPC 原型、基础 pack 转置、小-GEMM kernel；SVE/SME 上做 simdjson 解析、iceoryx2 零拷贝、小-GEMM kernel、SVE gather/scatter 序列化、SME outer-product micro-bench 实测。中期 6-18 月 SVE/SME 生产化/库化——SVE 序列化、SME 小批量 kernel 库、通信原语标准化；新一代鲲鹏 early-access 已可得但暂无公开来源。长期 18-36 月 Agent Workload Benchmark + 反哺 ARM。三类团队能力全部映射，无需另起炉灶。
-->

---

## 总结 + 讨论问题（收束）

整场一条主线：**P1** loop = compute↔comm 交织 → **P2** 通信随规模上升、渐成主导 → **P3** 在 ARM 平台把 comm 与 compute 融合 = **主场**。

**核心收束 — 通信计算融合就是主场**
- **NEON（鲲鹏 920 现役）今天就能做**：simdjson 式 GB/s、vzip/vuzp/vtrn pack、小-GEMM kernel（LibXSMM/LibShaLom 风格）、iceoryx2 零拷贝 IPC——鲲鹏 920 立即可做
- **SVE/SME 是同一融合方向的延续**（新一代鲲鹏 early-access 已可得，⚠️ 暂无公开来源，他平台数字需在本团队硅 re-measure）
- **团队能力高度匹配**：GEMM/Attention + MPI/UCX/SDMA + 性能方法论

**留给团队的三个讨论问题**（= 三个可立即启动的调研任务，建议分组攻关）
1. **通信侧**：Agent 专用通信库接口长什么样？复用 MPI/UCX 原语还是全新设计？
2. **算子侧**：Agent 算子与训练/推理算子的边界在哪？哪些复用 GEMM/Attention 经验，哪些打破重来？（"瘦高"小批量 vs 训练大方阵 GEMM）
3. **工具链侧**：现有剖析工具分析 Agent 工作流缺什么？我们准备好 Agent-native 了吗？

> 🎯 **主线三次点题**：P1-4（loop = compute↔comm 交织）→ P2-7（通信主导，谁来付）→ P3-6（同一组单元、同一份布局，climax）。**通信计算融合，就是我们在 Agent 时代的主场。**

<!--
收束。一条主线串三部分。核心收束：通信计算融合就是主场——NEON（鲲鹏 920 现役）今天就能做；SVE/SME（新一代鲲鹏 early-access）已可得，团队可在自有硅上实测，无需等未来平台。三个讨论问题对应三个可立即启动的调研任务，建议分组攻关：通信侧接口、算子侧边界、工具链侧缺口。短期即可启动 NEON simdjson 解析、iceoryx2 原型、小-GEMM micro-benchmark 三条线，并在新一代鲲鹏 early-access 硅上并行启动 SVE/SME 实测，为中期生产化/库化铺路。谢谢大家，欢迎讨论。
-->
