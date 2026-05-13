# 184 — Strongest Memory Techniques: Cross-Paper + Cross-Repo Synthesis (May 2026)

**Scope.** A unified synthesis of the strongest agent-memory techniques across the canon ([docs/100](100-contextual-memory-is-a-memo.md), [docs/106-109](106-memento-paper-theory.md), [docs/151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md), [docs/72](72-claude-mem-persistent-memory-compression.md), [docs/79](79-skill-rag.md), [docs/81](81-reasoningbank.md), [docs/100](100-contextual-memory-is-a-memo.md)) and the post-canon May-2026 frontier ([docs/181-183](181-mem0-deep-dive.md)). Mirrors [docs/177](177-skills-discovery-curator-strongest-2026-techniques.md) (the strongest-skill-techniques synthesis).

**One-paragraph thesis.** Five structural commitments cut across the entire 2026 memory landscape: **(1) retrieval architecture is the binding ceiling, not memory substrate** ([MEMTIER](151-memtier-why-flat-memory-breaks-at-72-hours.md) thesis, validated by every comparative benchmark); **(2) tiered memory beats flat memory** at scale ≥ 72h, regardless of substrate (mem0 single-tier degrades; Letta + MemGPT + Cognee tier-aware win); **(3) PPR-over-entity-graph fusion is the strongest retrieval algorithm** for combined factual + multi-hop + sense-making ([HippoRAG 2](https://arxiv.org/abs/2502.14802) sweep); **(4) temporal validity is non-optional** for any memory whose facts change (Graphiti `valid_at`/`invalid_at`); **(5) memory ops belong in the reasoning trace** (Letta inversion of MemGPT's runtime-paging), not out of band. The strongest **2026 production stack** is opinionated: tiered (hot in-context / warm vector-retrieved / cold KG-traversed) substrate with HippoRAG-2-style PPR fusion, Graphiti-style temporal validity, mem0-style scope stratification at the warm tier, Cognee-style explicit `forget` + periodic `improve`, and memory-ops-as-tool-calls (Letta) so every read/write is auditable in the trace. No single OSS system ships all five today; the package below names the integration recipe for Polaris and Lyra.

---

## §1 — Five structural commitments

| Commitment | Evidence (canon + frontier) |
|---|---|
| **Retrieval architecture binds, not substrate** | MEMTIER ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)) shows BM25 ceiling regardless of model; Zep > mem0 same-stack ([n1n.ai](https://explore.n1n.ai/blog/ai-agent-memory-comparison-2026-mem0-zep-letta-cognee-2026-04-23)) confirms architecture differentiates more than substrate; HippoRAG 2 PPR fusion sweeps multiple benchmarks |
| **Tiered > flat at long horizon** | MEMTIER 14pp/72h degradation on flat; MemGPT/Letta tier-aware design wins; Cognee `improve()` consolidation; claude-mem's progressive-disclosure |
| **PPR over entity graph + dense embedding fusion is the strongest retrieval** | HippoRAG 2 SOTA across NaturalQA, 2WikiMultiHop, MuSiQue, NarrativeQA, HotpotQA, PopQA simultaneously |
| **Temporal validity is non-optional for changing facts** | Graphiti `valid_at`/`invalid_at`; Zep beats mem0 by 15pts on LongMemEval same-stack; Polaris RED-RETRACTED tier is the same idea |
| **Memory ops belong in the reasoning trace** | Letta tool-call writes; ReasoningBank's structured items; auditability + replayability concern |

These five together define the memory analogue of [docs/177](177-skills-discovery-curator-strongest-2026-techniques.md)'s skill-side commitments.

---

## §2 — The full design-space matrix

Six axes; every memory system in [docs/183](183-oss-memory-landscape-may-2026.md) plots somewhere in the cube.

### Axis 1 — Substrate

```
flat text rows ← typed text rows ← entity graph ← temporal KG ← parametric model adapter
(mem0)         (ReasoningBank,    (Cognee,        (Graphiti)    (RAG → fine-tune)
                Skill-RAG)         HippoRAG 2)
```

### Axis 2 — Retrieval algorithm

```
keyword (BM25) ← dense embedding ← hybrid (RRF) ← entity-link fusion ← PPR-graph fusion ← LLM rerank
(MEMTIER       (Voyager, mem0     (LlamaIndex)   (mem0 v2)             (HippoRAG 2)     (SkillRouter
 baseline)      semantic)                                                                pattern)
```

### Axis 3 — Tier structure

```
single tier ← two-tier (hot + cold) ← three-tier (hot + warm + cold) ← four-tier (Letta core/archival/recall + sub-tiers)
(mem0,        (claude-mem)            (MEMTIER prescription)             (Letta, MemGPT)
 Cognee)
```

### Axis 4 — Write semantics

```
single-pass ADD-only ← multi-pass ADD/UPDATE/DELETE ← temporal-append ← agent-tool-call writes
(mem0 v2)              (mem0 v1, MemGPT)               (Graphiti)         (Letta)
```

### Axis 5 — Forget / decay

```
none ← manual delete ← periodic `improve` ← temporal expiry ← TTL-based decay
(mem0)  (most)        (Cognee)             (Graphiti)         (cache layers)
```

### Axis 6 — Operational mode

```
batch only ← online streaming ← hybrid (online write, batch consolidation)
(legacy RAG)  (mem0, Zep)         (Cognee, Polaris heartbeat-driven)
```

---

## §3 — The strongest 2026 stack (recommended)

Opinionated stack, mirroring the [docs/177 §3](177-skills-discovery-curator-strongest-2026-techniques.md) shape.

### Tier 0 — Hot context (always in the LLM context window)

The current turn + the most recent K turns + an LLM-summarised prefix of older turns.

- Use Anthropic-style auto-compaction OR claude-mem's progressive-disclosure pattern.
- Budget: ≤ 30% of context window.

### Tier 1 — Warm vector + BM25 retrieval

Retrieved on demand based on query similarity. Per-turn retrieval; results enter Tier 0.

- **Substrate**: mem0-style scope-stratified rows (`user_id` / `session_id` / `agent_id`) **PLUS** typed rows (ReasoningBank-style structured items) when memory is trace-shaped or claim-shaped.
- **Retrieval**: hybrid semantic + BM25 + entity-link fusion (mem0 multi-signal pattern) **PLUS** RRF for production robustness (over mem0's score-additive default).
- **Embedder**: ≥ 600M-param (BGE-M3 or Voyage-3.5).
- **Forget**: periodic `improve()` cycle (Cognee-style) that runs in the heartbeat scheduler ([Polaris P28](../projects/polaris/POLARIS_V2_2_DEEP_RESEARCH_PLAN.md)).

### Tier 2 — Cold knowledge graph

Long-term structured store; queried via PPR fusion when Tier 1 retrieval is ambiguous or multi-hop.

- **Substrate**: temporal KG (Graphiti shape) with `valid_at` / `invalid_at` on every fact.
- **Retrieval**: HippoRAG 2's PPR + dense fusion (the strongest multi-hop algorithm published).
- **Storage**: Neo4j (production), Kuzudb (embedded), or pgvector + a graph table (lightweight).

### Cross-cutting: memory ops as tool calls

Every read/write to Tiers 1-2 is exposed to the agent as a tool call (Letta inversion). Reasoning trace records each op:

```python
agent.memory.recall(query="...", tier="warm", filters={"user_id": "..."})
agent.memory.remember(text="...", tier="warm", scope="user")
agent.memory.forget(memory_id="...", reason="superseded")
agent.memory.improve()    # heartbeat-driven
```

This makes memory auditable, replayable, and *the agent's responsibility* rather than runtime infrastructure.

### Cross-cutting: temporal validity

Every fact in any tier carries `valid_at` and `invalid_at`. Stale facts are *never* deleted from the audit log; they're just out-of-window for current queries. This composes with Polaris's RED-RETRACTED tier and with Lyra's procedural-memory drift detection.

### Cross-cutting: cross-system A/B via MemoryBench

Run [Supermemory's MemoryBench](https://github.com/supermemoryai/supermemory) on a sample of your traces to validate the stack. Treat published vendor benchmarks as upper bounds, not portable rankings.

---

## §4 — Per-memory-shape recipe

Different memory shapes need different stack compositions. The decision matrix:

| Shape | Tier 0 | Tier 1 | Tier 2 | Notes |
|---|---|---|---|---|
| **Chat / personal assistant** | last 10 turns + summary | mem0 user/session/agent rows | optional Graphiti for changing facts | mem0 default + Graphiti for temporal |
| **Customer support** | last 5 turns + ticket context | mem0 session-scoped + agent expertise | KB graph for product knowledge | mem0 + HippoRAG 2 KB layer |
| **Coding agent** (Lyra) | current file + recent edits | procedural skills (FTS5) + user prefs (mem0) | project graph (Cognee) | hybrid mem0 + Cognee + Lyra existing |
| **Research agent** (Polaris) | current claim + reasoning trace | typed claims + reasoning bank | Program Graph (PPR-fused) | Polaris existing + HippoRAG 2 + Graphiti |
| **Multi-agent** | per-agent core + shared blackboard | per-agent rows + shared knowledge | shared Program Graph | Letta-style tool-call writes |
| **Long-running daemon** | current cycle | telemetry / drift detector | full archive | heartbeat-driven Cognee `improve()` |

---

## §5 — Polaris adoption recipe

Polaris already has the substrate; the gap is **retrieval algorithm + temporal validity + memory-as-tool-call**.

```text
packages/polaris-core/src/polaris_core/memory/
  ppr_fusion.py             # NEW [HippoRAG 2] — PPR over Program Graph + dense
                            # embedding fusion. Replaces flat retrieval.
  temporal_validity.py      # NEW [Graphiti] — valid_at/invalid_at on every
                            # claim row. Composes with RED-RETRACTED tier.
  memory_tools.py           # NEW [Letta] — agent-tool-call wrappers for
                            # claim_emit, claim_search, claim_forget, etc.
                            # Records ops in HIR for auditability.
```

Existing primitives Polaris reuses (no replacement):

- `polaris-core/memory/program_graph` — the Tier-2 substrate.
- `polaris-core/memory/reasoning_bank` — Tier-1 trace-shaped layer.
- `polaris-core/memory/provenance` — the audit log.
- `polaris-core/memory/wiki` — the human-readable Tier-0 surface.
- `polaris-skills/auto_creator` — the Tier-1 → Tier-2 distillation pipeline.
- v2.2 P28 heartbeat scheduler — the engine that runs Cognee-style `improve()`.

Estimated effort: ~5 weeks for the three NEW files + integration tests + benchmark validation.

### v2.5 phases (proposed)

- **P42** PPR-fusion retrieval. ~2 weeks.
- **P43** Temporal validity on claim rows. ~1.5 weeks.
- **P44** Memory tool-call surface. ~1.5 weeks.

---

## §6 — Lyra adoption recipe

Lyra's gap is **user-pref layer + project-knowledge graph**.

```text
packages/lyra-core/src/lyra_core/memory/
  mem0_adapter.py           # NEW [mem0] — user/session scope wrapper.
                            # Extends v3.7 L37-6 auto_memory.
  cognee_adapter.py         # NEW [Cognee] — project-knowledge graph;
                            # local-first via Kuzudb embedded.
  ppr_fusion.py             # NEW [HippoRAG 2] — PPR fusion over the
                            # project graph + procedural memory FTS5.
  memory_tools.py           # NEW [Letta] — tool-call surface.
```

Existing primitives Lyra reuses:

- `lyra-core/memory/procedural` (SQLite FTS5) — Tier-1 keyword.
- `lyra-skills/extractor` — auto-extraction pipeline.
- `lyra-core/arena/elo` — A/B harness substrate (composes with MemoryBench).

Estimated effort: ~6 weeks across the four NEW adapters + tests + MemoryBench wiring.

### v3.8 phases (proposed)

- **L38-1** mem0 user-pref adapter. ~1.5 weeks.
- **L38-2** Cognee project-knowledge adapter. ~1.5 weeks.
- **L38-3** PPR fusion over procedural + project layers. ~2 weeks.
- **L38-4** Letta-style memory tool-call surface. ~1 week.

---

## §7 — What NOT to do

Three patterns the canon explicitly argues against:

### Don't ship flat single-tier memory at production scale

MEMTIER's 14pp/72h degradation is a hard ceiling. Tiered architecture is non-optional past 72h horizon.

### Don't trust vendor benchmarks as portable rankings

mem0 / Zep / Supermemory each report being best on different benchmarks; the eval-setup determines the winner. Run MemoryBench on your own traces.

### Don't conflate context window size with memory

A 1M-context model is *not* a memory replacement — context is hot-tier only. Tier 1 + Tier 2 still need real retrieval. (The opposite mistake — running cold-archive retrieval on every turn — is the LangChain `ConversationBufferMemory` anti-pattern.)

---

## §8 — One-paragraph summary

Memory in 2026 converges on a **three-tier stack**: hot context (last K turns + LLM summary, ≤30% of context window), warm scope-stratified retrieval (mem0-style rows + RRF-fused semantic+BM25+entity-link, ≥600M-param embedder), cold temporal knowledge graph (Graphiti shape with `valid_at`/`invalid_at`, retrieved via HippoRAG-2-style PPR fusion). Cross-cutting: every memory op is an agent tool call (Letta inversion); periodic `improve()` runs in the heartbeat scheduler (Cognee pattern); Supermemory's MemoryBench validates the stack on your own traces. **No single OSS system ships all five layers**; the integration is the recipe. Polaris's adoption: PPR fusion + temporal validity + memory-as-tool-call over the existing Program Graph (3 NEW files, ~5 weeks). Lyra's adoption: mem0 user-pref + Cognee project graph + PPR fusion + Letta tool-call (4 NEW adapters, ~6 weeks). The single biggest unrealised lever in either harness today is the *retrieval algorithm* — flat semantic + BM25 leaves 31-44pp on the table per [SkillRouter](https://arxiv.org/abs/2603.22455) (the [docs/179](179-skill-retrieval-routing-and-activation.md) finding generalises to memory).

---

## §9 — Reading list

In recommended order:

### Critical priors

1. [docs/100 — Contextual Agentic Memory Is a Memo](100-contextual-memory-is-a-memo.md) — structural critique that frames everything below.
2. [docs/151-153 MEMTIER](151-memtier-why-flat-memory-breaks-at-72-hours.md) — retrieval architecture is the binding ceiling.
3. [docs/106-109 Memento](106-memento-paper-theory.md) — M-MDP + case-based reasoning.

### Frontier (post-canon)

4. **HippoRAG 2** — *From RAG to Memory: Non-Parametric Continual Learning* — [arXiv:2502.14802](https://arxiv.org/abs/2502.14802) (ICML 2025).
5. **Graphiti / Zep** — [github.com/getzep/zep](https://github.com/getzep/zep) — temporal knowledge graph.
6. **Letta** — [github.com/letta-ai/letta](https://github.com/letta-ai/letta) — memory-as-tool-call inversion.
7. **mem0** — [github.com/mem0ai/mem0](https://github.com/mem0ai/mem0) + [docs/181](181-mem0-deep-dive.md) deep dive.
8. **Cognee** — [github.com/topoteretes/cognee](https://github.com/topoteretes/cognee) — cognitive-science GraphRAG.
9. **Supermemory + MemoryBench** — [github.com/supermemoryai/supermemory](https://github.com/supermemoryai/supermemory).

### Synthesis chapters

10. [docs/157 — May-2026 Memory + Skills Synthesis](157-may-2026-synthesis-memory-and-skills.md).
11. [docs/182 — Memory Frontiers 2026](182-memory-frontiers-2026.md) — paper-side companion.
12. [docs/183 — OSS Memory Landscape](183-oss-memory-landscape-may-2026.md) — repo-side companion.

### Adjacent

13. [docs/72 claude-mem](72-claude-mem-persistent-memory-compression.md), [docs/79 Skill-RAG](79-skill-rag.md), [docs/81 ReasoningBank](81-reasoningbank.md).
14. [docs/128-129 KG / KG-RAG hybrid](128-knowledge-graphs-as-substrate.md).
15. [docs/130-132 Memory infra](130-distributed-sql-as-agent-memory.md).
16. [docs/177 — Strongest Skill Techniques](177-skills-discovery-curator-strongest-2026-techniques.md) — sister synthesis on the skill side.

---

## §10 — Where this fits

- Predecessor synthesis: [docs/157](157-may-2026-synthesis-memory-and-skills.md) (memory + skills together).
- Sister synthesis on the skill side: [docs/177](177-skills-discovery-curator-strongest-2026-techniques.md).
- Implementation guide: [docs/185 — Memory Integration Playbook](185-memory-integration-playbook.md).
- **Adversarial-input extension**: [docs/186 — Mnema](186-mnema-witness-lattice.md), [docs/187 — Multi-Agent Shared Memory Landscape](187-multi-agent-shared-memory-landscape.md), [docs/188 — Witness/Provenance Memory Synthesis](188-witness-provenance-memory-techniques-synthesis.md). The five commitments above assume cooperative authorship; doc 188 names the sixth (provenance is non-optional once shared memory or external content enters the write path) and the closed-form 1−α detection floor that bounds redundancy.
