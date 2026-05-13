# 185 — Memory Integration Playbook (May 2026)

**Scope.** Concrete, executable adoption guide for the memory stack recommended in [docs/184](184-strongest-memory-techniques-synthesis-may-2026.md). Three integration paths covered: **Polaris** (research-shaped memory; existing Program Graph), **Lyra** (coding-shaped + user-preference; existing procedural memory), **Claude Code-direct user** (no harness; just personal Claude Code installation). Each path enumerates exact file paths, sequencing, prerequisites, cost / latency / privacy trade-offs, and migration steps from the current state.

**One-paragraph thesis.** Adopting the [docs/184](184-strongest-memory-techniques-synthesis-may-2026.md) three-tier stack (hot context + warm scope-stratified retrieval + cold temporal-KG with PPR fusion) is **incremental**, not a forklift. Polaris's existing substrate (Program Graph + Provenance Ledger + ReasoningBank + Wiki) is already aligned with the recommended Tier-2 substrate; the gaps are *PPR fusion* + *temporal validity* + *memory-as-tool-call* (3 new files, ~5 weeks per [docs/184 §5](184-strongest-memory-techniques-synthesis-may-2026.md)). Lyra's gaps are *user-preference layer (mem0)* + *project-knowledge graph (Cognee)* + *PPR fusion over both* + *Letta-style tool-call surface* (4 new adapters, ~6 weeks). For Claude Code-direct users, the cleanest path is **claude-mem (already installed) + mem0 Claude Code skill + Supermemory MCP server** — three drop-in pieces that together approximate the recommended three-tier stack without code changes. Trade-offs are explicit per path: Polaris's PPR fusion is bounded by the embedding-model cost per claim retrieval; Lyra's mem0 dependency adds a non-trivial vendor lock-in; the Claude Code-direct path is the most ergonomic but the least typed (no provenance, no trust tiers).

---

## §1 — Path 1: Polaris (research-shaped memory)

Polaris's existing Program Graph + Provenance Ledger + ReasoningBank + Wiki align with the recommended Tier-2 substrate. The integration is **additive**, not a migration.

### §1.1 — Prerequisites

- Polaris v2.2 (heartbeat scheduler shipped in [P28](../projects/polaris/POLARIS_V2_2_DEEP_RESEARCH_PLAN.md)).
- Polaris v2.3 (mind-map graph shipped in [P34](../projects/polaris/POLARIS_V2_3_DEEP_RESEARCH_PLAN.md)).
- Embedder dependency: BGE-M3 (open-weights default) or Voyage-3.5 (paid).

### §1.2 — Three new modules (estimated ~5 weeks)

```text
packages/polaris-core/src/polaris_core/memory/
  ppr_fusion.py             # NEW — Personalized PageRank over Program Graph
                            # entities + dense embedding fusion. Replaces
                            # flat semantic retrieval in research-lit.
                            #
                            # Class: PPRRetriever
                            #   - build_entity_index(graph)
                            #   - retrieve(query, top_k) -> list[ClaimRow]
                            #   - fuse(ppr_scores, dense_scores) -> ranked
                            #
                            # Algorithm: HippoRAG 2 (arXiv:2502.14802):
                            #   1. Extract entities from query
                            #   2. Seed PPR walk on Program Graph from those entities
                            #   3. Score each candidate claim row by:
                            #      α * cosine(query, claim_embedding)
                            #      + (1-α) * PPR_weight(claim's entities)
                            #   4. Return top-k

  temporal_validity.py      # NEW — valid_at / invalid_at on every claim
                            # row in the Provenance Ledger. Composes with
                            # the existing RED-RETRACTED tier.
                            #
                            # Schema additions:
                            #   class Claim:
                            #     valid_at: float           # unix ts
                            #     invalid_at: float | None  # None = still valid
                            #
                            # Retrieval filter:
                            #   active_at(query_ts) -> claims valid at that ts.
                            #
                            # Composes with RED-RETRACTED via auto-set
                            # invalid_at = retraction_ts.

  memory_tools.py           # NEW — agent-tool-call wrappers (Letta pattern).
                            # Each op is recorded in the HIR trace.
                            #
                            # Tools exposed:
                            #   memory.claim_emit(text, evidence)
                            #   memory.claim_search(query, top_k)
                            #   memory.claim_retract(claim_id, reason)
                            #   memory.improve()  # heartbeat-driven
```

