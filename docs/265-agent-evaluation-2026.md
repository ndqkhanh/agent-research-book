# 265 — Agent Evaluation 2026: GAIA, OSWorld, GDPval, MAST, eval-driven development, and the production-eval feedback loop

**Anchors.** GAIA — *General AI Assistants benchmark* — arXiv:2311.12983 (deep-dive in [222](222-agent-trajectory-scaling.md), referenced extensively). OSWorld — arXiv:2404.07972 (deep-dive in [95](95-osworld.md)). GDPval — deep-dive in [96](96-gdpval.md). SWE-bench Verified — Princeton + OpenAI — production-tier subset of SWE-bench. AgentBench — arXiv:2308.03688. MultiAgentBench / MARBLE — arXiv:2503.01935 ([251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md)). MAST — arXiv:2503.13657 ([251](251-multi-agent-teams-2026-synthesis.md)). REALM-Bench — enterprise benchmark. AgentArch — enterprise architecture benchmark. Companions: [115-evaluating-llm-systems](115-evaluating-llm-systems.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md), [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md), [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md), [268-agent-operations-synthesis-2026](268-agent-operations-synthesis-2026.md).

**One-line definition.** A 2026 picture of **agent evaluation** that goes well beyond LLM-benchmarks — the field has **per-task verifiable benchmarks** (GAIA Levels 1-3, OSWorld, GDPval, SWE-bench Verified, AIME, AssistantBench), **multi-agent failure taxonomies** (MAST's 14-mode 1,600-trace audit; MultiAgentBench's milestone KPIs), **production eval pipelines** that integrate with observability ([264](264-agent-observability-stack-2026.md)), **eval-driven development (EDD)** as a discipline that mirrors test-driven development, and **statistical methodology** (variance, confidence intervals, paired comparisons, Bonferroni correction) imported from data science — collectively making "is this agent better than yesterday's" a question with rigorous answers, not anecdotes.

## Why this paper matters (eval is the load-bearing discipline; without it, agent quality regresses silently)

Karpathy's *verifiability-as-bottleneck* point ([242](242-verifiability-as-bottleneck.md), if present in corpus) is exactly right: **per-task verifier density is the load-bearing axis** in 2026 agent engineering. RL training requires verifiable rewards; Best-of-N selection requires a verifier; loop termination requires a "done" signal; CI for agents requires regression suites; production observability without eval-link annotations is half a feedback loop. The 2026 evaluation landscape gives the field the verifier density it needs across (a) **canonical benchmarks** for capability claims, (b) **eval-driven development** as a daily discipline, and (c) **production-eval feedback loops** that close the operations cycle.

The benchmark landscape consolidated. **GAIA** (arXiv:2311.12983) — three difficulty levels (5-min / 5-min-with-tools / 30-min-with-tools-multi-step) — is the de-facto "general assistant" benchmark; frontier scores moved from ~30% (early 2024) to ~65% (2026 frontier). **OSWorld** ([95](95-osworld.md)) — multimodal real computer environments — is the production-OS-agent benchmark; scores from ~12% to ~38%. **GDPval** ([96](96-gdpval.md)) — economically-meaningful work tasks — bridges to ROI questions. **SWE-bench Verified** is the canonical software-engineering benchmark; scores from 12.5% to 60%+ in two years ([222-agent-trajectory-scaling](222-agent-trajectory-scaling.md)). **MultiAgentBench / MARBLE** (arXiv:2503.01935) is the multi-agent benchmark; **MAST** (arXiv:2503.13657) the failure taxonomy. **AgentBench**, **AssistantBench**, **WebArena**, **AppBench**, **AIME-2024/2025** for math reasoning, **MMLU-Pro** for knowledge — the canon is rich.

What's new in 2026 is **eval-driven development (EDD)** — the discipline of writing eval cases first, building agents to pass them, and treating eval suites as living regression tests. The pattern mirrors test-driven development from the 2000s software-engineering era and is rapidly becoming the production-quality discipline for agent teams. Combined with **production-eval feedback loops** (production traces feed back into eval datasets via [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md)'s eval-link annotations) and **statistical methodology** (variance, confidence intervals, paired comparisons), eval becomes a continuous-feedback engineering practice rather than a periodic snapshot.

Take this seriously and three things change. **First**, **EDD becomes your daily practice** — write eval cases before features; CI runs evals on every PR; regressions block merge. **Second**, you adopt a **production-eval feedback loop** — production traces with eval-link annotations flow back into the eval suite; the eval suite grows as production matures. **Third**, you use **statistical methodology** — A/B tests with proper sample sizing, paired comparisons for model swaps, Bonferroni correction for multiple-comparison families — making "is X better than Y" a quantitative question rather than vibes-based.

