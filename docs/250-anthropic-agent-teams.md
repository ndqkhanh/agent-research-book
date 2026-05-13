# 250 — Anthropic Agent Teams: lead-and-spokes runtime with shared task list and peer messaging

**Source.** Anthropic — *Claude Code Agent Teams* — https://code.claude.com/docs/en/agent-teams — Feb 2026 launch alongside Claude Opus 4.6. Experimental flag: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. Requires Claude Code v2.1.32+. Companion docs: [Subagents](https://code.claude.com/docs/en/sub-agents), [Hooks](https://code.claude.com/docs/en/hooks), [Costs](https://code.claude.com/docs/en/costs).

**One-line definition.** A runtime coordination feature in Claude Code that spawns multiple **fully independent Claude Code sessions** ("teammates") under a **lead** session, with a **shared task list** at `~/.claude/tasks/{team-name}/`, **direct peer-to-peer messaging** via mailbox, and **automatic dependency resolution** — distinct from subagents (which run inside one session and report back) and from worktrees (which require manual coordination) — costing roughly **7× tokens in plan mode** because each teammate is a full Claude instance with its own context window.

## Why this paper matters (Anthropic's own consensus 2026 architecture for agent teams, productized)

Agent Teams is the first time Anthropic has shipped a **first-party multi-agent runtime** as a Claude Code feature, and the shape it takes is informative: it is not the free-form-chat debate model (CrewAI / AutoGen pre-2025) and not the strict SOP-document pattern (MetaGPT / ChatDev). It is a **lead-and-spokes hybrid** with two load-bearing innovations: (a) **teammates are full independent Claude Code sessions** — own permission scope (inherited from lead at spawn), own MCP connectors, own context — running in their own tmux/iTerm2 pane or cycled in-process; (b) **coordination happens through three structured channels** (shared task list, mailbox messages, plan approvals) rather than free dialogue. The architecture is the practical answer to two failure modes the multi-agent literature has been documenting in 2025–2026: monolithic-context overrun and free-chat debate collapse ([224](224-multi-agent-parallel-scaling.md), [251](251-multi-agent-teams-2026-synthesis.md), [98-diversity-collapse-mas](98-diversity-collapse-mas.md)).

The version of multi-agent that ships is also a deliberate retreat from the "agents debate freely until consensus" pattern of 2023–2024. Teammates do not chat at each other in the open; they post to a shared task list, claim work, and exchange targeted messages by name. The lead session is the only orchestrator (no nested teams, no leadership transfer). This is closer to the OrgAgent / hierarchical-MAS taxonomy frame ([251](251-multi-agent-teams-2026-synthesis.md)) than to AutoGen's GroupChat. The cost story is unflinching: Anthropic states ~7× tokens in plan mode versus a single session, and recommends 3–5 teammates with 5–6 tasks each — the team is a *parallelism amplifier* for naturally-parallel work, not a capability multiplier on sequential reasoning.

Take this seriously and three things change. **First**, you stop treating "agent teams" as a research curiosity and start treating it as the de-facto Claude-Code-native pattern for parallel review, multi-hypothesis investigation, and divide-and-conquer refactors — the shape that ships is the shape your harness should support. **Second**, you understand the **architectural primitives** are minimal and reusable: shared task list (filesystem) + mailbox (filesystem messages) + isolated worktrees + lead-managed permissions — none of these require a new protocol or service; they are filesystem and process patterns. **Third**, the **token-cost model** is a hard ceiling on team size: 7× for K=3–5 teammates means parallel teams are economical only when the wall-clock saved exceeds the dollars spent, which is true for code reviews and multi-file refactors but not for sequential reasoning chains.

## Problem it solves (parallel agentic work without context overrun or debate collapse)

