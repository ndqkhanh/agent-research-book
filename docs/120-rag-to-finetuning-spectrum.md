# 120 — RAG → Fine-Tuning Spectrum for Agents: Where Each Lives, the Hybrid Mid-Points, and the Decision Framework

**Sources.** Arsanjani & Bustos, *Agentic Architectural Patterns*, Chapter 3 (The Spectrum of LLM Adaptation for Agents: RAG to Fine-tuning); Devlin, *Building LLM Agents with RAG Knowledge Graphs*, Chapter 3; Lakshmanan & Hapke, *Generative AI Design Patterns*, Chapter 3 (RAG patterns) + Chapter 5 (Adapter Tuning); plus the foundational RAG literature (Lewis et al. 2020) and the recent RAG-vs-FT empirical work (Ovadia et al. 2024, Yang et al. 2026).

**One-line definition.** Adapting an LLM to a domain is not a binary choice between *RAG* (retrieval-augmented generation; knowledge in an external store) and *fine-tuning* (knowledge in weights), but a *spectrum* with at least six distinct positions — pure prompting, RAG, RAG+rerank, RAG+structured grounding, PEFT-on-domain-text, full fine-tune — each with different cost, latency, freshness, and capability trade-offs, and the right architectural choice depends on whether your problem is *knowledge gaps* (RAG wins), *skill / format / style* (PEFT wins), or *both* (hybrid is the answer).

## Why this matters

The RAG-vs-fine-tuning question is the most-asked, least-understood question in agent engineering. Most teams arrive at it after a frustrating week of prompt engineering: the model "almost" gets the domain right, but keeps making small errors, and the team needs to *adapt* the model. The naïve framing is binary — "should we RAG or fine-tune?" — and the answer is "neither, both, or something in between, depending on which problem you actually have."

The empirical literature (Ovadia et al. 2024) makes the distinction sharp: RAG excels at *rare-entity recall* and *current-knowledge tasks*; fine-tuning excels at *skill, style, and reasoning patterns*. A team that fine-tunes when they should have built RAG burns weeks of compute and ends up with a model that still doesn't know recent facts. A team that builds RAG when they should have fine-tuned ends up with a model that retrieves correctly but mis-formats its outputs.

This chapter is the spectrum, the per-position trade-offs, the decision framework that maps problems to positions, and the hybrid pattern that combines positions for compounding wins.

## Problem it solves

Six concrete situations the spectrum disambiguates:

1. **"Our model doesn't know our private docs."** Knowledge gap → RAG.
2. **"Our model gets recent news wrong."** Knowledge gap (freshness) → RAG with frequent re-indexing.
3. **"Our model writes in the wrong tone."** Skill/style → PEFT.
4. **"Our model formats output inconsistently."** Skill/format → PEFT plus constrained decoding.
5. **"Our model can't reason about our domain."** Skill (reasoning prior) → PEFT or full FT.
6. **"All of the above."** Hybrid: RAG for knowledge + PEFT for skill.

Each maps to a specific position on the spectrum.

## Core idea in one paragraph

There are six adaptation positions: **(1) Pure prompting** — system prompt + few-shot, no external retrieval, no weight changes. **(2) RAG** — retrieve relevant chunks at query time, condition the LLM on them. **(3) RAG with rerank / hybrid retrieval** — improve retrieval quality with cross-encoder rerankers and structured retrieval. **(4) RAG with structured grounding** — citations, faithfulness verification, refusal-on-unknown. **(5) PEFT** — small parametric updates (LoRA / adapters / prefix) on domain text or task examples. **(6) Full fine-tuning** — all weights updated; rare in agent stacks. Each position has a different cost, latency, freshness, and capability profile. RAG handles *knowledge gaps* (the model didn't see the doc); PEFT handles *skill gaps* (the model can't do the thing). They compose: RAG + PEFT is the default high-quality pattern for narrow domains. The decision framework is: characterise your gap as knowledge or skill (or both); pick the position that addresses it; layer where compounding helps.

