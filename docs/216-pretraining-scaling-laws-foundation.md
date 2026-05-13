# 216 — Pretraining Scaling Laws (Kaplan → Chinchilla): the foundation agents inherit

**Papers.** Jared Kaplan, Sam McCandlish, Tom Henighan, Tom B. Brown, Benjamin Chess, Rewon Child, Scott Gray, Alec Radford, Jeffrey Wu, Dario Amodei — *Scaling Laws for Neural Language Models* — arXiv:2001.08361 — OpenAI / Johns Hopkins — 2020. Jordan Hoffmann, Sebastian Borgeaud, Arthur Mensch, Elena Buchatskaya, Trevor Cai, Eliza Rutherford, Diego de Las Casas, Lisa Anne Hendricks, Johannes Welbl, Aidan Clark, Tom Hennigan, Eric Noland, Katie Millican, George van den Driessche, Bogdan Damoc, Aurelia Guy, Simon Osindero, Karen Simonyan, Erich Elsen, Jack W. Rae, Oriol Vinyals, Laurent Sifre — *Training Compute-Optimal Large Language Models* — arXiv:2203.15556 — DeepMind — 2022. Companion: Chinchilla 70B model (1.4 T tokens) outperforms Gopher 280B (300 B tokens) at less than half the params.

**One-line definition.** Two papers that pinned base-model capability to a closed-form law `L(N, D) ≈ E + A·N^-α + B·D^-β` and corrected the constant such that **for compute-optimal training, params and tokens should scale roughly equally** (≈ 20 tokens per parameter), overturning Kaplan's earlier "scale parameters ~3× faster than data" recipe and reshaping every frontier training run since 2022.

## Why this paper matters (the agent era inherits whatever base capability these laws produce; agents augment, they do not replace)

Every agent harness — Polaris, Lyra, argus, the SWE-agents, the deep-research agents — sits on top of a base model whose intrinsic capability is governed by these two papers. The agent loop, tool use, verifier, multi-agent debate, and skill memory all *amplify* what the base model knows; none of them can manufacture knowledge the pretraining curve did not buy. If you do not understand why a 70 B model trained on 1.4 T tokens dominates a 280 B model trained on 300 B tokens, you will misallocate the *one* budget item that dwarfs every harness decision combined: training compute.

Kaplan-2020 was the first paper to claim that test loss falls as a clean **power law** in **non-embedding parameters** (N), **dataset tokens** (D), and **training compute** (C), with empirical exponents `α_N ≈ 0.076`, `α_D ≈ 0.095`, `α_C ≈ 0.050`, and that **architectural details (depth, width, attention heads) within a wide band do not matter** at fixed parameter count. That paper's *quantitative* claim — that compute-optimal training should scale parameters about three times faster than tokens — drove the GPT-3 / Gopher / Megatron-NLG 530 B / Jurassic-1 178 B era of "throw parameters at it." It was empirically wrong by ~3× on the relative scaling exponent, and the correction was Chinchilla.

Hoffmann-2022 trained **over 400 models** spanning 70 M to 16 B parameters and 5 B to 400 B tokens, fit three separate methods (training-curve isoFLOP profile, fixed-FLOP frontier sweep, and parametric loss surface fit), and **all three converged on the same conclusion**: the compute-optimal allocation has roughly equal exponents on N and D, so each unit of additional FLOPs should be split roughly equally between bigger model and more data. The headline empirical demonstration: **Chinchilla 70B (1.4 T tokens) at the same compute as Gopher 280B (300 B tokens)** beats Gopher on **MMLU (67.6 % vs 60.0 %)**, **BIG-bench (65.1 % vs 54.4 %)**, **TriviaQA**, and the LM perplexity benchmarks DeepMind tracks; downstream finetune is then cheaper because the model is smaller.

