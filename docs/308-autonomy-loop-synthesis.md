# 293 — Autonomy-Loop Synthesis: The 2026 Stack for Long-Running Coding Agents

**Anchors.** This synthesis ties together the seven primitives the 2026 field has converged on for *making coding agents finish a task without asking and stay safe overnight*. Source materials: [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md), [160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md), [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md), [144-build-your-own-harness](144-build-your-own-harness.md), [62-everything-claude-code](62-everything-claude-code.md), [05-hooks](05-hooks.md), [01-agent-loop](01-agent-loop-architecture.md), [09-memory-files](09-memory-files.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md), [238-karpathy-agentic-engineering-shift](238-karpathy-agentic-engineering-shift.md), [242-verifiability-bottleneck-and-jagged-skills](242-verifiability-bottleneck-and-jagged-skills.md), [277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md), [278-agent-unit-economics-2026](278-agent-unit-economics-2026.md), [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md), [267-agent-sre-2026](267-agent-sre-2026.md), [273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md), [305-agent-contracts-formal-framework](305-agent-contracts-formal-framework.md), [306-stop-hook-auto-continue-pattern](306-stop-hook-auto-continue-pattern.md), [307-ralph-loop-variations-2026](307-ralph-loop-variations-2026.md). Operational anchor: [LYRA_V3_12_AUTONOMY_LOOP_PLAN.md](../projects/lyra/LYRA_V3_12_AUTONOMY_LOOP_PLAN.md).

**One-line definition.** A 2026 picture of the **autonomy-loop stack** — the seven composable primitives that, taken together, turn the field-level complaint *"every coding agent finishes a task and stops"* into a programmable, safe, long-running substrate: **(1) inner agent loop** (think → act → observe with iteration budget), **(2) Stop / SubagentStop hook** (the seam where "I'm done" becomes a programmable decision), **(3) AgentContract envelope** (FULFILLED / VIOLATED / EXPIRED / TERMINATED), **(4) fresh-context outer loop** (Ralph, with file-based memory in git + progress + PRD), **(5) HUMAN_DIRECTIVE.md asynchronous control file** (the brake), **(6) hibernate-and-wake scheduler** (sleep-until-next-fire cron + after-event triggers), **(7) long-running supervisor** (autopilot daemon with checkpointed state) — each independently shippable, all composable into a single run that completes 50 stories overnight on a $5 budget without asking and without burning $47k on a clarification loop.

## Why this synthesis matters

Through 2024–2025 the field shipped point solutions: a Stop hook here, a cron daemon there, a Ralph script elsewhere. By May 2026 it is clear these are *not independent*; they are the seven layers of a single stack. The user-visible problem ("the agent stops and asks") was the *symptom* of a missing stack; the seven primitives are the cure.

The synthesis matters because:

- **Layer omission is the dominant failure mode.** Ship Ralph without a contract envelope → $47k incident class ([305](305-agent-contracts-formal-framework.md), Appendix A.3). Ship a Stop hook without an iteration cap → infinite loop ([306](306-stop-hook-auto-continue-pattern.md), claude-mem #1288). Ship a long-running supervisor without HUMAN_DIRECTIVE.md → "how do I stop this without Ctrl-C?" → operator types Ctrl-C → state lost. Each missing layer is a documented incident.
- **Layer ordering is non-arbitrary.** The contract evaluates before the hook; the hook fires before the outer loop's PRD increment; the directive file is checked at iteration boundary; the supervisor checkpoints after the contract finalises. Reordering breaks safety properties.
- **The economics flip at the stack level, not the layer level.** A Ralph run with no contract is $5 ± $47,000. A Ralph run inside a contract inside a supervisor with a directive file is $5 ± $5. The variance is what determines whether a feature is *shippable*; the stack is what bounds the variance.

Take this synthesis seriously and three things change. (1) Architectural reviews of any "agent that runs while I'm not watching" feature start with "which of the seven layers are missing?" (2) Plugin marketplaces grow a *layer-coverage* dimension alongside the trust-tier dimension. (3) The team that ships layer 1 also ships layer 2 in the same PR — they are not independent surfaces.

## The seven primitives

```text
        ┌────────────────────────────────────────────────────────────┐
        │  LAYER 7 — Long-running supervisor (autopilot)             │
        │           checkpointed state, /hibernate, crash recovery   │
        ├────────────────────────────────────────────────────────────┤
        │  LAYER 6 — Hibernate-and-Wake scheduler                    │
        │           sleep-until-next-fire cron, after-event triggers │
        ├────────────────────────────────────────────────────────────┤
        │  LAYER 5 — HUMAN_DIRECTIVE.md async control file           │
        │           the brake, the redirect, the rollback request    │
        ├────────────────────────────────────────────────────────────┤
        │  LAYER 4 — Fresh-context outer loop (Ralph variants)       │
        │           git + progress.txt + prd.json + COMPLETE token   │
        ├────────────────────────────────────────────────────────────┤
        │  LAYER 3 — AgentContract envelope                          │
        │           FULFILLED / VIOLATED / EXPIRED / TERMINATED      │
        ├────────────────────────────────────────────────────────────┤
        │  LAYER 2 — Stop / SubagentStop hook                        │
        │           stop_hook_active flag, deny → re-feed seam       │
        ├────────────────────────────────────────────────────────────┤
        │  LAYER 1 — Inner agent loop                                │
        │           think → act → observe, iteration budget          │
        └────────────────────────────────────────────────────────────┘
```

Read it bottom-up: every primitive depends on the one below. A Stop hook (layer 2) requires an inner loop (layer 1) to fire from. A contract (layer 3) is meaningless without a hook (layer 2) seam to enforce at. A Ralph outer loop (layer 4) without a contract (layer 3) is the unsafe variant the field shipped in 2024–2025. The directive file (layer 5) is meaningless without a fresh-context outer loop (layer 4) to consume it at iteration boundaries. The hibernate-and-wake scheduler (layer 6) is the right scheduler for the directive-file (layer 5) cycle. The supervisor (layer 7) is what makes the whole stack survive a reboot.

### Layer 1 — inner agent loop

Anchored at [01-agent-loop](01-agent-loop-architecture.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md). The think → act → observe loop with an iteration budget. Every harness has one; the disagreements are minor (sync vs async, one tool dispatch vs many per turn, plugin shape).

The 2026 invariant: the loop terminates when the model returns a message *with no tool calls*. That terminal moment is the seam where layer 2 attaches. Without it, "the agent stops and asks" is the only behaviour.

### Layer 2 — Stop / SubagentStop hook

Anchored at [05-hooks](05-hooks.md), [62-everything-claude-code](62-everything-claude-code.md), [306-stop-hook-auto-continue-pattern](306-stop-hook-auto-continue-pattern.md). The hook fires *between* the model's "I'm done" signal and the runtime's "the turn is over" finalisation. The hook's `Decision.deny` injects a synthetic user turn and resumes the inner loop. The `stop_hook_active` flag prevents infinite re-entry; the four safeguards (flag + cap + watermark + predicate) prevent runaway.

The 2026 invariant: the hook is *advisory*, the contract (layer 3) is *authoritative*. The hook can ask to keep going; the contract can refuse.

### Layer 3 — AgentContract envelope

Anchored at [305-agent-contracts-formal-framework](305-agent-contracts-formal-framework.md), [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md), [267-agent-sre-2026](267-agent-sre-2026.md). The four-state machine — FULFILLED / VIOLATED / EXPIRED / TERMINATED — wrapping any inner loop. BudgetEnvelope (USD, iterations, wall-clock, per-tool quotas, deny patterns). Three formal guarantees: bounded blast (G1), deterministic terminal state (G2), auditable cause (G3).

The 2026 invariant: every long-running run gets a contract; "let it run" is no longer a default; the contract's `terminal_cause` field is the postmortem starting point.

### Layer 4 — fresh-context outer loop (Ralph variants)

Anchored at [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md), [307-ralph-loop-variations-2026](307-ralph-loop-variations-2026.md), [09-memory-files](09-memory-files.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md), [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md). The bash-loop-around-coding-tool that solves context-degradation by spawning a fresh agent each iteration. Memory lives in `git history + progress.txt + prd.json + CLAUDE.md/AGENTS.md`. Three variants in production (snarktank / frankbria / vercel-labs); the load-bearing parameter is the completion-signal hierarchy (Tier 1 string match → Tier 4 typed callback).

The 2026 invariant: fresh-context is the safer default for runs >30 iterations; shared-session is fine for shorter runs ([277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md) `/loop` self-paced variant).

### Layer 5 — HUMAN_DIRECTIVE.md async control file

Anchored at [160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md). A single Markdown file that the running loop watches per iteration. On `mtime` advance, the file's contents are prepended to the next iteration's user prompt under a `<directive priority="high">…</directive>` envelope. The user can append "STOP", "FOCUS ON US-007", "ROLLBACK LAST COMMIT" without Ctrl-C-ing the loop.

The 2026 invariant: the directive is the *human override surface* for an unattended run. Without it, the only stop signal is Ctrl-C, which is hostile to checkpointed state.

### Layer 6 — hibernate-and-wake scheduler

Anchored at the Autobot agent-loop-and-cron pattern, Meta REA's hibernate-and-wake, "Let Them Sleep" sleep-cycle pattern. The scheduler computes the next-fire time and *sleeps* until then, rather than polling every second. New trigger types: `after sess-X +5m`, `on git-push origin/main`, `on signal SIGUSR1`, `after lyra-ralph US-007`.

The 2026 invariant: poll-mode is fine for second-resolution schedules; sleep-mode is required for hourly/daily/weekly horizons and laptop-suspend-friendly long runs.

### Layer 7 — long-running supervisor (autopilot)

Anchored at [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md), [267-agent-sre-2026](267-agent-sre-2026.md), Lyra L312-7. A user-space daemon that drives Ralph + cron + `/loop` schedules concurrently, checkpoints to SQLite, recovers across reboots. Explicit-only resume (no silent re-fire of crashed loops). `/hibernate` slash to checkpoint a REPL session and exit cleanly.

The 2026 invariant: explicit resume; no auto-replay of crashed loops; the human inspects state before unfreezing.

## Composition rules

Six rules describe how the layers compose. Each rule is a property the field has discovered the hard way.

### Rule 1 — Contract preempts hook preempts loop

```
Iteration step:
    1. inner_loop.step()                         # layer 1
    2. contract.step(observation)                # layer 3
       → if terminal: emit terminal state, exit
    3. stop_hook.fire(ctx)                       # layer 2
       → if deny: re-feed and continue
    4. directive_file.consume_if_changed()       # layer 5
    5. outer_loop.advance()                      # layer 4
```

The contract is checked first because it is the authoritative bound. The hook is checked second because it is advisory. The directive is checked at iteration boundary because mid-iteration consumption breaks atomicity.

### Rule 2 — Fresh-context bounds context-degradation, contract bounds blast radius, directive bounds operator anxiety

Each outer-layer primitive solves a distinct anxiety class:

| Layer | Anxiety solved |
|---|---|
| 4 (Ralph fresh-context) | "the agent's context will degrade after N hours" |
| 3 (AgentContract) | "the agent will burn $50,000 if I don't watch" |
| 5 (directive) | "I want to redirect without Ctrl-C" |
| 7 (autopilot) | "I want to leave it running and not lose state to a reboot" |

If any of these anxieties is unresolved, the operator falls back to manual continue-prompts, which is the very behaviour we are trying to eliminate.

### Rule 3 — Verifier density bounds the entire stack

[242-verifiability-bottleneck-and-jagged-skills](242-verifiability-bottleneck-and-jagged-skills.md) says it explicitly: *autonomy is bounded by verifier density*. The seven layers cannot exceed the verifier they share:

- Layer 2's `auto_continue_until` predicate *is* a verifier.
- Layer 3's `fulfillment` predicate *is* a verifier.
- Layer 4's completion-signal Tier 3 *is* a verifier.

If the verifier is weak, every layer that depends on it degrades — auto-continue fires forever, the contract never reaches FULFILLED, Ralph never sees its COMPLETE signal validated. Ship the verifier first; the stack second.

### Rule 4 — Explicit resume, never auto-replay

A crashed loop is in an undefined state. The contract may or may not have logged its last step; the directive file may have been mid-edit; the PRD may be partially advanced. The supervisor must require *explicit* `lyra autopilot resume <id>` (or equivalent). Auto-replay corrupts state.

### Rule 5 — Layer omission is documented incidents

The field has filed a public incident for every common omission:

| Missing layer | Incident class | Reference |
|---|---|---|
| 3 (contract) | $47k recursive-clarification-loop | [305](305-agent-contracts-formal-framework.md), Appendix A.3 |
| 2 (`stop_hook_active` flag) | claude-mem infinite Stop loop | [306](306-stop-hook-auto-continue-pattern.md), claude-mem #1288 |
| 5 (directive) | Ctrl-C-loses-state user reports | Multiple, e.g. ralph-claude-code issues |
| 6 (sleep-mode scheduler) | Laptop-suspend-kills-cron | Common in cron mode |
| 7 (explicit resume) | Auto-replayed-corrupt-state | Documented in autopilot designs |

Each incident is the unbiased estimator for "we needed that layer."

### Rule 6 — `--dangerously-skip-permissions` is a layer 0 escape hatch and should be refused

snarktank and frankbria document the flag because they assume a trusted environment. In any other context — multi-tenant, regulated, audited — the flag bypasses the entire stack. Lyra's L312-2 explicitly refuses it for `lyra ralph`; the contract envelope's deny patterns + path quarantine + cost guard substitute. The flag is a *layer-zero* escape hatch; the stack assumes it is unset.

## Decision matrix — which layers do I need?

| Task profile | Layers needed | Notes |
|---|---|---|
| One-shot interactive ("write me a function") | 1 only | The user is the budget |
| Medium-horizon (3–30 min, fire-and-forget) | 1 + 2 + 3 | Auto-continue + contract is enough |
| Long-horizon coding (decomposable into stories) | 1 + 2 + 3 + 4 | Ralph + contract |
| Long-horizon research (24/7) | 1 + 2 + 3 + 4 + 5 | + directive for human override |
| Multi-day, multi-host, multi-loop | 1 + 2 + 3 + 4 + 5 + 6 + 7 | Full stack |

The field's mistake in 2024 was applying layer 1+2 patterns to layer 1+2+3+4+5+6+7 problems. The result was the runaway-loop incident class.

## Empirical results — full-stack vs partial-stack

Aggregated from the Lyra L312 eval corpus + the Agent Contracts paper Section 5 + the Deep Researcher Agent 24/7 production deployment:

| Configuration | Median time-to-completion (5-story PRD) | p95 cost | Runaway rate |
|---|---:|---:|---:|
| Manual continue (layers 1) | 45 min | $1.40 | 0 % |
| Auto-continue only (layers 1–2) | 12 min | $1.20 | 8 % (when predicate buggy) |
| + Contract (layers 1–3) | 12 min | $1.20, hard ≤ $5 | 0 % |
| + Ralph (layers 1–4) | 28 min | $1.30 | 0 % |
| + Directive (layers 1–5) | 28 min | $1.30 | 0 %, +human-override |
| + Sleep scheduler (1–6) | 28 min on laptop, 12 min on desktop | same | 0 % |
| + Autopilot (full stack) | 28 min, survives reboot | same | 0 %, survives crash |

The cost is *flat* across full-stack vs partial-stack at the median — the additional layers add zero cost on the happy path. They add value on the long tail where partial-stack is unsafe.

## Implementation guide — what to ship first

If you are building a new harness, the empirical-best ordering:

1. **Ship layers 1+3 in the same release.** Inner loop with iteration budget, plus a contract envelope that wraps every long-running entry-point. This bounds the worst case before you have any auto-continue features.
2. **Ship layer 2 next.** The Stop hook seam, with the four safeguards. This is the user-visible "stop asking me to continue" win. Lyra's L312-1 + L312-8 covers this.
3. **Ship layer 4 next.** Fresh-context outer loop (Ralph). This unlocks unattended overnight work. Lyra's L312-2.
4. **Ship layers 5+6 together.** Directive file + sleep-mode scheduler. This unlocks 24/7 deployments. Lyra's L312-5 + L312-6.
5. **Ship layer 7 last.** Long-running supervisor. This unlocks "leave it running for a week". Lyra's L312-7.

This is exactly the sequencing in [LYRA_V3_12_AUTONOMY_LOOP_PLAN.md](../projects/lyra/LYRA_V3_12_AUTONOMY_LOOP_PLAN.md) §6.

## Implications for harness engineering

- **Stop framing autonomy as a model property.** It is not. It is a stack property — the model is one of seven layers. A weaker model with a stronger stack outperforms a stronger model with a weaker stack on long-horizon tasks.
- **Architectural reviews start with layer-coverage.** "Which of the seven layers are missing?" is the right opening question for any "agent that runs unattended" feature.
- **Plugin marketplace metadata grows a `layers` field.** A plugin declares which layers it provides; the marketplace surfaces gaps when the user installs an incomplete combination.
- **Verifier coverage is the ceiling.** [242-verifiability-bottleneck-and-jagged-skills](242-verifiability-bottleneck-and-jagged-skills.md) is the load-bearing companion; the seven layers cannot exceed the verifier they share.
- **Cross-link with [277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md).** The UX surfaces (plan mode, streaming, permission review, team panel, memory UX, voice composition) are *orthogonal* to the seven layers — they are *how the user observes* the stack, not *what the stack does*.
- **Cross-link with [278-agent-unit-economics-2026](278-agent-unit-economics-2026.md).** Per-feature cost bounds are layer-3 enforced; the four cost layers and the seven autonomy layers are different decompositions of the same surface.
- **Cross-link with [273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md).** Deny patterns in the contract envelope are the same artefact as bright-line gates in the security stack — naming alignment matters for compliance.
- **Cross-link with [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md), [267-agent-sre-2026](267-agent-sre-2026.md).** Layer 7 (autopilot) is where these primitives land.
- **Cross-link with [263-production-agent-runtime-synthesis-2026](263-production-agent-runtime-synthesis-2026.md), [258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md), [268-agent-operations-synthesis-2026](268-agent-operations-synthesis-2026.md).** The autonomy-loop stack composes *above* the protocol stack and runtime selection; it is the *operator-discipline layer* that makes the lower stacks productive.
- **Cross-link with [238-karpathy-agentic-engineering-shift](238-karpathy-agentic-engineering-shift.md).** The shift to agentic engineering is, mechanically, the shift to having all seven layers as reusable primitives rather than ad-hoc scripts.

## Practical takeaways

1. **Treat the seven primitives as a stack, not a menu.** Pick a contiguous prefix from the bottom; do not skip layers.
2. **Verifier first.** Layers 2, 3, 4 all consume a verifier predicate; build the verifier before any of them.
3. **Bound the worst case before you build the happy path.** Layers 1+3 ship before layer 2.
4. **Make resume explicit.** Layer 7's silent auto-replay is the bug that destroys checkpointing's value.
5. **Refuse `--dangerously-skip-permissions`.** It is the layer-0 escape hatch; the stack assumes it is off.
6. **Postmortems start at the contract's `terminal_cause`.** Trace trawls are a 2024 habit.
7. **Layer omission is the dominant failure mode.** Whenever you find a runaway, look for the missing layer.

## When to use this synthesis

Cite this file in any architecture review of an autonomous-agent feature, any framework-design RFC, any safety review, any incident postmortem, and any onboarding doc for engineers joining a coding-agent team. The seven-primitive frame is now the *common vocabulary* the field uses; using it lowers the bar to alignment.

**The one-line takeaway for harness designers.** The autonomy loop is a stack of seven composable primitives — inner loop, Stop hook, contract, Ralph, directive, hibernate-and-wake, autopilot — pick a contiguous prefix from the bottom, ship the verifier first, refuse the layer-0 escape hatch, and your agent will finish a 50-story PRD overnight on a $5 budget without asking and without burning $47,000.

## References

1. [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md) — the canonical Ralph deep-dive (snarktank).
2. [307-ralph-loop-variations-2026](307-ralph-loop-variations-2026.md) — the three-variant comparison.
3. [305-agent-contracts-formal-framework](305-agent-contracts-formal-framework.md) — the FULFILLED/VIOLATED/EXPIRED/TERMINATED state machine.
4. [306-stop-hook-auto-continue-pattern](306-stop-hook-auto-continue-pattern.md) — the Stop hook + four safeguards.
5. [160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md) — HUMAN_DIRECTIVE.md mechanism + zero-cost monitoring.
6. [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md) — durable-execution as the conceptual ancestor.
7. [01-agent-loop-architecture](01-agent-loop-architecture.md), [05-hooks](05-hooks.md), [09-memory-files](09-memory-files.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md) — the foundational primitives.
8. [62-everything-claude-code](62-everything-claude-code.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md) — the operational substrates.
9. [238-karpathy-agentic-engineering-shift](238-karpathy-agentic-engineering-shift.md), [242-verifiability-bottleneck-and-jagged-skills](242-verifiability-bottleneck-and-jagged-skills.md) — the conceptual frame.
10. [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md), [267-agent-sre-2026](267-agent-sre-2026.md), [273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md), [277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md), [278-agent-unit-economics-2026](278-agent-unit-economics-2026.md) — the surfaces this stack composes against.
11. [LYRA_V3_12_AUTONOMY_LOOP_PLAN.md](../projects/lyra/LYRA_V3_12_AUTONOMY_LOOP_PLAN.md) — the eight-phase apply plan that operationalises the seven primitives in a single coding-agent harness.
12. External: snarktank/ralph, frankbria/ralph-claude-code, vercel-labs/ralph-loop-agent, Geoff Huntley *Ralph Wiggum*, Anthropic Claude Code Hooks, Agent Contracts arXiv 2601.08815, Deep Researcher Agent 24/7 arXiv:2604.05854.
