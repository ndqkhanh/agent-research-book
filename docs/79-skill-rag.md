# 79 — Skill-RAG: Failure-State-Aware Retrieval via Hidden-State Probing

**Paper.** Kai Wei, Raymond Li, Xi Zhu, Zhaoqian Xue, Jiaojiao Han, Jingcheng Niu, Fan Yang (corresponding) — *Skill-RAG: Failure-State-Aware Retrieval Augmentation via Hidden-State Probing and Skill Routing* — arXiv:2604.15771v1 — University of Michigan; UBC; Rutgers; UPenn; NJIT; TU Darmstadt; Wake Forest — 17 April 2026.

**One-line definition.** Skill-RAG augments RAG with a **lightweight hidden-state prober** that gates *whether* to retrieve and *whether* evidence suffices, plus a **prompt-based skill router** that on failure selects among four retrieval *skills*—query rewriting, question decomposition, evidence focusing, and exit—so recovery is **diagnosis-conditioned**, not undifferentiated re-retrieval.

## Why this paper matters

Adaptive RAG has improved *when* and *how often* to retrieve (FLARE, DRAGIN, Self-RAG, Adaptive-RAG) and *whether* via hidden states (Probing-RAG). Skill-RAG argues many **persistent** failures are not missing documents but **query–evidence misalignment**: passages are near the need yet the query’s surface form, hop structure, or scope does not match indexing and writing—so “retrieve again” wastes rounds. The authors show in **t-SNE** of hard cases (wrong after three vanilla retrieval rounds) **two separable clusters**: fixable alignment gaps versus sparser **irreducible** failures (retriever floor / missing knowledge). A **prober** supplies cheap gating; a **skill router** applies *named* alignment fixes instead of generic re-query—positioning Skill-RAG between **document-level** correction (CRAG) and coarse adaptive loops.

On **Gemma2-9B**, Skill-RAG improves over Probing-RAG by **+6.1** ACC (MuSiQue) and **+13.6** ACC (2WikiMultiHopQA), isolating **failure-conditioned routing** from probe-only gating. Table 1 shows **average ACC 46.8** (best in the block) with the usual trade-off that some baselines (e.g. DRAGIN) win **HotpotQA EM** while Skill-RAG leads **Hotpot ACC**—emphasizing answer correctness over raw string hit rate.

## Problem it solves

1. **The “retry, not diagnose” gap.** Post-retrieval failure is usually treated as “iterate retrieval,” not “classify *why* query and evidence do not meet”—so **alignment** failures persist.
2. **Four-class view from parallel inference.** **With-retrieval** and **no-retrieval** answers vs. gold yield **correct-with-retrieval**, **wrong-with-retrieval**, **correct-without-retrieval**, and **wrong-without-retrieval** (Figure 1), encoding when retrieval helps, hurts, is unnecessary, or the model fails unaided. Sec. 3.1 still trains the prober with a **binary** correctness target on hidden states; the **taxonomy** structures data and analysis.
3. **Coarse scheduling is insufficient** when the fix is a **different query transformation**—Skill-RAG’s second stage is **skill selection** (literature-grounded operations).
4. **Probe-only gating drifts.** The **My Hero Academia** case (Table 2): Probing-RAG concatenation steers the query to an unrelated band; **query_misaligned → rewrite** recovers the Japanese premiere.

## Core idea in one paragraph

**Parallel** runs: **Path A** = single-step retrieval, **Path B** = none; both use CoT + answer. A small **MLP prober** on pooled hidden states (posterior **two-thirds of layers**; reasoning+answer tokens) learns **sufficiency / correctness**; per-layer probers **average** at inference. The prober decides **parametric-only OK**, or runs **BM25** retrieval and re-checks. If still bad, a **prompted router** picks **rewrite / decompose / focus / exit**; a new query **re-retrieves**, the model **regenerates**, the **prober re-gates**; loop until ok, **exit**, or **max rounds**.

## Mechanism (step by step)

### (a) Four-class-structured data from parallel LLM paths

