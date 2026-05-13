# 138 — Text-to-SQL Agents: Schema Awareness, Query Repair, Safety Wrapping

**Sources.** Kar, *Building Multimodal Generative AI*, Chapters 14–15 (Text-to-SQL Systems, Agentic Text-to-SQL); the academic text-to-SQL literature (Spider benchmark, BIRD-bench, GPT-4 with schema linking); plus production text-to-SQL systems (Snowflake Cortex Analyst, Databricks AI/BI, Vanna AI, dbt Copilot).

**One-line definition.** A text-to-SQL agent translates natural-language questions into executable SQL queries against a known database schema, with a complete production stack including *schema-aware retrieval* (the agent only knows tables it's been told about), *query generation* (LLM + constraints), *query repair* (catch and fix errors before execution), and *safety wrapping* (no destructive ops, row-level security, query timeouts) — and it is one of the most-deployed agentic patterns in 2026 enterprise contexts because the value (self-service analytics) and the risk (a bad query can leak data or DROP TABLE) are both high.

## Why this matters

Text-to-SQL is the agent pattern most likely to be in production at any given enterprise: data-analyst self-service, customer-facing query interfaces, internal admin tools. It is also the agent pattern with the sharpest gap between *demo* (works on a clean schema with a smart user) and *production* (handles a 500-table data warehouse with naive users, without leaking sensitive columns or generating queries that scan the whole warehouse).

For agent builders, the discipline is dense: schema-aware retrieval to handle large schemas, semantic-search over tables and columns, constrained generation to enforce SQL grammar, validation before execution, safety wrappers, observability of query patterns, and feedback loops that learn from successes and failures.

This chapter is the production playbook for text-to-SQL agents that survive contact with real enterprise data.

## Problem it solves

Six concrete text-to-SQL failures:

1. **Schema hallucination.** Agent invents tables and columns that don't exist; query fails.
2. **Wrong joins.** Agent joins unrelated tables; returns garbage results.
3. **Destructive operations.** User asks "remove duplicates"; agent emits `DELETE`; data lost.
4. **Privacy leak.** Query returns columns the user shouldn't see (PII, salary, internal metadata).
5. **Runaway queries.** Agent emits unindexed joins on billions of rows; warehouse stalls.
6. **Silent semantic errors.** Query runs and returns *wrong* answer; user trusts the answer.

Each is a structural risk that production text-to-SQL agents address through a multi-layer architecture.

## Core idea in one paragraph

A production text-to-SQL agent has six layers: **schema introspection** (reads database schema, extracts table descriptions, column types, foreign keys, sample values, statistics); **semantic schema retrieval** (per query, retrieves the relevant subset of tables/columns from the schema repository — the schema is too large to fit in the prompt); **query generation** (LLM with constrained decoding to ensure syntactically valid SQL, schema-grounded prompt with relevant tables only); **query repair** (parses the query, validates against schema, on errors LLM fixes); **safety wrapping** (whitelist of allowed operations — typically only `SELECT`; row-level security; query timeouts; explain-plan checks); **execution + result presentation** (run the query against a sandboxed read-replica; format results; cite source tables and columns to the user). Layered, this turns text-to-SQL from a Friday demo into a stage-3 production tool. The safety layer is the load-bearing one for enterprise deployment; without it, no team will ship.

## Mechanism (step by step)

### 1. Schema introspection

Read the database catalog:
- All tables: name, schema, comments.
- All columns: name, type, nullability, comments, foreign keys.
- Statistics: row count, distinct values, sample values.
- Constraints, indexes.

Stored in a schema repository — often a vector index over table+column descriptions for retrieval.

For large data warehouses (500+ tables), this introspection is non-trivial; do it once at deploy and refresh on schema-change events.

### 2. Semantic schema retrieval

Per query, retrieve the relevant subset:

```text
[query: "What was the revenue from European customers last quarter?"]
   ↓
[embed query]
   ↓
[search schema repository: tables, columns]
   ↓
top relevant: customers (region), orders (amount, date), regions (...)
   ↓
[present only these tables to the LLM in the prompt]
```

Why retrieve and not include all? A 500-table schema is too large for the prompt; it also dilutes the LLM's attention. Top 5–10 tables typically suffice.

### 3. Query generation

```text
Prompt:
  Schema:
    customers (id, name, region_id, ...)
    regions (id, name)
    orders (customer_id, amount, order_date)

  Question: What was the revenue from European customers last quarter?

  Output ONLY a SELECT query. No DROP, DELETE, UPDATE, INSERT.

Output:
  SELECT SUM(o.amount) AS revenue
  FROM orders o
  JOIN customers c ON c.id = o.customer_id
  JOIN regions r ON r.id = c.region_id
  WHERE r.name = 'Europe'
    AND o.order_date >= '2024-01-01'
    AND o.order_date <  '2024-04-01';
```

Constrained decoding ([112-constrained-decoding](112-constrained-decoding.md)) enforces SQL grammar; the prompt enforces SELECT-only.

### 4. Query repair

Before execution, parse and validate:

```text
[parse query]
   ↓ (if parse fails)
   [LLM fixes: original query + parse error → corrected query]
   ↓
[validate against schema]
   ↓ (if column doesn't exist or join invalid)
   [LLM fixes: original query + validation error → corrected query]
   ↓
[explain plan]
   ↓ (if scan estimate exceeds threshold)
   [reject or escalate to user]
```

Repair iterates; cap iterations to avoid loops. Modern LLMs typically fix on first attempt for simple errors.

### 5. Safety wrapping

The execution layer enforces:
- **Whitelist operations**: only SELECT (and maybe specific stored-procedure calls).
- **Row-level security**: query rewritten to include user's access constraints (e.g. `tenant_id = $current_tenant`).
- **Column masking**: PII columns automatically excluded or redacted.
- **Query timeout**: kill queries that run too long.
- **Resource limits**: max rows scanned, max bytes returned.
- **Read-only execution**: queries run against a read replica or via a read-only role.

These are non-optional. Skipping any is a path to incident.

### 6. Execution + result presentation

Run the query; format the results:
- **Tabular results** with column headers.
- **Source citations**: which tables and columns contributed.
- **The query itself**, surfaced for transparency (advanced users can audit).
- **Caveats**: "Showing first 100 rows" / "Query estimated to scan 50 GB; truncated."

Provenance: every result row carries metadata about its source.

### 7. Feedback loop

User accepts the answer? Mark as positive. Asks for re-query? Negative signal. Edits the query? Used as training data for retrieval improvements.

Same pattern as [107-memento-cbr-memory](107-memento-cbr-memory.md): build a case bank of (question, schema_subset, correct_query, user_feedback); use for retrieval-augmented generation in future queries.

### 8. Multi-database support

Production deployments often span multiple databases. Each has:
- Its own schema repository.
- Its own dialect (Postgres vs Snowflake vs BigQuery vs Trino).
- Its own access controls.

The agent routes queries to the right database based on the schema retrieval; SQL is dialect-aware.

### 9. Iterative refinement (agentic text-to-SQL)

For complex questions, single-shot text-to-SQL fails. The agent decomposes:

```text
[complex question]
   ↓
[planner: decompose into sub-questions]
   ↓
[per sub-question]
   ├── retrieve schema
   ├── generate sub-query
   ├── execute
   ├── observe result
   ↓
[synthesise final answer from sub-results]
```

This is [16-plan-and-solve](16-plan-and-solve.md) applied to text-to-SQL.

### 10. The complete production pattern

```text
[user question]
   ↓
[semantic schema retrieval — top tables/columns]
   ↓
[query generation — constrained decoding]
   ↓
[query repair — parse + validate + plan-check]
   ↓
[safety wrapping — RLS, column masking, timeouts]
   ↓
[execution against read-replica]
   ↓
[result formatting + citations + caveats]
   ↓
[feedback collection]
```

Layered. Each layer's failure mode contained.

## Empirical anchors

- **Spider benchmark**: top systems (GPT-4 + retrieval + repair) ~85% execution accuracy.
- **BIRD-bench** (more realistic): ~65% on best systems.
- **Production accuracy** depends on schema clarity; well-named schemas → 80–90% useful, poorly-named → 50–70%.
- **Safety incidents** are the #1 deployment risk; multi-layer defence is non-optional.
- **Adoption**: high in enterprise self-service analytics; growing in customer-facing data interfaces.
- **Cost**: tens of cents per query at most; unrealistic ones can blow up.

## Variants and counter-arguments addressed

- **"GPT-4 is good enough at SQL."** It generates valid SQL most of the time; the production layer is around it (retrieval, repair, safety).
- **"Just give the LLM the whole schema."** Schemas are too large; retrieval is essential.
- **"Don't allow DELETE/UPDATE/INSERT."** Correct; many production systems further restrict.
- **"Use a vendor service (Snowflake Cortex, Databricks AI)."** Increasingly viable; the principles still apply.
- **"Text-to-SQL is solved."** Demo level yes; enterprise production no.

## Failure modes and limitations

1. **Ambiguous questions.** "Top customers" — by what? Revenue, count, recency? Need clarification or assumptions.
2. **Schema drift.** Tables added/removed; agent's repository stale.
3. **Domain knowledge gaps.** "Active customers" has a domain definition the agent doesn't know.
4. **Join hallucination.** Agent joins on similar-named columns that aren't actually keys.
5. **Aggregation errors.** Avg of avg; double-counted joins; subtle.
6. **Privacy leakage.** Wrong row-level security; user sees data they shouldn't.
7. **Cost runaway.** Unindexed query on petabyte tables.
8. **Wrong-but-runs.** Query executes successfully and returns plausible but wrong results.

## When to use, when not

**Use text-to-SQL for** self-service analytics, customer-facing query interfaces (where access controlled), internal admin tools.

**Skip for** systems where users should be writing SQL directly (data engineers).

**Wrap aggressively** for any external-facing deployment.

**Pair with verifier loops** ([11-verifier-evaluator-loops](11-verifier-evaluator-loops.md)) for high-stakes or numerical-heavy queries.

## Implications for harness engineering

- **Schema introspection as an asset.** Versioned, refreshed on schema events.
- **Semantic schema retrieval.** Vector index over table/column descriptions.
- **Constrained decoding.** [112-constrained-decoding](112-constrained-decoding.md). SQL grammar enforcement.
- **Safety wrapping mandatory.** RLS, column masking, query timeouts, read-only execution.
- **Repair loop.** Parse + validate + plan-check; capped iterations.
- **Citations to user.** Source tables and columns surfaced.
- **Eval set.** Question / correct-query pairs; per-database; refresh.
- **Feedback loop into retrieval.** Successes / failures into the case bank.
- **Multi-tenant isolation** if the agent serves multiple tenants.
- **Cost controls.** Query plan cost estimation before execution.

The one-sentence takeaway: **text-to-SQL is a production-ready agent pattern when the architecture wraps the LLM with semantic schema retrieval, constrained decoding, query repair, and aggressive safety controls — without the wrapping it's a Friday demo, with it a stage-3 capability.**

## See also

- [25-agentic-rag](25-agentic-rag.md), [134-semantic-indexing](134-semantic-indexing.md) — the retrieval foundation.
- [112-constrained-decoding](112-constrained-decoding.md) — SQL grammar enforcement.
- [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) — verifier patterns for query correctness.
- [16-plan-and-solve](16-plan-and-solve.md) — decomposition for complex queries.
- [107-memento-cbr-memory](107-memento-cbr-memory.md) — feedback-driven case bank.
- [122-explainability-compliance](122-explainability-compliance.md) — privacy and audit for query results.
- [123-robustness-fault-tolerance](123-robustness-fault-tolerance.md) — query timeout, fallback patterns.
- [136-multimodal-rag](136-multimodal-rag.md), [139-ocr-document-agents](139-ocr-document-agents.md) — adjacent specialised agents.
