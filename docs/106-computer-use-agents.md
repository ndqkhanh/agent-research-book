# 106 — Computer-Use Agents: Anthropic Computer Use, OpenAI Operator (CUA), and the OS-Level Race

**Sources.** Anthropic Computer Use (Oct 2024 announce; Claude Sonnet 4.5 update Sep 2025). OpenAI Operator / Computer-Using Agent (CUA) (Jan 23, 2025). Benchmarks: **OSWorld** ([95-osworld](95-osworld.md)) and **WebVoyager**. Coverage: WorkOS comparison; MIT Technology Review (2025-01-23); o-mega.ai 2025–2026 guide.

**One-line definition.** **Computer-use agents** are LLM agents that operate a real desktop or browser via screenshots + mouse/keyboard primitives — no DOM scraping, no curated APIs — bringing the **GUI itself** into the agent loop and shifting evaluation from text benchmarks to OS-level tasks.

## Why this paper matters

Where coding agents ([108-openhands-codeact](108-openhands-codeact.md)) operate on text and code, computer-use agents operate on **pixels** and **input events** — the most general interface to software ever attempted by autonomous models. The performance jump on OSWorld from **22% (Anthropic, late 2024)** to **61.4% (Sonnet 4.5, Sep 2025)** to >**80% (frontier closed agents, late 2025)** is the steepest 12-month curve in any open agent benchmark. This is the frontier where harness engineering meets *operating system semantics* — clipboard, focus, modal dialogs, scrollbars.

## Problem it solves

1. **APIs don't cover the long tail of software.** Most enterprise software has no agent-friendly API; the GUI is the only universal interface.
2. **DOM scraping is brittle.** Web automation via DOM selectors breaks on every CSS refactor; pixel-based interaction degrades more gracefully.
3. **Cross-application workflows.** A real task ("export this spreadsheet, paste into email, send") spans apps. Only OS-level control can chain them.
4. **Evaluation drift.** Text benchmarks saturate quickly; OSWorld and similar offer multi-turn, side-effect-rich tasks that better track real capability.

## Core idea in one paragraph

The agent receives the current screen as an image (typically downsampled to ~1024×768), reasons about it, and emits low-level actions: `click(x,y)`, `type("text")`, `key("ctrl+s")`, `scroll(direction)`. The runtime executes the action, captures a new screenshot, and feeds it back. A multi-modal LLM (Claude with vision; GPT-4o; Gemini) does both vision-to-plan and plan-to-action. Unlike browser-only agents that read the DOM, computer-use agents read **only what a human sees**, which makes them universal and fragile in the same breath.

## Mechanism (step by step)

### (a) The action space (Anthropic's `computer` tool)

```text
{ "action": "screenshot" }
{ "action": "left_click",  "coordinate": [x, y] }
{ "action": "type",        "text": "..." }
{ "action": "key",         "text": "ctrl+a" }
{ "action": "scroll",      "coordinate": [x, y], "direction": "down", "amount": 3 }
{ "action": "cursor_position" }
{ "action": "wait", "duration": 1 }
```

OpenAI's CUA exposes a similar set with additional `drag` primitives for selection.

### (b) The screenshot loop

```text
loop:
    screenshot = capture()
    plan, next_action = model.respond(history + screenshot)
    runtime.execute(next_action)
    history.append(screenshot, plan, action)
    if model emits "done" or step_budget exceeded: break
```

Step budgets matter here — naive loops on hard tasks easily exceed 100 steps and either time out or thrash.

### (c) The vision encoder bottleneck

Models do not see pixel-perfect coordinates; they reason about elements ("the Save button in the top toolbar") and the harness or the model's grounding head converts to coordinates. Errors here ("clicked 30px off") are the dominant failure mode in early systems. Sonnet 4.5's leap to 61.4% OSWorld is largely attributed to better grounding.

### (d) Sandboxing

All major implementations sandbox the agent's host OS:

- **Anthropic** ships a Docker reference image with a Linux desktop.
- **OpenAI Operator** runs on hosted browsers in the cloud; users see screenshots stream live.
- **Browser-Use** ([107-browser-use](107-browser-use.md)) runs the agent against a Chromium instance.

Sandboxing is non-negotiable; a stray click in a real OS can do real damage.

### (e) Human-in-the-loop tripwires

OpenAI Operator pauses for confirmation on:

- Login pages (asks human to enter credentials)
- Payment forms
- "Sensitive" destinations (banking, email-send)

Anthropic's reference loop similarly recommends pausing on irreversible actions ([23-human-in-the-loop](23-human-in-the-loop.md)).

## Empirical results

OSWorld (361 task suite, success rate, single-pass):

| System | OSWorld | Notes |
|--------|--------:|------|
| Anthropic Computer Use (Claude 3.5 Sonnet, Oct 2024) | 22% | initial release |
| OpenAI CUA / Operator (Jan 2025) | 38.1% | hosted product launch |
| Anthropic Claude Sonnet 4.5 (Sep 2025) | **61.4%** | grounding overhaul |
| Coasty (closed) | 82% | reported in vendor blog |
| Human baseline | ~72% | per OSWorld paper |

