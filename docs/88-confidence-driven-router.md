# 88 — Confidence-Driven LLM Router

**Paper.** Tuo Zhang, Asal Mehradfar, Dimitrios Dimitriadis, Salman Avestimehr — *Leveraging Uncertainty Estimation for Efficient LLM Routing* (Confidence-Driven LLM Router) — arXiv:2502.11021v1 [cs.NI] — 16 February 2025 — University of Southern California; Amazon AI.

**One-line definition.** **Semantic entropy (weak + strong) measures self-consistency in meaning space;** the router’s training labels mark **offload** when **normalized** uncertainty **δ_SE** exceeds **τ** and the **lower-SE** model wins—otherwise **tie** (cheap path friendly). **Escalation is tied to uncertainty structure**, not a fixed “always use GPT-4 for class X” map.

## Why this paper matters (next evolution beyond preference-data routing; calibration-based escalation)

RouteLLM (human win/loss) and TO-Router (benchmark accuracy) are brittle: Arena-style preferences are sparse and **non-monotonic**—the authors show routing efficiency can “**even become worse** as the number of training samples increases” (Figure 1). This work derives labels from **whether two models’ answer clouds disagree in *meaning*** (low vs high **semantic entropy**), not from aggregate human taste. The evaluation pairs **CPT (cost)** with **GPT-o1** judging **correctness and reasoning precision**, not accuracy alone.

## Problem it solves (preference data is expensive; calibration is cheaper)

Human ratings are **expensive and inconsistent**; **accuracy** ignores confidence—two “correct” models can differ sharply in **reliability**. The edge–cloud setting needs *when to offload* without hand-labeling every query pair. They fix **N = 12,247** training prompts (3,610 from each of Natural Questions, TriviaQA, and PopQA, plus 1,418 from MAWPS) to **match** their RouteLLM training volume so improvements trace to **synthetic SE labels**, not data quantity.

## Core idea in one paragraph

For each prompt, the system draws multiple stochastic completions from both a **strong** and **weak** model, clusters them into meaning-equivalence classes (bidirectional **natural language entailment** with a DeBERTa-large–style classifier, following Kuhn et al.), and computes **semantic entropy (SE)** over those clusters. The models’ entropies are compared via a **normalized difference** δ_SE; only when |δ_SE| exceeds a hand-tuned **τ** is a “winner” label assigned to the model with **lower** SE; otherwise a **tie** is recorded. That label set becomes preference supervision for a standard **embedding → router** (Similarity-Weighted / Bradley–Terry, Matrix Factorization, MLP, or kNN), evaluated with **CPT (Call-Performance Threshold)** and dollar cost at fixed CPT, plus an LLM judge. The bet is: **uncertainty is a cheaper and more direct training target than large-scale human preference** for the routing decision problem.

## Mechanism (step by step)

**(a) Confidence / uncertainty signal.** Primary scalar: **semantic entropy (Kuhn et al.)**, not **token logprob**. Cluster paraphrases via **bidirectional entailment**, aggregate `p(c|x)`, then **SE(x) = −(1/|C|) Σ_i log p(C_i|x)** (Eqs. 1–2). **Low SE** ⇒ **tight** meaning distribution; **high SE** ⇒ **disagreeing** samples—kin to **self-consistency** / multi-draw **sampling uncertainty**, but **paraphrase-aware**. They do **not** route on raw **NLL** or unclustered entropy.

```text
# Semantic-entropy win rule (per prompt x), after K samples / cluster each model
SE_s ← SemanticEntropy(ClusterSamples(M_strong, x, K))
SE_w ← SemanticEntropy(ClusterSamples(M_weak,  x, K))
δ_SE ← (SE_s - SE_w) / SE_s   # strong-model denominator; Eq. 3
if δ_SE > τ:
    winner ← argmin( SE_s, SE_w )   # pick model with lower semantic entropy, Eq. 4
else:
    winner ← TIE
```

**(b) Thresholding policy (τ).** The hyperparameter **τ** trades **sensitivity to subtle linguistic change** (low τ, more “ties” and more reliance on the router) against **agreement with accuracy-style “who’s clearly better”** (high τ, fewer ties, stricter calls). The paper states explicitly: higher τ **aligns preference labels more closely with traditional accuracy**; lower τ increases sensitivity to **fine-grained** variation.

