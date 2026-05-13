# 264 — Agent Observability Stack 2026: cross-runtime tracing on OpenTelemetry, with HIR-shape typed events

**Anchors.** OpenTelemetry — https://opentelemetry.io/, the de-facto standard for distributed tracing the agent ecosystem converged on. LangSmith — https://smith.langchain.com/ (LangChain's observability surface, native LangGraph integration). Arize Phoenix — https://github.com/Arize-ai/phoenix (OSS LLM-app tracing, ~5k+ stars). Helicone — https://www.helicone.ai/ (LLM-call analytics + caching). Honeycomb — https://www.honeycomb.io/ (general-purpose observability with strong agent-trace UX as of 2026). Companions: [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), [24-observability-tracing](24-observability-tracing.md), [263-production-agent-runtime-synthesis-2026](263-production-agent-runtime-synthesis-2026.md), [265-agent-evaluation-2026](265-agent-evaluation-2026.md), [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md), [267-agent-sre](267-agent-sre.md), [268-agent-operations-synthesis-2026](268-agent-operations-synthesis-2026.md).

**One-line definition.** A 2026 convergence picture for **agent observability** — the field consolidated on **OpenTelemetry-shape spans** as the wire format with LangGraph / Agents SDK / AutoGen / ADK / Anthropic Agent Teams all emitting OTel-compatible traces, plus a layered ecosystem of consumers — **LangSmith** (per-LangGraph), **Phoenix** (OSS general-purpose), **Helicone** (LLM-call analytics), **Honeycomb / Datadog / Grafana / Cloud Trace** (general APM with agent-trace UX) — and a **typed HIR-shape event taxonomy** (`AgentLoop.start`, `Tool.call`, `Verifier.verdict`, `BrightLine.escalation`, `Memory.write`, `Permission.decision`) layered on top of OTel that gives agent operators the cross-runtime correlation, cost attribution, latency drill-down, and quality-degradation visibility production deployments require.

## Why this paper matters (observability finally has cross-runtime convergence in 2026)

Observability for agents was an unsolved problem in 2024. Each runtime had its own trace schema (LangSmith spans, OpenAI dashboard event types, AutoGen logs, ad-hoc print statements), tracing required separate instrumentation per runtime, and **cross-runtime correlation** (a Polaris orchestrator calling an Anthropic Agent Teams team via A2A through a Cloudflare Tunnel) was effectively impossible without bespoke trace_id propagation. By May 2026 the field has converged on OpenTelemetry as the wire format with three observable shifts: (a) **every major runtime emits OTel spans** natively, (b) **trace_id propagates across A2A boundaries** so a cross-vendor call shows up as one trace, and (c) **typed HIR-shape event taxonomies** layer on top of OTel to give agent-specific structure — `Tool.call` is a span type, not just a generic `function_call`, with attributes like `tool.name`, `tool.cost_usd`, `tool.tokens_in`, `tool.tokens_out`.

The convergence matters because it lets operators answer the questions production agents actually generate: **"this trace cost $4.20 — where did the money go?"** (cost attribution per span), **"this trace took 47 seconds — where was the latency?"** (per-span latency), **"this trace produced a hallucination — which tool call introduced it?"** (per-span quality flagging), **"the verifier escalated to bright-line — what was the precipitating event?"** (bright-line traceback), **"agent A in runtime X invoked agent B in runtime Y — show me the joined trace"** (cross-runtime correlation). Pre-2026 these questions required runtime-specific tooling and bespoke trace stitching; in 2026 they are queries against a unified OTel-shape store.

The ecosystem layered on OTel matters too. **LangSmith** is the polished UX for LangGraph applications with replay, time-travel, evals integrated. **Arize Phoenix** is the OSS alternative — runs locally or as managed service, free for personal use, OTel-native. **Helicone** focuses on LLM-call analytics (cost dashboards, latency P99s, cache hit rates) with proxy-shape integration. **Honeycomb** brings BubbleUp-style observability to agent traces — query high-cardinality attributes (per-skill, per-user, per-routine) to find anomalies. General-purpose APMs (Datadog, Grafana, New Relic) added agent-aware dashboards through 2025-2026.

Take this seriously and three things change. **First**, you stop instrumenting agents per-runtime and start instrumenting **once via OTel** — the same instrumentation works across LangGraph, Agents SDK, AutoGen, ADK, and Anthropic Agent Teams. **Second**, you adopt a **typed HIR-shape event taxonomy** ([16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md)) on top of OTel — `Tool.call`, `Verifier.verdict`, `BrightLine.escalation`, `Permission.decision`, `Memory.write` — so queries are agent-aware, not just generic span queries. **Third**, **cross-runtime trace correlation** becomes the norm — your production-agent platform shows joined traces across runtime, vendor, and machine boundaries via consistent `trace_id` propagation in A2A and MCP headers.

