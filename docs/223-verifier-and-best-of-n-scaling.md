# 223 — Verifier and Best-of-N Scaling: how PRMs turn samples into capability

**Papers.** Hunter Lightman, Vineet Kosaraju, Yura Burda, Harri Edwards, Bowen Baker, Teddy Lee, Jan Leike, John Schulman, Ilya Sutskever, Karl Cobbe — *Let's Verify Step by Step* — arXiv:2305.20050 — OpenAI — 2023. Karl Cobbe, Vineet Kosaraju, Mohammad Bavarian, Mark Chen, Heewoo Jun, Lukasz Kaiser, Matthias Plappert, Jerry Tworek, Jacob Hilton, Reiichiro Nakano, Christopher Hesse, John Schulman — *Training Verifiers to Solve Math Word Problems* — arXiv:2110.14168 — OpenAI — 2021. Bradley Brown, Jordan Juravsky, Ryan Ehrlich, Ronald Clark, Quoc V. Le, Christopher Ré, Azalia Mirhoshei — *Large Language Monkeys: Scaling Inference Compute with Repeated Sampling* — arXiv:2407.21787 — Stanford / Together AI — 2024. Companion: Qwen2.5-Math-PRM ([97](97-qwen-prm.md)), Math-Shepherd (Wang et al. 2024), DeepSeek-Prover-V1.5.

**One-line definition.** A line of work showing that **inference accuracy of an LLM scales as a clean curve in the number of samples N**, with **process-supervised verifiers (PRMs) extracting more capability per sample than outcome-supervised verifiers (ORMs) or majority vote** — Lightman 2023's PRM800K-trained PRM achieves **78.2 % on MATH at BoN=1860 vs ~72 % for ORM and ~67 % for majority vote**, and Brown 2024 ("Large Language Monkeys") generalizes the effect: pass@N keeps rising as `1 − (1 − p_1)^N`-style curves up to enormous N, **provided the verifier is good enough to pick the winner**.

## Why this paper matters (the verifier is the harness component that turns raw model samples into usable capability, and its quality dominates inference-compute ROI)

The Snell-2024 test-time-compute story ([217](217-test-time-compute-scaling.md)) and the agent-trajectory story ([237](237-agent-trajectory-scaling.md)) both *depend* on a verifier — you can sample widely, branch widely, revise sequentially, but **without a verifier you cannot pick the right answer from the pool**. Lightman-2023 is the canonical paper establishing that **process-supervised reward models (PRMs)** — verifiers that score each *step* of a solution — extract more capability from a fixed inference budget than outcome reward models (ORMs) — verifiers that only score final answers — at MATH-class problems. The headline result reframed the whole field: 78.2 % on MATH at BoN=1860 with a PRM versus 72.4 % with an ORM trained on the same trajectories and 67.6 % with majority vote. The PRM is "free" capability if you have the right training data.

Cobbe-2021 was the precedent: training a GSM8K verifier on (problem, solution, label) tuples and using it for BoN gave a 7B GPT-3 verifier-augmented pipeline that beat a 170B vanilla model on GSM8K. The general lesson — *verifier-augmented inference makes a small model behave like a large one* — has held up across three years of follow-on work. Brown-2024's "Large Language Monkeys" pushes the idea further: take a small open-weights model, sample 10 000 times, pick the right one with a verifier, and watch pass@N climb past flagship-model pass@1 across MATH, GSM8K, MiniF2F, and SWE-bench Lite. This is not just "more samples is better" — it is the empirical verification of the **inference-time scaling law for verifier-augmented sampling**.

The Qwen2.5-Math-PRM paper ([97](97-qwen-prm.md)) extends the line with the practitioner's playbook: **PRMs trained from MC rollouts alone are biased toward outcome correctness**, **LLM-as-judge labels alone are noisy**, and **the consensus of MC + judge** filtered down to ~40 % of the data is the right training set. The release Qwen2.5-Math-PRM-72B reaches **ProcessBench mean F1 73.5** versus o1-mini 87.9 and GPT-4o 61.9; downstream BoN gains transfer.

Take this evidence seriously and three things change. **First**, you stop treating the verifier as a "scoring head" and treat it as a *first-class trained model* with its own data pipeline, eval suite (ProcessBench, BoN curves), and Chinchilla-shaped training budget. **Second**, you architect inference to **scale samples + verifier together** — adding samples without verifier improvements is wasted compute; adding verifier improvements without samples is wasted potential. **Third**, you adopt **process supervision over outcome supervision** wherever the task admits step-by-step decomposition, because the per-FLOP capability extraction is materially higher.

