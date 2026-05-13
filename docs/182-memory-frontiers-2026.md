# 182 — Memory Frontiers 2026: The Post-MEMTIER Wave

**Scope.** What's new in agent memory between [docs/151-153 MEMTIER](151-memtier-why-flat-memory-breaks-at-72-hours.md) (the binding-ceiling argument), [docs/106-109 Memento](106-memento-paper-theory.md) (M-MDP / case-based reasoning), and [docs/157 May-2026 synthesis](157-may-2026-synthesis-memory-and-skills.md). The post-canon wave: HippoRAG 2 (ICML 2025), Graphiti's temporal knowledge graph, Letta self-managed memory (MemGPT successor), MemoryBench / LoCoMo / LongMemEval / BEAM evaluation infrastructure, and the streaming-context-compression family.

**One-paragraph thesis.** Three frontier papers / systems define the May-2026 memory frontier beyond what the canon already covers: (1) **HippoRAG 2** — neurobiologically-inspired retrieval combining knowledge-graph construction + dense embeddings + Personalized PageRank, the strongest non-parametric continual-learning result on factual + sense-making + multi-hop benchmarks; (2) **Graphiti** (powering Zep) — temporal knowledge graphs with `valid_at`/`invalid_at` fact-validity windows, the cleanest published answer to the "yesterday-Alice-was-in-NY-today-LA" problem mem0's flat-row model cannot solve; (3) **Letta self-managed memory** — agents *that manage their own memory tier as tool calls*, a post-MemGPT iteration that exposes core/archival/recall as first-class agent operations rather than runtime infrastructure. Cross-cutting these three: **the evaluation infrastructure has finally caught up** — LoCoMo (Locomo benchmark), LongMemEval, BEAM (10M-scale), MemoryBench (Supermemory's open framework), HippoRAG's NaturalQA + 2WikiMultiHop + MuSiQue suite are now standard, and they reveal that *retrieval architecture diverges by 30+ percentage points across systems* on the same benchmark — the MEMTIER thesis fully validated. The right frame for 2026: memory is a *retrieval architecture* problem, not a memory-substrate problem.

---

## §1 — HippoRAG 2: non-parametric continual learning

