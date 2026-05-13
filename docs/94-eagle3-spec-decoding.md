# 94 — EAGLE-3: Speculative Decoding via Training-Time Test

**Paper.** Yuhui Li, Fangyun Wei, Chao Zhang, Hongyang Zhang — *EAGLE-3: Scaling up Inference Acceleration of Large Language Models via Training-Time Test* — Peking University, Microsoft Research, University of Waterloo, Vector Institute — arXiv:2503.01840 — 2025.

**One-line definition.** EAGLE-3 is **lossless speculative decoding** in which a small draft transformer consumes **fused low-/mid-/high-layer** hidden states from the frozen target model and is trained with **training-time test (TTT)**: the **feature-prediction loss is removed** so the draft head learns **direct next-token** objectives while simulated multi-step rollouts (with **draft attention masks** matching inference) keep training aligned with the acceptance dynamics at decode time.

## Why this paper matters (silent ~6× speedup for self-hosted models, validated on Llama 3.3 70B)

Autoregressive LLMs pay one full forward per token. For on-prem and open-weights stacks (vLLM, SGLang), the main latency lever that does not change the sampled distribution is **speculative decoding**: draft cheaply, verify in one parallel target pass, accept a prefix. EAGLE-3 continues EAGLE / EAGLE-2 but makes **draft training data scaling** move the Pareto curve: **up to 6.47×** speedup (HumanEval) and **4.12×** mean over five tasks on **LLaMA-Instruct 3.3 70B** at **T=0**—i.e. roughly **6×** vs vanilla on strong models without a separate frontier LM as drafter. Code (`SafeAILab/EAGLE`) plus **SGLang** / **vLLM** throughput tables make it a deployable inference module, not target-weight training.

## Problem it solves (autoregressive bottleneck, prior EAGLE limitations)

**Bottleneck:** decode is sequential; speculative sampling parallelizes *verification* of a draft chain or tree. **EAGLE** predicts the **next top-layer feature**, then maps through the target LM head, with **feature + token** loss. The authors show more drafter training data barely improves EAGLE speedup or acceptance—**scaling failure**. Causes: **feature prediction** over-constrains the drafter; **top-layer features** encode mainly the *next* token, so multi-step drafting is ill-posed. **EAGLE-2** adds a **dynamic draft tree** (inference-side) but leaves the EAGLE **training** pathology that EAGLE-3 removes.

## Core idea in one paragraph

EAGLE-3 drops matching intermediate features and trains a **one-layer** Transformer drafter on **token NLL** only, with **TTT**: for steps where the target has not yet produced hidden states (future positions), training **injects the drafter’s own prior outputs** and **embeddings of sampled draft tokens**—matching the real decode loop. Per-position input is **FC-fused** vectors **g** from **low, middle, high** target blocks instead of only the last layer. Verification stays **lossless** (target forward + standard accept); EAGLE-3 inherits EAGLE-2’s **context-aware** **tree** and **tree attention**. **Figure 1:** with the new architecture, more ShareGPT-relative scale **increases** speedup and accept length; EAGLE-2’s curve is **flat**—an **inference acceleration scaling law** they say was absent before.

## Mechanism (step by step)

### (a) Draft head architecture (smaller transformer)

**Drafter:** one **Transformer decoder layer** plus FCs. After the target forward on the current prefix, save **l, m, h** (low/mid/high layer chunks), each **k**-dim; **concat → 3k → FC → k** giving **g** per position. Step 1 of drafting uses **g** on verified positions only. Later steps lack **g** for unverified tokens—use prior **draft vector a** plus **embedding e** of the sampled draft token (Figure 5: “How can” → “I” → “do”). **Tree depth** rises from **6 to 8** vs EAGLE-2 at the **same node budget** because acceptance is higher.

### (b) Training-time test (TTT) loss

EAGLE used **l_fea** + **l_token**; EAGLE-3 **drops l_fea**. Training alternates a **normal** causal mask on gold tokens with **simulated** decode steps: attention masks as in **Figure 6** (diagonal blocks for draft rounds; tree-shaped dependencies). **HASS** also changes masks but still predicts **features** with **l_fea** and top-layer **f**; EAGLE-3 **eliminates** feature regression to avoid accumulation error and representation mismatch.

### (c) Tree decoding with verification

