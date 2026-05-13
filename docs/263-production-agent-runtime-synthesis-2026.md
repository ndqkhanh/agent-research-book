# 263 — Production Agent Runtime Synthesis 2026: a decision matrix across LangGraph, Agents SDK, AutoGen, ADK, CrewAI, Agent Teams

**Synthesis of:** [259-langgraph-deep-dive](259-langgraph-deep-dive.md), [260-openai-agents-sdk-deep-dive](260-openai-agents-sdk-deep-dive.md), [261-autogen-v04-deep-dive](261-autogen-v04-deep-dive.md), [262-google-adk-deep-dive](262-google-adk-deep-dive.md), [164-crewai-multi-agent-framework](164-crewai-multi-agent-framework.md), [250-anthropic-agent-teams](250-anthropic-agent-teams.md). Cross-references: [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md), [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md), [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md), [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md), [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md), [258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md), [42-langchain-deep-agents](42-langchain-deep-agents.md), [91-metagpt-deep](91-metagpt-deep.md), [92-chatdev](92-chatdev.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md), [126-frameworks-comparison](126-frameworks-comparison.md), [144-build-your-own-harness](144-build-your-own-harness.md), [145-comparing-coding-harnesses](145-comparing-coding-harnesses.md), [147-vendor-lock-in](147-vendor-lock-in.md), [166-research-agent-frameworks-synthesis](166-research-agent-frameworks-synthesis.md).

**One-line definition.** A **decision matrix** for picking among the six dominant production agent runtimes in 2026 — **LangGraph** (state-machine, production-tier, durable execution, time-travel), **OpenAI Agents SDK** (handoff workflows, sessions, native tracing, OpenAI-stack), **AutoGen v0.4** (event-driven actors, code-execution pairs, Magnetic-One coordination, research lineage), **Google ADK** (multi-language Python/JS/Go/Java, A2A-native, Vertex Agent Engine, enterprise), **CrewAI** ([164](164-crewai-multi-agent-framework.md), declarative role-playing crews, 50.8k stars, 2B agent runs), and **Anthropic Agent Teams** ([250](250-anthropic-agent-teams.md), Claude Code lead-and-spokes with shared task list and peer messaging) — each occupying a distinct architectural pattern (state-machine vs handoff vs actors vs crews vs Coordinator/Worker vs lead-and-spokes), each with a clear best-fit regime, with a unified comparison across **architectural pattern, durability, cross-vendor support, language portability, observability, and unit economics** that lets you pick deliberately rather than by tribal allegiance.

## Why this synthesis matters (the "which framework should we pick" question finally has a structured answer)

By May 2026 the agent-runtime landscape has six dominant production options plus a long tail of niche frameworks. For most of 2024–2025 the answer to "which one should we pick" was tribal — your team's familiarity, your investor's stack, your favorite blog post. By 2026 the field has enough deployments and enough comparison data that **the answer is structurable** by architectural pattern, deployment requirements, and unit economics. This synthesis is that decision matrix.

The six runtimes occupy six distinct architectural patterns:

| Runtime | Architectural pattern |
|---|---|
| **LangGraph** ([259](259-langgraph-deep-dive.md)) | Explicit state machine (TypedDict state + graph nodes + edges + checkpointer) |
| **OpenAI Agents SDK** ([260](260-openai-agents-sdk-deep-dive.md)) | Handoff workflow (`transfer_to_X` as LLM tool) |
| **AutoGen v0.4** ([261](261-autogen-v04-deep-dive.md)) | Event-driven actors with typed messages over a bus |
| **Google ADK** ([262](262-google-adk-deep-dive.md)) | Coordinator + Greeter + Worker (lead-and-spokes productized, A2A-native) |
| **CrewAI** ([164](164-crewai-multi-agent-framework.md)) | Declarative crews (Agent + Task + Crew + Process) |
| **Anthropic Agent Teams** ([250](250-anthropic-agent-teams.md)) | Lead spawns full Claude Code teammates with shared task list + mailbox |