**(c) “Calibration” in this framework.** Not **ECE** on token probabilities; **τ** sets how large **δ_SE** must be before a **non-tie** label, **ties** absorb borderline cases, and **domain shift** forces **τ** / clustering **re-fits** (operational calibration of **uncertainty → action**).

**(d) Cost-aware evaluation and tuning.** **CPT(x%)** = minimum **fraction of queries to the strong model** to reach an **x%** accuracy gain over the weak model alone. At **CPT(80%)** on **MT-Bench**, they report API spend: Random **$4.06**; TO-Router **$3.88**; RouteLLM **$4.04**; their method **$3.74** (lowest in their table). In **Table 1 (MT-Bench)**, the **lowest (best) CPT** values shown are for **RouteLLM SW: CPT(50%) 27.31%** and **CPT(80%) 55.61%**; among **Confidence-Driven** variants, **MLP 35.54%** / **74.92%** and **MF 42.94%** / **63.53%** sit between that frontier and the TO baselines. On **GSM8K**, **Confidence-Driven MF** is **CPT(50%) 41.89%** with **(80%) 75.34%**—**best CPT(50%) in the published table** for that column. **τ + architecture** must be co-tuned per task distribution.

**Training pipeline (three phases).** (1) Factual and math sources; **multi-sample** generation; **DeBERTa-large**-style **bidirectional entailment** clustering (Kuhn et al.). (2) **SE-based winner/tie** labels. (3) **Embed prompts**, train **SW (Bradley–Terry), MF, MLP, or kNN** on `{winner_a, winner_b, tie}`.

## Empirical results (concrete cost reduction with quality preserved on benchmarks)

**Setup.** Strong: **GPT-4 (gpt-4-0613)**; weak: **Mixtral-8x7B**; test **MT-Bench, GSM8K, MMLU** (MMLU **14,042** items / 57 subjects, per Appendix). Baselines: **Random**; **TO-Router (kNN, MLP)**; **RouteLLM (SW, MF)**.

**CPT and dollars.** **Table 1** is split by benchmark: on **GSM8K**, **Confidence-Driven MF** leads **CPT(50%)** in the table (**41.89%**); on **MT-Bench CPT**, the **lowest** numbers printed are **RouteLLM SW** (as above), while the authors still report **lowest** **MT-Bench** **API cost** at the **CPT(80%)** operating point for their system (**$3.74** vs **$4.04** RouteLLM in their accounting). Read **$** and **CPT** together—the paper optimizes a **quality–cost** bundle, not CPT alone.

**Judge quality (GSM8K, same accuracy tier).** Table 2: scores **78.88 / 79.72 / 79.95** at CPT(50%) and **85.97 / 88.88 / 89.21** at CPT(80%) for TO-Router / RouteLLM / Confidence respectively—**highest** for the uncertainty-trained router, consistent with *confident* generations reading better, not just matching labels.

## Variants and ablations (different confidence signals)

**In-paper ablations** are **router head** and **τ**, not a grid over uncertainty definitions. **Related** signals: **(i)** mean **token log-likelihood**; **(ii)** **self-consistency** without **semantic** clustering; **(iii)** **MARS**-style scorers (Bakman et al. 2024); **(iv) verifier-first** gating. **Table 1** is **not** a single “winner” row: e.g. **GSM8K** favors **Confidence MF** on CPT(50%), while **MT-Bench CPT** is numerically best for **RouteLLM SW** in the printed table—**SE-supervised** heads are competitive on **GSM8K** and in **$ + judge** metrics even when one CPT cell is not the global minimum.

## Failure modes and limitations (overconfident wrong answers, calibration drift)

1. **Overconfident but wrong:** SE measures **diversity in meaning space**, not ground-truth alignment. A wrong model that **narrowly** repeats a single false statement gets **low SE** and may *never* trigger escalation—**same class of pathologies** as any confidence-based dispatch without an external **verifier** or **tool execution** check.  
2. **Domain and distribution shift:** The clustering+entailment step assumes a fixed **NLI model**; medical/legal domains can shift *entailment* boundaries, reshaping SE so **τ** and tie rates drift. The paper’s **τ** and datasets are open-domain / factual, not a guarantee under covariate shift.  
3. **Tie overuse / underuse:** A poorly chosen **τ** either floods ties (washing out labels) or forces crisp winners where no model is actually better, hurting router training.  
4. **Stated author limitations (Section 5):** **(i)** only **text** routing; **VLM** uncertainty may require different **clustering**; **(ii)** they **do not profile router inference overhead** (SW vs MLP cost at production QPS) despite architectural sensitivity.

