# 21 — LLM-as-Judge & Trajectory Evaluation

**Definition.** LLM-as-Judge uses a language model to score the outputs of another model — producing nuanced, semantic judgments at scale where traditional metrics (exact match, BLEU, ROUGE) fail. Trajectory evaluation applies the same idea to full agent runs: scoring not just the final answer but the *path* (reasoning, tool calls, observations, error recovery) that produced it.

## Problem it solves

Agents don't have one right answer. They have a sequence of decisions that might all be defensible or might cascade into failure. Exact-match metrics over final outputs miss the difference between a correct answer reached cleanly and a correct answer reached after six wrong turns and a lucky recovery. Conversely, they penalize legitimate output variation (paraphrases, different-but-equivalent tool sequences).

LLM-judges can capture subjective dimensions — correctness, factuality, coherence, style, safety — at a cost and throughput humans can't match. For agents, trajectory evaluation surfaces the *process-level* failures (wasted tool calls, misread observations, bad decomposition) that final-answer metrics hide.

## Mechanism

### LLM-as-Judge

Three common shapes:

- **Direct scoring.** "Rate this response 1–5 on correctness."
- **Pairwise preference.** "Response A or B is better? Why?" Used for reward modeling (Chatbot Arena, RLHF).
- **Rubric-based.** Multi-criterion scoring with explicit definitions — often produces the best calibration.

Key practices:

1. **Calibration.** Judges need few-shot anchor examples showing what each score means. Without them, scores drift between runs.
2. **Structured output.** Force JSON with `{score, reasoning, failure_mode}` so results are machine-parseable.
3. **Temperature 0.** Deterministic judgments are auditable and reproducible.
4. **Bias audits.** Position bias (A before B is favored), verbosity bias (longer is favored), self-preference bias (model judges its own family higher). Mitigate by swapping, length-normalizing, using a different-family judge.
5. **Judge validation.** Periodically have humans rate a sample and correlate with judge scores. If correlation drops, retrain the rubric.

### Trajectory Evaluation

Scoring an agent run requires more than scoring its final message. Dimensions:

- **Task success.** Did the agent achieve the goal (objective test / assertion / side-effect check)?
- **Path efficiency.** Steps taken vs. minimum; redundant tool calls; dead-end branches.
- **Decision quality.** At each branch, was the chosen tool/action plausible given the observation?
- **Error recovery.** When a tool failed, did the agent recover sensibly?
- **Faithfulness.** Did the final answer accurately reflect observed evidence (no hallucinated citations)?

LangSmith and similar observability tools let you pipe every run through a configurable evaluator chain. Online evals run on production traffic samples; offline evals run on curated datasets during CI.

## Concrete pattern

Rubric-based judge for a coding-agent trajectory:

```json
{
  "task_success": {"score": 0 | 1, "evidence": "tests pass (pytest output)"},
  "path_efficiency": {
    "score": 1-5,
    "steps_taken": 42,
    "estimated_minimum": 12,
    "wasted_actions": ["3 redundant Grep calls in ./src/", "retried same failing test twice"]
  },
  "decision_quality": {
    "score": 1-5,
    "worst_decision": "chose to edit file X before reading it",
    "best_decision": "used Plan mode before destructive migration"
  },
  "faithfulness": {
    "score": 1-5,
    "hallucinated_claims": ["claimed to fix bug in `lib/foo.ts:42` but file wasn't touched"]
  },
  "overall": {"score": 1-5, "summary": "..."}
}
```

Eval pipeline in LangSmith-style pseudo-code:

```python
@evaluator
def trajectory_eval(run, dataset_example):
    prompt = build_judge_prompt(
        task=dataset_example.input,
        trajectory=run.trace,
        expected=dataset_example.expected_output,
        rubric=load_rubric("coding_agent_v3"),
    )
    return judge_model.invoke(prompt, temperature=0)
```

Run offline over a regression suite on every agent version bump; flag regressions on any dimension, not just task_success.

## Variants & related techniques

- **Pairwise Arena-style eval.** Two agent versions on the same task; judge picks winner. Great for relative ranking, less for absolute quality.
- **Verifier loops** ([11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md)) apply the same machinery at runtime: evaluator decides whether to accept or reject a step.
- **Ground-truth evals.** Tests, SQL query results, mathematical correctness checks. Prefer these over LLM judges whenever feasible.
- **τ-bench, SWE-bench Verified, GAIA, AgentBench** — standardized agent benchmarks.
- **Eugene Yan / Hamel Husain / Shreya Shankar** write extensively about eval rubric design.

## Failure modes & anti-patterns

- **Rubric drift.** Judge rubric evolves without versioning; scores aren't comparable across runs. Fix: version rubrics as code; pin to eval snapshots.
- **Self-evaluation.** Same model judges its own output. Fix: different family, or humans.
- **Overweight on style.** Judge rewards well-phrased wrong answers. Fix: rubric emphasizes factual criteria with evidence requirements.
- **Ignoring trajectory, scoring only final answer.** Misses the "lucky-correct-via-wrong-path" failures. Fix: trajectory-level dimensions; reward clean process.
- **Eval set leakage.** The eval prompts are in the training set of the model being evaluated. Fix: use held-out / novel evaluation data; rotate.
- **Overfitting to judge.** Optimizing the agent until judge scores are high but humans rate it the same or worse. Fix: periodic human recalibration; diversify judges.
- **Noisy small samples.** Declaring a "win" based on 10 examples. Fix: ensure enough samples for statistical significance; report confidence intervals.

## When to use (and when not to)

**Use** LLM-as-Judge / trajectory eval when:

- Traditional metrics don't capture what you care about (factuality, reasoning quality, process).
- You need to evaluate thousands of runs per week (humans can't scale).
- You can validate judge calibration against humans on a representative slice.
- You control enough of the pipeline to produce structured traces worth evaluating.

**Don't** use them when:

- Ground-truth metrics are available and sufficient.
- You can't afford to rerun judges on every update (they drift).
- Your judges aren't materially better than the agent you're evaluating.
- Stakes are so high that only human review will do.

## References

- Zheng et al., "Judging LLM-as-a-Judge", arXiv:2306.05685 — <https://arxiv.org/abs/2306.05685>
- LangSmith docs, Evaluation — <https://docs.smith.langchain.com/evaluation>
- Hamel Husain, "Creating a LLM-as-a-Judge That Drives Business Results" — <https://hamel.dev/blog/posts/llm-judge/>
- "Evaluation and Benchmarking of LLM Agents: A Survey", arXiv:2507.21504 — <https://arxiv.org/html/2507.21504v1>
- Shreya Shankar et al., "Who Validates the Validators? Aligning LLM-Assisted Evaluation of LLM Outputs with Human Preferences" — <https://arxiv.org/abs/2404.12272>
- Chip Huyen, "AI Engineering" (O'Reilly) — evaluation chapter.