Take these two papers seriously and three things change. **First**, you stop treating "the model" as a fixed axis the harness perturbs around — the harness's marginal capability gain is bounded by where the base model sits on the L(N, D) curve, and a 14× harness improvement on a 7 B model can be matched by a *minimum-effort* harness on a 70 B Chinchilla-trained model. **Second**, you stop reading parameter counts as capability ([84-swe-search-mcts](84-swe-search-mcts.md), [89-voyager-deep](89-voyager-deep.md)) and start reading **token budgets and FLOP allocation**: a "30 B" model trained on 30 B tokens is materially weaker than a "7 B" model trained on 2 T tokens. **Third**, you understand why the agent-era scaling story ([217](217-test-time-compute-scaling.md), [237](237-agent-trajectory-scaling.md), [223](223-verifier-and-best-of-n-scaling.md), [224](224-multi-agent-parallel-scaling.md)) is *orthogonal* to these laws, not a replacement: pretraining compute sets the floor, agent-era axes set the ceiling above that floor.

## Problem it solves (the parametric capability law, and the constant in front of it)

1. **No prior closed-form predicting test loss from compute.** Pre-2020 the field had qualitative "bigger is better" intuition but no quantitative relationship — every new GPT-2 / GPT-3 / T5 scale-up was a leap of faith. Kaplan turned it into `L(N) ≈ (8.8e13 / N)^0.076` (Kaplan §3 fits) — a prediction you could *budget* against.
2. **Architecture confounds capacity.** Kaplan §5 fixed this by holding non-embedding parameters constant and sweeping aspect ratios; loss varied by < 0.05 nats across feed-forward ratios from 0.5 to 8 and depth-to-width ratios across 4× — meaning that *N alone* is the right capacity coordinate, not "depth" or "heads."
3. **The wrong coupling between N and D in Kaplan.** Kaplan's compute-optimal recipe (`N* ∝ C^0.73`, `D* ∝ C^0.27`) said data growth should *lag* parameter growth by roughly 3:1. Hoffmann showed this came from Kaplan's training schedule choice (a fixed-LR cosine that did not reach the loss floor for small-N runs), and that re-fitting with proper training schedules gives `N* ∝ C^0.5`, `D* ∝ C^0.5` — equal exponents.
4. **Compute is the binding constraint, not parameters or data alone.** Both papers agree the relevant *budget* is FLOPs, and the question is how to spend a fixed FLOP budget. Kaplan's answer was wrong; Chinchilla's "20 tokens per parameter" is approximately right for the current architecture-data regime.
5. **Inference cost was previously not the optimization target.** Both papers optimize for *training loss at fixed training FLOPs*, ignoring inference. The Chinchilla recipe accidentally produces smaller, cheaper-to-serve models, which is why every frontier lab adopted it for open weights (Llama-2, Llama-3, Mistral, Qwen, Gemma) where inference cost dominates.
6. **Emergent capabilities looked unbudgetable.** The scaling-laws frame says emergent capabilities (in-context learning, instruction following, code) are smooth in **loss**; what looks discontinuous in *task accuracy* is a threshold function of a smoothly improving loss — implying you can *budget* for capability emergence by budgeting for loss reduction. (See also [109-emergent-capabilities-myth](109-emergent-capabilities-myth.md) if present, otherwise the Schaeffer/NeurIPS-2023 critique.)

## Core idea in one paragraph

