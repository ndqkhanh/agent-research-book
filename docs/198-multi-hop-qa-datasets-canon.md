# 198 — The Multi-Hop QA Datasets Canon: From HotpotQA to SealQA

> **Disambiguation.** This file is the **datasets / benchmarks** half of a four-part block (198–201) on multi-hop reasoning in agents. The technique arc lives in [199-multi-hop-reasoning-techniques-arc](199-multi-hop-reasoning-techniques-arc.md), the graph-grounded retrieval line in [200-graph-grounded-multi-hop-retrieval](200-graph-grounded-multi-hop-retrieval.md), and the formal-theoretical compositionality-gap lineage in [201-compositionality-gap-canon](201-compositionality-gap-canon.md).

## One-line definition

A **multi-hop QA dataset** is a benchmark whose questions cannot be answered from any single retrieved passage in isolation — the answer requires **chaining 2+ pieces of evidence** drawn from different sources, and the canon spans 8 years (2018 → 2026), six difficulty regimes (depth, fan-out, decomposition, retrieval-noise, browsing-depth, conflict), and four orders of magnitude in size (2.7k → 800k+).

## Why this canon matters

Multi-hop QA is the **load-bearing benchmark family** for almost every "agentic reasoning" claim made between 2022 and 2026. When a paper says *"our agent improved over RAG by +Δ"*, the Δ is almost certainly measured on HotpotQA, 2WikiMultiHopQA, MuSiQue, MultiHop-RAG, FRAMES, FanOutQA, BrowseComp, or SealQA. Knowing how each one is constructed — and what it secretly does and doesn't measure — is the single most productive lens for reading the field.

The benchmarks have evolved along a clean axis. **HotpotQA (2018)** introduced *supporting-fact* labels — making the reasoning chain itself a first-class evaluation target rather than just the final answer. **2WikiMultiHopQA (2020)** grounded chains in a Wikidata KG so every hop is verifiable as a triple. **MuSiQue (2022)** flipped the construction direction to bottom-up composition with answerable-vs-unanswerable contrast sets that defeat shortcut reasoning. **MultiHop-RAG (2024)** made the *retriever* the explicit unit of evaluation rather than the reader. **FanOutQA (2024)** swapped depth for breadth — questions that fan out to 5+ Wikipedia articles. **FRAMES (2024)** unified factuality + retrieval + multi-hop reasoning into a single number. **BrowseComp (2025)** moved the corpus to the open web and added browsing-depth as a difficulty axis. **SealQA (2025)** added retrieval *noise* — three flavors of "the search results are conflicting / unhelpful / adversarial" — and showed that frontier reasoning models *plateau or decline* under more test-time compute when retrieval is bad enough.

Take this canon seriously and three things change. (1) You stop quoting unqualified "+Δ on HotpotQA" — you say "*on the distractor setting,* +Δ supporting-fact F1," because supporting-fact F1 is what changes when reasoning improves and answer F1 mostly reflects guessing. (2) You stop trusting any agent paper that doesn't report MuSiQue-Full, because MuSiQue-Ans alone admits shortcuts. (3) You start treating BrowseComp and SealQA as the *real* deep-research evals — HotpotQA-class accuracy >95% means a frontier model has roughly memorised the corpus.

## Problem each generation solves

- **2018 — HotpotQA.** Move beyond single-passage SQuAD-style QA; force composition across 2–3 paragraphs; expose the reasoning chain via supporting-fact labels.
- **2020 — 2WikiMultiHopQA + IIRC + StrategyQA.** Add KG-grounded provenance, the "knowing when you need more info" detection task, and *implicit* decomposition (no surface cues).
- **2022 — MuSiQue.** Defeat shortcut reasoning by composing single-hop questions bottom-up + adding unanswerable contrast sets.
- **2024 — MultiHop-RAG + FanOutQA + FRAMES.** Stress the *retriever* (not the reader); test breadth (fan-out) rather than depth; unify factuality + retrieval + reasoning into a single number.
- **2025 — BrowseComp + SealQA.** Open-web browsing depth; reasoning under *noisy / conflicting / adversarial* retrievals.