## Mechanism (step by step)

### 1. The six positions in detail

**Position 1: Pure prompting.**
- *What it is*: system prompt, instructions, a few examples, no external knowledge, no weight changes.
- *Cost*: lowest.
- *Latency*: lowest.
- *Freshness*: bound by base model knowledge cutoff.
- *Capability ceiling*: bound by base model + prompt cleverness.
- *Right for*: tasks the base model already does well.

**Position 2: RAG (basic).**
- *What it is*: retrieve top-K relevant chunks from a vector store, prepend to prompt.
- *Cost*: prompting cost + retrieval cost; modest.
- *Latency*: prompting + retrieval round-trip; usually < 2× pure prompting.
- *Freshness*: as fresh as the index (re-index on doc change).
- *Capability ceiling*: bound by retrieval quality + LLM's ability to ground in retrieved text.
- *Right for*: knowledge-gap tasks (private docs, recent info, large knowledge bases).

**Position 3: RAG + rerank / hybrid retrieval.**
- *What it is*: retrieve broadly (vector + lexical), then re-rank with a cross-encoder for top-K final selection.
- *Cost*: + reranker compute (small).
- *Latency*: + reranker latency (moderate).
- *Freshness*: same as basic RAG.
- *Capability ceiling*: lifts retrieval precision 10–30% over basic RAG.
- *Right for*: high-stakes RAG, large knowledge bases, multi-domain queries.
- *Detail in*: [133-reranking-for-agentic-retrieval](133-reranking-for-agentic-retrieval.md), [134-semantic-indexing](134-semantic-indexing.md).

**Position 4: RAG + structured grounding (citations, faithfulness, refusal).**
- *What it is*: RAG with explicit citation generation, faithfulness verification, refusal-on-unknown.
- *Cost*: + verification compute; non-trivial.
- *Latency*: + verifier round-trip.
- *Freshness*: same as basic RAG.
- *Capability ceiling*: catches hallucination; ships citations.
- *Right for*: regulated domains, customer-facing answers, auditable outputs.
- *Detail in*: [135-trustworthy-generation](135-trustworthy-generation.md).

**Position 5: PEFT (LoRA / adapters / prefix).**
- *What it is*: small parametric update on domain corpus or task pairs.
- *Cost*: training cost (one-time, hours-to-days); inference cost ≈ same as base.
- *Latency*: same as base inference.
- *Freshness*: re-train when domain shifts.
- *Capability ceiling*: lifts skill/style/format; bounded by training data quality.
- *Right for*: skill / style / format / domain-specific reasoning patterns.
- *Detail in*: [116-adapter-tuning-lora-peft-for-agents](116-adapter-tuning-lora-peft-for-agents.md).

**Position 6: Full fine-tuning.**
- *What it is*: all weights updated.
- *Cost*: large (compute + dataset).
- *Latency*: same as base.
- *Freshness*: re-train when domain shifts (expensive).
- *Capability ceiling*: highest, but risks catastrophic forgetting.
- *Right for*: foundational changes to the model's behavior; rare in agent stacks.

### 2. The decision framework

Three questions:

**Q1: Is the gap knowledge or skill?**

- The model doesn't *know* something specific (a private doc, a recent fact, a rare entity) → knowledge gap.
- The model can't *do* something (output in style X, follow format Y, reason in domain Z) → skill gap.
- Both → hybrid.

**Q2: How frequently does the knowledge / skill change?**

- Knowledge changes daily → RAG with frequent re-index.
- Knowledge changes monthly → RAG with monthly re-index.
- Skill is stable → PEFT (re-train infrequently).
- Skill needs continual learning → consider Memento-style CBR ([107-memento-cbr-memory](107-memento-cbr-memory.md)) instead.

**Q3: What's the safety / verification requirement?**

