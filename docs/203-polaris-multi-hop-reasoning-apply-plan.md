# 203 — Applying Multi-Hop Reasoning Techniques into Polaris

> **Disambiguation.** This file is the **apply-to-Polaris** synthesis of the multi-hop reasoning block (198–202). It maps the canonical multi-hop techniques onto Polaris's existing modules and the v2.2/v2.3/v2.4 roadmap from [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md). Read after [198](198-multi-hop-qa-datasets-canon.md), [199](199-multi-hop-reasoning-techniques-arc.md), [200](200-graph-grounded-multi-hop-retrieval.md), [201](201-compositionality-gap-canon.md), [202](202-multi-agent-multi-hop-reckoning-2026.md).

## One-line definition

A staged plan to fold multi-hop reasoning capability into Polaris **without breaking its existing distinctive surface** (typed trust tiers, twenty bright-line gates, structural provenance, multi-domain shells, daemon idle policy) — by treating multi-hop as a **first-class subsystem** that plugs into the existing module boundaries and is delivered across three tiers (Tier 0 hardening: 14 days; Tier 1 architecture: 30 days; Tier 2 training: 60 days) tracking the [172](172-polaris-2026-deep-research-roadmap.md) v2.2/v2.3/v2.4 phasing.

## §1 — Why apply this to Polaris now

Polaris already wins on **discipline**. It does **not** yet win on **multi-hop reasoning capability**. Both [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md) §3 (the five gaps) and [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md) (the equal-budget rebuttal of multi-agent gains) point to the same truth: Polaris's executor is **prompted not trained**, **single-threaded not tree-searched**, and runs without a **graph-grounded multi-hop retrieval cache**. Without these capabilities Polaris cannot close the gap to Tongyi DeepResearch on BrowseComp / FRAMES / SealQA, and the structural provenance / two-axis evidence gates that *are* its edge become weaker than they could be when fed by a stronger multi-hop substrate.

The right multi-hop subsystem for Polaris is *not* a clone of Tongyi. It is a Polaris-native synthesis:

- **HippoRAG** under the hood for query-time multi-hop retrieval (cheap, single-shot, cache-amortised) — see [200](200-graph-grounded-multi-hop-retrieval.md).
- **DSPy multi-hop** as the declarative composition layer over the operator bank — [199](199-multi-hop-reasoning-techniques-arc.md) Phase 2.
- **Search-R1 / Tongyi GRPO recipe** for the *trained* Tier-2 executor — [199](199-multi-hop-reasoning-techniques-arc.md) Phase 4 + [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md) §7.
- **BELLE-style routing** as a question-type pre-classifier — [202](202-multi-agent-multi-hop-reckoning-2026.md) §1.
- **Always emit the chain** (CoT, sub-questions, graph walks) — the compositionality gap ([201](201-compositionality-gap-canon.md)) is architectural; externalising the chain is the only known robust fix.
- **Yenugula's pure-function-agent pattern** ([202](202-multi-agent-multi-hop-reckoning-2026.md) §4) for the orchestration layer — already Polaris-shaped in `polaris-core/orchestration/`.

## §2 — The Polaris × multi-hop mapping table

Each row is one technique from [198–202], mapped to a Polaris module path and a tier slot.

