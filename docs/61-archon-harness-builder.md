# 61 — Archon: The Open-Source Harness Builder for AI Coding

**Definition.** Archon (Cole Medin / `coleam00`, complete v0.3 rewrite April 7 2026; [github.com/coleam00/Archon](https://github.com/coleam00/Archon), docs at [archon.diy](https://archon.diy)) is a **YAML-first workflow engine for AI coding agents** that bills itself as "the first open-source harness builder for AI coding." Rather than shipping yet another hard-coded agent, Archon hands the developer the *harness* — DAG nodes, worktree isolation, loops, approval gates, tool restrictions, structured outputs, retries, MCP, skills, hooks — and lets them compose deterministic workflows that wrap Claude Code, Codex, or a community provider. MIT-licensed; TypeScript on Bun; SQLite or Postgres. Star count on 2026-04-20: ~19k (the first v1 Python incarnation had pushed it past 17k before the April-7 rewrite; trajectory since has been strongly positive). Latest release at research time: **v0.3.6 (2026-04-12)**.

## 1. What Archon is

Archon inverts the usual open-source-agent framing. LangChain Deep Agents ([42](42-langchain-deep-agents.md)), SemaClaw ([54](54-semaclaw-general-purpose-agent.md)), and Hermes ([55](55-hermes-agent-self-improving.md)) each ship an opinionated **agent**; Archon ships the **scaffold you wrap around someone else's agent**. The mental model the README offers is explicit and memorable: *"Like what Dockerfiles did for infrastructure and GitHub Actions did for CI/CD — Archon does for AI coding workflows."* The unit of reuse is a `.archon/workflows/*.yaml` file and its companion markdown commands in `.archon/commands/`; both are version-controlled alongside the repo they serve.

The original Archon (archived on branch `archive/v1-task-management-rag`, see [issue #957](https://github.com/coleam00/Archon/issues/957)) was a Python-based "agent that builds agents" plus a RAG/task-management stack. The April 2026 rewrite is a clean break — no migration path — and it reflects a stronger thesis: the hard problem in agentic coding is not *generating* code but *making the process around code-generation repeatable*. A harness builder, not an agent. The v0.3 Archon reports dramatic PR-acceptance-rate jumps (the [AgentConn review](https://agentconn.com/blog/archon-open-source-harness-builder-ai-coding-deterministic-review/) cites a 6.7% → ~70% lift when wrapping the same LLM in a structured harness, though reproduction conditions are not specified — treat as illustrative, unverified).

**Meta at time of writing (2026-04-20, via WebFetch):**
- Stars: ~19,000 (unverified to the individual; reported as 17,906 + daily growth of +452 around April 11; trending #2 on GitHub that week)
- License: MIT
- Language: TypeScript 98%
- Releases: 8 tagged (v0.1.0 2025-12-08 → v0.3.6 2026-04-12)
- Forks: ~2,900 (unverified)
- Contributor count: unverified

## 2. Architecture

The system is four loose layers. The README diagram reads top-to-bottom:

```
Platform Adapters
  Web UI | CLI | Telegram | Slack | Discord | GitHub webhooks
               │
               ▼
Orchestrator  (message routing, context, session continuity)
               │
     ┌─────────┼──────────────────────┐
     ▼         ▼                      ▼
Command    Workflow            AI Assistant Clients
Handler    Executor            (Claude Code / Codex / Pi-provider ~20 LLMs)
(Slash)    (YAML → DAG)
               │
               ▼
       SQLite / PostgreSQL
    (7 tables: codebases, conversations, sessions,
     workflow runs, isolation environments, messages,
     workflow events)
```

**Data flow for a single workflow invocation.** User submits a request from any adapter; the orchestrator routes it to a command handler or directly to the workflow executor. The executor parses the YAML, constructs a DAG, and begins execution. Each node is either (a) AI-driven — spawned as a Claude/Codex subprocess on a fresh or shared session — or (b) deterministic — bash, script, approval, or control-flow. Every node runs inside a **git worktree** so parallel runs do not collide. Node outputs are persisted; downstream prompts reference them by `$nodeId.output` substitution. Node lifecycle events (`node_started`, `node_completed`, `node_failed`) are written to the DB, enabling the "auto-resume on restart" property (no `--resume` flag).

**How agents are constructed.** There is no "Agent" class in Archon's sense. An agent is *a composed object* — a YAML DAG plus a set of markdown commands plus (optionally) a set of MCP servers, skills, and hooks attached per-node. Archon orchestrates *other people's agents* (Claude Code, Codex, Pi-community providers) and exposes only the *harness-level* primitives above them. This is the key architectural inversion versus Deep Agents / SemaClaw / Hermes: those projects define the agent loop; Archon *wraps* agent loops.

**Monorepo layout** (from GitHub tree): `packages/` (core engine, CLI, web UI, auth-service, paths), `.archon/` (bundled default workflows and commands), `.github/` (CI `test.yml`), `migrations/` (SQL schema), `deploy/` (Docker, cloud VPS docs), `homebrew/`, `scripts/`. Package manager: **Bun**. Web server: **Hono** (migrated in v0.2.1). Structured logging: **Pino** (added v0.2.5).

## 3. Harness primitives exposed

This is the section that matters most for this corpus, because Archon is the first open-source project we have seen that ships *nearly the full set* of primitives in explicit, declarative form. Mapping onto existing docs:

| Primitive | Archon surface | Corpus ref |
|-----------|----------------|------------|
| Agent loop | External (Claude/Codex/Pi); Archon composes around it | [01](01-agent-loop-architecture.md) |
| Subagent delegation | Inline sub-agent definitions on DAG nodes (v0.3, `[Unreleased]`); each AI node can spawn a scoped sub-call | [02](02-subagent-delegation.md) |
| Plan mode | Convention: `plan` → `implement` → `validate` node chain; no runtime mode, but structural | [03](03-plan-mode.md) |
| **Skills** | Per-node `skills: [remotion-best-practices, ...]` pre-loading; "skills system overhaul with specialized cookbooks" in v0.2.8 | [04](04-skills.md), [19](19-voyager-skill-libraries.md) |
| **Hooks** | Per-node `hooks: { on_tool_call: ... }` SDK callbacks; `.husky/` for git hooks | [05](05-hooks.md) |
| **Permissions** | Per-node `allowed_tools: []` / `denied_tools: []`; `sandbox: {network, filesystem}` at OS level; `--allow-env-keys` consent flag | [06](06-permission-modes.md) |
| **MCP** | Per-node `mcp: .archon/mcp/servers.json`; attachable in isolation | [07](07-model-context-protocol.md) |
| Context compaction | `context: fresh | shared` per node — explicit, not heuristic | [08](08-context-compaction.md) |
| Memory files | Artifact chain: *"the artifact you produce IS the specification for the next step"* — files are the only cross-node memory | [09](09-memory-files.md) |
| Multi-session continuity | SQLite-persisted sessions; auto-resume of failed runs | [10](10-multi-session-continuity.md) |
| Verifier / evaluator loops | `loop: { until: COMPLETE, max_iterations: 15 }`; `archon-test-loop-dag` default workflow | [11](11-verifier-evaluator-loops.md) |
| Todo scratchpad | Artifact files; commands follow LOAD→EXPLORE→ANALYZE→GENERATE→VALIDATE→COMMIT→REPORT phases | [12](12-todo-scratchpad-state.md) |
| Human-in-the-loop | First-class `approval` node with `capture_response`, `on_reject.prompt`, `max_attempts` | [23](23-human-in-the-loop.md) |
| Observability | Pino structured logs; every DAG step emits DB events; "CLI-Web observability overhaul" v0.2.10 | [24](24-observability-tracing.md) |
| Control plane | `archon serve` web UI with dashboard, DAG graph viewer, execution timeline | [41](41-product-control-plane.md) |

**The six DAG node types** (mutually exclusive per node):

1. `command` — execute a markdown-file command from `.archon/commands/`
2. `prompt` — inline AI prompt
3. `bash` — shell; stdout becomes `$nodeId.output`
4. `loop` — iterate AI prompt until a sentinel (`<promise>COMPLETE</promise>` → `until: COMPLETE`) or `max_iterations`
5. `approval` — pause for human decision
6. `cancel` — terminate with reason

A `script` node type (TypeScript/Python) was added in v0.3.3, making seven.

**Cross-cutting fields on every node:** `id`, `depends_on[]`, `when: "$other.output == 'X'"`, `trigger_rule: all_success | one_success | none_failed_min_one_success | all_done`, `context: fresh|shared`, `output_format: {JSON Schema}`, `retry: {max_attempts, delay_ms, on_error: transient|all}`, plus Claude-specific `effort`, `thinking`, `maxBudgetUsd`, `systemPrompt`, `fallbackModel`, `betas`, `sandbox`, `skills`, `mcp`, `hooks`.

**Structured output as control-flow primitive.** `output_format` enforces a JSON Schema on a node's reply; downstream `when` conditions can then test `$classify.output.type == 'BUG'` with confidence. This is the same pattern as LLM-as-judge ([21](21-llm-as-judge-trajectory-eval.md)) but surfaced as a routing primitive rather than a scoring one.

**Isolation.** Each run occupies a dedicated git worktree, so parallel workflows — even on the same branch — do not stomp each other. This is a substantial advance over any in-tree MVP's convention-based sandboxing.

**Resume semantics.** DB-logged `node_completed` events are the ground truth; on server restart, in-flight runs are marked `failed`, and the next invocation loads the event log, skips completed nodes, and continues. No `--resume` flag required. This is production-quality checkpointing exposed as a harness feature, not a runtime detail.

## 4. Novel contributions

**What Archon invents.** The combination below is unique to Archon among open-source projects surveyed:

1. **Workflow-as-artifact.** YAML DAGs that sit next to code, get reviewed in PRs, and *are* the harness. MetaGPT ([20](20-metagpt-role-based-workflows.md)) codifies role workflows but hard-codes them; Archon makes them user-authored.
2. **Per-node declarative harness configuration.** `allowed_tools`, `skills`, `mcp`, `hooks`, `sandbox`, `thinking`, `maxBudgetUsd` — *per node*, not per agent. This is finer-grained than anything in Deep Agents, OpenClaw, Hermes, or SemaClaw.
3. **`when` + `trigger_rule` + `output_format` as a three-piece routing primitive.** Together these let a workflow classify, branch, and rejoin without writing imperative glue. The fail-closed semantics (invalid expression → `false`) are a deliberately conservative choice that most homebrew workflow engines get wrong.
4. **Worktree-level isolation as default.** Not opt-in, not convention — the engine refuses to let two runs share a tree.
5. **Platform-agnostic surface.** Same YAML runs from CLI, Web UI, Slack, Telegram, Discord, or a GitHub webhook. The [12 harness patterns](43-twelve-harness-patterns.md) doc notes platform portability as a goal; Archon actually ships it.
6. **DB-driven auto-resume.** No flag, no config — the default behavior is "if you invoke the same workflow on the same working path after a failure, it continues where it stopped."

**What Archon reuses.**

- **LangChain / LangGraph:** Not used. Archon does not wrap LangChain at all — it wraps Claude Code / Codex subprocesses directly, sidestepping the abstraction tax.
- **AutoGen / CrewAI:** Not used. Archon's multi-agent is DAG-shaped, not conversation-shaped; the explicit contrast is to Cognition's "don't build multi-agents" critique, echoed in SemaClaw ([54](54-semaclaw-general-purpose-agent.md)).
- **Claude Agent SDK:** Heavily used under the hood; `hooks`, `thinking`, `skills`, and `betas` are pass-throughs.
- **MCP:** Reused unchanged.
- **agentskills.io SKILL.md format:** Reused for skill definitions.
- **GitHub Actions mental model:** Reused as *inspiration* for the YAML shape, not code.
- **DAG-based planning:** SemaClaw's DAG-Teams and ReWOO ([17](17-rewoo.md)) are close relatives, but Archon commits more heavily to letting the *developer*, not a planner LLM, author the DAG.

## 5. Extension model

**Adding a custom workflow.** Copy `.archon/workflows/defaults/some-workflow.yaml` into `.archon/workflows/some-workflow.yaml`, edit, commit. Same-named workflow in the repo wins over the bundled default. The YAML shape is documented at [archon.diy/guides/authoring-workflows/](https://archon.diy/guides/authoring-workflows/).

**Adding a custom command.** Drop a markdown file into `.archon/commands/<command-name>.md` with frontmatter (`description`, `argument-hint`) and a phase-structured body (LOAD / EXPLORE / ANALYZE / GENERATE / VALIDATE / COMMIT / REPORT with checkpoint checklists). Reference it from any workflow node as `command: <command-name>`.

**Adding a tool.** Tools are not first-class in Archon the way they are in Deep Agents. Tool *access* is controlled via `allowed_tools`/`denied_tools` whitelists drawn from the underlying agent SDK (Claude Code's built-ins) or via MCP. To expose a *new* capability you either (a) author a bash node, (b) author a TypeScript/Python `script` node (v0.3.3+), or (c) attach an MCP server with `mcp: .archon/mcp/servers.json`.

**Adding a skill.** Author a `SKILL.md` in the agentskills.io format, place it in the repo's skill directory, and attach it per-node with `skills: [my-skill]`. The v0.2.8 "Skills system overhaul" introduced specialized cookbooks.

**Adding an LLM backend.** Three options:
- **Native clients** — Claude Code and Codex ship as first-party AI assistant clients.
- **Pi community provider** (v0.3 `[Unreleased]`) — a provider aggregator registered via `registerCommunityProviders()` that brings ~20 LLM backends under a single interface. Contributing guide available for adding a new provider.
- **`provider: X` / `model: Y`** — per-workflow or per-node override.

**Adding a platform adapter.** The README lists Slack (15-min setup), Telegram (5 min), Discord (5 min), GitHub webhooks (15 min). Each is a concrete adapter in the top-layer code; adding a new one means implementing the adapter protocol against the orchestrator.

## 6. Comparison

**vs. LangChain Deep Agents ([42](42-langchain-deep-agents.md)).** Deep Agents ships *primitives inside a single Python agent* — planning tool, virtual filesystem, subagent spawn, async subagents. Archon ships *primitives around a subprocess agent* — DAG, worktrees, approvals, per-node permissions. The overlap is small; the two are complementary. A team could reasonably use Deep Agents *as the agent Archon invokes* inside a bash/script/subprocess node.

**vs. SemaClaw ([54](54-semaclaw-general-purpose-agent.md)).** Both commit to DAG-based orchestration. SemaClaw's DAG Teams are *planner-generated* and targeted at personal-agent tasks; Archon's DAGs are *developer-authored* and targeted at the inner loop of software engineering. SemaClaw's PermissionBridge is runtime-enforced authorization on risky operations; Archon's `allowed_tools` + `sandbox` are compile-time-like per-node lists that achieve a narrower flavor of the same goal. SemaClaw has SOUL.md for persona; Archon has nothing analogous (it is agnostic to the agent's identity because it wraps someone else's agent). SemaClaw is personal-agent-shaped; Archon is SDLC-shaped.

**vs. Hermes ([55](55-hermes-agent-self-improving.md)).** Hermes's thesis is *the agent gets better the longer it runs* (skill promotion from traces). Archon's thesis is *the agent does not need to get better; the harness around it does*. Hermes is a single-agent continual learner; Archon is a multi-workflow deterministic dispatcher. These are the cleanest philosophical opposites in the 2026 open-source harness landscape. You would combine them by letting Hermes's learned skills feed back into Archon's `.archon/commands/` directory — a pipeline not yet prototyped publicly.

**vs. the 10 in-tree MVP projects** (`research/harness-engineering/projects/`: aegis-ops, atlas-research, cipher-sec, harmony-voice, helix-bio, mentat-learn, orion-code, quanta-proof, syndicate, vertex-eval). Those projects are *domain-specialized harnesses* (ops, research, security, voice, bio, learning, code, proof, multi-agent, eval). Archon sits one level up as a *builder* for any of them. If the MVP projects are Dockerfiles, Archon is the `docker build` toolchain. Concretely: Orion Code's [context engine](../../projects/orion-code/blocks/04-context-engine.md) could be rewritten as a set of Archon commands with `context: shared` on retrieval nodes and `context: fresh` on code-generation nodes; Vertex Eval's verifier loops map directly onto Archon `loop` nodes with JSON-Schema `output_format` scoring.

## 7. Production-readiness assessment

**Packaging.** Excellent. Homebrew formula (`brew install coleam00/archon/archon`); single-line install scripts for macOS, Linux, Windows; binary distribution via tarball (v0.3.3); Bun monorepo with `bun install` and `bun run validate`.

**Docker & deployment.** `Dockerfile`, `docker-compose.yml`, `deploy/` directory, `--profile with-db` for bundled Postgres, cloud VPS guide with automatic HTTPS, native Windows and WSL2 support.

**Testing & CI.** GitHub Actions with a `test.yml` workflow (CI badge in README). The v0.2.10 changelog entry specifically calls out "CLI-Web observability overhaul with comprehensive test coverage." Specific coverage numbers unverified.

**Observability.** Pino structured logging across the server (v0.2.5 onwards). Every DAG step writes `node_started` / `node_completed` / `node_failed` events to the DB. The Web UI exposes a DAG graph viewer (v0.2.12) and step-by-step execution progress. Loop iteration progress displays in the execution view (v0.3.6). Update-check cache and environment-leak gates (v0.3.x) indicate active attention to operator concerns.

**Security.** `auth-service/` package; `.env.example`; `SECURITY.md`; explicit environment-leak gate that scans `.env` files and refuses to auto-register sensitive keys without `--allow-env-keys` consent (v0.3.0). Per-node OS-level sandbox with network-domain and filesystem-write denylists (Claude-only). `denied_tools` blacklist. The [AgentConn review](https://agentconn.com/blog/archon-open-source-harness-builder-ai-coding-deterministic-review/) notes audit-trail completeness as a strength.

**Honest caveats.**
- The new codebase is ~4 months old in its current TypeScript form (v0.1.0 shipped 2025-12-08). The project is stable but young.
- Authentication/RBAC details beyond the `auth-service/` directory name are not documented in the public deployment docs — mark **unverified**.
- The AgentConn "6.7% → 70% PR acceptance" figure is widely cited but I did not find reproduction methodology; treat as **directional, not benchmark-grade**.
- Most primitives described as "Claude-only" (sandboxing, allowed-tools, effort, thinking, hooks) reduce to no-ops on Codex and Pi providers. Cross-provider parity is aspirational.

## 8. Takeaway — copy-worthy ideas for builders

1. **Make the workflow an artifact.** A YAML DAG checked into the repo is reviewable, diffable, ownable. Every harness should ship one.
2. **Per-node harness configuration beats per-agent.** `allowed_tools`, `skills`, `mcp`, `sandbox`, `context: fresh|shared`, and `maxBudgetUsd` *on every node* eliminate a class of "my one permissive tool ruined the whole agent" bugs.
3. **JSON Schema + `when` is the cleanest branch primitive.** If your nodes emit schema'd JSON, your routing is deterministic with the LLM providing only the value — not the control flow.
4. **Worktree isolation by default, not opt-in.** Parallelism is the whole point of a DAG; stop stomping on branches.
5. **Event-log resume over state snapshots.** Writing every node transition to SQLite and replaying on restart is simpler, more debuggable, and more honest than snapshotting complex object graphs.
6. **Artifacts-as-memory is the cheapest cross-agent protocol.** *"The artifact you produce IS the specification for the next step"* — it's how Archon sidesteps shared-context rot, and it generalizes.
7. **Structured fail-closed semantics.** Invalid `when` expressions default to `false`; unparseable numerics skip; FATAL errors never retry. Conservative defaults compound into reliable systems.
8. **Ship bundled defaults users can override.** 17 default workflows plus per-project override files via same-name shadowing is a pattern any harness builder should copy.
9. **Portable platform adapters off one orchestrator.** Same workflow from CLI / Web / Slack / GitHub is a product-control-plane ([41](41-product-control-plane.md)) idea, realized.

The most distinguishing feature: **Archon is the first open-source project to treat the AI coding harness itself as a first-class, version-controlled, composable, YAML-authored artifact** — turning every development process into a Dockerfile-for-agents.

## 9. References

- Repository: [github.com/coleam00/Archon](https://github.com/coleam00/Archon) — verified via WebFetch 2026-04-20.
- README (dev branch): [github.com/coleam00/Archon/blob/dev/README.md](https://github.com/coleam00/Archon/blob/dev/README.md) — verified.
- Changelog: [github.com/coleam00/Archon/blob/dev/CHANGELOG.md](https://github.com/coleam00/Archon/blob/dev/CHANGELOG.md) — verified; latest release v0.3.6 dated 2026-04-12.
- Rewrite announcement: [Issue #957 — Archon: Complete Rewrite](https://github.com/coleam00/Archon/issues/957) — verified; posted 2026-04-07.
- Pre-rewrite migration issue: [Issue #952 — Open source Archon: migrate new codebase to coleam00/Archon](https://github.com/coleam00/Archon/issues/952) — verified.
- Archived v1 branch: `archive/v1-task-management-rag` — verified to exist via README reference.
- Docs home: [archon.diy](https://archon.diy) — verified.
- Getting Started: [archon.diy/getting-started/overview/](https://archon.diy/getting-started/overview/) — verified.
- The Book of Archon: [archon.diy/book/](https://archon.diy/book/) — verified; 10-chapter curriculum.
- Authoring Workflows: [archon.diy/guides/authoring-workflows/](https://archon.diy/guides/authoring-workflows/) — verified; full YAML schema extracted.
- Authoring Commands: [archon.diy/guides/authoring-commands/](https://archon.diy/guides/authoring-commands/) — verified.
- Deployment: [archon.diy/deployment/](https://archon.diy/deployment/) — verified.
- AgentConn third-party review: [agentconn.com/blog/archon-open-source-harness-builder-ai-coding-deterministic-review/](https://agentconn.com/blog/archon-open-source-harness-builder-ai-coding-deterministic-review/) — verified; PR-acceptance figures unverified for methodology.
- HelloGitHub feature: [hellogithub.com/en/repository/coleam00/Archon](https://hellogithub.com/en/repository/coleam00/Archon) — verified via search index.
- AIToolly coverage (2026-04-11 and 2026-04-14): [aitoolly.com](https://aitoolly.com/ai-news/article/2026-04-11-archon-the-first-open-source-benchmark-builder-designed-to-make-ai-programming-deterministic-and-rep) — verified via search index.
- Announcement video: [YouTube — Introducing Archon: The Revolutionary Operating System for AI Coding](https://www.youtube.com/watch?v=8pRc_s2VQIo) — URL appears in search; content not independently fetched, mark **partially verified**.
- Author: [Cole Medin — github.com/coleam00](https://github.com/coleam00) — verified.

**Numbers marked unverified in this doc:** exact star count (~19k is a range, not a pinned value), forks (~2,900), total contributor count, AgentConn PR-acceptance methodology, specific test coverage percentage. All feature claims above are grounded in the README, changelog, docs pages, or announcement issue cited; nothing is invented.
