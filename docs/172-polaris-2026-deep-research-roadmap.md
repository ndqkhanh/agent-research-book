# 172 — Polaris in May 2026: Deep-Research-Agent Landscape Synthesis

**Scope.** A Polaris-mapped synthesis of the 2025-2026 deep-research-agent landscape across three independent surfaces: frontier papers (Jan 2025 – May 2026), high-impact open-source repositories, and public benchmarks. This chapter complements [166-research-agent-frameworks-synthesis](166-research-agent-frameworks-synthesis.md) by adding the *post-166 wave* — Tongyi DeepResearch, AI-Scientist-v2, Co-Scientist, Search-R1 / WebSailor / WebDancer, CORAL, CiteGuard, OpenScholar, and the AstaBench / DeepResearch-Bench / BrowseComp-Plus evaluation triple — and explicitly turns each finding into a Polaris adoption decision.

**One-paragraph thesis.** Polaris already wins on *discipline* — structural provenance, ClaimGate, twenty bright-line gates, multi-source typed federation with trust tiers, persistent filesystem memory, daemon idle policy, multi-domain shells. None of the 2025-2026 OSS systems have all of these. Polaris is *behind* on five mechanical levers: it is **prompted** not RL-trained (Tongyi / Search-R1), **single-threaded** not tree-searched (Sakana / IterResearch), **flat-markdown** not typed-mind-map (Co-STORM), **opportunistic** not heartbeat-scheduled (CORAL), and runs a **one-axis** evidence gate where the literature has converged on **two-axis** (CiteGuard attribution-alignment × Contradiction-to-Consensus inter-source agreement). The synthesis is in the table; the adoption sequence follows.

## §1 — The convergence — findings that appear in two or more streams

| Finding | Frontier paper | OSS repo | Benchmark | Verdict for Polaris |
|---|---|---|---|---|
| **Tongyi DeepResearch (Alibaba)** | WebSailor V1/V2 (arXiv 2507.02592 / 2509.13305), WebDancer (arXiv 2505.22648) | 18.8k★ at [Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch); WebShaper, AgentFold, ParallelMuse | Owns SOTA on HLE / BrowseComp / FRAMES / SimpleQA | The system to beat |
| **Sakana AI-Scientist-v2** | TRACK (arXiv 2504.08066) | VENDOR controller (6.1k★) | PaperBench reference | Pattern to lift |
| **Search-R1 + GRPO** | PILOT (arXiv 2503.09516) | 4.6k★ — STUDY | Underpins BrowseComp-Plus baselines | RL recipe for executor |
| **AgentRxiv / Agent Laboratory** | TRACK→PILOT (arXiv 2503.18102) | STUDY | — | When Polaris fleets |
| **OpenScholar** | ADOPT (arXiv 2411.14199, Nature 2026) | 45M-paper datastore | Underlies AstaBench/ScholarQA | Drop-in T1 federation |
| **Deep Research Bench** | — | VENDOR (LangChain `open_deep_research`) | COMPETE — top priority (RACE+FACT) | Daily CI eval |

The diagonal pattern is significant: Tongyi DeepResearch shows up as papers, repo, and SOTA-holder simultaneously; Sakana shows up as paper + repo + reference; Search-R1 shows up as paper + repo. **Independent streams agreed on the same load-bearing systems** — that's the credibility check.

## §2 — Polaris's distinctive surface (where it already wins)

Verified against `projects/polaris/packages/`:

| Polaris module | Distinctive capability | What no OSS competitor reproduces |
|---|---|---|
| `polaris-trust` | Typed trust tiers (T1-PEER-REVIEWED / T2-PREPRINT / T3-DISCUSSION / RED-RETRACTED) on every Evidence row | Tongyi has no trust layer; STORM is over-confident; gpt-researcher returns un-tiered citations |
| `polaris-core/permissions` | 20 bright-line gates including PHI / IRB / Embargo / Anonymity / ProofImportClosure | LangChain deepagents has *no* gates; Biomni admits "executes code with full system privileges" |
| `polaris-core/memory` | Filesystem-first Program Graph + Research Wiki + ReasoningBank + Provenance Ledger that survives restarts | smolagents is stateless across runs; Sakana lacks structural provenance |
| `polaris-domains` | Pluggable shells (ML / biomed / math / physics / social / eng) each with its own ethics gate | Biomni is biomed-only; Sakana is ML-only |
| `polaris-daemon` | Long-running daemon with budget caps, retry policy, idle policies | Tongyi/Sakana are batch-style; Ralph is single-threaded loop |
| `polaris-skills/auto_creator` | Self-evolving skills via repeating-pattern detection + held-out eval gate + BL-PROMOTE-SKILL | Most OSS ships *static* skill sets; AutoSkill family ([167-171](171-skill-self-evolution-2026-synthesis.md)) is the one place where Polaris and the literature already align |

These are not theoretical advantages — they are the reason Polaris would score on **AstaBench's lit-understanding category, DeepResearch Bench's FACT axis, and BrowseComp-Plus's citation-accuracy axis** in ways that frontier-model-plus-thin-harness systems cannot.

## §3 — What Polaris is behind on — the five mechanical gaps

### Gap 1 — Prompted, not RL-trained

Polaris's executor is purely policy-driven. **Tongyi DeepResearch + Search-R1** prove the ceiling is much higher with end-to-end **GRPO with retrieved-token masking and outcome-only reward** (correct-claim-grounded-in-T1 source = +1; T3 grounding when T1 was reachable = penalty). This is the single biggest unrealised lever — months of work, but the eventual delta is the difference between "best harness" and "best system."

### Gap 2 — Single-threaded, not tree-searched

Polaris's daemon loop is essentially linear. **Sakana AI-Scientist-v2** (`bfts_config.yaml` parameterising `num_workers`, `steps`, debug-retry per failing node) and **Tongyi's IterResearch "Heavy" mode** (parallel rollouts + selection) show that branching the executor and pruning failing branches beats a single loop. Polaris should add a tree-controller skill with budget-aware pruning, gated by ClaimGate at every commit point.

### Gap 3 — Flat-markdown Wiki, not typed mind-map

The Research Wiki today is unstructured Markdown. **Co-STORM's `co_storm_agents.py:DynamicMindMap`** is updated turn-by-turn during the discourse, with explicit Moderator + Expert agent roles. Porting this gives Polaris a graph-typed Wiki backbone where every claim, source, and contradiction has a typed node — and where `polaris-graph` (citation graph substrate) and `polaris-core/memory` can finally share a schema.

### Gap 4 — Opportunistic idle modes, not heartbeat-scheduled

Polaris's daemon idle policy fires only when the queue is empty. **CORAL (arXiv 2604.01658)** formalises this as a *heartbeat-driven scheduler* that periodically interrupts to force **Reflection** (write notes), **Consolidation** (convert notes to skills), and **Redirection** (pivot when no improvement). The closest paper to Polaris's existing design philosophy — and the cheapest structural lift on the entire roadmap.

### Gap 5 — One-axis evidence gate, not two-axis

ClaimGate today asks "is there an evidence row?" The 2025-2026 literature has converged on a two-axis test:

- **CiteGuard (arXiv 2510.17853)** — attribution alignment: *would a human author cite this same paper for this exact sentence?* Retrieval-grounded validation; 68.1% on CiteME approaching the 69.2% human ceiling. GPT-4o-as-judge gets 16-17% recall.
- **Contradiction-to-Consensus (arXiv 2602.18693)** — inter-source agreement: retrieve evidence for the claim *and its negation* across federated sources; quantify disagreement.
- **JMIR 2026 e88766** — empirical confirmation that retraction-awareness must be a *retrieval-side* gate, not an LLM prompt; major AI tools cite retracted literature without warning even when explicitly asked.

Polaris's existing ClaimGate × the two-axis composition × hardened RED-RETRACTED gate gives it a 3D acceptance criterion no OSS competitor implements.

## §4 — The benchmark triple Polaris should target

