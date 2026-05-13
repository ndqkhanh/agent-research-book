# 109 — Memento Results, Ablations, and a Critical Reading: What Top-1 GAIA Without Fine-Tuning Actually Buys, and How to Bolt CBR Memory Onto an Existing Harness

**Source.** Zhou et al., *Memento: Fine-tuning LLM Agents without Fine-tuning LLMs*, arXiv:2508.16153v2 (Aug 2025); repository https://github.com/Agent-on-the-Fly/Memento (mirror https://github.com/Memento-Teams/Memento). This file extends [106](106-memento-paper-theory.md), [107](107-memento-cbr-memory.md), [108](108-memento-codebase-mcp.md) with the empirical results, the ablations, and the harness-integration playbook — read together with [100](100-contextual-memory-is-a-memo.md), which is the strongest theoretical critique of Memento's class of system.

**One-line definition.** Memento posts top-1 GAIA validation (87.88% Pass@3) and test (79.40%), 66.6 F1 / 80.4 PM on DeepResearcher (beating training-based SOTA), 95.0% SimpleQA, 24.4% HLE, and +4.7 to +9.6 absolute OOD gain — all with a frozen LLM and a small dual-encoder retriever — but read against [100](100-contextual-memory-is-a-memo.md)'s Compositional Sample Complexity Separation theorem, those wins describe the *upper bound of what's achievable inside C-engineering*, not the end of the road; this file lays out the numbers, the ablations, the head-to-head with the [100] critique, and a concrete playbook for bolting Memento-style CBR memory onto an existing harness without rewriting it.

## Why this synthesis matters

Memento's headline numbers are unusually clean: top-1 GAIA on the leaderboard, beating training-based DeepResearcher SOTA, with no LLM gradient updates. This is the strongest empirical evidence to date that *case-conditioned in-context learning, with a learned retriever, can match or exceed parameter-update-based agent learning on standard benchmarks*.

But the paper sits in tension with [100](100-contextual-memory-is-a-memo.md)'s theoretical argument that retrieval-based memory is bounded by ᾱ < 1 on compositionally novel inputs and requires Ω(k²) coverage to close to 1 − δ. Both can be true simultaneously: Memento can be SOTA *within the class of frozen-LLM systems* while [100]'s separation theorem still applies on truly novel compositions. The careful read is "Memento is the best C-engineering system published; the bound is real but not yet binding on these benchmarks."

This synthesis matters for harness engineers because it tells you exactly *what kind of improvement* CBR memory delivers and exactly *what it does not*. The numbers point at a concrete integration pattern that is incremental, low-risk, and high-leverage for any deep-research-adjacent harness.

## Problem it solves

Three concrete questions for the harness engineer:

1. **Is it worth integrating?** What size of empirical lift should you expect on tasks adjacent to GAIA / DeepResearcher / SimpleQA?
2. **Does it generalise OOD?** Reflexion-style C-engineering frequently degrades on distribution shift; does CBR?
3. **What does it cost?** Operational complexity of deploying and maintaining the retriever, the bank, the post-episode hook, the offline retraining cadence.

This file answers each with the paper's empirical evidence, then provides a concrete integration playbook.

## Core idea in one paragraph

Memento delivers SOTA without fine-tuning, with the gain concentrated where it matters most for harness engineers: out-of-distribution tasks. The K=4 ablation is the cleanest single empirical signal — small, high-quality memory beats large banks. Integration onto an existing harness is a four-step retrofit: (i) define a structured `(s_T, a_T, r_T)` case schema, (ii) add a post-episode `MemoryWriter` hook with reward thresholding, (iii) deploy a dual-encoder retriever as a sidecar service with offline retraining, (iv) inject top-K=4 cases into the planner's prompt only. None of these steps require touching the underlying LLM. The bounds from [100](100-contextual-memory-is-a-memo.md) still apply — Memento is C-engineering, and the Ω(k²) coverage demand on compositional novelty is unaffected by retriever quality — but inside that envelope, CBR is the strongest published return on engineering effort for frozen-LLM agents.

