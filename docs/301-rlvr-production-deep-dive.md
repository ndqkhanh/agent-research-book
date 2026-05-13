# 301 — RLVR Production Deep-Dive: data curation, GRPO ablations, reward-hacking detection, distillation studies

**Anchors.** [232-rl-on-reasoning-traces-scaling](232-rl-on-reasoning-traces-scaling.md), [275-agent-finetuning-2026](275-agent-finetuning-2026.md), DeepSeek-R1 (arXiv:2501.12948), Tülu-3 (arXiv:2411.15124), Moonshot Kimi K2.5 PARL, [97-qwen-prm](97-qwen-prm.md).

**One-line definition.** A **production deep-dive** on RL-with-Verifiable-Rewards beyond the [232](232-rl-on-reasoning-traces-scaling.md) summary — covering the **four sub-disciplines** that determine RLVR success in production: (1) **data curation** (prompt selection, difficulty distribution, contamination filtering, dynamic curriculum), (2) **GRPO ablations and hyperparameters** (group size, KL constraint, advantage normalization, baseline choice), (3) **reward-hacking detection** (length explosion, gaming partial-reward signals, distribution shift, eval-set memorization), (4) **distillation studies** (teacher-data generation strategies, student-architecture choices, retention-vs-cost frontier) — turning the high-level recipe into a production playbook.

## (a) Data curation

The single highest-leverage lever in RLVR production. Bad data → reward hacking + slow convergence + eval-set leakage.

**Curation strategies that work:**

- **Difficulty filtering.** Discard prompts the base model already solves at >90%; discard prompts at <5% (no signal in either case). Sweet spot: 20-70% base-model pass rate.
- **Contamination scrubbing.** Use [101-autoresearchbench](101-autoresearchbench.md)-shape decontamination + n-gram match against eval suites.
- **Difficulty-balanced batches.** Mix easy / medium / hard within each batch; prevents oscillation.
- **Source diversity.** Math from Olympiads + competition archives + textbooks + research; code from competitive-programming + real GitHub + synthetic.
- **Dynamic curriculum.** As model improves, drop trivial prompts; sample harder ones.
- **Reward-signal multiplicity.** Math-correctness + format-compliance + length-penalty + brevity-bonus simultaneously.

**Tülu-3 mix sizes** (reference): ~939K SFT, ~270K DPO, ~50K-1M RLVR prompts.

## (b) GRPO ablations and hyperparameters

