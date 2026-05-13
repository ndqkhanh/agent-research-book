# 87 — RouteLLM: Learning to Route Queries with Preference Data

**Paper.** Isaac Ong, Amjad Almahairi, Vincent Wu, Wei-Lin Chiang, Tianhao Wu, Joseph E. Gonzalez, Waleed Kadous, Ion Stoica — *RouteLLM: Learning to Route LLMs with Preference Data* — ICLR 2025 (arXiv:2406.18665) — UC Berkeley / Anyscale / Canva — 2024.

**One-line definition.** A **binary router** trained on Chatbot Arena human preferences (optionally augmented with golden MMLU or GPT-4–judge labels) that predicts \(P(\text{win}_s \mid q)\) and applies a **cost threshold** \(\alpha\) to route each query to a **strong** (expensive) or **weak** (cheap) LLM — a single forward pass per query, no multi-model cascades at inference.

## Why this paper matters (the open-source LLM routing baseline; LMSYS/Chatbot Arena leverage)

**Open** framework to **train, serve, and evaluate** routers (authors’ release) on **~80K** **LMSYS** **Chatbot Arena** battles instead of private **(query, correct)** corpora. **Decontaminated** evals on MMLU (**14,042** q, **5-shot**), MT-Bench (**160**, LLM-judge), GSM8K (**8-shot**). **Zero-shot** swap of **\(M_s/M_w\)** to Claude **3** or Llama **3.1** pairs **without** retraining — **query-centric** signal, not memorized **model** IDs.

## Problem it solves (cost-quality routing without training data; needs preference data not (cost, correct) labels)

Frontier vs small models can differ by **\(\sim\)50×+** per token (paper: e.g. Claude Haiku vs Opus). **Gold** labels alone don’t define *which* backend to invoke — you need **relative** quality \((q, l_{s,w})\), \(l_{s,w}\in\{\text{win}_s,\text{tie},\text{win}_w\}\). Arena supplies this at scale; gaps are filled with **MMLU-val** agreement or **GPT-4 judge** on Nectar (**~120K** pairs, **~\$700**).

## Core idea in one paragraph

Train \(\theta\) by **maximizing** \(\sum_{(q,l)\in\mathcal{D}} \log P_\theta(l \mid q)\) on preferences, then **route** with threshold \(\alpha\in[0,1]\): \(R^\alpha(q)=M_s\) iff \(P_\theta(\text{win}_s\mid q)\ge \alpha\), else \(M_w\). Quality vs **always-weak** / **always-strong** is **PGR**\(=\frac{r(M_R)-r(M_w)}{r(M_s)-r(M_w)}\); **APGR** averages PGR over **10** cost bins (paper’s discrete form of \(\int_0^1 \mathrm{PGR}\, d(c)\)). **CPT\((x\%)\)** is the **minimum** **strong-call** % to hit PGR \(x/100\). One **downstream** LLM call per query — no **FrugalGPT**-style **multi-query** cascade (lower **latency**, **no** serial **billing** chain).

## Mechanism (step by step)

