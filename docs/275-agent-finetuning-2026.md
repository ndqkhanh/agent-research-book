# 275 — Agent Finetuning 2026: SFT × DPO × RLVR × distillation production playbook

**Anchors.** Tülu 3 — arXiv:2411.15124 — Allen AI's open post-training recipe. DeepSeek-R1 — arXiv:2501.12948 ([232-rl-on-reasoning-traces-scaling](232-rl-on-reasoning-traces-scaling.md)). Direct Preference Optimization (DPO) — Rafailov et al., arXiv:2305.18290. Constitutional AI — Anthropic 2022 → 2024–2026 refinements. RLVR (RL with Verifiable Rewards) — Tülu-3 chapter; production at Meta, DeepSeek, Moonshot. Distillation — Hinton 2015 → R1-Distill (2025–2026). Companions: [232-rl-on-reasoning-traces-scaling](232-rl-on-reasoning-traces-scaling.md), [216-pretraining-scaling-laws-foundation](216-pretraining-scaling-laws-foundation.md), [217-test-time-compute-scaling](217-test-time-compute-scaling.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md), [97-qwen-prm](97-qwen-prm.md), [80-knowrl](80-knowrl.md), [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md), [235-inference-compression-scaling](235-inference-compression-scaling.md).

**One-line definition.** A 2026 production playbook for **post-training your own agent model** — the **canonical four-stage pipeline** (Stage 1: **SFT** on curated demonstrations, Stage 2: **DPO** on preference pairs, Stage 3: **RLVR** with rule-based verifiable rewards on math / code / format, Stage 4: **distillation** from the trained teacher to deployment-size students) — with **Tülu-3** (Allen AI, Nov 2024, fully open recipe) as the **reference open implementation** that any team can reproduce on Llama / Qwen / Mistral base, and **DeepSeek-R1** as the proof that pure RLVR alone elicits o1-class reasoning from a strong base; the production economics: **$50K–$5M for the teacher run** (depending on scale), **$5K–$50K for distillation**, with the resulting student model running at **5–10% of frontier-API cost** while reaching 85–95% of teacher capability — the highest-ROI capability-engineering move in 2026 for teams with consistent task domains.

## Why this matters (when prompt + context engineering hits a ceiling, finetuning is the next lever)

Production agent teams in 2026 hit two ceilings prompt engineering alone can't break: (a) **per-task cost** — even with prompt caching, GPT-4-class API calls at $5–15/M tokens add up; (b) **task-specific quality** — domain-specific reasoning (medical, legal, financial, scientific) often plateaus before reaching expert-grade. **Finetuning** breaks both ceilings: train (or distill) a model on your domain, run it on cheaper infra, and frequently **outperform a frontier API on your specific task** at 5–10× lower cost. The 2025-2026 inflection: **open recipes** (Tülu-3) and **strong open base models** (Llama-3, Qwen-2.5, DeepSeek-V3) make this accessible to teams without hyperscaler infra.

The production playbook converged on a four-stage pipeline. **Stage 1 — SFT** (Supervised Fine-Tuning): curate ~10K-100K (prompt, gold-response) pairs from human writers, distillation from a stronger teacher, or synthetic generation; train base model with standard cross-entropy loss; this is the cheapest and gives the largest jump. **Stage 2 — DPO** (Direct Preference Optimization): curate ~10K (prompt, preferred, rejected) preference pairs; train with the DPO loss to prefer chosen over rejected; gives instruction-following + helpfulness uplift. **Stage 3 — RLVR** (RL with Verifiable Rewards): curate prompts with rule-based correctness checks (math gold answers, code test suites, format regex); run GRPO/PPO on these prompts with binary rewards; elicits reasoning behavior including emergent long thinking traces (DeepSeek-R1 §3.2 "aha moment"). **Stage 4 — Distillation**: use the trained teacher to generate (prompt, completion) data; train smaller students on that data via SFT; students retain 85-95% of teacher capability at 5-15% of inference cost.

Take this seriously and three things change. **First**, **finetuning is reachable** for teams without hyperscaler resources — Tülu-3 ran on ~$50K of compute for the full pipeline on Llama-3.1-8B. **Second**, **distillation is the deployment lever** — train the largest teacher you can afford (or use a published thinking model like R1), distill to 7B/14B/32B for serving. **Third**, the **eval pipeline ([265](265-agent-evaluation-2026.md)) drives the recipe** — every stage's value is measured on held-out evals; without this, finetuning becomes vibes-driven.

## Core idea

The four-stage canonical post-training pipeline:

| Stage | Method | Data | Compute | Lift |
|---|---|---|---|---|
| 1 SFT | Cross-entropy on (prompt, gold) | 10K-100K curated pairs | Lowest | Largest single jump |
| 2 DPO | Preference loss on (prompt, chosen, rejected) | 10K preference pairs | Low | Helpfulness + instruction-following |
| 3 RLVR | GRPO/PPO with rule-based rewards | Math + code + format prompts | Medium-high | Reasoning, emergent thinking |
| 4 Distill | SFT on teacher outputs | Generated by teacher | Medium | Compress for deployment |

Each stage adds independently; ablations (Tülu-3 §5) show all four matter.

## Mechanism (step by step)

### (a) Stage 1 — SFT data sources and curation

- **Human-curated demonstrations.** Most expensive, highest quality. ~$10-50/example for expert work.
- **Distillation from teacher.** Strong-model rollouts on your prompts; cheap, scales.
- **Synthetic generation.** Self-instruct, Magpie, evol-instruct. Cheap; quality varies.
- **Public datasets.** ShareGPT, OpenHermes, Tülu-3 SFT mix.

Tülu-3's mix: ~939K SFT examples mixing reasoning, math, code, multilingual, safety. Train 2-3 epochs on Llama-3.1-8B at modest LR (~5e-6).

### (b) Stage 2 — DPO

DPO loss (Rafailov 2023):

```
L_DPO(θ) = -E[log σ(β · (logp_θ(y_w|x) - logp_ref(y_w|x) - logp_θ(y_l|x) + logp_ref(y_l|x)))]
```

where (x, y_w, y_l) is (prompt, chosen, rejected). Tülu-3 mix: ~270K preference pairs from human + LLM-judge. Train 1-2 epochs at lower LR.

### (c) Stage 3 — RLVR

```python
def compute_reward(prompt, completion):
    if is_math(prompt):
        return 1.0 if extract_answer(completion) == gold_answer(prompt) else 0.0
    elif is_code(prompt):
        return 1.0 if all_tests_pass(completion, test_suite(prompt)) else 0.0
    elif is_format(prompt):
        return 1.0 if matches_format(completion, format_spec(prompt)) else 0.0
```

GRPO ([232-rl-on-reasoning-traces-scaling](232-rl-on-reasoning-traces-scaling.md)) over K rollouts per prompt with group-baseline normalization. ~50K-1M prompts; 1000-10000 GRPO steps. Length emergence: trace length grows from ~200 tokens at start to 5K-30K at convergence; "aha moment" reflective tokens emerge spontaneously.

### (d) Stage 4 — Distillation

```python
# Generate teacher data
teacher_completions = [teacher.generate(prompt) for prompt in distill_prompts]

# SFT student on teacher output
student_loss = cross_entropy(student.forward(prompt), teacher_completion)
```

