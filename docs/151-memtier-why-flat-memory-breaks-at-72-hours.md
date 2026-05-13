# 151 — MEMTIER (I): Why Flat Memory Breaks at 72 Hours — The 4 Stacked Failure Modes

**Paper.** Bronislav Sidik & Prof. Lior Rokach — *MEMTIER: Tiered Memory Architecture and Retrieval Bottleneck Analysis for Long-Running Autonomous AI Agents* — arXiv:2605.03675v1 [cs.AI] — submitted 5 May 2026 — Institute for Applied AI Research, Faculty of Computer and Information Science, **Ben-Gurion University of the Negev**, Beer Sheva, Israel — ACM classes I.2.11; I.2.7 — code "to be made available upon acceptance" — implemented as an open-source plugin for the **OpenClaw** agent runtime (>250,000 deployments; ref. issues #33406 and #62488).

**One-line definition.** MEMTIER is a tripartite memory architecture for long-running LLM agents that replaces the de facto industry standard of *flat-file Markdown memory* (a single MEMORY.md plus daily logs) with a structured **episodic JSONL store**, an asynchronously consolidated **semantic fact tier**, and a **five-signal weighted retrieval engine** with a PPO-trainable policy — explicitly motivated by, and benchmarked against, a measured **14-percentage-point tool-execution success degradation across 72-hour operation windows** that the authors trace to four compounding failure modes in flat-text memory.

This is **part one of three** on the MEMTIER paper. Part one is the diagnostic argument: *what is broken, why is it broken, and how do we know it is broken*. Part two ([152](152-memtier-3-tier-architecture.md)) walks through the architecture and retrieval engine. Part three ([153](153-memtier-llm-distillation-and-results.md)) covers LLM distillation, the LongMemEval-S results, and the three-layer invariance finding that names BM25 retrieval — not the model, not the weights — as the binding ceiling.

## Why this chapter matters

The harness-engineering canon on this site has spent forty-odd chapters arguing that the agent loop is constructed around four pillars: state, context, guardrails, entropy ([44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md)). Of those, **state** is the one that has been doing most of the load-bearing work in production deployments — agents survive across sessions because they read and write Markdown memory files (`CLAUDE.md`, `MEMORY.md`, `AGENTS.md`, daily logs, scratchpads). Files [09-memory-files](09-memory-files.md), [10-multi-session-continuity](10-multi-session-continuity.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md), [72-claude-mem-persistent-memory-compression](72-claude-mem-persistent-memory-compression.md), [78-ngc-neural-garbage-collection](78-ngc-neural-garbage-collection.md), [81-reasoningbank](81-reasoningbank.md), [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md), [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md), [130-distributed-sql-as-agent-memory](130-distributed-sql-as-agent-memory.md), and [131-temporal-bitemporal-tables](131-temporal-bitemporal-tables.md) all touch this.

What MEMTIER contributes is not "memory architectures help" — that is the bare minimum prior — but **a quantified failure curve and a forensic decomposition of why the de facto pattern breaks at scale**. The 14pp/72h figure is the first published longitudinal measurement of memory-induced degradation in a production-scale agent runtime. The four failure modes (context collapse, compaction discontinuity, structural blindness, no attribution loop) are the first taxonomy that survives contact with a 250 K-deployment runtime. And the empirical finding that *neither scaling the generator to a 284B MoE nor learning the retrieval weights via PPO with true credit assignment moves the needle* is, to my knowledge, the strongest evidence to date that **the retrieval architecture, not the language model, is the binding constraint for long-horizon agentic work**.

For builders, the chapter answers three questions the field has been asking out loud for two years:

1. *Is flat-Markdown memory good enough?* No. It degrades 14pp in 72h on the most widely deployed open-source runtime.
2. *Will a bigger model fix it?* No. DeepSeek-V4-Flash (284B MoE, 13B active) ties Qwen2.5-7B at the same retrieval layer.
3. *Will a smarter retrieval policy fix it?* No, not within BM25. PPO-learned weights ≡ default weights to four decimal places of accuracy.

The implication is direct: **if you are running a long-horizon agent, the next thing to fix is not the model, not the prompt, not the weights — it is the retrieval architecture.**

## Problem it solves

MEMTIER is the *first* paper that frames long-running agent memory degradation as a *measurable* engineering problem with a quantitative failure curve, a four-axis taxonomy, and a falsifiable architecture-level hypothesis. The five gaps in the prior literature it closes:

1. **No published longitudinal failure curve.** Industry essays asserted that long-horizon agents lose coherence (the BDTechTalks "art of harness engineering" piece, [40](40-harness-engineering-principles.md); the Adaline "missing product layer" piece, [41](41-product-control-plane.md)) but no paper put a number on the degradation. MEMTIER puts the number at **14 percentage points of tool-execution success across a 72-hour operation window** in the AgentRun-72 protocol.
2. **No mechanistic decomposition.** Prior work attributed decay variously to "context window saturation" (MemGPT, [Packer et al. 2023](https://arxiv.org/abs/2310.08560)), "drift" (Reflexion, [14](14-reflexion.md)), or "attention degradation" (the Todo/scratchpad pattern, [12](12-todo-scratchpad-state.md)). MEMTIER's four-mode taxonomy is operational: each mode is detectable in production telemetry, falsifiable, and addressed by a specific architectural change.
3. **No discriminating benchmark.** LoCoMo ([Maharana et al. 2024](https://arxiv.org/abs/2402.17753)) presents 30-session conversations, but it inserts the *full conversation* into context at query time, which makes the memory architecture irrelevant — MEMTIER's null result on LoCoMo (∆F1 = −0.005) replicates this issue. **LongMemEval-S** ([Wu et al. 2025](https://arxiv.org/abs/2501.08956); 500 questions × 53 sessions × 5 ability types) is the right benchmark because it forces *storage and retrieval* across 53 sessions without the haystack in context.
4. **No diagnostic separation of model from retrieval.** Until MEMTIER, scaling the LLM and tuning the retrieval policy had been studied together (RAG papers; agentic RAG, [25](25-agentic-rag.md)). The MEMTIER ablation deliberately disentangles them, holding architecture fixed across a 7B Qwen and a 284B DeepSeek MoE and across default-vs-PPO-learned retrieval weights.
5. **No principled attribution loop.** Prior memory systems wrote entries indiscriminately and never connected tool-call success or failure back to the memory entry that informed it. MEMTIER's *cognitive weight* signal ([−1, 1] per entry, attribution-attributed via SGLang logprobs in spec, lexical Jaccard in production) is the first concrete proposal for closing this loop in a production-scale runtime.

## The headline measurement: AgentRun-72

The 14pp/72h degradation figure is reported under the **AgentRun-72** protocol, defined by the authors against issues #33406 and #62488 in the OpenClaw issue tracker. The mechanics are simple but telling:

- Spin up an OpenClaw agent on a fixed task suite (the issue threads contain the suite).
- Let it operate continuously for 72 hours, with the default flat-text MEMORY.md cap of 20 KB and the standard daily JSONL append-only log.
- Score tool-execution success at four checkpoints: t=0, t=24h, t=48h, t=72h.

The success rate at t=0 is some baseline X%. At t=72h it is X − 14pp. The **causal mechanism** is not "the model got worse" — the model is the same — it is that the memory state that the model is conditioned on has degraded.

What is striking is the linearity. The degradation does not fall off a cliff; it accumulates. The four failure modes the paper names are each independent contributors that reinforce one another. Each mode is also independently falsifiable: a controlled experiment can isolate any one mode and measure its incremental effect.

## The four stacked failure modes

These four modes, in order of how the paper presents them and in roughly the order they bite over a 72h operation window:

### Mode 1 — Context collapse (the 20 KB cap)

OpenClaw stores per-session memory in a flat Markdown file at `~/.openclaw/workspace/memory/MEMORY.md`. The runtime imposes a 20 KB cap. When the file exceeds the cap, the runtime truncates *non-gracefully*: it does not summarise, it does not promote, it does not warn the user — it deletes the oldest entries to make room.

The consequence is *information non-preservation*. Entries that the agent wrote about facts it learned on day 1 are gone by day 3. There is no record that they ever existed; there is no compaction summary; there is no "promoted to long-term store" event. The agent's behavioural state at hour 72 simply lacks information that drove its behaviour at hour 0.

This mode is the simplest to detect in telemetry — the cap event has a measurable file-size threshold — but it is the easiest to overlook because the runtime does not surface a warning. From the user's perspective, the agent simply stops "remembering" certain things mid-session.

### Mode 2 — Compaction discontinuity (62% behavioural break rate)

The OpenClaw runtime, like Claude Code and most modern harnesses, performs *context compaction* — recursive or hierarchical summarisation of the prompt window when it approaches the model's context limit. This is a routine operation; it happens silently many times per session.

The MEMTIER team measured the *effect* of compaction on agent behaviour using a behavioural-break detector (the protocol details are in the §3 description but not fully replicated in the paper text shown; the headline number is reported across the longitudinal sample). They found that **62% of context-compaction events produce a measurable behavioural break** — a discontinuity in which the agent suddenly behaves as though it has forgotten a constraint, repeats a step it just completed, or contradicts a decision it just made.

This is structurally distinct from Mode 1. Compaction does not delete memory; it summarises the conversation buffer. But the summary is a *lossy compression of state*, and the loss is not deterministic. It depends on what the summariser model considers important, which depends on the prompt template and the model's own biases. So 62% of the time, something load-bearing falls out.

### Mode 3 — Structural blindness (entity vs incidental co-occurrence)

The third mode is the deepest. Flat-text retrieval — whether BM25 over a Markdown file or dense embedding over the same corpus — *cannot distinguish entity relationships from incidental co-occurrence*. If the agent's memory says "Customer Acme uses our /v3/ingest API" and "API: avoid /v3/ as it deprecates next quarter", BM25 will surface both for any query about Acme — and the model will then have to disambiguate, in-context, that "the deprecating API" and "the API Acme uses" are *the same API*.

This is structural. No amount of tuning BM25 weights helps. No amount of swapping in dense retrieval over the same flat text helps either, because the corpus itself does not encode the relations — it encodes only the surface tokens.

The only fix is *structurally* organising the memory: separating entities from facts about entities, separating *episodic* observations (what happened in this session) from *semantic* facts (what is true about the world that we have learned). This is the first principle that motivates the tripartite architecture.

### Mode 4 — No attribution loop

The fourth mode is the absence of *credit assignment from outcome to memory*. The agent calls tools; tools succeed or fail; but in flat-text memory, *the entry that informed the tool call is never credited or debited based on the tool's outcome*. The agent has no way to learn "this entry has helped me three times" or "this entry has caused four failures, demote it".

Without attribution, retrieval *cannot improve over time*. New entries are added but old entries are never re-evaluated. Bad entries persist and continue to mislead. Good entries are not preferentially surfaced. The system has no internal feedback mechanism — it is open-loop.

MEMTIER's response is the **cognitive weight** scalar: a number in [−1, 1] attached to every memory entry that accumulates positive evidence on tool successes and negative evidence on failures. The attribution can be done two ways: SGLang logprob attribution (which entries the model attended to most when producing the call) or lexical Jaccard fallback (which entries share tokens with the call). The resulting weight enters the retrieval scoring function as one of five signals.

## How the modes stack

The modes are not redundant. They compound *multiplicatively*, in a sense, over time:

```
t=0        t=24h               t=48h                       t=72h
│          │                   │                           │
│          │ Mode 1 truncates  │ Mode 2 lossily compacts   │ Mode 4: bad entries
│          │ early entries     │ what's left               │ never get demoted
│          │                   │                           │
│          │ Mode 3: structural blindness amplifies the
│          │ damage from 1 and 2 by making retrieval
│          │ less able to distinguish good from bad
│          │ entries even when they are present.
│          │
└──────────┴───────────────────┴───────────────────────────┘
```

A single mode might produce mild degradation. The four together produce 14pp.

The architecture's design follows directly: **tier the memory** (episodic vs semantic, addressing Mode 3), **promote rather than truncate** (consolidation daemon, addressing Mode 1), **encode in structure rather than free text** (JSONL with explicit fields, addressing Mode 2's lossiness during compaction), and **close the loop** (cognitive weights via attribution, addressing Mode 4).

## Why OpenClaw is the right runtime to measure this on

Three properties of OpenClaw make it the natural target:

1. **Scale.** 250 K+ deployments means the issue tracker has the long-tail bug reports needed to identify systemic failure modes, and the community has the breadth to confirm or refute claims against real workloads.
2. **Open-source under MIT.** The plugin can be merged upstream, and other researchers can replicate the AgentRun-72 measurement on the same runtime. Compare with the closed-source proprietary harnesses where this kind of measurement would not even be reproducible.
3. **Plugin architecture with `before_prompt_build` and `agent_end` hooks.** MEMTIER hooks into both: it injects retrieved memories into the prompt before it is built (intercepting the read path) and writes back attribution + cognitive-weight updates after the agent finishes (intercepting the write path). No core runtime modifications.

Compare with attempts to study memory degradation on Claude Code (proprietary) or Cursor (proprietary) — the measurement could not be reproduced by anyone other than the vendor. OpenClaw's openness *is the experimental apparatus*.

## What this paper is *not* claiming

A few clarifications, because the diagnostic findings are easy to misread:

1. **It is not claiming that all flat-Markdown memory systems degrade at 14pp/72h.** The number is specific to OpenClaw's MEMORY.md + daily logs configuration. Different runtimes will have different degradation curves. The point is that *some* such curve exists, and the four-mode taxonomy generalises.
2. **It is not claiming that MEMGPT-style OS-paging is the answer.** MEMTIER is explicitly contrasted with MemGPT: consolidation in MEMTIER is *asynchronous and policy-driven*, not interrupt-triggered, and the retrieval policy adapts via RL rather than remaining fixed.
3. **It is not claiming that knowledge graphs are the answer.** The semantic tier in MEMTIER is *flat distilled facts*, not a typed graph. The structural separation is between *what happened* and *what is true*, not between *entities* and *relations*. (Compare with [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md) and [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md), which advocate the typed-graph route.)
4. **It is not claiming that big models cannot help.** The three-layer invariance result (covered in part 3) shows that *within the BM25 retrieval architecture*, scaling the generator does not help. MEMTIER's recommendation for Phase 3 is dense retrieval, at which point a bigger generator will start to matter again.

## The benchmark argument

A side argument the paper makes — and one of its more methodologically important contributions — is **how to choose a memory benchmark**.

LoCoMo (200 QA pairs over 10-session conversations) is the most-cited memory benchmark in the field. MEMTIER evaluated on it and got a *null result*: F1 = 0.120 vs OpenClaw default 0.125, ∆F1 = −0.005. The finding the authors draw from this is **not** that MEMTIER doesn't work; it is that **LoCoMo cannot distinguish memory architectures because it puts the conversation in context at query time**. Both systems use the same context-stuffed prompt, so the memory system is irrelevant.

LongMemEval-S, by contrast, has 500 hand-crafted questions that require retrieval across 53 sessions, *with the haystack not present in context*. The five ability-type breakdown (single-session recall, multi-session synthesis, temporal reasoning, knowledge update, abstention) tests *what the memory architecture actually does for the agent*.

The recommendation: **future work on agent memory should report LongMemEval-S as the primary metric, with LoCoMo only as a secondary in-context-comprehension baseline.** This is one of those quiet methodological contributions that ends up shaping the next two years of measurement.

## Implications for harness engineering

Three concrete things change for harness builders if MEMTIER's diagnosis is correct:

### 1. Treat memory cap as an architectural choice, not a default

The OpenClaw 20 KB MEMORY.md cap is presented in the paper as a failure mode, but the deeper finding is that *any flat-cap policy will produce Mode 1 failures*. The remedy is not to raise the cap (the agent will still hit it eventually) but to *promote rather than evict*: when the file gets full, distil it into a higher tier rather than truncate it. This is what the consolidation daemon does (covered in part two).

If you are building on a harness with a hard cap (and most have one), you have three options: (a) implement consolidation now, (b) accept Mode 1 degradation as a known limitation, (c) move to a runtime that supports tiered storage natively. Option (c) is the long-run direction.

### 2. Treat compaction events as observability signals

The 62% behavioural-break-after-compaction rate is enormous. It says that the most common harness operation in long-horizon work — context compaction — is the single biggest source of agent incoherence.

This implies two practices for production:
  - **Log compaction events** with before/after summaries so post-mortems can correlate behavioural breaks with their cause.
  - **Avoid compaction when possible** by writing more state to durable memory (episodic JSONL) and less to in-context working memory. The harder you can lean on durable retrieval, the less you have to compact, and the fewer breaks you accumulate.

This connects to the broader observability practice from [24-observability-tracing](24-observability-tracing.md): if you cannot see compaction events, you cannot debug long-horizon failures. Most production tracing today does not log compaction. That is a gap.

### 3. Build attribution from day one

Mode 4 — no attribution loop — is the most insidious because it is *invisible*. There is no warning sign. The agent runs, the agent writes memory, the agent reads memory, and *bad memory entries silently corrupt downstream behaviour for the rest of the run* without any signal.

If you are designing a memory system today, even if you do not implement RL-based retrieval policy adaptation, **at minimum attach a counter to every memory entry that increments on co-occurrence with successful tool calls and decrements on failures**. The Jaccard fallback in MEMTIER is essentially this — a coarse but useful signal. It costs effectively nothing to maintain, and it gives you a knob to expose for later policy improvement.

The cognitive weight in MEMTIER is initialised at 0 (neutral) and updated at agent_end. The math is trivial. The architectural impact is enormous: now retrieval has *evidence* about which entries have proven useful, not just *recency*.

## Failure modes & anti-patterns

Things that will *not* fix the 14pp/72h problem, even though they are popular suggestions:

- **Bigger context window.** Longer contexts delay Mode 1 (truncation) but compound Mode 2 (compaction is more expensive when context is bigger) and do nothing for Modes 3 and 4. The MEMTIER LongMemEval-S full-context baseline is 0.050 — with the entire 53-session haystack in context.
- **Bigger model.** The three-layer invariance finding (part 3) explicitly tests this: DeepSeek-V4-Flash (284B MoE) ties Qwen2.5-7B (7B). The model is not the bottleneck *given the same retrieval architecture*.
- **Better RAG embedding.** Within BM25 retrieval over a flat corpus, embedding upgrades are marginal because Mode 3 (structural blindness) is corpus-level, not embedding-level.
- **Just a single semantic tier.** The semantic tier alone is project-shared and stripped of episodic detail. Without the episodic tier underneath, the agent loses the *grounding* in specific session contexts. Two tiers are required, not one.
- **Manual MEMORY.md curation.** Some teams have tried to fix degradation by hand-curating their MEMORY.md weekly. This *helps* but does not scale, and it does not address Modes 2–4. It is a temporary fix.

## When to pay attention to this chapter

If your agent runs for less than ~6 hours per session and is restarted between sessions, the four modes apply weakly. Mode 1 takes a while to bite (depending on writing rate). Mode 2 happens but is bounded. Mode 3 is always present but has less material to corrupt. Mode 4 has fewer entries to mismanage.

If your agent runs for more than 24 hours per session, or operates on a multi-week ticket / project / customer, all four modes are biting hard, and the quantitative degradation curve is approximately what MEMTIER reports.

The harder cases, where you should treat this chapter as a near-mandate:

- Customer-support agents that persist across multi-week tickets ([147-vendor-lock-in](147-vendor-lock-in.md))
- Coding agents on multi-day or multi-week tasks ([46-components-of-coding-agent](46-components-of-coding-agent.md))
- Research agents tracking a topic over weeks ([155-feynman-multi-agent-research-harness](155-feynman-multi-agent-research-harness.md))
- Operational agents in production substrate (24/7; [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md))
- Multi-agent orchestrations where shared state outlives any single subagent ([121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md))

## Where this fits in the harness-engineering arc

This is the chapter that closes the loop opened in [09-memory-files](09-memory-files.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md), and [72-claude-mem-persistent-memory-compression](72-claude-mem-persistent-memory-compression.md). Those chapters introduced flat-Markdown memory and persistent compression as harness primitives. MEMTIER says: *those primitives are now load-bearing in production, they are degrading measurably under load, and here is the diagnostic that points at the architectural fix.*

The architectural fix itself is part two ([152](152-memtier-3-tier-architecture.md)). The empirical evidence that the fix is correct, and the deeper finding that BM25 retrieval is the binding ceiling, is part three ([153](153-memtier-llm-distillation-and-results.md)).

It also pairs naturally with [81-reasoningbank](81-reasoningbank.md) (which proposes a different memory primitive — distilled reasoning patterns), [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md), and [130-distributed-sql-as-agent-memory](130-distributed-sql-as-agent-memory.md). MEMTIER and these chapters represent three different bets on what the right substrate is for long-running agent memory: tiered JSONL+facts (MEMTIER), distilled reasoning patterns (ReasoningBank), and typed graph + DB (KG-substrate). The next two to three years of agent infrastructure will be a tournament across these three bets.

## References

1. Sidik & Rokach, MEMTIER: Tiered Memory Architecture and Retrieval Bottleneck Analysis for Long-Running Autonomous AI Agents — arXiv:2605.03675v1.
2. OpenClaw Contributors. 2026. OpenClaw: Personal AI assistant framework — open-source agent runtime; issues #33406 and #62488 referenced for the AgentRun-72 protocol.
3. Wu et al. 2025. *LongMemEval: Benchmarking chat assistants on long-term interactive memory*. arXiv:2501.08956. The 500-question, 53-session benchmark MEMTIER uses as its primary evaluation.
4. Maharana et al. 2024. *Evaluating very long-term conversational memory of LLM agents*. arXiv:2402.17753. The LoCoMo benchmark MEMTIER explicitly argues against using as a memory architecture benchmark.
5. Packer et al. 2023. *MemGPT: Towards LLMs as operating systems*. arXiv:2310.08560. The OS-paging analogue MEMTIER differs from along two axes (asynchronous vs interrupt-triggered consolidation; RL-adaptive vs fixed retrieval policy).
6. Yao et al. 2023. *ReAct: Synergizing reasoning and acting in language models*. ICLR. The think/act/observe loop MEMTIER plugs into via OpenClaw's `before_prompt_build` and `agent_end` hooks.
7. Lewis et al. 2020. *Retrieval-augmented generation for knowledge-intensive NLP tasks*. NeurIPS. RAG ancestor of MEMTIER's retrieval engine; MEMTIER extends it with five-signal scoring and an RL-adaptable policy.
8. Robertson et al. 1994. *Okapi at TREC-3*. The BM25 baseline MEMTIER uses with k1=1.5, b=0.75, and which the paper ultimately identifies as the binding performance ceiling.
9. [09-memory-files.md](09-memory-files.md) — the prior canon on persistent memory files.
10. [40-harness-engineering-principles.md](40-harness-engineering-principles.md) — industry framing of harness state as a load-bearing architectural concern.
11. [72-claude-mem-persistent-memory-compression.md](72-claude-mem-persistent-memory-compression.md) — adjacent work on memory compression in Claude Code.
12. [128-knowledge-graphs-as-substrate.md](128-knowledge-graphs-as-substrate.md) and [129-kg-rag-hybrid-retrieval.md](129-kg-rag-hybrid-retrieval.md) — alternative typed-graph route to MEMTIER's tiered-text route.