## Problem it solves (turn an N-sample distribution over the model's competence into a usable single answer)

1. **Outcome rewards are too coarse.** A model can produce a correct final answer via wrong reasoning ("right for the wrong reason"); ORMs reward this and propagate the noise. PRMs penalize the bad steps, biasing learning toward valid derivations.
2. **Majority vote ignores quality.** On problems where the modal answer is wrong (because most samples take a common wrong path), majority vote fails. A verifier that looks at *content* rather than *frequency* recovers correctness.
3. **Best-of-N requires a comparator.** Without a verifier, BoN collapses to "the first one we tried"; with a verifier, it becomes "the most defensible." The verifier IS the BoN mechanism.
4. **PRM training data is hard.** PRM800K (Lightman 2023) was hand-labelled at step granularity by humans and cost OpenAI tens of thousands of annotation hours. Math-Shepherd (Wang 2024) and Qwen-PRM ([97](97-qwen-prm.md)) replaced this with MC-rollout estimation; quality depends on rollout-policy quality.
5. **PRMs over-fit to their training distribution.** A MATH-trained PRM transfers poorly to AIME-style problems, GSM8K, or out-of-distribution math; production needs domain-specific PRMs or composable critics.
6. **Inference-compute scaling needs a sample-vs-cost curve.** Without one, you do not know whether to spend a marginal dollar on more pretraining, more samples per query, or a better verifier. Brown 2024 fits explicit pass@N curves that let you compare.

## Core idea in one paragraph

A **verifier** is a model V that takes a (problem, candidate solution) pair and returns a score, ideally calibrated to "probability the solution is correct." Given N candidate solutions sampled from a generator G, the verifier-augmented prediction is `argmax_{s ∈ samples} V(problem, s)` (or weighted aggregation). For a fixed G, the accuracy of this pipeline scales smoothly with N: `acc(N) ≈ 1 − (1 − p_correct(G, V))^k(N)` with k determined by sample diversity and verifier discrimination. Three regimes matter: (a) **majority vote** — V degenerates to "most common answer," which fails when the modal answer is wrong; (b) **outcome reward model (ORM)** — V is a final-answer classifier trained on (problem, full-solution, label) data, and reaches plateau as N grows because it cannot distinguish "right for wrong reason" from "right for right reason"; (c) **process reward model (PRM)** — V scores each step and aggregates (product / min / last-step), trained on PRM800K-style step labels, and continues to lift accuracy at large N because it can punish reasoning errors even when final answers happen to match. Lightman 2023 demonstrated PRM > ORM > majority vote at MATH; Brown 2024 generalized to arbitrary tasks where a verifier can be defined; Qwen-PRM ([97](97-qwen-prm.md)) provided the practitioner's recipe (MC + judge consensus filter, hard labels at scale). The compute-optimal allocation between (samples, verifier-investment) is a frontier; current best practice on agentic stacks is to invest in PRM quality first and N second.

## Mechanism (step by step)

### (a) Outcome reward model (ORM) — the baseline

Train a binary classifier on (problem, full-solution, correct?) tuples. Architecture: replace the LM head of the base model with a scalar; logistic regression on the final hidden state. Loss: cross-entropy on terminal label. At inference: score each of N samples, return argmax.

**Limit:** the ORM cannot tell *why* a solution is wrong; if a wrong-reasoning-right-answer solution is in the pool, ORM may rank it above a correct solution.

### (b) Process reward model (PRM) — Lightman-2023's contribution

Train a per-step classifier on (problem, partial solution up to step k, step k correct?) tuples. Architecture: same as ORM but scoring is *per-step* — produce a score for each "\n\n"-delimited step. Aggregation:

- **Product:** ∏ V(step_k) — Lightman's recommended.
- **Min:** min_k V(step_k) — punishes the worst step (Math-Shepherd preferred).
- **Last-step:** V(step_K) — for MC-trained PRMs, surprisingly often best (Qwen-PRM finding).

PRM training data:

- **Human-labelled:** PRM800K (Lightman): ~800K step-level labels by domain experts; gold standard but ~800K × $$/label is expensive.
- **MC-rollout labelled:** Math-Shepherd, Qwen-PRM ([97](97-qwen-prm.md)) — for each step, do K continuation rollouts and label the step by whether ≥1 reach the gold answer; cheap but biased toward outcome.
- **LLM-as-judge labelled:** A strong instruct model assigns step labels via prompted critique; cheap, noisier; Qwen-PRM showed it is *complementary* to MC, not a replacement.
- **Consensus filter (Qwen-PRM recipe):** Keep only (problem, solution) pairs where MC and judge agree on the *index* of the first wrong step; reduces ~860K MC labels to ~350K consensus labels; trained PRM matches full-judge ProcessBench F1 with 40 % of the data.

### (c) Best-of-N with PRM scoring

For each problem:

- Sample N candidate solutions from G.
- Score each with the PRM.
- Aggregate (sum-of-step-scores, product, min, last-step — ablate per task).
- Return the argmax candidate.

Lightman §4 reports BoN curves (PRM vs ORM vs majority vote vs pass@N upper bound) on MATH:

- **N = 4:** PRM ~62 %, ORM ~58 %, MV ~52 %, pass@4 ~73 %.
- **N = 64:** PRM ~72 %, ORM ~67 %, MV ~62 %, pass@64 ~83 %.
- **N = 1860:** PRM **78.2 %**, ORM ~72 %, MV ~67 %, pass@1860 ~85 %.

The PRM closes the gap to pass@N (the oracle-verifier upper bound) more than any other method.

### (d) Beam search and lookahead with PRM (Snell-2024 extension)

Once you have a step-level PRM, you can replace BoN's "sample N independent rollouts" with **beam search**: at each generation step, keep the top-K beams ranked by per-step PRM. **Lookahead search** rolls out partial continuations from each candidate to estimate the value, à la MCTS rollouts. These methods are FLOP-efficient relative to BoN at hard problems — you spend compute on promising branches rather than all N independent draws. See [217-test-time-compute-scaling](217-test-time-compute-scaling.md), [p18-tree-search-thinking](../projects/polaris/docs/research/p18-tree-search-thinking.md), [84-swe-search-mcts](84-swe-search-mcts.md).

### (e) "Large Language Monkeys" — pass@N at extreme N

Brown 2024 sampled up to 10 000 times from open-weights models on MATH, GSM8K, MiniF2F, SWE-bench Lite. Headline: pass@10000 with a Llama-3-8B base + verifier-selected answer **beats GPT-4 pass@1 on MATH and GSM8K**. The shape of the curve is a smooth power-law-with-floor; the position depends on the model and task. Caveat: pass@N is the *oracle-verifier upper bound* (any sample correct counts) — to *realize* it requires a near-oracle verifier. The paper acknowledges this and treats verifier quality as the rate-limiter on closing the pass@1 → pass@N gap.

### (f) ProcessBench and the verifier eval problem

Zheng et al. (2024) introduced **ProcessBench** as the first benchmark for *step-level error identification*: given a (problem, solution, gold-step-where-error-occurs) tuple, can the verifier identify the index of the first wrong step? This decouples PRM evaluation from BoN — a PRM might help BoN even if it is bad at process ID, or vice versa ([97-qwen-prm](97-qwen-prm.md) shows this inversion). Production PRMs should track *both* prm@N and ProcessBench F1.

### (g) Online RLHF with verifier reward

[80-knowrl](80-knowrl.md), DeepSeek-R1, RLHF on reasoning traces — the verifier is *also* the RL reward signal. The same PRM that selects BoN at inference becomes the reward model for RL fine-tuning; the loop closes.

## Empirical results (table)

**Table 1 — Lightman 2023 §4 BoN curves on MATH (12.5K test set, GPT-4-base sampler)**

| Method | N=4 | N=64 | N=256 | N=1860 |
|---|---:|---:|---:|---:|
| Majority vote | ~52 % | ~62 % | ~65 % | ~67 % |
| ORM | ~58 % | ~67 % | ~70 % | ~72 % |
| PRM | ~62 % | ~72 % | ~75 % | **78.2 %** |
| Pass@N (oracle) | ~73 % | ~83 % | ~84 % | ~85 % |

**Table 2 — Brown 2024 "Monkeys" pass@N on MATH (Llama-3-8B sampler, oracle verifier)**

