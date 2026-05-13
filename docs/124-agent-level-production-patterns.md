# 124 — Agent-Level Production Patterns: Cold Start, Warm-Up, Graceful Shutdown, State Checkpointing, Dead-Letter Handling

**Sources.** Arsanjani & Bustos, *Agentic Architectural Patterns*, Chapter 9 (Agent-Level Patterns); plus the broader production-systems literature (Heroku 12-factor app methodology, Kubernetes lifecycle management, distributed-systems checkpointing).

**One-line definition.** Agent-level production patterns are the per-agent operational concerns — *cold start*, *warm-up*, *graceful shutdown*, *state checkpointing*, *idempotency*, *dead-letter handling* — that make a single agent process production-grade; they are the agent-system equivalent of "is this microservice healthy?" and the patterns are mostly translations of classic service operations into the LLM-agent context, with the wrinkle that an agent's "state" includes not just code state but trajectory state, memory state, and tool state.

## Why this matters

A demo agent runs in a notebook, with no concern for what happens when the process restarts, when half a task has been done, when memory needs to survive a deploy, when 100 simultaneous requests arrive. A production agent must handle all of that. The patterns here are the operational primitives that turn "the agent works" into "the agent works at 3am on a holiday after a partial deploy with three requests queued."

For agent builders moving from stage 1 to stage 2 (per [118-genai-maturity-models](118-genai-maturity-models.md)), these patterns are the productionisation layer. They're not glamorous — no new model capability, no clever prompting — but their absence is what makes "deployable" become "actually deployed."

This chapter is the catalog: what each pattern is, what it solves, how to implement it for an LLM agent specifically, and what the failure mode is when you skip it.

## Problem it solves

Six recurring production headaches:

1. **Slow first call.** First request after deploy takes 30 seconds because retriever index, model adapters, and tool clients all load lazily.
2. **Restart loses progress.** Agent was on step 30 of 50; process restarts; task fails or restarts from scratch.
3. **Deploy interrupts in-flight work.** Rolling deploy kills agents mid-task; users get errors.
4. **Stuck queue.** A failed task keeps getting re-tried, blocking the queue.
5. **Duplicate work.** Same task processed twice because of an idempotency bug.
6. **Resource leaks.** Agents hold tool connections open; over time, the process degrades.

Each maps to one of the patterns below.

## Core idea in one paragraph

Production agents are processes (or threads, or pods) with lifecycles. A complete agent-lifecycle catalog: **cold start** (load model adapters, retriever indexes, tool clients before serving) makes first-request latency predictable. **Warm-up probes** verify all dependencies are ready before declaring the agent healthy. **Graceful shutdown** lets in-flight tasks complete (or checkpoint) before the process terminates. **State checkpointing** writes trajectory + memory + tool state to durable storage at intervals so a restart can resume. **Idempotency keys** ensure the same task processed twice doesn't double its effects. **Dead-letter handling** routes unprocessable tasks to a quarantine queue with operator notification. **Health checks** distinguish "alive but not ready" from "ready" from "dead." **Resource hygiene** closes connections, releases memory, drops caches at task boundaries. Layered, these patterns produce an agent that runs in production for months without operator intervention.

## Mechanism (step by step)

### 1. Cold start — predictable first-request latency

```text
on process start:
    1. Load model gateway connections (verify auth)
    2. Load retriever index into memory (or attach to remote)
    3. Load PEFT adapters into model server
    4. Initialise MCP tool clients (connect, verify schema)
    5. Run smoke test: a known input → known output
    6. Mark health check "ready"
```

Two principles:
- **Eager load.** All dependencies up front; no lazy loading on the request path.
- **Smoke test before ready.** A known-good test must pass before the agent serves real traffic.

For LLM agents specifically: the most common cold-start delays are retriever index load (seconds to minutes for large indexes) and model warmup (first few requests are slower).

### 2. Warm-up probes — staged readiness

Three readiness states:
- **Live.** Process is up; responding to ping.
- **Warming.** Process is up but dependencies not ready.
- **Ready.** All dependencies ready; serving traffic.

Kubernetes liveness/readiness probes implement this directly. The agent must distinguish them; load balancers route only to "ready" instances.

