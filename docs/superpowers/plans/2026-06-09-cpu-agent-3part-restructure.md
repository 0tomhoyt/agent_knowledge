# CPU Agent-Era Deck: Three-Part Restructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Produce the restructured three-part deck — a new outline (`…-3part-ppt-outline.md`) and a new detailed doc (`…-3part-ppt-detailed.md`) — with all new content (A2A landscape, ARM SME/SVE compute-comm fusion) researched and adversarially verified per the repo's citation-rigor convention.

**Architecture:** Documentation-only; **no code, no tests**. "Implementation" = (1) research new content via a multi-agent Workflow + adversarial verification, (2) carry over verified content from the old three docs, (3) draft a 21-page outline skeleton, (4) draft a per-page detailed doc (core claim / detailed points / cited data / speaker notes), (5) append an extended verification report. Verification is adapted to docs: citation-rigor (every number has `[来源: …]` + 确认✅/数量级⚠️ classification), structural (page count / time / spine touchpoints), and 口径 (carried-over estimates keep 约/数量级 qualifiers; R1 SME-availability pivot applied consistently).

**Tech Stack:** Markdown; the Workflow tool for parallel research + adversarial verification; `grep` heuristics for verification; git for frequent commits. All content in Chinese (technical terms in English), matching repo convention.

**Driving spec:** `docs/superpowers/specs/2026-06-09-cpu-agent-3part-restructure-design.md` (read it first — every page table below is defined there).

**Carry-over sources (read-only):**
- `2026-06-09-cpu-agent-era-direction-ppt-detailed.md` —（**已于 2026-06-10 合并移除**，已验证数据全部并入 `…-3part-ppt-detailed.md`；git 历史可恢复）原 primary verified-data source (SWE-bench, SWE-agent, simdjson, JSON/Protobuf/FlatBuffers, 锁竞争 10-100×, 共识延迟, latency/speedup appendices, 数据校验报告)
- `2026-06-09-cpu-agent-era-direction-ppt-outline.md`, `docs/superpowers/specs/2026-06-09-cpu-agent-era-direction-design.md` — secondary（**同样已于 2026-06-10 合并移除**）

**Unified spine (must appear and stay consistent across all tasks):**
> Agent 运行时 = 计算↔通信反复交织的长循环；通信占比随规模上升、渐成主导；ARM SME/SVE 是通信计算融合的物理基础 = 我们的主场。Spine 触点页：P1-4（第一次点题）、P2-6/P2-7（diagnosis + handoff）、P3-3/P3-6（thesis 展开 + prescription climax）。

---

## File Structure

| File | Responsibility | Created in |
|------|----------------|------------|
| `docs/superpowers/research/2026-06-09-platform-sme-sve.md` | R1 decision + P3-2 research: ARM platform SME/SVE availability + capability, with citations + classification | Task 1 |
| `docs/superpowers/research/2026-06-09-3part-research-findings.md` | Consolidated cited findings for P2-2, P3-3, P3-4, P3-5 (from the research Workflow) | Task 2 |
| `2026-06-09-cpu-agent-3part-ppt-outline.md` | 21-page deck skeleton (title + 要点 + 主线触点 + 来源 per page) | Task 3 |
| `2026-06-09-cpu-agent-3part-ppt-detailed.md` | Full per-page deck: header → 开场+P1 → P2 → P3+总结 → appendix + 数据校验报告 | Tasks 4–7 |

**Detailed-doc page format (follow the old detailed doc exactly):**
```
========== PAGE N: <title> ==========
## 核心观点（一句话）
...
## 详细要点
### 要点1: ...
...
## 关键数据速览
... (every number followed by [来源: ...])
## 演讲备注 / 过渡语
...
```

**Page numbering:** 开场=PAGE 1; P1=PAGE 2–6; P2=PAGE 7–13; P3=PAGE 14–20; 总结+讨论=PAGE 21; 附录=PAGE 附录1/附录2.

---

## Task 1: Resolve R1 + research P3-2 (ARM platform SME/SVE)

**Files:**
- Create: `docs/superpowers/research/2026-06-09-platform-sme-sve.md`
- Modify: `docs/superpowers/specs/2026-06-09-cpu-agent-3part-restructure-design.md` (R1 resolution line)

