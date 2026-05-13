# 101 — RAGEN: Multi-Turn RL for LLM Agents and the "Echo Trap"

**Paper.** Zihan Wang, Kangrui Wang, Qineng Wang, Pingyue Zhang, Linjie Li, Zhengyuan Yang, Kefan Yu, Minh Nhat Nguyen, Licheng Liu, Eli Chen, Yi Lu, Chuyi Shang, Jiateng Liu, Lyumanshan Ye, Xiangkun Hu, Mengyue Yang, Qingyun Wu, Manling Li, Heng Ji, Kevin Lin, Lijuan Wang, Jianfeng Gao — *RAGEN: Understanding Self-Evolution in LLM Agents via Multi-Turn Reinforcement Learning* — arXiv:2504.20073 — Northwestern, Microsoft Research, Stanford, et al. — 2025. Code: https://github.com/RAGEN-AI/RAGEN. Project page: https://ragen-ai.github.io/.

**One-line definition.** RAGEN trains LLM agents end-to-end with **multi-turn RL** in interactive, stochastic environments using **StarPO** (State-Thinking-Action-Reward Policy Optimization), and *names* the dominant failure mode — the **Echo Trap**, where agents collapse from diverse symbolic reasoning into deterministic templates that hurt long-term generalization.

## Why this paper matters

[100-deepseek-r1-rl-reasoning](100-deepseek-r1-rl-reasoning.md) showed RL works for **single-shot reasoning** (one input, one chain, one verifiable answer). But agents are **multi-turn**: they observe, act, observe again, and the reward arrives only at the trajectory's end. RAGEN is the cleanest open framework for the multi-turn case and the first to characterize *why* naive PPO/GRPO on agent trajectories collapses — the agent learns to repeat a templated success path and stops exploring.

For harness engineers, RAGEN provides the **diagnostic vocabulary** ("Echo Trap", "rollout-update interleaving") that reframes a class of agent regressions you may have already observed in production training pipelines.

## Problem it solves

1. **Sparse, delayed reward** in multi-turn interaction means the credit-assignment surface is brutal — most steps get the same outcome reward.
2. **Distribution shift between rollout and update.** Standard RL alternates: collect on-policy data, update, collect more. With long agent trajectories, the post-update policy is so different that the next batch is effectively off-policy → instability.
3. **Diversity collapse.** Trained agents drop reflective tokens ("let me reconsider...") and converge on a fixed action template — high training reward, low test generalization.
4. **No public benchmark for multi-turn agentic RL** — every team had been training in private; RAGEN ships environments, baselines, and evaluation protocols.

## Core idea in one paragraph

Frame each interactive episode as a sequence `(s_0, t_0, a_0, r_0, s_1, t_1, a_1, ...)` where `t_i` is the model's *thinking* (CoT) and `a_i` is its emitted action. **StarPO** treats the joint `(thinking, action)` as the policy output and applies group-relative RL across N parallel rollouts of the *same* initial state, with a critical twist: **interleaved rollout-update stages with a small inner step count** keep the train data nearly on-policy. The paper then ablates what saves and breaks training: **format reward**, **temperature**, **diversity bonuses**, and **trajectory length budgets** are the four levers that determine whether the agent ends up reflective and general — or stuck in an Echo Trap.

## Mechanism (step by step)

### (a) StarPO objective

For initial state s₀, sample N trajectories under current policy. For each trajectory τ, compute final reward r(τ). Normalize within group:

```text
A(τ) = (r(τ) - mean(r)) / std(r)
L_StarPO = - E_τ[ A(τ) · Σ_t log π_θ(t_i, a_i | s_{0..i}) ]
         + β · KL[ π_θ || π_ref ]
```

The advantage `A(τ)` is *trajectory-level* (not per-step), broadcast to every (think, act) emission. PPO clipping is applied at the *trajectory* level; group baseline replaces the critic, à la GRPO.

### (b) Rollout–update interleaving

Each "epoch" is:

```
for outer in 1..K_outer:
    rollouts = collect(N=8 trajectories per prompt, batch_size=64 prompts)
    for inner in 1..K_inner:    # small! K_inner ∈ {1,2,4}
        update_policy(rollouts)
```

The paper shows `K_inner = 1` is the safest — one gradient step per fresh rollout batch. Higher inner steps reuse stale data and accelerate Echo Trap.

### (c) The Echo Trap, characterized

Symptoms emerge around 200–500 update steps:

- Reflection token frequency ("wait", "let me check", "actually") drops to near zero.
- Trajectory entropy collapses; KL to ref diverges in one direction.
- Test-set generalization peaks then declines while training reward rises.

The paper plots **diversity-reward curves** (Fig. 4) showing the inflection: agents that look optimal mid-training are actually overfitting to template paths.

### (d) The four levers (Section 4 ablations)

| Lever | Echo-Trap risk | Best practice |
|-------|----------------|---------------|
| **Format reward** (penalty for non-template output) | High when too strict | Lenient regex, only require valid action tag |
| **Sampling temperature** | High when low T | Train at T≥0.7; cool to T=0 only at eval |
| **Diversity bonus** (e.g. KL to uniform over actions) | Low when used | Small bonus (β_div≈0.01) recovers reflection tokens |
| **Trajectory length budget** | High when fixed-short | Allow variable horizons up to env max; truncated trajectories teach termination, not skill |

### (e) Environments shipped

RAGEN includes three: **Sokoban** (planning), **FrozenLake-Stochastic** (noisy outcomes), and a **two-armed bandit text task** (exploration). Each has a verifier that emits trajectory-level reward. The paper deliberately avoids real-web/code environments to isolate the RL dynamics from tool noise.

