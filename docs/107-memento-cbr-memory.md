# 107 — Memento's Case-Based Reasoning Memory: Non-Parametric and Parametric Variants, Dual-Encoder Retrievers, and the K=4 Sweet Spot

**Source.** Zhou, Chen, Guo, Yan, Lee, Wang, Lee, Zhang, Shao, Yang, Wang — *Memento: Fine-tuning LLM Agents without Fine-tuning LLMs* — arXiv:2508.16153v2 (Aug 2025) — repo https://github.com/Agent-on-the-Fly/Memento (mirror https://github.com/Memento-Teams/Memento).

**One-line definition.** Memento's memory is a flat bank of **(state_T, action_T, reward_T)** *final-step* tuples drawn from past episodes; retrieval over the bank is the agent's *learnable policy*, and Memento ships two versions of that policy — a **non-parametric** dense-similarity retriever (zero training, plug-and-play) and a **parametric** dual-encoder retriever trained with contrastive loss on accumulated positives and hard negatives — with the K=4 ablation showing that *small, high-quality* memory beats large case banks.

## Why this paper matters for memory design

Pre-Memento, the typical "agentic memory" stack was: dense embeddings → vector DB → top-K retrieval → prepend to context. Each step was an engineering artefact, not a learned component, and the field had no theory for what made one stack better than another. Memento's CBR design re-frames every one of those steps:

- **What is stored** is no longer "summarised text" or "raw transcripts" — it is a structured tuple anchored to the final reward, which gives every case an explicit success label.
- **How retrieval scores cases** is no longer "cosine similarity on whatever embeddings you have lying around" — it is the soft-Q value Q̂(query, case) of the M-MDP (see [106](106-memento-paper-theory.md)), and the retriever is *trained* to approximate it.
- **How the bank evolves** is no longer ad-hoc append-everything — it is a deliberate write rule with reward thresholding and hard-negative retention, which gives contrastive learning a usable signal.

The K-ablation finding (peak at K=4) is the most actionable empirical claim: it disagrees with the RAG community's "more retrieved chunks → better grounding" instinct and aligns with the [100](100-contextual-memory-is-a-memo.md) "lost in the middle" / Paulsen-2026 "effective utilisation saturates at ~20K tokens" results. For harness engineers building Memento-style memory, K=4 is the default to start from.

## Problem it solves

CBR (Riesbeck & Schank 1989; Aamodt & Plaza 1994) is a 30-year-old AI tradition: solve new problems by adapting solutions to similar past problems. It has four classical phases — Retrieve, Reuse, Revise, Retain — and was largely abandoned in the deep-learning era because (a) similarity metrics were brittle, (b) "reuse" required hand-coded adaptation rules, and (c) there was no principled way to learn the metric end-to-end.

Memento brings CBR back by solving each of those:

1. **Brittle similarity.** Replaced by a learned dual-encoder. The retriever sees thousands of (positive case, query) and (negative case, query) pairs and learns the metric.
2. **Hand-coded reuse.** Replaced by the LLM. The planner consumes retrieved cases as in-context examples and adapts them to the new query implicitly. No symbolic adaptation rule.
3. **No end-to-end learning.** Replaced by the soft-Q framing — the retriever is the policy, the LLM is the simulator, and the soft-Q optimal policy gives the contrastive training objective.
4. **Bank quality drift.** Replaced by the reward-thresholded write rule plus periodic retriever retraining, which together act as the M-MDP's policy improvement step.

The *additional* problem Memento solves is: **how do you make a retriever that generalises out-of-distribution?** Pure embedding similarity collapses on OOD because the embedding space is trained on a different distribution than the bank. Memento's contrastive retriever is trained on the bank's own distribution, so OOD performance is bounded by case quality, not embedding-model coverage. This is where the +4.7–9.6 absolute OOD gain on DeepResearcher comes from.

## Core idea in one paragraph