**Why first:** Spec R1 is the top blocker — the team's exact ARM chip's SME availability determines P3's whole 口径 (have-SME vs SVE-first pivot). This is also the P3-2 page content. Must resolve before drafting P3 (Task 6).

- [ ] **Step 1: Research platform SME/SVE availability**

Use WebSearch/WebFetch on public spec sheets for the candidate ARM server CPUs (the team is "ARM 为主"; if the user later names the exact chip, narrow to it): 阿里倚天 710/810, 华为鲲鹏 920/930, 飞腾 S2500/腾锐, AWS Graviton 3/4, Arm Neoverse N2/V2/V3/N3/V3AE. For each: ISA base (ARMv8.x/ARMv9), **SVE vector width (128/256-bit)**, **FEAT_SME / FEAT_SME2 present?**, FEAT_SVE2, streaming mode. Cite ARM Architecture Reference Manual / vendor product pages / Wikichip.

- [ ] **Step 2: Determine the team's exact platform / 口径 decision**

If the exact chip is still unspecified after the spec, surface it via AskUserQuestion ("你们主力是哪款 ARM 服务器 CPU？") — the answer picks the row from Step 1. Then decide the **P3 口径**:
  - **have-SME**: platform has FEAT_SME → P3-4 written as "已具备的算子机会"; P3-6 punchline uses "SME+SVE 同一组单元".
  - **SVE-first pivot**: platform has SVE/SVE2 but NOT SME yet → P3 thesis pivots to "SVE-first 融合 + SME 路线图"; P3-6 punchline restated as "同一组向量单元、同一份布局" (no SME dependency); P3-4 framed as "前瞻/即将具备".

- [ ] **Step 3: Write the P3-2 research doc**

Create `docs/superpowers/research/2026-06-09-platform-sme-sve.md` containing: (a) platform table (chip → SVE width / SME / SVE2, each `[来源: …]`), (b) the 口径 decision + one-line rationale, (c) P3-2 page content draft — "SME/SVE 是什么、为什么对 agent 特别" (SVE: 向量·predicate·gather/scatter·向量长度可扩展; SME: 2D tile·outer-product 矩阵引擎; why 变长/不规则/小批量 matches, vs x86 AVX-512 fixed width) — every claim `[来源: ARM ARM / vendor]` + 确认✅/数量级⚠️.

- [ ] **Step 4: Record R1 resolution in the spec**

In the spec's §11 R1, append one line: `**R1 决议（实现期）**: 口径 = <have-SME | SVE-first pivot>；依据 = <chip + source>.`

- [ ] **Step 5: Commit**

```bash
git add docs/superpowers/research/2026-06-09-platform-sme-sve.md docs/superpowers/specs/2026-06-09-cpu-agent-3part-restructure-design.md
git commit -m "Resolve R1: ARM platform SME/SVE availability + P3-2 research"
```

---

## Task 2: Research Workflow for P2-2, P3-3, P3-4, P3-5 + adversarial verification

**Files:**
- Create: `docs/superpowers/research/2026-06-09-3part-research-findings.md`

