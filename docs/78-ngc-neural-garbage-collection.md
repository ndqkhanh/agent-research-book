# 78 — Neural Garbage Collection: Learning to Forget while Learning to Reason

**Paper.** Michael Y. Li, Jubayer Ibn Hamid, Emily B. Fox, Noah D. Goodman — *Neural Garbage Collection: Learning to Forget while Learning to Reason* — Stanford University — arXiv:2604.18002 — April 2026.

**One-line definition.** Neural Garbage Collection (NGC) trains a transformer LM, end-to-end from binary task reward only, to interleave normal chain-of-thought token generation with *learned* KV-cache evictions—treating each eviction as a discrete policy action alongside tokens so policy-gradient RL jointly optimizes reasoning and what the model is allowed to remember.

## Why this paper matters

CoT steps linearly grow the KV cache—deployment cost and a ceiling on “think longer” even when compute is available. Heuristic eviction and compress-then-matching (SnapKV, KeyDiff, StreamingLLM, Breadcrumbs, Memento, etc.) encode *designer* priors. NGC asks: the same RLVR that trains on discrete *tokens* can treat discrete *cache keeps* as actions and optimize both from *one* binary task reward (AlphaZero-style tabula rasa). No SFT for eviction, no teacher, no importance labels.

In practice: **Countdown** at 50% evict/round (2.4× peak KV) reaches **49.6%** pass@1 vs **21.2%** (best baseline KeyDiff) while others fall to 7.8% and below. **DAPO-17k** math (AMC, AIME) at 3–5× reduction: NGC leads all four baselines on pass@*k*; heuristics are inconsistent and sometimes near zero. **Budget-aware interoception** (prompt the eviction rate ρ) adds **8–13%** pass@1 when test budgets are tighter than training. The contribution is a proof that *efficiency is learnable* alongside reasoning in the same GRPO stack.

## Problem it solves

1. **Heuristic eviction is task-fragile and misaligned with objectives.** Importance scores from attention mass, recency, or key norms can work on one distribution and fail on another; the paper notes SnapKV is catastrophic on Countdown but decent on some math settings, while KeyDiff and SnapKV swap second place across AMC 2023 vs 2025. A fixed proxy cannot track what a particular reasoning policy actually needs to retain.
2. **Off-policy mismatch when you train without respecting evictions in the log-prob path.** If you drop KV entries during training but re-score tokens with a full causal mask, you optimize the wrong policy: tokens were sampled under a *pruned* mask. Naive “KV dropout + token RL” (their “Targeted KV dropout” ablation) initially learns, then collapses with exploding gradient norms around step 100 when the eviction curriculum sharpens.
3. **Split training of “indexer” and LM breaks credit assignment for memory.** The paper contrasts DeepSeek-style sparse indexers that are KL-warmed to dense attention and then graph-detached, versus NGC’s unified θ for queries, keys, and evictions, so the gradient that improves answers also migrates the representations that produce eviction scores.
4. **Proxy-based cache compression and teacher distillation add pipelines.** Breadcrumbs, Memento, and similar methods bake in what to preserve via distillation, rubrics, or reconstruction. NGC aims to avoid parallel annotation, separate stages, and designer objectives beyond verifiable task reward.

## Core idea in one paragraph

At fixed cadence every δ new tokens, each transformer layer runs an *eviction round*: block-wise importance scores (from recent-query attention into prefix keys) define logits; a size-K *keep* subset is sampled with Gumbel top-k, exposing an exact log-probability; and generation continues attending only to surviving KV blocks. Token sampling and eviction sampling are both discrete actions from π_θ, so group-normalized advantages from binary correctness (Dr. GRPO) backprop as **L = L_token + L_mem** with the *same* advantage for both. A replay pass with per-layer *replay attention masks* recomputes token log-probabilities and eviction log-probabilities under the *actual* visibilities from rollouts, fixing off-policy error and routing ∇_θ L_mem through the attention tensors that produced eviction logits.

## Mechanism (step by step)

### State and grow-then-evict dynamics

Standard RL on chains assumes monotonic context growth. NGC’s environment state is the KV cache; every δ tokens the cache is trimmed. If each round keeps a (1−ε) fraction of entries per layer and evicts ε, the pre-eviction per-layer cache size converges to **c\* = δ/ε** and total pre-eviction footprint to **C\* = L·δ/ε** (Proposition 1, Appendix A.2). The budget is therefore enforced by construction of the process, not by a sparsity penalty in the loss.

They also motivate a generic max **E[ R(τ) ]** under resource cost (paper Eq. 1); the grow-then-evict process implements a hard per-round retention budget without an extra ℓ1 term.

### Scoring and block coarsening

For layer ℓ, use queries from the w most recent tokens (w = 5 in experiments), attention into prefix keys, average across heads and queries to a per-key score ψ_t, then average within contiguous blocks of size b = 32 to block logits s_j. Coarsening reduces combinatorial credit-assignment noise and matches paged-KV systems.

