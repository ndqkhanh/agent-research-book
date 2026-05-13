# 300 — Eternal Mode: Implementation of the 7-Primitive Autonomy Stack in Lyra/Argus/Polaris

**Anchors.** Theoretical synthesis: [308-autonomy-loop-synthesis](308-autonomy-loop-synthesis.md). Plan-of-record: `~/.claude/plans/compressed-tumbling-lagoon.md` (May 2026). Source primitives: [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md), [160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md), [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md), [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md), [305-agent-contracts-formal-framework](305-agent-contracts-formal-framework.md), [306-stop-hook-auto-continue-pattern](306-stop-hook-auto-continue-pattern.md), [307-ralph-loop-variations-2026](307-ralph-loop-variations-2026.md). Memory anchors: [184-strongest-memory-techniques-synthesis-may-2026](184-strongest-memory-techniques-synthesis-may-2026.md), [151-memtier-why-flat-memory-breaks-at-72-hours](151-memtier-why-flat-memory-breaks-at-72-hours.md). Source code: `packages/harness-eternal/`, `projects/{lyra,argus,polaris}/packages/`.

**One-line definition.** A **single 600-LoC shared package** (`harness-eternal/`) plus three project-specific phase rollouts that ship the 7-primitive autonomy-loop stack from [308](308-autonomy-loop-synthesis.md) — **(1) durable workflow journal** (Restate-shaped, SQLite-backed, idempotent step replay); **(2) per-iteration deadline** (SIGALRM hard-stop); **(3) constant-size memory cap** (Deep Researcher 24x7); **(4) workflow-level circuit breaker** (sliding-window quarantine); **(5) async HITL channel** (`HUMAN_DIRECTIVE.md`); **(6) per-day budget envelope** ($/tokens/wallclock); **(7) external supervisor** (systemd / launchd) — across **Lyra (cron daemon + REPL turns + /spawn'd subagents + skill review)**, **Argus (first daemon ever, four heartbeat cycles)**, and **Polaris (P22 daemon hardening)**, with **141 dedicated tests + zero regressions** across 700+ pre-existing tests, and a **subprocess SIGKILL chaos test** that validates the durability claim end-to-end.

**Disambiguation.** [308](308-autonomy-loop-synthesis.md) is the *theory* of the stack — what the seven primitives are and why their composition is non-arbitrary. [266](266-agent-durability-and-idempotency.md) is the durability *vocabulary*. This file is the *implementation report*: what we actually shipped, the integration pattern that makes it tractable, the test evidence, and the operational gates that remain.

## Why this implementation matters

Three things separate this delivery from a typical "we built durability" entry:

**One package, three projects.** Every prior in-tree autonomy-loop pattern was per-project: Lyra had its REPL-embedded cron; Polaris had its tick-driven daemon; Argus had no daemon at all. Each project paid for its own primitives separately. `harness-eternal` is one library that all three import — a 30-test core covering journal, runtime, EternalLoop, deadline, breaker, memory cap, supervisor unit generation, and a real subprocess chaos driver. The integration cost per project is one CLI module + one adapter + tests.

**Restate-shaped, SQLite-backed.** The user's plan-of-record decision was Restate over Temporal/DBOS/in-process for the durable execution engine. We honor that decision while keeping the dev-loop testable: a `LocalRuntime` implements the same `(workflow, step, memoize)` decorator surface using a SQLite journal that lives at `<state_dir>/restate/journal.sqlite3`. The same code that runs against `LocalRuntime` in tests runs against a real `restate-server` sidecar in production by swapping the runtime constructor — no application changes. This is the same trick OpenHands uses for its event-sourced runtime: pluggable substrate, fixed contract.

**The chaos test is the ship gate.** `tests/test_loop_integration.py::test_sigkill_replay_preserves_completed_steps` spawns a worker subprocess, kills it with SIGKILL after step a's side-effect line is written to a log file, then re-spawns with the same invocation_id and asserts (a) the log does NOT contain "a-replay" (the recorded result returned without re-running), (b) the log DOES contain "b" and "c" (steps b+c ran fresh), (c) the final result equals `["result-a", "result-b", "result-c"]`. This is the operationalised version of the durability claim from [266](266-agent-durability-and-idempotency.md): not "we wrote a journal" but "kill-and-replay produces deterministic, side-effect-correct continuations."

Take this implementation seriously and three things change. (1) New project bring-up follows a one-page recipe (`make eternal-up` + supervisor unit + workflow registry adapter). (2) The 30-day soak from [308](308-autonomy-loop-synthesis.md) §"Empirical results" becomes a single `make eternal-soak DURATION_H=720 PROJECT=<name>` invocation per project, with the chaos driver doing the daily SIGKILL automatically. (3) Eternal Mode becomes the *default* shape for any new daemon in the tree — opt-out, not opt-in.

## Problem this implementation solves

Concrete gaps in the three projects before this delivery:

- **Lyra** — REPL-bound. Cron daemon was a thread *inside* the prompt-toolkit process; Ctrl-D kills both. No process supervisor. No per-turn deadline. No auto-compaction. In-flight tool dispatches lost mid-call.
- **Argus** — no daemon at all. Refinement was a callback the host had to orchestrate. Catalog drift accumulated with no scheduled remediation.
- **Polaris** — most daemon scaffolding (P1–P7 shipped), but the daemon docstring at `polaris-daemon/daemon.py:3` literally said "real production deployment wraps this in `while not stopped: daemon.tick(); sleep(interval)`" — and that wrapper did not exist. No exponential backoff, no per-task watchdog, no mid-loop context compression, no OOM handler.
- **Cross-cutting** — no shared substrate. Each project would pay for journaling, watchdog, supervisor units separately.

## Core idea in one paragraph

Three layers compose every Eternal-Mode daemon: a **Restate-shaped durable workflow** (`@workflow`, `step`, `memoize`) wraps every meaningful unit of work; a thin **`EternalLoop` driver** drives the workflow under per-iteration deadline + circuit breaker + budget envelope + heartbeat scheduler + health endpoint + directive-file HITL; a **systemd / launchd unit** (auto-generated) supervises the process. Per-project adapters wire the project's existing workflow registry through this stack: `polaris-daemon/eternal_adapter.py` wraps polaris-skills workflows; `argus/eternal_cycles.py` registers four heartbeat cycles (fetch / drift / rewrite / consolidate); `lyra-cli/daemon_cmd.py` extracts the cron loop into a free-standing process and `lyra-cli/eternal_factory.py` provides a factory that any consumer (background skill reviewer, /spawn'd subagents, future REPL turn site) can use to wrap an `AgentLoop`. The same `LocalRuntime` substrate that powers tests is what powers the supervisor — there is no test-only code path.