Same **speculative sampling** contract as Leviathan et al.; Chen et al.: **tree draft** (EAGLE-2 **prunes** branches from draft confidence), **one** batched target forward, **sequential** acceptance. For draft token **t̂**, accept with prob **min(1, p(t̂)/p̂(t̂))**; on reject, sample from **norm(max(0, p − p̂))** and discard the rest. EAGLE-3 only improves the drafter’s **p̂** and **τ**, not the rule.

```python
def speculative_accept_loop(prefix, draft, target, max_draft: int):
    out = list(prefix)
    while not done(out):
        draft_tokens, p_hat = draft.tree_draft(out, g_cache=from_last_target_forward)
        p = target.forward_par_verify(out, draft_tokens)
        for i, t_hat in enumerate(draft_tokens):
            a = min(1.0, p[i].prob(t_hat) / p_hat[i].prob(t_hat))
            if random() < a:
                out.append(t_hat)
            else:
                t_new = sample(norm_clip(p[i] - p_hat[i]))
                out.append(t_new)
                break
    return out
```

### (d) Acceptance rate scaling laws

**n-α** = P(n-th draft token accepted | prior accepted). **Figure 4:** removing **l_fea** helps **0-α** vs data, but without TTT **1-α** collapses (bad **â** at step 1). TTT (Figure 3, bottom) restores **1-α** so the chain benefits from data. **Figure 7** (MT-bench, 8B): **EAGLE**’s **n-α** falls with **n**; **EAGLE-3** stays flat—evidence TTT fixes self-conditioning.

### (e) The 6× speedup at scale

**Table 1 (T=0, task means):** EAGLE-3 **5.51×** (Vicuna 13B), **4.44×** (LLaMA-Instruct 3.1 8B), **4.12×** (LLaMA-Instruct 3.3 70B), **4.16×** (DeepSeek-R1-Distill-LLaMA 8B). Vanilla **1.0×**; standard SpS **~1.7–1.9×**; EAGLE-2 **2.85–4.22×** on those rows. Peaks: **6.47×**, **τ = 7.54** (HumanEval, Vicuna 13B); R1 on GSM8K **5.01×**, **τ = 6.93**. Abstract **~6.5×** matches code-heavy tasks. L33 70B: per-task speedup **~3.27–4.79×**, **τ ~5.0–6.5** (Table 1). Paper reports **~20–40%** gain over EAGLE-2 aggregated.

## Empirical results (table — speedup vs vanilla decoding, vs EAGLE-2, on various model sizes; acceptance rate)

| Target model | EAGLE-2 speedup (mean τ) | EAGLE-3 speedup (mean τ) | Notes (T=0) |
|--------------|--------------------------|--------------------------|------------|
| Vicuna 13B | 4.22× (4.83) | **5.51× (6.62)** | Best task 6.47× (HumanEval) |
| LLaMA-Instruct 3.1 8B | 3.23× (4.11) | **4.44× (6.23)** | Ablations Table 2 on this target |
| LLaMA-Instruct 3.3 70B | 2.85× (3.78) | **4.12× (5.88)** | Self-hosted 70B validation |
| DeepSeek-R1-Distill-LLaMA 8B | 3.26× (3.92) | **4.16× (5.84)** | Draft trained with OpenThoughts-114k-math |
| LLaMA 3.3 70B | 2.65× (3.54) | **3.45× (4.92)** | T=1; sampling narrows gaps |

**Throughput (concurrent serving):** SGLang H100, L31 8B, MT-bench: EAGLE-3 **1.38×** vs no spec at **bs=64**; EAGLE can fall **below 1×** at high batch (Table 3). **bs=1** tok/s: **373.25** (EAGLE-3) vs **158.34** (no spec), **244.10** (EAGLE-2) — Table 4. vLLM A100, chain length 2 (Table 5): EAGLE-3 **1.21×** at bs 48, **1.01×** at 56 vs EAGLE **0.82× / 0.71×**—spec that still helps under load.

## Variants and ablations

**Table 2 (L31 8B):** EAGLE-2 **3.16×**, **τ 4.05** (MT-bench) → **+ remove feature constraint** **3.82×**, **5.37** → **+ fused features** **4.40×**, **6.13** (GSM8K **4.48× / 6.23**). Training: ShareGPT **200K** / **464K** entries (~**8×** more data than EAGLE in their setup), responses from the **target** model. **AdamW** **β=(0.9,0.95)**, **lr 5e-5**, **clip 0.5**. Tree **depth 8** vs EAGLE-2’s **6**, same expansion nodes.

