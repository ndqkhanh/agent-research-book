# 143 — UX Design for Agentic Systems: Asynchronous Interfaces, Progress Disclosure, Intervention Affordances

**Sources.** Albada, *Building Applications with AI Agents*, Chapter 3 (User Experience Design for Agentic Systems); the practitioner UX literature (Don Norman, Bruce Tognazzini); plus the emerging UX patterns from Claude Code, Cursor, ChatGPT Pro, Devin, Replit Agent.

**One-line definition.** UX for agentic systems is fundamentally different from UX for chatbots — agents *act over time*, *fail visibly*, and require *user oversight* — so the design vocabulary has to expand beyond "send message, receive reply" to include *progress disclosure*, *intervention affordances*, *trust calibration*, *cost transparency*, and *asynchronous notification* — and the systems that get this right (Claude Code's planner streaming, Cursor's diff preview, Devin's task dashboard) are differentiated from those that don't (anything that makes the user wait without context).

## Why this matters

Chatbot UX is mature: send message, see reply, react. Agent UX is younger and harder. Agents take seconds to minutes to complete tasks; they make multi-step plans; they call tools; they sometimes fail mid-way; they sometimes need user approval. Without UX patterns specific to these realities, users either over-trust (the agent does something destructive) or under-trust (users won't delegate anything meaningful).

For agent builders, UX is not a nice-to-have. It is the user-facing manifestation of every harness decision in this book. The agent loop is invisible without progress disclosure. Permission modes are pointless without intervention affordances. Trustworthy generation is wasted without citation surfaces. UX is where the harness meets the user.

This chapter is the UX vocabulary specific to agents — the new patterns the chatbot vocabulary doesn't include, the failure modes specific to agent UX, and the trust-calibration techniques that distinguish products users actually delegate to from those they only experiment with.

## Problem it solves

Six UX failures specific to agentic systems:

1. **Unexplained pauses.** Agent thinks for 30 seconds; user sees nothing; abandons.
2. **Surprising actions.** Agent edited 12 files; user expected 1; confused or angry.
3. **Lost progress.** Agent crashed mid-task; user has to start over.
4. **No off-ramp.** User wants to stop; no clear stop button.
5. **Cost surprise.** User runs an agent; bill is 10× expected; no warning.
6. **Trust collapse on first error.** Agent makes one mistake; user permanently loses trust.

Each is structurally addressable through UX patterns.

## Core idea in one paragraph

Agentic UX has six new primitives the chatbot vocabulary doesn't include. **Progress disclosure**: stream the agent's plan and current step so users always know what's happening (Claude Code's "I'll do X, Y, Z..." streaming). **Intervention affordances**: visible "stop", "approve", "modify" controls available throughout the task, not just at the end. **Trust calibration**: confidence indicators, citation surfaces, and explicit refusal language so users know when to verify. **Cost transparency**: live cost meter, projected total, budget cap. **Asynchronous notification**: long-running tasks notify on completion (push, email, in-app) so users don't have to babysit. **Failure recovery**: when agents fail, surface the failure, the partial progress, and recovery options. Layered, these patterns turn a black-box agent into a delegate users feel they can supervise without being chained to. The systems that ship these well differentiate; those that don't lose to chat-bot reduction.

## Mechanism (step by step)

### 1. Progress disclosure — the foundational pattern

The agent streams its plan and execution to the user:

```text
[user] "Refactor this module to use the new auth API"
[agent]
  📋 Plan:
    1. Read auth-api docs
    2. Find all callers of the old API
    3. Generate refactored code for each
    4. Run tests
    5. Commit changes
  🔄 Step 1: Reading auth-api docs... done
  🔄 Step 2: Finding callers... 12 files identified
  🔄 Step 3: Refactoring 12 files... 5/12 complete
  ...
```

The user sees:
- The plan up front.
- Which step is current.
- Output of each step.
- Estimated remaining time / cost.

Without this, the user is staring at a spinner.

### 2. Intervention affordances

Visible controls always available:
- **Stop**: cancel the task immediately. Critical for runaway agents.
- **Approve**: required for high-stakes actions ([06-permission-modes](06-permission-modes.md), [23-human-in-the-loop](23-human-in-the-loop.md)).
- **Modify**: change the plan mid-execution.
- **Skip**: skip the current step; continue.
- **Rollback**: undo what's been done.

Implementation: persistent toolbar in the agent UI. Always visible, always actionable.

### 3. Trust calibration

Make the agent's confidence visible:
- **Confidence indicator**: "high confidence" / "verify this" badges.
- **Citation surface**: every claim clickable to source.
- **Refusal language**: "I'm not sure about X — would you like me to look more carefully?"

Users learn when to trust and when to verify. The product that does this well builds calibrated trust over time.

### 4. Cost transparency

Live cost display:
- **Per-task estimate** before start: "Estimated cost: $0.30."
- **Live meter** during: "$0.18 spent so far."
- **Budget cap** ("stop if over $1.00").
- **Cost summary** after completion.

For high-cost tasks, the agent confirms before exceeding a threshold. Aligns with [123-robustness-fault-tolerance](123-robustness-fault-tolerance.md)'s cost circuit breaker.

### 5. Asynchronous notification

Long-running agents notify on completion:
- **In-app**: badge or banner.
- **Push**: mobile notification.
- **Email**: for very long tasks.
- **Webhook**: for programmatic integration.

The user doesn't have to babysit. Devin, Replit Agent, ChatGPT Pro all do this.

### 6. Failure recovery

When agents fail:
- **Surface the failure**: clear, non-technical explanation.
- **Show partial progress**: what got done before failure.
- **Recovery options**: "retry from step 3", "modify plan", "abandon".
- **Lessons learned**: feed into [107-memento-cbr-memory](107-memento-cbr-memory.md) memory.

Failure surfacing builds trust. Hidden failures destroy it.

### 7. Trajectory replay

For asynchronous agents, the user often wasn't present during execution. Replay UI:
- **Step-by-step playback**: scrub through the agent's actions.
- **Filter** by step type (tool call, reasoning, error).
- **Drill into details** per step.

Used for review, debugging, audit. See [122-explainability-compliance](122-explainability-compliance.md) for the audit-side requirement.

### 8. Citation surfaces (UX side of [135-trustworthy-generation](135-trustworthy-generation.md))

Per claim:
- Inline footnote markers.
- Click to expand: source paragraph, document, timestamp (for video/audio).
- Visual distinction between cited (high-trust) and uncited (use caution) claims.

### 9. Multi-modal UX

For agents handling images, audio, video:
- **Image attachments** with captions.
- **Audio playback** with transcript scroll-sync.
- **Video** with chapter markers.
- **PDF viewer** with citation highlighting.

See [136-multimodal-rag](136-multimodal-rag.md), [137-voice-agents](137-voice-agents.md), [139-ocr-document-agents](139-ocr-document-agents.md).

### 10. Permission UX

For permission-mode-driven agents ([06-permission-modes](06-permission-modes.md)):
- **Mode indicator**: which mode is active (plan / acceptEdits / bypass).
- **Clear approval flow**: when high-stakes action requires approval.
- **Mode-switching UI**: easily change mode mid-task.
- **Action preview**: show what's about to happen before doing.

This is the Cursor/Claude Code model: diff preview before edit.

## Empirical anchors

- **Streaming plans** measurably increase perceived agent quality vs unstreamed.
- **Stop buttons used** ~5–15% of the time in production agent UIs; agents that don't have them are abandoned more.
- **Citation surfaces** correlate with user trust in 5–20% improved acceptance rate.
- **Cost transparency** prevents most "shock bill" abandonment events.
- **Async notification** distinguishes products users keep open from those they don't.
- **Adoption**: the leading agent products (Claude Code, Cursor, Devin, Replit Agent) implement all these patterns; chat-bot-style products are differentiating themselves out of the agent space.

## Variants and counter-arguments addressed

- **"UX is polish; it doesn't matter."** It's the surface of every harness decision; users see only the UX.
- **"Streaming is technically hard."** Mature solutions exist; difficult in 2023, standard in 2026.
- **"Users want simplicity."** Yes — UX must distill complexity, not eliminate it. Hidden agents fail.
- **"Show only the final answer."** Loses trust; users want to see how the agent arrived.
- **"Cost transparency scares users."** Cost surprises scare them more.

## Failure modes and limitations

1. **Information overload.** Too much streamed; user can't follow.
2. **Spurious progress signals.** "Thinking..." with no actual progress; users learn it's meaningless.
3. **Stop button latency.** User clicks stop; agent keeps running; trust eroded.
4. **Approval-flow friction.** Every action requires approval; user tunes out and approves blindly.
5. **Cost meter inaccuracy.** Estimates wrong; actual costs surprise.
6. **Notification overload.** Too many notifications; users disable them.
7. **Citation misuse.** Citations to plausible but wrong sources; verifier needed ([135-trustworthy-generation](135-trustworthy-generation.md)).
8. **Trust calibration mismatch.** UI confidence doesn't match actual reliability.

## When to use, when not

**Use the full UX vocabulary for** any production agent users delegate to.

**Use a subset for** prototypes; minimum is progress disclosure + stop button.

**Skip for** purely API-only agents (no UI); but the same primitives appear as logged events.

**Iterate based on usage telemetry**: which controls users actually use; where they get stuck.

## Implications for harness engineering

- **Streaming everywhere.** Plan generation, step execution, output. See [110-transformer-llm-architecture-for-agent-builders](110-transformer-llm-architecture-for-agent-builders.md) for the technical streaming side.
- **Persistent toolbar with controls.** Stop, approve, modify, rollback. Always visible.
- **Trust signals in every response.** Confidence, citations, refusals.
- **Cost meter as a primitive.** [125-system-level-production-patterns](125-system-level-production-patterns.md) cost-tracking surfaced to user.
- **Async notification infrastructure.** Push, email, webhook.
- **Trajectory replay UI.** For review, debug, audit.
- **Permission flows as UI primitive.** [06-permission-modes](06-permission-modes.md), [23-human-in-the-loop](23-human-in-the-loop.md).
- **Telemetry-driven iteration.** Track which UX elements work, refine.
- **Mobile + desktop parity.** Async notifications mobile-first; deep workflows desktop.

The one-sentence takeaway: **UX for agentic systems is a new vocabulary — progress disclosure, intervention affordances, trust calibration, cost transparency, async notification, failure recovery — that turns black-box agents into delegates users actually delegate to.**

## See also

- [01-agent-loop-architecture](01-agent-loop-architecture.md) — the loop UX makes visible.
- [06-permission-modes](06-permission-modes.md), [23-human-in-the-loop](23-human-in-the-loop.md) — permission UX.
- [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) — confidence signals come from verifiers.
- [122-explainability-compliance](122-explainability-compliance.md) — audit replay UX.
- [135-trustworthy-generation](135-trustworthy-generation.md) — citation UX.
- [62-everything-claude-code](62-everything-claude-code.md), [29-dive-into-claude-code](29-dive-into-claude-code.md) — Claude Code UX patterns.
- [137-voice-agents](137-voice-agents.md) — voice-specific UX.
- [148-beginner-onramp-what-is-agentic-ai](148-beginner-onramp-what-is-agentic-ai.md) — UX for non-technical users.
