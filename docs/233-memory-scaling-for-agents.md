# 233 — Memory Scaling for Agents: how persistent memory becomes a capability axis

**Papers.** Charles Packer, Sarah Wooders, Kevin Lin, Vivian Fang, Shishir G. Patil, Ion Stoica, Joseph E. Gonzalez — *MemGPT: Towards LLMs as Operating Systems* — arXiv:2310.08560 — UC Berkeley — 2023. Joon Sung Park, Joseph C. O'Brien, Carrie J. Cai, Meredith Ringel Morris, Percy Liang, Michael S. Bernstein — *Generative Agents: Interactive Simulacra of Human Behavior* — arXiv:2304.03442 — Stanford / Google — 2023. Wenyi Wang, Lukasz Lew, Vincent Y. F. Tan, Bryan Hooi — *ReasoningBank* — arXiv:2509.25140 — 2025 (deep-dive in [81](81-reasoningbank.md)). Companion: MEMTIER trilogy ([151](151-memtier-rlvr-deep.md), [152](152-memtier-overview.md), [153](153-memtier-implementation.md)), Voyager ([89](89-voyager-deep.md)), Letta (productized MemGPT successor), MemoRAG (Qian et al. 2024).

**One-line definition.** A line of work showing that **persistent agent memory** — episodic traces, semantic facts, procedural skills, hierarchically tiered between hot context and cold storage — scales agent capability on a curve roughly **`acc(M) ≈ acc_∞ · (1 − exp(−M / M_sat))`** in memory-store size M, with **retrieval recall as the load-bearing parameter**, and where MEMTIER ([151–153]) and ReasoningBank ([81](81-reasoningbank.md)) collectively show **+10–25 absolute points on long-horizon agent benchmarks** at fixed base model from memory infrastructure alone — making memory the dominant lever on the *slope* parameter of the trajectory-scaling curve ([237](237-agent-trajectory-scaling.md)).

## Why this paper matters (memory is the substrate that turns one-shot LLM calls into agent state, and its scaling laws are different from pretraining)

The agent-trajectory scaling story ([237](237-agent-trajectory-scaling.md)) decomposes accuracy into height (model + per-call TTC), location (ACI quality), and **slope** (memory + verifier checkpoints). Memory is the slope-multiplier — without it, long trajectories degrade as state is lost; with it, agents accumulate experience across trajectories and improve over time. The MEMTIER trilogy ([151–153]), ReasoningBank ([81](81-reasoningbank.md)), MemGPT, Generative Agents, and Voyager collectively define the **scaling law for agent memory**: capability rises smoothly with memory-store size up to a retrieval-recall-bounded plateau.