## Failure modes and limitations

Authors did not run **405B / 671B** (GPU). **Medusa** at **T=1** excluded from greedy comparisons where relaxed acceptance breaks **lossless** alignment (Table 1 note). **Task-dependent:** code and templated tasks peak; open chat/summarization lower **τ**. **High batch:** memory-bound story weakens; gains are **~1–1.4×** throughput in SGLang/vLLM tables, not **5×**. **Overhead:** feature capture, tree, kernel integration. **Per-checkpoint** draft: new base weights need **retrained** drafter. **R1**-style models stay **long**; EAGLE-3 cuts **per-step** cost, not reasoning **horizon** variance.

## When to use, when not to

**Use:** you control the **serving** stack and **weights**; you can **train** a draft on **in-domain** ShareGPT-like (and optional math) data; decode latency or $/token dominates; you need **distribution-matched** lossless decoding to the target (greedy / same sampling protocol). **Avoid:** **API-only** black-box LMs with no hidden states; no **spec** support in the engine; expecting this to substitute **routing** or **prompt** quality; **reusing** a draft across **incompatible** checkpoints.

## Implications for harness engineering. Reference [29-dive-into-claude-code](29-dive-into-claude-code.md), [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md), [86-frugalgpt](86-frugalgpt.md). Position as: EAGLE-3 is the inference-side cost lever orthogonal to routing; together routing + spec decoding deliver compounding speedups

[29-dive-into-claude-code](29-dive-into-claude-code.md) maps the harness to **loop**, **tools**, and **policy**; [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md) places **state** and **context** as reliability layers—they do not reduce **per-token** compute at the LM. **EAGLE-3** is a **serving** optimization: fewer target steps per emitted token for a **fixed** policy model. [86-frugalgpt](86-frugalgpt.md) splits savings into **prompt**, **approximation**, and **cascade routing**; **EAGLE-3** is **orthogonal**—it speeds the **forward** of whichever model the router **selects**. **Router** saves **queries** (cheap-first, escalate); **spec** saves **steps** on **expensive** trunk calls—**effects compound** on cost×latency when both are tuned. Sandboxes, compaction, hooks remain **harness** concerns; EAGLE-3 only cheapens the LM **node** where the runtime exposes it.

## Connections to other work in this corpus

- [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md), [88-confidence-driven-router](88-confidence-driven-router.md): **routing** vs **intra-model** decode acceleration—production stacks combine both.  
- [77-meta-tts-agentic-coding](77-meta-tts-agentic-coding.md): Meta TTS spends inference for **quality** (summaries, tournaments); EAGLE-3 adds **draft** work for **latency**—different objective.  
- Spec-Bench / speculative-decoding survey line (cited by authors): positions EAGLE-3 in open **tree + hidden-state** drafting.  
- Skills / memory / Voyager entries: no change to **retrieval**; assumes standard LM **inference** hooks.

## Key takeaways

1. **Drop l_fea**; use **TTT masks** so training **matches** inference self-conditioning → **data scaling** for draft quality (Figure 1).  
2. **Multi-layer fusion g** instead of top-layer-only **f** for all steps.  
3. **~4.1–5.5×** mean speedup vs vanilla on reported open models; **~6.5×** task peaks; **~20–40%** over EAGLE-2 (paper’s aggregate claim).  
4. **SGLang:** **1.38×** throughput at **bs=64**; EAGLE-3 stays **above 1×** where EAGLE **degrades** at batch.  
5. Needs **white-box** target **hidden states** and **per-checkpoint** draft training.

## References

Yuhui Li, Fangyun Wei, Chao Zhang, Hongyang Zhang. *EAGLE-3: Scaling up Inference Acceleration of Large Language Models via Training-Time Test.* arXiv:2503.01840, 2025. Code: `https://github.com/SafeAILab/EAGLE`.

Leviathan, Kalman, Matias — speculative sampling, ICML 2023; Chen et al. — arXiv:2302.01318; Yuhui Li et al. — *EAGLE* (ICML 2024), *EAGLE-2* (EMNLP 2024). Zheng et al. — SGLang, NeurIPS 2024; Kwon et al. — vLLM, SOSP 2023. Benchmarks: MT-bench, HumanEval, GSM8K, Alpaca, CNN/DM. Training data: ShareGPT (Ding et al., 2023); OpenThoughts-114k-math for R1-distill draft.

