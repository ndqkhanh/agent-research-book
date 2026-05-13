# 199 — Multi-Hop Reasoning Techniques: From Self-Ask to Search-R1

> **Disambiguation.** This file is the **techniques** half of a four-part block (198–201) on multi-hop reasoning. Datasets/benchmarks live in [198-multi-hop-qa-datasets-canon](198-multi-hop-qa-datasets-canon.md), graph-grounded retrieval in [200-graph-grounded-multi-hop-retrieval](200-graph-grounded-multi-hop-retrieval.md), formal/theoretical compositionality in [201-compositionality-gap-canon](201-compositionality-gap-canon.md). The 2026 multi-agent reckoning is in [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md).

## One-line definition

**Multi-hop reasoning techniques** are the family of prompting, declarative-pipeline, and RL-trained methods that interleave reasoning with retrieval — moving from monolithic "retrieve once, read once" RAG to *adaptive*, *iterative*, *outcome-supervised* loops — and the 2022 → 2026 arc is best read as four phases: prompted (Self-Ask, IRCoT), declarative (DSP/DSPy), reflective (Self-RAG, CRAG, Self-Refine, Chain-of-Note), and trained (Search-R1, R1-Searcher, ReSearch, DeepResearcher, WebThinker).

## Why this arc matters

Single-shot RAG fails on multi-hop because what to retrieve next depends on what was just derived, which depends on what was just retrieved. **IRCoT** (Trivedi et al. 2023) named this loop precisely; the entire 2023–2026 multi-hop literature is a search for the right way to *close* it. The arc has four overlapping phases, and each phase made one thing the *unit of optimisation* — prompts, programs, reflection tokens, finally the policy itself.

The 2022 prompted phase (Self-Ask, IRCoT, ITER-RETGEN) treated retrieval as something the model could *condition on* between reasoning steps. The 2023 declarative phase (DSP / DSPy) realised the prompt template was the wrong abstraction — programs of LM modules, optimised by a compiler, dominated hand-crafted prompts. The 2023–2024 reflective phase (Self-RAG, CRAG, Self-Refine, Chain-of-Note) gave the LM *meta-tokens* — `Retrieve`, `ISREL`, `ISSUP`, `ISUSE` — so the loop became *adaptive*. The 2025–2026 trained phase (Search-R1, R1-Searcher, ReSearch, DeepResearcher, WebThinker) applied **outcome-supervised RL with retrieved-token masking** to fold the loop into the model itself, often beating prompted GPT-4o by double digits with a 7B Qwen base.

Take this arc seriously and three things change. (1) You stop hand-engineering retrieval pipelines — DSPy compiles the multi-hop program against your held-out set. (2) You stop using `agent.run("retrieve and answer")` — you wire the LLM with `<search>...</search>` action tokens and either prompt them strictly or RL-train on outcome rewards. (3) You stop assuming "more retrieval = better" — Chain-of-Note and CRAG make per-passage *quality* a first-class signal, and SealQA shows that frontier reasoning models *plateau or decline* under noisy retrieval if you don't filter.

## Problem this arc solves

- Single-shot RAG breaks for any question where the second hop depends on the first hop's answer.
- Hand-crafted prompts don't survive base-model swaps; the prompt brittleness gets worse as agents go multi-hop.
- Without a quality gate, one bad retrieval poisons the entire chain (the "garbage-in, multi-step-amplified-out" failure).
- Without a *trained* policy, the model uses its retrieval tool the same way regardless of how informative the result is.
- Without action-token discipline, the LM hallucinates retrieved-text-shaped output and claims it as evidence.

## Phase 1 — Prompted retrieval-reasoning (2022–2023)

### Self-Ask (Press et al. 2022)