## Problem it solves (uniform observability across the heterogeneous 2026 stack)

1. **Per-runtime trace schemas were incompatible.** Pre-OTel-convergence, observability was lock-in. Adopting OTel means you can swap LangSmith for Phoenix without re-instrumenting.
2. **Cross-runtime correlation was bespoke.** A2A + MCP standards now propagate `traceparent` headers; one trace spans many runtimes.
3. **Cost attribution was per-LLM-call.** OTel-shape spans with `tool.cost_usd` and `llm.tokens_*` attributes let cost drill-down per span.
4. **Latency drill-down required runtime expertise.** Generic OTel UX (Honeycomb BubbleUp, Phoenix waterfall) doesn't require runtime-specific knowledge.
5. **Quality observability was absent.** Verifier-verdict spans + eval-link annotations enable production-eval-feedback loops ([265-agent-evaluation-2026](265-agent-evaluation-2026.md)).
6. **Bright-line escalations had no trace context.** Linking escalation events to trace_id makes incident response straightforward.
7. **Cardinality explosion.** Agent traces have very high cardinality (per-skill, per-user, per-tool); OTel + Honeycomb / Phoenix handle this well.
8. **Long-running trace storage.** Agents run for hours; trace stores need partition by trace start time and lifetime indexing.

## Core idea in one paragraph

The 2026 agent-observability stack has three layers. **Wire format** — OpenTelemetry spans with `traceparent` propagation across A2A ([254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md)) and MCP ([256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md)) headers; every major runtime (LangGraph, Agents SDK, AutoGen, ADK, Anthropic Agent Teams) emits OTel-compatible spans natively. **Typed HIR-shape event taxonomy** — on top of generic OTel spans, agent-specific span types and attributes provide structure: `AgentLoop.{start, step, end}`, `Tool.{call, result}`, `LLM.call`, `Verifier.{request, verdict}`, `Permission.decision`, `BrightLine.escalation`, `Memory.{write, read, evict}`, `Skill.{invoke, complete}`, `Routine.{fire_start, fire_end}`, `Team.{spawn, message, idle, task_completed}` — each with documented attributes (`tool.name`, `tool.cost_usd`, `verifier.verdict`, `bright_line.code`, `permission.action_kind`). **Consumer ecosystem** — LangSmith (LangGraph-shop polished UX), Phoenix (OSS general-purpose), Helicone (LLM-cost analytics), Honeycomb / Datadog / Grafana / Cloud Trace (general-purpose APMs with agent-aware dashboards). The wire format and taxonomy are interoperable; the consumer layer is pluggable. The convergence enables three new operator capabilities: **cost attribution** (drill down `$4.20 per trace` to per-span dollars), **latency drill-down** (the 47s spent in `Tool.call(name='web_search')` is visible), and **cross-runtime correlation** (Polaris orchestrator → ADK Worker via A2A → MCP-exposed tool call all in one trace). The HIR-shape taxonomy is the agent-specific contribution; OTel was already converged for general distributed tracing.

## Mechanism (step by step)

### (a) OTel span emission per runtime

Each runtime emits OTel-compatible spans:

- **LangGraph:** every node execution emits a span with `langgraph.node.name`, state snapshot pre/post, links to parent.
- **OpenAI Agents SDK:** `agent_run`, `function_call`, `handoff`, `guardrail`, `llm_call` span types; auto-uploaded to OpenAI dashboard; OTel custom processors export elsewhere.
- **AutoGen v0.4:** OTel-native by design; spans per actor message handler invocation.
- **Google ADK:** Cloud Trace + Logging native; OTel exporters for non-Google backends.
- **Anthropic Agent Teams:** HIR events written to `~/.claude/teams/{name}/events.jsonl`; OTel bridge available.
- **Custom runtimes:** instrument with OTel SDK directly.

### (b) Trace propagation across A2A and MCP

```python
# A2A request includes W3C traceparent header
headers = {
    "traceparent": f"00-{trace_id_hex}-{span_id_hex}-01",
    "Authorization": f"Bearer {oauth_token}",
}
response = await a2a_client.submit_task(endpoint, headers=headers, capability=..., input=...)

# Receiving agent extracts and continues the trace
incoming_trace_context = extract_traceparent(request.headers)
with tracer.start_as_current_span("a2a.task", context=incoming_trace_context) as span:
    span.set_attribute("a2a.capability", capability)
    span.set_attribute("a2a.task_id", task_id)
    ...
```

Same pattern for MCP: client passes `traceparent`; server continues the trace.

### (c) Typed HIR-shape taxonomy

