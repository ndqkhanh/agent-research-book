# 279 — Polaris × Seven-Layer Stack Apply Plan 2026

**Anchors.** Polaris — autonomous polymath research agent ([projects/polaris](../projects/polaris/)). Extends [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md) and [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md) with the seven-layer stack from [225](225-agent-era-scaling-synthesis.md), [258](258-agent-protocol-stack-synthesis-2026.md), [263](263-production-agent-runtime-synthesis-2026.md), [268](268-agent-operations-synthesis-2026.md), [273](273-agent-security-synthesis-2026.md). Per-layer source files: [254](254-a2a-protocol-deep-dive.md)–[273](273-agent-security-synthesis-2026.md).

**One-line definition.** A **per-layer apply plan** for Polaris adopting the seven-layer stack — the project already has strong **L1 Foundation** (Permission Bridge, Daemon, Bright-line Gates, Cost Router as canonical blocks ([projects/polaris/docs/blocks](../projects/polaris/docs/blocks/index.md))) and partial **L2 Capability** + **L3 Protocol** + **L6 Operations** (HIR observability, ReasoningBank, Memory Tiers); the apply plan stages adoption of **L4 Runtime selection** (LangGraph ([259](259-langgraph-deep-dive.md)) for production-tier durability), **L5 Security** ([269](269-prompt-injection-2026.md)–[273](273-agent-security-synthesis-2026.md)), **completed L6 Operations** ([264](264-agent-observability-stack-2026.md)–[268](268-agent-operations-synthesis-2026.md)), and **L7 Compliance** ([272](272-agent-compliance-and-audit.md)) for high-risk research workflows — across five 90-day phases that build production-grade Polaris from current Phase-0 / Phase-1 starting point.

## Polaris-specific shape

Polaris is a **long-running autonomous research agent** running as a daemon, executing programs over weeks-to-months, with multi-domain shells (ML / bio / math / physics / social / engineering), memory-driven continuous learning, and bright-line escalation as the primary HITL pattern. The seven-layer stack maps unusually cleanly because polaris's existing `polaris-core` blocks already encode many of the primitives — agent loop, permissions, hooks, ReasoningBank, observability, daemon, bright-line gates. The work is **completing the layers** rather than starting from scratch.

## Per-layer plan

### L1 Foundation — already strong

