# 80 — KnowRL: Knowledgeable Reinforcement Learning for Factuality

**Paper.** Baochang Ren, Shuofei Qiao, Ningyu Zhang, Da Zheng, Huajun Chen — *KnowRL: Exploring Knowledgeable Reinforcement Learning for Factuality* — arXiv:2506.19807 — Zhejiang University / Ant Group — 2025 (arXiv v4, 16 Apr 2026).

**One-line definition.** KnowRL augments **GRPO** with a **process-level factuality reward**: decompose chain-of-thought into atomic facts, verify each fact against retrieved knowledge \(K_x\), and add the score to format and **correctness (including explicit refusal)** rewards so the policy learns **faithful steps** and **knowledge-boundary** behavior.

## Why this paper matters

“Slow thinking” models can climb **GPQA**-style reasoning curves yet **stagnate or worsen** on **SimpleQA**-style factuality (Figure 2: larger DeepSeek-R1–distill checkpoints improve GPQA but not hallucination resistance). **Outcome-only RL** widens a **supervision gap**: a **correct final answer** with **fabricated** intermediate steps still yields reward, so the policy can learn to be articulate and “lucky” without being **grounded**—the **knowledge-boundary** problem in training form.

Long-horizon agents amplify this: one wrong fact early can **snowball** across tools and steps. KnowRL treats **grounding the reasoning trace** (not just the string answer) as the object of RL, with **verifiable** facts as the interface—arguing for **objective-level** factuality work **before** you spend inference budget on band-aids.

The work is also a reminder that **“more reasoning”** and **“more factuality”** are not a single axis: the paper reports **300-example** slices for long-CoT evals because each trace is long—exactly the regime where a **one-number outcome reward** is an inadequate proxy for what went wrong in the **middle** of the chain. KnowRL is one concrete way to make that middle **supervised**.

## Problem it solves

1. **Outcome-oriented RL for slow thinking** optimizes the final answer while treating the trace as a black box, which can **reinforce fabricated reasoning** when the answer is still judged correct.
2. **Missing knowledge boundaries**: models are incentivized to guess to maximize reward when they should **abstain** (“I don’t know” / explicit refusal).
3. **Inferential–factual disconnection**: strong scores on tasks like GPQA or math olympiads can coexist with collapse on open-domain factuality (e.g. SimpleQA), unless training explicitly couples reasoning with **grounded** steps.
4. **Alternatives:** **RAG** is poor at verifying *every* long step; **SFT** risks **catastrophic forgetting** and rote fact injection without boundary judgment; post-hoc filters do not replace **in-RL** credit on the model’s own CoT.

## Core idea in one paragraph

Pair each training prompt with **external knowledge** (a subset \(K_x\) of a knowledge base \(K\), retrieved per prompt). For every rollout, split the thinking segment into **atomic facts** \(\Phi(o_{\mathrm{think}}) = \{f_1,\ldots,f_M\}\) and assign each fact a binary support score \(v(f_j, K_x) \in \{0,1\}\) via a verifier (GPT-4o-mini in the main experiments). Aggregate these into a **factuality reward** \(r_{\mathrm{fact}}\) (proportion of supported facts, or 0 if \(M=0\)). Combine \(r_{\mathrm{fact}}\) with a **format** reward (required tags for thinking/answer) and a **correctness** reward on the final answer (including positive credit for **explicit refusal**). Optimize with **GRPO-style group-relative advantages** plus entropy and KL regularization against a frozen reference policy so trajectories with better grounded intermediate reasoning receive positive credit, not only final-string matches.

## Mechanism (step by step)

1. **Data + \(K_x\).** Training prompts are paired with a KB; **gtr-t5-large** (sentence-transformers) retrieves **\(K_x\)** (Figure 3, Appendix B).

2. **Rollout, format, atoms.** The policy outputs a delimited **thinking** + **answer** block. **\(r_{\mathrm{format}} \in \{+1,-1\}\)** enforces that schema. The thinking string is decomposed into atomic facts \(\{f_j\}\) (**FactScore-style** granularity, per §2).

