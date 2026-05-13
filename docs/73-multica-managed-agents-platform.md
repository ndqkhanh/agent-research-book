# 73 — multica-ai/multica: Open-Source Managed-Agents Platform for Vendor-Neutral Team Orchestration

**Definition.** Multica ([github.com/multica-ai/multica](https://github.com/multica-ai/multica), [multica.ai](https://multica.ai)) is an **open-source managed-agents platform** that treats coding agents as first-class teammates — assignable to issues, posting comments, updating statuses, reporting blockers, and compounding skills over time. The project is led by Jiayuan "Forrest" Chang (creator of [karpathy-skills](71-karpathy-skills-single-file-guardrails.md) — Multica is his follow-on at a platform scale) and has **18,350 stars** (April 2026). The tagline — *"Your next 10 hires won't be human"* — captures the positioning: instead of adding a ChatGPT integration to Jira, Multica rebuilds the issue-tracking + agent-management substrate with agents-as-teammates as the primary design constraint. Multica works across **eight agent harnesses**: Claude Code, Codex, OpenClaw, OpenCode, Hermes, Gemini, Pi, and Cursor Agent — positioning itself as the **vendor-neutral orchestration layer** above the individual coding-agent CLIs.

Architecture: Next.js 16 frontend + Go backend (Chi router, sqlc, gorilla/websocket) + PostgreSQL 17 with pgvector + a local daemon that executes agent CLIs on the user's machine (Cloud) or a self-hosted server (optional `--with-server` install via Docker). Deployment: cloud-first at [multica.ai/app](https://multica.ai/app) with optional self-hosting via `multica setup self-host`.

This is the **managed-agents platform** category that does not yet have a dominant incumbent as of April 2026. Multica is one of the strongest contenders alongside commercial offerings (Paperclip, CharmbraceletAI) and internal platforms at large engineering orgs.

## Problem it solves — the "agents as workflow" gap

Running a coding agent once is easy: `claude code`, type a prompt, watch it work. Running a coding agent *as a teammate* is much harder:

1. **No durable task ownership.** A one-shot CLI agent session is forgotten the moment it ends. If the agent did excellent work but you need to come back to it in 3 days, you re-explain from scratch.
2. **No multi-agent dispatch.** If you have five agents (one good at frontend, one at testing, one at infra), you have no way to route tasks among them.
3. **No team-facing interface.** Your human teammates can't see what the agent is doing, can't assign it work, can't comment on its PRs.
4. **No skill compounding.** When an agent solves a novel problem well (say, a tricky migration pattern), the solution is stuck in that one session. Nobody else on the team — human or agent — benefits.
5. **No progress tracking.** Agents can produce PRs but don't "report to standup" or show up on a board. Managers can't tell what fraction of the backlog is being worked by agents vs humans.
6. **Vendor lock-in.** Every agent CLI has its own configuration, invocation, and delivery surface. Building a team-level process around Claude Code specifically means you can't easily switch to Codex or Cursor Agent.

These pain points map 1:1 onto Multica's architecture:

| Pain point | Multica solution |
|---|---|
| No durable task ownership | Issues backed by PostgreSQL, persisted across sessions |
| No multi-agent dispatch | Agent profiles + runtime detection + board-level assignment |
| No team-facing interface | Next.js board, comments, status updates in a shared UI |
| No skill compounding | Reusable-skills system (team-wide, shared across agents) |
| No progress tracking | WebSocket real-time progress streaming + board view |
| Vendor lock-in | Eight agent CLIs supported; adapter pattern for new ones |

## Architecture — six layers

### 1. Next.js 16 Frontend (App Router)

A web app that developers and managers use to:
- Browse the issue board (kanban / list view).
- Assign issues to agents (from a dropdown of connected agents).
- Comment on issues (visible to both humans and agents).
- See real-time agent progress (WebSocket feeds).
- Configure workspaces, agents, runtimes.
- Review skills the team has accumulated.

Next.js 16 App Router is a 2026-current choice and aligns with the modern React Server Components model. The frontend is deployed at `multica.ai/app` in cloud mode; in self-hosted mode it ships as part of the Docker compose stack.

### 2. Go Backend (Chi router, sqlc, gorilla/websocket)

The backend provides:
- REST API (via Chi) for board, issues, workspaces, agents, runtimes.
- WebSocket subscriptions (via gorilla/websocket) for real-time progress streaming from daemons to the frontend.
- sqlc-generated type-safe database access layer over Postgres.
- Agent protocol — the backend dispatches tasks to daemons over long-polling or WebSocket, and receives streaming progress back.

**Why Go instead of Node.js/Python:** WebSocket fan-out with thousands of concurrent daemon connections per server is where Go's goroutines shine. The 2026 managed-agents space has rediscovered Go as the right backend language for agent-orchestration servers (similar choices appear in Kubernetes, HashiCorp's stack, and several competing agent platforms).

### 3. PostgreSQL 17 with pgvector

The single source of truth. Postgres stores:
- Workspaces, users, roles, permissions.
- Issues (title, description, status, assignee, labels, comments).
- Agents (name, runtime, provider, configuration).
- Runtimes (machine identity, available CLIs, status, heartbeat).
- Skills (reusable task templates, often embedded as pgvector for semantic retrieval).
- Task execution history (what agent ran what issue, outputs, failures).

**pgvector for skill retrieval.** This is important — when an agent picks up a new issue, Multica queries the skills table with pgvector semantic search to surface reusable skills that match the task. This is the team-level counterpart to the claude-mem semantic retrieval system ([72-claude-mem-persistent-memory-compression.md](72-claude-mem-persistent-memory-compression.md)) — memory and skills are the same retrieval substrate at different granularities.

### 4. Local Agent Daemon

A process running on the developer's machine (or on a self-hosted server) that:
- Authenticates to the Multica backend.
- Auto-detects available agent CLIs on the user's `PATH` (`claude`, `codex`, `openclaw`, `opencode`, `hermes`, `gemini`, `pi`, `cursor-agent`).
- Registers the machine as a **Runtime** (a compute environment capable of executing agent tasks).
- Polls the backend for assigned tasks and executes them by invoking the appropriate agent CLI.
- Streams progress back via WebSocket — stdout/stderr capture, file edits, PR creation events.
- Reports completion / failure / blocker states to the backend.

**The daemon is the trust boundary.** It runs on the user's hardware, uses the user's credentials, accesses the user's code. The backend never sees the code — only the task definitions, progress events, and metadata. This is the **zero-trust execution pattern** ([41-product-control-plane.md](41-product-control-plane.md) / [49-agents-of-chaos-red-teaming.md](49-agents-of-chaos-red-teaming.md)) applied to multi-user agent management.

### 5. Agent Abstraction Layer

The daemon translates the Multica task into the specific CLI's invocation format:
- Claude Code: `claude code "task description"` in the target directory.
- Codex: OpenAI Codex CLI invocation.
- OpenClaw: OpenClaw gateway API call.
- OpenCode: self-hosted OpenCode instance.
- Hermes: Nous Research's Hermes Agent API ([55-hermes-agent-self-improving.md](55-hermes-agent-self-improving.md)).
- Gemini: Google Gemini CLI.
- Pi: Pi (assumed an LLM agent CLI).
- Cursor Agent: Cursor's CLI / background-agent mode.

This abstraction is the hardest engineering challenge. Each agent CLI has different:
- Configuration paths (`~/.claude/`, `~/.gemini/`, `~/.cursor/`, etc.).
- Input formats (plain text vs structured JSON vs MCP).
- Output formats (stdout parsing vs streaming JSON).
- Failure semantics (exit codes, error formats, recovery conventions).
- Tool/capability surfaces (what tools each can call natively).

Multica's adapter layer normalizes these differences into a common agent-execution interface. The maintenance burden scales with the number of agents supported — eight is already a lot, and agent-CLI APIs are still churning rapidly.

### 6. Skills System

Every solution can become a reusable skill. Skills are:
- **Persisted in Postgres**, searchable via pgvector semantic retrieval.
- **Team-shared** (all workspace members' agents can use them).
- **Compound** — when an agent learns a new migration pattern, the next agent facing a similar task has the skill available.
- **Versioned** (skills evolve as tasks change).
- **Composable** — a complex task can pull multiple skills simultaneously.

This is the platform-level counterpart to:
- [04-skills.md](04-skills.md) — Claude's per-project SKILL.md.
- [19-voyager-skill-libraries.md](19-voyager-skill-libraries.md) — Voyager's research-scale skill library.
- [65-karpathy-new-programming.md](65-karpathy-new-programming.md) — Karpathy's "new programming" thesis (skills as a new artifact type).

Multica makes skills a **team-level asset** rather than a per-developer or per-agent artifact. This is the step that unlocks "skills compound" as a team strategy.

## Feature-by-feature tour

### Feature 1 — Agents as Teammates

Agents in Multica have:
- **Profiles** (name, avatar, bio, "skillset", provider).
- **Assignments** on the board — same drag-and-drop affordance as assigning to a human.
- **Comment streams** — agents post comments on issues as they work, and humans can @-mention them.
- **Blocker reporting** — if an agent gets stuck (e.g., needs clarification, is missing credentials), it posts a blocker on the issue.
- **Proactive issue creation** — agents can create issues themselves (e.g., for refactoring debt they noticed while working on a related task).

The design philosophy inverts the usual pattern: most agent-orchestration tools treat agents as *backend workers*. Multica treats them as *UI citizens* — they show up on the board, they comment, they get @-mentioned, they file issues. This reframing aligns with [65-karpathy-new-programming.md](65-karpathy-new-programming.md)'s thesis that agents are "a new kind of programmer."

### Feature 2 — Autonomous Execution

Full task lifecycle management:
- **enqueue** — user creates issue, assigns to agent.
- **claim** — daemon picks up the task.
- **start** — agent begins execution.
- **progress** — WebSocket streaming to UI (every file edit, every tool call).
- **complete / fail / block** — final state with artifact links.

"Set it and forget it" — users don't babysit the run. If the agent needs help, it asks (via blocker). If it finishes, the board updates automatically. This is the loop-level contract described in [01-agent-loop-architecture.md](01-agent-loop-architecture.md) extended to multi-task team scale.

### Feature 3 — Reusable Skills

The skills system is the most differentiating feature. The core claim: *"every solution becomes a reusable skill for the whole team"*. Mechanism:

1. Agent completes a novel task (say, a complex database migration).
2. The session output is summarized into a skill template.
3. The skill is embedded (pgvector) and tagged.
4. Next time any agent on the team faces a similar task, the skill surfaces as a candidate approach.
5. The skill can be used as-is, or refined through use (next version updates the skill).

This is the **organizational memory** pattern — skills accumulate across the team, so the team's agent-productivity grows superlinearly with time. A single developer with Claude Code has access only to their own skills; a 20-person team on Multica has access to the union of everyone's discovered solutions. The network effect is real and is probably the strongest economic moat in the managed-agents category.

### Feature 4 — Unified Runtimes

A Runtime is a **compute environment that can execute agent tasks** — either:
- A local machine (via the daemon).
- A cloud instance (via the daemon running in the cloud).

The **Runtimes dashboard** shows:
- Which machines are connected.
- Which agent CLIs are available on each.
- Real-time status (idle / running / errored).
- Resource utilization.

**Routing logic.** When a task is assigned to an agent, Multica checks which runtimes can execute that agent's CLI. If only one runtime has `claude` installed, the task goes there. If multiple runtimes have it, load-balancing or preference-based routing applies.

This abstraction — the runtime as a separate concept from the agent — is key. It means a single team can have:
- Alice's laptop (Claude Code, Cursor Agent).
- Bob's laptop (Claude Code, Codex, Gemini).
- A cloud VM (Hermes, OpenCode).
- A beefy shared workstation (all eight CLIs).

Tasks are dispatched to the runtime that can execute them, not tied to a specific developer. This is essentially a **fleet scheduler for coding agents**.

### Feature 5 — Multi-Workspace

Workspace-level isolation. Each workspace has:
- Its own agents (with provider, name, skillset).
- Its own issues (with statuses, labels, assignees).
- Its own settings (security policies, runtime allowances).
- Its own skills library (isolated from other workspaces).

A developer can be a member of multiple workspaces (e.g., one for personal projects, one for their company). Agents are workspace-scoped, so your company's agent can't access your personal projects and vice versa.

This is the **tenancy model** that distinguishes platform tools from tools for solo developers. For enterprise adoption, workspace isolation is table stakes.

## Multica vs Paperclip — the 2026 managed-agents category

The Multica README includes an explicit comparison with **Paperclip**, another managed-agents product:

| | Multica | Paperclip |
|---|---|---|
| **Focus** | Team AI agent collaboration platform | Solo AI agent company simulator |
| **User model** | Multi-user teams with roles & permissions | Single board operator |
| **Agent interaction** | Issues + Chat conversations | Issues + Heartbeat |
| **Deployment** | Cloud-first | Local-first |
| **Management depth** | Lightweight (Issues / Projects / Labels) | Heavy governance (Org chart / Approvals / Budgets) |
| **Extensibility** | Skills system | Skills + Plugin system |

**TL;DR** from the Multica README: *"Multica is built for teams that want to collaborate with AI agents on real projects together."*

This framing reveals a design *choice* — Multica prioritizes *team collaboration with agents* over *solo agent management*. Paperclip goes the other direction — treat the solo operator as a CEO running an "AI agent company" with heavy governance, org charts, approvals, budgets. Both are defensible. Neither is strictly correct.

**The 2026 managed-agents category is bifurcating along this exact axis:**
- **Team-first platforms** (Multica, CharmbraceletAI, likely others): focus on multi-human + multi-agent collaboration.
- **Solo-operator platforms** (Paperclip): focus on one human managing an "AI company" of many specialized agents.

Which approach wins depends on whether the target user is "a developer team adding agents" (Multica's thesis) or "a solo developer building with many specialized agents" (Paperclip's thesis). Both markets are real in April 2026; it's too early to call a winner.

## Installation and setup — one-command friendliness

Multica's setup flow is aggressively streamlined:

```bash
brew install multica-ai/tap/multica    # Install CLI
multica setup                           # Configure + auth + start daemon
```

Two commands, and the user is assignable-to-agent. The `multica setup` command handles:
1. Browser-based OAuth to authenticate.
2. Workspace selection or creation.
3. Daemon installation and startup.
4. Runtime registration.
5. CLI auto-detection.

**For self-hosting** (Docker required):

```bash
curl -fsSL https://raw.githubusercontent.com/multica-ai/multica/main/scripts/install.sh | bash -s -- --with-server
multica setup self-host
```

The `--with-server` flag pulls the full backend + Postgres + daemon stack via Docker Compose. Users get a locally hosted Multica in a few minutes.

**Windows support via PowerShell:**

```powershell
irm https://raw.githubusercontent.com/multica-ai/multica/main/scripts/install.ps1 | iex
```

Cross-platform one-liner installs — mac/Linux Homebrew, mac/Linux shell script, Windows PowerShell. This is the *minimum* acceptable 2026 install UX for tools targeting broad developer adoption.

## CLI Surface

```
multica login                 # Browser-based OAuth
multica daemon start          # Start local runtime
multica daemon status         # Check daemon health
multica setup                 # Cloud one-command setup
multica setup self-host       # Self-hosted setup
multica issue list            # Board operations
multica issue create          # Create issue
multica update                # Self-update
```

The CLI is the "everything you can do in the web UI, from your terminal" surface. The design mirrors the `gh` CLI (GitHub's) and `linear` CLI (Linear's) — these tools have normalized the expectation that issue-tracker UIs must have CLI equivalents.

## Unique design properties

### A. Vendor neutrality across eight agent CLIs

Multica is the only managed-agents platform in April 2026 that supports all eight of: Claude Code, Codex, OpenClaw, OpenCode, Hermes, Gemini, Pi, Cursor Agent. Competitors usually specialize (e.g., a Claude-Code-only platform, or a Cursor-only platform). Multica's thesis: developers will increasingly run multiple agent CLIs side-by-side, and teams will have mixed fleets. The orchestration layer should be vendor-neutral.

**Cost:** the adapter layer for eight CLIs is significant engineering overhead. Each CLI has its own output format, config location, error semantics, and tool surface. Maintaining all eight as they churn is a full-time effort.

**Benefit:** if the bet pays off — mixed-CLI teams being the dominant mode — Multica becomes indispensable infrastructure. Nobody wants to rebuild their team's issue tracker every time they switch agent CLIs.

### B. Skills as team-shared assets

The skills system is not a per-developer artifact or a per-agent configuration; it is a **workspace-level pool** that all agents in the workspace can draw from. This is the crucial architectural choice that unlocks "skills compound" as a team-level strategy.

### C. Issues as the universal work unit

Multica chose Issues (not Projects, not Workflows, not Pipelines) as the primary work unit. This aligns with how engineering teams already think — most teams already organize work around issues in Jira / GitHub / Linear. Multica inherits the mental model and adds agent-assignability on top, rather than inventing a new work-unit type.

### D. Real-time WebSocket progress streaming

When an agent starts work, its file edits, tool calls, and stdout stream to the web UI in real time. This is the **observability affordance** ([24-observability-tracing.md](24-observability-tracing.md)) integrated into the issue-tracker UI — you don't need a separate tracing tool to see what the agent is doing; you watch it live on the issue.

### E. Cloud-first with optional self-hosting

The default deployment is cloud (multica.ai/app). Self-hosting is explicitly supported via Docker (`--with-server` + `multica setup self-host`) for users who need data sovereignty, air-gapped deployment, or custom policies. This is the **dual-deployment model** increasingly common in developer tools (Linear, PostHog, Retool all do similar).

## Failure modes and open questions

1. **Eight-CLI adapter churn.** As each underlying CLI releases breaking changes, Multica must update its adapter. If one CLI rev breaks the daemon, users on that CLI lose runtime connectivity until Multica releases a fix. This is an inherent risk of the vendor-neutral bet.

2. **Skills quality / curation gap.** The auto-generated skill system assumes that "every solution becomes a reusable skill" — but not every solution deserves to be. Without curation, the skills pool becomes noisy, and retrieval quality drops. Multica's current approach is unclear; this is likely an area of active product development.

3. **Multi-agent race conditions.** If two agents are simultaneously assigned to related issues and both modify overlapping code, merge conflicts or logic bugs can emerge. Multica's coordination model (beyond the one-agent-per-issue default) is not described in the public README.

4. **Trust boundary between cloud and daemon.** The cloud backend assigns tasks; the daemon executes them. If the daemon is compromised, the attacker can execute arbitrary code via agent CLIs under the developer's credentials. This is fundamentally the same risk profile as running any CI/CD runner on a developer machine — but explicit threat modeling is not visible in the public docs.

5. **Budget / cost controls.** Unlike Paperclip (which emphasizes heavy governance with budgets and approvals), Multica's lightweight model does not surface budget controls by default. For enterprises, this is a gap — a misbehaving agent can burn through substantial LLM credits before anyone notices.

6. **Workspace-level isolation strength.** The README claims workspace isolation, but the daemon running on a developer's machine has access to all workspaces the developer is a member of. A malicious issue in Workspace A could theoretically exfiltrate code from a project in Workspace B via the shared daemon. The security model needs careful threat modeling that is not yet public.

7. **pgvector skill retrieval latency.** As the skills pool grows, pgvector lookup latency can degrade. Postgres 17's pgvector is good but not the fastest vector DB available. At large team scales, this may bottleneck and require migration to dedicated vector stores (similar to claude-mem's use of Chroma).

8. **Star growth vs feature maturity gap.** 18,350 stars in ~6 months is rapid adoption; the feature set is still relatively early. Users may encounter rough edges in the self-hosting path, the adapter layer, or the skills system.

## Patterns for harness engineers to steal

1. **Cloud-first with opt-in self-host.** The dual-deployment pattern (Docker for self-host, default SaaS for cloud) captures both hobbyist and enterprise markets with the same codebase. Pick this pattern unless you have a strong reason to prefer one or the other exclusively.

2. **Vendor neutrality as a bet.** If you are building infrastructure above agent CLIs, supporting all major CLIs costs more engineering but maximizes optionality for the user base. Multica's bet is that mixed-CLI teams are the future; if you agree, you invest in the adapter layer.

3. **Issues as the universal work unit.** Don't invent new work-unit types. Issues are familiar and universally understood. The interesting design work is in *how agents interact with issues*, not in replacing the issue model.

4. **Real-time progress as UI affordance.** WebSocket streaming of agent progress into the issue view is the right default. Asynchronous polling feels broken; live-streaming feels alive.

5. **pgvector for team-level skills retrieval.** Postgres 17 + pgvector is the right 2026 default for < 100K-vector workloads. You get transactional consistency for free, and you don't need a separate vector-DB service. Migrate to dedicated vector stores only when scale requires.

6. **Go for the orchestration backend.** WebSocket fan-out at scale is Go's sweet spot. Node.js or Python work for smaller deployments; Go is the right choice when you expect thousands of concurrent daemons per server.

7. **CLI + web UI at parity.** Users hate having to choose between "use the web UI" and "use the CLI." Build both, keep them at parity, and you capture both the keyboard-first and the mouse-first audiences.

8. **One-command setup or you die.** `multica setup` does everything. If your onboarding requires more than one command, a significant fraction of users will bounce. Fewer commands → more adoption.

9. **Workspace isolation from day 1.** Even if your target user is a solo developer, build multi-workspace from day 1. Adding tenancy later is a rewrite.

10. **Agents-as-UI-citizens reframing.** The "agents post comments, get @-mentioned, file issues" framing is what makes Multica distinctive. If you are building an agent orchestration product, don't hide agents in the backend — surface them in the UI alongside humans.

## Relationship to the corpus

Multica sits at several corpus-level intersections:

- **Multi-agent orchestration** ([02-subagent-delegation.md](02-subagent-delegation.md)). Multica is a team-level multi-agent system — not multi-agent within a single task, but multi-agent across a team's backlog.
- **Skill libraries** ([04-skills.md](04-skills.md), [19-voyager-skill-libraries.md](19-voyager-skill-libraries.md)). Multica's team-shared skill system is the platform-scale version of Claude Skills and Voyager skill libraries.
- **Control plane** ([41-product-control-plane.md](41-product-control-plane.md)). Multica is essentially a control plane for coding-agent fleets.
- **Karpathy's new programming** ([65-karpathy-new-programming.md](65-karpathy-new-programming.md)). Multica materially advances the thesis that agents are the new workforce — by building the HR system for that workforce.
- **Observability** ([24-observability-tracing.md](24-observability-tracing.md)). Real-time progress streaming into the issue view is observability built into the work-tracking layer.
- **Community Claude Code ecosystem** ([62-everything-claude-code.md](62-everything-claude-code.md)). Multica integrates Claude Code as one of eight supported CLIs.
- **Karpathy-skills sister project** ([71-karpathy-skills-single-file-guardrails.md](71-karpathy-skills-single-file-guardrails.md)). The progression from a 60-line CLAUDE.md to a full orchestration platform is Forrest Chang's signature move. Both are valuable; together they bracket the "minimum viable" and "maximum viable" ends of the Claude Code tooling spectrum.
- **SEA self-evolving agents** ([36-autogenesis-self-evolving-agents.md](36-autogenesis-self-evolving-agents.md), [55-hermes-agent-self-improving.md](55-hermes-agent-self-improving.md)). Multica's skill-compounding mechanism is a lightweight form of self-evolution — not at the model level, but at the organizational level.
- **Meta-harness landscape** ([66-meta-harness-landscape.md](66-meta-harness-landscape.md)). Multica is one of the clearest *meta-harnesses* in the corpus — it sits above eight underlying harnesses and normalizes them.

## References — primary artifacts

- **Main repo.** [github.com/multica-ai/multica](https://github.com/multica-ai/multica) — 18,350 stars (April 2026).
- **Website.** [multica.ai](https://multica.ai).
- **Cloud app.** [multica.ai/app](https://multica.ai/app).
- **Self-hosting guide.** [SELF_HOSTING.md](https://github.com/multica-ai/multica/blob/main/SELF_HOSTING.md).
- **CLI and daemon guide.** [CLI_AND_DAEMON.md](https://github.com/multica-ai/multica/blob/main/CLI_AND_DAEMON.md).
- **Contributing guide.** [CONTRIBUTING.md](https://github.com/multica-ai/multica/blob/main/CONTRIBUTING.md).
- **X.** [@MulticaAI](https://x.com/MulticaAI).
- **Author.** Jiayuan Chang ([@jiayuan_jy](https://x.com/jiayuan_jy)).
- **Sister project.** [github.com/forrestchang/andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills) — see [doc 71](71-karpathy-skills-single-file-guardrails.md).

## Cross-references in this corpus

- [01-agent-loop-architecture.md](01-agent-loop-architecture.md) — loop-level contract extended to team scale.
- [02-subagent-delegation.md](02-subagent-delegation.md) — multi-agent at the task level vs Multica at the team level.
- [04-skills.md](04-skills.md) — per-project Claude Skills.
- [07-model-context-protocol.md](07-model-context-protocol.md) — MCP-style interoperability across eight CLIs.
- [19-voyager-skill-libraries.md](19-voyager-skill-libraries.md) — research-scale skill library.
- [24-observability-tracing.md](24-observability-tracing.md) — real-time progress streaming.
- [36-autogenesis-self-evolving-agents.md](36-autogenesis-self-evolving-agents.md) — SEA parallel to skill compounding.
- [41-product-control-plane.md](41-product-control-plane.md) — control-plane pattern.
- [55-hermes-agent-self-improving.md](55-hermes-agent-self-improving.md) — Hermes as one of Multica's supported providers.
- [62-everything-claude-code.md](62-everything-claude-code.md) — Claude Code ecosystem Multica integrates.
- [65-karpathy-new-programming.md](65-karpathy-new-programming.md) — the "new workforce" thesis Multica operationalizes.
- [66-meta-harness-landscape.md](66-meta-harness-landscape.md) — Multica as an exemplar meta-harness.
- [71-karpathy-skills-single-file-guardrails.md](71-karpathy-skills-single-file-guardrails.md) — sister project in single-file form.
- [72-claude-mem-persistent-memory-compression.md](72-claude-mem-persistent-memory-compression.md) — memory-layer analog to Multica's skills layer.
