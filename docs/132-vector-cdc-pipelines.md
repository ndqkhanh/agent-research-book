# 132 — Vector + CDC Pipelines for Live RAG: Embedding Freshness via Change Data Capture

**Sources.** Stewart & Huang, *Agentic AI Data Architectures* (the CDC + vector embedding pattern); the CDC literature (Debezium, Kafka Connect, Flink CDC); plus modern RAG-freshness papers and the staleness literature on RAG quality degradation.

**One-line definition.** Vector + CDC pipelines treat embedding regeneration as a *streaming side-effect* of source-data changes — every insert, update, or delete in the source table triggers an embedding re-compute and vector-index update through a Kafka-style change stream — turning RAG from a periodic-rebuild model that goes stale between rebuilds into a continuous-freshness system where the index is always consistent with the source within seconds.

## Why this matters

The default 2024 RAG pipeline was: nightly job re-embeds new documents; on busy days the index lags. For agents serving customer-facing queries, lag is bad: the customer asks about a policy update from this morning; the index doesn't know it yet; the agent gives an outdated answer; user trust erodes.

CDC (Change Data Capture) is the well-established pattern from data engineering for streaming database changes. Applying it to vector indexes means embedding updates flow as events through a queue, picked up by an embedding worker, written to the index — same primitives, new application. The result is that vector freshness becomes a function of pipeline latency (typically seconds), not batch cadence (hours or days).

For agent builders, this is the difference between "RAG is occasionally embarrassing" and "RAG is reliably current." The engineering cost is moderate; the trust gain is large.

## Problem it solves

Five concrete failures stale RAG produces:

1. **Outdated answers.** User asks about today's policy update; index has yesterday's. Agent answers wrong.
2. **Phantom data.** A document was deleted; index still returns it. User confused.
3. **Inconsistent multi-store state.** Source DB updated; vector index lagging; KG updated; three sources disagree.
4. **Re-index storms.** Nightly batch re-embeds millions of unchanged documents because change tracking is missing.
5. **Embedding-model migration pain.** Switching embedding models requires re-embedding everything; without CDC, this is a manual orchestration nightmare.

CDC streaming addresses each.

## Core idea in one paragraph

Source-of-truth data lives in a relational store (or document store, or KG). Every change to that data emits a change event — typically through built-in CDC (CockroachDB changefeeds, Debezium for Postgres, MySQL binlog) — onto a streaming substrate (Kafka, Pub/Sub, Kinesis). Downstream consumers subscribe: an *embedding worker* reads events, computes embeddings for changed rows, writes the embeddings (with metadata) to the vector index. The index stays within seconds of the source. When the embedding model changes, a *re-embedding worker* re-processes all rows; CDC infrastructure makes the re-process safe to interleave with normal updates. This pattern generalises: the same CDC stream feeds vector indexes, KGs, search indexes, audit logs, analytics warehouses — any downstream that needs to stay in sync with the source. The cost is the streaming infrastructure; the benefit is *freshness as a property*, not an aspiration.

## Mechanism (step by step)

### 1. Architecture overview

```text
[source DB / table updates]
   ↓ CDC emits change events
[event stream: Kafka / Pub/Sub / Kinesis]
   ↓ topics partitioned by entity / table
   ├──→ [embedding worker]    → vector index
   ├──→ [KG updater]          → graph DB
   ├──→ [search indexer]      → ElasticSearch / OpenSearch
   ├──→ [audit log writer]    → audit store
   └──→ [analytics warehouse] → BigQuery / Snowflake / Redshift
```

One source, many sinks. CDC is the fan-out point.

### 2. CDC emission

Modern relational engines emit changes natively:

```sql
-- CockroachDB
CREATE CHANGEFEED FOR TABLE documents
  INTO 'kafka://broker:9092'
  WITH updated, resolved = '5s';

-- Postgres + Debezium
-- Configured at Debezium level; reads WAL.

-- TiDB
-- TiCDC reads TiKV change log and emits to Kafka.
```