## Problem it solves (rigorous quality measurement for agents)

1. **Vibes-based quality assessment is unreliable.** Without eval, "the agent feels better today" is the only signal; quality regresses silently.
2. **LLM-benchmarks don't measure agents.** MMLU + HumanEval + MATH measure base-model capability; agents need agent-task benchmarks (GAIA, OSWorld, SWE-bench).
3. **Multi-agent quality has its own taxonomy.** MAST's 14-mode failure taxonomy + MultiAgentBench give multi-agent specific eval primitives.
4. **CI for agents was absent.** Eval pipelines plug into CI/CD; PRs that regress evals block merge.
5. **Production traces were not eval data.** Eval-link annotations on observability spans close the loop ([264](264-agent-observability-stack-2026.md)).
6. **A/B testing was ad-hoc.** Statistical methodology gives proper sample sizing and confidence intervals.
7. **LLM-as-judge calibration matters.** Judge quality affects conclusions; calibration suites + cross-judge agreement become standard.
8. **Drift detection.** Agent quality drifts as models update or prompts change; production-eval detects it.

## Core idea in one paragraph

Agent evaluation in 2026 has four pillars. **Canonical benchmarks** — GAIA (three levels), OSWorld, GDPval, SWE-bench Verified, AgentBench, MultiAgentBench, MAST — provide standardized capability and failure-mode measurements; new agents publish results on the relevant subset. **Eval-driven development (EDD)** — the discipline of writing eval cases first, building agents to pass them, treating evals as regression suites; CI runs evals on every PR; regressions block merge. **Production-eval feedback loop** — production traces with eval-link annotations flow back into eval datasets ([264-agent-observability-stack-2026](264-agent-observability-stack-2026.md)); LLM-as-judge calibrated against human gold; the eval suite grows as production matures. **Statistical methodology** — proper sample sizing (power analysis), confidence intervals on benchmark scores, paired comparisons for model swaps (DA = difference-in-accuracy + paired t-test), Bonferroni or false-discovery-rate correction for multi-comparison families, MAST-cluster-rate breakdown alongside accuracy for multi-agent systems. The verifier hierarchy is **hard-rule > simulator > PRM > LLM-judge > human spot-check** — each level cheaper but noisier than the next; production pipelines combine them. The 2026 production-quality discipline for agents is a continuous-feedback engineering practice that mirrors test-driven development from the 2000s software era; it's the difference between "agents that work in demos" and "agents that ship to production with quality guarantees."

## Mechanism (step by step)

### (a) The benchmark landscape — what to publish on

| Benchmark | Domain | Format | Frontier 2026 |
|---|---|---|---:|
| **GAIA Lvl-1** | General assistant, 5-min | Tool-use Q&A | ~85% |
| **GAIA Lvl-2** | General, 5-min + tools | Multi-step | ~65% |
| **GAIA Lvl-3** | General, 30-min, complex | Long horizon | ~50% |
| **OSWorld** | Real-OS multimodal | Visual + tool use | ~38% |
| **GDPval** | Economic work tasks | ROI-aligned | varies |
| **SWE-bench Verified** | Software engineering | Code patches | 60%+ |
| **AgentBench** | Multi-domain agent | 8 environments | varies |
| **MultiAgentBench (MARBLE)** | Multi-agent collaboration | Milestone KPIs | varies |
| **MAST** | Multi-agent failure taxonomy | 14-mode classification | 41–86.7% failure rates |
| **AIME 2024 / 2025** | Math reasoning | Olympiad problems | 79.8% (R1), 83.3% (o1), 89% (o3) |
| **AssistantBench** | Web-based assistant | Real-web tasks | varies |
| **WebArena** | Web environment | Action-traces | ~40% |
| **REALM-Bench** | Enterprise scenarios | Realistic workloads | private |
| **MMLU-Pro** | Knowledge | Knowledge Q&A | 80%+ |

Publish on the subset relevant to your agent's domain.

### (b) Eval-driven development (EDD)

```
1. Write eval case before feature.
2. Run agent against eval; expect failure.
3. Implement feature to pass eval.
4. Verify pass.
5. Add to regression suite.
6. CI runs full suite on every PR.
7. Regressions block merge.
8. Production traces feed new cases into suite.
```

The mirror of TDD. Frameworks: LangSmith Datasets + Evaluators, Phoenix Datasets, deepeval, ragas, promptfoo.

### (c) The verifier hierarchy