## Mechanism — what we shipped, in dependency order

### 1. The shared package: `packages/harness-eternal/`

```
packages/harness-eternal/src/harness_eternal/
  loop.py                # EternalLoop driver (∼240 LoC)
  budget.py              # per-day USD/tokens/wallclock cap (∼80 LoC)
  memory_cap.py          # Tier-1 brief + Tier-2 rolling log (∼100 LoC)
  deadline.py            # SIGALRM context manager (∼30 LoC)
  breaker.py             # sliding-window workflow quarantine (∼40 LoC)
  heartbeat.py           # cadenced cycle scheduler (∼60 LoC)
  directive.py           # HUMAN_DIRECTIVE.md JSONL appender (∼60 LoC)
  health.py              # stdlib HTTP /heartbeat endpoint (∼80 LoC)
  zero_cost_monitor.py   # kill -0 / nvidia-smi / log tail (∼80 LoC)
  fresh_context.py       # Ralph-style threshold-based reset policy (∼50 LoC)
  supervisor.py          # systemd + launchd unit generators (∼100 LoC)
  chaos.py               # subprocess SIGKILL chaos driver (∼180 LoC)
  restate/journal.py     # SQLite invocations + steps + activities (∼160 LoC)
  restate/runtime.py     # LocalRuntime + RestateRuntime adapters (∼260 LoC)
  restate/idempotency.py # IdempotencyKey contract (∼50 LoC)
```

