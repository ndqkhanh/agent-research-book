# 281 — Mentat-Learn × Seven-Layer Stack Apply Plan 2026

**Anchors.** Mentat-Learn — self-improving multi-channel personal assistant ([projects/mentat-learn](../projects/mentat-learn/)) fusing OpenClaw (multi-channel gateway), SemaClaw (DAG teams + SOUL.md + PermissionBridge), Hermes Agent (closed skill-learning loop + Honcho dialectic user model). Extends [210-mentat-learn-collaborative-apply-plan](210-mentat-learn-collaborative-apply-plan.md). Per-layer source: [225](225-agent-era-scaling-synthesis.md), [258](258-agent-protocol-stack-synthesis-2026.md), [263](263-production-agent-runtime-synthesis-2026.md), [268](268-agent-operations-synthesis-2026.md), [273](273-agent-security-synthesis-2026.md), [277](277-agent-ux-patterns-2026.md).

**One-line definition.** A **per-layer apply plan** for Mentat-Learn — the **multi-channel self-improving personal assistant** that meets users across Slack / WhatsApp / Telegram / email / iMessage / web / voice with a coherent SOUL.md-anchored persona, a Hermes-style closed skill-learning loop, and a Honcho dialectic user model — emphasizing the **multi-channel UX layer** ([277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md), voice + text composition), the **Hermes-style self-improvement** loop (skill auto-creator ([10-skill-auto-creator](../projects/polaris/docs/blocks/10-skill-auto-creator.md)) submitting to argus's curator), and the **per-user privacy + scope** that distinguishes a personal assistant from an enterprise agent — staged across four 90-day phases targeting the $5-VPS-deployable envelope Hermes Agent cites.

## Mentat-Learn shape

Mentat-Learn is a **per-user always-on personal assistant** running on a small VPS ($5-tier deployable target), serving one user across many channels with one coherent persona. The architecture combines: SOUL.md persona partition (SemaClaw-shape) + Hermes closed-skill-learning loop + Honcho dialectic user model + multi-channel gateway (OpenClaw-shape) + DAG teams orchestration. The seven-layer stack maps with a **multi-channel UX overlay** and **per-user privacy + sovereignty** as the load-bearing concerns.

## Per-layer plan

### L1 Foundation — adapt with per-user context

Vendor `harness_core/foundation/`. **Per-user permission contexts** scoped tighter than enterprise (no `--dangerously-skip-permissions`); bright-line escalation through user's preferred channel. Daemon as the always-on substrate.

### L2 Capability — small + cheap-runtime profile

[225-agent-era-scaling-synthesis](225-agent-era-scaling-synthesis.md). Mentat targets $5-VPS deployment, so capability sits below frontier:

- **Pretraining**: Phi-4 / Qwen-2.5-7B for tier-1 tasks; cloud opt-in for tier-2.
- **TTC**: lightweight; thinking only on hard tasks via cloud.
- **Trajectory**: Hermes-style skill loop is a trajectory amplifier.
- **Multi-agent**: DAG teams for complex workflows (rare).
- **Verifier**: cross-channel via cloud opt-in.

### L3 Protocol — full stack with multi-channel emphasis

[258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md).

- **MCP** ([256](256-mcp-2025-2026-evolution.md)): tools per integrated channel (Slack MCP, Telegram MCP, email MCP, iMessage bridge MCP).
- **A2A** ([254](254-a2a-protocol-deep-dive.md)): expose Mentat as A2A endpoint; consume argus's skill-curator via A2A.
- **AGNTCY** ([255](255-agntcy-oasf-acp-deep-dive.md)): published OASF manifest for argus discovery.
- **Tailscale + NATS** ([253](253-tailscale-nats-mesh-for-distributed-agents.md)): VPS + user's home device on tailnet; sync persona + skills.
- **SKILL.md + marketplace** ([257](257-agent-skill-marketplace-landscape.md)): consumes argus's curated marketplace; min-trust-tier `audited`.
- **Routines + Agent Teams** ([250](250-anthropic-agent-teams.md), [252](252-routines-pattern-for-self-hosted-agents.md)): Routines for scheduled summaries / reflections / weekly reviews; Agent Teams sparingly.

### L4 Runtime — OpenAI Agents SDK for handoff + voice

[260-openai-agents-sdk-deep-dive](260-openai-agents-sdk-deep-dive.md). Mentat's **multi-channel** + **voice composition** + **triage-to-specialist** workflow fits OpenAI Agents SDK's handoff pattern naturally. Voice agent (Realtime) hands off to text specialists for complex tasks ([277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md)). LangGraph for the Hermes skill-learning loop (state-machine).

**Hybrid:** OpenAI Agents SDK for user-facing multi-channel; LangGraph for backend reflection / consolidation.

### L5 Security — multi-channel attack surface

[273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md).

- **Prompt-injection defense.** Critical — Mentat reads emails, Slack messages, web links shared by user. Spotlight + classifier ensemble + instruction hierarchy.
- **Supply chain.** Vendored core skills; marketplace skills via argus with strict trust-tier; SBOM.
- **Isolation.** Per-user container; per-channel adapter container; egress allowlist.
- **Memory provenance.** SOUL.md is highest-trust; Hermes-derived skills next; channel-derived data lowest-trust.
- **Bright-line gates.** Send-message-on-user-behalf is bright-lined; user reviews before send (especially financial / professional channels).

### L6 Operations — Phoenix + per-user SLOs

[268-agent-operations-synthesis-2026](268-agent-operations-synthesis-2026.md).

- **Observability.** Phoenix on the VPS; per-channel + per-skill spans.
- **Evaluation.** Persona-fidelity ≥ 0.9 across 5+ interactions (per [210](210-mentat-learn-collaborative-apply-plan.md) target); skill-pass-rate per skill.
- **Durability.** LangGraph SQLite checkpointer; idempotency on every channel send.
- **SRE.** Simple — one user; cost-runaway + bright-line escalation log; user is on-call.

### L7 Compliance — per-user GDPR

[272-agent-compliance-and-audit](272-agent-compliance-and-audit.md).

- **EU AI Act:** personal-use minimal-risk; transparency only.
- **GDPR:** user is controller; right-to-erasure single-button.
- **HIPAA / regulated workflows:** only if user opts into healthcare-channel.

## Phased rollout

| Phase | Scope | Duration |
|---|---|---|
| **P1** | L1 Foundation + L2 Capability (small-tier) + partial L3 (channel adapters via MCP) | 90 days |
| **P2** | Complete L3 Protocol + L4 Runtime (OpenAI Agents SDK + LangGraph) + voice composition | 90 days |
| **P3** | L5 Security (multi-channel) + L6 Operations + Hermes skill loop | 90 days |
| **P4** | L7 Compliance (privacy primitives) + persona polish + dialectic user modeling | 90 days |

## Mentat-Learn-specific patterns

- **Multi-channel gateway** (OpenClaw-shape) routes inbound from any channel to the agent; outbound replies via the same channel.
- **SOUL.md persona** loaded per session; anchors identity across channels.
- **Hermes closed skill-learning loop** — completed task → pattern extraction → SKILL.md → submitted to argus curator for marketplace publication → re-installed from marketplace.
- **Honcho dialectic user model** — separate component modeling "who the user is to the agent."
- **Persona fidelity ≥ 0.9** as a tracked SLI ([210](210-mentat-learn-collaborative-apply-plan.md)).
- **Voice + text composition** ([277](277-agent-ux-patterns-2026.md)) — voice agent hands off to text specialists for complex tasks.
- **$5-VPS deployable** — small-tier model + lean ops + per-user scope.

## Cost economics

[278-agent-unit-economics-2026](278-agent-unit-economics-2026.md). Per-user economics:

- $5/mo VPS for the always-on daemon + memory + Phoenix.
- LLM cost: ~$5-15/mo on cheap-tier API (DeepSeek-V3 / Haiku) for typical use.
- Optional: $0 marginal if user runs LLM on their home device, accessed via Tailscale.

Total: ~$10-25/mo per user. Sustainable as freemium SaaS or open-source self-host.

## Cross-project dependencies

- **Argus** ([282-argus-seven-layer-stack-apply-plan](282-argus-seven-layer-stack-apply-plan.md)) provides the skill-curator + marketplace Mentat consumes.
- **Lyra** ([280-lyra-seven-layer-stack-apply-plan](280-lyra-seven-layer-stack-apply-plan.md)) provides the local-first memory primitives.

## One-line takeaway

**Mentat-Learn adopts the seven-layer stack as the canonical multi-channel self-improving personal assistant — fusing OpenClaw (multi-channel) + SemaClaw (SOUL.md + PermissionBridge) + Hermes (closed skill loop) + Honcho (dialectic user model) at $5-VPS deployable scale across four 90-day phases — using OpenAI Agents SDK for handoff + voice, LangGraph for reflection loops, argus-served marketplace skills with audited trust tier, Tailscale + NATS for cross-device sync, and per-user GDPR-shape compliance; persona fidelity ≥ 0.9 as the load-bearing SLI; total cost $10-25/user/month or $0 marginal if user runs local LLM.**