The patterns are not interchangeable. A workflow that's natural as a state machine (research → critique → revise → publish) is awkward as a handoff workflow (each transition is a `transfer_to_*` tool call), and a workflow that's natural as a handoff (triage → specialist) is awkward as a state graph (every transition needs a node + conditional edge). **Picking by pattern is the first-order decision**; everything else is second-order.

The second-order considerations are unit economics, durability requirements, cross-vendor needs, language portability, and ecosystem alignment. These are decision-matrix dimensions, not tribal preferences. The tables below are the cheat sheet.

Take this synthesis seriously and three things change. **First**, you make framework selection an explicit architectural decision with documented tradeoffs, not a default choice. **Second**, you understand that **multiple runtimes can coexist in one organization** — your customer-service team uses Agents SDK for handoff triage; your research team uses LangGraph for stateful workflows; your enterprise integration team uses ADK for A2A-native cross-vendor calls — and the protocol stack ([258](258-agent-protocol-stack-synthesis-2026.md)) makes them interoperable. **Third**, **`harness_core/runtime_adapter/`** becomes a useful abstraction — a thin layer that lets the same agent definition target multiple runtimes when needed (rare, but valuable for portability).

## The six runtimes side-by-side

### Architectural pattern (pick this first)

| Runtime | Pattern | Natural workflow | Awkward workflow |
|---|---|---|---|
| LangGraph | State machine | Multi-step pipelines with branching and rollback | Free-chat conversational MAS |
| OpenAI Agents SDK | Handoff | Triage + specialists; tier-1 → tier-2 escalation | Stateful long-running pipelines |
| AutoGen v0.4 | Event-driven actors | Code-execution pairs; distributed actors; research patterns | Simple linear handoffs |
| Google ADK | Coordinator/Worker | Lead-and-spokes hierarchies; A2A cross-vendor | State-machine workflows |
| CrewAI | Declarative crews | Role-playing teams (PM/Eng/QA); SOP-driven | Highly dynamic routing |
| Anthropic Agent Teams | Lead + isolated teammates | Parallel review; multi-hypothesis investigation | Single-host single-runtime workflows where 7× cost matters |

### Durability and execution model

| Runtime | Crash recovery | Time-travel | Human-in-the-loop |
|---|---|---|---|
| LangGraph | Yes (checkpointer) | Yes (replay any checkpoint) | First-class (`interrupt()`) |
| OpenAI Agents SDK | Session persists; mid-run not durable | No native | Via guardrail tripwire (abort, not pause) |
| AutoGen v0.4 | Basic state save | No | Via `UserProxyAgent` |
| Google ADK | State persists; durable via Agent Engine | Limited | Via Workflow primitives |
| CrewAI | Limited | Limited | Via `human_input=True` |
| Anthropic Agent Teams | Filesystem state survives | Manual via task-list inspection | First-class (lead approves plans) |

### Cross-vendor / multi-language

| Runtime | Cross-vendor (A2A) | Multi-language |
|---|---|---|
| LangGraph | Community A2A wrapper | Python primary, JS lagging |
| OpenAI Agents SDK | Via LiteLLM (provider flex); A2A community | Python primary, JS/TS official |
| AutoGen v0.4 | Community A2A | Python primary, .NET parity 2026, Java/JS community |
| Google ADK | **Native A2A** (reference implementation) | **Python + JS + Go + Java parity** |
| CrewAI | Community | Python primary |
| Anthropic Agent Teams | Not native; community wrapper | Claude Code only (TypeScript-derived) |

### Observability

| Runtime | Native tracing | Observability backend |
|---|---|---|
| LangGraph | Yes | LangSmith primary; OTel exporters |
| OpenAI Agents SDK | Yes (auto-uploaded) | OpenAI dashboard; OTel custom processors |
| AutoGen v0.4 | Yes | OTel-native; Datadog/Grafana/Azure Monitor |
| Google ADK | Yes | Cloud Trace + Logging; OTel exporters |
| CrewAI | Limited; community OTel | LangSmith / Phoenix integrations |
| Anthropic Agent Teams | Yes (HIR-shape) | `~/.claude/teams/.../events.jsonl` |