R1-Distill: 800K teacher completions across math + code + science + reasoning; SFT Qwen-7B/14B/32B and Llama-8B/70B; AIME results reach 55-72% (vs R1's 79.8%).

### (e) Eval-driven gates

Every stage gated by eval:

```python
stage_1_passed = eval_suite.run(model_after_sft) > eval_suite.run(base) + 5
stage_2_passed = eval_suite.run(model_after_dpo) > eval_suite.run(model_after_sft) + 1
stage_3_passed = eval_suite.run(model_after_rlvr) > eval_suite.run(model_after_dpo) + 5  # AIME
stage_4_passed = eval_suite.run(distilled_student) > 0.85 * eval_suite.run(teacher)
```

If a stage doesn't lift, debug data quality before proceeding.

### (f) Cost economics

| Stage | Compute | Wall-clock | Cost (cloud) |
|---|---|---|---:|
| Llama-3.1-8B SFT (3 epochs, 939K) | 8× H100 × 24h | 24h | ~$2K |
| DPO (270K, 1 epoch) | 8× H100 × 8h | 8h | ~$700 |
| RLVR (10K prompts, 5K steps) | 16× H100 × 72h | 72h | ~$25K |
| Distill 70B → 7B (800K) | 8× H100 × 48h | 48h | ~$4K |
| **Total Tülu-3 reproduction** | | | **~$32K** |
| DeepSeek-R1 reproduction | 2048× H800 × weeks | ~$5.5M reported | $5M+ |
| R1-distill-7B | 16× H100 × 24h | 24h | ~$2K |

Mid-scale finetuning is well within range of small teams.

### (g) Production deployment of finetuned model

Quantize ([235-inference-compression-scaling](235-inference-compression-scaling.md)): INT4 via GPTQ/AWQ. Serve via vLLM / TensorRT-LLM. Speculative decoding for thinking models. Result: 7B INT4 R1-Distill at ~$0.10/M tokens vs frontier API at $5-15/M — ~50-100× cheaper.

## Empirical results (table)

**Tülu-3 (Llama-3.1-8B, Allen AI 2024, [275 anchor]):**

| Stage | MATH | GSM8K | IFEval | AlpacaEval LC |
|---|---:|---:|---:|---:|
| Base | 17.2% | 56.7% | 49.4% | 22.8% |
| + SFT | 31.5% | 76.2% | 64.2% | 35.4% |
| + DPO | 33.9% | 79.4% | 66.4% | 40.0% |
| + RLVR | **35.7%** | **82.5%** | **75.4%** | **41.6%** |

**DeepSeek-R1 / Distill series ([232](232-rl-on-reasoning-traces-scaling.md)):**

| Model | AIME-2024 | MATH-500 |
|---|---:|---:|
| R1-Zero (pure RL) | 71.0% | 95.9% |
| R1 (SFT + RL) | 79.8% | 97.3% |
| R1-Distill-Qwen-32B | 72.6% | 94.3% |
| R1-Distill-Qwen-7B | 55.5% | 92.8% |

## Variants and ablations

- **Constitutional AI for safety alignment.** Self-critique on harm; explicit principles.
- **Online vs offline DPO.** Online (sample, judge, train) higher quality.
- **Synthetic preference data.** LLM-judge generates pairs; cheap but biased.
- **Multi-task RLVR.** Math + code + logic combined.
- **PRM-as-reward.** Use process reward model ([97-qwen-prm](97-qwen-prm.md)) for non-rule-based domains.
- **LoRA / QLoRA finetuning.** Low-rank adapters; cheap; less capability lift.
- **Continued pretraining.** Domain corpora before SFT.
- **Curriculum learning.** Easy → hard during RLVR.

## Failure modes

- **Reward hacking.** [97-qwen-prm](97-qwen-prm.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md). Tight verifiers required.
- **Catastrophic forgetting.** Heavy RL degrades general capability; mix general + reasoning data.
- **Length explosion.** [232-rl-on-reasoning-traces-scaling](232-rl-on-reasoning-traces-scaling.md) — runaway thinking traces.
- **Distillation ceiling.** ~85-95% retention; last 5-15% requires full RL.
- **Eval-set contamination.** Public benchmarks leak into pretraining.
- **Compute budget overrun.** RLVR runs are weeks-long; gate carefully.
- **Mode collapse.** Without entropy / KL penalties, single high-reward strategy dominates.
- **Distribution shift in deployment.** Training distribution ≠ production traffic.

## When to use, when not

**Adopt full pipeline** for teams with consistent task domain, sufficient compute budget ($30-100K minimum), eval pipeline maturity, and 6-month payback horizon for inference savings. Strongest cases: **high-volume reasoning agents** (math tutoring, code review, scientific QA), **regulated-domain agents** (legal, medical) requiring on-premises deployment, **specialized verticals** (finance, security).

**Skip finetuning** for low-volume agents (API cost < finetune amortization), heterogeneous tasks (general-purpose serves better), early prototypes (premature optimization), or teams lacking eval infrastructure.

## Implications for harness engineering

- **`harness_core/finetuning/` package.** Tülu-3-shape pipeline as reusable infra.
- **Eval pipeline gates each stage.** [265-agent-evaluation-2026](265-agent-evaluation-2026.md).
- **Distillation as deployment lever.** [235-inference-compression-scaling](235-inference-compression-scaling.md).
- **PRM as reward source.** [97-qwen-prm](97-qwen-prm.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md).
- **Cross-channel verifier as judge.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md).
- **HeavySkill RLVR.** [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md).
- **KnowRL.** [80-knowrl](80-knowrl.md).
- **Compliance for training-data provenance.** [272-agent-compliance-and-audit](272-agent-compliance-and-audit.md).
- **Supply-chain audit on finetuned model.** [270-agent-supply-chain-security](270-agent-supply-chain-security.md).
- **Sleeper-agent detection.** [270-agent-supply-chain-security](270-agent-supply-chain-security.md).
- **Cost router with finetuned-tier.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md).

**One-line takeaway.** **Production agent finetuning in 2026 is a four-stage pipeline (SFT + DPO + RLVR + distillation) with Tülu-3 as the open reference recipe at ~$30K compute; the highest-ROI move for teams with consistent task domain and inference-cost pressure — train (or distill) a domain-specialized teacher, distill to deployment-size, quantize for serving, and run at 5-10% of frontier-API cost while frequently outperforming the API on your specific task.**