| Technique | Source | Polaris module | Tier | Effort | Payoff |
|---|---|---|---|---|---|
| HippoRAG single-shot multi-hop retrieval | [200](200-graph-grounded-multi-hop-retrieval.md) | `polaris-mcp/sources/hipporag.py` + `polaris-graph/multihop_index.py` | 0 | M | High — closes vector-only gap |
| LightRAG-style incremental graph ingest | [200](200-graph-grounded-multi-hop-retrieval.md) | `polaris-graph/incremental_ingest.py` | 0 | S | Online ingest of new sources |
| BELLE-style question-type router | [202](202-multi-agent-multi-hop-reckoning-2026.md) §1 | `polaris-skills/research/question_type_router.py` | 0 | S | Method routing without debate cost |
| Chain-of-Note per-doc filter | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 3 | `polaris-core/permissions/gates/chain_of_note.py` | 0 | S | Per-doc quality gate before reasoning |
| Self-Ask / IRCoT explicit decomposition | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 1 | `polaris-skills/research/self_ask.py` + `ircot.py` | 0 | S | Externalises bridge entity; closes [201](201-compositionality-gap-canon.md) gap |
| Decomposition cache | [198](198-multi-hop-qa-datasets-canon.md) | `polaris-core/memory/decomposition_cache.py` | 0 | S | Latency cut on repeated sub-questions |
| MuSiQue-Full + FRAMES + Seal-0 nightly CI | [198](198-multi-hop-qa-datasets-canon.md) | `polaris-evals/benchmarks/multi_hop_triple/` | 0 | S | Regression signal for the whole subsystem |
| DSPy multi-hop as declarative composition layer | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 2 | `polaris-skills/research/dspy_multihop_program.py` | 1 | M | Compiled pipelines vs hand-prompted |
| Plan-on-Graph adaptive backtracking | [200](200-graph-grounded-multi-hop-retrieval.md) | `polaris-skills/research/branch_and_prune.py` (extends existing tree-controller) | 1 | M | Recovers from wrong-path / wrong-anchor failures |
| AnchorRAG anchor-predictor (open-world entity linking) | [200](200-graph-grounded-multi-hop-retrieval.md) | `polaris-graph/anchor_predictor.py` | 1 | M | Robust to imperfect entity linking |
| Beam Retrieval for >2 hop | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 3 | `polaris-graph/beam_retrieval.py` | 1 | M | SOTA on MuSiQue when paired with reader |
| Search-o1 Reason-in-Documents denoiser | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 4 | `polaris-skills/research/reason_in_documents.py` | 1 | S | Filters noisy retrievals before injection |
| Co-STORM mind-map as Wiki backbone (graph schema) | [172](172-polaris-2026-deep-research-roadmap.md) §3 Gap 3 | `polaris-core/memory/wiki_graph.py` | 1 | M | Unifies graph schema across modules |
| BELLE bi-level monitor (fast/slow debate watcher) | [202](202-multi-agent-multi-hop-reckoning-2026.md) §1 | `polaris-core/orchestration/debate_monitor.py` | 1 | S | Catches sycophantic agreement |
| Search-R1 RL recipe (GRPO + retrieved-token masking) | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 4 | `polaris-research/rl/search_r1.py` | 2 | L | Closes Gap 1; biggest unrealised lever |
| Tongyi IterResearch "Heavy" mode parallel rollouts | [172](172-polaris-2026-deep-research-roadmap.md) §5 + [156](156-heavyskill-parallel-reasoning-deliberation.md) | `polaris-core/loop/heavy_mode.py` | 2 | M | Test-time scaling on multi-hop |
| ReSearch / R1-Searcher outcome-only RL fine-tune | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 4 | `polaris-research/rl/r1_searcher.py` | 2 | L | RL alternative to Search-R1; lower data needs |
| RAGEN StarPO + Echo-Trap monitor | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 4 | `polaris-research/rl/starpo.py` + `polaris-evals/echo_trap_monitor.py` | 2 | M | Multi-turn RL stability |
| Active-parameter cost accounting | [202](202-multi-agent-multi-hop-reckoning-2026.md) §2 | `polaris-evals/cost_dashboards/active_params.py` | 0 | S | MoE comparisons on multi-hop |
| Equal-budget benchmark harness | [202](202-multi-agent-multi-hop-reckoning-2026.md) §3 | `polaris-evals/benchmarks/equal_budget.py` | 0 | S | Settle multi-agent vs single-agent disputes |
| Pure-function agent pattern (Yenugula side-effect-free) | [202](202-multi-agent-multi-hop-reckoning-2026.md) §4 | `polaris-core/orchestration/pure_function_agent.py` | 0 | S | Replay-safe execution |
| Test-time-compute curve plotter | [199](199-multi-hop-reasoning-techniques-arc.md) + [156](156-heavyskill-parallel-reasoning-deliberation.md) | `polaris-evals/ttc_curve.py` | 0 | S | Detect SealQA-style plateau-or-decline |

## §3 — Tier 0 (days 1-14): the multi-hop hardening surface

The cheapest, highest-payoff lifts. All compose with Polaris's existing modules without touching the executor architecture.

### 0.1 — HippoRAG single-shot multi-hop retrieval

**File.** `polaris-mcp/sources/hipporag.py` + `polaris-graph/multihop_index.py` + `polaris-core/orchestration/multi_hop_retrieve.py`