## Mechanism (step by step) — results, ablations, then integration

### 1. The headline benchmark numbers

| Benchmark | Metric | Memento Result | Notes |
|---|---|---|---|
| GAIA | Validation, Pass@3 Top-1 | **87.88%** | Top-1 on the leaderboard at submission time. |
| GAIA | Test (held-out) | **79.40%** | Top-1 on test; the gap from val to test (~8 pts) reflects standard generalisation drop. |
| DeepResearcher | F1 | **66.6%** | Beats DeepResearcher's own training-based RL agent. |
| DeepResearcher | PM (passage match) | **80.4%** | Same. |
| DeepResearcher | OOD F1 / PM gain | **+4.7 to +9.6 absolute** | The strongest evidence that CBR generalises out-of-distribution. |
| SimpleQA | Accuracy | **95.0%** | Saturated benchmark; useful as a sanity check. |
| HLE (Humanity's Last Exam) | Performance Metric | **24.4%** | Low absolute score; the paper attributes the ceiling to tool capability, not memory quality. |

What is *not* in the table is also informative:
- **No GAIA Level breakdown is publicly headlined**, though the repo notes that Level-3 (the longest-horizon tasks) remains the hardest.
- **No HLE-with-better-tools comparison.** HLE is bounded by what the tool stack can answer; better tools (more compute, better search, better citations) would lift it independently of memory.

### 2. The K-size ablation — the most actionable empirical claim

Memento sweeps the number of cases retrieved per planner step:

| K | DeepResearcher F1 / PM trend |
|---|---|
| 1 | Modest gain over no-memory baseline; under-coverage. |
| 2 | Improving, still under-coverage. |
| **4** | **Peak F1 / PM. The recommended default.** |
| 8 | Diminishing returns; some metrics begin to drop. |
| > 8 | Context dilution; planner reasoning degrades. |

The qualitative finding behind the curve is **"small, high-quality memory works best"**. Two factors drive the K=4 sweet spot:

- **Planner attention budget.** Each retrieved case occupies hundreds of tokens. At K > 8 the planner's attention to the original query starts to suffer — the same "lost in the middle" effect from Liu et al. 2023 / Paulsen 2026 that [100](100-contextual-memory-is-a-memo.md) cites.
- **Case relevance distribution.** The marginal relevance of the K-th case decays fast with K. The 5th-best case is much worse than the 4th-best, and adds noise without adding signal.

For practitioners: **start at K=4, ablate K ∈ {1, 2, 4, 8} on a held-out validation split, and pick the curve's empirical peak**. Do not assume more retrieval is better.

### 3. The OOD gain — where CBR's advantage is sharpest

Memento's OOD result on DeepResearcher (+4.7 to +9.6 absolute F1/PM on out-of-distribution tasks) is the most theoretically interesting number. Reflexion-style verbal reflection commonly *degrades* OOD because the reflections were tuned to in-distribution problems. Memento's CBR generalises OOD because:

- **Cases encode strategies, not answers.** A case is `(s_T, a_T, r_T)` where `a_T` is a tool call structure — the *how*, not the *what*. A novel query that is structurally similar to a past query (same plan shape) finds a useful case even if the surface answer differs.
- **The retriever is contrastive-trained.** The dual encoder learned to map *strategically similar* queries together, not just lexically similar. OOD queries that share strategy-level features still retrieve useful cases.
- **Negative cases anchor the retriever.** Hard negatives in training prevent the retriever from collapsing to "everything looks like the most populous bucket" — the standard failure mode for OOD retrieval.

The strategic-vs-surface distinction is the paper's quiet contribution. It is what makes CBR's OOD gain real rather than a benchmark artefact.

### 4. Comparison with neighbouring memory designs

| System | Storage | Retrieval | Learning | Empirical finding |
|---|---|---|---|---|
| Reflexion | Free-text reflections | Prepend all (or trivial filter) | None | Helps in-distribution; degrades OOD. |
| MemGPT | Hierarchical context | Heuristic page-in/page-out | None | Solves capacity, not policy. |
| A-MEM | Hierarchical with summaries | Embedding-similarity | None | Marginal gains; complexity high. |
| Voyager | Code skill library | Embedding-similarity | None | Strong in procedural domains; brittle in declarative. |
| **Memento (parametric)** | **Structured (s, a, r) tuples** | **Learned dual encoder, soft-Q** | **Contrastive on positives + negatives** | **SOTA on GAIA / DeepResearcher; +4.7–9.6 OOD.** |

Three observations from the table:

- Every prior system's "learning" column is empty. Memento is the first deployed system whose retrieval policy is *learned* end-to-end with a contrastive objective.
- Memento's storage schema is the most structured. Free-text → hierarchical-text → tuples is the progression, and the structured tuple is what makes contrastive training feasible.
- Voyager's procedural-skills library is closest in spirit; Memento generalises to declarative cases and adds the parametric retriever.

### 5. Critical reading vs. paper [100]

[100-contextual-memory-is-a-memo](100-contextual-memory-is-a-memo.md) argues that all retrieval-based memory ("C-engineering") is bounded by ᾱ < 1 on compositionally novel inputs, with Ω(k²) coverage required to reach 1 − δ. The paper explicitly addresses the Memento-class of system: *"learned-retrieval (MemRL, Memento) closes the gap. No: optimizing which exemplars to surface still feeds them to the frozen model at inference. Theorem 1 still binds. The gains reflect reduced noise on seen compositions, not generalization to novel ones."*

The honest synthesis: **both papers are right about different regimes**.

- On the GAIA / DeepResearcher / SimpleQA distributions, the compositional novelty rate is bounded enough that Memento's K=4 retrieval is sufficient to cover nearly all task structure. Within this distribution, [100]'s Ω(k²) bound is not binding because the practical k is small.
- On a hypothetical *truly compositionally novel* benchmark (think: SCAN-style systematic generalisation, or cross-domain scientific reasoning), Memento's gains would compress toward the frozen-LLM baseline. The +4.7–9.6 OOD gain is impressive but the OOD distribution still shares strategic features with the training distribution; it is "near-OOD" not "compositionally novel."
- The deeper point [100] makes is the **frozen-novice problem** — even on tasks the model could solve in principle, agents that operate exclusively via C-engineering cannot reorganise their representations. Memento's case bank grows; the cognition does not. After a year of running, Memento is the same agent with a bigger filing cabinet.

The harness implication: **Memento is the strongest C-engineering option, but for production deployments that must improve over a long horizon, a θ-engineering consolidation pathway is still on the table**. See [100]'s two-pathway architecture — fast path = retrieval (Memento-style), slow path = consolidation into weights (offline SFT / knowledge editing / circuit-aware fine-tuning).

### 6. Integration playbook — bolting CBR onto an existing harness

A four-step retrofit that does not require rewriting your agent loop.

#### Step 1 — Define the case schema

```json
{
  "id": "case_2026_05_06_001",
  "query": "<the user's task>",
  "plan": "<planner output, optional>",
  "final_action": "<terminal tool call or answer>",
  "reward": 1.0,
  "is_negative": false,
  "meta": {
    "ts": "2026-05-06T00:00:00Z",
    "planner_model": "gpt-4o",
    "executor_model": "o3",
    "tool_versions": { "search": "v3", "code": "v2" },
    "trajectory_hash": "sha256:..."
  }
}
```

The schema is JSONL, append-only, on disk. Provenance is non-negotiable — without `meta`, the bank is unauditable.

#### Step 2 — Add the `MemoryWriter` post-episode hook

```python
# pseudo-code
def on_episode_end(trajectory, ground_truth):
    reward = grade(trajectory.final_answer, ground_truth)
    if reward >= TAU_POSITIVE:
        bank.append(make_case(trajectory, reward, is_negative=False))
    elif reward <= TAU_NEGATIVE and random() < NEGATIVE_KEEP_RATE:
        bank.append(make_case(trajectory, reward, is_negative=True))
```

This is the M-MDP's *policy improvement step*. Two tunable thresholds (`TAU_POSITIVE` for what counts as success, `TAU_NEGATIVE` for what's worth keeping as a hard negative) plus a sampling rate for negative retention. Implement as a deterministic harness hook ([05-hooks](05-hooks.md)), not as an LLM action.