### Token cost

| Runtime | Cost characteristic |
|---|---|
| LangGraph | Standard per-node LLM cost; no multiplier |
| OpenAI Agents SDK | Each handoff = LLM tool call; small overhead |
| AutoGen v0.4 | GroupChat patterns can chat-loop; structured patterns standard |
| Google ADK | Standard; Agent Engine adds infra cost |
| CrewAI | Standard per-agent; Process orchestration adds turns |
| Anthropic Agent Teams | **~7× plan-mode cost** (each teammate full Claude instance) |

## The decision matrix

### Decision 1: What's the workflow shape?

```
State machine (multi-step, branching, rollback) ............... LangGraph
Handoff (triage → specialist) ................................. OpenAI Agents SDK
Code-executing pair (assistant + executor) .................... AutoGen v0.4
Lead-and-spokes hierarchy ..................................... Google ADK or Anthropic Agent Teams
Role-playing crew (PM / Eng / QA) ............................. CrewAI or MetaGPT
Parallel review / multi-hypothesis ............................ Anthropic Agent Teams
Distributed actors / event-driven ............................. AutoGen v0.4
```

### Decision 2: What's your stack alignment?

```
Anthropic + Claude Code primary ............................... Anthropic Agent Teams (in-runtime); LangGraph (cross-vendor)
OpenAI primary ................................................ OpenAI Agents SDK
Microsoft / Azure / .NET ...................................... AutoGen v0.4
Google Cloud / Vertex AI ...................................... Google ADK
Multi-cloud / agnostic ........................................ LangGraph or CrewAI
```

### Decision 3: What's your durability requirement?

```
Crash recovery + time-travel + HITL pause/resume .............. LangGraph (only)
Conversation persistence (no mid-run durability) .............. OpenAI Agents SDK
Basic state save .............................................. AutoGen v0.4 / ADK / CrewAI
Long-running stateful workflows ............................... LangGraph + Postgres checkpointer
Durable workflow primitive (alternative) ...................... Temporal + LangGraph
```

### Decision 4: What's your language portability requirement?

```
Python only ................................................... Any
Python + JS/TS ................................................ Google ADK or OpenAI Agents SDK
.NET / Java ................................................... Google ADK or AutoGen v0.4
Go ............................................................ Google ADK
```

### Decision 5: Are you crossing vendor boundaries?

```
Yes (cross-vendor team composition) ........................... Google ADK (A2A-native) or any with A2A wrapper
No (single-runtime) ........................................... Pick by pattern (Decision 1)
```

### Decision 6: What's your team size + maturity?

```
Small team, prototyping ....................................... CrewAI (declarative, ergonomic)
Mid-size team, production-bound ............................... LangGraph or OpenAI Agents SDK
Enterprise, multi-language .................................... Google ADK
Research / experimentation .................................... AutoGen v0.4
```

## Composite recommendations

### "I'm building a customer-service triage agent."

→ **OpenAI Agents SDK**. Handoff pattern is the natural fit; sessions provide conversation continuity; guardrails handle PII filtering; native tracing for QA review. Skip LangGraph (state-machine ceremony) and AutoGen (chat-pattern overkill).

### "I'm building a research agent (Polaris-style) that needs durable execution."

→ **LangGraph**. Stateful pipeline (research → critique → revise → publish) is naturally graph-shape; checkpointer + Postgres for crash recovery; time-travel for debugging; `interrupt()` for HITL approval gates. Skip Agents SDK (no durability) and AutoGen (less mature checkpointing).

### "I'm building parallel multi-reviewer code review."

→ **Anthropic Agent Teams** (if Claude Code primary; eat the 7× cost for the parallelism). Or **LangGraph with `Send` fan-out** for production-tier with durability. Or **AutoGen v0.4 RoundRobinGroupChat** if review is conversational. Skip Agents SDK (no native parallel) and CrewAI (overhead high).

### "I'm building a multi-language enterprise agent platform."

→ **Google ADK**. Python + JS + Go + Java parity; A2A-native cross-vendor; Vertex Agent Engine for managed cloud. Skip Python-only options.

