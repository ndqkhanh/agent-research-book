# 113 — Test-Time Compute Scaling for Agents: Parallel, Sequential, Verifiers, Diversity

**Papers.**
- *Scaling Test-time Compute for LLM Agents* — arXiv:2506.12928 — June 2025.
- *The Art of Scaling Test-Time Compute for Large Language Models* — arXiv:2512.02008 — December 2025 (large-scale empirical study).
- *Scaling Test-Time Compute for Agentic Coding* — arXiv:2604.16529 — April 2026.
- Foundational: Snell et al., *Scaling LLM Test-Time Compute Optimally Can Be More Effective than Scaling Parameters*, arXiv:2408.03314 (ICLR 2025); s1 (*Simple test-time scaling*, EMNLP 2025).

**One-line definition.** **Test-time compute (TTC) scaling** for agents is the family of techniques — **parallel sampling**, **sequential revision**, **verifier-guided selection**, and **diversification** — that improve agent task success by spending more inference compute *per task* rather than scaling model parameters. The 2026 frontier is figuring out **which lever to pull when**.

## Why this paper matters

Snell et al. (2024) showed that on math, optimal TTC scaling can outperform parameter scaling. The 2025–2026 work extends this to **agents** — multi-turn, tool-using, environment-coupled — where parallel rollouts and sequential refinements interact with environment side effects. The Dec-2025 *Art of Scaling TTC* study is the largest empirical sweep yet (>30B tokens, 8 LLMs from 7B to 235B, four reasoning datasets), and *Scaling TTC for Agentic Coding* (Apr 2026) finds the dominant challenge is **representing prior experience** for selection — an explicit bridge to the memory work in [81-reasoningbank](81-reasoningbank.md), [77-meta-tts-agentic-coding](77-meta-tts-agentic-coding.md), and [104-mem0-production-memory](104-mem0-production-memory.md).

## Problem it solves

1. **Parameter scaling is expensive and slow.** TTC is an alternative axis you can dial per-task at deploy time.
2. **Single-shot agent runs leave performance on the table.** Many tasks succeed at pass@k but fail at pass@1.
3. **Verifier-gated selection** can convert pass@k to a useful pass@1 if the verifier is reliable.
4. **Sequential revision** ([18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md), [14-reflexion](14-reflexion.md)) is a different lever from parallel sampling — they don't trade off the same way.
5. **No single TTS strategy universally dominates** — the *Art of Scaling TTC* sweep makes this empirical.

## Core idea in one paragraph

For a fixed task with budget B (tokens, dollars, seconds), agents can spend that budget along (at least) four axes: **parallel sampling** (run N independent rollouts, select via majority/verifier), **sequential revision** (run one rollout, then refine in-place), **verifier guidance** (rank/prune mid-trace using a PRM or trajectory judge), and **diversification** (deliberately spread initial conditions to cover the solution space). The 2025–2026 papers benchmark these axes against one another and find: *parallel wins on hard problems and easy verifiability; sequential wins where iteration genuinely improves quality; verifier guidance is multiplicative on top of either; diversification matters most when the policy is high-entropy and the verifier is reliable*.

## Mechanism (step by step)

### (a) Parallel sampling

```text
sample N=k completions / trajectories
score each (verifier, judge, majority vote)
return best
```

Best-known instance: **Best-of-N** with a process reward model ([97-qwen-prm](97-qwen-prm.md)). Cost ≈ k×; performance grows roughly logarithmically in k for hard tasks, plateaus on easy ones.

### (b) Sequential revision

```text
draft completion
loop until budget or convergence:
    critique
    revise
return latest
```

Reflexion ([14-reflexion](14-reflexion.md)), Self-Refine ([18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md)), and ReAct-with-replan are all sequential schemes. Cost grows with chain length; benefits depend on whether each revision genuinely changes the trajectory.

### (c) Verifier guidance

Two flavors:

- **Selection verifier**: scores final outputs; pick best (works with parallel sampling).
- **Process verifier**: scores intermediate steps; prunes / re-routes during the trace (works with parallel + sequential, à la Tree-of-Thoughts / LATS, [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md)).

### (d) Diversification

The 2026 work (per *Scaling TTC for LLM Agents*) finds that **diversifying initial conditions** — varying temperature, prompt wording, or even system instructions — yields more reliable parallel-sampling gains than just sampling N times at the same temperature. Echoes of the [101-ragen](101-ragen.md) "Echo Trap" finding: diversity is the antidote to convergent collapse.

### (e) Adaptive budgets

The *Art of Scaling TTC* (arXiv:2512.02008) finds **no single strategy universally dominates**: optimal TTS performance scales monotonically with compute budget but the *strategy* should switch based on problem difficulty and trace-quality patterns. Adaptive routers that pick parallel-vs-sequential per task outperform fixed strategies.

### (f) The agentic-coding wrinkle

Per *Scaling TTC for Agentic Coding* (arXiv:2604.16529), the main bottleneck is **representing prior experience** for effective selection — single-shot trace ranking is too noisy on long agent trajectories. Memory systems ([81-reasoningbank](81-reasoningbank.md), [104-mem0-production-memory](104-mem0-production-memory.md)) become a TTC lever: stored distillates make selection cheaper and more accurate.

## Empirical results

Headline (synthesized across the cited papers):