3. **Factuality reward.** Each \(f_j\) is checked vs **\(K_x\)**; **\(v(f_j,K_x)\in\{0,1\}\)**. **\(r_{\mathrm{fact}}=\frac1M\sum_j v\)** for \(M>0\), else **0** (Eq. 1).

4. **Correctness (with refusal).** **GPT-4o-mini** scores the final answer: **+2** correct, **+1** **explicit refuse**, **−1** wrong (Eq. 2: **\(R_{\mathrm{total}}=r_{\mathrm{format}}+r_{\mathrm{correct}}+r_{\mathrm{fact}}\)**).

5. **GRPO update.** Sample **G** rollouts / prompt from **\(\pi_{\theta}^{\mathrm{old}}\)**; **group-relative** advantages **\(A_g=(R_g-\mu_x)/(\sigma_x+\varepsilon)\)** (Eq. 3), **clipped** surrogate **\(\hat J(\theta)\)** (Eq. 4), loss **\(-\hat J+\beta_H H+\beta_{\mathrm{KL}}KL\)** (Eq. 5). **LoRA** r=128, \(\alpha=256\), **lr=1e-5** (Table 5).

6. **Eval protocol.** **300** examples each: TruthfulQA, SimpleQA, ChineseSimpleQA; **T=0**. TruthfulQA: **ROUGE/BLEU/PAQ**. **Hallucination:** Incorrect%, Refusal, PAQ, F1. **Reasoning:** **GPQA Diamond**, **AIME 2025**. (Paper does **not** report **FactScore** on model outputs in main tables; **FActScore** is the *decomposition* prior; **FactTune-FS** baselines use FactScorer in training-data construction.)

## Empirical results

Headline: **DeepSeek-R1-Distill-Qwen-7B** **SimpleQA Incorrect** **78.00% → 57.67%** (−20.3 pp vs the 78% baseline), with **ChineseSimpleQA** and **TruthfulQA** (Rouge/Bleu) moving in the same direction on several rows (Table 2). **DPO/SFT/FactTune-FS/TruthRL** are mixed: e.g. DPO can cut Incorrect% while hurting PAQ or other columns; **TruthRL** underperforms KnowRL on the same grid.

| Model | TruthfulQA (Rouge / Bleu) | SQA Inc% | CSQA Inc% | GPQA Diamond | AIME 2025 |
|--------|---------------------------|----------|------------|---------------|----------|
| Skywork-OR1-7B (zero-shot) | 56.67 / 55.33 | 76.33 | 67.00 | 37.37 | 26.67 |
| Skywork-OR1-7B (KnowRL) | **57.67** / **54.33** | **60.33** | **52.33** | **42.42** | **36.67** |
| DeepSeek-7B (zero-shot) | 53.33 / 51.00 | 78.00 | 68.33 | 40.91 | 30.00 |
| DeepSeek-7B (KnowRL) | 57.33 / 51.60 | **57.67** | **58.33** | 36.87 | 33.33 |

**Avg@5, T=0.6** (Table 4): e.g. DeepSeek-7B **AIME 29.33→34.00%**. **14B** (Table 6): SQA Inc **83→68.33%**, **GPQA 46.97→51.01%**, **AIME 40→36.67%** (↓ on AIME for that run).

## Variants and ablations

