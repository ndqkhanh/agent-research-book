# 100 — DeepSeek-R1: Incentivizing Reasoning via Pure Reinforcement Learning

**Paper.** DeepSeek-AI — *DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning* — arXiv:2501.12948 (22 Jan 2025); peer-reviewed version: *Nature* s41586-025-09422-z (2025). Open weights and distilled checkpoints released under MIT-style terms on Hugging Face. Companion model: **DeepSeek-R1-Zero** (RL-only, no SFT cold start).

**One-line definition.** DeepSeek-R1 demonstrates that **pure outcome-reward RL** (no human reasoning traces) on a strong base model can elicit emergent self-reflection, verification, and long chains-of-thought — the recipe that defined the **reasoning-model era** for open weights.

## Why this paper matters

Before R1, the dominant assumption in industry was that o1-style reasoning required **massive curated chain-of-thought SFT**. R1 falsifies that: with **GRPO** (Group Relative Policy Optimization) and a verifier-only reward (correct answer / passes unit tests), reasoning patterns *emerge* — including the famous **"aha moment"** where R1-Zero spontaneously rewrites its own reasoning mid-trace. The harness consequence: every agent loop downstream of a reasoning model now inherits *latent* deliberation that the harness must budget, observe, and sometimes interrupt.

For agent harness engineers this paper is the **upstream supply** — it is the model assumption file that all of [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md), and [51-rebalance-efficient-reasoning](51-rebalance-efficient-reasoning.md) now reason against.

## Problem it solves

1. **Cost of CoT data.** Hand-curated reasoning traces don't scale; OpenAI's o1 corpus is closed.
2. **PPO instability for reasoning.** Per-token value baselines are noisy on long traces; GRPO replaces the value model with an in-group baseline (mean reward across N samples for the same prompt).
3. **Reward hacking.** Pure RL on reward-model scores teaches the model to game the RM. R1 uses **rule-based verifiable rewards** (math: gold-answer match; code: unit tests) that cannot be hacked by surface form.
4. **Cold-start linguistic chaos.** R1-Zero produces correct-but-unreadable CoT (mixed languages, no punctuation). R1 adds a small **cold-start SFT** stage on a few thousand human-curated CoT examples to anchor format before the RL phase.

## Core idea in one paragraph

Start from `DeepSeek-V3-Base`. Train **R1-Zero** with GRPO directly: for each prompt, sample G=16 completions, score with rule-based verifiers, normalize rewards within the group, update policy with KL anchor to a frozen reference. Watch reasoning length grow from a few hundred to thousands of tokens; watch self-reflection emerge unprompted. Then for **R1**: SFT on a small "cold-start" corpus of well-formatted CoT → run the same GRPO RL on reasoning tasks → SFT-distill onto general-purpose data → final RL with both verifiable and human-preference rewards. **Distillation** to Qwen2.5/Llama-3 yields R1-Distill checkpoints (1.5B–70B) that retain a large fraction of the reasoning gains without the RL training cost.

## Mechanism (step by step)

### (a) GRPO objective

For prompt q, sample group `{o_1,...,o_G}` from policy `π_θ_old`, score with verifier r_i:

```text
A_i = (r_i - mean(r)) / std(r)        # group-relative advantage
L = E_q[ (1/G) Σ_i min(ratio_i · A_i, clip(ratio_i, 1-ε, 1+ε) · A_i) ]
   - β · D_KL(π_θ || π_ref)
```

No critic network; the group mean *is* the value baseline. Compute saved per step ≈ 30–50% vs PPO; variance higher per sample but stabilized by larger G.

### (b) Verifier reward design

- **Math:** extract final boxed answer; string/numeric match against gold. Format reward: `<think>...</think><answer>...</answer>` template — penalize malformed.
- **Code:** run hidden unit tests in a sandbox; compile-fail = 0, test-pass-rate = reward.
- **No process reward model.** The paper deliberately avoids PRMs (cf. [97-qwen-prm](97-qwen-prm.md)) — they tried it and found exploits where the model games the PRM.

### (c) The four-stage R1 pipeline

1. **Cold-start SFT** — ~10⁴ curated CoT examples in clean format.
2. **Reasoning-RL** — GRPO on math + code + logic with verifier rewards; this stage produces most of the capability.
3. **Rejection-sampling SFT** — sample many completions, keep correct ones, retrain on the union with general SFT data → restores helpfulness/safety.
4. **All-task RL** — second RL pass mixing verifiable rewards with a learned helpfulness/safety RM.

### (d) Distillation

Generate ~800k R1 completions on diverse prompts; SFT into smaller bases (Qwen2.5-1.5B/7B/14B/32B, Llama-3.1-8B/70B). The 7B distill *beats* GPT-4o on AIME 2024 (55.5% vs 9.3% reported in the paper). **Distillation > direct RL on small bases** — small models lack the latent capacity for RL to surface.

## Empirical results

Headline numbers (from the paper, single-shot pass@1 unless noted):

| Benchmark | R1 | R1-Distill-Qwen-32B | OpenAI o1-1217 |
|-----------|----|---------------------|----------------|
| AIME 2024 | 79.8% | 72.6% | 79.2% |
| MATH-500 | 97.3% | 94.3% | 96.4% |
| Codeforces (rating) | 2029 | 1691 | 2061 |
| GPQA Diamond | 71.5% | 62.1% | 75.7% |
| LiveCodeBench | 65.9% | 57.2% | 63.4% |