**What.** Add HippoRAG as a federated source backed by a Polaris-managed graph index. At ingest, run LLM-extracted KG construction and Personalized PageRank weighting. At query time, expose `multi_hop_retrieve(query, top_k=10) -> list[Evidence]` returning trust-tiered, multi-hop-coherent passages in a single pass.

**Integration with existing Polaris surface.**

- **Trust tiers.** Each Evidence row inherits the source's trust tier (T1-PEER-REVIEWED → T3-DISCUSSION). The graph index doesn't bypass trust; it federates within it.
- **ClaimGate.** Every passage emitted by HippoRAG carries its PPR path as provenance — feeds directly into ClaimGate's "is there an evidence row" check.
- **Two-axis evidence gates.** The PPR path becomes a CiteGuard attribution candidate (the path *is* the citation chain) and a Contradiction-to-Consensus candidate (multiple paths = multiple sources to compare).

**Decision.** Use HippoRAG-2 (improved entity linking + learned reranker) as the upstream code; fork only as needed.

### 0.2 — LightRAG incremental ingest

**File.** `polaris-graph/incremental_ingest.py`

**What.** Adopt LightRAG's incremental-update algorithm so adding a new source to a federated corpus doesn't require a full graph reindex. This closes the most common operational complaint about graph-RAG in production.

### 0.3 — BELLE-style question-type router

**File.** `polaris-skills/research/question_type_router.py`

**What.** A small classifier (start with prompted + few-shot, not trained) that types incoming research queries into BELLE's four shapes (bridge / comparison / intersection / decomposition) plus a Polaris-specific fifth (global-sensemaking, where GraphRAG community summarisation wins). Routes the query to the right operator chain.

**Why this not full debate.** [202](202-multi-agent-multi-hop-reckoning-2026.md) §3 (Tran & Kiela): under equal compute, single-agent matches multi-agent. The BELLE *router* keeps the routing gain without the debate overhead — and the bi-level monitor pattern is reserved for edge cases (Tier 1).

**Operators in the bank.** Self-Ask / IRCoT / DSPy-MultiHop / GraphRAG-community / SubQuestion-fanout. Each is a separate skill in `polaris-skills/research/`.

### 0.4 — Chain-of-Note per-doc filter

**File.** `polaris-core/permissions/gates/chain_of_note.py`

**What.** Promote Chain-of-Note to a first-class Polaris gate. After retrieval, before reasoning, the LM writes a per-doc note assessing relevance and contribution. Docs scored irrelevant are dropped from the chain.

**Why a gate, not a skill.** Polaris's gating philosophy ([172](172-polaris-2026-deep-research-roadmap.md) §2) is that quality checks belong in the gate layer where they can fail closed. Chain-of-Note becomes BL-DOC-RELEVANT, joining the existing twenty bright-line gates.

### 0.5 — Self-Ask / IRCoT externalised chains

**File.** `polaris-skills/research/self_ask.py` + `polaris-skills/research/ircot.py`

**What.** Two operator skills exposing the canonical externalised-chain patterns from [199](199-multi-hop-reasoning-techniques-arc.md) Phase 1. **The compositionality gap from [201](201-compositionality-gap-canon.md) is the formal justification** — never let a multi-hop question execute without the chain materialised as tokens. Each emitted sub-question becomes a checkpoint in the Provenance Ledger; each sub-answer is a separately-trustable Evidence row.

### 0.6 — Decomposition cache

**File.** `polaris-core/memory/decomposition_cache.py`

**What.** Memoise sub-question decompositions under a normalised question key. When the same multi-hop question recurs (or a near-duplicate), reuse the decomposition. Sits next to ReasoningBank in `polaris-core/memory/`.

### 0.7 — Multi-hop benchmark triple in nightly CI

**File.** `polaris-evals/benchmarks/multi_hop_triple/{musique_full,frames,sealqa_seal0}/`

**What.** [198](198-multi-hop-qa-datasets-canon.md) §"When to use" recommends MuSiQue-Full + FRAMES + Seal-0 as the minimum regression triple. Bake all three into nightly CI alongside the existing Deep Research Bench.

**Cost cap.** Each benchmark capped at $X/run; the dashboard surfaces $/query alongside accuracy.