### "I'm building a code-executing agent that runs Python."

→ **AutoGen v0.4 CodeExecutorAgent + AssistantAgent** pair. Canonical pattern; Docker-sandboxed by default; first-class. Skip alternatives (less ergonomic for code-execution-specific patterns).

### "I'm building a role-playing software-development team (PM/Eng/QA)."

→ **CrewAI** ([164](164-crewai-multi-agent-framework.md)) or **MetaGPT** ([91-metagpt-deep](91-metagpt-deep.md)). CrewAI is more declarative and ergonomic; MetaGPT has stronger SOP-driven structured-artifact pattern. Skip LangGraph (overhead) and Agents SDK (handoffs less natural for collaborative roles).

### "I'm building research-style multi-agent experiments."

→ **AutoGen v0.4**. Magnetic-One coordination, code-execution agents, Society-of-Mind, free-form GroupChat (with awareness of MAST cluster-2 risks) — research patterns are first-class.

### "I'm building a self-hosted distributed personal-assistant agent (Lyra-class)."

→ **LangGraph for state + Tailscale + NATS for distribution** ([253](253-tailscale-nats-mesh-for-distributed-agents.md)). LangGraph's checkpointer can run across hosts via Postgres; NATS for cross-host pub/sub. Skip cloud-managed runtimes ($5/mo budget).

### "I'm shipping an agent on Vertex AI."

→ **Google ADK + Vertex AI Agent Engine**. Native fit; managed deployment; IAM-integrated; OTel via Cloud Trace.

### "I'm shipping an agent on AWS Bedrock."

→ **LangGraph + AWS Lambda / ECS** (Bedrock Agents has its own runtime but ecosystem narrower than ADK + Vertex; LangGraph + Bedrock-LLM via LiteLLM is more flexible).

## Empirical results (table — May 2026 production deployments)

| Runtime | Notable production deployments | Stars |
|---|---|---:|
| LangGraph | Klarna, Replit, Elastic, NVIDIA, Snowflake | ~12.8 k |
| OpenAI Agents SDK | OpenAI internal, partner deployments | growing rapidly |
| AutoGen v0.4 | Microsoft research, Microsoft 365 Copilot Studio | ~42 k |
| Google ADK | Google Cloud customers, partner integrations | growing |
| CrewAI | 2B agent runs reported; 60+ Fortune-500 evaluations | ~50.8 k |
| Anthropic Agent Teams | Claude Code v2.1.32+ user base | n/a (feature) |

## Cross-runtime composition

The protocol stack ([258](258-agent-protocol-stack-synthesis-2026.md)) makes runtimes interoperable:

- **A2A bridges any two runtimes** ([254](254-a2a-protocol-deep-dive.md)). LangGraph agent invokes ADK Worker via A2A.
- **MCP exposes tools across runtimes** ([256](256-mcp-2025-2026-evolution.md)). All six runtimes consume MCP tool servers.
- **AGNTCY discovery finds agents across runtimes** ([255](255-agntcy-oasf-acp-deep-dive.md)). Registry serves any A2A-exposed agent regardless of underlying runtime.
- **SKILL.md portable specialists** ([257](257-agent-skill-marketplace-landscape.md)). Specialist definition portable across LangGraph nodes, Agents SDK Agents, AutoGen actors, ADK Workers, CrewAI Agents, Agent Teams subagent definitions.
- **Routines as universal trigger surface** ([252](252-routines-pattern-for-self-hosted-agents.md)). Fire any runtime's agents from cron / API / webhook.
- **Tailscale + NATS as distributed transport** ([253](253-tailscale-nats-mesh-for-distributed-agents.md)). All runtimes can use it for cross-host deployment.

## Variants and ablations