- **Math reasoning (Snell et al. 2024)**: TTC-optimal can match a 14× parameter-scaled model.
- **Agentic web tasks (RAGEN-style, [101-ragen](101-ragen.md))**: parallel sampling at k=5 with verifier improves SR by 5–10 points over k=1.
- **Agentic coding (arXiv:2604.16529)**: memory-curated selection dominates raw best-of-N; ReasoningBank-style stores ([81-reasoningbank](81-reasoningbank.md)) lift TTC efficacy by 3–8 points on SWE-Bench-class tasks.
- **Cross-strategy (Art of Scaling TTC)**: no universal winner; reasoning models exhibit *distinct* TTS patterns by problem difficulty.

## Variants and ablations

- **Best-of-N with majority vote vs verifier**: verifier wins when reliable; majority wins when verifier is noisy.
- **Temperature sweep**: parallel sampling at fixed temperature plateaus; sweeping temperature across the N samples adds 2–5 points.
- **Sequential refinement count**: monotonic returns up to ~3 refinements; >3 often hurts (the model talks itself out of correct answers).
- **Memory-augmented selection**: small memory of past distillates improves selection quality in repeated-task settings.

## Failure modes and limitations

1. **Verifier reliability is the cap.** A noisy verifier turns parallel sampling into expensive randomness ([97-qwen-prm](97-qwen-prm.md) — "metric inversion").
2. **Sequential revision can degrade** if each revision compounds errors — common when the critique itself is wrong.
3. **Side-effecting environments break parallel sampling.** Two parallel browser agents both clicking "buy" is bad. Parallelism requires reversible or sandboxed environments.
4. **Cost can outrun benefit.** Doubling compute for 2 points of accuracy may not be worth it; ROI must be measured per task class.
5. **Latency budgets** rule out high-N parallel for interactive agents; sequential is friendlier to user-facing latency.

## When to use, when not

**Use parallel TTC** when verifiers are reliable, environments are reversible/sandboxed, and the user can wait. Best for batch agentic-coding (run k tries, pick the one with passing tests) and BrowseComp-style research ([112-browsecomp](112-browsecomp.md)).

**Use sequential TTC** when the task is genuinely iterative (writing, refinement, debugging) and the critique step adds real signal.

**Use verifier guidance** whenever you have a non-trivial PRM or judge; it amplifies whichever axis you're already on.

**Don't reach for TTC** when the model is small enough that parameter scaling buys more per dollar (often true for 7B-class), when the environment is irreversible without sandboxing, or when the task is easy enough that single-shot is already at ceiling.

## Implications for harness engineering

- **TTC is a runtime-routing concern.** The harness can detect task difficulty and choose the strategy ([88-confidence-driven-router](88-confidence-driven-router.md)).
- **Sandbox or simulation is a TTC enabler.** Without reversibility, parallelism is unsafe.
- **Memory systems are TTC infrastructure** in the agentic-coding case ([81-reasoningbank](81-reasoningbank.md), [104-mem0-production-memory](104-mem0-production-memory.md)).
- **Cost dashboards must split inference into baseline vs TTC** so the ROI is legible per task class.
- **Verifiers are TTC's ceiling** — invest in PRMs and judges before more samples.

## Connections to other work in this corpus

- **[97-qwen-prm](97-qwen-prm.md):** verifier quality, the cap on TTC efficacy.
- **[81-reasoningbank](81-reasoningbank.md):** Memory-aware Test-Time Scaling (MaTTS) — TTC + memory in one design.
- **[77-meta-tts-agentic-coding](77-meta-tts-agentic-coding.md):** Meta TTS — sister concept for agentic coding.
- **[14-reflexion](14-reflexion.md), [18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md):** sequential TTC instantiations.
- **[15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md):** structured search is parallel-+-process-verifier TTC.
- **[51-rebalance-efficient-reasoning](51-rebalance-efficient-reasoning.md):** the inverse problem — *constraining* TTC when the model overthinks.
- **[88-confidence-driven-router](88-confidence-driven-router.md):** an adaptive TTC router.

## Key takeaways

1. **TTC is a four-dimensional lever**: parallel sampling, sequential revision, verifier guidance, diversification.
2. **No universal winner** — pick the strategy by task class and verifier quality.
3. **Verifier reliability caps everything** — invest there first.
4. **Memory systems are TTC infrastructure** for repeated-task settings.
5. **Sandboxing or reversibility is the prerequisite for parallel** in side-effecting environments.

## References

- Snell, C. et al. (2024). *Scaling LLM Test-Time Compute Optimally Can Be More Effective than Scaling Parameters.* arXiv:2408.03314 (ICLR 2025). https://arxiv.org/abs/2408.03314
- *Scaling Test-time Compute for LLM Agents* (June 2025). arXiv:2506.12928. https://arxiv.org/abs/2506.12928
- *The Art of Scaling Test-Time Compute for Large Language Models* (Dec 2025). arXiv:2512.02008. https://arxiv.org/abs/2512.02008
- *Scaling Test-Time Compute for Agentic Coding* (April 2026). arXiv:2604.16529. https://arxiv.org/abs/2604.16529
- Muennighoff et al., *s1: Simple test-time scaling*, EMNLP 2025.
- Survey: *What, How, Where, and How Well? A Survey on Test-Time Scaling in Large Language Models* — https://testtimescaling.github.io/