| N | pass@N |
|---:|---:|
| 1 | ~32 % |
| 10 | ~57 % |
| 100 | ~74 % |
| 1000 | ~85 % |
| 10000 | **~90 %** (matches GPT-4 pass@1 of ~84 % at much smaller per-sample cost) |

**Table 3 — Qwen2.5-Math-PRM (97) ProcessBench F1 vs prm@8 (math benchmarks avg)**

| PRM | ProcessBench mean F1 | prm@8 mean |
|---|---:|---:|
| Math-Shepherd 440K | 28.9 | 64.3 |
| MC 860K (ours) | 40.1 | **65.9** |
| Human PRM800K | **56.5** | 64.9 |
| LLM-judge 860K | 46.5 | 65.3 |
| Qwen2.5-Math-PRM-7B (consensus + filter) | 73.5 | 67.6 |
| Qwen2.5-Math-PRM-72B | 78.6 | **69.3** |

**Table 4 — Cobbe 2021 GSM8K: verifier-augmented small model vs vanilla large**

| Setup | GSM8K accuracy |
|---|---:|
| 6B GPT-3 vanilla | 6.9 % |
| 175B GPT-3 vanilla | 33.0 % |
| 6B GPT-3 + verifier-BoN-100 | 55 % |
| 175B GPT-3 + verifier-BoN-100 | **78 %** |

The 6B + verifier closes most of the gap to the 175B vanilla; the 175B + verifier outperforms both.

## Variants and ablations

- **Hard vs soft labels.** Hard labels (binary correct/wrong) work best at scale once consensus filter is applied (Qwen-PRM). Soft labels (continuous MC-success-rate) are noisier but simpler.
- **Aggregation: product / min / last-step.** Product is Lightman's choice. Min is Math-Shepherd's. Last-step works best for *MC-trained* PRMs because they implicitly behave like ORMs ([97-qwen-prm](97-qwen-prm.md)). Pick aggregation matched to training semantics.
- **Generative verifiers.** Replace classifier head with a generative critique; the PRM produces a natural-language critique whose terminal binary embedding gives the score. Slower but more interpretable.
- **Self-verification.** [11-feynman-verifier](../projects/polaris/docs/research/p11-feynman-verifier.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md) — the base model verifies its own samples; cheap, lower ceiling than a trained PRM.
- **Cross-model adversarial pair.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [19-verifier-cross-channel](../projects/polaris/docs/blocks/19-verifier-cross-channel.md) — verifier model family ≠ generator family. Reduces self-confirmation bias.
- **Domain-specific PRMs.** Math-PRM, Code-PRM, Theorem-PRM, Plan-PRM. Each trained on its own data; routed by task type.
- **Verifier ensembles.** Multiple PRMs (different training data, different architectures); combine via weighted vote. Closes pass@N gap further.
- **Online active learning.** Sample queries where verifier confidence is low; collect step labels; retrain verifier. PRM800K used active learning explicitly.
- **Reward-model RLHF / DPO loop.** Use the PRM as RL reward to fine-tune the generator; capability moves into the model itself.

## Failure modes and limitations

- **PRM bias from MC labels.** [97-qwen-prm](97-qwen-prm.md) shows MC-trained PRMs implicitly score "value to gold answer" not "step correctness"; mass concentrates on the final-answer step. Mitigate with judge-consensus filter.
- **Reward hacking.** Generators trained against a fixed PRM learn to game the PRM (produce solutions the PRM scores high but humans rate low). Periodic PRM re-training and adversarial holdouts are necessary.
- **Distribution shift.** A MATH-PRM applied to GSM8K-style word problems mis-ranks. PRMs are domain-bound.
- **Noise in step segmentation.** "\n\n" splitting works for math but is brittle for code (multi-line statements), prose (variable paragraph length), or agent traces (tool-call boundaries). Need explicit step delimiters.
- **Verifier latency dominates BoN.** At N=1860, scoring 1860 candidate solutions costs more than the original generation. Mitigations: cheap pre-filter, scalable PRM serving, beam-search instead of BoN ([217](217-test-time-compute-scaling.md)).
- **Pass@N is oracle, not realized.** The Brown-2024 curves show pass@N keeps climbing; realized BoN-N stalls when the verifier's separation drops. Closing the gap is verifier-bound, not sampler-bound.
- **PRM eval is non-trivial.** ProcessBench measures step-id F1; prm@N measures BoN selection. They can disagree ([97](97-qwen-prm.md)). Track both.
- **Process supervision does not exist for many tasks.** Open-ended writing, conversation, deep research synthesis — there is no clean step-level correctness label. Process supervision applies where steps have ground truth.
- **PRM cost vs gain trade-off.** Training a 7B PRM costs comparably to fine-tuning a 7B generator. At what point is the PRM not worth it? Currently: it's worth it for any reasoning-heavy task with verifiable correctness.