```python
# Span types with documented attributes
span_taxonomy = {
    "AgentLoop.start": ["agent.id", "agent.runtime", "session.id"],
    "AgentLoop.step": ["step.index", "step.action_kind"],
    "AgentLoop.end": ["end.reason", "step.count", "tokens.total"],
    "Tool.call": ["tool.name", "tool.kind", "tool.args_hash"],
    "Tool.result": ["tool.success", "tool.cost_usd", "tool.duration_ms"],
    "LLM.call": ["llm.model", "llm.tokens_in", "llm.tokens_out", "llm.cost_usd"],
    "Verifier.request": ["verifier.kind", "verifier.target_span_id"],
    "Verifier.verdict": ["verifier.verdict", "verifier.confidence"],
    "Permission.decision": ["permission.action_kind", "permission.verdict", "permission.reason"],
    "BrightLine.escalation": ["bright_line.code", "bright_line.summary"],
    "Memory.write": ["memory.kind", "memory.tier", "memory.size_bytes"],
    "Memory.read": ["memory.query", "memory.hit_count"],
    "Skill.invoke": ["skill.name", "skill.version", "skill.source"],
    "Routine.fire_start": ["routine.id", "routine.trigger_kind"],
    "Routine.fire_end": ["routine.success", "routine.duration_ms"],
    "Team.spawn": ["team.name", "team.size", "team.lead_id"],
    "Team.message": ["team.from", "team.to", "team.kind"],
}
```

Every runtime should emit these span types when applicable; consumers query by span type.

### (d) Cost attribution per span

Every span carries cost attributes; consumer aggregates per-trace, per-user, per-routine, per-skill:

```sql
-- Phoenix / Honeycomb / Datadog query
SELECT
    skill.name,
    SUM(tool.cost_usd + llm.cost_usd) AS total_cost_usd,
    COUNT(*) AS invocation_count,
    AVG(duration_ms) AS avg_latency_ms
FROM agent_spans
WHERE trace_id = '...'
GROUP BY skill.name
ORDER BY total_cost_usd DESC
```

### (e) Latency drill-down via waterfall

Phoenix / LangSmith / Honeycomb show per-span waterfall — visually identifies which Tool.call or LLM.call took the long tail. P99 queries reveal slow-path attribution.

### (f) Cross-runtime correlation example

```
Trace t-abc123:
├─ AgentLoop.start [polaris, runtime=langgraph]
│   ├─ Tool.call [name=literature_search, runtime=langgraph]
│   │   └─ MCP[server=papers-mcp, traceparent propagated]
│   └─ A2A.submit_task [endpoint=adk-research-agent, capability=fact-check]
│       └─ AgentLoop.start [adk-fact-checker, runtime=adk, parent=trace t-abc123]
│           ├─ Tool.call [name=web_search, runtime=adk]
│           └─ LLM.call [model=gemini-2.5-pro]
│       └─ A2A.task_complete
└─ Verifier.request [verifier=cross-channel, runtime=langgraph]
    └─ A2A.submit_task [endpoint=anthropic-team, capability=adversarial-verify]
        └─ ... (continues across runtime boundary)
```

One trace, four runtimes, one query.

### (g) Sampling strategies

High-volume agents generate too many spans for full retention. Strategies:

- **Always sample bright-line escalations and errors.**
- **Random-sample 1% of routine fires.**
- **Tail-based sampling** — keep traces with anomalous latency or cost.
- **Per-user sampling** — debug-mode for specific users keeps full traces.
- **Cardinality-aware sampling** — keep at least N traces per (skill.name, runtime) pair.

### (h) Eval-link annotations

Spans annotated with eval verdicts ([265-agent-evaluation-2026](265-agent-evaluation-2026.md)):

```python
span.set_attribute("eval.verdict", "incorrect")
span.set_attribute("eval.judge", "llm-judge-claude-sonnet-4-6")
span.set_attribute("eval.confidence", 0.87)
```

Closes the production-eval feedback loop — production traces feed back into eval datasets.

## Empirical results (table — May 2026)

| Layer | Tool | Free tier | Best at |
|---|---|---|---|
| Wire format | OpenTelemetry | OSS | Standard wire |
| LangGraph polished UX | LangSmith | Generous free; paid scale | LangGraph-shop production |
| OSS general-purpose | Phoenix | Free local + cloud | Free, polished UX, OTel-native |
| LLM-call analytics | Helicone | Free tier 100k req/mo | Cost dashboards, caching |
| General APM | Honeycomb | Free 20M events/mo | High-cardinality queries |
| General APM | Datadog | Paid | Existing Datadog shops |
| Cloud-vendor | Cloud Trace (GCP) / X-Ray (AWS) / App Insights (Azure) | Pay-per-use | Vendor-stack alignment |