Each change event:
```json
{
  "op": "u",                    // c=create, u=update, d=delete
  "before": {...},               // for update/delete
  "after": {...},                // for create/update
  "ts_ms": 1715000000000,
  "source": {"table": "documents", "id": "doc-123"}
}
```

### 3. Embedding worker

A consumer reads events from the stream:

```python
for event in stream:
    if event.op in ("c", "u"):
        text = event.after["content"]
        embedding = embed(text)
        vector_index.upsert(
            id=event.after["id"],
            vector=embedding,
            metadata={
                "table": "documents",
                "version": event.after["version"],
                "embedding_model": "text-embedding-3-small",
                "updated_at": event.ts_ms,
            },
        )
    elif event.op == "d":
        vector_index.delete(id=event.before["id"])
```

Each event triggers one embedding call + one vector-index write.

### 4. Idempotency and ordering

CDC events have ordering guarantees per partition (often per source primary key). The embedding worker must:
- **Idempotent writes**: same event produces same effect (use upsert by id).
- **Ordering preservation**: process events for a single id in order.
- **Retry safe**: failed events go to a dead-letter queue; replays don't double-write.

Without these, race conditions cause stale or wrong embeddings.

### 5. Re-embedding on model change

When the embedding model is swapped:

```text
1. Deploy new embedding model in worker (alongside old).
2. Worker writes new-model embeddings to a NEW vector index (or new dimension column).
3. A backfill job replays historical CDC events through the new worker.
4. Once backfill is complete: switch retrieval to new index.
5. Decommission old.
```

CDC makes this clean: the backfill is just a replay of past events through new logic; live updates continue without interruption.

### 6. Multi-tenant + multi-region

For multi-tenant systems:
- CDC topics per tenant or per shard.
- Embedding worker fleet partitioned by tenant.
- Vector index sharded by tenant.

For multi-region:
- CDC streams replicated cross-region (Kafka MirrorMaker, Pub/Sub regional).
- Per-region embedding workers, per-region indexes.
- Source-of-truth in one region; replicas elsewhere.

### 7. Latency budgets

Typical end-to-end latency for source change → vector-index visibility:

| Stage | Latency |
|---|---|
| CDC emission | < 100ms |
| Stream propagation | < 100ms |
| Embedding compute | 100ms–2s (depending on model) |
| Vector-index write | < 100ms |
| **End-to-end** | **< 3s** for most workloads |

For interactive RAG, this is fast enough that the user perceives freshness.

### 8. Backpressure and rate limits

Embedding APIs have rate limits. The pipeline must:
- Throttle to embedding-API capacity.
- Buffer in the stream when rate-limited.
- Alert when buffer grows.
- Self-host embedding for high-volume to avoid rate limits.

### 9. Observability

Per-pipeline metrics:
- **Lag**: time between source change and index update.
- **Throughput**: events/sec processed.
- **Failure rate**: events to DLQ.
- **Cost**: embedding API spend per pipeline.
- **Index freshness**: per-tenant max-lag dashboards.

Dashboards aggregate these; alerts on lag breaches.

### 10. The complete production pattern

```text
[application writes to source DB]
   ↓
[CDC emits to Kafka]
   ↓ (partitioned by entity_id; ordered per partition)
[embedding worker pool]
   ├── reads events
   ├── computes embeddings (batched for efficiency)
   ├── upserts to vector index
   ├── on failure → DLQ
   └── emits metrics
[vector index — always within seconds of source]
   ↓
[agent retrieves]
```

Operational primitives: dead-letter queue, retry policy, rate limiting, observability dashboard, cost tracking. All mature CDC patterns.

## Empirical anchors

- **End-to-end latency** of source change → index visibility: < 3s for most production setups.
- **Stale-RAG complaints** drop dramatically after CDC adoption — typically reduced 90%+.
- **Re-embedding cost** is a noticeable line item but manageable; batch + cache.
- **Migration to new embedding model** with CDC takes hours; without, days to weeks.
- **CDC infrastructure cost** is moderate; less than running batch re-index every hour.
- **Adoption** in 2026 production agent platforms is high; rare in stage-1/2 prototypes.