### 3. Graceful shutdown — finish or checkpoint

```text
on SIGTERM:
    1. Stop accepting new requests (load balancer drains).
    2. Wait for in-flight tasks to complete (with timeout).
    3. For tasks that don't finish: checkpoint state, requeue task.
    4. Close tool connections.
    5. Flush logs.
    6. Exit.
```

Drain timeout is workload-dependent: 30s for short interactive tasks, 5+ minutes for long agent runs. Beyond timeout, force-checkpoint and requeue.

### 4. State checkpointing — survive restarts

Per-task state to checkpoint:

```json
{
  "task_id": "...",
  "agent_version": "v3.2",
  "step": 23,
  "trajectory": [...],
  "scratchpad": "...",
  "tool_state": {"db_conn": "..."},  // serialised tool state
  "ts": "..."
}
```

Frequency: after every step, or every N steps, or every M seconds. Trade-off: more frequent = more overhead + smaller restart-loss; less frequent = lower overhead + larger restart-loss.

Restart logic:
```text
on task start:
    if checkpoint exists for this task_id:
        load trajectory, scratchpad, tool_state
        resume from step+1
    else:
        start from scratch
```

This is the agent equivalent of [10-multi-session-continuity](10-multi-session-continuity.md) at the *operational* level rather than the *user* level.

### 5. Idempotency — safe retries

Every task has an *idempotency key* (a hash of inputs, or a client-supplied ID). The harness records: "I've started/completed task with key X." On retry:

```text
on task with idempotency_key K:
    if K is in completed: return cached result
    if K is in in_progress: wait or reject as duplicate
    else: proceed; mark as in_progress
```

Idempotency is critical when the agent has *side effects* (sends email, updates database, triggers actions). Without it, retries cause duplicate effects.

### 6. Dead-letter handling — quarantine for poison tasks

A task that fails repeatedly cannot continue blocking the queue:

```text
on task failure:
    failure_count += 1
    if failure_count >= MAX:
        move task to dead-letter queue
        notify operator
        proceed with next task
    else:
        retry with backoff
```

Dead-letter queue characteristics:
- **Operator-visible.** Dashboard or alert.
- **Inspectable.** Operator can examine the failed task.
- **Replayable.** After fix, operator can replay.
- **Bounded.** Doesn't grow unbounded; archive or expire.

### 7. Resource hygiene — preventing leaks

Per-task resource lifecycle:
- **Acquire on use.** Tool connection, file handle, GPU lease.
- **Release on completion.** Including on exception.
- **Reset on task boundary.** Clear scratchpad, prompt cache, intermediate state.

Long-running agents accumulate state: connection pool grows, log buffers fill, cache pollutes. Without explicit hygiene, processes degrade over hours.

### 8. Multi-tenancy isolation

If the agent serves multiple tenants:
- **Per-tenant rate limits.** One tenant cannot starve others.
- **Per-tenant cost budgets.** One tenant cannot consume the global LLM quota.
- **Per-tenant data isolation.** No leakage of trajectories, memory, or retrieved context.
- **Per-tenant feature flags.** Different tenants on different feature versions.

Multi-tenancy is the difference between a SaaS product and an internal tool.

### 9. Composition

```text
[load balancer]
   ↓ (health check: ready?)
[agent process]
   ↓
[receive task]
   ↓
[idempotency check]
   ↓
[task: load checkpoint if exists]
   ↓
[agent loop with periodic checkpointing]
   ↓
[mark complete; release resources]
   ↓
[on failure: dead-letter after MAX retries]
```

Plus, in parallel: graceful shutdown handler, health probes, resource hygiene.

## Empirical anchors

- **Cold-start time** for a typical agent: 5–30 seconds; with eager loading and warm caches, 1–5 seconds.
- **Long-horizon agents** without checkpointing lose 5–10% of tasks to restarts.
- **Idempotency bugs** cause duplicate effects in 1–2% of retried tasks; in side-effect-bearing systems, this is unacceptable.
- **Dead-letter queues** typically receive 0.1–1% of traffic; spikes signal upstream problems.
- **Multi-tenancy isolation failures** are the #1 SaaS-agent incident category.
- **Resource leaks** manifest as gradual latency creep over hours/days; restart "fixes" them but masks the underlying bug.