### Gumbel top-k for keep-sets

Instead of deterministic top-k at train time, they sample a keep-set σ with probability

\[
\log p(\sigma \mid s) = \sum_{j=1}^{K} \left( s_{\sigma_j} - \log \sum_{t \notin \{\sigma_1,\ldots,\sigma_{j-1}\}} \exp(s_t) \right)
\]

(prefix-sum implementation avoids K full logsumexps; eval: greedy top-k).

### Joint policy gradient and replay masks

For G rollouts per prompt, binary rewards r_i, group mean baseline, advantage **Â_i = r_i − (1/G) Σ_j r_j** (no extra reward model).

- **L_token (Eq. 3):** −E[ Σ_t log π(o_{i,t}|C_{i,t}) Â_i ], C = pruned cache at t.
- **L_mem (Eq. 4–5):** per layer, mean over eviction rounds of −log π_gumbel(σ|H) · Â_i; **L = L_token + L_mem** (one scalar, not a separate aux loss).

**Replay masks (Fig. 2):** per-layer Boolean visibility = what the decoder *saw* when sampling; one forward with those masks matches rollout log π(token). Recompute eviction-step Q/K on-graph for L_mem; cached rollout tensors would detach ∇_θ.

### Eviction-rate curriculum (Equation 6)

Retention schedule ρ_0 > ρ_1 > … > ρ_K in stages of ∆ steps, with a linear blend in the last 60% (α = 0.6) of each stage between ρ_ℓ and ρ_{ℓ+1} to avoid destabilizing large early evictions. Countdown and DAPO runs share the staircase design but different hyperparameters (Appendix Tables 1–2).

### Budget-aware interoception

For each group of G rollouts, sample one eviction rate ρ and append a structured tag to every prompt: `<eviction_rate>ρ%</eviction_rate>`. At test, sweep ρ; the model generalizes to peak cache sizes stricter than training min, with 8–13% pass@1 lift versus NGC without the tag on aggressive budgets (Figure 6).

### Joint RL training loop (Dr. GRPO)

```python
def grpo_ngc_step(model, batch_prompts, G, delta, p0, b=32):
    """Dr. GRPO: group-normalized Advantages; L = L_token + L_mem."""
    for prompt in batch_prompts:
        rollouts = []
        for _ in range(G):
            cache, trace = run_rollout(
                model, prompt, delta,
                on_evict=lambda: gumbel_topk_keep(attn_blocks(cache, w=5, b=b), p0),
            )
            rollouts.append((trace, binary_correctness(trace)))
        r_bar = mean(r for _, r in rollouts)
        for trace, r in rollouts:
            A = r - r_bar
            L_tok = sum(log pi(next_t | replay_context(t, trace)) for t in gen_steps(trace))
            L_mem = sum(log p_gumbel(keep_set | s) for s in evict_moments_per_layer(trace))
            loss = -(L_tok + L_mem) * A
            loss.backward()  # grad clip, AdamW as in paper tables
```

**Appendix (meta-token g).** Forced summary before evict; gist-like when prefix fully evicted. Extension, not main tables.

## Empirical results

**50% of KV evicted per round (per layer)**; retention (1−ε)=0.5 → **~2.4×** peak-KV cut on Countdown, **3–5×** on DAPO math. Baselines = same schedule on a model RL-trained with full cache only.

| Benchmark | NGC | SnapKV | KeyDiff | KNorm | StreamLLM | Setting |
|-----------|-----|--------|---------|-------|-----------|--------|
| **Countdown** pass@1 (Fig. 3) | **49.6%** | 7.8% | 21.2% | 0.1% | 0.2% | ~2.4× vs full cache |
| **AIME 2025** pass@1 (Fig. 1) | **21.4%** | 10.7% | 7.1% | 0.0% | 0.0% | ~4.6× on those traces |
| **AMC 23/25** (Fig. 7) | best | 2nd varies by year & *k* | idem | near-zero in places | idem | pass@*k*, 3–5×; **no pass@1 table** in text |

Fig. 8: pass@32 vs reduction (AIME ~2.2×–4.6×; 2–3× band tracks upper bound). Train: Countdown 250 steps, δ=256, 1024 max, 32×16; DAPO 469 steps, δ=350, 1050/3850 train/test max, 256×8, eval top_p=0.95, T=0.6.

## Variants and ablations

- **Fig. 4 (Countdown):** NGC **49.6%** vs *token log-probs + replay only* (no L_mem) **35.7%** vs *Targeted KV dropout* (evict in rollouts, **causal** token loss) **2.5%** — replay **and** L_mem are both necessary.
- **Fig. 5:** Same staircase ε curriculum for Targeted dropout → reward collapse & grad explosion ~step 100 when ε ramps; NGC stable.
- **Interoception (Fig. 6):** `<eviction_rate>ρ%</eviction_rate>` in prompt: **+8–13%** pass@1 past training min cache.
- **DAPO:** *r* = 0 for completions shorter than first eviction (Appendix) to kill spurious positives.