R1-Zero alone (no SFT) scores **71.0%** on AIME 2024 and **86.7%** on MATH-500 — i.e. *most* of R1's reasoning is delivered by RL, not by SFT.

## Variants and ablations

- **GRPO group size G:** sweet spot at G=16 in the paper; G=4 is unstable, G=64 saturates.
- **KL coefficient β:** higher β slows learning but prevents mode collapse; the paper uses ~0.001–0.04 across stages.
- **No cold start (= R1-Zero):** competitive accuracy, but mixed-language and disorganized CoT — unusable as a chat model.
- **Direct RL on a small base** (Qwen-32B): worse than distilled-from-R1 32B by ~10 points on AIME — supporting the distillation-first claim.

## Failure modes and limitations

1. **Multilingual leakage.** R1-Zero often code-switches mid-CoT (Chinese, English, math symbols). Cold-start SFT only partially fixes this.
2. **Long-form drift.** Reasoning traces inflate to 5–20k tokens on hard math; for harnesses without [08-context-compaction](08-context-compaction.md) this can blow context windows of *downstream* tools.
3. **Reward-spec brittleness.** The boxed-answer regex sometimes misses correct answers in non-canonical form, depressing reward; the paper mitigates with format reward shaping.
4. **No general-domain RL signal.** Reasoning gains transfer modestly to open-ended tasks (creative writing, agentic web tasks); the all-task RL stage is small and uses learned RMs that introduce their own biases.
5. **Distill ≠ free.** Distill-7B/14B underperform R1 on multi-step proofs; they retain *style* better than they retain *capacity*.

## When to use, when not

**Use R1 (or distills) as the harness model when** the task is verifier-rich (code, math, structured deliverables), the harness can budget thousands of "thinking" tokens, and you have observability over `<think>` blocks for moderation. The 32B/70B distills are the practical sweet spot for self-hosted agent harnesses today.

**Avoid R1 when** the agent needs cheap, fast tool-calling on routine tasks — reasoning models systematically overthink simple queries (the [51-rebalance-efficient-reasoning](51-rebalance-efficient-reasoning.md) problem). Use a smaller chat model with [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) supervision instead, or apply ReBalance-style confidence gating.

## Implications for harness engineering

- **Token budgets become reasoning budgets.** [01-agent-loop-architecture](01-agent-loop-architecture.md) step budgets must now also cap *internal* think-tokens per step.
- **Verifier-first harness design.** R1 validates the [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) thesis at training time — verifiable rewards are what the model was *built* with; agent loops should inherit that contract.
- **The `<think>` channel is observable.** Tracing harnesses ([24-observability-tracing](24-observability-tracing.md)) should split think-tokens from user-facing tokens; cost dashboards should show both.
- **Distillation as deployment.** Production agent fleets can't afford R1-671B per call. The 14B/32B distills + good harness are the realistic baseline; this is how OSS agent stacks ([108-openhands-codeact](108-openhands-codeact.md), [109-smolagents](109-smolagents.md)) ship reasoning today.
- **Reward hacking moves to the harness.** Without PRMs, the harness becomes the de facto step-checker — Plan Mode, Hooks, and HITL are reasserted as primary safety levers ([03-plan-mode](03-plan-mode.md), [05-hooks](05-hooks.md), [23-human-in-the-loop](23-human-in-the-loop.md)).

## Connections to other work in this corpus

- **[31-glm-5-agentic-engineering](31-glm-5-agentic-engineering.md):** parallel async-RL recipe specialized for *agent trajectories* (tool-calling) rather than pure reasoning chains.
- **[97-qwen-prm](97-qwen-prm.md):** R1's deliberate rejection of PRMs; Qwen-PRM is the alternative path.
- **[101-ragen](101-ragen.md):** multi-turn RL for *agents* (not single-shot reasoning) — extends the R1 recipe to interactive environments.
- **[102-artist-agentic-rl-tools](102-artist-agentic-rl-tools.md):** brings tool calling into the GRPO loop.
- **[51-rebalance-efficient-reasoning](51-rebalance-efficient-reasoning.md):** runtime control of the over-thinking R1 induces.

## Key takeaways

1. **Pure RL on a strong base + verifiable reward = reasoning** — no need for closed CoT data.
2. **GRPO replaces PPO's value model** with a group-mean baseline; cheaper and stable enough at G≈16.
3. **Cold-start SFT is a formatter, not a teacher** — most of the capability comes from the RL stage.
4. **Distillation, not direct RL, is how small models get reasoning** in this paradigm.
5. **The harness must now budget, observe, and sometimes throttle** the reasoning the model brings for free.

## References

- DeepSeek-AI (2025). *DeepSeek-R1.* arXiv:2501.12948. https://arxiv.org/abs/2501.12948
- DeepSeek-AI (2025). *DeepSeek-R1 incentivizes reasoning in LLMs through reinforcement learning.* Nature. https://www.nature.com/articles/s41586-025-09422-z
- Hugging Face model card: https://huggingface.co/deepseek-ai/DeepSeek-R1
- GRPO (paper origin): Shao et al., *DeepSeekMath* — https://arxiv.org/abs/2402.03300
- Survey context: *RL Foundations for Deep Research Systems* (arXiv:2509.06733).