- **Multi-runtime org.** Different teams use different runtimes; A2A + AGNTCY for inter-team interop.
- **Hybrid graph + handoff.** LangGraph subgraph wraps an Agents SDK handoff workflow.
- **CrewAI + LangGraph composition.** CrewAI for crew definition; LangGraph for production execution layer.
- **Agent Teams + LangGraph.** Anthropic Agent Teams in Claude Code; each teammate is a LangGraph agent for production-grade durability.
- **`harness_core/runtime_adapter/`.** Thin abstraction layer; same agent definition targets multiple runtimes.

## Failure modes when picking the wrong runtime

- **Picking LangGraph for simple chat workflows.** Overhead exceeds value; Agents SDK or CrewAI ergonomic.
- **Picking Agents SDK for stateful research pipelines.** No durability; crash loses progress; choose LangGraph.
- **Picking AutoGen GroupChat for production triage.** MAST cluster-2 failure modes; choose Agents SDK handoffs.
- **Picking ADK for Python-only Anthropic team.** Multi-language overhead unnecessary; choose LangGraph or Agents SDK.
- **Picking CrewAI for time-travel debugging.** No native; choose LangGraph.
- **Picking Anthropic Agent Teams for high-volume production.** 7× cost; choose lighter runtime.

## When to use, when not (the meta-rule)

**Pick by architectural pattern first** (Decision 1), **stack alignment second** (Decision 2), **durability third** (Decision 3). Multi-runtime coexistence is fine; pick the right one per workflow.

**Skip framework adoption entirely** when prototyping single-shot agents (a Python script + Anthropic SDK suffices), when the workflow is simpler than any framework's overhead, or when no durability / observability / multi-agent requirement exists.

## Implications for harness engineering

- **`harness_core/runtime_adapter/` package.** Thin wrapper exposing common agent-definition fields; each adapter targets one runtime.
- **Multi-runtime org policy.** Different teams pick different runtimes; A2A + AGNTCY for interop.
- **Per-project runtime choice.** [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md), [208-lyra-multi-hop-collaborative-apply-plan](208-lyra-multi-hop-collaborative-apply-plan.md), [209-argus-multi-hop-collaborative-apply-plan](209-argus-multi-hop-collaborative-apply-plan.md), [210-mentat-learn-collaborative-apply-plan](210-mentat-learn-collaborative-apply-plan.md), [211-cross-project-power-up-plan-with-tradeoffs](211-cross-project-power-up-plan-with-tradeoffs.md) — per-project runtime selection per the matrix above.
- **Protocol-stack alignment.** [258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md) — every runtime exposes A2A endpoint; consumes MCP; publishes OASF.
- **Observability uniform via OTel.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), [24-observability-tracing](24-observability-tracing.md) — span schema convergence across runtimes.
- **Cost router across runtimes.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md), [88-confidence-driven-router](88-confidence-driven-router.md) — pick runtime by per-task cost.
- **Agent Teams for parallel review across runtimes.** [250-anthropic-agent-teams](250-anthropic-agent-teams.md) — teammates can be any runtime exposed via A2A.
- **Routines fire across runtimes.** [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md).
- **Distributed across runtimes.** [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md).
- **Skill marketplace alignment.** [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md) — SKILL.md portable across all six runtimes.
- **Cross-channel verifier across runtimes.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md) — verifier in different runtime than generator.
- **Memory tiering pluggable.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md) — memory backend is a separate concern from runtime.
- **Production deployment pluggable.** Vertex Agent Engine, AWS Bedrock Agents, self-hosted on Tailscale + NATS — orthogonal to runtime choice.
- **Vendor lock-in tracking.** [147-vendor-lock-in](147-vendor-lock-in.md) — each runtime adds some lock-in; A2A + MCP + OASF reduce it.

**One-line takeaway for harness designers.** **The 2026 production agent-runtime landscape has six dominant options each occupying a distinct architectural pattern (LangGraph state machines, OpenAI Agents SDK handoffs, AutoGen v0.4 actors, Google ADK Coordinator/Worker, CrewAI declarative crews, Anthropic Agent Teams lead-and-spokes); pick by pattern first, stack alignment second, durability third — and lean on the protocol stack (A2A + MCP + AGNTCY + SKILL.md) to keep multiple runtimes interoperable in one organization without per-pair adapter code.**
