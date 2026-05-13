# 252 — Routines Pattern for Self-Hosted Agents: triggered, isolated, fire-and-forget agent runs

**Anchor.** Anthropic — *Routines in Claude Code* — https://code.claude.com/docs/en/routines, https://platform.claude.com/docs/en/api/claude-code/routines-fire — launched 2026 (research preview). Companion: [Claude Code on the Web](https://code.claude.com/docs/en/claude-code-on-the-web), [Desktop Scheduled Tasks](https://code.claude.com/docs/en/desktop-scheduled-tasks). The pattern below extracts the architectural primitives so any self-hosted agent runtime — Lyra, Polaris, argus, mentat-learn, orion-code, the rest of `projects/` — can ship the same fire-from-anywhere remote-execution surface.

**One-line definition.** A four-component architecture — **server-side routine config store** (prompt + repos + connectors + triggers) + **three trigger ingresses** (cron + REST API with bearer token + webhook) + **isolated execution per fire** (sandboxed worktree, fresh agent loop) + **per-routine bearer-token auth** — that lets a user fire an agent run from anywhere (phone, alerting tool, CI, GitHub webhook, scheduled cron) without keeping a local CLI alive, with the work executing on a remote machine the user does not have to manage minute-to-minute. The pattern is what Claude Code Routines productizes; this file is the blueprint for replicating it in self-hosted agent runtimes.

## Why this pattern matters (the agent's value compounds when you can fire it from anywhere)

