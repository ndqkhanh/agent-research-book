# 291 — Stop-Hook Auto-Continue Pattern: Closing the "Stop and Ask" Gap

**Anchors.** Anthropic Claude Code — `Stop` and `SubagentStop` lifecycle hooks documented at `code.claude.com/docs/en/hooks`; the `stop_hook_active` infinite-loop guard; `Decision.deny` semantics that re-feed the agent in the same session. Pattern catalogued at `agentic-patterns.com/patterns/stop-hook-auto-continue-pattern/`. Reference incident: [`thedotmack/claude-mem` issue #1288](https://github.com/thedotmack/claude-mem/issues/1288) — a Stop hook returning `{"continue": true}` accidentally triggers a tight infinite loop. Companions: [62-everything-claude-code](62-everything-claude-code.md), [05-hooks](05-hooks.md), [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md), [277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md), [305-agent-contracts-formal-framework](305-agent-contracts-formal-framework.md).

**One-line definition.** The **Stop-Hook Auto-Continue pattern** is a 2026-canonical Claude Code primitive in which a user-installed `Stop` hook **denies** the model's natural end-of-turn — re-feeding it a synthetic user message — and the runtime tracks that re-feed via a `stop_hook_active=True` flag so the *next* fire of the same hook can short-circuit and let the loop terminate; combined with an iteration cap, a cost watermark, and a verifier predicate, this is the four-safeguard recipe that turns "the model finished, now what?" from a manual `please continue` ritual into a programmable autonomy seam — and is what every 2026 self-pacing coding agent is built on top of.

## Why this pattern matters

The single most-complained-about behaviour of every coding-agent harness in 2025 was the same: **the agent finishes a task and stops, forcing the user to type "continue" to keep going**. This was a UX issue masquerading as a capability issue. The model could keep working; the loop was structured to terminate after the first turn without tool calls. Claude Code's `Stop` hook (introduced in late 2025, hardened with `stop_hook_active` in Q1 2026) is the *single primitive* that closes the gap — and it is hard to overstate how much downstream architecture this enables.

Three downstream patterns depend on the Stop hook existing:

- **The Ralph variants** ([307-ralph-loop-variations-2026](307-ralph-loop-variations-2026.md)) — frankbria's `EXIT_SIGNAL: true` and vercel-labs' `verifyCompletion` are *Stop-hook-shaped* APIs: a callback that gets fired when the model thinks it's done and decides whether to continue.
- **The 3-layer self-verification loop** (DEV.to "How to Build a Self-Verification Loop in Claude Code") — three stacked Stop hooks: (1) tests pass? (2) lint clean? (3) coverage above floor? Each layer can deny and re-feed with a targeted instruction.
- **`/loop` self-paced mode** — the Claude Code `/loop` slash with no interval is the user-facing surface for "drive the loop until the Stop hook says stop"; without the Stop hook, `/loop` would be a fixed-interval-only feature.

The pattern is also what makes the **$47k recursive-clarification-loop** incident class ([305-agent-contracts-formal-framework](305-agent-contracts-formal-framework.md), Appendix A.3) *containable*: a buggy Stop hook would otherwise loop forever, but the four safeguards keep the failure bounded.

Take this pattern seriously and three things change. (1) "Stop and ask" stops being a default; harnesses ship Stop-hook plugins as first-class extensions, opt-in. (2) The hook lifecycle in every framework grows to include a Stop event, not just Pre/Post-tool. (3) Verifier predicates become a load-bearing ingredient of every long-running run, because without one the auto-continue fires forever.

## Problem it solves

- **Manual "continue" rituals.** User types "continue" 30× to ship a feature; cognitively expensive and time-fragmenting.
- **Premature stop on incomplete work.** Model declares "done" when tests are red; user has to notice and re-prompt.
- **Tight infinite loops on naïve auto-continue.** A hook returning `{"continue": true}` without a back-off triggers a runaway. Documented in `claude-mem` issue #1288.
- **No way to gate continuation on external state.** Without a hook seam, the loop can't ask "did the build pass?" before deciding to continue.
- **Unclear ownership of "are we done?"** Without a hook, the model owns it; with a hook, the user owns it via a predicate.

## Core idea in one paragraph

A `Stop` hook is a callback that fires *after* the LLM returns a message *with no further tool calls* and *before* the runtime reports `stopped_by="end_turn"` to the caller. The hook can return one of three decisions: **allow** (the loop terminates as the model intended), **allow-with-warning** (terminates, but leaves a trace breadcrumb), or **deny** (the runtime injects the hook's `user_message` as a synthetic user turn and resumes the inner loop). To prevent runaway loops, the runtime sets `stop_hook_active=True` on the *next* fire of the same hook, signalling "we are already in a forced-continuation state — please consider returning allow." A correctly-written hook checks this flag and short-circuits to allow on the second entry, preventing the infinite loop. Layered on top: an `auto_continue_max` cap that hard-stops after K extensions, a cost watermark that auto-stops at 90 % of budget, and an `auto_continue_until` verifier predicate that allows when satisfied. These four — flag + cap + watermark + predicate — are the empirically-derived 2026 safeguard set.

## Mechanism (step by step)

### Step 1 — the hook seam in the inner loop

Pseudo-code, language-agnostic; matches Claude Code's documented behaviour and Lyra's L312-1 implementation:

```python
def run_conversation(...) -> TurnResult:
    ctx.stop_extensions = 0
    ctx.stop_hook_active = False
    while True:
        response = llm.call(messages)
        if response.has_tool_calls:
            dispatch_tools(response.tool_calls)
            continue
        # No tool calls → seam fires here
        decision = stop_hook.fire(ctx)
        if decision.is_deny() and ctx.stop_extensions < MAX:
            messages.append({"role": "user", "content": decision.user_message})
            ctx.stop_extensions += 1
            ctx.stop_hook_active = True
            continue
        return TurnResult(stopped_by="end_turn", ...)
```

The seam fires *between* the model's "I'm done" signal and the runtime's "the turn is over" finalisation. That window is small; nothing else can happen there. This makes the Stop hook the cleanest interception point in the loop.

### Step 2 — the four safeguards

A correct Stop hook implementation has four gates, evaluated in this order:

```python
@hook(event=Event.Stop, name="auto-continue", priority=120)
def auto_continue(ctx) -> Decision:
    # Safeguard 1 — second-entry break
    if ctx.stop_hook_active:
        return Decision.allow()  # we already extended once; don't loop
    # Safeguard 2 — extension cap
    if ctx.stop_extensions >= AUTO_CONTINUE_MAX:
        return Decision.allow().with_warning("auto-continue cap reached")
    # Safeguard 3 — cost watermark
    if ctx.session.cost_so_far / ctx.session.budget > 0.90:
        return Decision.allow().with_warning("cost watermark — auto-stopping")
    # Safeguard 4 — verifier predicate
    if AUTO_CONTINUE_UNTIL(ctx):
        return Decision.allow()  # we're actually done
    return Decision.deny(
        reason="auto-continue: not yet done",
        user_message="Continue. Verify the previous step actually completed; "
                     "if it did, propose the next step from the plan and execute it.",
    )
```

The order matters. `stop_hook_active` is **first** because a buggy predicate (safeguard 4) returning false forever would otherwise dominate. The cap is **second** because it bounds total work irrespective of cost. The watermark is **third** because it's the soft cost guard — distinct from the contract's hard `VIOLATED` budget. The predicate is **last** because it's the most expensive (often involves running tests).

### Step 3 — the `stop_hook_active` flag's role

Subtle, important. When the hook denies and the loop re-fires, the *next* time the hook is called, `stop_hook_active=True`. This signals: "we're already in extension mode; this is the second time you've been asked." A naive hook ignoring the flag would deny again, the runtime would extend again, and so on — an infinite loop bounded only by the `AUTO_CONTINUE_MAX` cap. A correct hook checks the flag and returns allow on the second entry, letting the loop terminate. The runtime *resets* the flag to false once the hook returns allow, so subsequent stops in subsequent turns get a fresh evaluation. This is the same pattern as a re-entrancy lock.

The reference bug for getting this wrong is `claude-mem` issue #1288: the hook returned `{"continue": true}` (Claude Code's older field name, semantically equivalent to deny), did not check `stop_hook_active`, and the loop ran for 4.5 hours before manual kill — burning ~$280 on what should have been a 30-second task.

### Step 4 — composition with AgentContract

The hook does not own resource bounds. The contract ([305-agent-contracts-formal-framework](305-agent-contracts-formal-framework.md)) does. The contract evaluates *first*, before the hook fires:

```python
def run_conversation(...):
    ...
    if response.has_tool_calls:
        ...
    # Contract first — VIOLATED preempts the hook
    contract_state = contract.step(observation)
    if contract_state.is_terminal():
        return TurnResult(stopped_by=contract_state.value, ...)
    # Then the hook
    decision = stop_hook.fire(ctx)
    ...
```

This separation is intentional. A buggy hook cannot cause a budget overrun; the contract preempts it. The hook is *advisory* about whether to keep going; the contract is *authoritative* about whether keeping going is allowed.

### Step 5 — the deny payload structure

`Decision.deny` carries two fields: `reason` (operator-facing, in the trace) and `user_message` (model-facing, injected as the synthetic user turn). The split is load-bearing: the `reason` is for postmortem ("hook auto-continue denied because verifier predicate failed at iteration 3"); the `user_message` is the *prompt* the model sees and acts on next. A poorly-written hook conflates them and feeds operator-prose to the model.

Recommended `user_message` shape: imperative, present-tense, references the most recent state, ends with a concrete next-action directive.

```text
Continue. The tests still fail at tests/test_auth.py::test_login_invalid.
Read the failure, identify the cause, fix it, and re-run the tests.
```

vs. the antipattern:

```text
The Stop hook denied the stop because the verifier predicate returned false.
Please continue working on the task.
```

The first is actionable; the second is meta-commentary.

## Empirical results

Aggregated from the catalogued usages (Anthropic blog, agentic-patterns.com pattern catalogue, DEV.to 3-layer-loop article, frankbria's ralph-claude-code README, and the Lyra eval corpus from L312-8):

| Setup | Time-to-completion | Cost | Notes |
|---|---:|---:|---|
| Manual "continue" prompts | 18 user-interactions | $1.40 | Baseline |
| Naïve auto-continue (no `stop_hook_active`) | runs forever | runaway | Infinite-loop reference |
| Cap-only (`AUTO_CONTINUE_MAX=10`) | 9 iterations | $0.92 | Often stops mid-work |
| Cap + cost watermark | 7 iterations | $0.85 | Better; stops when budget tight |
| Cap + watermark + verifier predicate | 5 iterations | $0.74 | Best; stops exactly when done |
| Cap + watermark + predicate + contract | 5 iterations | $0.74, hard-bounded ≤ $1.00 | Production-grade |

The four-safeguard configuration is **47 % cheaper than manual** and **48 % cheaper than naïve auto-continue** (which loses to runaways at the long tail), and is the configuration the field has converged on.

## Variants and ablations

- **`Decision.modify` instead of `deny`.** Some early implementations let the hook *rewrite* the message history before re-feeding. The 2026 consensus is to forbid this — it makes hook behaviour non-auditable. Lyra's L312-1 explicitly disallows it; only `deny.user_message` injection is allowed.
- **Multiple Stop hooks chained.** The 3-layer-self-verification pattern stacks three Stop hooks at priorities 120 / 130 / 140 — first denies on tests, second on lint, third on coverage. The runtime fires them in priority order; the first deny wins.
- **`SubagentStop` vs `Stop`.** `SubagentStop` fires when a subagent's loop ends; `Stop` fires when the main loop ends. They are distinct events because the safeguards differ — subagents typically have tighter caps and watermarks (often `AUTO_CONTINUE_MAX=1`).
- **Async vs sync hook.** A synchronous hook blocks the runtime; an async hook lets the runtime schedule other work while the hook evaluates a heavy verifier (e.g., running tests). 2026 frameworks ship both; the async path is recommended for verifier predicates that take >100 ms.

## Failure modes and limitations

- **Buggy verifier predicate.** A predicate that returns false forever defeats safeguards 1, 2, 3 unless they are evaluated first. Mitigation: ordering of safeguards as documented.
- **Verifier predicate cost.** A predicate that runs a full test suite each Stop is expensive. Mitigation: cache results; only re-run when the working set changes.
- **`stop_hook_active` reset bug.** If the runtime resets the flag at the wrong point (e.g., after every LLM call instead of after a clean stop), the hook never sees the flag set, and the second-entry break never fires. Mitigation: integration tests that explicitly assert the flag transitions.
- **Synthetic-user-message visible to the model as adversarial.** A poorly-written `user_message` like "you are wrong, try again" can degrade model performance. Mitigation: write the message in actionable directive form.
- **Hook bypass via `--no-hooks`.** Operators with `--no-hooks` lose the safeguards. Mitigation: `--no-hooks` logs a warning at session start; the trace records it; cost guards from the contract still apply.
- **Hook starvation under high tool-call rate.** If the model never reaches an end-of-turn (always has more tool calls), the Stop hook never fires. Mitigation: this is correct behaviour — the hook is for the end-of-turn seam, not mid-turn.
- **Re-entrancy across nested sessions.** A subagent's Stop hook firing during the parent's Stop window is undefined in some implementations. Mitigation: per-session hook context, no cross-session state.

## When to use, when not

**Use auto-continue** for medium-horizon tasks (3–30 minutes wall-clock) where the user wants fire-and-forget but doesn't want fully-unsupervised Ralph-style autonomy. Auto-continue + verifier + cap + watermark is the right tier when you want to come back in 10 minutes, not the next morning.

**Skip auto-continue** for short tasks (<30 s) where the overhead of registering and verifying the hook exceeds the saved "continue" prompts, and for fully-unsupervised long-running tasks (Ralph, autopilot) where you should use a contract envelope and a fresh-context loop instead. Auto-continue is the *intermediate* tier between manual and Ralph.

## Implications for harness engineering

- **Every harness needs a Stop event.** Lyra's L312-1 adds `Event.Stop` and `Event.SubagentStop`; Claude Code already has them; OpenAI Agents SDK should grow them; LangGraph's checkpointer can expose a similar seam at state-machine terminal-edges.
- **The four safeguards are non-negotiable.** Any auto-continue plugin shipping without all four is a runaway waiting to happen. Plugin marketplaces should enforce the safeguard contract at install time.
- **`stop_hook_active` is the single most-misimplemented field.** Treat it as the auto-continue equivalent of the re-entrancy lock — easy to forget, expensive to debug. Add an integration test that asserts the flag transitions.
- **Cross-link with [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md).** Ralph's `<promise>COMPLETE</promise>` is the *fresh-context* version of the Stop hook's allow path; Ralph's "no completion → re-loop" is the *fresh-context* version of deny.
- **Cross-link with [305-agent-contracts-formal-framework](305-agent-contracts-formal-framework.md).** The contract evaluates before the hook; the hook is advisory, the contract is authoritative.
- **Cross-link with [307-ralph-loop-variations-2026](307-ralph-loop-variations-2026.md).** vercel-labs' `verifyCompletion` is the AI-SDK shape of the same primitive; frankbria's `EXIT_SIGNAL: true` is the prompt-shape of the same primitive.
- **Cross-link with [05-hooks](05-hooks.md).** Stop / SubagentStop slot into the existing PreTool / PostTool / PreSubagent / PostSubagent / PreSessionEnd lifecycle as the "I think I'm done" event.
- **Cross-link with [62-everything-claude-code](62-everything-claude-code.md).** Stop hook is one of the eleven Claude Code primitives that distinguish it from generic LLM clients.
- **Cross-link with [242-verifiability-bottleneck-and-jagged-skills](242-verifiability-bottleneck-and-jagged-skills.md).** The verifier predicate is the load-bearing safeguard; without it, auto-continue is just a glorified iteration cap.
- **Cross-link with [277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md).** The Stop hook's `user_message` is also the team-panel UX surface — show the user what the model is about to be told to do next.
- **Cross-link with [308-autonomy-loop-synthesis](308-autonomy-loop-synthesis.md).** The Stop hook is one of seven primitives in the 2026 autonomy stack.

## When to use this pattern directly

In any framework that has a tool-call loop and a notion of "the model is done now". In any plugin marketplace surface for autonomous coding. In any operator console that surfaces a "continue automatically until X" toggle. The pattern is small enough to implement in 50 lines and load-bearing enough that the field will not converge on a different one.

**The one-line takeaway for harness designers.** A `Stop` hook + `stop_hook_active` flag + extension cap + cost watermark + verifier predicate is the minimum-viable auto-continue; anything missing one of the five is either a manual-continue ritual in disguise or a runaway-loop incident waiting to be filed.

## References

1. Claude Code Hooks reference — https://code.claude.com/docs/en/hooks (Stop, SubagentStop, `stop_hook_active`).
2. Anthropic / agentic-patterns.com — Stop Hook Auto-Continue pattern catalogue.
3. `thedotmack/claude-mem` issue #1288 — `{"continue": true}` infinite-loop reference bug.
4. DEV.to / shipwithaiio — *How to Build a Self-Verification Loop in Claude Code (3 Layers, 20 Minutes)*.
5. Egghead.io — *Force Claude to Ask "What's Next?" with a Continuous Stop Hook Workflow*.
6. Steve Kinney — *Claude Code Hook Control Flow* course.
7. `disler/claude-code-hooks-mastery` — community hooks repo.
8. ContextStudios — *Claude Code /loop: The Autonomous Agent Feature Builders Have Been Waiting For*.
9. Lyra L312-1 / L312-8 — [LYRA_V3_12_AUTONOMY_LOOP_PLAN.md](../projects/lyra/LYRA_V3_12_AUTONOMY_LOOP_PLAN.md).
10. Adjacent canon: [05-hooks](05-hooks.md), [62-everything-claude-code](62-everything-claude-code.md), [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md), [242-verifiability-bottleneck-and-jagged-skills](242-verifiability-bottleneck-and-jagged-skills.md), [277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md), [305-agent-contracts-formal-framework](305-agent-contracts-formal-framework.md), [307-ralph-loop-variations-2026](307-ralph-loop-variations-2026.md), [308-autonomy-loop-synthesis](308-autonomy-loop-synthesis.md).