- **Table 3 (DeepSeek-7B):** **\(r_{\mathrm{format}}+r_{\mathrm{fact}}\)** maxes **AIME 40%** / **GPQA 47.47** in that ablation but **SQA Incorrect 80.67%**; **full \(R_{\mathrm{total}}\)** is better **overall**. **Format+\(r_{\mathrm{correct}}\)** w/o **\(r_{\mathrm{fact}}\)** can look good on SQA but **break CSQA** (e.g. very high incorrect%)—**process** signal matters.
- **Refusal:** flipping refusal to **negative** in the study raises incorrect rate and **reward-hacking** (Figure 5); **positive** refusal credit stabilizes **abstain**.
- **RL:** **DAPO, BNPO, Dr.GRPO** all beat zero-shot on Incorrect%; trade-offs on PAQ/AIME differ.
- **Evaluator:** **Qwen2.5-72B** vs **GPT-4o-mini** during training (Table 7) → same broad story; GPT-4o-mini somewhat **stricter** on SQA, Qwen nudges **GPQA** in one setting.

## Failure modes and limitations

Bad **\(K_x\)** or **retrieval** → noisy **\(v(f_j,K_x)\)**. **LM verifier** on the training path: **cost + judge bias**, not a proof system. **Decomposition** \(\Phi\) can mangle claims. **Too many RL steps** → overfit (§3.4; Figure 4). **14B:** **AIME** can **drop** while SQA/GPQA improve. **Open:** cross-lingual mechanism (EN KB, ZH test), **multimodal** extension (Limitations in paper).

## When to use, when not

**Use** when you control post-training, have (or can build) a **knowledge-backed** prompt set, need **slower, boundary-aware** behavior from a reasoning model, and can pay for **per-step** verification at training time. **Avoid** (or add mitigations) when no stable KB exists, when **judge** bias is unacceptable, or when **latency/cost** rules out dense verification; in those cases **harness-time** tools or lighter rewards may be the pragmatic choice.

## Implications for harness engineering

- **[25-agentic-rag](25-agentic-rag.md).** **Product RAG** feeds context; **KnowRL** shapes a **weight-level habit** of checking claims against **a KB during training**—complementary, not substitutable.
- **[18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md).** **CoVe / self-refine** = **harness-time** (second best when weights are fixed or domains shift fast). **KnowRL** = **train-time** factuality and refusal shaping **where** you own post-training. Prefer (1) then add (2).
- **[22-guardrails-prompt-injection](22-guardrails-prompt-injection.md).** **Abstain-on-unsupported** behavior helps **epistemic** robustness; it does **not** replace **trust boundaries** or **policy** for injection, tools, and data exfil.

**Order:** (1) **train** (KnowRL-class) if feasible → (2) **runtime** verification + RAG → (3) **guardrails**.

## Connections to other work in this corpus

[77](77-meta-tts-agentic-coding.md) scales **inference**; KnowRL scales **supervision at train time**—real systems combine. CoVe (paper cites Dhuliawala et al.) is the **inference** analogue of their **atomic** training signal.

## Key takeaways

1. **Process** \(r_{\mathrm{fact}}\) targets **outcome-RL** that rewards **lucky** or **dishonest** CoT.
2. **Ablations** (Table 3) show **format**, **correctness (incl. refusal)**, and **\(r_{\mathrm{fact}}\)** are jointly load-bearing; **removing** any piece yields **pathological** trade-offs.
3. **GRPO** plus group-relative **\(A_g\)** is one implementation; **DAPO / BNPO / Dr.GRPO** slots preserve the same **factual** story.
4. **ChineseSimpleQA** gains with **English-leaning** training KBs are **reported** as transfer of **verification behavior**, not proven mechanistically.
5. **Reasoning** metrics (**AIME** on 7B/14B) are **not** uniformly monotonic—treat as a **multi-objective** result.

## References

- Ren, B., Qiao, S., Zhang, N., Zheng, D., Chen, H. *KnowRL: Exploring Knowledgeable Reinforcement Learning for Factuality.* arXiv:2506.19807, 2025. Code: `github.com/zjunlp/KnowRL` (per paper).
- Min, S. et al. FActScore (atomic factual precision). Lin et al. TruthfulQA. Wei et al. SimpleQA. He et al. ChineseSimpleQA. Guo et al. DeepSeek-R1 / distillation; He et al. Skywork-OR1 (baselines in experiments).
