# 154 — Ctx2Skill: Self-Evolving Context Skills via Adversarial Multi-Agent Self-Play

**Paper.** Shuzheng Si, Haozhe Zhao, Yu Lei, Qingyi Wang (equal contribution); Dingwei Chen, Zhitong Wang, Zhenhailong Wang, Kangyang Luo, Zheng Wang, Gang Chen, Fanchao Qi, Minjia Zhang, Maosong Sun — *From Context to Skills: Can Language Models Learn from Context Skillfully?* — arXiv:2604.27660v2 [cs.AI] — submitted 30 Apr 2026 (v1), revised 3 May 2026 (v2). Affiliations: **Tsinghua University (THU)**, **DeepLang AI**, **UIUC**, **Fudan University (FDU)**, **Chinese University of Hong Kong (CUHK)** — code at https://github.com/S1s-Z/Ctx2Skill.

**One-line definition.** Ctx2Skill is a self-evolving multi-agent framework that autonomously discovers, refines, and selects context-specific natural-language skills *from a single dense context document, without human annotation, without external feedback, and without parameter updates* — using a five-role adversarial self-play loop (Challenger, Reasoner, Judge, plus per-side Proposer/Generator pairs) whose two competing skill sets co-evolve through failure-driven textual edits, with a **Cross-time Replay** mechanism that selects the most generalisable skill set across iterations to prevent adversarial collapse. The resulting Reasoner skill set (a Markdown file prepended to the system prompt) is portable across any LM and lifts CL-bench solving rates by **+5.4 / +4.6 / +3.2 absolute points** on GPT-4.1 / GPT-5.1 / GPT-5.2 respectively.

## Why this paper matters

Two intersecting movements in the agent-engineering canon set this paper up.

**Movement 1 — Skills as the model's portable layer.** [04-skills](04-skills.md) introduced the SKILL.md format (model-invocable Markdown capability packages). [19-voyager-skill-libraries](19-voyager-skill-libraries.md) and [89-voyager-deep](89-voyager-deep.md) showed skill libraries as a curriculum-learning primitive. [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md), [71-karpathy-skills-single-file-guardrails](71-karpathy-skills-single-file-guardrails.md), [79-skill-rag](79-skill-rag.md) deepened the case. [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md) (the next chapter in this set) argues *the agentic harness itself can be distilled into a single skill*. The arc: skills are not a curiosity, they are the *trainable portable artifact* of the agent stack.

**Movement 2 — Self-evolving / self-improving agent training.** [36-autogenesis-self-evolving-agents](36-autogenesis-self-evolving-agents.md), [45-hyperagents-self-modification](45-hyperagents-self-modification.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md), [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md), and [81-reasoningbank](81-reasoningbank.md) are all variants of the question *can an agent improve itself without human labels?* The dominant answer has been "yes, via verifiable rewards" — RLVR, execution feedback, ground-truth comparison. But that whole line *requires verifiable feedback* — execution traces, code that runs, math problems with ground truth.

Ctx2Skill is the cross-product: **skill-based self-improvement without verifiable feedback**. It is the first paper, to my knowledge, that solves the problem of "the agent has only a long technical document; how does it learn to use that document well?" without (a) human annotation or (b) ground-truth task labels. The trick is to internalise the feedback loop: the *Judge* is itself an LM, and the *Challenger* generates the tasks and rubrics that the Judge evaluates against. The whole system is closed-loop in natural language.

The empirical headline — +5.4 absolute points on CL-bench for GPT-4.1, with **GPT-4.1 + Ctx2Skill (16.5%) beating Gemini 3 Pro without skills (15.8%)** — is a striking demonstration that *context-specific skills can bridge model capability gaps*. The 4.1B → "Gemini 3 Pro-equivalent" jump is the kind of result that, if it generalises, has practical implications for cost and deployment that are not subtle.

## Problem it solves