- Outputs need citations → RAG + structured grounding (position 4).
- Outputs need refusal-on-unknown → RAG + grounding + refusal hooks.
- Outputs are internal-only, error-tolerant → basic RAG suffices.

### 3. The hybrid pattern — RAG + PEFT

The dominant high-quality pattern for narrow domains:

```text
[query]
   ↓
[retrieval: vector + rerank from domain corpus]   ← knowledge layer
   ↓
[PEFT-tuned model: domain skill, style, format]    ← skill layer
   ↓
[constrained decoding: structural guarantees]      ← output layer
   ↓
[verifier: faithfulness / citation check]          ← safety layer
   ↓
[output with citations]
```

Each layer adds compounding capability. Cost: highest. Capability: highest. Right for: production-grade narrow-domain agents.

### 4. RAG-or-not: the empirical rule of thumb

Ovadia et al. 2024's empirical finding: for rare entities and current-knowledge tasks, RAG > fine-tuning consistently. For compositional reasoning (multi-hop, novel combinations), fine-tuning > RAG (or at least is necessary in addition). This aligns with [100-contextual-memory-is-a-memo](100-contextual-memory-is-a-memo.md): retrieval has a structural ceiling on compositional generalisation.

For agent builders: **RAG for knowledge, PEFT for skill, neither for foundational reasoning capacity (use a bigger base model)**.

### 5. RAG → fine-tune transition signs

Three signs you should add PEFT on top of RAG:

- The model retrieves correctly but mis-formats outputs.
- The model retrieves correctly but reasons in the wrong style for your domain.
- The model retrieves correctly but consistently misses domain-specific implications.

In each, knowledge is correctly delivered but the model can't *use* it correctly. PEFT teaches the model how to use the knowledge it now has access to.

### 6. The freshness-cost trade-off

| Position | Time to ship | Time to update | Cost to update |
|---|---|---|---|
| Pure prompting | minutes | minutes | trivial |
| RAG | hours-days | minutes (re-index incremental) | low |
| RAG + rerank | days | minutes | low |
| RAG + grounding | days-weeks | minutes | low |
| PEFT | weeks | hours-days (re-train) | moderate |
| Full FT | months | weeks (re-train) | high |

RAG wins on freshness; PEFT wins on capability per dollar at inference; full FT is rarely justified in agent stacks.

### 7. Anti-patterns

- **Fine-tuning as a panacea.** Teams burn compute fine-tuning on tasks RAG would have solved.
- **RAG as a panacea.** Teams build elaborate retrieval for problems that were skill, not knowledge.
- **Fine-tuning the planner.** Almost always wrong; the planner is the broad-reasoning component.
- **Re-indexing too rarely.** Stale RAG silently degrades; monitor index freshness vs corpus update rate.
- **Re-training too often.** PEFT updates that don't move the eval bar are pure cost.

## Empirical anchors

- **RAG + PEFT compounding gains** of 5–15% over either alone on narrow-domain tasks.
- **Reranking lifts retrieval precision** 10–30% over vector-only.
- **PEFT-tuned 7B beats prompted GPT-4** on narrow tasks routinely (see [116-adapter-tuning-lora-peft-for-agents](116-adapter-tuning-lora-peft-for-agents.md)).
- **Citations + grounding cut hallucinations** dramatically when implemented correctly ([135-trustworthy-generation](135-trustworthy-generation.md)).
- **Full fine-tuning is rare in 2026 agent stacks** — most teams report PEFT is sufficient.
- **RAG freshness cost** is mostly storage + embedding re-compute; usually < $100/month for moderate corpora.

## Variants and counter-arguments addressed

- **"RAG is enough; never fine-tune."** True for many tasks; not for skill / style / format / domain-reasoning gaps.
- **"Fine-tuning replaces RAG."** Confuses skill with knowledge. Fine-tuning a model on the latest news is the wrong tool — RAG is.
- **"Hybrid is too complex."** It is more complex; the trade-off is capability. For high-quality narrow domains, the complexity earns its keep.
- **"Just use a bigger model."** Bigger base models help on broad reasoning; they don't solve "the model didn't see our private docs" or "we need outputs in our company's voice."
- **"Memento makes both obsolete."** Memento ([106-memento-paper-theory](106-memento-paper-theory.md)) avoids LLM fine-tuning specifically; it doesn't replace RAG, and it benefits from a PEFT-tuned retriever.

