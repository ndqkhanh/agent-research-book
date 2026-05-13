# 282 — Argus × Seven-Layer Stack Apply Plan 2026

**Anchors.** Argus — provider of marketplace, trust tiering, and skill-auto-creation services for the cross-project ecosystem ([projects/argus](../projects/argus/)). Per [194-argus-omega-enhanced-design](194-argus-omega-enhanced-design.md), [195-argus-omega-vol-2-trajectory-temporal-horizon](195-argus-omega-vol-2-trajectory-temporal-horizon.md), [196-argus-vs-field-skill-loading-comparison](196-argus-vs-field-skill-loading-comparison.md), [197-argus-omega-vol-3-recursive-skills-curator](197-argus-omega-vol-3-recursive-skills-curator.md), [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md), [209-argus-multi-hop-collaborative-apply-plan](209-argus-multi-hop-collaborative-apply-plan.md). Per-layer source: [225](225-agent-era-scaling-synthesis.md), [255](255-agntcy-oasf-acp-deep-dive.md), [257](257-agent-skill-marketplace-landscape.md), [258](258-agent-protocol-stack-synthesis-2026.md), [263](263-production-agent-runtime-synthesis-2026.md), [268](268-agent-operations-synthesis-2026.md), [273](273-agent-security-synthesis-2026.md).

**One-line definition.** A **per-layer apply plan** for argus — the **provider** of marketplace + trust tiering + skill-auto-creation services every other project consumes — emphasizing the **AGNTCY-shape registry** ([255](255-agntcy-oasf-acp-deep-dive.md)) and **marketplace economy** ([257](257-agent-skill-marketplace-landscape.md)) layers as load-bearing, the **provider role** that requires high availability + SOC 2 Type II + supply-chain attestation discipline ([270](270-agent-supply-chain-security.md), [272](272-agent-compliance-and-audit.md)), and the **recursive-skill-curator pattern** that auto-generates SKILL.md candidates from production traces of consumer projects (Polaris, Lyra, Mentat-Learn, etc.) — staged across five 90-day phases that turn argus from project into shared infrastructure that all other agents depend on.

## Argus shape

Argus inverts the consumer pattern of [203](203-polaris-multi-hop-reasoning-apply-plan.md), [208](208-lyra-multi-hop-collaborative-apply-plan.md), [210](210-mentat-learn-collaborative-apply-plan.md): instead of being a consumer of marketplace + trust + skill-curator services, argus is the **provider**. This makes argus's role unique — its production SLAs apply to every other agent in `projects/`. The seven-layer stack maps with **provider-role overlay**: high-availability, SOC 2-grade audit, supply-chain attestation, and provider-side trust-tier enforcement.

## Per-layer plan

### L1 Foundation — provider-grade

`harness_core/foundation/` baseline plus argus-specific provider primitives: high-availability daemon (active-passive failover), per-tenant Permission Bridge contexts, SLA-grade observability.

### L2 Capability — recursive skill curator + skill router

[225-agent-era-scaling-synthesis](225-agent-era-scaling-synthesis.md). Argus's distinctive capability is the **recursive skill curator** ([197-argus-omega-vol-3-recursive-skills-curator](197-argus-omega-vol-3-recursive-skills-curator.md)) and **skill router** ([180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md)):

- **Skill router**: queries from consumer projects → match to skills in argus's registry → return ranked candidates.
- **Skill auto-creator** ([10-skill-auto-creator](../projects/polaris/docs/blocks/10-skill-auto-creator.md)): consumes production traces from Polaris / Lyra / Mentat / others; extracts patterns; drafts SKILL.md candidates; submits to curator.
- **Curator**: reviews candidates; signs approved; publishes to marketplace.

**Phase:** P1-P2.

### L3 Protocol — argus is the AGNTCY backbone

[258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md).

- **MCP** ([256](256-mcp-2025-2026-evolution.md)): argus exposes registry-query as MCP tools (`marketplace.search`, `marketplace.install`, `marketplace.audit`).
- **A2A** ([254](254-a2a-protocol-deep-dive.md)): argus exposes A2A endpoints for marketplace operations + skill-curator submissions; consumer projects invoke via A2A.
- **AGNTCY OASF + ACP** ([255](255-agntcy-oasf-acp-deep-dive.md)): **argus IS the AGNTCY-shape registry** for the in-tree ecosystem. Self-host the discovery layer; OASF schemas published; trust attributes enforced.
- **Tailscale + NATS** ([253](253-tailscale-nats-mesh-for-distributed-agents.md)): argus accessible via tailnet for in-tree projects; NATS pub-sub for skill-publication and revocation events.
- **SKILL.md + marketplace** ([257](257-agent-skill-marketplace-landscape.md)): argus IS the marketplace; signed SKILL.md packages; OAuth 2.1 install flows.
- **Routines + Agent Teams** ([252](252-routines-pattern-for-self-hosted-agents.md), [250](250-anthropic-agent-teams.md)): Routines for nightly skill-curator runs; Agent Teams for multi-perspective skill review (security / capability / quality reviewers).

**Phase:** P1-P2 (load-bearing for the rest of the ecosystem).

### L4 Runtime — LangGraph + Postgres

[259-langgraph-deep-dive](259-langgraph-deep-dive.md). Argus has stateful workflows (curator → skill review → publication) requiring durability + audit. LangGraph + Postgres checkpointer is the natural fit. Vertex AI Agent Engine if Google-Cloud hosted.

