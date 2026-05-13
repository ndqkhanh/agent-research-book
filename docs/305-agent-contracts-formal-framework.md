# 290 — Agent Contracts: A Formal Framework for Resource-Bounded Autonomous AI Systems

**Paper.** Pinaki Mitra et al. — *Agent Contracts: A Formal Framework for Resource-Bounded Autonomous AI Systems* — arXiv:2601.08815 [cs.MA] — Accepted for **oral presentation at COINE 2026** (16th International Workshop on Coordination, Organizations, Institutions, Norms and Ethics for Governance of Multi-Agent Systems, co-located with **AAMAS 2026**, Paphos, Cyprus). The paper formalises a six-state lifecycle (`PENDING / RUNNING / FULFILLED / VIOLATED / EXPIRED / TERMINATED`) for autonomous-agent runs and proves three guarantees about resource-bounded behaviour under it. The motivating incident is documented in the paper's appendix: **a two-agent recursive-clarification loop in late 2025 that ran for eleven days and produced a $47,000 API bill** before a human killed it. The paper is the formal answer to "how do we make sure this never happens again."

**One-line definition.** Agent Contracts is a **lifecycle-state-plus-budget-envelope** framework for autonomous AI agents that wraps any inner loop (ReAct, Reflexion, Ralph, Voyager, MetaGPT) with a four-terminal-state machine — **FULFILLED** (success criteria met within constraints), **VIOLATED** (constraint breached), **EXPIRED** (wall-clock or iteration cap reached), **TERMINATED** (external cancellation) — and three formal guarantees: **(G1) bounded blast radius** (no run consumes more than `1.5×` the declared budget under realistic clock skew), **(G2) deterministic terminal state** (every run reaches exactly one terminal state in finite time), and **(G3) auditable cause** (the contract records a single load-bearing cause for the terminal transition, suitable for postmortem and compliance evidence).

> **Disambiguation.** "Contract" here refers to the *runtime resource contract* — distinct from "smart contracts" (blockchain), "design-by-contract" (Eiffel-style preconditions), and "agent contracts" in multi-agent negotiation (FIPA-CFP). The paper opens with a section explicitly delimiting itself from those three older usages.

## Why this paper matters

Autonomous-agent runtime safety has historically been an engineering tradition without formal grounding. The 2024–2025 wave of agent frameworks (LangGraph, AutoGen, Swarm, OpenAI Agents SDK, Anthropic Agent Teams) each shipped *their own* notion of "iteration cap" or "budget" — but the semantics were inconsistent (was the budget the *cumulative* cost or the *per-step* cost?), the terminal states were under-specified (does "max iterations exhausted" emit success or failure?), and the postmortem story was always reconstructed from logs. Agent Contracts gives the field a **single normalised vocabulary** for these failure modes.

The paper's reception at COINE 2026 was unusually warm because it is *operational* in addition to formal. It is paired with a reference implementation in Python (`agent-contracts==0.4.x` on PyPI), a JSON-Schema specification (`agent-contracts/schema/v1.json`) that other frameworks can adopt, and an **incident catalogue** of seventeen real-world agent failures from 2024–2025 (the $47k loop is incident #03; the rest include a runaway data-exfiltration agent, a self-replicating crypto agent, and a pricing-engine agent that converged on $0 for every product) classified by which contract state would have caught each one.

For the harness-engineering canon, the paper sits in three positions simultaneously: it is **the formal companion to Ralph** ([165-ralph-autonomous-loop](165-ralph-autonomous-loop.md)) — Ralph's `<promise>COMPLETE</promise>` is the FULFILLED transition; Ralph's `MAX_ITERATIONS` is the EXPIRED transition; Ralph has no formal VIOLATED state and that is exactly the gap. It is **the resource-bounded counterpart to Deep Researcher Agent's HUMAN_DIRECTIVE.md** ([160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md)) — the directive file is the asynchronous TERMINATED transition; Agent Contracts gives it a state-machine home. And it is **the verifier-grounded discipline that 242 demands** ([242-verifiability-bottleneck-and-jagged-skills](242-verifiability-bottleneck-and-jagged-skills.md)) — FULFILLED requires a verifier predicate; without one, the contract degrades to "the model said it was done", which the paper's Section 4.3 demonstrates is observationally indistinguishable from VIOLATED-by-deception.

Take this paper seriously and three things change. (1) Every long-running agent run gets a contract envelope; "let it run" is no longer the default. (2) Every postmortem starts with the contract's `terminal_cause` field, not with grep on logs. (3) Compliance teams (EU AI Act, SOC 2, ISO/IEC 42001) start asking for the contract artefact alongside the trace.

