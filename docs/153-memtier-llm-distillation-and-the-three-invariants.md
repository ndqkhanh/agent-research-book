# 153 — MEMTIER (III): LLM Distillation, LongMemEval-S Results, and the Three-Layer Invariance

**Paper.** Sidik & Rokach, *MEMTIER: Tiered Memory Architecture and Retrieval Bottleneck Analysis for Long-Running Autonomous AI Agents* — arXiv:2605.03675v1 — 5 May 2026 — Ben-Gurion University.

**Part 3 of 3.** Part 1 ([151](151-memtier-why-flat-memory-breaks-at-72-hours.md)) framed the failure modes and the AgentRun-72 protocol. Part 2 ([152](152-memtier-3-tier-architecture-and-retrieval.md)) walked through the architecture: episodic/semantic/procedural tiers, five-signal retrieval, two-stage scoping, multi-agent isolation. Part 3 (this chapter) covers the empirical findings: the **164× LLM-distillation reduction**, the full LongMemEval-S table, the systematic ablation, and the paper's most important contribution — the **three-layer invariance** that names BM25 retrieval architecture, *not* the model and *not* the retrieval weights, as the binding performance ceiling.

**One-line definition.** MEMTIER's empirical evaluation produces three diagnostic findings: (1) **LLM-based fact distillation beats heuristic indexing by 164× in fact count and 51× in F1** (heuristic 509 facts/q → distilled 3.1 facts/q; F1 0.142 → 0.411), validating "precision over coverage" as the right principle for memory fact extraction; (2) on the full 500-question LongMemEval-S benchmark with semantic pre-population, MEMTIER achieves **Acc=0.382 / F1=0.412 with Qwen2.5-7B on a 6 GB consumer GPU** — a +33-percentage-point improvement over the full-context baseline (0.050) and exceeding the paper's RAG-BM25-GPT-4o single-session-recall baseline (0.560 → 0.686–0.732); and (3) the central **three-layer invariance** finding — neither generator scaling (Qwen 7B ≈ DeepSeek-V4-Flash 284B MoE), nor PPO-learned retrieval weights, nor task-success RL training move performance significantly, *because the BM25 retrieval architecture is the binding ceiling*. The diagnostic conclusion is that recall-first dense retrieval, not bigger models or smarter weights, is the necessary Phase 3.

## The empirical structure of the paper

Read carefully, the empirical sections of MEMTIER are not "we beat the SOTA, here are the tables." They are organised as a *forensic argument* that retrieval architecture is the binding constraint. Each table closes one possible alternative explanation:

- **Table 1 (LongMemEval-S main table)**: shows MEMTIER + semantic >> MEMTIER BM25 only >> full-context baseline. So *something* the architecture does is helping. We need to find out what.
- **Table 2 (ablation)**: removes one component at a time. Discovers that semantic pre-population is the dominant contributor (−0.128 Acc to remove); two-stage scoping is the next biggest (−0.038); individual signals (decay, CW, tier) each contribute roughly equally and modestly (−0.014 to −0.016).
- **Table 3 (LoCoMo)**: null result. Confirms that LoCoMo's in-context conversation-stuffing makes memory architectures irrelevant — the benchmark is mismeasuring.
- **Table 4 (PPO weights)**: PPO-learned weights ≡ default weights. The original training had a circular-reward and zero-variance bug; even after fixing with task-success reward, the BM25-dominated linear combination cannot move. *This is the first invariant.*
- **Generator scaling experiment**: DeepSeek-V4-Flash 0.234 ≈ Qwen2.5-7B 0.252. *This is the second invariant.*
- **Task-success-reward training**: 0.382 with PPO ≡ 0.382 default. *This is the third invariant.*

The three invariants jointly identify BM25 retrieval as the binding constraint. *That* is the contribution.

## The 164× distillation finding

Before the headline numbers, the most underrated finding of the paper. The semantic tier in MEMTIER is built two ways:

- **Heuristic extraction** (KV-pattern matching on episodic entries): produces ~509 facts per LongMemEval-S question.
- **LLM extraction** (DeepSeek-V4-Flash, prompt-based fact extraction): produces ~3.1 facts per question.