### §1.3 — Sequencing

| Phase | Work | Effort |
|---|---|---|
| **P42** | PPR-fusion retrieval | ~2 weeks |
| **P43** | Temporal validity on claim rows | ~1.5 weeks |
| **P44** | Memory tool-call surface | ~1.5 weeks |

Sequencing rationale:

- **P42 first**: PPR fusion's value is independent of temporal validity. Ship the retrieval upgrade as the first deliverable.
- **P43 second**: depends on P42's claim_search to honour `active_at(query_ts)` filters.
- **P44 last**: tools surface composes both prior phases.

### §1.4 — Trade-offs

| Trade-off | Cost | Mitigation |
|---|---|---|
| **PPR walk cost** | Each retrieval runs PPR over the entity graph; O(graph-size) per query | Cache PPR vectors by entity; recompute on graph mutation only |
| **Embedding regeneration** | All claims need re-embedding when model upgrades | [docs/184 §3 Tier 1](184-strongest-memory-techniques-synthesis-may-2026.md#tier-1) — version-pin embedding model per claim |
| **Temporal-filter overhead** | Every retrieval applies the `active_at` window | SQL index on `(valid_at, invalid_at)`; sub-ms cost at any reasonable scale |
| **HIR trace size growth** | Memory tool calls multiply HIR rows | Polaris already gates on cost-envelope; the heartbeat scheduler governs |

### §1.5 — Migration steps from current state

1. **Verify** v2.2 P28 (heartbeat) and v2.3 P34 (mind-map graph) are landed.
2. **Land P42**: ship `ppr_fusion.py` + integration tests on a planted research bench. Validate PPR > flat retrieval on a 100-claim corpus.
3. **Backfill embeddings** for the existing Program Graph (one-time batch job).
4. **Land P43**: extend the Claim schema with `valid_at` / `invalid_at`. Default `valid_at = imported_ts`, `invalid_at = None`. Backwards compatible.
5. **Land P44**: register `memory.*` tools with the agent loop. Existing claim emission becomes a tool call.
6. **Validate** end-to-end on a planted multi-claim research scenario; verify retrieval quality + temporal filter + tool-call audit trail.

### §1.6 — Success criteria

- ✅ PPR-fusion retrieval Hit@1 ≥ 80% on a planted 100-claim research bench (baseline flat-retrieval ~50%).
- ✅ Temporal-filter correctness: 100% of pre-retraction claims excluded after retraction.
- ✅ Memory tool-call HIR rows visible in the trace.
- ✅ Zero regressions on Polaris's 562-test suite.

---

## §2 — Path 2: Lyra (coding-shaped + user-preference memory)

Lyra needs the user-preference layer (already started in v3.7 L37-6) plus a project-knowledge graph plus PPR fusion plus tool-call writes. **Larger surface than Polaris** because Lyra didn't start with a Program Graph.

### §2.1 — Prerequisites

- Lyra v3.7 (L37-6 auto_memory shipped).
- Decision: bring in `mem0` as a dependency? (yes — recommended).
- Decision: bring in `cognee` as a dependency? (yes for project-knowledge layer).
- Embedder: same options as Polaris.

### §2.2 — Four new adapters (estimated ~6 weeks)

```text
packages/lyra-core/src/lyra_core/memory/
  mem0_adapter.py           # NEW — user/session/agent scope memory layer
                            # via mem0. Extends v3.7 L37-6 auto_memory.
                            #
                            # Replaces direct ~/.lyra/memory/<project>/
                            # memory.md writes with mem0.add() calls
                            # scoped by user_id = current OS user.
                            #
                            # API:
                            #   write(text, scope="user")
                            #   read(query, top_k=3)

  cognee_adapter.py         # NEW — local-first project-knowledge graph
                            # via Cognee. Embedded Kuzudb backend.
                            #
                            # API:
                            #   ingest(file_or_text)  # cognee.remember
                            #   query(prompt)         # cognee.recall
                            #   forget(node_id)
                            #   improve()             # heartbeat-driven

  ppr_fusion.py             # NEW — PPR fusion over Cognee graph +
                            # procedural-memory FTS5 + mem0 user-scope
                            # rows. Three-source merge.
                            #
                            # Algorithm: HippoRAG 2 over the union;
                            # source diversity bonus for cross-source
                            # convergence.

  memory_tools.py           # NEW — Letta-style tool surface registered
                            # with the Lyra agent loop. Tools:
                            #   memory.recall(query, scope?)
                            #   memory.remember(text, scope)
                            #   memory.forget(memory_id)
                            #   memory.improve()
```

### §2.3 — Sequencing

| Phase | Work | Effort |
|---|---|---|
| **L38-1** | mem0 user-pref adapter | ~1.5 weeks |
| **L38-2** | Cognee project-knowledge adapter | ~1.5 weeks |
| **L38-3** | PPR fusion over procedural + project layers | ~2 weeks |
| **L38-4** | Letta-style memory tool-call surface | ~1 week |

Sequencing rationale: L38-1 and L38-2 are independent and can ship in parallel. L38-3 depends on both. L38-4 depends on all prior.

### §2.4 — Trade-offs

| Trade-off | Cost | Mitigation |
|---|---|---|
| **mem0 dependency** | Adds an external Python package + LLM-extractor cost per write | Use the cheapest configured Lyra LLM (mem0's `gpt-5-mini` swap-out) |
| **Cognee Neo4j dependency** | Local Neo4j install adds ops burden | Use embedded Kuzudb instead (no daemon) |
| **Three-source PPR fusion** | Score-distribution drift across sources | RRF-style fusion (rank-based, distribution-invariant) |
| **Vendor lock-in** | mem0 + Cognee both API-stable but external | Adapter pattern keeps the dependency at one file each; swap-in is feasible if either pivots |
| **L37-6 auto_memory migration** | Existing `~/.lyra/memory/<project>/memory.md` data | Backwards-compat: read both stores; new writes go to mem0; legacy writes deprecate over 1 release cycle |

### §2.5 — Migration steps from current state

1. **Land L38-1**: add `mem0_adapter.py`. Backwards-compat with L37-6: read old `memory.md` files, route new writes through mem0.
2. **Land L38-2**: add `cognee_adapter.py`. Index the project's existing CLAUDE.md + README + key markdown into Cognee.
3. **Land L38-3**: PPR fusion over the three sources (procedural FTS5 + mem0 user-scope + Cognee project-graph). Validate on a planted multi-source bench.
4. **Land L38-4**: register memory tools with the agent loop. Migrate `lyra_skills/extractor.py` to call `memory.remember` (was direct).
5. **Validate** with the existing Lyra 562-test suite + new memory integration tests.

### §2.6 — Success criteria

- ✅ User-preference recall Hit@1 ≥ 90% on planted preference bench (baseline `memory.md` ~80%).
- ✅ Project-knowledge multi-hop recall (Cognee) Hit@k=3 ≥ 80% on planted multi-doc bench.
- ✅ PPR fusion outperforms any single source by ≥ 5pp.
- ✅ Memory tool calls visible in agent trace.
- ✅ Zero regressions on Lyra's 562-test suite.

---

## §3 — Path 3: Claude Code-direct user (no harness)

The fastest adoption path — three drop-in pieces approximate the three-tier stack without writing any code.

### §3.1 — The three pieces

| Piece | Tier | Install | What it adds |
|---|---|---|---|
| **claude-mem** ([docs/72](72-claude-mem-persistent-memory-compression.md)) | Tier 0 | `claude plugin install pixegami/claude-mem` | Session-capture → semantic-summary → progressive-disclosure retrieval. The Tier-0 hot-context optimiser. |
| **mem0 Claude Code skill** ([anthropics/skills](https://github.com/anthropics/skills)) | Tier 1 | `claude skill install mem0/cli` | mem0's user/session/agent scope layer accessible via the `@mem0` skill. Tier-1 warm-retrieval. |
| **Supermemory MCP server** | Cross-IDE Tier 1 | `claude mcp add supermemory` | Cross-platform memory sync (Claude.ai, Cursor, ChatGPT, Perplexity). Tier-1 on its own; complements mem0. |

### §3.2 — What this gets you

- **Hot tier**: claude-mem + Anthropic's auto-compaction.
- **Warm tier**: mem0 (per-project / per-user) OR Supermemory (cross-IDE).
- **Cold tier**: NOT covered. The Claude Code-direct path doesn't ship a knowledge graph. If you need Tier-2, install Cognee's Claude Code plugin separately.

### §3.3 — What this does NOT get you

- **No PPR fusion** — Claude Code's plugin system doesn't expose retrieval-algorithm swapping at this layer.
- **No temporal validity** — claude-mem and mem0 are both flat-row.
- **No memory-as-tool-call audit** — plugins handle reads/writes opaquely.
- **No provenance / trust tiers** — for that you need a typed substrate (Polaris).

### §3.4 — When this path is right

Use the Claude Code-direct path when:

- You're a single user running Claude Code.
- You don't need typed claim provenance.
- You don't need temporal-validity-aware facts.
- Cost matters and you want zero infra.

Don't use this path when:

- You need audit-grade memory (use Polaris).
- You're shipping coding-agent infrastructure to others (use Lyra).
- You need multi-hop graph reasoning (add Cognee plugin separately).

### §3.5 — Optional fourth piece — Cognee Claude Code plugin

For users who want a knowledge graph layer:

```bash
claude plugin install topoteretes/cognee
```

Adds the four-op API (`remember` / `recall` / `forget` / `improve`) with a local Kuzudb backend. Adds Tier-2 cold-storage to the stack.

---

## §4 — Cross-cutting: MemoryBench validation

Whichever path you adopt, validate via [Supermemory's MemoryBench](https://github.com/supermemoryai/supermemory):

```bash
git clone https://github.com/supermemoryai/supermemory
cd supermemory/memorybench
# Configure the systems-under-test in config.yaml — point at:
# - your local mem0 instance
# - your local Cognee instance
# - your Polaris/Lyra adapter
python -m memorybench --systems mem0 cognee polaris-adapter
```

Output: head-to-head ranking on LongMemEval, LoCoMo, ConvoMem.

For Polaris and Lyra: integrate MemoryBench into CI. Run the bench nightly on a frozen test catalog. Regression on any of the three benchmarks should block deploys.

---

## §5 — Cost / latency / privacy comparison

The three paths land at different points on the operational triangle:

| Axis | Path 1 (Polaris) | Path 2 (Lyra) | Path 3 (Claude Code direct) |
|---|---|---|---|
| **Cost** | Embedder cost per claim retrieval; one-time backfill. ~$0.001 / retrieval. | mem0 LLM-extractor + embedder + Cognee local. ~$0.005 / write. | Plugin/Skill subscriptions; Supermemory SaaS tier. ~$10–20/mo flat. |
| **Latency** | PPR + temporal-filter < 100ms. Memory tool-call adds 1 round-trip. | mem0 ~100ms; Cognee local sub-50ms; PPR fusion ~200ms total. | claude-mem sub-100ms; mem0 skill ~150ms; MCP server ~200ms. |
| **Privacy** | Fully local; no vendor data leaves the harness. | mem0 self-hostable; Cognee local-first. Possible egress via embedder API calls. | Supermemory hosted-by-default; mem0 + Cognee self-hostable. |
| **Debug / audit** | Full HIR trace; every memory op recorded. | Tool-call trace via L38-4. | Opaque (plugin layer). |
| **Effort** | ~5 weeks engineering | ~6 weeks engineering | <1 hour install |

---

## §6 — Decision flowchart

```
What is your starting point?

├── I'm building Polaris (research-agent harness)
│   └── Path 1: PPR fusion + temporal validity + memory tools
│       (3 new files, ~5 weeks, see §1)
│
├── I'm building Lyra (coding-agent harness)
│   └── Path 2: mem0 + Cognee + PPR fusion + memory tools
│       (4 new adapters, ~6 weeks, see §2)
│
├── I'm a Claude Code user (single-user, no harness)
│   ├── Need typed claim provenance? → switch to Polaris
│   ├── Need cross-IDE sync? → Path 3 with Supermemory MCP
│   ├── Need knowledge graph? → Path 3 + Cognee plugin
│   └── Default: Path 3 (claude-mem + mem0 skill)
│
└── I'm building something else
    ├── Chat-shaped → mem0 + Graphiti (Zep)
    ├── Multi-hop QA → HippoRAG 2 + Cognee
    ├── Long-running daemon → Letta + Cognee improve()
    └── Multi-modal → Supermemory
```

---

## §7 — Migration anti-patterns

Three patterns to avoid when migrating:

### Anti-pattern 1 — Forklift migration

Don't replace Polaris's Program Graph with mem0. Don't replace Lyra's `memory.md` with Cognee. The recommended stack *layers* on top of existing primitives; it doesn't replace them.

### Anti-pattern 2 — Premature retrieval-algorithm swap

Don't ship PPR fusion before validating that flat retrieval is the bottleneck. MEMTIER's binding-ceiling argument is empirical — confirm it on your traces first via MemoryBench.

### Anti-pattern 3 — Skipping temporal validity for "current-state-only" memory

Even coding-agent memory has temporal facts: deprecated APIs, retired libraries, removed CLI flags. Skipping `valid_at` / `invalid_at` saves engineering time today and creates a debugging mess in 18 months.

---

## §8 — One-paragraph summary

The recommended three-tier memory stack adopts incrementally. Polaris's existing substrate is already aligned; the three gaps (PPR fusion, temporal validity, memory-as-tool-call) ship in three new files over ~5 weeks under v2.5 phases P42-P44. Lyra needs four new adapters (mem0 user-pref, Cognee project-knowledge, PPR fusion over both, Letta-style tool surface) over ~6 weeks under v3.8 phases L38-1 through L38-4. Claude Code-direct users get the fastest path: claude-mem + mem0 skill + Supermemory MCP server installs in under an hour, approximating Tier 0+1; add Cognee plugin for Tier 2. Validate every path with [Supermemory's MemoryBench](https://github.com/supermemoryai/supermemory) on your own traces — vendor benchmarks are not portable. Single biggest risk: premature retrieval-algorithm swap before confirming the binding ceiling on your traces. Single biggest opportunity: PPR fusion over the existing Program Graph (Polaris) is the highest-leverage memory upgrade either harness can ship.

---

## §9 — Where this fits

- [docs/181 — Mem0 Deep Dive](181-mem0-deep-dive.md), [docs/182 — Memory Frontiers 2026](182-memory-frontiers-2026.md), [docs/183 — OSS Memory Landscape](183-oss-memory-landscape-may-2026.md), [docs/184 — Strongest Memory Techniques Synthesis](184-strongest-memory-techniques-synthesis-may-2026.md) — the package this implements.
- [docs/151-153 MEMTIER](151-memtier-why-flat-memory-breaks-at-72-hours.md), [docs/106-109 Memento](106-memento-paper-theory.md), [docs/100](100-contextual-memory-is-a-memo.md) — the canon priors.
- [docs/72 claude-mem](72-claude-mem-persistent-memory-compression.md) — the Tier-0 reference for Path 3.
- [POLARIS_V2_2_DEEP_RESEARCH_PLAN.md](../projects/polaris/POLARIS_V2_2_DEEP_RESEARCH_PLAN.md), [POLARIS_V2_3_DEEP_RESEARCH_PLAN.md](../projects/polaris/POLARIS_V2_3_DEEP_RESEARCH_PLAN.md), [LYRA_V3_7_CLAUDE_CODE_PARITY_PLAN.md](../projects/lyra/LYRA_V3_7_CLAUDE_CODE_PARITY_PLAN.md) — the existing roadmaps this plan extends.
- Sister synthesis on the skill side: [docs/177 — Strongest Skill Techniques](177-skills-discovery-curator-strongest-2026-techniques.md), [docs/180 — Argus Skill-Router Design](180-argus-skill-router-agent-design.md).