Build the memory bank from final-step trajectory tuples (s_T, a_T, r_T) with positives (high reward) and negatives (low reward) both retained. At inference, encode the current query and every case with a dual encoder, compute Q̂ scores by inner product, and softmax over the top-K (K=4 by default). Pass those K cases as in-context examples to a frozen planner LLM, which adapts them to the new query. After each episode, append the new (s_T, a_T, r_T) tuple to the bank, and periodically re-train the retriever on the updated positive/negative pool with InfoNCE-style contrastive loss. The CBR loop runs forever, the LLM is never touched, and the agent improves measurably with each new case.

## Mechanism (step by step)

### 1. Case representation — what goes into the bank

Each case c is a tuple:

```text
c = (s_T, a_T, r_T, meta)
```

where:
- **s_T** is the *final-step state*: the original query, the trajectory summary, and the planner's last reasoning step;
- **a_T** is the *final action*: the tool call (or terminal answer) that produced the outcome;
- **r_T** is the *terminal reward*: success label, often binary, sometimes graded;
- **meta** carries provenance (timestamp, model, tool versions, optional plan trace).

This compresses a multi-step trajectory to its final-step summary. The information loss is real but deliberate: it makes contrastive training tractable (cases are short), it focuses the retriever on the strategic outcome (what worked), and it dovetails with the M-MDP's terminal-reward MDP.

The repository persists the bank as `memory/memory.jsonl` — one tuple per line, JSON-encoded. This is intentionally simple: it is human-readable, append-only, and trivially backed up.

### 2. Bank construction — the write rule

After each episode, the harness runs a `MemoryWriter` that:

1. **Computes r_T** from environmental feedback (e.g. answer-grading function for QA tasks, unit tests for code tasks, or scalar reward for graded benchmarks).
2. **Decides whether to append.** Two policies:
   - **Reward-thresholded:** append only if r_T ≥ τ (where τ is a per-task threshold, typically the success cut-off). Keeps the bank biased toward demonstrations the agent should imitate.
   - **Negative-bucket retention:** also append a sample of low-reward cases tagged `is_negative=True`. These are not surfaced to the planner, but they are critical for contrastive retriever training (without negatives, you cannot teach the retriever what to avoid).
3. **Deduplicates** approximately. Two cases with near-identical s_T are pruned to one (the highest-reward variant) to keep the bank healthy.
4. **Appends** the survivor to `memory.jsonl` with provenance.

This write rule is the M-MDP's *policy improvement step*: each new case shifts the implicit Q-distribution that the retriever later approximates.

### 3. Non-parametric retrieval — the zero-training baseline

The simplest version of the retriever is *non-parametric*. No learnable parameters. It uses an off-the-shelf encoder (e.g. a strong pretrained sentence-transformer or an OpenAI embedding model) to encode q_t and every c ∈ M, and scores by cosine similarity:

Q̂_np(q, c) = sim(enc(q), enc(c.s_T))

Top-K retrieval picks the K cases with highest Q̂_np. This is essentially classical dense retrieval — no training, no learnable parameters, no contrastive objective. It is what `client/no_parametric_cbr.py` implements.

**Why it works at all:** strong pretrained encoders already place semantically related queries near each other; for a fresh agent with a small bank, this is enough. The downside is that the embedding space was trained on a different distribution than the bank, so the *most similar* case may not be the *most useful* case. This is exactly where the parametric retriever wins.

### 4. Parametric retrieval — the learned dual-encoder

The full Memento retriever is a *dual encoder*:

```text
enc_q : query → R^d        (query tower)
enc_c : case → R^d         (case tower)
Q̂_p(q, c) = enc_q(q)^T · enc_c(c.s_T) / τ
```

τ is the soft-Q temperature (the same α from [106](106-memento-paper-theory.md), realised as a softmax temperature here). The two towers can share parameters (Siamese) or be separate; the codebase's default is a single shared backbone with two projection heads on top, which is standard for dual-encoder retrieval (cf. DPR, SBERT).

**Loss function — contrastive (InfoNCE-style).** For a query q with positive case c⁺ and a set of negatives {c⁻_1, ..., c⁻_n}:

L = − log ( exp(Q̂(q, c⁺)) / ( exp(Q̂(q, c⁺)) + Σ_i exp(Q̂(q, c⁻_i)) ) )

The contract of `train_memory_retriever.py` exposes the relevant flags:

- `--use_plan` — also condition encoding on the planner's plan, not just the query, so the retriever learns to surface cases relevant to the *planning intent*, not the literal query string.
- `--val_ratio 0.1` — reserved 10% for validation.
- `--batch_size 32` — modest batch.
- `--lr 2e-5`, `--epochs 10`, `--save_best` — standard.

The repo defaults `MEMORY_TOP_K=8` for inference but the paper's K=4 finding applies — the codebase exposes both retrieval-side K (how many to encode) and prompt-side K (how many to actually surface to the planner).

**Positive sampling.** From `memory.jsonl`, take cases with r_T ≥ τ. Pair each with queries from training data that the case is known to help on.

**Negative sampling.** Two classes:
- **In-batch negatives:** other queries' positives in the same minibatch. Cheap, high recall.
- **Hard negatives:** cases that look superficially similar to the query (high Q̂_np) but were tagged failed in the bank. These force the retriever to learn the *strategic* distinction, not the surface distinction. The repo's `MEMORY_MAX_NEG_EXAMPLES=8` controls the cap.

The combined contrastive loss over both classes is what produces a retriever that generalises OOD: in-batch negatives anchor the metric, hard negatives sharpen it.

### 5. Inference — the read path

Per query at inference time:

```text
1. q_t = (current_task, partial_trajectory_so_far)
2. cands = top_K_by_Q̂(q_t, M, K=4)         # the K=4 sweet spot
3. plan  = planner_LLM(q_t, cands)         # cases conditioned-on
4. while plan not exhausted:
     step  = next_step(plan)
     act   = executor_LLM(step, registry)
     obs   = mcp_tool_call(act)
     plan  = update_plan(plan, obs)
6. r_T    = reward(q_t, final_answer)
7. memory_writer.append((s_T, a_T, r_T))
```

Two engineering details matter:

- **Cases enter the *planner* prompt, not the executor prompt.** The executor is a tight tool-call loop and benefits from structured plan steps, not free-text examples. Mixing cases into the executor pollutes the tool-selection distribution.
- **K=4 is enforced at the prompt boundary.** Even if the retriever returns 8 candidates, only the top-4 enter the planner prompt. The other 4 are kept for re-ranking experiments and for retriever ablation.

### 6. Periodic retraining — the offline RL step

The retriever is *not* updated online (per-episode). Online updates are unstable for contrastive retrievers because new cases shift the embedding manifold faster than the optimizer can converge. Instead, Memento batches:

1. Run N episodes with the current retriever, accumulating new (s_T, a_T, r_T) tuples.
2. After every N (e.g. 1000) episodes, re-train the retriever from scratch on the updated bank, validating on held-out queries.
3. Swap in the new retriever atomically; keep the old one as a fallback for rollback.

This is the M-MDP's *offline policy iteration*: each retraining pass is a complete sweep of policy evaluation (loss minimisation) plus policy improvement (new retriever weights).

### 7. The K-ablation (the headline finding)

The paper's K-sweep on DeepResearcher reports peak F1/PM at **K=4**. Below K=4, coverage is too sparse — relevant cases are not retrieved often enough. Above K=4, the prompt fills with cases that are individually relevant but collectively confusing, and the planner's downstream tool-selection accuracy degrades. The exact cliff depends on prompt budget; on smaller-context planners, K=2 may be the new sweet spot, while on 200K-context planners K=8 may be tolerable. The robust harness rule is: **start at K=4, ablate down to 2 and up to 8, pick by held-out validation**.

The "small, high-quality memory works best" insight generalises this: a 1000-case bank with strict τ outperforms a 100k-case bank with lax τ on the same retriever architecture. Bank quality dominates bank size.

## Empirical anchors