On **HotpotQA, NQ, TriviaQA** training splits, each example is run with **no retrieval** and **single-step retrieval**; hidden states and answers are recorded. Comparing each path to **gold** yields the **four outcome classes** in Figure 1. For the **prober**, each path contributes examples with a **binary** label 1[answer matches gold] (Sec. 3.1). **3,000** train / **500** dev examples; **500**-example OOD test sets for **MuSiQue** and **2WikiMultiHopQA**.

### (b) Prober training and inference

```text
h[l] = pool_tokens(hidden_states[layer=l], span={CoT, answer})
train MLP_l(h[l], y)   where y = 1[ match(generated, gold) ]
# Inference:
p = mean_l sigmoid(MLP_l(h[l]))    # paper: average prob across layers
accept / skip_retrieve if p indicates parametric sufficiency; idem after 1st retrieval
```

**One** hidden layer inside each MLP; **binary** head. Aggregation across depths follows evidence that different layers carry complementary **readiness** signals (Jin et al. on concept depth, cited in-paper context).

### (c) Four skill-router actions

**Inputs:** question, failed reasoning+answer, current passages. **One** of:

1. **Query rewriting** — surface / BM25 index mismatch (Ma et al.).
2. **Question decomposition** — entangled multi-hop → **sub-query sequence** (Press et al.).
3. **Evidence focusing** — overly broad query → **slot** gaps from context, grounded follow-up (CRAG-**spirit**, failure-type not per-doc score).
4. **Exit** — irreducible; **stop** to limit cost.

The router is **not** a second trained 7B head; it is **prompting** the backbone—still an LLM call, but not an extra *weight matrix* for routing. Abstract loop:

```text
while not done:
  if prober_allows_bypass: return parametric answer
  docs = retrieve(q); gen = LLM(q, docs)
  if prober_sufficient(gen): return answer
  skill = route(q, gen, docs)
  if skill==exit: return answer_or_stop
  q = apply_skill(skill, ...);  # new retrieval round
```

### (d) Inference-time retrieval pipeline

(1) Prober: **need retrieval?** If no → **finalize without retrieval** (Sec. 3, Fig. 1). (2) **BM25** top-k, generate with evidence. (3) Prober: **sufficient?** If yes → finalize. (4) Else **skill router** → new query. (5) Re-retrieve, regenerate, **re-probe**. (6) Terminate on **exit**, prober accept, or **round cap** (Sec. 3.3). **4-shot** prompts; **Gemma2-9B** throughout Table 1.

## Empirical results

**Table 1 (Gemma2-9B, BM25)** — EM / ACC per dataset; last column = paper’s **Average**.

| Method | Hotpot | NQ | TriviaQA | MuSiQue (OOD) | 2Wiki (OOD) | Avg |
|--------|--------|-----|----------|---------------|-------------|-----|
| No Retrieval | 24.6/37.1 | 27.3/42.7 | 52.4/65.7 | 6.1/9.8 | 29.3/44.5 | 28.0/40.0 |
| Single-step RAG | 27.8/42.6 | 26.1/40.7 | 46.5/59.3 | 6.4/15.2 | 26.5/43.8 | 26.7/41.5 |
| FLARE | 29.5/33.8 | 29.7/36.4 | 44.8/57.2 | 5.3/8.4 | 24.1/36.8 | 26.7/34.5 |
| DRAGIN | 35.6/37.5 | 34.2/38.9 | 55.3/63.4 | 7.4/11.2 | 30.8/42.3 | 32.7/38.7 |
| Adaptive-RAG | 29.1/32.3 | 29.4/34.5 | 46.2/58.6 | 5.8/9.1 | 25.6/38.4 | 27.2/34.6 |
| Probing-RAG | 22.8/44.7 | 34.1/48.5 | 59.3/65.9 | 7.6/13.9 | 26.6/38.9 | 30.1/42.4 |
| **Skill-RAG** | **24.2/46.1** | **34.3/49.7** | **59.3/65.9** | **7.8/20.0** | **28.9/52.5** | **31.0/46.8** |

