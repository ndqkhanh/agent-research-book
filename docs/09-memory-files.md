# 09 — Memory Files

**Definition.** Memory files are durable, human-readable files the agent writes to and reads from to persist knowledge across turns and sessions. Unlike transcript context (ephemeral) or compaction summaries (session-scoped), memory files live on disk and are curated over time. Examples: `CLAUDE.md`, `AGENTS.md`, `memory/user_role.md`, `memory/feedback_testing.md`.

## Problem it solves

Context is forgotten after compaction. Compaction summaries are forgotten after a session ends. If the agent learned something durable — "the user prefers integration tests against a real DB, not mocks" — that knowledge has to live somewhere more persistent than a transcript. Fine-tuning is heavyweight and slow. Vector memory is opaque and hard to edit. Plain files are grep-able, diff-able, version-controllable, human-auditable, and instantly editable by the user. That combination makes them the default persistence mechanism for modern harnesses.

Memory files also give the user direct control: they can open a file and redact, edit, or forget things. That transparency is crucial for trust, especially as agents accumulate knowledge about the user.

## Mechanism

Three layers are common:

1. **Project-level.** `CLAUDE.md` at the repo root. Loaded at session start. Contains project conventions, build/test commands, architectural notes. Shared via version control.
2. **User-level.** `~/.claude/CLAUDE.md` and `~/.claude/memory/*.md`. Personal preferences, role, frequently-given corrections, cross-project lessons. Not shared.
3. **Session-level.** Scratchpads, plan files, todo lists — see [12-todo-scratchpad-state.md](12-todo-scratchpad-state.md). Ephemeral.

An index file (`MEMORY.md`) points at individual memory files so the full set isn't loaded eagerly. Each memory file has YAML frontmatter (name, description, type) and a body structured by type:

```markdown
---
name: testing_preference
description: User prefers integration tests against real DB, not mocks
type: feedback
---
Integration tests must hit a real database; do not mock the DB layer.

**Why:** prior Q4 2024 incident where mocked tests passed but the prod
migration failed — the mock/prod divergence masked a broken schema change.

**How to apply:** when adding tests for DB-touching code, use the test
container; do not reach for `jest.mock`/`unittest.mock` on DB calls.
```

Standard types (per the harness's built-in memory system):

- **user** — who the user is, what they do, what they know. Informs *how* to explain things.
- **feedback** — corrections and validated choices. Informs *what* to do or avoid.
- **project** — current initiatives, deadlines, stakeholder facts. Informs *context of the work*.
- **reference** — pointers to external systems (Linear project names, dashboards, runbooks).

Retrieval discipline: load the index eagerly, load individual files lazily when a turn touches their topic. Before acting on a memory, verify it's still true (file paths may have moved, people may have changed roles).

## Concrete pattern

A user's memory layout after a few weeks:

```
~/.claude/memory/
├── MEMORY.md                           # one-line-per-file index
├── user_role.md                        # "backend eng focused on auth"
├── feedback_testing.md                 # integration tests over mocks
├── feedback_terse_responses.md         # "don't summarize, I read diffs"
├── project_auth_migration.md           # current initiative + why
├── project_merge_freeze_2026-03-05.md  # absolute dates, not "Thursday"
└── reference_linear_INGEST.md          # bug tracker pointer
```

`MEMORY.md` is the index, one line per file:

```markdown
- [testing_preference](feedback_testing.md) — integration tests, never mocks.
- [terse_responses](feedback_terse_responses.md) — no trailing summaries.
- [auth_migration](project_auth_migration.md) — compliance-driven rewrite.
```

Write rule of thumb: **don't store what's already derivable from the code** (architecture, file paths, conventions), the git history, or CLAUDE.md. Memory is for things the code doesn't tell you — user preferences, why a decision was made, validated judgments, external system pointers.

## Variants & related techniques

- **Vector memory** embeds past turns or documents and retrieves by similarity. Better for recall of specific quotes; worse for user-editable, overlapping, or negation-heavy facts ("do not do X" retrieves poorly).
- **Reflexion** ([14-reflexion.md](14-reflexion.md)) appends verbal reflections to an episodic memory — a special-case memory file written after each episode.
- **Agent Memory (Cloudflare, Memobase, Letta/MemGPT)** — external memory services with APIs. Good when one memory store backs many agents; adds operational surface.
- **CLAUDE.md vs. MEMORY.md.** CLAUDE.md is checked-in project context; MEMORY.md is a personal, cross-session index. Both can coexist.
- **Frontmatter as structured facets.** Frontmatter lets tooling filter memories by type/scope without parsing bodies.

## Failure modes & anti-patterns

- **Memory bloat.** The index grows to hundreds of lines; relevant memories can't be found. Fix: cap index size, archive stale memories, consolidate overlapping ones.
- **Stale memories outlive their truth.** A project memory says "shipping on March 5"; it's now June. Fix: convert relative dates to absolute dates at write time; periodically sweep for stale facts; trust current observation over old memory when they conflict.
- **Storing what the code says.** "This repo uses FastAPI." The repo directory tells us that. Memory should only hold the non-obvious.
- **Judgment memories.** "The user is bad at frontend." Not useful and corrosive. Memory rule: write what helps future collaboration, not evaluations of the user.
- **Auto-saving every utterance.** If the agent saves on every prompt, signal drowns in noise. Save on *surprise* — corrections, validated non-obvious choices, explicit requests.
- **Memory without verification.** The agent recommends a file that used to exist. Rule: before acting on a memory that names a specific path/function/flag, verify it still exists.

## When to use (and when not to)

**Use** memory files for:

- User preferences the agent should honor across sessions.
- Feedback (corrections and validated choices) with the *why* preserved.
- Project context that won't last (deadlines, initiatives, why-are-we-doing-this).
- Pointers to external systems (Linear, Grafana, Slack channels).

**Don't** use memory files for:

- Code conventions — the code is the source of truth; add CLAUDE.md if something's genuinely hidden.
- Debugging solutions — they belong in commit messages.
- Current-task scratch state — use a todo/plan file; it's ephemeral on purpose.
- Secrets — memory files are plaintext on disk; put secrets in a secrets manager.

## References

- Claude Code docs, Memory — <https://docs.claude.com/en/docs/claude-code/memory>
- Anthropic Engineering, "Effective context engineering for AI agents" — <https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents>
- Cloudflare, "Agents that remember: introducing Agent Memory" — <https://blog.cloudflare.com/introducing-agent-memory/>
- "Memory in the Age of AI Agents: A Survey", arXiv:2512.13564 — <https://arxiv.org/abs/2512.13564>
- Letta / MemGPT, "MemGPT: LLMs as Operating Systems" — <https://memgpt.ai/>
