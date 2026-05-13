# 43 — Twelve Agentic Harness Patterns (Catalog)

**Definition.** Generative Programmer's "12 Agentic Harness Patterns from the Claude Code Leak" (April 2026) catalogues twelve recurring patterns that structure modern coding-agent harnesses, grouped by concern: memory and context, workflow and orchestration, tools and permissions, and automation. Several patterns already have their own deep-dive in this folder; the catalog's value is the *grouping* and the concrete naming.

## Problem it solves

Working engineers face an overwhelming catalog of primitives — skills, hooks, permission modes, MCP servers, subagents, todo tools, compaction pipelines. Without shared names, every team invents its own vocabulary and re-learns the same trade-offs. A pattern catalog — in the tradition of GoF Design Patterns — gives the community a shared language and a decision surface.

## The twelve patterns

### Memory and Context

1. **Persistent Instruction File Pattern.** Project-level configuration (CLAUDE.md, AGENTS.md) loaded at session start, defining commands, conventions, coding standards. See [09-memory-files.md](09-memory-files.md).
2. **Scoped Context Assembly Pattern.** Multiple instruction files at different levels (project, user, session), composed dynamically so context matches the agent's current location in the codebase.
3. **Tiered Memory Pattern.** Agent memory organized as always-loaded index + on-demand topic files + searched transcripts. Extension of [09-memory-files.md](09-memory-files.md).
4. **Dream Consolidation Pattern.** Background process periodically deduplicates, reorganizes memory during idle time — "autoDream" daemon exposed by the Claude Code leak.
5. **Progressive Context Compaction Pattern.** Multiple compression stages progressively collapse older turns while keeping recent context raw. See [08-context-compaction.md](08-context-compaction.md).

### Workflow and Orchestration

6. **Explore-Plan-Act Loop Pattern.** Three phases with increasing permissions: exploration (read-only), planning (discussion), action (full tool access). See [03-plan-mode.md](03-plan-mode.md).
7. **Context-Isolated Subagents Pattern.** Separate agents with own context windows and restricted tools. See [02-subagent-delegation.md](02-subagent-delegation.md).
8. **Fork-Join Parallelism Pattern.** Subagents work in parallel on independent repo copies (worktrees), no sequential bottleneck. Operationalized by LangChain Deep Agents async subagents ([42-langchain-deep-agents.md](42-langchain-deep-agents.md)).

### Tools and Permissions

9. **Progressive Tool Expansion Pattern.** Agent starts with a small default toolset (<20 tools) and activates more on demand, rather than exposing everything at once.
10. **Command Risk Classification Pattern.** Deterministic pre-parsing + per-tool rules classify shell command risk, auto-approving low-risk, gating high-risk. ML classifier in Claude Code adds a learned layer. See [06-permission-modes.md](06-permission-modes.md).
11. **Single-Purpose Tool Design Pattern.** Purpose-built tools (typed inputs, bounded scope) replace general shell for common ops. See [05-hooks.md](05-hooks.md) and [07-model-context-protocol.md](07-model-context-protocol.md).

### Automation

12. **Deterministic Lifecycle Hooks Pattern.** Actions that must happen consistently (formatting, validation, notify) execute at lifecycle points, not via prompt. See [05-hooks.md](05-hooks.md).

## Concrete pattern — how to use the catalog

Treat the twelve as a design checklist when building or auditing a harness:

```
For each of the 12 patterns, answer:
  - Do we have this pattern?
  - Where is it implemented (file / config / service)?
  - What failure modes does it protect against?
  - What does its absence cost us?

Missing pattern ≠ bug, but each absence should be a deliberate choice, not an oversight.
```

## Variants & related techniques

The catalog sits alongside and largely overlaps with:

- **Dive into Claude Code** ([29-dive-into-claude-code.md](29-dive-into-claude-code.md)) — academic formalization with five values and thirteen principles.
- **Harness engineering principles** ([40-harness-engineering-principles.md](40-harness-engineering-principles.md)) — BDTechTalks industry framing.
- **Four-pillars harness engineering** ([44-four-pillars-harness-engineering.md](44-four-pillars-harness-engineering.md)) — pillars-level view.
- **Components of a Coding Agent** ([46-components-of-coding-agent.md](46-components-of-coding-agent.md)) — Raschka's six-component view.

Each catalog is lossy; cross-referencing them is how you find gaps.

## Failure modes & anti-patterns

- **Pattern-cargo-culting.** Adding Dream Consolidation because the catalog lists it, without a concrete reason. Fix: tie each pattern adoption to a failure mode you actually observe.
- **Treating patterns as complete.** Twelve is not exhaustive; novel patterns emerge fast in this space.
- **Ignoring negative-space.** The absence of a pattern can be correct. Document the reason.
- **Pattern inflation.** A pattern catalog invites adding too many names; some "patterns" are really variations of one pattern.

## When to use the catalog

- When auditing an existing harness for gaps.
- When onboarding a new engineer who needs a map.
- When documenting design choices for reviewers.
- When comparing peer harnesses — patterns present/absent is a useful diff.

## References

- Generative Programmer, "12 Agentic Harness Patterns from the Claude Code Leak" (April 2026). <https://generativeprogrammer.com/p/12-agentic-harness-patterns-from>
- BDTechTalks, "The art of AI harness engineering". <https://bdtechtalks.com/2026/04/06/ai-harness-engineering-claude-code-leak/>
- "Dive into Claude Code" (arXiv:2604.14228). <https://arxiv.org/abs/2604.14228>
- Anthropic Engineering, "Building agents with the Claude Agent SDK". <https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk>
- awesome-claude-code community patterns. <https://github.com/hesreallyhim/awesome-claude-code>