## Empirical results

Headline numbers on Sokoban (from the paper, 4-room 7-box variant):

| Method | Train SR | Test SR (held-out levels) |
|--------|---------:|--------------------------:|
| Base (Qwen-2.5-7B-Instruct) | 11% | 9% |
| SFT on expert traces | 47% | 22% |
| StarPO (no diversity bonus) | 84% | 31% |
| StarPO + format-lenient + diversity | **78%** | **62%** |

Note the **train-vs-test inversion**: the highest training reward run is *not* the best test run. The Echo Trap rewards the model on training distribution while degrading transfer.

On FrozenLake-Stochastic, StarPO+diversity recovers the **right behavior in noisy outcomes**: it learns to retry rather than re-plan from scratch — a behavior absent from SFT and the no-diversity baseline.

## Variants and ablations

- **K_inner sweep:** K=1 stable; K=4 collapses by 800 steps; K=8 unusable.
- **Critic vs no-critic (PPO-classical vs StarPO):** comparable peak reward; StarPO is ~30% cheaper per step and easier to tune (no value-loss balancing).
- **Distill-then-RL:** SFT on R1-distilled CoT → StarPO outperforms either alone by ~10 SR points on held-out tasks.
- **Frozen base (LoRA-only RL):** plateaus 8–12 points below full-parameter; useful when compute is tight.

## Failure modes and limitations

1. **Toy environments.** Sokoban and FrozenLake are not real-web/real-code. The paper is honest: RAGEN proves *RL dynamics*, not *production performance*. [102-artist-agentic-rl-tools](102-artist-agentic-rl-tools.md) and [103-agent-lightning](103-agent-lightning.md) push toward realistic domains.
2. **Scale ceiling.** Experiments cap at 7B-class models. Whether StarPO scales to 70B+ as elegantly as GRPO does for reasoning is *open*.
3. **Reward shaping fragility.** A small mis-spec on the format regex can replace one Echo Trap with another; iterative manual tuning required.
4. **No multi-agent setting.** RAGEN is single-agent; multi-agent RL (cf. ChatDev-style — [92-chatdev](92-chatdev.md)) is left for follow-up.
5. **Sample efficiency.** Multi-turn RL still costs 10–100× more than SFT for comparable end-task performance; don't reach for it unless the SFT path is exhausted.

## When to use, when not

**Use RAGEN-style multi-turn RL when** you have a programmable environment with a clean trajectory-level reward, your agent can be queried tens of thousands of times cheaply (no API rate limits), and you've already exhausted SFT-on-expert-traces gains. Best fit: in-house simulators (web automation, game environments, IDE agents with sandboxed tests).

**Don't reach for it** when reward signal is non-stationary (chat with users), when each rollout costs >$0.10 (compute economics break), or when the production deployment can pivot quickly via SFT updates — the Echo Trap risk often outweighs the marginal gain.

## Implications for harness engineering

- **The harness *generates* training data.** Once your harness is good (verifiers, sandboxes, traces), it can be repointed at RL training. Harness-as-training-environment is a real architecture pattern (cf. [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md)).
- **Format-reward / format-hooks symmetry.** The same regex you use in [05-hooks](05-hooks.md) to validate tool calls can serve as the format reward in RL. Reuse pays off.
- **Trajectory observability is the prerequisite.** Without [24-observability-tracing](24-observability-tracing.md), you cannot diagnose an Echo Trap; train-reward-only dashboards lie.
- **Diversity is a runtime concern too.** StarPO's diversity bonus has a runtime sibling — sampling temperature scheduling and best-of-N at deploy. Don't only tune at training.

## Connections to other work in this corpus

- **[100-deepseek-r1-rl-reasoning](100-deepseek-r1-rl-reasoning.md):** RAGEN is the multi-turn cousin; GRPO → StarPO is the principal change.
- **[14-reflexion](14-reflexion.md):** Reflexion adds reflection *at deploy time* without weight updates; RAGEN trains *the same behavior* into weights, but the Echo Trap warns reflection can be *un*trained.
- **[31-glm-5-agentic-engineering](31-glm-5-agentic-engineering.md):** GLM-5 trains agents on long-horizon tool trajectories at scale; RAGEN provides the diagnostic vocabulary GLM-5's training team would care about.
- **[81-reasoningbank](81-reasoningbank.md):** ReasoningBank achieves multi-turn improvement *without weight updates* via memory; RAGEN does it *with* weight updates. Complements, not substitutes.

## Key takeaways

1. **Multi-turn RL ≠ multi-shot reasoning RL.** Trajectory-level reward and rollout-update drift change the dynamics fundamentally.
2. **Echo Trap is the dominant failure mode** — diversity collapse from over-fit to template paths.
3. **K_inner=1 (one update per fresh rollout batch) is safest.**
4. **Format reward + temperature + diversity bonus + variable trajectory length** are the four levers that control trap risk.
5. **Train-reward-only dashboards lie** — track diversity, reflection-token frequency, and held-out SR.

## References

- Wang et al. (2025). *RAGEN: Understanding Self-Evolution in LLM Agents via Multi-Turn RL.* arXiv:2504.20073. https://arxiv.org/abs/2504.20073
- Project: https://ragen-ai.github.io/
- Code: https://github.com/RAGEN-AI/RAGEN
- Successor work: *RAGEN-2: Reasoning Collapse in Agentic RL* — extends the Echo Trap analysis. https://ragen-ai.github.io/
- Companion survey: *Training Recipes for Agentic RL in LLMs* — https://github.com/blacksnail789521/Agentic-RL-Training-Recipes
