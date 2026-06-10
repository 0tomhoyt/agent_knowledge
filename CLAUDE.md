# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

`agent_knowledge` ("Agent Knowledge / 知识库管理系统") is a **knowledge-management / technical-writing repository**. It contains Markdown documents — primarily internal technical-sharing decks and their source material for a CPU low-level software team. There is **no source code, no build system, no test suite, and no linter**. Do not look for or invent build/test/lint commands; the work here is authoring, refining, and verifying Markdown content.

Content is written primarily in **Chinese (中文)**, with technical terms kept in English (e.g. *Agent, FlashAttention, IPC, GEMM, MPI/UCX*). Match this bilingual convention when editing.

## Document workflow and structure

A single topic is typically expressed as **three documents representing successive refinement tiers** — understand which tier you are editing before changing content:

| Tier | Location | Role |
|------|----------|------|
| 1. Design spec | `docs/superpowers/specs/<DATE>-<slug>-design.md` | Planning/design doc (audience, structure, section-level talking points, data tables). The starting point. |
| 2. Outline | `./<DATE>-<slug>-ppt-outline.md` | Slide-by-slide outline (~25–27 pages), section + page structure with bullet talking points. |
| 3. Detailed deliverable | `./<DATE>-<slug>-ppt-detailed.md` | Fully fleshed-out deck: per-page core claim, detailed points, key data with citations, speaker notes. Also contains the **data verification report** and reference appendix. |

The current (and so far only) topic is the **CPU底层软件团队：Agent时代研究方向（三部分重构版）** deck (`2026-06-09-cpu-agent-3part-…`). An earlier `era-direction` four-part version existed but was consolidated out on 2026-06-10 (verified data merged into the `-3part-` docs; recoverable via git history) — the repo now holds exactly one deck version. When adding a new topic, replicate this three-tier layout and the date-prefixed filename convention (`YYYY-MM-DD-<slug>.md`).

## The cardinal authoring rule: citation rigor + adversarial data verification

This is the single most important convention and the main thing that distinguishes the detailed deliverable (tier 3) from the earlier tiers. **Every quantitative/performance claim in a tier-3 document must:**

1. Carry an inline citation in the form `[来源: <paper / vendor benchmark / official doc, arXiv number / URL>]`.
2. Be classified as either **✅ 确认 (confirmed)** — backed directly by a paper's abstract or a measured benchmark — or **⚠️ 数量级估算 (order-of-magnitude estimate)** — a rule-of-thumb whose real value depends heavily on language, runtime, data shape, and topology.
3. Appear in the doc's **🔍 数据校验报告 (adversarial data-verification report)**, which grades each headline number and prescribes how it may be quoted in the deck.

Concretely, the detailed doc enforces this with a verification table that flags, for example, that "JSON比Protobuf慢5-10×" and "GPT-4延迟500ms-5s" are **order-of-magnitude estimates** and must be quoted with a qualifier (约/数量级) and a bounded context (language / backend / batch / topology), whereas "FlashAttention 2-4×" and "lock contention 10-100× slowdown" are **confirmed** and may be quoted directly.

**When editing tier-3 content:** do not introduce new performance numbers without a citation and a confirmed-vs-estimate classification, and do not strip the qualifier ("约"/"数量级") off an estimate to make a headline punchier. If a claim can't be sourced, soften it to a qualitative statement rather than inventing a precise-looking figure. Absolute latencies belong in **附录1 (Appendix 1)**; relative speedup multiples belong in **附录2 (Appendix 2)**.

The tier-1 design spec and tier-2 outline are *less strict* — they predate the verification pass and contain some un-qualified estimates. Treat the tier-3 document as the source of truth for any number; if the tiers disagree, the tier-3 verified value wins.

## How work happens here

The `docs/superpowers/specs/` path signals that this repo uses the **Superpowers skill workflow** (the `superpowers:*` skills available in this environment). New substantive work should follow it: `brainstorming` → `writing-plans` (spec goes in `docs/superpowers/specs/`) → drafting the outline → expanding to the verified detailed doc. When asked to add content or a new deck, follow that progression rather than jumping straight to a finished document.

## Reference pointers

- `docs/superpowers/specs/2026-06-09-cpu-agent-3part-restructure-design.md` — the design spec; the fastest way to understand the deck's overall structure, sections, and intended audience.
- `2026-06-09-cpu-agent-3part-ppt-detailed.md` — the authoritative, citation-checked deliverable; start here for any data question, then check its 数据校验报告 and 附录1 (绝对延迟) / 附录2 (加速比) / 附录3 (平台 ISA).
- `2026-06-09-cpu-agent-3part-slides.marp.md` + `2026-06-09-cpu-agent-3part-speaker-cheatsheet.md` — the presentable Marp deck (renders to html/pdf/pptx) and the speaker cheat-sheet.
