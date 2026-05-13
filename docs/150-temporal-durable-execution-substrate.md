# 150 — Temporal: Durable Execution as the Production Substrate for Long-Running Agents

**Repository.** https://github.com/temporalio/temporal — MIT license — ~20K stars / 1.5K forks — Go (~99.5%) — current line `v1.31.0` (April 2026) — born as a fork of Uber's *Cadence* and maintained by Temporal Technologies (founded by Cadence's original creators) — SDKs in Go, Java, TypeScript, Python, .NET, PHP, Ruby — managed offering at Temporal Cloud.

**One-line definition.** Temporal is a *durable execution* platform: you write workflow code as if failures don't exist, the platform persists every step of execution to an *event history*, and on any failure (worker crash, network partition, rate-limit, deploy) the workflow *resumes from the last recorded step* — turning multi-step long-running processes (hours, days, weeks, months, years) from "fragile in-memory state machines" into reliable code that runs effectively-once-to-completion, which is exactly the missing substrate underneath production agents that need to survive crashes, retries, and deploys without losing progress.

## Why this chapter matters

Production agents are long-running processes. A research agent runs for 30 minutes; a coding agent runs for hours; a multi-day deep research run takes days; a customer-support agent persists across a multi-week ticket. Each has the same structural problem: the agent's progress lives in memory; if the process crashes, the progress is lost. Naive solutions (write checkpoints to disk, retry on failure, use a queue) re-implement, badly, what mature distributed-systems infrastructure already solves.

