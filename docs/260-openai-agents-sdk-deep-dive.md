# 260 — OpenAI Agents SDK: Swarm successor with handoffs as first-class primitives

**Anchor.** OpenAI — *Agents SDK (openai-agents-python, openai-agents-js)* — https://github.com/openai/openai-agents-python (rapidly growing in 2026), https://openai.github.io/openai-agents-python/. Successor to **Swarm** (https://github.com/openai/swarm — frozen as educational, ~21k stars). Production-ready release in early 2026 with the `Agents`, `Tools`, `Handoffs`, `Guardrails` four-primitive design. Companions: [259-langgraph-deep-dive](259-langgraph-deep-dive.md), [261-autogen-v04-deep-dive](261-autogen-v04-deep-dive.md), [262-google-adk-deep-dive](262-google-adk-deep-dive.md), [263-production-agent-runtime-synthesis-2026](263-production-agent-runtime-synthesis-2026.md), [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md), [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md).

**One-line definition.** OpenAI's production-grade multi-agent SDK — successor to the educational Swarm release — with **four primitives** (`Agent`, `Tool`, `Handoff`, `Guardrail`) and a **distinctive design choice**: **handoffs are exposed to the LLM as `transfer_to_X` tools**, making the routing decision part of the model's normal tool-call flow rather than a hidden orchestration layer; combined with **first-class tracing**, **session-state primitives**, **input/output guardrails for validation**, and **native MCP tool support**, the SDK is OpenAI's opinionated answer to the question "how should multi-agent applications be structured" — favouring **handoff-style sequential workflows** over LangGraph's state-machine and AutoGen's GroupChat.

## Why this paper matters (handoff-first multi-agent is a third architectural pattern, distinct from graphs and chat)

Multi-agent runtimes by 2026 had converged on three major patterns: **state graphs** (LangGraph), **conversational chat** (AutoGen pre-v0.4 GroupChat), and **handoff workflows**. OpenAI Agents SDK is the canonical handoff-pattern implementation. The architectural insight: in many real workflows the question "which agent should do this next" is itself a *modelling decision* the LLM can make. Rather than wiring an explicit orchestrator that decides routing, **expose `transfer_to_X` as a tool** the LLM calls, and the LLM picks the next agent the same way it picks any other tool. The handoff carries the conversation history; the receiving agent picks up where the previous left off, with full context. This is operationally simpler than a graph (no explicit routing logic) and more controllable than free-chat (each handoff is a discrete tool call).

The Swarm precursor (released Oct 2024 as an educational reference, frozen in 2025) was the first time OpenAI shipped this pattern publicly. The minimal API — `Agent` + `function_tool` + `transfer_to(...)` returning an `Agent` — was deliberately tiny, designed to teach the pattern rather than be a production SDK. The 2026 Agents SDK is the production version: same handoff pattern, but with the added primitives needed for real deployments — input/output guardrails, tracing, session state, MCP tool integration, OpenAI-Realtime + Voice support, and async-first design. The lineage from Swarm makes it the most pedagogically clean of the major runtimes.

The design choices reveal OpenAI's opinionated take on multi-agent. **Handoffs over orchestrators** because the LLM is good at deciding what to do next when given the right tools. **Sessions as the unit of conversation** with built-in state management rather than each agent maintaining its own. **Guardrails as a first-class concern** at the input and output boundaries, not interleaved through the agent body. **Tracing built-in** with no opt-in dance — every run produces a trace; you go to OpenAI's dashboard to inspect. **MCP tool support native** so the SDK plays clean with the broader protocol stack ([256](256-mcp-2025-2026-evolution.md)). The result is a runtime that is simpler than LangGraph but adds production primitives Swarm lacked.

Take this seriously and three things change. **First**, you stop architecting routing as orchestrator code and start treating it as **a tool the LLM calls** — the agent body becomes "do your job; if you need to hand off, call `transfer_to_X`." **Second**, **guardrails at boundaries** become structural — input validation before the model ever sees a request; output validation before the response is returned to user — rather than scattered throughout the agent code. **Third**, **session state as the conversation primitive** unifies the multi-agent case with the single-agent case: a "team" is just multiple agents sharing a session, with handoffs deciding who's active.

## Problem it solves (production multi-agent without graph complexity)