### 0.8 — Active-parameter cost accounting

**File.** `polaris-evals/cost_dashboards/active_params.py`

**What.** Steele & Katz ([202](202-multi-agent-multi-hop-reckoning-2026.md) §2) show MoE multi-hop performance correlates with *active*, not total, parameters. Polaris's MoE comparisons must normalise against active params, not total params, to be fair.

### 0.9 — Equal-budget benchmark harness

**File.** `polaris-evals/benchmarks/equal_budget.py`

**What.** Tran & Kiela ([202](202-multi-agent-multi-hop-reckoning-2026.md) §3): when comparing single-agent vs multi-agent multi-hop, control thinking-token budget — otherwise you're benchmarking budget, not architecture. Harness exposes a `--budget` flag enforced across all comparisons.

### 0.10 — Pure-function agent pattern

**File.** `polaris-core/orchestration/pure_function_agent.py`

**What.** Yenugula et al. ([202](202-multi-agent-multi-hop-reckoning-2026.md) §4): every multi-hop sub-agent is **side-effect-free** — deterministic given the same input, replayable, testable. Side-effecting tools (writes, posts, payments) go through gated APIs that log and require explicit BL-* permission grants.

**Why now.** This is foundational for Tier 1's tree-search (you can't replay a non-deterministic agent), so it has to land in Tier 0.

### 0.11 — Test-time-compute curve plotter

**File.** `polaris-evals/ttc_curve.py`

**What.** For each multi-hop benchmark, plot accuracy vs thinking-token budget. SealQA-style noise can make the curve *invert* past a threshold ([199](199-multi-hop-reasoning-techniques-arc.md), [156](156-heavyskill-parallel-reasoning-deliberation.md)). The plotter surfaces the inflection point as the production budget cap.

## §4 — Tier 1 (days 15-45): architecture surface

After Tier 0, the gating surface is hardened and the retrieval substrate is multi-hop-aware. Tier 1 makes the executor *branch-able* and *type-routable*.

### 1.1 — DSPy multi-hop as declarative composition

**File.** `polaris-skills/research/dspy_multihop_program.py`

**What.** Compose Tier 0's operator bank into a DSPy `Module` with a `MultiHop` loop. The DSPy compiler bootstraps few-shot demonstrations against held-out Polaris evals (MuSiQue-Full, FRAMES) and produces a versioned, optimised program checked into `polaris-skills/`.

**Why.** The optimiser-not-prompt mindset ([93-dspy](93-dspy.md)). Multi-hop quality becomes a function of the program, not the prompt — which makes it survive base-model swaps.

### 1.2 — Plan-on-Graph adaptive backtracking

**File.** `polaris-skills/research/branch_and_prune.py` (extends the tree-controller skill scoped in [172](172-polaris-2026-deep-research-roadmap.md) §3 Gap 2)

**What.** Adopt PoG's three mechanisms — Guidance / Memory / Reflection — for any multi-hop chain that walks a graph. When a path is wrong, the chain backtracks; the visited subgraph is logged as part of the Provenance Ledger.

### 1.3 — AnchorRAG anchor-predictor

**File.** `polaris-graph/anchor_predictor.py`

**What.** A learned (or prompted+few-shot) classifier that proposes plausible anchor entities when entity linking is ambiguous. Drops the assumption of perfect entity linking — necessary for any open-world deployment.

### 1.4 — Beam Retrieval for ≥3 hops

**File.** `polaris-graph/beam_retrieval.py`

**What.** Beam Retrieval ([199](199-multi-hop-reasoning-techniques-arc.md) Phase 3) is SOTA when hops > 2. Polaris's deep-research workloads frequently hit 3–4 hops; a beam-search retriever sits between HippoRAG and the reader.

### 1.5 — Reason-in-Documents denoiser

**File.** `polaris-skills/research/reason_in_documents.py`

**What.** Search-o1's denoising step. Before injecting retrieved passages into the reasoning context, a separate LM call summarises and filters. Composes orthogonally with Chain-of-Note (different granularity).

### 1.6 — Co-STORM mind-map Wiki backbone

**File.** `polaris-core/memory/wiki_graph.py`