#### Step 3 — Deploy a retriever sidecar

```text
agent ──HTTP──▶ retriever-service
                   │
                   ├── encode(query) → q_vec
                   ├── score(q_vec, cases) → top_K by Q̂
                   └── return top-K cases as JSON
```

The sidecar runs the dual encoder and serves a `/retrieve` endpoint. Two implementations:

- **Cold start (non-parametric):** wrap an off-the-shelf embedding model. No training. Sufficient until the bank has a few thousand cases.
- **Production (parametric):** dual encoder trained offline on `(positive_case, query)` and `(hard_negative_case, query)` pairs. Periodic retraining.

The sidecar is a separately deployable service with its own lifecycle. This is critical: do not run the retriever in-process with the agent — you want to swap retriever versions independently.

#### Step 4 — Inject top-K=4 cases into the planner prompt only

```text
planner_prompt = template.format(
    query=query,
    cases=retrieved_cases[:4],     # K=4 hard cap at the prompt boundary
    plan_template=plan_skeleton
)
```

Cases enter the *planner* prompt, not the executor prompt. The executor stays case-blind to keep its tool-call distribution clean. K=4 is the prompt-side hard cap even if the retriever returned 8.

#### Step 5 — Periodic retraining cadence

```text
every N episodes (e.g. N=1000):
  1. snapshot bank
  2. build training pairs (positives, in-batch negatives, hard negatives)
  3. retrain retriever from scratch on snapshot
  4. validate on held-out queries
  5. atomic-swap new retriever; keep old as fallback
  6. monitor regression on held-out retrieval quality + downstream agent reward
```