1. **Orchestrator code is brittle.** Hand-written routing logic doesn't generalize; LLM-decided routing via `transfer_to` is more flexible.
2. **Conversation state across handoffs needs care.** Sessions own the message list; agents pick up cleanly when handed control.
3. **Validation at boundaries was scattered.** Input/output guardrails as first-class primitives at the edges of agent execution clean this up.
4. **Tracing was per-team to bolt on.** Native first-party tracing reduces friction to zero; debug productively from day one.
5. **MCP integration ergonomics.** Native MCP tool support means agents can use any MCP server as a tool source without per-agent wiring.
6. **Voice + Realtime agents need first-class support.** OpenAI's Realtime API + Voice agents integrate natively in the SDK; voice agents become an Agent subtype, not a separate runtime.
7. **Educational gap from Swarm.** Swarm taught the pattern but wasn't production; Agents SDK is the production version with the same pedagogy.

## Core idea in one paragraph

An OpenAI Agents SDK application is a set of `Agent` objects each defined by `(name, instructions, tools, handoffs, guardrails, model)` where `instructions` is the system prompt, `tools` are callable functions (or MCP-exposed tools, or hosted-tools like `WebSearchTool`), `handoffs` are other Agent references the LLM can transfer-to, and `guardrails` are validators run on input and output. The SDK exposes **`transfer_to_<agent>` as a tool to the LLM** — the LLM chooses to hand off the same way it chooses any tool call. A `Runner.run(agent, input, session)` invocation runs the agent loop until terminal output (or a handoff to another agent, which becomes the active agent and continues the same loop). **Sessions** are the conversation primitive — a `Session` owns the message list, persists across runs, and is automatically threaded through handoffs. **Guardrails** are validators with a "tripwire" semantic: if a guardrail trips, the run is aborted with a structured error. **Tracing** is native and automatic — every run produces a trace viewable in the OpenAI dashboard. Native **MCP tool support** lets agents use MCP servers as tool sources. The SDK is **provider-agnostic** in principle (works with non-OpenAI models via LiteLLM proxy or direct support) but optimized for OpenAI models. Voice agents are first-class via `RealtimeAgent` and the Realtime API. The lineage from Swarm gives it the cleanest mental model among the production runtimes; the trade-off vs LangGraph is less explicit state machinery in exchange for less ceremony.

## Mechanism (step by step)

### (a) Agent definition

```python
from agents import Agent, function_tool, Runner

@function_tool
def get_weather(city: str) -> str:
    return f"The weather in {city} is sunny."

@function_tool
def book_flight(origin: str, destination: str, date: str) -> str:
    return f"Booked flight {origin} → {destination} on {date}."

triage_agent = Agent(
    name="Triage",
    instructions="You route users to the right specialist.",
    handoffs=[],  # populated below
)

weather_agent = Agent(
    name="Weather",
    instructions="You answer weather questions.",
    tools=[get_weather],
)

travel_agent = Agent(
    name="Travel",
    instructions="You book flights.",
    tools=[book_flight],
)

triage_agent.handoffs = [weather_agent, travel_agent]
```

### (b) Handoff exposed as tool

The SDK auto-generates a `transfer_to_weather` and `transfer_to_travel` tool exposed to the triage agent. The LLM sees them as ordinary tools:

```
User: "What's the weather in Tokyo?"
Triage agent (model output): tool_call: transfer_to_weather()
→ Runtime switches active agent to Weather
Weather agent (model output): tool_call: get_weather(city="Tokyo")
→ Tool returns "sunny"
Weather agent: "The weather in Tokyo is sunny."
```

The handoff is part of the LLM's normal tool-call flow.

### (c) Sessions for conversation state

```python
from agents import SQLiteSession

session = SQLiteSession("user-123")

result = await Runner.run(triage_agent, "What's the weather in Tokyo?", session=session)
print(result.final_output)
# session now contains the full message history

# Continue the conversation
result = await Runner.run(triage_agent, "Book me a flight there.", session=session)
# Triage hands off to Travel; Travel sees the prior context
```

`SQLiteSession`, `RedisSession`, `PostgresSession`, custom backends. Session persists across runs and across handoffs.

### (d) Guardrails

