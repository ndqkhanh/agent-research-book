# 286 — Atlas-Research × Seven-Layer Stack Apply Plan 2026

**Anchors.** Atlas-Research — research-literature retrieval + ReWOO planning + citation verification ([projects/atlas-research](../projects/atlas-research/)). Companion: [218-atlas-research-multi-hop-collaborative-apply-plan](218-atlas-research-multi-hop-collaborative-apply-plan.md), [17-rewoo](17-rewoo.md). Per-layer source: [225](225-agent-era-scaling-synthesis.md), [258](258-agent-protocol-stack-synthesis-2026.md), [263](263-production-agent-runtime-synthesis-2026.md), [268](268-agent-operations-synthesis-2026.md), [273](273-agent-security-synthesis-2026.md).

**One-line definition.** A **per-layer apply plan** for Atlas-Research — the **research-literature retrieval and ReWOO-planned synthesis agent** — emphasizing the **multi-hop retrieval pipeline** (citation graph traversal, cross-source triangulation), the **citation-verifier as load-bearing security control** ([223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md)), and the **routine-driven scheduled-research pattern** ([252](252-routines-pattern-for-self-hosted-agents.md)) — staged across four 90-day phases targeting integration as a service for Polaris, Mentat-Learn, and external research consumers.

## Per-layer plan

### L1 Foundation
Standard `harness_core/foundation/`. Permission Bridge gates external API calls (Semantic Scholar, arXiv, Google Scholar). Daemon for scheduled lit-watch.

### L2 Capability
[225](225-agent-era-scaling-synthesis.md). **Pretraining**: domain-tuned reasoning model. **TTC**: thinking model for multi-hop synthesis ([199-multi-hop-reasoning-techniques-arc](199-multi-hop-reasoning-techniques-arc.md), [200-graph-grounded-multi-hop-retrieval](200-graph-grounded-multi-hop-retrieval.md)). **Trajectory**: long-horizon for deep dives. **Multi-agent**: ReWOO planner + retriever + synthesizer + verifier as Agent Team. **Verifier**: citation verifier mandatory; cross-channel adversarial pair on synthesis output.

### L3 Protocol
- **MCP**: Semantic Scholar MCP, arXiv MCP, S2 API MCP, citation-graph MCP, OpenAlex MCP.
- **A2A**: Atlas exposes A2A endpoints `lit-search`, `multi-hop-synthesis`, `citation-verify`. Polaris consumes.
- **AGNTCY**: published OASF; trust tier `audited`.
- **Tailscale + NATS**: not load-bearing (mostly read-only operations).
- **SKILL.md + marketplace**: lit-search / synthesis skills via argus marketplace.
- **Routines**: scheduled "weekly papers in topic X" routines.

### L4 Runtime
LangGraph for the ReWOO state machine (plan → retrieve → reason → verify → synthesize). Postgres checkpointer for reproducible research traces.

### L5 Security
- **Prompt injection**: critical — atlas reads attacker-controllable web pages and PDFs. Spotlight + classifier + instruction hierarchy.
- **Supply chain**: vendored skills primary; min-trust-tier `audited` for marketplace.
- **Isolation**: container per research session; egress allowlist (academic APIs only).
- **Bright-line**: `PUBLISH_REPORT`, `EXTERNAL_API_RATE_LIMIT_OVERRIDE`.

### L6 Operations
- **Observability**: full OTel + HIR; per-citation provenance.
- **Eval**: citation-correctness eval suite; multi-hop-faithfulness eval against gold ([198-multi-hop-qa-datasets-canon](198-multi-hop-qa-datasets-canon.md), [201-compositionality-gap-canon](201-compositionality-gap-canon.md)).
- **Durability**: LangGraph checkpointer; idempotency on API calls (Semantic Scholar rate-limited).
- **SRE**: SLO on citation-correctness ≥ 95%, P99 synthesis latency.

### L7 Compliance
- **EU AI Act**: limited risk (research assistance) unless used in high-risk research domains.
- **GDPR**: minimal — no personal data unless user-supplied.
- **Provenance audit trail** for citation chain-of-custody.

## Phased rollout

| Phase | Scope | Duration |
|---|---|---|
| **P1** | L1-L2 + MCP integrations + LangGraph skeleton | 90 days |
| **P2** | L3 (A2A endpoints) + L5 Security + citation verifier | 90 days |
| **P3** | L6 Operations + multi-hop eval suite | 90 days |
| **P4** | L7 Compliance + production hardening + Polaris integration | 90 days |

## Cross-project dependencies

- **Polaris** ([279](279-polaris-seven-layer-stack-apply-plan.md)) — primary consumer.
- **Argus** ([282](282-argus-seven-layer-stack-apply-plan.md)) — skill marketplace.
- **Mentat-Learn** ([281](281-mentat-learn-seven-layer-stack-apply-plan.md)) — secondary consumer for user-facing research.

## One-line takeaway

**Atlas-Research adopts the seven-layer stack as the research-lit-retrieval-and-ReWOO-synthesis service — exposing A2A endpoints (lit-search, multi-hop-synthesis, citation-verify) consumed by Polaris and Mentat-Learn, with citation-verifier as load-bearing security control, LangGraph-backed durable ReWOO state machines, and routine-driven scheduled-research as the daemon-mode contribution; ~$15K-50K hardware/infra and 12 months to production.**