Treat the retriever like any other production ML artefact: versioned, A/B-able, rollbackable.

### 7. What about θ-engineering as a follow-on?

The [100] critique points at a complement, not a replacement. Once you have a Memento-style bank running, the natural next step is offline distillation of the bank into LLM weights — not full SFT, but circuit-aware fine-tuning (Ye et al. 2025) targeting the fact-memory units, with trace provenance and regression guards. This is not in Memento's scope; it is the *next* harness-engineering frontier. See [100]'s two-pathway architecture and [78-ngc-neural-garbage-collection](78-ngc-neural-garbage-collection.md) for adjacent ideas.

## Empirical anchors (consolidated)

| Result | Value | Conditions | Source |
|---|---|---|---|
| GAIA Validation Pass@3 Top-1 | 87.88% | Parametric CBR, K=4, GPT-4 + o3 | Repo README, paper abstract |
| GAIA Test | 79.40% | Same | Repo README, paper abstract |
| DeepResearcher F1 | 66.6% | Same | Paper abstract |
| DeepResearcher PM | 80.4% | Same | Paper abstract |
| DeepResearcher OOD gain | +4.7 to +9.6 absolute | CBR vs no-memory baseline | Paper abstract |
| SimpleQA accuracy | 95.0% | Same default config | Repo README |
| HLE Performance | 24.4% | Bounded by tool capability | Repo README |
| Optimal K | 4 | Across DeepResearcher metrics | Paper / repo ablation |
| OOD lift mechanism | Strategy-level retrieval | Contrastive retriever + hard negatives | Paper analysis |
| Negative result | GAIA Level-3 long-horizon | Compounding errors, (s,a,r) compression too lossy | Repo limitations |

