# 12 — Todo / Scratchpad State

**Definition.** A todo (or scratchpad) is a structured, externalized record of the agent's own plan and progress, maintained as a tool-writable artifact the agent updates between actions. It is memory the agent keeps about *itself mid-task* — not about the user, not about the repo, not about the world.

## Problem it solves

LLMs lose the thread. Somewhere between step 10 and step 30 of a multi-tool task, the original request, the intended plan, and the current position blur. The model attends to whatever's freshest in its context and drops subtasks on the floor. A scratchpad anchors the plan outside the model's working memory: the agent re-reads it before each move, updates it after each completion, and crucially the *user* can see what the agent believes it's doing.

Related: context rot (see [08-context-compaction.md](08-context-compaction.md)). Even when the context fits, U-shaped attention means mid-context material is under-weighted. Items on a scratchpad regain salience when the agent re-reads them at the top of a turn.

## Mechanism

Implemented as a tool (e.g., `TodoWrite`) that owns a small JSON list of tasks, each with content, active-form phrasing, and status (`pending` | `in_progress` | `completed`).

1. On first relevant user request, the agent calls `TodoWrite` with the decomposed plan.
2. Before starting a task, the agent marks it `in_progress` — exactly one at a time.
3. On completion, the agent marks it `completed` immediately (not batched).
4. New tasks discovered mid-work are appended, not piled silently into the current task.
5. Obsolete tasks are removed, not left as stale `pending`.

The harness usually renders the todo list prominently in the UI, so the user sees real-time progress. This dual-use — private agent memory *and* public status board — is what makes scratchpads uniquely valuable.

Two disciplines matter:

- **Exactly one in-progress.** Prevents the agent from pretending to parallelize when it actually context-switches.
- **Complete immediately.** Batching completions erodes the signal — the user sees a static list for minutes, then a flood of "done"s.

## Concrete pattern

Initial plan created on receipt of a multi-step request:

```json
[
  {"content": "Find all usages of getCwd", "activeForm": "Finding all usages of getCwd", "status": "in_progress"},
  {"content": "Rename to getCurrentWorkingDirectory in each file", "activeForm": "Renaming to getCurrentWorkingDirectory in each file", "status": "pending"},
  {"content": "Run tests", "activeForm": "Running tests", "status": "pending"},
  {"content": "Run build", "activeForm": "Running build", "status": "pending"}
]
```

After the Grep lands, the agent updates:

```json
[
  {"content": "Find all usages of getCwd", "status": "completed", ...},
  {"content": "Rename in src/fs.ts:18", "status": "in_progress", ...},
  {"content": "Rename in src/fs.ts:42", "status": "pending", ...},
  {"content": "Rename in src/cli.ts:7", "status": "pending", ...},
  {"content": "Run tests", "status": "pending", ...},
  {"content": "Run build", "status": "pending", ...}
]
```

The single "rename" task expanded into specific, file-scoped ones after the search gave concrete targets. That pattern — plan coarse, refine on evidence — is the default.

## Variants & related techniques

- **Plan files** ([03-plan-mode.md](03-plan-mode.md)) are a heavier-weight sibling: written for human review, often once per task. Todos are lightweight, written by the agent for itself, and updated frequently.
- **Agent memory files** ([09-memory-files.md](09-memory-files.md)) persist cross-session; todos are ephemeral per task.
- **Reflexion** ([14-reflexion.md](14-reflexion.md)) writes retrospective reflections, not active-plan items.
- **Voyager-style skill libraries** ([19-voyager-skill-libraries.md](19-voyager-skill-libraries.md)) use a similar idea — track discovered skills — but persisted across episodes.
- **Chain-of-thought in prompt.** An older, less effective form of scratchpad: the reasoning lives in the transcript and decays with compaction. Externalized todos survive compaction if the harness pins the todo list into the post-compaction context.

## Failure modes & anti-patterns

- **Todo porn.** The agent spends its cycles writing todos instead of doing the work. Fix: prompt discipline — only decompose when the task has ≥ 3 distinct steps.
- **Stale in-progress.** The agent starts a task, hits a blocker, moves on, but never updates the todo. Fix: on every turn, check "am I still on the in-progress task? If blocked, re-plan explicitly."
- **Batched completion.** Agent does five things, then marks five todos done at once. User sees no progress, then a burst. Fix: mark-immediately discipline in the system prompt.
- **Single giant task.** "Implement feature X." Provides no signal. Fix: a todo list of one item is a smell; either the task is trivial (don't use a todo) or it should be decomposed.
- **Todos that never complete.** "Refactor auth" sits in in-progress for hours. Fix: such a task is a project, not a todo; break it down before entering it.
- **Hidden parallelism.** Agent sets three todos in-progress simultaneously. The single-in-progress rule exists to prevent the model from deceiving itself about progress.
- **Re-planning without resolution.** After each tool result, the agent rewrites the whole list. This thrashes context. Fix: only amend on meaningful new information.

## When to use (and when not to)

**Use** a todo/scratchpad when:

- The task has ≥ 3 distinct steps.
- You want the user to see progress in real time.
- The agent will run long enough for attention to degrade without an anchor.
- There's a benefit to surfacing discovered subtasks (e.g., during a rename, finding extra call sites).

**Don't** use a todo when:

- The task is a single tool call or two.
- You're doing pure research / Q&A where output is the plan.
- The decomposition is obvious and already encoded in a skill or subagent.

## References

- Claude Code TodoWrite tool documentation — <https://docs.claude.com/en/docs/claude-code>
- Anthropic Engineering, "Effective context engineering for AI agents" — <https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents>
- Chroma Research, "Context Rot" (motivates re-anchoring plans) — <https://research.trychroma.com/context-rot>
- "Plan-and-Solve Prompting", arXiv:2305.04091 — <https://arxiv.org/abs/2305.04091>
- Simon Willison, blog posts on Claude Code scratchpads — <https://simonwillison.net/tags/claude-code/>
