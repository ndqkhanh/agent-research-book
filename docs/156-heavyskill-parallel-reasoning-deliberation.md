# 156 — HeavySkill: Heavy Thinking as the Inner Skill in Agentic Harness

**Paper.** Jianing Wang, Linsen Guo, Zhengyu Chen, Qi Guo, Hongyu Zang, Wenjie Shi, Haoxiang Ma, Xiangyu Xi, Xiaoyu Li, Wei Wang, Xunliang Cai — *HeavySkill: Heavy Thinking as the Inner Skill in Agentic Harness* — arXiv:2605.02396v1 [cs.AI] — Preprint 5 May 2026 (submitted 4 May 2026) — **Meituan LongCat Team** + **National Engineering Research Center for Software Engineering, Peking University** — code at https://github.com/wjn1996/HeavySkill — 18 pages, 10 figures.

**One-line definition.** HeavySkill is the thesis — and its empirical validation — that *the inner mechanism actually driving agentic-harness performance is a single distillable skill: parallel reasoning followed by sequential deliberation*; the paper provides (a) a training-free two-stage workflow that any harness can drop in beneath its orchestrator, (b) a *readable Markdown skill file* (`HeavySkill.md`) that lets a frontier LLM self-orchestrate the workflow with no harness modifications, and (c) RLVR evidence that the depth and width of heavy thinking can be *trained into the model's parameters* — turning what looks like orchestration glue into a learnable inner capability that consistently lifts metrics like LiveCodeBench (GPT-OSS-20B 69.7 → 85.5%) and IFEval (R1-Distill-Qwen-32B 35.7 → 69.3%).

## Why this paper matters

Harness engineering and model engineering are usually treated as separate disciplines. Harness builders write orchestration glue around fixed models ([29-dive-into-claude-code](29-dive-into-claude-code.md), [42-langchain-deep-agents](42-langchain-deep-agents.md), [52-dive-into-open-claw](52-dive-into-open-claw.md)). Model trainers train models that are *good enough* to plug into harnesses, treating the harness as a downstream consumer. The two communities meet at API boundaries.

HeavySkill argues this divide is the wrong cut. The actual lift you observe when you wrap an LLM in an orchestrated harness — multiple subagents reasoning in parallel, a summariser aggregating the results — is, mechanistically, *one skill*: parallel reasoning + sequential deliberation. And that skill can be (1) extracted out of the harness into a portable Markdown file, (2) executed by any sufficiently capable LM following the file's instructions, and (3) further internalised into the model's weights via RLVR.

The harness wins, when you trace them carefully, are model wins waiting to happen.

The implications:

- **For harness builders**: a substantial fraction of the orchestration code in modern harnesses is implementing this one skill. Once the skill is internalised into model weights (heavy-mode-aware RLVR is the path), the orchestration glue becomes optional.
- **For model trainers**: the long arc is models that come with parallel-reasoning-plus-deliberation baked in, not orchestration glue around them. Frontier providers are already moving this direction (Kimi K2 Thinking, GPT-5 Thinking, DeepSeek V3.2 Thinking, GLM-4.6).
- **For the field**: the harness/model boundary is *moving*. Every season, capabilities migrate from the harness layer down into the model. HeavySkill names the next major migration.

This pairs with three other May-2026 papers ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md), [154](154-ctx2skill-self-evolving-context-skills.md), [155](155-feynman-multi-agent-research-harness.md)) to redraw what we mean by "harness." The synthesis chapter ([157](157-may-2026-synthesis-memory-and-skills.md)) collects this argument.

## Problem it solves

Five concrete problems HeavySkill closes:

1. **Opaque harness performance.** Modern harnesses orchestrate multi-agent loops with memory, skills, and tool use, achieving impressive results on complex reasoning. But *which mechanism inside the harness drives the gain* has been unclear. HeavySkill identifies the mechanism: parallel reasoning + sequential deliberation, isolated from the rest of the orchestration.
2. **Brittle orchestration glue.** Harness-level subagent orchestration is fragile — depends on framework SDK versions, system-prompt templates, tool schemas. HeavySkill shows the same effect can be achieved by a *single skill file* that any sufficient LM follows in-context, with no framework-specific glue.
3. **Best-of-N falls short of Pass@N.** The dominant test-time-scaling pattern (Best-of-N + voting) leaves a substantial gap between the *intrinsic potential* of the model (Pass@K) and the actual delivered accuracy (Vote@K). HeavySkill shows that sequential deliberation closes this gap dramatically — frontier models approach Pass@K under HM@4, and HP@4 sometimes *exceeds* P@K because deliberation can synthesise correct answers that no single trajectory produced.
4. **Test-time scaling without architectural changes.** Specialised parallel-reasoning architectures (Hogwild, ParaThinker, APAR, Multiverse, APE) modify decoding or attention. HeavySkill achieves much of the same lift with a vanilla two-stage pipeline that works under any decoder.
5. **Skill-as-orchestration in production harnesses.** Modern harnesses (Claude Code, CodeX, Hermes, OpenClaw) load Markdown skills into context. HeavySkill is the first paper to *write down a skill file that turns any harness into a heavy-thinking harness*, demonstrably portable across Claude Code and custom orchestrators.

## Core idea in one paragraph

Most of the lift you see from orchestrated multi-agent harnesses on reasoning tasks comes from one thing: *spawn K independent reasoning attempts, then have a single aggregator deliberate over them to produce a final answer*. Decompose a harness into this structure, run it with vanilla LLM calls, and you can match (and frequently surpass) what the orchestrator produces. The deliberator is not a voter — it is a *re-reasoner* that critically evaluates each parallel trajectory, identifies logical errors, and synthesises a final answer that may agree with the majority, with a minority, or with none of them. Empirically, HM@4 (Heavy-Mean@4 — accuracy of deliberated outputs) consistently exceeds M@K (mean of parallel outputs) and frequently exceeds V@K (majority voting). On frontier models, HM@4 approaches P@K (the boundary of what any single trajectory can solve); HP@4 (Heavy-Pass@4) sometimes *exceeds* P@K because deliberation can rescue answers no parallel run produced. The skill is portable as a Markdown file that any sufficiently capable LLM can self-orchestrate. RLVR can further train the depth (deliberation) and width (parallel) of the skill into model weights — early experiments show ~10% HM@4 lift in the first 100 RL steps.

## Mechanism (step by step)

### 1. The two-stage pipeline

For a problem q, define:

```
Stage 1 (Parallel Reasoning):
   T_{π_θ}(q, K)  =  {y_1, …, y_K}
```

K independent reasoning trajectories, each y_i = {π_θ(y_{i,j} | q, y_{i,<j})}_j, generated by an LLM π_θ. Each trajectory reasons from scratch with no awareness of the others.

```
Stage 2 (Sequential Deliberation):
   x_c     =  C(T_{π_θ}(q, K))                    [serialized memory cache]
   {z_1, …, z_{K^{(1)}}}  =  T_{π_φ}(x_c, K^{(1)})  [aggregator outputs]
```

A second LLM π_φ (which can be the same model as π_θ) reads the serialized memory cache x_c and produces K^(1) summary contents (default K^(1) = 4). Each summary content is *its own* deliberation — multiple deliberation samples with shuffled trajectory order to get a diversity of synthesised answers.

### 2. Serialized memory cache

The cache C(·) is the bridge between stages. Two design problems it solves:

- **Trajectory pruning**: complete reasoning trajectories with internal thinking + answer often exceed the model's max length. The cache prunes each trajectory (typically keeping the answer + a compressed reasoning summary) to fit within budget.
- **Position bias prevention**: pruned trajectories are *shuffled* before serialisation so the deliberator does not develop a positional preference. This is the same anti-bias hygiene used in [98-diversity-collapse-mas](98-diversity-collapse-mas.md) and many multi-agent voting systems.

The serialisation prompt format (Figure 7 in the paper) is roughly:

```
Here is a problem, and multiple thinkers attempt to give their thought processes
independently. Each thinker has written its own thought process toward the final
answer. Each thinker is encouraged to take the other thinkers' progress into
account to reach the final answer.

# ====== Problem ======
{problem}
# ====== Problem End ======

# ====== Thinkers Thought Process ======
# ----- Thinker #1 -----
{trajectory 1}
# ----- Thinker #2 -----
{trajectory 2}
…
# ====== Thinkers Thought Process End ======

Look at the above problem and thought process from each thinker, summarize…
```

The deliberation prompt (the meta-instruction that the aggregator follows) explicitly instructs:

1. Adhere to *professionalism and critical thinking*; identify each thought process carefully.
2. Recognise that *majority is a signal, not proof* — the correct answer may come from very few thinkers, or none.
3. If all thinkers are wrong, *re-reason the problem from scratch* using their mistakes as evidence.
4. Output format consistency — `\boxed{·}` for math, code blocks for programming.

### 3. Iterative deliberation (optional, N>1)

For especially challenging problems, deliberation can iterate. At iteration t ∈ {2, …, N}:

```
x_c^{(t)}  =  T_{π_φ}(x_c^{(t-1)}, K^{(t-1)})  ||  x^{(t-1)}
```

i.e., the previous iteration's deliberation output is concatenated to the cache, and the aggregator re-deliberates over the augmented cache. This recursively refines the synthesis.

The empirical finding (Figure 4 in the paper):

- HM@K shows a *consistent upward trend* across iterations 1 → 4 on multiple models (R1-Distill-Qwen-7B, R1-Distill-Qwen3-8B, DeepSeek-R1-0528) and benchmarks (HMMT25-Feb, GPQA-Diamond).
- HP@K, however, *degrades* with iteration count. Cumulative noise from earlier iterations interferes with refinement.

The interpretation: iterative deliberation helps the *mean* converge but reduces the *boundary* of what is accessible. There is a real trade-off between iteration depth and information consistency. The paper's default setting is N=1 (single deliberation), with N=4 reserved for hard problems where you are willing to trade ceiling for floor.

### 4. The four metrics

A principled set of metrics for understanding HeavySkill:

| Metric | Definition |
|--------|------------|
| **M@K** (Mean@K) | Average accuracy of K parallel trajectories. The "naive parallel" baseline. |
| **P@K** (Pass@K) | Proportion of K trajectories where at least one is correct. The intrinsic-potential ceiling. |
| **V@K** (Vote@K) | Accuracy of the majority-vote answer. The Best-of-N baseline. |
| **HM@K** (Heavy-Mean@K) | Average accuracy of K^(1) deliberation outputs. The "heavy thinking" headline metric. |
| **HP@K** (Heavy-Pass@K) | Proportion of K^(1) deliberation outputs where at least one is correct. The deliberation potential. |

The paper reports a consistent *performance hierarchy* on STEM tasks:

```
Heavy-Pass@K  ≥  Heavy-Mean@K  ≥  Vote@K  ≥  Mean@K
```

with the gap between V@K and M@K representing the value of voting, and the gap between HM@K and V@K representing the *additional* value of deliberation on top of voting.

### 5. The readable skill (HeavySkill.md)

The most operationally important contribution of the paper. The two-stage workflow is normally implemented as a Python pipeline: an external orchestrator spawns K LLM calls, collects responses, runs another LLM call for deliberation, returns the result.

HeavySkill argues this is *unnecessary* for any modern harness. A skill — a Markdown document that the orchestrator loads into context — can specify:
- **Activation conditions**: when to trigger heavy thinking (complex reasoning, math, code, uncertain initial approach) vs not (simple factual queries, casual conversation).
- **Parallel reasoning protocol**: instructions for the orchestrator to spawn K independent reasoning agents in parallel, each from scratch.
- **Deliberation prompt**: the meta-instruction template for the aggregator.
- **Output constraints**: format conventions per domain.

The result is a single `HeavySkill.md` file (Figures 8–10 of the paper) that the orchestrator loads when relevant. The orchestrator itself does parallel-spawn + deliberate using its native subagent capabilities (e.g., Claude Code's subagent tool, OpenClaw's Lobster engine). *No code modifications to the harness.*

The shift from workflow to skill changes the *locus of control*:

| Mode      | Orchestration controlled by      | Determinism                     | Portability                            |
|-----------|----------------------------------|---------------------------------|----------------------------------------|
| Workflow  | External Python pipeline         | High (deterministic flow)       | Bound to the pipeline implementation   |
| Skill     | LLM following Markdown protocol  | Lower (LLM-mediated)            | Portable across any skill-loading harness |

The paper verifies the same `HeavySkill.md` works under both Claude Code and custom orchestration harnesses *without modification*. The skill is portable because it is plain text.

This is the same thesis that [04-skills](04-skills.md) and [71-karpathy-skills-single-file-guardrails](71-karpathy-skills-single-file-guardrails.md) introduced and that [154-ctx2skill-self-evolving-context-skills](154-ctx2skill-self-evolving-context-skills.md) operationalised for context learning: skills are the portable, model-agnostic capability artefact.

