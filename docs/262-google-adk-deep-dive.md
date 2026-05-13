# 262 — Google ADK: multi-language code-first agent runtime, A2A-native from day one

**Anchor.** Google — *Agent Development Kit (ADK)* — https://github.com/google/adk-python, https://github.com/google/adk-js, https://github.com/google/adk-go, https://github.com/google/adk-java/. Code-first multi-language SDK released 2025; native **A2A integration** ([254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md)); composition primitives **Coordinator**, **Greeter**, **Worker**; Vertex AI Agent Engine for managed deployment. Companions: [259-langgraph-deep-dive](259-langgraph-deep-dive.md), [260-openai-agents-sdk-deep-dive](260-openai-agents-sdk-deep-dive.md), [261-autogen-v04-deep-dive](261-autogen-v04-deep-dive.md), [263-production-agent-runtime-synthesis-2026](263-production-agent-runtime-synthesis-2026.md), [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md).

**One-line definition.** Google's **code-first multi-language** agent runtime (Python, JavaScript, Go, Java reaching parity through 2025–2026) with **native A2A integration from day one** — Agent Cards, Signed Cards, OAuth 2.1, and task-stream semantics built into the runtime, not bolted on — built around three composition primitives (**Coordinator**, **Greeter**, **Worker**) that map cleanly to the lead-and-spokes / hierarchical pattern the field consolidated on; deployed via **Vertex AI Agent Engine** for managed cloud or self-hosted via the open-source SDK, and the natural pick for **enterprise / Google-stack / cross-vendor** deployments where multi-language and A2A compliance matter.

## Why this paper matters (Google's bet on enterprise + multi-language + cross-vendor protocols)

The agent runtime market in 2026 has clear vendor blocs: OpenAI's Agents SDK ([260](260-openai-agents-sdk-deep-dive.md)) for OpenAI-stack deployments, Anthropic's Claude Code + Agent Teams ([250](250-anthropic-agent-teams.md)) for Claude-stack engineering work, Microsoft's AutoGen v0.4 ([261](261-autogen-v04-deep-dive.md)) for research + Microsoft-stack, LangChain's LangGraph ([259](259-langgraph-deep-dive.md)) for production-tier-cross-vendor. Google's ADK occupies the **enterprise + cross-vendor** corner with three explicit bets: **multi-language parity** (Python + JS + Go + Java, not just Python), **A2A as the native protocol** (Google donated A2A to the Linux Foundation; their runtime is the reference implementation), and **Vertex AI Agent Engine** as the managed-cloud deployment story.

The multi-language bet matters for enterprise. JavaScript shops want JS-native agents; Go shops want Go-native agents; Java shops have been left out by Python-only runtimes. ADK ships across all four with feature parity as the explicit goal — by May 2026 Python is canonical, JS is at parity, Go is close, Java is ~80% there. Combined with .NET parity in AutoGen v0.4, this is the moment language-portability stops being a Python-versus-everyone-else gap and starts being a normal multi-language ecosystem.

The A2A-native bet is also strategic. Google's Agent2Agent protocol ([254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md)) was donated to the Linux Foundation in 2025 and reached 150+ member organizations by early 2026. ADK is the reference implementation — Agent Cards, Signed Cards (v1.0), OAuth 2.1, task-stream semantics are all native primitives, not optional add-ons. For deployments where cross-vendor agent comms matter (an ADK agent invoking an Anthropic Agent Teams teammate, a Claude Code routine triggering an ADK research agent), this native fit reduces integration friction substantially.

The composition primitives — **Coordinator** (orchestrator), **Greeter** (entry-point agent), **Worker** (specialist) — encode the lead-and-spokes pattern from [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md). They're not a research breakthrough; they're a productized version of the consensus 2026 architecture. Combined with Vertex AI Agent Engine for managed deployment (autoscaling, observability, identity), the package is the natural pick for an enterprise that wants **a single multi-language SDK**, **A2A compliance**, and **a managed cloud option**.

Take this seriously and three things change. **First**, **multi-language parity** becomes a real consideration in runtime selection. JavaScript / Go / Java agents are no longer second-class; the pattern library is the same across languages. **Second**, **A2A-native deployments** become trivial — your ADK agent is automatically discoverable via OASF + AGNTCY ([255](255-agntcy-oasf-acp-deep-dive.md)), invocable cross-vendor via A2A, with Signed Agent Cards working out of the box. **Third**, **Vertex AI Agent Engine** is the first credible managed-cloud agent platform from a hyperscaler (AWS Bedrock Agents and Azure AI Studio are competitors; Vertex Agent Engine is the most A2A-native of the three).

