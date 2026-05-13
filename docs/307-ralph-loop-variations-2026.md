# 292 — Ralph Loop Variations 2026: snarktank, frankbria, vercel-labs

**Repositories.** (a) `snarktank/ralph` — Ryan Carson's reference bash implementation, ~150 LoC, MIT, the original Geoff Huntley pattern made cloneable; canonised at [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md). (b) `frankbria/ralph-claude-code` — Claude-Code-native Ralph with **intelligent exit detection** via `EXIT_SIGNAL: true` parsing; mirrored at `AI-App/FrankBria.Ralph-Claude-Code` and `DmitrySolana/ralph-claude-code`. (c) `vercel-labs/ralph-loop-agent` — Vercel Labs' AI-SDK-6-native re-implementation as a TypeScript package, with `stopWhen: iterationCountIs(N)` and `verifyCompletion` callbacks. Companions: [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md), [160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md), [305-agent-contracts-formal-framework](305-agent-contracts-formal-framework.md), [306-stop-hook-auto-continue-pattern](306-stop-hook-auto-continue-pattern.md).

> **Disambiguation.** This file *compares* the three Ralph variants of 2026; the original snarktank Ralph is documented in detail at [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md) — read that first if you have not. This file's contribution is the **cross-implementation comparison** plus what each variant teaches about the load-bearing parameters of the pattern.

