# 77 — Scaling Test-Time Compute for Agentic Coding (Meta TTS)

**Paper.** Joongwon (Daniel) Kim, Winnie Yang, Kelvin Niu, Hongming Zhang, Yun Zhu, Eryk Helenowski, Ruan Silva, Zhengxing Chen, Srini Iyer, Manzil Zaheer, Daniel Fried, Hannaneh Hajishirzi, Sanjeev Arora, Gabriel Synnaeve, Ruslan Salakhutdinov, Anirudh Goyal — *Scaling Test-Time Compute for Agentic Coding* — arXiv:2604.16529 — Meta Superintelligence Labs / UW / NYU / Google DeepMind / CMU / Princeton — April 2026.

**One-line definition.** Meta TTS reframes test-time scaling for long-horizon coding agents as a **representation problem**: instead of comparing or refining raw rollout trajectories, the agent compresses each rollout into a compact structured summary, then uses **Recursive Tournament Voting (RTV)** for parallel selection and **Parallel-Distill-Refine (PDR)** for sequential reuse — turning frontier-coding-agent pass@1 from a one-shot lottery into a controllable scaling axis.

## Why this paper matters

Test-time scaling (TTS) — the family of techniques that spends additional inference compute to lift a model's output quality — has been the dominant lever for closed-form domains: math (multi-sample + majority vote), single-turn code (best-of-N + LM judge), and reasoning (chain-of-thought + self-consistency). All of those techniques share an assumption: each candidate is a **bounded, comparable artifact** that fits in context.

Long-horizon agentic coding **violates that assumption**. Each rollout is an interleaved trajectory of `(thought, bash_command, observation)` triples that can run dozens of steps. A single SWE-Bench Verified attempt by Claude-4.5-Opus averages **41 steps**; the failing tail goes to 60+. Stuffing 16 of those into a context window for tournament comparison is impossible *and* would not work (the trajectories are too noisy to compare directly even if they fit). The classical TTS playbook breaks at exactly the regime — autonomous coding — where TTS would matter most.

Meta TTS is the first paper to give a **principled answer**: the bottleneck is *representation*, not compute or model strength. If you can produce a faithful compact representation of each rollout, then the standard TTS levers (selection, refinement) snap back into place — but operating over summaries, not trajectories. The empirical proof is striking: **+6.66 pp on SWE-Bench Verified for Claude-4.5-Opus (70.94 → 77.60), +12.14 pp on Terminal-Bench v2.0 (46.95 → 59.09)**, with similar gains across all five frontier models tested. Equivalent gains have historically required model retraining.

## Problem it solves

Three concrete problems that block naive TTS for coding agents:

1. **Rollouts are too long to compare.** A 41-step trajectory spanning many tool calls is rich in low-signal noise (repeated `ls` outputs, dead-end edits, terminal echo) that drowns the actual decisive moments (which file was edited, which test failed, what hypothesis is being pursued). Pairwise comparison of two raw trajectories degrades to noise matching.
2. **Rollouts are too long to reuse.** Conditioning a fresh attempt on a previous trajectory pollutes the context with the previous attempt's wrong turns, often causing the new attempt to recapitulate them. Token cost is also prohibitive — 16 prior trajectories at 50K tokens each is 800K tokens before the new agent makes its first move.
3. **Single-attempt pass@1 wastes the diversity already inside the model.** A frontier model may have a 70% chance of solving a SWE task, but the *specific 30%* it gets wrong on attempt 1 is partially decorrelated from the 30% it gets wrong on attempt 2. Naive best-of-N gives you pass@N (an oracle metric); the open question is how to recover that without ground truth.

Meta TTS directly addresses all three through one design move (compact summaries) plus two algorithms (RTV, PDR) that operate on those summaries.

## Core idea in one paragraph

