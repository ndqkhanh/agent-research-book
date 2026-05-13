# 86 — FrugalGPT: Cost-Aware LLM Cascades and Routing

**Paper.** Lingjiao Chen, Matei Zaharia, James Zou — *FrugalGPT: How to Use Large Language Models While Reducing Cost and Improving Performance* — Stanford University — arXiv:2305.05176 — 2023.

**One-line definition.** FrugalGPT organizes inference savings along three **orthogonal** axes — **prompt adaptation** (shorter or shared prompts), **LLM approximation** (caches and distilled smaller models that mimic expensive APIs on a task), and **LLM cascade** (sequential routing through cheap-then-expensive models with confidence-based early exit) — of which the **cascade** is the paper’s main algorithmic contribution and the pattern that subsumes learned routing under a budget.

## Why this paper matters (the foundational paper for LLM routing/cascading; precursor to RouteLLM, MoE routing, etc.)

It is a **foundational** budget-aware LLM paper: generative open answers (not fixed label sets), API cost as **input + output + per-request** terms, and **up to two orders of magnitude** spread on comparable token volumes across 12 commercial APIs in Table 1 (e.g. **\$30/10M** input tokens for GPT-4 vs **\$0.2/10M** for GPT-J on Textsynth, March 2023). The **LLM cascade** — ordered models, a **reliability score** \(g\), **thresholds** \(\tau\), **early exit** — is the same pattern later used by **learned routers**, MoE-style gating, and “cheapest-first” products. Empirically, cascades can sit on the **Pareto frontier** and even **dominate** one “best” API (cheaper *and* better when cheap models fix expensive ones’ errors).

## Problem it solves (cost dominates, accuracy varies)

Frontier LLM serving is **expensive** and **env/energy**-heavy at scale. **Price does not order accuracy monotonically** — the **MPI** matrices show e.g. on **COQA** **~13%** of items where **GPT-4** is wrong and **GPT-3** is right. The stated goal: maximize **expected reward** \( \mathbb{E}[r] \) s.t. **expected cost** \( \mathbb{E}[c] \le b \) over choices of **prompt, APIs, and stop rules**. “Always GPT-4” wastes budget and can lose to **cheaper** models on slices.

## Core idea in one paragraph

FrugalGPT proposes a **unified three-strategy** toolkit. **Prompt adaptation** attacks **prompt-token cost** by selecting compact in-context material or **batching** multiple user queries behind one shared instruction block. **LLM approximation** attacks **per-call API price** by reusing past completions (cache) or by **fine-tuning** a small open model on expensive-teacher labels so future calls avoid long prompts and costly APIs. **LLM cascade** attacks **expected spend per query** by running a **learned chain** of APIs from cheap to strong, accepting an answer when a lightweight **reliability score** clears a per-stage **threshold** and only then paying for the next tier. The paper’s main empirical instantiation, also named *FrugalGPT*, is an **LLM cascade** (length 3 in experiments) with a small regression model as scorer, trained to optimize thresholds and order under the budget.

## Mechanism (step by step)

Three strategies (Figures 1–2): **(a) prompt adaptation** — prompt **selection** and **query concatenation**; **(b) LLM approximation** — **completion cache** and **student fine-tuning**; **(c) LLM cascade** — cheap APIs first, **reliability score** and **thresholds** for escalation. **(c)** is the paper’s main implemented contribution.

### (a) Prompt adaptation

Cost scales with **prompt length**; many accuracy tricks (long few-shot, CoT) add tokens. **Prompt selection (Figure 2a):** keep a **small** subset of in-context exemplars; choose which examples per query class without tanking quality (combinatorial search). **Query concatenation (Figure 2b):** one system prompt, **multiple** user questions in one call so shared prefix is not re-billed; answers must map cleanly back to sub-queries.

### (b) LLM approximation

**Completion cache (Figure 2c):** if a **similar** past query exists, return cached answer; else call the API. Strong when traffic repeats; **semantic near-dup** errors or **stale** cache entries hurt. **Fine-tune student (Figure 2d):** label with **expensive** teacher, train **smaller** model; pay training once, save per-query tokens and API tier (drift if deploy ≠ train).

### (c) LLM cascade (key innovation)

**Ordered** APIs \(L\): for \(i=1..m\), get \(\hat{a}_i=f_{L_i}(q)\), score **\(g(q,\hat{a}_i)\in[0,1]\)** (Figure 2e). If **\(\ge \tau_i\)**, return \(\hat{a}_i\); else **escalate**. **Optimize** \((L,\tau)\) to maximize **reward** of the **stopping** answer with **\(\mathbb{E}[\text{cost}]\le b\)**; objective is **mixed-integer**; their solver **prunes** low-disagreement orderings and **interpolates** from samples. **Scorer in runs:** **DistilBERT** regression (cheap vs LLMs). **Example HEADLINES chain:** **GPT-J → J1-L → GPT-4** with thresholds **0.96**, **0.37** (Figure 3).

**Cascade pseudocode (paper semantics):**

```text
def llm_cascade(q, ordered_apis L[1..m], score_fn g, thresholds τ[1..m]):
    for i in 1..m:
        y = call_api(L[i], q)
        if g(q, y) >= τ[i]:
            return y, total_cost_for_calls(1..i)
    return y, total_cost_for_calls(1..m)   # all stages exhausted
# L, τ, and optionally the prompt map are learned to maximize E[reward] s.t. E[cost] ≤ b.
```