## Variants and counter-arguments addressed

- **"Periodic rebuild is simpler."** True for small datasets that change rarely. Breaks at scale or freshness sensitivity.
- **"CDC is over-engineering."** It's the standard data-engineering pattern; agent platforms benefit from established practice.
- **"Just use a vendor RAG service."** They often run CDC under the hood; you pay for the abstraction. For high control or compliance, build your own.
- **"Streaming infrastructure is heavy."** Kafka has gotten lighter; managed (MSK, Confluent, GCP Pub/Sub) reduces ops.
- **"What about graph + CDC?"** Same pattern; KG updater is another consumer of the same stream.

## Failure modes and limitations

1. **Backfill ordering issues.** Replaying past events while live updates flow can produce out-of-order writes. Use sequence numbers.
2. **Embedding-API rate-limit pile-up.** Spikes overwhelm capacity; buffer + throttle.
3. **Cost surprise.** Re-embedding millions of documents on model swap is expensive; budget.
4. **Schema drift.** Source schema changes; embedding worker breaks. Schema-versioning at the pipeline.
5. **Dead-letter accumulation.** No one watches the DLQ; failed events compound. Operator dashboard non-optional.
6. **Multi-tenant cross-talk.** Tenant A's high write rate slows Tenant B's freshness. Per-tenant partitioning.
7. **Vendor lock-in via Kafka.** Mitigatable; abstract behind a streaming API; keep portable.
8. **Cold start.** Bringing up a new index from scratch requires backfill of all historical data; can take hours.

## When to use, when not

**Use CDC + vector pipelines when** RAG freshness matters (interactive customer-facing agents), source data changes frequently, or operational complexity of periodic rebuild has become unmanageable.

**Skip CDC for** small static corpora that change rarely (annual policy docs, archival).

**Skip for** stage-1/2 prototypes; build CDC at stage 3 transition.

## Implications for harness engineering

- **CDC at stage-3 transition.** [118-genai-maturity-models](118-genai-maturity-models.md) — the right time to invest.
- **Streaming substrate is platform infrastructure.** Owned by platform team, used by all agents.
- **Embedding workers as a service.** One pool serves many tables; multi-tenant.
- **DLQ + alerting from day one.** Without, problems compound silently.
- **Index freshness SLAs.** Explicit; per agent or per data type.
- **Re-embedding playbook.** Documented; tested; quarterly drill.
- **Cost dashboards.** Per-pipeline; per-table; alert on spend drift.
- **Schema-version events.** When source schema changes, emit a marker event; downstream consumers handle it explicitly.
- **Multi-substrate coupling.** Same stream feeds vector + KG + search + audit. Loose coupling at the consumer level. See [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md), [134-semantic-indexing](134-semantic-indexing.md).

The one-sentence takeaway: **vector + CDC pipelines turn RAG freshness from "rebuild nightly" into a streaming property — source change → embedding update → index visibility within seconds — at the cost of streaming infrastructure that earns its place at any production scale.**

## See also

- [25-agentic-rag](25-agentic-rag.md) — RAG with self-critique.
- [120-rag-to-finetuning-spectrum](120-rag-to-finetuning-spectrum.md) — RAG positioning.
- [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md) — KG updates from same CDC stream.
- [130-distributed-sql-as-agent-memory](130-distributed-sql-as-agent-memory.md) — distributed SQL emits CDC natively.
- [131-temporal-bitemporal-tables](131-temporal-bitemporal-tables.md) — temporal data is CDC's natural source.
- [134-semantic-indexing](134-semantic-indexing.md), [135-trustworthy-generation](135-trustworthy-generation.md) — downstream patterns this enables.
- [125-system-level-production-patterns](125-system-level-production-patterns.md) — streaming infra is platform.
- [24-observability-tracing](24-observability-tracing.md) — per-pipeline metrics.
