# 107 — Browser-Use: The Open-Source Browser Agent That Beat Operator

**Project.** Browser-Use — open-source browser agent. Repo: https://github.com/browser-use/browser-use. Technical Report (project-published) reports performance surpassing OpenAI Operator on WebVoyager.

**One-line definition.** Browser-Use is a **DOM-aware Python agent** that drives a real Chromium browser by extracting a structured representation of the page (clickable elements + text + bounding boxes), letting the LLM choose actions by **element index** rather than pixel coordinates — yielding lower latency and higher reliability than pure-pixel computer-use agents on most web tasks.

## Why this paper matters

[106-computer-use-agents](106-computer-use-agents.md) covered the pixel-only frontier. Browser-Use represents the **opposite design choice**: take the universally available *DOM* (and its accessibility tree) seriously, render the page to elements with stable indices, and let the LLM choose by index. This is the design that wins on web tasks today by every public benchmark — and it's open source. The repo's star growth in 2025–2026 (from a side project to one of the most-starred web-agent OSS projects) is the empirical vote.

## Problem it solves

1. **Pixel grounding is hard and slow.** Why pay vision tokens to identify a button when the DOM already tells you where it is?
2. **Web pages have semantic structure.** Forms, buttons, links have ARIA roles, accessible names, and stable identifiers — most of what an agent needs to act.
3. **OpenAI Operator and Anthropic Computer Use are closed.** Open-source web automation needs an OSS-stack equivalent.
4. **Pure Selenium/Playwright + LLM scripting** is too brittle — selectors break, waits are race-conditioned. Browser-Use packages the patterns that actually work.

## Core idea in one paragraph

For each step, Browser-Use extracts from the live Chromium page a **structured snapshot**: a numbered list of interactable elements (`[12]<button>Save</button>`, `[13]<input type="email">`), the visible text, and the URL. The LLM receives this snapshot plus the task and emits actions like `click(12)`, `input(13, "alex@example.com")`, `extract("price")`, or `done`. A small Python runtime executes via Playwright, captures the new state, and loops. Optionally, screenshots are included for visual confirmation, but element indexing — not coordinates — is the primary action mechanism.

## Mechanism (step by step)

### (a) DOM-to-element extraction

A custom JavaScript injected via Playwright walks the DOM, filters interactable elements (clickables, inputs, links, ARIA roles), assigns each a stable per-step index, and returns:

```json
[
  { "index": 12, "tag": "button", "text": "Save", "ariaLabel": "Save document", "x": 600, "y": 300 },
  { "index": 13, "tag": "input", "type": "email", "placeholder": "your@email.com" }
]
```

The LLM sees this list as a compact text block — an order of magnitude fewer tokens than a screenshot description.

### (b) Action vocabulary

```python
ACTIONS = [
    "click(index)",
    "input(index, text)",
    "scroll(direction)",
    "go_to_url(url)",
    "go_back()",
    "extract(query)",      # use LLM to pull info from current page
    "done(success: bool, text: str)",
]
```

The vocabulary is intentionally narrow — broad enough for the long tail of web tasks, narrow enough that policies don't have to learn an unbounded API.

### (c) The orchestration loop

```python
agent = Agent(task="Find the cheapest flight from NYC to Tokyo next week", llm=llm)
result = agent.run(max_steps=30)
```

Internally: `Browser.snapshot() → LLM.plan(task, snapshot, history) → Browser.act(action) → loop`.

### (d) Multi-tab and downloads

Browser-Use models multi-tab as a tab list, with `switch_tab(i)` and `new_tab(url)` actions. Downloads are saved to a configured directory and surfaced in the snapshot as completed.

### (e) Vision fallback

When element extraction yields nothing useful (canvas-rendered apps, video players, custom pixel UIs), the runtime falls back to screenshot + bounding-box reasoning — the *same* design as [106-computer-use-agents](106-computer-use-agents.md). Hybrid by default.

### (f) Memory / state

A short scrolling history of `(snapshot, plan, action, observation)` is kept in context. For long sessions, summarization compresses older steps ([08-context-compaction](08-context-compaction.md) pattern).

## Empirical results

Per the project's technical report:

- **WebVoyager**: Browser-Use **surpasses** OpenAI Operator (Operator at 87%; Browser-Use technical report claims higher on the same task set with comparable LLMs).
- **Latency**: 1–3s per step typical (vs 5–10s for pure-pixel agents that pay screenshot+vision per step).
- **Cost per task**: substantially lower than computer-use agents because the input context is text-dominant rather than image-dominant.