**Compositions:** joint **prompt+LLM** search; **API + fine-tuned** slots — larger **training** cost.

## Empirical results (concrete cost-quality Pareto, e.g. matched GPT-4 with 98% cost reduction)

**12 APIs**, **5** providers; tasks **HEADLINES** (finance, 10k, 8-shot), **OVERRULING** (legal, 2.4k, 5-shot), **COQA** (reading, 7,982, 2-shot). **FrugalGPT** = **3-stage** cascade, train/test split per dataset. **Table 3** — cost to **match** best single-LLM accuracy:

| Dataset   | Best single LLM | Best-LLM cost ($) | FrugalGPT cost ($) | **Savings** |
|-----------|-----------------|-------------------|--------------------|--------------|
| HEADLINES | GPT-4           | 33.1              | 0.6                | **98.3%**    |
| OVERRULING| GPT-4           | 9.7               | 2.6                | **73.3%**    |
| COQA      | GPT-3           | 72.5              | 29.6               | **59.2%**    |

**Abstract** also: **~4%** accuracy gain **at same** cost, or **to ~98%** cost cut vs best LLM; Figure 5 notes **up to ~5%** accuracy at matched spend in some plots. **HEADLINES** case **budget = \$6.5** (one-fifth of GPT-4’s **\$33.1**): same cascade **GPT-J→J1-L→GPT-4**; **~80%** cost cut vs always-GPT-4, **+1.5 pp** accuracy (Figure 3c). **MPI** (Figure 4): e.g. **GPT-C / GPT-J / J1-L** can lift error against **GPT-4** by up to **~6%** on **HEADLINES** — *cheap fixes expensive*. **Cost order is task-dependent** (e.g. **J1** second-priciest on **HEADLINES**, **GPT-3** second on **OVERRULING** / **COQA**) because **input vs output** rates interact with each task’s token mix; a costly API can still **lose** to a cheaper one — **routing** helps even without a hard budget.

## Variants and ablations (the three strategies independently and combined)

Strategies (1–3) are **orthogonal**; **Section 4** tests **cascade (length 3)** only, not a full factorial vs cache-only or prompt-only. Pareto plots stack **FrugalGPT** against **each single API**; no exhaustive “strategy × strategy” ablation. **Compositions** are proposed, not the main **numbers**.

## Failure modes and limitations

**Labels** for \(L,\tau,g\); bad labels → wrong accept/reject. **Same distribution** as deployment (shift kills scorer + thresholds). **Upfront** search/train cost; worth it when **query volume** ≫ setup. **Scorer** can force **full** chain (Figure 5c: all models agree **wrong** but low scores). **Not** about kernel **speedups** (quant, flash-attn, etc.); **latency, fairness, privacy, carbon** “future work” (Sec. 5).

## When to use, when not to use

**Use:** per-token $, **multi-tier** models, **label** or **proxy** for training scorer, **MPI**-style complementarity, budget caps. **Skip / weak:** tiny volume, **no** labels, one model already near-optimal, **tight** sequential-latency budget, or **policy** that forbids bypassing a fixed model.

## Implications for harness engineering

**Corpus cross-refs:** [87-routellm](87-routellm.md), [88-confidence-driven-router](88-confidence-driven-router.md), [29-dive-into-claude-code](29-dive-into-claude-code.md), [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md). **Position:** **routing/cascading** is one pillar of **harness cost engineering** in the [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md) framing — same harness surface, **different marginal $** per query when leaves are tiered.

[29-dive-into-claude-code](29-dive-into-claude-code.md) and [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md) document harness shape; **inference** at the leaves is still the **bill**. FrugalGPT: don’t over-provision every hop — **route** spend. [87-routellm](87-routellm.md) and [88-confidence-driven-router](88-confidence-driven-router.md) inherit the same **skeleton** — try cheap, **score / uncertainty**, escalate. A **model portfolio** becomes one **budgeted** service. **Practical:** log **cost, latency, outcome**; use a **cheaper** scorer; **order** by marginal error reduction; **retune** \(\tau\) when **prices** change (2023 table is a snapshot). Agent stacks map the same idea to **slots** (heuristic / mid / code-LM) inside one product.

## Connections to other work in this corpus

Ties to **test-time** “spend when needed,” **FrugalML**-style API **selection** (fixed labels vs this paper’s open generation), **synthetic** distillation, and **semantic** **caching**. Complements **router** and **governance** notes when **escalation** is a **policy** choice, not only a **price** knob.

## Key takeaways

1. **Three** levers: **prompt** compression/batching, **cache + distillation**, **cascade** \((L,\tau,g)\) under **budget** \(b\).  
2. **Table 3:** match best LLM for **98.3% / 73.3% / 59.2%** **lower** cost (HEADLINES / OVERRULING / COQA) at same accuracy; abstract adds **~4%** acc at **same** cost (Figure 5: **~5%** in places).  
3. **MPI** quantifies when **cheap** fixes **expensive** — routing’s fuel.  
4. **Honest** limits: **labels**, **shift**, **setup** cost, **latency** of chains, **scorer** mistakes.

## References

- Chen, L., Zaharia, M., & Zou, J. (2023). *FrugalGPT: How to Use Large Language Models While Reducing Cost and Improving Performance.* arXiv:2305.05176.  
- Related: Viola & Jones (2004) cascade; FrugalML / ML API selection (Chen, Zaharia, Zou, NeurIPS 2020 and follow-ons); distillation and prompting literature as cited in the paper’s Related Works.