**[arXiv:2502.14802](https://arxiv.org/abs/2502.14802)** — *From RAG to Memory: Non-Parametric Continual Learning for Large Language Models* (ICML 2025). Successor to [HippoRAG 1 (NeurIPS 2024)](https://arxiv.org/abs/2405.14831).

### Architecture

Three components, hippocampal-indexing-inspired:

1. **Knowledge graph construction** — entity + relation extraction over the corpus, materialised into a graph store.
2. **Dense embedding fusion** — passages embedded (default `BGE-M3` or `text-embedding-3`) and indexed; fused with KG nodes by entity grounding.
3. **Personalized PageRank ranking** — at query time, the query's entities seed a PPR walk over the graph; passages are ranked by PPR-weighted embedding score.

The biological metaphor is the *hippocampal index*: the hippocampus tags neocortical memories so retrieval traverses the index rather than scanning all memories. HippoRAG implements that as PPR over an entity graph.

### Benchmark gains

HippoRAG 2 reports SOTA across three categories:

| Category | Benchmarks | Result |
|---|---|---|
| **Factual** | NaturalQuestions, PopQA | "surpasses other methods across all categories" |
| **Sense-making** | NarrativeQA | leading; specific numbers in the paper |
| **Multi-hop / associative** | MuSiQue, 2WikiMultiHopQA, HotpotQA | leading |

The aggregate claim: **strongest non-parametric continual-learning result** across factual + sense-making + multi-hop simultaneously — a sweep no prior system achieves.

### Why it matters

HippoRAG 2 is the empirical answer to the "should I use a knowledge graph or vector RAG?" question that has lingered since [docs/128 KG-as-substrate](128-knowledge-graphs-as-substrate.md) and [docs/129 KG-RAG hybrid](129-kg-rag-hybrid-retrieval.md). The answer: **both, fused via PPR.** Knowledge-graph traversal handles multi-hop; dense embeddings handle factual recall; PPR is the right *fusion algorithm* (not RRF, not weighted-sum).

For Polaris: HippoRAG 2's PPR-over-entity-graph maps cleanly onto Polaris's existing Program Graph. Wiring HippoRAG retrieval over the Program Graph would unify research-claim retrieval (currently flat-text) with provenance-graph retrieval (currently structured-only) under one ranking algorithm.

For Lyra: HippoRAG 2's `BGE-M3` + PPR is a drop-in upgrade for the procedural-memory layer's flat BM25.

---

## §2 — Graphiti: temporal knowledge graphs

**[github.com/getzep/zep](https://github.com/getzep/zep)** — Apache-2.0, 4,500★. Graphiti is the open-source temporal-KG framework that powers Zep (Zep is the hosted product; Graphiti is the substrate).

### What's new

Each fact in Graphiti carries `valid_at` and `invalid_at` timestamps. The store *isn't* a snapshot at time T; it's a temporal graph where the same entity can have conflicting facts that are simultaneously *valid* and *invalid* depending on the query timestamp.

```
Alice
  ├── lives_in: New York   (valid 2024-01-01 → 2025-06-30)
  ├── lives_in: Los Angeles (valid 2025-07-01 → present)
```

A query "where does Alice live?" returns LA. A query "where did Alice live in 2024?" returns NY. **Mem0's flat-row store cannot distinguish these without a manual filter; Graphiti distinguishes natively.**

### The retrieval contract

Zep wraps Graphiti for sub-200ms relationship-aware context retrieval. The contract:

- Add chat messages, business data, app events as they occur.
- At query time, get **pre-formatted, relationship-aware context blocks** with temporal facts pre-filtered to the relevant window.

### Benchmark vs mem0

Per [n1n.ai 2026 comparison](https://explore.n1n.ai/blog/ai-agent-memory-comparison-2026-mem0-zep-letta-cognee-2026-04-23): on LongMemEval with **GPT-4o**, **Zep 63.8% vs mem0 49.0%** — a 15-point gap "driven by Zep's temporal knowledge graph, which stores fact validity windows rather than timestamped snapshots."

This is the strongest single piece of evidence that **temporal-validity-aware memory beats flat-row memory** at long-horizon scale. Polaris's existing bitemporal-table thinking ([docs/131](131-temporal-bitemporal-tables.md)) is the right shape; Graphiti is the open-source implementation.

### Why it matters

Three implications:

1. **For chat-shaped memory**, Graphiti's temporal model dominates mem0's flat model in apples-to-apples eval.
2. **For research-shaped memory** (Polaris Program Graph), the temporal layer matters even more — papers retract, claims update, evidence shifts. The `RED-RETRACTED` tier in Polaris's trust framework is a special case of temporal validity.
3. **For coding agents** (Lyra), the temporal model matters less — most code-context memory is *current state*, not "how it changed."

---

## §3 — Letta: self-managed memory

**[github.com/letta-ai/letta](https://github.com/letta-ai/letta)** — Apache-2.0, 22,500★. Successor to MemGPT. v0.16.7 March 2026.

### The architectural inversion

MemGPT (the predecessor) modelled memory as **OS-level paging**: core memory (always in context), recall memory (full chat history, retrieved on demand), archival memory (long-term store, retrieved on demand). The runtime managed the tiers; the agent saw a unified surface.

Letta inverts this: **the agent manages its own tiers as tool calls.**

```python
# Pseudo — exact API in docs.letta.com
agent.tools.memory.write_to_archival(text="Alice prefers oat milk")
agent.tools.memory.search_archival(query="dairy preferences")
agent.tools.memory.update_core_block(label="user_prefs", value=...)
```

The agent decides what to remember. The agent decides when to retrieve. The agent decides what to evict from core.

### Why it matters

Two consequences:

1. **The model becomes the memory-manager.** The same way [docs/179](179-skill-retrieval-routing-and-activation.md) argues skill *activation* is the model's job (progressive disclosure), Letta argues memory *management* is the model's job. This is the strongest implementation of the "let the LLM manage state via tool calls" thesis.
2. **Memory tier becomes part of the agent's reasoning trace.** Every memory write/read is a tool call in the trace; auditable, replayable, debuggable. Mem0's writes happen *out of band* via the extractor LLM call; Letta's writes are *in band* in the agent's own ReAct loop.

For Polaris (research-claim heavy): partial fit — claim emission is in-band already, but the auto_creator extracts skills out-of-band. Letta's pattern argues even auto_creator should run as a tool call.

For Lyra (coding-agent): strong fit — Lyra's existing tool-emit pattern naturally extends to memory tools. v3.8 should consider exposing memory.write_to_archival as a Lyra tool.

---

## §4 — Cognee: cognitive-science-inspired GraphRAG

**[github.com/topoteretes/cognee](https://github.com/topoteretes/cognee)** — Apache-2.0, 17,100★. v1.0.8 May 2026.

### Architecture

Cognee combines **embeddings + graphs + cognitive-science approaches** in a unified pipeline. The agent operations are a clean four-tuple:

- `cognee.remember(text)` — ingest into the cognitive pipeline.
- `cognee.recall(query)` — retrieve via auto-routed query selection.
- `cognee.forget(id)` — explicit deletion.
- `cognee.improve()` — periodic refinement (drift, redundancy, ontology gaps).

Local-first by default; Cognee Cloud as managed alternative. Storage backends include Neo4j (graph) and Kuzudb (embedded graph).

### Why it matters

Cognee fills a gap mem0 leaves: **explicit `forget` + periodic `improve`**. Mem0 has neither (no decay, no first-class consolidation). Cognee bakes both in.

For Polaris and Lyra: Cognee's `improve` operation is the cleanest published implementation of the [docs/151-153 MEMTIER §3](153-memtier-llm-distillation-and-the-three-invariants.md) "memory-management cycle" thesis — Polaris's heartbeat scheduler ([P28](../projects/polaris/POLARIS_V2_2_DEEP_RESEARCH_PLAN.md)) should call something Cognee-shaped.

---

## §5 — Supermemory + MemoryBench

**[github.com/supermemoryai/supermemory](https://github.com/supermemoryai/supermemory)** — MIT, 22,500★.

Supermemory's most interesting contribution isn't its own architecture (Cloudflare-edge + PostgreSQL + MCP server) — it's **MemoryBench**, an open-source benchmarking framework for comparing memory systems head-to-head:

> *"Compare Supermemory, Mem0, Zep — across LongMemEval, LoCoMo, ConvoMem"*

Supermemory ranks **#1 across all three benchmarks** in their published numbers (LongMemEval 81.6%, plus #1 on LoCoMo and ConvoMem). With the obvious caveat that Supermemory ships the bench.

### Why it matters

MemoryBench is the **first OSS framework that lets you A/B compare memory systems on your own data**, rather than reading vendor blog posts. For harness designers this is the load-bearing tool: when picking between mem0 / Zep / Letta / Cognee / Supermemory, run MemoryBench on a sample of your own traces; pick the winner.

[docs/183 §6](183-oss-memory-landscape-may-2026.md) lists the comparison axes MemoryBench exposes.

---

## §6 — The new evaluation infrastructure

Three benchmarks now define the field; understanding which to trust matters as much as the systems themselves.

### LongMemEval

**The de-facto standard.** Multi-turn conversations spanning days or weeks; agent must answer questions whose answers depend on facts from earlier sessions. ~75 tasks across "knowledge update," "reasoning," "abstention," "single-session-preference," and "multi-session." Subject to the eval-setup mess from [docs/181 §3](181-mem0-deep-dive.md#3--benchmark-claims) — different stacks report wildly different numbers.

### LoCoMo

**The conversational-recall benchmark.** ~15 multi-session conversations averaging 50K tokens. Question: can the agent recall facts after long stretches?

### BEAM (1M / 10M)

**The scale benchmark.** Synthetic but production-shaped: 1M and 10M memories, agent must retrieve. Mem0 reports BEAM-1M 64.1, BEAM-10M 48.6. Lower at 10M is the expected scaling pattern; the absolute numbers are a useful upper bound.

### ConvoMem

**The personalisation benchmark.** Tests whether agents learn preferences over a multi-week conversation. Supermemory's claimed #1.

### MemoryBench

**The cross-system runner** (Supermemory-published). Lets you eval any of the above on any system.

### NaturalQA / 2WikiMultiHop / MuSiQue / NarrativeQA / PopQA

**The factual / multi-hop / sense-making suite** HippoRAG 2 reports against. Less conversation-shaped; more knowledge-base-shaped. The right benchmark when the memory is *facts about the world* rather than *facts about the user*.

---

## §7 — The streaming context-compression frontier

A separate frontier worth flagging — not a single paper but a family of systems treating long contexts as **streaming + compressing**, not retrieval:

- **Anthropic Claude's auto-compaction** — the runtime summarises older turns when context fills.
- **NGC: Neural Garbage Collection** ([docs/78](78-ngc-neural-garbage-collection.md)) — KV-cache eviction as a learned GRPO action.
- **Memtier + LLM distillation** ([docs/153](153-memtier-llm-distillation-and-the-three-invariants.md)) — LLM-driven compression of cold-tier memory.
- **Claude-mem** ([docs/72](72-claude-mem-persistent-memory-compression.md)) — session-capture → semantic-summary → progressive-disclosure retrieval.

The shared insight: **context compression is memory by another name** — the "hot tier" of any tiered memory system is just the compressed prefix in the LLM's context. As context windows grow (Anthropic 1M context, Gemini 2M+), the line between "retrieved memory" and "compressed context" blurs.

For 2026 production: don't pick "memory framework vs context window" — pick *both*, with the framework managing the cold tier and the context window absorbing the hot tier.

---

## §8 — Five frontier patterns to absorb

Distilling the post-MEMTIER wave into actionable patterns:

| Pattern | Source | Polaris fit | Lyra fit |
|---|---|---|---|
| **Personalized PageRank fusion** of KG + dense | HippoRAG 2 | Strong (Program Graph) | Medium (procedural memory) |
| **`valid_at`/`invalid_at` temporal validity** | Graphiti / Zep | Strong (claim retraction) | Weak (current-state focus) |
| **Memory ops as agent tool calls** | Letta | Medium (claim emission already in-band) | Strong (extends tool pattern) |
| **Explicit `forget` + periodic `improve`** | Cognee | Strong (heartbeat Consolidation) | Strong (lyra-curator) |
| **MemoryBench cross-system A/B** | Supermemory | Strong (regression eval) | Strong (CI baseline) |

[docs/184](184-strongest-memory-techniques-synthesis-may-2026.md) folds these into the recommended stack.

---

## §9 — Where this fits

- [docs/100](100-contextual-memory-is-a-memo.md) — the structural critique that frames why retrieval architecture matters more than memory substrate.
- [docs/151-153 MEMTIER](151-memtier-why-flat-memory-breaks-at-72-hours.md) — the binding-ceiling argument.
- [docs/106-109 Memento](106-memento-paper-theory.md) — the M-MDP alternative.
- [docs/79 Skill-RAG](79-skill-rag.md), [docs/81 ReasoningBank](81-reasoningbank.md) — adjacent retrieval-as-memory work.
- [docs/128-129 KG / KG-RAG hybrid](128-knowledge-graphs-as-substrate.md) — predecessor to HippoRAG 2's PPR fusion.
- [docs/181 — Mem0 Deep Dive](181-mem0-deep-dive.md) — the most-starred system.
- [docs/183 — OSS Memory Landscape](183-oss-memory-landscape-may-2026.md) — the systems-side companion.
- [docs/184 — Strongest Techniques Synthesis](184-strongest-memory-techniques-synthesis-may-2026.md) — the recommended stack.
- [docs/185 — Integration Playbook](185-memory-integration-playbook.md) — concrete adoption.
