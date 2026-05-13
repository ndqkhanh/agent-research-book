# 266 — Agent Durability and Idempotency: LangGraph checkpointer × Temporal × Restate × Inngest, and the idempotency-key discipline

**Anchors.** Temporal — https://temporal.io/ (deep-dive in [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md)). LangGraph checkpointer — [259-langgraph-deep-dive](259-langgraph-deep-dive.md). Restate — https://restate.dev/ (durable-execution-as-a-service, 2024 launch). Inngest — https://www.inngest.com/ (durable workflows for serverless). Cloudflare Workflows — https://developers.cloudflare.com/workflows/ (2024 GA, edge-native durable). Companions: [263-production-agent-runtime-synthesis-2026](263-production-agent-runtime-synthesis-2026.md), [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md), [265-agent-evaluation-2026](265-agent-evaluation-2026.md), [267-agent-sre](267-agent-sre.md), [268-agent-operations-synthesis-2026](268-agent-operations-synthesis-2026.md), [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md).

**One-line definition.** A 2026 picture of **durable execution and idempotency for agents** — beyond Temporal ([150](150-temporal-durable-execution-substrate.md)) the field has **multiple credible options** (LangGraph checkpointer for in-runtime durability, Restate as durable-execution-as-a-service, Inngest for serverless durable workflows, Cloudflare Workflows for edge-native, custom event-sourcing for bespoke needs), each with different trade-offs across **operational complexity, latency overhead, vendor lock-in, and pricing** — and the **idempotency-key discipline** (deterministic actions, idempotency keys on tool calls, dedup via DB unique constraints, saga/outbox transactional patterns) becomes the load-bearing engineering discipline that makes agents safe to retry, replay, and recover from crashes mid-run; without these primitives, the "agent crashed at step 47 of 100" failure mode either loses 47 steps of work or commits 47 steps twice on retry.

## Why this paper matters (durability is the production-quality multiplier; without it, agents stay prototypes)

Production agents that run for hours or days face a problem prototype agents do not: **what happens when the process crashes at step 47?** Without durability, you lose 47 steps of work and rerun from start; with naive retry, you commit 47 steps twice (sending duplicate emails, double-charging accounts, creating duplicate database rows). The combination of **durable execution** (state checkpointed, resumable from the last checkpoint) and **idempotency** (each effectful action has a key that prevents duplicate execution) is the engineering discipline that distinguishes production agents from prototypes. The 2026 ecosystem has multiple credible options for both halves; picking and integrating them is a load-bearing decision.

The durability landscape stabilized in 2024-2026. **Temporal** ([150](150-temporal-durable-execution-substrate.md)) is the canonical option — workflow-as-code with deterministic replay, signals, queries, child workflows, child workflow handles, query API, signal API; production-tier, used by Snap / Coinbase / Box / DoorDash / many others. **LangGraph checkpointer** ([259](259-langgraph-deep-dive.md)) is the in-runtime alternative — Postgres-backed state checkpoints per node; lighter-weight than Temporal, well-fit for LangGraph workflows. **Restate** is the durable-execution-as-a-service entrant; turns any function into a durable workflow with annotation-shape syntax. **Inngest** is the serverless-native option for workflows that fit the function-as-a-service shape. **Cloudflare Workflows** is the edge-native option for global low-latency durable execution. Custom event-sourcing (CQRS) remains an option for teams with specific needs.

The idempotency-key discipline is more universal. **Every effectful tool call should carry an idempotency key** — a deterministic-from-input hash that the receiving service uses to dedup. **Saga patterns** (compensating actions) handle the case where a multi-step operation fails partway. **Outbox patterns** (write to DB + outbox in same transaction; deliver from outbox) ensure exactly-once delivery semantics. **Replay safety** requires deterministic functions; non-determinism (random IDs, timestamps, network calls) must be sandboxed.

Take this seriously and three things change. **First**, **agent runs become durable** — a process crash at step 47 resumes at step 48, no work lost, no duplicates. **Second**, **idempotency keys become a habit** in tool design — every external action has a key; receivers dedup; retries are safe. **Third**, **saga and outbox patterns** become the multi-step transactional primitives — agents that touch external systems can do so atomically across multiple calls.

## Problem it solves (production agents that crash, retry, and resume safely)

