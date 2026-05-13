# 256 — MCP 2025–2026 Evolution: from HTTP+SSE to Streamable HTTP to stateless servers

**Anchor.** Anthropic — *Model Context Protocol* — https://modelcontextprotocol.io/ — initial release Nov 2024. Spec evolution: `2024-11-05` (HTTP+SSE), `2025-03-26` (Streamable HTTP, the major refactor), `2025-09-XX` (auth refinements, OAuth 2.1 alignment), 2026 roadmap: stateless servers, official multi-tenant patterns. Existing canonical file: [07-model-context-protocol](07-model-context-protocol.md). Companions: [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md), [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md), [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md), [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md), [258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md).

> **Disambiguation.** [07-model-context-protocol](07-model-context-protocol.md) is the foundational MCP deep-dive (architecture, JSON-RPC, primitives — Tools, Resources, Prompts, Sampling). This file (256) is a *2025–2026 evolution* deep-dive focused on the **Streamable HTTP transition** and the architectural shifts that landed after the original.

**One-line definition.** A 2025–2026 evolution narrative for MCP — the protocol moved from **HTTP+SSE dual-endpoint transport** (`2024-11-05` spec) to **Streamable HTTP single-endpoint with optional SSE** (`2025-03-26` spec), aligned **auth on OAuth 2.1** (`2025-09` refinements), and the **2026 roadmap** explicitly pushes toward **stateless servers** (each call carries full context; trivial horizontal scaling) — collectively turning MCP from "the SDK pattern that works locally" into **production multi-tenant infrastructure** with 5,000+ public servers, native integrations from OpenAI Codex / Anthropic Claude / Google ADK / Microsoft Copilot Studio / GitHub, and a clear orthogonality with A2A ([254](254-a2a-protocol-deep-dive.md)) for inter-agent comms.

## Why this paper matters (the protocol that won the tool-exposure race, and how it grew up)

When Anthropic launched MCP in November 2024, the immediate critique was twofold: the **HTTP+SSE dual-endpoint design** was awkward for production deployment (two ports, two URL paths, complex load balancing), and **authentication** was vague ("OAuth-style" but not specified). The 2025–2026 evolution addressed both, and the protocol's adoption curve in that window — going from "interesting Anthropic-only spec" to **5,000+ public servers and native first-class support in every major agent runtime** — tracks closely with the spec maturity. The 2025-03-26 Streamable HTTP refactor and the 2025-09 OAuth 2.1 alignment are the two changes that flipped MCP from "SDK demo" to "production infrastructure."

The protocol's design choices, in retrospect, were the right ones. **Tool exposure is a separate problem from agent-to-agent comms** — MCP and A2A are orthogonal, not competitive. **JSON-RPC 2.0 is a deliberate not-REST decision** — it gives bidirectional notifications and request-response in one transport without REST's per-resource path proliferation. **Resources, Tools, Prompts, Sampling** as four primitives is a clean separation of capability surfaces — Resources are read-only data, Tools are callable functions with side effects, Prompts are reusable system-prompt templates, Sampling is the inverse direction (server requests LLM completion from client). The 2026 spec preserves all four.

What the 2025–2026 evolution added matters for production. **Streamable HTTP** collapsed the two endpoints into one (`POST /` with optional SSE upgrade for streaming) — load balancing trivial, deployment matches every existing HTTP API. **OAuth 2.1 alignment** pinned the auth model to RFC-grade primitives — bearer tokens, scopes, RFC 8414 metadata, dynamic client registration via RFC 7591. **Stateless server pattern** (in the 2026 roadmap) lets each request carry its full session state — no in-memory server state — making horizontal scaling trivial for high-traffic deployments.

Take this seriously and three things change. **First**, you stop deploying MCP servers as long-lived stateful processes ("the MCP server for our tools") and start deploying them as **stateless HTTP services** behind standard load balancers — same shape as any other API. **Second**, the **auth boundary** between agent runtime and MCP server is OAuth 2.1, the same as A2A, the same as the rest of the modern web — no MCP-specific auth dance. **Third**, the **MCP-marketplace economy** ([257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md)) becomes credible: a public registry of MCP servers (Smithery, official Anthropic registry, community catalogs) with verified identity and trust tiering, accessed via standard OAuth flows.

## Problem it solves (turning a tool-exposure SDK into production multi-tenant infrastructure)

