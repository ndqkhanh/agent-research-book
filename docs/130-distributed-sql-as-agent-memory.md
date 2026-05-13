# 130 — Distributed SQL as Agent Memory: When Transactional Consistency Is the Right Substrate for Shared Agent State

**Sources.** Stewart & Huang, *Agentic AI Data Architectures*, Chapter 2 (How Distributed SQL Unifies Enterprise Scale and AI-Native Application Design); the distributed-SQL literature (CockroachDB, YugabyteDB, TiDB, Spanner, PingCAP papers); plus the broader HTAP / hybrid-transactional-analytical processing discipline.

**One-line definition.** Distributed SQL — relational databases that preserve ACID semantics across geographically distributed nodes (CockroachDB, YugabyteDB, TiDB, Google Spanner) — is the right substrate for agent memory when the memory is *shared* (multiple agents read/write), *transactional* (consistency matters), *queryable in SQL* (relational filtering and aggregation), and *unified with vectors* (semantic + relational in one engine); for these workloads it dominates document stores, key-value caches, and pure vector databases by collapsing what would otherwise be a multi-store stack into one.

## Why this matters

Agent memory in 2024 was usually one of: a JSONL file ([108-memento-codebase-mcp](108-memento-codebase-mcp.md)), a vector DB, a Redis cache, a SQLite database. Each works for a single agent on a small scale. None work well for *many agents sharing state in production*: shared workspaces, agent-to-agent commitments, multi-tenant memory, multi-region deployment.

Distributed SQL emerged in 2018–2024 (CockroachDB, Yugabyte, TiDB, Spanner) to solve the same problem for general workloads — relational consistency at cloud scale. By 2026, the same engines have added vector indexes, transforming them into substrate for agentic AI. They unify three things that previously required three stores: relational state (tables, joins, transactions), vector search (embeddings, similarity), and time-aware reasoning (temporal tables — see [131-temporal-bitemporal-tables](131-temporal-bitemporal-tables.md)).

For agent builders running multi-agent systems, distributed SQL is the substrate to consider when the existing stack of vector DB + Redis + Postgres has become operationally complex. Consolidating to one engine reduces operational surface, eliminates consistency hazards across stores, and simplifies the data model.

## Problem it solves

Five concrete pain points distributed SQL addresses for agent platforms:

1. **Cross-store inconsistency.** Vector DB has the embeddings; Postgres has the metadata; they go out of sync after a partial-failure write.
2. **Multi-agent shared state.** Agent A and Agent B need to agree on the value of `loan_application.status`. Without ACID, they don't.
3. **Multi-region.** EU agent needs EU-resident data; US agent needs low-latency access. Distributed SQL handles regional placement.
4. **Burst capacity.** Agentic workloads are spiky; cloud-native distributed SQL elastically scales.
5. **Vector + relational in one query.** "Find applications similar to this one *and* status = 'pending' *and* assigned_to = me." Single SQL query.

Each is a real production limitation of the multi-store status quo.

## Core idea in one paragraph

Distributed SQL preserves the SQL data model — tables, joins, ACID transactions — across a cluster of nodes. Modern distributed SQL engines (CockroachDB, YugabyteDB, TiDB, Spanner) add: **vector indexes** (HNSW, IVF) so embeddings live in the same engine as relational data; **temporal tables** ([131-temporal-bitemporal-tables](131-temporal-bitemporal-tables.md)) so time-aware reasoning is native; **change data capture** ([132-vector-cdc-pipelines](132-vector-cdc-pipelines.md)) so downstream consumers stay synchronised; **multi-region replicas** so data follows users; **strong consistency** so multi-agent state is unambiguous. For agent platforms, this means one engine for: agent memory tables, embedded case banks, shared workspace state, audit logs, multi-tenant data partitions. Operational footprint shrinks from "vector DB + cache + Postgres + queue" to "distributed SQL plus a queue", and consistency hazards drop dramatically. Cost: distributed SQL is more expensive per query than single-node Postgres or pure vector DBs; the trade-off pays off when multi-agent, multi-region, or transactional consistency matters.

## Mechanism (step by step)

### 1. Why distributed SQL — the consistency-availability trade

Classical CAP framing: in a partition, you choose consistency or availability. Distributed SQL engines typically choose **strong consistency** (Spanner-style, Raft-based) and provide **high availability** through replication, accepting modest latency cost for cross-region writes.

For agent state, this is usually the right choice:
- Agent A writing "approved" must be visible to Agent B reading "status".
- Without strong consistency, you get either lost writes (bad) or lengthy retry logic in every agent.
- The latency cost (milliseconds for cross-region commits) is acceptable for agent workloads (already at hundreds of ms for LLM calls).

### 2. The unified data model

A typical agent platform schema in distributed SQL:

```sql
-- Agent state
CREATE TABLE agents (
  id UUID PRIMARY KEY,
  name TEXT,
  version TEXT,
  status TEXT,
  created_at TIMESTAMPTZ
);

-- Tasks shared across agents
CREATE TABLE tasks (
  id UUID PRIMARY KEY,
  tenant_id UUID,
  type TEXT,
  status TEXT,                      -- 'pending', 'in_progress', 'done'
  assigned_agent_id UUID,
  payload JSONB,
  created_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ
);

-- Memory (case bank a la Memento)
CREATE TABLE cases (
  id UUID PRIMARY KEY,
  tenant_id UUID,
  task_type TEXT,
  state JSONB,
  action JSONB,
  reward FLOAT,
  embedding VECTOR(384),            -- pgvector / native vector index
  meta JSONB,
  created_at TIMESTAMPTZ
);
CREATE INDEX cases_emb ON cases USING hnsw (embedding);

-- Audit log (immutable; see [122])
CREATE TABLE audit_log (
  id UUID PRIMARY KEY,
  tenant_id UUID,
  ts TIMESTAMPTZ,
  trace_id UUID,
  event JSONB,
  signature BYTEA
);
```

A single engine holds: relational task state, vector-indexed case memory, audit log. One backup, one access pattern, one transaction model.

### 3. Multi-agent shared state via transactions

Two agents claim the same task safely:

```sql
BEGIN;
UPDATE tasks
SET status = 'in_progress', assigned_agent_id = $1, updated_at = NOW()
WHERE id = $2 AND status = 'pending';
-- If 0 rows updated: another agent claimed it.
SELECT * FROM tasks WHERE id = $2 AND assigned_agent_id = $1;
COMMIT;
```

Transactional CAS (compare-and-set) on the status column. No lost updates, no duplicate work.

This is a classic pattern; the difference in distributed SQL is that it works *across regions* with the same semantics.

### 4. Vector + relational queries — the unified retrieval

```sql
SELECT id, state, reward,
       embedding <=> $query_embedding AS similarity
FROM cases
WHERE tenant_id = $tenant
  AND task_type = 'underwriting'
  AND reward >= 0.8
ORDER BY embedding <=> $query_embedding
LIMIT 4;
```

One query: tenant filter (relational), task-type filter (relational), reward filter (relational), vector similarity (vector index), top-K (relational). Cannot do this in pure-vector engines.

### 5. Multi-region placement

Distributed SQL engines support per-table regional pinning:

```sql
ALTER TABLE eu_tasks SET LOCALITY REGIONAL BY ROW;
-- Each row's region is determined by a column (e.g. tenant_region).
```

Practical effect: EU customers' tasks are stored on EU nodes, satisfying GDPR data-residency. Multi-region clusters route reads to the nearest replica.

### 6. CDC for downstream consumers

Distributed SQL emits change streams:

```text
-- CockroachDB
CREATE CHANGEFEED FOR TABLE cases INTO 'kafka://...';

-- TiDB
CREATE TABLE cases ... ;
-- TiCDC streams to Kafka or other sinks
```

Consumers (vector index re-indexers, analytics warehouses, downstream agents) subscribe to the stream. See [132-vector-cdc-pipelines](132-vector-cdc-pipelines.md).

### 7. Temporal queries

Time-aware reasoning is built in (or close to it):

```sql
-- 'system_time' temporal column tracks valid time
SELECT * FROM tasks
WHERE id = $1
AND system_time AT '2026-04-01';
```

Returns the row's value as of the specified time. Critical for "what did the agent know when" reasoning. Detail in [131-temporal-bitemporal-tables](131-temporal-bitemporal-tables.md).

### 8. Cost and operational considerations

- **Per-query cost**: distributed SQL is 2–10× the cost of single-node Postgres for the same query, due to consensus overhead.
- **Cross-region writes**: 50–200ms vs 1–10ms intra-region. Plan accordingly.
- **Operational complexity**: less than running a fleet of single-node DBs; more than serverless single-node.
- **Schema evolution**: easier than legacy Postgres at scale (online schema changes are typical).
- **Backup and restore**: built-in continuous backup; point-in-time recovery.

### 9. Vendor landscape (2026)

| Engine | License | Strengths | Weaknesses |
|---|---|---|---|
| **CockroachDB** | OSS + commercial | Postgres compatibility, strong global ACID, mature | Cost; query optimiser still evolving |
| **YugabyteDB** | OSS + commercial | Postgres + Cassandra APIs, strong consistency | Smaller community than CockroachDB |
| **TiDB** | OSS + commercial | MySQL compatibility, HTAP via TiFlash | Operational complexity at scale |
| **Google Spanner** | Managed only | Most mature, global SQL | Locked to GCP; cost |
| **Postgres + Citus** | OSS | Familiar; sharding | Not true distributed SQL; weaker consistency story |
| **Aurora Distributed** | Managed only | AWS native | Newer; vendor-locked |