Five concrete problems Ctx2Skill addresses, each closing a gap in prior context-learning and skill-construction work:

1. **Long, technically dense contexts that exceed parametric knowledge.** Many real tasks need an LM to reason over a multi-section technical document, a regulatory specification, a product manual, a research dataset description. The relevant rules and procedures are embedded *implicitly*; surface retrieval is not enough. The CL-bench paper ([12]) frames this as *context learning* and shows that even frontier LMs solve only ~20% of tasks. *Ctx2Skill targets exactly this regime.*
2. **Prohibitive cost for manual skill annotation.** Existing agent skill libraries are predominantly built by human annotators reading the source documents and writing skill text. For long technically-dense contexts, this is cognitively demanding and economically infeasible. *Ctx2Skill does it without humans.*
3. **No external feedback for automated skill construction.** Unlike coding or math, context-learning tasks have no automatic feedback signal. You can't "execute" a skill against an interpretation rule the way you can execute Python. AutoSkill, AutoRefine, CoEvoSkills, EvoSkill, SkillX all rely on execution feedback or ground-truth comparison, which are unavailable here. *Ctx2Skill closes the loop with adversarial self-play.*
4. **Closed-source models cannot be parameter-tuned.** SKILL₀ and SkillRL internalise skills via in-context RL or distillation, but require parameter access — useless for GPT-5.1, Claude Opus 4.5, Gemini 3 Pro, Kimi K2.5. *Ctx2Skill produces a Markdown file that plugs into any frontier model unchanged.*
5. **Adversarial collapse in self-play loops.** Multi-agent self-play with co-evolving curricula often collapses: the Challenger gets too aggressive, the Reasoner over-specialises to pathological cases, generalisation degrades. *Ctx2Skill's Cross-time Replay mechanism is the explicit defence against this collapse.*

## Core idea in one paragraph

Treat skill construction as a *self-play game* between two co-evolving roles. A **Challenger** generates probing tasks and rubrics from the context, getting tighter as iterations progress. A **Reasoner** tries to solve them using a current skill set distilled from the same context. A **Judge** decides per-rubric pass/fail. Failed tasks route to a **Reasoner Proposer/Generator** pair that diagnoses missing contextual knowledge and edits the Reasoner's skill set; solved tasks route to a **Challenger Proposer/Generator** pair that tightens the Challenger's task-generation strategy. Iterate N times. To prevent adversarial collapse, run **Cross-time Replay**: collect a hard probe set (the worst failure each iteration) and an easy probe set (the simplest success each iteration), re-evaluate every iteration's skill set against both probes, and select the iteration whose skill set maximises the *product* of solving rates on hard and easy probes. The result is a single Markdown skill set that prepends to any LM's system prompt, encoding *reusable contextual knowledge of C* — and amortises across all unseen tasks over the same context.

## Mechanism (step by step)

### 1. Problem formulation

A *context learning task* is defined as ⟨C, T, R⟩ where:

- **C** is a context document (book, paper, manual, dataset description) whose content lies outside the LM's pre-training corpus.
- **T = {t_j}** is a set of tasks whose correct answers depend on C.
- **R_j = {r_{j,k}}** is the binary rubric set for task t_j; the task is *solved* iff every rubric passes.

Given an LM π and a context C, the goal is to produce a natural-language skill set S — a short Markdown file pre-pended to the system prompt — such that

```
a_j ~ π(· | S, C, t_j)
```

solves arbitrary unseen tasks t_j over the same context C, with no parameter updates and no external feedback during construction.

S is instantiated as two skill sets during self-play: **S^R** for the Reasoner, **S^C** for the Challenger. At inference time, only S^R is deployed.

### 2. The five-role adversarial loop

Each iteration i ∈ {1, …, N} runs five frozen-LM roles:

#### Challenger

```
Inputs:  context C, current Challenger skill set S^C_{i-1}
Outputs: batch of M tasks {t_m}, each with rubrics R_m
```