**Notes.** DRAGIN leads **Hotpot EM (35.6)** vs Skill-RAG **24.2**; Skill-RAG has **highest Hotpot ACC (46.1)** and **best average ACC (46.8)**. OOD: **+6.1** and **+13.6** ACC over Probing-RAG on MuSiQue and 2Wiki. Trivia **EM 59.3** ties Probing-RAG.

## Variants and ablations

- **6+ auto-generated skills** (Fig. 2D) **destroy** t-SNE cluster separation—**skill sprawl** removes routable geometry vs **3 rewrite + exit** (C) that **shrink** the left “alignment” cluster.
- **(B)** diagnosis + re-retrieve sparsifies/drifts clusters vs **(A)** baseline hard cases; supports typed correction narrative.
- Layer-mean prober is the core architectural ablation (vs single-layer) implied by per-layer training.

## Failure modes and limitations

- **Router depends on** instruction-following; weak models may **misdiagnose** or over-**exit** (authors’ limitation).
- **Prober: train per backbone, on hidden states**—**model-specific**, **not** shown to port across families; production **model churn** = **re-train**.
- **Single model family in paper (Gemma2-9B)**; scale/architecture sweep “future work.”
- **Open-domain QA** only; **scientific / multilingual** corpora may need **new probes** and revisited skill set—large skill lists **hurt** (ablation).
- **BM25-only** main table; **dense / hybrid** interaction unknown.
- **Binary** prober elides fine distinctions the **four-class** view captures analytically.

## When to use, when not

**Use** with a **stable backbone** exposing **activations**, hard **multi-hop** / **shifted** test domains where **+ACC OOD** matters, and a need for **named telemetry** (which skill fired). **Avoid** if failure mode is **no evidence in corpus**, you **cannot** run **probe training**, or you **swap models** without retraining. **Do not** grow the **skill list** ad hoc.

## Implications for harness engineering

See [25-agentic-rag](25-agentic-rag.md): implement **routed re-query**—not only `retrieve()` in a loop but **typed** repair steps with logs. [04-skills](04-skills.md) **packages** document procedural skills; Skill-RAG’s four items are **runtime retrieval operators** invoked when a **readiness** signal fails—**progressive disclosure** of *retrieval* tactics parallel to **tool** skills. [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md): the **prober** is a **readiness verifier**; the **router** picks **structural** fixes rather than only **textual** critique of the draft—**verifier → router** for **retrieval**, analogous to **evaluator** routing **generator** revisions. **Ship** prober weights + **router system prompt** **pinned** to **one** backbone build; add **query-drift** guards (Table 2 lesson).

## Connections to other work in this corpus

**Probing-RAG** = gating baseline; **Skill-RAG** adds **OOD** gains from **routing**. **Self-RAG / FLARE / DRAGIN / Adaptive-RAG** schedule retrieval; this work repairs **post-aug** misalignment. **CRAG** scores **docs**; Skill-RAG diagnoses **failure state**. **IRCoT / Iter-RetGen**-style interleaving can **compose** with Skill-RAG as the **retrieval-repair** layer.

## Key takeaways

1. **Alignment** failures need **typed** interventions, not only more loops.
2. **Small per-layer MLPs** + **mean** pooling replace a giant critic LM for **gating**.
3. **Four** skills match **separable** failure geometry; **>4** prompted skills can **break** it.
4. **+6.1 / +13.6** ACC (MuSiQue / 2Wiki) vs Probing-RAG shows **routing**’s value on **OOD** multi-hop.

## References

- Wei, K., et al. *Skill-RAG: Failure-State-Aware Retrieval Augmentation via Hidden-State Probing and Skill Routing.* arXiv:2604.15771v1, 17 Apr 2026.
- As cited in-paper: Self-RAG; Probing-RAG; FLARE; DRAGIN; Adaptive-RAG; CRAG; query rewriting (Ma et al.); decomposition (Press et al.); HotpotQA, NQ, TriviaQA, MuSiQue, 2WikiMultiHopQA; BM25 (Robertson & Zaragoza).