| Convergence indicator | Status May 2026 |
|---|---|
| OTel native in major runtimes | All 6 runtimes |
| `traceparent` propagation in A2A | Yes (v1.0 spec) |
| `traceparent` propagation in MCP | Yes (Streamable HTTP) |
| HIR-shape typed span vocabulary | Emerging convergence |
| Cross-runtime correlation in production | Common at sophisticated shops |

## Variants and ablations

- **Self-hosted Phoenix.** OSS, runs in Docker, no SaaS dependency.
- **LangSmith Cloud.** Managed; enterprise SAML; integration-ready.
- **Helicone proxy.** Sits between agent and LLM API; transparent metering.
- **OTel Collector pipeline.** Forward to multiple backends simultaneously.
- **Trace + Logs + Metrics correlation.** Full three-pillar observability via OTel.
- **High-cardinality dashboards.** Honeycomb BubbleUp; per-(skill, user, model, routine) groupings.
- **Eval-link integration.** [265-agent-evaluation-2026](265-agent-evaluation-2026.md) — production traces feed eval datasets.
- **Cost-router observability feedback.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md) — observed cost informs routing decisions.

## Failure modes and limitations

- **Cardinality explosion.** Per-user, per-skill, per-tool spans grow fast; sampling necessary.
- **Cross-runtime trace_id propagation gaps.** A poorly-instrumented hop loses correlation.
- **Cost attribution noise.** LLM cost varies by model + provider + caching; aggregation requires care.
- **PII in span attributes.** Tool args may contain sensitive data; redaction at instrumentation boundary.
- **Storage cost.** Full trace retention at scale is expensive; tail-based sampling + archive policy.
- **Span schema drift.** Without governance, attribute names drift across teams.
- **Privacy of LLM outputs.** Storing LLM responses in spans needs explicit policy.
- **Real-time alerting on traces.** Span-based alerts have latency vs metric-based alerts.
- **Cross-vendor billing reconciliation.** Cost-attribution depends on vendor billing API.
- **Vendor lock-in via proprietary span types.** LangSmith-specific attributes lose value if you switch.

## When to use, when not

**Adopt the OTel + typed-HIR observability stack** for any production agent deployment with more than a handful of users; for any deployment with cross-runtime calls; for any deployment where cost attribution or quality regression matters; for any deployment with bright-line escalations needing incident response. The strongest cases are **multi-runtime production platforms**, **cost-sensitive deployments**, and **regulated agents** requiring audit trails.

**Skip detailed observability** for single-shot prototypes (overhead exceeds value), local dev (use stdout), or low-stakes single-user agents (basic logs suffice).

## Implications for harness engineering

- **OTel as the wire format for `harness_core/observability/`.** All 14 agents in `projects/` emit the same span types.
- **HIR-shape taxonomy for agent-specific structure.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), [24-observability-tracing](24-observability-tracing.md) — extend with `Verifier.verdict`, `BrightLine.escalation`, `Memory.*`, `Skill.invoke`, `Routine.fire_*`, `Team.*`.
- **Trace propagation through A2A + MCP.** [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md), [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md) — `traceparent` headers everywhere.
- **Cost attribution per span.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md) — observed cost feeds router.
- **Eval-link annotations on production spans.** [265-agent-evaluation-2026](265-agent-evaluation-2026.md), [115-evaluating-llm-systems](115-evaluating-llm-systems.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md).
- **Bright-line escalation traceback.** [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [267-agent-sre](267-agent-sre.md) — incident response from trace.
- **Memory observability.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md) — `Memory.*` spans.
- **Skill observability.** [04-skills](04-skills.md), [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md), [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md).
- **Routine observability.** [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md) — every fire emits trace.
- **Team observability.** [250-anthropic-agent-teams](250-anthropic-agent-teams.md), [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md) — per-teammate spans.
- **Sampling strategy per project.** Per-routine importance, per-user debug mode.
- **Phoenix or LangSmith as default consumer.** Free tiers cover most personal-agent use cases.
- **Cross-runtime correlation for distributed deployments.** [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md) — joined traces across hosts.
- **Cost router feedback loop.** [88-confidence-driven-router](88-confidence-driven-router.md) — observed quality informs routing.

**One-line takeaway for harness designers.** **The 2026 agent-observability stack converged on OpenTelemetry-shape spans plus a typed HIR-shape event taxonomy (Tool.call, Verifier.verdict, BrightLine.escalation, Permission.decision, Memory.* , Skill.invoke, Routine.fire_*, Team.*) with `traceparent` propagation across A2A and MCP — adopt OTel as wire, layer typed taxonomy on top, pick a consumer (LangSmith for LangGraph shops, Phoenix for OSS, Honeycomb for high-cardinality queries), and you get cross-runtime correlation, per-span cost attribution, latency drill-down, and the eval-link feedback loop production agents need.**