| Verifier kind | Cost / call | Reliability | Use case |
|---|---|---|---|
| **Hard-rule check** | $0.001 | Deterministic | Math correctness, code-test-pass, format compliance |
| **Simulator / sandbox** | $0.01 | High | Code execution result, environment behavior |
| **PRM (process reward model)** | $0.01–0.10 | High when calibrated | Step-level correctness ([97-qwen-prm](97-qwen-prm.md)) |
| **LLM-as-judge** | $0.01–0.10 | Variable | Open-ended quality, factuality, helpfulness |
| **Human spot-check** | $0.10+ | Gold standard | Calibration, escalation, trust ground-truth |

Production pipelines combine: hard-rule first, simulator next, PRM/judge for ambiguous cases, human for calibration.

### (d) LLM-as-judge calibration

```python
# Judge calibration suite
calibration_set = [
    (input, output_a, output_b, human_preferred="a"),
    (input, output_a, output_b, human_preferred="b"),
    ...
]

judge_outputs = [judge(case.input, case.output_a, case.output_b) for case in calibration_set]
agreement_rate = sum(j == c.human_preferred for j, c in zip(judge_outputs, calibration_set)) / len(calibration_set)

# Use multiple judges; majority vote; check inter-judge agreement (Cohen's kappa)
```

Calibration suites of 200-500 human-labeled cases per domain. Re-calibrate when judge model changes.

### (e) Production-eval feedback loop

```
Production span emits eval-link annotation:
  span.set_attribute("eval.verdict", "potentially-incorrect")
  span.set_attribute("eval.judge_confidence", 0.65)

Nightly job:
  1. Pull production spans with eval.judge_confidence < 0.8
  2. Sample for human review
  3. Add reviewed cases to eval dataset
  4. Re-run agent on dataset; track aggregate quality

Result: eval suite grows ~10-50 cases per week from production.
```

### (f) Statistical methodology

**Sample sizing for A/B test** (two-proportion z-test):

```python
from statsmodels.stats.power import zt_ind_solve_power
n = zt_ind_solve_power(effect_size=0.05, alpha=0.05, power=0.80)
# → ~600 per arm to detect 5pp absolute difference
```

**Paired comparison** (model swap):

```python
# Same eval cases run with model A and model B
deltas = [acc_b[i] - acc_a[i] for i in range(N)]
from scipy.stats import ttest_rel
t, p = ttest_rel(acc_b, acc_a)
# Effect size: Cohen's d_z
```

**Multi-comparison correction**:

```python
# Comparing 5 model variants → 10 pairwise tests → Bonferroni: alpha = 0.05/10 = 0.005
# Or FDR via Benjamini-Hochberg
```

**MAST-cluster-rate reporting** (multi-agent):

```
System X:
  Cluster 1 (system design): 12% (CI: 9-15%)
  Cluster 2 (inter-agent): 28% (CI: 24-32%)
  Cluster 3 (verification): 8% (CI: 6-11%)
  Total failure rate: 48% (CI: 44-52%)
```

### (g) Drift detection

Production-eval continuous monitoring; compare per-week aggregate quality metrics; statistical-process-control charts; alert on shifts beyond control limits ([267-agent-sre](267-agent-sre.md)).

### (h) CI integration

```yaml
# .github/workflows/agent-evals.yml
name: Agent Evals
on: [pull_request]
jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -e .
      - run: python -m harness_core.evals.run --suite=regression --max-budget-usd=10
      - run: python -m harness_core.evals.compare --baseline=main --threshold=-0.02
```

`compare` step blocks the PR if eval scores regressed > 2pp absolute on any subset.

### (i) Eval datasets and frameworks

- **LangSmith Datasets + Evaluators** — first-class for LangGraph apps.
- **Arize Phoenix Datasets** — OSS, OTel-native.
- **deepeval** — pytest-shaped LLM evaluations.
- **ragas** — RAG-specific evals.
- **promptfoo** — prompt-eval CLI.
- **Inspect AI** (UK AISI) — comprehensive agent eval framework.
- **OpenAI Evals** — OpenAI's eval framework.
- **Helm Lite** (Stanford) — academic-grade reproducible evals.

## Empirical results (table — eval framework adoption May 2026)

| Framework | Best at | License |
|---|---|---|
| LangSmith | LangGraph apps | SaaS, free tier |
| Phoenix | OTel-native, OSS | Apache-2.0 |
| deepeval | pytest-shape | Apache-2.0 |
| ragas | RAG | Apache-2.0 |
| promptfoo | Prompt regression CLI | MIT |
| Inspect AI | Comprehensive agent eval | MIT |
| OpenAI Evals | Reference framework | MIT |
| Helm Lite | Reproducible academic | Apache-2.0 |

## Variants and ablations

