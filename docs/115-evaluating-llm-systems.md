# 115 — Evaluating LLM-Powered Systems: Offline, Online, Human, and the Eval Set That Decides Everything

**Sources.** Ozdemir, *Building Agentic AI*, Chapter 3 (AI Evaluation Plus Experimentation); Albada, *Building Applications with AI Agents*, Chapter 9 (Measurement and Validation); Lakshmanan & Hapke, *Generative AI Design Patterns*, Pattern 17 (LLM-as-Judge); plus the academic evaluation literature (HELM, MT-Bench, AlpacaEval, Chatbot Arena).

**One-line definition.** Evaluating LLM-powered systems splits into four orthogonal disciplines — *offline eval* (held-out test set), *online eval* (production traffic), *human eval* (preference labels), *trajectory eval* (multi-step agent runs) — each with its own metric class, error sources, and statistical machinery; the eval set you build is the *single most important artifact* of an LLM project, and most teams under-invest in it by an order of magnitude.

## Why this matters

The classic ML evaluation pattern — train/test split, accuracy/F1, train new model, compare — does not transfer cleanly to LLM systems. Three things break:
1. The "model" you're evaluating is often the prompt, not the weights; you're not retraining, you're rewording.
2. The output is open-text, not a class label; metrics require a judge.
3. The system is multi-step (retrieval, planning, tool use, synthesis), and final-answer accuracy obscures which step failed.

Without a deliberate evaluation discipline, "this prompt feels better" replaces evidence, and prompt-engineering effort is wasted on changes that don't compound. With a deliberate discipline, every change is measurable, regressions are caught, and improvements stack.

For agent builders, the eval set is the asset. Models change, prompts change, frameworks change; a well-curated eval set survives all of them and remains the source of truth for whether things got better.

## Problem it solves

Five common eval failures:

1. **No eval set.** Decisions made by reading sample outputs. No reproducibility.
2. **Tiny eval set.** 20 examples, hill-climbed to 100% accuracy in a day; tells you nothing about generalisation.
3. **Test set leakage into prompt examples.** Few-shot examples that overlap with the eval set inflate metrics.
4. **Single-metric tunnel vision.** Optimising for one metric (e.g. accuracy) while regressing on another (e.g. cost or latency).
5. **Step-level blindness.** Final-answer accuracy is the only metric tracked; when accuracy drops, no one knows which sub-step caused it.

A disciplined eval setup prevents each.

## Core idea in one paragraph

LLM evaluation is four jobs, run in parallel: **offline evaluation** measures performance on a held-out, curated set of representative inputs (your "test set"); **online evaluation** measures performance on real production traffic via metrics that don't require ground truth (latency, cost, refusal rate, user actions); **human evaluation** collects preference labels from real users or paid raters for quality dimensions metrics can't capture; **trajectory evaluation** scores not just final answers but the multi-step paths an agent took to reach them. Each job uses different metrics and different statistical tools. The first artifact you build is the eval set — 200–2000 representative inputs with ground truth or rubric scoring. Without it, no other eval works. With it, every model swap, prompt change, and harness modification becomes a measurable A/B comparison.

## Mechanism (step by step)

### 1. The eval set — your most important asset

Before you can evaluate anything, you need an eval set. Five rules:

- **Representative.** The distribution of the eval set should match the production distribution. Skewed eval sets give skewed metrics.
- **Stratified.** Cover edge cases, common cases, hard cases, and adversarial cases. A flat eval set lets one segment dominate the metric.
- **Versioned.** Tag the eval set with a version. Adding examples invalidates prior comparisons.
- **Held out.** Examples in the eval set must never appear in prompts, training, or fine-tuning data. Leakage destroys validity.
- **Sized 200–2000.** Smaller is hill-climbed; larger is expensive to score. The sweet spot depends on metric noise.

Building the first 200 is the highest-leverage week of the project.

### 2. Offline evaluation — held-out test set + metrics + judge

The standard offline pipeline:

```text
for example in eval_set:
    prediction = system(example.input)
    score      = scorer(prediction, example.ground_truth)
metrics = aggregate(scores)
```

