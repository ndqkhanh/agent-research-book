# 97 — Qwen2.5-Math PRM: Lessons in Building Process Reward Models

**Paper.** Zhenru Zhang, Chujie Zheng, Yangzhen Wu, Beichen Zhang, Runji Lin, Bowen Yu, Dayiheng Liu, Jingren Zhou, Junyang Lin — *The Lessons of Developing Process Reward Models in Mathematical Reasoning* — arXiv:2501.07301 — Qwen Team, Alibaba Group — 2025 (v2: 5 Jun 2025). Artifacts: [Qwen2.5-Math-PRM-7B](https://hf.co/Qwen/Qwen2.5-Math-PRM-7B), [Qwen2.5-Math-PRM-72B](https://hf.co/Qwen/Qwen2.5-Math-PRM-72B).

**One-line definition.** A hardcore practitioner report on what works (and does not) when training process reward models (PRMs) for mathematical reasoning at scale: Monte Carlo (MC) rollouts for step labels, LLM-as-judge critics, human PRM800K data, consensus filtering, and why response-level Best-of-N (BoN) alone mis-ranks and mis-trains “process” models.

## Why this paper matters (PRMs are the search reward signal — Voyager, MCTS, RLHF for reasoning all need it; this paper documents what actually works)

Search, tree expansion, and preference optimization need a **per-step value signal**. PRMs should **verify** steps, but the common **Math-Shepherd-style MC** recipe often trains **value** estimators (“eventual correct answer from here”), not deterministic verifiers (“this step is valid”). The paper ablates **MC vs LLM judge vs human**, pairs **prm@8** with **ProcessBench** (Zheng et al., 2024), and shows **MC–judge consensus** as a practical filter before wiring a PRM into exploration.

## Problem it solves (PRM data scale, label quality, BoN aggregation, MC-rollout cost)

1. **Label scale vs quality.** PRM800K-style human work does not scale; Math-Shepherd-style MC does, but the paper shows MC-labeled PRMs can trail human-labeled ones on *step* identification (e.g. ProcessBench mean F1 40.1 for MC on 860k vs 56.5 for human on 264k) while **beating** human-labeled data on prm@8 (65.9 vs 64.9 average across seven math benchmarks in one table)—an explicit **inversion** between metrics.
2. **BoN is a biased lens.** If the policy often produces **right answers with wrong derivations**—5.1% on GSM8K rising to 43.4% on Omni-MATH in their sampled manual study—a PRM that penalizes those chains loses BoN credit even when it is doing the *right* process job; conversely, weak process discrimination **inflates** BoN by tolerating “lucky” wrong-process solutions.
3. **MC cost and variance.** They use 8 completions per step in core experiments; more rollouts would reduce variance but the paper flags diminishing returns and dollar cost. Thresholding soft MC scores matters: stricter than “any completion hits the answer” (their recommended hard-negative threshold) monotonically **hurts** both BoN and ProcessBench (Figure 5).
4. **Scoring rule mismatch.** For MC-trained PRMs, **last-step** (final-position) score tracks BoN better than product or min over steps; for human- and judge-trained PRMs, **product** or **min** (Lightman-style) behaves as intended. Mixing training semantics and test aggregation silently changes what you optimized.

## Core idea in one paragraph

A PRM should implement **deterministic step correctness**; MC estimation instead estimates **rollout value** to the terminal gold answer, injecting noise (correct final from bad steps, failure from good steps) *unless* the completion policy is an oracle. The authors pair MC-derived labels with an **independent** Qwen2.5-72B **LLM-as-judge** pass and **drop** (query, response) instances where the two disagree on where the first error is—roughly **40% of 860k kept (~350k)**, a consensus subset. On that cleaner data, hard labels beat soft ones markedly *after* filtering (while nearly tied before), and the released **Qwen2.5-Math-PRM-7B/72B** improve both prm@8 and ProcessBench F1 over strong open baselines, with 7B reaching **ProcessBench** average F1 **73.5** (vs o1-mini **87.9** and GPT-4o-0806 **61.9** in their Table 7).

## Mechanism (step by step)

### (a) Data construction: MC sampling vs human labels vs LLM-as-judge

- **MC (Math-Shepherd style).** Gold answer per query; 6–8 solutions (Qwen2 / Qwen2.5 Math Instruct, 7B/72B); steps on `\n\n`; **8** continuations per step; hard = any completion hits the answer; soft = empirical success rate; truncate after first wrong step. **860k** MC: prm@8 mean **65.9** vs Math-Shepherd **440k** **64.3**; ProcessBench F1 **40.1** vs **28.9** (same tables)—scale helps BoN, not enough for process F1 vs human.
- **Human.** PRM800K (~264k after dedup) gives **64.9** prm@8 but **56.5** mean ProcessBench F1—best **step** model in that comparison, worst **BoN** model.
- **LLM-as-judge (vanilla, same 860k trajectories).** Qwen2.5-72B-Instruct scores each step with a template (Appendix C). It reaches **65.3** prm@8 (below MC 65.9) but **46.5** mean ProcessBench F1 (above MC 40.1). Relying on judge labels **alone** is therefore *not* a uniform win: you trade BoN-optimized quality for better step F1, and remain **10 F1 points** below human on ProcessBench.
- **Consensus filter.** Keep only examples where **MC and judge agree** on the **index** of the first incorrect step. ~**350k** examples remain. Post-filter PRMs **match** full-judge ProcessBench F1 (Figure 2) while using **~40% of the data**, and beat unfiltered MC on both prm@8 and ProcessBench. At **3M** MC + filter → **1.5M**, hard vs soft and pre/post-filter gaps widen (Figure 3–4).

```python
def consensus_filter(samples):
    """
    sample: (question, response_steps[], gold_answer, mc_label_per_step[], judge_label_per_step[])
    Agreement is on the *index* of the first wrong step.
    """
    kept = []
    for s in samples:
        i_mc = first_error_index(s.mc_label_per_step)
        i_j = first_error_index(s.judge_label_per_step)
        if i_mc == i_j:
            kept.append(s)
    return kept  # expect ~0.4 * len(samples) in the paper's 860k run
```

### (b) PRM model architecture (token/sequence-level scoring)

Initialize from **Qwen2.5-Math-7B/72B-Instruct**; replace the LM head with a **small scalar head** (two linear layers). Each step is treated as a span; the PRM produces a **score on the last token of each step** (binary for hard labels, regression for soft in early experiments). Response-level score for final models in Table 6/7: **product** of step scores (ORM baseline uses a single Qwen2.5-Math-RM-72B score).

### (c) Loss formulations

- **Primary release:** **cross-entropy** on the **terminal token of each step** for **binary** (hard) step labels.
- **Preliminary MC runs:** also **MSE** for soft step targets.
- If forced to use raw MC only: **threshold 0/8** as negative vs any positive hit works best; higher thresholds (1/8…7/8) degrade prm@8 and ProcessBench together (Figure 5).

### (d) The consensus–filter–then-train pattern

**Expand** with high-volume MC, **critic** each step with a strong instruct model, **discard disagreements**, then train on the **smaller, higher-agreement** set—optionally on **1.5M** after 3M expansion (Section 3.1.3–3.1.4). The pattern trades **brute count** for **annotation alignment** between two *different* error models.

### (e) Best-of-N vs majority-vote aggregation

- **pass@8** (oracle upper bound) averages **74.7** in Table 6; **maj@8** is **66.2**.
- **prm@8** vs **maj@8** (**66.2**): preliminary runs (Section 2.3) had *no* PRM above maj@8; the **released** **Qwen2.5-Math-PRM-7B/72B** hit **67.6 / 69.3** mean—**+1.4** for 7B over maj@8—and beat maj@8 on **all seven** tasks (Table 6), still below pass@8 **74.7**.

### (f) The trap: “vanilla” LLM-as-judge step labels and pure MC are both wrong as *sole* objects

- **LLM-judge only** (same 860k): **worse** prm@8 than MC (**65.3 < 65.9**) but **better** ProcessBench F1 than MC (**46.5 > 40.1**). Optimizing the labelling pipeline for **one** public leaderboard (BoN) **hides** regression on the other (process ID).
- **MC-only** at scale: strong BoN, weak ProcessBench; large **min-score mass on the final-answer step** in many public PRMs (>40% for several baselines in Figure 8) reveals **process-to-outcome shift**—models behave like ORMs in practice.
- **Human** gold: best process ID, not best BoN in their 264k run. *None* of the annotation strategies dominates both metrics without the mixed recipe + evaluation portfolio.

## Empirical results (table — PRMs on ProcessBench / MATH / GSM8K with different data construction methods)

**Table 3 — Best-of-8 (policy Qwen2.5-Math-7B-Instruct), mean over seven benchmark columns**

| Data construction | # samples | Avg. prm@8 |
|---------------------|----------:|-----------:|
| MC (Math-Shepherd) | 440k | 64.3 |
| MC (ours) | 860k | **65.9** |
| LLM-as-judge (ours) | 860k | 65.3 |
| Human (PRM800K) | 264k | 64.9 |

**Table 4 — ProcessBench mean F1 (GSM8K, MATH, OlympiadBench, Omni-MATH averaged)**

| Method | # samples | Avg. F1 |
|--------|----------:|--------:|
| MC (Math-Shepherd) | 440k | 28.9 |
| MC (ours) | 860k | 40.1 |
| LLM-as-judge (ours) | 860k | 46.5 |
| Human (PRM800K) | 264k | **56.5** |

**Released models — Table 6 (prm@8) and Table 7 (ProcessBench F1, selected rows)**

| Model | Avg. prm@8 | Avg. ProcessBench F1 |
|-------|------------|----------------------|
| maj@8 baseline | 66.2 | — |
| Qwen2.5-Math-PRM-7B | **67.6** | **73.5** |
| Qwen2.5-Math-PRM-72B | **69.3** | **78.3** |
| o1-mini (judge) | — | 87.9 |
| Qwen2.5-Math-7B-PRM800K | 64.9 | 56.5 |

## Variants and ablations

- **MC scale:** 860k in-house MC vs 440k Math-Shepherd—both prm@8 and ProcessBench F1 **improve** with the larger MC corpus (65.9 vs 64.3; 40.1 vs 28.9).
- **Soft vs hard labels on MC data:** pre-consensus, nearly identical; **post-consensus**, **hard** labels **substantially** beat soft on both metrics; soft labels smear *deterministic* correctness with *stochastic* completion frequencies (Section 3.1.4).
- **3M → 1.5M filtered:** amplifies the hard-vs-soft gap (Figure 3–4); consensus benefits **both** label types.
- **MC threshold sweeps (3M unfiltered):** any threshold above 0/8 for a positive step **hurts** (Figure 5).
- **Scoring for BoN (Figure 9):** MC PRMs: **last-step** score best; human/judge PRMs: **min** and **product** best—tied to dependent vs per-step score semantics (MC estimates are not independent across positions).

## Failure modes and limitations

- **Gap to pass@8:** Even the best prm@8 **69.3** (72B) is far from **74.7** pass@8—large headroom; limitations section acknowledges this explicitly.
- **Open RL recipes:** “Best practices for utilizing PRMs in RL” are **unexplored** in the study (limitation).
- **Human + weak supervision:** Merging their consensus pipeline with *more* human data in a **curriculum** is left for future work.
- **Proprietary ceiling:** o1-mini **87.9** mean ProcessBench F1 vs **78.3** (72B PRM) shows proprietary judges/PRMs still ahead on step ID.
- **BoN as sole fitness:** Encourages **outcome** weight on final steps (Figure 8: many public PRMs 40–54% of min-scores on last step) — **miscalibrated** if the deployment goal is intermediate tool or proof checking.

## When to use, when not

**Use a PRM + consensus (or high-quality human) labels when** you need **search-time pruning** or **step-level auditing** in math-style chains, and you can pay for **ProcessBench- or task-specific step evaluation** in development—not only prm@N.

**Favor last-step BoN scoring** only if you *must* keep an MC-estimated model and you accept value-model semantics.

**Do not** trust **prm@N alone**: on ProcessBench slices where the answer is correct but the chain is wrong, most open PRMs stay **under 50%** detection accuracy; **Qwen2.5-Math-PRM-72B** reaches **76.6%** on MATH in that setting (Table 5). **BoN vs ProcessBench** can trend oppositely by data source (Figure 7).

## Implications for harness engineering

PRMs are **harness-time evaluator infrastructure**: multi-step agents need **per-step** scores aligned with environment verifiers, not just final reward. Pair with [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md): outcome RMs are not interchangeable with process verifiers when policies can get **lucky**. [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md): judge labels are cheap but not equivalent to MC—**disagreement** drives consensus filtering before online reward. [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md): a value-flavored PRM **misprioritizes** search branches when completion luck ≠ local validity (their MC vs judge split). [18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md): refinement needs **process** signal; BoN-only training drifts to **final-step** weighting (Figure 8). **Data hygiene risk:** annotation source and BoN scoring rule can **invert** which checkpoint looks best.

## Connections to other work in this corpus

Grounds **PRM800K**, **Math-Shepherd**, **ProcessBench**, ORM vs PRM, and implicit-PRM lines (e.g. Eurus) in one empirical story. Complements test-time scaling work in this set (e.g. #77) by isolating **reward-model** quality upstream of selection.

## Key takeaways

1. **MC labels** push PRMs toward **value-model** behavior unless **filtered** or combined with judges; **human** labels win ProcessBench but not always prm@8 (metric inversion).
2. **Consensus** on first-error **index** keeps ~**40%** of data (~**350k** from 860k), lifts both **prm@8** and ProcessBench vs raw MC.
3. **Hard** labels after filtering beat **soft**; raw MC threshold **0/8** beats stricter thresholds (Figure 5).
4. Report **BoN and ProcessBench** (and hard cases: right answer, wrong chain); **scoring**: **last** for MC-trained, **product/min** for judge/human (Figure 9).
5. **Qwen2.5-Math-PRM-7B/72B**: prm@8 **67.6 / 69.3**, ProcessBench F1 **73.5 / 78.3** (Table 6–7).

## References

Zhang et al. (2025), arXiv:2501.07301v2. See PRM800K; Math-Shepherd; ProcessBench (Zheng et al.); GSM8K; MATH; Qwen2.5-Math.