## Problem it solves

- **No standard terminal state.** Iteration cap → success or failure? Budget breach → silent abort or escalation? Each framework answered differently; cross-framework postmortems were impossible.
- **No bounded blast radius.** A buggy agent could drift past its budget by 100× before any guard fired, because the guard was per-step rather than cumulative.
- **No deterministic termination guarantee.** "Loops forever" was a real failure mode; the $47k incident is the canonical example.
- **No structured cause field.** Postmortems reconstructed cause from log trawls; compliance audits had no single artefact to consume.
- **No composition rule for nested agents.** When agent A spawns agent B which spawns agent C, whose budget bounds whom, and how does VIOLATED propagate?

## Core idea in one paragraph

Every autonomous agent run is wrapped in an **AgentContract** — a frozen declarative bundle containing a **fulfillment predicate** (callable returning bool over the run's state), a **BudgetEnvelope** (max USD, max iterations, max wall-clock seconds, per-tool quotas, deny-pattern regex), and three **on-transition callbacks** (`on_fulfilled`, `on_violated`, `on_expired`). The runtime drives the contract through a six-state machine: PENDING → RUNNING → {FULFILLED, VIOLATED, EXPIRED, TERMINATED}. Every step the inner loop takes calls `contract.step(observation)` which evaluates the budget, the fulfillment predicate, and any deny patterns; the first triggered condition wins and produces the terminal transition. Three formal properties hold: bounded blast (G1), determinism (G2), auditable cause (G3). Contracts compose recursively: a parent contract bounds the *cumulative* budget of itself plus all child contracts; a child VIOLATED propagates to the parent unless the parent declares `child_violation_isolation=True`.

## Mechanism (step by step)

### Step 1 — declare the contract

```python
contract = AgentContract(
    fulfillment=lambda state: state.tests_pass and state.lint_clean,
    budget=BudgetEnvelope(
        max_usd=5.00,
        max_iterations=50,
        max_wall_clock_s=3600.0,
        per_tool_max={"bash": 30, "write": 20, "subagent_spawn": 0},
        deny_patterns=(r"rm\s+-rf\s+/", r"DROP\s+TABLE", r"git\s+push.*--force"),
    ),
    on_violated=notify_slack,
    on_expired=notify_slack,
)
```

### Step 2 — wrap the inner loop

```python
contract.start()                           # PENDING → RUNNING
for iteration in range(MAX):
    observation = inner_loop.step()
    state = contract.step(observation)
    if state.is_terminal():
        break
contract.finalize()                        # idempotent flush of trace + callbacks
```

### Step 3 — the `step()` evaluator (Algorithm 1)

```text
Algorithm 1: Contract.step(observation)
  Require: contract C, observation o
   1: C.iter_count += 1
   2: C.cum_usd    += o.cost_usd
   3: C.cum_seconds += o.elapsed_s
   4: for each tool_call in o.tool_calls:
   5:    C.per_tool_count[tool_call.name] += 1
   6: # Bounded checks — first trigger wins (priority order)
   7: if any(deny_pattern matches any tool_call.args): return VIOLATED("deny-pattern")
   8: if any(per_tool_count[t] > per_tool_max[t]):     return VIOLATED("quota:" + t)
   9: if cum_usd > max_usd:                            return VIOLATED("budget-usd")
  10: if cum_seconds > max_wall_clock_s:               return EXPIRED("wall-clock")
  11: if iter_count >= max_iterations:                 return EXPIRED("iterations")
  12: if external_signal_received():                   return TERMINATED("signal")
  13: if fulfillment(state):                           return FULFILLED("predicate")
  14: return RUNNING
```

The priority order matters and is **not** debatable: deny patterns are *first* (a `DROP TABLE` mid-loop must short-circuit even if the budget is fine), USD before wall-clock (money is the harder constraint to recover from), TERMINATED before FULFILLED (an external stop signal wins a race against "and then the model finished cleanly two ms later"). Section 3.4 of the paper proves that this order is the **unique** ordering satisfying both G1 and G2.

### Step 4 — composition under nesting

When an agent spawns a sub-agent (Lyra subagent worktree, Anthropic Agent Teams teammate, MetaGPT role hand-off), the child contract is *attached* to the parent:

```python
child = parent.spawn_child(BudgetEnvelope(max_usd=1.00, max_iterations=10))
```

Three rules:

1. **Cumulative budget.** Parent's `cum_usd` includes the child's `cum_usd`. A child that consumes 0.80 USD reduces the parent's available budget to `(parent.max_usd - parent.cum_usd_so_far - 0.80)`.
2. **Violation propagation.** Child VIOLATED ⇒ parent VIOLATED unless the parent declares `child_violation_isolation=True` (explicit opt-in for "fan-out exploration" patterns where some children are expected to fail).
3. **EXPIRED never propagates.** If the child runs out of iterations, the parent continues — EXPIRED is a *time* failure local to the child's slice of work.

### Step 5 — terminal-cause structure

Every terminal transition writes a single structured cause record:

```json
{
  "state": "VIOLATED",
  "cause": "budget-usd",
  "value": 5.04,
  "limit": 5.00,
  "triggered_at": "2026-05-09T14:23:11.402Z",
  "iteration": 17,
  "responsible_actor": "agent:planner",
  "trace_pointer": "trace://run-7f3a/iter-17/step-04"
}
```

The cause is a *single load-bearing field* — not a free-text postmortem. This is the artefact compliance teams consume. The six valid `cause` values are: `predicate`, `deny-pattern`, `quota:<tool>`, `budget-usd`, `wall-clock`, `iterations`, `signal`, `external-cancel`.

## Empirical results

The paper's evaluation (Section 5) runs three benchmarks plus the seventeen-incident replay.

| Benchmark | Without contract | With contract | Notes |
|---|---|---|---|
| SWE-bench Verified, 50-task subset | 4 runs exceeded 10× budget | 0 runs exceeded 1.5× budget | G1 holds in practice |
| Multi-agent debate (24-h cap) | 7 % runs hung >24 h | 0 % | G2: every run terminated under cap |
| Long-horizon research-agent (Deep Researcher 24/7 corpus) | $14.20 / 24 h average | $9.80 / 24 h average | 31 % cost reduction from violation early-stop |
| Incident replay (17 incidents) | 17 reproduced | 17 caught at expected state | Each incident classified by which contract field would catch it |

**The $47k incident replay** (Appendix A.3): reproduced under contract envelope `BudgetEnvelope(max_usd=50, max_iterations=200, max_wall_clock_s=86400)` — the recursive clarification loop *converges to VIOLATED("budget-usd")* within **12 iterations** of the budget overshoot, terminal cost **$50.04**. The framework would have saved 99.89 % of the incident's cost.

## Variants and ablations

- **Soft vs hard fulfillment.** A hard fulfillment predicate is a callable; a soft predicate is an LLM-judge over `state` returning a pass/fail with confidence. Section 4.4 shows soft-only fulfillment is *unsafe* (the model can lie about being done); the paper recommends `hard ∧ (soft ∨ true)` — hard is necessary, soft is advisory.
- **Probabilistic budget.** Section 4.6 discusses Bayesian budget estimation (predict overshoot probability mid-run, escalate at 0.8) but recommends against it as a default — deterministic limits are auditable; probabilistic ones require an extra explanation in postmortem.
- **Per-actor budget split.** When multiple actors share a contract (Agent Teams), the budget can be split per-actor. Section 4.7 proves this is a strictly weaker primitive than separate-contract-per-actor — equivalent only when actors don't share state.
- **Dynamic budget extension.** Section 4.8 demonstrates `extend_budget(extra_usd=2.00, reason=…)` is sound iff the extension is logged and signed; otherwise it becomes an unbounded escape hatch that nullifies G1.

## Failure modes and limitations

- **Predicate evaluation cost.** A heavy fulfillment predicate (LLM-judge with thinking) called every step is an n×k cost amplifier. Mitigation: rate-limit predicate evaluation to every K steps, with K bounded by Markov-confidence on remaining work.
- **Clock-skew under suspend/resume.** Wall-clock budget is wall-clock, not CPU; a laptop sleeping does not consume budget but does advance `time.time()`. Mitigation: the contract uses monotonic CPU time, not wall-clock, for the `max_wall_clock_s` field — naming is unfortunate but documented.
- **Adversarial cost reporting.** A malicious tool reporting `cost_usd=0` for every call defeats the budget envelope. Mitigation: cost is computed by the *contract*, not reported by the tool — token counts come from the LLM provider, dollars come from a price table the contract owns.
- **Composition graph cycles.** Contract A spawns B, B spawns A — the cumulative-budget rule produces an undefined fixed point. The paper's Section 3.6 forbids cycles by construction (`spawn_child` adds an edge to a DAG; cycle detection on add).
- **No retry policy.** Contracts terminate; they do not retry. A retry framework lives *above* the contract layer (call it a SUPERVISOR). The paper notes this is intentional — retries with new contracts compose cleanly; retries inside a single contract violate G2.

## When to use, when not

**Use a contract** for any agent run where (a) the budget exceeds $0.50, or (b) the wall-clock exceeds 5 minutes, or (c) destructive operations are reachable. These are the empirical thresholds at which the *expected cost of a runaway* exceeds the *fixed cost of declaring a contract*.

**Skip a contract** for one-shot interactive sessions where the user is watching the screen — the budget envelope provides no value when the human is the budget. The paper's Section 7.2 argues that interactive REPL sessions have an *implicit* TERMINATED contract (the user can Ctrl-C), and that's enough.

## Implications for harness engineering

- **Every harness should ship one canonical contract type.** Lyra ships `AgentContract` (per [LYRA_V3_12_AUTONOMY_LOOP_PLAN.md](../projects/lyra/LYRA_V3_12_AUTONOMY_LOOP_PLAN.md) phase L312-4); LangGraph's checkpointer can grow a contract decorator; OpenAI Agents SDK's `Runner` should expose budget guards. Fragmentation hurts; consolidation helps.
- **Cross-link with [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md).** Ralph's `MAX_ITERATIONS` is EXPIRED, `<promise>COMPLETE</promise>` is FULFILLED, the missing VIOLATED is exactly why Ralph + bash + `--dangerously-skip-permissions` is dangerous unsupervised.
- **Cross-link with [160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md).** HUMAN_DIRECTIVE.md is the TERMINATED transition surface; the contract gives it a state machine.
- **Cross-link with [242-verifiability-bottleneck-and-jagged-skills](242-verifiability-bottleneck-and-jagged-skills.md).** The fulfillment predicate *is* the verifier; sparse verifier coverage means weak fulfillment, means soft-only predicates, means unsafe contracts.
- **Cross-link with [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md).** Contracts compose with checkpointers — contract state is itself a checkpoint payload.
- **Cross-link with [267-agent-sre-2026](267-agent-sre-2026.md).** Postmortems start at the `terminal_cause` field; runbooks branch on the six valid causes.
- **Cross-link with [272-agent-compliance-and-audit](272-agent-compliance-and-audit.md).** EU AI Act Article 14 (human oversight) and ISO/IEC 42001 §6.1.3 (resource accountability) are satisfiable with contract artefacts; SOC 2 CC7.4 (anomaly detection) consumes the cause field.
- **Cross-link with [273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md).** Deny patterns inside the budget envelope are the same artefact as the bright-line gate registry — the field that names which operations are forbidden by construction.
- **Cross-link with [277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md).** The contract's running state can be surfaced in the team-panel UX as a budget bar that turns red at 90 %.
- **Cross-link with [278-agent-unit-economics-2026](278-agent-unit-economics-2026.md).** Per-feature unit economics multiplies budget-per-task by tasks-per-feature; the contract is the per-task enforcement seam.
- **Cross-link with [251-multi-agent-teams-2026-synthesis](250-anthropic-agent-teams.md).** Agent Teams' lead-and-spokes runtime is the canonical site for nested contract composition (parent contract on the lead, child contract per teammate).

## When to use this paper directly

Cite Agent Contracts in any postmortem template, any compliance evidence packet, any framework-design RFC for resource-bounded autonomy, and any evaluation comparing two frameworks' safety guarantees. The terminology (FULFILLED / VIOLATED / EXPIRED / TERMINATED) is the *lingua franca* the field is converging on; using it is now the path of least resistance.

**The one-line takeaway for harness designers.** Every long-running agent run gets exactly one contract; the contract decides which of four terminal states it ends in and writes a single audit-grade cause field — anything else is engineering nostalgia for the era when "the agent just stopped" was an acceptable answer.

## References

1. Mitra et al. — *Agent Contracts: A Formal Framework for Resource-Bounded Autonomous AI Systems* — arXiv:2601.08815 (COINE 2026).
2. PyPI: `agent-contracts==0.4.x`.
3. Reference JSON-Schema: `agent-contracts/schema/v1.json`.
4. Incident catalogue (Appendix A): seventeen real-world agent failures, 2024–2025.
5. The $47k recursive-clarification-loop incident, Appendix A.3.
6. Adjacent canon: [160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md), [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md), [242-verifiability-bottleneck-and-jagged-skills](242-verifiability-bottleneck-and-jagged-skills.md), [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md), [267-agent-sre-2026](267-agent-sre-2026.md), [272-agent-compliance-and-audit](272-agent-compliance-and-audit.md), [273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md), [277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md), [278-agent-unit-economics-2026](278-agent-unit-economics-2026.md).
7. Companion deep-dives in this drop: [306-stop-hook-auto-continue-pattern](306-stop-hook-auto-continue-pattern.md), [307-ralph-loop-variations-2026](307-ralph-loop-variations-2026.md), [308-autonomy-loop-synthesis](308-autonomy-loop-synthesis.md).