The Challenger generates tasks designed so that a correct answer requires *inducing the rules of C* rather than paraphrasing surface spans. By conditioning on its own evolving skill set S^C, the Challenger sustains adversarial pressure: as the Reasoner improves, the Challenger learns to probe deeper.

#### Reasoner

```
Inputs:  context C, task t_m, current Reasoner skill set S^R_{i-1}
Output:  answer a_m
```

The Reasoner reads C and produces an answer guided by S^R. The skill set S^R distils the most relevant rules and procedures from C into a concise structured form, so the Reasoner does not have to extract them from scratch on every task.

#### Judge

```
Inputs:  task t_m, rubrics R_m, answer a_m
Output:  per-rubric binary verdict z_{m,k} ∈ {0, 1}
         solving indicator y_m = ∏_k z_{m,k}
```

Strict all-or-nothing: the task is solved iff *every* rubric passes. The Judge is a frozen LM (in CL-bench, GPT-5.1, with cross-verifier and human agreement >90%).

The Judge partitions the batch into:
- **Failed cases** F_i = {m : y_m = 0}
- **Solved cases** P_i = {m : y_m = 1}

Each set routes to *its own side's* Proposer/Generator pair.

#### Proposer (one per side — Reasoner Proposer, Challenger Proposer)

The Judge says *which* tasks failed/solved, but not *why*. The Proposer's job is causal: collect the routed cases for its side and synthesise a high-level diagnosis specifying:
- an **action** (add or merge)
- a **target skill name**
- a **description**
- a **justification**

Crucially, the Proposer synthesises **across cases**, not in isolation: if multiple failed tasks share a missing piece of contextual knowledge, that diagnosis is a *single* skill update, not N redundant ones.

- The **Reasoner Proposer** takes failed cases F_i + S^R_{i-1} and diagnoses *which contextual knowledge is missing or misrepresented*.
- The **Challenger Proposer** takes solved cases P_i + S^C_{i-1} and identifies *gaps in the current task and rubric generation strategy* — i.e., where can the Challenger probe deeper next iteration?

#### Generator (one per side — Reasoner Generator, Challenger Generator)

The Generator materialises the Proposer's diagnosis into an actual skill set update. It returns a *complete replacement skill set* that adds or merges the diagnosed entries while *preserving every unrelated entry*.

- The **Reasoner Generator** updates S^R_{i-1} → S^R_i with the diagnosed missing knowledge.
- The **Challenger Generator** updates S^C_{i-1} → S^C_i to tighten task generation.

### 3. The strict adversarial discipline

Two design choices keep the loop honest:

1. **Per-side routing**: failed cases *only* update the Reasoner; solved cases *only* update the Challenger. The Reasoner never learns from successes (because that would reinforce existing skills rather than expose gaps), and the Challenger never learns from failures (because that would teach it to *avoid* the Reasoner's weaknesses rather than probe them).
2. **No cross-conditioning of prompts on the opposing side's skill set**. The Reasoner never sees S^C; the Challenger never sees S^R. They co-evolve adversarially without information leakage.

The whole loop is failure-driven textual edits: no parameter updates, no gradient steps, no embeddings recomputed. Just Markdown skill files getting longer and more refined each iteration.

### 4. Cross-time Replay — the defence against adversarial collapse

The risk in any adversarial self-play loop is that the curriculum becomes pathological. As iterations progress:

- Challenger generates increasingly extreme tasks that concentrate on the Reasoner's residual weaknesses.
- Those tasks gradually deviate from the *representative* knowledge of C.
- Reasoner's skills over-specialise to the pathological cases, accumulating redundancy that hurts generalisation.
- The collapse is *undetectable within the loop* because each iteration's Judge only evaluates the Challenger's newly-generated tasks — there is no signal that earlier-mastered knowledge has been corrupted.

Returning the last iteration's skill set S^R_N unconditionally would be unreliable. Cross-time Replay solves this by selecting from {S^R_1, …, S^R_N} the one that best balances representative cases.

#### How the probe sets are built

At each iteration i, two probes are added incrementally:

- **Hard probe set Q_h**: the *failed task with the lowest rubric pass rate* from iteration i. Captures the hardest concrete failure observed in that iteration.
- **Easy probe set Q_e**: the *solved task with the fewest rubrics* from iteration i. Captures the simplest success observed.

These probes are curated *during self-play*, without external supervision. After N iterations, Q_h has up to N hard probes and Q_e has up to N easy probes.

#### How the final skill set is selected

After self-play terminates, the Reasoner re-answers every task in both probes under each candidate skill set S^R_i. For each i, compute the **Laplace-smoothed solving rates**:

```
                  Σ_{q∈Q_h} y_q(π^R; C, S^R_i) + 1
    ρ_h(i)  =  ──────────────────────────────────
                          |Q_h| + 1

                  Σ_{q∈Q_e} y_q(π^R; C, S^R_i) + 1
    ρ_e(i)  =  ──────────────────────────────────
                          |Q_e| + 1
```

Then select:

```
    S*^R  =  S^R_{i*},   i*  =  arg max  [ ρ_h(i) · ρ_e(i) ]
                                    i
```

The **multiplicative form** is essential. A skill set that resolves late, idiosyncratic failures at the cost of regressing on easier tasks incurs a penalty on ρ_e(i) and is rejected. Conversely, a skill set that solves every easy probe but fails every hard one is rejected through ρ_h(i). Laplace smoothing keeps the product well-defined when a probe set is empty for a given iteration.

This mechanism is elegant precisely because it requires *no external supervision*. The probes come from the self-play itself; the selection criterion is internal.

### 5. Inference-time deployment

Once self-play terminates and Cross-time Replay selects S*^R, the deployment is trivially simple:

```
S*^R  is prepended to the Reasoner's system prompt
For any unseen task t_u over the same context C:
    a_u  ~  π(· | S*^R, C, t_u)
```

Three properties of this deployment:

1. **Reusability across tasks.** S*^R encodes reusable contextual knowledge of C, not task-specific solutions. It generalises to arbitrary tasks over the same context.
2. **Cost amortisation.** S*^R is produced *once per context* and reused across all |T| tasks for that context. Per-query inference cost is unchanged.
3. **Model portability.** S*^R is a Markdown file. Any LM that accepts a system prompt can use it. The paper demonstrates skill transfer between GPT-4.1 and GPT-5.1 (table 5 in the paper).

## Experimental setup and headline results

### CL-bench

Ctx2Skill is evaluated on **CL-bench** ([12], Si et al.):
- **500 complex contexts** spanning four categories.
- **1,899 tasks** total.
- **31,607 verification rubrics**.
- All-or-nothing scoring per task; only when *every* rubric passes is the task counted as solved.
- Four task categories:
  - **Domain Knowledge Reasoning**
  - **Rule System Application**
  - **Procedural Task Execution**
  - **Empirical Discovery & Simulation**
- LLM-as-a-judge: GPT-5.1, with cross-verifier and human agreement both exceeding 90%.

### Frontier-LM-without-skills baseline

| Model                  | Overall | Domain | Rules | Procedural | Empirical |
|------------------------|---------|--------|-------|------------|-----------|
| GPT-5.1                | 21.1    | 22.4   | 21.0  | 22.8       | 13.6      |
| Claude Opus 4.5        | 21.0    | 23.7   | 19.0  | 22.6       | 15.1      |
| Kimi K2.5              | 19.2    | 19.1   | 19.4  | 21.3       | 14.4      |
| GPT-5.2                | 18.2    | 19.5   | 18.0  | 19.1       | 12.1      |
| Gemini 3 Pro           | 15.8    | 15.5   | 17.7  | 16.4       | 10.1      |
| DeepSeek V3.2 Thinking | 13.2    | 13.6   | 13.8  | 14.2       | 8.0       |
| GPT-4.1                | 11.1    | 10.6   | 14.8  | 10.4       | 4.6       |

The headline observation: **even frontier LMs solve only ~20% of CL-bench tasks**. Context learning is *not* a solved problem for current models.

### Main results — Ctx2Skill lift across backbones

The paper compares three skill-construction methods using each backbone for both skill construction and inference:

| Backbone & method                    | Overall   | Δ vs base |
|--------------------------------------|-----------|-----------|
| **GPT-4.1 (base)**                   | 11.1      | —         |
| GPT-4.1 + Prompting                  | 12.3      | +1.2      |
| GPT-4.1 + AutoSkill4Doc              | 13.4 (+2.3) | …       |
| **GPT-4.1 + Ctx2Skill**              | **16.5**  | **+5.4**  |
| **GPT-5.1 (base)**                   | 21.1      | —         |
| GPT-5.1 + Prompting                  | 22.1      | +1.0      |
| **GPT-5.1 + Ctx2Skill**              | **25.8**  | **+4.6**  |
| **GPT-5.2 (base)**                   | 18.2      | —         |
| GPT-5.2 + Prompting                  | 19.1      | +0.9      |
| **GPT-5.2 + Ctx2Skill**              | **21.4**  | **+3.2**  |

The pattern: **Ctx2Skill consistently and substantially beats both Prompting (single-pass extraction) and AutoSkill4Doc** across all backbones. Single-pass extraction (Prompting) gives small lifts (+1.0 to +1.2) and sometimes regresses on individual categories (e.g., −2.5 on GPT-4.1 Rule System Application).

### The "small model + good skills > big model alone" observation

The paper highlights this directly: **GPT-4.1 with Ctx2Skill skills (16.5%) surpasses Gemini 3 Pro without skills (15.8%)**.

This is Ctx2Skill's most striking practical finding. A weaker model with carefully-distilled context-specific skills *outperforms a stronger model without them*. The implication for deployment cost is significant: instead of upgrading from GPT-4.1 to Gemini 3 Pro, you can stay at GPT-4.1, run Ctx2Skill once per context as a one-time amortised cost, and exceed the more expensive model's performance.

This generalises the precision/cost trade-off seen in MEMTIER's distillation finding ([153](153-memtier-llm-distillation-and-the-three-invariants.md)): high-quality structured artifacts compound and let smaller models punch above their weight class.

### Skill quality (intrinsic evaluation)

The paper assesses generated skills with GPT-4.1 as judge across five dimensions (faithfulness, clarity, coverage, conciseness, structure). Average scores:

| Backbone | Method            | Faithful | Clear | Coverage | Concise | Structure | Avg  |
|----------|-------------------|----------|-------|----------|---------|-----------|------|
| GPT-4.1  | Prompting         | 81.2     | 79.7  | 80.0     | 83.3    | 84.7      | 81.8 |
| GPT-4.1  | **Ctx2Skill**     | 85.2     | 84.8  | 96.2     | 90.5    | 92.5      | **89.8** |
| GPT-5.2  | Prompting         | 80.5     | 84.4  | 92.0     | 90.4    | 92.0      | 87.9 |

Ctx2Skill skills are scored as **+8.0 average** above single-pass Prompting on GPT-4.1. The biggest improvements: faithfulness (+4.0), clarity (+5.1), and coverage (+16.2). The iterative failure-driven refinement produces skills that more *accurately* and *completely* capture the contextual knowledge.

### Ablation — Cross-Time Replay matters

A key ablation result:

| Configuration                                | GPT-4.1 | GPT-5.1 |
|----------------------------------------------|---------|---------|
| Ctx2Skill (full, with Cross-Time Replay)     | 16.5    | 25.8    |
| − Cross-Time Replay (use last iteration)     | 14.7    | 23.0    |

Removing Cross-Time Replay costs **−1.8 on GPT-4.1, −2.8 on GPT-5.1**. This is direct empirical evidence that adversarial collapse is real: the last iteration's skill set is over-specialised and generalises worse than the cross-time-selected one. The mechanism is doing meaningful work, not just smoothing the table.

### Skill transfer across backbones

A nice property: skills are *portable* across LMs.

| Backbone | Skills source     | Overall |
|----------|-------------------|---------|
| GPT-4.1  | (none)            | 11.1    |
| GPT-4.1  | + GPT-5.1 skills  | 16.1    |
| GPT-4.1  | + GPT-4.1 skills  | 16.5    |
| GPT-5.1  | (none)            | 21.1    |
| GPT-5.1  | + GPT-4.1 skills  | 23.1    |
| GPT-5.1  | + GPT-5.1 skills  | 25.8    |

Cross-backbone transfer works (GPT-4.1 with GPT-5.1 skills gets +5.0; GPT-5.1 with GPT-4.1 skills gets +2.0), though same-backbone is best. Skills built by GPT-5.1 are slightly more transferable (presumably because the Reasoner uses them more demanding-ly during self-play, producing more rigorous distillations).

This portability is what makes skill libraries economically interesting: build once, deploy across many models.

## Implementation specifics

The paper's implementation parameters:

- **N = 5 self-play iterations** per context.
- **M = 5 tasks per iteration** generated by the Challenger.
- **Roles' backbone** matches the corresponding series (e.g., GPT-4.1 for all four agents — Challenger, Reasoner, Proposers, Generators — in the GPT-4.1-based methods).
- **Judge** uses GPT-5.1 consistently (matches CL-bench protocol).
- API budget constrained the exploration of larger N or M; the paper notes meaningful gains likely with more iterations and tasks.

The construction cost is dominated by inference calls: N × (M × 5 roles × API calls) ≈ a few hundred LM calls per context. At GPT-4.1 pricing this is on the order of $0.50–$2.00 per context — and amortises across all downstream tasks for that context.

## Why the architecture works

Pulling apart the design choices that make Ctx2Skill effective:

### 1. Failure routes to one side, success routes to the other

This is non-obvious. The naive design would be: both sides update on everything they see. The Ctx2Skill design says: *failures are evidence of Reasoner's gaps; successes are evidence of Challenger's leniency*. Routing accordingly creates a stable adversarial gradient.

### 2. Cross-case Proposer synthesis (not per-case)

The Proposer does not look at a single failure and patch it. It looks at *all* failures from this iteration and synthesises a high-level pattern. This prevents skill bloat (one skill addition per failure → unbounded growth) and produces skills that generalise (one skill addition explaining a class of failures).

### 3. Generators preserve unrelated entries

When the Generator returns a *complete replacement* skill set, it must preserve every unrelated entry. Without this rule, the skill set would oscillate as later iterations accidentally overwrite earlier wisdom. With this rule, skills accumulate monotonically.

### 4. Strict adversarial separation

No cross-conditioning. The Reasoner cannot learn the Challenger's strategy; the Challenger cannot learn the Reasoner's weaknesses except through observed task outcomes. This forces the loop to communicate via *task outcomes*, not via shared internal state — which is the discipline that makes the curriculum legitimate.

### 5. Cross-time Replay is the meta-stabiliser

Every other component drives the loop *toward* a fixed point. Cross-Time Replay protects against drift *away from* generalisation by selecting an interior iteration. Without it, the system has no way to recognise that S^R_3 was actually better-generalising than S^R_5 even though S^R_5 performs better on the curriculum-of-the-moment.

## What this paper opens up

If Ctx2Skill's pattern generalises, three downstream possibilities:

### 1. Per-context skill libraries as the deployment artifact

Imagine your enterprise-LLM deployment. You have 50 long product specs, regulatory documents, technical manuals. Each one gets one Ctx2Skill self-play run; the resulting Markdown skills are checked into a git repo. At inference time, the right skill prepends the prompt for queries against that document. The skills are stable, version-controlled, auditable, and portable across model upgrades.

### 2. Multi-context skill composition

Most tasks span multiple documents. The current Ctx2Skill paper is per-context. But the framework extends: run Ctx2Skill on each context independently, then at inference time, concatenate (or compose) the relevant skill sets. The Cross-Time Replay mechanism becomes a per-context evaluator; composition is the next research step.

### 3. Self-play loops as the default skill-construction primitive

Today, skill libraries are mostly hand-written or heuristically extracted. Ctx2Skill is the first credible argument that *adversarial self-play is the right primitive for automated skill construction*. The pattern (Challenger / Reasoner / Judge / Proposer / Generator + Cross-Time Replay) probably generalises well beyond context learning — to coding skills, browsing skills, tool-use skills, anywhere the skill quality matters.

## Limitations

The paper is forthright about three:

1. **API budget constraints prevented exploration of larger N or M.** Bigger self-play loops (N=10, M=10) are likely to produce stronger skills; the paper's headline numbers are a *lower bound* of what the method can achieve.
2. **The Judge is itself an LM.** Judge biases — known LLM-as-a-judge issues like positional bias, length bias, sycophancy — bound the loop's quality. The 90% human agreement reported on CL-bench is reassuring, not perfect.
3. **No verifiable feedback during construction.** This is by design — the whole point is to handle the no-external-feedback regime — but it means the system relies entirely on Judge LM correctness. For tasks where you *do* have verifiable feedback (code execution, math), a hybrid Ctx2Skill + verifiable signal would presumably do even better, but is not explored.

A fourth implicit limitation: the loop's compute cost. N=5, M=5, 5 roles → ~125 LM calls per context. For contexts where you have only one or two unseen downstream queries, the amortisation does not pay off. The method is most economical when you expect to handle many tasks per context.

## How Ctx2Skill relates to neighbouring work

The paper positions itself against several adjacent lines:

- **Reflexion / Self-Refine** ([14](14-reflexion.md), [18](18-chain-of-verification-self-refine.md)): per-episode in-context reflection. Ctx2Skill's skill set persists *across* episodes and over a *whole context*; reflection is per-episode and ephemeral.
- **Voyager / Atomic skills** ([19](19-voyager-skill-libraries.md), [68](68-atomic-skills-scaling-coding-agents.md)): skill libraries built from interaction traces. Ctx2Skill builds skills *from a static document*, not from interaction; it is the missing complement to Voyager-style libraries.
- **AutoSkill / EvoSkill / SkillX**: automated skill construction with execution feedback. Ctx2Skill explicitly handles the *no feedback* regime.
- **SKILL₀ / SkillRL**: parameter-internalising skill methods. Require parameter access; useless for closed models.
- **Memento case-bank** ([106-109](106-memento-paper-theory.md)): episodic case bank with soft-Q retrieval. Ctx2Skill is closer to "build an *index*" than "build a *case bank*"; the artifact is a single Markdown skill set, not a retrieved case set. Compatible: a system could have a Ctx2Skill skill prepended to the prompt *and* a Memento case bank for retrieval.
- **MEMTIER** ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)): structured memory architecture. Where MEMTIER is about *what the agent has experienced*, Ctx2Skill is about *what it has been told to know*. Both can coexist in a single agent.

