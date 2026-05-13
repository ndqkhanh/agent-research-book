# 218 — Atlas-Research Apply Plan: Polaris-Lite Deep Research with Relaxed Gates

> **Disambiguation.** This file is the **Atlas-Research–specific** deep apply plan, extending the per-project apply-plan family ([203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md), [208-lyra-multi-hop-collaborative-apply-plan](208-lyra-multi-hop-collaborative-apply-plan.md), [209-argus-multi-hop-collaborative-apply-plan](209-argus-multi-hop-collaborative-apply-plan.md), [210-mentat-learn-collaborative-apply-plan](210-mentat-learn-collaborative-apply-plan.md)). The cross-project matrix is [207](207-cross-project-collaborative-multi-hop-apply-plan.md); the executable strategy doc is [211](211-cross-project-power-up-plan-with-tradeoffs.md). Atlas-Research's role in the portfolio: **deep research with relaxed gates** — Polaris's discipline backbone is missing; throughput and breadth are the priorities.

## One-line definition

A staged plan to fold multi-hop reasoning + collaborative-AI techniques into Atlas-Research — the **long-horizon research agent** baselined against OpenAI Deep Research, Perplexity Pro, and Gemini Deep Research, evaluated on GAIA / BrowseComp / internal synthesis rubrics — with the design target of higher faithfulness-cited outputs at bounded cost per report, and the executable answer to "what would Polaris look like if you stripped out the twenty bright-line gates and prioritised throughput?"

## §1 — Why apply these techniques to Atlas-Research

Atlas-Research and Polaris are **siblings** with deliberately different positioning ([projects/README.md](../projects/README.md)):
- **Polaris** = discipline winner. Twenty bright-line gates, structural provenance, multi-domain shells, typed trust tiers (T1–T3 + RED-RETRACTED), claim-gate rigor.
- **Atlas-Research** = throughput winner. Same multi-hop substrate, far fewer gates, faster cycle time, broader coverage, less claim-grade trust enforcement.

Both projects need the multi-hop reasoning canon ([198](198-multi-hop-qa-datasets-canon.md) → [202](202-multi-agent-multi-hop-reckoning-2026.md)) and most of the collaborative-AI canon ([206](206-collaborative-ai-canon-2026.md)). What differs is the *gate surface* — Atlas-Research is the right place to ship multi-hop-substrate-only without the twenty BL-* gates, prove the substrate works at speed, then let Polaris layer rigor on top.

Take this seriously and three things change. (1) Atlas becomes the **fast-iteration testbed** for techniques that later harden in Polaris. (2) The shared `harness_core/` library proposed in [211](211-cross-project-power-up-plan-with-tradeoffs.md) §4 ships in Atlas first because it doesn't have to compose with Polaris's gate surface. (3) The OpenAI Deep Research / Perplexity / Gemini Deep Research baseline becomes a public benchmark Atlas can iterate against weekly without enterprise-grade rigor blocking ship velocity.

## §2 — The Atlas-Research × technique mapping table

Atlas's existing module structure is `src/atlas_research/`, with vendored `harness_core/`, `skills/`, `tests/`, `docs/`. The mapping is similar to Polaris's but with relaxed gates and faster tiers.

