# 258 — Agent Protocol Stack Synthesis 2026: how MCP × A2A × AGNTCY × NATS × SKILL.md compose

**Synthesis of:** [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md), [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md), [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md), [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md). Cross-references: [07-model-context-protocol](07-model-context-protocol.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md), [04-skills](04-skills.md), [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md), [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md), [177-skills-discovery-curator-strongest-2026-techniques](177-skills-discovery-curator-strongest-2026-techniques.md), [250-anthropic-agent-teams](250-anthropic-agent-teams.md), [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md), [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md), [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md). Also: [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md), [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [225-agent-era-scaling-synthesis](225-agent-era-scaling-synthesis.md).

**One-line definition.** A unified **layered protocol stack** for the 2026 agent ecosystem — **MCP** for tool exposure, **A2A** for inter-agent semantic comms, **AGNTCY (OASF + ACP + SLIM)** for schema-and-discovery, **NATS + Tailscale** for distributed transport, **SKILL.md + marketplaces** for portable specialist definitions, **Routines + Agent Teams** for invocation surfaces — six orthogonal layers that **compose without overlap**, each with a single load-bearing job, each adoptable independently, and together giving the field its first complete **production-grade agent infrastructure stack** matching the maturity of REST-over-HTTP for the previous era of services.

## Why this synthesis matters (the field finally has a complete stack, and the layers don't fight)

For most of 2024–2025 the agent-protocol landscape was a fight between competing standards — MCP vs A2A vs custom, AGNTCY vs A2A vs in-house registries, OAuth vs API-keys vs nothing — and the consensus position oscillated. By May 2026 the picture stabilized, and what stabilized is **not a winner-takes-all single protocol** but a **stack of orthogonal protocols** where each layer has one job and the layers don't compete. This is the same pattern as the web's REST-over-HTTPS-over-TCP-over-IP — one stack, many layers, each independently adoptable. The difference is that the agent stack adds primitives the web didn't need: signed capability manifests (OASF, Signed Agent Cards), inter-agent task semantics (A2A), tool-exposure standardization (MCP), and portable specialist definitions (SKILL.md).

The six-layer stack ([Table 1](#empirical-results-table-the-six-layer-stack-may-2026)) is what every agent in `projects/` should adopt. The implication is concrete: **one shared `harness_core/protocols/` package** implementing client and server for all six layers (where applicable), and every project — polaris, lyra, argus, atlas-research, aegis-ops, mentat-learn, helix-bio, cipher-sec, harmony-voice, vertex-eval, gnomon, orion-code, syndicate, quanta-proof, open-fang — imports that package and gets the full stack for free. This is the high-leverage architectural move of 2026.

The orthogonality matters because it lets each agent **adopt the layers it needs and skip the rest**. A purely-internal agent (no external invocations) needs MCP for tools but not A2A. A specialist that's published in a marketplace needs SKILL.md and Signed Agent Cards but maybe not full AGNTCY. A two-host distributed agent needs Tailscale + NATS but not necessarily A2A (if both hosts run the same runtime). The stack is **opt-in per layer**, not all-or-nothing.

Take this synthesis seriously and three things change. **First**, you stop debating "which protocol should we use" and start asking "for which layer." The answer is: pick the standardized one in each layer, not invent your own. **Second**, you understand that **trust and identity** is the cross-cutting primitive that ties the stack together — Signed Agent Cards (A2A), OASF trust attributes (AGNTCY), OAuth 2.1 (MCP, A2A), signed SKILL.md manifests (marketplace) — they all use the same EdDSA / OAuth 2.1 / JWK primitives, and the harness's permission bridge is the integration point. **Third**, **observability across the stack** becomes uniform via HIR-shape events ([16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md)) — every protocol-level call emits a typed event, trace_id propagates cross-protocol, and your single HIR log captures the full picture.

## The six layers

### Layer 1 — Tool exposure: **MCP** ([256](256-mcp-2025-2026-evolution.md))

- **Job:** how an LLM-agent calls tools.
- **Primitives:** Tools, Resources, Prompts, Sampling.
- **Wire:** Streamable HTTP (`POST /mcp`, optional SSE).
- **Auth:** OAuth 2.1, RFC 8414 metadata, RFC 7591 dynamic registration.
- **Spec status:** 2025-03-26 stable, 2026 roadmap stateless servers.
- **Ecosystem:** 5,000+ public servers, native in Claude Code, Codex, Cline, Continue, Cursor.

### Layer 2 — Inter-agent comms: **A2A** ([254](254-a2a-protocol-deep-dive.md))

- **Job:** how one agent calls another agent.
- **Primitives:** Agent Cards, Tasks, Messages, Artifacts.
- **Wire:** HTTPS, OAuth 2.1, SSE for streaming.
- **Identity:** Signed Agent Cards (EdDSA over canonical JSON).
- **Spec status:** v1.0 (early 2026), Linux Foundation, 150+ orgs.
- **Ecosystem:** Native in Google ADK, Salesforce AgentForce, Microsoft Copilot Studio.

### Layer 3 — Schema and discovery: **AGNTCY (OASF + ACP + SLIM)** ([255](255-agntcy-oasf-acp-deep-dive.md))

- **Job:** describe agent capabilities (OASF), discover agents (registry), invoke when A2A is overkill (ACP).
- **Primitives:** OASF schemas, ACP REST API, SLIM intra-DC transport.
- **Wire:** OASF is JSON-Schema-shape, ACP is REST, SLIM is mTLS-over-QUIC.
- **Trust:** OASF trust attributes (audited / signed / sandboxed / verified-vendor).
- **Spec status:** OASF 0.3, ACP 0.2 (May 2026); pre-1.0 but production-ready for early adopters.
- **Ecosystem:** Cisco-led, donated to LF, 46 GitHub repos.

### Layer 4 — Distributed transport: **Tailscale + NATS** ([253](253-tailscale-nats-mesh-for-distributed-agents.md))

- **Job:** L3 NAT-traversed encrypted connectivity (Tailscale) + L7 subject-routed pub/sub with durable replay (NATS leaf nodes).
- **Primitives:** WireGuard direct or DERP relay; NATS subjects + JetStream streams.
- **Auth:** Tailscale identity (SSO + ACLs), NATS NKey or JWT.
- **Status:** Tailscale GA, NATS Apache-2.0, JetStream production-stable.
- **Ecosystem:** 100-device free tier (Tailscale), self-hostable (Headscale + nats-server).

### Layer 5 — Portable specialist definitions: **SKILL.md + marketplaces** ([257](257-agent-skill-marketplace-landscape.md))

- **Job:** package, distribute, discover, install agent specialists across runtimes.
- **Primitives:** SKILL.md (frontmatter + body), Signed Agent Cards / OASF manifests, marketplace search APIs.
- **Wire:** Markdown files; OAuth 2.1 install flows.
- **Marketplaces:** Smithery (MCP), subagents.cc, buildwithclaude, LobeHub Skills.
- **Status:** SKILL.md aligned across Anthropic + OpenAI Codex (Dec 2025); marketplaces production-grade.

### Layer 6 — Invocation surfaces: **Routines + Agent Teams** ([250](250-anthropic-agent-teams.md), [252](252-routines-pattern-for-self-hosted-agents.md))

- **Job:** trigger and orchestrate agent runs — fire-from-anywhere (Routines) and parallel-team-of-agents (Agent Teams).
- **Primitives:** routine config + cron/API/webhook triggers + isolated execution; lead + teammates + shared task list + mailbox.
- **Wire:** HTTPS for Routines API; filesystem for Agent Teams coordination.
- **Status:** Anthropic-productized in Claude Code; replicable in self-hosted agents per [252](252-routines-pattern-for-self-hosted-agents.md).

## How the layers compose

### Reference flow: cross-vendor agent invocation

```
1. Caller (CrewAI agent) wants to invoke a literature-review specialist.
2. Caller queries AGNTCY discovery: GET /v1/agents/search?capability=literature-review&min_trust=audited.
3. Registry returns OASF manifest for polaris-research agent, including A2A endpoint.
4. Caller verifies OASF signature against polaris vendor's JWK set.
5. Caller obtains OAuth 2.1 token from polaris's auth server (RFC 8414).
6. Caller submits A2A task: POST /a2a/v1/tasks with capability="literature-review".
7. Polaris-research agent receives task, runs its agent loop using MCP-exposed tools internally.
8. Polaris emits SSE progress updates over A2A; caller streams them.
9. Polaris produces artifacts (markdown report, citations JSON), returns via A2A.
10. Caller stores artifacts; HIR-equivalent log records the cross-vendor call with trace_id.
```

Each step uses one layer; layers don't overlap.

### Reference flow: distributed personal agent

```
1. User has Lyra-home and Lyra-laptop, both behind NAT.
2. Both hosts run Tailscale, joined the same tailnet (Layer 4).
3. Both hosts run nats-server in leaf-node mode, dialing $5-VPS hub (Layer 4).
4. Lyra-home auto-captures a memory observation; emits NATS message on lyra.home.memory.delta.
5. Lyra-laptop's JetStream consumer receives the delta; applies to its memory store.
6. User invokes a skill on Lyra-laptop; skill is a marketplace-installed SKILL.md (Layer 5).
7. Skill calls MCP-exposed tools internally (Layer 1).
8. If skill needs to delegate to a specialist on Lyra-home, uses A2A over the tailnet (Layer 2).
9. Routine schedules a nightly memory consolidation on Lyra-home (Layer 6 / Routines).
```

Six layers, six orthogonal jobs, no fight.

### Reference flow: Anthropic Agent Teams + cross-vendor

```
1. Lead session in Claude Code spawns a 4-teammate Agent Team (Layer 6 / Agent Teams).
2. Three teammates are Claude Code instances; one teammate is an A2A-exposed Google ADK agent.
3. Lead writes shared tasks to ~/.claude/tasks/{team-name}/ (filesystem, in-process for Claude teammates).
4. The Google ADK teammate is invoked via A2A (Layer 2); its responses appear as artifacts.
5. Each teammate uses MCP for its own tool calls (Layer 1).
6. The skill specifications loaded into each teammate are SKILL.md from internal marketplace (Layer 5).
```

Cross-vendor team composition without bespoke adapter code.

## Empirical results (table — the six-layer stack May 2026)

| Layer | Standard | Status | When optional | When mandatory |
|---|---|---|---|---|
| 1 — Tool exposure | MCP | Stable (2025-03-26 spec) | Single-binary agent with hardcoded tools | Any production agent with tools |
| 2 — Inter-agent comms | A2A | v1.0 (Linux Foundation) | All agents in one runtime | Cross-vendor / cross-runtime |
| 3 — Schema + discovery | AGNTCY (OASF, ACP, SLIM) | OASF 0.3, ACP 0.2 (production-ready early adopter) | Internal-only single-tenant | Multi-tenant, marketplace, registry-driven |
| 4 — Distributed transport | Tailscale + NATS | Tailscale GA, NATS Apache-2.0 | Single-host deployment | Multi-host distributed agents |
| 5 — Portable specialists | SKILL.md + marketplaces | Aligned across Anthropic + OpenAI Codex | Hand-rolled prompts | Marketplace consumption / publication |
| 6 — Invocation surfaces | Routines + Agent Teams | Anthropic-productized; self-host per [252](252-routines-pattern-for-self-hosted-agents.md) | Manual CLI invocation | Fire-from-anywhere or parallel teams |

## Adoption matrix per agent in `projects/`

| Agent | L1 MCP | L2 A2A | L3 AGNTCY | L4 Tailscale+NATS | L5 SKILL.md | L6 Routines+Teams |
|---|---|---|---|---|---|---|
| polaris | ✅ | ✅ | ✅ (registry) | ✅ (multi-host runs) | ✅ (researcher / verifier / synthesizer SKILL.md's) | ✅ (scheduled research routines, Agent Teams for parallel review) |
| lyra | ✅ | ✅ | ⚠ (optional) | ✅ (two-Lyra deployment) | ✅ (skill memory) | ✅ (Routines for periodic consolidation) |
| argus | ✅ | ✅ | ✅ (skill auto-creator marketplace) | ✅ | ✅ | ✅ |
| atlas-research | ✅ | ✅ | ⚠ | ⚠ | ✅ | ✅ |
| aegis-ops | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| mentat-learn | ✅ | ✅ | ⚠ | ⚠ | ✅ | ✅ |
| helix-bio | ✅ | ⚠ | ⚠ | ⚠ | ✅ | ✅ |
| cipher-sec | ✅ | ✅ | ✅ (high trust required) | ⚠ | ✅ | ✅ |
| harmony-voice | ✅ | ⚠ | ⚠ | ⚠ | ⚠ | ⚠ (latency-bound) |
| vertex-eval | ✅ | ✅ | ⚠ | ⚠ | ✅ | ✅ |
| gnomon | ✅ | ⚠ | ⚠ | ⚠ | ✅ | ✅ |
| orion-code | ✅ | ✅ | ⚠ | ⚠ | ✅ | ✅ |
| syndicate | ✅ | ✅ | ⚠ | ⚠ | ✅ | ✅ |
| quanta-proof | ✅ | ⚠ | ⚠ | ⚠ | ✅ | ✅ |
| open-fang | ✅ | ⚠ | ⚠ | ⚠ | ✅ | ✅ |

## Decision matrix: which layer to invest in next

### Regime 1 — solo agent on one host

Priorities: L1 (MCP) for tools, L5 (SKILL.md) for portable specialists. Skip L2, L3, L4 until you have multi-runtime or multi-host needs.

### Regime 2 — multi-runtime ecosystem

Priorities: + L2 (A2A endpoint), + L3 (AGNTCY OASF for schema). L4 still optional if all hosts on same network.

### Regime 3 — distributed personal agents

Priorities: + L4 (Tailscale + NATS). L2 over the tailnet for cross-host inter-agent calls.

### Regime 4 — marketplace publisher

Priorities: + L3 (OASF trust attributes), + L5 (signed SKILL.md, marketplace publication). All-six adoption.

### Regime 5 — production agent platform

Priorities: All six layers. Full stack with HIR observability across all protocols.

## How the stack composes with the four scaling axes ([225](225-agent-era-scaling-synthesis.md))

The protocol stack is **infrastructure**; the scaling axes are **capability levers**. They compose:

- **A1 (pretraining)** — base model selection, unchanged by protocol choices.
- **A2 (test-time compute)** — TTC happens inside one agent's loop; MCP exposes tools for sequential revision; A2A delegates to specialist agents for parallel exploration.
- **A3 (trajectory)** — trajectory budget per A2A task; per-routine budget gate; ACI design lives in MCP servers.
- **A4 (multi-agent)** — Agent Teams ([250](250-anthropic-agent-teams.md)) implements A4 in-runtime; A2A implements A4 cross-runtime; AGNTCY discovery enables dynamic team composition.
- **V (verifier)** — verifier exposed as MCP server (tool) or A2A agent (specialist); cross-channel adversarial pair across vendor boundary.

## Variants and ablations

- **Layer 1 only.** Internal agents with MCP-exposed tools, no inter-agent or distributed needs.
- **Layers 1 + 5.** Hand-rolled agents that use marketplace-published SKILL.md specialists.
- **Layers 1 + 2.** Multi-runtime agents that interoperate but stay single-host.
- **Layers 1 + 4.** Distributed agents on a tailnet with NATS, but all running the same runtime (no need for A2A).
- **Full six-layer stack.** Production agent platforms.
- **Layer-bypass via custom protocol.** When a custom A2A-shape protocol is sufficient for a single integration; pragmatic but reduces interop.
- **MCP-as-A2A-substitute.** Some teams expose agent capabilities as MCP tools; works for simple cases but conflates trust and semantic boundaries.

## Failure modes and limitations of the stack

- **Spec churn at the lower-version layers.** OASF 0.3, ACP 0.2 — pre-1.0 churn risk; pin versions.
- **Layer adoption asymmetry.** Some vendors adopt L1 (MCP) but skip L2 (A2A); cross-vendor benefits gated.
- **Trust primitive sprawl.** Signed Agent Cards (A2A) + OASF trust attributes (AGNTCY) + signed SKILL.md (marketplace) — three signature systems; convergence ongoing.
- **Observability across protocols.** HIR-shape uniform logging not yet a standard; trace_id propagation requires care.
- **Versioning and back-compat.** Each layer evolves independently; stack-wide back-compat testing absent.
- **Permission scope vocabularies differ.** OAuth scopes per layer use different naming conventions.
- **Federation gaps.** Cross-marketplace federation (Smithery + LobeHub + AGNTCY) is informal.
- **Cost economics fragmented.** Each layer has its own cost model; no unified per-task cost view.
- **Distillation between layers (e.g., A2A → MCP) erodes semantics.** If you implement an A2A agent on MCP, callers see only the tool surface, losing task-stream semantics.
- **Vendor lock-in via custom layer extensions.** Each marketplace adds metadata; portability erodes if leveraged.

## When to use, when not

**Adopt the full six-layer stack** when building a production agent platform with cross-vendor / multi-tenant / distributed / marketplace-publishing requirements. The strongest cases are **enterprise agent platforms**, **multi-team research orgs running heterogeneous agent runtimes**, and **published agent specialists** consumed externally.

**Adopt subset layers** when: solo agent on one host (L1 + L5 only); single-runtime team (L1 + L5 + L6); distributed personal agents (L1 + L4 + L5 + L6); cross-runtime ecosystem (L1 + L2 + L3 + L5 + L6).

**Skip the stack entirely** when prototyping; when the agent is one-off; when no external integration is planned.

## Implications for harness engineering (the consensus 2026 stack made concrete)

- **One shared `harness_core/protocols/` package.** All six layers; every agent in `projects/` imports it.
- **MCP-server library: `harness_core/mcp_servers/`.** Tools exposed uniformly across agents.
- **A2A endpoint per agent.** [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md) — single entry point for cross-runtime invocation.
- **OASF manifest published per agent.** [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md), [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md) — verified identity + trust attributes.
- **AGNTCY-shape internal registry.** Discovery for the 14 agents in `projects/` and any external A2A endpoints.
- **Tailscale tailnet across all dev/prod hosts.** [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md) — encrypted L3 mesh.
- **NATS hub on shared $5 VPS.** Subject namespace `harness.<runtime>.<host>.<domain>` — uniform across agents.
- **SKILL.md as the specialist format.** [04-skills](04-skills.md), [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md) — internal marketplace + selective external consumption.
- **Routines + Agent Teams as invocation primitives.** [250-anthropic-agent-teams](250-anthropic-agent-teams.md), [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md) — fire-from-anywhere + parallel teams.
- **HIR observability with trace_id propagation.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — uniform event shape across protocols.
- **Permission Bridge integrates with OAuth 2.1 scopes.** [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — bridge action kinds map to OAuth scopes.
- **Cost router with per-protocol metadata.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md), [88-confidence-driven-router](88-confidence-driven-router.md).
- **Cross-channel verifier as A2A specialist.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md) — verifier published, signed, discoverable.
- **Worktree isolation per A2A task.** [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md) — blast-radius bounded.
- **Memory tiering per agent, with cross-host sync via NATS.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md) — JetStream durable replay across hosts.
- **Daemon-driven schedule across the stack.** [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [p7-daemon](../projects/polaris/docs/research/p7-daemon.md) — one daemon, all six layers.

**One-line takeaway for harness designers.** **The 2026 agent protocol stack is six orthogonal layers — MCP for tools, A2A for inter-agent, AGNTCY (OASF + ACP + SLIM) for schema-and-discovery, Tailscale + NATS for distributed transport, SKILL.md + marketplaces for portable specialists, Routines + Agent Teams for invocation — composed without overlap, each adoptable independently, together giving the field its first production-grade agent infrastructure stack; the high-leverage move is one shared `harness_core/protocols/` package implementing all six layers, and every agent in `projects/` adopting the subset its use case requires.**
