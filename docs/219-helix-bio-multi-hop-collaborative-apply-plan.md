# 219 — Helix-Bio Apply Plan: Biomedical Research with Retraction-Aware Safety + Dual-Use as First-Class

> **Disambiguation.** This file is the **Helix-Bio–specific** deep apply plan, extending the per-project apply-plan family ([203](203-polaris-multi-hop-reasoning-apply-plan.md), [208](208-lyra-multi-hop-collaborative-apply-plan.md), [209](209-argus-multi-hop-collaborative-apply-plan.md), [210](210-mentat-learn-collaborative-apply-plan.md), [218](218-atlas-research-multi-hop-collaborative-apply-plan.md)). The cross-project matrix is [207](207-cross-project-collaborative-multi-hop-apply-plan.md); the executable strategy doc is [211](211-cross-project-power-up-plan-with-tradeoffs.md). Helix-Bio's role in the portfolio: **biomedical research** with retraction-aware safety, dual-use risk as a first-class design concern, and structured-KG fusion (UniProt / PDB / AlphaFold) on top of literature retrieval.

## One-line definition

A staged plan to fold multi-hop reasoning + collaborative-AI techniques into Helix-Bio — the **biomedical research agent** baselined against GPT-Rosalind / RadAgent / generalist LLM + PubMed-RAG, evaluated on BixBench / LABBench2 / CloningQA / domain-specific F1 — with the design target of faithfulness-gated outputs against UniProt / PDB / AlphaFold and **dual-use safety as first-class** (not bolted on), making Helix the project where domain-specialised gates, retraction awareness, and bench-validated tool use compose with the multi-hop substrate.

## §1 — Why apply these techniques to Helix-Bio

Helix-Bio is the in-tree project where **the consequences of getting it wrong are highest**. A biomedical research agent that:

- Cites a retracted paper as load-bearing evidence damages a real patient or experiment.
- Helps synthesise a pathogen variant or a controlled-substance analog crosses the dual-use bright line.
- Misuses a PubChem / UniProt / AlphaFold lookup misleads a researcher who *believes* the agent.

This shapes the apply plan more than any other in-tree project. Where Atlas-Research relaxes gates for throughput, Helix-Bio *adds gates beyond Polaris's*: retraction-aware retrieval, dual-use intent classification, KG-grounded fact gating, IRB / ethics gates per domain shell, and HITL approval for any computation that touches a regulated substance class. The multi-hop substrate is the same as Polaris ([203](203-polaris-multi-hop-reasoning-apply-plan.md)) and Atlas ([218](218-atlas-research-multi-hop-collaborative-apply-plan.md)); the gate surface is *richer*.

The collaborative-AI canon also fits Helix unusually well. The user is a researcher with a stable identity (lab, organism, assay focus) — ICPEA Identity layer is high-signal. Their preferences are sharp (which databases, which papers, which protocols) — Preference layer matters. Their experience is rich (what worked in the wet lab last month) — Experience layer carries enormous value. Voice + screen-share is on-niche because researchers want to *show the agent* a gel image, an FPLC trace, an Excel sheet — not just describe them.

Take this plan seriously and three things change. (1) Helix becomes the in-tree project where retraction-aware gates + dual-use classification ship as production primitives ready to be lifted to Polaris's biomed shell. (2) The KG layer expands beyond Polaris's literature-only graph to include structural KGs (UniProt / PDB / AlphaFold). (3) Voice + screen-share shipping in Helix establishes the multimodal-research-teammate pattern for the portfolio.

## §2 — The Helix-Bio × technique mapping table

Helix-Bio's existing module structure is `src/helix_bio/`, with vendored `harness_core/`, `skills/`, `tests/`, `docs/`. The mapping is broader than Atlas's because Helix has a domain-specialised gate surface.

