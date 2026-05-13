# 165 — Ralph: A Bash Loop Around Coding Agents That Persists Memory in Git

**Repository.** https://github.com/snarktank/ralph — author: **Ryan Carson** (snarktank) — based on **Geoffrey Huntley's "Ralph" pattern** (https://ghuntley.com/ralph/) — single-author, MIT-licensed, ~150 LoC of bash + Markdown skill files. Companion article: https://x.com/ryancarson/status/2008548371712135632 ("Read my in-depth article on how I use Ralph"). Distributed via three install paths: per-project copy, global skills install, and **Claude Code Marketplace plugin** (`/plugin marketplace add snarktank/ralph` → `/plugin install ralph-skills@ralph-marketplace`).

**One-line definition.** Ralph is an **autonomous AI agent loop** that runs a coding tool (**Amp CLI** by default, or **Claude Code**) repeatedly until all PRD items are complete — with the central architectural insight that **each iteration is a fresh AI instance with completely clean context**, and **memory persists outside the AI's context** via three durable artefacts: **`git` history** (commits per completed story), **`progress.txt`** (a Codebase Patterns section + chronological iteration log), and **`prd.json`** (the structured Product Requirements Document with `passes: true/false` per user story). The whole orchestration is a ~150-line bash loop that prints the prompt to the AI tool's stdin, watches stdout for a `<promise>COMPLETE</promise>` signal, and reruns until completion or a max-iteration cap.

## Why a 150-line bash loop is one of the most important agent patterns

Most "agent loop" architectures get more sophisticated over time: longer prompts, more tools, more memory, more state, more orchestration code. Ralph goes the other direction: it strips the loop down to its essentials and shows that **a fresh AI instance + durable external memory + well-structured PRD is enough** for autonomous coding work over many iterations.

