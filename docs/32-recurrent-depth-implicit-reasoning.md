# 32 — Recurrent-Depth Transformers: Implicit Reasoning by Looping

**Definition.** "Loop, Think & Generalize" (Kohli et al., arXiv:2604.07822) investigates transformers that perform multi-hop reasoning not by generating extra tokens or using a longer chain-of-thought, but by **iterating the same transformer layers multiple times over the same tokens** — recurrent-depth computation. The paper shows that recurrence depth directly enables systematic generalization on implicit multi-hop reasoning tasks.

## Problem it solves

LLMs store enormous factual knowledge but famously struggle to compose it for implicit multi-hop reasoning without spelled-out chain-of-thought. Two common fixes — deeper networks or more output tokens — are both expensive and brittle (deeper networks are hard to train; long CoT hurts latency and leaks reasoning that should stay compressed).

Recurrent-depth transformers propose a third path: reuse layers *in depth* by iterating, and let the model choose how many iterations to spend. This is cheaper than scaling up (no new parameters) and more controllable than free-form CoT (iteration count is an explicit knob).

## Mechanism

Core architectural and training ideas:

1. **Recurrent application of the transformer stack.** Instead of L stacked layers applied once, the same K-layer block is applied N times. Parameters are shared across iterations; computation grows linearly in N.
2. **Iterations as "reasoning depth."** Each pass can refine the internal representation; empirically, more iterations enable deeper compositional reasoning (e.g., 5-hop → 10-hop generalization).
3. **Three-stage grokking.** Training exhibits memorization → in-distribution generalization → systematic generalization, with systematic generalization emerging only after enough training and enough recurrence.
4. **Depth extrapolation.** Models trained to reason to depth D can, under the right training regime, extrapolate to depths beyond D at inference by simply running more iterations.
5. **Overthinking limitation.** Too many iterations degrade predictions — the model begins to over-process, losing answers it had earlier.

## Concrete pattern

For agent designers, the concept maps onto runtime knobs:

```
Reasoning budget per step:
  - "easy" call: 1 iteration (fast, cheap).
  - "hard" call: up to N iterations (expensive, higher accuracy).

Budget controller:
  - confidence-based: iterate until confidence plateaus.
  - task-typed: simple classification → 1; multi-hop inference → N.
  - cost-capped: never exceed K iterations regardless of confidence.
```

The practical lesson for harness builders is that **reasoning depth is a tunable axis distinct from reasoning length**. A model with recurrent-depth support gives the harness a second dial. Even without architectural support, the pattern mirrors "best-of-N" or "self-consistency with variable N" — spend more compute on harder prompts.

## Variants & related techniques

- **Chain-of-Thought.** The explicit-token analogue; expensive in latency, human-auditable. Recurrence is cheaper but less auditable.
- **Self-consistency / best-of-N.** External "more compute per question" knob at the harness level.
- **Tree of Thoughts / LATS** ([15-tree-of-thoughts-lats.md](15-tree-of-thoughts-lats.md)) — branching rather than deepening.
- **Adaptive computation time (ACT)**, Universal Transformers, and Neural ODE-style continuous-depth models are intellectual ancestors.
- **ReBalance** ([51-rebalance-efficient-reasoning.md](51-rebalance-efficient-reasoning.md)) — confidence-guided steering to prevent overthinking, complements recurrent-depth's own overthinking pathology.

## Failure modes & anti-patterns

- **Overthinking degradation.** More iterations sometimes make things worse. Fix: dynamic halting (confidence plateau, learned halt predictor, compute cap).
- **Train-test depth mismatch.** Extrapolation to deeper reasoning is fragile if training never pushed the system near its limits. Fix: curriculum on recurrence depth.
- **Auditability loss.** Recurrence leaves no trace; the model "thought harder" but you cannot see how. Fix: pair with CoT or tool-call logs if inspectability matters.
- **Budget inflation.** Every call uses max iterations "just in case". Fix: per-call routing.
- **Generalization from toy tasks.** Implicit multi-hop benchmarks are narrow. Fix: evaluate on real multi-hop QA, coding tasks, agent trajectories.

## When to use (and when not to)

**Useful** when:

- You control the model (or your provider exposes iteration control).
- Latency is precious and CoT is too slow.
- Tasks have measurable "reasoning depth" that varies by prompt.

**Not useful** when:

- You need explainability — you'll have to add CoT anyway.
- The task is not reasoning-bound; extra iterations just waste compute.
- The model is off-the-shelf with no iteration knob — fall back to harness-level "best-of-N."

## References

- arXiv:2604.07822 — "Loop, Think, & Generalize: Implicit Reasoning in Recurrent-Depth Transformers". <https://arxiv.org/abs/2604.07822>
- Dehghani et al., "Universal Transformers" (2018). <https://arxiv.org/abs/1807.03819>
- Graves, "Adaptive Computation Time for Recurrent Neural Networks" (2016). <https://arxiv.org/abs/1603.08983>
- Wang et al., "Self-Consistency Improves Chain of Thought", arXiv:2203.11171.
- ReBalance (arXiv:2603.12372) — confidence-guided overthink control. <https://arxiv.org/abs/2603.12372>