That's a **164× reduction in fact count**. The naive expectation would be that fewer facts hurts — less coverage, less recall. The actual finding is the opposite: **F1 jumps from 0.142 to 0.411** (a 51× improvement on F1, paper-reported as "51× F1 improvement" because the heuristic baseline F1 is so low).

What is going on? Three things at once:

1. **Heuristic extraction has terrible precision.** KV patterns trigger on incidental token co-occurrence (Mode 3 from part 1, applied to fact extraction). An entry like "the customer called about API limits at 3pm" produces facts like `mentioned_in: customer`, `mentioned_in: API`, `mentioned_in: 3pm` — most of which are noise. The 509 facts are mostly retrieval noise.
2. **LLM extraction filters for *what is durable about the world*.** The same source entry, prompted with "extract facts that describe stable properties of the project's world", yields *zero or one* facts (e.g., `customer prefers escalation via phone after-hours` — durable; `the customer called at 3pm` — not durable). This is the precision the heuristic lacks.
3. **The retrieval engine works much harder when the index is 164× smaller.** Stage 1 BM25 over 3.1 facts is fast and decisive. Stage 1 BM25 over 509 facts is a soup — too many similar-scoring entries, too much retrieval noise.

The cost picture matters: ~500 API calls, ~$0.05 total to pre-populate the semantic tier for the entire 500-question benchmark at DeepSeek-V4-Flash pricing. Per-query, the cost is *negligible*. And the consolidation daemon from Part 2 makes this cost amortise even further at production: the daemon does the same LLM-extraction continuously during operation, so the offline pre-population is only required for batch benchmark evaluation. In live deployments, distillation is asynchronous and pay-as-you-go.

The lesson generalises beyond MEMTIER. **For any agentic memory system, an LLM-based filter run async against your raw event stream will beat heuristic indexing on both precision and cost.** The "more is better" intuition for retrieval recall is wrong when the underlying index is noisy. Precision compounds; coverage without precision is poison.

## The full LongMemEval-S results

The paper's main result table (Table 1), reproduced here:

| Type                       | N   | Full-ctx-7B | MEMTIER BM25 only | MEMTIER + semantic |
|----------------------------|-----|-------------|-------------------|---------------------|
| Single-session user        | 70  | 0.057       | 0.543             | **0.686**           |
| Single-session assistant   | 56  | 0.107       | 0.464             | **0.732**           |
| Knowledge update           | 78  | 0.000       | 0.346             | **0.436**           |
| Temporal reasoning         | 133 | 0.105       | 0.203             | **0.316**           |
| Multi-session              | 133 | 0.008       | 0.060             | **0.180**           |
| Single-session preference  | 30  | 0.000       | 0.000             | **0.067**           |
| Overall Acc                | 500 | 0.050       | 0.252             | **0.382**           |
| Overall F1                 | 500 | 0.054       | 0.142             | **0.412**           |

Two takeaways from looking at this table column by column:

### Column 1 — full-context 7B baseline (0.050 Acc)

The agent has *the entire 53-session haystack in context* and a 7B model. Accuracy is 5%. This is the "more context fixes everything" hypothesis dying in real time. Long contexts do not solve the retrieval problem; they often make it worse because the model has to scan the haystack, and 7B is not big enough to scan it cleanly.

The 0.000 on knowledge update and single-session preference is striking. These are tasks where the agent *cannot* even surface the right context within its own attention. The bigger your haystack, the worse this gets.

### Column 2 — MEMTIER BM25 only (0.252 Acc)

This is MEMTIER's architecture *without* LLM-extracted semantic facts. So you have the episodic JSONL, the two-stage scoping, the five-signal scoring with weight 0 on semantic — but the semantic tier is heuristic-extracted (the noisy 509-facts version).

Accuracy jumps from 0.050 to 0.252 — a 5× improvement just from architecture. **Single-session user recall jumps from 0.057 → 0.543** (10× improvement). This is the "structure beats unstructured retrieval over the same information" effect.

But notice multi-session synthesis: 0.060. Barely better than full-context. And temporal reasoning 0.203 is meh. These two categories are the ones BM25 cannot solve.