1. **Crash at step 47 = lost work.** Without durability, restart from step 0.
2. **Naive retry = duplicate side effects.** Without idempotency, retrying a "send email" tool call sends two emails.
3. **Multi-step transactions need atomicity.** Without saga/outbox, partial-success states leave the system inconsistent.
4. **Replay must be deterministic.** Non-determinism in agent code makes replay diverge from original execution.
5. **Long-running agents exhaust runtime memory.** Without checkpointing, hour-long agents run on cliff edges.
6. **Resume across deploys.** A new deployment kills running agents; durable workflows resume in the new process.
7. **Distributed agent crashes.** [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md) — durable state shared across hosts via NATS JetStream + Postgres.
8. **Bright-line escalation pause.** [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — escalation pauses durably; user resumes from any state.

## Core idea in one paragraph

Agent durability decomposes into three primitives. **Durable execution** — the agent's state (variables, partial results, message history) is checkpointed at well-defined boundaries (per-node in LangGraph, per-step in Temporal, per-task in Restate); on crash, the runtime resumes from the latest checkpoint, replays minimally, and continues. The implementation choices in 2026 are LangGraph checkpointer (in-runtime, Postgres-backed, simplest), Temporal (workflow-as-code, deterministic-replay, production-tier), Restate (durable-execution-as-a-service, function-annotation syntax), Inngest (serverless-native), Cloudflare Workflows (edge-native), or custom event-sourcing for bespoke needs. **Idempotency** — every effectful tool call (database write, email send, API call, file modification) carries an **idempotency key** deterministic from the call's inputs; the receiving service uses the key to dedup duplicate retries. The pattern: `idem_key = hash(routine_id + step_index + tool_args_canonical)`; tool implementation: `if exists(idem_key) return prior_result else compute_and_store(idem_key, result)`. **Replay safety** — agent code must be deterministic with respect to the durable boundary; non-determinism (random UUIDs, current timestamp, external API calls without idempotency) is "isolated" by being recorded at the durability boundary so replay reproduces it. The combination of durable execution + idempotency + replay safety is what makes agents production-grade. The trade-off is operational complexity: every layer adds infrastructure (Postgres for LangGraph, Temporal cluster, Restate runtime, Inngest account, etc.), latency overhead (per-checkpoint write), and code discipline (idempotency keys, deterministic functions). The 2026 production sweet spot for personal/team agents is **LangGraph checkpointer + Postgres** for state; **idempotency keys** on every effectful tool; **saga/outbox** for multi-step external transactions. For high-scale enterprise, **Temporal** is still the industry standard.

## Mechanism (step by step)

### (a) Durable execution options

| Option | Architecture | Best for | Operational cost |
|---|---|---|---|
| **LangGraph checkpointer** | In-runtime, Postgres-backed | LangGraph workflows; in-house | Postgres |
| **Temporal** ([150](150-temporal-durable-execution-substrate.md)) | Workflow-as-code, deterministic replay, separate cluster | Production-tier, polyglot | Cluster ops or Temporal Cloud |
| **Restate** | Durable-execution-as-a-service, annotation syntax | Newer projects, less ops burden | SaaS or self-host runtime |
| **Inngest** | Serverless durable workflows | Function-as-a-service shape | SaaS, free tier |
| **Cloudflare Workflows** | Edge-native, global low-latency | Edge-deployed agents | Cloudflare account |
| **Custom event-sourcing (CQRS)** | Custom; full control | Bespoke requirements | High |

### (b) LangGraph checkpointer pattern

```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string("postgresql://...")
app = graph.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "user-123-job-456"}}

# First run — checkpoints at every node
async for event in app.astream({...}, config):
    ...

# Crash. Process restart. Resume from last checkpoint:
async for event in app.astream(None, config):
    ...
```

### (c) Temporal workflow-as-code pattern

```python
from temporalio import workflow, activity

@workflow.defn
class ResearchWorkflow:
    @workflow.run
    async def run(self, topic: str) -> str:
        papers = await workflow.execute_activity(
            search_papers, topic,
            start_to_close_timeout=timedelta(seconds=30),
            retry_policy=RetryPolicy(maximum_attempts=3),
        )
        summary = await workflow.execute_activity(
            summarize_papers, papers,
            start_to_close_timeout=timedelta(minutes=5),
        )
        return summary
```

The runtime records every activity invocation; on replay, it returns recorded results without re-executing activities (deterministic replay).

### (d) Idempotency keys on tool calls

```python
import hashlib

def idempotency_key(routine_id: str, step_index: int, tool_args: dict) -> str:
    canonical = json.dumps({"routine": routine_id, "step": step_index, "args": tool_args}, sort_keys=True)
    return hashlib.sha256(canonical.encode()).hexdigest()

@function_tool
async def send_email(to: str, subject: str, body: str, *, idem_key: str = None):
    idem_key = idem_key or idempotency_key(...)
    # Check dedup table
    if existing := await db.execute("SELECT result FROM email_sends WHERE idem_key = ?", idem_key):
        return existing.result
    # Send
    result = await email_provider.send(to, subject, body)
    await db.execute("INSERT INTO email_sends (idem_key, result) VALUES (?, ?)", idem_key, result)
    return result
```

Every effectful tool gets this pattern. The dedup table prevents duplicate execution on retry.

### (e) Saga pattern for multi-step transactions

```python
@workflow.defn
class BookingSaga:
    @workflow.run
    async def run(self, request: BookingRequest):
        compensations = []
        try:
            flight = await workflow.execute_activity(reserve_flight, request)
            compensations.append(lambda: workflow.execute_activity(cancel_flight, flight))

            hotel = await workflow.execute_activity(reserve_hotel, request)
            compensations.append(lambda: workflow.execute_activity(cancel_hotel, hotel))

            payment = await workflow.execute_activity(charge_payment, request, flight, hotel)
            return BookingResult(flight=flight, hotel=hotel, payment=payment)
        except Exception as e:
            for compensation in reversed(compensations):
                await compensation()  # best-effort rollback
            raise
```

Compensating actions provide rollback semantics for multi-step external operations.

### (f) Outbox pattern for exactly-once delivery

```python
async def book_and_notify(request):
    async with db.transaction():
        booking_id = await db.execute("INSERT INTO bookings ... RETURNING id")
        await db.execute(
            "INSERT INTO outbox (event_type, payload, status) VALUES (?, ?, 'pending')",
            "booking_created",
            {"booking_id": booking_id, "user_email": request.email},
        )

# Separate worker drains outbox:
async def outbox_worker():
    while True:
        events = await db.execute("SELECT * FROM outbox WHERE status = 'pending' LIMIT 10")
        for event in events:
            await deliver(event)
            await db.execute("UPDATE outbox SET status = 'delivered' WHERE id = ?", event.id)
```

Atomic write of state + outbox; separate delivery process. Exactly-once at the consumer's idempotency boundary.

### (g) Replay safety — non-determinism isolation

```python
# Bad — non-deterministic, breaks replay:
async def my_workflow():
    request_id = str(uuid4())  # different UUID on replay → divergence
    timestamp = datetime.now()
    result = await api.call(...)  # different result on replay → divergence

# Good — non-determinism captured at boundary:
async def my_workflow():
    request_id = await workflow.uuid4()  # recorded
    timestamp = await workflow.now()  # recorded
    result = await workflow.execute_activity(api_call, ...)  # recorded
```

The runtime records the result of each non-deterministic operation at first execution; on replay, it returns the recorded value.

### (h) Distributed durability via NATS JetStream

[253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md) — JetStream durable streams provide replay log; consumer position persists across crashes. Combine with Postgres checkpointer for full distributed durability.

## Empirical results (table — May 2026)

| Tool | Model | Free tier | Production scale |
|---|---|---|---|
| Temporal | Self-hosted cluster or Cloud | OSS free; Cloud paid | Trillions of executions/year industry-wide |
| LangGraph checkpointer | Postgres-backed | OSS free | Sufficient for most personal/team-scale |
| Restate | Self-host or Cloud | OSS, Cloud paid | Mid-scale |
| Inngest | SaaS, serverless | Generous free tier | Mid-scale, function-shape |
| Cloudflare Workflows | Cloudflare account | Free tier | Edge global scale |
| Custom event-sourcing | DIY | n/a | Bespoke |

## Variants and ablations

- **Hybrid LangGraph + Temporal.** LangGraph for in-process state; Temporal for cross-system orchestration.
- **Idempotency at LLM-call layer.** [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md) — MCP tool calls carry idempotency keys.
- **Idempotency at A2A layer.** [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md) — A2A `Task` UUID is the idempotency key.
- **Outbox over NATS JetStream.** [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md) — JetStream as the outbox.
- **Saga via Restate.** Restate's annotation syntax is well-suited to saga patterns.
- **Snapshots vs event log.** Snapshot full state periodically vs replay full event log; trade-off.
- **Cross-region durability.** Cloudflare Workflows for edge; Temporal multi-region for cluster.
- **Replay-as-debugger.** [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md), [259-langgraph-deep-dive](259-langgraph-deep-dive.md) — replay traces for debugging.

## Failure modes and limitations

- **Idempotency key collision.** Bad hashing → false dedup. Use canonical-form serialization.
- **Idempotency dedup table growth.** Indefinite retention exhausts DB; TTL eviction needed.
- **Replay divergence.** Non-determinism leaks → replay produces different result; debugging hell.
- **Saga compensation failures.** Best-effort rollback can fail; manual intervention required.
- **Outbox worker lag.** Slow drain → user-visible delay.
- **Postgres pressure on checkpointer.** Per-node row writes scale linearly; partition / archive needed.
- **Temporal cluster ops complexity.** Self-host cluster is non-trivial; Temporal Cloud is the easier path.
- **Cross-vendor durability gaps.** A2A task UUID is the dedup key, but receiving runtime must respect it.
- **Vendor lock-in via durable-execution-as-a-service.** Restate / Inngest / Cloudflare Workflows code doesn't trivially port.
- **Long retention costs.** Years of trace data is expensive; tier to cold storage.
- **Time-travel cost.** Storing checkpoints for time-travel is storage-heavy.
- **State serialization size.** Large state blobs (10 MB+) bloat checkpoints; reference external storage.

## When to use, when not

**Adopt durable execution + idempotency** for any production agent that runs for more than a few seconds; any agent that touches external systems; any agent with bright-line escalation needs ([14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md)); any multi-step transactional workflow; any agent users pay for. The strongest cases are **scheduled routines** ([252](252-routines-pattern-for-self-hosted-agents.md)), **long-running research workflows**, **multi-day approval pipelines**, and **financial / regulated agents** where every action needs audit + retry safety.

**Skip durable execution** for one-shot prompts, ephemeral interactive agents (chat where each turn is a fresh request), or prototypes where the cost of building durability exceeds the value. **Skip idempotency** never — every effectful tool call should carry a key, regardless of durability layer.

## Implications for harness engineering

- **Pick durable-execution layer per project.** [263-production-agent-runtime-synthesis-2026](263-production-agent-runtime-synthesis-2026.md) — LangGraph checkpointer for LangGraph apps, Temporal for polyglot, Restate/Inngest/CF Workflows per ops preference.
- **Idempotency keys on every effectful tool.** [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md) — MCP tool implementations check dedup.
- **Saga/outbox patterns for multi-step external.** [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md).
- **Replay safety in agent code.** No `uuid4`, `datetime.now()`, raw network calls; everything through workflow boundary.
- **Bright-line escalation pauses durably.** [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md) — pause survives crashes; user resumes.
- **Routines as durable workflows.** [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md) — fire creates a durable workflow instance.
- **Distributed durability via NATS JetStream + Postgres.** [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md).
- **Observability on durable boundaries.** [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md) — span per checkpoint, span per saga compensation.
- **Eval against replay.** [265-agent-evaluation-2026](265-agent-evaluation-2026.md) — replay production traces against new model/prompt.
- **Cost-router with retry semantics.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md) — retries don't double-count cost.
- **Cross-channel verifier on durable boundary.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md).
- **Memory writes are idempotent.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md) — write-with-idem-key.
- **Permission Bridge gates retries.** [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md) — retries auto-allowed if first call was approved.
- **Trajectory simulation as replay.** [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md), [195-argus-omega-vol-2-trajectory-temporal-horizon](195-argus-omega-vol-2-trajectory-temporal-horizon.md).

**One-line takeaway for harness designers.** **Production agents need durable execution (LangGraph checkpointer for in-runtime, Temporal for cross-system, Restate / Inngest / Cloudflare Workflows for SaaS-shape) plus idempotency keys on every effectful tool plus saga/outbox patterns for multi-step external transactions plus replay safety via non-determinism isolation — without these primitives, agents that crash mid-run lose work or commit duplicates, and the difference between a prototype and a production-grade agent is exactly this discipline.**
