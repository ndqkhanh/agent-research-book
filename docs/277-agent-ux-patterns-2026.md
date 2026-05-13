# 277 — Agent UX Patterns 2026: plan mode, streaming, tool-call review, team panels, voice composition

**Anchors.** Anthropic Claude Code — `/plan` mode, streaming UX, tool-call permission flow, Agent Teams in-process and split-pane modes ([250-anthropic-agent-teams](250-anthropic-agent-teams.md)). Cursor — agent panel + plan mode + auto-apply UX. Cline — VSCode agent panel with explicit approval. Continue.dev — extension model. LobeHub — multi-agent collaborative UX ([205-lobehub-collaborative-teammate-platform](205-lobehub-collaborative-teammate-platform.md)). Companions: [03-plan-mode](03-plan-mode.md), [23-human-in-the-loop](23-human-in-the-loop.md), [143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md), [148-beginner-onramp-what-is-agentic-ai](148-beginner-onramp-what-is-agentic-ai.md), [137-voice-agents](137-voice-agents.md), [205-lobehub-collaborative-teammate-platform](205-lobehub-collaborative-teammate-platform.md), [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md), [250-anthropic-agent-teams](250-anthropic-agent-teams.md).

**One-line definition.** A 2026 picture of **agent UX patterns** — six canonical patterns the field converged on for users to **observe and control** running agents: (1) **plan mode** (agent emits plan; user reviews/edits/approves before execution; Claude Code's `/plan`, Cursor, Cline); (2) **token-by-token streaming** with structured-event delineation (agent thinking / tool-calling / output stages visually distinct); (3) **per-tool-call permission review** (auto-approve safe / explicit-approve risky / always-deny banned, with consequence-aware UI); (4) **team panel UX** for multi-agent (cycle teammates with Shift+Down or split-pane via tmux/iTerm2; per-teammate dedicated view); (5) **memory browse + edit + delete UX** (treat memory as visible substrate the user can audit and curate); (6) **voice + text composition** (voice-first invocation handing off to text agents for complex tasks via [137-voice-agents](137-voice-agents.md)) — combined into a coherent operator experience that turns "the agent did something" into "I watched the agent do something with explicit gates I controlled".

## Why this matters (UX is the trust contract; without it, users either over-trust or under-use)

The 2024 generation of agent UIs treated agents as black boxes — user sends prompt, agent does something, output appears. This pattern broke down at scale because it forced users to either **over-trust** (let the agent do everything, occasionally catastrophically) or **under-use** (fall back to manual work because the agent was opaque). The 2025–2026 inflection: **agent UX became its own discipline**, with patterns the field converged on for making running agents **observable**, **controllable**, and **trustworthy**.

The six patterns answer concrete operator questions. **Plan mode** answers "what is the agent about to do?" — the agent emits a plan; the user reviews/edits/approves before any action; this is the load-bearing pattern for any agent with consequential actions. **Streaming with stage delineation** answers "what's the agent doing right now?" — token-by-token streaming with explicit "thinking" / "calling tool X" / "output" stages so the user sees mental-model-aligned progress. **Per-tool-call permission review** answers "is this action safe to run?" — with three tiers (auto-approve safe, explicit-approve risky, always-deny banned) and consequence-aware UI (color-coded by impact). **Team panel UX** answers "what are my teammates doing?" — cycling or split-pane visualization of N concurrent teammates. **Memory UX** answers "what does the agent know about me?" — browsable, editable, deletable memory store. **Voice + text composition** answers "how do I invoke without typing?" — voice-first with handoff to specialists.

The strongest insight from 2025-2026 production UX: **trust is built incrementally through transparent UX, not through model capability alone**. A 70%-accurate agent with great UX outperforms a 90%-accurate agent with opaque UX in user satisfaction and adoption. Plan mode + streaming + permission review + memory UX is the trust-building combo.

## Core idea

Six agent UX patterns the 2026 production field converged on:

| Pattern | Question answered | Reference impl |
|---|---|---|
| 1 Plan mode | "What is the agent about to do?" | Claude Code `/plan`, Cursor agent, Cline plan |
| 2 Streaming + stage delineation | "What's it doing now?" | All major IDEs |
| 3 Permission review | "Is this action safe?" | Claude Code permission modes ([06-permission-modes](06-permission-modes.md)) |
| 4 Team panel | "What are my teammates doing?" | Claude Code Agent Teams ([250](250-anthropic-agent-teams.md)) |
| 5 Memory UX | "What does the agent know about me?" | Lyra `mem` commands |
| 6 Voice + text composition | "How do I invoke without typing?" | OpenAI Agents SDK + Realtime ([260](260-openai-agents-sdk-deep-dive.md)) |

Each pattern is independently adoptable; combined they form the **operator-trust contract**.

## Mechanism (step by step)

### (a) Plan mode

Three-phase: (1) agent generates plan in read-only mode (no side effects); (2) user reviews + edits + approves; (3) agent executes the approved plan.

```
User: "Refactor the auth module to use JWT instead of sessions."

Agent (plan mode):
  Plan:
  1. Read auth/sessions.py to understand current logic.
  2. Create auth/jwt.py with JWT token generation.
  3. Update auth/__init__.py to expose JWT functions.
  4. Update tests/test_auth.py with JWT tests.
  5. Run tests; fix failures.

  [Approve] [Edit] [Cancel]

User clicks [Approve].

Agent (execute mode):
  ✓ Reading auth/sessions.py
  ✓ Creating auth/jwt.py
  ⠋ Updating auth/__init__.py...
```

Implementations: Claude Code `/plan` slash command, Cursor's agent mode plan preview, Cline's "Plan vs Act" toggle, OpenAI Agents SDK with `require_approval=True`.

### (b) Streaming + stage delineation

```
[thinking] Reading the file structure to understand the codebase...
[tool] read_file(auth/sessions.py)
[result] (45 lines, abbreviated)
[thinking] The session implementation uses Flask-Session...
[tool] write_file(auth/jwt.py, ...)
[result] file written
```

Each stage is visually distinct (color, indent, icon). Token-by-token streaming inside `[thinking]`; structured events for tool calls. UX libraries: `rich` for TUI, custom Markdown renderers for web, `streamz` for React.

### (c) Per-tool-call permission review (three tiers)

| Tier | Examples | UX |
|---|---|---|
| **Auto-approve** | `read_file`, `search_dir`, `git_status`, retrieved doc | No prompt; just runs |
| **Explicit-approve** | `write_file`, `git_commit`, `pip_install`, `run_tests` | Prompt with diff/preview; yes/no |
| **Always-deny** (or escalate) | `rm`, `git_push origin main --force`, `transfer_money` | Bright-line gate ([14](../projects/polaris/docs/blocks/14-bright-line-gates.md)); requires explicit override |

UI affordances:

```
┌─────────────────────────────────────────────────────────┐
│ ✏️  Tool call: write_file                                │
│ Path: auth/jwt.py                                        │
│ Action: create new file                                  │
│ Lines: 87                                                │
│                                                          │
│ Preview:                                                 │
│   import jwt                                             │
│   from typing import Optional                            │
│   ...                                                    │
│                                                          │
│ [✓ Approve] [Approve always for write_file] [✗ Deny]   │
└─────────────────────────────────────────────────────────┘
```

"Approve always" promotes the tool to auto-approve scope for the session.

### (d) Team panel UX

For multi-agent ([250-anthropic-agent-teams](250-anthropic-agent-teams.md)):

**In-process mode** — single terminal, cycle teammates with Shift+Down:

```
┌─ research [active] ───────────────────┐
│ [thinking] Searching for recent papers│
│ on agent scaling laws...              │
│                                        │
│ Switch teammates: Shift+Down          │
└────────────────────────────────────────┘
```

**Split-pane mode** (tmux / iTerm2):

```
┌─ research ─────┬─ critic ──────┬─ writer ─────┐
│ [thinking]     │ [reading docs]│ [waiting]    │
│ Searching...   │ Reviewing the │              │
│                │ findings...   │              │
└────────────────┴───────────────┴──────────────┘
```

Click into a pane to interact directly; Shift+Down to cycle.

### (e) Memory UX

Treat memory as a first-class user-visible artifact:

```bash
$ lyra mem search "JWT preference"
3 results:
  1. (preference, 2026-04-12) "User prefers JWT over sessions" [12 citations]
  2. (fact, 2026-03-08) "User's main project uses Flask"
  3. (decision, 2026-04-15) "Agreed to use PyJWT library not python-jose"

$ lyra mem edit obs-abc123
... opens in $EDITOR ...

$ lyra mem delete obs-abc123 --confirm
Deleted observation obs-abc123 (with 12 citations).
```

Web UI: browsable timeline, search, edit, delete, export, import. Right-to-erasure ([272-agent-compliance-and-audit](272-agent-compliance-and-audit.md)) is a single button.

### (f) Voice + text composition

```
User (voice): "Research the latest papers on agent scaling laws."

Voice agent: "On it. Handing off to research agent for the deep work."
[handoff to text-mode research agent]

Research agent (background):
  ... (multi-step research, takes 5 minutes) ...

Voice agent: "Done. The research agent found 47 relevant papers. 
              Want me to summarize the top 5 now or send the full report to email?"
```

Voice-first invocation; text agents do the heavy lifting; voice agent reports back. OpenAI Realtime + Agents SDK handoffs is the canonical implementation.

### (g) Bright-line escalation UX

When a bright-line gate fires, the UX must:

```
┌─────────────────────────────────────────────────────────┐
│ ⚠️  Permission required                                  │
│                                                          │
│ The agent wants to: send_email                          │
│ To: ceo@company.com                                     │
│ Subject: Q3 financial summary                           │
│ Body: (preview, full text below)                        │
│                                                          │
│ This is a consequential action. Please review.          │
│                                                          │
│ [✓ Send] [✗ Cancel] [📝 Edit before sending]           │
└─────────────────────────────────────────────────────────┘
```

Agent durably pauses ([266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md)) waiting for response; user can resume from web/mobile/email/CLI.

### (h) Cost + budget UX

```
Session cost so far: $0.34 (budget $5.00)
████░░░░░░░░░░░░░░░░ 6.8%

Token usage:
  Input:  124,567 tokens
  Output:  18,921 tokens
  Cached: 89% hit rate
```

Real-time budget visualization; alerts on approach.

## Empirical results (table — pattern adoption May 2026)

| Pattern | Cline | Cursor | Claude Code | LobeHub | Continue |
|---|:---:|:---:|:---:|:---:|:---:|
| Plan mode | ✅ | ✅ | ✅ | ✅ | ✅ |
| Streaming + stages | ✅ | ✅ | ✅ | ✅ | ✅ |
| Permission review | ✅ | ✅ | ✅ | ⚠ | ⚠ |
| Team panel | ⚠ | ⚠ | ✅ | ✅ | ❌ |
| Memory UX | ⚠ | ⚠ | ⚠ | ✅ | ❌ |
| Voice composition | ❌ | ❌ | ❌ | ✅ | ❌ |

## Variants and ablations

- **Auto-pilot mode** (no plan / permission review). For trusted simple tasks; Claude Code's `--dangerously-skip-permissions`. High blast radius.
- **Verbose plans with reasoning.** Show *why* each step.
- **Diff preview for file writes.** Side-by-side diff in approval UI.
- **Cost-budget gates.** Pause before exceeding budget.
- **Role-color coding for teams.** Different teammate = different color.
- **Memory tier visualizations.** Show which tier (procedural / episodic / semantic) is being read.
- **Time-travel UX.** Branch from any past checkpoint ([259-langgraph-deep-dive](259-langgraph-deep-dive.md)).
- **Postmortem UX.** Replay incident traces for review.

## Failure modes

- **Permission fatigue.** Too many prompts → users click-through. Auto-approve safe tools aggressively.
- **Plan-mode bypass.** Agent emits good plan, executes wrong code. Plans must constrain execution.
- **Streaming UX latency.** Token-by-token isn't always faster perception-wise.
- **Team panel cognitive overload.** N=8 teammates → user can't follow.
- **Memory UX privacy leakage.** Memory browser shows things the user forgot they shared.
- **Voice transcription errors.** Misheard commands → wrong agent invoked.
- **Bright-line bypass via "approve always".** User over-grants then forgets.

## When to use, when not

**Adopt full UX patterns** for any user-facing agent — every user interaction with an agent benefits from plan mode + streaming + permission review at minimum. **Adopt subset** for backend / scheduled agents (no user interaction) — but still emit observability events ([264-agent-observability-stack-2026](264-agent-observability-stack-2026.md)) so admins can review.

**Skip UX patterns** for headless services where there's no user; for prototypes; for batch-processing agents. Even there: at least observability for auditing.

## Implications for harness engineering

- **Plan mode as harness primitive.** [03-plan-mode](03-plan-mode.md), [23-human-in-the-loop](23-human-in-the-loop.md).
- **Streaming + stage delineation in `harness_core/ux/`.** Common library.
- **Permission UI from Permission Bridge.** [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [06-permission-modes](06-permission-modes.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md).
- **Team panel for Agent Teams.** [250-anthropic-agent-teams](250-anthropic-agent-teams.md).
- **Memory UX over Memory module.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [Lyra Block 07](../projects/lyra/docs/blocks/07-memory-three-tier.md).
- **Voice handoff via OpenAI Realtime.** [260-openai-agents-sdk-deep-dive](260-openai-agents-sdk-deep-dive.md), [137-voice-agents](137-voice-agents.md), [48-voiceagentrag-dual-agent](48-voiceagentrag-dual-agent.md).
- **Bright-line escalation UX with durable pause.** [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md).
- **Cost budget UX.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md), [278-agent-unit-economics-2026](278-agent-unit-economics-2026.md).
- **Right-to-erasure single-button.** [272-agent-compliance-and-audit](272-agent-compliance-and-audit.md).
- **Time-travel branch UX.** [259-langgraph-deep-dive](259-langgraph-deep-dive.md).
- **LobeHub-style collaborative UX.** [205-lobehub-collaborative-teammate-platform](205-lobehub-collaborative-teammate-platform.md), [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md).
- **Trust contract via UX.** [148-beginner-onramp-what-is-agentic-ai](148-beginner-onramp-what-is-agentic-ai.md), [143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md).

**One-line takeaway.** **Agent UX in 2026 is the trust contract — six canonical patterns (plan mode + streaming with stage delineation + per-tool permission review + team panel + memory UX + voice composition) make running agents observable and controllable so users build incremental trust through transparent UX rather than blind capability; a 70%-accurate agent with great UX outperforms a 90%-accurate agent with opaque UX in adoption and satisfaction.**