```python
from agents import input_guardrail, output_guardrail, GuardrailFunctionOutput

@input_guardrail
async def block_pii(ctx, agent, input):
    if contains_pii(input):
        return GuardrailFunctionOutput(
            output_info={"reason": "PII detected"},
            tripwire_triggered=True,
        )
    return GuardrailFunctionOutput(output_info=None, tripwire_triggered=False)

@output_guardrail
async def block_unsafe_output(ctx, agent, output):
    if contains_unsafe_content(output):
        return GuardrailFunctionOutput(
            output_info={"reason": "Unsafe output"},
            tripwire_triggered=True,
        )
    return GuardrailFunctionOutput(output_info=None, tripwire_triggered=False)

agent = Agent(
    name="Production",
    instructions="...",
    input_guardrails=[block_pii],
    output_guardrails=[block_unsafe_output],
)
```

When a guardrail trips, `Runner.run` raises `InputGuardrailTripwireTriggered` or `OutputGuardrailTripwireTriggered` — clean error handling at the boundary.

### (e) Tracing

```python
result = await Runner.run(triage_agent, "...", session=session)
# Trace automatically uploaded to OpenAI; viewable at https://platform.openai.com/traces

# Inspect locally:
from agents.tracing import get_current_trace
trace = get_current_trace()
for span in trace.spans:
    print(span.span_data)
```

Every run produces an OpenTelemetry-shape trace with span types `agent_run`, `function_call`, `handoff`, `guardrail`, `llm_call`. Native dashboard, no opt-in.

### (f) MCP integration

```python
from agents.mcp import MCPServerSse

mcp_server = MCPServerSse(
    params={"url": "https://my-mcp-server.example.com/mcp"},
)
await mcp_server.connect()

agent = Agent(
    name="Production",
    instructions="...",
    mcp_servers=[mcp_server],  # tools auto-discovered from MCP server
)
```

Agent's tool list automatically includes MCP-exposed tools. Multi-MCP support; tool conflicts disambiguated by namespacing.

### (g) Realtime / Voice agents

```python
from agents.realtime import RealtimeAgent, RealtimeRunner

voice_agent = RealtimeAgent(
    name="Voice Assistant",
    instructions="You are a voice assistant.",
    tools=[get_weather, book_flight],
    handoffs=[triage_agent],  # voice agent can hand off to text agents
)

runner = RealtimeRunner(voice_agent)
async with runner.run() as session:
    # session is bidirectional audio stream
    ...
```

Voice + handoffs in one runtime; the voice agent can hand off to a non-voice specialist for tasks better handled in text.

### (h) Hosted tools

OpenAI provides ready-made hosted tools:

```python
from agents.tool import WebSearchTool, FileSearchTool, CodeInterpreterTool

agent = Agent(
    name="Researcher",
    instructions="...",
    tools=[WebSearchTool(), FileSearchTool(vector_store_ids=[...]), CodeInterpreterTool()],
)
```

These run on OpenAI's infra, no setup; trade-off is OpenAI dependency.

### (i) Provider flexibility

Via LiteLLM:

```python
agent = Agent(
    name="Anthropic Agent",
    model="anthropic/claude-sonnet-4-6",
    instructions="...",
)
```

The SDK is OpenAI-shape but any LLM provider works through LiteLLM proxy.

## Empirical results (table — May 2026)

| Metric | Value |
|---|---:|
| GitHub stars (openai-agents-python) | rapidly growing; ~10k+ |
| Swarm stars (frozen) | ~21k |
| Reference implementations | Python, JS/TS |
| Hosted tool count | WebSearch, FileSearch, CodeInterpreter, ImageGen + more |
| Native MCP support | Yes (sse, http, stdio) |
| Voice / Realtime | Yes |
| Tracing dashboard | platform.openai.com/traces |

| Pattern | Strength |
|---|---|
| Triage + specialists with handoffs | Strongest fit |
| Single-agent + tools | Trivial |
| Voice + text agent composition | First-class |
| Cross-vendor (LiteLLM) | Supported |
| Long-running stateful workflows | Less natural — LangGraph stronger |

## Variants and ablations

- **Swarm (educational, frozen).** Minimal pedagogy version; `Agent` + `function_tool` + `transfer_to`.
- **Custom Session backends.** Plug in any K/V store as session storage.
- **`output_type` for structured outputs.** Pydantic schema for typed responses.
- **Streaming via `Runner.run_streamed`.** Per-token streaming with structured event types.
- **Realtime + voice.** First-class integration with Realtime API.
- **Hosted tools vs custom tools.** Hosted runs on OpenAI infra; custom runs locally.
- **Provider flexibility via LiteLLM.** Use Anthropic / Gemini / open-weights models.
- **Tracing custom processors.** Send traces to your own observability backend in addition to OpenAI's.