## Anti-patterns and failure modes

Predictable ways to misuse Ctx2Skill:

- **Running it on a context with too few downstream tasks.** The 125-LM-call construction cost amortises only when you have many queries per context. For one-off lookups, it is wasteful.
- **Skipping Cross-Time Replay.** The −1.8/−2.8 ablation cost suggests this is not optional. The naive "use last iteration" loses meaningful performance.
- **Letting the Challenger and Reasoner share information.** The strict adversarial separation is what makes the loop work. Implementations that leak the opposing skill set into the prompt should expect collapse.
- **Using a weak Judge.** The Judge is the *only* feedback signal. A weak Judge produces noisy gradients; the whole system degrades. Use the strongest available LM for the Judge role.
- **Treating skills as model-specific.** The transfer table shows skills *do* transfer across models with some degradation. Build skills against a strong LM (e.g., GPT-5.1), deploy across a fleet of weaker models for cost savings.
- **Mixing context skills with coding skills.** Ctx2Skill produces *context-specific* skills. Mixing them with general procedural skills (e.g., HeavySkill ([156](156-heavyskill-parallel-reasoning-deliberation.md))) is fine but should be done at the system-prompt composition layer, not by mashing into the same Markdown file.

## When to use Ctx2Skill

The right use cases are:

- You have a long, technically dense, domain-specific document.
- You expect many downstream queries against it.
- The model's parametric knowledge is not enough to handle the document directly.
- You do not have human annotators to write skills.
- You do not have execution-style verifiable feedback for skill construction.
- You want a portable artifact (Markdown file) that works across models.