Treat the rollout trajectory `R = [(T_i, B_i, O_i)]` not as the unit of scaling but as the **input** to a summarization step `S = LM[P_sum(R)]` whose output `S` is a structured artifact capturing the rollout's *salient hypotheses, decisions, progress, and failure modes* while discarding low-signal trace detail. With `{S_1, ..., S_N}` as the substrate, do two things in parallel: (a) select the best rollout via **Recursive Tournament Voting (RTV)** — recursively divide summaries into small groups (G=2 is best), have the LM compare each group V times (V=8), and vote-aggregate the winner — until a single summary remains; (b) **refine** future rollouts via **Parallel-Distill-Refine (PDR)** — run a fresh batch of N rollouts in clean environments, but condition each on the K best summaries from the prior iteration. Then run RTV on the refined population to pick the final answer.

## Mechanism (step by step)

Notation. `Π_LM` is the agent (LM + scaffold). For a coding problem `P_in`, the agent emits actions `A_i = (T_i, B_i)` (thought + bash commands) into environment `E`, receiving observations `O_i = E(C_{i-1}, B_i)`. Rollout `R = [(A_i, O_i)]_{i=1..H}` accumulates over horizon H.

### Step 1 — Summarize each rollout

For each completed rollout `R_i`, run `S_i = Π_LM[P_sum(R_i)]`. The summarization prompt instructs the LM to extract:
- the **hypothesis** the agent was testing (e.g., "the bug is in `parser.py:handle_quotes`"),
- the **decisions** made (which files edited, which approach taken),
- the **progress signals** (which tests pass now, which still fail, what error was eliminated),
- the **failure modes** observed (what didn't work and why),
- the **final code change** (concise diff or function-level patch).

Empirical anchor (Finding 1, Figure 4): on SWE-Bench Verified with Claude-4.5-Sonnet, comparing structured summaries beats comparing raw trajectories by 4–7 pp at the final round of tournament selection. The gap widens as the tournament eliminates easy pairs and the surviving candidates require attention to nuance.

### Step 2 — Recursive Tournament Voting (RTV)

Given `{S_1, ..., S_N}` with N=16:

```
def RTV(summaries, G=2, V=8):
    population = summaries
    while len(population) > 1:
        groups = chunk(population, size=G)
        next_population = []
        for group in groups:
            votes = [LM(P_comp(P_in, group)) for _ in range(V)]
            winner_idx = argmax(count(votes, label) for label in range(G))
            next_population.append(group[winner_idx])
        population = next_population
    return population[0]
```

The recursion does `log_G(N)` rounds. With N=16 and G=2 this is 4 rounds; each round involves N/G groups of G summaries each, with V=8 votes per group, so the total LM calls for RTV are roughly `V * (N - 1) = 120` comparison calls — much cheaper than running 16 more rollouts.

Two key design findings:
- **G=2 (pairwise) > G=4 > G=8 > G=16 (Finding 2, Figure 5 left).** Smaller groups force easier local decisions; large flat comparisons drown in detail. This is the same intuition as Tree-of-Thoughts using local expansions over flat reranking.
- **V=8 votes hits diminishing returns (Figure 5 right).** Going from V=1 to V=8 lifts pass@1 by 3–5 pp; V=16 adds only 0.5–1 pp.

RTV alone — applied to N=16 iteration-0 rollouts — already beats single-rollout baselines by 5–6 pp on SWE-Bench Verified and 8–12 pp on Terminal-Bench v2.0 (Figure 6). It is therefore a usable standalone TTS recipe even without sequential refinement.

### Step 3 — Parallel-Distill-Refine (PDR)

After getting the top-K rollouts from iteration 0 (selected by RTV, hence "select-K" rather than the original PDR's random-K), execute iteration 1:

```
context_iter1 = concat(S_j for j in top_K_indices)
for i in range(N):
    R_i_iter1 = run_agent(P_in, refinement_context=context_iter1, fresh_env=True)
    S_i_iter1 = summarize(R_i_iter1)
final_winner = RTV([S_1_iter1, ..., S_N_iter1])
return final_winner.code_change
```

Each iteration-1 rollout starts in a **freshly initialized container** (no state pollution from iteration 0), but the agent's first prompt now includes the K selected summaries as "prior experience". The agent can read what was already tried, which hypotheses panned out, which dead-ends to skip — and importantly, it can choose to *adopt or override* any of those leads.

Three findings about PDR design:
- **K > 1 is required (Finding 3, Table 1).** Refining with a single prior summary lifts Gemini-3.1-Pro from 72.69% → 73.75% — almost noise. Random-K with K=4 lifts to 76.94%. Select-K (RTV-chosen) with K=4 lifts to 79.25%.
- **Quality of refinement context dominates (Table 2).** When all K=4 context rollouts pass, iteration-1 pass rate is 97–99%. When 0/4 pass, iteration-1 pass rate collapses to 1–2%. The context is *not* a soft hint — it directly determines next-iteration outcomes.
- **Step count drops by ~50% (Table 4).** Claude-4.5-Opus uses 41 steps in iteration 0 and 14 steps in iteration 1 on SWE-Bench Verified. The refinement context lets the agent skip directory exploration, false hypotheses, and redundant tool calls — it goes more directly to the diff.

### Step 4 — Final RTV over refined population

Iteration-1 rollouts are not all winners (their pass rate is ~75–80% on the easy half of SWE-Bench, lower on Terminal-Bench). A final RTV pass over the iteration-1 summaries selects the single best one. This adds 1–2 pp on SWE and 5–8 pp on Terminal-Bench (Table 3), because Terminal-Bench has higher residual diversity even after refinement.

## Concrete pipeline pseudo-code

```python
def meta_tts(problem, model, N=16, T=2, K=4, G=2, V=8):
    rollouts = [run_agent_in_fresh_env(problem, model) for _ in range(N)]
    summaries = [summarize(r) for r in rollouts]
    for iteration in range(T - 1):
        top_K_summaries = recursive_tournament_select_K(
            summaries, G=G, V=V, K=K
        )
        rollouts = [
            run_agent_in_fresh_env(
                problem, model,
                refinement_context=concat(top_K_summaries),
            )
            for _ in range(N)
        ]
        summaries = [summarize(r) for r in rollouts]
    final_winner = RTV(summaries, G=G, V=V)
    return final_winner.code_diff
```

Compute budget at default parameters (N=16, T=2, K=4, G=2, V=8):
- 32 rollouts (16 per iteration × 2 iterations).
- 32 summarization calls.
- ~240 RTV comparison calls (120 per iteration).
- Equivalent token cost: ~2× a single attempt for rollouts, ~0.5× for summaries+RTV. Net: ~3× compute per problem, ~7 pp avg lift on SWE, ~11 pp avg lift on Terminal-Bench.

## Empirical results

Main results (Table 3 of the paper, average pass@1 %):

| Model | SWE Iter 0 | SWE Final | SWE Δ | TB Iter 0 | TB Final | TB Δ |
|---|---|---|---|---|---|---|
| Claude-4.5-Opus | 70.94 | 77.60 | **+6.66** | 46.95 | 59.09 | **+12.14** |
| Gemini-3.1-Pro | 72.25 | 76.60 | **+4.35** | 52.49 | 64.77 | **+12.28** |
| Claude-4.5-Sonnet | 67.41 | 75.60 | **+8.19** | 40.62 | 56.82 | **+16.20** |
| Gemini-3-Flash | 70.79 | 76.00 | **+5.21** | 37.93 | 48.86 | **+10.93** |
| GPT-5-0825 | 61.41 | 69.80 | **+8.39** | 31.32 | 38.64 | **+7.32** |

Notable patterns:
- **Larger gains on Terminal-Bench v2.0 than on SWE-Bench Verified.** Terminal-Bench has higher rollout variance (more open-ended, no precomputed test fixtures), so selection has more headroom.
- **Smaller models gain more from RTV.** GPT-5-0825 (lowest baseline) gains +8.39 pp on SWE; Claude-4.5-Sonnet gains +8.19 pp. Strong models that already pass most things gain less from selection.
- **Tasks not solvable in 16 single-shot rollouts become solvable.** Claude-4.5-Opus solves `gpt2-codegolf` after refinement that no iteration-0 attempt cracked; Gemini-3.1-Pro does the same for `large-scale-text-editing` (Appendix F.2).
- **Step count drops ~half on iteration 1** (Table 4). The agent spends fewer steps on exploration and more on the diff.

## Variants and ablations the paper studies

- **Substrate (raw vs. summary).** Raw trajectories underperform structured summaries by 4–7 pp at the final tournament round. The gap is small at round 0 (easy pairs decided by either substrate) and grows monotonically.
- **Group size G ∈ {2, 4, 8, 16}.** G=2 wins; G=16 (single-shot ranking over all summaries) is consistently worst.
- **Vote count V ∈ {1, 2, 4, 8, 16}.** V=8 is the knee.
- **K (refinement context size) ∈ {1, 4}.** K=1 is essentially no help; K=4 is the operating point. Higher K not tested in detail.
- **Random-K vs Select-K.** Select-K (RTV-chosen) consistently beats random-K by ~1–3 pp.
- **PDR alone vs PDR+RTV.** PDR with random-K shows the iteration-1 lift but is brittle when the random sample misses the good rollouts. PDR+RTV with select-K is the strict generalization.
- **Iteration count T ∈ {1, 2}.** T=2 is the default; the paper does not push to T=3+ extensively but notes diminishing returns past T=2 on the benchmarks tested.

## Failure modes and limitations

1. **Compute cost.** ~3× per problem at the default config. Budget-constrained deployments need to choose N, T carefully. The paper does not explore Pareto curves at lower N (e.g., N=4) where the tournament shrinks.
2. **Summary fidelity bound.** The whole pipeline rests on summaries faithfully capturing rollout content. If the summarizer hallucinates progress that didn't happen, RTV can pick the hallucinated winner. The paper uses the same model for summarization and rollout generation, which keeps the bias consistent but doesn't eliminate the failure mode.
3. **No external verifier.** RTV is fully internal — the LM is judge of itself. The paper notes this is a deliberate choice (no test-suite leakage, applicable to tasks without oracle tests), but it caps performance below what an oracle ranker would achieve.
4. **Diversity collapse risk.** All N rollouts and all V votes come from the same model. Correlated errors (the model has the same bug in its hypothesis space across all 16 rollouts) cannot be smoothed out by RTV. This is the [diversity-collapse-mas](98-diversity-collapse-mas.md) failure mode at the per-rollout scale.
5. **Refinement context can poison if all K rollouts are wrong.** Table 2 shows that 0/4 passing context produces 1–2% iteration-1 pass rate — meaning the refinement context can actively harm the agent if the prior iteration was uniformly bad. The agent does not know to ignore the context.
6. **Domain narrowness.** The paper evaluates only on SWE-Bench Verified and Terminal-Bench v2.0. Generalization to web agents, GUI agents, or tool-using research agents is plausible but unproven.

## When to use, when not to

**Use when:**
- You have a frontier coding agent (LM + scaffold) and want to lift pass@1 by 5–15 pp without retraining.
- You can afford 2–4× compute per problem.
- The task has high rollout variance — i.e., pass@N is meaningfully higher than pass@1.
- You have isolated container infrastructure to run N parallel rollouts in fresh environments.
- You don't have access to an oracle test suite, so external best-of-N selection isn't possible.

**Don't use when:**
- You already have an oracle test suite (just use pass@N directly with the test as judge — cheaper and stronger).
- Single-shot pass@1 is already at ceiling (>95%) — RTV can't improve much.
- The task is short-horizon (one bash call, one diff). The summarization step adds overhead without representational benefit.
- Compute is tight and you'd rather invest the 3× budget in a stronger base model.
- All your rollouts will be uniformly wrong (then PDR poisons iteration 1 — fall back to RTV-only).

## Implications for harness engineering

Meta TTS reshapes several harness primitives:

1. **Rollout summaries become a first-class harness artifact.** Beyond CLAUDE.md, memory files, and skills, the harness now owns a *trajectory representation* layer: a contract for how a rollout is compressed into something selectable, refinable, and storable. Existing harnesses that store full transcripts are leaving a TTS lever on the table. See [09-memory-files](09-memory-files.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md), and [72-claude-mem](72-claude-mem-persistent-memory-compression.md) for adjacent concepts.

2. **Parallel-rollout infrastructure is now table stakes.** RTV+PDR requires N=16 fresh containers per problem. Harnesses that run a single agent at a time can't apply this. Cube Sandbox, OpenClaw's executor pool, and similar microVM-fast-fork infrastructures become essential.

3. **The harness is the TTS scheduler.** Iteration count T, N, K, G, V are knobs the harness sets per problem. A "thinking harder" mode is no longer just longer chain-of-thought — it's "more rollouts + bigger tournament + more iterations". The harness exposes a TTS budget knob analogous to the model's `max_thinking_tokens`.

4. **Cross-rollout context engineering becomes the new context engineering.** The K selected summaries are loaded into iteration-1's prompt — they compete for context space with system prompts, skills, project memory, and tool definitions. Harness designers now need a context-budget allocator that decides how much of the window goes to TTS refinement context vs. per-task context. See [08-context-compaction](08-context-compaction.md).

5. **Pass@N becomes a routinely-attainable metric in production.** Historically pass@N was a benchmark artifact; production deployments shipped pass@1. Meta TTS shows pass@N - 5pp is achievable at 3× cost in production. Product teams now have a knob for quality-vs-cost that didn't exist.

6. **The "verifier-evaluator loop" pattern (see [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md)) generalizes.** RTV is a verifier that operates on summaries instead of full trajectories. The same compression-then-judge structure applies to multi-agent systems, agent-as-tool wrappers, and recursive agent calls.

## Connections to other work in this corpus

- **[14-reflexion](14-reflexion.md) and [86-reflexion-deep](86-reflexion-deep.md)** — Reflexion writes verbal lessons into episodic memory; PDR's refinement context is a structurally similar idea but with summaries instead of free-form lessons, and at rollout granularity instead of episode granularity.
- **[15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md)** — RTV's small-group recursion is a flat tournament version of LATS's tree search; the lesson that "small local decisions beat one big global decision" is shared.
- **[68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md)** — Atomic Skills decomposes the agent's repertoire; Meta TTS leaves the agent monolithic but scales its outputs. The two stack: atomic skills inside each rollout, then RTV+PDR over rollouts.
- **[80-swe-search-mcts](80-swe-search-mcts.md)** — SWE-Search uses MCTS *within* a single attempt; Meta TTS uses tournament selection *across* attempts. They are orthogonal and combinable (see paper Section 5 discussion).
- **[98-diversity-collapse-mas](98-diversity-collapse-mas.md)** — RTV's reliance on a single model for all rollouts and all votes is a diversity-collapse risk; the paper does not address it. A mixture-of-models extension is an obvious follow-up.

## Key takeaways

1. **For long-horizon agents, test-time scaling is fundamentally a problem of representation, selection, and reuse** (the paper's own one-line summary).
2. Compact structured summaries are the right substrate; raw trajectories are not.
3. RTV (G=2, V=8) is the right selector — small-group recursion with vote aggregation beats flat ranking.
4. PDR with select-K (K=4) is the right refinement — multiple selected summaries beat single or random.
5. **+7 pp on SWE-Bench Verified, +11 pp on Terminal-Bench v2.0, on average across five frontier models, at ~3× compute** — equivalent to a generation of base-model progress in many cases.
6. The technique is harness-level, not model-level: every Claude-Code-style harness can adopt it without model retraining.

## References

- **Paper.** Kim et al., *Scaling Test-Time Compute for Agentic Coding*, arXiv:2604.16529, April 2026.
- **PDR origin.** Madaan et al., 2025 — Parallel-Distill-Refine, the sequential refinement primitive Meta TTS adapts to the agentic setting.
- **Tournament voting in TTS.** Snell et al., 2024 — early use of tournament-style voting in math TTS.
- **SWE-Bench Verified.** Jimenez et al., 2024.
- **Terminal-Bench v2.0.** Merrill et al., 2026.
- **Adjacent corpus files.** [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md), [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md), [80-swe-search-mcts](80-swe-search-mcts.md), [86-reflexion-deep](86-reflexion-deep.md), [98-diversity-collapse-mas](98-diversity-collapse-mas.md).