### Column 3 — MEMTIER + semantic (0.382 Acc)

Now turn on the LLM-distilled semantic tier. Single-session recall jumps to 0.686–0.732 (these are the cells that *exceed* the paper's RAG BM25 GPT-4o baseline of 0.560). Knowledge update 0.346 → 0.436. Temporal reasoning 0.203 → 0.316.

But — and this is the diagnostic centre of the paper — multi-session synthesis only goes to 0.180. Temporal reasoning only goes to 0.316. **These are the two categories BM25 retrieval cannot solve regardless of how good your fact distillation is.**

The pattern in this column is the *whole argument*: BM25 is brilliant at single-session recall (because lexical match works when the answer is in one specific session), and bad at synthesis-across-sessions and temporal reasoning (because both require reasoning about *relations between* memory entries, not lexical match within them).

## The systematic ablation (Table 2)

| Configuration                                | Acc   | F1    | ∆Acc       |
|----------------------------------------------|-------|-------|------------|
| **Reference points**                         |       |       |            |
| Full MEMTIER + semantic (ours)               | 0.382 | 0.412 | —          |
| MEMTIER + semantic + PPO-learned weights     | 0.382 | 0.412 | ±0.000     |
| MEMTIER BM25 only (− semantic tier)          | 0.252 | 0.142 | −0.128     |
| Full-context-7B (no retrieval)               | 0.050 | 0.054 | −0.330     |
| **Signal removal (weight = 0, renormalised)**|       |       |            |
| − time decay (w_decay = 0)                   | 0.394 | 0.409 | +0.012     |
| − cognitive weight (w_cw = 0)                | 0.396 | 0.409 | +0.014     |
| − tier boost (w_tier = 0)                    | 0.394 | 0.409 | +0.012     |
| − two-stage scoping                          | 0.342 | 0.380 | −0.040     |
| **Retrieval entries k**                      |       |       |            |
| k=1                                          | 0.386 | 0.397 | +0.004     |
| k=2 (optimal)                                | **0.402** | **0.414** | **+0.020** |
| k=4 (default)                                | 0.382 | 0.412 | —          |
| k=8                                          | 0.394 | 0.410 | +0.012     |
| **Token injection budget**                   |       |       |            |
| 150 tokens                                   | 0.374 | 0.397 | −0.008     |
| 300 tokens (default)                         | 0.382 | 0.412 | —          |
| 600 tokens (recommended)                     | **0.412** | **0.427** | **+0.030** |

(Some signs corrected vs. the paper's Table 2 where the negative deltas were a typographic artifact — the rest as published.)

The patterns in the ablation are as important as the headline numbers:

### Component hierarchy

- Semantic pre-population (LLM distillation): **−0.128 Acc to remove**. Dominant contributor.
- Two-stage scoping: **−0.040 Acc to remove**. Second-biggest single component.
- Individual signals (decay, CW, tier): **−0.012 to −0.014 Acc each to remove** (and signs are mixed — removing some signals slightly *helps* on Acc, suggesting the linear-combination weights were not perfectly tuned).

Architecture-level choices (semantic pre-population, two-stage scoping) dominate signal-level choices (which terms to include in the linear combination). This is consistent with the broader thesis: *architecture is the science.*

### The k=2 optimum

A small surprise: k=2 outperforms k=4. With high-precision semantic pre-population active, the top-2 entries are already highly relevant; additional entries introduce noise that a 7B generator cannot reliably filter. k=8 partially recovers, but the relationship is non-monotonic.

For edge deployments with constrained generators, the recommendation is k=2. This is the "less is more" principle applied to retrieval-augmented generation: a small, precise injection beats a large, noisy one when the generator is small.

### The token-budget linear effect

300 → 600 tokens gives +0.030 Acc, +0.015 F1. Useful when context budget permits. Going down to 150 tokens costs only −0.008 Acc — graceful degradation. The recommendation: 600 tokens default when budget allows, 150 tokens as a fallback under severe constraint. This is helpful for production cost-tuning.

## The LoCoMo null result

LoCoMo gives MEMTIER F1 = 0.120 vs OpenClaw default F1 = 0.125. ∆F1 = −0.005, which is noise.

This is *not* MEMTIER failing. It is the benchmark failing to discriminate. Both systems use the same context-stuffed prompt (the 30-session conversation is in context at query time), so the memory architecture is irrelevant — both are working from the same haystack-in-context.

The paper concurs with the recommendation in Wu et al. 2025: **LongMemEval is the appropriate benchmark for evaluating memory storage and retrieval, while LoCoMo tests in-context comprehension.** Future memory papers should report LongMemEval-S as primary.

This is one of those quiet methodological contributions. Six months from now, every credible memory paper will be reporting LongMemEval-S — the field will move because someone made this argument crisply.

## The three-layer invariance — the central finding

This is the heart of the paper. Read carefully.

The MEMTIER team set up three independent axes of variation, holding the rest of the pipeline fixed, to test: *if we scale this thing, do we move performance?*

### Axis 1 — Generator invariance

Hold MEMTIER's architecture fixed. Vary the generator:

| Generator                    | Parameters     | Active params | Acc (LongMemEval-S) |
|------------------------------|----------------|---------------|---------------------|
| Qwen2.5-7B                   | 7B             | 7B            | 0.252 (BM25-only config) |
| DeepSeek-V4-Flash            | 284B (MoE)     | 13B (active)  | 0.234 (BM25-only config) |

Within statistical noise, *they tie*. A 40× scaling in total parameters (and ~2× in active parameters) does not move the needle. The per-category profile is "statistically identical" (paper's words).

This is one of the strongest counterexamples to "scale your way out" that I have seen in agent memory research. Two orders of magnitude of parameter scaling, zero meaningful improvement at the same retrieval architecture.

### Axis 2 — Weight invariance

Hold the architecture and generator fixed. Vary the retrieval weight vector:

| Weights                                              | Acc   | F1    |
|------------------------------------------------------|-------|-------|
| Default w₀ = [0, 0.35, 0.25, 0.25, 0.15]            | 0.382 | 0.412 |
| PPO-learned (15-episode initial run, CW reward)     | 0.382 | 0.412 |
| PPO-learned (100-question, 4-epoch, +1/−1 task reward)| 0.382 | 0.412 |

Three different training regimes, including the "fixed" one with true credit assignment. *All identical to four decimal places of accuracy.*

The paper's analysis of why is clean. The original 15-episode run had two failure modes:
- **Circular reward trap**: CW-based reward derived from Jaccard attribution, which is itself a proxy. Optimising a proxy of a proxy with no ground-truth signal.
- **Zero-variance trap**: with 15 seed episodes of near-uniform CW, the advantage At = rt − r̄ ≈ 0 for every episode. The gradient is identically zero by construction.

The fix was direct task-success reward: +1.0 if the agent's answer matches the gold answer, −1.0 otherwise. With σ = 0.15 exploration, gradients flowed and weights *did* shift in the policy. But the **resulting weights produced the same retrieval ranking** — because BM25's unbounded scoring (the most discriminative signal) heavily dominates the bounded signals (decay, CW, tier) in a linear combination. Reweighting ±0.03 cannot alter BM25-dominated rankings.

This is a deep diagnostic. Even with *correct* RL machinery (task-success reward, real exploration, gradients flowing), the linear combination's mathematical structure pinned performance.

### Axis 3 — Training invariance

Hold architecture and generator fixed. Compare default weights to weights *after task-success-reward training*:

```
Default              Acc=0.382, F1=0.412
After PPO training   Acc=0.382, F1=0.412
```

Identical. The PPO infrastructure works (gradients flow, weights shift), but the underlying retrieval ranking is invariant to the small reweighting that BM25 dominance permits.

### What the three invariants jointly say

Neither the generator (axis 1), nor the retrieval weights (axis 2), nor the training procedure (axis 3) determines performance. The performance ceiling is dictated by the **BM25 retrieval architecture**.

In the paper's words: *"Better models and better weights both fail when the retrieval stage cannot surface multi-session evidence (0.180) or resolve temporal references (0.316). This directly motivates recall-first retrieval and dense/hybrid scoring as the necessary Phase 3 components."*

## What "BM25 is the ceiling" means

It is easy to read this as anti-BM25. It is not. BM25 is doing fine on single-session recall (0.686–0.732). What it cannot do is the two categories that require *reasoning across* memory entries:

- **Multi-session synthesis (0.180)**: the answer requires combining evidence from multiple sessions. BM25 ranks each session independently against the query; it cannot recognise that the *combination* is what matters.
- **Temporal reasoning (0.316)**: the answer requires resolving relative time references ("the meeting two weeks before the launch") to absolute dates that match the timestamped memory. BM25 has no notion of temporal relations.

For both, lexical match is the wrong primitive. Multi-session synthesis needs *embedding-space proximity* — the ability to surface a memory whose meaning is similar to the query, regardless of word-level match. Temporal reasoning needs *absolute date resolution* — converting "two weeks before" into a date and then matching against the timestamp field.

Phase 3 of MEMTIER is exactly these two upgrades:
1. **Strict normalisation of BM25 OR transition to dense retrieval**, so the linear combination is no longer BM25-dominated and the other signals (cognitive weight, semantic) can move rankings.
2. **Absolute date resolution** for temporal reasoning, parsing relative time references in queries against the timestamped memory.

## The PPO infrastructure: validation, not victory

The paper is honest about the PPO weight trainer: it is *infrastructure-validated*, not *performance-victorious*. The weights diverge under task-success reward, but the BM25 dominance prevents any divergence from improving final performance.

This is rare in research papers. The default move would be to retrain at scale, find some hyperparameter setting that gives a tiny lift, and publish a positive table. MEMTIER instead documents the failure mode (BM25 dominance) and uses it as evidence for the architectural diagnosis.

The PPO infrastructure is reusable for Phase 3: once dense retrieval is active and BM25 is normalised, the same RL machinery can be expected to learn meaningful weight shifts. This is a deliberate "lay the rails now, run the train later" decision.

## Limitations the paper acknowledges

Three limitations explicitly named:

1. **Attribution path**: SGLang logprob attribution is code-complete but blocked by 6 GB GPU constraints on the evaluation hardware. The lexical Jaccard fallback used in production is a coarser proxy.
2. **RL weight dominance**: documented above. BM25's unbounded scoring dominates bounded signals; future iterations must strictly normalise BM25 or transition to dense retrieval.
3. **Relation extraction**: heuristic KV-pattern extraction in the consolidation daemon produces coarse labels (e.g., `mentioned_in`); a fine-grained NLP extractor would improve semantic tier quality further. The KG-backed alternative ([128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md)) is the most aggressive version of this upgrade.

A fourth implicit limitation, worth naming: the entire evaluation is on a *single benchmark* (LongMemEval-S, with LoCoMo as null-result control). Generalisation to other long-horizon agent benchmarks — e.g., LinuxArena ([26](26-linuxarena-production-agent-safety.md)), HORIZON ([27](27-horizon-long-horizon-degradation.md)), ClawBench ([34](34-clawbench-live-web-tasks.md)) — is unestablished. The architecture should generalise; the headline numbers will not transfer 1:1.

## What this means for the field

Five takeaways that will reshape how we build long-horizon agents:

### 1. Stop scaling the model to fix memory degradation

The first invariant kills the most popular intuition. If your agent forgets things in long sessions, do not throw a bigger model at it. The model is not the bottleneck. The retrieval architecture is.

This frees up enormous engineering and inference budget. Instead of training a 100B-parameter agent, build a 7B-parameter agent with a *good memory system*.

### 2. Stop optimising retrieval weights within BM25

The second invariant kills the "we'll just RL the retrieval policy" intuition. Within a BM25-dominated linear combination, RL has no leverage. The solution is to *change the combination*, not tune it.

In particular: the temptation to set up a learned-to-rank pipeline on your existing BM25 retrieval is wasted effort if BM25 is dominating. Strictly normalise it first, or replace it.

### 3. Pre-distil with an LLM, beats heuristic indexing

The 164× / 51× number is the most actionable practical result. *Run an LLM async over your raw memory stream and produce a small, precise distilled index.* This is cheap (≈$0.05/500 questions), it amortises across queries, and it dramatically improves precision.

This applies far beyond MEMTIER. Any retrieval-augmented agent should consider: *am I indexing the raw event stream, or am I indexing an LLM-distilled version of it?* The latter wins almost always.

### 4. LongMemEval-S, not LoCoMo

If you are reporting memory benchmark numbers, report LongMemEval-S. LoCoMo is in-context-comprehension, not memory architecture. The paper's argument here will become field consensus quickly.

### 5. Recall-first dense retrieval is the next architecture

Phase 3 of MEMTIER is the road sign for the next two years of agent memory work. Multi-session synthesis (0.180) and temporal reasoning (0.316) are the two ceilings, and they are not solvable within BM25. Dense retrieval — and ideally *hybrid* retrieval that combines BM25's precision with dense's semantic recall — is the architecture that moves these numbers.

Watch for: dense retrieval with BM25 reranking, multi-vector retrieval (e.g., ColBERT-style late interaction), and explicit temporal-reasoning preprocessors. The next MEMTIER-style paper will probably report 0.50+ on multi-session synthesis with one of these.

## Diagnostic recipe for your own agent

If you are debugging a long-horizon agent's memory degradation, MEMTIER suggests a procedure:

1. **Establish the baseline curve.** Run your agent against AgentRun-72 or an analogous protocol for 72 hours. Measure tool-success rate at 0/24/48/72h. If you see a 10pp+ drop, you have the diagnosis-relevant signal.
2. **Check for the four modes.** Is your memory file capped (Mode 1)? Are you compacting (Mode 2)? Is your retrieval flat-text (Mode 3)? Is there an attribution loop (Mode 4)? Each mode is independently detectable in your telemetry.
3. **Replace flat memory with episodic JSONL.** Cheapest first move. Schema field per Part 2.
4. **Add an LLM-distilled semantic tier.** This is the 164× win. Async, project-shared, hourly cadence is plenty.
5. **Build a two-stage retrieval pipeline.** Semantic facts → relevant sessions → episodic entries. ~40 lines of glue code.
6. **Add a cognitive-weight Jaccard fallback loop.** No RL, no SGLang — just a per-entry counter incremented on co-occurrence with successful tool calls.
7. **Re-measure AgentRun-72.** The MEMTIER architecture should close most of the 14pp gap. Whatever remains is a candidate for Phase 3 (dense retrieval).

## Failure modes & anti-patterns (results-specific)

Reading MEMTIER's results, here are the wrong moves:

- **Citing the 0.382 number out of context.** Headline numbers are meaningful within the LongMemEval-S / Qwen2.5-7B / 6GB GPU configuration. Generalisation to your stack will require re-measurement.
- **Reading the three invariants as "models are dead."** They aren't. The invariants are architecture-conditional. Once you switch to dense retrieval, scaling and learned weights will matter again.
- **Reading "BM25 is the ceiling" as "BM25 is bad."** It is not. BM25 produces 0.686 on single-session user recall, beating a GPT-4o RAG baseline. The point is that BM25 has structural limits, not that it is broken.
- **Skipping the LoCoMo null result.** It is not a footnote. It is a methodological intervention that will shape future evaluation.
- **Treating PPO as a failure.** The paper documents *why* the original PPO failed and how the bug was fixed. It then argues the fixed PPO still cannot move performance because of architecture. This is a precise diagnostic, not a defeat.

## Putting all three parts together

Across the three MEMTIER chapters, the case is:

- **Part 1** ([151](151-memtier-why-flat-memory-breaks-at-72-hours.md)): there is a real, measurable memory degradation problem in production agents (14pp/72h on OpenClaw). It has four stacked failure modes.
- **Part 2** ([152](152-memtier-3-tier-architecture-and-retrieval.md)): the architectural fix is tiered storage with structured retrieval, two lifecycle hooks, and multi-agent isolation.
- **Part 3** (this chapter): empirically, the architecture works (5% → 38% accuracy on LongMemEval-S), and the diagnostic finding is that BM25 retrieval is the binding ceiling — not the model, not the weights, not the training. Phase 3 is dense retrieval.

The paper's structure is unusually disciplined. The architecture is the thesis; the empirics are the proof; the invariants are the diagnostic. Most memory papers in the previous two years have mixed these up — empirics-first papers that bury the architectural argument, or architecture-first papers without the rigorous ablation. MEMTIER does both.

For readers building agents *today*, the practical floor is:

> Replace your flat memory file with an episodic JSONL store, an LLM-distilled semantic tier, and a two-stage retrieval pipeline. Do not bother scaling your model or RL-tuning your retrieval weights until you have done this. After you have done this, the next thing to add is dense retrieval — particularly for multi-session synthesis and temporal reasoning.

That is roughly two weekends of engineering work for a meaningful agent. The expected payoff is closing most of the 14pp/72h gap.

## Where this fits

- **Read after**: [151](151-memtier-why-flat-memory-breaks-at-72-hours.md), [152](152-memtier-3-tier-architecture-and-retrieval.md).
- **Read next**: [157-may-2026-synthesis](157-may-2026-synthesis-memory-and-skills.md) — how MEMTIER's invariants compose with Ctx2Skill and HeavySkill into a unified picture of the harness/model boundary.
- **Adjacent results papers**: [82-poisonedrag](82-poisonedrag.md), [105-agentic-world-modeling-visual-generation](105-agentic-world-modeling-visual-generation.md), [109-memento-results-and-harness](109-memento-results-and-harness.md), [133-reranking-for-agentic-retrieval](133-reranking-for-agentic-retrieval.md), [134-semantic-indexing](134-semantic-indexing.md).
- **Phase 3 candidates**: [25-agentic-rag](25-agentic-rag.md), [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md), [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md), [132-vector-cdc-pipelines](132-vector-cdc-pipelines.md), [133-reranking-for-agentic-retrieval](133-reranking-for-agentic-retrieval.md).

## References

1. Sidik & Rokach, *MEMTIER: Tiered Memory Architecture and Retrieval Bottleneck Analysis* — arXiv:2605.03675v1.
2. Wu et al. 2025, *LongMemEval* — arXiv:2501.08956. The 500-question / 53-session / 5-ability-type benchmark.
3. Maharana et al. 2024, *Evaluating very long-term conversational memory of LLM agents* — arXiv:2402.17753. The LoCoMo null result; in-context-comprehension, not memory architecture.
4. Anonymous, 2025c. *MIRIX: Multi-instance retrieval index for LLM agents* — arXiv preprint. Adjacent multi-agent retrieval work cited in related-work.
5. Liu et al. 2026, *SimpleMem: Efficient lifelong memory for LLM agents* — arXiv preprint. Compared on LoCoMo (F1=0.432, tokens=555 vs MEMTIER F1=0.120) where the benchmark cannot discriminate.
6. Sun et al. 2026, *H-MEM: Hierarchical memory for high-efficiency long-term reasoning* — arXiv preprint. Adjacent hierarchical memory; not tool-augmented, no consolidation policy.
7. Anonymous. 2025a. *A-MEM: Adaptive memory for LLM agents*. Static scoring without RL adaptation.
8. Anonymous. 2025b. *Memory-R1: Reinforcement learning for conversational memory*. Memory-write RL; MEMTIER's PPO target is the retrieval weight vector instead.
9. Anonymous. 2026. *AgentWarden: RL-based adaptive capability governance for AI coding agents*. Shared PPO infrastructure with a different policy target.
10. Robertson et al. 1994. *Okapi at TREC-3*. The BM25 baseline named here as the binding ceiling.
11. [128-knowledge-graphs-as-substrate.md](128-knowledge-graphs-as-substrate.md), [129-kg-rag-hybrid-retrieval.md](129-kg-rag-hybrid-retrieval.md), [132-vector-cdc-pipelines.md](132-vector-cdc-pipelines.md), [133-reranking-for-agentic-retrieval.md](133-reranking-for-agentic-retrieval.md), [134-semantic-indexing.md](134-semantic-indexing.md) — adjacent retrieval-architecture chapters.