## Empirical anchors

### STEM benchmarks (Table 1)

The headline empirical study is across four STEM benchmarks (AIME25, BeyondAIME, HMMT25-Feb, GPQA-Diamond) and a wide model menagerie. K ∈ {8, 16}, K^(1) = 4. A representative cross-section:

| Model                  | K  | M@k  | P@k  | V@k  | HM@4 | HP@4 |
|------------------------|----|------|------|------|------|------|
| **AIME25**             |    |      |      |      |      |      |
| GPT-5 Thinking         | 16 | 91.9 | 100  | 96.7 | 99.2 | 100  |
| Kimi K2 Thinking       | 16 | 95.2 | 100  | 96.7 | 99.2 | 100  |
| GLM-4.6                | 16 | 93.1 | 100  | 100  | 96.7 | 96.7 |
| GPT-OSS-20B            | 16 | 92.0 | 96.7 | 96.7 | 93.3 | 96.7 |
| R1-Distill-Qwen-32B    | 16 | 52.3 | 80.0 | 66.7 | 68.3 | 76.7 |
| R1-Distill-Qwen-7B     | 16 | 41.7 | 66.7 | 60.0 | 56.7 | 60.0 |
| **BeyondAIME**         |    |      |      |      |      |      |
| GPT-5 Thinking         | 16 | 70.1 | 91.0 | 73.0 | 82.5 | 88.0 |
| Kimi K2 Thinking       | 8  | 76.8 | 87.0 | 81.0 | 83.0 | 84.0 |
| Claude 4.5 Thinking    | 16 | 59.4 | 83.4 | 70.0 | 68.3 | 71.5 |
| **HMMT25-Feb**         |    |      |      |      |      |      |
| Gemini-3 Pro           | 16 | 96.5 | 100  | 93.3 | 100  | 100  |
| GLM-4.6                | 16 | 90.4 | 100  | 96.7 | 99.2 | 100  |
| GPT-5 Thinking         | 16 | 89.8 | 96.7 | 86.7 | 95.0 | 96.7 |
| **GPQA-Diamond**       |    |      |      |      |      |      |
| GPT-5 Thinking         | 16 | 85.6 | 97.5 | 86.4 | 87.2 | 90.9 |
| Kimi K2 Thinking       | 16 | 85.3 | 97.0 | 80.3 | 87.5 | 91.4 |
| GLM-4.6                | 16 | 82.5 | 96.0 | 80.3 | 87.5 | 91.4 |

Read across models: HM@4 ≥ V@K consistently. HM@4 frequently approaches P@K. HP@4 sometimes exceeds P@K — deliberation can produce correct answers that no parallel trajectory generated.

### General reasoning tasks (Table 2)

| Model                  | LiveCodeBench M@k → HM@4 | IFEval M@k → HM@4 | IFEval HP@4 | IMO HM@4 → HP@4 |
|------------------------|--------------------------|-------------------|--------------|------------------|
| GPT-OSS-20B            | **69.7 → 85.5**          | 90.8 → 91.1       | 97.6         | 71.0 → 84.5      |
| R1-Distill-Qwen3-8B    | 56.3 → 56.8              | **35.7 → 69.3**   | 86.5         | 47.2 → 62.3      |
| Kimi K2 Thinking       | 81.2 → 83.7              | 92.5 → 92.0       | 97.6         | 77.2 → 88.0      |
| GLM-4.6                | 81.0 → 81.3              | 88.8 → 88.5       | 94.8         | 75.1 → 86.0      |
| Qwen3-8B               | 55.5 → 56.8              | 85.4 → 85.2       | 92.4         | 50.3 → 63.3      |

The big jumps:

- **GPT-OSS-20B on LiveCodeBench**: 69.7% → 85.5%. A +15.8 pp lift on a coding benchmark, just from heavy thinking.
- **R1-Distill-Qwen-32B on IFEval**: 35.7% → 69.3%. *Nearly doubling* on instruction-following — IFEval is highly verifiable, and deliberation distils high-quality solutions from multiple reasoning paths.
- **HP@4 on IFEval ≈ 86–98**: across all models, the deliberation potential on IFEval is enormous. Many of the best-possible answers are within reach if the deliberator picks correctly.

