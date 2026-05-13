# 24 — Observability, Tracing & Cost Attribution

**Definition.** Observability for agents is the discipline of making their internal behavior visible, measurable, and billable. The three pillars are **tracing** (end-to-end spans for every LLM call, tool invocation, and sub-agent run), **metrics** (latency, tokens, cost, error rate, step count), and **logs** (the actual prompts, responses, tool I/O). Cost attribution assigns those numbers to users, features, and organizations so spend can be budgeted and controlled.

## Problem it solves

Agents fail in weird, emergent ways. Why did step 17 loop? Why did this conversation cost $40? Where's the latency coming from — the LLM or the slow tool? Without observability you can't answer these questions; without answers you can't improve the agent, control costs, or meet SLOs.

Cost is its own sub-problem. An agent happily runs a hundred tool calls when a human would have stopped at five. A chatty retrieval step can accidentally ship 50k tokens per query. One infinite loop during a demo burns a month's budget. Attribution turns the aggregate number into a per-feature signal you can act on.

## Mechanism

### Tracing

OpenTelemetry-compatible spans are now the standard. Each turn of the agent loop emits a span tree:

```
run
├── llm.chat (model=claude-opus-4, tokens_in=1.2k, tokens_out=0.3k, cost=$0.04)
├── tool.Grep (pattern="TODO", hits=28, duration=220ms)
├── llm.chat ...
├── subagent.Explore (run_id=abc123)  ← linked sub-trace
│   └── ...
└── tool.Edit (...)
```

Key attributes per span:

- `gen_ai.system` (anthropic, openai), `gen_ai.request.model`
- `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.usage.cache_*`
- `agent.run_id`, `agent.step`, `agent.parent_run_id` (for subagents)
- `user.id`, `feature.name`, `session.id` for attribution

Tool spans carry tool name, argument size, result size, and execution duration.

Products: LangSmith (LangChain-native), Langfuse (open-source), Arize Phoenix, Helicone, OpenLLMetry (OTel semantic conventions for LLMs), Honeycomb/Datadog/Grafana via OTel.

### Metrics

Derived time series over the trace stream:

- Token usage per user/day/feature.
- Cost per conversation.
- p50/p95/p99 time-to-first-token, total turn latency.
- Tool error rate; agent-step-budget exhaustion rate.
- Compaction frequency; context-length histogram.
- Judge scores ([21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md)) aggregated over runs.

### Logs

Full prompts and responses are the debugger's gold. Redact secrets before storing; PII-scrub where regulated; keep retention bounded. Sampling (e.g., 1 in 100 full captures plus 100% of failures) balances coverage and cost.

### Cost attribution

Two levels:

- **Per-request** — every LLM call carries `user_id` / `feature` / `tenant_id` so usage rolls up cleanly.
- **Per-run aggregate** — a single agent run sums all its internal LLM + tool costs so you can say "the Resume Optimizer feature costs $0.18 per run on average; P99 is $2.10".

From attribution flow the levers:

- **Caching** (Anthropic prompt caching, OpenAI cached_tokens): 50–90% cost cut on repeated system prompts, RAG contexts, tool schemas.
- **Model routing** (cheap model for simple steps, strong model for reasoning-heavy ones) ([16-plan-and-solve.md](16-plan-and-solve.md)).
- **Step budgets + early stop** — detect runaway loops before they blow the budget.
- **Output compression** (summarize observations, stream only diffs).

## Concrete pattern

Instrumenting an agent loop with OpenTelemetry semantic conventions:

```python
from opentelemetry import trace
tracer = trace.get_tracer("agent")

with tracer.start_as_current_span("agent.run",
    attributes={"agent.run_id": run_id, "user.id": user.id, "feature.name": "coder"}):
    while True:
        with tracer.start_as_current_span("agent.step",
            attributes={"agent.step": step}) as span:
            with tracer.start_as_current_span("llm.chat",
                attributes={
                    "gen_ai.system": "anthropic",
                    "gen_ai.request.model": "claude-opus-4-7",
                }) as llm_span:
                resp = model.chat(...)
                llm_span.set_attribute("gen_ai.usage.input_tokens", resp.usage.input)
                llm_span.set_attribute("gen_ai.usage.output_tokens", resp.usage.output)
                llm_span.set_attribute("gen_ai.cost_usd", compute_cost(resp.usage))

            for call in resp.tool_calls:
                with tracer.start_as_current_span(f"tool.{call.name}",
                    attributes={"tool.args.size": len(json.dumps(call.args))}):
                    result = run_tool(call)
```

Cost dashboard rollups worth having day-1:

- Daily spend × feature × model.
- Top-10 most expensive runs this week (useful for catching loops).
- Cost-per-successful-run (from LLM-as-judge scores joined to traces).

## Variants & related techniques

- **OpenLLMetry / OTel GenAI semantic conventions** — an emerging standard for vendor-neutral instrumentation.
- **Langfuse / LangSmith / Phoenix** — full-stack LLM observability products.
- **Helicone** — proxy-based; minimal code change to capture traces.
- **Prompt caching** (Anthropic) — directly cuts cost and is visible in traces via `cache_read_input_tokens`.
- **Eval + trace integration** — pipe judge scores into trace metadata so you can query "traces where score < 3".
- **Hooks** ([05-hooks.md](05-hooks.md)) — a neat place to emit spans without touching agent loop code.

## Failure modes & anti-patterns

- **Logging secrets.** Full prompt/response capture with raw env var contents. Fix: redaction layer at ingest; deny-list + allow-list.
- **Sampling too aggressively.** 1-in-1000 captures misses 90% of real issues. Fix: always capture failures, anomalous runs, high-cost runs; sample the rest.
- **No run_id correlation.** Subagent spans aren't linked to their parent. Fix: explicit parent/child IDs; propagate via context.
- **Cost dashboards without units.** "This feature costs $1.2k" — per day? per hour? per 1k runs? Fix: dashboards default to per-successful-run and per-day.
- **Metrics without alerts.** Beautiful dashboards, but a 10× cost spike pages nobody. Fix: alert on budget overruns, loop detection, latency regressions.
- **Ignoring tool latency.** LLM tuning improves the 20% of time spent in generation; the 80% in slow tools is the real win. Fix: attribute time to tools, not just LLM.
- **Logging without retention policy.** Unbounded growth; compliance risk. Fix: retention per data class; regular purge.

## When to use (and when not to)

Observability is essentially **mandatory** for any agent that goes past the demo stage. Scale your investment to:

- **Prototype:** capture enough to debug — a simple JSON log of each call is fine.
- **Internal beta:** structured tracing, a dashboard for costs and errors.
- **Production:** OTel tracing, LLM-specific observability product, alerting, eval integration, cost attribution at the user/tenant level.

The only case for *less* is a strictly local, ephemeral tool where users can't or shouldn't share telemetry — even then, local logs help debug.

## References

- LangSmith docs — <https://docs.smith.langchain.com/>
- OpenLLMetry (OTel semantic conventions for LLMs) — <https://github.com/traceloop/openllmetry>
- Langfuse — <https://langfuse.com/>
- Anthropic prompt caching docs — <https://docs.claude.com/en/docs/build-with-claude/prompt-caching>
- Chip Huyen, "AI Engineering" (O'Reilly) — observability and cost chapters.
- Eugene Yan, "Evaluating LLMs and RAG" essays — <https://eugeneyan.com/writing/>