**(a) Four \(P_\theta(\text{win}_s \mid q)\) heads.** **SW:** each train pair contributes to Bradley–Terry with weight \(\omega'\propto \gamma^{1+S(q,q')}\) on **cosine** \((q,q')\) in `text-embedding-3-small` space, **\(\gamma{=}10\)** — **optimize** \((\xi_s,\xi_w)\) at **inference**; then \(P(\text{win}_s\mid q)=(1+e^{\xi_w-\xi_s})^{-1}\). **MF:** \(\delta(M,q)=w_2^\top(v_m\odot(W_1^\top v_q+b))\); \(P=\sigma(\delta(s,q)-\delta(w,q))\). **BERT-Base** CLS **→** **sigmoid**; **Causal** **Llama 3 8B** adds **label** tokens, **softmax** over **L**. Hparams: **MF** Adam **64**, **3e-4**, ~**10** ep, **8GB**; **BERT** **2000** st, **512**, **1e-5**; **Causal** **8×A100**, **2000** st, **2048**, **1e-6**.

**(b) Data.** **\(D_\text{arena}\):** ~**80K** battles → **~65K** pairs (drop **&lt;16** char), **64** models, **5K** val; pair labels **&lt;0.1%** dense — **10** **tiers** (leaderboard + DP), **0–1** strong, **2** weak. Store **winner** id, not always full text. **\(D_\text{gold}\):** **~1500** MMLU-val MCQ. **\(D_\text{judge}\):** Nectar + Mixtral + GPT-4, judge de-bias per Zheng et al.

**(c) Routing decision.** Fixed \(\alpha \in [0,1]\):

```text
p ← P_θ(wins | q)          # from SW / MF / BERT / Causal-LLM head
if p ≥ α:  return M_strong(q)
else:      return M_weak(q)
```

Sweeping \(\alpha\) traces **% strong calls** vs **PGR**; **CPT(50%)** is the *minimum* strong-call percentage achieving **50%** of the (strong–weak) quality gap.

**(d) Inference cost.** Table 7 (Arena-sampled, includes embedding where needed): MF **$3.32**/M req, **155** req/s (L4 **g2-standard-4**); BERT **$3.19**, **70** req/s; Causal **$5.23**, **42** req/s; SW (CPU **n2-standard-8**) **$39.26**/M req, **2.9** req/s. Authors report SW overhead **≲0.4%** of GPT-4 generation cost at **95** in / **264** out tokens (**~$24.7** vs **$0.24** per 1M tokens for GPT-4-1106 vs Mixtral in their accounting).

## Empirical results (table — RouteLLM vs random vs cost-only on MT-Bench, MMLU, GSM8K; cost reductions e.g. 40% routing to weak still matching strong)

Eval pair **\(M_s\)** = `gpt-4-1106-preview`, **\(M_w\)** = **Mixtral-8x7B**. **CPT\((x\%)\)**: % of **strong** calls to reach PGR \(x/100\) (lower = cheaper at fixed quality). **APGR**: integrated PGR–cost (higher better). **“Cost-only”** is approximated by **Random** (same call budget, random strong/weak).

**MT-Bench (Table 1).**

| Training | Method | CPT(50%) | CPT(80%) | APGR |
|----------|--------|----------|----------|------|
| — | **Random** | **49.0%** | **78.1%** | **0.50** |
| \(D_\text{arena}\) | SW / MF (best unaug.) | 37.9% / **25.3%** | 59.0% / 74.3% | 0.610 / 0.580 |
| \(D_\text{arena}+D_\text{judge}\) | **MF** / BERT (best aug.) | **13.4%** / 19.6% | **31.3%** / 34.0% | **0.802** / 0.751 |

**MMLU (Table 2):** Arena-only **≈ random**; **+ \(D_\text{gold}\)** lifts all — e.g. SW **CPT(50%) 35.4%**, **APGR 0.603** (vs Random ~**50%** CPT(50%)). **GSM8K (Table 3):** Arena-only weak; **+ \(D_\text{judge}\)** best is **Causal** (**CPT(50%) 33.6%**, **APGR 0.622**; strong-call share **~17%** below Random at **CPT(50%)** in paper text).

**Savings vs always-strong:** Table 6 “cost saving ratio” **3.66×** (MT-Bench), **1.41×** (MMLU), **1.49×** (GSM8K) at CPT(50) quality targets **~95% / 92% / 87%** of raw GPT-4 on each set. **vs Random / commercial:** up to **~75%** fewer strong calls than Random on MT-Bench (headline, \(D_\text{arena}+D_\text{judge}\) MF); Appendix E: **~40%** fewer **GPT-4** calls than Unify / Martian at matched MT-Bench quality. **Example operating point:** **MF + judge** on MT-Bench **CPT(50%) 13.4%** strong → **~86.6%** traffic to the weak model while recovering half the (strong–weak) gap; abstract **&gt;2×** end-to-end savings is consistent.

## Variants and ablations

**Arena-only:** high-capacity **BERT/causal** ≈ **random** on MT-Bench; **MMLU/GSM8K** random — **OOD** + **small** data. **+gold** (**&lt;2%** of rows) still **+14–21%** APGR on MMLU; **+judge** fixes MT-Bench/GSM8K. **MF** best MT-Bench **with judge**; **Causal** best **GSM8K**. **Similarity** score (max-cos of eval→train) **0.61** (MT) vs **0.48/0.49** (MMLU/GSM) explains gap. **Table 4** unseen pairs: **CPT(80%)** **~0.5×** Random — **portable** query router.

## Failure modes and limitations

**Shift** off benchmark mixes; **binary** only — **N-way** / **tools** TBD. **No** clear **winner** arch (paper: wide variance, unexplained). **Tiers** = **noisy** weak/strong. **Judge** bias/cost; **gold** / eval **overlap** (they **decontaminate** eval). **SW** = **high** $/req — use **MF/BERT** for **QPS** if needed.

## When to use, when not to

**Use:** **pairwise** prefs, **one** call/turn, **strong/weak** cost gap, **chat-like** or **add** small in-domain **gold/judge** for skew. **Avoid:** expect **zero-shot** on **unlike** task mix; confuse with **RM** (post-gen) or **multi-LM** **cascades**; need **ultra-cheap** CPU **routing** at huge QPS — **measure** **MF/BERT** first.

## Implications for harness engineering. Reference [86-frugalgpt](86-frugalgpt.md), [88-confidence-driven-router](88-confidence-driven-router.md), [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md). Position as: the natural next step from FrugalGPT — preference-data routing is the bridge from heuristic cascades to learned routing

Per [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md), a harness must balance **models, tools, and budgets**. [86-frugalgpt](86-frugalgpt.md) showed **cascades** to save cost — often **multi-call** and hand-tuned. **RouteLLM** is the next step: **learned gating** from **preference** data, **one** LLM call per request — i.e. moving from **heuristic** cascades to **supervised** routing. Versus [88-confidence-driven-router](88-confidence-driven-router.md), both ask whether the **cheap** model suffices; RouteLLM **fits** that boundary from **human/judge** pairs, not from **self**-confidence alone. In **agentic** stacks the router is a **budget policy** on the query, not a **verifier** or **planner**.

## Connections to other work in this corpus

Links [86-frugalgpt](86-frugalgpt.md) to **FrugalGPT / LLM-Blender / AutoMix** (paper): **single-call** vs **multi-query** routing. **Hybrid-LLM** / **Zooter**: **synthetic** or **RM** labels vs RouteLLM’s **Arena** **human** pairs + **augmentation**. **Arena** / **MT-Bench** culture: **train** and **eval**. [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md): **CPT/APGR** as **harness** metrics vs **Random** and vendors (App. E).

## Key takeaways

1. **Pairwise** prefs + **log-lik** train **P(wins|q)**; **α** → **one** **backend** call.  
2. **Tiers** + **aug** (tiny **gold** or **~120K** **judge**) fix **sparse** Arena and **OOD** benches.  
3. **CPT/APGR** make **cost/quality** **comparable** (e.g. **CPT(50%)=13.4%** **strong** for **MF+judge** on MT-Bench = **~86.6%** **weak** at half-gap).  
4. **Same** router, **new** **(M_s,M_w)**: still beats **Random** — **query** signal.  
5. **Router** cost **≪** **frontier** generation if you deploy **MF/BERT** instead of **embedding-heavy** **CPU** **SW** at scale. **Data > params:** **BERT** underperforms on **Arena-only**; **MF**+**judge** wins on **MT-Bench**.

## References

- Ong, I., Almahairi, A., Wu, V., Chiang, W.-L., Wu, T., Gonzalez, J. E., Kadous, W., & Stoica, I. (2024). *RouteLLM: Learning to Route LLMs with Preference Data.* arXiv:2406.18665. ICLR 2025.
- Zheng, L., et al. (2023). *Judging LLM-as-a-judge with MT-bench and chatbot arena* — **MT-Bench** / judge protocol.
- Chiang, W.-L., et al. (2024). *Chatbot arena* — **Arena** preference data.
- Hendrycks, et al. (2020). **MMLU**; Cobbe, et al. (2021). **GSM8K** (eval).
- Chen, L., Zaharia, M., & Zou, J. (2023). **FrugalGPT** — **cascade** baseline in related work.
- Project artifacts and code as cited in the paper (training / serving / evaluation).