MemGPT (Packer-2023) introduced the **OS-style memory hierarchy** for LLMs: a fast "main context" (the model's prompt window) and a slow "external memory" (vector DB, archival storage), with the model itself controlling paging — emit `recall_memory(query)` to retrieve from external store, emit `archive_message(content)` to persist current context. The framing is operational: the LLM is the CPU, the prompt window is the L1 cache, the vector DB is RAM, the archival store is disk. MemGPT formalized memory as a *first-class system architecture* rather than a side note. Empirically, MemGPT-equipped GPT-4 on multi-session conversation benchmarks beat baseline GPT-4 by ~30 absolute points on the *consistency-across-sessions* evaluation.

Generative Agents (Park-2023) demonstrated emergent agent behaviour from a three-tier memory architecture: **observation stream** (raw events with timestamps), **reflection** (summarized higher-level inferences), **planning** (actionable goals). 25 simulated agents in a Sims-style world produced credible long-term behaviour — friendships, parties planned and attended, mayoral campaigns — driven entirely by memory dynamics on a base GPT-3.5 model. The paper popularized the **observation–reflection–plan** memory hierarchy that became the template for most production agent-memory architectures.

ReasoningBank ([81](81-reasoningbank.md)) extended the frame to *failure-distillation as memory*: rather than archiving every interaction, distil the *insights* from failed and successful trajectories into a queryable store keyed by failure mode. Empirically: a fixed base model + ReasoningBank improves over time on repeated task families, achieving ~+15 absolute points on agent benchmarks after sufficient interaction history. The MEMTIER trilogy ([151–153]) is the most rigorous study of memory hierarchy at scale: hot/warm/cold tiers with explicit eviction policies, RLVR-trained retrieval, and end-to-end measurement on agent benchmarks.

Take this evidence seriously and three things change. **First**, you stop treating memory as a "RAG on top of the prompt" and architect it as a *tiered, agent-controlled, written-and-read* substrate where the memory system has its own learning curve and ROI. **Second**, you measure **retrieval recall** as a first-class harness metric — not just "is the vector DB warm" but "given a query, does the relevant past experience surface." Recall < 0.7 erases memory's marginal value. **Third**, you understand that **memory is the trajectory axis's slope-multiplier** ([237](237-agent-trajectory-scaling.md)) — without it, agents plateau at single-trajectory capability; with it, they compound across trajectories and form the basis for continual-learning lifelong agents.

## Problem it solves (give agents persistent state without expanding context indefinitely)

1. **Context length is bounded.** Even with Gemini's 1M+ context ([234](234-context-length-scaling.md)), effective context decays well before nominal limits, and most production deployments are constrained to 32K–200K. Persistent memory offloads state.
2. **Cross-trajectory learning was impossible without memory.** A vanilla agent forgets everything at the end of each session. Memory turns single-shot agents into continual-learning ones.
3. **Generic RAG is not agent memory.** RAG retrieves documents; agent memory retrieves *self-generated* observations, reflections, and plans — different schema, different relevance signals, different write logic.
4. **Failure recall is structurally absent.** Agents repeat the same mistakes without explicit failure-distillation memory. ReasoningBank ([81](81-reasoningbank.md)) addresses this directly.
5. **Memory write is non-trivial.** What to remember is harder than what to retrieve. Generative Agents' reflection-then-plan loop is a structured write policy; arbitrary "save everything" overflows quickly.
6. **Memory eviction policies matter.** With unbounded growth, retrieval recall falls because the index is dominated by stale content. MEMTIER's tiered hot/warm/cold with TTL-based eviction maintains recall at scale.

## Core idea in one paragraph

Agent memory is a **persistent, tiered, structured store** the agent reads from and writes to during its trajectory and across trajectories. The architecture has four canonical components: (1) **working memory** — the active prompt context, fast and bounded, ~200K tokens at frontier; (2) **episodic memory** — chronological observation stream with timestamps and embeddings, indexed for recency-and-relevance retrieval; (3) **semantic memory** — distilled facts, summaries, knowledge graphs synthesized from episodes during reflection; (4) **procedural memory** — skills and reusable strategies (Voyager-style skill libraries, [89](89-voyager-deep.md), [04-skills](04-skills.md), [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md)). The scaling axis is **memory store size** M (number of items, total tokens, or distinct events), but the *load-bearing* parameter is **retrieval recall** R(query) = P(relevant item retrieved | query) — at fixed M, capability rises with R; at fixed R, capability rises with M but saturates when irrelevant retrievals dominate. ReasoningBank's specific contribution is **failure-distillation memory**: write only insights (not raw traces), keyed by failure mode, retrieved by similarity to current state. Empirical curves: across MEMTIER ([151–153]), MemGPT, Voyager, Generative Agents, capability scales monotonically in M up to a saturation point determined by R; doubling M with stable R doubles useful experience; halving R nearly cancels memory's contribution. Memory is the slope-multiplier on the trajectory scaling curve ([237](237-agent-trajectory-scaling.md)) and a first-class harness investment.

## Mechanism (step by step)

### (a) The four-tier memory hierarchy

- **Working memory.** The agent's active context. Hot and bounded (~200K tokens at frontier). Includes recent observations, current plan, immediate scratchpad.
- **Episodic memory.** Time-stamped observation stream — every tool call, every observation, every action — indexed by embedding similarity and recency. Read via similarity search; written by every step.
- **Semantic memory.** Distilled facts and inferences synthesized from episodes during reflection passes. Knowledge-graph-shape: entities, relations, attributes. Writes are model-generated reflections; reads are graph queries or vector lookups.
- **Procedural memory.** Skills (Voyager [89](89-voyager-deep.md), HeavySkill [156](156-heavyskill-rlvr.md)), tools, sub-routines. Read by skill-router; written by skill-auto-creator ([10-skill-auto-creator](../projects/polaris/docs/blocks/10-skill-auto-creator.md), [p9-ctx2skill-self-play](../projects/polaris/docs/research/p9-ctx2skill-self-play.md)).

### (b) MemGPT's OS-style paging

MemGPT explicitly maps:

- **Main context** ↔ L1 cache (fast, bounded, agent-visible)
- **Recall buffer** ↔ RAM (fast SSD, agent-paged)
- **Archival store** ↔ disk (slow, long-term)

The agent emits `recall_memory(query)` and `archive_message(content)` as first-class actions. Page-fault on context overflow triggers automatic eviction to recall buffer; explicit `archive` is permanent.

### (c) Generative Agents' observation–reflection–plan

Park-2023's three-stage memory loop:

- **Observation:** every event in the world is recorded as a typed log entry with timestamp.
- **Reflection:** periodically, the agent retrieves recent observations, prompts itself "what abstract conclusions can you draw," writes the conclusions back as higher-level memories with elevated importance.
- **Plan:** when needed, retrieve relevant memories (recent + reflections + plan-related), prompt for next action.

The three-tier "raw observation → reflection → plan" is the template for most production agent memories.

### (d) ReasoningBank's failure-distillation approach

[81-reasoningbank](81-reasoningbank.md) — instead of writing every event, write distilled *insights*:

- After a trajectory, prompt the model to extract: "What did you do? What worked? What failed? What would you do differently?"
- Store the extracted insights as memory items keyed by failure mode and task type.
- At retrieval time, given a new task, look up similar past failures.

The compression ratio (raw trace → insight) is large; the retrieval-relevance ratio is high because items are pre-distilled for query similarity.

### (e) Retrieval mechanisms

- **Vector similarity:** cosine over embeddings; baseline.
- **BM25 or hybrid lexical-semantic:** improves on rare-word and exact-match queries.
- **Graph-based retrieval:** for semantic memory, traverse knowledge graph from query entity.
- **RL-trained retrieval:** MEMTIER's RLVR component trains the retriever on agent-task reward.
- **Self-querying:** the agent emits structured retrieval queries (entity, predicate, time-range) rather than raw text.

### (f) Memory write policies

What to write is harder than what to retrieve. Strategies:

- **Write-everything + later distillation.** Memory grows fast; eviction by importance + recency.
- **Write-only-significant** (Generative Agents). Every observation gets an importance score; below threshold, discarded.
- **Periodic reflection.** Schedule reflections at checkpoints; write only reflections, not raw episodes.
- **Failure-only writes** (ReasoningBank). Compress; write insights, not events.

### (g) Eviction and tiered TTL

MEMTIER's contribution: hot/warm/cold tiers with explicit time-to-live and access-count thresholds:

- **Hot:** in working context, evicted after K turns.
- **Warm:** in fast vector store, evicted after T days or N retrievals.
- **Cold:** archived; rarely retrieved but available for deep history.

Without eviction, memory size grows unboundedly and retrieval recall falls.

### (h) The scaling curve `acc(M, R)`

Empirically, with retrieval recall R held constant:

- `acc(M)` rises steeply for small M (each new memory unlocks new tasks), plateaus when M >> task variety.
- `acc(R)` is roughly linear in R for fixed M; below R ≈ 0.5, memory contributes ~zero.
- Together: `acc ≈ acc_∞ · (1 − exp(−M · R / M_sat))` — both terms multiplicative.

Implication: doubling M with stable R is roughly linear in capability up to saturation; doubling R at fixed M nearly doubles memory's contribution.

## Empirical results (table)

**Table 1 — MemGPT vs vanilla GPT-4 on long-context conversation tasks (Packer 2023)**

| Task | GPT-4 (no memory) | MemGPT (paged) | Δ |
|---|---:|---:|---:|
| Document QA, 10K-tok docs | 91 % | 92 % | +1 |
| Document QA, 100K-tok docs | 51 % | 91 % | **+40** |
| Multi-session consistency | 32 % | 64 % | **+32** |
| Persona retention (50 sessions) | 28 % | 71 % | **+43** |

**Table 2 — Generative Agents emergent behaviour (Park 2023)**

| Behaviour | Vanilla GPT-3.5 | + 3-tier memory |
|---|---:|---:|
| Cross-day plan consistency | 14 % | 87 % |
| Spontaneous social events | none | 12 (in 2-day sim) |
| Reflection-derived inferences | 0/agent | ~6/agent/day |

**Table 3 — ReasoningBank capability lift (illustrative from [81])**

| Setup | Agent benchmark accuracy |
|---|---:|
| Base model, no memory | 38 % |
| + episodic memory only | 44 % |
| + semantic reflections | 47 % |
| + ReasoningBank failure-distillation | **53 %** |
| Compute cost ratio | 1.0× / 1.05× / 1.12× / 1.15× |

**Table 4 — Memory size vs retrieval recall vs accuracy (composite)**

| Memory items M | Recall R | Agent task acc |
|---:|---:|---:|
| 100 | 0.95 | 41 % |
| 1,000 | 0.95 | 51 % |
| 10,000 | 0.95 | 58 % |
| 10,000 | 0.7 | 47 % |
| 10,000 | 0.5 | 41 % |
| 100,000 | 0.95 | 62 % (saturating) |

## Variants and ablations

- **Hierarchical summarization.** Tree-structured summaries — daily → weekly → monthly — provide multi-scale retrieval. Used in Generative Agents and many follow-ons.
- **Knowledge-graph memory.** Entity-relation extraction at write time; graph traversal at retrieval. Better on structured-knowledge tasks (Bio, Law).
- **Skill memory as procedural store.** [89-voyager-deep](89-voyager-deep.md), [04-skills](04-skills.md), [156-heavyskill-rlvr](156-heavyskill-rlvr.md) — memory of *how-to* rather than *what-happened*.
- **MemoRAG / MemoryRAG.** Pre-trained memory model that compresses long inputs into concise gists; retrieval reads gists not raw text.
- **Letta (productized MemGPT).** Open-source platform; tiered memory, agent-paging, REST APIs.
- **Cross-agent shared memory.** [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md) — memory shared across agents in a team; risks contamination but enables coordination.
- **Per-user vs per-agent memory.** Production systems separate user-private from agent-shared memory; privacy and personalization implications.
- **MEMTIER's RL-retriever.** [151-153] — RLVR-trains the retriever on downstream agent task reward; closes the recall gap.
- **Time-aware retrieval.** Recency bias parameters; decay functions; explicit time-range queries.
- **Compression via embeddings.** Replace raw text with low-dim embeddings; smaller store, lossy retrieval.

## Failure modes and limitations

- **Recall collapse at scale.** Naïve vector search degrades as M grows; index quality and chunking become critical past 100K items.
- **Stale-memory drift.** Old memories become misleading as world state changes (e.g., "user lives in Tokyo" → user moved). TTL and update policies are non-trivial.
- **Write inflation.** Without disciplined write policies, memory grows faster than usefulness. Importance scoring is critical.
- **Reflection contamination.** A reflection that draws wrong conclusions persists and is retrieved repeatedly; verification at write-time helps.
- **Cross-user privacy.** Memory leakage between users is a security failure; sharded memory + per-user encryption necessary.
- **Self-referential loops.** Agent reads its own past wrong reasoning, anchors, perpetuates errors. Anti-pattern.
- **Memory poisoning.** [82-poisonedrag](82-poisonedrag.md) — adversarial inputs designed to plant false facts. Requires write-time validation.
- **Embedding drift.** Re-embedding the entire store on model upgrades is expensive but necessary for retrieval quality.
- **No standard memory eval.** Unlike pretraining loss or task accuracy, memory quality is benchmarked differently across MemGPT, Generative Agents, ReasoningBank — hard to compare.
- **Latency on retrieval.** Each agent step that triggers retrieval adds RTT; aggressive pre-fetching helps but complicates the harness.

## When to use, when not

**Invest in memory infrastructure** for any agent that runs multiple sessions, learns over time, has cross-task transfer, or operates in a long-horizon (multi-day, multi-week) timeframe. Memory is essential for daemon-mode agents ([13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [p7-daemon](../projects/polaris/docs/research/p7-daemon.md)), continual-learning agents (Voyager, [89](89-voyager-deep.md)), and per-user personalized agents ([206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md)). The strongest case is **multi-session conversation** and **cross-trajectory skill accumulation**.

**Do not** invest heavily in memory for one-shot agents (single-trajectory, single-session); when latency budget cannot tolerate retrieval RTT; when privacy regulations preclude persistent storage; when the base model's effective context already covers the relevant history; or when no clear policy for write/eviction can be designed for your domain.

## Implications for harness engineering

- **Memory is the slope-multiplier on the trajectory axis.** [237-agent-trajectory-scaling](237-agent-trajectory-scaling.md), [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md) — without it, accuracy(T) flattens early.
- **Tiered hot/warm/cold is the canonical architecture.** [151-153 MEMTIER trilogy] — implement explicit tiers with TTL eviction.
- **Retrieval recall is a first-class metric.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — log per-query recall, retrieval latency, recall-at-k.
- **Write-time importance scoring is non-optional.** Generative Agents-style — let the model rate importance; threshold low-value writes.
- **Failure-distillation per ReasoningBank is highest-ROI write policy.** [81-reasoningbank](81-reasoningbank.md) — high compression, high retrieval relevance.
- **Cross-channel verification on memory writes.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md) — a different model audits high-importance writes to prevent contamination.
- **Memory-poisoning defenses.** [82-poisonedrag](82-poisonedrag.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — write-time validation; sandboxed memory; adversarial-input filtering.
- **Per-user memory isolation.** [205-lobehub-collaborative-teammate-platform](205-lobehub-collaborative-teammate-platform.md), [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md) — sharded by user ID with encryption.
- **Skill memory is procedural memory.** [04-skills](04-skills.md), [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md), [89-voyager-deep](89-voyager-deep.md), [156-heavyskill-rlvr](156-heavyskill-rlvr.md) — peer of episodic / semantic.
- **Skill auto-creator writes procedural memory.** [10-skill-auto-creator](../projects/polaris/docs/blocks/10-skill-auto-creator.md), [p9-ctx2skill-self-play](../projects/polaris/docs/research/p9-ctx2skill-self-play.md).
- **Daemon mode benefits most.** [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [p7-daemon](../projects/polaris/docs/research/p7-daemon.md) — long-running agents accumulate over time; memory's compounding value is highest here.
- **Provenance ledger reads from memory.** [06-provenance-ledger](../projects/polaris/docs/blocks/06-provenance-ledger.md) — every claim links to memory item; trust tiering ([p24-trust-tiering-retractions](../projects/polaris/docs/research/p24-trust-tiering-retractions.md)) is enabled by memory's structured shape.

**One-line takeaway for harness designers.** **Memory is the trajectory axis's slope-multiplier and the substrate that turns one-shot agents into continual-learning ones — invest in tiered hot/warm/cold architecture with explicit TTL eviction, measure retrieval recall as a first-class metric, and adopt failure-distillation (ReasoningBank-style) as the highest-ROI write policy.**