1. **Two-endpoint dual-transport was awkward.** `2024-11-05` spec: `POST /` for requests, `GET /` for SSE notifications. Load balancers hated it; reverse-proxies broke it. Streamable HTTP (`2025-03-26`) collapsed to one endpoint.
2. **Authentication was unclear.** Original spec said "OAuth-like" without specifying flows. 2025-09 pinned OAuth 2.1 with RFC 8414 metadata; transitioned implementations from bespoke API keys to standard OAuth.
3. **Stateful servers don't horizontally scale.** Per-session in-memory state (active tool list, session capabilities) made multi-instance deployments hard. 2026 stateless-server pattern lets each request carry full state.
4. **Multi-tenancy was not a first-class concept.** Single-user MCP servers were the assumption. 2026 patterns explicitly support multi-tenant servers behind one endpoint with per-user OAuth scopes.
5. **Tool-call costs were unpredictable.** No native cost-estimate field; runtime had to guess. 2026 extensions add cost hints.
6. **Trust and identity for MCP servers.** Original spec had no signature; trust was "you trust the URL." 2026 alignment with OAuth 2.1 + signed server descriptions closes the gap.

## Core idea in one paragraph

The 2025–2026 evolution of MCP made three architectural shifts. **Transport** — `2024-11-05`'s HTTP+SSE dual endpoint became `2025-03-26`'s **Streamable HTTP**: a single `POST /mcp` endpoint that accepts JSON-RPC 2.0 requests and either returns a JSON response or upgrades to Server-Sent Events for streaming notifications. Load balancers handle it like any HTTPS API. **Auth** — vague "OAuth-like" became OAuth 2.1 with RFC 8414 Authorization Server Metadata for discovery, RFC 7591 dynamic client registration, and bearer-token scopes that map per-tool (`scope=tools.read_file:execute`). **Stateless servers** — the 2026 roadmap pushes the pattern where MCP servers hold no per-session state; each request carries its full context (active tool subset, session ID, user identity), and the server can scale horizontally behind a load balancer without sticky sessions. The four primitives (Tools, Resources, Prompts, Sampling) are unchanged — the protocol's *semantic model* is stable; what changed is the *deployment model*. Composition with A2A ([254](254-a2a-protocol-deep-dive.md)) is clean: MCP is for LLM-to-tool calls inside one agent's runtime; A2A is for one agent calling another agent. An A2A-exposed agent can use MCP tools internally; the caller doesn't see the MCP layer. The 2026 production stack is **MCP for tools, A2A for agents, AGNTCY/OASF ([255](255-agntcy-oasf-acp-deep-dive.md)) for schema and discovery, NATS+Tailscale ([253](253-tailscale-nats-mesh-for-distributed-agents.md)) for distributed transport** — four orthogonal layers, no overlap.

## Mechanism (step by step)

### (a) Streamable HTTP transport (`2025-03-26` spec)

Single endpoint, two response modes:

```
POST /mcp HTTP/1.1
Content-Type: application/json
Authorization: Bearer <oauth-token>

{
  "jsonrpc": "2.0",
  "id": "req-1",
  "method": "tools/call",
  "params": { "name": "read_file", "arguments": {"path": "src/main.py"} }
}

# Response mode 1: simple JSON
HTTP/1.1 200 OK
Content-Type: application/json
{
  "jsonrpc": "2.0",
  "id": "req-1",
  "result": { "content": [...] }
}

# Response mode 2: SSE for streaming
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"jsonrpc":"2.0","id":"req-1","method":"notifications/progress","params":{"progress":0.3}}
data: {"jsonrpc":"2.0","id":"req-1","result":{"content":[...]}}
```

The server picks the mode based on the request shape (notification subscription requires SSE, simple call returns JSON). Reverse proxies and load balancers see one endpoint, standard HTTP semantics.

### (b) OAuth 2.1 alignment (2025-09 refinements)

Server publishes RFC 8414 Authorization Server Metadata at `/.well-known/oauth-authorization-server`:

```json
{
  "issuer": "https://mcp-server.example.com",
  "authorization_endpoint": "...",
  "token_endpoint": "...",
  "registration_endpoint": "https://mcp-server.example.com/oauth/register",  // RFC 7591
  "scopes_supported": [
    "tools.read_file:execute",
    "tools.write_file:execute",
    "resources.repo:read"
  ]
}
```