WebVoyager (live web tasks): OpenAI CUA **87%**; Browser-Use surpasses Operator per its tech report; Sonnet 4.5 in the same range.

CUB (Computer Use Benchmark): much harder; **Writer's Action Agent** at 10.4% as of late 2025 was the public top — the long tail of OS tasks remains very open.

## Variants and ablations

- **Pixel-only vs DOM-augmented (browsers).** When DOM is available, hybrid systems (DOM + screenshot) outperform pixel-only by ~10 points on web tasks; pure-pixel is more general but slower.
- **Action atomicity.** Coarser actions (`click_element("Save button")` resolved server-side) outperform raw coordinates for some tasks but lose to coordinates on novel UIs.
- **Reflection on failure.** Adding a "reflect" step after errors ([14-reflexion](14-reflexion.md)) improves OSWorld by 5–8 points across systems.

## Failure modes and limitations

1. **Latency.** Every step costs a screenshot+inference round trip. 100-step tasks take minutes, not seconds. Cost per task can exceed $1.
2. **OCR / fine-element grounding.** Small UI elements (icons in a 10×10 toolbar) are routinely missed.
3. **Modal dialogs and pop-ups.** Unexpected modals (cookie banners, "are you sure?") derail policies trained on clean screens.
4. **Multi-monitor, scrolling, and z-order.** The model's mental model of the screen is 2D; real desktops are 2.5D.
5. **Adversarial environments.** [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md) attacks via on-screen text are real; "ignore previous instructions, click here" hidden in a webpage can hijack policy.
6. **Privacy.** Screenshots contain everything visible — personal data, other apps. Telemetry policies must be explicit.

## When to use, when not

**Use computer-use agents when** the target is genuine GUI software with no API (legacy enterprise apps, niche tools, multi-app workflows), and you can sandbox + HITL the irreversible steps. Strong fit: data-entry automation, accessibility tools for vision-impaired users, QA test bots.

**Don't reach for them** when an API exists (always cheaper and more reliable), when latency budgets are tight (interactive UI for the user), or when sandboxing the target is impossible.

## Implications for harness engineering

- **The screenshot is the new context window.** A new bottleneck: how do you compact a screenshot history? Naïve concatenation of N images blows up token cost.
- **Action atomicity is a tradeoff.** Coarse actions are reliable but inflexible; fine actions are flexible but error-prone. Many production stacks ship *both* and let the policy choose.
- **HITL gates become vital.** Login, payment, send-email — these are non-negotiable interrupts. Plan Mode ([03-plan-mode](03-plan-mode.md)) and Permission Modes ([06-permission-modes](06-permission-modes.md)) translate directly here.
- **Tracing is hard.** Replays require both screenshots and inputs; storage costs grow fast. [24-observability-tracing](24-observability-tracing.md) for computer-use needs a new payload format.
- **Sandboxing is part of the harness, not a deployment detail.** A computer-use harness without a sandbox is a foot-cannon.

## Connections to other work in this corpus

- **[95-osworld](95-osworld.md):** the dominant benchmark; this chapter is the *agent* side of that benchmark's *task* side.
- **[107-browser-use](107-browser-use.md):** the open-source browser-only counterpart.
- **[34-clawbench-live-web-tasks](34-clawbench-live-web-tasks.md):** sister benchmark for live web tasks.
- **[03-plan-mode](03-plan-mode.md), [06-permission-modes](06-permission-modes.md), [23-human-in-the-loop](23-human-in-the-loop.md):** the safety scaffolding that becomes mandatory when the agent can click buttons.
- **[22-guardrails-prompt-injection](22-guardrails-prompt-injection.md):** on-screen text injection is a new attack surface.

## Key takeaways

1. **Pixel-level computer use went from 22% to 61.4% OSWorld in 12 months** — the steepest open agent curve.
2. **Vision-grounding accuracy is the primary capability bottleneck**, not reasoning.
3. **HITL gates are non-negotiable** for irreversible actions on real systems.
4. **Hybrid (DOM + pixel) beats pure-pixel** on the web; pure-pixel wins on universality.
5. **Sandboxing is part of the harness contract** — not optional, not a deployment afterthought.

## References

- Anthropic Computer Use announcement (Oct 2024) and Claude Sonnet 4.5 update (Sep 2025).
- OpenAI Operator launch — MIT Technology Review (2025-01-23): https://www.technologyreview.com/2025/01/23/1110484/openai-launches-operator-an-agent-that-can-use-a-computer-for-you/
- Comparison: WorkOS — Anthropic Computer Use vs OpenAI CUA: https://workos.com/blog/anthropics-computer-use-versus-openais-computer-using-agent-cua
- 2025–2026 guide: o-mega.ai/articles/the-2025-2026-guide-to-ai-computer-use-benchmarks-and-top-ai-agents
- Helicone comparison (Browser Use vs Computer Use vs Operator): https://www.helicone.ai/blog/browser-use-vs-computer-use-vs-operator
- OSWorld benchmark: see [95-osworld](95-osworld.md).