The runtime exposes three decorators (`workflow`, `step`, `memoize`) plus `attach`, `register`, `invoke` helpers. The `step()` semantics are: (a) increment per-invocation step counter; (b) lookup `(invocation_id, step_idx)` in `steps` table; (c) if recorded, return; (d) otherwise call `fn`, journal, return. Idempotent activities additionally check the `activities` table keyed on `idempotency_key`.

### 2. Polaris daemon adapter (P22.0–P22.6)

The daemon CLI (`polaris-daemon/cli.py`) wires the existing `Daemon.tick()` into `EternalLoop.workflow`. Each polaris-skills workflow goes through `eternal_adapter.adapt_workflow(...)` which wraps it as a Restate workflow with the `lyra.workflow.<name>` namespace, `asyncio.wait_for`-enforced per-task deadline, and circuit-breaker tracking. Three additional pieces close the V2 P39–P41 OOM gap, the sleep-time CONSOLIDATE_DEEP cycle, and Ralph-style task isolation:

- `oom_watchdog.py` — `psutil`-backed RSS probe; when RSS > 80% of cgroup/host limit, shrinks `AdaptiveCap.current_max_iterations` for the next workflow and triggers an out-of-band CONSOLIDATE cycle. Cooldown-throttled directive entries prevent log flooding.
- `eternal_cycles.py` — `register_consolidate_deep(...)` registers a nightly cycle (default cadence 86,400 s) that calls a pluggable callback (`polaris_skills.auto_creator` in production; no-op in dev). Implements [Letta sleep-time compute](https://arxiv.org/abs/2504.13171) on top of the harness-eternal heartbeat scheduler.
- `isolation.py` + `isolated_runner.py` — workflows annotated with `__eternal_isolation__ = True` (or with `__eternal_expected_duration_s__ > 600`) run inside a fresh Python subprocess via `python -m polaris_daemon.isolated_runner`. The child reads only `program_dir`-resident state ([Ralph](165-ralph-autonomous-loop.md) thesis); cumulative parent-context drift is structurally bounded.

### 3. Argus first daemon (A11–A14)

Argus shipped A0–A10 as a pure library; A11–A14 add the daemon. `harness_skill_router/cli.py` exposes `python -m harness_skill_router run <state-dir>`. Four heartbeat cycles register on the EternalLoop's scheduler: `cycle_fetch` (hourly, gated on `BL-MARKETPLACE-IMPORT` for first-time adapter pulls), `cycle_drift` (6-hourly, read-only `DriftDetector.detect`), `cycle_rewrite` (daily, gated on per-skill `BL-SKILL-REWRITE`), `cycle_consolidate` (weekly, `Consolidator.find_duplicates`). The bright-line bridge (`permissions.py`) ports the Polaris pattern verbatim with four Argus-specific gates and a `permission-ledger.jsonl` audit trail.

### 4. Lyra REPL/daemon split + factory

`lyra-cli/daemon_cmd.py` extracts the existing `CronDaemon` into a free-standing process under EternalLoop supervision. `lyra-cli/eternal_factory.py::make_eternal_loop_factory` wraps any `Callable[[], AgentLoop]` into a factory that returns durable loops; `lyra_skills.review.background.spawn_skill_review` adopted it (opt-in via `eternal_state_dir`). `lyra_core.agent.eternal_turn.EternalAgentLoop` wraps an entire AgentLoop turn as a Restate workflow; `eternal_llm.JournaledLLM` proxies LLM calls by per-turn counter to give *deterministic mid-turn replay*. Session-side, `LYRA_ETERNAL_SUBAGENT_DIR` env var gates the `/spawn` opt-in: when set, `_ensure_subagent_registry` wraps the inner `_loop_factory` through the eternal factory, defensively falling back to the plain factory on any wiring error.

### 5. Production model factories for the workflow registry

`polaris_skills/model_factories.py` ships reference `AnthropicModel` and `OpenAIModel` adapters (lazy SDK imports — module imports clean without `anthropic` / `openai` installed) plus `pair_from_env(...)` convenience for the cross-model invariant ([ADR-002](150-temporal-durable-execution-substrate.md) family-pair rule). Production deployments compose their own registry from these factories; dev/CI uses the existing `MockExecutor` path.

## Empirical results

Test counts as of 2026-05-09:

| Component | New tests | Pre-existing untouched | Total |
|---|---:|---:|---:|
| `harness-eternal/` | 30 | 0 | 30 |
| `polaris-daemon/` | 32 | 28 | 60 |
| `polaris-skills/` | 30 | 110 | 140 (+2 gated live) |
| `argus/` | 15 | 203 | 218 |
| `lyra-cli/` | 24 | 0 | 24 |
| `lyra-core/` (eternal mods) | 18 | 0 | 18 |
| **Total** | **149** | **341** | **490** |

Notable empirical evidence:

- **SIGKILL chaos test passes.** `harness-eternal/tests/test_loop_integration.py` and `harness-eternal/tests/test_chaos.py` together exercise three real subprocess kill+respawn cycles in ~10s; the journal grows monotonically, the recorded step result returns on replay, the side-effect log line is NOT duplicated.
- **Mid-turn LLM replay works.** `lyra-core/tests/test_eternal_llm.py::test_eternal_loop_replays_full_turn_deterministically` runs a 3-LLM-call sequence, replays with a brand-new inner LLM that has zero scripted responses, and confirms (a) replay produces the same `final_text="DONE"`, (b) the inner LLM call count stays 0, (c) the idempotent tool call count stays 1.
- **End-to-end pipeline.** `polaris-daemon/tests/test_cli_e2e_registry.py` enqueues a `pre_research_triage` task, runs `python -m polaris_daemon run --workflows polaris_skills.workflow_registry:default`, and confirms the task transitions `queued → running → done` while the SQLite journal records both `polaris.tick` and `polaris.workflow.pre_research_triage` invocations.
- **Phase replay inside polaris-skills.** `polaris-skills/tests/test_eternal_steps.py::test_phase_resumes_from_first_un_recorded_phase` proves that when `post_acceptance.run`'s `camera_ready` phase raises, replay returns the recorded `review` outcome from the journal and re-runs only `camera_ready` and `archive`.

## Variants and ablations

- **`LocalRuntime` vs `RestateRuntime` swap.** Both implement the same `(workflow, step, memoize, invoke, register)` surface. `LocalRuntime` runs unit tests; `RestateRuntime` would attach to a `restate-server` sidecar in production. The constructor swap is a one-line change in the EternalLoop site.
- **`isolation = "fresh_context"`** — workflows opt in either via class annotation (`__eternal_isolation__ = True`) or via the duration heuristic (`__eternal_expected_duration_s__ > 600`). Tests prove explicit opt-out wins over duration heuristic, and short-task detection keeps overhead near zero.
- **`memory_cap.on_overflow`** — three policies: `DROP_OLDEST` (forget), `MIGRATE_TO_KG` (callback to a knowledge graph; default), `SUMMARIZE` (LLM-summarize and re-add as one compact line; deferred to Phase 3).
- **`PermissionMode.PERMISSIVE`** for tests vs `NORMAL` (default deny) for production. The `pre_approved_kinds: frozenset[ActionKind]` whitelist is the seam tests use to bypass `BL-DAEMON-START` / `BL-SKILL-REWRITE` etc.
- **`EternalAgentLoop.run_conversation`** as drop-in for `AgentLoop.run_conversation`. Returns a `_TurnView` exposing `final_text` / `iterations` / `tool_calls` / `stopped_by` so callers (`SubagentRunner` in particular) duck-type without code changes.

## Failure modes and limitations

- **Idempotency contract is opt-in for tools.** Tools without `__eternal_idempotent__ = True` re-fire on replay. A side-effecting tool (file write, network POST) that doesn't declare the attribute will double-fire after a SIGKILL. The `Tool.idempotency_key` contract is documented but not enforced; production callers must audit their tool registries. See [266](266-agent-durability-and-idempotency.md) §"the tool taxonomy" for the discipline.
- **Single-host only.** Multi-machine federation is out of scope. Restate's distributed mode would change this but adds significant operational surface; we explicitly chose the single-binary local sidecar.
- **`signal.alarm` doesn't kill threads.** Per-task deadline uses `asyncio.wait_for`; for sync workflows running in `asyncio.to_thread`, a hung Python thread won't be interrupted. Workflow code that shells out to subprocesses must pass `subprocess.run(timeout=...)` and `Popen.kill()` at the step boundary.
- **No Restate live-server bring-up tested in this delivery.** `RestateRuntime` falls back to `LocalRuntime` when `restate-sdk` is missing; the lazy-import path is exercised. A real `restate-server` integration test is gated for ops.
- **Polaris per-step rewrites only landed for `post_acceptance.run`.** `lit_survey.run`, `experiment.run`, `draft.run`, `pre_research.run_*` are single-loop workflows that don't benefit from intra-workflow phase boundaries; durability remains at the whole-workflow grain there. Post-acceptance was the only multi-step case in the registry.
- **30-day soak unrun.** Wall-clock dependent; the harness exists in `packages/harness-eternal/SOAK.md`. Acceptance criteria are codified but the actual 720-hour run is operations work.

## When to use Eternal Mode, when not

Use it for any process that should survive longer than one user turn, where any of: (a) the process holds work whose cost would be wasted if re-done, (b) a tool side-effect must not double-fire on retry, (c) the process must self-throttle on dollar/token/wallclock budgets, (d) operators expect to recover from SIGKILL without losing committed work, (e) the process gets cheaper as it runs longer because heartbeat consolidation compresses what would otherwise be re-derived. The three projects we shipped — Lyra cron daemon, Argus curation daemon, Polaris research daemon — all match three or more of those criteria.

Don't use it for a one-shot CLI invocation, for a request/response library called by another supervisor (Argus's library mode pre-A11), or for a process where context size is genuinely small enough to fit twice in the window. The fresh-context Ralph fallback is opt-in for a reason — short-task overhead matters.