**What.** Closes [172](172-polaris-2026-deep-research-roadmap.md) §3 Gap 3. The Research Wiki becomes a typed mind-map where every claim, source, and contradiction is a typed node — the same schema `polaris-graph` already uses for citations. Unifies the schema and lets multi-hop chains reuse the Wiki as both *source* and *output*.

### 1.7 — BELLE bi-level monitor

**File.** `polaris-core/orchestration/debate_monitor.py`

**What.** When a multi-hop query escalates from single-agent to debate (rare; gated on confidence), monitor the debate for sycophantic flips with the fast/slow watcher pattern. Reused across any future multi-agent skill.

## §5 — Tier 2 (days 46-90): training surface

Tier 2 closes [172](172-polaris-2026-deep-research-roadmap.md) §3 Gap 1 (prompted, not trained). The prerequisite is a working tournament + reward signals from Tier 1, plus the Tier 0 gating surface as the source of training rewards.

### 2.1 — Search-R1 RL recipe

**File.** `polaris-research/rl/search_r1.py`

**What.** Fork Search-R1's veRL setup, plug in Polaris's BL-* gates as reward shaping. Action tokens `<search>...</search>`, `<answer>...</answer>`. Retrieved-token masking. Outcome rewards: correct-claim-grounded-in-T1 = +1; T3 grounding when T1 reachable = penalty; un-cited claim = penalty; contradicted claim = penalty; retracted source = strong penalty.

**Backbone.** Start with Qwen2.5-7B for the Polaris-tuned executor; consider 32B once the recipe validates.

**Why this and not GPT-5-class API only.** Polaris's distinctive surface (trust tiers, gates) becomes *training signal* in this regime. The proprietary API path is good for capability, but it can't internalise Polaris's discipline. The trained executor is the only way to fold discipline into the model itself.

### 2.2 — Tongyi IterResearch Heavy mode

**File.** `polaris-core/loop/heavy_mode.py`

**What.** Parallel rollouts at inference time (test-time scaling). Run K independent multi-hop chains, score each with the gating surface, select the best. The K-vs-quality curve is plotted and the inflection point becomes the production budget cap.

### 2.3 — ReSearch / R1-Searcher outcome-only RL alternative

**File.** `polaris-research/rl/r1_searcher.py`

**What.** Two-stage outcome-only RL ([199](199-multi-hop-reasoning-techniques-arc.md) Phase 4) without SFT cold-start. Lower data requirements than Search-R1; useful when the gold multi-hop QA set is small. R1-Searcher++'s memory mechanism also internalises retrieved facts over training — a structural fit with Polaris's filesystem-first memory.

### 2.4 — RAGEN StarPO + Echo-Trap monitor

**File.** `polaris-research/rl/starpo.py` + `polaris-evals/echo_trap_monitor.py`

**What.** Multi-turn RL stability ([199](199-multi-hop-reasoning-techniques-arc.md) Phase 4 / [202](202-multi-agent-multi-hop-reckoning-2026.md) caveats). Conditional-entropy monitor detects "Echo Trap" reasoning collapse during training; alarms before the model degrades into boilerplate.

## §6 — How this maps onto the [172] v2.x phasing

| 172 tier | 172 lift | 203 lift |
|---|---|---|
| Tier 0 (v2.2, days 1-14) | CORAL heartbeat scheduler | + HippoRAG + Question-Type Router + Chain-of-Note gate + multi-hop benchmark triple + equal-budget harness + pure-function agents |
| Tier 0 | Two-axis evidence gates | (multi-hop subsystem feeds two-axis gates with stronger evidence chains) |
| Tier 0 | OpenScholar T1 federation | (HippoRAG indexes OpenScholar as a federated T1 source) |
| Tier 0 | Deep Research Bench nightly CI | + MuSiQue-Full + FRAMES + Seal-0 in the same nightly |
| Tier 1 (v2.3, days 15-45) | Tournament + Elo | (multi-hop chains become tournament hypotheses) |
| Tier 1 | Tree-search experiment manager | + PoG backtracking + Beam Retrieval + DSPy multi-hop program + Reason-in-Documents + AnchorRAG |
| Tier 1 | Co-STORM mind-map | (shared schema across `polaris-graph` and `wiki_graph`) |
| Tier 1 | TTD-DR draft-as-state writer | (per-patch ClaimGate now includes per-hop CiteGuard) |
| Tier 2 (v2.4, days 46-90) | IterResearch Heavy mode | + Search-R1 + R1-Searcher + RAGEN StarPO + Echo-Trap monitor |
| Tier 2 | smolagents CodeAgent dispatch | (orthogonal to multi-hop subsystem) |
| Tier 2 | Search-R1 RL recipe | (this is the Tier 2 anchor of the multi-hop subsystem) |