Temporal is that mature infrastructure. It is the workflow-engine sibling of what Kubernetes is to compute and what Postgres is to data: *boring, durable, trusted, the right default*. Temporal predates the LLM-agent wave by years (it descended from Uber's Cadence, born 2017–2018) and is in production at Snap, Box, Stripe, Twilio, Coinbase, DoorDash, HashiCorp, Datadog, Netflix, Yum Brands, Airbus, and dozens of others — running everything from financial transactions to media-encoding pipelines to data-engineering DAGs. By 2024–2026 the agent ecosystem started building on top of it: DBOS, Restack.io, Temporal's own AI-agent samples, custom enterprise builds.

For agent builders, the choice is: (1) re-build agent-level production patterns ([124-agent-level-production-patterns](124-agent-level-production-patterns.md)) — checkpointing, idempotency, dead-letter, graceful shutdown — yourself, or (2) adopt Temporal and inherit them as a substrate. Most teams ramping past stage 2 ([118-genai-maturity-models](118-genai-maturity-models.md)) eventually arrive at (2). This chapter is the architectural and operational case for why.

## Problem it solves

Six concrete production-agent problems Temporal addresses:

1. **Crash recovery without bespoke checkpointing.** Naive agents lose state on worker crash; Temporal persists every step to the event history and resumes automatically.
2. **Long-horizon retries on rate limits / outages.** An LLM call that 429s repeatedly, an external API that's down for 10 minutes, a tool that's flaky — Temporal's activity retries with exponential backoff handle each transparently, with no application-side retry code.
3. **Multi-day workflows.** A workflow waiting on a human approval that arrives 3 days later costs zero compute while suspended; classical message-queue approaches require either constant polling or bespoke state machines.
4. **Idempotency by construction.** Each workflow has a workflow ID; duplicate starts are coalesced; activities can be made idempotent via known activity IDs. The whole class of "did the agent send the email twice?" bugs disappears.
5. **Deploy without losing in-flight work.** Worker process restarts (rolling deploy, OS update, host failure) cause Temporal to replay the workflow on a fresh worker; in-flight workflows survive deploys without intervention.
6. **Inspection and intervention.** Workflows can be queried for state and signaled to update behavior — a HITL approval, a configuration change, a graceful cancel — without rewriting the workflow.

Each is structurally the same problem: *application code wants to act like failure doesn't exist, but reality keeps reminding it that it does*. Temporal closes the gap.

## Core idea in one paragraph

Application code should express *what* should happen, not how to handle every failure mode while it happens. Temporal achieves this by separating **workflow code** (deterministic, replay-safe, persistent) from **activity code** (the side-effecting work — API calls, tool invocations, LLM calls, database writes). When a workflow calls an activity, Temporal records a *command* in the workflow's *event history*; when the activity completes, Temporal records the *result*. If the worker crashes mid-workflow, a fresh worker reads the event history and *replays* the workflow code, which deterministically arrives at the same point — and continues from there. The state is the event history; the workflow code is a pure function of it. Activities can retry on failure with configurable policies; workflows can suspend for hours, days, or months while waiting on a signal, a timer, or a child workflow's result, consuming zero compute while suspended. The cost of all this is a *determinism constraint* on workflow code (no non-deterministic operations in the workflow body — only inside activities) and the operational reality of running a Temporal cluster (or paying Temporal Cloud). The benefit is that the entire class of "what happens when X fails?" engineering becomes someone else's problem.

## Mechanism (step by step)

### 1. The two-tier code model — workflows vs activities

```text
Workflow code:     deterministic, replay-safe, no side effects
   - calls activities
   - awaits timers
   - awaits signals
   - spawns child workflows
   - decides control flow

Activity code:     non-deterministic, side-effecting, retried by Temporal
   - HTTP calls, DB writes, LLM API calls, tool invocations
   - file I/O, external system interactions
   - everything that can fail and retry
```

The split is the discipline. *Anything that can fail* lives in an activity. *The orchestration logic* lives in the workflow.

### 2. The event history — durable execution's storage primitive

Every workflow execution has an *event history* — an append-only log of events: workflow started, activity scheduled, activity completed (with result), timer fired, signal received, workflow completed. Stored in the Temporal cluster's database (Cassandra, MySQL, Postgres, or SQLite for dev).

When a worker picks up a workflow task, it does not load "current state". It reads the event history and *replays the workflow code*, applying each event as it goes. The workflow code, being deterministic, arrives at the same point it was at when the last event was recorded. Then it advances.

Replay is the magic. State is the history. The application code never explicitly writes state; the framework writes events.

### 3. Determinism constraints

Workflow code must be deterministic across replays:
- No `random()`, no `now()`, no direct network calls, no direct disk I/O.
- All non-determinism must go through Temporal SDK primitives: `workflow.now()`, `workflow.random()`, `workflow.sleep()`, `workflow.executeActivity()`.
- Versioning APIs (`workflow.getVersion()`) handle code changes between deploys.

The constraint is real but learnable. Most teams adapt within weeks.

### 4. Activities — where side effects live

```text
@activity
async def call_llm(prompt: str) -> str:
    return openai.chat.completions.create(prompt=prompt).text

@activity
async def execute_tool(tool: str, args: dict) -> dict:
    return mcp_client.call(tool, args)

@activity
async def save_to_db(record: dict) -> None:
    db.insert(record)
```

Activities have configurable retry policies, timeouts, heartbeats (for long-running activities), and execution-time limits. Activity failures propagate to the workflow as `ActivityFailure` exceptions, which the workflow can catch and route.

### 5. Signals and queries — interaction with running workflows

```text
@signal
def approve(self):
    self._approved = True

@query
def status(self) -> str:
    return self._current_step

# Externally
client.signal_workflow(workflow_id, "approve")
status = client.query_workflow(workflow_id, "status")
```

A *signal* sends a message into a running workflow (HITL approval, configuration change, cancel). A *query* reads workflow state without affecting execution (status check, observability).

For agents: signals are the primitive for human-in-the-loop ([23-human-in-the-loop](23-human-in-the-loop.md)). Queries are the primitive for the UX progress disclosure ([143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md)).

### 6. Workers and task queues

```text
[Temporal cluster]
   ├── frontend service (gRPC API)
   ├── matching service (task queues)
   ├── history service (event histories)
   └── worker service (internal Temporal workers)

[Your worker processes]
   ├── poll task queues
   ├── execute workflow tasks (replay + advance)
   ├── execute activity tasks (call your code)
   └── return results to cluster
```

Workers are *your code* — programs running the Temporal SDK that connect to the cluster, poll task queues, and execute the work. Multiple workers per task queue scale the work horizontally. Different task queues route work to different worker pools (e.g. one pool for GPU-bound LLM activities, another for cheap CPU activities).

### 7. Architecture in production

```text
Producers (CLI, services, schedulers)
    ↓ start workflows
[Temporal cluster: frontend / matching / history / worker]
    ↓ persists events
[Database: Cassandra / MySQL / Postgres]
    ↓ reads tasks
[Worker processes (your code)]
    ├── workflow workers (replay + advance)
    └── activity workers (call activities)
    ↓ external systems
[LLM APIs, tools, databases, external services]
```

Two layers of operational concern: the Temporal cluster (managed service or self-hosted), and your worker fleet. Temporal Cloud handles the cluster; you operate the workers.

### 8. The agent translation — what each part of an agent maps to

| Agent concept | Temporal concept |
|---|---|
| Agent task | Workflow execution |
| Agent step (LLM call, tool call) | Activity |
| Plan execution loop | Workflow body (with `workflow.executeActivity` per step) |
| Tool failures and retries | Activity retry policies |
| Long-running agent waiting on user | Workflow suspended on signal |
| Subagent delegation | Child workflow |
| Periodic agent task | Schedule |
| Per-task idempotency | Workflow ID |
| Cost / step budget | Activity timeout + workflow execution timeout |
| Agent state at decision time | Event history |
| HITL approval | Signal |
| Trajectory inspection | Query + workflow history viewer |

The mapping is dense and natural. Agents *are* workflows. Tools and LLM calls *are* activities. Everything in [01-agent-loop-architecture](01-agent-loop-architecture.md) maps to Temporal primitives.

### 9. Concrete agent workflow shape

```text
@workflow
class ResearchAgentWorkflow:
    async def run(self, query: str) -> Report:
        # Phase 1: planning (an activity that calls planner LLM)
        plan = await workflow.execute_activity(
            "plan", query,
            start_to_close_timeout=timedelta(seconds=60),
            retry_policy=RetryPolicy(maximum_attempts=3)
        )

        # Phase 2: execute each plan step (each is an activity)
        results = []
        for step in plan.steps:
            r = await workflow.execute_activity(
                "execute_step", step,
                start_to_close_timeout=timedelta(seconds=120),
                retry_policy=RetryPolicy(
                    initial_interval=timedelta(seconds=2),
                    maximum_attempts=5,
                    non_retryable_error_types=["InvalidQuery"]
                )
            )
            results.append(r)

            # HITL gate for high-stakes steps
            if step.requires_approval:
                await workflow.wait_condition(lambda: self._approved)

        # Phase 3: synthesise (activity)
        report = await workflow.execute_activity(
            "synthesize", results,
            start_to_close_timeout=timedelta(seconds=180)
        )

        return report
```

The workflow code expresses the agent loop. Each side-effecting call is an activity. Retries are declarative. HITL is a signal. Crash recovery is automatic. The workflow can run for seconds or for weeks; either way it produces the report exactly once.

### 10. Operational primitives Temporal ships with

- **Schedules** — cron-like recurring workflow starts.
- **Timers** — durable sleep, days/weeks tolerated.
- **Heartbeats** — for long-running activities; if heartbeats stop, Temporal retries.
- **Versioning** — `workflow.get_version()` for safe code changes between deploys.
- **Continue-as-new** — for unbounded-history workflows; rolls history forward.
- **Search attributes** — index workflows for queryability (find by user, by status, by date).
- **Web UI** — visual workflow inspector at port 8233.
- **CLI (`temporal`)** — start workflows, query, signal, list, terminate.
- **Replay test framework** — verify workflow code is deterministic.

Each is a production-essential. Re-implementing each in a custom agent harness is months of work.

## Empirical anchors

- **Production deployments**: Snap, Box, Stripe, Twilio, Coinbase, DoorDash, HashiCorp, Datadog, Netflix, Airbus, Yum Brands — confirmed public users.
- **Repository scale**: ~20K stars, 1.5K forks, ~9000 commits, MIT-licensed.
- **Origin**: forked from Uber's Cadence (2017–2018); maintained by the original creators.
- **SDK coverage**: Go, Java, TypeScript, Python, .NET, PHP, Ruby — broad language support is unusual for workflow engines.
- **Adoption pattern**: most teams arrive at Temporal at stage-2 / stage-3 maturity ([118-genai-maturity-models](118-genai-maturity-models.md)), after the third or fourth time they've built bespoke checkpointing.
- **Agent ecosystem**: DBOS, Restack.io, Temporal's own AI samples, multiple enterprise agent platforms in 2025–2026 build on Temporal.
- **Suspended-workflow cost**: zero compute while waiting; this enables long-horizon agents at sane cost.
- **Replay correctness**: Temporal ships replay-test frameworks; teams that adopt them catch determinism violations in CI.

## Variants and counter-arguments addressed

- **"Just use a job queue (Celery, Sidekiq, BullMQ)."** Job queues handle one-shot tasks; they do not handle multi-step durable orchestration. The first time you build "step 1 → wait 2 days → step 2 → if fail, retry from step 2 not step 1," you've started building a worse Temporal.
- **"Just write your own checkpointing."** Many teams do, badly. The gap from "checkpoint after each step" to "production-grade durable execution" is large; Temporal closes it.
- **"Temporal is too heavy for our agent."** True for prototypes. As soon as the agent runs longer than a few minutes, has multiple steps, or interacts with flaky external systems, the weight earns its place.
- **"We use Airflow / Step Functions / Argo / Cadence / Prefect."** Closest competitors; differences in determinism guarantees, replay model, and developer ergonomics. Temporal's deterministic-replay-with-event-history model is the cleanest of the bunch in 2026; competitors have caught up partially. Migration cost is real either way.
- **"Determinism constraints are restrictive."** They are. Adapting takes weeks; the discipline of "side effects in activities, control flow in workflows" is generally healthy.
- **"What about LangGraph's persistent state?"** LangGraph adds checkpoint-based persistence; closer to a job-queue + manual checkpoint than Temporal's deterministic replay. For LangGraph + Temporal, the agent loop runs in Temporal, the LangGraph node logic runs in activities. Composable, not competing.
- **"Temporal Cloud is locked in."** Temporal-the-server is MIT-licensed and self-hostable; Cloud is convenience. Lock-in is at the workflow-code level, which is portable across self-host and Cloud.

## Failure modes and limitations

1. **Determinism violations.** Workflow code that calls `time.now()` or `random()` directly will replay differently and crash. The replay-test framework catches this in CI; teams that don't run it ship determinism bugs.
2. **Workflow event-history bloat.** Long-running workflows accumulate large histories; activity result sizes compound. Solutions: continue-as-new, externalise large payloads to S3-style storage with references in events, paginate.
3. **Versioning complexity.** Code changes that affect workflow control flow must use `workflow.get_version()` to keep replays of old workflows correct. Forgetting this breaks in-flight workflows.
4. **Cluster operational cost.** Running a self-hosted Temporal cluster (frontend / matching / history / DB) is real ops work. Temporal Cloud removes this at the cost of vendor.
5. **Activity timeouts as the binding constraint.** An activity that runs longer than its `start_to_close_timeout` is killed; long LLM calls or research tasks need careful timeout calibration plus heartbeats.
6. **Learning curve.** "Workflows are deterministic" is the central abstraction; it takes a project or two to internalise.
7. **Latency overhead.** Per-workflow overhead is small (milliseconds) but non-zero; very-low-latency workloads (sub-100ms agent steps) pay a noticeable percentage cost.
8. **Cassandra / MySQL / Postgres backend ops.** Cassandra is the most-deployed backend; not all teams know it well. Postgres is the easiest for small-to-medium scale.
9. **Cost at scale.** Temporal Cloud pricing scales with action count; high-volume short-task workloads pay more than equivalent self-hosted.

## When to use, when not

**Use Temporal as the agent substrate when:**
- Agents run for minutes / hours / days / weeks / months.
- Agents must survive worker crashes, deploys, network partitions.
- Agents need durable retries (LLM rate limits, external-API outages).
- Agents wait on humans (HITL), other agents, or scheduled events.
- Multi-step workflows must execute exactly-once-to-completion.
- The cost of re-implementing checkpointing/idempotency/dead-letter outweighs the cost of adopting Temporal.

**Skip Temporal when:**
- Agent runs once, completes in seconds, no failure-recovery needs.
- Single-shot inference behind an HTTP endpoint (use a normal request handler).
- Prototypes (use the Memento JSONL pattern; revisit at stage 2+).
- Throughput requirements > 100K workflows/sec at very low latency (Temporal scales but at a cost).
- Team unwilling to learn the determinism model (a real organisational cost).

**Defer until stage-2 transition** ([118-genai-maturity-models](118-genai-maturity-models.md)): premature adoption is overhead; late adoption is rework. The signal to migrate is when bespoke checkpointing becomes a maintenance burden.

## Implications for harness engineering

- **Temporal replaces a class of agent-harness code you'd otherwise write.** Specifically: [124-agent-level-production-patterns](124-agent-level-production-patterns.md)'s checkpointing, idempotency, dead-letter, graceful shutdown become substrate features, not application code. This is a 30–50% reduction in agent-harness production code for stage-3+ deployments.
- **Workflows = agent loops; activities = tools and LLM calls.** Adopt this mapping as the architectural rule. The workflow body is the planner-and-orchestration logic; every external interaction is an activity. See [01-agent-loop-architecture](01-agent-loop-architecture.md), [108-memento-codebase-mcp](108-memento-codebase-mcp.md).
- **Activities are MCP tools.** A natural pattern: each MCP tool ([07-model-context-protocol](07-model-context-protocol.md)) wraps to an activity, with retry policy and timeout per tool. The `documents_tool`, `search_tool`, `code_agent` from Memento's repo each map cleanly.
- **Signals are the HITL primitive.** [23-human-in-the-loop](23-human-in-the-loop.md). Approval workflows wait on signals; emit a notification; resume on signal. No polling, no bespoke state.
- **Queries are the observability primitive.** Combine with [143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md)'s progress-disclosure UX: the UI queries the running workflow for "current step" and renders.
- **Schedules are the periodic-agent primitive.** Cron-like recurrences without a separate scheduler service; useful for [56-sea-landscape-2026](56-sea-landscape-2026.md)-style continuously-evolving agents.
- **Child workflows are the multi-agent primitive.** [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md)'s hierarchical orchestrator-specialist maps cleanly: parent workflow spawns child workflows for specialists; awaits their completion.
- **Workflow ID is the idempotency primitive.** Each agent task has a workflow ID; duplicate starts are coalesced. Solves the "did we run this twice?" class of bug structurally.
- **Replay-test framework is the regression-test primitive.** Workflow code changes can be tested against past event histories before deploy; catches determinism violations and behavior regressions.
- **Audit trail is built in.** The event history *is* the audit log ([122-explainability-compliance](122-explainability-compliance.md)). Combined with retention policy, satisfies most regulatory recording requirements out of the box.
- **Pair with the model gateway.** [119-agent-ready-llm-selection](119-agent-ready-llm-selection.md), [147-vendor-lock-in](147-vendor-lock-in.md). LLM-call activities go through the gateway; retry policies handle vendor outages; fallback paths are activities themselves.
- **Pair with eval harness.** [115-evaluating-llm-systems](115-evaluating-llm-systems.md). Temporal workflows are inspectable; eval can pull histories from completed runs for trajectory analysis.
- **The build-vs-adopt rule applies sharply here.** [144-build-your-own-harness](144-build-your-own-harness.md). Adopting Temporal for the durable-execution layer while building your differentiating agent loop on top is the canonical wrap-not-write pattern.

The one-sentence takeaway: **Temporal is the durable-execution substrate underneath production agents — workflows are agent loops, activities are tools and LLM calls, replay handles crashes, signals handle HITL, and an entire class of agent-harness code (checkpointing, idempotency, dead-letter, retry) becomes substrate features rather than application code.**

## See also

- [01-agent-loop-architecture](01-agent-loop-architecture.md) — the agent loop maps to a Temporal workflow.
- [05-hooks](05-hooks.md), [07-model-context-protocol](07-model-context-protocol.md) — activities are the natural home for MCP tool calls and pre/post hooks.
- [10-multi-session-continuity](10-multi-session-continuity.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md) — multi-session continuity is built-in for Temporal workflows.
- [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md) — verifier patterns sit naturally as workflow steps.
- [23-human-in-the-loop](23-human-in-the-loop.md) — Temporal signals are the HITL primitive.
- [24-observability-tracing](24-observability-tracing.md) — the event history is the trace; pair with OpenTelemetry for cross-system tracing.
- [108-memento-codebase-mcp](108-memento-codebase-mcp.md) — Memento's planner-executor loop is a natural fit to migrate onto Temporal.
- [114-workflows-vs-agents](114-workflows-vs-agents.md) — Temporal blurs the distinction; bounded agents are workflows-with-flexibility.
- [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md) — child workflows implement the hierarchical pattern.
- [122-explainability-compliance](122-explainability-compliance.md) — event history is the built-in audit trail.
- [123-robustness-fault-tolerance](123-robustness-fault-tolerance.md) — Temporal ships retry, circuit-breaker-style timeouts, and graceful degradation primitives.
- [124-agent-level-production-patterns](124-agent-level-production-patterns.md) — Temporal substrates checkpointing, idempotency, dead-letter, graceful shutdown.
- [125-system-level-production-patterns](125-system-level-production-patterns.md) — Temporal Cloud or self-hosted cluster is the platform-level concern.
- [144-build-your-own-harness](144-build-your-own-harness.md) — adopt Temporal as the durable-execution layer; build your differentiation on top.
- [147-vendor-lock-in](147-vendor-lock-in.md) — Temporal-the-server is MIT-licensed and self-hostable; the workflow-code abstraction is portable.
