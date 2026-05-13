# 46 — Six Components of a Coding Agent (Raschka)

**Definition.** Sebastian Raschka's "Components of a Coding Agent" (April 2026) identifies **six core components** that distinguish an effective coding agent from a bare LLM chat interface: (1) Live Repo Context, (2) Prompt Shape and Cache Reuse, (3) Tool Access and Use, (4) Context Reduction, (5) Structured Session Memory, (6) Bounded Subagents. His central claim: a well-designed harness can make a non-reasoning model feel substantially stronger — the harness does much of the work people attribute to the model.

## Problem it solves

Engineers comparing coding agents often misattribute differences to model quality when in fact the harness accounts for most of the gap. Raschka's decomposition is a corrective: inspect these six components in any coding agent and you can reason about why it performs as it does. Teams also use the list as a checklist when building or evaluating their own agent.

## The six components

### 1. Live Repo Context

On session start, the harness gathers workspace metadata: git branch status, project layout, documentation files (CLAUDE.md, README, AGENTS.md). This snapshot gives the agent a map before any action. Without it, the agent's first tool call is usually an orientation search — wasted cost and token budget.

### 2. Prompt Shape and Cache Reuse

The prompt is designed as a **stable prefix + changing suffix**:

- **Stable prefix:** system instructions, workspace summary, tool schemas — rarely changes within a session.
- **Changing suffix:** recent turns, new user input.

Anthropic and OpenAI prompt caching dramatically reduce cost when the prefix is reused across turns. This is worth 50–90% cost savings for long sessions; the only thing you need to do is stop rebuilding the prompt from scratch.

### 3. Tool Access and Use

The model picks from **predefined, validated tools**, not free-form shell. The harness:

- Executes each tool.
- Validates arguments.
- Enforces permissions.
- Bounds file access to the repository.

"Tool" here is specifically a typed operation with a schema — not "bash with some examples." This is the discipline that prevents most agent escapes.

### 4. Context Reduction

Coding sessions are verbose. A single `grep -r` can dump tens of thousands of lines. The harness:

- Clips overlong tool outputs.
- Deduplicates older file reads (the agent doesn't need the same 500-line file five times).
- Compresses older transcript turns.

Same problem space as [08-context-compaction.md](08-context-compaction.md), specialized to coding.

### 5. Structured Session Memory

Two layers:

- **Full transcript** — for resumption, audit, debugging.
- **Distilled working memory** — the agent's current task focus, decisions made, open questions.

The working-memory layer is what the model attends to; the transcript is there for humans and for re-hydration. Separating them keeps the active context lean.

### 6. Bounded Subagents

Side tasks (search the codebase, run a specific test, check a condition) are delegated to subagents with tighter scope than the parent. See [02-subagent-delegation.md](02-subagent-delegation.md). Raschka's emphasis: subagents *inherit sufficient context* but don't carry the parent's full state.

## Concrete pattern

A minimal coding-agent scaffolding checklist:

```
- [ ] On session start, collect & cache: git status, file tree (depth-limited),
      README / CLAUDE.md.
- [ ] System prompt is stable across turns; only the most recent user message
      and tool results change. Cache breakpoints at prefix boundaries.
- [ ] Tools are typed: Read, Grep, Edit, Write, Bash(allowlist), plus MCP.
      No free-form command builder.
- [ ] Every tool output > N lines is clipped with a "truncated" marker;
      dedup repeat file reads.
- [ ] Maintain a working-memory block separate from transcript.
- [ ] One tool `spawn_subagent(type, prompt)` with a bounded result contract.
```

Any coding-agent harness scoring zero on any of these has room for a straightforward improvement.

## Variants & related techniques

- **Prompt caching** (Anthropic / OpenAI) — concrete Component 2 mechanism; see [24-observability-tracing.md](24-observability-tracing.md) for cost attribution.
- **Memory files** ([09-memory-files.md](09-memory-files.md)) — where Live Repo Context persists beyond a single session.
- **Subagent delegation** ([02-subagent-delegation.md](02-subagent-delegation.md)) — Component 6.
- **Context compaction** ([08-context-compaction.md](08-context-compaction.md)) — Component 4.
- **Twelve harness patterns** ([43-twelve-harness-patterns.md](43-twelve-harness-patterns.md)) / **Dive into Claude Code** ([29-dive-into-claude-code.md](29-dive-into-claude-code.md)) — overlapping but broader catalogs.

## Failure modes & anti-patterns

- **Ignoring cache reuse.** Rebuilding prompts means burning tokens on unchanged prefixes. A classic easy-50%-cost-cut.
- **Raw shell masquerading as tools.** If `Bash` is your main tool, you don't really have "tool access and use" discipline.
- **Transcript as working memory.** Mixing the two means working memory is lost to context rot. Keep them separate.
- **Unlimited subagent tokens.** Subagents inherit tight scope *including* budget; unbounded subagents just relocate the context explosion.
- **Stale live-repo context.** Snapshot at session start is fine; stale for a 3-hour session. Refresh on major events.

## When to use (and when not to)

Essentially **always**, if you're building a coding agent. The components are foundational enough that a system missing any of them will be measurably weaker than one with them. The only reason to skip: you're deliberately building a minimal Q&A interface, not a coding agent.

## References

- Sebastian Raschka, "Components of a Coding Agent" (April 2026). <https://magazine.sebastianraschka.com/p/components-of-a-coding-agent>
- Anthropic prompt caching docs. <https://docs.claude.com/en/docs/build-with-claude/prompt-caching>
- Anthropic Engineering, "Building agents with the Claude Agent SDK". <https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk>
- "Dive into Claude Code" (arXiv:2604.14228). <https://arxiv.org/abs/2604.14228>
- Generative Programmer, "12 Agentic Harness Patterns". <https://generativeprogrammer.com/p/12-agentic-harness-patterns-from>