| Technique | Source | Helix module | Tier | Effort | Payoff |
|---|---|---|---|---|---|
| **Multi-hop substrate** | | | | | |
| HippoRAG-2 over PubMed + UniProt + PDB + AlphaFold | [200](200-graph-grounded-multi-hop-retrieval.md), promoted from `polaris-core` per [211](211-cross-project-power-up-plan-with-tradeoffs.md) | `harness_core/multi_hop/hipporag.py` (vendored) | 0 | M | Multi-hop biomed retrieval across heterogeneous KGs |
| Structured KG ingest: UniProt / PDB / AlphaFold | new | `src/helix_bio/graph/{uniprot,pdb,alphafold}_ingest.py` | 0 | M | Structural data fused with literature graph |
| LightRAG incremental ingest for new papers | [200](200-graph-grounded-multi-hop-retrieval.md) | `src/helix_bio/graph/incremental.py` | 0 | S | Online ingest |
| BELLE-style biomed-question router | [202](202-multi-agent-multi-hop-reckoning-2026.md) §1 | `src/helix_bio/routing/biomed_router.py` | 0 | M | Different operators for "what does X gene do" vs "what's the binding affinity" vs "what protocol works for Y" |
| Chain-of-Note per-paper filter | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 3 | `harness_core/gates/chain_of_note.py` | 0 | S | Per-paper relevance gate |
| Self-Ask / IRCoT externalised chain | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 1 | `src/helix_bio/reasoning/{self_ask,ircot}.py` | 0 | S | Externalises bridge entity (the protein, the pathway, the assay) |
| DSPy multi-hop program for biomed | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 2 | `skills/research/dspy_helix_program.py` | 1 | M | Compiled-against-evals biomed pipeline |
| Plan-on-Graph for pathway / interaction graphs | [200](200-graph-grounded-multi-hop-retrieval.md) | `src/helix_bio/graph/plan_walk.py` | 1 | M | Adaptive backtracking over biological pathway graphs |
| AnchorRAG for ambiguous gene / protein names | [200](200-graph-grounded-multi-hop-retrieval.md) | `src/helix_bio/graph/anchor_predictor.py` | 1 | M | Robust to imperfect entity linking ("BRCA" → BRCA1 vs BRCA2) |
| Beam Retrieval for ≥3 hop biomed questions | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 3 | `src/helix_bio/graph/beam_retrieval.py` | 1 | M | SOTA on protein-pathway-disease chains |
| Reason-in-Documents on retrieved abstracts | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 4 | `src/helix_bio/reasoning/reason_in_papers.py` | 1 | S | Filters noisy retrievals before injection |
| Search-R1 RL with biomed-grounded reward | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 4 | `src/helix_bio/rl/search_r1.py` | 2 | L | Trained executor with retraction-aware reward shaping |
| **Domain-specialised gates (Helix's distinctive surface)** | | | | | |
| Retraction-aware retrieval gate | JMIR 2026 (cf. [172](172-polaris-2026-deep-research-roadmap.md) §3 Gap 5) | `src/helix_bio/gates/retraction_gate.py` | 0 | M | Retracted papers fail closed at retrieval, not LLM-time |
| Dual-use intent classifier | new | `src/helix_bio/gates/dual_use_classifier.py` | 0 | M | Classify queries about controlled-substance / pathogen synthesis; HITL gate |
| KG-grounded fact gate | new | `src/helix_bio/gates/kg_fact_gate.py` | 1 | M | Numerical / structural claims must be grounded in UniProt / PDB / AlphaFold |
| IRB / ethics gate per domain shell | new (cf. [172](172-polaris-2026-deep-research-roadmap.md)) | `src/helix_bio/gates/irb_ethics_gate.py` | 1 | M | Human-subjects research queries route through ethics review |
| Two-axis evidence gate (CiteGuard + Contradiction-to-Consensus) | [172](172-polaris-2026-deep-research-roadmap.md) §3 Gap 5 | `src/helix_bio/gates/two_axis_evidence.py` | 1 | M | CiteGuard attribution + multi-source agreement |
| **Collaborative-AI substrate** | | | | | |
| ICPEA five-layer personal memory | [205](205-lobehub-collaborative-teammate-platform.md) §2.1 | `src/helix_bio/memory/icpea/` | 1 | M | Per-researcher organism / assay / chemistry preferences |
| Async memory extractor on session-finalize | [206](206-collaborative-ai-canon-2026.md) §1 | `src/helix_bio/memory/icpea/extractor.py` | 1 | S | Decouples memory write from response latency |
| Per-agent layer access control | [205](205-lobehub-collaborative-teammate-platform.md) §2.1 | `src/helix_bio/memory/icpea/access.py` | 1 | S | IRB-grade subagents see Identity but not Experience |
| Branching as alternative-hypothesis UX | [206](206-collaborative-ai-canon-2026.md) §4 | `src/helix_bio/conversation/branching.py` | 1 | M | "Three plausible mechanisms — explore each in parallel" |
| Lobe Pages-style co-authored protocol / paper | [205](205-lobehub-collaborative-teammate-platform.md) §2.2 | `src/helix_bio/output/pages.py` | 1 | M | Co-authored protocol document with timeline |
| MCP-first marketplace via argus trust gate | [205](205-lobehub-collaborative-teammate-platform.md) §2.3, [209](209-argus-multi-hop-collaborative-apply-plan.md) | `src/helix_bio/tools/mcp_marketplace.py` | 1 | S | Trust-gated biomed tool installation (PubChem, RDKit, AlphaFold MCP, etc.) |
| Voyager-line skill auto-creation for protocols | [167](167-autoskill-experience-driven-lifelong-learning.md), [197](197-argus-omega-vol-3-recursive-skills-curator.md) | `src/helix_bio/skills/protocol_auto_creator.py` | 2 | M | Successful protocols become reusable; HITL gate on biomed promotion |
| Per-user constitution | [206](206-collaborative-ai-canon-2026.md) §5 | `src/helix_bio/constitution/` | 2 | M | Researcher's lab principles + dual-use stance |
| **Voice + multimodal (high-priority for Helix)** | | | | | |
| Voice + screen-share with VLM (GLM-4.5V / Qwen3-VL) | [206](206-collaborative-ai-canon-2026.md) §6 | `src/helix_bio/channels/voice/` + `src/helix_bio/channels/vision/` | 1 | L | "Show me this gel image / cryo-EM map / FPLC trace and discuss" |
| **Cross-cutting discipline** | | | | | |
| Pure-function agents | [202](202-multi-agent-multi-hop-reckoning-2026.md) §4 | `harness_core/orchestration/pure_function.py` | 0 | S | Replayable trajectories; HITL replay |
| Equal-budget benchmark harness | [202](202-multi-agent-multi-hop-reckoning-2026.md) §3 | `harness_core/evals/equal_budget.py` | 0 | S | Honest baselines vs GPT-Rosalind |
| TTC curve plotter | [199](199-multi-hop-reasoning-techniques-arc.md) | `harness_core/evals/ttc_curve.py` | 0 | S | Detect plateau-or-decline |
| BixBench + LABBench2 + CloningQA in CI | [projects/README.md](../projects/README.md) | `tests/benchmarks/{bixbench,labbench2,cloningqa}.py` | 0 | S | Domain-specific regression triple |
| MuSiQue + FRAMES + Seal-0 (general multi-hop) | [198](198-multi-hop-qa-datasets-canon.md) | `tests/benchmarks/multi_hop_triple/` | 0 | S | General multi-hop regression |
| gnomon HIR emit | [211](211-cross-project-power-up-plan-with-tradeoffs.md) §4 | `src/helix_bio/observability/hir_emit.py` | 0 | S | Cross-harness eval signal |

## §3 — Tier 0 (days 1-14): the retraction-aware substrate

Helix's Tier 0 differs from Atlas/Polaris in a critical way: **retraction-aware retrieval gate + dual-use classifier ship in days 1-14**, not later. These are non-negotiable for biomed.

### 0.1 — Retraction-aware retrieval gate

**File.** `src/helix_bio/gates/retraction_gate.py`.

**What.** From the JMIR 2026 finding ([172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md) §3 Gap 5): retracted papers must be filtered at the *retrieval* layer, not at the LLM-prompt layer. Subscribe to RetractionWatch, PubMed retraction notices, and journal-specific retraction databases. Every paper retrieved is checked against the retraction index; retracted papers are flagged in the Evidence row with `BL-RETRACTED` and excluded from chain-of-note injection.

**Why Tier 0.** A retracted paper cited in a biomed report is the kind of failure that ends an agent's deployment.

### 0.2 — Dual-use intent classifier

**File.** `src/helix_bio/gates/dual_use_classifier.py`.

**What.** A classifier (start prompted + few-shot, train later) that types incoming queries on a dual-use risk axis: pathogen-of-concern synthesis, controlled-substance analog design, gain-of-function-adjacent assays, etc. High-risk queries route to HITL approval; medium-risk queries get a hard rate limit + audit log; low-risk queries flow normally. Cf. [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md), [122-explainability-compliance](122-explainability-compliance.md).

**Why Tier 0.** Dual-use is not a Tier-2 feature. It must ship before any deployment.

### 0.3 — HippoRAG-2 over PubMed + structured KGs

**File.** `harness_core/multi_hop/hipporag.py` (vendored from shared library).

**What.** Helix's KG is heterogeneous: literature graph (PubMed) + structural KG (UniProt / PDB / AlphaFold). The HippoRAG-2 PPR fusion handles literature; the structured KGs are first-class graph nodes the literature graph links into.

### 0.4 — Structured KG ingest

**Files.** `src/helix_bio/graph/{uniprot,pdb,alphafold}_ingest.py`.

**What.** Pull UniProt / PDB / AlphaFold entries as typed nodes in the Helix graph. Each protein, each structure, each predicted fold is a node. PubMed papers link to these nodes by entity-mention extraction. This is what makes Helix's multi-hop substrate *biomed-aware* rather than generic.

### 0.5 — BELLE-style biomed-question router

**File.** `src/helix_bio/routing/biomed_router.py`.

**What.** Classifier types queries into biomed-specific shapes:
- **Functional** — what does X gene do? (literature primary; KG secondary)
- **Structural** — protein structure / binding site (PDB / AlphaFold primary)
- **Pharmacological** — binding affinity / IC50 (databases primary; literature secondary)
- **Methodological** — what protocol works for Y? (recipe-shaped; literature primary)
- **Diagnostic / clinical** — IRB-gated; routes through ethics gate

### 0.6 — Self-Ask / IRCoT + Chain-of-Note + decomposition cache

Standard from [199](199-multi-hop-reasoning-techniques-arc.md), shipped here for the same reasons as in Polaris / Atlas.

### 0.7 — Pure-function agents + equal-budget + TTC + benchmark suite

Cross-cutting Tier 0 discipline per [211](211-cross-project-power-up-plan-with-tradeoffs.md). BixBench + LABBench2 + CloningQA + MuSiQue + FRAMES + Seal-0 nightly.

### 0.8 — gnomon HIR emit

Helix's traces are higher-stakes than most projects' (biomed claims, dual-use audit log) — gnomon HIR emit is the cross-harness eval substrate that lets vertex-eval run regression suites against Helix's history.

## §4 — Tier 1 (days 15-45): the architecture surface

### 4.1 — DSPy multi-hop program for biomed

**File.** `skills/research/dspy_helix_program.py`.

**What.** Compose Tier 0's operators. Compile against held-out evals: BixBench + LABBench2 subset + CloningQA. The compiled program is the canonical Helix research pipeline.

### 4.2 — Plan-on-Graph + AnchorRAG + Beam Retrieval for biomed graphs

**Files.** `src/helix_bio/graph/{plan_walk,anchor_predictor,beam_retrieval}.py`.

**What.** Three composable graph-walk improvements:
- **PoG.** Backtrack when a pathway-walk leads to wrong subgraph.
- **AnchorRAG.** Predict plausible anchors for ambiguous gene / protein names ("BRCA" → BRCA1 / BRCA2 / etc.).
- **Beam Retrieval.** SOTA on protein-pathway-disease 3+ hop chains.

### 4.3 — Reason-in-Documents on retrieved abstracts

**File.** `src/helix_bio/reasoning/reason_in_papers.py`.

**What.** Search-o1's denoising step adapted for biomed papers. Refines abstracts before injection (especially useful for pre-print servers where quality varies).

### 4.4 — KG-grounded fact gate + IRB/ethics gate + two-axis evidence gate

**Files.** `src/helix_bio/gates/{kg_fact_gate,irb_ethics_gate,two_axis_evidence}.py`.

**What.** Three additional gates beyond Polaris's standard set:
- **KG-grounded fact gate.** Numerical claims (binding affinity, IC50, fold accuracy) must cite UniProt / PDB / AlphaFold or be flagged.
- **IRB / ethics gate.** Human-subjects-related queries route through an ethics review skill before retrieval.
- **Two-axis evidence gate.** CiteGuard attribution alignment + Contradiction-to-Consensus inter-source agreement (cf. [172](172-polaris-2026-deep-research-roadmap.md) §3 Gap 5).

### 4.5 — ICPEA personal memory

**File.** `src/helix_bio/memory/icpea/`.

**What.** Standard ICPEA tuned for biomed:
- **I**dentity — "I'm a postdoc in a structural biology lab studying GPCR pharmacology."
- **C**ontext — current project, organism, assay, deadline.
- **P**reference — preferred databases (UniProt vs ChEMBL), citation style, depth.
- **E**xperience — what worked in past wet-lab attempts; what failed.
- **A**ctivity — queries, papers cited, protocols saved.

### 4.6 — Branching + Pages + MCP marketplace

Standard collaborative-AI surface — branching for alternative-hypothesis exploration; Pages for co-authored protocols; argus marketplace for biomed MCP servers (PubChem, RDKit, AlphaFold MCP).

### 4.7 — Voice + screen-share with VLM

**Files.** `src/helix_bio/channels/voice/` + `src/helix_bio/channels/vision/`.

**What.** Voice + screen-share is **Tier 1, not Tier 2** in Helix because researchers want to *show the agent* a gel image, an FPLC trace, an Excel sheet — describing them in text loses information. GLM-4.5V or Qwen3-VL handles the vision side; OpenAI Realtime handles voice; ICPEA Context layer captures *what the user is currently looking at*.

## §5 — Tier 2 (days 46-90): training surface + protocol auto-creation

### 5.1 — Search-R1 RL with biomed-grounded reward

**File.** `src/helix_bio/rl/search_r1.py`.

**What.** Reward function:
- Correct answer grounded in non-retracted T1 source = +1
- Retracted source cited = strong penalty
- Dual-use intent triggered without HITL = strong penalty
- KG-grounded fact unsupported = penalty
- BL-* gate violation = penalty

The reward is *richer* than Polaris's because it includes biomed-specific signals.

### 5.2 — Voyager-line protocol auto-creation

**File.** `src/helix_bio/skills/protocol_auto_creator.py`.

**What.** Successful protocols (multi-step procedures the researcher uses repeatedly) are auto-extracted as reusable skills, gated by argus's curator + a HITL biomed reviewer. Promoted protocols land in `skills/protocols/` with full provenance.

### 5.3 — Per-user constitution

**File.** `src/helix_bio/constitution/`.

**What.** Researcher's lab principles + dual-use stance + IRB compliance defaults. Examples:
- "Always cite peer-reviewed primary sources for mechanism claims."
- "Never recommend protocols involving Schedule I substances."
- "Default to UniProt over RCSB for protein metadata."

## §6 — Five sequencing decisions worth defending

### Decision 1 — Retraction-aware gate + dual-use classifier in Tier 0

These ship in days 1-14, not later. Biomed without these is a non-starter.

### Decision 2 — Structured KG (UniProt / PDB / AlphaFold) is first-class

Helix's multi-hop substrate is heterogeneous-by-design. Literature alone is insufficient.

### Decision 3 — Voice + screen-share Tier 1, not Tier 2

Researchers think with images (gels, traces, structures, plots). Text-only is a productivity ceiling.

### Decision 4 — Helix has *more* gates than Polaris, not fewer

Atlas relaxes Polaris's gates; Helix adds three more (KG-fact, IRB/ethics, two-axis evidence) plus retraction + dual-use. The total gate surface is the richest in the portfolio.

### Decision 5 — Protocol auto-creation HITL-gated

Skill auto-promotion in biomed needs human review. Argus's curator surrogate verifier is necessary but not sufficient; a biomed reviewer signs off on protocol promotion.

## §7 — Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Retracted paper cited as load-bearing evidence | Med-High (without gate) → Low (with gate) | Critical | Tier-0 retraction gate; weekly re-check of cited papers in past reports |
| Dual-use query bypasses classifier | Low-Med | Critical | Layered classifier (prompted + few-shot + small LM); audit log; HITL on uncertain calls |
| KG-grounded fact gate false-positives reject valid claims | Med | Med | Tunable threshold; reviewer override path; calibration on held-out set |
| Structured KG (UniProt / PDB) version drift | Med | Med | Version-pinned KG snapshots; nightly validation against current versions; change-log alerts |
| Voice + screen-share latency exceeds threshold | Med | Med | OpenAI Realtime / Gemini Live tuning; ICPEA Context cached at session-warm |
| ICPEA Experience layer leaks confidential lab data | Low | High | Per-lab namespace; per-agent layer access; encryption at rest |
| Protocol auto-creator promotes unsafe procedure | Med | High | HITL biomed reviewer + surrogate verifier + dry-run validation in simulation before promotion |
| Search-R1 reward-hacks on dual-use | Low-Med | Critical | Held-out adversarial-prompt eval; sample-based human review of all dual-use trajectories |
| HippoRAG entity linking conflates BRCA1 / BRCA2 | Med | Med | AnchorRAG anchor-predictor (Tier 1); per-canonical-name disambiguation table |
| IRB gate over-blocks legitimate research | Med | Low-Med | Tunable threshold; reviewer override path |

## §8 — Concrete day-by-day Tier 0 checklist

A 14-day Tier 0 plan:

- **Day 1-2** — `harness_core/orchestration/pure_function.py` adopted; agent loop refactored.
- **Day 3** — `src/helix_bio/gates/retraction_gate.py`; subscribe to RetractionWatch + PubMed.
- **Day 4** — `src/helix_bio/gates/dual_use_classifier.py` first cut (prompted + few-shot).
- **Day 5** — `harness_core/multi_hop/hipporag.py` vendored; first ingest of PubMed sample.
- **Day 6-7** — `src/helix_bio/graph/{uniprot,pdb,alphafold}_ingest.py`; integrate structured KGs.
- **Day 8** — `harness_core/gates/chain_of_note.py` wired.
- **Day 9** — `src/helix_bio/routing/biomed_router.py` BELLE-style.
- **Day 10** — `src/helix_bio/reasoning/{self_ask,ircot}.py` operators.
- **Day 11-12** — `tests/benchmarks/{bixbench,labbench2,cloningqa}.py` + multi-hop triple.
- **Day 13** — `harness_core/evals/{equal_budget,ttc_curve,active_params}.py` + gnomon HIR emit.
- **Day 14** — Tier 0 retro: BixBench / LABBench2 / CloningQA baselines published; retraction gate rejection rate measured; dual-use classifier accuracy on held-out adversarial set; sign-off for Tier 1.

## §9 — How Helix-Bio relates to the rest of the portfolio

| Project | Niche | Distinctive surface | Multi-hop substrate |
|---|---|---|---|
| **Polaris** | General deep research, discipline-grade | Twenty BL-* gates, typed trust tiers | Full multi-hop + RL contract |
| **Atlas-Research** | General deep research, throughput | Chain-of-Note only | Same, simpler reward |
| **Helix-Bio** | Biomedical research | Retraction + dual-use + KG-fact + IRB + two-axis evidence | Same + structured KG ingest |
| **Lyra** | CLI coding | Hooks + permissions | Code-search-shaped |
| **Mentat-Learn** | Personal assistant | ICPEA + SOUL.md | Sparingly applied |

Helix shares the multi-hop substrate with Polaris / Atlas / Lyra (HippoRAG-2 PPR fusion from shared library) and the collaborative-AI substrate with Polaris / Atlas / Mentat / Lyra (ICPEA, branching, Pages, argus marketplace, voice). What's distinct is the **gate surface**: Helix has the richest gate surface in the portfolio because biomed has the highest stakes.

## §10 — Cross-references

- [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md), [218-atlas-research-multi-hop-collaborative-apply-plan](218-atlas-research-multi-hop-collaborative-apply-plan.md) — sibling research plans.
- [211-cross-project-power-up-plan-with-tradeoffs](211-cross-project-power-up-plan-with-tradeoffs.md) — strategy doc.
- [28-radagent-agentic-radiology](28-radagent-agentic-radiology.md), [30-gpt-rosalind-domain-specialized](30-gpt-rosalind-domain-specialized.md), [33-dnahnet-genomic-foundation](33-dnahnet-genomic-foundation.md) — biomed-specialised agent canon Helix's design follows.
- [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md), [122-explainability-compliance](122-explainability-compliance.md) — dual-use / safety lineage.
- [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md), [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md), [200-graph-grounded-multi-hop-retrieval](200-graph-grounded-multi-hop-retrieval.md) — structured-KG context.
- [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md) — argus integration substrate.

## §11 — One-paragraph summary

Helix-Bio's multi-hop + collaborative-AI subsystem ships in three tiers: Tier 0 (14 days) hardens the **retraction-aware substrate** with retraction gate + dual-use classifier + HippoRAG-2 over PubMed + structured KG ingest (UniProt / PDB / AlphaFold) + BELLE biomed-question router + Chain-of-Note + Self-Ask/IRCoT + pure-function agents + equal-budget benchmarks + BixBench/LABBench2/CloningQA nightly + gnomon HIR emit; Tier 1 (30 days) adds DSPy multi-hop program + Plan-on-Graph + AnchorRAG (gene-name disambiguation) + Beam Retrieval + Reason-in-Documents + KG-fact / IRB / two-axis evidence gates + ICPEA + branching as alternative-hypothesis UX + Lobe Pages-style co-authored protocols + argus trust-gated MCP marketplace + voice + screen-share with VLM (Tier 1 priority because researchers think with images); Tier 2 (60 days) trains the executor with biomed-grounded Search-R1 + HITL-gated protocol auto-creation + per-user constitution. The five sequencing decisions defended in §6 keep retraction + dual-use as Tier-0 non-negotiables, structured KGs as first-class, voice + screen-share as Tier-1 (not Tier-2), and the gate surface as the richest in the portfolio because biomed has the highest stakes.

**The one-line takeaway for harness designers:** Helix-Bio is the in-tree project where **retraction + dual-use + KG-grounded fact + IRB ethics gates ship in Tier 0**, where structured biomed KGs (UniProt / PDB / AlphaFold) are first-class graph nodes alongside literature, and where voice + screen-share with VLM is Tier-1 because researchers think with images — adopt the multi-hop substrate from Polaris's shared library, but layer biomed's distinctive gate surface and structural KGs on top.