The wrong use cases:

- The document is short enough to fit in context as-is.
- You have one or two queries against it; amortisation does not pay.
- You have execution feedback (use AutoSkill / SkillX style methods that exploit it).
- You need skills that generalise across documents (Ctx2Skill is per-context).

## Where this fits in the harness arc

- **Read alongside**: [04-skills](04-skills.md), [19-voyager-skill-libraries](19-voyager-skill-libraries.md), [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md), [71-karpathy-skills-single-file-guardrails](71-karpathy-skills-single-file-guardrails.md), [79-skill-rag](79-skill-rag.md), [89-voyager-deep](89-voyager-deep.md), [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md).
- **Self-evolving family**: [36-autogenesis-self-evolving-agents](36-autogenesis-self-evolving-agents.md), [45-hyperagents-self-modification](45-hyperagents-self-modification.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md), [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md), [80-knowrl](80-knowrl.md), [81-reasoningbank](81-reasoningbank.md).
- **Multi-agent self-play patterns**: [20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md), [91-metagpt-deep](91-metagpt-deep.md), [92-chatdev](92-chatdev.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md).
- **Evaluation as a feedback signal**: [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [38-claw-eval](38-claw-eval.md), [115-evaluating-llm-systems](115-evaluating-llm-systems.md).
- **Synthesis next**: [157-may-2026-synthesis](157-may-2026-synthesis-memory-and-skills.md) — Ctx2Skill + MEMTIER + HeavySkill + Feynman as a coherent picture of the May-2026 harness/model boundary.

## References

1. Si, Zhao, Lei, Wang et al. *From Context to Skills: Can Language Models Learn from Context Skillfully?* — arXiv:2604.27660v2.
2. Code: https://github.com/S1s-Z/Ctx2Skill.
3. CL-bench (Si et al. 2026, [12] in paper) — 500 contexts, 1,899 tasks, 31,607 rubrics across four context-learning categories.
4. AutoSkill ([44] in paper), AutoRefine ([30]), CoEvoSkills ([45]), EvoSkill ([1]), SkillX ([40]) — automated skill-construction baselines requiring external feedback.
5. SKILL₀ ([23]) and SkillRL ([43]) — parameter-internalising skill methods requiring weight access.
6. Voyager ([41] in paper, [19](19-voyager-skill-libraries.md) in this canon) — interaction-trace skill libraries.
7. [04-skills.md](04-skills.md), [19-voyager-skill-libraries.md](19-voyager-skill-libraries.md), [68-atomic-skills-scaling-coding-agents.md](68-atomic-skills-scaling-coding-agents.md), [89-voyager-deep.md](89-voyager-deep.md) — the skill-library canon Ctx2Skill extends.
8. [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md), [21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md), [38-claw-eval.md](38-claw-eval.md) — the LLM-as-judge tradition Ctx2Skill's Judge inherits.
