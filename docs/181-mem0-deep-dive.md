# 181 — Mem0 Deep Dive: Universal Memory Layer for AI Agents

**Source.** [github.com/mem0ai/mem0](https://github.com/mem0ai/mem0) — **55,000 stars**, Apache-2.0, Python 55% / TypeScript 35% / MDX 4%. Self-described "Universal memory layer for AI Agents." Published companion paper claims headline gains over prior work; full benchmark numbers in §3 below.

**One-paragraph thesis.** Mem0 is the **most-starred open-source agent-memory framework on the internet** as of May 2026, and it is also the most architecturally opinionated: rather than match the classical *semantic / episodic / procedural* memory taxonomy, it treats memory as **scope-stratified state** (User / Session / Agent), uses a **single-pass ADD-only extraction** (one LLM call, no UPDATE/DELETE) introduced in the April 2026 redesign, and fuses **semantic + BM25 + entity-link retrieval** at lookup time. The April redesign is the load-bearing change: prior versions ran multi-pass write-side conflict resolution; the new design accumulates everything and resolves conflicts *at retrieval time* through ranking. This matters because every other framework in the space (Zep, Letta, Cognee, Supermemory) makes a *different* trade-off on this axis — see [docs/183](183-oss-memory-landscape-may-2026.md) for the full comparison. For Polaris and Lyra the question is not "should we use mem0?" but "is mem0's scope stratification + single-pass write a fit for *our* memory shape?" — and the answer turns out to be: yes for chat-shaped programs (Polaris's `mentat-learn` style); no for trace-shaped programs (Lyra's coding-agent ReasoningBank).

---

## §1 — Architecture: scope stratification + single-pass write

### Three scopes, not four taxonomic memory types

Mem0 explicitly **rejects the classical semantic/episodic/procedural taxonomy** ([docs/100](100-contextual-memory-is-a-memo.md) lays out why most deployed memory is just context-engineering anyway). Instead it stratifies by *scope*:

| Scope | Lifetime | Examples | Identifier |
|---|---|---|---|
| **User** | persistent across sessions | preferences, biographical facts, allergies, project ownership | `user_id` |
| **Session** | per-conversation | transient context, in-progress task state | `session_id` |
| **Agent** | per-agent across users | shared expertise the agent has accumulated, organisational knowledge | `agent_id` |

The shape this lifts from is *closer to a database's row-level multi-tenancy* than to cognitive psychology. Filters like `{"user_id": "alice", "session_id": "sess-7"}` are first-class on every API call.

### Single-pass ADD-only extraction (April 2026 redesign)

The headline architectural change in April 2026:

> *"Single-pass ADD-only extraction — one LLM call, no UPDATE/DELETE. Memories accumulate; nothing is overwritten."*

Mechanically:

1. A new conversation chunk arrives.
2. A single LLM call (default `gpt-5-mini`) extracts facts as candidate memory rows.
3. Rows are appended to the store, with an entity-linking pass to detect coreferential entries.
4. **No conflict resolution at write time.** Conflicting facts coexist.
5. At retrieval time, the **multi-signal ranker** (semantic + BM25 + entity-link match) surfaces the most relevant row.

This trade is interesting in both directions:

- **Pro:** single LLM call per write keeps latency low; UPDATE/DELETE conflict graphs are the source of half the bugs in earlier mem0 versions and in MemGPT-style systems.
- **Con:** without explicit decay or conflict resolution, the store grows monotonically. Stale facts compete with current ones at retrieval time. Ranking has to do *all* the work.

This is the opposite of [docs/151-153 MEMTIER](151-memtier-why-flat-memory-breaks-at-72-hours.md)'s prescription, which argues retrieval-architecture is the binding ceiling and a flat append-only store degrades 14 percentage points per 72 hours. The mem0 bet is that the multi-signal retrieval *is* the architecture — that ranking can absorb the noise a write-side conflict resolver would have removed.

### Multi-signal retrieval

Three signals fuse at retrieval time:

| Signal | What it scores |
|---|---|
| **Semantic** | dense embedding cosine — `text-embedding-3-small` default; supports any embedder via Embedder Protocol |
| **BM25** | classic keyword match for proper nouns, dates, exact phrases that embeddings smear |
| **Entity-link** | post-extraction entity graph; query terms that resolve to known entities boost matching memory rows |

The fusion is *score-additive* with configurable weights, not the cleaner reciprocal-rank-fusion (RRF) used in production search. This is a trade-off worth flagging: score-additive fusion is sensitive to score-distribution drift across embedder upgrades; RRF is more robust. (Polaris/Lyra adapters should probably wire RRF in front.)

---

## §2 — API surface

Three methods. The simplicity is the point.

```python
from mem0 import Memory

mem = Memory()

# Add — extract facts from a conversation chunk
mem.add(messages, user_id="alice")

# Search — retrieve top-k relevant memories
mem.search(query="what does alice prefer for breakfast?",
           filters={"user_id": "alice"}, top_k=3)

# Get — paginated row dump (rare in practice; mostly admin)
mem.get_all(user_id="alice")
```

That's the contract. Every framework integration (LangGraph, CrewAI, OpenAI Assistants, Claude, OpenAI/Anthropic SDKs) wraps these three calls.

CLI surface (`mem0-cli`) exposes the same three operations plus `delete`, `reset`, `stats`. Browser extensions (Chrome / Edge / Firefox) use the same API to provide cross-platform memory between ChatGPT, Claude.ai, Perplexity, etc.

---

## §3 — Benchmark claims (and the eval-setup mess)

The mem0 paper reports headline gains:

| Benchmark | Mem0 | Prior best | Δ |
|---|---:|---:|---:|
| **LoCoMo** | 91.6 | 71.4 | +20.2 |
| **LongMemEval** | 93.4 | 67.8 | +25.6 |
| **LongMemEval** (assistant recall) | — | — | +53.6 |
| **BEAM (1M scale)** | 64.1 | — | — |
| **BEAM (10M scale)** | 48.6 | — | — |

These are the published numbers from the mem0 paper running the mem0 stack with `gpt-5-mini`.

**But the eval-setup story is messier.** Two competing benchmarks tell a different story:

- **Zep's published comparison** ([Zep blog](https://www.getzep.com)): on `LongMemEval` with **GPT-4o**, Zep scores 63.8% vs mem0 49.0% — a **15-point gap in Zep's favour** (per the [n1n.ai 2026 comparison](https://explore.n1n.ai/blog/ai-agent-memory-comparison-2026-mem0-zep-letta-cognee-2026-04-23)).
- **Supermemory's MemoryBench**: ranks Supermemory **#1 at 81.6% on LongMemEval**, mem0 lower, with **Mem0 and Zep both** below the 81.6 mark.

Reading these together:
- Mem0's own 93.4 number is on the most favourable embedder + reasoner combination.
- Zep's 63.8 vs mem0 49.0 is on a controlled GPT-4o eval — the *equal-stack* comparison.
- Supermemory's 81.6 #1 is on a third benchmark (MemoryBench, which it ships).

**Practical takeaway**: the published mem0 numbers are real but **not portable** across stacks. Reproducing the 93.4 requires the exact `gpt-5-mini` + entity-link + multi-signal config. Apples-to-apples on GPT-4o, Zep wins. This is exactly the kind of evaluation-setup confusion [MEMTIER](151-memtier-why-flat-memory-breaks-at-72-hours.md) warned about: retrieval architecture > model choice, but you can't tell from the marketing.

[docs/184](184-strongest-memory-techniques-synthesis-may-2026.md) §3 unifies these into a recommended stack.

---

## §4 — Storage backends

Mem0's vector-store layer is pluggable; supported backends as of v2 (Apr 2026):

| Backend | Mode | Notes |
|---|---|---|
| **Qdrant** | self-hosted / managed | recommended default for >100K memories |
| **Pinecone** | managed | classic SaaS path; pay-per-query |
| **Weaviate** | self-hosted | strong multi-tenancy story |
| **Chroma** | local | dev / single-user |
| **pgvector** | self-hosted | when memory must live next to relational data |
| **Redis** | low-latency | cache tier; best for User scope |

Default embedder: `text-embedding-3-small` (OpenAI). Recommended for hybrid search: any **≥600M-parameter** embedder (Qwen-Embedding-600M cited as the floor below which BM25-fusion accuracy degrades).

### The reasoner / extractor

Mem0's fact-extraction LLM is configurable; default is `gpt-5-mini`. The single LLM call is the bottleneck on every write — most of mem0's per-write latency budget is the extractor call, not the vector insert. Self-hosted deployments swap in `Llama-3.3-70B` / `Qwen-3-32B` / `Mistral-Large` for cost.

---

## §5 — Integration surfaces

Mem0 ships first-class adapters for:

- **LangGraph** — checkpoint memory across nodes; example bot.
- **CrewAI** — task / agent memory with role-tailored output.
- **OpenAI Agents SDK** + **OpenAI Assistants API** — wraps the Assistants memory primitive.
- **Anthropic Claude** — Claude Skills for `mem0-cli`, `mem0-vercel-ai-sdk`. Claude Code, Cursor, Windsurf integrations as published [agentskills.io](https://agentskills.io/home) skills.
- **AutoGen / Microsoft Agent Framework** — community-maintained adapter.
- **Browser extensions** — Chrome / Edge / Firefox; cross-platform memory between ChatGPT, Claude.ai, Perplexity, Cursor (web).

Mem0's Claude Code skills are the cleanest path for *retrofitting* memory onto an existing Claude Code session — see [docs/185](185-memory-integration-playbook.md) §3.

---

## §6 — Anti-patterns and limitations

Mem0's README + docs are unusually candid about trade-offs. The notable ones:

1. **No explicit decay.** Long-term forgetting is *only* via retrieval ranking. Stale facts coexist with current ones forever; the ranker has to do the demotion. Mem0 publishes no recommended TTL; users implementing decay layer it manually via filtered `delete` calls.
2. **LLM-extractor dependency.** Fact quality directly correlates with extractor model capability. With weak extractors, garbage memories pollute the store with no write-side filter.
3. **Embedding-model sensitivity.** Hybrid search needs ≥600M-param embedders; smaller models break BM25-fusion.
4. **Single-pass design accumulates.** Without UPDATE/DELETE, redundant memories grow over time. The framework recommends periodic `delete` cleanup, but offers no first-class deduplication.
5. **Auth defaults on for self-hosted.** Local testing requires `AUTH_DISABLED=true` env var — easy to miss; bug reports cite this regularly.
6. **Score-additive fusion vs RRF.** Multi-signal fusion is configurable but defaults to score-addition; embedder upgrades that change score scale break the fusion silently. RRF would be more robust.

---

## §7 — Where mem0 fits (and doesn't)

### Fits well

- **Personal-assistant chatbots** with `user_id` stratification — the canonical mem0 use case.
- **Customer-support bots** where each session has clear scope boundaries.
- **Multi-agent systems** where `agent_id` carves out per-agent expertise.
- **Cross-platform consumer memory** via browser extensions.

### Doesn't fit well

- **Trace-shaped memory** (Lyra's ReasoningBank, EvoSkill failure traces) — mem0's three-scope model has no slot for "the agent's history of action sequences." Lyra's `ReasoningBank` is a different abstraction.
- **Structured-claim memory** (Polaris's Program Graph + Provenance Ledger) — mem0 has no equivalent of typed claim/evidence rows; the multi-signal retrieval is text-only.
- **Temporal-fact memory** — mem0 has no `valid_at`/`invalid_at` like Zep's Graphiti. *"Yesterday Alice was in NY; today she's in LA"* is a tier-2 retrieval problem mem0's stack doesn't solve cleanly.
- **Strict-budget cold-storage** — mem0 has no native tiering between hot working memory and cold archive; everything is one tier. MEMTIER's three-tier prescription ([docs/151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)) is unmet.

---

## §8 — Polaris / Lyra integration verdict

| Polaris module | Mem0 fit |
|---|---|
| `polaris-core/memory/program_graph` | **Poor.** Typed-claim graph; mem0 is text rows. |
| `polaris-core/memory/wiki` | **Partial.** Could front mem0 for the user-preference wiki layer. |
| `polaris-core/memory/reasoning_bank` | **Poor.** Reasoning items are typed (heuristic / trigger / score); mem0's text-row model loses the structure. |
| `polaris-core/memory/provenance` | **Poor.** Mem0 has no provenance row equivalent. |

| Lyra module | Mem0 fit |
|---|---|
| `lyra-core/memory/procedural` (SQLite FTS5) | **Partial.** Mem0's BM25 channel overlaps; could re-host on mem0 if user-scope memory is the goal. |
| `lyra-core/memory/distillers` | **Poor.** Lyra distillers produce typed reasoning items; mem0 flattens. |
| User-preference layer | **Strong fit.** Mem0's `user_id` scope is exactly what `lyra` needs for cross-session preference memory (the L37-6 auto-memory work in v3.7). |

**Net verdict**: mem0 is a strong *user-preference layer* — Lyra's L37-6 auto-memory could replace `~/.lyra/memory/<project>/memory.md` with mem0's `user_id`-scoped store and gain hybrid-retrieval at the cost of a non-trivial dependency. For trace-shaped or structured-claim memory neither Polaris nor Lyra should adopt mem0 — those need typed substrates ([docs/183](183-oss-memory-landscape-may-2026.md) compares; [docs/184](184-strongest-memory-techniques-synthesis-may-2026.md) recommends).

---

## §9 — Where this fits

- [docs/100 — Contextual Agentic Memory Is a Memo](100-contextual-memory-is-a-memo.md) — the structural critique of context-injection memory; mem0 is the most prominent C-engineering implementation that critique applies to.
- [docs/151-153 MEMTIER](151-memtier-why-flat-memory-breaks-at-72-hours.md) — the case for tiered memory; mem0 is single-tier.
- [docs/106-109 Memento](106-memento-paper-theory.md) — the M-MDP alternative to mem0's text-row model.
- [docs/79 Skill-RAG](79-skill-rag.md) — failure-state-aware retrieval; mem0's multi-signal is the sibling pattern.
- [docs/72 claude-mem](72-claude-mem-persistent-memory-compression.md) — the Anthropic-centric competitor; different scope.
- [docs/183 — OSS Memory Landscape](183-oss-memory-landscape-may-2026.md) — full comparison vs Zep / Letta / Cognee / Supermemory.
- [docs/184 — Strongest Memory Techniques Synthesis](184-strongest-memory-techniques-synthesis-may-2026.md) — the recommended stack folds mem0's multi-signal retrieval at the right tier.
- [docs/185 — Memory Integration Playbook](185-memory-integration-playbook.md) — how to actually retrofit mem0 onto Polaris / Lyra / Claude Code.