## Failure modes and limitations

1. **Wrong-axis diagnosis.** Picking RAG when the gap is skill (or vice versa) wastes weeks.
2. **Retrieval staleness.** Index drift kills RAG silently. Monitor.
3. **PEFT over-training.** Catastrophic forgetting on small datasets ([116-adapter-tuning-lora-peft-for-agents](116-adapter-tuning-lora-peft-for-agents.md)).
4. **Hybrid sequencing errors.** Running PEFT before retrieval is in place produces a model that's tuned to data it can't access.
5. **Cost surprise.** Hybrid stacks have many components; track cost per layer.
6. **Eval gap.** Without a domain-specific eval set ([115-evaluating-llm-systems](115-evaluating-llm-systems.md)), you cannot tell which position is winning.
7. **Provider RAG drift.** Hosted RAG offerings (e.g. provider-managed retrieval) change behavior silently; pin or self-host critical retrieval.

## When to use, when not

**Pure prompting** for tasks the base model handles well.

**RAG** for knowledge gaps; the default for any agent that touches private documents or recent information.

**RAG + rerank + grounding** for high-stakes, customer-facing, regulated, or auditable outputs.

**PEFT** for skill, style, format, domain-reasoning gaps with stable training data.

**Hybrid (RAG + PEFT)** for production-grade narrow-domain agents.

**Full FT** only when a foundational behavior must change.

## Implications for harness engineering

- **Diagnose before adapting.** Run an eval to characterise the gap; don't reach for RAG or PEFT reflexively.
- **Build RAG infra first.** Retrieval is the broadest enabler; PEFT compounds on top.
- **Treat PEFT adapters as production ML artifacts.** Versioned, evaluated, rollbackable. See [116-adapter-tuning-lora-peft-for-agents](116-adapter-tuning-lora-peft-for-agents.md).
- **Index-freshness monitoring.** Per-corpus update cadence vs reindex cadence; alert on drift.
- **Citations as a default.** [135-trustworthy-generation](135-trustworthy-generation.md). Customer-facing outputs without citations age badly.
- **Evaluate per position.** Each layer has its own eval (retrieval recall, PEFT skill score, citation accuracy). See [115-evaluating-llm-systems](115-evaluating-llm-systems.md).
- **Consider Memento-style CBR** as an alternative to PEFT when the data is task-trajectories with reward signal. See [107-memento-cbr-memory](107-memento-cbr-memory.md).

The one-sentence takeaway: **the RAG-vs-fine-tuning question is the wrong question — the right question is which gap (knowledge or skill) you have, which point on the spectrum addresses it, and where layering compounds.**

## See also

- [25-agentic-rag](25-agentic-rag.md) — agentic RAG with self-critique.
- [100-contextual-memory-is-a-memo](100-contextual-memory-is-a-memo.md) — the theoretical compositional ceiling of retrieval.
- [106-memento-paper-theory](106-memento-paper-theory.md) — fine-tuning agents without fine-tuning LLMs.
- [116-adapter-tuning-lora-peft-for-agents](116-adapter-tuning-lora-peft-for-agents.md) — PEFT mechanics.
- [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md), [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md) — KGs as a complementary retrieval substrate.
- [133-reranking-for-agentic-retrieval](133-reranking-for-agentic-retrieval.md), [134-semantic-indexing](134-semantic-indexing.md), [135-trustworthy-generation](135-trustworthy-generation.md) — the RAG-side depth.
- [115-evaluating-llm-systems](115-evaluating-llm-systems.md) — the eval that diagnoses gap type.