- **Synthetic eval generation.** LLM-generated eval cases supplement human-curated.
- **Adversarial eval.** Red-team cases that probe known failure modes.
- **Cost-budget evals.** Run eval suite under $X budget; fail if exceeded.
- **Latency budget evals.** P99 latency must stay under threshold.
- **Verifier-of-verifier.** Calibrate the LLM-judge against human spot-checks.
- **Multi-judge consensus.** Bagging of judges; reduces noise.
- **MAST-aware multi-agent eval.** Report per-cluster failure rates.
- **Production replay.** Save production traces; replay against new model/prompt; compare quality.

## Failure modes and limitations

- **Eval-set contamination.** Public benchmarks leak into pretraining; must use held-out + private subsets.
- **Goodhart's law.** Optimizing for eval metric can degrade real-world quality; multi-axis evals + holdouts mitigate.
- **LLM-judge bias.** Judges have their own biases; calibration + multi-judge consensus help.
- **Variance.** Single-run scores noisy; report confidence intervals, run k=5+ trials.
- **Multi-comparison inflation.** Without correction, 10 comparisons at α=0.05 give ~40% chance of false positive.
- **Eval-cost spiral.** Comprehensive evals can cost $100s per run; sampling and partial suites required.
- **Production-eval feedback loop privacy.** Production traces may contain PII; redaction pipeline required.
- **Calibration drift.** As models update, judge calibration drifts; re-calibrate quarterly.
- **MAST taxonomy is opinionated.** Some failure modes don't fit cleanly; allow "other" category.
- **Sandbox effects.** Agent behavior in eval sandbox may differ from production environment.

## When to use, when not

**Adopt eval-driven development** for any production agent deployment, any multi-agent system (MAST is mandatory), any team larger than two people, any application where quality regression has user-visible consequences. The strongest cases are **regulated agents** (compliance auditors require eval evidence), **multi-team org agents** (consistent quality across teams), **paying-customer SaaS agents** (regressions = churn).

**Skip detailed eval** for personal-prototype agents (basic spot-checks suffice), exploratory research (eval suite premature), or single-shot tools (no production to regress). Skip cross-vendor benchmark comparisons when you don't ship public; use private holdouts only.

## Implications for harness engineering

- **`harness_core/evals/` package.** Shared evals across all 14 agents; CI integration.
- **Per-project eval suite.** [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md), [208-lyra-multi-hop-collaborative-apply-plan](208-lyra-multi-hop-collaborative-apply-plan.md), etc.
- **Production-eval feedback loop via observability.** [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md), [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — eval-link annotations.
- **Verifier hierarchy.** [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [97-qwen-prm](97-qwen-prm.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md), [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md).
- **Cross-channel verifier mandatory for production.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [19-verifier-cross-channel](../projects/polaris/docs/blocks/19-verifier-cross-channel.md).
- **MAST-aware multi-agent eval.** [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md), [98-diversity-collapse-mas](98-diversity-collapse-mas.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md).
- **Routine eval pipeline.** [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md) — scheduled eval-runs as routines.
- **Statistical-rigor in apply plans.** [211-cross-project-power-up-plan-with-tradeoffs](211-cross-project-power-up-plan-with-tradeoffs.md) — sample sizes, CIs, multi-comparison correction.
- **Drift detection in production.** [267-agent-sre](267-agent-sre.md) — alert on quality regression.
- **Cost-budget eval gates.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md).
- **Eval marketplace.** [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md) — eval suites as portable artifacts.
- **A/B testing infrastructure.** [88-confidence-driven-router](88-confidence-driven-router.md), [87-routellm](87-routellm.md) — proper traffic-splitting + statistical analysis.
- **Verifiability density per task.** [242-verifiability-as-bottleneck](242-verifiability-as-bottleneck.md) — eval pipeline density correlates with production-grade quality.
- **Eval-driven RL.** [80-knowrl](80-knowrl.md), [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md), [232-rl-on-reasoning-traces-scaling](232-rl-on-reasoning-traces-scaling.md) — eval signal becomes RL reward.

**One-line takeaway for harness designers.** **Agent evaluation in 2026 is a continuous-feedback engineering discipline — canonical benchmarks (GAIA, OSWorld, GDPval, SWE-bench Verified, MAST, MultiAgentBench), eval-driven development with CI integration, production-eval feedback loops via observability ([264](264-agent-observability-stack-2026.md)) eval-link annotations, and statistical methodology (sample sizing, CIs, paired comparisons, Bonferroni); the verifier hierarchy is hard-rule > simulator > PRM > LLM-judge > human, and the 2026 production-quality discipline mirrors TDD from the 2000s software era — adopt EDD, calibrate your judges, run regression suites in CI, and close the production-eval feedback loop.**
