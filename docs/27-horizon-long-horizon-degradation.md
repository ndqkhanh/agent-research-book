# 27 — HORIZON: Diagnosing Long-Horizon Agent Failure

**Definition.** HORIZON (arXiv:2604.11978) is a diagnostic benchmark and methodology for *why* LLM agents fail as tasks get longer — not just *that* they do. It introduces a trajectory-grounded LLM-as-Judge pipeline that attributes each failure to a specific mechanism, enabling systematic diagnosis rather than aggregate pass-rate reporting.

## Problem it solves

"Give the model more steps and it will handle more complex tasks" has been the dominant operating thesis behind agentic AI. HORIZON's empirical finding is a sharp challenge: performance degrades in systematic, *horizon-dependent* patterns even when the model technically has the capability for every individual step. Without understanding where long-horizon failures come from, teams mis-attribute them to "bad model" and ship increasingly futile prompt patches.

HORIZON replaces the practice of staring at pass/fail with process-level attribution: which step broke, what was happening earlier that set it up, and what class of failure this is. That turns agent engineering into a fixable problem rather than a mystery.

## Mechanism

Components of the methodology:

1. **Task suite across four domains.** ~3,100 trajectories collected on GPT-5 variants and Claude models over tasks chosen to force long chains.
2. **Failure taxonomy.** Categories include context forgetting, plan divergence, tool misuse, recovery failure, and verification gaps.
3. **Trajectory-grounded LLM-as-Judge.** A judge model reads the full trajectory and assigns each failure a category with evidence (quoted step numbers). Human agreement is strong (κ = 0.84 against human annotators).
4. **Horizon-length analysis.** Plot failure modes as a function of trajectory length to reveal *which* failures scale badly.

Headline finding from the paper (loosely): degradation is not uniform. Some failure modes (tool misuse) are roughly flat with horizon. Others (context-forget, plan-divergence) grow super-linearly. That decomposition tells you where harness effort pays off.

## Concrete pattern

A minimal "HORIZON dashboard" any production team can build:

```
For a sample of N runs, capture:
  - step count (horizon)
  - final success / partial success
  - automatic attribution label (if failure) from:
      {context-forget, plan-divergence, tool-misuse,
       recovery-failure, verification-gap, environment-error}
  - citation: failing step index + quoted evidence

Dashboard rollups:
  - failure rate × horizon bucket × attribution class
  - top-5 attribution classes last 7 days
  - example trajectories per class
```

The key discipline is *attribution with evidence*. A label without a quoted step is a guess; with a quoted step it becomes a ticket against the harness or prompt.

## Variants & related techniques

- **LLM-as-Judge & trajectory eval** ([21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md)) — the same machinery, here specialized to failure attribution.
- **Context compaction** ([08-context-compaction.md](08-context-compaction.md)) — a direct mitigation for the context-forget class.
- **Multi-session continuity** ([10-multi-session-continuity.md](10-multi-session-continuity.md)) — attacks "horizon" by reducing single-session length.
- **Plan-and-Solve / ReWOO / verifier loops** ([16-plan-and-solve.md](16-plan-and-solve.md), [17-rewoo.md](17-rewoo.md), [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md)) — mitigations for plan-divergence and verification-gap respectively.
- **Claw-Eval** ([38-claw-eval.md](38-claw-eval.md)) — evidence-channel-rich evaluation aligned with HORIZON's attribution principle.

## Failure modes & anti-patterns

- **Aggregate pass-rate thinking.** Reporting only success rate loses the signal HORIZON is designed to produce. Fix: decompose into classes.
- **Judge drift.** Over time the judge's labels shift. Fix: version the rubric, periodically recalibrate against humans.
- **One-trick fixes.** Ship a context-compaction tweak, overall numbers improve, but plan-divergence gets worse and isn't noticed because the aggregate is up. Fix: track class-level deltas.
- **Evidence-free attribution.** Labels become vibes. Fix: require quoted step evidence for every label.
- **Over-attribution to "model".** "The model is bad at X" — rarely fixable; the harness can often route around the weakness. Fix: always ask what harness change would hide this failure.

## When to use (and when not to)

**Use** HORIZON-style diagnostics when:

- You run enough agent traffic that aggregate pass rates are no longer informative.
- You suspect horizon-dependent failures (you probably have them).
- You have the engineering capacity to fix issues class-by-class.

**Don't** use when:

- You have very few runs — stick to human review until sample size grows.
- Your agent's failures are dominated by environment errors (network, API quota) — attribution is unhelpful there.

## References

- arXiv:2604.11978 — "HORIZON" (accessed April 2026). <https://arxiv.org/abs/2604.11978>
- LangSmith evaluation docs. <https://docs.smith.langchain.com/evaluation>
- "Evaluation and Benchmarking of LLM Agents: A Survey", arXiv:2507.21504. <https://arxiv.org/html/2507.21504v1>
- Chip Huyen, "AI Engineering" (O'Reilly) — evaluation chapter.
