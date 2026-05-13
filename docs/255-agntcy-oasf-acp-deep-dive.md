# 255 — AGNTCY / OASF / ACP: Cisco-led discovery + schema layer for the agent ecosystem

**Anchor.** AGNTCY (https://docs.agntcy.org/, GitHub org https://github.com/agntcy — 46 repos) — initiated by Cisco in 2024, donated to the Linux Foundation in 2025. Three composable layers: **OASF** (Open Agent Schema Framework — describes agent capabilities), **ACP** (Agent Connect Protocol — discovery + basic comms), **SLIM** (Secure Low-latency Interactive Messaging — embedded transport). Companion: [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md), [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md), [258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md).

**One-line definition.** A three-layer Cisco-led standard, donated to the Linux Foundation, that **complements A2A** by providing the **discovery and schema layers A2A deliberately does not own** — OASF defines a schema for describing agent capabilities (the agent equivalent of OpenAPI's machine-readable interface descriptors), ACP defines a REST-shape protocol for agent registration and basic invocation (the lightweight alternative when A2A's full task-stream model is overkill), and SLIM provides an embedded mTLS transport for the latency-critical case — together giving the agent ecosystem the **registry / catalog / discovery infrastructure** that lets cross-vendor agents *find each other* before they invoke each other.

## Why this paper matters (A2A is the wire format; AGNTCY is the catalog)

A2A ([254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md)) standardizes how agents exchange tasks and artifacts but deliberately leaves discovery to "static `.well-known` files or registry services." That gap is what AGNTCY fills. The Cisco-led project (donated to LF in 2025) is the agent-ecosystem equivalent of DNS + Container Registry + OpenAPI + npm — discovery, schema, and a lightweight transport — and is positioned as **complementary to A2A**, not competitive. The 2026 production stack composes naturally: OASF describes what an agent can do; ACP or A2A is the wire to invoke it; AGNTCY's discovery service is where you find it; AGNTCY trust tiering says how much to believe the manifest.

The three layers have distinct responsibilities and are usable independently. **OASF (Open Agent Schema Framework)** is a JSON-Schema-shape vocabulary for describing agent capabilities — input/output types, expected duration, cost-per-call estimates, supported languages, accepted authentication schemes, dependency requirements (this agent needs Python ≥3.11), and trust attributes (this agent has been audited by X). It is adoptable in isolation: even if you use A2A as your wire, you can publish an OASF document instead of the bespoke Agent-Card-format that A2A v1.0 defines. **ACP (Agent Connect Protocol)** is a lightweight REST-shape API for agent registration ("here is my OASF manifest"), discovery ("find agents matching this capability spec"), and basic invocation ("run this agent on this input"). It's positioned as the entry-level protocol — when A2A's full task-stream + Signed-Agent-Card + OAuth 2.1 stack is overkill (e.g., an internal agent registry where transport and trust are handled by the surrounding network), ACP gets you to working in less code. **SLIM (Secure Low-latency Interactive Messaging)** is the optional embedded transport for sub-millisecond agent-to-agent calls in the same data center.

The strategic position is "we're not competing with A2A; we're filling the gaps A2A doesn't try to fill." This is rare in standards politics and important: the field doesn't need another wire format; it needs the *registry and schema* infrastructure A2A explicitly punted on. AGNTCY's bet is that **OASF becomes the schema language** even when A2A is the wire — i.e., your A2A endpoint serves an OASF-shaped Agent Card, and the discovery layer that makes it findable is AGNTCY-shape.

Take this seriously and three things change. **First**, the **registry/catalog problem** has a credible standardization path — agent ecosystems no longer need bespoke discovery for every deployment. **Second**, the **schema layer separates from the wire layer** — you can adopt OASF for capability descriptions independently of whether you use A2A or ACP for invocation. **Third**, **trust tiering** ([p24-trust-tiering-retractions](../projects/polaris/docs/research/p24-trust-tiering-retractions.md)) becomes infrastructure-supported via AGNTCY's planned trust-attribute extensions to OASF.

## Problem it solves (the gaps A2A leaves and the smaller-than-A2A use cases)

1. **No standard agent capability schema.** A2A's Agent Card is one shape; vendor-specific shapes proliferated. OASF aims to be the field's converged schema vocabulary.
2. **No standard agent registry / discovery API.** Where do you publish your agent? How do consumers find it? AGNTCY's discovery layer fills this.
3. **A2A is heavy for trivial cases.** Internal-only agent registries don't need OAuth 2.1 + Signed Agent Cards + task streams. ACP is the simpler RPC for those cases.
4. **Trust tiering has no infrastructure.** OASF extensions plan to encode trust attributes (audited-by, vulnerability-scanned, signed-by-vendor, runtime-isolated).
5. **Sub-millisecond agent-to-agent in same-DC.** A2A's HTTPS overhead is too much for high-frequency intra-DC calls; SLIM provides an embedded mTLS transport.
6. **Schema portability.** OASF documents are usable across A2A, ACP, MCP-server descriptions, and custom protocols; one schema vocabulary, multiple wires.

## Core idea in one paragraph

AGNTCY decomposes the agent-ecosystem standard into three independent layers. **OASF (Open Agent Schema Framework)** is a JSON-Schema-shape vocabulary describing agent capabilities — each capability has input schema, output schema, expected duration, cost estimate, auth schemes, runtime requirements, and trust attributes — published as `agent.oasf.json` and signed with the same EdDSA primitives A2A uses. **ACP (Agent Connect Protocol)** is a lightweight REST API: `POST /v1/agents/register` (publish OASF manifest), `GET /v1/agents/search?capability=code-review&trust_tier=audited` (discover), `POST /v1/agents/{id}/invoke` (run with input, get output) — a stateless request-response shape suitable for internal registries and simple invocations. When A2A's full task-stream model is needed, ACP can wrap A2A: the registry returns an A2A endpoint URL, caller switches transports. **SLIM (Secure Low-latency Interactive Messaging)** is an embedded mTLS-encrypted UDP/QUIC transport for sub-millisecond intra-DC agent calls — a niche but documented use case (high-frequency monitoring agents, real-time decision agents). The three layers are independent: you can adopt OASF without ACP, ACP without OASF (using your own schema), or all three. The strategic positioning is **complementary to A2A**: same trust primitives (EdDSA signatures), same auth model (OAuth 2.1 compatible), but adding the discovery and schema layers A2A intentionally left to other standards. The 2026 production architecture for cross-vendor agent ecosystems composes them: OASF for schemas, AGNTCY discovery for registry, A2A for stateful tasks, ACP for stateless RPC, SLIM for intra-DC, MCP for tool calls. Each layer has its own scope and they don't fight.

## Mechanism (step by step)

### (a) OASF schema example

```json
{
  "oasf_version": "0.3",
  "agent_id": "polaris-research",
  "vendor": "polaris-contributors",
  "version": "2.4.1",
  "capabilities": [
    {
      "id": "literature-review",
      "description": "Systematic literature review for a research topic",
      "input": {
        "type": "object",
        "properties": {
          "topic": {"type": "string"},
          "depth": {"enum": ["light", "standard", "deep"]}
        },
        "required": ["topic"]
      },
      "output": {
        "artifacts": [
          {"content_type": "application/markdown", "max_size_bytes": 1048576},
          {"content_type": "application/json", "schema": {"$ref": "schemas/citations.json"}}
        ]
      },
      "expected_duration_seconds": 1800,
      "cost_estimate": {
        "currency": "USD",
        "min_per_call": 0.50,
        "max_per_call": 2.00
      },
      "supported_languages": ["en", "ja", "zh"],
      "auth_schemes": ["oauth2.1", "bearer"],
      "runtime_requirements": {
        "min_python_version": "3.11",
        "required_mcp_servers": []
      },
      "trust_attributes": {
        "audited_by": ["lyra-security-team"],
        "audit_date": "2026-03-15",
        "vulnerability_scan_passing": true,
        "signed_by": "polaris-prod-2026",
        "runtime_isolation": "worktree"
      }
    }
  ],
  "signature": {
    "algorithm": "EdDSA",
    "value": "...",
    "key_id": "polaris-prod-2026"
  }
}
```

The `trust_attributes` block is the critical OASF extension that A2A doesn't have natively.

### (b) ACP — Agent Connect Protocol

A simple REST API:

```
POST /v1/agents/register
{ "oasf_manifest": { ... }, "endpoint": "https://polaris.example.com/agents/research" }
→ 201 Created { "agent_id": "polaris-research" }

GET /v1/agents/search?capability=literature-review&trust_tier=audited
→ 200 OK { "agents": [{"agent_id": "polaris-research", "manifest_url": "...", "endpoint": "..."}] }

POST /v1/agents/polaris-research/invoke
Headers: Authorization: Bearer <token>
Body: { "capability": "literature-review", "input": { "topic": "agent scaling laws" } }
→ 200 OK { "result_id": "...", "artifacts": [...] }

GET /v1/agents/polaris-research/results/{result_id}
→ 200 OK { ... }
```

Stateless polling instead of A2A's streaming. Suitable for short-lived calls.

### (c) Bridge to A2A

When a discovered agent supports A2A, ACP returns the A2A endpoint URL. Caller submits via A2A for full task-stream semantics; uses ACP for stateless calls.

```python
# ACP discovery
result = acp.search(capability="literature-review", trust_tier="audited")
agent = result.agents[0]

if "a2a" in agent.supported_protocols:
    # Use A2A for streaming
    task = a2a.submit_task(agent.endpoints["a2a"], capability="literature-review", input=...)
    async for update in a2a.stream(task):
        ...
else:
    # Fall back to ACP RPC
    result = acp.invoke(agent.agent_id, capability="literature-review", input=...)
```

### (d) AGNTCY discovery service

The AGNTCY registry is a hosted (or self-hosted) service exposing the ACP search API across many agents:

```
GET https://registry.agntcy.org/v1/agents/search?
    capability=literature-review&
    vendor=polaris-contributors&
    min_trust_tier=audited&
    runtime=python3.11
```

Self-hosted via the AGNTCY reference implementation; cloud-hosted by Cisco's free service or AGNTCY-member providers.

### (e) Trust tiering

OASF defines a vocabulary for trust attributes; AGNTCY discovery lets consumers filter by trust tier:

| Tier | Criteria |
|---|---|
| `unverified` | No audit, no signature |
| `signed` | EdDSA-signed by vendor |
| `audited` | Third-party security audit within last 12 months |
| `runtime-isolated` | Runs in worktree / sandbox / container |
| `verified-vendor` | Vendor identity attested by registry |

Consumers reject below their threshold. Trust tiers compose with [p24-trust-tiering-retractions](../projects/polaris/docs/research/p24-trust-tiering-retractions.md) — retraction events propagate through the registry.

### (f) SLIM — embedded transport

For intra-DC sub-millisecond calls. mTLS over QUIC. Not relevant for personal-agent use cases; documented for completeness.

```
[agent A] ─SLIM─> [agent B]   # same DC, mTLS, sub-1ms
       \
        ─A2A─>     [agent C]   # cross-vendor, HTTPS
```

### (g) Composition with the rest of the stack

The 2026 stack:

| Layer | Tool | Role |
|---|---|---|
| Wire (cross-vendor) | A2A | Stateful tasks, signed cards, OAuth 2.1 |
| Wire (lightweight) | ACP | Stateless RPC, simple discovery |
| Wire (intra-DC) | SLIM | Sub-millisecond mTLS |
| Schema | OASF | Capability descriptions, trust attributes |
| Discovery | AGNTCY registry | Find agents by capability |
| Tool exposure | MCP | LLM-to-tool calls (different problem) |
| Transport (NAT-traversed) | Tailscale | L3 mesh ([253](253-tailscale-nats-mesh-for-distributed-agents.md)) |
| Distributed pub/sub | NATS leaf | L7 message bus ([253](253-tailscale-nats-mesh-for-distributed-agents.md)) |

## Empirical results (table — AGNTCY ecosystem May 2026)

| Metric | Value |
|---|---:|
| Member organizations | Cisco, founding partners + LF members |
| GitHub repos in agntcy org | 46 |
| OASF version (May 2026) | 0.3 (pre-1.0) |
| ACP version | 0.2 |
| SLIM availability | Reference implementation |
| Reference impl languages | Go, Python, JavaScript |

| Component | What it competes / cooperates with |
|---|---|
| OASF | Cooperates with A2A Agent Card (OASF can serve as the Card schema) |
| ACP | Lightweight alternative to A2A for simple RPC |
| SLIM | Niche; competes with custom mTLS RPC, not A2A |
| AGNTCY discovery | Fills A2A's discovery gap |

## Variants and ablations

- **OASF without ACP.** Use OASF as your schema, A2A as your wire. Common 2026 pattern.
- **ACP without AGNTCY registry.** Run your own ACP-compliant registry inside your org.
- **OASF trust attributes via cosign / sigstore.** EdDSA via OASF or PKI via cosign — both work.
- **MCP-server descriptions in OASF.** Not core, but emerging — OASF can describe MCP tools too.
- **Federated registries.** Multiple AGNTCY registries discover each other via standard catalog query.
- **SLIM bypass.** When A2A latency is acceptable, skip SLIM entirely.

## Failure modes and limitations

- **OASF version churn (0.3 in May 2026).** Pre-1.0 schema may break compatibility; pin versions.
- **Discovery registry SPOF.** Centralized registry → single point of failure; mitigate via federation.
- **Trust tier inflation.** Vendors self-attest; without third-party verifiers, tiers become marketing.
- **OASF and A2A Agent Card schema overlap.** Two schemas for similar concepts; convergence in progress.
- **ACP under-spec'd for streaming.** When tasks need progress, must escalate to A2A.
- **SLIM adoption nascent.** Few production deployments; relevance niche.
- **Anthropic Agent Teams not natively integrated.** Community ports only.
- **Permission scopes vendor-specific.** OAuth 2.1 standard but scope vocabularies still differ.
- **Catalog schemas not standardized for tools (vs. agents).** OASF describes agents; tool descriptions still MCP-shape.

## When to use, when not

**Adopt AGNTCY** when you publish agents to a multi-vendor ecosystem and need discoverability ([257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md)); when trust tiering matters (security-sensitive deployments); when internal agent registries need a standard schema (OASF); when ACP's lightweight RPC fits better than A2A's full stack. The strongest case is **enterprise agent registries** where trust tiering and discovery are first-class.

**Skip AGNTCY** when all agents are in one runtime (in-process); when A2A discovery via static `.well-known` is sufficient; when no registry is needed; when the schema layer is already MCP-server-only (different scope); when adoption overhead exceeds project scope.

## Implications for harness engineering

- **Publish OASF manifests for every agent in `projects/`.** [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md) — OASF as Agent Card schema; uniform across runtimes.
- **Run an internal AGNTCY-shaped registry.** Even if not federated, internal discovery is high-ROI.
- **Trust tiering integrated with permission bridge.** [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — refuse calls to agents below threshold trust tier.
- **OASF + Skills marketplace alignment.** [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md) — OASF schemas as marketplace listings.
- **OASF trust attributes encode audit history.** [p24-trust-tiering-retractions](../projects/polaris/docs/research/p24-trust-tiering-retractions.md) — retraction events update OASF manifests.
- **ACP for internal cross-agent RPC.** [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md), [187-multi-agent-shared-memory-landscape](187-multi-agent-shared-memory-landscape.md).
- **A2A for cross-runtime stateful tasks.** [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md).
- **MCP for tool calls (orthogonal).** [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md), [07-model-context-protocol](07-model-context-protocol.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md).
- **Discovery + Routines.** [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md) — routine discovers a target agent via AGNTCY before firing.
- **Distributed deployments via Tailscale + NATS.** [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md) — registry traffic over the same mesh.
- **OASF in HIR observability.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — log capability used + trust tier per call.
- **Cost router using OASF cost estimates.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md) — budget-aware routing across providers.
- **Cross-channel verifier from OASF directory.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [19-verifier-cross-channel](../projects/polaris/docs/blocks/19-verifier-cross-channel.md) — discover audited verifier of different family.

**One-line takeaway for harness designers.** **AGNTCY fills the gaps A2A intentionally leaves — OASF for schemas, ACP for lightweight RPC, AGNTCY discovery for registry, SLIM for intra-DC — and the 2026 stack composes naturally: OASF as your capability schema (even on A2A wire), AGNTCY-shape registry for discovery, A2A for stateful tasks, MCP for tool calls; trust tiering as a first-class registry filter is the architectural primitive that finally makes "find an agent for X" production-grade.**