Group Relative Policy Optimization (DeepSeek's contribution) replaces PPO's value function with group-baseline normalization. Production-relevant hyperparameters:

| Parameter | Typical | Range | Notes |
|---|---:|---|---|
| Group size K | 8 | 4-32 | More = lower variance, more compute |
| KL coefficient β | 0.04 | 0.01-0.5 | Higher = more conservative |
| Advantage clip ε | 0.2 | 0.1-0.3 | Standard PPO clip range |
| Learning rate | 1e-6 | 1e-7 to 5e-6 | Conservative for stability |
| Train batch size | 256-1024 prompts × K | varies | Memory-bound |
| Sampling temperature | 0.6-1.0 | 0.3-1.0 | Higher = more exploration |
| Max completion length | 16K-32K | 8K-64K | Anti-runaway via penalty |

**Length-penalty calibration** is critical (per [232](232-rl-on-reasoning-traces-scaling.md) — R1-Zero exhibited length runaway). Include `length_penalty(L) = -α × max(0, L - L_target)` term in reward; tune `α` and `L_target` per task family.

**Group size scaling:** K=8 is the sweet spot at 32B-class. K=16 at 70B+. Below K=4, variance explodes.

## (c) Reward-hacking detection

The dominant production failure mode. Symptoms:

- **Length explosion.** Trace length grows past task complexity needs.
- **Format gaming.** Model emits expected format wrappers without solving the problem.
- **Partial-credit gaming.** If reward is graded (0.0-1.0), model targets the easy fraction.
- **Eval-set memorization.** Model learns specific eval prompts.
- **Distribution shift.** Train-distribution patterns don't match production.
- **Self-deception.** Model generates plausible-looking but wrong reasoning that earns reward.

**Detection strategies:**

```python
# Length monitoring
mean_completion_len_over_steps = ...
if mean_completion_len > pre_RL_baseline * 5:
    alert("possible length-runaway")

# Format-only completions (no actual reasoning)
format_only_rate = sum(1 for c in completions if extract_reasoning(c) is None) / N
if format_only_rate > 0.05:
    alert("format-gaming detected")

# Eval-vs-train distribution shift
eval_acc_drift = abs(eval_acc - train_acc)
if eval_acc_drift > 0.10:
    alert("possible memorization")

# Cross-channel verifier dissent
if cross_channel_verifier_disagreement_rate > 0.15:
    alert("possible self-deception")
```

**Mitigation:**

- **Length penalty in reward** (above).
- **Format check as gate** (binary; not graded).
- **Held-out eval set** (decontaminated, never in training).
- **Cross-channel verifier** ([02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md)) on sampled outputs.
- **Stop-RL signal** when reward-hacking metrics breach thresholds; revert to last good checkpoint.

## (d) Distillation studies

Teacher-data generation strategies (production-validated):

| Strategy | Pros | Cons |
|---|---|---|
| Sample teacher with temperature 0.7 on curated prompts | Simple; high quality | Single trajectory per prompt |
| Sample K=4 with temperature, take majority-vote | Higher quality | 4× cost |
| Sample with PRM-guided beam search | Best quality | Most expensive |
| Generate then verify-filter (>0 verified) | Filtered noise | May over-prune diverse strategies |

**Student-architecture choices:**

| Student size | % of teacher capability retained | Inference cost vs teacher |
|---|---|---|
| 7B (from 671B teacher) | ~70% | ~5% |
| 14B | ~85% | ~10% |
| 32B | ~92% | ~20% |
| 70B | ~95% | ~50% |

**Retention-vs-cost frontier:** the 32B distill is the production sweet spot; ~90% capability at 20% cost. R1-Distill-Qwen-32B is the canonical example.

**Distillation data size:** 800K-1M trajectories typical for full retention. Diminishing returns past 1M.

## (e) Training-to-eval pipeline

```python
@routine(schedule="*/30 * * * *")  # every 30 min during training
async def rl_training_health_check():
    metrics = await fetch_training_metrics()

    # Reward hacking detection
    if metrics.length_explosion or metrics.format_gaming or metrics.memorization:
        await pause_training()
        await alert_oncall(metrics)
        return

    # Eval drift
    eval_score = await run_eval_subset()
    if eval_score < last_known_good - 0.02:
        await rollback_to_last_good_checkpoint()
        await alert_oncall(metrics)
        return

    # Quality progress
    if metrics.steps_since_improvement > 1000:
        await flag_for_curriculum_review()
```

## (f) Compute economics

Per [275](275-agent-finetuning-2026.md):

| Stage | Compute (8x H100) | Cost |
|---|---|---:|
| SFT (3 epoch, 939K) | 24h | ~$2K |
| DPO (1 epoch, 270K) | 8h | ~$700 |
| RLVR (10K prompts, 5K steps) | 72h | ~$25K |
| Distillation 70B → 32B (800K) | 48h | ~$4K |
| **Tülu-3 reproduction total** | | **~$32K** |

DeepSeek-R1 reportedly $5.5M for the full 671B run. R1-Distill: ~$2K.

## (g) Production deployment

After training:

1. Quantize ([235-inference-compression-scaling](235-inference-compression-scaling.md)) to INT4 / INT8 / FP4.
2. Set up speculative decoding ([94-eagle3-spec-decoding](94-eagle3-spec-decoding.md)) with smaller draft model.
3. Deploy via vLLM / TensorRT-LLM / SGLang.
4. Monitor with [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md).
5. Eval with held-out set on rolling basis ([265-agent-evaluation-2026](265-agent-evaluation-2026.md)).
6. Drift detection per [267-agent-sre](267-agent-sre.md).

## One-line takeaway

**Production RLVR is four sub-disciplines (data curation + GRPO hyperparameter tuning + reward-hacking detection + distillation studies) — data quality is the highest-leverage lever, GRPO sweet-spot is K=8 with β=0.04 KL constraint, reward-hacking detection requires length-penalty + format-gate + decontaminated eval + cross-channel verifier dissent monitoring, and distillation 70B→32B retains ~90% capability at ~20% inference cost; total Tülu-3 reproduction ~$30K compute, R1-class ~$5M, R1-distill ~$2K — feasible at startup-scale for the 32B sweet spot.**