Client registers dynamically (RFC 7591), obtains tokens, calls with `Authorization: Bearer <token>`. Scopes map to per-tool execute permissions; bridge classifies on tool name.

### (c) Stateless server pattern (2026 roadmap)

Each request carries the full session context:

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "read_file",
    "arguments": {...},
    "session_context": {
      "session_id": "...",
      "active_tools": ["read_file", "write_file"],
      "user_id": "...",
      "trace_id": "..."
    }
  }
}
```

Server has no in-memory state per session; scales horizontally behind a round-robin load balancer. Trade-off: each request payload is larger (carries full context); benefit: trivial deployment, no sticky-session config.

### (d) The four primitives (unchanged across spec revisions)

| Primitive | Direction | Use case |
|---|---|---|
| **Tools** | Client → Server | Callable functions with side effects (read_file, run_query, send_email) |
| **Resources** | Client → Server | Read-only data (file contents, DB rows) |
| **Prompts** | Client → Server | Reusable system-prompt templates the server provides |
| **Sampling** | Server → Client | Server asks client's LLM for a completion (inverse direction) |

The Sampling primitive is the architectural detail that distinguishes MCP from REST — the *server* can request LLM inference from the *client*'s model, enabling tool implementations that need LLM capabilities without bringing their own model.

### (e) Discovery and capabilities

Initial handshake (`initialize` method) negotiates capabilities:

```json
{
  "jsonrpc": "2.0",
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-03-26",
    "capabilities": {
      "tools": {},
      "resources": { "subscribe": true },
      "sampling": {}
    },
    "clientInfo": { "name": "claude-code", "version": "2.1.32" }
  }
}
```

Server responds with its supported capabilities; client uses only those.

### (f) Marketplaces and multi-tenant patterns

Production patterns by 2026:

- **Smithery** (https://smithery.ai/) — public MCP marketplace, signed servers, OAuth-mediated install.
- **Anthropic official registry** — curated MCP servers vetted by Anthropic.
- **Self-hosted MCP servers in `harness_core/mcp_servers/`** — vendored or built in-house.
- **Multi-tenant single-server** — one MCP endpoint serves N users via OAuth scopes; per-user data isolation via token scope.

### (g) Composition examples

**MCP only (single-agent runtime):** Claude Code uses MCP servers for git, fs, search, web. Standard local pattern.

**MCP + A2A:** A custom A2A-exposed "code-review" agent uses MCP internally for its tool calls. Caller invokes via A2A; agent uses MCP under the hood.

**MCP + Routines:** A scheduled routine invokes the agent loop, which has access to a curated set of MCP servers per the routine config ([252](252-routines-pattern-for-self-hosted-agents.md)).

**MCP over NATS:** When MCP servers are distributed across hosts, NATS as the transport layer ([253](253-tailscale-nats-mesh-for-distributed-agents.md)) — JSON-RPC over NATS subjects.

## Empirical results (table — MCP ecosystem May 2026)

| Metric | Value |
|---|---:|
| Spec versions shipped | 2024-11-05, 2025-03-26 (current), 2026 roadmap |
| Public MCP servers | 5,000+ |
| Native client integrations | Claude Code, Claude Desktop, OpenAI Codex, Google ADK, Microsoft Copilot Studio, GitHub Copilot, Cursor, Cline, Continue |
| Languages with reference servers | TypeScript, Python, Rust, Go, Java |
| Marketplaces | Smithery, official Anthropic registry, mcpregistry.org |

| Spec change | Date | Impact |
|---|---|---|
| Initial release | 2024-11-05 | Foundation, HTTP+SSE dual-endpoint |
| Streamable HTTP | 2025-03-26 | Single endpoint, easier deployment |
| OAuth 2.1 alignment | 2025-09 | Standard auth, dynamic client registration |
| Stateless servers (roadmap) | 2026 | Horizontal scaling, multi-tenant patterns |

## Variants and ablations

- **Local stdio MCP servers.** stdin/stdout transport for in-process subprocess servers; original use case, still supported.
- **MCP over WebSocket.** Community ports for low-latency interactive cases.
- **MCP over NATS subjects.** Distributed deployments where each tool is on a different host.
- **MCP server-bundling.** One server exposing many tools (default) vs many small servers (microservice-style).
- **MCP + Sampling.** Servers that need LLM access for tool implementation (a search server using LLM for query expansion).
- **Federated MCP.** Multiple MCP servers under one OAuth umbrella; client sees a unified tool catalog.
- **Skill-aware MCP.** [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md) — SKILL.md references MCP servers; tool dependencies declared.

## Failure modes and limitations

- **Stateful servers leak memory across sessions.** If you keep state, evict aggressively.
- **OAuth scope sprawl.** Servers with 50+ tools have 50+ scope strings; UX gets unwieldy.
- **Sampling primitive underused.** Most servers don't implement it; capability negotiates around.
- **Trust on first use.** Connecting a new MCP server is trust-on-first-use unless the marketplace verifies.
- **Tool collisions.** Multiple servers expose `read_file`; client must disambiguate.
- **Sub-second latency for chatty agents.** Each tool call is HTTPS round-trip; matters for high-frequency loops.
- **MCP doesn't solve cross-agent comms.** Tool calls are not agent calls — A2A required for that ([254](254-a2a-protocol-deep-dive.md)).
- **Spec churn cost.** Implementations on 2024-11-05 spec needed migration to 2025-03-26.
- **Multi-tenant security non-trivial.** Per-user data isolation must be implemented per-server; protocol provides scope but not enforcement.
- **Resource subscription complexity.** Long-lived `notifications/resources/updated` over SSE has reconnect / replay complications.

## When to use, when not

**Use MCP** for **all tool exposure** in your agent runtime — file access, shell, web search, DB queries, custom tools. The protocol is the field's converged standard for agent-to-tool calls; not adopting it requires per-runtime adapter code. The strongest cases are **shared tool catalogs** (one MCP server used by Claude Code + Codex + Copilot), **community-published tools** (Smithery / marketplace), and **vendored internal tools** (your `harness_core/mcp_servers/` package).

**Don't use MCP** for **inter-agent comms** — that's A2A's domain ([254](254-a2a-protocol-deep-dive.md)); for **schema description** of agents — that's OASF ([255](255-agntcy-oasf-acp-deep-dive.md)); for **discovery** of agents — that's AGNTCY ([255](255-agntcy-oasf-acp-deep-dive.md)). Don't conflate; each layer has a job.

## Implications for harness engineering

- **MCP is the tool-exposure standard; adopt for every agent in `projects/`.** [07-model-context-protocol](07-model-context-protocol.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md), [11-domain-shell](../projects/polaris/docs/blocks/11-domain-shell.md) — domain shells expose tools via MCP.
- **Streamable HTTP is the production transport.** Single-endpoint deployment, standard load balancing.
- **OAuth 2.1 for auth.** [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — OAuth scopes map to bridge action kinds.
- **Stateless servers for horizontal scaling.** Production deployments, multi-tenant patterns.
- **MCP + A2A composition.** [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md) — A2A agent uses MCP internally; clean separation.
- **MCP server signatures & marketplaces.** [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md), [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md) — Smithery / official registry; verified servers.
- **MCP + Skills bundling.** [04-skills](04-skills.md), [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md) — SKILL.md declares MCP server deps.
- **Distributed MCP via NATS.** [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md) — JSON-RPC over NATS subjects when tools span hosts.
- **Permission bridge gates per tool.** [05-hooks](05-hooks.md), [08-hooks-and-claim-gate](../projects/polaris/docs/blocks/08-hooks-and-claim-gate.md) — pre-tool-use hooks redact / audit.
- **HIR observability of MCP traffic.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — log per-tool call with trace_id.
- **Cost router by MCP server.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md) — different MCP servers have different latency / cost.
- **Sampling primitive for skill auto-creation.** [10-skill-auto-creator](../projects/polaris/docs/blocks/10-skill-auto-creator.md) — server requests LLM completion for new skill draft.
- **Verifier as MCP server.** [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md) — verifier exposed as a tool.
- **Worktree-isolated MCP fs server.** [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md) — fs MCP scoped to worktree.

**One-line takeaway for harness designers.** **MCP is the converged tool-exposure standard for agents; the 2025–2026 evolution from HTTP+SSE to Streamable HTTP plus OAuth 2.1 plus the 2026 stateless-server roadmap turned it into production multi-tenant infrastructure with 5,000+ servers — adopt it for every tool surface in `projects/`, compose with A2A for inter-agent comms and AGNTCY/OASF for discovery, and treat the four-primitive semantic model (Tools / Resources / Prompts / Sampling) as stable while the deployment model continues to mature.**
