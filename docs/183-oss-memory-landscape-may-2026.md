# 183 — OSS Memory Landscape (May 2026)

**Scope.** A comparative survey of every high-impact open-source agent-memory framework on the public internet in May 2026 — what each ships, what it costs, where each is the right answer, and which is the wrong choice for which use case. Mirrors the structure of [docs/176](176-skill-discovery-curator-oss-landscape-may-2026.md) (the OSS skill landscape).

**One-paragraph thesis.** Six systems carry the bulk of the open-source agent-memory ecosystem: **Mem0** (55K★, scope-stratified single-pass writes), **Letta** (22.5K★, MemGPT-successor with self-managed tiers), **Supermemory** (22.5K★, hosted-SaaS-plus-OSS with #1 MemoryBench rank), **Cognee** (17.1K★, cognitive-science GraphRAG with `forget` + `improve`), **Zep** (4.5K★, temporal-KG via Graphiti), **HippoRAG 2** (3.5K★, neurobiologically-inspired PPR fusion). Plus coding-agent-specialized memory systems such as **AgentMemory** (5.3K★, MCP/REST/hook-based persistent memory for coding agents), the Anthropic-centric **claude-mem** (65K★ as a Claude Code plugin), **MemGPT** (12K★ archival ancestor), **Memento** (research-grade M-MDP), and adjacent infrastructure (LangChain Memory, LlamaIndex memory, Weaviate, Qdrant, pgvector). No single system dominates every benchmark — **mem0 wins eval-setup-favourable LongMemEval; Zep wins controlled-stack LongMemEval; Supermemory wins its own MemoryBench; HippoRAG 2 wins multi-hop QA; AgentMemory reports strong LongMemEval-S retrieval for coding-agent traces**. The landscape conclusion: pick by *memory shape* (chat / coding / research / multi-modal), not by star count or aggregate benchmark. Polaris should adopt Zep + HippoRAG 2 patterns; Lyra should adopt mem0 or AgentMemory for coding/user-pref memory plus Letta-style tool-call writes where memory management must be explicit in the reasoning trace.

---

## §1 — The six load-bearing systems

### 1. `mem0ai/mem0` — 55,000★, Apache-2.0

**The most-starred.** Scope-stratified (User / Session / Agent), single-pass ADD-only extraction (April 2026 redesign), multi-signal retrieval (semantic + BM25 + entity-link), six storage backends (Qdrant default).

Self-reported benchmarks: **LongMemEval 93.4 / LoCoMo 91.6 / BEAM-1M 64.1**. Apples-to-apples on GPT-4o (per [n1n.ai](https://explore.n1n.ai/blog/ai-agent-memory-comparison-2026-mem0-zep-letta-cognee-2026-04-23)): **49.0% on LongMemEval — Zep beats it by 15pts**.

**Best for**: chat-shaped memory with clear `user_id` boundaries (personal assistant, customer support).
**Worst for**: trace-shaped memory, structured claims, temporal-validity-aware facts.

Full treatment: [docs/181](181-mem0-deep-dive.md).

### 2. `letta-ai/letta` — 22,500★, Apache-2.0

**MemGPT-successor.** Memory ops as agent tool calls (`write_to_archival`, `search_archival`, `update_core_block`); the agent manages its own tiers. v0.16.7 March 2026.

> *"Letta is the platform for building stateful agents: AI with advanced memory that can learn and self-improve over time."*

**Best for**: long-running agents whose memory management *is* part of the reasoning trace; debuggable / replayable memory.
**Worst for**: short ephemeral sessions where the agent shouldn't pay tool-call latency for memory ops.

### 3. `supermemoryai/supermemory` — 22,500★, MIT

**Hosted SaaS + OSS hybrid.** Cloudflare Workers + PostgreSQL backend, MCP server for Claude / Cursor / VS Code / Windsurf integration, ships **MemoryBench** as the cross-system A/B framework.

**Self-reported benchmarks**: #1 on LongMemEval (81.6%), LoCoMo, ConvoMem.

**Best for**: cross-IDE consumer memory; MCP-server-mediated agent integration; teams that want a hosted backend.
**Worst for**: privacy-critical / on-prem deployments (the SaaS path is the opinionated default).

### 4. `topoteretes/cognee` — 17,100★, Apache-2.0

**Cognitive-science GraphRAG.** Embeddings + graphs + cognitive-science approaches; four-op API (`remember` / `recall` / `forget` / `improve`); local-first via Neo4j or Kuzudb. v1.0.8 May 2026.

> *"Memory control plane for AI Agents in 6 lines of code"*

**Best for**: privacy-critical local-first deployments; complex multi-document environments where graph reasoning is load-bearing; periodic memory consolidation as a first-class operation.
**Worst for**: simple personalisation use cases where graph + ontology is overkill.

### 5. `getzep/zep` — 4,500★, Apache-2.0

**Temporal knowledge graph** via Graphiti substrate. `valid_at` / `invalid_at` fact validity windows; sub-200ms relationship-aware context retrieval; LangChain / LlamaIndex / AutoGen integrations.

**Benchmark**: 63.8% LongMemEval on GPT-4o (vs mem0's 49.0% same-stack).

**Best for**: facts that change over time; conversational memory where temporal validity is load-bearing; relationship-aware context (who-knows-whom-when).
**Worst for**: stateless or single-shot memory needs; flat-text recall use cases.

### 6. `OSU-NLP-Group/HippoRAG` — 3,500★, MIT

**Neurobiologically-inspired PPR fusion.** Knowledge graph + dense embedding + Personalized PageRank; ICML 2025 paper ([arXiv:2502.14802](https://arxiv.org/abs/2502.14802)).

**Benchmark**: SOTA across NaturalQA + 2WikiMultiHop + MuSiQue + NarrativeQA + HotpotQA + PopQA simultaneously. The strongest non-parametric continual-learning result.

**Best for**: factual + multi-hop QA where the knowledge base is structured documents; research-shaped memory (Polaris Program Graph).
**Worst for**: conversation-shaped memory with no clear entity structure.

---

## §2 — Adjacent / specialised systems

### `pixegami/claude-mem` — 65,000★ (as Claude Code plugin)

Anthropic-centric. Session-capture → semantic-summary → progressive-disclosure retrieval. The most-installed agent-memory plugin specifically for Claude Code.

Full treatment: [docs/72](72-claude-mem-persistent-memory-compression.md).

### `rohitg00/agentmemory` — 5,300★, Apache-2.0

Coding-agent-specialized. Runs as a local memory engine plus MCP/REST server for Claude Code, Cursor, Gemini CLI, Codex CLI, OpenCode, Cline/Roo, Goose, Aider, and other MCP/HTTP clients. The repo reports hybrid BM25 + vector + graph search, 12 auto hooks, 51 MCP tools, session replay, real-time viewer, SQLite + iii-engine storage, no external DB requirement, 827 passing tests, and LongMemEval-S retrieval of **95.2% R@5 / 98.6% R@10 / 88.2% MRR**. Treat the numbers as repo-reported until reproduced, but the shape is important: it is memory for coding traces and project context, not a generic product personalization API.

**Best for**: cross-session coding agents where the same project, architecture decisions, bugs, fixes, and test commands need to survive across tool sessions.
**Worst for**: application-level user memory, temporal CRM/support facts, or graph-heavy research corpora.

### `cpacker/MemGPT` — 12,000★+ (now letta-ai/letta)

Letta's predecessor. Still cited; the OS-level memory-paging metaphor that Letta inverted is the load-bearing prior art.

### `microsoft/Memento` (research repo)

The M-MDP / case-based-reasoning research stack. 87.88% GAIA Pass@3 without LLM fine-tuning. K=4 sweet spot for case retrieval.

Full treatment: [docs/106-109](106-memento-paper-theory.md).

### LangChain `langchain.memory` + LlamaIndex `llama_index.core.memory`

Framework-bundled memory primitives. Both ship `ConversationBufferMemory`, `ConversationSummaryMemory`, `EntityMemory`, vector-store memory. Adequate for prototyping; production stacks usually swap in mem0 / Zep / Letta.

### Vector store backends — Qdrant / Weaviate / Pinecone / Chroma / pgvector / Redis / Milvus / lancedb

Not memory frameworks per se — they're the substrate every framework above sits on top of. Worth noting:

| Backend | Stars | License | Where it shines |
|---|---:|---|---|
| **Qdrant** | 25K★+ | Apache-2.0 | Default for self-hosted production |
| **Weaviate** | 13K★+ | BSD-3 | Strong multi-tenancy; built-in modules for embedders |
| **Chroma** | 16K★+ | Apache-2.0 | Local dev / single-user |
| **pgvector** | 14K★+ | PostgreSQL | When memory must live next to relational data |
| **Milvus** | 32K★+ | Apache-2.0 | Massive scale (billion-vector) |
| **lancedb** | 5K★+ | Apache-2.0 | Columnar, embedded, modern API |

For Polaris and Lyra production: Qdrant default; pgvector when relational locality matters.

---

## §3 — Architecture comparison matrix

|  | mem0 | Letta | Supermemory | Cognee | Zep / Graphiti | HippoRAG 2 |
|---|---|---|---|---|---|---|
| **Stars** | 55K | 22.5K | 22.5K | 17.1K | 4.5K | 3.5K |
| **License** | Apache-2.0 | Apache-2.0 | MIT | Apache-2.0 | Apache-2.0 | MIT |
| **Memory model** | scope-stratified rows | self-managed tiers (core/archival/recall) | hybrid SaaS+OSS | KG + cognitive-science ops | temporal KG | KG + dense + PPR |
| **Write path** | single-pass ADD-only LLM call | agent tool call | API call | `remember()` ingest pipeline | event stream | document-ingest |
| **Conflict resolution** | retrieval-time ranking | agent decides | API-side | `improve()` consolidation | temporal validity windows | KG entity merge |
| **Forget / decay** | none (manual delete) | agent `evict` | API delete | first-class `forget` | timestamp expiry | none (KG mutation) |
| **Retrieval** | semantic + BM25 + entity-link | tool-call search | hybrid + RAG | graph traversal + auto-route | relationship-aware temporal | PPR + dense fusion |
| **Best benchmark** | LongMemEval 93.4 (own) | (no published) | LongMemEval 81.6 #1 (own) | (none published) | LongMemEval 63.8 (controlled GPT-4o) | NaturalQA / MuSiQue / 2WikiMH SOTA |
| **Native temporal facts** | ❌ | ❌ | ❌ | ❌ | **✅** | ❌ |
| **Native graph reasoning** | partial (entity-link) | ❌ | partial (RAG) | **✅** | **✅** | **✅** |
| **Local-first / privacy** | partial (self-host) | ✅ | partial | **✅** | partial | ✅ |
| **MCP server** | ❌ | partial | **✅** | ✅ (Claude Code plugin) | ❌ | ❌ |
| **agentskills.io skills** | **✅** | ❌ | **✅** | ✅ | ❌ | ❌ |

---

## §4 — Eval mess: why no one wins every benchmark

Same data, different rankings, depending on who's running the eval:

| Benchmark | Winner | Source |
|---|---|---|
| LongMemEval (mem0 stack, gpt-5-mini) | **Mem0** at 93.4 | mem0 paper |
| LongMemEval (controlled GPT-4o) | **Zep** at 63.8 (mem0 49.0) | n1n.ai 2026 |
| LongMemEval (MemoryBench) | **Supermemory** at 81.6 #1 | Supermemory bench |
| LoCoMo (mem0 stack) | **Mem0** at 91.6 | mem0 paper |
| LoCoMo (MemoryBench) | **Supermemory** #1 | Supermemory bench |
| ConvoMem | **Supermemory** #1 | Supermemory bench |
| BEAM-1M | **Mem0** at 64.1 | mem0 paper |
| NaturalQA / 2WikiMH / MuSiQue / NarrativeQA | **HippoRAG 2** SOTA | HippoRAG paper |

**Reading these together**: nothing is universally best. The aggregate ranking depends on (a) which model is the reasoner, (b) which embedder is wired in, (c) which benchmark, (d) who ran the bench. Treat published numbers as **upper-bounds achievable in the favourable configuration**, not as portable rankings.

The right approach for harness designers: **run [Supermemory's MemoryBench](https://github.com/supermemoryai/supermemory) on a sample of your own traces**. Pick the winner for *your* memory shape.

---

## §5 — Pick-by-shape decision tree

Memory shape determines the right system. The decision:

```
What shape is your memory?

├── Conversation-shaped (chat with clear user/session boundaries)
│   ├── Need temporal-validity facts? → Zep / Graphiti
│   ├── Heavy personalisation? → mem0 (or Supermemory if you want hosted)
│   └── Self-managed by the agent? → Letta
│
├── Document-shaped (large corpora, multi-hop questions)
│   ├── Multi-hop QA emphasis? → HippoRAG 2
│   ├── Local-first / privacy? → Cognee
│   └── Need relationship reasoning? → Cognee or Zep
│
├── Trace-shaped (agent action sequences, ReasoningBank-style)
│   ├── Should be in the reasoning trace? → Letta tool-call writes
│   ├── Distilled into skills? → see [docs/167-171] skill canon
│   └── M-MDP / case-based? → Memento
│
├── Research-shaped (typed claims + provenance + retraction)
│   └── → Polaris Program Graph + HippoRAG 2 retrieval
│
├── Coding-shaped (current code state, no temporal validity)
│   ├── Turnkey cross-agent memory? → AgentMemory MCP/REST + hooks
│   ├── User-pref layer? → mem0 user_id scope
│   ├── Project knowledge? → Cognee local-first
│   └── Cross-session memory? → Lyra L37-6 auto_memory + AgentMemory or mem0 hybrid
│
└── Multi-modal (images, audio, video alongside text)
    └── → Supermemory (best multi-modal pipeline today)
```

---

## §6 — Adoption recommendations

### For Polaris (research-shaped)

- **Primary memory layer**: keep the existing Program Graph + Provenance Ledger. Don't migrate.
- **Retrieval upgrade**: layer in HippoRAG 2's PPR fusion over the Program Graph. The strongest single retrieval upgrade for research-shaped memory.
- **Temporal validity**: adopt Graphiti's `valid_at`/`invalid_at` pattern in the Provenance Ledger — naturally extends the existing `RED-RETRACTED` tier.
- **Skip**: mem0, Supermemory, Letta. Wrong shape.
- **Use for telemetry/cross-system A/B**: Supermemory's MemoryBench harness.

### For Lyra (coding-shaped + user-pref)

- **User-pref layer**: adopt mem0 with `user_id` scope. Extends the v3.7 L37-6 auto_memory work cleanly.
- **Procedural / skill memory**: keep existing `lyra-skills/` + `lyra-core/memory/procedural` (SQLite FTS5).
- **Cross-session knowledge**: layer Cognee for graph-reasoning over project knowledge. Local-first matches Lyra's no-vendor-lock design.
- **Skip**: Letta tool-call writes — too disruptive for the current architecture; reconsider in v4.x.

### For Claude Code users (consumer / Anthropic-stack)

- **Out-of-the-box**: claude-mem (already most-installed; covered in [docs/72](72-claude-mem-persistent-memory-compression.md)).
- **Cross-IDE**: Supermemory MCP server.
- **Personalisation**: mem0 Claude Code skill.

---

## §7 — Three patterns to steal

### Pattern 1 — Scope stratification (mem0)

`user_id` / `session_id` / `agent_id` is the right cardinality for chat-shaped memory. Polaris's existing Program Graph effectively has this via program-id scoping; Lyra should adopt user/session cleanly in v3.8.

### Pattern 2 — Temporal validity (Graphiti)

`valid_at` / `invalid_at` is the cleanest pattern for facts that change. Polaris's RED-RETRACTED tier is a special case; the general pattern is broader.

### Pattern 3 — Cognitive-science ops (Cognee)

The four-op API (`remember` / `recall` / `forget` / `improve`) is the cleanest mental model for *all* memory systems. Polaris's heartbeat scheduler should expose this surface.

---

## §8 — Where this fits

- [docs/100](100-contextual-memory-is-a-memo.md) — the structural critique that frames the landscape.
- [docs/151-153 MEMTIER](151-memtier-why-flat-memory-breaks-at-72-hours.md) — retrieval architecture is binding.
- [docs/106-109 Memento](106-memento-paper-theory.md), [docs/79 Skill-RAG](79-skill-rag.md), [docs/81 ReasoningBank](81-reasoningbank.md) — adjacent retrieval-as-memory work.
- [docs/72 claude-mem](72-claude-mem-persistent-memory-compression.md) — Anthropic-centric memory plugin.
- [docs/128-129 KG / KG-RAG](128-knowledge-graphs-as-substrate.md) — the graph-reasoning lineage.
- [docs/130-132 Memory infra](130-distributed-sql-as-agent-memory.md) — the distributed-SQL / vector-CDC layer.
- [docs/181 — Mem0 Deep Dive](181-mem0-deep-dive.md) — single-system absorption.
- [docs/182 — Memory Frontiers 2026](182-memory-frontiers-2026.md) — the frontier-paper companion.
- [docs/184 — Strongest Memory Techniques Synthesis](184-strongest-memory-techniques-synthesis-may-2026.md) — folds these into the recommended stack.
- [docs/185 — Memory Integration Playbook](185-memory-integration-playbook.md) — how to actually adopt.
- [docs/187 — Multi-Agent Shared Memory Landscape](187-multi-agent-shared-memory-landscape.md) — the *multi-agent / adversarial* corner this single-agent landscape leaves out.
- [docs/186 — Mnema](186-mnema-witness-lattice.md), [docs/188 — Witness/Provenance Memory Synthesis](188-witness-provenance-memory-techniques-synthesis.md) — the provenance-aware extension to this landscape.