1. **Subagents are bottlenecked by the main session.** Subagents run inside one session, return summaries to the caller, and the main agent context fills with their results. For multi-file refactors or parallel reviews, this becomes a context-overrun problem before the work is done.
2. **Free-chat debate fails.** "Talk Isn't Always Cheap" (arXiv:2509.05396, [251](251-multi-agent-teams-2026-synthesis.md)) shows debate accuracy can *decline* as agents prefer agreement over challenge. Teams need structure, not dialogue.
3. **Worktrees were manual.** Developers had to spawn multiple Claude sessions in separate worktrees and coordinate by hand — task lists in shared docs, status in scratch files, no claim-locking.
4. **Permission inheritance was unclear.** When you spawn a child agent in a different worktree, what tools does it have? Agent Teams encodes the rule: teammates inherit lead's permission mode at spawn (with `--dangerously-skip-permissions` propagating).
5. **No structured way to fire parallel investigations.** Reviewers, multi-hypothesis debugging, multiple competing implementations — these all want K parallel agents that compare results, with zero ceremony to set up.
6. **Tokens-per-task were unpredictable.** No prior framework for budgeting parallel agent runs. Agent Teams's 7× number gives a floor.

## Core idea in one paragraph

A **lead session** spawns N **teammate sessions** (each a full Claude Code instance with its own context window) by natural-language request — `"Spawn three teammates to review the security, performance, and test-coverage of this PR"` or by-name reference to a registered subagent definition. The lead does not pass the conversation history to teammates; each gets a **focused spawn prompt** plus the shared project context (CLAUDE.md, MCP servers, skills). The team coordinates through three channels: a **shared task list** stored as files in `~/.claude/tasks/{team-name}/` (states: pending / in-progress / completed; with file-locking to prevent race conditions on claim and automatic dependency unblocking), a **mailbox** for asynchronous direct messages between any two members (delivered automatically, idle notifications back to lead), and **plan approvals** the lead can require before teammates make changes. The display is **in-process** (cycle teammate panes with Shift+Down) or **split-pane** via tmux/iTerm2. Token cost is **~7× a single session in plan mode** because each teammate is a full Claude instance; the recommended team size is 3–5 with 5–6 tasks each. **Hooks** fire on `TeammateIdle`, `TaskCreated`, `TaskCompleted` — exit code 2 blocks the action. Permissions are inherited from the lead at spawn and are not per-teammate by default. The Claude Code Agent Teams pattern is the consensus 2026 multi-agent runtime architecture made concrete: lead-and-spokes hybrid with structured channels, isolated context per teammate, file-based coordination state, no nesting, no leadership transfer.

## Mechanism (step by step)

### (a) Enabling the feature

```bash
# settings.json
{
  "experimental": {
    "agentTeams": true
  },
  "teammateMode": "in-process"  // or "tmux" | "auto"
}
```

