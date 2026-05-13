# 123 — Robustness & Fault-Tolerance Patterns: Adaptive Retry, Circuit Breakers, Parallel Execution Consensus, Graceful Degradation

**Sources.** Arsanjani & Bustos, *Agentic Architectural Patterns*, Chapter 7 (Robustness and Fault Tolerance Patterns) — naming Adaptive Retry, Circuit Breaker, Parallel Execution Consensus; classic distributed-systems literature (Nygard's *Release It!*, Netflix Hystrix, AWS resilience patterns); plus agent-specific robustness work (LinuxArena production safety, [26-linuxarena-production-agent-safety](26-linuxarena-production-agent-safety.md)).

**One-line definition.** Robustness and fault-tolerance for agents adapts five well-tested distributed-systems patterns — *adaptive retry with backoff*, *circuit breaker*, *bulkhead isolation*, *parallel execution consensus*, *graceful degradation* — to LLM-specific failure modes (model outage, rate limit, format malformation, hallucination, infinite loop, cost runaway), with the key difference that the agent itself can introduce fault patterns (looping, mis-tool-use) that classic resilience playbooks don't anticipate, requiring agent-specific extensions.

## Why this chapter matters

The classical distributed-systems robustness playbook — retry, circuit-break, bulkhead, fallback — was developed for services that failed in mechanical ways: a server died, a network partitioned, a queue backed up. LLM-based agents fail in those ways *and* in new ways: the model rate-limits at the provider, the output mis-formats, the agent hallucinates a tool, the agent loops infinitely on the same retry, the cost runs away because the agent kept generating tokens.

A naive transplant of distributed-systems robustness to agents misses half the problem. Naive *retry* on a hallucinated output produces a hallucinated output again. Naive *circuit-break* on an LLM provider doesn't help if the only fallback is the same provider's smaller model. Naive *bulkhead* doesn't isolate the agent's cost-runaway from the rest of the system.

The agentic robustness playbook combines classical patterns with agent-specific extensions: cost circuit breakers, format-validation retry strategies, semantic verifier loops, infinite-loop detection. This chapter is that combined catalog.

## Problem it solves

Six concrete agent failure modes the patterns address:

1. **Provider outage.** OpenAI is down; agent hangs. Need fallback path.
2. **Rate limit.** Agent hits provider quota; subsequent calls fail. Need retry with backoff *and* circuit-break to fallback.
3. **Format malformation.** Model produces invalid JSON. Need validation + retry with constraint enforcement.
4. **Hallucinated tool.** Model calls a tool that doesn't exist. Need refusal + recovery.
5. **Infinite loop.** Agent keeps trying the same broken approach. Need loop detection + escape.
6. **Cost runaway.** Agent generates 50K tokens before terminating. Need cost circuit breaker.

Each maps to one or more patterns below.

## Core idea in one paragraph

Five robustness patterns layer to produce a fault-tolerant agent. **Adaptive retry with exponential backoff** handles transient failures (network, rate limits, occasional format errors) — but caps retries at low N (3) and never retries on errors that won't self-resolve. **Circuit breaker** detects sustained failure and routes to a fallback (different model, simpler logic, refusal) — preventing pile-up. **Bulkhead isolation** caps the resources any single agent run can consume — bounding blast radius. **Parallel execution consensus** runs the same task multiple times and takes the agreed answer — handling hallucination at the cost of compute. **Graceful degradation** falls back to a simpler-but-working capability when the full agent fails — preserving partial usefulness. Add four agent-specific extensions: **format-validation retry** (retry only with stricter constraints), **semantic verifier loops** (catch wrong-but-well-formatted outputs), **loop detection** (no agent should make the same call twice), **cost circuit breaker** (kill the agent before the bill explodes). Apply all nine and you have an agent that fails as intended: visibly, recoverably, and within budget.

## Mechanism (step by step)

### 1. Adaptive retry with exponential backoff

```text
attempts = 0
while attempts < MAX:
    try:
        result = call(...)
        return result
    except RetryableError as e:
        attempts += 1
        sleep(BASE * (2 ** attempts) + jitter())
    except NonRetryable:
        raise
```

Calibration:
- **MAX**: typically 3. Not 10. Many retries hide bugs.
- **Retryable errors**: rate limit (429), transient timeout, transient 5xx.
- **Non-retryable**: 401 (auth), 400 (bad request), persistent 5xx (something is wrong).
- **Jitter**: avoid thundering herd; add random ±20% to backoff.
- **Per-call budget**: total time across retries should not exceed the user's latency tolerance.

For LLM calls specifically: retry on rate-limit, retry on transient timeout, *do not* retry on parse-fail without changing the prompt or constraint (otherwise you'll get the same parse-fail).

### 2. Circuit breaker

```text
states: CLOSED → OPEN → HALF-OPEN → CLOSED

CLOSED:    calls flow normally; failures counted.
            on threshold (e.g. 50% failure in 1min): → OPEN.
OPEN:      all calls short-circuit to fallback; no real calls made.
            after timeout (e.g. 60s): → HALF-OPEN.
HALF-OPEN: a few test calls let through.
            if successful: → CLOSED.
            if failures: → OPEN.
```

For agents: a circuit on each LLM provider, each tool, each MCP server. When the primary provider trips, fall over to fallback ([119-agent-ready-llm-selection](119-agent-ready-llm-selection.md)).

### 3. Bulkhead isolation

Resources are partitioned so failure in one partition cannot exhaust others:

- **Per-agent thread pool.** Agent A's parallelism cannot starve agent B.
- **Per-agent token budget.** Agent A cannot consume the global LLM quota.
- **Per-agent timeout pool.** Long-running calls don't pile up.
- **Per-tool concurrency limit.** Tool X failing slowly doesn't hang the agent.

Implemented at the harness layer; the agent code stays oblivious.

### 4. Parallel execution consensus

```text
results = [agent.run(task) for _ in range(K)]   // run K times
consensus = vote(results)                         // majority / weighted / verifier-judged
return consensus
```

For tasks where the answer is verifiable and individual runs are cheap relative to value:
- **K = 3** is the most common; K = 5 for high-stakes.
- **Vote function**: majority for classification; verifier for open-ended.
- **Cost**: K× single-run; reserve for high-stakes tasks.

This is the [self-consistency](https://arxiv.org/abs/2203.11171) pattern at the agent level.

### 5. Graceful degradation

When the full agent fails, fall back to:
- A smaller model with simpler prompt.
- A pure RAG response (retrieve and concatenate; skip reasoning).
- A canned "I cannot complete this task right now" message with HITL escalation.
- A pre-computed answer for the most common variant of the task.

The principle: **better to return a partial answer than no answer**.

### 6. Format-validation retry — agent-specific

When format validation fails:

```text
result = call(prompt)
if not validates(result):
    result = call(prompt + " Output MUST match schema X.")
    if not validates(result):
        return error_with_partial(result)
```

Note: better than naive retry is to *change the constraint* on the second attempt — tighter prompt, constrained decoding ([112-constrained-decoding](112-constrained-decoding.md)), or refusal. Naive retry on the same prompt usually produces the same failure.

### 7. Semantic verifier loops — agent-specific

For correctness, not just format:

```text
result = call(prompt)
verification = verifier(prompt, result)
if not verification.acceptable:
    result = call(prompt + verification.feedback)
```

A separate, cheaper LLM (or rule engine) verifies the result. On verification failure, the verifier's feedback is incorporated into a retry. This is the [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) pattern productionised.

Cap retries (typically 2–3); if verifier never accepts, fall back to graceful degradation.

### 8. Loop detection — agent-specific

```text
trajectory = []
while not done:
    action = next_action()
    if action in trajectory[-N:]:
        // same action recently → likely looping
        force_diversify() or escalate_to_human()
    trajectory.append(action)
    execute(action)
```

The simplest implementation: track the last 5 actions; if the agent repeats one, it's looping. Force the agent to take a different action or escalate.

### 9. Cost circuit breaker — agent-specific

```text
running_cost = 0
while not done:
    cost_estimate = estimate(next_action())
    if running_cost + cost_estimate > BUDGET:
        graceful_terminate()
        return partial_result
    result = execute(next_action())
    running_cost += actual_cost(result)
```

Per-task budget enforced before the agent commits to expensive operations. Kills cost-runaway regardless of intent.

### 10. Composition — the complete pattern

```text
[user request]
   ↓
[bulkhead: per-agent thread pool, token budget]
   ↓
[cost circuit breaker: per-task budget]
   ↓
[adaptive retry on transient errors]
   ↓
[circuit breaker per provider/tool]
   ↓
[agent step]
   ↓
[format validation → retry with constraint]
   ↓
[semantic verifier → retry with feedback]
   ↓
[loop detection → escape if needed]
   ↓
[parallel consensus for high-stakes]
   ↓
[graceful degradation if all else fails]
   ↓
[result OR partial-result-with-explanation]
```

Each layer adds resilience. None individually is sufficient.

## Empirical anchors

- **Retry hides bugs.** Without metrics on retry rate, transient errors mask real bugs that should be fixed.
- **Circuit breakers** prevent cascading failures; widely deployed in mature distributed systems.
- **Bulkheads** are the difference between "one bad agent runs hot" and "one bad agent takes the system down."
- **Parallel consensus** improves correctness 5–15% on verifiable tasks, at K× cost.
- **Format-validation retry** with constrained decoding has effectively 100% success after 1–2 attempts.
- **Cost runaway** is the most common production agent incident; cost circuit breakers prevent it.
- **Loop detection** catches a non-trivial fraction of stuck agents; without it, they run to budget exhaustion.

## Variants and counter-arguments addressed

- **"Just retry more aggressively."** Hides bugs; pile-ups; doesn't fix non-self-resolving failures.
- **"Circuit breakers are over-engineering."** Until your primary provider has an outage; then they're the only thing keeping you up.
- **"Parallel consensus is too expensive."** For non-high-stakes calls, yes; for high-stakes, the cost is justified.
- **"Graceful degradation is admitting defeat."** It's admitting *partial* defeat in service of partial usefulness. The alternative is total failure.
- **"Constrained decoding makes format-retry unnecessary."** Mostly true. Pair both for belt-and-braces.
- **"Verifier loops are slow."** The verifier is usually a smaller, cheaper model; the loop terminates in 1–2 iterations on most workloads.

## Failure modes and limitations

1. **Retry storm.** Many clients retrying simultaneously after an outage; provider can't recover. Solved by jitter + circuit breakers.
2. **Circuit-breaker thresholds wrong.** Too sensitive: opens on noise; too lax: too late. Tune on production data.
3. **Bulkhead starvation.** Wrong partition sizes; one bulkhead constantly hits its limit. Monitor utilisation.
4. **Verifier mis-calibration.** Verifier rejects correct answers or accepts wrong ones; periodic re-calibration on labelled examples.
5. **Loop-detection false positives.** Legitimate cases where the agent repeats an action; tune sensitivity.
6. **Cost-circuit-breaker mid-task termination.** Partial result might be useless or misleading; design graceful-termination semantics carefully.
7. **Fallback complexity.** Each layer's fallback is its own subsystem; cumulative complexity is real.
8. **Test coverage of failure paths.** Most teams test the happy path; failure paths fail in production because they're untested. Chaos engineering ([53-chaos-engineering-next-era](53-chaos-engineering-next-era.md)) is the discipline.

## When to use, when not

**All five distributed patterns + four agent-specific** are mandatory for stage-3 production agents ([118-genai-maturity-models](118-genai-maturity-models.md)).

**Subset (retry + circuit breaker + cost circuit breaker)** is the minimum viable for any agent that touches an LLM provider's API.

**Skip parallel consensus** when cost matters more than the marginal correctness gain.

**Skip elaborate verifier loops** when the task is low-stakes and re-do is cheap.

## Implications for harness engineering

- **Hooks own resilience.** [05-hooks](05-hooks.md) is where pre/post-action retries, circuit breakers, cost gates live. Not the agent's responsibility.
- **Per-task budget is a harness primitive.** Cost, time, step budgets — declared at task start, enforced through the loop.
- **Provider abstraction owns retry + fallback.** A `model.complete(...)` call goes through retry-and-fallback machinery transparently. See [119-agent-ready-llm-selection](119-agent-ready-llm-selection.md).
- **Loop detection is harness-level.** Trajectory inspection at every step; escape on repeat.
- **Verifier as a first-class component.** [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md). Production agents should ship with one.
- **Chaos-test the fallbacks.** Periodically inject failures and verify the system gracefully degrades. See [53-chaos-engineering-next-era](53-chaos-engineering-next-era.md).
- **Dashboard the resilience metrics.** Retry rate, circuit-breaker state, fallback rate, cost-circuit trips. Drift in any signals a problem.
- **Document the failure budget.** Per-task: max retries, max steps, max cost, max time. Visible to the user. See [143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md).

The one-sentence takeaway: **agentic robustness layers five distributed-systems patterns (retry, circuit-break, bulkhead, parallel consensus, graceful degradation) with four agent-specific patterns (format-retry, verifier-loops, loop-detection, cost-circuit) — apply all nine and the agent fails recoverably within budget.**

## See also

- [05-hooks](05-hooks.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [24-observability-tracing](24-observability-tracing.md) — harness primitives that implement these patterns.
- [26-linuxarena-production-agent-safety](26-linuxarena-production-agent-safety.md) — production agent safety in real Linux environments.
- [27-horizon-long-horizon-degradation](27-horizon-long-horizon-degradation.md) — long-horizon failure modes; resilience patterns are part of the answer.
- [53-chaos-engineering-next-era](53-chaos-engineering-next-era.md) — chaos engineering for agents.
- [119-agent-ready-llm-selection](119-agent-ready-llm-selection.md) — the multi-vendor fallback substrate.
- [122-explainability-compliance](122-explainability-compliance.md) — failures must be observable and auditable.
- [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md) — broader production patterns.