## Variants and counter-arguments addressed

- **"Numbers are tool-stack-bound."** True for HLE explicitly; partially true for GAIA. A weaker tool stack would shift absolute scores; relative gains from CBR are more robust.
- **"Why does Memento beat training-based methods?"** Likely a combination of (i) the planner-executor split being a strong inductive bias, (ii) cases capturing strategies that GRPO over-fits, and (iii) the frozen LLM being a strong prior that fine-tuning can degrade. Not a contradiction with the value of fine-tuning in general.
- **"Is the OOD gain real?"** Replicated across two distinct OOD splits in DeepResearcher; the contrastive retriever's strategy-level matching is the plausible mechanism. Independent reproduction is needed before the claim is uncontroversial.
- **"Can a smaller model close the gap with cases?"** Open question; the planner-on-GPT-4 dependency suggests not — the planner needs reasoning headroom to use cases well. Smaller-model studies are future work.
- **"Why not consolidate the bank into the LLM?"** Out of paper scope. [100]'s consolidation pathway is the clean complement; Memento intentionally restricts itself to the fast-retrieval side.

## Failure modes and limitations

1. **The frozen-novice ceiling.** [100]'s argument applies. Memento improves at the retrieval layer; cognition stays put. Long-horizon deployments will eventually need a consolidation channel.
2. **Bank quality dominates bank size.** Bad cases poison the retriever. Operational hygiene (reward thresholding, deduplication, periodic curation) is non-optional.
3. **Compositional novelty unaddressed.** [100]'s Ω(k²) bound on compositionally novel inputs holds. Benchmarks where novelty is genuine (SCAN-style, cross-domain reasoning) will show the gap.
4. **Multi-step strategies under-represented.** Single (s_T, a_T, r_T) tuples discard trajectory shape. Trajectory-aware case schemas are an open area.
5. **Retriever drift.** Each retraining shifts the retrieval distribution; without held-out monitoring, downstream regressions can hide.
6. **Reproducibility caveats.** Tool-stack-dependent numbers (especially HLE, GAIA Level-3) do not transfer across deployments. Always re-benchmark on your own stack.
7. **Persistent-attack surface.** [100]'s persistent-compromise math applies: a poisoned case in the bank persists across sessions. Prompt-injection defenses ([22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [82-poisonedrag](82-poisonedrag.md)) extend to case-bank curation.
8. **Negative curation is still manual.** Automated identification of "useful negatives" vs "noise negatives" is unsolved; current pipelines rely on simple reward thresholds.

## When to use, when not

**Bolt CBR onto your harness when** (i) you have a deep-research-adjacent task distribution with verifiable success signal, (ii) you face out-of-distribution generalisation requirements that pure retrieval struggles with, (iii) your LLM is closed-source or fine-tuning is operationally hard, (iv) you can afford a retriever-training pipeline and a sidecar service, and (v) your task budget per query is high enough that K=4 case-conditioned planning is affordable.

**Skip CBR when** (i) tasks are i.i.d. and well-covered by pretraining (the retriever has nothing to add), (ii) the success signal is unreliable or absent (the retriever will learn noise), (iii) the agent runs single-session with no compounding benefit, or (iv) latency budget cannot tolerate a retrieval round-trip per planner call.

**Skip Memento *as a whole framework* but adopt parts when** the planner-executor split, the MCP tool design, or the JSONL bank schema are valuable on their own. None of those parts require the full CBR loop.

## Implications for harness engineering

