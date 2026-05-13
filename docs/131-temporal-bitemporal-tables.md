# 131 — Temporal & Bitemporal Tables for Agent Context: Reasoning About "What Did We Know When"

**Sources.** Stewart & Huang, *Agentic AI Data Architectures*, Chapter 5 (Temporal patterns for agentic systems); plus the SQL temporal-tables literature (SQL:2011 system-versioned tables, Snodgrass's *Developing Time-Oriented Database Applications in SQL*); the bitemporal modelling tradition from financial data systems.

**One-line definition.** Temporal tables track *valid time* (when a fact was true in the world) and *transaction time* (when the system recorded it); the bitemporal pattern captures both axes simultaneously, letting agents reason precisely about *what we knew when* (transaction time) and *what was actually the case at that point* (valid time) — a distinction that is a footnote in most systems but a *load-bearing* primitive for agents that audit decisions, train on past data, or reconstruct historical context.

## Why this matters

Agent decisions get audited months later. The question is rarely "what is true now?" but "what did the agent know when it made decision X, and was that information correct at that time?" Without temporal data, this question is unanswerable; the agent's decision context cannot be reconstructed.

For regulated agents (financial, medical, legal), temporal correctness is not a feature — it is a regulatory requirement. The auditor asks: "the agent denied this loan on March 12; what was the credit score *as known on March 12*, and what did the database actually contain about this applicant *as recorded on March 12*?" Without bitemporal tables, the answer is "we don't know" — and the audit fails.

For training agents on historical data (the consolidation pathway in [100-contextual-memory-is-a-memo](100-contextual-memory-is-a-memo.md), the case bank in [107-memento-cbr-memory](107-memento-cbr-memory.md)), temporal correctness prevents *temporal leakage* — using future information to predict past events.

This chapter is the temporal data primitive that production agents need but rarely build correctly the first time.

## Problem it solves

Five concrete failures temporal/bitemporal modelling prevents:

1. **Audit reconstruction failure.** "What was the customer's address when we sent that letter?" — the address column has only the current value.
2. **Temporal leakage in training.** Training a fraud-detection model uses today's labels for yesterday's transactions; model overfits to information unavailable at the time.
3. **Retroactive correction confusion.** A bug in the credit-score pipeline corrected scores yesterday; today's decisions used the old scores; the audit trail conflates the two.
4. **Late-arriving data.** A transaction from March 1 arrives in the system on March 15; agents seeing data on March 10 didn't know about it; how do you query "what was the balance on March 5" — the answer depends on which axis you mean.
5. **Eventual consistency across sources.** Two source systems disagree on a fact; merging them requires temporal context to determine which version applied when.

Each is a class of bug that bitemporal modelling structurally prevents.

## Core idea in one paragraph

Every fact in a database has two time dimensions: **valid time** (the period during which the fact was true in the modelled domain) and **transaction time** (the period during which the system recorded the fact). Most databases collapse both into "now" — they store only the current value, with no history. Temporal tables track *one* axis: SQL:2011 system-versioned tables track transaction time; application-time period tables track valid time. **Bitemporal tables track both axes simultaneously**, which is the most general and the only model that fully answers "what did the system know at time T about what was the case at time T'?" For agents, the bitemporal model is essential for audit reconstruction, regulatory compliance, training-time leakage prevention, and reasoning about late-arriving or retrospectively-corrected data. The cost is schema complexity (two extra columns or period definitions) and query verbosity; the benefit is correctness on questions that are otherwise unanswerable.

## Mechanism (step by step)

### 1. The two time axes — a worked example

Consider a customer's address:

| customer_id | address | valid_from | valid_to | tx_from | tx_to |
|---|---|---|---|---|---|
| 42 | 123 Main St | 2020-01-01 | 2024-06-01 | 2020-01-01 | 2024-06-15 |
| 42 | 456 Oak Ave | 2024-06-01 | (open) | 2024-06-15 | (open) |

- **Valid time** (`valid_from` / `valid_to`): when the address was true in reality.
- **Transaction time** (`tx_from` / `tx_to`): when the system recorded that address.

The customer moved on June 1; the system was updated on June 15. Both timestamps are real and distinct.

Queries:
- *"Where did customer 42 live as of March 1, 2024?"* → 123 Main St (valid time query).
- *"What did our system say about customer 42's address on July 1, 2024?"* → 456 Oak Ave (transaction time query).
- *"What did our system say about customer 42's address as of June 10, 2024?"* → 123 Main St (transaction time query, before the update).

Three different correct answers depending on the axis.

### 2. SQL:2011 system-versioned tables

Modern SQL engines support transaction-time tracking natively:

```sql
CREATE TABLE customers (
  id UUID PRIMARY KEY,
  name TEXT,
  address TEXT,
  ...
  tx_from TIMESTAMPTZ GENERATED ALWAYS AS ROW START,
  tx_to TIMESTAMPTZ GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME (tx_from, tx_to)
) WITH SYSTEM VERSIONING;
```

Updates create new versions; old rows move to a history table. Queries can target any past system time.

### 3. Application-time period tables

For valid-time tracking:

```sql
CREATE TABLE policies (
  id UUID,
  premium NUMERIC,
  valid_from DATE,
  valid_to DATE,
  PERIOD FOR application_time (valid_from, valid_to),
  PRIMARY KEY (id, application_time WITHOUT OVERLAPS)
);
```

The `WITHOUT OVERLAPS` constraint ensures no two rows for the same id have overlapping valid times — the temporal integrity rule.

### 4. Bitemporal pattern

Combine both:

```sql
CREATE TABLE customer_addresses (
  customer_id UUID,
  address TEXT,
  valid_from DATE,
  valid_to DATE,
  tx_from TIMESTAMPTZ GENERATED ALWAYS AS ROW START,
  tx_to TIMESTAMPTZ GENERATED ALWAYS AS ROW END,
  PERIOD FOR application_time (valid_from, valid_to),
  PERIOD FOR SYSTEM_TIME (tx_from, tx_to),
  PRIMARY KEY (customer_id, application_time WITHOUT OVERLAPS)
) WITH SYSTEM VERSIONING;
```

Now every fact has both axes. Queries can combine them.

### 5. Bitemporal queries

```sql
-- What did we know on July 1 about the customer's address as of March 1?
SELECT address
FROM customer_addresses
FOR SYSTEM_TIME AS OF '2024-07-01'
WHERE customer_id = $1
  AND '2024-03-01' BETWEEN valid_from AND valid_to;
```

This is the canonical bitemporal query. Two `AS OF` clauses (one per axis); the answer is precisely what the system knew at that moment about that point in time.

### 6. Common patterns for agent context

**Pattern: agent-decision audit trail.**
- Every agent decision references the bitemporal data it used.
- Audit reconstruction: query data `AS OF` the decision time; verify the agent saw what it should have.

**Pattern: training-time correctness.**
- Training set built with bitemporal queries.
- For each training example, target value comes from valid time; features come from "what was known at training cutoff" (transaction time).
- Prevents temporal leakage.

**Pattern: late-arriving data handling.**
- Data arrives late; recorded with `tx_from = now()` and `valid_from = actual_date`.
- Agents that ran before `tx_from` saw the absence of the data; agents after see it.
- Audit and training both correct.

**Pattern: retroactive correction.**
- Bug fixed; corrected data inserted with new transaction time but original valid time.
- Old version preserved in transaction-time history.
- Decisions made under the old version remain reconstructable.

### 7. Engine support

| Engine | Temporal support |
|---|---|
| **PostgreSQL** | Limited native; `pg_history`, `temporal_tables` extension |
| **CockroachDB** | `AS OF SYSTEM TIME` (transaction time) |
| **YugabyteDB** | Inherited Postgres; temporal extension |
| **Oracle** | Workspace Manager; full bitemporal |
| **SQL Server** | System-versioned tables (SQL:2011) |
| **MariaDB** | Application-time period tables |
| **DB2** | Full bitemporal support, mature |

For most agent platforms in 2026: **CockroachDB or YugabyteDB with temporal extensions**, or **DB2 / SQL Server** in enterprise contexts.

### 8. Cost and storage

- **Storage**: each update creates a history row; storage scales with write volume × retention period.
- **Query cost**: queries that filter by time range scan history; index on time columns.
- **Operational**: rebuild indexes can be expensive on large history.
- **Retention**: define per-table retention; archive or delete beyond regulatory requirements.

### 9. Migration from non-temporal to temporal

Practical migration steps:
1. Add `tx_from`, `tx_to` columns; default current time.
2. Enable system versioning.
3. Backfill `valid_from`, `valid_to` from domain knowledge if needed.
4. Update writes to maintain temporal integrity.
5. Update reads to use `AS OF` where temporal correctness matters.

Migration is incremental but non-trivial. Start with the highest-stakes tables (audit, agent decisions, financial state).

## Empirical anchors

- **Bitemporal modelling** is standard in financial data infrastructure for two decades.
- **Audit reconstruction** with bitemporal data is a regulatory expectation in financial services.
- **Temporal leakage in ML** is a documented major source of overfitting.
- **Storage overhead** is typically 1.5–3× of non-temporal; manageable.
- **Query performance** is acceptable with proper indexing; up to 2× slower for historical queries.
- **Adoption** in agent platforms is low in 2025; growing fast in 2026 as audit requirements bite.

## Variants and counter-arguments addressed

- **"Just keep audit logs."** Logs are immutable but unstructured; bitemporal tables make audit queryable.
- **"Use Kafka and replay."** Replay is expensive; temporal queries are O(log N) with proper indexes.
- **"Snapshot-based versioning is enough."** Snapshots lose intra-snapshot history; bitemporal preserves all.
- **"Too complex for our team."** True for stage-1 / 2; mandatory for stage-3+ regulated work.
- **"Postgres doesn't support this well."** Use extensions or migrate to a more capable engine for high-stakes data.

## Failure modes and limitations

1. **Schema drift over time.** Adding columns breaks history; design schemas for evolution.
2. **Time-zone mistakes.** Mixing UTC and local times in temporal columns causes silent bugs.
3. **Retention policy gaps.** History grows unbounded; storage cost balloons.
4. **Index bloat.** Temporal indexes on large history tables get large; rebuild periodically.
5. **Query mistakes.** Forgetting `AS OF` when audit-querying; getting current values when historical were needed.
6. **Migration data quality.** Backfilled valid times can be wrong if domain knowledge is incomplete.
7. **Engine quirks.** SQL:2011 implementations differ; portable code is hard.
8. **Performance surprise on history queries.** Without indexes, full-table scans of history tables are slow.

## When to use, when not

**Use bitemporal modelling for** any data the agent's decisions depend on, in regulated contexts, or where training-time correctness matters.

**Use system-versioning (transaction time only) for** audit-only purposes where the world's truth is single-valued.

**Skip temporal modelling for** ephemeral data (current task state, scratchpads), low-stakes data, or pure cache layers.

**Migrate incrementally**: start with the most audit-critical tables; expand over months.

## Implications for harness engineering

- **Audit-critical tables get bitemporal modelling.** Decision records, agent state at decision time, source data the agent consumed.
- **Training datasets built with bitemporal queries.** Prevents temporal leakage.
- **Document the time semantics.** Every column with a time component is documented: valid time, transaction time, both, or neither.
- **`AS OF` in audit queries.** Standard pattern; reviewer-verifiable.
- **Retention policies per regulatory regime.** GDPR storage limitation, HIPAA 6-year retention. See [122-explainability-compliance](122-explainability-compliance.md).
- **Index temporal columns.** Without indexes, history queries are slow; with, they're acceptable.
- **Time zone discipline.** All temporal columns in UTC; presentation layer handles local time.
- **Late-arriving data handling**: deliberate insert pattern with valid_from set to actual event time.
- **Storage budget**: model the history-table growth before enabling at scale.

The one-sentence takeaway: **bitemporal tables track valid time and transaction time as orthogonal axes — the only data model that correctly answers "what did the agent know when, about what was actually the case then" — load-bearing for audit, compliance, and training-time correctness.**

## See also

- [09-memory-files](09-memory-files.md) — file-based memory; lacks temporal correctness.
- [122-explainability-compliance](122-explainability-compliance.md) — audit reconstruction depends on this primitive.
- [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md) — production patterns this enables.
- [127-loan-processing-multi-agent-case-study](127-loan-processing-multi-agent-case-study.md) — bitemporal data is implicit in regulated case study.
- [130-distributed-sql-as-agent-memory](130-distributed-sql-as-agent-memory.md) — the substrate that ships temporal natively.
- [115-evaluating-llm-systems](115-evaluating-llm-systems.md) — eval-data temporal correctness depends on this.
- [100-contextual-memory-is-a-memo](100-contextual-memory-is-a-memo.md) — consolidation pathway needs leakage-free training.