The benchmarks stream's call: only three public 2025-2026 venues score Polaris's distinctive surface (verifiable, trust-graded, multi-domain research synthesis). Everything else is either off-axis or a venue where frontier-model raw capability dominates harness-level advantages.

| Venue | Owner | Why Polaris fits | What it scores |
|---|---|---|---|
| **AstaBench** | Ai2 ([allenai/asta-bench](https://github.com/allenai/asta-bench)) | 2,400+ problems × 11 sub-benchmarks × 16 leaderboards including **lit-understanding (ScholarQA-CS2)** + **DiscoveryBench** + **E2E-Bench**; cost axis is part of the score | Federation, trust-tier, cross-family review |
| **DeepResearch Bench** | Tsinghua (arXiv 2506.11763) | 100 PhD tasks × 22 fields, RACE rubric + **FACT axis** (citation trustworthiness + factual abundance) directly mirrors `polaris-evals/run_program_evals()` | Trust-tier metadata is a measurable edge |
| **BrowseComp-Plus** | Waterloo (arXiv 2508.06600, ACL 2026 Main) | 830 Qs on a *fixed corpus* with human-verified supporting docs and mined hard negatives; citation accuracy graded directly; reproducible offline | Federation + trust-tier = better retrieval than BM25/dense |

### Decline / track silently

- **HLE / GAIA / BrowseComp** — frontier-model territory; Polaris's rigor is a tax not a boost.
- **PaperBench / MLE-Bench / ScienceAgentBench / BixBench** — coding-agent benchmarks wearing research-paper costumes.
- **OSWorld / WebArena / AgentBench** — transactional/computer-use; off-axis.
- **FActScore / FELM / ALCE / RAGAs / RULER / LongBench** — useful as internal regression metrics inside `polaris-evals`, never as competitive submissions.

## §5 — The adoption matrix

Each row is one finding; columns place it on (effort, payoff). The three-tier sequence is the basis for [POLARIS_V2_2_DEEP_RESEARCH_PLAN.md](../projects/polaris/POLARIS_V2_2_DEEP_RESEARCH_PLAN.md) (P28-P32).

| Tier | Lift | Maps to | Effort | Payoff |
|---|---|---|---|---|
| **0 (days 1-14)** | CORAL heartbeat scheduler | `polaris-daemon/heartbeat.py` | S | Closes Gap 4; closest to existing design |
| **0** | Two-axis evidence gates (CiteGuard + C2C + retraction) | `polaris-core/permissions/gates/` + `polaris-mcp/retraction.py` | S | Closes Gap 5; hardens ClaimGate |
| **0** | OpenScholar T1 federation source | `polaris-mcp/sources/openscholar.py` | S | 45M-paper drop-in; T1-grade |
| **0** | Deep Research Bench nightly CI | `polaris-evals/benchmarks/deep_research_bench/` | S | Visibility; signal |
| **1 (days 15-45)** | Tournament + Elo over hypotheses | `polaris-core/orchestration/tournament.py` | M | Generalises adversarial pair to population |
| **1** | Tree-search experiment manager | `polaris-skills/research/branch_and_prune.py` | M | Closes Gap 2; ML domain shell win |
| **1** | Co-STORM mind-map as Wiki backbone | `polaris-core/memory/wiki_graph.py` | M | Closes Gap 3; unifies graph schema |
| **1** | TTD-DR draft-as-state writer | `polaris-skills/writing/draft_diffusion.py` | M | Per-patch ClaimGate |
| **1** | Compete on AstaBench + BrowseComp-Plus | `polaris-evals/benchmarks/{asta_bench,browsecomp_plus}/` | M | Public visibility |
| **2 (days 46-90)** | Tongyi IterResearch "Heavy" mode | `polaris-core/loop/heavy_mode.py` | M | Test-time scaling; SOTA pattern |
| **2** | smolagents CodeAgent dispatch | `polaris-skills/codeagent/` | M | ~30% fewer steps; sandbox abstraction |
| **2** | Search-R1 RL recipe | `polaris-research/rl/` | L | Closes Gap 1; biggest unrealised lever |
| **2** | AgentPRM / HiPRAG rubrics | `polaris-skills/auto_creator/process_reward.py` | S | Process-reward toolkit; usable without training |
| **2** | Biomni datalake + Know-How template | `polaris-domains/biomed/` | M | Datalake + protocol library + domain finetune triad |

## §6 — Three structural commitments shared across the 2025-2026 wave

Beyond individual systems, the wave commits to three structural choices that Polaris already partially embodies:

1. **Skills are files, not code.** AutoSkill / Ctx2Skill / EvoSkill / CoEvoSkills / SkillRL — every skill paper agrees. Polaris's `polaris-skills/` Markdown library already does this.
2. **Memory is filesystem, not in-context.** DeerFlow 2.0, Ralph, AgentLab, deepagents, Polaris all converge here. The dissenters (pure-LLM RAG systems) lose at long horizons.
3. **MCP is the universal tool-server protocol.** LangChain deepagents, gpt-researcher, Biomni, Tongyi DeepResearch all expose tools via MCP. Polaris's `polaris-mcp` is the right shape; the next move is making federation sources MCP-first.

## §7 — The single move that would matter most

If you read nothing else in this synthesis, read this:

**Polaris's prompted executor is the ceiling.** Tongyi DeepResearch holds SOTA on HLE / BrowseComp / FRAMES because the executor is *trained*, not prompted, with **rule-based outcome rewards over GRPO**. Polaris's distinctive disciplines (cross-family review, trust-tiered sources, structural provenance) become *training signals* in this regime — every BL-* gate becomes a reward shaping rule. The work is months, the engineering is non-trivial, but the eventual delta closes the gap to "best system." Search-R1 + Tongyi's GRPO setup is the canonical recipe; AgentPRM / HiPRAG provide the rubric scaffolding even before retraining.

This sits in v2.2 Tier 2 not Tier 0 because the prerequisites (a working tournament for Elo signals, trained reward models, evaluation harnesses) live in Tier 1. The sequence is: Tier 0 hardens the gate surface, Tier 1 makes the architecture branch-able and population-based, Tier 2 trains the executor against the gates the prior tiers built.

## §8 — Where this fits

Read after the wave it synthesises:

- **Surveys**: [158-deep-research-agents-survey-huang-et-al](158-deep-research-agents-survey-huang-et-al.md), [159-deep-research-survey-zhang-et-al](159-deep-research-survey-zhang-et-al.md)
- **Frameworks synthesis (predecessor)**: [166-research-agent-frameworks-synthesis](166-research-agent-frameworks-synthesis.md)
- **Skill self-evolution synthesis (sibling)**: [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md)
- **24/7 operational reality**: [160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md)

Acts as input to:

- [POLARIS_V2_2_DEEP_RESEARCH_PLAN.md](../projects/polaris/POLARIS_V2_2_DEEP_RESEARCH_PLAN.md) — the executable phasing of this synthesis (P28-P32, days 1-14)
- The deferred Tier-1 / Tier-2 plan (will land as POLARIS_V2_3 once Tier-0 ships)

## §9 — Reference list (verified May 2026)

### Frontier papers (the post-166 wave)

1. CORAL — Heartbeat-based long-running multi-agent evolution. arXiv:2604.01658.
2. AI Co-Scientist. arXiv:2502.18864 (Google DeepMind, Feb 2025).
3. AI Scientist v2 — Agentic tree search. arXiv:2504.08066 (Sakana, Apr 2025).
4. AgentRxiv — Cross-lab autonomous research collaboration. arXiv:2503.18102.
5. TTD-DR — Test-time diffusion deep researcher. arXiv:2507.16075 (Google).
6. Search-R1 — RL-trained interleaved search & reasoning. arXiv:2503.09516.
7. R1-Searcher / R1-Searcher++. arXiv:2503.05592 / 2505.17005.
8. WebDancer. arXiv:2505.22648 (Tongyi).
9. WebSailor / WebSailor-V2. arXiv:2507.02592 / 2509.13305 (Tongyi).
10. CiteGuard — Faithful citation attribution. arXiv:2510.17853.
11. Contradiction-to-Consensus — Multi-source verification. arXiv:2602.18693.
12. OpenScholar. arXiv:2411.14199 (AI2/UW, *Nature* 2026).
13. AgentPRM / HiPRAG / GRPO-is-Secretly-PRM. arXiv:2511.08325 / 2510.07794 / 2509.21154.
14. JMIR 2026 — AI tools cite retracted literature. JMIR 28:e88766.

### High-impact OSS repos (post-166)

1. [Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch) — 18.8k★, NeurIPS 2025.
2. [SakanaAI/AI-Scientist-v2](https://github.com/SakanaAI/AI-Scientist-v2) — 6.1k★, *Nature* 2025.
3. [langchain-ai/open_deep_research](https://github.com/langchain-ai/open_deep_research) — 11.3k★ + [langchain-ai/deepagents](https://github.com/langchain-ai/deepagents) — 10k★.
4. [stanford-oval/storm](https://github.com/stanford-oval/storm) — 28.2k★ (Co-STORM).
5. [assafelovic/gpt-researcher](https://github.com/assafelovic/gpt-researcher) — 26.9k★, v3.4.4 Apr 2026.
6. [huggingface/smolagents](https://github.com/huggingface/smolagents) — 27.1k★; example [`open_deep_research`](https://github.com/huggingface/smolagents/tree/main/examples/open_deep_research) hit 55% on GAIA.
7. [snap-stanford/Biomni](https://github.com/snap-stanford/Biomni) — biomed datalake + Know-How Library.
8. [PeterGriffinJin/Search-R1](https://github.com/PeterGriffinJin/Search-R1) — 4.6k★.
9. [SamuelSchmidgall/AgentLaboratory](https://github.com/SamuelSchmidgall/AgentLaboratory) + [AgentRxiv](https://agentrxiv.github.io/).
10. [zou-group/virtual-lab](https://github.com/zou-group/virtual-lab) — *Nature* 2025 (SARS-CoV-2 nanobodies).

### Benchmarks

1. [AstaBench (Ai2)](https://allenai.org/asta/bench) — 2,400+ problems × 11 sub-benchmarks × 16 leaderboards, cost axis.
2. [DeepResearch Bench (Tsinghua)](https://deepresearch-bench.github.io/) — arXiv:2506.11763, RACE + FACT.
3. [BrowseComp-Plus (Waterloo, ACL 2026)](https://github.com/texttron/BrowseComp-Plus) — arXiv:2508.06600, fixed corpus + graded citations.
4. [LAB-Bench (FutureHouse)](https://huggingface.co/datasets/futurehouse/lab-bench) — biomed; Polaris targets LitQA2 / DbQA only.
5. [FACTS Grounding / FACTS Suite (DeepMind)](https://deepmind.google/blog/facts-grounding-a-new-benchmark-for-evaluating-the-factuality-of-large-language-models/) — Polaris evals already mirror.

## §10 — One-paragraph summary

Polaris is the discipline winner; Tongyi is the capability winner; the gap is closeable. v2.2 (P28-P32) closes the gate surface — heartbeat scheduler, two-axis evidence gates, OpenScholar T1 ingest, Deep Research Bench nightly CI — in 14 days. v2.3 closes the architecture surface — tournament + Elo, tree-search experiment manager, mind-map Wiki, TTD-DR writer — in the next 30 days. v2.4 closes the training surface — IterResearch Heavy mode, smolagents CodeAgent dispatch, Search-R1 RL recipe — over the following two months. The benchmark plan (AstaBench → DeepResearch Bench → BrowseComp-Plus) runs in parallel and provides the regression signal each tier needs. Submit the synthesis here; the executable phasing is in [POLARIS_V2_2_DEEP_RESEARCH_PLAN.md](../projects/polaris/POLARIS_V2_2_DEEP_RESEARCH_PLAN.md).
