# 31 — GLM-5: Training Models for Agentic Engineering

**Definition.** GLM-5 (arXiv:2602.15763) is a foundation model explicitly trained to transition from "vibe coding" — casual, single-shot code generation — to **agentic engineering**: disciplined, long-horizon, tool-integrated software development. Its technical contributions are asynchronous RL infrastructure that decouples generation from training and novel agent-RL algorithms tuned for long-horizon trajectories.

## Problem it solves

Models trained primarily on static code corpora get good at completing snippets but mediocre at executing long agent workflows — debugging cascades, multi-file refactors, tool chains with tens of calls. The failure isn't reasoning capacity; it's that the training signal rewarded "produce plausible code" rather than "complete an engineering task." RL on agent trajectories addresses this, but naive synchronous RL (generate rollouts → train → repeat) is wildly inefficient at long horizons where a single rollout can take minutes.

GLM-5's core architectural contribution is asynchronous RL: rollout workers generate trajectories independently of trainer workers, so neither waits on the other. Combined with DSA (a cost-reduction mechanism) and agent-RL algorithms designed for long trajectories, the approach produces end-to-end coding performance that, per the paper, surpasses prior baselines on major open benchmarks.

## Mechanism

Key ingredients:

1. **Asynchronous RL infrastructure.** Decouples trajectory generation from gradient updates. Rollout machines run the current policy and produce trajectories; trainer machines consume them at their own pace. Major throughput gains when rollouts are slow (long horizons, external tool calls).
2. **Novel async agent-RL algorithms.** Careful handling of off-policy data produced while weights update mid-rollout — a correctness issue naive async setups get wrong.
3. **DSA.** A cost-reduction technique (described in the paper) that preserves long-context fidelity while cutting training and inference cost.
4. **Agentic coding focus.** Evaluation emphasizes end-to-end software engineering tasks where the model uses tools, runs tests, and iterates — not just HumanEval-style snippet completion.
5. **Continuous post-training loop.** The gap between frontier models on agentic tasks is increasingly decided at post-training, not pre-training; GLM-5 operationalizes that.

## Concrete pattern

For teams training or post-training their own agent models:

```
Pre-training:   rich code + tool-trace corpus.
SFT:            high-quality human agent trajectories.
Agent RL:       async, with correctness-preserving off-policy corrections.
Reward:         mix of test-pass (objective) and LLM-judge (trajectory quality).
Eval:           SWE-bench Verified, LinuxArena, ClawBench, internal agent eval.
Cadence:        continuous post-training; treat agent capability as a product metric.
```

Even teams not training their own models should adopt the *eval cadence*: benchmarks of end-to-end agent capability are the frontier that matters.

## Variants & related techniques

- **Training vs harness.** GLM-5 is the training-side answer to "how do we make the model better at agent work". Most of this folder (01–25) is the harness-side answer. Both compound; neither substitutes for the other.
- **Voyager** ([19-voyager-skill-libraries.md](19-voyager-skill-libraries.md)) — shares the "accumulate capability across episodes" goal but accumulates *skills*, not weights.
- **Adaptation of Agentic AI survey** ([47-adaptation-of-agentic-ai-survey.md](47-adaptation-of-agentic-ai-survey.md)) — positions GLM-5-style post-training as one of four adaptation paradigms.
- **Reflexion** ([14-reflexion.md](14-reflexion.md)) — a zero-gradient analogue that can be used alongside full RL.
- **Hyperagents / DGM-H** ([45-hyperagents-self-modification.md](45-hyperagents-self-modification.md)) — self-modifying systems push beyond gradient-based post-training.

## Failure modes & anti-patterns

- **Async RL correctness bugs.** Off-policy data is silently biased; training appears stable but downstream quality degrades. Fix: measured importance-sampling corrections, strict eval gating.
- **Reward hacking.** Test-pass reward is gamed by models that write tests that pass themselves. Fix: held-out test suites, LLM-judge review of trajectories.
- **Benchmark overfitting.** Good on SWE-bench, bad on real codebases. Fix: evaluate on private, rotating real-world tasks.
- **Long-horizon ≠ "more steps."** Rewards must reflect task success, not step count; otherwise models learn to pad.
- **"Our model + anyone's harness" is not enough.** A strong agent model paired with a weak harness still ships mediocre agents. Invest in both.
- **Ignoring cost.** Async RL + long-horizon rollouts is expensive; without cost discipline, each model generation blows the budget.

## When to use (and when not to)

**Apply** GLM-5's approach when:

- You own a model and the infrastructure budget for RL.
- Your users' tasks are agentic (tool use, long horizons), not snippet completion.
- You can curate strong trajectory datasets and objective rewards.

**Do not** when:

- You don't own the model — use a strong generalist + strong harness + skills.
- Your distribution is dominated by short, single-shot code — SFT on snippets is cheaper.
- You cannot define rewards; RL's main failure mode is a bad reward.

## References

- arXiv:2602.15763 — "GLM-5: from Vibe Coding to Agentic Engineering". <https://arxiv.org/abs/2602.15763>
- "Adaptation of Agentic AI: A Survey of Post-Training, Memory, and Skills" (arXiv:2512.16301). <https://arxiv.org/abs/2512.16301>
- SWE-bench Verified benchmark. <https://www.swebench.com/>
- ClawBench (arXiv:2604.08523) — web-task agent benchmark. <https://arxiv.org/abs/2604.08523>
- LinuxArena (arXiv:2604.15384). <https://arxiv.org/abs/2604.15384>