The pattern across general tasks:

- **Verifiable, correctness-oriented tasks** (LiveCodeBench, IFEval) → big HM@4 gains.
- **Subjective, preference-oriented tasks** (Arena-Hard) → marginal or slightly negative gains. The "mean of multiple responses" doesn't necessarily align with the stylistic nuances a reward model favours.

### Pass-rate distribution analysis (Figure 2)

The paper conducts a granular study: take 10K queries, compute the parallel pass rate for each (with K=16), bucket into intervals {0.125, 0.375, 0.625, 0.875}, and measure the heavy pass rate per bucket.

Key insights:

1. **Below 0.5 parallel pass rate**: where Vote@K is at its weakest, heavy thinking shows substantial corrective potential. ~500 queries are rescued through deliberation that voting could not solve.
2. **Above 0.5 parallel pass rate**: heavy thinking maintains >98% accuracy. The few cases of degradation are negligible.

This is the strongest empirical evidence that deliberation is *not just voting in disguise*. Voting can only pick from existing answers; deliberation can synthesise new ones.

### Model-pairing analysis (Figure 3)

Fix the parallel-reasoning model as R1-Distill-Qwen-7B. Vary the deliberation model:

| Deliberation model        | AIME25 K=8 HM@K | HMMT25 K=8 HM@K |
|---------------------------|-----------------|-----------------|
| R1-Distill-Qwen-7B        | 56.66           | 39.16           |
| R1-Distill-Qwen3-8B       | 60.00           | 38.33           |
| Qwen2.5-32B-Instruct      | 53.33           | 30.00           |

Counter-intuitive finding: *Qwen2.5-32B-Instruct* — which is *worse* on these benchmarks than R1-Distill-Qwen-7B as a primary solver — still produces a positive HM@K lift over M@K when used as the deliberator. The deliberation phase does not need *peak* reasoning. It needs *comprehensive analysis, synthesis, and summarisation* of the parallel traces.

The implication: **separate optimisation of thinking and deliberation models may yield additional gains.** Use a strong reasoner for parallel; use a strong instruction-follower for deliberation. Don't assume the same model is best for both.

### Tool-interleaved reasoning (Table 3)

When parallel trajectories include tool calls (e.g., Python interpreter):

| Model        | AIME25 V@4 → HM@4 | HMMT25 V@4 → HM@4 |
|--------------|--------------------|--------------------|
| Qwen3-8B     | 68.3 → **76.7**    | 54.1 → **69.3**    |
| Qwen3-32B    | 83.3 → 80.0        | 63.3 → 68.5        |
| GPT-OSS-20B  | 83.3 → **90.0**    | 73.3 → **85.7**    |

Tool-interleaved heavy thinking shows the same HM@4 > V@4 pattern, with execution feedback from the Python interpreter providing a signal the deliberator can leverage. The framework generalises to tool use without modification.

### Permutation strategies (Appendix A)

Given 256 parallel trajectories, four ways to select the K used in the cache:

| Strategy              | Effect |
|-----------------------|--------|
| Random                | Baseline |
| Max-Diversity         | ~Equivalent to Random — explicit diversity optimisation does not help. |
| Max-Length            | *Worst* — verbosity is not a signal of correctness. |
| Max-Answer-Number     | *Best* — majority-vote-derived selection of trajectories yields more correct paths in the candidate set. |

The takeaway: leveraging consensus to *pre-filter* trajectories before deliberation strengthens the candidate set the deliberator works from. This pairs naturally with V@K — but as a *preprocessor* for HM@K, not as the final answer.

### Heavy-mode-aware RLVR (Appendix B)

The most forward-looking experiment. Take the parallel trajectories from the analysis above, select queries with pass rate ∈ [0, 0.625], package them as serialised memory caches, and train R1-Distill-Qwen-7B with Group Sequence Policy Optimisation (GSPO) under the VeRL framework.

Results (Figure 6):

- First ~100 training steps: HM@4 *improves by ~10% on both training and test sets* — substantial lift.
- K=8: stable training throughout.
- K=16: *entropy collapse* after step 100 — the longer serialised contexts exceed model max-sequence limits, training signals become truncated, instability sets in.

The promise: heavy-thinking depth and width are *learnable parameters*, not just inference-time configurations. With more careful training (longer sequences, better RL algorithms), the gap between HM@4 and HP@4 can be further closed — pushing the limits of LLM reasoning beyond per-trajectory constraints.