Or environment variable: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 claude`. Requires v2.1.32+. Disabled by default to prevent accidental token overage.

### (b) Spawning a team

The lead session decides whether to spawn teammates based on task complexity, or you explicitly request:

> "Create a team with 4 teammates to refactor the auth, logging, billing, and notifications modules in parallel."

You can name teammates and specify their model:

> "Spawn a security-focused reviewer using Sonnet, a performance reviewer using Opus, and a test-coverage reviewer using Sonnet."

Optionally reference an existing **subagent definition** (`.claude/agents/<name>.md`) by name to apply tool restrictions and a custom system prompt to that specific teammate. Subagent definitions have higher precedence than spawn prompts for tool-grant scope.

### (c) The shared task list

Stored at `~/.claude/tasks/{team-name}/`:

- One file per task, with frontmatter for state, owner, dependencies, created-at.
- States: `pending`, `in-progress`, `completed`, `blocked`.
- Lead and teammates all read; teammates self-claim unassigned tasks via file-locking.
- **Automatic dependency resolution** — when a task in `dependencies: [...]` completes, blocked tasks unblock.
- Status can lag: teammates sometimes fail to mark `completed`; documented limitation.

### (d) The mailbox

- Each teammate has an inbox at `~/.claude/teams/{team-name}/mailbox/{member-name}/`.
- Send by name: `"Tell the security-reviewer that auth.py uses bcrypt, not SHA-1"`.
- Idle notifications are auto-sent to the lead when a teammate finishes its claimed work.
- Direct peer-to-peer messaging is allowed (any teammate → any teammate); lead is not in the path.

### (e) Display modes

- **In-process** (default): all teammates run in the main terminal; cycle with Shift+Down.
- **Split panes**: each teammate gets its own tmux or iTerm2 pane; click into a pane to interact directly.
- Configurable via `"teammateMode"` in settings or `--teammate-mode` flag.
- Split-pane requires tmux or iTerm2; not supported in VS Code integrated terminal, Windows Terminal, or Ghostty.

### (f) Permissions and tool scope

- Teammates inherit lead's `permissionMode` at spawn.
- `--dangerously-skip-permissions` propagates from lead to all teammates (intended).
- Per-teammate permissions cannot be set at spawn; can be changed individually post-spawn via lead instructions.
- **Subagent definitions** override the inherited scope for tool grants — a teammate using `.claude/agents/reviewer.md` gets the tools listed in that definition's frontmatter, not the lead's full set.
- Tool grants are inherited from lead unless a subagent definition overrides; common operations should be pre-approved in `permissions.allow` to reduce friction.

### (g) Hooks

Three new hook events for team observability and gating:

- **`TeammateIdle`** — fires when a teammate is about to go idle. Exit 2 to send feedback and keep them working.
- **`TaskCreated`** — fires when a new task is added to the shared list. Exit 2 to prevent creation.
- **`TaskCompleted`** — fires when a task is marked complete. Exit 2 to prevent completion (forces revision).

Hooks fire on the lead and run in the lead's process; teammates do not run hooks individually.

### (h) Limits and known issues

- **No session resumption with in-process teammates.** `/resume` and `/rewind` do not restore them.
- **Task status can lag** — manual nudge sometimes required.
- **Shutdown can be slow** when many teammates are active.
- **One team per lead.** Clean up before creating a new one.
- **No nested teams.** Teammates cannot spawn teams.
- **Lead is fixed** — cannot promote a teammate or transfer leadership.

## Empirical results (table — Anthropic's recommended-pattern table)

| Workflow pattern | Recommended K | Recommended tasks/teammate | Best for |
|---|---:|---:|---|
| Code review (security / performance / coverage) | 3 | 1 (one PR each) | Independent reviewers, no shared state |
| Multi-module refactor | 4–5 | 5–6 | Each owns separate files, no overwrites |
| Multi-hypothesis investigation | 5 | 3–4 | Adversarial debate (short rounds), debate-collapse-aware |
| Parallel feature dev | 3 | 5–6 | Each owns a module, integrates at end |

## Variants and ablations

- **Subagent-template teammate.** `.claude/agents/<name>.md` defines a reusable role; spawn by name to apply system prompt + tool restrictions. The closest Anthropic ships to MetaGPT-style typed roles ([91-metagpt-deep](91-metagpt-deep.md)).
- **Plan-approval gating.** `"Require plan approval before they make changes"` makes teammates produce a plan first, lead approves, then execution. Reduces wasted token spend on misaligned plans.
- **Custom hooks for quality gates.** Use `TaskCompleted` hooks to run lint/test/security-scan before marking complete.
- **MCP-equipped teammates.** All teammates inherit project's MCP servers; can be scoped via subagent definition.
- **Split-pane debug mode.** When a teammate stalls, click into its pane and prompt directly.

## Failure modes and limitations

- **7× token cost in plan mode** caps team size economically.
- **Permission inheritance is coarse.** Cannot give one teammate broader scope than another at spawn.
- **No nesting / leadership transfer.** Limits orchestrator-of-orchestrators patterns.
- **Task status lag** is a documented bug; not yet fixed.
- **No programmatic API** — teams are NL-orchestrated only; no SDK, no REST.
- **Display fragmentation** — split-pane only works with tmux/iTerm2; in-process display can be confusing past K=4.
- **Cross-teammate context bleed prevented but at the cost of forgetting** — teammates do not see the lead's prior conversation, which means re-explaining the task fully in the spawn prompt.
- **Concurrency on shared filesystem** — file-locking is documented but race conditions on the task list have been reported in early reports.
- **Plan-mode multiplier** — non-plan-mode teams cost less but lose the gate that prevents misaligned execution.

## When to use, when not

**Use Agent Teams** for naturally parallel work where the cost of K full sessions is justified by wall-clock savings — multi-reviewer code review, multi-hypothesis debugging, multi-module refactors with clear file ownership, parallel feature implementation across independent modules. The strongest case is when each teammate has a **disjoint file scope** so merge complexity is low.

**Do not use Agent Teams** for sequential reasoning chains (single-agent + tools is cheaper); for tightly-coupled tasks where merge complexity dominates parallelism gain; when the token budget cannot absorb 7× per query; when running on a session that needs `/resume` capability; when the workflow already has a deterministic orchestrator (subagents are simpler); when you need K > 5 (coordination overhead and token cost dominate).

## Implications for harness engineering

- **Lead-and-spokes is the consensus 2026 architecture.** [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md), [224-multi-agent-parallel-scaling](224-multi-agent-parallel-scaling.md) — your in-house agent runtime should support this pattern.
- **Shared task list as filesystem state.** [02-subagent-delegation](02-subagent-delegation.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md), [03-program-graph](../projects/polaris/docs/blocks/03-program-graph.md) — file-based task list with claim-locking is reusable infrastructure.
- **Mailbox messaging beats free-chat.** [98-diversity-collapse-mas](98-diversity-collapse-mas.md), [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md) — structured per-name messages > GroupChat; debate-collapse risk lower.
- **Worktree isolation per teammate.** [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md), [02-subagent-delegation](02-subagent-delegation.md) — each teammate's filesystem changes contained.
- **Subagent definitions as role templates.** [04-skills](04-skills.md), [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md), [91-metagpt-deep](91-metagpt-deep.md) — `.claude/agents/<role>.md` is the reusable role spec.
- **Permission Bridge gates spawn.** [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — `TEAM_SPAWN` action kind.
- **Hooks for team-event observability.** [05-hooks](05-hooks.md), [08-hooks-and-claim-gate](../projects/polaris/docs/blocks/08-hooks-and-claim-gate.md), [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — TeammateIdle / TaskCreated / TaskCompleted patterns.
- **Cost-router for team size.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md), [88-confidence-driven-router](88-confidence-driven-router.md) — 7× cap should fire bright-line escalation.
- **Cross-channel verifier in teams.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [19-verifier-cross-channel](../projects/polaris/docs/blocks/19-verifier-cross-channel.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md) — verifier teammate from a different model family.
- **Memory tiering across teams.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [187-multi-agent-shared-memory-landscape](187-multi-agent-shared-memory-landscape.md) — per-teammate context, lead-aggregated handoffs.
- **Daemon-driven team scheduler.** [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [p7-daemon](../projects/polaris/docs/research/p7-daemon.md) — long-running teams driven by the daemon's heartbeat.
- **MAST-aware logging.** [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md) — every team trace tagged for failure-mode replay.
- **The gap to A2A protocol.** [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md) — Agent Teams is single-runtime; cross-vendor inter-agent comms is A2A's domain.

**One-line takeaway for harness designers.** **Anthropic Agent Teams is the consensus 2026 multi-agent runtime architecture made concrete in Claude Code — lead-and-spokes hybrid with shared task list and structured per-name messaging, isolated full-Claude-Code teammates per worktree, file-based coordination state, no nesting — and it is the shape your in-house agent runtime should mirror, costing 7× tokens for the privilege of parallelism on naturally-parallel work only.**