The choice of `scorer` is the substantive decision:

- **Exact match.** For tasks with a single correct answer (numerical, classification).
- **F1 / BLEU / ROUGE.** Token-overlap metrics for summarisation/translation; correlate weakly with quality.
- **Embedding similarity.** Semantic similarity between prediction and reference; better than token overlap, still imperfect.
- **LLM-as-judge.** Another LLM rates the prediction against a rubric. Best correlation with human judgement; scale-friendly. See [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md).
- **Programmatic check.** Run the prediction's code; check tests pass. Best metric where applicable.
- **Constraint check.** Validate output against schema, required fields, format. Cheap and necessary.

For agent systems, **LLM-as-judge + programmatic checks** is the dominant pattern.

### 3. Online evaluation — production traffic, no ground truth

You cannot wait for offline eval; production traffic delivers signal continuously. Online metrics:

- **Latency** (p50, p95, p99). Plotted over time. Regressions show up here first.
- **Cost per query.** Token usage × price. Watch for prompt-length drift.
- **Refusal rate.** Fraction of queries the system refuses. Sudden spikes signal prompt or model regression.
- **User-action signals.** Did the user accept the answer? Click through? Ask a follow-up? Each signals satisfaction.
- **Error rate.** Tool failures, parse errors, timeouts.
- **Retention / engagement.** For products: did users come back?

Online metrics correlate weakly with quality but strongly with operational health. They catch regressions offline eval cannot see.

### 4. Human evaluation — when LLM-judges aren't enough

For dimensions metrics struggle with (creativity, helpfulness, harmlessness, factuality), human ratings remain the gold standard. Three patterns:

- **Internal team rating.** Cheapest; biased toward what the team values.
- **Paid rater pools.** Mid-cost; better calibrated. Tools: Surge, Scale, Mechanical Turk.
- **Real users via in-product feedback.** Best correlation with what matters; lowest signal density (most users don't rate).

Human eval should sample from the same eval set as offline. Disagreements between human ratings and LLM-judge ratings are diagnostic — they show where the judge is mis-calibrated.

### 5. Trajectory evaluation — for multi-step agents

Final-answer accuracy is necessary but not sufficient for agents. A correct answer reached via a wasteful trajectory is a problem. Three trajectory metrics:

- **Step efficiency.** Average steps to completion. Compare across versions.
- **Cost efficiency.** Average tokens / dollars per task. Correlates with step efficiency but not identical.
- **Trajectory quality.** Did the agent take a *reasonable* path? Use an LLM-judge over the full trajectory. See [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [38-claw-eval](38-claw-eval.md).

Trajectory eval catches regressions that final-answer accuracy misses: an agent that gets the right answer by trying every tool in random order is technically passing tests, practically broken.

### 6. Step-level eval — debugging the multi-step pipeline

When final-answer accuracy drops, you need to know *which step* failed. Step-level eval scores each subsystem independently:

- **Retrieval.** Hit rate at K, MRR. Did we fetch the right docs?
- **Planner.** Plan validity, completeness, ordering. Was the plan reasonable?
- **Executor.** Tool-call accuracy, parse-fail rate. Did we execute correctly?
- **Synthesiser.** Final-answer faithfulness given the trajectory.

Step-level eval is what makes regression debuggable. Without it, "accuracy dropped 5%" is just a panic trigger.

### 7. Statistical discipline — the part everyone skips

Once you have metrics, you need to know whether differences are real:

- **Confidence intervals.** Bootstrap a 95% CI on the metric. A 2% gain with overlapping CIs is noise.
- **Per-segment metrics.** Gains on average can hide losses on important segments.
- **Multiple-comparison correction.** When testing many prompt variants, Bonferroni or BH; otherwise you're guaranteed false positives.
- **A/A test.** Run the same system twice; the metric variance is your noise floor. Differences smaller than that are noise.

Most teams skip statistical discipline and over-react to small movements. The cost is wasted effort and brittle decisions.

## Empirical anchors

- **LLM-judge agreement with humans** is 70–90% on most rubric tasks; 95%+ with chain-of-thought judges.
- **A/A noise floor** is typically 1–2% on accuracy metrics; treat smaller differences as noise.
- **Test-set leakage** is the #1 source of inflated metrics in published agent papers.
- **200-example eval sets are statistically reasonable** for accuracy metrics with 95% CI; 50-example sets are not.
- **Online > offline correlation is imperfect.** Offline gains often shrink in production; user behaviour is the final arbiter.
- **Trajectory eval catches regressions** offline final-answer eval misses, especially in long-horizon tasks ([27-horizon-long-horizon-degradation](27-horizon-long-horizon-degradation.md)).

## Variants and counter-arguments addressed

- **"LLM-judge bias is fatal."** It's real but tractable. Use multiple judges, calibrate against human labels periodically, and audit disagreements.
- **"Just A/B test in production."** Not safe for high-stakes apps; offline eval first, online second.
- **"Evaluation slows me down."** It accelerates you. Without eval, decisions are speculation; with eval, every change is measurable.
- **"My task is too creative for metrics."** Use rubric-based human eval, not no eval.
- **"Models will be too good for evaluation soon."** Models will keep improving; the eval set will keep ratcheting (harder examples). The discipline is invariant.

## Failure modes and limitations

1. **Hill-climbing the eval set.** Repeatedly tuning on the same eval set leaks into the model; performance drops on truly held-out data. Hold a "vault" set you only touch quarterly.
2. **Goodhart's Law.** Optimising the metric until it stops measuring what you cared about. Track multiple metrics and read sample outputs regularly.
3. **Eval set staleness.** As production distribution shifts, the eval set drifts out of representativeness. Refresh quarterly.
4. **Judge model drift.** LLM judges shift as the underlying model changes. Re-calibrate periodically.
5. **Latency-only optimisation.** Cheaper, faster, but worse answers. Always pair latency with quality.
6. **Single-rater human eval.** One person's preferences are not the user base. Multi-rater agreement is the discipline.

## When to use, when not

**Always have at least offline + online.** Anything else is shipping blind.

**Add human eval for** quality dimensions metrics can't catch (creativity, tone, helpfulness).

**Add trajectory eval for** agent systems with multi-step execution.

**Add step-level eval when** you need to debug regressions in a multi-stage pipeline.

**Skip elaborate eval setups when** the task is one-shot and exploratory; minimum offline-only eval is still wise.

## Implications for harness engineering

- **Build the eval set before the prompt.** This is the most under-prioritised step. 200 representative examples > any prompt cleverness.
- **Version eval sets in git.** Treat eval data as code; tag, review, deprecate.
- **Run offline eval on every prompt change.** Pre-commit hook or CI pipeline. Block changes that regress without justification.
- **Plot online metrics on a dashboard.** Latency, cost, refusal rate, user actions per day. Catch drift.
- **Pair LLM-judge with human spot-checks.** Re-calibrate the judge against humans on a sample monthly.
- **Eval the eval.** A/A tests reveal noise floor; leakage audits catch contamination; per-segment breakouts catch hidden regressions.
- **Build trajectory eval for any agent.** Step efficiency, cost efficiency, trajectory quality. See [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md).
- **Tie eval into the deploy gate.** No prompt or model change ships without passing the eval bar. See [05-hooks](05-hooks.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md).

The one-sentence takeaway: **the eval set is the asset, four eval disciplines (offline, online, human, trajectory) run in parallel, and statistical discipline distinguishes real gains from noise.**

## See also

- [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md) — the canonical pattern for trajectory and judge-based evaluation.
- [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) — verifier loops as a special case of evaluation in the production loop.
- [24-observability-tracing](24-observability-tracing.md) — online metrics live here.
- [27-horizon-long-horizon-degradation](27-horizon-long-horizon-degradation.md) — long-horizon failure attribution depends on trajectory eval.
- [38-claw-eval](38-claw-eval.md), [96-gdpval](96-gdpval.md) — production-grade evaluation benchmarks for agents.
- [101-autoresearchbench](101-autoresearchbench.md) — modern eval benchmark for research agents.
- [111-prompt-engineering-as-discipline](111-prompt-engineering-as-discipline.md) — prompt optimisation depends on eval.
