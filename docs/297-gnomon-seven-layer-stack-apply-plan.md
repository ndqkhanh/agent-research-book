# 297 — Gnomon × Seven-Layer Stack Apply Plan 2026

**Anchors.** Gnomon — observability / tracing / metrics agent ([projects/gnomon](../projects/gnomon/)). Companion: [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), [24-observability-tracing](24-observability-tracing.md), [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md).

**One-line definition.** A **per-layer apply plan** for Gnomon — the **observability + tracing + metrics agent** that aggregates OTel-shape spans across the in-tree ecosystem — emphasizing the **provider-role overlay** (every other project sends Gnomon their HIR-shape events for centralized analysis), **OTel collector + Phoenix-shape consumer** as the runtime, **trace-correlation across runtimes** ([264](264-agent-observability-stack-2026.md)), and the **eval-link feedback loop** to evaluation systems ([265](265-agent-evaluation-2026.md)) — staged across four 90-day phases.

## Per-layer plan

### L1 Foundation
Standard. Permission Bridge gates writes to centralized observability store. Daemon as always-on collector.

### L2 Capability
**OTel-shape span ingestion**, **HIR-shape typed events**, **per-runtime correlation via traceparent**, **anomaly detection** as core capabilities.

### L3 Protocol
- **MCP**: query-spans MCP, query-metrics MCP, alert-config MCP.
- **A2A**: gnomon exposes `query_traces`, `correlate_runtime_calls`, `compute_slo` capabilities.
- **AGNTCY**: published OASF.
- **Tailscale + NATS**: NATS as transport for span ingestion (high-volume).
- **Routines**: scheduled SLO computation, drift detection, capacity forecasting.

### L4 Runtime
LangGraph for analytical pipelines (anomaly detection, root-cause analysis). High-volume span ingestion via Phoenix-shape consumer + columnar storage (Parquet / ClickHouse).

### L5 Security
- **Trace data privacy**: PII redaction at ingestion ([233](233-memory-scaling-for-agents.md)).
- **Tamper-evident audit log** for compliance-relevant events (Sigstore transparency log).
- **Isolation**: multi-tenant per-project isolation.

### L6 Operations
- **Recursive observability**: gnomon observes itself.
- **Eval**: trace-correctness eval; correlation-accuracy eval.
- **Durability**: durable streams (NATS JetStream) for span ingestion.
- **SRE**: 99.9% availability SLO; ingestion-rate SLO; query latency SLO.

### L7 Compliance
**SOC 2 Type II** mandatory (gnomon is shared infrastructure). **GDPR** for PII in traces. **Tamper-evident audit trail** for EU AI Act Art. 12 logging requirements.

## Phased rollout

| Phase | Scope | Duration |
|---|---|---|
| **P1** | L1-L2 + OTel collector + Phoenix-shape consumer | 90 days |
| **P2** | L3 (A2A query) + cross-runtime correlation | 90 days |
| **P3** | L6 Operations (recursive) + per-project integration | 90 days |
| **P4** | L7 Compliance (SOC 2 + tamper-evident) | 90 days |

## One-line takeaway

**Gnomon is the **observability provider** in the in-tree ecosystem — OTel-shape span ingestion + HIR-shape typed events + cross-runtime correlation + tamper-evident audit log for compliance — across four 90-day phases adopting the seven-layer stack with provider-role overlay (high-availability, multi-tenant isolation, recursive observability of itself), serving as the centralized observability surface that satisfies both per-project SLO measurement and EU AI Act Art. 12 logging requirements.**