## When to use, when not

**Use PRM-augmented BoN / beam / lookahead** for tasks with step-decomposable solutions and verifiable terminal correctness — math, formal proofs, code, structured planning, agent trajectories with clear subgoals. Use it whenever you have a strong PRM available or can train one (or compose one from cross-model adversarial pairs). Use it especially for the *medium-to-hard difficulty* bucket from [217](217-test-time-compute-scaling.md) where additional samples and verifier discrimination compound.

**Do not** use PRM-BoN when the task lacks step-level ground truth (creative writing, open-ended conversation), when latency is the binding constraint and N must be 1, when no PRM exists for the domain (and you cannot train one), when the PRM is uncalibrated against ProcessBench (you don't know what you're getting), or when the verifier is the same model as the generator (self-confirmation amplifies errors — see [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md)).

## Implications for harness engineering

- **Verifier is infrastructure, not an experiment.** Train, version, monitor, eval — the PRM is a load-bearing model with its own lifecycle. [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [19-verifier-cross-channel](../projects/polaris/docs/blocks/19-verifier-cross-channel.md), [97-qwen-prm](97-qwen-prm.md).
- **Cross-channel verifier mandatory.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md) — generator and verifier from different model families to break self-confirmation; this is a *bright-line* design rule.
- **Process > outcome supervision wherever applicable.** [80-knowrl](80-knowrl.md), [156-heavyskill-rlvr](156-heavyskill-rlvr.md), [97-qwen-prm](97-qwen-prm.md). When you can label steps, label steps.
- **Track ProcessBench and prm@N together.** [11-feynman-verifier](../projects/polaris/docs/research/p11-feynman-verifier.md) — disagreement reveals process-vs-outcome shift; if your PRM is great at BoN but bad at ProcessBench, you have an ORM in disguise.
- **Allocate inference compute as (N, verifier-quality) pair.** [217-test-time-compute-scaling](217-test-time-compute-scaling.md), [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md) — a doubling of N without verifier improvements often beats a doubling of verifier-quality without N, but the converse is true past a saturation point.
- **Use beam search and lookahead, not just BoN, on hard problems.** [p18-tree-search-thinking](../projects/polaris/docs/research/p18-tree-search-thinking.md), [84-swe-search-mcts](84-swe-search-mcts.md) — FLOP-efficient verifier-guided search dominates BoN at the hardest bucket.
- **Verifier-as-reward closes the loop to RL.** [80-knowrl](80-knowrl.md), DeepSeek-R1 — PRM at inference becomes RL reward at training; the same investment serves both phases.
- **Active-learning loops for PRM data.** [10-skill-auto-creator](../projects/polaris/docs/blocks/10-skill-auto-creator.md), [p9-ctx2skill-self-play](../projects/polaris/docs/research/p9-ctx2skill-self-play.md) — collect step labels at the verifier's confidence frontier to compound improvement.
- **Verifier ensembles for sensitive bright-line gates.** [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — when refusal/escalation is the action, multiple verifiers and require-consensus.
- **Calibrate verifier confidence per domain.** [88-confidence-driven-router](88-confidence-driven-router.md) — a calibrated verifier supplies the difficulty signal for the test-time-compute router from [217](217-test-time-compute-scaling.md).
- **Cost-discipline on BoN budget.** [p13-cost-discipline](../projects/polaris/docs/research/p13-cost-discipline.md), [86-frugalgpt](86-frugalgpt.md) — N is a cost knob; tune to per-task accuracy/cost frontier.
- **HIR observability for verifier traces.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — log per-sample PRM scores, aggregation method, BoN selection rationale; without it you cannot debug verifier failures or detect reward hacking.

**One-line takeaway for harness designers.** **A verifier is the harness component that converts inference compute into capability — invest in PRM quality first (process supervision, cross-model adversarial pair, ProcessBench-tracked), scale N second, and treat the *(samples, verifier-quality)* pair as a single co-budgeted axis whose ROI dominates pretraining at the agent-era frontier.**