## The canon, in order

### HotpotQA (Yang et al. 2018) — the supporting-facts ancestor

- **Citation.** *HotpotQA: A Dataset for Diverse, Explainable Multi-hop Question Answering*. Yang, Qi, Zhang, Bengio, Cohen, Salakhutdinov, Manning. EMNLP 2018, arXiv 1809.09600.
- **Size.** 112,779 English Wikipedia QA pairs.
- **Settings.** *Distractor* (2 gold + 8 TF-IDF distractor paragraphs per question) and *fullwiki* (retrieve from ~5M articles).
- **Reasoning-types.** ~42% bridge-entity (chain), ~27% comparison, ~15% intersection, ~6% property-transfer.
- **What it actually measures.** Answer EM/F1 *and* supporting-fact EM/F1 — the latter is the canonical "did the model walk the right chain?" signal. Joint metrics combine both.
- **Why it matters.** First dataset to make the *reasoning chain* a first-class eval target. Every subsequent multi-hop dataset cites HotpotQA as the baseline to beat or critique.
- **Failure mode.** The distractor setting is now too easy — frontier models with retrieval saturate at >85 EM. The "easy split" is widely shortcut-able from a single paragraph.

### 2WikiMultiHopQA (Ho et al. 2020) — KG-grounded chains

- **Citation.** *Constructing A Multi-hop QA Dataset for Comprehensive Evaluation of Reasoning Steps*. Ho, Nguyen, Sugawara, Aizawa. COLING 2020.
- **Size.** ~190k examples over Wikipedia + Wikidata (~5.95M entities).
- **Question types.** comparison, inference, compositional, bridge-comparison.
- **Innovation.** Each question has an explicit *evidence path* in the KG (subject, relation, object triples) plus the matching unstructured passages. The chain is *verifiable* as a graph walk.
- **Why it matters.** The standard benchmark for any system that claims "graph-aware" multi-hop retrieval — HippoRAG, IRCoT, Beam Retrieval, R1-Searcher all anchor evals here.
- **Failure mode.** Very high agreement between gold passages and gold KG triples → systems can shortcut by retrieving on entity-overlap alone.

### MuSiQue (Trivedi et al. 2022) — the shortcut-killer

