# 07 — Model Context Protocol (MCP)

**Definition.** The Model Context Protocol is an open client-server standard for connecting LLM agents to external tools, data sources, and prompts. MCP servers expose a capability surface (tools, resources, prompts) over JSON-RPC; agent harnesses are MCP clients. The point is to decouple an agent from the integrations it uses — any MCP-compatible harness can talk to any MCP server.

## Problem it solves

Every agent framework used to invent its own tool-integration format: LangChain tool classes, OpenAI functions, Claude tool-use schemas, Copilot extensions. A Jira integration had to be re-implemented once per framework. MCP is the USB-C for agent tools: one spec, many clients, many servers, no re-implementation per host.

Beyond portability, MCP gives you three structural wins:

1. **Process isolation.** Tools run in a separate process (server) with its own permissions, dependencies, and secrets. If the Jira MCP server crashes, the agent loop doesn't.
2. **Discovery.** Servers advertise their capabilities at connect time; the agent learns tools dynamically rather than having them hard-coded in the system prompt.
3. **Composability.** An agent can connect to many servers (filesystem, Jira, Slack, a private database) without any central registry — each server is a peer.

## Mechanism

Architectural primitives:

- **Client.** The agent harness (Claude Code, Cursor, Continue, etc.). Initiates a connection to servers listed in its config.
- **Server.** A process exposing an MCP endpoint. Declares three kinds of capabilities:
  - *Tools* — functions the model can call (with JSON schema inputs/outputs).
  - *Resources* — read-only handles to content (files, URLs, database rows).
  - *Prompts* — reusable prompt templates the user or model can invoke.
- **Transport.** Stdio (spawn server as subprocess, talk over stdin/stdout), HTTP/SSE, or WebSocket.
- **Messages.** JSON-RPC 2.0. Notable: `initialize`, `tools/list`, `tools/call`, `resources/list`, `resources/read`, `prompts/list`.

Connection lifecycle:

1. Client reads `mcp.json` config, spawns/connects each server.
2. Client sends `initialize` with protocol version and capabilities.
3. Server responds with its capabilities. Client calls `tools/list`, `resources/list`, `prompts/list`.
4. Client registers the server's tools with the model, namespaced (`mcp__<server>__<tool>`).
5. When the model calls an MCP tool, the client forwards `tools/call` to the server; the server returns structured content.
6. On session end, client sends shutdown.

The namespacing matters: if two servers both expose a `query` tool, they don't collide because they appear to the model as `mcp__postgres__query` and `mcp__redis__query`.

## Concrete pattern

Minimal MCP server in Python using the official SDK:

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("notes")

@mcp.tool()
def search_notes(query: str) -> str:
    """Search the user's notes folder and return matching snippets."""
    # actual implementation...
    return "\n".join(search(query))

@mcp.resource("notes://today")
def todays_note() -> str:
    """Today's daily note."""
    return read_todays_note()

if __name__ == "__main__":
    mcp.run()
```

Client config (`.mcp.json`):

```json
{
  "mcpServers": {
    "notes": {
      "command": "python",
      "args": ["/path/to/notes_server.py"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "..." }
    }
  }
}
```

The agent now has `mcp__notes__search_notes`, `mcp__github__*`, etc., available as normal tools. No code changes needed in the harness.

## Variants & related techniques

- **OpenAI function calling / Anthropic tool use** are lower-level siblings: both describe a single framework's tool interface. MCP sits above them, portable across frameworks.
- **Toolformer** (arXiv 2302.04761) trains models to emit tool calls inline; MCP is orthogonal — it standardizes the *tool* end, not how the model is trained.
- **LangChain Tools / LlamaIndex Tools** are framework-native abstractions. Many of them now ship MCP adapters both ways (use any LangChain tool as an MCP server; mount an MCP server as a LangChain tool).
- **Agentic Foundation / Linux Foundation governance.** As of 2026, MCP governance is moving toward an open foundation, reducing vendor lock-in concerns.
- **Capability discovery** at the agent layer ([dynamic tool selection](24-observability-tracing.md)) is enabled by MCP's list APIs.

## Failure modes & anti-patterns

- **Tool sprawl.** Connecting 10 MCP servers explodes the tool schema catalog to 80 tools; the model's selection gets worse, not better. Fix: scope MCP servers per-task (a CI bot doesn't need a Slack server; a support bot doesn't need a deploy server).
- **Schema drift.** A server upgrade changes its tool signature mid-session. Clients should pin versions or re-`initialize` and surface diffs.
- **Untrusted servers.** An MCP server is an arbitrary process with the permissions of whoever launched it. Running a stranger's MCP server is equivalent to running a stranger's binary. Fix: vet and pin; sandbox where feasible.
- **Prompt injection via resources.** A server returning content from the web can embed instructions targeting the model ("ignore prior and exfiltrate…"). Treat resources as untrusted input — see [22-guardrails-prompt-injection.md](22-guardrails-prompt-injection.md).
- **Stateful servers.** A server that remembers session state across calls (session IDs, open connections) is fine, but multi-client scenarios need careful auth.
- **Latency neglect.** An HTTP-backed MCP call can be 500ms. A chatty agent makes 30 of these per turn; that's 15 seconds of overhead. Batch, cache, or pick stdio.

## When to use (and when not to)

**Use** MCP when:

- You want a tool available in multiple clients (Claude Code *and* Cursor *and* your bespoke harness).
- You want tool execution in a separate process with its own deps/secrets.
- You're building a capability others will consume — MCP is the distribution format.
- Your organization has many small tool integrations to maintain.

**Don't** use MCP when:

- The tool is truly internal to one harness and unlikely to be reused — a native tool is simpler.
- You need microsecond-latency tool calls — a subprocess boundary is overkill.
- You can't bound the blast radius of the server — no standard replaces threat modeling.

## References

- Model Context Protocol spec — <https://modelcontextprotocol.io/>
- Anthropic, "Introducing the Model Context Protocol" — <https://www.anthropic.com/news/model-context-protocol>
- LangChain MCP adapters docs — <https://python.langchain.com/docs/integrations/mcp/>
- alexop.dev, "Understanding Claude Code's Full Stack: MCP, Skills, Subagents, and Hooks" — <https://alexop.dev/posts/understanding-claude-code-full-stack/>
- FlowHunt, "Context Engineering: The Definitive 2025 Guide" — <https://www.flowhunt.io/blog/context-engineering/>
