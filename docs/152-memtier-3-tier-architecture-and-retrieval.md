# 152 — MEMTIER (II): The 3-Tier Architecture, 5-Signal Retrieval, and Multi-Agent Isolation

**Paper.** Sidik & Rokach, *MEMTIER: Tiered Memory Architecture and Retrieval Bottleneck Analysis for Long-Running Autonomous AI Agents* — arXiv:2605.03675v1 [cs.AI] — 5 May 2026 — implemented as an OpenClaw plugin.

**Part 2 of 3.** Part 1 ([151](151-memtier-why-flat-memory-breaks-at-72-hours.md)) framed the problem: flat memory breaks at 72 h, four stacked failure modes, why OpenClaw is the right runtime to measure on. Part 2 (this chapter) walks through the architecture: tiers, schema, retrieval engine, scoring function, two-stage scoping, consolidation daemon, multi-agent isolation. Part 3 ([153](153-memtier-llm-distillation-and-results.md)) covers LLM distillation, LongMemEval-S results, and the three-layer invariance finding.

**One-line definition.** MEMTIER's architecture is a three-tier memory store (episodic JSONL, semantic distilled facts, procedural skills) wired to the OpenClaw runtime through two lifecycle hooks (`before_prompt_build`, `agent_end`), retrieved via a five-signal linear scoring function (BM25 + time decay + cognitive weight + tier boost; semantic similarity reserved for Phase 3), staged in a two-pass scoping pipeline (semantic facts first to identify relevant sessions, then episodic entries scoped to those sessions), with episodic logs **agent-private** by default and the distilled semantic tier **project-shared**, so multi-agent orchestrations can transfer knowledge without context contamination.

## Why architecture, not algorithm, is the contribution

The temptation reading a paper like MEMTIER is to treat the headline F1 numbers as the contribution and the architecture as plumbing. Read in that order, the paper looks unremarkable: "use BM25 with extra weights, get 0.382 F1, beats baselines." Read the other way — *architecture is the thesis; F1 numbers are the proof* — the paper is far more important. The three-layer invariance result (part 3) is only legible *because* the architecture cleanly disentangles the model, the retrieval policy, and the storage from each other. Without the three-tier separation, you cannot run the ablation that proves the BM25 ceiling.

The architecture is the science. Everything else is corroboration.

## The three tiers

MEMTIER is named for its three-tier structure. The tiers are not arbitrary; they correspond to three structurally distinct kinds of agent knowledge:

1. **Episodic** — *what happened in this session*. Per-session, agent-private, JSONL-backed, append-only, time-stamped, project-scoped. This is the stream of observations: tool calls, tool results, decisions, failures, user messages. It is high-volume and short-lived (most entries are stale within a few weeks; default time-decay half-life is 14 days).
2. **Semantic** — *what is true about the project's world that we have learned*. Distilled, project-shared, time-invariant, tier-boosted. This is the stable knowledge: the API endpoints we use, the customer's preferences, the rules of the system, the decisions we are committed to. It is low-volume and long-lived. The consolidation daemon (§3.3) populates it asynchronously by distilling clusters of episodic entries.
3. **Procedural** — *how we do things*. Reserved tier for skills/procedures. The architecture defines a tier multiplier µ = 1.4 for procedural entries (vs 1.2 semantic, 1.0 episodic), but the paper does not heavily exercise this tier in evaluation. It is the slot for things like the Voyager-style skill libraries ([19](19-voyager-skill-libraries.md), [89](89-voyager-deep.md)) or the HeavySkill-style readable skills ([156](156-heavyskill-parallel-reasoning-deliberation.md)).

The structural reason for this split is exactly Mode 3 of part 1 — flat text cannot distinguish *what happened* from *what is true*. By separating them at the storage level, the retrieval engine can score them differently, prioritise them differently, and consolidate one into the other on a policy schedule.

