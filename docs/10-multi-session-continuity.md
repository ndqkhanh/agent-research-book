# 10 — Multi-Session Continuity

**Definition.** Multi-session continuity is the set of harness techniques that let an agent work on a task spanning multiple context windows, days, or users — so a session that ends without finishing doesn't throw its work away. The canonical pattern is an *initializer* agent that sets up state once, plus a *coding* agent that makes incremental progress per session and writes clear handoff artifacts for the next one.

## Problem it solves

A context window is finite. A real engineering task isn't. Migrations, refactors, multi-week features, and autonomous background agents all need to survive across sessions. Naive approaches fail in predictable ways:

- **"Just keep going tomorrow"** assumes the next session inherits perfect memory. It doesn't — compaction, session resets, or a different user destroys state.
- **"Dump everything to memory"** floods the next session with irrelevant history.
- **"Resume from commit log"** loses in-progress reasoning and next-step intent.

The initializer + coder split, introduced in Anthropic's "Effective harnesses for long-running agents", addresses this: the initializer establishes a consistent working environment and task record on first run; the coder, on every subsequent session, reads that record, does a bounded increment of work, and updates the record so the *next* session can pick up.

## Mechanism

Three roles collaborate across sessions:

1. **Initializer agent** (runs once).
   - Clones the target repo; sets up dependencies; runs any bootstrap commands.
   - Decomposes the task into a roadmap of smaller work items.
   - Writes a task-state file (`.agent/STATE.md` or similar) that encodes current step, completed steps, open decisions, environment invariants.
   - Exits.

2. **Coding agent** (runs per session).
   - Reads the state file as first action.
   - Picks up the next work item, completes a self-contained chunk (bounded by steps/time/cost).
   - Writes updated state before exit: what it did, what's next, any blockers, new decisions made.
   - Commits progress to a dedicated branch so future sessions have a diff to read.

3. **Evaluator** (optional, per session or per milestone; see [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md)).
   - Checks that the coding agent's output meets objective criteria (tests pass, lint clean, plan step actually done).
   - Updates state accordingly, or rolls back if the evaluator fails the work.

The state file is the crucial artifact. It is *not* a transcript dump — that would be useless. It's a structured hand-off:

```markdown
# STATE.md

## Objective
Migrate 42 endpoints from Flask → FastAPI. Keep behavior identical.

## Progress (8 / 42 endpoints)
- [x] /users/me (session 1)
- [x] /users/<id> (session 1)
- [x] /auth/login (session 2)
- [ ] /auth/register   ← NEXT
- [ ] ...34 more

## Environment invariants
- Run tests with `uv run pytest -q`.
- Mypy strict mode; new code must pass `mypy src`.
- Session must end with `main` green; branch = `migration/fastapi`.

## Open decisions
- Dependency injection: using fastapi.Depends; confirmed via PR #312 comment.

## Blockers / questions for next session
- /auth/register uses a 2-step captcha flow; need Product sign-off on UX parity.

## Last session summary (auto-generated)
- Touched: src/routes/auth.py:40-180
- Tests added: tests/auth/test_login.py (6 cases).
- Cost: 22k tokens, 38 steps, 14 min wall.
```

## Concrete pattern

Session entry script (conceptual):

```python
def start_coding_session(state_path):
    state = read(state_path)
    task = pick_next_work_item(state)

    system_prompt = build_prompt(
        role="coding agent",
        state=state,
        rules=[
            "Work only on the single NEXT item.",
            "Update STATE.md before exiting.",
            "Commit progress on `migration/fastapi`.",
        ],
    )
    run_agent_loop(system_prompt, step_budget=60)

    evaluator.run(task, expectations=state.invariants)  # optional
    update_state(state_path, session_outcome)
```

Anthropic's three-agent harness (planner + generator + evaluator, see [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md)) is a specific elaboration: planner produces the roadmap the initializer wrote; generator is the coding agent; evaluator is the per-session gate.

## Variants & related techniques

- **Checkpointing.** A subset of continuity: snapshot full agent state (transcript, env, process) and resume. Heavier, less portable, but lossless.
- **Worktrees.** Running each session in its own git worktree lets concurrent sessions coexist without stepping on each other. Claude Code SDK has built-in isolation modes.
- **Episodic memory (Reflexion, [14-reflexion.md](14-reflexion.md)).** Cross-episode learning from reflections is orthogonal to state-file continuity — you can do both.
- **External memory services.** Memory is stored in a dedicated service (Cloudflare Agent Memory, Letta, etc.) rather than in-repo files. Better for fleets; adds ops.
- **Batch jobs / cron agents.** Continuity reduces to "run the agent every hour on this state" — an external scheduler replaces interactive sessions.

## Failure modes & anti-patterns

- **State file drift from reality.** Code says endpoint migrated; state says it didn't. Fix: regenerate state partially from repo scan on every session start, not purely from prior state.
- **"Pick up where I left off" with no recap.** The coding agent reads STATE.md and immediately writes code; forgets to read the environment invariants or the last session's blocker. Fix: mandatory first step is to re-read invariants and decide whether the blocker has been resolved.
- **Too-large work items.** If a single "next item" takes 3 sessions, progress is invisible and each session rediscovers the context. Fix: break work items to ≤ 1 session of effort.
- **Too-small work items.** 100 items of 2-minute work drown in session overhead. Fix: batch.
- **Uncommitted work.** Session ends with local changes but no commit; next session can't see what was done. Rule: always end with a commit (or explicit "no progress, here's why").
- **Silent invariant violation.** Agent evolves the state file but ignores the "tests must pass" rule. Fix: evaluator runs pre-commit, not post-session.

## When to use (and when not to)

**Use** multi-session continuity when:

- The task will clearly exceed a single context window or wall-clock budget.
- You want autonomous overnight/weekend runs.
- Multiple humans (or multiple agent variants) will collaborate on the same task.
- You want progress to survive process crashes, OS restarts, or rate-limit windows.

**Don't** use it when:

- The task fits comfortably in one session — you'll spend more effort on state bookkeeping than on work.
- The task is highly interactive and depends on moment-to-moment user guidance.
- You can't define invariants a per-session evaluator can check; without those, continuity accumulates subtle drift.

## References

- Anthropic Engineering, "Effective harnesses for long-running agents" — <https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents>
- Anthropic Engineering, "Harness design for long-running application development" — <https://www.anthropic.com/engineering/harness-design-long-running-apps>
- InfoQ, "Anthropic Designs Three-Agent Harness Supports Long-Running Full-Stack AI Development" — <https://www.infoq.com/news/2026/04/anthropic-three-agent-harness-ai/>
- ZenML LLMOps DB, "Long-Running Agent Harness for Multi-Context Software Development" — <https://www.zenml.io/llmops-database/long-running-agent-harness-for-multi-context-software-development>
- Cognition, "Devin's context engineering" / engineering blog — <https://cognition.ai/blog>