- **Citation.** *MuSiQue: Multihop Questions via Single-hop Question Composition*. Trivedi, Balasubramanian, Khot, Sabharwal. TACL 2022, arXiv 2108.00573.
- **Size.** ~25k 2–4 hop questions, built bottom-up by composing connected single-hop questions sourced from existing SQuAD-style datasets.
- **Splits.** *MuSiQue-Ans* (answerable only) and *MuSiQue-Full* (adds unanswerable contrast questions that are minimally different from answerable ones).
- **Innovation.** Bottom-up construction means every hop is verifiable; the contrast set defeats single-paragraph shortcuts (a single-hop model loses ~30 F1 vs a multi-hop model — *3× larger human-machine gap than HotpotQA*).
- **Why it matters.** Currently the most discriminating multi-hop benchmark. Beam Retrieval's ~50% gain over baselines is on MuSiQue-Ans. If a paper omits MuSiQue-Full, suspect shortcut leakage.
- **Repo.** [github.com/StonyBrookNLP/musique](https://github.com/StonyBrookNLP/musique) — 219★.

### IIRC (Ferguson et al. 2020) — incomplete-info detection

- **Citation.** *IIRC: A Dataset of Incomplete Information Reading Comprehension Questions*. Ferguson, Gardner, Hajishirzi, Khot, Dasigi. EMNLP 2020, arXiv 2011.07127.
- **Size.** 13k+ questions.
- **Innovation.** Crowd-workers wrote questions *without* seeing the linked documents — forcing low lexical overlap and many unanswerable questions. Baseline F1: 31.1; human: 88.4.
- **Why it matters.** First dataset to make "knowing *when you need to retrieve more*" a first-class skill. Spiritual ancestor of CRAG and Self-RAG's adaptive retrieval.

### StrategyQA (Geva et al. 2021) — implicit decomposition

- **Citation.** *Did Aristotle Use a Laptop? A Question Answering Benchmark with Implicit Reasoning Strategies*. Geva, Khashabi, Segal, Khot, Roth, Berant. TACL 2021, arXiv 2101.02235.
- **Size.** 2,780 yes/no questions.
- **Innovation.** Reasoning steps are *implicit* — no surface cues like "the Xer of Y." Each example carries (1) decomposition into sub-questions, (2) Wikipedia paragraphs answering each step. Humans 87%, best baseline ~66%.
- **Why it matters.** Canonical benchmark for "the model has to *invent* the decomposition." Self-Ask and IRCoT both leverage StrategyQA-style decomposition supervision.

### Complex WebQuestions (Talmor & Berant 2018) — the KGQA analog

- **Citation.** *The Web as a Knowledge-base for Answering Complex Questions*. NAACL 2018.
- **Size.** 34,689 examples.
- **Innovation.** Extends WebQuestionsSP with conjunctions, superlatives, comparatives, and compositions, built by combining SPARQL queries over Freebase. Each example has a natural-language question *and* an executable SPARQL query.
- **Why it matters.** The KG-side analog of HotpotQA. Standard benchmark for KGQA agents — Plan-on-Graph, Paths-over-Graph, Tree-of-Thought-on-Graph anchor here. See [200-graph-grounded-multi-hop-retrieval](200-graph-grounded-multi-hop-retrieval.md).

### MultiHop-RAG (Tang & Yang 2024) — the retriever-stress benchmark

- **Citation.** *MultiHop-RAG: Benchmarking Retrieval-Augmented Generation for Multi-Hop Queries*. Tang, Yang. COLM 2024, arXiv 2401.15391.
- **Size.** 2,556 queries with 2–4 documents of evidence each, drawn from a news knowledge base.
- **Innovation.** Each query has explicit ground-truth supporting evidence per hop, scored *separately* from the final answer. RAG pipelines must demonstrate they retrieved the right N documents, not just that they answered correctly.
- **Headline result at release.** GPT-4 + naive RAG: "performs unsatisfactorily." The retriever — not the reasoner — is the bottleneck.
- **Why it matters.** Catalyzed the wave of HippoRAG / GraphRAG / iterative-retrieval / RL-trained-retrieval work in 2024–2025.
- **Repo.** [github.com/yixuantt/MultiHop-RAG](https://github.com/yixuantt/MultiHop-RAG) — 442★.

### FanOutQA (Zhu et al. 2024) — fan-out, not depth

- **Citation.** *FanOutQA: A Multi-Hop, Multi-Document Question Answering Benchmark for Large Language Models*. Zhu et al. ACL 2024 (short), arXiv 2402.14116.
- **Size.** 1,034 fan-out questions; 7,305 human-written sub-question decompositions.
- **Innovation.** Each question requires **at least 5 different Wikipedia articles**. Three settings: closed-book, open-book (full Wikipedia), evidence-provided.
- **Why it matters.** Tests *wide* rather than *deep* multi-hop. Naive sequential CoT-with-retrieval fails badly here — the right shape is parallel sub-question fan-out (cf. LlamaIndex `SubQuestionQueryEngine`).
- **Repo.** [github.com/zhudotexe/fanoutqa](https://github.com/zhudotexe/fanoutqa) — 61★.

### FRAMES (Krishna et al. 2024) — factuality × retrieval × multi-hop

- **Citation.** *Fact, Fetch, and Reason: A Unified Evaluation of Retrieval-Augmented Generation*. Krishna et al. NAACL 2025, arXiv 2409.12941. (Google + Harvard.)
- **Size.** 824 challenging multi-hop questions, each requiring 2–15 Wikipedia articles.
- **Composition.** ~36% multi-constraint, 20% numerical comparison, 16% temporal disambiguation.
- **Headline numbers.** Frontier LLMs: 0.40 with no retrieval; multi-step retrieval pipeline reaches 0.66.
- **Why it matters.** First benchmark to *jointly* score retrieval-quality + fact-recall + reasoning composition. Single-shot retrieval caps at ~0.40 even with frontier models — the gap is multi-step retrieval, not raw model strength.

### BrowseComp (Wei et al., OpenAI 2025) — open-web browsing depth

- **Citation.** *BrowseComp: A Simple Yet Challenging Benchmark for Browsing Agents*. arXiv 2504.12516, April 2025.
- **Size.** 1,266 problems.
- **Innovation.** Each question requires **deep browsing** — agents may need to traverse tens to hundreds of websites for a single fact. Builds on the saturation of SimpleQA.
- **Headline numbers.** GPT-4o + browsing: 1.9%. OpenAI Deep Research: 51.5%. GPT-5.5 Pro: 90.1%. The 27×→47× spread is **architectural** (tool-use orchestration), not knowledge-related — the model size doesn't change.
- **Why it matters.** Concretizes "open-web multi-hop" as an *agent* task, not a retrieval task. The benchmark has driven most 2025–2026 deep-research-agent design (see [158-deep-research-agents-survey-huang-et-al](158-deep-research-agents-survey-huang-et-al.md), [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md)).

### SealQA (2025) — multi-hop under retrieval noise

- **Citation.** *SealQA: Raising the Bar for Reasoning in Search-Augmented Language Models*. arXiv 2506.01062.
- **Splits.** *Seal-0* (hardest, near-zero accuracy for chat models), *Seal-Hard* (factual + reasoning), *LongSeal* (long-context multi-document needle-in-haystack).
- **Construction.** Questions are deliberately built around the regime where **web search returns conflicting / noisy / unhelpful results** — the realistic open-web condition.
- **Headline numbers.** o3 reaches 17.1%, o4-mini 6.3% on Seal-0 even at maximal reasoning effort. **Test-time compute often plateaus or declines** as the model spends more thinking tokens chasing red herrings.
- **Why it matters.** Shifts evaluation toward "reasoning under noisy retrievals" — the regime no other benchmark targets. The plateau-or-decline curve is a key empirical hook for the test-time-compute claims of [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md).

### BrowseComp-Plus (Waterloo, 2025) — fixed-corpus reproducible browsing

- **Citation.** arXiv 2508.06600, ACL 2026 Main.
- **Size.** 830 questions on a *fixed corpus* with human-verified supporting docs and mined hard negatives.
- **Innovation.** Re-runs BrowseComp's question style on a closed corpus, so federated-retrieval systems are scored offline and reproducibly. Citation accuracy is graded directly.
- **Why it matters.** The *only* deep-research benchmark that controls for retrieval-corpus drift. Polaris targets this in v2.3 (see [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md) §4).

## Cross-cut: which dataset measures what

| Axis | HotpotQA | 2WikiMHQA | MuSiQue | IIRC | StrategyQA | MultiHop-RAG | FanOutQA | FRAMES | BrowseComp | SealQA |
|---|---|---|---|---|---|---|---|---|---|---|
| Depth ≥3 hops | ✓ | ✓ | ✓ | partial | ✓ | partial | weak | ✓ | ✓ | ✓ |
| Fan-out ≥5 docs | — | — | — | — | — | partial | **primary** | partial | — | LongSeal |
| KG-grounded | — | **primary** | — | — | — | — | — | — | — | — |
| Shortcut-resistant | partial | partial | **primary** | partial | ✓ | partial | weak | ✓ | ✓ | ✓ |
| Decomposition supervision | sentences | KG triples | sub-questions | — | sub-questions | — | sub-questions | — | — | — |
| Retriever stress | distractor | partial | — | — | — | **primary** | — | ✓ | open-web | noise |
| Open-web browsing | — | — | — | — | — | — | — | partial | **primary** | partial |
| Adversarial / conflicting evidence | — | — | — | — | — | — | — | — | — | **primary** |
| Test-time-compute saturation | saturated | saturating | active | saturated | saturated | saturating | active | active | active | active |

Read this row by row when choosing benchmarks for an agent paper. **Picking only saturated benchmarks is the most common reviewer-flag** in 2025–2026 multi-hop submissions.

## Empirical results — current SOTA snapshot (May 2026)

| Dataset | Strongest reported single-system result | System | Reference |
|---|---|---|---|
| HotpotQA-distractor (joint F1) | ~85 | DSPy + ColBERTv2 + GPT-4o | [93-dspy](93-dspy.md) |
| HotpotQA-fullwiki (answer F1) | ~78 | Search-R1 (Qwen2.5-7B) | arXiv 2503.09516 |
| 2WikiMultiHopQA (joint F1) | ~99.9 precision passages | Beam Retrieval | arXiv 2308.08973 |
| MuSiQue-Ans (F1) | ~70 | HippoRAG + IRCoT compose | arXiv 2405.14831 |
| MultiHop-RAG (acc) | ~0.71 | LightRAG | arXiv 2410.05779 |
| FanOutQA (open-book) | ~0.55 | LangGraph deep research | [open_deep_research](https://github.com/langchain-ai/open_deep_research) |
| FRAMES (acc) | 0.66 | multi-step retrieval pipeline | arXiv 2409.12941 |
| BrowseComp | 90.1% | GPT-5.5 Pro Deep Research | OpenAI |
| SealQA Seal-0 | 17.1% | o3 (max reasoning) | arXiv 2506.01062 |

These numbers move *fast*. The headline pattern: **whoever wins MuSiQue-Full + Seal-0 + BrowseComp simultaneously holds the multi-hop crown.** As of May 2026 nobody does — Tongyi DeepResearch wins BrowseComp, Search-R1 / HippoRAG wins MuSiQue regimes, no system clears 25% on Seal-0.

## Variants and ancillary benchmarks

- **CWQ-Bench / KQA-Pro / GrailQA / ComplexWebQuestions-Bridge** — KGQA variants for graph-aware agents.
- **TopiOCQA / QReCC** — multi-hop *conversational* QA, where each turn adds a hop.
- **HybridQA / OTT-QA** — multi-hop over heterogeneous sources (tables + free text).
- **NarrativeQA / QuALITY / QASPER** — long-document multi-hop, tested heavily by [ReadAgent](199-multi-hop-reasoning-techniques-arc.md).
- **WebWalkerQA / WebShop / WebArena** — agent-action multi-hop (more about computer use than QA per se, but increasingly cited together).
- **GAIA / HLE** — broad agent benchmarks where multi-hop is a *subset* of the task surface.

## Failure modes and limitations of the canon

- **Saturation on the easy splits.** HotpotQA-distractor easy split, 2WikiMHQA bridge type, and StrategyQA are now saturated for frontier models. Reporting only these is a flag.
- **Annotation artifacts.** HotpotQA's supporting-facts can sometimes be inferred from question wording; MuSiQue-Ans alone admits shortcuts; FanOutQA closed-book leaks via parametric memory.
- **Retrieval-corpus drift.** FullWiki / open-web setups vary across labs because corpora aren't pinned (Wikipedia changes monthly, the open web continuously). BrowseComp-Plus is the fix.
- **Retrieval *vs.* reasoning is not separable.** Multi-hop pipelines compound retrieval failures into reasoning failures; single-axis evals (answer-only) hide whether the failure was a missed document or a missed deduction. Always co-report retrieval recall@k and supporting-fact F1 when both are available.
- **Cost is rarely scored.** AstaBench (see [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md)) is the only major suite that puts $-per-query on the leaderboard. Multi-agent multi-hop systems can win accuracy by 50× the budget — see [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md).

## When to use, when not

**Use HotpotQA + 2WikiMultiHopQA + MuSiQue-Full** as the standard 3-pack for any retrieval-aware multi-hop method paper — this is the minimum evidence reviewers will accept, and the trio covers depth, KG-groundedness, and shortcut-resistance. Add MultiHop-RAG when the contribution is on the retriever, FanOutQA when it's on parallel decomposition, FRAMES when the claim is unified factuality + retrieval, and BrowseComp/SealQA when the claim is "deep research agent."

**Don't use HotpotQA-distractor alone** — it's saturated, and any reported gain there has to be re-checked on MuSiQue-Full to rule out shortcut exploitation. Don't use only 2WikiMultiHopQA either — its KG-question generation makes it lexical-overlap-friendly. And don't quote BrowseComp without specifying which model + browsing tool — the spread between configurations dwarfs the spread between methods.

## Implications for harness engineering

- **Eval pinning.** Pin a snapshot of Wikipedia / open-web corpora per benchmark run — corpus drift kills reproducibility on FullWiki, FanOutQA-open-book, and BrowseComp. Cf. [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md) on reproducible eval scaffolding.
- **Co-report retrieval and answer metrics.** Every multi-hop harness should log retrieval recall@k *per hop* and supporting-fact F1 alongside answer EM/F1. See [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md).
- **Add a noise stress test.** A SealQA-style adversarial-retrieval sub-bench inside CI catches the regression where a refactor doubles the agent's confidence in conflicting sources. See [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [82-poisonedrag](82-poisonedrag.md).
- **Track decomposition fidelity.** When the harness uses Self-Ask / IRCoT / DSPy multi-hop, log generated sub-questions and compare against StrategyQA / MuSiQue gold decompositions on a held-out set — this catches "the agent answers right but its chain is wrong" regressions ([18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md)).
- **Per-hop budget caps.** Sequential multi-hop pipelines benefit from per-hop token / latency caps so a runaway second hop doesn't exhaust the global budget. See [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md).
- **Treat fan-out and depth as different scheduler shapes.** Fan-out questions (FanOutQA) want parallel sub-question dispatch; deep questions (MuSiQue) want sequential chains. The router can be a simple type-classifier — this is exactly what BELLE [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md) does.
- **Retraction / conflict gates at the retriever.** SealQA-class noise + JMIR's retracted-paper finding mean the agent harness must score *evidence quality* before reading. Cf. [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md) §3 Gap 5.
- **Cost-aware leaderboards.** Bake $/query into your internal regression dashboard — a method that wins MuSiQue at 50× cost is not necessarily a win. Cf. [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md), [88-confidence-driven-router](88-confidence-driven-router.md).
- **Decomposition cache.** Sub-question decompositions (especially fan-out ones) repeat across queries; caching them as part of the [09-memory-files](09-memory-files.md) layer cuts latency materially.
- **Benchmarks as harness regression tests.** Pick three (e.g. MuSiQue-Full + FRAMES + Seal-0) and run them per release of the agent harness. Surfacing a regression early is worth more than chasing 1-point absolute gains. See [115-evaluating-llm-systems](115-evaluating-llm-systems.md).
- **Cross-link to graph-RAG.** When the corpus is enterprise (not Wikipedia), reach for graph-RAG ([200-graph-grounded-multi-hop-retrieval](200-graph-grounded-multi-hop-retrieval.md)) before flat vector RAG — these benchmarks reveal where the latter breaks.
- **Benchmark choice signals harness shape.** A team optimising for HotpotQA will produce a different harness than a team optimising for SealQA — the eval choice quietly fixes the architectural decisions downstream.

**The one-line takeaway for harness designers:** Pick **MuSiQue-Full + FRAMES + Seal-0** as your minimal multi-hop regression triple — anything that survives all three has earned the right to be called a multi-hop reasoning agent.