The cross-entropy test loss of a transformer language model is well-fit by a **smooth power law** in three independent budget axes — **non-embedding parameters N**, **training tokens D**, and **training compute C** — with the bivariate form `L(N, D) ≈ E + A·N^-α + B·D^-β` (Hoffmann eq. 3, with `α ≈ 0.34`, `β ≈ 0.28`, `E ≈ 1.69` for the BPB units they fit). Architecture choices within a wide band do not affect the fit; the optimization budget is **C ≈ 6 · N · D** (Kaplan's well-known forward+backward FLOPs heuristic for transformers); and the **compute-optimal allocation** of a fixed FLOP budget over N and D is found by minimizing `L(N, D)` subject to `C = 6 · N · D`. Kaplan's incorrect 3:1 split came from training-schedule artefacts; Chinchilla's three independent fits all give a roughly **1:1 allocation** — about 20 tokens per parameter at the FLOP scales they probed (1e17 to 1e21 FLOPs). The headline check was a controlled comparison: train **Chinchilla 70 B** on 1.4 T tokens at ~5.76e23 FLOPs and **Gopher 280 B** on 300 B tokens at the *same* FLOPs; Chinchilla wins MMLU, BIG-bench, language modelling, and most downstream tasks while costing **4× less to serve**. The whole agentic stack — every harness in this docs corpus — is built on top of base models whose capability ceiling is set by where their training landed on this surface.

## Mechanism (step by step)

### (a) The Kaplan parametric form and exponents (2020)

Kaplan §3 fits three univariate power laws on WebText2 BPC loss, holding the other axes large:

- `L(N) = (N_c / N)^α_N` with `N_c ≈ 8.8e13`, `α_N ≈ 0.076`
- `L(D) = (D_c / D)^α_D` with `D_c ≈ 5.4e13`, `α_D ≈ 0.095`
- `L(C_min) = (C_min,c / C_min)^α_C` with `α_C ≈ 0.050` (using "minimum compute," i.e. the full forward+backward pass, ≈ 6ND)

Crucially, Kaplan fixes the *training schedule* such that small-N runs are not run to convergence — the cosine LR schedule reaches its terminal LR at a step count tuned for the *largest* model. This means small-N losses are *underestimated* (those models would have improved further with more steps), which biases the joint fit toward the conclusion that N is more important than D.

### (b) Kaplan's compute-optimal recipe (incorrect but historically dominant)

Combining the three univariate laws with `C = 6ND`, Kaplan derives:

- `N*(C) ∝ C^0.73`
- `D*(C) ∝ C^0.27`
- `B*(batch) ∝ C^0.24`

This recipe says: as you 10× your FLOPs, grow params by ~5.4× and data by ~1.9×. **GPT-3 175 B trained on 300 B tokens is the canonical Kaplan-recipe model**, and so are Gopher 280 B, Megatron-Turing NLG 530 B, and Jurassic-1 178 B. Each was undertrained by Chinchilla's standard.

### (c) Chinchilla's three independent fits all give exponent ≈ 0.5

Hoffmann §3 runs three methods and reports they converge:

- **Approach 1: Fixed-model training curves.** Train one model size to many checkpoints, read off the loss-vs-FLOPs Pareto frontier. Repeat for several sizes; fit the joint surface. → `a ≈ 0.50`, `b ≈ 0.50`.
- **Approach 2: IsoFLOP profiles.** For each of nine FLOP budgets (6e18 → 3e21), train ~10 models at varying N and D *but the same C*, and find the N that minimizes loss. The minima trace `N*(C) ∝ C^0.49`, `D*(C) ∝ C^0.51`.
- **Approach 3: Parametric joint fit `L(N, D) ≈ E + A·N^-α + B·D^-β`.** Fitting all training points (with proper training schedules to the loss floor) gives `α ≈ 0.34`, `β ≈ 0.28`, `E ≈ 1.69`. The compute-optimal exponent then comes out to `a ≈ α / (α + β) ≈ 0.55`, `b ≈ β / (α + β) ≈ 0.45` — close to but not exactly 0.5; nonetheless an order-of-magnitude correction from Kaplan's 0.73:0.27.

The recipe of record: **at compute-optimal, ratio D / N ≈ 20**. Frontier open-weights models trained 2023–2025 (Llama-2, Llama-3, Mistral, Qwen-2.5, Gemma-2) all train at D/N ≫ 20 (often 100–1000) — *deliberately overshooting Chinchilla* to push inference cost down further at the expense of training cost.

### (d) The Chinchilla 70 B controlled comparison

Hoffmann §4 trains Chinchilla 70 B on 1.4 T tokens (D/N = 20) at the *same* FLOPs as Gopher 280 B on 300 B tokens (D/N ≈ 1.07). Chinchilla wins:

- **MMLU 5-shot:** 67.6 % vs Gopher 60.0 %
- **BIG-bench 0-shot/few-shot avg:** 65.1 % vs 54.4 %
- **TriviaQA filtered 1-shot:** 73.8 % vs 64.9 %
- **C4 / Wikitext / LAMBADA / The Pile** all in Chinchilla's favour
- **Inference FLOPs:** ~4× less per token (250 B vs 1.7 T forward pass FLOPs at common context lengths)

This is the cleanest possible empirical falsification of Kaplan's recipe at FLOP-equal budgets.

### (e) Why architecture barely matters within a band

Kaplan §5 sweeps depth/width ratios, FFN ratios, attention head counts, and shows that within a roughly 4× window in any of these, the loss-at-fixed-N is constant to within 0.05 nats. **Implication:** capacity is a function of `N_non_embedding`; harness-level "improve the architecture" interventions inside that band have negligible effect compared to moving along the L(N, D) curve.

### (f) FLOPs accounting: the 6ND identity

For a transformer with N non-embedding parameters and D training tokens:

- forward pass: ~2ND FLOPs (each parameter participates once in a matmul, once with itself)
- backward pass: ~4ND FLOPs (twice the forward — gradient w.r.t. activations + gradient w.r.t. weights)
- **total: C ≈ 6ND** (Kaplan §2.1)

This is the constant every scaling-law analysis uses. It is approximate (ignores attention quadratic in seq len for short contexts; correct for long contexts only when N includes the FFN dominant term), but it is the field's coordinate system.

## Empirical results (table)

**Table 1 — Chinchilla 70 B vs Gopher 280 B at equal FLOPs (~5.76e23)**

| Benchmark | Gopher 280 B (300 B tokens) | Chinchilla 70 B (1.4 T tokens) | Δ |
|---|---:|---:|---:|
| MMLU 5-shot | 60.0 % | 67.6 % | **+7.6** |
| BIG-bench (avg) | 54.4 % | 65.1 % | **+10.7** |
| TriviaQA filtered 1-shot | 64.9 % | 73.8 % | **+8.9** |
| WikiText-103 PPL | 9.3 | 8.0 | **−14 %** |
| Inference FLOPs / token | ~1.7 T | ~0.42 T | **4× cheaper** |

**Table 2 — Compute-optimal N* and D* (Hoffmann Approach 2 IsoFLOP)**

| Compute budget C (FLOPs) | Optimal N* (params) | Optimal D* (tokens) | D*/N* |
|---:|---:|---:|---:|
| 6e18 | 400 M | 8 B | 20 |
| 1.2e21 | 6.7 B | 134 B | 20 |
| 5.76e23 | 67 B | 1.4 T | 21 |
| 1e25 | 280 B | 6 T | 21 |
| 1e26 | 1 T | 17 T | 17 |

**Table 3 — Kaplan vs Chinchilla compute-optimal exponents**

| Quantity | Kaplan (2020) | Chinchilla (2022) |
|---|---:|---:|
| N exponent vs C | 0.73 | 0.49–0.55 |
| D exponent vs C | 0.27 | 0.45–0.51 |
| D / N at optimum | ~1 | ~20 |
| Implied recipe at 10× C | 5.4× N, 1.9× D | 3.2× N, 3.2× D |

## Variants and ablations

- **Hoffmann §A8 — robustness to dataset.** Refits on MassiveText vs The Pile vs C4; exponents remain `α ≈ 0.3`, `β ≈ 0.3` to within ±0.05 — **the law is dataset-robust at this granularity**.
- **Vision and multimodal extensions.** Henighan et al. (arXiv:2010.14701, "Scaling Laws for Autoregressive Generative Modelling") generalize Kaplan to image, video, math, image-to-text — same power-law family, different exponents per modality.
- **Mixture-of-experts scaling laws.** Clark et al. (DeepMind 2022, arXiv:2202.01169) and Krajewski et al. (arXiv:2402.07871) show MoE has a *third* axis (sparsity / number of active experts) and the compute-optimal frontier shifts; the relevant capacity coordinate is roughly `N_active` not `N_total`.
- **Data-constrained scaling laws.** Muennighoff et al. (arXiv:2305.16264, "Scaling Data-Constrained Language Models") show that when D is bounded (e.g., common-crawl quality cap), repeating tokens up to ~4 epochs is loss-equivalent to fresh tokens, then degrades. Crucial for any frontier above 10 T tokens where unique high-quality D runs out.
- **Inference-aware scaling laws.** Sardana & Frankle (arXiv:2401.00448, "Beyond Chinchilla-Optimal") and Hoffmann's own *training-vs-serving* analysis recommend **overshooting Chinchilla** (D/N = 100–1000) when inference cost is the binding constraint — this is the rationale for the Llama-3 70 B trained on 15 T tokens recipe.
- **Architecture-level breaks.** Tay et al. and the *Inductive bias and scaling* line argue that very different architectures (state-space models like Mamba, retentive networks) can shift the constants but not the form. The form survives.
- **Chinchilla's missing factor — finetuning data.** Chinchilla optimizes pretraining loss; downstream-task accuracy after finetune scales differently and depends on finetune-data shape — relevant for [84-swe-search-mcts](84-swe-search-mcts.md), [85-alphaevolve](85-alphaevolve.md), and any RLHF stack.

## Failure modes and limitations

- **Scope is pretraining loss only.** The laws say nothing about *task accuracy* after instruction-tune, RLHF, DPO, agent finetune, or domain finetune. The capability that matters for an agent harness is downstream; pretraining loss is necessary but not sufficient.
- **Constants are dataset-, tokenizer-, and architecture-dependent.** The numerical fits in Hoffmann are for MassiveText with Gopher's tokenizer and the Chinchilla architecture family. Re-fits on other regimes give different `A`, `B`, `E` even when the form is preserved.
- **The α, β fits depend on training-schedule choices.** Kaplan's recipe was wrong precisely because of an LR-schedule artefact. Any new claim of "different exponents" should be audited for whether small-N runs reached the loss floor.
- **Out-of-distribution evaluations are not predicted by the law.** Loss is in-distribution next-token prediction; whether the model generalizes to e.g. agentic tool use, multi-hop reasoning ([198-202]), or out-of-distribution code (SWE-bench) requires extrapolation outside the law's tested regime.
- **Diminishing returns of pretraining at frontier scale.** The α exponent is small (~0.3); reducing loss by 0.1 nats requires roughly 100× compute. The agent-era axes (test-time compute, trajectory length, multi-agent parallelism) often deliver larger downstream-task gains for less marginal cost — this is the *thesis* of [217-test-time-compute-scaling](217-test-time-compute-scaling.md).
- **Token quality and de-duplication are uncounted.** D in the law is "tokens trained on," but a deduplicated, filtered, high-quality token is worth more than a raw common-crawl token. Recent work (e.g. SmolLM-corpus, FineWeb-Edu, DCLM) shows 5–10× effective D from quality filtering alone.

## When to use, when not

**Use these laws as a budgeting tool** when you are picking which base model to anchor an agent harness on, choosing whether to finetune a smaller compute-optimal model versus prompting a larger undertrained one, planning a from-scratch pretraining run, or arguing about whether further pretraining or further harness investment is the higher-marginal-return next dollar. The laws are also the right frame for explaining to a non-technical stakeholder why "a bigger model" is not always better — Chinchilla 70 B is concretely smarter than Gopher 280 B at the same training budget.

**Do not use them to predict downstream-task accuracy directly**, to compare across modalities or architectures with confidence the constants port over, or to argue that an agent-era harness is a poor investment. The whole point of [217](217-test-time-compute-scaling.md), [237](237-agent-trajectory-scaling.md), [223](223-verifier-and-best-of-n-scaling.md), and [224](224-multi-agent-parallel-scaling.md) is that the *agent-era* axes scale on different curves with different exponents and different unit economics — and at the inference margin, an agent-era axis often dominates.

## Implications for harness engineering

- **Anchor model selection is the largest single capability lever.** Picking Llama-3-70B-Instruct (15 T tokens, D/N=215) versus Llama-2-70B (2 T tokens, D/N=29) is a much bigger capability change than any harness intervention in this corpus — see [01-agent-loop-architecture](01-agent-loop-architecture.md), [04-skills](04-skills.md), [89-voyager-deep](89-voyager-deep.md). Pick last.
- **Treat parameter count as a noisy proxy for capability.** A 7 B Llama-3 trained on 15 T tokens beats a 13 B Llama-2 on most agent benchmarks; report training-token counts and D/N alongside parameter counts in any harness eval.
- **Inference-cost-aware overshoot is the new norm.** [15-cost-router-and-budget](15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md), and [88-confidence-driven-router](88-confidence-driven-router.md) all assume base models trained well past Chinchilla so inference is cheap; budget routers should specialize per (model, D/N) tier rather than per "size class."
- **Capability ceiling for tool-use and code is set by pretraining data mix, not just N.** Code-model scaling (StarCoder-2, CodeLlama) shows the C-fraction of training data dominates SWE-bench performance; harness work on coding agents ([84-swe-search-mcts](84-swe-search-mcts.md), [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md)) cannot recover capability missing from pretraining.
- **Verifier and PRM training inherit pretraining curves.** [97-qwen-prm](97-qwen-prm.md) and [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md) — the PRM is itself a model whose Chinchilla position determines verification quality; do not under-train PRMs.
- **Multi-agent debate amplifies, does not create capability.** [98-diversity-collapse-mas](98-diversity-collapse-mas.md), [224-multi-agent-parallel-scaling](224-multi-agent-parallel-scaling.md) — adding more agents of the same Chinchilla-position model has diminishing returns; mixing models of *different* training corpora is what produces ensemble lift.
- **Skill memory and ReasoningBank are post-pretraining knowledge stores.** [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md), [151-153 MEMTIER trilogy] — these only matter to the extent the base model can *use* them at inference; capability to consult memory is itself a pretraining-determined skill.
- **Distillation is a Chinchilla-aware lever.** Distilling a Chinchilla-overshot 70 B teacher into a 7 B student often outperforms training the 7 B from scratch at the same FLOPs; the implication for harness eng is that a *small + cheap + harness-rich* deployment can match a *large + expensive + harness-thin* deployment.
- **Emergent capabilities are budgetable.** If "agent-grade tool use" emerges around a particular pretraining loss, you can budget for it as `tokens × params` rather than waiting for it to appear. Plan capability roadmap against the L(N, D) curve.
- **Research agents pick papers; pretraining picks priors.** [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [195-argus-omega-vol-2-trajectory-temporal-horizon](195-argus-omega-vol-2-trajectory-temporal-horizon.md) — a research-agent's literature priors are baked in at pretraining; cutoff date and corpus quality matter for what it can synthesize.
- **Permission and safety models are Chinchilla-shape too.** [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — when a refusal model is itself an LM, its calibration tracks pretraining loss; do not put under-trained classifiers on the safety path.
- **The "next 10×" of capability comes off the agent axes.** This is the punchline that makes the rest of this docs corpus relevant: the marginal return on agent-era axes ([217](217-test-time-compute-scaling.md), [237](237-agent-trajectory-scaling.md), [223](223-verifier-and-best-of-n-scaling.md), [224](224-multi-agent-parallel-scaling.md)) is steeper than the marginal return on pretraining at the current frontier. Pretraining sets the floor; agents set the ceiling.

**One-line takeaway for harness designers.** **Pretraining sets the capability floor and the unit economics; whatever you build on top of a base model can never exceed what its position on the L(N, D) curve already paid for — pick the model with the highest D/N you can afford and budget every harness investment as a multiplier on *that* baseline, not a substitute for it.**