## Variants and counter-arguments addressed

- **"Cold start doesn't matter; we're behind a load balancer."** Until you scale to zero or deploy frequently. Then it dominates.
- **"Just restart on failure."** Without checkpointing, restart = lose progress = bad UX.
- **"Idempotency is hard."** It's mechanical. Idempotency keys + a lookup table.
- **"Dead-letter is overkill."** It's the only thing preventing one bad task from blocking the entire pipeline.
- **"Multi-tenancy isolation is just resource limits."** It's also data isolation, audit isolation, eval isolation. More than rate limits.

## Failure modes and limitations

1. **Lazy loading bugs.** Cold-start latency creeps because someone added a lazy-loaded dependency.
2. **Checkpoint serialisation issues.** Tool state with non-serialisable handles; checkpoint write fails silently.
3. **Idempotency key collisions.** Bad hash function; different tasks share keys.
4. **Dead-letter accumulation.** No one watches the dead-letter queue; problems compound silently.
5. **Graceful-shutdown timeout too short.** Tasks killed mid-step; checkpointing didn't catch them.
6. **Resource hygiene gaps.** Some path doesn't release resources; leak slowly.
7. **Multi-tenant cross-talk.** Cache pollution, prompt-template leakage, retrieval results from wrong tenant.
8. **Health-check theatre.** Health endpoint returns 200 even when the agent is broken; load balancer keeps routing.

## When to use, when not

**All eight patterns** are mandatory for stage-2+ production agents ([118-genai-maturity-models](118-genai-maturity-models.md)).

**Subset (cold start + idempotency + graceful shutdown)** is the minimum viable for any production agent.

**Skip checkpointing for** very short tasks (< 10s) where restart-from-scratch is cheap.

**Skip multi-tenancy for** internal-only single-tenant deployments.

## Implications for harness engineering

- **Make the agent a 12-factor process.** Configuration via env vars; logs to stdout; ephemeral filesystem; explicit dependencies.
- **Build cold start into the deploy pipeline.** Run smoke tests on every deploy; gate traffic until ready.
- **Checkpoint is a harness primitive.** [12-todo-scratchpad-state](12-todo-scratchpad-state.md), [09-memory-files](09-memory-files.md) are user-level; this chapter is operational.
- **Idempotency at the harness layer.** Before the agent starts, the harness checks the key.
- **Dead-letter queue with operator dashboard.** Engineering or ops owns this; visible.
- **Resource hygiene in hooks.** Per-task setup/teardown in [05-hooks](05-hooks.md).
- **Health checks distinguish live/warming/ready.** Kubernetes-style probes.
- **Multi-tenant isolation as a hard requirement** for any external-facing agent product.
- **Test the lifecycle.** Restart the agent mid-task in a chaos test ([53-chaos-engineering-next-era](53-chaos-engineering-next-era.md)) and verify recovery.

The one-sentence takeaway: **agent-level production patterns are the operational primitives — cold start, warm-up, graceful shutdown, checkpointing, idempotency, dead-letter, resource hygiene, multi-tenancy isolation — that turn a working agent into a deployable one.**

## See also

- [05-hooks](05-hooks.md), [09-memory-files](09-memory-files.md), [10-multi-session-continuity](10-multi-session-continuity.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md) — primitives this chapter operationalises.
- [24-observability-tracing](24-observability-tracing.md) — health, metrics, dashboards.
- [27-horizon-long-horizon-degradation](27-horizon-long-horizon-degradation.md) — long-horizon failures need checkpointing.
- [53-chaos-engineering-next-era](53-chaos-engineering-next-era.md) — chaos testing of these patterns.
- [118-genai-maturity-models](118-genai-maturity-models.md) — when these become mandatory.
- [122-explainability-compliance](122-explainability-compliance.md), [123-robustness-fault-tolerance](123-robustness-fault-tolerance.md) — adjacent concerns.
- [125-system-level-production-patterns](125-system-level-production-patterns.md) — the system-level companion to this chapter.