The three-tier structure is reminiscent of MemGPT's main-memory + external store, but it is fundamentally different in that **MemGPT's tiers are recency-based** (hot = main memory, cold = external) **whereas MEMTIER's tiers are knowledge-type-based** (episodic = events, semantic = facts, procedural = how-tos). This is the more important axis for long-horizon agents: a 72-hour-old fact that is still *true* should not be evicted just because it is old.

## Phase 1a: the episodic JSONL store

The episodic tier is the backbone. Every agent session writes structured entries to:

```
~/.openclaw/workspace/memory/episodic/YYYY-MM-DD.jsonl
```

One file per UTC day, append-only. Each line is a JSON object with the schema (paraphrased from the paper):

```text
{
  "id":               "<unique entry id>",
  "timestamp":        "<ISO-8601>",
  "session_id":       "<the agent session this came from>",
  "project":          "<project this entry belongs to>",
  "content":          "<the actual text, e.g., observation, tool result>",
  "tokens":           <int, the token count>,
  "promoted":         false,                                // whether consolidation has copied this into semantic
  "cognitive_weight": 0.0                                   // ∈ [-1, 1], updated by the attribution loop
}
```

Four design choices are worth pulling out:

1. **Per-day file split**: makes archival, rotation, and bulk operations easy. The JSONL format is line-grep-friendly, line-streamable, and trivially appendable from concurrent processes (each line is a complete JSON object). Compare with a single monolithic file (which is what MEMORY.md was) — daily files give you natural sharding without a database.
2. **Append-only with no in-place mutation**: critical for auditability. If you want to debug "why did the agent do X at hour 47?", you can replay the exact memory state by reading the JSONL files up to that timestamp. There is no risk of mutation by a concurrent agent corrupting an entry mid-write.
3. **Project scope**: prevents cross-project contamination. If you run two agents on two different projects on the same machine, they do not see each other's episodic entries (this is in addition to the multi-agent isolation described below).
4. **System entries are kept but not retrieved**: entries prefixed `[system]` are written for auditing but excluded from retrieval. So the JSONL contains both retrieval-eligible content and system observability events, distinguishable by prefix.

### The cognitive weight scalar

The single-most-important field in the schema is `cognitive_weight ∈ [−1, 1]`, initialised at 0. It is the per-entry scalar that accumulates evidence of retrieval quality over time. Positive values mean *the entry has contributed to successful tool executions*. Negative values mean *the entry has been associated with failures*. Zero means *not yet evaluated, or the evidence is balanced*.

Two important properties:

- It is the long-term memory of *which memories have proven useful*, learned online without human labelling. This is the thing the paper calls the "attribution loop."
- It feeds into the retrieval scoring function (§3.2) directly. Entries with high CW preferentially surface; entries with negative CW are demoted.

The update mechanism (§3.2 on attribution) supports two paths:

- **SGLang logprob attribution**: the canonical path. After a tool call, attribute the prompt tokens to memory entries by their logprob contribution to the call, and credit/debit each entry's CW by an outcome-weighted amount. This requires an SGLang-compatible runtime; in the paper's evaluation, this path was code-complete but blocked by 6 GB GPU constraints.
- **Lexical Jaccard fallback**: the production path. Compute the Jaccard similarity between the call text and each memory entry's content; entries with high overlap are credited proportionally. This is a coarse proxy but it is *cheap and always available*. It is what the paper's reported numbers use.

## Phase 1b: the five-signal weighted retrieval engine

The scoring function is the centrepiece. For a query q and a memory entry mᵢ, the score is:

```
S(q, mᵢ) = w⊤ ϕ(q, mᵢ)
```

where the feature vector is

```
ϕ = [ϕ_sem, ϕ_bm25, ϕ_decay, ϕ_cw, ϕ_tier]
```

and the default weights are

```
w₀ = [0, 0.35, 0.25, 0.25, 0.15]
```

Notice: the **semantic-similarity weight is 0** by default. The paper explicitly reserves this signal for Phase 3 — the dense retrieval transition. In the evaluation, MEMTIER scores using *only* BM25, time decay, cognitive weight, and tier boost. This is the architectural setup that lets the paper diagnose BM25 as the binding ceiling: if dense retrieval were active in the headline numbers, the diagnosis would be confounded.