**Why a Workflow:** Spec §10 mandates parallel research + adversarial verification; ultracode is on. Fan out one finder per research area, then an adversarial-verification stage that classifies each claim 确认✅/数量级⚠️ (the repo's signature 数据校验报告 discipline).

- [ ] **Step 1: Author and run the research Workflow**

Invoke the Workflow tool with a script shaped as: `phase('Research')` → `parallel` over 4 finder agents, each with a `schema` returning `{ area, claims: [{ claim, value, source, sourceUrl, context }] }`:
  - **P2-2 (A2A landscape):** mainstream multi-agent frameworks (AutoGen, MetaGPT, CrewAI, LangGraph/LangChain) — communication/topology model each; Google **A2A protocol (2025)** — what it standardizes, transport, status; academic multi-agent benchmarks with comm-overhead data (e.g., token/cost multipliers, message counts).
  - **P3-3 (fusion prior art):** compute-communication overlap / fusion on ARM and in HPC generally; any SVE/SME-accelerated data-movement or serialization work; gather/scatter for messaging. Establish what "通信计算融合" prior art exists.
  - **P3-4 (SME small-batch):** evidence that SME outer-product/tile suits small/skinny GEMM, embedding lookup, rerank, attention (vs large-batch training GEMM). Micro-benchmark or paper numbers, else flag as 数量级 + a self-measurable micro-bench plan.
  - **P3-5 (SVE serialization/messaging):** SVE/NEON-accelerated JSON/serialization (simdjson NEON → SVE prospects), gather/scatter for irregular message marshaling, zero-copy IPC on ARM, pack/transpose via vector ops.

Then `phase('Verify')` → for each claim, spawn an adversarial verifier prompted to refute / find the precise scope; classify 确认✅ (directly backed by paper abstract / measured benchmark) vs 数量级⚠️ (rule-of-thumb, depends on language/runtime/shape/topology). Return `{ claim, verdict, correctValueRange, quoteAdvice }`.

- [ ] **Step 2: Consolidate into the findings doc**

Create `docs/superpowers/research/2026-06-09-3part-research-findings.md`, one section per area (P2-2 / P3-3 / P3-4 / P3-5), each claim with `[来源: …, URL]` + classification (确认✅/数量级⚠️) + the verifier's quoteAdvice. No unsourced numbers.

- [ ] **Step 3: Verify every claim is sourced + classified**

```bash
grep -nE '[0-9]+(\.[0-9]+)?(ms|×|x|ns|GB/s|μs|s|%)' docs/superpowers/research/2026-06-09-3part-research-findings.md | grep -v '来源'
```
Expected: empty (every numeric claim has a nearby `来源`). If output appears, add the source or soften to qualitative.

- [ ] **Step 4: Commit**

```bash
git add docs/superpowers/research/2026-06-09-3part-research-findings.md
git commit -m "Research findings (P2-2, P3-3, P3-4, P3-5) with adversarial verification"
```

---

## Task 3: Create the new outline (21-page skeleton)

**Files:**
- Create: `2026-06-09-cpu-agent-3part-ppt-outline.md`

- [ ] **Step 1: Draft the 21-page skeleton**

Follow spec §2–§6 page tables exactly. Page-by-page (format `### 【第N页】标题` then 3–5 bullet 要点 + a `> 主线触点: …` line where applicable + a `来源: 承袭|新研究` tag). Order: 开场(P1) → P1-1..P1-5 → P2-1..P2-7 → P3-1..P3-7 → 总结+讨论. Tag each page's 来源 from the spec's 来源 column. For 来源=新研究 pages, reference the research doc (Task 1/2).

- [ ] **Step 2: Verify structure**

```bash
echo "page count:"; grep -cE '^### 【第' 2026-06-09-cpu-agent-3part-ppt-outline.md   # expect 21
echo "spine markers:"; grep -cE '主线触点|主线' 2026-06-09-cpu-agent-3part-ppt-outline.md   # expect ≥5 (P1-4, P2-6, P2-7, P3-3, P3-6)
echo "part headers:"; grep -cE '^## (开场|Part|第[一二三])|P[123] ' 2026-06-09-cpu-agent-3part-ppt-outline.md   # expect ≥3 part boundaries
```
Expected: page count = 21; spine markers ≥ 5; 3 part boundaries present. If off, fix the skeleton.

- [ ] **Step 3: Commit**

```bash
git add 2026-06-09-cpu-agent-3part-ppt-outline.md
git commit -m "Add 21-page outline for three-part restructure"
```

---

## Task 4: Detailed doc — header + 开场 + P1 (PAGE 1–6)

**Files:**
- Create: `2026-06-09-cpu-agent-3part-ppt-detailed.md`

- [ ] **Step 1: Write the doc header**

Title, 版本 (`2026-06-09 三部分重构版`), 受众 (同层级工程师; GEMM/Attention + MPI/UCX/SDMA on ARM), 边界 (不碰训练/推理内核), the 统一主线 block (verbatim from spec §1.2), and the 教学结构总览 table (spec §2). Add a 数据严谨性声明 pointer (per repo convention) saying new claims are verified in the extended 数据校验报告.

- [ ] **Step 2: Draft 开场 (PAGE 1) + P1-1..P1-5 (PAGE 2–6)**

Per spec §3 + §6. Carry over verified data from the old detailed doc: SWE-bench 250步/$3 上限, SWE-agent 12步/$1.21 vs 21步/$2.52, API 等待 500ms-5s 间隙本地 CPU 利用率~0% (label 数量级), workload 特征. Each page in the PAGE format (核心观点/详细要点/关键数据 with `[来源: …]`/演讲备注). **P1-4 must contain the 主线第一次点题** (the spine sentence + the 乒乓 diagram: 本地计算 ↔ 跨边界通信).

- [ ] **Step 3: Verify citations + spine**

```bash
echo "PAGE markers:"; grep -c '^========== PAGE' 2026-06-09-cpu-agent-3part-ppt-detailed.md   # expect 6 after this task
echo "unsourced numbers:"; grep -nE '[0-9]+(\.[0-9]+)?(ms|×|x|ns|GB/s|μs|min|%)' 2026-06-09-cpu-agent-3part-ppt-detailed.md | grep -v '来源'   # expect empty or qualitative-only
echo "spine first touch:"; grep -n '计算↔通信' 2026-06-09-cpu-agent-3part-ppt-detailed.md   # expect a hit in P1-4
```
Fix any gap (add `[来源: …]` or soften to qualitative).

- [ ] **Step 4: Commit**

```bash
git add 2026-06-09-cpu-agent-3part-ppt-detailed.md
git commit -m "Detailed doc: header + opening + Part 1 (pages 1-6)"
```

---

## Task 5: Detailed doc — P2 (PAGE 7–13)

**Files:**
- Modify: `2026-06-09-cpu-agent-3part-ppt-detailed.md` (append)

- [ ] **Step 1: Draft P2-1..P2-7 (PAGE 7–13)**

Per spec §4. Sources:
  - **P2-2** ← `docs/superpowers/research/2026-06-09-3part-research-findings.md` (A2A landscape: frameworks + A2A protocol + academic benchmarks).
  - **P2-3** ← carry over + **re-label 口径**: "2-5×" as token/成本经验法则 (数量级, NOT 总时间); "~58%" as 调度气泡 (scheduling bubbles) per Continuum arXiv:2511.02230, NOT 工具调用时间. Add 约/数量级 + context qualifiers.
  - **P2-4 / P2-5** ← carry over from old detailed doc (通信模式/序列化 JSON vs Protobuf vs FlatBuffers; 锁竞争 10-100× 已确认✅; 共识延迟 Raft/Paxos).
  - **P2-6** ← diagnosis synthesis (单→小swarm→大swarm, comm 占比上升成主导).
  - **P2-7** ← handoff: only the question, no answer; **主线第二次点题**.

- [ ] **Step 2: Verify citations + carried-over 口径**

```bash
echo "PAGE markers total:"; grep -c '^========== PAGE' 2026-06-09-cpu-agent-3part-ppt-detailed.md   # expect 13
echo "estimate qualifiers (2-5× / 58% must be qualified):"; grep -nE '2-5|58%|调度气泡' 2026-06-09-cpu-agent-3part-ppt-detailed.md   # check each line has 约/数量级/经验法则/scheduling bubbles nearby
echo "handoff page:"; grep -n '谁来付\|桥到 P3\|handoff' 2026-06-09-cpu-agent-3part-ppt-detailed.md   # expect a hit in P2-7
```
Fix any unqualified estimate or sourced gap.

- [ ] **Step 3: Commit**

```bash
git add 2026-06-09-cpu-agent-3part-ppt-detailed.md
git commit -m "Detailed doc: Part 2 A2A insight report (pages 7-13)"
```

---

## Task 6: Detailed doc — P3 + 总结/讨论 (PAGE 14–21)

**Files:**
- Modify: `2026-06-09-cpu-agent-3part-ppt-detailed.md` (append)

- [ ] **Step 1: Draft P3-1..P3-7 (PAGE 14–20) + 总结/讨论 (PAGE 21)**

Per spec §5 + Task 1's R1 口径 + Task 2 findings. Sources:
  - **P3-1** ← handoff落地 (主场一句话).
  - **P3-2** ← `docs/superpowers/research/2026-06-09-platform-sme-sve.md` (Task 1).
  - **P3-3** ← thesis 三件事 (重叠 / 数据搬运即计算 / kernel+原语协同) + P3-3 findings (prior art). **主线 thesis 展开.**
  - **P3-4** ← P3-4 findings, **口径 per R1** (have-SME = 算子机会; SVE-first = 前瞻).
  - **P3-5** ← carry-over simdjson + P3-5 findings (SVE serialization/messaging).
  - **P3-6** ← punchline, **口径 per R1** ("SME+SVE 同一组单元" if have-SME; "同一组向量单元、同一份布局" if SVE-first). **prescription climax.**
  - **P3-7** ← roadmap (短期 SVE 序列化/零拷贝 IPC → 中期 SME kernel 库/原语标准化 → 长期 benchmark + CPU-Agent 协同设计).
  - **总结/讨论 (PAGE 21)** ← thesis 回扣 + 2–3 ARM-ized 讨论问题 (from old 前瞻讨论问题, adapted: e.g. "若设计 Agent 专用通信原语，SVE gather/scatter 接口长什么样？").

- [ ] **Step 2: Verify citations + R1 口径 consistency**

```bash
echo "PAGE markers total:"; grep -c '^========== PAGE' 2026-06-09-cpu-agent-3part-ppt-detailed.md   # expect 21
echo "SME references vs R1:"; grep -nE 'SME|矩阵引擎|outer-product' 2026-06-09-cpu-agent-3part-ppt-detailed.md   # confirm P3-4/P3-6 match the R1 decision (have-SME vs SVE-first)
echo "punchline page:"; grep -n '同一组.*单元\|同一份布局\|融合点' 2026-06-09-cpu-agent-3part-ppt-detailed.md   # expect hit in P3-6
```
If R1 = SVE-first and P3-6 still asserts "SME+SVE 同一组单元", fix to the SVE-first wording. Fix any unsourced number.

- [ ] **Step 3: Commit**

```bash
git add 2026-06-09-cpu-agent-3part-ppt-detailed.md
git commit -m "Detailed doc: Part 3 ARM SME/SVE fusion + closing (pages 14-21)"
```

---

## Task 7: Appendix + extended 数据校验报告

**Files:**
- Modify: `2026-06-09-cpu-agent-3part-ppt-detailed.md` (append)

- [ ] **Step 1: Appendix 1 (延迟速查) + Appendix 2 (加速比速查)**

Carry over from old detailed doc's 附录1/附录2, extend with new SME/SVE/A2A data points (each `[来源: …]`). Note absolute latencies in 附录1, relative speedups in 附录2.

- [ ] **Step 2: Extended 数据校验报告**

Merge the old 10 claims (2 确认 / 8 数量级) + all NEW claims from Task 1 & Task 2 research. Table columns: 数据声明 / 校验结论(确认✅|数量级⚠️) / 正确值或范围 / PPT引用建议. Every new claim gets a row.

- [ ] **Step 3: Core 参考文献 (📚 核心参考文献汇总)**

Carry over the old reference groups (Agent 工作流 / Coding Agent 基准, 多 Agent 协同, LLM 推理优化, 系统软件/IPC/序列化, 共识/并发, CPU 推理/量化) + add new (A2A 协议/框架, ARM SME/SVE, 通信计算融合).

- [ ] **Step 4: Verify the report covers all new claims**

```bash
echo "校验报告 rows (声明 count):"; grep -cE '^\| .* \||确认|数量级' 2026-06-09-cpu-agent-3part-ppt-detailed.md | tail -1   # sanity: report table non-empty
echo "new research terms present in report:"; grep -nE 'SME|SVE|A2A|融合|simdjson' 2026-06-09-cpu-agent-3part-ppt-detailed.md | grep -E '校验|确认|数量级'   # expect new claims classified
```
If a new claim (e.g., an SME speedup number) appears in the body but not in the 校验报告, add its row.

- [ ] **Step 5: Commit**

```bash
git add 2026-06-09-cpu-agent-3part-ppt-detailed.md
git commit -m "Detailed doc: appendices + extended data-verification report + references"
```

---

## Task 8: Final verification pass

**Files:**
- Modify: `2026-06-09-cpu-agent-3part-ppt-detailed.md`, `2026-06-09-cpu-agent-3part-ppt-outline.md` (fixes only)

- [ ] **Step 1: Full citation-rigor sweep**

```bash
echo "=== unsourced numeric claims in detailed doc ==="
grep -nE '[0-9]+(\.[0-9]+)?(ms|×|x|ns|GB/s|μs|min|s|%)' 2026-06-09-cpu-agent-3part-ppt-detailed.md | grep -v '来源' | grep -vE '^\s*$'
```
Expected: empty (or only qualitative ranges clearly marked 数量级). Add `[来源: …]` or soften every line that appears.

- [ ] **Step 2: Structural + spine check**

```bash
echo "正文 pages:"; grep -c '^========== PAGE' 2026-06-09-cpu-agent-3part-ppt-detailed.md   # expect 21 + 附录 markers
echo "outline pages:"; grep -cE '^### 【第' 2026-06-09-cpu-agent-3part-ppt-outline.md   # expect 21
echo "spine touchpoints present:"; grep -nE 'P1-4|P2-6|P2-7|P3-3|P3-6|主线' 2026-06-09-cpu-agent-3part-ppt-detailed.md | head
```
Confirm: 21 content pages in both files; spine touchpoints (P1-4 第一次点题, P2-6 diagnosis, P2-7 handoff, P3-3 thesis展开, P3-6 climax) all present.

- [ ] **Step 3: 口径 consistency check**

```bash
echo "=== carried-over estimates must keep qualifier ==="
grep -nE '2-5|5-10|10-100|500ms-5s' 2026-06-09-cpu-agent-3part-ppt-detailed.md
echo "=== R1 SME口径 consistency (have-SME vs SVE-first) ==="
grep -nE 'SME' 2026-06-09-cpu-agent-3part-ppt-detailed.md | grep -E 'P3-4|P3-6|路线图|前瞻|已具备'
```
Confirm each carried-over estimate has 约/数量级 nearby; confirm P3-4/P3-6 SME references are consistent with the R1 decision recorded in Task 1.

- [ ] **Step 4: Cross-check outline ↔ detailed doc titles match**

Eyeball that the 21 page titles in the outline match the 21 `PAGE N:` titles in the detailed doc (same order, same spine markers). Fix mismatches.

- [ ] **Step 5: Commit (if any fixes)**

```bash
git add 2026-06-09-cpu-agent-3part-ppt-detailed.md 2026-06-09-cpu-agent-3part-ppt-outline.md
git commit -m "Final verification pass: citation rigor, structure, spine, 口径 consistency"
```
If no fixes were needed, skip the commit and note "verification clean."

---

## Self-Review (run after writing this plan — already done, findings below)

- **Spec coverage:** §1–§2 → Tasks 3–4 header; §3 P1 → Task 4; §4 P2 → Task 5; §5 P3 → Task 6; §6 开场/总结 → Tasks 4/6; §7 carry-over → Tasks 4/5/6; §8 citation → all tasks + Task 8; §9 deliverables → Tasks 3 + 4–7; §10 impl method → Task 1 (R1 first) + Task 2 (workflow) + drafting tasks; §11 risks → Task 1 (R1) + Task 6 (R1口径) + Task 7 (R2/R3 mitigated via 数量级 + micro-bench plan). **No gaps.**
- **Placeholder scan:** No TBD/TODO/"add error handling"/"similar to Task N". Every step has concrete files, content sources, and verification commands.
- **Type/name consistency:** Page numbering (开场=1, P1=2–6, P2=7–13, P3=14–20, 总结=21) is consistent across Tasks 3–8 and the File Structure section. Spine touchpoint page IDs (P1-4/P2-6/P2-7/P3-3/P3-6) consistent everywhere. Research filenames referenced identically in Task 1/2 and the drafting tasks that consume them.
