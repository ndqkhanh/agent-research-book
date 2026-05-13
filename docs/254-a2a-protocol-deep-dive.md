# 254 — A2A Protocol: the Linux Foundation cross-vendor agent-to-agent standard

**Anchor.** Google → Linux Foundation — *Agent2Agent (A2A) Protocol* — https://a2a-protocol.org/latest/ — initial announcement Apr 2024 (Google Developers Blog, *A2A: A new era of agent interoperability*); donated to the Linux Foundation in 2025; v1.0 stabilized early 2026 with **Signed Agent Cards** for verified agent identity. Repository: https://github.com/a2aproject/A2A (~22k+ stars). Companion: [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md), [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md), [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md), [258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md).

**One-line definition.** An open, vendor-neutral protocol for **agent-to-agent semantic communication** — agents exchange opaque **tasks**, **messages**, and **artifacts** under **signed Agent Cards** (cryptographically-verified capability + identity manifests) over HTTPS with OAuth 2.1, **task-stream semantics for long-lived work**, and a deliberate decoupling from the underlying tool-call protocol (MCP) — adopted by **150+ organizations** as of early 2026 with deployments on AWS, Azure, and GCP, becoming the de-facto standard for crossing vendor boundaries (Anthropic agent ↔ Google ADK agent ↔ CrewAI agent ↔ in-house custom agent).

## Why this paper matters (the vendor-boundary problem finally has a standard)

Multi-agent runtimes proliferated in 2023–2025 (CrewAI, AutoGen, LangGraph, MetaGPT, OpenHands, Anthropic Agent Teams, Google ADK, custom in-house) and the natural assumption was that "the protocol war" would settle on whoever's runtime won. It didn't. By mid-2025 the field accepted that **multiple runtimes will coexist** — different teams pick different stacks for different reasons (latency, role specialization, evaluation tooling) — and the *cross-vendor* problem was unsolved. Anthropic agents could not natively talk to Google ADK agents. CrewAI couldn't invoke an OpenHands SWE team. Custom in-house agents had to write per-vendor adapters for every external runtime they integrated with. **A2A is the standard that makes those bridges native rather than bespoke.**

The protocol's specific choices are informative. **Agents are opaque** — A2A does not standardize how an agent reasons, what model it uses, what its memory looks like, or how it executes tools. It standardizes only **the messages between agents**: a request to perform a task, an artifact returned, a streaming update, an authentication challenge. **Agent Cards** (the discovery primitive) are signed JSON documents that describe an agent's capabilities (`["code-review", "doc-generation"]`), endpoints, auth methods, version, and identity. This is the same shape as OpenAPI for REST APIs — a manifest you can publish, sign, and verify. The signature (added in v1.0) closes the obvious supply-chain attack: a malicious agent claiming to be your security-reviewer.

What A2A is **not** is also load-bearing. A2A is *not* MCP. MCP is for an LLM-agent to invoke tools provided by a server; A2A is for one agent to invoke another agent. The two protocols are **orthogonal and composable** — an A2A agent can invoke MCP tools internally; an A2A endpoint can be backed by an agent that uses MCP for its own tool access. The split is deliberate and is the right shape: **tool calls** (MCP) and **agent calls** (A2A) have different semantic and trust surfaces, and conflating them produces cross-cutting failures.

Take A2A seriously and three things change. **First**, you stop writing per-vendor adapters and start exposing your agents via a standard A2A endpoint — one endpoint, N consumers, no per-pair integration. **Second**, the **cross-runtime team** ([251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md)) becomes a first-class architectural option: a Polaris research orchestrator can hire a Google ADK browser agent, an Anthropic SWE-team teammate, and a custom-trained verifier without writing connector code per pair. **Third**, **Signed Agent Cards** become the basis for an agent marketplace ([257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md)) where agent capabilities are published, signed, discoverable, and trust-tiered — the long-promised "find an agent for X" experience.

## Problem it solves (cross-vendor agent comms without per-pair integration code)