Each signal's role:

### ϕ_bm25 — Okapi BM25 (k1=1.5, b=0.75), normalised to [0, 1], weight 0.35

The lexical relevance score of the entry against the query. Standard Robertson et al. 1994 defaults. Receives the *highest* default weight because BM25 is the most direct measure of relevance. There is one nuance: scores above 2.0 (very high lexical match) **bypass time decay** — a strong lexical match overrides recency preference. This is the "BM25 bypass rule," a sensible heuristic that prevents recent-but-irrelevant entries from outranking older-but-precisely-matching entries.

### ϕ_decay — exponential time decay, weight 0.25

```
ϕ_decay(t) = exp(−λ · Δt),   λ = 0.05
```

with Δt measured in days. Half-life ≈ 14 days. The justification in the paper: knowledge-work context windows typically span one to two weekly sprint cycles, so entries older than two weeks are unlikely to be directly relevant to current work.

The exception: entries from semantically-relevant sessions (identified in Stage 1, see two-stage retrieval below) **bypass decay regardless of age**. This is a recall-protection mechanism — if a fact from 90 days ago is still load-bearing for the current task, the agent should be able to surface it.

### ϕ_cw — cognitive weight, mapped to [0, 1], weight 0.25

```
ϕ_cw = (CW + 1) / 2
```

The retrieval signal corresponding to the attribution loop. Entries that have proven useful in past tool calls get a boost; entries linked to past failures get a discount. Initialised at CW=0 (ϕ_cw = 0.5), so a brand-new entry has neutral standing.

### ϕ_tier — tier boost, weight 0.15 (applied as additive bonus, not in dot product)

The tier multiplier µ_k ∈ {1.0, 1.2, 1.4} for episodic, semantic, and procedural tiers respectively. Applied as an additive bonus *outside* the dot product:

```
S = w⊤ ϕ + w_tier · (µ_k − 1)
```

where ϕ contains only the first four signals. This preserves the interpretation of `w⊤ ϕ` as a relevance score and treats the tier bonus as a separable promotion incentive. Procedural entries get a 40% boost over episodic; semantic entries get a 20% boost.

### ϕ_sem — semantic / dense embedding similarity, weight 0 (Phase 3)

The slot is reserved but not used in Phase 1/2. The paper's Phase 3 plan: turn this on, *normalise BM25 strictly*, and let dense retrieval shoulder the heavy lifting on multi-session synthesis (which BM25 cannot do — see the 0.180 multi-session score).

### Why the weight defaults

The paper calls these defaults "deliberately conservative starting values — the PPO trainer is designed to learn away from them." The reasoning is signal-reliability ordering:

- BM25 is the most direct measure of query relevance → highest weight (0.35).
- Time decay and cognitive weight are informative but noisier proxies → equal lower weight (0.25 each).
- Tier boost is structural (a tiebreaker, not a relevance signal) → smallest weight (0.15).

The PPO trainer (Phase 2b) is supposed to learn around these. Whether it does — and what the result tells us about the architecture — is a part-3 story.

## Two-stage retrieval: semantic scoping → episodic search

The retrieval engine does not just score against the entire memory pool. It runs a **two-stage** pipeline:

```
Stage 1: BM25 over semantic facts (project-shared)
         → top-5 most relevant session_ids

Stage 2: load episodic entries scoped to those session_ids
         → score with full 5-signal formula
         → top-k entries by score, packed into 300-token (default) budget
```

Why two stages? Three reasons:

1. **Scope reduction.** With 53 sessions of episodic data, the retrieval pool is enormous. Stage 1 cuts it down to the 5 sessions most likely to contain relevant material. Stage 2 then does the precise scoring on a small pool.
2. **Decay bypass.** Entries from semantically-relevant sessions bypass time decay. So if Stage 1 says "session #14 is relevant", every entry from session #14 enters Stage 2 without recency penalty.
3. **Better retrieval architecture.** This is conceptually a *coarse-to-fine* retrieval pattern: first find relevant *contexts*, then find relevant *details* within those contexts. It is more robust than a flat BM25 over all entries.