A locally-running CLI is an agent that needs you to be at a keyboard. A daemon ([13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [p7-daemon](../projects/polaris/docs/research/p7-daemon.md)) is an agent that wakes up on a heartbeat. A **routine** is an agent that wakes up on **any external event** — a cron tick, an HTTPS POST, a GitHub PR, a Slack mention, a monitoring alert — runs to completion in an isolated sandbox, and returns a session URL. The combination of **fire-from-anywhere triggers** + **isolated remote execution** + **fire-and-forget semantics** is what turns an agent from "a tool I run sometimes" into "a system that runs while I sleep."

Anthropic's Routines launch (2026 research preview) is the cleanest production example. The architecture — three trigger ingresses (schedule / API / GitHub webhook), server-side stored configs, isolated per-fire cloud sessions, per-routine bearer tokens, MCP-connector inheritance — works equally well in a self-hosted runtime running on a $5 VPS or a home server. The four primitives below are filesystem-and-process patterns that any agent in `projects/` (polaris, lyra, argus, mentat-learn, atlas-research, helix-bio, cipher-sec, harmony-voice, vertex-eval, gnomon, orion-code, syndicate, quanta-proof, open-fang, aegis-ops) can adopt by extending its existing daemon and adding an HTTPS endpoint.

The pattern matters now because three things converged in 2025–2026: (a) **agent capabilities crossed the threshold for unsupervised long-running tasks** (SWE-bench → 60 %+, GAIA Level-3 → 50 %+ — see [222](222-agent-trajectory-scaling.md)); (b) **memory infrastructure stabilized** so cross-session learning is real ([233](233-memory-scaling-for-agents.md), MEMTIER trilogy [151–153]); (c) **trigger sources are everywhere** — every monitoring tool, every CI pipeline, every messaging app exposes webhooks. The bottleneck is no longer capability or memory; it is **wiring the agent into the user's existing event topology**, and Routines is the wiring pattern.

Take this seriously and three things change. **First**, every agent in your `projects/` gets a `harness_core/routines/` import that turns it into a fire-from-anywhere runtime — one piece of shared infrastructure, fourteen agents enabled. **Second**, the unit of "agent invocation" shifts from "open the CLI and prompt it" to **"a routine fires from a trigger, runs to completion, returns a URL"** — the user touches the agent only to configure the routine and to inspect outputs after the fact. **Third**, you treat **routine config as durable data** with its own lifecycle (versioning, approval, rotation), not as one-shot prompts; this is the boundary where agent-as-tool becomes agent-as-infrastructure.

## Problem it solves (turning a per-command CLI into a fire-from-anywhere production agent)

1. **Per-command CLI requires the user at the keyboard.** Locally-running Claude Code is at the user's terminal; if the user is asleep or away, the agent is idle. A routine fires from any trigger, anywhere.
2. **Daemon wakes on heartbeat, not events.** The daemon ([13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md)) handles cron-style cadence and queue draining, but lacks an external-event ingress (HTTPS, webhook). Routines complete the trigger model.
3. **No per-trigger config means no reuse.** Without a stored "routine config," every cron job or webhook handler reinvents the prompt + repo + tool grants. Stored configs make recurring agent runs deployable infrastructure.
4. **Execution sandboxing was ad-hoc.** Without explicit isolation per-fire (own worktree, fresh agent loop), concurrent routines stomp on each other's filesystem state.
5. **Auth was binary.** Either the agent has full user credentials or it has none. Per-routine bearer tokens give scoped, single-use, per-trigger auth.
6. **Observability was per-session.** Without a session URL returned per fire, users cannot inspect routine outputs after the fact.

## Core idea in one paragraph

A **routine** is a server-side stored config of `(prompt, repo_or_workspace, connectors, triggers, schedule_or_webhook_or_api_spec, bearer_token_hash, owner_user_id)`. When a trigger fires — cron tick, HTTPS POST `POST /routines/{id}/fire` with the per-routine bearer token, or a webhook from GitHub/Slack/monitoring/email — the routine handler **spins up an isolated execution context**: clones the repo into a fresh worktree ([18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md)), instantiates a fresh agent loop ([01-agent-loop-architecture](01-agent-loop-architecture.md)) with the routine's prompt, attaches the configured MCP connectors / tools / skills, runs to completion under bright-line gates ([14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md)), persists results to the routine's run history, and returns a `session_id` URL the user can inspect later. The user does not need to be present; the local CLI does not need to be running; the runtime does not maintain a long-lived connection to the user's machine. The four primitives are: (1) **routine config store** — SQLite or Postgres `routines` table with versioned configs; (2) **trigger ingresses** — cron daemon thread + HTTPS API endpoint + webhook receivers; (3) **isolated execution per fire** — fresh worktree, fresh agent loop, sandboxed; (4) **per-routine bearer tokens** — generated at create-time, hash stored, single-view secret, scoped to that one routine. The four primitives compose into a fire-from-anywhere remote-agent surface that any agent in `projects/` can ship by extending its daemon and adding ~500 lines of shared `harness_core/routines/` code.

## Mechanism (step by step)

### (a) The routine config schema

```python
@dataclass
class RoutineConfig:
    routine_id: str                    # uuid
    owner_user_id: str
    name: str                          # human-readable
    prompt: str                        # injected as initial agent message
    workspace: WorkspaceSpec           # repo URL + branch + ref OR project dir
    connectors: list[ConnectorSpec]    # MCP servers, tool grants
    skills: list[SkillSpec]            # .claude/skills/ refs
    triggers: list[TriggerSpec]        # see (b)
    bearer_token_hash: bytes           # for API-fire trigger
    permission_mode: PermissionMode    # ask | normal | dangerous
    cost_cap_per_run: int              # token cap; bright-line escalates on excess
    created_at: datetime
    updated_at: datetime
    version: int                       # for audit trail of edits
    status: RoutineStatus              # active | paused | archived

@dataclass
class TriggerSpec:
    kind: TriggerKind                  # CRON | API | GITHUB | SLACK | WEBHOOK
    spec: dict                         # cron expression, github filter, etc.
```

Stored in SQLite (`harness_core/routines.db`) or Postgres. Versioned: every edit creates a new row; the `routine_id` resolves to the latest active version.

### (b) Three trigger ingresses

**Cron** (extends existing daemon's heartbeat loop):

```python
def _cron_tick(now: datetime) -> None:
    for routine in store.list_active_routines_with_cron_trigger():
        if cron_due(routine.cron_spec, now, routine.last_fired_at):
            _fire_routine(routine, trigger_kind=TriggerKind.CRON, payload={})
```

**REST API** (new HTTPS endpoint):

```python
# POST /v1/routines/{routine_id}/fire
# Authorization: Bearer <per-routine token>
# Body: {"text": "optional context to inject into prompt"}

def fire_routine_endpoint(routine_id: str, request: Request) -> Response:
    token = request.headers.bearer_token()
    routine = store.get(routine_id)
    if not bcrypt.verify(token, routine.bearer_token_hash):
        return 401
    payload = request.json()
    session_id = _fire_routine(routine, trigger_kind=TriggerKind.API, payload=payload)
    return {"session_id": session_id, "session_url": f"/sessions/{session_id}"}
```

**Webhooks** (per-source receivers):

```python
# POST /v1/webhooks/github   (GitHub App webhook)
# POST /v1/webhooks/slack    (Slack event API)
# POST /v1/webhooks/generic  (HMAC-signed external)

def github_webhook(event_type: str, payload: dict) -> Response:
    for routine in store.list_active_routines_with_github_trigger():
        if matches_filter(routine.github_filter, event_type, payload):
            _fire_routine(routine, trigger_kind=TriggerKind.GITHUB, payload=payload)
    return 204
```

### (c) Isolated execution per fire

Every fire gets its own sandbox. The pattern from [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md):

```python
def _fire_routine(
    routine: RoutineConfig,
    trigger_kind: TriggerKind,
    payload: dict,
) -> str:
    session_id = uuid4().hex
    run_dir = Path(f"~/.lyra/runs/{session_id}").expanduser()
    run_dir.mkdir(parents=True)

    # 1. Permission-bridge gate on the fire itself
    decision = bridge.decide(ResearchAction(
        kind=ActionKind.ROUTINE_FIRE,
        summary=f"fire routine {routine.name} (trigger={trigger_kind.value})",
    ))
    if decision.verdict != AllowVerdict.ALLOW:
        emit_hir(HIREventKind.ROUTINE_DENIED, {...})
        raise PermissionError(decision.reason)

    # 2. Clone workspace into isolated worktree
    if isinstance(routine.workspace, RepoSpec):
        git.worktree_add(run_dir / "repo", routine.workspace.url, routine.workspace.ref)

    # 3. Instantiate fresh agent loop
    loop = AgentLoop(
        cwd=run_dir / "repo",
        prompt=_compose_prompt(routine.prompt, payload),
        permission_mode=routine.permission_mode,
        connectors=routine.connectors,
        skills=routine.skills,
        cost_cap=routine.cost_cap_per_run,
    )

    # 4. Run to completion, emit HIR events
    emit_hir(HIREventKind.ROUTINE_FIRE_START, {
        "routine_id": routine.routine_id,
        "trigger_kind": trigger_kind.value,
        "session_id": session_id,
    })
    try:
        result = loop.run_until_done()
    except CostCapExceeded as e:
        emit_hir(HIREventKind.BRIGHT_LINE_ESCALATION, {"reason": str(e)})
        raise
    finally:
        emit_hir(HIREventKind.ROUTINE_FIRE_END, {"session_id": session_id, ...})
        store.record_run(routine.routine_id, session_id, result)

    return session_id
```

### (d) Per-routine bearer tokens

Generated at routine creation; **single-view secret** (no later retrieval); hash stored in DB:

```python
def create_routine(config: RoutineConfig) -> tuple[RoutineConfig, str]:
    raw_token = f"sk-rt-{secrets.token_urlsafe(32)}"
    hashed = bcrypt.hashpw(raw_token.encode(), bcrypt.gensalt())
    config.bearer_token_hash = hashed
    store.insert(config)
    # raw_token shown ONCE in response; user must store it now
    return config, raw_token
```

Tokens are scoped to one routine (cannot fire other routines), revocable by routine archival, single-use viewable.

### (e) Webhook signature verification

Every webhook source must HMAC-sign:

```python
def verify_github_signature(payload_bytes: bytes, sig_header: str) -> bool:
    expected = "sha256=" + hmac.new(
        os.environ["GITHUB_WEBHOOK_SECRET"].encode(),
        payload_bytes,
        hashlib.sha256,
    ).hexdigest()
    return hmac.compare_digest(expected, sig_header)
```

### (f) Run history and observability

Every fire emits HIR events ([16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md)) to a per-routine event log:

```
~/.lyra/routines/<routine_id>/runs/<session_id>/events.jsonl
~/.lyra/routines/<routine_id>/runs/<session_id>/diff.patch
~/.lyra/routines/<routine_id>/runs/<session_id>/summary.md
```

Users inspect via web UI, CLI (`lyra routines runs <routine_id>`), or programmatic API.

### (g) Bright-line escalation per fire

Every fire is bound by:

- `cost_cap_per_run` — fires bright-line on token overage.
- Routine `permission_mode` — bridges every tool call.
- Workspace scope — file edits only inside the cloned worktree.
- Bridge classifies `ROUTINE_FIRE` actions distinctly from `DAEMON_START` / `AGENT_LOOP_START`.

Bright-line escalation pauses the routine, writes status to the run's `STATUS.md`, notifies via the user's notification channel — never silently retries past a gate.

## Empirical results (table — Anthropic Routines pricing/limits as the reference)

**Table 1 — Anthropic's published routine limits (2026 research preview)**

| Plan | Routine runs / day | Notes |
|---|---:|---|
| Pro | 5 | Subscription pool; one-off runs exempt |
| Max | 15 | |
| Team / Enterprise | 25 | Per user |

**Table 2 — Trigger model summary**

| Trigger | Latency | Use case |
|---|---|---|
| Schedule (cron) | seconds | Daily PR review, weekly cleanup |
| API (REST POST) | sub-second | Alert-driven response, mobile-fired runs |
| GitHub webhook | seconds | PR-opened review, release-cut release-notes |
| Slack mention | seconds | Conversational invocation from chat |
| Generic webhook | seconds | Custom integrations (monitoring, SaaS) |

**Table 3 — Self-hosted footprint estimate**

| Component | LoC | Storage | Daemon CPU |
|---|---:|---:|---:|
| Routine config store | ~200 | SQLite, <100 MB / 1k routines | trivial |
| Cron ingress | ~100 | n/a | one thread, periodic check |
| HTTPS API | ~150 | n/a | aiohttp / FastAPI handler |
| Webhook receivers | ~250 | n/a | per-source verifier |
| Isolated executor | ~300 (extends agent loop) | per-run worktree, ~50 MB | one process per fire |
| Run history + HIR | ~100 | append-only JSONL, GiB / month at scale | trivial |
| **Total** | **~1100** | **modest** | **modest** |

Roughly one engineer-week of work for an existing agent runtime that already has a daemon and agent loop.

## Variants and ablations

- **Cron-only routines.** Simplest variant; skip API and webhook ingresses. Most "personal scheduled assistant" use cases fit here.
- **API-only routines.** For programmatic / CI / mobile triggering; skip cron.
- **Webhook-only routines.** Pure event-driven; skip cron and API.
- **Multi-trigger routines.** One routine fires from any of N triggers (e.g., PR-opened *or* /command-fire-pr-review).
- **Routine versioning.** Every edit creates a new version; runs reference version-at-fire-time for reproducibility.
- **Routine approval workflow.** Two-person review for high-impact routines (deploy automation, prod data access).
- **Templated routines.** Marketplace of pre-defined routine templates ([175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md)).
- **Routine chains.** One routine's completion triggers another (DAG composition).
- **A2A-bridged routines.** Cross-runtime triggers via [A2A protocol](https://a2a-protocol.org); fire a routine in argus from polaris.
- **MCP connector inheritance.** Routines inherit user's linked MCP connectors at fire-time.

## Failure modes and limitations

- **Bearer-token leakage.** Single-view secret means a leak is permanent until rotated; webhook signature verification mitigates partially but not fully.
- **Webhook source spoofing.** Without HMAC verification, anyone can fire your routine. Always verify.
- **Routine drift.** Stored prompt / repo state drifts from current best practice; needs versioning and review cadence.
- **Cost runaway.** Without `cost_cap_per_run`, a misbehaving routine burns budget. Bright-line escalation mandatory.
- **Concurrent fires of same routine.** Default behaviour: spawn N parallel runs in N worktrees. Optional: serialize via routine-level lock.
- **Worktree disk usage.** N concurrent runs × cloned repo = N copies; cleanup policy needed.
- **GitHub webhook delivery failures.** GitHub retries with backoff; receiver must be idempotent.
- **Per-routine permission scope is coarse.** Permission mode is per-routine, not per-tool; production needs finer grants.
- **No native cross-vendor inter-routine trigger.** A2A is the path; not yet integrated.
- **Run history grows unboundedly.** Append-only JSONL needs rotation / archival.
- **Routine reuse across users hard.** Routines are owner-bound; sharing requires templating.

## When to use, when not

**Build a routines pattern** when your agent harness has reached the point where users want to fire it from outside the CLI — when scheduled runs, mobile triggers, CI invocations, monitoring-driven runs, or webhook-driven runs would add value. The strongest cases are **personal-assistant agents** (mentat-learn, lyra), **research agents** (atlas-research, polaris), **SWE agents** (orion-code), and **ops / monitoring agents** (aegis-ops, argus). Each project in `projects/` benefits in proportion to how often the user would want to invoke it asynchronously.

**Skip routines** when the agent is intrinsically interactive (voice agents like harmony-voice with sub-second latency requirements have different ergonomics); when no external trigger sources matter to your workflow; when single-machine, single-user, single-CLI is sufficient; when the security perimeter cannot accommodate inbound HTTPS or webhook traffic; when the project lifecycle is too early to justify ~1000 LoC of routine infrastructure.

## Implications for harness engineering (one shared `harness_core/routines/` package, fourteen agents enabled)

- **Pull the routines pattern up to `harness_core/routines/`** — not 14 per-project ports. One module: config store, cron ingress, HTTPS API, webhook receivers, isolated executor, run history. Each agent imports it.
- **Daemon as cron ingress** — [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [p7-daemon](../projects/polaris/docs/research/p7-daemon.md) — extend existing tick to scan due routines.
- **Permission Bridge gates fire** — [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — `ROUTINE_FIRE` action kind.
- **Worktree per fire** — [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md), [02-subagent-delegation](02-subagent-delegation.md).
- **Cost-cap as bright-line** — [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md), [p13-cost-discipline](../projects/polaris/docs/research/p13-cost-discipline.md).
- **HIR observability per fire** — [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — events: `ROUTINE_CREATE`, `ROUTINE_FIRE_START`, `ROUTINE_FIRE_END`, `ROUTINE_DENIED`.
- **MCP connector inheritance** — [07-model-context-protocol](07-model-context-protocol.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md) — routines reuse user's linked MCP servers.
- **Skill bundling per routine** — [04-skills](04-skills.md), [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md).
- **Cross-channel verifier on terminal step** — [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md).
- **Routine config as durable data** — versioned, audited, rotation cadence; like infrastructure, not prompts.
- **A2A bridge for cross-vendor triggers** — [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md) — long term.
- **Routines + Agent Teams composition** — [250-anthropic-agent-teams](250-anthropic-agent-teams.md) — a routine can fire that spawns a team; powerful pattern for parallel scheduled work.
- **Tailscale + NATS for distributed routines** — [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md) — fire on machine A, execute on machine B.
- **Trajectory simulation for routine validation** — [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md), [195-argus-omega-vol-2-trajectory-temporal-horizon](195-argus-omega-vol-2-trajectory-temporal-horizon.md) — replay routine traces offline before scheduling live.

**One-line takeaway for harness designers.** **Routines turn an agent from a per-command CLI into fire-from-anywhere infrastructure — four primitives (server-side config + three triggers + isolated execution + per-routine bearer tokens) compose into a Claude-Code-grade remote-agent surface that ships in one engineer-week as a shared `harness_core/routines/` package, enabling every agent in `projects/` simultaneously and turning every monitoring tool, CI pipeline, and messaging app into an agent trigger.**