## Problem it solves (enterprise multi-language + A2A-compliant + managed-cloud)

1. **Python-only runtimes lock out multi-language stacks.** ADK's parity strategy fixes this for Google + many enterprise shops.
2. **A2A integration was always a bolt-on.** ADK is the first runtime where it's native primitive.
3. **Vertex AI ecosystem alignment.** Enterprise customers on Google Cloud want a runtime that integrates with Vertex AI, BigQuery, Cloud Run, IAM.
4. **Enterprise auth + identity.** OAuth 2.1, IAM-integrated, Workload Identity Federation; the auth model enterprise auditors expect.
5. **Coordinator/Greeter/Worker primitives.** Productized lead-and-spokes pattern that maps cleanly to the consensus architecture.
6. **Managed-cloud agent deployment.** Vertex AI Agent Engine handles autoscaling, observability, and identity; competitor to AWS Bedrock Agents and Azure AI Foundry.
7. **Cross-vendor interop without per-pair adapters.** ADK consumes any A2A-exposed agent (Anthropic, OpenAI, custom) and exposes itself via A2A — no per-vendor connector code.

## Core idea in one paragraph

A Google ADK application is built around three composition primitives. **`Coordinator`** is an orchestrator agent that manages a pool of specialists, decides routing, and synthesizes results — the lead-and-spokes lead. **`Greeter`** is an entry-point agent typically responsible for triage, intent classification, and authentication; it routes to the appropriate Coordinator or Worker. **`Worker`** is a specialist agent doing one thing well — search, code-gen, doc-write, verification. Composition is via configuration (YAML / Python class hierarchy / TypeScript object) not visual graphs. The runtime is **A2A-native**: every agent gets an Agent Card auto-generated from its OASF-shape capability declarations; Signed Cards via Cloud KMS; OAuth 2.1 via Google IAM or any OAuth 2.1 server; task-stream semantics for long-lived work via SSE; the agent is automatically a valid A2A endpoint. **Multi-language parity** is the design goal: the same Coordinator/Greeter/Worker pattern is expressible in Python (`adk-python`), JavaScript (`adk-js`), Go (`adk-go`), Java (`adk-java`) with shared OASF/A2A wire compatibility — a JS Coordinator can hire a Python Worker over A2A. **Tools** are exposed via MCP (cross-vendor) or Google's hosted toolkit (Vertex Search, Code Execution, Function Calling, Image Gen). **Memory** plugs into Vertex AI Vector Search or any external store. **Deployment** is either self-hosted via the SDK or managed via **Vertex AI Agent Engine** — autoscaling, native observability via Cloud Trace + Logging, IAM-integrated identity. The runtime is enterprise-shape: code-first (no visual builder unlike AutoGen Studio), production-native (Vertex Agent Engine), cross-vendor (A2A-first), and the natural pick for **Google-stack** deployments and **multi-language enterprise**.

## Mechanism (step by step)

### (a) Python ADK example

```python
from adk import Coordinator, Greeter, Worker, Tool, run

class SearchTool(Tool):
    name = "search"
    description = "Search the web"

    async def __call__(self, query: str) -> dict:
        ...

class ResearchWorker(Worker):
    name = "research"
    capabilities = ["literature-review", "fact-check"]
    tools = [SearchTool()]
    instructions = "You research topics deeply..."

class CodeWorker(Worker):
    name = "code"
    capabilities = ["code-gen", "code-review"]
    tools = [...]
    instructions = "You write and review code..."

class ResearchCoordinator(Coordinator):
    name = "research-coord"
    workers = [ResearchWorker(), CodeWorker()]
    instructions = "Coordinate research and code-gen tasks..."

class TriageGreeter(Greeter):
    name = "triage"
    coordinators = [ResearchCoordinator()]
    instructions = "Route users to the right coordinator..."

if __name__ == "__main__":
    run(TriageGreeter())  # auto-exposes A2A endpoint
```

The `run()` function starts an HTTPS server, exposes the A2A endpoint, registers Agent Cards, and sets up tracing.

### (b) JavaScript / TypeScript parity