## Failure modes and limitations

- **OpenAI ecosystem bias.** Provider flexibility exists via LiteLLM but native fit is OpenAI; tracing dashboard is OpenAI-only.
- **No durable execution.** Sessions persist conversation; runs do not checkpoint mid-run. A crash mid-run loses progress (compare LangGraph's checkpointer).
- **Handoff overhead at LLM cost.** Each handoff is a tool call in the LLM's flow; many handoffs = many model calls.
- **No native graph branching.** The runtime is sequential per-active-agent; parallel exploration requires `asyncio.gather`-style orchestration outside the SDK.
- **Guardrails are aborts, not interventions.** A tripped guardrail kills the run; soft-correction patterns require custom code.
- **Less-mature multi-agent debate / verifier patterns.** Excels at handoff workflows; for cross-channel verifier loops, less ergonomic than LangGraph.
- **Session state schema rigid.** `Session` API is fixed; complex state needs separate persistence.
- **Tracing privacy.** Auto-uploaded to OpenAI by default; opt out for sensitive deployments.
- **MCP support is recent and evolving.** Spec compliance complete but feature parity with the broader ecosystem still maturing.

## When to use, when not

**Adopt OpenAI Agents SDK** for production agent applications dominated by **handoff-style workflows** (triage + specialists, tier-1 → tier-2 escalation, multi-step funnels with role transitions); for teams primarily on the OpenAI stack who want first-class tracing without ceremony; for **voice agents** that need to hand off to text specialists; for prototypes that need to scale up to production without a runtime rewrite. The strongest cases are **customer-service triage**, **sales-assistant funnels**, and **voice-first agents with text fallback specialists**.

**Skip OpenAI Agents SDK** for state-machine-shape workflows (LangGraph stronger), free-chat conversational MAS (AutoGen v0.4 stronger), heavy time-travel / branching needs (LangGraph), durable-execution-required workflows (Temporal + LangGraph), or environments that explicitly avoid OpenAI dependency (use Anthropic SDK + custom multi-agent layer or AutoGen / Google ADK).

## Implications for harness engineering

- **Handoffs as a first-class harness primitive.** [02-subagent-delegation](02-subagent-delegation.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md), [250-anthropic-agent-teams](250-anthropic-agent-teams.md) — handoff is a pattern that complements team-spawn and subagent-delegation.
- **Sessions as conversation memory.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md) — session backend is one tier of memory.
- **Guardrails at boundaries.** [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [05-hooks](05-hooks.md), [08-hooks-and-claim-gate](../projects/polaris/docs/blocks/08-hooks-and-claim-gate.md) — input/output guardrails align with bright-line gates.
- **Tracing as harness observability.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), [24-observability-tracing](24-observability-tracing.md) — span schema convergence.
- **MCP integration first-class.** [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md), [07-model-context-protocol](07-model-context-protocol.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md).
- **Voice + handoff for harmony-voice / mentat-learn.** [137-voice-agents](137-voice-agents.md), [48-voiceagentrag-dual-agent](48-voiceagentrag-dual-agent.md) — voice agent hands off to text specialist for complex tasks.
- **Cross-channel verifier as a guardrail or handoff.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md).
- **Routines + Agents SDK.** [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md) — fire a session-backed agent run from a scheduled routine.
- **A2A endpoint over Agents SDK.** [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md) — wrap an Agent as an A2A endpoint.
- **Cost router across providers.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md) — LiteLLM lets the router pick.
- **Agent Teams alignment.** [250-anthropic-agent-teams](250-anthropic-agent-teams.md), [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md) — handoff vs lead-and-spokes are different patterns; pick by workflow shape.
- **Skill marketplace integration.** [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md), [04-skills](04-skills.md) — Agent definitions are SKILL.md-equivalent; portable across runtimes.
- **HITL via guardrail intervention.** [23-human-in-the-loop](23-human-in-the-loop.md) — guardrail trip → user notification → resume / abort.

**One-line takeaway for harness designers.** **OpenAI Agents SDK is the cleanest production handoff-pattern multi-agent runtime — `transfer_to_X` exposed as a tool the LLM calls, sessions for conversation state, guardrails at boundaries, native tracing and MCP — making it the natural choice for triage-and-specialist workflows on the OpenAI stack and a credible production option for cross-vendor handoff workflows via LiteLLM; pick it when handoffs dominate, pick LangGraph when state machines dominate.**
