# 38 — Claw-Eval: Trajectory-Aware Evaluation of Autonomous Agents

**Definition.** Claw-Eval (Peking University, arXiv:2604.06132) is an evaluation suite for autonomous agents that grades **trajectories, not just final outputs**, using three independent evidence channels — execution traces, audit logs, and environment snapshots — against 2,159 fine-grained rubric items across 300 human-verified tasks. It reports Completion, Safety, and Robustness dimensions with Average Score, Pass@k, and Pass^k metrics. Headline finding: trajectory-opaque evaluation misses **44% of safety violations** and **13% of robustness failures**.

## Problem it solves

Most agent benchmarks check whether the agent's final output matches a reference (SWE-bench pass, WebArena success). This is dangerously lossy:

- An agent that solves the task via a *dangerous* path gets the same score as one that solved it safely.
- An agent that happens to succeed on a lucky retry gets the same score as one that was robust.
- An agent that reached the right answer with hallucinated citations gets the same score as one grounded in evidence.

Claw-Eval rebuilds the evaluation pipeline around process-level evidence. Three channels cross-check each other; 2,159 fine-grained rubric items scored per run; Pass^k captures *consistency* rather than any-success.

## Mechanism

### Three evidence channels

1. **Execution traces.** Complete ordered record of the agent's actions and observations — what tools were called, with which arguments, and what they returned.
2. **Audit logs.** External logs from the environment and downstream systems (server access logs, DB writes, API calls). These catch actions the trace might omit (side effects via tools the agent didn't report).
3. **Environment snapshots.** Pre- and post-run state comparison. Detects latent changes to files, DB rows, configs that execution traces alone miss.

Cross-channel disagreement is itself a signal: if the trace says "I didn't write the file" but the snapshot shows the file was written, a safety violation has occurred.

### Scoring

- **Completion.** Did the agent accomplish the task (per rubric).
- **Safety.** Did the agent avoid prohibited actions.
- **Robustness.** Does the agent succeed under perturbation (prompt rewrites, unexpected tool outputs).

Reported metrics:

- **Average Score** — aggregate over rubric items.
- **Pass@k** — probability that at least one of k runs succeeds (generosity toward retries).
- **Pass^k** — probability that *all* k runs succeed (consistency; stricter, often more actionable).

Error injection tests show peak performance is roughly stable but Pass^3 drops up to 24%, revealing that consistency — not capability — is the binding constraint for many agents.

## Concrete pattern

For teams running their own agent evals, the Claw-Eval discipline is:

```
For each task:
  1. Start from a reproducible environment snapshot.
  2. Run the agent; capture execution trace + env-side audit log.
  3. Take post-run snapshot.
  4. Score against rubric items via:
        - explicit tool-call expectations
        - forbidden-action list (safety)
        - snapshot diff assertions
        - LLM-judge on output quality (completion)
  5. Repeat k times with perturbations (robustness).
  6. Report Average, Pass@k, Pass^k per dimension.
```

## Variants & related techniques

- **LLM-as-Judge + trajectory eval** ([21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md)) — Claw-Eval is the benchmark-scale, cross-channel version.
- **HORIZON** ([27-horizon-long-horizon-degradation.md](27-horizon-long-horizon-degradation.md)) — complementary; HORIZON attributes failures; Claw-Eval scores consistency.
- **LinuxArena** ([26-linuxarena-production-agent-safety.md](26-linuxarena-production-agent-safety.md)) — production-safety-focused; Claw-Eval could be its scoring framework.
- **ClawBench** ([34-clawbench-live-web-tasks.md](34-clawbench-live-web-tasks.md)) — web-task benchmark; Claw-Eval the eval framework family.
- **AgentBench, SWE-bench Verified, τ-bench, GAIA** — earlier benchmarks with less evidence-channel discipline.

## Failure modes & anti-patterns

- **Audit-log gaps.** If the environment's logs are incomplete, the cross-check is weakened. Fix: instrument aggressively before running evals.
- **Snapshot cost.** Full environment snapshots can be expensive; teams economize and miss state changes. Fix: diff-based snapshots; focus on paths the agent touched.
- **Rubric item explosion.** Thousands of items per task becomes a maintenance burden. Fix: factor shared rubric primitives; auto-generate from safety policy.
- **Pass@k optimism.** Reporting only Pass@k lets consistency regressions hide. Always report Pass^k alongside.
- **Judge on trace calibration.** LLM judges evaluating traces drift. Fix: pin judge model versions; periodic human recalibration.
- **Running too few k.** k=1 yields Pass@1 only, no consistency signal. Budget for at least k=3.

## When to use (and when not to)

**Use** Claw-Eval-style evaluation when:

- You ship agents to production and need safety + consistency signals.
- You can afford k repeated runs per task.
- The environment supports cross-channel instrumentation.

**Don't** use it as-is when:

- Tasks are one-shot with no reproducible environment.
- You lack rubric items and cannot author them — hire domain experts or do not claim rigor.
- Your agent's variance is dominated by external flakiness (network, rate limits) — Pass^k becomes misleading.

## References

- arXiv:2604.06132 — "Claw-Eval: Toward Trustworthy Evaluation of Autonomous Agents". <https://arxiv.org/abs/2604.06132>
- ClawBench (arXiv:2604.08523). <https://arxiv.org/abs/2604.08523>
- HORIZON (arXiv:2604.11978). <https://arxiv.org/abs/2604.11978>
- "Evaluation and Benchmarking of LLM Agents: A Survey", arXiv:2507.21504. <https://arxiv.org/html/2507.21504v1>
- τ-bench. <https://arxiv.org/abs/2402.10200>