This is the empirical foothold for the paper's central thesis: *the inner skill can be trained into the model.*

## What this paper changes

Three reframings that, if HeavySkill's argument generalises, will reshape harness engineering over the next 12–24 months:

### 1. Harness wins are model wins waiting to happen

For a long time, the playbook has been: take the best frontier model you can afford, wrap it in clever orchestration, ship the harness as your product. The harness is the moat. The model is a commodity dependency.

HeavySkill suggests this is unstable. The cleverness in the harness — at least the kind that drives reasoning lifts — is one identifiable skill. That skill can be (a) extracted into a portable Markdown file, (b) further internalised into model weights via RLVR. Either path collapses the harness's contribution.

What does that mean operationally?

- **If you are a harness vendor**: your differentiation must come from things RLVR cannot easily train into a model — UX polish, integrations, organisational policy, observability, durable execution. The reasoning lift is, increasingly, table stakes the model itself provides.
- **If you are a model trainer**: heavy-thinking-aware RLVR is now an obvious thing to do. Frontier providers will (and probably already do) bake parallel-reasoning-plus-deliberation into post-training.
- **If you are a builder**: write your reasoning capability as a skill rather than orchestration code. When the underlying model gets the skill internalised, your skill file becomes redundant — but until then, it is your portable lift.

### 2. Skills are the new training target

This is the second-order implication of the first. If the *thing you want the model to do* can be specified as a Markdown skill file, and if RLVR can train arbitrary procedural skills into the model, then the right pipeline is:

1. Write skills.
2. Verify they work in-context.
3. Use the skills as RLVR training data — generate trajectories from the skill, reward verifiable outputs, train.
4. Once internalised, retire the skill from the prompt (it's now in weights).

This loop is not yet productionised at scale, but HeavySkill's experiments show it *works for at least one skill*. The same loop should work for many skills — debugging skills, browsing skills, tool-use skills, refactoring skills.

This connects to [80-knowrl](80-knowrl.md), [81-reasoningbank](81-reasoningbank.md), [85-alphaevolve](85-alphaevolve.md), and the broader self-evolving canon. HeavySkill names the *concrete pattern*: turn skill files into training data for RLVR.

### 3. Pass@N becomes the new ceiling to chase

Before HeavySkill, Pass@N was the *theoretical* ceiling — what the model could solve given infinitely many tries. Practical performance was Vote@K, somewhere between M@K and P@K but not at P@K. HeavySkill shows HM@4 frequently approaches and sometimes *exceeds* P@K.

The new ceiling is HP@K — the deliberation potential. And it is empirically reachable in many cases.

This reframes a lot of evaluation. *Single-shot accuracy* is the wrong metric for hard reasoning tasks. M@K, V@K, HM@K, HP@K together tell you the full story: how good is the model alone (M), how good is voting (V), how good is deliberation (HM), how good *could* deliberation be at the boundary (HP).

## Implementation guidance

If you are operationalising HeavySkill in your own harness or model training pipeline:

### As a skill

The simplest path. Drop `HeavySkill.md` (or your own version following the structure) into your harness's skill loader. Activation rules:

- Activate on: complex math, code competition, tasks where correctness is critical and verifiable, problems where uncertainty about initial approach is high.
- Do *not* activate on: simple factual queries, casual conversation, straightforward edits, information retrieval.

Default settings: K = 3-5 in harness mode (cost of subagent spawn is non-trivial), K = 8+ in workflow mode (Python pipeline).

Spawn parallel agents *in a single message* (most harnesses support this). Wait for all to complete. Do the deliberation yourself (lead agent, do not delegate). Output the synthesised answer in domain-correct format.

### As a workflow

A Python pipeline if you need deterministic control. Pseudocode:

```python
def heavy_skill(query, K=8, K_summary=4, N=1):
    # Stage 1: parallel reasoning
    trajectories = parallel_call(model_theta, query, K)

    # Build memory cache
    cache = serialize(prune(trajectories), shuffle=True)

    # Stage 2: sequential deliberation, optionally iterative
    for t in range(N):
        outputs = parallel_call(model_phi, cache, K_summary)
        if t < N - 1:
            cache = cache + serialize(outputs)

    # Return the deliberated outputs (HM@K) or just the best (HP@K)
    return outputs
```

Picking K^(1): default 4 in the paper. Higher K^(1) gives more deliberation samples to choose from but is costly.

Picking N: default 1. Use N=4 for very hard problems where you want HM@K convergence; expect HP@K to degrade.

Picking deliberator: per Figure 3, do not assume the strongest reasoner is the best deliberator. A strong instruction-follower may do better. Ablate.

### As a training signal

Use HeavySkill traces as RLVR training data:

1. Run the workflow on a reasoning dataset (DAPO, DeepScaler, Skywork OR1, MATH).
2. Collect (query, parallel trajectories, deliberation output, ground-truth verdict) tuples.
3. Train with GSPO under VeRL (or your equivalent).
4. Reward: HM@4-correct → +1, HM@4-wrong → −1.
5. Watch for entropy collapse in long-K settings; clip K to your model's stable max-sequence regime.

The first 100 steps should give visible HM@4 lift. Diminishing returns past ~500 steps.

## Limitations

The paper acknowledges several:

1. **Iterative deliberation has a ceiling.** N>1 helps HM@K but hurts HP@K. The mechanism is cumulative noise from earlier iterations interfering with refinement. This is a real bound on iterative scaling.
2. **Subjective tasks don't gain.** Arena-Hard shows marginal or slightly negative gains. Deliberation distils correctness, not preference. Style/tone tasks need different mechanisms.
3. **Sequence-length constraints in RLVR.** K=16 RL training collapses on R1-Distill-Qwen-7B due to max-sequence-length truncation. Larger contexts and bigger models will be needed for RLVR to scale to longer K.
4. **Evaluator coverage is uneven.** Several models lack measurements on some columns of Tables 1–2. The complete picture is partially missing.
5. **Deliberator strength matters but in an unintuitive way.** Figure 3's surprise (Qwen2.5-32B-Instruct as a useful deliberator despite low intrinsic reasoning) is poorly explored. Why does instruction-following matter more than reasoning at this stage? Unclear.
6. **Skill versus workflow trade-off.** Skill mode trades determinism for portability. The paper claims they "function correctly" under different harnesses but does not quantify the gap.

A seventh implicit limitation: cost. Heavy thinking with K=16 + K^(1)=4 means ~20 LLM calls per query. For production at scale, this is expensive. The economics make sense for high-stakes reasoning (math contests, hard coding, scientific QA) and break down for high-throughput, low-cost queries.

## How HeavySkill fits with the other May-2026 papers

The four papers in this set form a coherent argument:

- **MEMTIER ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md))**: long-horizon agents need *structured durable memory*; the model is not the bottleneck.
- **Ctx2Skill ([154](154-ctx2skill-self-evolving-context-skills.md))**: context-specific knowledge can be *distilled into portable skill files* via adversarial self-play; weaker model + good skills > stronger model alone.
- **Feynman ([155](155-feynman-multi-agent-research-harness.md))**: the harness layer is *Markdown subagents and Markdown skills with file-based handoffs*; the runtime is glue.
- **HeavySkill (this chapter)**: the inner reasoning skill that drives harness performance is *parallel + deliberation*, distillable into a skill file and trainable into model weights.

The synthesis ([157](157-may-2026-synthesis-memory-and-skills.md)) collects the joint argument: **memory and skills are the two trainable substrates of the agent stack; the harness is the file-based glue that connects them; the model is increasingly an interchangeable runtime.**

## Failure modes & anti-patterns

- **Activating heavy thinking on every query.** Massively expensive, no real benefit on simple queries. Use the activation rules.
- **Using the same model for parallel and deliberation when the deliberator should be an instruction-follower.** Per Figure 3, ablate.
- **Letting parallel trajectories share state.** The whole loop depends on independence. Trajectories that see each other are no longer parallel reasoning; they are early collaboration, which loses the diversity benefit.
- **Skipping the meta-reasoning prompt.** The deliberator must be told to *critically evaluate*, not just *summarise*. A "summarise these" prompt collapses to majority voting.
- **Iterating beyond N=4.** HP@K degrades past this point. More iterations is not better.
- **Using verbosity (Max-Length) to select trajectories.** Worst possible heuristic per Figure 5.
- **Treating HM@K and HP@K as the same.** HP@K is an upper bound — the *potential* of deliberation. HM@K is what you actually deliver. Reporting only one or the other can mislead.

