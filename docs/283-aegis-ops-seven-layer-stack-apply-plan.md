# 283 — Aegis-Ops × Seven-Layer Stack Apply Plan 2026

**Anchors.** Aegis-Ops — operations / SRE / monitoring agent ([projects/aegis-ops](../projects/aegis-ops/)). Existing per-project plan: [221-aegis-ops-multi-hop-collaborative-apply-plan](221-aegis-ops-multi-hop-collaborative-apply-plan.md). Per-layer source: [225](225-agent-era-scaling-synthesis.md), [258](258-agent-protocol-stack-synthesis-2026.md), [263](263-production-agent-runtime-synthesis-2026.md), [268](268-agent-operations-synthesis-2026.md), [273](273-agent-security-synthesis-2026.md), [267](267-agent-sre.md), [122-explainability-compliance](122-explainability-compliance.md).

**One-line definition.** A **per-layer apply plan** for Aegis-Ops — the **ops / SRE / monitoring agent** that watches production systems and runs incident response — emphasizing the **load-bearing operations + security layers** ([268](268-agent-operations-synthesis-2026.md), [273](273-agent-security-synthesis-2026.md)) since aegis-ops *operates* other systems and is itself an SRE-grade production deployment, the **highest bright-line discipline** ([14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md)) for production-modifying actions, and **multi-channel alerting + runbook execution** UX ([277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md)) — staged across four 90-day phases that build production-grade ops automation while maintaining strict human-in-the-loop for consequential actions.

## Aegis-Ops shape

Aegis-Ops is an **agent that operates other systems**: monitors production services, detects incidents, executes runbooks, manages capacity, schedules maintenance, generates postmortems. It is itself a production-grade deployment, so its own operations stack must be rigorous. The seven-layer stack maps with **operator role overlay**: high-stakes actions, mandatory bright-line gates, multi-channel alerting, audit-grade observability of every action, and SOC 2 / ISO 42001 compliance because aegis-ops touches production infrastructure.

## Per-layer plan

### L1 Foundation — operator-grade

`harness_core/foundation/` baseline. **Strictest Permission Bridge** discipline — every prod-touching action gated; default-deny on unknown action kinds. Daemon as always-on monitoring substrate. Bright-line escalation as primary HITL pattern.

### L2 Capability — alert-aware + runbook-aware

[225-agent-era-scaling-synthesis](225-agent-era-scaling-synthesis.md). Aegis-Ops's distinctive capabilities:

- **Alert correlation**: incoming alerts → cluster by root cause → prioritize.
- **Runbook execution**: per-incident runbook ([267-agent-sre](267-agent-sre.md)) executed step-by-step with bright-line gates between steps.
- **Capacity forecasting**: tokens, compute, storage; aligned with [278-agent-unit-economics-2026](278-agent-unit-economics-2026.md).
- **Postmortem generation**: trace replay → MAST classification → action items.

**Phase:** P1.

### L3 Protocol — full stack with Ops integrations

[258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md).

- **MCP** ([256](256-mcp-2025-2026-evolution.md)): kubectl MCP, terraform MCP, AWS / GCP / Azure MCPs, Datadog / Grafana / Phoenix MCPs, GitHub Actions MCP, PagerDuty / Opsgenie MCP. Read-only MCP servers default; write-MCP gated by bright-line.
- **A2A** ([254](254-a2a-protocol-deep-dive.md)): aegis-ops exposes A2A endpoints for incident-handoff to other agents (e.g., security-incident → cipher-sec).
- **AGNTCY** ([255](255-agntcy-oasf-acp-deep-dive.md)): published OASF.
- **Tailscale + NATS** ([253](253-tailscale-nats-mesh-for-distributed-agents.md)): aegis-ops on tailnet; NATS for alert pub-sub.
- **SKILL.md + marketplace**: argus-served runbook-skills with audited trust tier.
- **Routines + Agent Teams** ([252](252-routines-pattern-for-self-hosted-agents.md), [250](250-anthropic-agent-teams.md)): Routines for cron health checks; Agent Teams for parallel-investigation incidents (multi-hypothesis debugging).

**Phase:** P1-P2.

### L4 Runtime — LangGraph for state-machine runbook execution

[259-langgraph-deep-dive](259-langgraph-deep-dive.md). Runbook execution is a state machine; LangGraph's checkpointer + `interrupt()` for HITL gates fits perfectly. Postgres checkpointer mandatory for audit.

### L5 Security — operator-role security

[273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md). Aegis-Ops touches production infrastructure; security is critical:

- **Prompt-injection defense.** Spotlight on alert content (especially user-reported / external-tool alerts). Classifier ensemble.
- **Supply chain.** Vendored runbooks primary; marketplace runbooks via argus with `audited` trust tier minimum.
- **Isolation.** Per-incident worktree; container for runbook execution; **micro-VM for prod-modifying actions**; network egress allowlist (only target services).
- **Bright-line gates.** **Highest discipline** — every prod-modifying action requires explicit user approval; bright-line action kinds: `KUBECTL_APPLY`, `TERRAFORM_APPLY`, `DB_MIGRATION`, `CONFIG_PROD_CHANGE`, `RESTART_SERVICE`, `SCALE_RESOURCE`, `DELETE_LOG`. Two-person review for highest-impact actions.
- **Cross-channel verifier.** Mandatory for runbook plan + post-execution verification.

### L6 Operations — recursive (aegis-ops operates aegis-ops)

[268-agent-operations-synthesis-2026](268-agent-operations-synthesis-2026.md). Aegis-Ops operates other systems but also itself; its own operations stack must be strong:

- **Observability.** Full OTel + HIR; production-eval feedback loop; alerts on aegis-ops own SLOs.
- **Evaluation.** Runbook-correctness eval suite; alert-classification accuracy.
- **Durability.** LangGraph checkpointer; idempotency on every kubectl / terraform / API call; saga + compensation for multi-step prod changes.
- **SRE.** Recursive — aegis-ops is on-call for itself + the systems it watches; runbooks for its own failure modes; rollback for its own state.

**Phase:** P2-P3.

### L7 Compliance — SOC 2 + audit-grade

[272-agent-compliance-and-audit](272-agent-compliance-and-audit.md). Aegis-Ops touches infrastructure so audit is critical:

- **SOC 2 Type II** for any commercial deployment.
- **EU AI Act** — high-risk if managing critical infrastructure (Annex III); risk-tier classification per environment.
- **Audit trail** in tamper-evident storage; every prod action logged with who-approved-what.
- **GDPR** as appropriate (handling logs may include personal data).
- **Compliance dashboard** showing prod-action approval rates, two-person review compliance.

**Phase:** P3-P4.

## Phased rollout

| Phase | Scope | Duration |
|---|---|---|
| **P1** | L1 Foundation + L2 (alert correlation + runbook executor) + read-only MCP integrations | 90 days |
| **P2** | Write-MCP integrations (with bright-line) + L4 Runtime (LangGraph) + L5 Security | 90 days |
| **P3** | L6 Operations (recursive on aegis-ops itself) + Agent Teams for multi-hypothesis | 90 days |
| **P4** | L7 Compliance (SOC 2 prep / audit) + capacity forecasting + production hardening | 90 days |

## Aegis-Ops-specific patterns

- **Recursive operator**: aegis-ops operates the systems aegis-ops itself runs on.
- **Runbook-as-code**: runbooks are version-controlled, eval-tested, deployed via the same pipeline as application code.
- **Two-person review** for highest-impact actions (cross-checked between aegis-ops and a human approver).
- **Multi-channel alerting**: PagerDuty / Slack / SMS / email per severity tier.
- **Postmortem generation** from trace replay + MAST classification.
- **Capacity forecasting** for token spend, compute, storage.
- **Game-day routines** ([49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md), [53-chaos-engineering-next-era](53-chaos-engineering-next-era.md)) — scheduled chaos exercises run by aegis-ops itself.

## Cost economics

[278-agent-unit-economics-2026](278-agent-unit-economics-2026.md). Aegis-Ops cost varies with scale:

- Always-on daemon: ~$20-100/mo VPS or container in target environment.
- LLM cost: variable by alert volume; ~$50-500/mo for production deployment.
- Runbook execution: bursty; capped per-incident.
- ROI: prevention of one $10K incident often pays for a year of aegis-ops.

## Cross-project dependencies

- **Argus** ([282](282-argus-seven-layer-stack-apply-plan.md)) for runbook-skills marketplace.
- **Cipher-Sec** ([284](284-cipher-sec-seven-layer-stack-apply-plan.md)) for security-incident handoff.
- **Lyra patterns** for memory primitives.

## One-line takeaway

**Aegis-Ops is the operator-role agent — watching production systems, executing runbooks, managing capacity, generating postmortems — across four 90-day phases adopting the seven-layer stack with operator-overlay (highest bright-line discipline, two-person review for prod actions, LangGraph state-machine runbook execution with `interrupt()` HITL gates, micro-VM isolation for prod-modifying steps, SOC 2 + EU-AI-Act-high-risk compliance for critical infrastructure deployments, and recursive operations of itself); the agent that runs other agents must be the most disciplined.**