## When to use, when not

**Use when** you can afford **K>1** samples from both models during **training** (or a distilled surrogate later), you care about **$ + latency** under **quality** constraints measurable by **LLM-as-judge** or task metrics, and your failure profile is more “**sometimes the small model is lost in hypothesis space**” than “**the small model silently hallucinates a single line**” (if the latter, add **tools or verifiers**). **Avoid when** multi-sample **cloud** calls at inference are unacceptable; when **entailment** is unreliable; or when **regulatory** settings demand an auditable, **rule-based** escalation policy **without** learned routing on top of ties.

## Implications for harness engineering

Generational framing (as in the corpus on multi-model harnesses; see [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md)): **(1) heuristic** cascades (FrugalGPT) → **(2) learned** routing from **preference** data (RouteLLM) → **(3) calibration/uncertainty**-synthetic labels + CPT + judge (this work). The bridge to [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) is **where the first model fork happens**; SE answers “**is the cheap model already sharp in meaning space?**” while a **verifier** answers “**is the chosen output actually true?**” Production **composition**: SE-based tiering **plus** tool/verifier on the routed output—not SE alone for high-stakes agents. Expose **SE features**, **τ**, and **router head** in observability; drift in **entailment** or domain shifts **τ** and tie rates.

## Connections to other work in this corpus

- [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md) — same **multi-expert** problem, different **label and policy** (heuristics, preferences, **SE**).  
- [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) — **downstream** checks after a route; this paper is **upstream feature generation** for the router.  
- External: **Kuhn et al.** (semantic entropy), **Chen et al. FrugalGPT** (sequential cascades the authors contrast with predictive routing for edge–cloud).

## Key takeaways

1. **Semantic entropy** (clustered, paraphrase-aware multi-sample analysis) is a **trainable, comparable** stand-in for *which model to trust* when human preference at scale is **noisy and non-scaling** (per their Arena study).  
2. **Normalized difference δ_SE** plus **τ** implements a **hysteresis-like** “only decide when the models disagree in *how spread out* their answers are,” producing **ties** that the learned head resolves with cost in mind.  
3. They report the **lowest** **MT-Bench** **API** **cost** at the **CPT(80%)** point among listed systems (**$3.74**), and on **GSM8K** they report **highest** **judge** scores vs TO-Router and RouteLLM **at the same CPT tiers** (Table 2: **79.95** and **89.21** for their system vs **79.72** and **88.88** for RouteLLM at CPT 50% and 80% respectively). **CPT(%)** and **$** are not the same objective—**Table 1**’s best CPT on **MT-Bench** is **RouteLLM SW** in the published numbers, so the headline is **method-level cost and judge quality**, not universal CPT dominance on every column.  
4. **Low entropy ≠ correctness**; add **verifiers, tools, or domain checks** in agent harnesses.

## References

1. T. Zhang, A. Mehradfar, D. Dimitriadis, S. Avestimehr, *Leveraging Uncertainty Estimation for Efficient LLM Routing*, arXiv:2502.11021, 2025.  
2. L. Kuhn, Y. Gal, S. Farquhar, *Semantic Uncertainty: Linguistic Invariances for Uncertainty Estimation in Natural Language Generation*, ICLR 2023.  
3. I. Ong *et al.*, *RouteLLM: Learning to Route LLMs with Preference Data*, arXiv:2406.18665, 2024.  
4. D. Stripelis *et al.*, *TensorOpera Router (TO-Router): A Multi-Model Router for Efficient LLM Inference*, EMNLP 2024.  
5. L. Chen, M. Zaharia, J. Zou, *FrugalGPT: How to Use Large Language Models While Reducing Cost and Improving Performance*, arXiv:2305.05176, 2023.  
6. W.-L. Chiang *et al.*, *Chatbot Arena*, arXiv:2403.04132, 2024.  
7. A. Mehradfar *et al.*, dataset *uncertainty* labels (HuggingFace: `AsalMehradfar/uncertainty_0.1` per the paper’s footnote).