1. **No standard for agent ↔ agent calls.** REST + custom JSON shapes exist per vendor; nothing portable. Pre-A2A, every cross-vendor integration was bespoke.
2. **No verified agent identity.** "Find an agent that does X" required trusting the source you found it from. Signed Agent Cards add cryptographic identity.
3. **No task-stream semantics in HTTP.** Long-running agent tasks (research jobs, multi-hour coding tasks, scheduled workflows) need streaming progress; raw HTTP is one-shot.
4. **Authentication was vendor-specific.** OAuth flows differed per vendor, requiring per-vendor wiring. A2A standardizes on OAuth 2.1.
5. **Tool calls and agent calls were conflated.** Some implementations used MCP for both; others invented bespoke RPC. A2A's split with MCP clarifies the trust and semantic boundaries.
6. **Capability discovery was ad-hoc.** No standard manifest describing "this agent can do X, takes input shape Y, returns Z." Agent Cards fill this gap with OpenAPI-shape semantics.

## Core idea in one paragraph

A2A defines four primitive types that flow between agents over HTTPS: **Agent Cards** (signed JSON manifests describing capabilities, endpoints, auth, identity), **Tasks** (a unit of work an agent asks another agent to perform — request, lifecycle, artifacts, terminal status), **Messages** (intermediate updates inside a task — progress, partial outputs, clarifying questions), and **Artifacts** (typed outputs the receiving agent produces — files, structured data, summaries). Tasks are **stateful** with explicit lifecycle (`submitted → working → input-required → completed | failed | canceled`) and support **streaming** via Server-Sent Events for progress updates and `application/json` polls for stateless deployments. Authentication is **OAuth 2.1** with bearer tokens or signed JWT; agent identity is established via **Signed Agent Cards** (RS256 / EdDSA over the manifest) verified against published JWK sets. Discovery is via static `/.well-known/agent.json` endpoints, registry services (AGNTCY's discovery layer, [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md)), or programmatic catalog queries. The protocol is **transport-agnostic at the framework level** — it ships HTTPS as the default but A2A messages can ride on any reliable transport (WebSocket, gRPC, NATS) with the same semantics. It does **not** standardize what's inside an agent; it standardizes only the wire between agents. The composition with MCP ([256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md)) is clean: A2A is for inter-agent calls, MCP is for tool calls; an A2A-exposed agent can use MCP tools internally without leaking that detail to the caller.

## Mechanism (step by step)

### (a) Agent Card structure

Published at `/.well-known/agent.json` (or registry-served):

```json
{
  "agent_id": "polaris-research",
  "version": "2.4.1",
  "vendor": "polaris-contributors",
  "endpoints": {
    "submit_task": "https://polaris.example.com/a2a/v1/tasks",
    "stream": "https://polaris.example.com/a2a/v1/tasks/{task_id}/stream"
  },
  "auth": {
    "schemes": ["oauth2.1"],
    "oauth2_metadata_url": "https://polaris.example.com/.well-known/oauth-authorization-server"
  },
  "capabilities": [
    {
      "id": "literature-review",
      "input_schema": {"$ref": "schemas/lit-review-input.json"},
      "output_artifacts": ["application/markdown", "application/json"],
      "expected_duration_seconds": 1800,
      "supports_streaming": true
    }
  ],
  "signature": {
    "algorithm": "EdDSA",
    "value": "...",
    "key_id": "polaris-prod-2026"
  }
}
```

The `signature` block is verified against the vendor's JWK set; this is the v1.0 addition that closes the impersonation attack vector.

### (b) Task lifecycle

```
POST /a2a/v1/tasks
{
  "agent_id": "polaris-research",
  "capability_id": "literature-review",
  "input": { "topic": "agent scaling laws", "depth": "deep" },
  "callback_url": "https://caller.example.com/a2a/callback",
  "stream": true
}
→ 202 Accepted
{ "task_id": "tk_abc123", "status": "submitted" }

GET /a2a/v1/tasks/tk_abc123/stream  (SSE)
→ data: {"status": "working", "progress": 0.1}
  data: {"status": "working", "progress": 0.3, "message": "Found 47 relevant papers"}
  data: {"status": "input-required", "question": "Include preprints from 2025-2026 only or all years?"}

POST /a2a/v1/tasks/tk_abc123/input
{ "answer": "2025-2026 only" }
→ 200 OK
  data: {"status": "completed", "artifacts": ["art_xyz789"]}

GET /a2a/v1/tasks/tk_abc123/artifacts/art_xyz789
→ Content-Type: application/markdown
  # Literature Review: Agent Scaling Laws ...
```

States: `submitted` → `working` → (`input-required` ↔ `working`)* → `completed | failed | canceled`. Each transition is timestamped and audit-logged.

### (c) Authentication via OAuth 2.1

The caller obtains a bearer token via standard OAuth 2.1 client-credentials or device flow against the agent's published auth metadata. Tokens are scoped per-capability (`scope=literature-review:execute`). The Agent Card's `auth.oauth2_metadata_url` is RFC 8414 OAuth Authorization Server Metadata.

### (d) Signed Agent Card verification (v1.0)

```python
def verify_agent_card(card_json: dict, jwks: dict) -> bool:
    sig = card_json.pop("signature")
    canonical = canonicalize_json(card_json)  # RFC 8785
    public_key = jwks["keys"][sig["key_id"]]
    return verify_eddsa(canonical, sig["value"], public_key)
```

The vendor publishes their JWK set at `/.well-known/jwks.json`; rotation is supported by including multiple keys with `kid` discrimination.

### (e) Streaming via SSE

Long-lived tasks emit progress via Server-Sent Events. Stateless deployments can poll `GET /a2a/v1/tasks/{task_id}` instead — the protocol supports both with the same semantics (this dual-mode design is intentional for agents running behind serverless platforms).

### (f) Artifacts

Outputs are returned as **typed artifacts**, not embedded in the task response. Each artifact has a content type (`application/json`, `application/markdown`, `image/png`, custom MIME), a URL or inline data, and optional metadata. This separation enables large-result handling (10 GB code analysis, hour-long generated audio) without bloating task records.

### (g) Composition with MCP

A2A specifies what flows *between* agents. The receiving agent might internally use MCP to invoke tools while servicing the task. Caller doesn't see this; receiver controls its own tool access. Example: an A2A "code-review" agent receives a task, internally uses MCP-exposed `git`, `read_file`, and `lint` tools, returns a markdown artifact with findings.

### (h) Error semantics

Failures return structured errors:

```json
{
  "task_id": "tk_abc123",
  "status": "failed",
  "error": {
    "code": "capability_not_supported",
    "message": "...",
    "retryable": false,
    "trace_id": "..."
  }
}
```

Standard error codes: `capability_not_supported`, `auth_failed`, `quota_exceeded`, `input_invalid`, `internal_error`, `timeout`, `canceled_by_caller`.

## Empirical results (table — A2A ecosystem as of May 2026)

| Metric | Value |
|---|---:|
| Member organizations (Linux Foundation A2A) | 150+ |
| GitHub stars (a2aproject/A2A) | ~22 k+ |
| Major-cloud deployments | AWS, Azure, GCP |
| Spec version (May 2026) | v1.0 (with Signed Agent Cards) |
| Reference implementations | Python (a2a-python), TypeScript (a2a-js), Go (a2a-go), Java (a2a-java) |
| Sample agents in prod | Google ADK agents, Salesforce AgentForce, Microsoft Copilot Studio agents |

**Adopting runtimes:**

| Runtime | A2A status |
|---|---|
| Google ADK | Native A2A endpoints |
| CrewAI | A2A consumer + producer (community port) |
| LangGraph | A2A consumer (graph nodes can call A2A endpoints) |
| OpenHands | A2A producer (exposes the SWE team) |
| Anthropic Agent Teams | Not native; community A2A wrapper exists |
| MetaGPT | Community port |
| Custom Anthropic SDK agents | Trivial — wrap with `a2a-python` server |

## Variants and ablations

- **A2A over WebSocket / gRPC.** Default is HTTPS; community ports onto WebSocket for lower-latency interactive agents.
- **A2A over NATS.** [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md) — NATS as transport for A2A messages with subject patterns mapping to capabilities.
- **A2A with AGNTCY discovery.** [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md) — AGNTCY's registry layer fronts A2A; agents discoverable by capability query.
- **Embedded MCP.** Receiving agent uses MCP internally; transparent to caller.
- **A2A + Routines.** [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md) — fire a routine in agent A from agent B via A2A `submit_task`.
- **A2A + Agent Teams.** [250-anthropic-agent-teams](250-anthropic-agent-teams.md) — A2A endpoint per teammate enables cross-vendor team composition.
- **A2A + Skills marketplace.** [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md) — Signed Agent Cards as marketplace listings.

## Failure modes and limitations

- **Agent Cards out of sync with reality.** Card claims `code-review` capability but agent fails on first call; needs runtime versioning + capability tests.
- **Signature verification expensive on every call.** Cache JWK fetches; rotate carefully.
- **OAuth 2.1 setup overhead.** A1.0 → 1.0 upgrade requires rolling out OAuth infra; legacy bearer-token auth still permitted.
- **No standard for inter-agent observability.** Trace IDs propagate but no shared HIR-equivalent.
- **Long-task SSE connections fragile across NAT.** Stateless poll mode mitigates; both sides must support.
- **Capability schemas drift.** Without strict schema versioning, callers break on agent updates.
- **Trust tiers underspecified.** A2A doesn't define how to assign trust to a discovered agent; complemented by AGNTCY trust-tiering.
- **Cost model not standardized.** Caller can't query "how many tokens will this task cost"; needs vendor-specific extension.
- **Streaming semantics ambiguous on partial failures.** Task partially succeeded — what state should the SSE stream show?

## When to use, when not

**Adopt A2A** when your agent harness exposes capabilities consumed by external systems or other agents; when you want cross-vendor interop without per-pair adapters; when you ship in a multi-runtime environment (Anthropic teams + Google ADK + custom); when you need verified agent identity (Signed Cards) for trust-sensitive deployments. The strongest cases are **public agent marketplaces**, **multi-vendor enterprise deployments**, and **cross-team collaborations** where each team owns its own agent runtime.

**Skip A2A** when all your agents are in one runtime ([250-anthropic-agent-teams](250-anthropic-agent-teams.md)'s in-process teammates don't need A2A); when latency is sub-100ms-bound (HTTPS overhead matters); when no external integration is planned; when you're prototyping and the protocol overhead exceeds project scope.

## Implications for harness engineering

- **One A2A endpoint per agent in `projects/`.** Every agent runtime should expose A2A; one shared `harness_core/a2a_server/` package, all 14 agents enabled.
- **Signed Agent Cards as identity infrastructure.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — verified identity for verifier agents.
- **Tasks as the unit of cross-runtime work.** [222-agent-trajectory-scaling](222-agent-trajectory-scaling.md), [237-agent-trajectory-scaling](237-agent-trajectory-scaling.md), [195-argus-omega-vol-2-trajectory-temporal-horizon](195-argus-omega-vol-2-trajectory-temporal-horizon.md) — trajectory budget per A2A task.
- **Cross-vendor team composition.** [250-anthropic-agent-teams](250-anthropic-agent-teams.md), [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md) — A2A is the protocol that lets your Polaris orchestrate Google ADK + custom agents.
- **OAuth 2.1 for permission scope.** [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [06-permission-modes](06-permission-modes.md) — OAuth scopes map to bridge action kinds.
- **A2A + Routines.** [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md) — fire a routine via A2A from external systems.
- **A2A + NATS transport.** [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md) — A2A messages over NATS for distributed deployments.
- **Discovery layer with AGNTCY.** [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md) — registry of A2A agents queryable by capability.
- **MCP composition for tool access.** [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md) — internally use MCP, externally expose A2A.
- **Marketplace publishing.** [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md) — Signed Agent Cards as marketplace entries.
- **Cost-router across A2A providers.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md) — choose provider by latency / cost / capability match.
- **HIR observability of A2A traffic.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — trace_id propagation across vendors.
- **Bright-line gates on A2A submission.** [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — `A2A_SUBMIT` action kind for outbound; `A2A_RECEIVE` for inbound.

**One-line takeaway for harness designers.** **A2A is the cross-vendor agent-to-agent standard the field consolidated on by 2026 — Signed Agent Cards for identity, OAuth 2.1 for auth, task-stream semantics for long-lived work, transport-agnostic and orthogonal to MCP — and the architectural move every agent in `projects/` should make is "expose A2A, consume A2A," letting you compose teams and routines across runtime boundaries without per-pair adapter code.**