Important caveat: the TR is project-published, not peer-reviewed; numbers should be replicated against your own task suite before relying on them.

## Variants and ablations

- **DOM-only vs DOM + screenshot:** DOM-only suffices for ~85% of web tasks; screenshot fallback adds ~10 percentage points on canvas-heavy or visually-encoded tasks.
- **Element index vs CSS selector:** index-based is more stable (selectors break on refactor); the cost is needing the snapshot up front each step.
- **Backbone choice:** GPT-4o, Claude 3.5/4.x Sonnet, and DeepSeek-R1-distills all work; reasoning models reduce step count on multi-stage tasks.
- **`extract` action:** the trick that lets the agent return structured data without writing parsers. The same LLM does the extraction in-place.

## Failure modes and limitations

1. **Heavy-JS and canvas-rendered apps.** Figma, Google Maps, online IDEs — DOM is sparse; falls back to vision and slows down.
2. **Authentication walls.** CAPTCHAs, MFA, bot detection — hostile environments. HITL is required for login flows ([23-human-in-the-loop](23-human-in-the-loop.md)).
3. **Index instability across DOM mutations.** Single-page apps that re-render frequently can shift indices mid-step; the runtime re-snapshots after each action to mitigate but races persist.
4. **Side effects.** `click(submit)` sends real form submissions. Sandboxing or test environments are mandatory for destructive flows.
5. **Prompt injection via page content** — the same as in [106-computer-use-agents](106-computer-use-agents.md), made worse by the fact that DOM text is concatenated directly into the LLM prompt. Any page can attempt to override the agent's task. Defenses: separate page-text from user-task spans, content-source tags, refusal training.

## When to use, when not

**Use Browser-Use when** the task is web-only, you want OSS, and DOM is a viable interface for the target sites. It's the practical default for web automation, scraping, form-filling, and price-comparison agents.

**Don't reach for it** when the target is desktop software (use [106-computer-use-agents](106-computer-use-agents.md)), when the site is heavily anti-bot (consent-driven manual workflow + recording is more honest), or when you need cross-app workflows (browser is too narrow).

## Implications for harness engineering

- **Element-index actions are the right default for web.** Don't reach for pixel coordinates unless DOM truly fails.
- **The snapshot extractor is harness-critical infra.** Bugs in the JS injection cause the policy to "see" the page wrong; treat it as a first-class component with tests.
- **Action vocabulary must be small and stable.** Add primitives sparingly; every new action is a new failure mode.
- **Page-text isolation matters for security.** Treat DOM-extracted text as untrusted input ([22-guardrails-prompt-injection](22-guardrails-prompt-injection.md)) — wrap it in a clearly delineated context block.
- **The `extract` pattern is reusable.** Calling the LLM as a structured-data extractor on observed content is a generally useful action for any agent that interfaces with text streams.

## Connections to other work in this corpus

- **[106-computer-use-agents](106-computer-use-agents.md):** the closed/proprietary alternatives; Browser-Use is the OSS web-only equivalent.
- **[34-clawbench-live-web-tasks](34-clawbench-live-web-tasks.md):** the benchmark genre Browser-Use is built for.
- **[13-react](13-react.md):** Browser-Use's loop is ReAct over a structured page representation.
- **[22-guardrails-prompt-injection](22-guardrails-prompt-injection.md):** page text injection is a real attack vector.
- **[42-langchain-deep-agents](42-langchain-deep-agents.md), [110-langgraph](110-langgraph.md):** Browser-Use is often used as a *tool* inside larger orchestrators.

## Key takeaways

1. **DOM-aware action by index** beats pixel coordinates on most web tasks.
2. **Element snapshots + small action vocabulary + Playwright** is the production stack of choice for OSS browser agents.
3. **Vision fallback** preserves universality where DOM is sparse.
4. **The `extract` action** is a quietly powerful pattern — LLM-as-parser at runtime.
5. **Page text is untrusted input** — treat it as such or get pwned.

## References

- Browser-Use repository: https://github.com/browser-use/browser-use
- Browser-Use Technical Report (linked from the repo).
- Comparison piece: Helicone — *The Best Web Agents: Computer Use vs Operator vs Browser Use* — https://www.helicone.ai/blog/browser-use-vs-computer-use-vs-operator
- WebVoyager benchmark: He et al. (2024), arXiv:2401.13919.
