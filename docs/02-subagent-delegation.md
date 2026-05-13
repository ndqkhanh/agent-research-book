# 02 — Subagent Delegation

**Definition.** Subagent delegation is the pattern where a lead ("orchestrator") agent spawns short-lived worker agents, each with its own isolated context window and task, and merges their distilled outputs back into its own reasoning. It is the most consequential architectural choice in modern harnesses — and the most contested.

## Problem it solves

A single-context agent has to carry every file it touched, every search result, every test output, through the whole session. By step 40 the model is attending to 150k tokens of noise and missing the instruction from step 3. Subagents solve this by letting the orchestrator say *"go find how authentication is wired up"* and get back a 300-token summary instead of 30,000 tokens of raw `grep` output. Context stays tight; parallelism becomes possible.

## Mechanism

The orchestrator-worker dance has four phases:

1. **Decomposition.** The orchestrator identifies a sub-task that is *self-contained* (doesn't require mid-task interaction with the orchestrator) and *context-isolatable* (the worker doesn't need to know everything the orchestrator knows).
2. **Dispatch.** The orchestrator calls a `Task` / `spawn_subagent` tool with a prompt, a subagent type (explore / plan / code / eval), and a scope. The harness spins up a fresh agent loop with its own system prompt, tools, and permission posture.
3. **Execution.** The subagent runs its own loop, often with narrower tools (e.g., read-only) and a tighter step budget. It cannot see the orchestrator's transcript; it only sees what its prompt carries.
4. **Return.** When the subagent emits a final answer, the harness sends that answer — and only that answer — back to the orchestrator as a tool result.

Claude Code ships with built-in subagent types (general-purpose, Explore, Plan, code-reviewer, etc.) and lets users define their own via `.claude/agents/*.md` files. Anthropic's multi-agent research system uses an Opus 4 lead agent with Sonnet 4 subagents and reports a ~90% performance improvement over the single-agent baseline on internal research tasks.

## Concrete pattern

Subagent definition file (simplified):

```markdown
---
name: Explore
description: Fast agent specialized for exploring codebases
tools: [Glob, Grep, Read, WebFetch]
model: sonnet
---
You search the repo and return a distilled map of relevant files
and patterns. Return under 500 words. Cite file:line for every claim.
```

Orchestrator-side dispatch:

```python
finding = call_tool("Agent", {
    "subagent_type": "Explore",
    "description": "Locate auth middleware",
    "prompt": "Find all middleware involved in session token validation. "
              "Return a call graph with file:line citations. Under 400 words."
})
# Orchestrator only sees `finding.text` — not the 50 reads the subagent did.
```

## Variants & related techniques

- **Parallel fan-out.** The orchestrator dispatches N independent subagents in one step. The harness executes them concurrently and returns results together. Works well for research and exploration.
- **Pipelined delegation.** Planner → Generator → Evaluator three-agent pipeline — see [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md).
- **Hierarchical delegation.** Sub-subagents — rare, because each level of indirection blurs responsibility. Usually capped at two levels.
- **Tool-wrapped agents.** An agent exposed to another agent as if it were a tool. Pattern popularized by AutoGen and echoed in the Claude Agent SDK.

## Failure modes & anti-patterns

Cognition's widely-cited 2025 post "Don't Build Multi-Agents" is a corrective to uncritical fan-out: when subagents do **implicit design work in parallel**, they make conflicting decisions (one builds a Mario background, one builds a bird that would belong in Flappy Bird) and the orchestrator has no way to reconcile them. The rule of thumb that emerged:

- **Decompose only where subtasks are genuinely independent.** Research is. UI design is not.
- **Share context, not tasks.** Cognition argues that the first-order fix is to pass the full conversation trace to each subagent — losing the token savings but keeping coherence. This is the opposite of the Anthropic pattern; which is right depends on whether subtasks are decision-coupled or information-parallel.
- **Never let subagents mutate shared state without locks.** Two coders editing the same file is a merge conflict at best, lost work at worst.
- **Beware the "just spawn more agents" reflex.** Every extra agent adds latency and a failure mode. If a single agent with better prompts can do the job, prefer that.

Other common pitfalls:

- **Opaque returns.** Subagent summarizes away the information the orchestrator needed. Fix: structured return schemas ("return a JSON with `files`, `patterns`, `open_questions`").
- **Unbounded subagent loops.** Subagent's own step budget must be smaller than the orchestrator's remaining budget.
- **Trust erosion.** Orchestrator treats subagent output as ground truth. Cite-or-die policy (subagent must quote file:line) forces verifiability.

## When to use (and when not to)

**Use** subagents when:

- The subtask would otherwise dump >5k tokens of observations into the main context.
- The subtask is read-only or has a well-scoped write surface.
- You have multiple independent subtasks that can run in parallel.
- You want a different model/permission/tool posture for the subtask (e.g., a cheap model for search, a powerful one for synthesis).

**Don't use** subagents when:

- The subtask is a series of tightly-coupled edits to the same file. Fan-out there creates merges the orchestrator can't resolve.
- The orchestrator needs to see every detail of the subtask. Delegation is summarization; if you don't want summarization, don't delegate.
- The task is short enough that the subagent's setup cost (fresh context, new model call) exceeds its savings.

## References

- Anthropic Engineering, "How we built our multi-agent research system" — <https://www.anthropic.com/engineering/multi-agent-research-system>
- Anthropic Engineering, "Effective harnesses for long-running agents" — <https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents>
- Cognition AI, "Don't Build Multi-Agents" — <https://cognition.ai/blog/dont-build-multi-agents>
- Claude Code docs, Subagents — <https://docs.claude.com/en/docs/claude-code/sub-agents>
- AutoGen (Microsoft), arXiv:2308.08155 — <https://arxiv.org/abs/2308.08155>
