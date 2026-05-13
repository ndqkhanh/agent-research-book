# 03 — Plan Mode

**Definition.** Plan Mode is a harness state in which the agent can read, search, and reason but cannot mutate anything. It produces an explicit plan artifact that the user (or a downstream agent) approves before execution resumes with write permissions.

## Problem it solves

Agents tend to conflate exploration with execution: they start editing before they've finished understanding. This leaks cost (edits that have to be undone), risks (wrong-file modifications), and trust (users watching the agent change things they didn't ask for). A plan-then-execute split makes intent auditable. The user sees exactly what the agent intends to do, in what order, and why — *before* any file changes.

Plan Mode is also the antidote to the common "agent goes off-script for twenty minutes" problem. By forcing the agent to commit to a plan, you get an early confirmation point where misunderstandings can be corrected cheaply.

## Mechanism

Plan Mode is implemented as a combination of (a) permission rules and (b) a final-state tool call. Concretely in Claude Code:

1. **Entry.** User invokes Plan Mode (Shift+Tab in the UI, or a slash-command). The harness switches the permission posture: `Edit`, `Write`, `Bash` (non-readonly), and any write-capable MCP tools are forbidden. Only read-only actions (Read, Grep, Glob, WebFetch, Explore subagents) are allowed.
2. **Workflow.** The agent runs a constrained loop over five phases:
   - *Understanding.* Read and search to map the request onto real code.
   - *Design.* Optionally spawn a Plan subagent for an alternative view.
   - *Review.* Read the critical files the plan will touch.
   - *Final plan.* Write the plan to a designated plan file (the only write allowed).
   - *Confirm.* Call `ExitPlanMode` or ask the user questions via `AskUserQuestion`.
3. **Exit.** `ExitPlanMode` surfaces the plan file's contents to the user for approval. On approval, the harness restores full permissions and resumes the normal loop; the agent's next step executes the approved plan.

The plan file typically lives in `~/.claude/plans/` and is the *only* mutable resource in the plan phase. This single-file-is-mutable policy is the cleanest way to preserve "plan-only" as a hard invariant.

## Concrete pattern

The plan file template that works well in practice:

```markdown
# Plan: <one-line task title>

## Context
<Why this change, what prompted it, what outcome.>

## Scope & decisions (confirmed with user)
- decision 1
- decision 2

## Technique / approach
<One recommended approach only; no ABCs.>

## Files to modify
- `path/to/file.ts:42` — change X because Y
- `path/to/test.ts` — add test for Z

## Reused components
- Existing util `foo()` at `lib/foo.ts:12` — reused, not reimplemented.

## Verification
- Run `npm test -- foo.test.ts`.
- Manual: open `/settings`, toggle X, confirm Y.

## Non-goals
- Not touching the auth module.
- Not refactoring error handling.
```

The *Context*, *Reused components*, and *Verification* sections are non-negotiable. Without them, plan review degenerates into rubber-stamping.

## Variants & related techniques

- **Plan-and-Solve** ([16-plan-and-solve.md](16-plan-and-solve.md)) is the prompt-level analogue: one call produces the plan, a second executes it, no permission enforcement.
- **Plan subagent.** Within Plan Mode, a dedicated `Plan` subagent can propose the plan while the main agent reviews it — a form of [verifier/evaluator loops](11-verifier-evaluator-loops.md) applied to planning itself.
- **ExitPlanMode as structured output.** Some harnesses treat the exit call as a structured tool call with a `plan` field, bypassing the plan-file round-trip. This is simpler but loses the persistent artifact.
- **Two-step approval.** Plan → pseudo-execute (dry run with a diff preview) → approve → execute. Popular in infrastructure and migration tooling.

## Failure modes & anti-patterns

- **Plan inflation.** The agent writes a 3,000-word plan nobody reads. Mitigation: length discipline in the template and in the system prompt. If a section would be boilerplate, remove it.
- **Planning in the wind.** The agent proposes a plan without reading the code first, then the plan turns out to be impossible. Mitigation: enforce a minimum exploration step count before `ExitPlanMode` can be called, or require the plan to cite specific `file:line` references.
- **Plan drift after approval.** The plan said A, the execution did B. Mitigation: after approval, the plan file becomes part of the system prompt for the execution phase, and a pre-commit hook checks that each edit touches a file listed in the plan.
- **Approval fatigue.** If every action needs a plan, users stop reading plans. Mitigation: reserve plan mode for non-trivial or destructive tasks.
- **Forgetting the plan exists.** Multi-hour sessions lose the plan to context compaction. Mitigation: pin the plan file path into memory so compaction preserves it.

## When to use (and when not to)

**Use** Plan Mode when:

- The task spans more than ~5 file edits.
- The blast radius is non-trivial (destructive migrations, schema changes, auth changes).
- You are collaborating with a human reviewer who wants an artifact to approve.
- The requirements are ambiguous — planning forces the ambiguity to surface early.

**Don't use** Plan Mode when:

- The task is a one-line fix or a typo.
- The user has explicitly asked for direct action (e.g., "just do it").
- The agent is already in a tight iteration loop where feedback is fast and cheap.

## References

- Claude Code docs, Plan Mode & Subagents — <https://docs.claude.com/en/docs/claude-code/sub-agents>
- SDpower blog, "Claude Code Subagents and Plan Mode" — <https://blog.sd.idv.tw/en/posts/2025-11-01_claude-code-subagents-and-plan-mode-guide/>
- alexop.dev, "Understanding Claude Code's Full Stack: MCP, Skills, Subagents, and Hooks" — <https://alexop.dev/posts/understanding-claude-code-full-stack/>
- Plan-and-Solve prompting, arXiv:2305.04091 — <https://arxiv.org/abs/2305.04091>
- Anthropic Engineering, "Harness design for long-running application development" — <https://www.anthropic.com/engineering/harness-design-long-running-apps>