## Failure modes and limitations

1. **Requires full RL + forked stack** (per-layer masks, DynamicCache, TRL/GRPO, replay forward)—not a closed-API switch.
2. **Binary / GRPO-suitable reward**; sparse or gameable tasks compound RL instability, and eviction can amplify it.
3. **1.5B-only results**; no 7B+ or production latency numbers for replay passes.
4. **Baselines are heuristics on a separately full-cache–trained model**; co-training fair comparators costs extra engineering.
5. **Periodic δ, ε** assumed; ad-hoc eviction cadences and tool-heavy harness traces need re-tuning; interoception helps ε transfer but is not a panacea.
6. **Appendix meta-token** path is illustrative, not a shipped feature in the main tables.

## When to use, when not to

- **Use** when you train open weights with GRPO/Dr.GRPO, peak KV (not MSA read sparsity alone) caps deployment, and you can implement replay masks + joint loss.
- **Use** when verifiable reward is available and you want keep/evict aligned to *outcomes*, not a static “importance” prior.
- **Do not use** for API-only inference, tiny contexts, or if you need non-destructive per-step sparsity only (MSA/NSA patterns).
- **Do not** omit replay or L_mem (35.7% / 2.5% vs 49.6% on Countdown).

## Implications for harness engineering

1. **NGC ≈ context compaction, but for KV, not the harness.** [08-context-compaction](08-context-compaction.md) trims *scaffold* context by policy you write; NGC trains *which* transformer KV blocks survive. Same “forget the scrap” problem, different layer—pairing them is plausible: tool-log compaction in the loop, NGC in long internal CoT if you own θ.
2. **External vs internal memory.** [12-todo-scratchpad-state](12-todo-scratchpad-state.md) is unbounded *file* state; the LM’s bounded state is still KV. Scratchpads offload durable intent; NGC optimizes *transient* CoT retention—compose both.
3. **Persistent mem hooks** ([72-claude-mem-persistent-memory-compression](72-claude-mem-persistent-memory-compression.md)) compress *across* turns without weight updates; NGC is *in-weight* co-training. Non-trainable products keep mem hooks; trainable open models get a third budget knob.
4. **Interoception in harnesses:** mirror `<eviction_rate>` with explicit `max_tokens` / `turn_budget` in structured system tags so the model can homeostat—same idea for compactor strength or when to subagent.
5. **Tool loops change δ effective cadence;** re-validate (δ, ε, interoception) when traces are not math-Cot-shaped.

## Connections to other work in this corpus

[01-agent-loop-architecture](01-agent-loop-architecture.md); [08-context-compaction](08-context-compaction.md); [12-todo-scratchpad-state](12-todo-scratchpad-state.md); [70-voltagent-awesome-ai-agent-papers](70-voltagent-awesome-ai-agent-papers.md); [72-claude-mem-persistent-memory-compression](72-claude-mem-persistent-memory-compression.md); [77-meta-tts-agentic-coding](77-meta-tts-agentic-coding.md) (Meta TTS: multi-rollout *summaries*; NGC: intra-rollout *KV*).

## Key takeaways

1. **Single reward, two discrete action types:** next-token and per-round eviction, optimized jointly with **L = L_token + L_mem** and identical GRPO advantage—no SFT, no distillation, no sparsity penalty.
2. **Gumbel top-k** supplies tractable log-probabilities for subset-keep decisions; **replay masks** and **on-graph recompute** fix off-policy token scoring and pass gradients into Q/K.
3. **At 2.4× peak KV on Countdown, 49.6% vs 21.2% (KeyDiff)** and near-zero for several heuristics; on AIME 2025 (Fig. 1) **21.4% vs 10.7% / 7.1%** for the strongest baselines shown.
4. **Budget interoception** (prompt-injected eviction rate) yields **8–13%** pass@1 improvements when test budgets are tighter than training minima.
5. Ablations show **replicate masks and eviction policy gradient** are each necessary; ignoring either yields 35.7% or 2.5% accuracy, respectively, vs 49.6%.

## References

- Li, Hamid, Fox, Goodman — arXiv:2604.18002 — 2026.
- Corpus: [08-context-compaction](08-context-compaction.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md), [72-claude-mem-persistent-memory-compression](72-claude-mem-persistent-memory-compression.md), [01-agent-loop-architecture](01-agent-loop-architecture.md), [70-voltagent-awesome-ai-agent-papers](70-voltagent-awesome-ai-agent-papers.md), [77-meta-tts-agentic-coding](77-meta-tts-agentic-coding.md).