- **K=4 ablation peak.** Reported on DeepResearcher F1/PM. Both higher (K=8) and lower (K=1, K=2) underperform.
- **OOD gain +4.7 to +9.6 absolute** on DeepResearcher when adding parametric CBR vs. no-memory baseline. This is the regime where Reflexion-style memory typically degrades.
- **GAIA validation 87.88% Pass@3 top-1**, **GAIA test 79.40%** — both achieved with the parametric retriever and K=4.
- **SimpleQA 95.0%** with non-parametric CBR (the simpler retriever is sufficient for SimpleQA's distribution).
- **HLE 24.4%** — even with parametric retrieval, HLE is bounded by tool capability, not retriever quality.
- **"Small, high-quality memory works best"** — qualitative finding from ablation, supported by repo authors' commentary.

## Variants and counter-arguments addressed

- **"Just use a vector DB."** Vector DBs implement the non-parametric variant. They work at small scale but fail OOD because the embedding space is fixed. The parametric retriever is the upgrade path.
- **"Just use BM25 / hybrid retrieval."** BM25 is competitive on lexical overlap tasks but loses on semantic transfer. The dual encoder strictly subsumes BM25 in the deep-research domain.
- **"Add a re-ranker."** Cross-encoder re-rankers can be stacked on top of the bi-encoder. The repo treats this as future work; the current K=4 finding is robust enough to be a reasonable stopping point.
- **"Why not store full trajectories?"** Full trajectories are noisy and expensive to encode. The paper argues final-step tuples preserve the strategic signal; partial-trajectory variants are an open ablation.
- **"Won't the bank grow unboundedly?"** Reward thresholding plus deduplication keeps growth roughly logarithmic in episode count, since most new cases either fail or duplicate prior wins.
- **"Why not Reflexion-style verbal reflection in addition?"** You can — the case `meta` field can carry a free-text reflection — but the paper found that the structured (s, a, r) signal alone is enough to drive the contrastive loss. Free-text reflection is value-add, not core.

## Failure modes and limitations

1. **Cold start.** With an empty bank, the parametric retriever has nothing to learn from. The fallback is non-parametric retrieval (or no retrieval at all) until enough cases accumulate. The repo handles this implicitly by failing over to similarity scoring when the retriever checkpoint is missing.
2. **Negative pollution.** Including the wrong negatives (e.g. cases that failed for environmental reasons rather than strategic ones) teaches the retriever the wrong contrast. Curation of the negative pool is the silent quality-determining step.
3. **Distribution drift between bank and queries.** If the bank is dominated by tasks from one distribution (say SimpleQA), and queries shift to another (say GAIA Level 3), the retriever's earlier specialisation can mislead. Periodic retraining helps; explicit distribution-aware sampling helps more.
4. **Final-step compression.** Strategy that requires multi-step reasoning patterns (e.g. "first try X, if it fails try Y") is hard to express in a single (s_T, a_T, r_T) tuple. Trajectory-aware case representations are an open area.
5. **Retriever capacity vs bank size.** A 110M-param retriever can over-compress a 1M-case bank. The codebase does not auto-scale retriever capacity to bank size; this is a manual ops concern.
6. **Reproducibility of the K=4 sweet spot.** The K-sweep depends on planner context budget, prompt template, and case verbosity. Re-running on a different planner may shift the optimum. Treat K=4 as a strong prior, not a constant.
7. **No formal regret bounds.** The soft-Q derivation gives optimality conditions but not finite-sample regret. For deployment, this means you cannot a priori bound how badly the retriever will perform after N episodes; empirical curves are the only signal.

## When to use, when not

**Reach for parametric CBR when** you have ≥ a few thousand labelled-reward cases, a stable task distribution, an OOD generalisation goal, and willingness to maintain a retraining pipeline. The deep-research and tool-use settings are canonical fits.

**Stay with non-parametric CBR when** you have < a thousand cases, the task distribution is broad/varied, or you cannot afford retriever retraining ops. Non-parametric is also the right choice for the first weeks of any new deployment — you cannot train a retriever before you have data.

**Skip CBR entirely when** the task is i.i.d. and well-covered by pretraining (the retriever has nothing to add), or the success signal is unreliable (the retriever will learn noise), or the agent runs once-off (no compounding benefit from a bank).

## Implications for harness engineering

- **Adopt (s, a, r) as a memory schema, not free-text.** Most harness memory today is unstructured ([09-memory-files](09-memory-files.md), [72-claude-mem-persistent-memory-compression](72-claude-mem-persistent-memory-compression.md)). Memento's structured tuple is a strict upgrade for tasks with reward signal: it gives every entry a label, makes contrastive training possible, and naturally supports negative-bucket retention. Free-text reflection can ride along in `meta`.
- **Add a `MemoryWriter` hook with reward thresholding.** A post-episode hook ([05-hooks](05-hooks.md)) that scores the trajectory and conditionally appends a tuple is the cleanest implementation. Reward-thresholding plus deduplication should be deterministic in the harness, not the LLM's responsibility.
- **K=4 is your default; ablate to confirm.** When integrating CBR memory into a harness, default to top-4 retrieval into the planner prompt. Run a K ∈ {1, 2, 4, 8, 16} sweep on a held-out validation split before any production deployment.
- **Train the retriever offline; never online.** The harness should treat the retriever as a versioned model artefact, with checkpoints, A/B comparison against the previous version, and rollback. This is the same operational maturity required for any production ML model — most current agentic memory stacks lack it.
- **Negatives are first-class.** The single biggest mistake is shipping CBR with positives only. Without hard negatives, the retriever is a similarity engine, not a soft-Q approximation. The harness must have a deliberate negative-collection pathway — typically failed episodes plus a small fraction of confounder examples.
- **Cases in planner, tools in executor.** Do not let cases leak into the executor's tool-selection prompt — the executor is a tight loop and case noise hurts more than it helps. This mirrors the planner/executor split in [42-langchain-deep-agents](42-langchain-deep-agents.md) and [16-plan-and-solve](16-plan-and-solve.md).
- **Compositional gap remains.** Even a perfect retriever cannot fix what the frozen LLM can't compose. [100](100-contextual-memory-is-a-memo.md)'s Ω(k²) coverage demand still applies. CBR is the best C-engineering option but not the end of the road; pair with a θ-engineering consolidation pathway when targets demand it.
- **Provenance unlocks audit.** The `meta` field should carry timestamp, model version, tool versions, and a hash of the originating trajectory. This is the cheapest way to make the bank auditable when something goes wrong (poisoned case, regression, etc.).
- **The retriever is a tiny model — own it.** A 110M-param retriever is small enough to host locally even when the LLM is closed-source. This is the only learnable component in the system; treat it as the agent's brain and own its lifecycle (versioning, eval, rollback).

The one-sentence takeaway: **structure your memory as labelled (s, a, r) tuples, train a small dual-encoder retriever on positives and hard negatives, retrieve K=4 into the planner prompt, and you have an agent that learns without touching the LLM.**

## See also

- [106-memento-paper-theory](106-memento-paper-theory.md) — the M-MDP and soft-Q derivation that justifies this retrieval scheme.
- [108-memento-codebase-mcp](108-memento-codebase-mcp.md) — the actual codebase implementation, file by file.
- [109-memento-results-and-harness](109-memento-results-and-harness.md) — benchmarks and harness integration patterns.
- [25-agentic-rag](25-agentic-rag.md) — agentic RAG with self-critique, the closest non-CBR cousin.
- [79-skill-rag](79-skill-rag.md) — the skill-RAG pattern that is essentially non-parametric CBR over skills.
- [81-reasoningbank](81-reasoningbank.md) — ReasoningBank's contrast-extraction is the strongest no-retrieval-training C-engineering signal.
- [100](100-contextual-memory-is-a-memo.md) — the theoretical limits of any retrieval-only memory, including Memento's.
