# 292 — Syndicate × Seven-Layer Stack Apply Plan 2026

**Anchors.** Syndicate — multi-agent collaboration / orchestration framework ([projects/syndicate](../projects/syndicate/)). Companion: [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md), [189-recursive-multi-agent-systems](189-recursive-multi-agent-systems.md), [193-recursive-world-organizations-synthesis](193-recursive-world-organizations-synthesis.md), [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md).

**One-line definition.** A **per-layer apply plan** for Syndicate — the **multi-agent collaboration / orchestration framework** that turns the consensus 2026 architecture (lead-and-spokes hierarchical with structured channels per [251](251-multi-agent-teams-2026-synthesis.md)) into a **reusable orchestration substrate** other projects deploy on top of — emphasizing **MAST-aware design** ([251](251-multi-agent-teams-2026-synthesis.md)) (audit pipeline + cluster-classified failure modes + structured-channel comms by default), **trained orchestrator support** (Puppeteer-shape RL on orchestration traces, [251](251-multi-agent-teams-2026-synthesis.md)), and **recursive multi-agent depth** ([189-recursive-multi-agent-systems](189-recursive-multi-agent-systems.md)) — staged across four 90-day phases.

## Per-layer plan

### L1 Foundation
Standard. Permission Bridge per-team scope. Daemon for long-running team coordination.

### L2 Capability
**Lead-and-spokes** as default architectural pattern. **Trained orchestrator** support via RL on orchestration traces. **Cross-channel verifier** mandatory at integration boundary. **MAST-classified observability**.

### L3 Protocol
- **MCP**: shared task list MCP, mailbox MCP.
- **A2A**: each member of a syndicate team is A2A-addressable; cross-vendor team composition possible.
- **AGNTCY**: published OASF; trust-tiered team-member discovery.
- **NATS**: pub/sub for inter-member communication; subject-routed by team / role / message-kind.
- **Routines + Agent Teams**: native first-class.

### L4 Runtime
**Hybrid**: LangGraph for stateful orchestration; AutoGen v0.4 actor model for event-driven; Anthropic Agent Teams for in-process parallel; A2A for cross-runtime peers. The **`harness_core/protocols/`** package as the shared substrate.

### L5 Security
- **MAST cluster-2 mitigation**: structured channels, no free-form GroupChat by default.
- **Cross-channel verifier** at integration boundary.
- **Per-member isolation**: worktree + container per member.
- **Memory tier isolation**: untrusted memory cannot promote without explicit approval.
- **Bright-line**: `MEMORY_PROMOTE`, `CROSS_VENDOR_INVOKE`, `RECURSIVE_DEPTH_EXCEEDED`.

### L6 Operations
- **MAST-aware observability**: every team trace tagged for failure-mode classification.
- **Eval**: per-team-pattern eval suite (from MultiAgentBench / MARBLE).
- **Durability**: LangGraph checkpointer per team; saga for multi-step orchestration.
- **SRE**: SLO on team-success-rate; runbooks for cluster-1/2/3 failures.

### L7 Compliance
**SOC 2** for commercial deployment. **EU AI Act**: tier classification per use case. **MAST-cluster-rate reporting** in compliance documentation.

## Phased rollout

| Phase | Scope | Duration |
|---|---|---|
| **P1** | L1-L2 + lead-and-spokes patterns + structured channels | 90 days |
| **P2** | L3-L4 (multi-runtime substrate) + cross-channel verifier | 90 days |
| **P3** | L5 Security (MAST cluster-2 mitigation) + L6 Operations (MAST-aware) | 90 days |
| **P4** | Trained orchestrator (Puppeteer-shape RL) + L7 Compliance | 90 days |

## One-line takeaway

**Syndicate is the multi-agent orchestration substrate — lead-and-spokes default, structured channels (shared task list + mailbox + plan approvals) as primary comms, MAST-aware observability and runbooks, cross-channel verifier mandatory, recursive depth bounded by bright-line gates, hybrid runtime (LangGraph + AutoGen v0.4 + Agent Teams + A2A) for cross-runtime team composition; the project that turns the consensus 2026 multi-agent architecture into a reusable substrate the rest of the ecosystem builds on.**