- **Pair Memento's fast path with a [100]-style slow path.** The fast path (CBR) is the now; the slow path (offline consolidation into weights) is the next year. Building a harness that *can* support both, even if the slow path isn't shipped yet, future-proofs the design. See [100], [78-ngc-neural-garbage-collection](78-ngc-neural-garbage-collection.md).
- **Adopt K=4 as the default top-K everywhere.** This generalises across CBR, RAG, and skill-libraries: small high-quality retrieval beats large naive retrieval. Audit existing harnesses for hard-coded K=10 or K=20 defaults; that is a likely loss source.
- **Make the retriever a first-class artefact.** Versioned, deployable, observable, rollbackable. Most harnesses treat retrieval as utility code. Promote it. ([24-observability-tracing](24-observability-tracing.md), [09-memory-files](09-memory-files.md).)
- **Reward thresholding plus negative retention is non-optional.** Shipping a CBR loop with positives only is the most common mistake. The harness must own negative collection deterministically.
- **Cases in planner only.** Never let cases pollute executor prompts. Empirically and theoretically the planner-side injection is where they belong.
- **Provenance unlocks audit.** Every case carries ts + model versions + tool versions + trajectory hash. This is the cheapest insurance against poisoned-case incidents and silent retriever regressions.
- **CGT (Compositional Generalisation over Time) is the missing benchmark.** Per [100], existing benchmarks measure recall@k, not learning. A harness that integrates CBR should also integrate a CGT eval — measure whether downstream accuracy strictly increases over T sessions on isolated concepts, then evaluate on novel combinations. A well-tuned Memento should rise on adjacent concepts; a hypothetical θ-engineering hybrid should rise on truly novel combinations.
- **The MCP server suite is reusable independent of the agent.** Even if you don't adopt CBR, lifting `documents_tool.py`, `ai_crawler.py`, and the four-interpreter ladder into your harness is high-leverage. ([108-memento-codebase-mcp](108-memento-codebase-mcp.md), [07-model-context-protocol](07-model-context-protocol.md).)
- **Planner-executor split is the right factoring.** GPT-4 (planning) + o3 (executing) maps to "reasoning model upstream, tool-use model downstream." Single-model harnesses are leaving performance and cost on the table. ([16-plan-and-solve](16-plan-and-solve.md), [02-subagent-delegation](02-subagent-delegation.md), [42-langchain-deep-agents](42-langchain-deep-agents.md).)
- **Security extension required.** Persistent-compromise via poisoned cases is real. Harness must implement: (i) untrusted retrievals into quarantine context only, (ii) consolidation-eligible cases require human review for high-impact updates, (iii) injection-detection on case writes. ([22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [82-poisonedrag](82-poisonedrag.md), [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md).)
- **Integration is incremental.** The four-step retrofit (schema → hook → sidecar → prompt injection) can ship behind a feature flag. A/B against the baseline. Roll forward when the OOD gain is replicated on your own data.

The one-sentence takeaway: **Memento is the strongest C-engineering memory we have, K=4 is the right default, the OOD gain is real, the [100] critique still binds on truly novel composition — so adopt CBR now, plan the consolidation pathway for next year.**

## See also

- [106-memento-paper-theory](106-memento-paper-theory.md) — the M-MDP and soft-Q derivation behind these results.
- [107-memento-cbr-memory](107-memento-cbr-memory.md) — the CBR mechanism, retriever architecture, and training pipeline.
- [108-memento-codebase-mcp](108-memento-codebase-mcp.md) — the working reference implementation.
- [100-contextual-memory-is-a-memo](100-contextual-memory-is-a-memo.md) — the theoretical critique of all retrieval-only memory; reading this together with Memento is the whole point.
- [14-reflexion](14-reflexion.md), [81-reasoningbank](81-reasoningbank.md), [19-voyager-skill-libraries](19-voyager-skill-libraries.md), [25-agentic-rag](25-agentic-rag.md) — the neighbours Memento generalises or competes with.
- [78-ngc-neural-garbage-collection](78-ngc-neural-garbage-collection.md) — adjacent work on learnt cache management.
- [82-poisonedrag](82-poisonedrag.md), [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md) — security extensions required when shipping a persistent case bank.
- [05-hooks](05-hooks.md), [09-memory-files](09-memory-files.md), [24-observability-tracing](24-observability-tracing.md) — harness primitives this integration depends on.