The pattern was originally articulated by **Geoffrey Huntley** in his blog post (https://ghuntley.com/ralph/). Snarktank's Ralph is the open-source operationalisation: a cloneable repo, a working bash script, prompts for both Amp and Claude Code, a Claude Code Marketplace plugin, and a flowchart visualisation. Because the implementation is so small, it serves as a *teaching artefact* — anyone reading the 150-line `ralph.sh` understands the entire pattern in 10 minutes.

For the harness-engineering canon, Ralph is significant for three reasons:

1. **It is the cleanest illustration of "memory is files, not context."** The agent's context is reset every iteration. Anything the agent needs to remember must be written to disk — git, progress.txt, prd.json. This *forces* the architectural discipline that MEMTIER ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)) and Feynman ([155](155-feynman-multi-agent-research-harness.md)) advocate, but at a far simpler scale.
2. **It demonstrates the durability advantage of git as memory.** The completed work, the commit messages, and the diff history are all queryable by future iterations. This is one of the cleverest reuses of existing infrastructure — git is already a content-addressable durable database with rich query semantics. Ralph leans on it.
3. **It is the multi-iteration coding loop that scales without growing context.** A long-running coding session in Claude Code or Cursor will eventually run into context degradation. Ralph sidesteps the issue: each iteration starts fresh; the cumulative work lives in git.

The pattern generalises beyond coding to any iterative task that can be decomposed into a PRD-style list of acceptance-criteria-bearing stories.

## The Ralph pattern (from Geoffrey Huntley)

The original pattern, as explained in the source blog post:

```
1. Have an agent (Claude Code, Amp, etc.) work on a task.
2. When the agent reports completion or hits a stopping point, kill it.
3. Spawn a fresh instance of the same agent with the same prompt.
4. The fresh instance reads the same external state (git history, progress notes, PRD).
5. The fresh instance picks up where the previous one left off, but with clean context.
6. Repeat until all tasks are complete.
```

Three properties make this work:

- **Context degradation is bounded** because each instance starts fresh.
- **Cross-iteration learning** happens via the external state files.
- **Fault tolerance** is automatic: if an iteration crashes, the next iteration sees the same external state and can recover.

This is mechanically similar to Temporal's durable-execution model ([150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md)) — *the workflow code expresses what should happen, the runtime handles re-running and resuming*. Ralph is a poor-man's Temporal for agent loops, where the "runtime" is `ralph.sh` and the "workflow code" is the agent prompt + PRD.

## The bash loop in detail

The full `ralph.sh` (paraphrased and simplified):

```bash
#!/bin/bash
# Ralph: long-running AI agent loop
# Usage: ./ralph.sh [--tool amp|claude] [max_iterations]

TOOL="amp"          # Default tool
MAX_ITERATIONS=10   # Default cap
# (argument parsing omitted)

PRD_FILE="$SCRIPT_DIR/prd.json"
PROGRESS_FILE="$SCRIPT_DIR/progress.txt"
ARCHIVE_DIR="$SCRIPT_DIR/archive"
LAST_BRANCH_FILE="$SCRIPT_DIR/.last-branch"

# Archive previous run if branch changed
if [ -f "$PRD_FILE" ] && [ -f "$LAST_BRANCH_FILE" ]; then
  CURRENT_BRANCH=$(jq -r '.branchName // empty' "$PRD_FILE")
  LAST_BRANCH=$(cat "$LAST_BRANCH_FILE")
  if [[ "$CURRENT_BRANCH" != "$LAST_BRANCH" ]]; then
    DATE=$(date +%Y-%m-%d)
    ARCHIVE_FOLDER="$ARCHIVE_DIR/$DATE-$(echo $LAST_BRANCH | sed 's|^ralph/||')"
    mkdir -p "$ARCHIVE_FOLDER"
    cp "$PRD_FILE" "$PROGRESS_FILE" "$ARCHIVE_FOLDER/"
    # Reset progress file for new run
    echo "# Ralph Progress Log" > "$PROGRESS_FILE"
  fi
fi

# Main loop
for i in $(seq 1 $MAX_ITERATIONS); do
  echo "Ralph Iteration $i of $MAX_ITERATIONS ($TOOL)"

  # Run the selected tool with the ralph prompt
  if [[ "$TOOL" == "amp" ]]; then
    OUTPUT=$(cat "$SCRIPT_DIR/prompt.md" | amp --dangerously-allow-all 2>&1 | tee /dev/stderr)
  else
    OUTPUT=$(claude --dangerously-skip-permissions --print < "$SCRIPT_DIR/CLAUDE.md" 2>&1 | tee /dev/stderr)
  fi

  # Check for completion signal
  if echo "$OUTPUT" | grep -q "<promise>COMPLETE</promise>"; then
    echo "Ralph completed all tasks!"
    exit 0
  fi

  echo "Iteration $i complete. Continuing..."
  sleep 2
done

echo "Ralph reached max iterations ($MAX_ITERATIONS) without completing."
exit 1
```

Five operational details worth highlighting:

### 1. Pipe the prompt to the agent

```bash
cat prompt.md | amp --dangerously-allow-all
# or
claude --dangerously-skip-permissions --print < CLAUDE.md
```

The prompt file (`prompt.md` for Amp, `CLAUDE.md` for Claude Code) is piped to the AI's stdin. The AI runs autonomously in the background, doing actual work (file edits, git commits, tests, etc.).

The `--dangerously-allow-all` (Amp) and `--dangerously-skip-permissions` (Claude Code) flags grant unattended autonomy. Without them, the AI would prompt for permission on each tool use — fatal for an autonomous loop. These flags are explicit acknowledgement that the user is giving up interactive supervision; the discipline that constrains the AI's behaviour comes from the *prompt*, not the runtime sandbox.

### 2. Watch for the completion signal

```bash
if echo "$OUTPUT" | grep -q "<promise>COMPLETE</promise>"; then
  exit 0
fi
```

The agent emits a literal `<promise>COMPLETE</promise>` tag in its output when it determines all work is done. The bash loop greps for this and exits successfully.

The choice of `<promise>...</promise>` is intentional: it's a structured marker that won't appear naturally in regular output. If the agent says "I think I'm complete", that doesn't trigger; only the explicit promise tag does. This is the agent equivalent of a SQL `RAISE` or a function `return` — explicit semantics for state transitions.

### 3. Archive on branch change

```bash
if [[ "$CURRENT_BRANCH" != "$LAST_BRANCH" ]]; then
  ARCHIVE_FOLDER="$ARCHIVE_DIR/$DATE-$(echo $LAST_BRANCH | sed 's|^ralph/||')"
  cp "$PRD_FILE" "$PROGRESS_FILE" "$ARCHIVE_FOLDER/"
fi
```

When the user starts work on a new feature (changes the `branchName` in `prd.json`), Ralph archives the previous run's PRD and progress log. This keeps a historical trail of every Ralph run organised by date and branch.

The archive directory becomes a reading log of "what Ralph did, when, on what branch" — extremely useful for retrospectives, blame analysis, and learning from past runs.

### 4. Reset progress on new branch

```bash
echo "# Ralph Progress Log" > "$PROGRESS_FILE"
echo "Started: $(date)" >> "$PROGRESS_FILE"
```

When the branch changes, the progress log is reset (not appended). This prevents progress from one feature contaminating the working memory of the next feature. The previous progress is preserved in the archive.

### 5. Sleep 2 seconds between iterations

```bash
sleep 2
```

Trivial but important: prevents tight-looping if the AI fails immediately. Gives operating-system time for git commits to settle, file handles to close.

## The PRD format (`prd.json`)

The Product Requirements Document is structured JSON:

```json
{
  "project": "MyApp",
  "branchName": "ralph/task-priority",
  "description": "Task Priority System - Add priority levels to tasks",
  "userStories": [
    {
      "id": "US-001",
      "title": "Add priority field to database",
      "description": "As a developer, I need to store task priority so it persists across sessions.",
      "acceptanceCriteria": [
        "Add priority column to tasks table: 'high' | 'medium' | 'low' (default 'medium')",
        "Generate and run migration successfully",
        "Typecheck passes"
      ],
      "priority": 1,
      "passes": false,
      "notes": ""
    },
    {
      "id": "US-002",
      "title": "Display priority indicator on task cards",
      ...
    }
    ...
  ]
}
```

Five fields per user story:

- **id**: stable identifier (US-001, US-002, ...).
- **title**: short imperative description.
- **description**: in-prose user-story format ("As a [role], I need [capability] so that [benefit]").
- **acceptanceCriteria**: explicit, testable criteria. The agent uses these to know when the story is "done".
- **priority**: ordering integer. Lower = higher priority.
- **passes**: boolean. The flag the agent updates to mark a story complete.
- **notes**: free-text annotations.

The PRD is also a *progress tracker*. As the agent completes stories, it sets `passes: true`. The next iteration's first action: pick the highest-priority story where `passes: false`. When all stories have `passes: true`, the agent emits `<promise>COMPLETE</promise>` and the loop exits.

This is *the* central data structure of Ralph. The PRD is simultaneously:

- **The plan** (what work needs to happen).
- **The contract** (what acceptance criteria define done).
- **The progress tracker** (which items are complete).
- **The communication channel** (notes per story).

Compare with [12-todo-scratchpad-state](12-todo-scratchpad-state.md): the PRD is the externalised working memory of the agent. The structure makes it readable, editable, and grep-able by humans and other agents.

## The agent prompts (`prompt.md` for Amp, `CLAUDE.md` for Claude Code)

The two prompt files are nearly identical — both ~150 lines, both implementing the same Ralph instructions. The differences:

- `prompt.md` includes a thread URL reference for Amp (`Thread: https://ampcode.com/threads/$AMP_CURRENT_THREAD_ID`).
- `prompt.md` requires browser testing for frontend stories ("MUST verify"); `CLAUDE.md` says "verify if available".
- `prompt.md` references the `dev-browser` skill explicitly.

Both prompts share the same structure:

```markdown
# Ralph Agent Instructions

You are an autonomous coding agent working on a software project.

## Your Task

1. Read the PRD at `prd.json` (in the same directory as this file)
2. Read the progress log at `progress.txt` (check Codebase Patterns section first)
3. Check you're on the correct branch from PRD `branchName`. If not, check it out or create from main.
4. Pick the **highest priority** user story where `passes: false`
5. Implement that single user story
6. Run quality checks (e.g., typecheck, lint, test - use whatever your project requires)
7. Update CLAUDE.md/AGENTS.md files if you discover reusable patterns (see below)
8. If checks pass, commit ALL changes with message: `feat: [Story ID] - [Story Title]`
9. Update the PRD to set `passes: true` for the completed story
10. Append your progress to `progress.txt`

## Progress Report Format
APPEND to progress.txt (never replace, always append):
[format with Date/Time, Story ID, what was implemented, files changed, learnings]

## Consolidate Patterns
[instructions to add reusable patterns to a "Codebase Patterns" section at top of progress.txt]

## Update CLAUDE.md/AGENTS.md Files
[instructions to add valuable learnings to nearby CLAUDE.md or AGENTS.md files in directories you modified]

## Quality Requirements
- ALL commits must pass your project's quality checks (typecheck, lint, test)
- Do NOT commit broken code
- Keep changes focused and minimal
- Follow existing code patterns

## Browser Testing
[verify UI changes if browser tools are available]

## Stop Condition
After completing a user story, check if ALL stories have `passes: true`.
If ALL stories are complete and passing, reply with:
<promise>COMPLETE</promise>

If there are still stories with `passes: false`, end your response normally
(another iteration will pick up the next story).

## Important
- Work on ONE story per iteration
- Commit frequently
- Keep CI green
- Read the Codebase Patterns section in progress.txt before starting
```

Three architectural patterns embedded in the prompt:

### 1. "One story per iteration" discipline

The agent works on *exactly one* user story per iteration. This bounds the work-in-progress, makes commits clean, and keeps the iteration small enough to fit in a single context window.

This is the Ralph pattern's most important constraint. Without it, the agent would try to "finish everything" in one iteration, accumulating context and stale state. With it, the work decomposes naturally into clean atomic units.

### 2. Progress.txt as growing memory

The agent appends to `progress.txt` every iteration:

```
## [Date/Time] - [Story ID]
- What was implemented
- Files changed
- **Learnings for future iterations:**
  - Patterns discovered (e.g., "this codebase uses X for Y")
  - Gotchas encountered
  - Useful context
---
```

The "Learnings" section is the *single most important* piece of cross-iteration memory. Future iterations read this and know:

- What conventions the codebase uses.
- What gotchas to avoid.
- Where specific functionality lives.

This is the agent equivalent of an institutional knowledge base — but generated *by the agent itself* during its own runs.

### 3. Codebase Patterns section as consolidated learnings

At the top of `progress.txt`, the agent maintains a `## Codebase Patterns` section:

```
## Codebase Patterns
- Use `sql<number>` template for aggregations
- Always use `IF NOT EXISTS` for migrations
- Export types from actions.ts for UI components
```

These are *general and reusable*, not story-specific. As the agent works, it consolidates the most important learnings into this section. Future iterations read it *first*, before doing work.

This is the same compaction discipline as MEMTIER's semantic tier ([152-memtier-3-tier-architecture-and-retrieval](152-memtier-3-tier-architecture-and-retrieval.md)): episodic learnings (the iteration logs) are distilled into stable knowledge (the Codebase Patterns). The agent does the distillation as part of its own workflow.

### 4. CLAUDE.md / AGENTS.md updates near touched files

If the agent discovers reusable patterns while working on files in a directory, it updates the *nearest* CLAUDE.md or AGENTS.md to record them. The convention:

> Examples of good CLAUDE.md additions:
> - "When modifying X, also update Y to keep them in sync"
> - "This module uses pattern Z for all API calls"
> - "Tests require the dev server running on PORT 3000"
> - "Field names must match the template exactly"

This is *file-local* memory — knowledge stored next to the files it pertains to. When the agent later works in that directory, it sees the CLAUDE.md / AGENTS.md context. Memory placement matters; co-locating context with code is the right discipline.

## The skills (skills/prd, skills/ralph)

Ralph ships two Claude Code skills:

### `/prd` — Generate Product Requirements Documents

Helps the user create a new PRD from scratch. Activates on phrases like:
- "create a prd"
- "write prd for"
- "plan this feature"

### `/ralph` — Convert PRD to prd.json format

Takes an existing PRD (in any format) and converts it to the structured `prd.json` Ralph expects. Activates on:
- "convert this prd"
- "turn into ralph format"
- "create prd.json"

These skills are installed via the Claude Code Marketplace:

```bash
/plugin marketplace add snarktank/ralph
/plugin install ralph-skills@ralph-marketplace
```

The marketplace integration is significant: Ralph isn't just a script you copy into your project — it's a *skill* that any Claude Code user can install. The friction to start using Ralph is dramatically reduced.

## Three install paths

The README documents three ways to use Ralph:

### Option 1 — Per-project copy

```bash
mkdir -p scripts/ralph
cp /path/to/ralph/ralph.sh scripts/ralph/
cp /path/to/ralph/prompt.md scripts/ralph/    # for Amp
# OR
cp /path/to/ralph/CLAUDE.md scripts/ralph/    # for Claude Code
chmod +x scripts/ralph/ralph.sh
```

Each project gets its own copy. Simple and self-contained.

### Option 2 — Global skills install (Amp)

```bash
cp -r skills/prd ~/.config/amp/skills/
cp -r skills/ralph ~/.config/amp/skills/
```

Or for Claude Code:

```bash
cp -r skills/prd ~/.claude/skills/
cp -r skills/ralph ~/.claude/skills/
```

Skills are available across all projects.

### Option 3 — Claude Code Marketplace plugin

```bash
/plugin marketplace add snarktank/ralph
/plugin install ralph-skills@ralph-marketplace
```

The cleanest path. No copying, no manual setup. Skills auto-update.

The progression illustrates the *skills-as-portable-artefacts* maturity curve: from copy-paste to global config to marketplace. Ralph's progression matches Karpathy's "skills as single-file guardrails" thesis ([71-karpathy-skills-single-file-guardrails](71-karpathy-skills-single-file-guardrails.md)) and the "skills as the new training target" insight at [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md).

## Auto-handoff via Amp settings

The README mentions Amp auto-handoff configuration:

```json
// ~/.config/amp/settings.json
{
  // ... add Ralph auto-handoff config
}
```

Auto-handoff lets Ralph hand off mid-iteration when a story is too large for a single context window. The current iteration writes a partial PRD update; the next iteration picks up the partial work. This extends Ralph's reach beyond stories that fit in one context.

## Why git-as-memory is the killer architectural insight

Most agent memory architectures are designed from scratch. Ralph reuses git, which is already:

- **Content-addressable**: every commit has a hash, every file has a hash, retrieval by hash is O(1).
- **Branchable**: parallel work doesn't interfere; merge semantics are well-defined.
- **Diffable**: the *change* between two states is computable in milliseconds.
- **Searchable**: `git log --grep`, `git log --author`, `git log -S` are powerful queries.
- **Auditable**: every change has an author, timestamp, message, parent commit.
- **Distributed**: works offline, syncs via push/pull.
- **Already deployed everywhere**: every developer machine, every CI server, every code host.

For an agent that produces code, git is a near-perfect memory substrate. The agent's "history of what it did" is the commit log. The agent's "current state of work" is `git diff main..HEAD`. The agent's "previous attempts that failed" are reflected in the branch history.

Ralph leans on this hard. The agent's iteration prompt explicitly says: *"commit ALL changes with message `feat: [Story ID] - [Story Title]`"*. Each commit becomes a durable memory record of what the agent accomplished.

This is one of the most reusable architectural insights in the canon: **don't invent agent memory; reuse durable infrastructure that already exists**. For coding agents, git is right there. For research agents, the filesystem + structured Markdown is right there. The discipline is to use what's already proven.

## Comparison with other autonomous loops

How Ralph compares to adjacent autonomous-loop patterns:

| System | Loop primitive | Memory substrate | Stop signal |
|--------|---------------|-------------------|-------------|
| **Ralph** | Bash for-loop | git + progress.txt + prd.json | `<promise>COMPLETE</promise>` |
| Deep Researcher Agent ([160](160-deep-researcher-agent-24x7.md)) | THINK→EXECUTE→REFLECT cycle | Two-tier constant memory (~5K chars) | Run forever (no terminal goal) |
| Voyager ([19-voyager-skill-libraries](19-voyager-skill-libraries.md), [89-voyager-deep](89-voyager-deep.md)) | Curriculum + skill library | Skill library + Minecraft world state | Curriculum exhaustion |
| Reflexion ([14-reflexion](14-reflexion.md), [90-reflexion-deep](90-reflexion-deep.md)) | Try-reflect-retry | Episodic memory (verbal reflection) | Task success |
| MetaGPT ([20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md), [91-metagpt-deep](91-metagpt-deep.md)) | SDLC-role pipeline | Artefacts at each role boundary | Final deliverable produced |
| Memento ([109-memento-results-and-harness](109-memento-results-and-harness.md)) | M-MDP with case bank | Persistent case bank with cognitive weights | Task success |

Ralph is the **simplest** of these patterns, by a wide margin. The bash-loop-around-coding-tool design is roughly 100 LoC including all the archive and branch-handling logic. The other patterns are orders of magnitude more complex.

What Ralph trades for simplicity:

- **No structured replanning**. The PRD is fixed; the agent works through it linearly. Reflexion-style adaptive replanning isn't part of the model.
- **No skill library**. Voyager-style accumulation of reusable skills is replaced by the simpler "Codebase Patterns" section.
- **No multi-agent collaboration**. MetaGPT's role-based pipeline isn't applicable.
- **No reward learning**. Memento's case-bank-driven policy improvement isn't part of the architecture.

What Ralph gains:

- **Trivial to understand**. 10 minutes of reading produces full comprehension.
- **Trivial to extend**. The bash loop is small enough to modify confidently.
- **Robust to crashes**. Each iteration is independent; a crash doesn't lose accumulated state.
- **Composable with any coding tool**. Amp, Claude Code, and (with adapters) any CLI-driven AI.

For many practical workloads — small-team coding tasks, well-decomposed PRDs, automation of repetitive feature additions — Ralph's simplicity is the *right* answer.

## Limitations

A few honest constraints:

1. **PRD-decomposability assumption**. The pattern requires the work to be expressible as a list of acceptance-criteria-bearing user stories. Open-ended exploration ("figure out why the system is slow") doesn't fit the PRD model.
2. **Quality gate is the test suite**. Ralph trusts your typecheck, lint, and test suite to catch broken code. If your tests are poor, Ralph commits broken code. The pattern transfers your quality discipline; it doesn't add to it.
3. **No mid-iteration human intervention**. The agent runs to completion (or to the iteration's natural endpoint). If you want to redirect mid-iteration, you wait for the iteration to end. Compare with Deep Researcher Agent's HUMAN_DIRECTIVE.md ([160](160-deep-researcher-agent-24x7.md)) for richer human-in-the-loop semantics.
4. **Bash-only orchestration**. Windows users need WSL. Native Windows orchestration would require a PowerShell port.
5. **Agent must support `--print` and stdin piping**. Amp and Claude Code do; other coding CLIs may not. The pattern adapts but requires CLI compatibility.
6. **No native rollback**. If a story is committed but later judged wrong, you must `git revert` manually. Ralph does not auto-detect "this commit shouldn't have happened".
7. **`--dangerously-allow-all` / `--dangerously-skip-permissions` are real risks**. The agent can do anything the running user can do — `rm -rf`, push to production, etc. The PRD prompt is the only constraint. For high-trust environments only.

## Where this pattern shines

Ralph is the right pattern when:

- You have a well-decomposed feature with clear user stories.
- Your test suite is reliable (Ralph leans on it).
- You want to operate "fire and forget" — start Ralph, come back later, see all stories committed.
- Your agent should not lose progress to context degradation across many hours.
- You want a simple, auditable orchestration that fits in 100 lines of bash.

Ralph is the wrong pattern when:

- The work is exploratory or research-style (no clear acceptance criteria).
- Your quality gates are weak (Ralph would commit broken code).
- You need fine-grained human-in-the-loop control.
- Multi-agent collaboration is required (Ralph is single-agent).
- You're not comfortable granting `--dangerously-allow-all` access.

## Practical takeaways

For builders, the architectural lessons:

### 1. Externalise memory to durable substrates

git, filesystem, structured Markdown — use what's already there. Don't reinvent agent memory; lean on what's proven.

### 2. Force fresh-context iterations

Every iteration starting fresh prevents context degradation. The work moves forward via the external state, not via accumulated context. Same insight as Deep Researcher Agent's per-cycle conversation reset ([160](160-deep-researcher-agent-24x7.md)).

### 3. Use structured stop signals

`<promise>COMPLETE</promise>` is unambiguous. The bash grep is exact. Don't try to parse "I think it's done" — define an explicit termination signal.

### 4. Distil learnings into a stable artefact

Ralph's "Codebase Patterns" section at the top of `progress.txt` is the agent's distilled wisdom. The same pattern in MEMTIER ([152](152-memtier-3-tier-architecture-and-retrieval.md)) is the consolidation daemon's semantic tier.

### 5. Co-locate memory with the code it documents

CLAUDE.md / AGENTS.md updates in modified directories. When the agent next works there, it sees the local context. Avoid centralised memory dumps; place context where it's needed.

### 6. Make termination explicit and auditable

The PRD's `passes` flag per story is queryable. The completion condition is "all stories pass". The audit trail is the commit log. Termination semantics are clear at every level.

### 7. One story per iteration

Bound the work-in-progress. Smaller iterations commit cleaner. Resume from clean state on failure. This is Conway's law for agent loops: keep the unit of work matched to the bounded context.

## Where this fits in the canon

- **Read alongside**: [09-memory-files](09-memory-files.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md), [144-build-your-own-harness](144-build-your-own-harness.md), [160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md), [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md).
- **Autonomous-loop family**: [14-reflexion](14-reflexion.md), [19-voyager-skill-libraries](19-voyager-skill-libraries.md), [20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md), [89-voyager-deep](89-voyager-deep.md), [90-reflexion-deep](90-reflexion-deep.md), [109-memento-results-and-harness](109-memento-results-and-harness.md).
- **Coding agents**: [29-dive-into-claude-code](29-dive-into-claude-code.md), [46-components-of-coding-agent](46-components-of-coding-agent.md), [62-everything-claude-code](62-everything-claude-code.md), [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md), [145-comparing-coding-harnesses](145-comparing-coding-harnesses.md).
- **Skills as portable artefacts**: [04-skills](04-skills.md), [71-karpathy-skills-single-file-guardrails](71-karpathy-skills-single-file-guardrails.md), [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md), [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md).
- **Synthesis**: [166-research-agent-frameworks-synthesis](166-research-agent-frameworks-synthesis.md).

## References

1. Repository: https://github.com/snarktank/ralph.
2. Original Ralph pattern (Geoffrey Huntley): https://ghuntley.com/ralph/.
3. Author's in-depth article: https://x.com/ryancarson/status/2008548371712135632.
4. Amp CLI: https://ampcode.com.
5. Claude Code: https://docs.anthropic.com/en/docs/claude-code.
6. Claude Code Marketplace plugin: `/plugin marketplace add snarktank/ralph`.
7. Adjacent canon chapters listed above.
8. Companion `prd.json.example` shows the structured PRD format.
9. The `flowchart/` subdirectory contains an interactive React Flow diagram visualising how Ralph works (run `npm run dev` from `flowchart/`).