- **Citation.** *Measuring and Narrowing the Compositionality Gap in Language Models*. arXiv 2210.03350, Findings of EMNLP 2023.
- **Mechanism.** The model emits "*Are follow up questions needed here?*" Yes → "*Follow up: ...*" → answer the sub-question (optionally via a search tool) → repeat → "*So the final answer is: ...*"
- **Insight.** The compositionality gap (the fraction of cases where the model knows both sub-facts but fails to compose) **doesn't shrink with scale**. Self-Ask narrows it because explicit follow-ups slot a search engine in cleanly. This is the formal hook used throughout [201-compositionality-gap-canon](201-compositionality-gap-canon.md).
- **Repo.** [github.com/ofirpress/self-ask](https://github.com/ofirpress/self-ask) — 326★.

### IRCoT (Trivedi et al. 2023) — interleaved retrieval

- **Citation.** *Interleaving Retrieval with Chain-of-Thought Reasoning for Knowledge-Intensive Multi-Step Questions*. ACL 2023, arXiv 2212.10509.
- **Mechanism.** Generate one CoT sentence → use it as the next retrieval query → append retrieved passages to context → generate next CoT sentence → repeat until "answer:" appears.
- **Headline numbers.** Up to **+21 retrieval / +15 QA points** over single-shot retrieve-then-read on HotpotQA / 2WikiMultiHopQA / MuSiQue / IIRC.
- **Insight.** The per-sentence retrieval granularity is the canonical formal answer to "single-shot RAG is not enough." Almost every later iterative-retrieval paper (Search-o1, ReSearch, R1-Searcher) is an evolution of IRCoT.
- **Repo.** [github.com/StonyBrookNLP/ircot](https://github.com/StonyBrookNLP/ircot) — 262★.

### ITER-RETGEN (Shao et al. 2023)

- **Citation.** *Enhancing Retrieval-Augmented LLMs with Iterative Retrieval-Generation Synergy*. Findings of EMNLP 2023, arXiv 2305.15294.
- **Mechanism.** Coarser than IRCoT — generates a *whole* answer, uses it as a query expansion for the next retrieval, regenerates. Each generation acts as a query-expansion seed for the next retrieval.
- **Trade-off.** Lower retrieval/generation overhead than IRCoT (sentence-by-sentence) but coarser interleaving. Better latency profile for production.

### Plan-and-Solve / Decomposed Prompting (2022–2023)

- **Citations.** *Plan-and-Solve Prompting* (arXiv 2305.04091); *Decomposed Prompting* (Khot et al., arXiv 2210.02406).
- **Mechanism.** Separate planning from execution: the planner produces an explicit list of sub-tasks, and a per-sub-task executor (often a different prompt or tool) handles each.
- **Why it matters.** The conceptual ancestor of DSPy's `MultiHop`/`Hop` modules and BELLE's "operator selection." See [16-plan-and-solve](16-plan-and-solve.md), [17-rewoo](17-rewoo.md).

## Phase 2 — Declarative pipelines (2022–2024)

### DSP — Demonstrate-Search-Predict (Khattab et al. 2022)

- **Citation.** *Demonstrate-Search-Predict: Composing Retrieval and Language Models for Knowledge-Intensive NLP*. arXiv 2212.14024.
- **Mechanism.** A pipeline of LM calls: *Demonstrate* (auto-generated few-shot exemplars) → *Search* (ColBERTv2 multi-hop retrieval) → *Predict*. The pipeline is declared in code; the prompts are bootstrapped automatically.
- **Headline numbers.** **+37–120% over vanilla GPT-3.5; +80–290% over Self-Ask** on multi-hop QA.

### DSPy (Khattab et al. 2023, ongoing)

- **Citation.** *DSPy: Compiling Declarative Language Model Calls into State-of-the-Art Pipelines*. arXiv 2310.03714.
- **Mechanism.** "Programming, not prompting." `Module` / `Signature` / `Optimizer` abstractions. The canonical multi-hop demo uses ColBERTv2 + a `MultiHop`/`Hop` module that loops `(num_hops, num_docs)` until a STOP signal, with auto-bootstrapped few-shot demonstrations. Optimisers (`MIPROv2`, `BootstrapFewShot`, `BootstrapFinetune`) make pipeline quality a *function of held-out eval scores*, not prompt-engineering taste.
- **Why it matters.** Made the *optimizer* — not the prompt — the locus of multi-hop improvement. A non-trivial multi-hop pipeline now compiles in tens of minutes against a small training set.
- **Repo.** [github.com/stanfordnlp/dspy](https://github.com/stanfordnlp/dspy) — **34,283★**, very active.
- **Cross-link.** [93-dspy](93-dspy.md) for full deep-dive.

## Phase 3 — Reflective / adaptive RAG (2023–2024)

### Self-RAG (Asai et al. 2023)

- **Citation.** *Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection*. ICLR 2024, arXiv 2310.11511.
- **Mechanism.** Trains an LM to emit four kinds of **reflection tokens** during generation:
  - `Retrieve` — does this position need retrieval?
  - `ISREL` — is the retrieved doc relevant?
  - `ISSUP` — does the next sentence follow from the doc?
  - `ISUSE` — is the answer ultimately useful?
- **Headline numbers.** 7B/13B Self-RAG beats much larger baselines on knowledge-intensive QA.
- **Why it matters.** Retrieval becomes *adaptive at the token level*. The reflection-token vocabulary is the precursor to R1-style `<search>` action tokens.
- **Repo.** [github.com/AkariAsai/self-rag](https://github.com/AkariAsai/self-rag) — 2,377★.

### CRAG — Corrective RAG (Yan et al. 2024)

- **Citation.** *Corrective Retrieval Augmented Generation*. arXiv 2401.15884.
- **Mechanism.** A lightweight retrieval evaluator scores each retrieved doc into Correct / Ambiguous / Incorrect; a decompose-then-recompose algorithm filters noise; large-scale web search is the fallback when retrieval is inadequate.
- **Why it matters.** First widely cited "RAG with a quality gate." Composes orthogonally with multi-hop chains — every hop is gated.
- **Repo.** [github.com/HuskyInSalt/CRAG](https://github.com/HuskyInSalt/CRAG) — 455★.

### Self-Refine (Madaan et al. 2023)

- **Citation.** *Self-Refine: Iterative Refinement with Self-Feedback*. NeurIPS 2023, arXiv 2303.17651.
- **Mechanism.** Single LM as generator + critic + refiner in a loop, no training data needed. **+20% absolute averaged across 7 tasks** on GPT-3.5/GPT-4.
- **Why it matters in multi-hop.** Catches "answer doesn't follow from chain" failures; pairs cleanly with IRCoT/Self-Ask. See [18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md).

### Chain-of-Note (Yu et al. 2024)

- **Citation.** *Chain-of-Note: Enhancing Robustness in Retrieval-Augmented Language Models*. arXiv 2311.09210.
- **Mechanism.** LM writes a sequential **note per retrieved doc** assessing relevance, then composes the answer.
- **Headline numbers.** **+7.9 EM** on entirely-noisy retrievals; **+10.5** rejection rate on out-of-knowledge questions.
- **Why it matters.** A multi-hop pipeline's fragility is often "one bad retrieval poisons the chain." CoN inserts an explicit per-doc filter step.

### Chain-of-Knowledge (Li et al. 2024)

- **Citation.** *Chain-of-Knowledge: Grounding LLMs via Dynamic Knowledge Adapting over Heterogeneous Sources*. ICLR 2024, arXiv 2305.13269.
- **Mechanism.** Three stages — reasoning preparation → dynamic knowledge adapting (over Wikidata, SQL tables, NL passages) → answer consolidation.
- **Why it matters.** Multi-hop chains often span heterogeneous sources (table + free text + KG). CoK is the canonical fusion pattern.

### ReadAgent (Lee et al. 2024)

- **Citation.** *A Human-Inspired Reading Agent with Gist Memory of Very Long Contexts*. ICML 2024, arXiv 2402.09727.
- **Mechanism.** Pages → gist memories → look-up actions. **3.5–20× effective context extension** on QuALITY / NarrativeQA / QMSum.
- **Multi-hop relevance.** When evidence is spread through a single very-long document, the bottleneck isn't retrieval — it's intra-document recall. Gist memory is the cheap workaround.

### Beam Retrieval (Zhang et al. 2024)

- **Citation.** *End-to-End Beam Retrieval for Multi-Hop Question Answering*. NAACL 2024, arXiv 2308.08973.
- **Mechanism.** Joint encoder + two classification heads across all hops, beam-width hypothesis tracking.
- **Headline numbers.** **~50% gain on MuSiQue-Ans; 99.9% precision on 2WikiMultiHopQA** at retrieval time.
- **Why it matters.** Most retrievers are trained per-hop and only support 2 hops; Beam Retrieval is the SOTA solution when hops > 2.
- **Repo.** [github.com/canghongjian/beam_retriever](https://github.com/canghongjian/beam_retriever) — 132★.

### MMR (Carbonell & Goldstein 1998) — the diversity ancestor

`MMR(d, q, S) = (1−λ)·rel(d,q) − λ·max_{dᵢ∈S} sim(d, dᵢ)`. Classical, but still load-bearing: when top-k for a multi-hop query collapses to near-duplicates of the *same hop*, MMR diversifies coverage. Canonical pre-step before passing to a multi-hop reader.

## Phase 4 — Trained policies (2025–2026)

### Search-o1 (Li et al. 2025)

- **Citation.** *Search-o1: Agentic Search-Enhanced Large Reasoning Models*. arXiv 2501.05366, EMNLP 2025.
- **Mechanism.** Wraps an o1-style long-CoT model with an **agentic search workflow** + a **Reason-in-Documents** module that *refines retrievals before injection*. Batch generation interleaves search detection with reasoning.
- **Why it matters.** First systematic answer to "how do you give an o1-class long-CoT model live retrieval *without* breaking the reasoning chain?" Reason-in-Documents (denoising before injection) is the key trick.
- **Repo.** [github.com/RUC-NLPIR/Search-o1](https://github.com/RUC-NLPIR/Search-o1) — 1,219★.

### Search-R1 (Jin et al. 2025) — the canonical RL recipe

- **Citation.** *Search-R1: Training LLMs to Reason and Leverage Search Engines via Reinforcement Learning*. arXiv 2503.09516.
- **Mechanism.** Built on veRL. Action tokens `<search>...</search>`, `<answer>...</answer>`. **Retrieved-token masking** prevents gradient flow through retrieval text — gradients flow only through the model's own decisions. Outcome-only rewards (binary correct/wrong on multi-hop QA gold).
- **Headline numbers.** **+41% (Qwen2.5-7B), +20% (Qwen2.5-3B)** over RAG baselines on HotpotQA / 2WikiMHQA / MuSiQue / NQ.
- **Why it matters.** The de facto recipe for "train your own multi-hop search agent." The repo's veRL setup is forked across most 2025–2026 follow-ups.
- **Repo.** [github.com/PeterGriffinJin/Search-R1](https://github.com/PeterGriffinJin/Search-R1) — 4,651★.

### R1-Searcher (Song et al. 2025)

- **Citation.** *R1-Searcher: Incentivizing the Search Capability in LLMs via RL*. arXiv 2503.05592. R1-Searcher++ at arXiv 2505.17005.
- **Mechanism.** **Two-stage outcome-only RL** (no SFT cold-start) — stage 1 teaches the format of the search action; stage 2 optimises for answer correctness with a search-format penalty. R1-Searcher++ adds a memory mechanism so retrievals are *internalised* into the model's parametric memory over time.
- **Headline numbers.** Beats GPT-4o-mini RAG.
- **Repo.** [github.com/RUCAIBox/R1-Searcher](https://github.com/RUCAIBox/R1-Searcher) — 710★.

### ReSearch (Hu et al. 2025)

- **Citation.** *Learning to Reason with Search for LLMs via RL*. arXiv 2503.19470.
- **Mechanism.** RL on Qwen2.5-7B/32B *without supervised reasoning data*; the model discovers `<search>` invocations from outcome rewards alone.
- **Headline numbers.** **+8.9 to +22.4 absolute** on multi-hop QA.
- **Repo.** [github.com/Agent-RL/ReSearch](https://github.com/Agent-RL/ReSearch) — 1,379★.

### DeepResearcher (Zheng et al. 2025)

- **Citation.** *DeepResearcher: Scaling Deep Research via RL in Real-world Environments*. arXiv 2504.03160, EMNLP 2025.
- **Mechanism.** End-to-end RL training in **real web environments** (not closed corpora). Captures the noisy / partial / paywalled / robots-blocked reality of the open web.
- **Headline numbers.** **+28.9** over prompting baselines, **+7.2** over RAG-RL. Emergent behaviours: planning, cross-validation, self-reflection, *honesty about limits*.
- **Repo.** [github.com/GAIR-NLP/DeepResearcher](https://github.com/GAIR-NLP/DeepResearcher) — 747★.

### WebThinker (Li et al. 2025)

- **Citation.** *WebThinker: Empowering Large Reasoning Models with Deep Research Capability*. arXiv 2504.21776, NeurIPS 2025.
- **Mechanism.** **Think-Search-Draft** loop with a Deep Web Explorer module. Two modes — problem-solving and report-generation.
- **Headline.** Beats proprietary deep-research systems on GPQA, GAIA, WebWalkerQA, HLE.
- **Repo.** [github.com/RUC-NLPIR/WebThinker](https://github.com/RUC-NLPIR/WebThinker) — 1,440★.

### Agentic-R1 (Du et al. 2025) — DualDistill

- **Citation.** *Agentic-R1: Distilling Tool-Aware Reasoning into a Single Student*. arXiv 2507.05707, EMNLP 2025.
- **Mechanism.** **DualDistill** distils a tool-using teacher (Claude-3.5-Sonnet) and a text-reasoning teacher (DeepSeek-R1) into a single student that **routes per-query** between modes — "is this question answerable from parametric memory or does it need search?"

### RAGEN (Wang et al. 2025) and StarPO

- **Citation.** *RAGEN: Understanding Self-Evolution in LLM Agents via Multi-Turn Reinforcement Learning*. arXiv 2504.20073. RAGEN-2 (arXiv 2604.06268) studies *reasoning collapse* with conditional-entropy / mutual-information metrics.
- **Mechanism.** **StarPO** = State-Thinking-Actions-Reward Policy Optimization for multi-turn RL agent training; identifies and fixes the "**Echo Trap**" reasoning collapse where the agent's chain-of-thought degrades over training into repeated boilerplate.
- **Repo.** [github.com/RAGEN-AI/RAGEN](https://github.com/RAGEN-AI/RAGEN) — 2,650★.

### Tongyi DeepResearch / WebSailor / WebDancer (Alibaba 2025)

- **Citations.** *WebDancer* (arXiv 2505.22648); *WebSailor V1/V2* (arXiv 2507.02592 / 2509.13305).
- **Mechanism.** Production-scale GRPO + retrieved-token masking, executed on real web environments and federated tools. Currently SOTA on HLE / BrowseComp / FRAMES / SimpleQA.
- **Repo.** [github.com/Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch) — 18.8k★.
- **Cross-link.** Driver of [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md) §7's "single move that matters most."

### ARM — Agentic Reasoning Modules (Oct 2025)

- **Citation.** arXiv 2510.05746.
- **Mechanism.** **Discovers ARMs by tree search over code space** starting from a CoT seed and mutating via reflection. The discovered modules outperform manually designed multi-agent systems.
- **Why it matters.** Multi-hop *architecture* itself becomes a search problem. Adjacent to [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md), [167-autoskill-experience-driven-lifelong-learning](167-autoskill-experience-driven-lifelong-learning.md).

## Empirical results — comparing across phases

A representative slice on MuSiQue-Ans (F1, comparable runs collated from each paper's main table; small variations in retrieval corpora and seeds):

| System | Backbone | F1 |
|---|---|---|
| Single-shot RAG (BM25 + Llama-3-70B) | Llama-3-70B | ~25 |
| Self-Ask + GPT-3.5 | GPT-3.5 | ~30 |
| IRCoT + GPT-3.5 | GPT-3.5 | ~40 |
| Iter-RetGen + GPT-3.5 | GPT-3.5 | ~38 |
| DSPy MultiHop + ColBERTv2 + GPT-4o | GPT-4o | ~55 |
| HippoRAG (single retrieval) | GPT-4o | ~58 |
| Beam Retrieval + reader | base + reader | ~64 |
| Search-R1 | Qwen2.5-7B | ~65 |
| WebThinker | Qwen2.5-32B | ~68 |

Read these as *trend signals* not exact apples-to-apples — corpora and retrievers differ across rows. The pattern that holds: **prompted < declarative ≤ reflective < trained**, holding model size constant.

## Variants and ablations worth knowing

- **R²AG / HopRAG / RELOOP** — 2025 expressions of the same "search tree over retrieval/reasoning, learn to navigate" pattern. R²AG: RL-trained retrieval steering over a multi-hop tree (+24.1% recall, +20.4% accuracy). HopRAG: LLM-pseudo-query edges between passages. RELOOP: recursive retrieval with multi-hop reasoner-planner stack.
- **Search-o1's Reason-in-Documents** — denoise the retrieved passage with a *separate* LM call before injection. Composes with anything.
- **Decomposition fan-out** — LlamaIndex `SubQuestionQueryEngine` (parallel) vs `MultiStepQueryEngine` (sequential). The former wins on FanOutQA, the latter on MuSiQue.
- **Reflection-on-empty-retrieval** — what does the agent do when the retriever returns nothing? CRAG's "incorrect → web fallback" branch is the canonical answer.
- **Retrieval-token masking** — the gradient-side trick that makes Search-R1 work. Without it, the policy memorises corpus text and stops actually searching.

## Failure modes and limitations

- **Echo Trap (RAGEN)** — multi-turn RL on multi-hop trajectories collapses the chain-of-thought into repeated boilerplate without entropy regularisation.
- **Retrieval-text leakage** — without retrieved-token masking, the model "learns" to predict the next retrieved fact instead of issuing a real search.
- **Reflection-token reward hacking** — Self-RAG-style models can issue `Retrieve` excessively when the reward shapes incentive that way; CRAG's quality gate partially fixes this.
- **Multi-hop with bad retriever = worse than one-hop** — when retrieval is poor, every additional hop *amplifies* the noise. Always co-track retrieval recall@k.
- **Cost blow-up** — deeply iterated trained agents (Tongyi Heavy mode, WebThinker max-depth) consume 30–100× the tokens of a single-shot baseline. Cf. [86-frugalgpt](86-frugalgpt.md), [88-confidence-driven-router](88-confidence-driven-router.md).
- **Out-of-domain generalisation** — Search-R1 / R1-Searcher models trained on Wikipedia QA underperform on news QA. The training domain bleeds into the search policy.
- **Tool-use sycophancy** — multi-hop agents trained against an automated judge can learn to produce confident-looking chains that the judge accepts even when wrong. AgentPRM / HiPRAG rubric scaffolding mitigates this; see [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md) Tier 2.

## When to use, when not

**Use Phase 4 (trained search policies)** when (a) you control or can fine-tune a 7B–32B base model, (b) your domain has plentiful gold multi-hop QA pairs, (c) latency at inference is dominated by search calls, not LM forward passes. The Search-R1 / Tongyi recipe pays back the months of engineering with double-digit gains.

**Use Phase 2 (DSPy declarative)** when you can't fine-tune (closed-API base) but have a small held-out eval set. The compiled multi-hop program will dominate any hand-prompted alternative.

**Use Phase 3 (reflective)** when adaptive retrieval is the binding constraint — many "should I retrieve" decisions per query, heterogeneous sources, mixed quality. Self-RAG/CRAG/CoN compose with anything else.

**Use Phase 1 (prompted)** when you're prototyping or when the LM is so strong that it doesn't need an optimiser. Self-Ask is still a great smoke test for new base models.

**Don't blindly stack** — a Phase-4 trained policy often *replaces* the reflective layer (the model has internalised its quality gate). Stacking trained + reflective frequently regresses.

## Implications for harness engineering

- **Action-token discipline.** The harness must validate `<search>...</search>` / `<answer>...</answer>` tokens deterministically — any deviation should fail closed. See [05-hooks](05-hooks.md), [06-permission-modes](06-permission-modes.md).
- **Retrieved-token masking at training time.** If you're training a multi-hop search agent, masking the retrieved tokens from the loss is non-negotiable. Bake it into the training-loop scaffolding alongside [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md).
- **Per-hop verifier.** Inject a Chain-of-Note-style per-doc filter between retrieval and reasoning ([18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md)). Cheap, composable, and catches noise before it amplifies.
- **DSPy-style program registry.** Treat the multi-hop pipeline as a *program* with a versioned signature, optimised against a held-out eval. The optimiser-not-prompt mindset extends to [12-todo-scratchpad-state](12-todo-scratchpad-state.md) and [04-skills](04-skills.md).
- **RL for search, not for reasoning.** Outcome-only RL on the *search* policy is now a well-trodden path; outcome-only RL on the *reasoning* is much harder and usually unnecessary. Start with the former. Cf. [80-knowrl](80-knowrl.md), [81-reasoningbank](81-reasoningbank.md).
- **Decomposition cache.** Sub-question decompositions repeat across queries; persist them in [09-memory-files](09-memory-files.md) under a normalised question key.
- **Test-time-compute curves.** Plot answer-quality vs thinking-token budget and stop at the inflection point — SealQA-style noise can make the curve *invert* past a threshold. Cf. [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md).
- **Cost-aware orchestrator.** Multi-hop chains burn tokens nonlinearly; instrument $/query and gate at the orchestrator. Cf. [124-agent-level-production-patterns](124-agent-level-production-patterns.md).
- **Echo-Trap monitor.** During RL fine-tuning, log conditional entropy of the chain-of-thought; spike-detect for collapse. Cf. RAGEN-2.
- **Honesty rewards.** DeepResearcher's emergent honesty (admitting limits) came for free from outcome-only RL in real environments — replicating this needs a noisy environment, not curated synthetic QA. Cf. [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md).
- **Composable retrieval gates.** CRAG-style quality gates compose with HippoRAG-style graph retrieval; stack them. See [200-graph-grounded-multi-hop-retrieval](200-graph-grounded-multi-hop-retrieval.md).
- **Evaluation pinning.** Multi-hop pipelines are extremely sensitive to retrieval-corpus drift. Pin the corpus per release. Cf. [115-evaluating-llm-systems](115-evaluating-llm-systems.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md).

**The one-line takeaway for harness designers:** Move along the arc — Phase 1 to demo, Phase 2 to ship, Phase 3 to harden, Phase 4 only when you control a tunable base model and have outcome rewards you trust.