The multi-hop subsystem is **load-bearing for Tier 2** — without HippoRAG-grade retrieval (Tier 0) and DSPy-compiled operator chains (Tier 1), the RL recipe in Tier 2 has nothing to optimise.

## §7 — Five sequencing decisions worth defending

Each is a decision that could go the other way; this section is the rationale.

### Decision 1 — HippoRAG, not full Microsoft GraphRAG, for Tier 0

GraphRAG's community summaries are powerful for global-sensemaking but expensive to ingest and full-reindex required for new docs. HippoRAG amortises multi-hop work into a PPR-weighted KG that costs less per ingest, runs faster per query, and works with online updates when paired with LightRAG's incremental algorithm. Adopt GraphRAG community-detection later as a *secondary* index for global queries (Tier 1, the GraphRAG-community operator in the bank).

### Decision 2 — BELLE *router* without BELLE *debate* in Tier 0

The router gain is the bulk of BELLE's value. The debate adds compute that [202](202-multi-agent-multi-hop-reckoning-2026.md) §3 (Tran & Kiela) shows often disappears under equal-budget control. Defer the debate (and the bi-level monitor) to Tier 1, gated on cases where the router can't decide confidently.

### Decision 3 — Externalise the chain *always*, even when the model could compose

[201](201-compositionality-gap-canon.md) shows the compositionality gap is architectural and scaling-resistant. Multi-hop chains in Polaris always emit Self-Ask / IRCoT-style sub-questions, even when a frontier model might compose silently. The auditability gain (Yenugula et al., [202](202-multi-agent-multi-hop-reckoning-2026.md) §4) compounds with the architectural fix from [201](201-compositionality-gap-canon.md).

### Decision 4 — Search-R1 RL recipe, not Tongyi IterResearch first

Search-R1 is a single-agent recipe; Tongyi IterResearch Heavy mode is multi-agent. [202](202-multi-agent-multi-hop-reckoning-2026.md) §3 says single-agent under equal compute matches multi-agent on multi-hop. Train the single-agent executor first (Search-R1), then add Heavy-mode parallel rollouts as test-time scaling — the order respects the equal-budget rebuttal.

### Decision 5 — Pure-function agents are Tier 0, not Tier 1

You cannot replay a non-deterministic agent, and you cannot RL-train against a non-replayable trajectory. Yenugula's pattern is the foundation for everything in Tier 1 and Tier 2; it ships first.

## §8 — Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Graph-extraction errors poison HippoRAG | Med | High | Periodic LLM-extraction audit; sample N triples / week and judge correctness; alarm on drift. |
| Question-type router misroutes | Low-Med | Med | Fall-through to a default operator chain; log all router decisions; nightly re-eval on held-out classification set. |
| Search-R1 training reward-hacks Polaris's gates | Med | High | Outcome reward = grounded-in-T1; gate signatures co-evaluated by separate model; use AgentPRM-style rubrics for process supervision (Tier 2 sub-task). |
| Echo Trap during RL fine-tune | Med | High | Conditional-entropy + mutual-information monitor (RAGEN-2) in `polaris-evals/echo_trap_monitor.py`; alarm at 1.5σ threshold. |
| Multi-hop chains amplify retrieval noise | High | Med | Chain-of-Note per-doc filter (Tier 0); SealQA-class tests in nightly CI; cap retrievals per hop. |
| DSPy program drifts when base model changes | Med | Med | Recompile DSPy programs nightly against held-out evals; alert on >5pt accuracy regression. |
| Trust-tier metadata gets lost across hops | Med | High | Carry tier as Evidence-row schema field; gate forbids un-tiered Evidence; HippoRAG provenance includes tier per node. |
| Decomposition cache poisons across users / projects | Low | High | Cache key includes project + permissions context; never share decompositions across security boundaries. |
| Heavy mode K-rollout exhausts budget | Med | Med | $/query gate before K runs; auto-cap K based on Tier 0 cost dashboard. |
| Multi-hop subsystem regresses single-hop performance | Low-Med | Med | Single-hop benchmark held in nightly CI alongside multi-hop triple; alarm on regression. |