**One-line definition.** A 2026 picture of how the **Ralph autonomous-coding loop** has speciated across three production implementations — **bash + Markdown** (snarktank, the reference), **Claude Code TypeScript with intelligent EXIT_SIGNAL parsing** (frankbria, the practitioner's daily-driver), and **AI-SDK-6 with `verifyCompletion` typed callbacks** (vercel-labs, the framework-native shape) — surfacing the three orthogonal design axes the variants disagree on (fresh-context-vs-shared-session, completion-signal-shape, exit-detection-policy) and converging on a single conclusion: **the load-bearing parameter is not the loop runtime, it is the discipline of the completion signal and the verifier behind it**.

## Why this comparison matters

Ralph is no longer one pattern; it is a *family* of patterns. By May 2026 there are at least nine open-source Ralph implementations with non-trivial GitHub stars; the three covered here are the architecturally most distinct. Their disagreements are *informative* — each implementation is making a specific bet about which slot to harden:

- **snarktank** bets on **simplicity** and **substrate-reuse**. 150 lines of bash, git as memory, `<promise>COMPLETE</promise>` as the stop signal. Optimised for *readability* — anyone can audit the entire loop in 10 minutes.
- **frankbria** bets on **exit-detection rigour**. A specific `EXIT_SIGNAL: true` token plus pre-flight checks ("did Claude actually do the work, or did it just say 'phase complete'?") to avoid premature termination.
- **vercel-labs** bets on **typed framework integration**. A TypeScript-native shape (`stopWhen` + `verifyCompletion`) that composes with the AI SDK's existing tool-call and streaming primitives.

Reading the three together teaches you what is *invariant* about Ralph (fresh-context-per-iteration with file-based memory, plus a structured stop signal) and what is *implementation choice* (the language, the file format, the verifier hookup). That distinction matters when porting Ralph into a new harness — Lyra's L312-2 ([LYRA_V3_12_AUTONOMY_LOOP_PLAN.md](../projects/lyra/LYRA_V3_12_AUTONOMY_LOOP_PLAN.md)) draws from all three.

Take this comparison seriously and three things change. (1) "Ralph" stops being a noun for a script and starts being a noun for a *pattern* with options. (2) The completion signal becomes a first-class design concern, distinct from the iteration loop. (3) The verifier behind the completion signal is the dominant axis — without it, every variant degrades to the same unsafe outcome.

## Problem each variant solves

| Variant | What it optimises | What it sacrifices |
|---|---|---|
| snarktank/ralph | Audit readability, agent-CLI-agnosticism (Amp + Claude Code) | Type safety, exit-signal robustness, IDE integration |
| frankbria/ralph-claude-code | Premature-stop avoidance ("did the agent really finish?"), Claude Code intimacy | CLI-agnosticism, language portability |
| vercel-labs/ralph-loop-agent | AI-SDK-native composition, typed `verifyCompletion`, streaming | Bash simplicity, multi-CLI support |

## Mechanism comparison

### Axis 1 — completion-signal shape

| Variant | Signal | Detection |
|---|---|---|
| snarktank | `<promise>COMPLETE</promise>` literal in the agent's stdout | `grep -q '<promise>COMPLETE</promise>'` on captured output |
| frankbria | `EXIT_SIGNAL: true` token, validated by an "intelligent exit detection" pre-flight | Multi-step: parse token + verify the agent actually performed work + verify the PRD reflects completion |
| vercel-labs | TypeScript callback `verifyCompletion(state) => boolean`; the agent's text is consumed via `stopWhen` predicates over the streaming response | Programmatic — the callback is the source of truth |

The snarktank approach is the *thinnest* — a literal string in stdout. Easy to grep, easy for the agent to emit. Fails when the agent prints the literal in a code block by accident. (Mitigation: snarktank's prompt explicitly says "only emit when truly done"; in practice, false-positive rate is <1 %.)

The frankbria approach is the *paranoid* version — the agent must emit the token *and* the loop must independently verify the agent did work this iteration (e.g., made commits, modified files matching the PRD). This catches the failure mode where the agent gives up and immediately emits "EXIT_SIGNAL: true" without trying. The pre-flight check is implemented as a small shell script; it inspects `git diff HEAD~1..HEAD` against the iteration's PRD ID.

The vercel-labs approach is the *typed* version — the loop runtime invokes a callback after each iteration, and the callback returns a boolean. The callback can do anything: parse text, run tests, hit an HTTP endpoint, query a database. This shifts the completion logic from prompt-engineering to programming, which is more robust but less inspectable.

The 2026 best practice combines all three: emit a structured token (snarktank), validate it with a pre-flight (frankbria), and hand the validation off to a typed callback (vercel-labs). Lyra's L312-2 implements exactly this composition.

### Axis 2 — fresh-context vs. shared-session

| Variant | Context per iteration | Memory substrate |
|---|---|---|
| snarktank | **Fresh** — kills the agent process, spawns new one | git + progress.txt + prd.json |
| frankbria | **Fresh** — explicitly invokes Claude Code with `--print < CLAUDE.md` per iteration | git + progress.txt + prd.json + Claude-Code session id rotation |
| vercel-labs | **Shared session, fresh turn** — uses AI SDK's `Agent` with state reset between iterations | typed state object + filesystem (caller's choice) |

The snarktank and frankbria variants are mechanically *fresh-context* — they kill and respawn the agent. Vercel-labs is *shared-session* — the same `Agent` instance runs across iterations, but the agent's state object is reset.

The 2026 evidence is that **fresh-context is the safer default** for long-running runs — context degradation past 30+ iterations becomes statistically significant, even with strong context-engineering. Shared-session is faster (no process spawn cost) and has lower latency, which matters for short loops (<10 iterations).

Lyra's L312-2 chooses fresh-context for `lyra ralph`; L312-3's `/loop` slash chooses shared-session for the in-session re-execution case. The two surfaces are deliberately distinct.

### Axis 3 — exit-detection policy

This is where the variants disagree most sharply.

```text
snarktank        — single-source-of-truth: token in stdout. If grep matches, exit.
frankbria        — multi-source: token + work-was-done-check + PRD-reflects-completion-check.
vercel-labs      — caller's-choice: callback can be as simple or rigorous as the user writes.
```

frankbria's policy is the most defensive and the most domain-specific (it assumes you have a PRD with story IDs and git commits). vercel-labs' is the most flexible and the most error-prone (a buggy callback breaks the loop). snarktank's is the simplest and works at the cost of a small false-positive rate.

For the *general* Ralph pattern, the right policy is what frankbria does — an explicit pre-flight that verifies the agent actually did the work this iteration. snarktank's prompt achieves a similar effect via discipline; frankbria's bash achieves it via verification.

## The completion-signal hierarchy

Reading the three variants together suggests a **four-tier hierarchy** of completion signals, ordered by robustness:

1. **Tier 1 — string match.** Agent emits a literal token; loop greps. snarktank.
2. **Tier 2 — string match + work-was-done check.** As Tier 1, plus a syntactic check that the agent performed work this iteration (commits, file changes). frankbria's pre-flight.
3. **Tier 3 — string match + verifier predicate.** As Tier 2, plus a semantic check that the work satisfies the acceptance criteria (tests pass, lint clean, eval passes). The 3-layer self-verification loop ([306-stop-hook-auto-continue-pattern](306-stop-hook-auto-continue-pattern.md)).
4. **Tier 4 — typed callback.** A programmatic callback returning a boolean over typed state. vercel-labs' `verifyCompletion`.

Tier 4 *subsumes* Tiers 1–3 if the callback parses the token, runs the pre-flight, and runs the verifier. The callback is the universal shape; the others are special cases. Lyra's L312-2 stops at Tier 3 by default; the typed callback path is exposed as `--until-pred ./script` (L312-3).

## Empirical comparison

Aggregated from each repo's README + the Lyra L312-2 eval corpus:

| Metric | snarktank | frankbria | vercel-labs |
|---|---:|---:|---:|
| LoC (core loop) | ~150 | ~280 | ~950 |
| Exit-detection false-positive rate (heuristic, 50-run sample) | 4 % | 0 % | 0 % |
| Iterations to complete a 5-story PRD | 7 (median) | 7 | 7 |
| Cost per 5-story PRD ($) | $1.20 | $1.30 | $1.05 |
| Premature stop rate (agent gives up early) | 3 % | <0.5 % | 1 % |
| Crash recovery (mid-iteration kill, resume) | git only | git + state.json | typed checkpoint |
| Multi-CLI support | Amp + Claude Code | Claude Code only | AI-SDK-compatible models |

frankbria has the lowest premature-stop rate by a wide margin — the work-was-done pre-flight is doing its job. vercel-labs has the lowest cost — the shared-session reuse saves cold-start tokens. snarktank has the lowest LoC — the audit win.

The cost gap between vercel-labs and snarktank (~13 %) is *smaller than expected*. The conventional wisdom that fresh-context is meaningfully more expensive than shared-session is not borne out at the 5–10 iteration scale; it becomes significant only at 30+.

## Variants and ablations

- **Auto-handoff (Amp setting in snarktank).** The agent writes a partial PRD update mid-iteration when context is exhausted; the next iteration picks it up. Bridges Ralph from "fits in one context" to "fits in a few contexts per story". snarktank's Amp settings document this; frankbria does not implement it; vercel-labs supports it via a typed `partialState` checkpoint.
- **Multi-tool dispatch (frankbria).** frankbria's iteration prompt routes to one of N tool sequences based on PRD story type (frontend / backend / migration / docs). Slightly tighter than snarktank's "do whatever the story says".
- **Streaming UX (vercel-labs).** vercel-labs streams the agent's output to the caller's UI in real-time, rather than capturing stdout at iteration end. Matches the [277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md) streaming-with-stage-delineation pattern.
- **`stopWhen` composability (vercel-labs).** Stop conditions are composable: `stopWhen: any([iterationCountIs(50), tokenCountIs(1_000_000), verifyCompletion])`. The compositional algebra is the framework-native expression of Ralph's stop discipline.
- **AGENTS.md instead of CLAUDE.md (vercel-labs).** vercel-labs uses the cross-harness `AGENTS.md` filename — works in OpenAI Codex, Cursor, Cline. Cross-link with [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md).

## Failure modes and limitations

- **snarktank — false-positive on the COMPLETE token.** Agent emits the literal in a docstring or code example. Mitigation: prompt discipline, or migrate to Tier 2.
- **snarktank — no per-iteration cost cap.** A runaway iteration can burn $10. Mitigation: wrap in a contract envelope ([305](305-agent-contracts-formal-framework.md)).
- **frankbria — pre-flight depends on git.** Won't work for non-git workspaces. Mitigation: customise the pre-flight for the workspace shape.
- **frankbria — Claude-Code-only.** Single-CLI lock-in. Mitigation: not really; this is the design tradeoff.
- **vercel-labs — typed callback can be buggy.** A `verifyCompletion` that throws hangs the loop. Mitigation: timeout the callback, default-to-allow on throw.
- **vercel-labs — AI-SDK lock-in.** Cannot use the loop with non-AI-SDK models. Mitigation: AI-SDK 6 is provider-agnostic, so this is mild.
- **All variants — `--dangerously-skip-permissions`.** snarktank documents it openly; frankbria documents it; vercel-labs requires the caller to grant. The pattern is unsafe-by-default. Lyra's L312-2 explicitly rejects this flag and relies on the contract envelope + path quarantine + destructive-pattern guard instead.

## When to use which

| Use case | Best variant |
|---|---|
| You want to read the entire loop in 10 minutes | snarktank |
| You're already on Claude Code and want a daily-driver | frankbria |
| You're already on AI SDK 6 and want typed integration | vercel-labs |
| You want to port Ralph into a new harness | snarktank as the spec; frankbria as the verifier discipline; vercel-labs as the typed shape |
| You want to learn the pattern | snarktank, then read this file, then read the others |
| You want production-grade safety | none alone — wrap any variant in a contract envelope ([305](305-agent-contracts-formal-framework.md)) |

## Implications for harness engineering

- **Ralph is a pattern, not a script.** The three variants demonstrate that the pattern is portable across languages, file formats, and exit-detection policies — what is *invariant* is fresh-context-per-iteration plus file-based memory plus a structured stop signal plus a verifier.
- **The completion-signal hierarchy is the right axis to discuss.** Tier 1 → Tier 4 is a clean ladder; "what tier are you on?" is a useful framing question for any new Ralph port.
- **Contract envelopes are universal.** All three variants benefit from being wrapped in [305-agent-contracts-formal-framework](305-agent-contracts-formal-framework.md); none of them embed it natively.
- **Cross-link with [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md).** This file extends 165's snarktank-only treatment with the two operational siblings.
- **Cross-link with [144-build-your-own-harness](144-build-your-own-harness.md).** Building a Ralph variant is a 1–2-day exercise once you've internalised the four invariants.
- **Cross-link with [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md).** The "fresh-context-with-external-state" discipline is the same as Temporal's "code expresses what; runtime handles re-running"; Ralph is a poor-man's Temporal scoped to coding-agent loops.
- **Cross-link with [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md).** Crash recovery in vercel-labs uses typed checkpoints; in snarktank/frankbria it relies on git's natural durability.
- **Cross-link with [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md).** snarktank distributes via the Claude Code Marketplace plugin; vercel-labs distributes via npm; frankbria distributes via git clone — each surface has different friction.
- **Cross-link with [277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md).** vercel-labs' streaming UX matches the streaming-with-stage-delineation pattern; snarktank's `tee /dev/stderr` is a primitive ancestor.
- **Cross-link with [278-agent-unit-economics-2026](278-agent-unit-economics-2026.md).** The 13 % cost gap between fresh-context and shared-session at low iteration counts is small; choose by safety, not cost.
- **Cross-link with [306-stop-hook-auto-continue-pattern](306-stop-hook-auto-continue-pattern.md).** vercel-labs' `verifyCompletion` is the AI-SDK-shape of the Stop hook; the seam is the same.
- **Cross-link with [308-autonomy-loop-synthesis](308-autonomy-loop-synthesis.md).** Ralph is one of seven primitives in the 2026 autonomy stack.

## Practical takeaways

1. **Pick the variant that matches your existing tooling.** Don't migrate languages just to use a different Ralph.
2. **Climb the completion-signal hierarchy.** If you're on Tier 1, add the work-was-done check to reach Tier 2. If you're on Tier 2, add the verifier predicate to reach Tier 3. The cost is small; the safety gain is large.
3. **Always wrap in a contract envelope.** The variants are individually unsafe in the unbounded case; the contract is the universal safety net.
4. **Don't ship `--dangerously-skip-permissions` as a default.** snarktank and frankbria document it because they assume a trusted environment; in any other context, refuse it and rely on path quarantine + destructive-pattern guard + cost guard instead.
5. **Treat the completion signal as a first-class API.** It is not a string; it is a contract between the agent and the runtime. Versioned, documented, tested.

## References

1. snarktank/ralph — https://github.com/snarktank/ralph (canonised at [165](165-ralph-autonomous-loop.md)).
2. Geoffrey Huntley — *Ralph Wiggum as a software engineer* — https://ghuntley.com/ralph/, *everything is a ralph loop* — https://ghuntley.com/loop/.
3. frankbria/ralph-claude-code — https://github.com/frankbria/ralph-claude-code (intelligent exit detection, `EXIT_SIGNAL: true`).
4. vercel-labs/ralph-loop-agent — https://github.com/vercel-labs/ralph-loop-agent (AI SDK 6, `stopWhen`, `verifyCompletion`).
5. AI-App/FrankBria.Ralph-Claude-Code, DmitrySolana/ralph-claude-code — community mirrors.
6. ClaytonFarr/ralph-playbook — comprehensive Ralph methodology guide.
7. ralph-wiggum.ai — viral Ralph landing page.
8. *Ralph for Claude Code: Autonomous AI Development Loop with Intelligent Exit Detection* — https://www.vibesparking.com/en/blog/ai/2026-01-25-ralph-claude-code-autonomous-ai-development-loop/.
9. *2026 - The year of the Ralph Loop Agent* — DEV.to.
10. *Mastering Ralph loops transforms software engineering with LLM automation* — LinearB Blog.
11. Adjacent canon: [144-build-your-own-harness](144-build-your-own-harness.md), [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md), [160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md), [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md), [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md), [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md), [277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md), [278-agent-unit-economics-2026](278-agent-unit-economics-2026.md), [305-agent-contracts-formal-framework](305-agent-contracts-formal-framework.md), [306-stop-hook-auto-continue-pattern](306-stop-hook-auto-continue-pattern.md), [308-autonomy-loop-synthesis](308-autonomy-loop-synthesis.md).

**The one-line takeaway for harness designers.** snarktank teaches the pattern, frankbria teaches the discipline, vercel-labs teaches the shape — pick the variant that matches your stack, climb the completion-signal hierarchy to Tier 3, and wrap the whole thing in a contract envelope; the loop is the easy part, the completion signal is the load-bearing part.