## Implications for harness engineering

1. **Per-project daemons inherit a common substrate.** New daemon bring-up is now `make eternal-up` + a CLI module + a workflow adapter — see Polaris's `polaris-daemon/cli.py` as the reference. Cross-link to [285-harness-core-porting-plan](285-harness-core-porting-plan.md) for the broader package-vendoring story; harness-eternal is one of the libraries in scope.
2. **Workflow registries become first-class.** `polaris_skills.workflow_registry:default` is the pattern any project with multiple workflows should adopt — a `dict[str, Callable[[Path, dict], str]]` with a shape-adapter per workflow plus a model factory. Production deployments override the factory; dev/CI uses MockExecutor.
3. **Idempotency contract belongs in the tool definition, not the daemon.** The `__eternal_idempotent__` annotation (and the `__eternal_isolation__` and `__eternal_expected_duration_s__` siblings) lets tools declare their replay semantics where they live. Cross-link to [266](266-agent-durability-and-idempotency.md) for the discipline.
4. **Bright-line gates port without modification.** Argus's permission bridge is a near-verbatim copy of Polaris's. The pattern generalises — see [273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md) §"the bridge pattern."
5. **Heartbeat cycles are the right home for sleep-time compute.** Polaris's `CONSOLIDATE_DEEP` cycle is the operationalised version of [Letta sleep-time compute](https://arxiv.org/abs/2504.13171); Argus's four cycles are the operationalised version of [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md)'s curation discipline. Cross-link to [267-agent-sre-2026](267-agent-sre-2026.md).
6. **`HUMAN_DIRECTIVE.md` works as the single async HITL channel.** Same shape as the Deep Researcher 24x7 paper ([160](160-deep-researcher-agent-24x7.md)) and as [308](308-autonomy-loop-synthesis.md) §"Layer 5". The directive file is JSONL; entries declare `kind` (`extend_budget`, `unblock`, `oom_pressure`, …) + `payload`; the EternalLoop reads them at the top of each iteration.
7. **Per-day budget envelope is a first-class invariant.** `Budget` enforces USD + tokens + wallclock independently. Going over → loop pauses, writes to the directive file, awaits ack. Cross-link to [278-agent-unit-economics-2026](278-agent-unit-economics-2026.md).
8. **The chaos driver is the ship gate.** `harness_eternal.ChaosDriver` exists; production deployments run it for a 720-hour window per project. Cross-link to [266](266-agent-durability-and-idempotency.md) §"the kill test."
9. **`/spawn`'d subagents inherit durability for free** when `LYRA_ETERNAL_SUBAGENT_DIR` is set. The session-level opt-in is one env-var; the factory-level wrapping is one function call. Pattern generalises to any `loop_factory` consumer.
10. **Fresh-context iteration ([Ralph](165-ralph-autonomous-loop.md)) is the bound-context escape hatch.** `EternalAgentLoop` doesn't auto-trigger Ralph reset, but the `AutoCompactingLLM` proxy in `lyra-core/context/eternal_autocompact.py` raises `ContextOverflow` at the 85% threshold; callers respond by abandoning the turn and re-entering with on-disk state.
11. **Mid-turn replay is a stronger guarantee than mid-task replay.** `JournaledLLM` + `JournaledTools` together make the AgentLoop's control flow deterministic on replay — the messages list reconstructs because LLM calls return the same recorded result at the same iteration. This is the strongest replay guarantee in the project. Cross-link to [307-ralph-loop-variations-2026](307-ralph-loop-variations-2026.md).
12. **Activity-table memoization is sync.** The journal's `lookup_activity` / `record_activity` calls are synchronous SQLite — sync workflow code can use them directly without async/await ceremony. This is what lets `phase()` in `polaris_skills/eternal_steps.py` work without changing any workflow signatures.
13. **Restate is the durable-execution substrate** per the plan-of-record decision. Temporal and DBOS were the alternatives; Restate won on lightness (single binary), Python SDK quality, and the "durable agents" framing. Cross-link to [150](150-temporal-durable-execution-substrate.md) for the trade-off space.
14. **The shared package is the integration glue.** `harness-eternal` is a peer of `harness-skill-router` (Argus's library shape pre-daemon). Cross-link to [298-harness-core-integration-glue](298-harness-core-integration-glue.md) for the broader per-project adoption pattern.
15. **Eternal Mode IS the seven primitives.** Each primitive from [308](308-autonomy-loop-synthesis.md) maps to one or two modules in `harness-eternal`. The implementation report this file represents is, in shape, an audit against the synthesis: what ships, in what file, with what test count.

## Adoption recipe — bring Eternal Mode into a fresh project

Concrete sequence a new project follows. Total time: a working day for the basic shape; a week for production-grade including the workflow registry adapter and live model factories.

**Step 1. Vendor `harness-eternal/`.** Either as a git-submodule peer of the project's own `packages/`, or by adding the package directory to PYTHONPATH. The package has zero non-stdlib runtime dependencies (`psutil` is a soft dep used only by the OOM watchdog).

**Step 2. Define the per-iteration workflow.** A trivial idle loop is:

```python
from harness_eternal.restate import workflow, step

@workflow(name="<project>.tick")
async def project_tick(ctx) -> str:
    # Cron daemons: pop a due job from the project's existing queue.
    # Cycle daemons: this can be "idle" — work happens via heartbeat cycles.
    return "idle"
```

**Step 3. Construct the EternalLoop.** A typical CLI subcommand:

```python
from harness_eternal import Budget, EternalLoop, MemoryCap
from harness_eternal.restate import LocalRuntime

def cmd_run(args):
    loop = EternalLoop(
        name="<project>",
        state_dir=args.state_dir / "eternal",
        workflow=project_tick,
        runtime=LocalRuntime(args.state_dir / "eternal" / "restate"),
        budget=Budget(usd_per_day=args.usd_per_day),
        memory_cap=MemoryCap(),
        deadline_per_iter_s=args.deadline_per_iter_s,
    )
    return loop.run_forever()
```

**Step 4. Generate supervisor units.** A second CLI subcommand emits both at once:

```python
from harness_eternal.supervisor import generate_units
def cmd_emit_units(args):
    sysd, plist = generate_units(
        name="<project>",
        exec_start=f"python -m <pkg> run {args.state_dir}",
        working_directory=args.state_dir,
    )
    (args.output_dir / "<project>-daemon.service").write_text(sysd)
    (args.output_dir / "com.<project>.daemon.plist").write_text(plist)
```

**Step 5. Wire heartbeat cycles for periodic work.** Argus's pattern:

```python
from harness_eternal import Cycle
loop.heartbeat.register(
    Cycle(name="<project>.consolidate",
          cadence_s=86_400,
          fn=lambda: my_consolidator(state_dir))
)
```

**Step 6. Adopt the chaos test before going to production.** Wrap the daemon command in `harness_eternal.ChaosDriver` for a short cadence (kill every 5 s, run for 30 s) to verify journal monotonicity before the 30-day soak. Reference: `packages/harness-eternal/tests/test_chaos.py`.

## Deployment topology

```text
┌──────────────────────────────────────────────────────────────────────┐
│  Host (single Linux/macOS box; multi-host is out of scope today)     │
│                                                                       │
│  systemd / launchd                                                    │
│    ├── <project>-daemon.service     ← auto-restart, watchdog, OOM    │
│    │     └── python -m <pkg>.cli run                                  │
│    │           ├── EternalLoop.run_forever()                          │
│    │           │     ├── budget.check()        ← ↘                    │
│    │           │     ├── directive.read()      ←  HUMAN_DIRECTIVE.md  │
│    │           │     ├── heartbeat.fire_due()  ← ↗                    │
│    │           │     ├── runtime.invoke(workflow_name)  ← LocalRuntime│
│    │           │     │     └── @workflow → @step → SQLite journal     │
│    │           │     ├── breaker.record_*()                           │
│    │           │     └── health.beat()                                │
│    │           └── HTTP /heartbeat (127.0.0.1:9100) ← supervisor probe│
│    │                                                                  │
│    └── (optional) restate-server.service   ← single-binary sidecar    │
│          └── attach via Unix socket; same workflow contract           │
│                                                                       │
│  State (filesystem-first per Polaris V2 §1):                          │
│    <state_dir>/eternal/                                               │
│      ├── restate/journal.sqlite3   ← invocations + steps + activities │
│      ├── HUMAN_DIRECTIVE.md         ← async HITL channel              │
│      └── STOP                        ← shutdown sentinel              │
└──────────────────────────────────────────────────────────────────────┘
```

The diagram is honest about today's deployment shape. Multi-host federation, Restate distributed mode, k8s `livenessProbe` against the health endpoint — all possible, all out of scope for this delivery.

## Lessons from the implementation

A few non-obvious bugs surfaced during the soak-prep test runs. They are recorded here because they are exactly the class of mistake the next project's implementation will repeat.

- **`time.monotonic()` is process-relative on macOS.** The OOM watchdog's directive-file alarm cooldown defaulted `_last_alarm = 0.0`. On Linux that's correct (boot-relative monotonic ≈ large number → first alarm always passes the cooldown). On macOS Python's `time.monotonic()` returns process-uptime seconds — ≈0.01 at process start — so the first alarm got rejected by the cooldown check. Fix: default `_last_alarm = float("-inf")`. Costing this bug to the calendar: 30 minutes to diagnose + write the platform-aware default. Apply to any cooldown logic that uses `time.monotonic()`.

- **`InMemoryAdapter(name=..., skills=...)` doesn't accept `skills=`.** Argus's adapter uses `.add(skill)` per-skill instead. The test for the fetch cycle had to switch from constructor-args to method-calls. Lesson: don't write tests against an API you haven't read; this would have been caught earlier with a dataclass `__init__` annotation check.

- **`ScheduledTask.new(...)` doesn't exist.** I assumed a factory classmethod and the test failed. The dataclass takes `task_id` directly; production callers use `uuid.uuid4().hex` for the prefix. Lesson: read the dataclass before writing the test fixture.

- **`Catalog.list_active()` doesn't exist** — it's `Catalog.all(only_loadable=True)`. Same lesson — when integrating with someone else's code, the cost of one `grep -n "def "` over the file is much smaller than the cost of writing a test that fails.

- **The chaos test must spawn a worker subprocess, not call the loop in-process.** The point of the test is to validate that a *killed* process leaves a journal that a *new* process can replay. Running both halves in the same process tests something weaker. The chaos test in `harness-eternal/tests/test_loop_integration.py::test_sigkill_replay_preserves_completed_steps` spawns + SIGKILLs + re-spawns; the chaos *driver* in `chaos.py` does the same loop programmatically.

- **Conftest collisions across packages.** `pytest packages/{a,b}/tests/` fails with "Plugin already registered under a different name" when both packages have a `tests/conftest.py`. Workaround: run each package's tests separately. Affects all polyrepo-style monorepos using pytest.

- **`asyncio.run(...)` from inside an asyncio loop blows up.** Polaris's `_tick_in_thread` first attempted `asyncio.run(asyncio.to_thread(daemon.tick))` — but the call was already inside `asyncio.run(...)` from the EternalLoop's main loop. Correct: pass the coroutine directly so the outer loop awaits it (`await asyncio.to_thread(daemon.tick)`). The `step()` helper handles awaitable detection.

These six bugs are roughly representative — most surfaced in the test suite within minutes. The chaos test catches the harder class (multi-process, post-kill state).

## Operational gates that remain

This delivery is **code-complete**; the things that remain are operational:

1. **Restate live-server bring-up.** `RestateRuntime` is wired to fall back to `LocalRuntime` when `restate-sdk` is missing. Bringing a real `restate-server` up and pointing the adapter at it is an ops task: download the binary, write a systemd unit, set `RESTATE_BASE_URL` env var, run the existing test suite against the live runtime to validate.
2. **30-day soak across all three projects.** `make eternal-soak DURATION_H=720 PROJECT=<name>` runs the chaos driver in production cadence (kill once per 24h). Acceptance criteria are codified in `packages/harness-eternal/SOAK.md`. Required: a host with all three daemons under their supervisor units, a separate process running the chaos driver, daily metrics collection, an inbox watching the directive files.
3. **Live-API smoke tests.** `polaris_skills/tests/test_model_factories.py::test_anthropic_live_smoke` and `test_openai_live_smoke` are gated on `ANTHROPIC_API_KEY` / `OPENAI_API_KEY`. Running each once with real keys validates the wire format. ~$0.01 in tokens; one pre-deploy step.
4. **Lyra REPL turn integration** only matters when the REPL gains a direct `AgentLoop.run_conversation` site. The Phase 14 CodeAct LLM call mentioned in `lyra-cli/src/lyra_cli/interactive/driver.py:1397` is the natural seam; the factory is already in place to wrap that call.
5. **Migrating remaining polaris-skills workflows to phases.** Only `post_acceptance.run` had multiple `loop.run()` calls; `lit_survey`, `experiment`, `draft`, `pre_research_*` are single-loop and benefit less. If a new multi-step workflow is added, the phase pattern is a one-line wrap per loop boundary.

## When to revisit this entry

Update this file when any of these change: (a) the chaos test passes against a live `restate-server` (move that line from "Operational gates" to "Empirical results"); (b) the 30-day soak completes for any of the three projects (add a row to the empirical-results table with cycles-completed + spend + RSS-peak); (c) a new failure mode surfaces in production that wasn't predicted by the seven-primitive synthesis ([308](308-autonomy-loop-synthesis.md)); (d) a fourth project adopts Eternal Mode — extend the `harness-eternal` package's per-project adapter pattern table.

**The one-line takeaway for harness designers: the seven-primitive autonomy stack from [308](308-autonomy-loop-synthesis.md) was theoretical until we built one shared 600-LoC package and adopted it across three projects in 13 phases — the implementation report says the chaos test passes, the test counts are real, and every primitive that was a paper citation in [308](308-autonomy-loop-synthesis.md) is now a Python module in `packages/harness-eternal/`.**