The ablation result confirms it: removing two-stage scoping costs −0.038 Acc on LongMemEval-S — the single largest signal-level removal cost in the table (more than removing time decay, cognitive weight, or tier boost individually). Two-stage scoping is the single most important component *after* semantic pre-population.

## Phase 1c: the attention-attributed cognitive weight loop

The cognitive weight is updated at agent_end (after the agent finishes). The loop:

1. Identify each tool call the agent made during the session.
2. For each tool call, identify the memory entries that *informed* it. Two methods:
   - **SGLang logprob attribution** — measure the logprob contribution of each prompt-injected memory entry to the actual tool-call tokens. Higher contribution → stronger attribution.
   - **Lexical Jaccard fallback** — compute Jaccard(call text, entry content). Higher overlap → stronger attribution.
3. For each (entry, tool-call) pair with non-trivial attribution, update the entry's CW:
   - If the tool call succeeded: CW += δ (proportional to attribution strength)
   - If it failed: CW -= δ
4. Clamp CW to [−1, 1].

The update is *online* and *non-parametric* — no model is trained, no embeddings are recomputed. The CW field is just incremented/decremented on the JSONL entry. (Implementation detail: because JSONL is append-only, the update is done by writing a *new* JSONL entry that supersedes the old one, or by maintaining a side-car CW table indexed by entry id. The paper does not specify which; both are valid.)