Polaris ships canonical implementations: [01-agent-loop](../projects/polaris/docs/blocks/01-agent-loop.md), [07-permission-bridge](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [08-hooks-and-claim-gate](../projects/polaris/docs/blocks/08-hooks-and-claim-gate.md), [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md). **Action:** continue Phase-1 development per existing roadmap.

### L2 Capability — extend with axes 216–237

[216-pretraining-scaling-laws-foundation](216-pretraining-scaling-laws-foundation.md) ÷ [237-agent-trajectory-scaling](237-agent-trajectory-scaling.md). **Existing:** ReasoningBank ([05](../projects/polaris/docs/blocks/05-reasoning-bank.md)), agent loop, subagent delegation. **Add:** test-time compute router (per [217](217-test-time-compute-scaling.md)), memory-tiered slope-multiplier (already partial via [05](../projects/polaris/docs/blocks/05-reasoning-bank.md)), per-difficulty agent-trajectory budgets (per [237](237-agent-trajectory-scaling.md)). **Phase:** P2 (foundational; before security / compliance work).

### L3 Protocol — adopt the six-layer protocol stack

[258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md). **MCP** ([256](256-mcp-2025-2026-evolution.md)) — already in [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md); upgrade to Streamable HTTP. **A2A** ([254](254-a2a-protocol-deep-dive.md)) — new; expose polaris-research / polaris-verifier / polaris-synthesizer as A2A endpoints with Signed Agent Cards. **AGNTCY OASF** ([255](255-agntcy-oasf-acp-deep-dive.md)) — publish capability schemas; participate in discovery. **Tailscale + NATS** ([253](253-tailscale-nats-mesh-for-distributed-agents.md)) — distributed deployments for multi-host research programs. **SKILL.md + marketplace** ([257](257-agent-skill-marketplace-landscape.md)) — research-skill marketplace integration; [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md) consumes. **Routines + Agent Teams** ([250-anthropic-agent-teams](250-anthropic-agent-teams.md), [252](252-routines-pattern-for-self-hosted-agents.md)) — Routines via existing daemon; Agent Teams for parallel-review research patterns. **Phase:** P2-P3.

### L4 Runtime — LangGraph as production-tier upgrade

[259-langgraph-deep-dive](259-langgraph-deep-dive.md). Polaris's current architecture is custom; [263-production-agent-runtime-synthesis-2026](263-production-agent-runtime-synthesis-2026.md) recommends **LangGraph** for the state-machine workflow shape (research → critique → revise → publish). **Action:** wrap existing polaris-core blocks as LangGraph nodes with TypedDict state; Postgres checkpointer for durability; preserve bright-line semantics in `interrupt()`. **Phase:** P3 (after L2 capability + L3 protocol).

### L5 Security — five-layer defense

[269](269-prompt-injection-2026.md), [270](270-agent-supply-chain-security.md), [271](271-agent-isolation-patterns.md), [273](273-agent-security-synthesis-2026.md).

- **Prompt-injection defense.** Anthropic Constitutional Classifiers + Microsoft Prompt Shield ensemble at retrieval boundaries. Spotlight prompting on retrieved papers / web pages / memory reads. Instruction hierarchy in research-agent system prompts. **High priority** because polaris reads attacker-controllable web content extensively.
- **Supply chain.** SBOM enumerating base model + MCP servers + skills + A2A peers + memory partitions. Min-trust-tier policy on installed skills. Vendored fallback for critical research skills. Revocation routine.
- **Isolation.** Worktree per research program ([18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md)). Container for code-executing research agents (math / physics shell with code-eval). Strict network egress allowlist (paper databases + retrieval APIs only).
- **Bright-line gates.** Existing [14](../projects/polaris/docs/blocks/14-bright-line-gates.md) extended for `PUBLISH_DRAFT`, `EXTERNAL_API_CALL`, `MEMORY_PROMOTE` action kinds.

**Phase:** P3-P4. Critical before any external-content research workflows go to production.

### L6 Operations — complete the four legs

[264](264-agent-observability-stack-2026.md)–[268](268-agent-operations-synthesis-2026.md).

- **Observability.** Existing HIR ([16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md)) extended to OTel-shape with traceparent propagation across A2A / MCP. Phoenix or LangSmith as consumer.
- **Evaluation.** Eval-driven development on per-domain shell (ML / bio / math / physics / social). Production-eval feedback loop via HIR.
- **Durability.** LangGraph checkpointer (P3) + idempotency keys on every external API call + saga pattern for multi-step research workflows.
- **SRE.** SLOs on research-program completion rate, cost-per-program, escalation-rate. Runbooks for the five canonical incidents. Four-axis rollback (model + prompt + skill + runtime).

**Phase:** P4. Production-readiness gate.

### L7 Compliance — research-program audit

[272-agent-compliance-and-audit](272-agent-compliance-and-audit.md).

- Polaris is **not** EU AI Act high-risk by default (research-assistance is not in Annex III), but high-stakes research (medical / clinical / legal) elevates risk tier.
- **Action:** EU AI Act risk-tier classification per research program; SOC 2 Type II if commercial; technical documentation maintained automatically from SBOM + observability; right-to-erasure for any user-provided context.
- **Compliance dashboard** alongside SLO dashboard; compliance regression tests in CI.

**Phase:** P5 (only after L1-L6 complete).

## Phased rollout

| Phase | Scope | Duration | Gate |
|---|---|---|---|
| **P1** (current) | L1 Foundation, partial L2 Capability | Ongoing | Existing Phase-0 deliverables |
| **P2** | Complete L2 Capability + L3 Protocol (MCP upgrade, A2A endpoints, SKILL.md marketplace) | 90 days | Eval suite + observability captures protocol traffic |
| **P3** | L4 Runtime (LangGraph wrap) + L5 Security (5-layer defense) | 90 days | Security checklist passing; durability proof |
| **P4** | L6 Operations (full 4 legs) | 90 days | SLOs measurable; runbooks rehearsed |
| **P5** | L7 Compliance + production hardening | 90 days | External audit pass for relevant frameworks |

**Total: ~12 months from current Phase-0 to production-ready Polaris.**

## Key cross-project dependencies

- **`harness_core/`** shared package: routines, protocols, observability, security, eval, finetuning, sre. Polaris imports.
- **Argus** ([209-argus-multi-hop-collaborative-apply-plan](209-argus-multi-hop-collaborative-apply-plan.md)) provides the skill-marketplace + skill-auto-creator services Polaris consumes.
- **Lyra** ([280-lyra-seven-layer-stack-apply-plan](280-lyra-seven-layer-stack-apply-plan.md)) provides reference local-first patterns.

## Polaris-specific implications

- **Long-running daemon mode** is polaris's value prop; durability is non-negotiable.
- **Multi-domain shells** require domain-specific eval suites + verifier pairs.
- **External-content reading** (papers, web, retrieval) makes prompt-injection defense critical.
- **Cross-model adversarial pair** ([02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md)) is structurally enforced; verifier-from-different-family.
- **Research-program backlog** as filesystem state ([13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md)) maps to Anthropic Agent Teams shared-task-list pattern ([250](250-anthropic-agent-teams.md)).

## Cost economics for Polaris

[278-agent-unit-economics-2026](278-agent-unit-economics-2026.md). Long-running research programs are token-heavy:

- Frontier-API direct: $50-200/program.
- + caching + routing: $15-60/program.
- + finetuned domain shell: $5-20/program.
- + local-first option (M4 Max for typical workloads): $0 marginal.

Polaris's daemon-driven scheduling absorbs latency, allowing local-first as the default for most programs.

## One-line takeaway

**Polaris adopts the seven-layer stack across five 90-day phases — L1 Foundation already strong, L2-L3 in P2 (capability + protocol upgrades), L4-L5 in P3 (LangGraph wrap + security), L6 in P4 (operations completion), L7 in P5 (compliance) — leveraging the existing canonical-block architecture (Permission Bridge, Daemon, Bright-line Gates, ReasoningBank, HIR observability) so the work is layer completion + protocol stack adoption + security hardening + LangGraph wrapping rather than starting from scratch; the result is a production-grade autonomous research agent at ~12 months from current Phase-0.**