```typescript
import { Coordinator, Greeter, Worker, Tool, run } from '@google/adk';

class SearchTool extends Tool {
  name = 'search';
  description = 'Search the web';

  async call(query: string): Promise<object> {
    ...
  }
}

class ResearchWorker extends Worker {
  name = 'research';
  capabilities = ['literature-review'];
  tools = [new SearchTool()];
  instructions = 'You research topics deeply...';
}

class TriageGreeter extends Greeter {
  name = 'triage';
  coordinators = [new ResearchCoordinator()];
  instructions = 'Route users...';
}

run(new TriageGreeter());
```

Same primitives; same wire format. A Python Greeter can dispatch to a JS Worker via A2A.

### (c) A2A native — Agent Card auto-generation

```python
agent_card = research_worker.agent_card()
# Returns A2A v1.0-compliant Agent Card:
# {
#   "agent_id": "research",
#   "version": "...",
#   "vendor": "...",
#   "endpoints": {...},
#   "auth": { "schemes": ["oauth2.1"], ... },
#   "capabilities": [
#     {
#       "id": "literature-review",
#       "input_schema": ...,
#       "output_artifacts": [...]
#     }
#   ],
#   "signature": { ... }  // signed via Cloud KMS
# }
```

Published at `/.well-known/agent.json` automatically when running.

### (d) Signed Cards via Cloud KMS

```python
from adk.security import KmsKeySigner

signer = KmsKeySigner(key_resource="projects/.../keyRings/.../cryptoKeys/...")
worker = ResearchWorker(card_signer=signer)
```

Production deployments use Cloud KMS for key management; Signed Cards verify against Google's published JWKs.

### (e) MCP tool integration

```python
from adk.tools import MCPToolset

mcp_toolset = MCPToolset.from_url("https://my-mcp-server.example.com/mcp", auth_token="...")

class ResearchWorker(Worker):
    tools = [mcp_toolset]  # all MCP-exposed tools available
```

Native MCP composition; per-tool OAuth scopes flow through.

### (f) Vertex AI Agent Engine deployment

```bash
adk deploy --runtime vertex-ai-agent-engine \
    --project my-project \
    --region us-central1 \
    --agent triage \
    --autoscaling min=1,max=10
```

Deploys the agent as a managed service; Agent Engine handles HTTPS, autoscaling, identity (Workload Identity Federation), tracing (Cloud Trace), logging (Cloud Logging).

### (g) Memory integration — Vertex Vector Search

```python
from adk.memory import VertexVectorMemory

memory = VertexVectorMemory(
    index_endpoint="...",
    embedding_model="textembedding-gecko",
)

worker = ResearchWorker(memory=memory)
```

Memory abstraction; pluggable across Vertex AI Vector Search, Pinecone, Weaviate, custom.

### (h) Observability native

OpenTelemetry-shape spans automatically emitted; Cloud Trace + Cloud Logging integrations zero-config; custom OTel exporters supported for non-Google-stack observability.

### (i) Cross-vendor invocation

```python
# An ADK agent invokes an Anthropic Agent Teams teammate via A2A
from adk.client import A2AClient

anthropic_team_endpoint = "https://anthropic-team.example.com/a2a"
client = A2AClient(anthropic_team_endpoint, oauth_token="...")
result = await client.submit_task(capability="code-review", input={...})
```

No vendor-specific adapter; A2A is the wire.

## Empirical results (table — May 2026)

| Metric | Value |
|---|---:|
| Languages | Python, JavaScript, Go, Java |
| GitHub stars (adk-python) | growing |
| Native A2A | Yes (reference implementation) |
| MCP support | Native (Streamable HTTP, SSE, stdio) |
| Managed deployment | Vertex AI Agent Engine |
| Identity | Google IAM, Workload Identity Federation, OAuth 2.1 generic |
| Observability | Cloud Trace + Logging native; OTel custom exporters |
| Composition primitives | Coordinator, Greeter, Worker |

| Pattern | Strength |
|---|---|
| Lead-and-spokes (Coordinator + Workers) | Strongest fit |
| Triage + specialists (Greeter + Coordinators) | First-class |
| Cross-vendor agent invocation | Trivial via A2A |
| Multi-language enterprise | First-class |
| State-machine workflows | Less natural — LangGraph stronger |
| Free-chat conversational | Less natural — AutoGen stronger |

## Variants and ablations