### L5 Security — provider-grade defense

[273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md). **Provider role intensifies the security stake**:

- **Supply-chain attestation.** Argus's marketplace must publish signed manifests with **provable provenance**; vendor-tier reputation; compromise of argus = compromise of all consumers. **Critical**: cosign / Sigstore for SKILL.md signatures.
- **Submission scanning.** Every SKILL.md submission scanned for prompt injection ([269](269-prompt-injection-2026.md)), supply-chain anomalies, scope-expansion attacks, sleeper-agent triggers.
- **Curator team** with cross-channel verifier across 3 model families.
- **Trust tiering enforcement.** Argus implements the trust-tier policy other projects depend on. Audit cadence: quarterly external for `audited` tier; annual SOC 2.
- **Revocation propagation.** When a skill is retracted, argus pub-subs to consumers via NATS for auto-rollback.
- **Isolation.** Submitted skills run in micro-VM sandbox during eval; never on host.

**Phase:** P2 + ongoing.

### L6 Operations — high-availability + 24/7

[268-agent-operations-synthesis-2026](268-agent-operations-synthesis-2026.md). Argus is shared infrastructure; its outages affect all consumers:

- **Observability.** Full OTel + per-tenant attribution; published status page.
- **Eval.** Per-skill regression suite; production traces with eval-link; blocked-merge on regression.
- **Durability.** Postgres checkpointer; saga for multi-step skill-publication; outbox for retraction-event delivery.
- **SRE.** 99.9% availability SLO; on-call rotation; runbooks for marketplace compromise / submission flood / curator overload / revocation propagation lag.

**Phase:** P3 (production-readiness gate).

### L7 Compliance — SOC 2 Type II as table-stakes

[272-agent-compliance-and-audit](272-agent-compliance-and-audit.md). Argus serves multiple tenants; SOC 2 is mandatory:

- **SOC 2 Type II** annual audit covering security, availability, confidentiality, privacy.
- **GDPR** as data processor (consumers are controllers).
- **EU AI Act**: argus is **provider of GPAI components** (skills); transparency obligations apply.
- **ISO/IEC 42001** voluntary certification for AI Management System.
- **Vendor-tier compliance metadata** in OASF manifests.
- **Compliance dashboard** alongside SLO dashboard.

**Phase:** P4-P5.

## Phased rollout

| Phase | Scope | Duration |
|---|---|---|
| **P1** | L1 Foundation + L2 Capability (skill router + auto-creator) + bootstrap L3 (MCP + A2A) | 90 days |
| **P2** | Complete L3 Protocol (AGNTCY backbone, marketplace) + L4 Runtime + L5 Security (provider grade) | 90 days |
| **P3** | L6 Operations (HA + SLOs) + curator team (multi-perspective review) | 90 days |
| **P4** | L7 Compliance (SOC 2 prep + audit) | 90 days |
| **P5** | Recursive skill-curator at scale; cross-project consumption growth | 90 days |

## Argus-specific patterns

- **Argus inverts the consumer pattern.** Per [209](209-argus-multi-hop-collaborative-apply-plan.md) — the only project that's a *provider*.
- **Recursive skill curator** ([197](197-argus-omega-vol-3-recursive-skills-curator.md)) — uses its own skills to curate other skills.
- **Skill auto-creator** ([10-skill-auto-creator](../projects/polaris/docs/blocks/10-skill-auto-creator.md)) — production traces feed candidate skills.
- **Trust-tier enforcement** ([p24-trust-tiering-retractions](../projects/polaris/docs/research/p24-trust-tiering-retractions.md)) — central to provider role.
- **Cross-channel verifier** in curator review for every submission.
- **Trajectory + temporal horizon** ([195-argus-omega-vol-2-trajectory-temporal-horizon](195-argus-omega-vol-2-trajectory-temporal-horizon.md)) — long-horizon skill validation.
- **In-tree consumers**: Polaris, Lyra, Mentat-Learn, Atlas, Aegis, Helix, Cipher, Harmony, Orion, Syndicate, Quanta-Proof, Vertex-Eval, Gnomon, Open-Fang.

## Cross-project dependencies (argus is the depended-upon)

- Every consumer project depends on argus for skill marketplace + trust tiers.
- Argus depends on Lyra patterns for memory ([233-memory-scaling-for-agents](233-memory-scaling-for-agents.md)).
- Argus depends on `harness_core/` shared infrastructure.

## Cost economics

[278-agent-unit-economics-2026](278-agent-unit-economics-2026.md). Argus runs as shared infrastructure:

- Hosting: $50-500/mo VPS or managed Postgres + compute, scaling with consumer count.
- Compute for curator runs: $1-10/skill-review.
- Per-consumer-project: amortized cost is low ($5-50/mo).
- Revenue model: free for in-tree projects; sustainable via foundation funding or sponsorship.

## One-line takeaway

**Argus is the **provider** in the in-tree ecosystem — marketplace + trust tiering + skill-auto-creation as shared infrastructure across five 90-day phases — adopting the seven-layer stack with provider-role overlay (high-availability + SOC 2 Type II + supply-chain attestation + cross-channel verifier in curator review), implementing AGNTCY-shape registry and SKILL.md marketplace as core deliverables, and turning the recursive skill-curator pattern into infrastructure that auto-generates skills from production traces of every consumer project; argus's outages affect all consumers, so its SLA discipline is unique among the projects.**