For most agent platforms in 2026: **CockroachDB or YugabyteDB** for cloud-portable; **Spanner** for GCP-committed.

### 10. When to migrate from Postgres / vector-DB stack

Three signals:
- **Cross-store consistency bugs** start appearing in incident reports.
- **Multi-region** becomes a regulatory or latency requirement.
- **Operational toil** of running multiple stores exceeds the cost of one distributed SQL engine.

Migration is non-trivial (data model translation, query rewrites); plan for 3–6 months for a meaningful platform.

## Empirical anchors

- **Cross-store inconsistency** is the #1 complaint of multi-store agent platforms.
- **Vector + relational in one engine** has become standard in 2026 distributed SQL.
- **Multi-region writes** are 50–200ms; agent workflows tolerate this.
- **Per-query cost** is 2–10× single-node Postgres; total TCO often lower due to operational savings.
- **Adoption** in agent platforms is growing fast in 2025–2026 enterprise contexts.
- **Postgres compatibility** in CockroachDB / YugabyteDB cuts migration cost dramatically.

## Variants and counter-arguments addressed

- **"Postgres + read replicas is enough."** Up to a point; not for multi-region writes or strong cross-region consistency.
- **"Vector DB is faster for vector workloads."** True at the limit; the distributed SQL vector indexes are within 2–3× of dedicated engines, often acceptable.
- **"Distributed SQL is too expensive."** Per-query yes; total TCO depends on what it replaces.
- **"NewSQL is hype."** It's mature in 2026; CockroachDB has been production for 8+ years.
- **"Just use a managed vector DB and Postgres."** Two stores; consistency hazard. Acceptable for many agents; problematic for multi-agent state.

## Failure modes and limitations

1. **Cross-region latency surprise.** Naive queries that touch multiple regions are slow; design schema for locality.
2. **Schema evolution at scale.** Online schema changes work but slow for very large tables; plan migrations.
3. **Vector-index cost.** Large vector indexes (millions of rows) need significant compute; tune.
4. **Query optimiser quirks.** Distributed query planners are less mature than single-node; some queries need hints.
5. **Cost model unfamiliarity.** Engineers used to Postgres-for-cents underestimate distributed-SQL spend.
6. **Operational complexity vs serverless.** Self-managed clusters need DBAs; managed offerings cost more.
7. **Multi-tenant noisy neighbour.** One tenant's bursty workload affects others; isolation requires care.
8. **Cold cache on regional fail-over.** Replica becomes leader; cache empty; latency spikes.

## When to use, when not

**Use distributed SQL when** multi-agent shared state matters, multi-region is required, vector + relational unification is valuable, or operational toil of a multi-store stack is high.

**Stick with Postgres + vector DB** for single-region, single-tenant, modest-scale agent platforms.

**Use lightweight stores (SQLite, JSONL)** for single-agent prototypes.

**Don't migrate prematurely.** The cost is real; advance maturity stage by stage ([118-genai-maturity-models](118-genai-maturity-models.md)).

## Implications for harness engineering

- **Plan for stage-3 substrate consolidation.** When multi-agent and multi-region kick in, distributed SQL is often the answer.
- **Keep schemas tenant-aware.** `tenant_id` column on every table; row-level security if available.
- **CDC from day one.** Downstream consumers shouldn't poll; they should subscribe. See [132-vector-cdc-pipelines](132-vector-cdc-pipelines.md).
- **Vector index governance.** Embeddings have lifecycle: re-embed on model change; version columns.
- **Audit log in the same engine.** [122-explainability-compliance](122-explainability-compliance.md) — keeps audit consistent with state.
- **Locality-aware schema for multi-region.** Per-row regional placement on user-bearing tables.
- **Observability of consensus latency.** Cross-region commit times; regional leader health.
- **Cost dashboards per workload.** Distributed SQL cost is non-trivial; attribute and optimise. See [125-system-level-production-patterns](125-system-level-production-patterns.md).

The one-sentence takeaway: **distributed SQL collapses the multi-store agent platform stack into one engine — relational + vector + temporal + audit — at the cost of higher per-query expense, justified when multi-agent state, multi-region, or transactional consistency are real requirements.**

## See also

- [09-memory-files](09-memory-files.md) — file-based memory, the simpler alternative.
- [108-memento-codebase-mcp](108-memento-codebase-mcp.md) — JSONL memory in practice.
- [125-system-level-production-patterns](125-system-level-production-patterns.md) — system-level concerns this substrate addresses.
- [131-temporal-bitemporal-tables](131-temporal-bitemporal-tables.md) — temporal queries in distributed SQL.
- [132-vector-cdc-pipelines](132-vector-cdc-pipelines.md) — CDC patterns.
- [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md), [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md) — graph alternative for relational data.
- [122-explainability-compliance](122-explainability-compliance.md) — audit log in the same engine.