This loop is the smallest possible closed feedback mechanism for memory quality. It does *not* replace RL-based retrieval policy adaptation (that is Phase 2b's job), but it is a *first-line* feedback that runs without any RL machinery.

## Phase 2a: the consolidation daemon

The consolidation daemon is the second-most-important architectural piece (after the retrieval engine itself). It runs *asynchronously* and *off the agent's critical path* — i.e., it does not block the agent loop. Its job: promote episodic entries into the semantic tier as distilled facts.

The pipeline:

```
Periodic (e.g., daily):
  1. Read recent episodic JSONL entries
  2. Group by topic / project / session
  3. Apply heuristic + LLM fact extraction:
        - Heuristic: KV pattern matching (e.g., "X uses Y", "X = Y")
        - LLM:       prompt a small model to extract facts from each cluster
  4. Jaccard-deduplicate against existing semantic facts
  5. Write new semantic facts to the project-shared semantic tier
  6. Tag each new fact with origin agent + source session_ids
```

Two paths to fact extraction:

1. **Heuristic path (default fallback)**: KV-pattern extraction. Matches templates like `mentioned_in`, `prefers_X`, `decided_on_Y`. Cheap, but produces coarse labels (the paper acknowledges this as Limitation 3).
2. **LLM path (production)**: the consolidation daemon prompts a small extractor LLM (e.g., DeepSeek-V4-Flash in the evaluation) to read an episodic cluster and emit structured facts. This is the path that drives the 164× reduction in semantic-tier entries (509 heuristic → 3.1 LLM-extracted facts per question; Part 3 covers this).

Why is the daemon asynchronous?

- It can run on a different cadence than the agent loop (daily, hourly, on-idle, etc.).
- It is *recoverable* — if the daemon crashes or is killed, the agent's episodic JSONL is unaffected; on next start, the daemon resumes from where it left off.
- It is *cost-amortised* — the LLM extraction cost is paid once per fact-cluster, not once per query. Over many sessions, this amortises to roughly $0.05 per 500 questions at DeepSeek-V4-Flash pricing.

Compare with a synchronous on-write design ("every time the agent writes an episodic entry, immediately try to distil it into a semantic fact") — that approach burns inference budget on every single write, most of which contain nothing worth distilling. Async + clustered is the right tradeoff.

## Phase 2a (continued): multi-agent isolation

This is the architectural innovation that lets MEMTIER work for *multi-agent* orchestrations (e.g., OpenClaw's "Lobster" engine, mentioned in §3.1). The rule:

```
Episodic logs are agent-private by default.
Distilled semantic facts are project-shared.
```

Concretely:

- Each subagent writes only to its own episodic ledger.
- An orchestrator agent may be granted global read access via project-level configuration.
- The consolidation daemon reads from all agents' episodic ledgers (with permission) and writes to the *shared* semantic tier.
- All agents read from the shared semantic tier when constructing prompts.

The architectural diagram from the paper:

```
Multi-agent isolation layer:
  Agent A (episodic, private)  ↘
  Agent B (episodic, private)  →  Consolidation Daemon → Shared Semantic Tier
  Agent N (episodic, private)  ↗

Per-agent retrieval pipeline:
  ↓ Stage 1: BM25 on semantic facts (project-shared)
  ↓ Stage 2: episodic entries scoped to relevant sessions (agent-private)
  ↓ 5-signal scoring: w·[0, BM25, decay, CW, tier]
  ↓ Top-k, 300-token budget → prependSystemContext
  ↓ LLM call
  ↓ agent_end: write episodic (private) + attribution + CW update
  ↓ Consolidation daemon → semantic facts tagged with origin agent
  ↓ PPO updates weight vector w from session rewards
```

Why this matters for multi-agent systems:

1. **Context contamination prevention.** If subagent A is debugging a different feature than subagent B, A's episodic entries about its current debugging path should not leak into B's prompt. Multi-agent orchestrations like the [42-langchain-deep-agents](42-langchain-deep-agents.md) pattern have suffered from this kind of leakage; MEMTIER fixes it by default.
2. **Knowledge transfer through facts.** What B *should* learn from A is not "A debugged X by trying Y, then Z, then W" (that is path-dependent and noisy). What B *should* learn is "X is configured with Y" (the durable fact). The consolidation daemon does this distillation, then publishes to the shared tier.
3. **Origin tagging.** Every semantic fact carries the origin agent and source session_ids. So if you find a fact in the semantic tier that turns out to be wrong, you can trace it back to its source — important for debugging and for retracting bad facts.

This is the architectural answer to the Adaline "missing product layer for multi-agent systems" framing ([41](41-product-control-plane.md)). MEMTIER provides one of the four pillars Adaline named — the *visibility* + *recovery* primitive.

## Lifecycle integration: two hooks

MEMTIER integrates with OpenClaw via exactly two hooks:

### `before_prompt_build`

Fires before the agent's prompt is constructed. MEMTIER:
1. Looks at the user's current query (or the agent's latest action).
2. Runs the two-stage retrieval pipeline.
3. Selects top-k entries within the 300-token budget.
4. Prepends them to the system context as `prependSystemContext`.

The agent's underlying LLM never knows MEMTIER is there — it just sees a richer prompt with relevant memory.

### `agent_end`

Fires when the agent session ends. MEMTIER:
1. Walks the session's tool-call log.
2. For each tool call, runs the attribution analysis.
3. Updates the cognitive weight on each implicated entry.
4. Triggers the consolidation daemon (asynchronously) to consider whether any episodic entries from this session should be promoted to semantic facts.
5. Updates the PPO retrieval-policy state (Phase 2b).

The two hooks are the entire integration surface. No core OpenClaw modifications are required. This is *the* model for memory plugins in modern agent runtimes — minimal hook surface, async backend, schema-stable storage.

## Hyperparameter justification: why these defaults

The paper gives explicit, principled justifications for each default — uncommonly thorough for a memory-system paper, and worth pulling out:

- **w₀ = [0, 0.35, 0.25, 0.25, 0.15]**: signal-reliability ordering (BM25 → decay/CW → tier).
- **k1 = 1.5, b = 0.75 for BM25**: canonical Okapi defaults from Robertson et al. 1994.
- **λ = 0.05 (half-life ≈ 14 days)**: matches typical knowledge-work sprint cycles.
- **BM25 bypass threshold = 2.0**: requires at least two rare shared terms (IDF > 1) in a short document; "indicates a strong lexical match that should override recency preference."
- **µ_k ∈ {1.0, 1.2, 1.4}**: deliberately modest 20% / 40% boosts; tier acts as tiebreaker, not reranker.
- **k = 4 entries, 300-token budget**: defaults that balance recall and context cost.

Several of these were explicit candidates for PPO learning (Phase 2b). The Part-3 finding that PPO didn't move them is itself diagnostic.

## What the architecture is *not* doing

To prevent over-reading, three things the architecture explicitly does not include:

1. **Dense / semantic retrieval as a first-class signal.** ϕ_sem has weight 0 in defaults. This is intentional — the paper's diagnosis (BM25 ceiling) is contingent on this choice. Phase 3 will turn it on, but Phases 1–2 are about diagnosing what flat BM25 *can and cannot* do.
2. **Fine-grained typed-relation extraction.** The semantic tier is flat *facts*, not a typed graph. The paper acknowledges (Limitation 3) that heuristic KV-pattern extraction produces coarse labels and that a fine-grained NLP extractor would improve the tier. The KG route (covered in [128](128-knowledge-graphs-as-substrate.md)) is an alternative; MEMTIER does not commit to it.
3. **Working memory / scratchpad inside the agent loop.** MEMTIER is durable storage *between* turns. It does not replace the agent's in-prompt working memory or scratchpad ([12-todo-scratchpad-state](12-todo-scratchpad-state.md)). Those are complementary.

## Implementation guidance for harness builders

If you are building a memory plugin for your own harness (Claude Code skill, custom OpenClaw plugin, LangChain memory, custom runtime), the lessons from MEMTIER's architecture are concrete:

### 1. Pick a JSONL-per-day storage format

Daily JSONL files give you:
- natural sharding,
- safe concurrent appends,
- easy archival,
- easy line-streaming for retrieval,
- audit-friendly append-only semantics.

Avoid: a single monolithic Markdown or JSON file, a database with online schema migrations, or anything that requires a long-running process to be present for writes.

### 2. Encode structure in the schema

Even if you do not implement multi-tier consolidation immediately, encode the *schema fields* (id, timestamp, session_id, project, content, tokens, promoted, cognitive_weight) from day one. Adding fields later is painful; not having them blocks every later improvement.

### 3. Start with the cognitive-weight Jaccard fallback

It is trivial to implement (Jaccard between query and entry content), and it gives you the closed feedback loop with no RL machinery. Your retrieval starts learning from outcomes immediately. The SGLang logprob path is a future upgrade.

### 4. Build a two-stage retrieval pipeline

Even with a small corpus, two-stage scoping (coarse facts → fine-grained details) outperforms flat retrieval. The paper's −0.038 Acc cost for removing it is a strong signal.

### 5. Pre-distil the semantic tier with an LLM

Heuristic extraction produces an explosion of low-quality facts (509 vs 3.1 in MEMTIER's case). A small extractor LLM run async, once per cluster, is cheap and dramatically improves precision. This is *the* highest-leverage architectural choice in the whole paper (covered fully in Part 3).

### 6. Keep episodic agent-private, semantic project-shared

If you are doing multi-agent, this is the key isolation rule. Subagent episodic logs leak path-dependent noise; only distilled facts should cross.

### 7. Hook into your runtime via two events

`before_prompt_build` (read path) and `agent_end` (write path). That is the entire integration surface. Anything more is over-engineering.

## When the architecture wins

This architecture is the right default when:

- Your agent runs for hours-to-weeks per session.
- Retrieval is the dominant memory access pattern (vs. e.g., timeline scans, structured queries).
- You have multiple agents on the same project and need cross-agent knowledge transfer without context pollution.
- You want the option to layer RL-based policy improvement later.

It is *not* the right architecture when:

- Your memory is a structured knowledge base with explicit relations (use a KG; see [128](128-knowledge-graphs-as-substrate.md)).
- Your agent operates on a single short session and is restarted between (the four failure modes don't bite hard enough yet).
- You need transactional consistency across writes (use a durable workflow substrate; see [150](150-temporal-durable-execution-substrate.md)).
- Your retrieval pattern is *not* lexical — e.g., it is purely semantic ("find me memories about the *idea* of X, not the *word* X"). MEMTIER's BM25-dominant scoring will struggle.

## Failure modes & anti-patterns

Predictable ways to get the architecture wrong:

- **Skipping the semantic tier**: just episodic JSONL with retrieval. Loses the fact-distillation benefit. The MEMTIER ablation shows this costs −0.128 Acc — the largest single architectural cost in the table.
- **Mixing tiers in a single store**: episodic + semantic in the same file with a `tier` field. Loses the structural separation of concerns; consolidation daemon becomes harder; multi-agent isolation breaks (semantic facts can't be shared without leaking episodic context).
- **Synchronous consolidation**: distilling on every write. Burns inference budget; agent loop becomes slow.
- **Cognitive weight outside [−1, 1]**: unbounded weights let one entry dominate retrieval forever. Clamp.
- **Agent-shared episodic by default**: lets bad context bleed across subagents. Default to private.
- **Forgetting the BM25 bypass for semantic-relevant sessions**: time decay then crushes recall on important old context.

## Where this fits

- **Read after**: [151](151-memtier-why-flat-memory-breaks-at-72-hours.md) — the diagnostic problem framing.
- **Read next**: [153](153-memtier-llm-distillation-and-results.md) — LLM distillation, LongMemEval-S results, three-layer invariance.
- **Adjacent storage architectures**: [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md), [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md), [130-distributed-sql-as-agent-memory](130-distributed-sql-as-agent-memory.md), [131-temporal-bitemporal-tables](131-temporal-bitemporal-tables.md).
- **Adjacent memory primitives**: [09-memory-files](09-memory-files.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md), [72-claude-mem-persistent-memory-compression](72-claude-mem-persistent-memory-compression.md), [78-ngc-neural-garbage-collection](78-ngc-neural-garbage-collection.md), [81-reasoningbank](81-reasoningbank.md).
- **Multi-agent isolation parallels**: [41-product-control-plane](41-product-control-plane.md), [42-langchain-deep-agents](42-langchain-deep-agents.md), [73-multica-managed-agents-platform](73-multica-managed-agents-platform.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md).
- **Consolidation analogues**: [108-memento-codebase-mcp](108-memento-codebase-mcp.md) — case-bank rewriting as policy improvement; [134-semantic-indexing](134-semantic-indexing.md) — semantic-index design.

## References

1. Sidik & Rokach, *MEMTIER* — arXiv:2605.03675v1.
2. OpenClaw Contributors. 2026. OpenClaw: Personal AI assistant framework.
3. Robertson, Walker, Jones, Hancock-Beaulieu, Gatford. 1994. *Okapi at TREC-3*.
4. Lin. 2021. *A few brief notes on DeepImpact, COIL, and a conceptual framework for information retrieval techniques*. arXiv:2106.14807.
5. Wu et al. 2025. *LongMemEval*. arXiv:2501.08956.
6. Packer et al. 2023. *MemGPT*. arXiv:2310.08560 — the OS-paging analogue.
7. Anonymous. 2026. *AgentWarden: RL-based adaptive capability governance for AI coding agents*. NeurIPS 2026 Agent Safety Workshop (under review) — shared PPO infrastructure.
8. [09-memory-files.md](09-memory-files.md), [12-todo-scratchpad-state.md](12-todo-scratchpad-state.md), [72-claude-mem-persistent-memory-compression.md](72-claude-mem-persistent-memory-compression.md), [128-knowledge-graphs-as-substrate.md](128-knowledge-graphs-as-substrate.md), [129-kg-rag-hybrid-retrieval.md](129-kg-rag-hybrid-retrieval.md), [130-distributed-sql-as-agent-memory.md](130-distributed-sql-as-agent-memory.md), [131-temporal-bitemporal-tables.md](131-temporal-bitemporal-tables.md), [134-semantic-indexing.md](134-semantic-indexing.md).