| Technique | Source | Atlas module | Tier | Effort | Payoff |
|---|---|---|---|---|---|
| **Multi-hop substrate** | | | | | |
| HippoRAG-2 PPR fusion | [200](200-graph-grounded-multi-hop-retrieval.md), promoted from `polaris-core` per [211](211-cross-project-power-up-plan-with-tradeoffs.md) | `harness_core/multi_hop/hipporag.py` (vendored from shared lib) | 0 | M | Multi-hop retrieval over federated sources, single-shot |
| LightRAG incremental ingest | [200](200-graph-grounded-multi-hop-retrieval.md) | `src/atlas_research/graph/incremental.py` | 0 | S | Online ingest of new sources |
| BELLE-style query-type router | [202](202-multi-agent-multi-hop-reckoning-2026.md) §1 | `src/atlas_research/routing/query_router.py` | 0 | S | Method routing without debate cost |
| Chain-of-Note per-doc filter | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 3 | `harness_core/gates/chain_of_note.py` | 0 | S | Per-doc quality gate (cheaper than Polaris's two-axis gate) |
| Self-Ask / IRCoT externalised chain | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 1 | `src/atlas_research/reasoning/{self_ask,ircot}.py` | 0 | S | Externalises bridge entity; closes [201](201-compositionality-gap-canon.md) gap |
| Decomposition cache | [199](199-multi-hop-reasoning-techniques-arc.md) | `src/atlas_research/memory/decomposition_cache.py` | 0 | S | Latency cut on recurring sub-queries |
| MuSiQue-Full + FRAMES + Seal-0 + GAIA + BrowseComp nightly | [198](198-multi-hop-qa-datasets-canon.md) | `tests/benchmarks/multi_hop_triple/` + `tests/benchmarks/{gaia,browsecomp}.py` | 0 | S | Atlas baselines GAIA and BrowseComp directly (Polaris does not) |
| DSPy multi-hop program | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 2 | `skills/research/dspy_atlas_program.py` | 1 | M | Compiled-against-evals multi-hop pipeline |
| Plan-on-Graph backtracking | [200](200-graph-grounded-multi-hop-retrieval.md) | `src/atlas_research/graph/plan_walk.py` | 1 | M | Recovers from wrong-path traversal |
| AnchorRAG entity linking | [200](200-graph-grounded-multi-hop-retrieval.md) | `src/atlas_research/graph/anchor_predictor.py` | 1 | M | Robust to imperfect entity linking on open-web |
| Beam Retrieval ≥3 hop | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 3 | `src/atlas_research/graph/beam_retrieval.py` | 1 | M | SOTA on deeper hops |
| Reason-in-Documents denoiser | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 4 | `src/atlas_research/reasoning/reason_in_documents.py` | 1 | S | Filters noisy retrievals before injection |
| Search-R1 RL recipe | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 4 | `src/atlas_research/rl/search_r1.py` | 2 | L | Trained executor at relaxed-gate baseline |
| **Collaborative-AI substrate** | | | | | |
| ICPEA five-layer personal memory | [205](205-lobehub-collaborative-teammate-platform.md) §2.1 | `src/atlas_research/memory/icpea/` | 0 | M | Per-user research-style + preferred-source memory |
| Async memory extractor on report-finalize | [206](206-collaborative-ai-canon-2026.md) §1 | `src/atlas_research/memory/icpea/extractor.py` | 0 | S | Decouples memory write from chat latency |
| Per-agent layer access control | [205](205-lobehub-collaborative-teammate-platform.md) §2.1 | `src/atlas_research/memory/icpea/access.py` | 1 | S | Sub-agents see only declared layers |
| Branching as research-tree UX | [206](206-collaborative-ai-canon-2026.md) §4 | `src/atlas_research/conversation/branching.py` | 1 | M | Surface alternative research directions |
| Lobe Pages-style co-authored report | [205](205-lobehub-collaborative-teammate-platform.md) §2.2 | `src/atlas_research/output/pages.py` | 1 | M | Co-authored markdown report with timeline |
| MCP-first marketplace via argus trust gate | [205](205-lobehub-collaborative-teammate-platform.md) §2.3, [209](209-argus-multi-hop-collaborative-apply-plan.md) | `src/atlas_research/tools/mcp_marketplace.py` (consumes `argus.HostAdapter`) | 1 | S | Trust-gated tool installation |
| Voyager-line skill auto-creation | [167](167-autoskill-experience-driven-lifelong-learning.md), [197](197-argus-omega-vol-3-recursive-skills-curator.md) | `src/atlas_research/skills/auto_creator.py` (submits to `argus.curator`) | 1 | M | Successful research patterns become reusable skills |
| Per-user constitution | [206](206-collaborative-ai-canon-2026.md) §5 | `src/atlas_research/constitution/` | 2 | M | Serialised research-style preference object |
| Voice + screen-share | [206](206-collaborative-ai-canon-2026.md) §6 | `src/atlas_research/channels/voice/` | 2 | L | "Research this paper while I show you my notebook" |
| **Cross-cutting discipline** | | | | | |
| Pure-function agents | [202](202-multi-agent-multi-hop-reckoning-2026.md) §4 | `harness_core/orchestration/pure_function.py` | 0 | S | Replayable trajectories |
| Equal-budget benchmark harness | [202](202-multi-agent-multi-hop-reckoning-2026.md) §3 | `harness_core/evals/equal_budget.py` | 0 | S | Honest single-vs-deep-research comparisons |
| TTC curve plotter | [199](199-multi-hop-reasoning-techniques-arc.md) | `harness_core/evals/ttc_curve.py` | 0 | S | Detect plateau-or-decline on noisy web |
| gnomon HIR emit | [211](211-cross-project-power-up-plan-with-tradeoffs.md) §4 | `src/atlas_research/observability/hir_emit.py` | 0 | S | Cross-harness eval signal |

## §3 — Tier 0 (days 1-14): the relaxed-gate multi-hop substrate

The cheapest, highest-payoff lifts. Atlas-Research diverges from Polaris by *not* installing the twenty BL-* gates — Tier 0 hardens the substrate with Chain-of-Note as the only first-class quality gate.

### 0.1 — HippoRAG-2 from shared library

**File.** `harness_core/multi_hop/hipporag.py` (vendored from `harness_core.multi_hop.hipporag@1.x` per [211](211-cross-project-power-up-plan-with-tradeoffs.md) §4).

**What.** Atlas-Research is one of the four projects (Polaris, Lyra, Atlas, Helix-Bio, Cipher-Sec) that consume the HippoRAG-2 PPR fusion module promoted from `polaris-core/memory/ppr_fusion.py`. Atlas's domain is open-web research; the KG is built from federated sources (web pages, papers, Wikipedia, OpenScholar).

**Atlas-specific configuration.** Larger graph than Polaris (open-web vs curated T1 sources), so PPR damping is tuned for noisier graph; entity-linking uses AnchorRAG (Tier 1) by default.

### 0.2 — BELLE query router + Self-Ask / IRCoT operators

**Files.** `src/atlas_research/routing/query_router.py`, `src/atlas_research/reasoning/{self_ask,ircot}.py`.

**What.** BELLE-style classifier types incoming queries into:
- **Single-hop factual** — direct retrieval, no chain.
- **Multi-hop bridging** — IRCoT externalised chain.
- **Fan-out comparative** — sub-question dispatch (a la [LlamaIndex SubQuestionQueryEngine](https://docs.llamaindex.ai/en/stable/examples/query_engine/sub_question_query_engine/)).
- **Global-sensemaking** — GraphRAG community summary.
- **Open-web browsing** — agentic browse (Search-R1-style at Tier 2; agentic search at Tier 0).

### 0.3 — Chain-of-Note as the *only* first-class gate

**File.** `harness_core/gates/chain_of_note.py`.

**What.** Atlas-Research deliberately ships *without* Polaris's twenty BL-* gates. Chain-of-Note is the one quality gate — per-doc relevance assessment by the LM before reasoning injection. **+7.9 EM** on entirely-noisy retrievals; **+10.5** rejection-rate on out-of-knowledge questions ([199](199-multi-hop-reasoning-techniques-arc.md) §"Chain-of-Note").

**Cost of this divergence.** Atlas accepts that occasional low-quality citations slip through. The trade-off: ship velocity. Polaris's gate surface comes later via Tier 2 if needed; Atlas may instead consume Polaris's gate-graded outputs as a federated source.

### 0.4 — ICPEA five-layer personal memory

**File.** `src/atlas_research/memory/icpea/{layers,extractor,injection}.py`.

**What.** The standard ICPEA schema ([205](205-lobehub-collaborative-teammate-platform.md) §2.1, [206](206-collaborative-ai-canon-2026.md) §1) tuned for research:
- **I**dentity — "I'm a venture-tech analyst at a fund covering biotech."
- **C**ontext — current research project, deadline, target audience.
- **P**reference — preferred sources (T1 papers vs analyst reports), citation style, language, depth.
- **E**xperience — what worked (research patterns the user accepted), what failed.
- **A**ctivity — research queries, sources cited, reports finalized.

**Async extractor.** Triggers on report-finalize. Extracts ICPEA rows from the trajectory + user feedback.

### 0.5 — Multi-hop benchmark suite + GAIA + BrowseComp

**File.** `tests/benchmarks/multi_hop_triple/` + `tests/benchmarks/{gaia,browsecomp}.py`.

**What.** Atlas-Research's design target ([projects/README.md](../projects/README.md)) baselines against GAIA + BrowseComp. Tier 0 baselines:
- MuSiQue-Full + FRAMES + Seal-0 (the [198](198-multi-hop-qa-datasets-canon.md) triple).
- GAIA — the canonical generalist agent benchmark.
- BrowseComp — open-web browsing.

Run nightly. Equal-budget + TTC + active-params accounting on every run.

### 0.6 — Pure-function agents + equal-budget + TTC + gnomon HIR emit

Cross-cutting Tier 0 discipline per [211](211-cross-project-power-up-plan-with-tradeoffs.md). Pure-function agents are non-negotiable — Atlas-Research's sub-agents fan out wide (5–15 parallel research queries) and need to be replayable.

## §4 — Tier 1 (days 15-45): the architecture surface

After Tier 0, Atlas has multi-hop substrate + ICPEA + Chain-of-Note gate. Tier 1 makes the executor branch-able and user-aware.

### 1.1 — DSPy multi-hop program

**File.** `skills/research/dspy_atlas_program.py`.

**What.** A DSPy `Module` that composes Tier 0's operators. Compiles against held-out evals (MuSiQue-Full, FRAMES, GAIA-subset). Atlas's program is *broader* than Polaris's (no T-tier gating) and *deeper* than Lyra's (long-horizon research).

### 1.2 — Plan-on-Graph backtracking + AnchorRAG + Beam Retrieval

**Files.** `src/atlas_research/graph/{plan_walk,anchor_predictor,beam_retrieval}.py`.

**What.** Three composable graph-walk improvements:
- PoG: backtrack when a research path is wrong.
- AnchorRAG: predict plausible entity anchors when entity linking is ambiguous (essential for open-web research where entities aren't pre-registered).
- Beam Retrieval: SOTA on hops > 2 (3–4 hop research questions are common).

### 1.3 — Reason-in-Documents denoiser

**File.** `src/atlas_research/reasoning/reason_in_documents.py`.

**What.** Search-o1's denoising step. Refines retrieved web pages before injection. Composes with Chain-of-Note (CoN is per-doc filter; RiD is per-doc summary).

### 1.4 — Branching as research-tree UX

**File.** `src/atlas_research/conversation/branching.py`.

**What.** "Pursue this lead" vs "Pursue this alternative lead" — fork at any point in the research trajectory; merge or discard branches at synthesis time. Each branch is a separate sub-agent run. Pure-function agents (Tier 0) make this safe.

### 1.5 — Lobe Pages-style co-authored research report

**File.** `src/atlas_research/output/pages.py`.

**What.** The user and the agent co-author the research report in a Notion-style timeline. Each citation is a typed evidence row; the user can edit, accept, or reject. The timeline preserves prior versions.

### 1.6 — MCP marketplace via argus trust gate

**File.** `src/atlas_research/tools/mcp_marketplace.py`.

**What.** Consume `argus.HostAdapter.install_mcp(...)` for tool installation. Atlas's typical MCP needs: web-search, paper-search (OpenScholar / Semantic Scholar / Arxiv), citation-extractor, table-extractor, chart-extractor.

### 1.7 — Voyager-line skill auto-creation

**File.** `src/atlas_research/skills/auto_creator.py`.

**What.** Successful research trajectories submit to `argus.curator.submit_trajectory(...)`. Atlas pulls promoted skills back from argus's curator (cross-project commons including skills promoted by Polaris, Lyra, Helix-Bio).

## §5 — Tier 2 (days 46-90): training surface + voice

### 2.1 — Search-R1 RL recipe

**File.** `src/atlas_research/rl/search_r1.py`.

**What.** Fork the Search-R1 contract from `polaris-research/rl/`. Atlas's reward function is **simpler than Polaris's** — Atlas doesn't have BL-* gates as reward-shaping signals. Reward = (faithfulness × cited-source-trust × answer-correctness), where trust is from argus's marketplace verdict + LobeHub reputation, *not* Polaris's typed T1-T3 system.

### 2.2 — Per-user constitution

**File.** `src/atlas_research/constitution/{model,inject,update}.py`.

**What.** ICAI-style 3-principle constitution per researcher. Examples:
- "Default to peer-reviewed sources; flag preprints as such."
- "Always include opposing-view sources for controversial claims."
- "Prefer recent (≤3y) sources for fast-moving fields; ≤10y for established fields."

### 2.3 — Voice + screen-share

**File.** `src/atlas_research/channels/voice/`.

**What.** "Research this paper while I show you my notebook" — voice + screen-share for collaborative exploration. Lower priority than Mentat or Helix because Atlas's user is typically at a desktop with a chat interface; voice is convenience, not core.

## §6 — Five sequencing decisions worth defending

### Decision 1 — No BL-* gates (relaxed-gate stance)

Atlas's *whole point* is being Polaris's faster sibling. Installing the twenty BL-* gates collapses the differentiation. Chain-of-Note is the one gate; everything else is taste.

### Decision 2 — Atlas as the testbed for shared `harness_core/`

Per [211](211-cross-project-power-up-plan-with-tradeoffs.md), shared library modules (`pure_function`, `equal_budget`, `chain_of_note`, `hipporag`) ship in Atlas first because Atlas doesn't have to compose with Polaris's gate surface. Atlas's CI catches integration issues early; Polaris consumes after.

### Decision 3 — GAIA + BrowseComp in CI from day one

Atlas's design target is GAIA + BrowseComp. These are public benchmarks; running them nightly creates a transparent leaderboard signal. Polaris doesn't ship these directly because Polaris's distinctive surface is rigor, not throughput.

### Decision 4 — Search-R1 reward simpler than Polaris's

Polaris's reward includes BL-* gate satisfaction. Atlas's omits these — reward is just (faithfulness × source-trust × correctness). Simpler reward = easier training; lower ceiling on rigor (which is fine because Atlas is throughput-tier).

### Decision 5 — Voice + screen-share Tier 2, not Tier 1

Atlas's user is typically desktop-first; voice is a "nice-to-have" not a "core" surface. Defer to Tier 2; Mentat and Helix-Bio prioritise voice higher.

## §7 — Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Relaxed gates let low-quality citations through | High | Med | Chain-of-Note gate + post-hoc audit sample; weekly quality-bar review; users marked when Atlas's source quality dips below a threshold |
| HippoRAG promotion churn from Polaris's v2.5 | Med | Med | Pin to `harness_core.multi_hop.hipporag@1.x`; consume Polaris's `2.x` only after explicit migration window |
| GAIA / BrowseComp leaderboard regressions | Med | Med | Nightly CI; alarm on >5pt regression; per-PR benchmark in mandatory CI for retrieval-path changes |
| Search-R1 reward-hacks faithfulness | Med | High | Held-out evaluation set + LLM-as-judge audit + sample-based human review |
| ICPEA cross-contaminates with Polaris's user namespace | Low-Med | Med | Per-project ICPEA namespace; user-scope keys include project-id |
| Argus marketplace SPOF on tool installation | Low | Med | Local cache for installed tools; offline install path for emergency |
| Branching exhausts API budget on fan-out | Med | Med | Per-branch budget caps + total-research budget cap; alarm on excessive fanout |
| Pure-function refactor blocks existing Atlas users | Low | Med | Phased rollout; existing trajectories migrated lazily |
| Equal-budget benchmark misrepresents fan-out architecture | High | Low-Med | Report per-fanout cost separately; document Atlas's parallel sub-agent dispatch in eval reports |
| Atlas-Polaris feature divergence makes shared library hard | Med | Med | Shared library has minimal API; project-specific glue in `src/atlas_research/` |

## §8 — Concrete day-by-day Tier 0 checklist

A 14-day Tier 0 plan:

- **Day 1-2** — `harness_core/orchestration/pure_function.py` adopted; agent loop refactored.
- **Day 3** — `harness_core/multi_hop/hipporag.py` vendored from shared library; first ingest of a federated corpus.
- **Day 4** — `harness_core/gates/chain_of_note.py` wired as the one Atlas gate.
- **Day 5-6** — `src/atlas_research/routing/query_router.py` (BELLE-style); operator stubs for Self-Ask / IRCoT.
- **Day 7-8** — `src/atlas_research/memory/icpea/{layers,extractor,injection}.py`; first session-finalize extraction.
- **Day 9** — `src/atlas_research/memory/decomposition_cache.py`.
- **Day 10-11** — `tests/benchmarks/multi_hop_triple/` + `tests/benchmarks/{gaia,browsecomp}.py`.
- **Day 12** — `harness_core/evals/{equal_budget,ttc_curve,active_params}.py` adopted.
- **Day 13** — `src/atlas_research/observability/hir_emit.py` to gnomon.
- **Day 14** — Tier 0 retro: GAIA / BrowseComp / MuSiQue / FRAMES / Seal-0 baselines published; sign-off for Tier 1.

## §9 — How Atlas-Research relates to Polaris and Lyra

After Tier 1:

| Project | Niche | Gate surface | Multi-hop substrate | When to use |
|---|---|---|---|---|
| **Polaris** | Discipline-grade deep research | Twenty BL-* gates + typed trust tiers + structural provenance | Full (HippoRAG + DSPy + PoG + RL contract) | Regulated industries; high-stakes claims; auditability matters |
| **Atlas-Research** | Throughput deep research | Chain-of-Note only | Same multi-hop substrate, simpler reward | Throughput-prioritising research; broader coverage; less rigor |
| **Lyra** | CLI-native general-purpose coding harness | Hooks + permission modes | Code-search-shaped (tree-sitter graph + HippoRAG) | Coding agent at the developer's terminal |

Atlas and Polaris **share `harness_core` modules** but **diverge on gate surface and reward function**. The architectural symmetry means a technique that works in Atlas usually ports cleanly to Polaris with the gate surface added on top.

## §10 — Cross-references

- [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md) — sibling Polaris plan (rigor-first).
- [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md), [211-cross-project-power-up-plan-with-tradeoffs](211-cross-project-power-up-plan-with-tradeoffs.md) — cross-project context.
- [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md) — Polaris's distinctive surface (what Atlas deliberately omits).
- [158-deep-research-agents-survey-huang-et-al](158-deep-research-agents-survey-huang-et-al.md), [159-deep-research-survey-zhang-et-al](159-deep-research-survey-zhang-et-al.md), [160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md), [166-research-agent-frameworks-synthesis](166-research-agent-frameworks-synthesis.md) — deep-research-agent landscape.
- [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md), [197-argus-omega-vol-3-recursive-skills-curator](197-argus-omega-vol-3-recursive-skills-curator.md) — argus integration substrate.

## §11 — One-paragraph summary

Atlas-Research's collaborative-AI + multi-hop subsystem ships in three tiers: Tier 0 (14 days) hardens the relaxed-gate substrate with HippoRAG-2 (vendored from shared library) + BELLE query router + Self-Ask/IRCoT externalised chains + Chain-of-Note as the one gate + ICPEA five-layer personal memory + decomposition cache + GAIA/BrowseComp/MuSiQue/FRAMES/Seal-0 benchmark suite + pure-function agents + gnomon HIR emit; Tier 1 (30 days) adds DSPy multi-hop program + Plan-on-Graph backtracking + AnchorRAG + Beam Retrieval + Reason-in-Documents + branching as research-tree UX + Lobe Pages co-authored report + argus trust-gated MCP marketplace + skill auto-creation; Tier 2 (60 days) trains the executor with Search-R1 (simpler reward than Polaris) + per-user constitution + voice/screen-share. The five sequencing decisions defended in §6 keep Atlas as Polaris's faster sibling — same multi-hop substrate, deliberately stripped gate surface, GAIA + BrowseComp as the public scoreboard, throughput as the differentiator.

**The one-line takeaway for harness designers:** Atlas-Research is the **throughput-tier sibling** of Polaris — adopt the same multi-hop substrate, omit the twenty BL-* gates, ship GAIA + BrowseComp leaderboard signal weekly, and let Polaris add the rigor surface back on top later.