- **Hosted tools** (Vertex Search, Code Execution, Image Gen) versus custom tools versus MCP toolsets.
- **Memory backends:** Vertex Vector Search, Pinecone, Weaviate, custom; pluggable via abstract `Memory` interface.
- **Identity:** Google IAM (default), Workload Identity Federation (cross-cloud), generic OAuth 2.1.
- **Deployment:** self-hosted SDK, Vertex Agent Engine (managed), Cloud Run, GKE.
- **Multi-language interop:** language doesn't matter once on A2A wire.
- **AGNTCY / OASF integration** ([255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md)) — ADK Agent Cards are OASF-compatible; native registry publishing.
- **Tracing exporters:** Cloud Trace native; OpenTelemetry generic.
- **Local development:** SDK runs locally; deploy to Agent Engine on push.

## Failure modes and limitations

- **Java and Go parity lag Python.** By May 2026 not 100% feature-equivalent; Python is the lead language.
- **Vertex AI Agent Engine lock-in for managed deployments.** Self-host SDK avoids it but loses the managed-cloud value.
- **Coordinator/Greeter/Worker primitives are opinionated.** Workflows that don't fit the lead-and-spokes pattern feel awkward.
- **Less time-travel / replay than LangGraph.** State persistence exists but isn't the core abstraction.
- **No native voice agent integration.** OpenAI Agents SDK + Realtime is more mature for voice.
- **Free-chat / conversational MAS not the strong suit.** AutoGen v0.4 fits better.
- **Documentation skew toward Vertex.** Self-hosted SDK docs less rich than managed-deployment docs.
- **Workflow flexibility lower than LangGraph.** Coordinator decides routing; harder to express graph-structured workflows.
- **Observability coupling to Google Cloud.** Native fit is Cloud Trace + Logging; OTel exporters work but require setup.
- **Pricing for Agent Engine non-trivial.** Managed deployment costs add up; self-host SDK is free.

## When to use, when not

**Adopt Google ADK** for **enterprise multi-language deployments** where Java / Go / JS parity matters; for **A2A-native cross-vendor architectures** where seamless A2A integration is mandatory; for **Google Cloud-stack** customers wanting Vertex AI + BigQuery + Cloud Run + IAM-integrated identity; for **lead-and-spokes workflows** that map cleanly to Coordinator/Greeter/Worker; for organizations adopting **Vertex AI Agent Engine** as their managed agent platform.

**Skip Google ADK** for state-machine workflows (LangGraph stronger), conversational MAS / research patterns (AutoGen v0.4 stronger), handoff-style triage workflows (OpenAI Agents SDK simpler), Python-only teams (multi-language overhead unnecessary), or non-Google-Cloud deployments (managed value diminishes; self-host SDK fine but ecosystem coupling lower).

## Implications for harness engineering

- **Multi-language SDK as enterprise primitive.** Not Python-only; agents in JS / Go / Java with feature parity.
- **A2A-native by design.** [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md) — Agent Cards, Signed Cards, OAuth 2.1, task streams native.
- **Coordinator/Greeter/Worker maps to lead-and-spokes.** [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md), [250-anthropic-agent-teams](250-anthropic-agent-teams.md) — productized consensus pattern.
- **MCP toolsets first-class.** [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md), [07-model-context-protocol](07-model-context-protocol.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md).
- **AGNTCY / OASF aligned.** [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md) — Agent Cards OASF-compatible.
- **Memory pluggable.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md) — Vertex Vector Search or custom.
- **Identity via Workload Identity Federation.** [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md).
- **OpenTelemetry observability.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), [24-observability-tracing](24-observability-tracing.md).
- **Vertex Agent Engine for managed deployment.** [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [p7-daemon](../projects/polaris/docs/research/p7-daemon.md).
- **Cross-vendor team composition.** [250-anthropic-agent-teams](250-anthropic-agent-teams.md) — ADK Coordinator hires Anthropic teammate via A2A.
- **Routines + ADK.** [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md) — fire ADK agents from routines.
- **Distributed deployment over Tailscale + NATS.** [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md) — for non-Google-Cloud distributed setups.
- **Skill marketplace alignment.** [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md) — Workers as portable specialists.
- **Cost router across managed and self-hosted.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md).

**One-line takeaway for harness designers.** **Google ADK is the multi-language, A2A-native, enterprise-shape agent runtime — Python + JS + Go + Java parity, Coordinator/Greeter/Worker primitives mapping cleanly to lead-and-spokes, native AGNTCY/OASF/A2A compliance, and Vertex AI Agent Engine for managed cloud — making it the natural pick for enterprise + cross-vendor + Google-stack deployments and a credible alternative to LangGraph and OpenAI Agents SDK when language portability or A2A compliance is the binding constraint.**