## Where this fits in the harness arc

- **Read alongside**: [04-skills](04-skills.md), [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md), [19-voyager-skill-libraries](19-voyager-skill-libraries.md), [54-semaclaw-general-purpose-agent](54-semaclaw-general-purpose-agent.md), [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md), [71-karpathy-skills-single-file-guardrails](71-karpathy-skills-single-file-guardrails.md), [79-skill-rag](79-skill-rag.md), [84-swe-search-mcts](84-swe-search-mcts.md), [89-voyager-deep](89-voyager-deep.md), [97-qwen-prm](97-qwen-prm.md), [98-diversity-collapse-mas](98-diversity-collapse-mas.md), [154-ctx2skill-self-evolving-context-skills](154-ctx2skill-self-evolving-context-skills.md).
- **Test-time scaling**: [16-plan-and-solve](16-plan-and-solve.md), [17-rewoo](17-rewoo.md), [32-recurrent-depth-implicit-reasoning](32-recurrent-depth-implicit-reasoning.md), [51-rebalance-efficient-reasoning](51-rebalance-efficient-reasoning.md), [94-eagle3-spec-decoding](94-eagle3-spec-decoding.md).
- **RLVR family**: [31-glm-5-agentic-engineering](31-glm-5-agentic-engineering.md), [80-knowrl](80-knowrl.md), [85-alphaevolve](85-alphaevolve.md), [97-qwen-prm](97-qwen-prm.md).
- **Multi-agent voting and ensembling**: [98-diversity-collapse-mas](98-diversity-collapse-mas.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md).
- **Synthesis next**: [157-may-2026-synthesis](157-may-2026-synthesis-memory-and-skills.md).

## References

1. Wang et al. *HeavySkill: Heavy Thinking as the Inner Skill in Agentic Harness* — arXiv:2605.02396v1.
2. Code: https://github.com/wjn1996/HeavySkill.
3. Brown et al. 2024. *Large language monkeys: Scaling inference compute with repeated sampling* — arXiv:2407.21787. The Best-of-N baseline HeavySkill consistently surpasses.
4. Bai et al. 2025. *Kimi K2: open agentic intelligence* — CoRR abs/2507.20534. Frontier model with native heavy-thinking-style reasoning.
5. Guo et al. 2025. *DeepSeek-R1: incentivizing reasoning in LLMs through reinforcement learning*. The R1-Distill family used as primary open-weight backbones.
6. Yang et al. 2025a. *Qwen3 technical report* — CoRR abs/2505.09388. Qwen3-8B and Qwen3-32B used as backbones.
7. OpenAI 2025a. *gpt-oss-120b & gpt-oss-20b model card* — CoRR abs/2508.10925. GPT-OSS-20B baseline.
8. Sheng et al. 2025. *HybridFlow: A flexible and efficient RLHF framework* (VeRL) — EuroSys 2025. RLVR infrastructure used in Appendix B.
9. Zheng et al. 2025a. *Group Sequence Policy Optimization* (GSPO) — CoRR abs/2507.18071. The RL algorithm used.
10. Pan et al. 2025; Liu et al. 2024; Hsu et al. 2025; Wen et al. 2025; Yang et al. 2025b,c — parallel-reasoning architectures HeavySkill compares against (architectural modifications, scheduled merging, native parallel decoding).
11. Yao et al. 2023, *Tree of Thoughts* — NeurIPS 2023. The tree-search ancestor; HeavySkill is closer to flat parallel + deliberate.
12. Zhang et al. 2024a, 2024b — MCTS-based reasoning self-refinement; adjacent line of work.
13. StepFun-AI 2025, *PaCoRe: Learning to scale test-time compute with parallel coordinated reasoning*. Adjacent two-stage parallel reasoning framework.
14. [04-skills.md](04-skills.md), [15-tree-of-thoughts-lats.md](15-tree-of-thoughts-lats.md), [19-voyager-skill-libraries.md](19-voyager-skill-libraries.md), [68-atomic-skills-scaling-coding-agents.md](68-atomic-skills-scaling-coding-agents.md), [97-qwen-prm.md](97-qwen-prm.md), [98-diversity-collapse-mas.md](98-diversity-collapse-mas.md), [154-ctx2skill-self-evolving-context-skills.md](154-ctx2skill-self-evolving-context-skills.md) — adjacent harness-canon chapters.