## §9 — Concrete day-by-day checklist for Tier 0

A 14-day Tier 0 with daily granularity. Owners and exact effort estimates remain to be assigned.

- **Day 1-2** — Stand up `polaris-graph/multihop_index.py` skeleton; benchmark HippoRAG-2 against a slice of Polaris's existing federated corpus.
- **Day 3-4** — Wire `polaris-mcp/sources/hipporag.py` into the existing MCP source registry; verify trust-tier passthrough.
- **Day 5** — Add LightRAG incremental ingest path in `polaris-graph/incremental_ingest.py`.
- **Day 6** — Implement `polaris-skills/research/question_type_router.py` (prompted + 5-shot) and the operator bank stubs.
- **Day 7** — Promote Chain-of-Note to BL-DOC-RELEVANT in `polaris-core/permissions/gates/`.
- **Day 8** — Implement `polaris-skills/research/self_ask.py` and `ircot.py` operators; integrate with Provenance Ledger.
- **Day 9** — `polaris-core/memory/decomposition_cache.py`, with a normalised-question-key hash and TTL policy.
- **Day 10** — Bake MuSiQue-Full + FRAMES + Seal-0 into `polaris-evals/benchmarks/multi_hop_triple/`; first nightly run.
- **Day 11** — `polaris-evals/cost_dashboards/active_params.py` and `polaris-evals/benchmarks/equal_budget.py`.
- **Day 12** — `polaris-core/orchestration/pure_function_agent.py` base class; refactor the existing executor to extend it.
- **Day 13** — `polaris-evals/ttc_curve.py` plotter; add the inflection-point flag to the budget gate.
- **Day 14** — Tier 0 retro: which gates fired most? which question types route most? what's the multi-hop accuracy / cost on the triple? sign-off for Tier 1.

## §10 — Cross-references

- [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md) — the v2.2/v2.3/v2.4 phasing this plan tracks.
- [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md) — skills-as-files philosophy that the operator bank inherits.
- [167-autoskill-experience-driven-lifelong-learning](167-autoskill-experience-driven-lifelong-learning.md) — pattern for promoting recurring multi-hop chains into reusable skills.
- [166-research-agent-frameworks-synthesis](166-research-agent-frameworks-synthesis.md) — predecessor synthesis on research-agent frameworks.
- [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md) — the parallel-rollout / Heavy mode line that Tier 2 builds on.
- [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md), [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md) — graph substrate that HippoRAG plugs into.
- [188-witness-provenance-memory-techniques-synthesis](188-witness-provenance-memory-techniques-synthesis.md) — provenance philosophy that Self-Ask checkpoints feed.
- [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md), [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md) — production substrate for the pure-function agent pattern.
- [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [115-evaluating-llm-systems](115-evaluating-llm-systems.md) — eval discipline for the multi-hop benchmark triple.

## §11 — One-paragraph summary

Polaris's multi-hop subsystem ships in three tiers: Tier 0 (14 days) hardens the retrieval surface with HippoRAG + LightRAG + Chain-of-Note + question-type routing + always-emit-the-chain operators (Self-Ask, IRCoT) + the MuSiQue-Full / FRAMES / Seal-0 nightly + equal-budget benchmarks + pure-function agents; Tier 1 (30 days) makes the executor branch-able with DSPy-compiled operator chains + Plan-on-Graph backtracking + AnchorRAG entity linking + Beam Retrieval + Co-STORM mind-map Wiki + BELLE monitor; Tier 2 (60 days) trains the executor with Search-R1 + IterResearch Heavy mode + R1-Searcher + RAGEN StarPO. The five sequencing decisions defended in §7 turn the multi-hop reasoning literature into a Polaris-shaped roadmap that respects the existing distinctive surface (trust tiers, gates, provenance, multi-domain shells) and makes those disciplines training signals when the executor finally trains in Tier 2.

**The one-line takeaway for harness designers:** Apply multi-hop techniques to Polaris by making each technique a pluggable module that *feeds* the existing gating / trust / provenance surface — never bypasses it — and stage the work so that each tier's outputs become the next tier's training signal.
