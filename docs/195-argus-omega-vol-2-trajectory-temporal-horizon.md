# 195 — Argus Omega Vol. 2: Trajectory Simulation, Durable Execution, Horizon-Aware Routing, ReWOO Planning, and Chaos-Engineered Hygiene

> **Continuation of [194-argus-omega-enhanced-design](194-argus-omega-enhanced-design.md).** Vol. 1 introduced five structural reframes (recursive cascade, capability × regime grid, Talent / Container / HR, heterogeneous federation, co-evolving verifier + RL + anti-hacking). Vol. 2 adds **five more reframes** drawn from corners of the corpus that map directly to Argus components but Vol. 1 didn't fully tap: trajectory simulation as the L2 substrate, Temporal as the durable execution layer, HORIZON-style failure attribution, ReWOO-style decoupled skill planning, and bitemporal + chaos-engineered catalog hygiene. Plus a concrete implementation skeleton for the strongest combination — Argus Omega-Lite — that any team can ship in 4–6 weeks.

**Status.** Plan + skeleton, not running implementation. Composes additively with Vol. 1; nothing in Vol. 1 needs to change.

**Reading order.** §1 (why more reframes). §2 (the five new reframes 6–10, mapped to source docs). §3 (the implementation skeleton — concrete Python module structure for Omega-Lite). §4 (capabilities delta on Vol. 1's 83). §5 (failure modes delta). §6 (phasing — delta on Vol. 1's B0–B16, new C0–C8 phases). §7 (bright lines additions). §8 (success criteria additions). §9 (one-paragraph summary).

---

## §1 — Why a Vol. 2

Vol. 1 already did substantial work. So why more? Three reasons.

1. **The corpus has direct-fit ideas Vol. 1 didn't fully exploit.** [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md) is *literally* the architecture for Argus's L2 layer; Vol. 1 referenced it once, not as a load-bearing component. [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md) is the obvious substrate for any production-grade agent that runs hours-to-days, but Vol. 1 didn't address durability beyond the witness lattice. [27-horizon-long-horizon-degradation](27-horizon-long-horizon-degradation.md) gives Argus a failure-attribution taxonomy it currently lacks. [17-rewoo](17-rewoo.md) is the cost-efficient analog of Argus's cascade for *multi-skill plans*. [53-chaos-engineering-next-era](53-chaos-engineering-next-era.md) is the verification discipline Vol. 1's COD probes hint at but don't operationalize.

2. **Vol. 1 is good at *shape*, less good at *operationalization*.** The five reframes there are conceptually clean; implementing them on top of v1.0 is non-obvious. Vol. 2 includes a concrete implementation skeleton — file paths, module signatures, dataclasses — so the gap from "we should do this" to "here's the file" closes.

3. **Production-agent reality bites at multi-day horizons.** Vol. 1's RL-trained policy assumes the policy lives in one process. The user's actual deployment surface (Polaris, Lyra, multi-day projects) lives across many processes, many hosts, many days, with crashes, deploys, rate limits, and human approvals interleaved. Without a durable execution substrate, every Vol. 1 reframe (catalog evolution, surrogate verifier, RL policy, periodic decoupled training) silently degrades in a real production environment. Reframe 7 fixes this.

---

## §2 — Five additional reframes

### Reframe 6 — Argus L2 *is* a trajectory-simulation agent (not just a "predictor with a prediction head")

**Source: [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md). Supporting: [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [16-plan-and-solve](16-plan-and-solve.md), [25-agentic-rag](25-agentic-rag.md), [105-agentic-world-modeling-visual-generation](105-agentic-world-modeling-visual-generation.md), [190-agentic-world-modeling-taxonomy](190-agentic-world-modeling-taxonomy.md).**

Vol. 1 specified Argus L2 as a "trajectory predictor" that emits (cost, latency, success-probability, intermediate states) per candidate skill set, with a separate COD probe module. That framing is *too narrow*: it treats L2 as a regression head, when [142] shows the right pattern is **simulator-augmented agent** — an LLM (the "L2 reasoner") that *iteratively explores* a domain simulator, runs scenarios, drills into edge cases, and produces a narrative-grounded prediction with simulator citations.

**The fix.** Promote Argus L2 from a regression module to a full simulator-augmented agent:

- **L2 has a simulator backend.** A Python simulator that, given (query, candidate_skill_set, environment_snapshot), runs a deterministic-or-Monte-Carlo simulation of the trajectory: predicted tool calls, predicted intermediate states, predicted token costs, predicted success/failure, predicted intermediate failure modes. The simulator can be lightweight (skill metadata + cost model + failure-rate priors) or heavy (full agent emulation in a sandbox).
- **L2 has a reasoner over the simulator.** An LLM that generates *scenarios* to run: "what if we replace skill A with B?", "what if the user asks a follow-up that needs C?", "what if the chosen tool returns an error?" Each scenario is a simulator call; results feed the next scenario.
- **L2 emits a structured rationale**, not just a number. The output is `(predicted_outcome, simulator_evidence_set, confidence_interval, top-3_failure_modes)`. The host harness can inspect the simulator-evidence set; the user can audit; downstream L3 can diagnose.
- **L2 is iterative**, like [142]'s loop: hypothesis → scenario → simulator-call → observation → next hypothesis. Vol. 1's "L2 simulator" was a one-shot predictor; the right shape is the iterative loop.

**Concrete consequences.**
- COD probes become a *natural sub-output* of the L2 simulator-loop, not a separate module: every iteration runs counterfactuals as scenarios.
- L2 outputs are *citeable* — the L3 evolver can trace any catalog-edit decision back to specific simulator runs ("retired skill X because COD probe runs 47, 51, 84 showed substitutability with Y under all common queries").
- L2 confidence becomes calibrated against simulator coverage: "L2 ran 12 scenarios; 11 agreed; 1 outlier" is a stronger signal than "L2 said 0.87."
- The simulator backend itself becomes an evolvable asset: skill-cost models, skill-failure-rate priors, skill-interaction matrices all sit inside the simulator and are tunable per regime.

**Mapping to Argus structure.**
- `argus/l2/simulator.py` (Vol. 1) → splits into `simulator/backend.py` (the runnable simulator) and `simulator/reasoner.py` (the LLM-driven scenario explorer).
- `argus/l2/cod.py` (Vol. 1) → folded into `simulator/scenarios/cod.py` as a scenario type; other scenario types: `intervention.py`, `recovery.py`, `composition.py`, `failure_mode.py`.
- New: `argus/l2/citeable_output.py` — the structured-rationale emitter.

### Reframe 7 — Argus runs on Temporal as its durable execution substrate

**Source: [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md). Supporting: [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md), [123-robustness-fault-tolerance](123-robustness-fault-tolerance.md), [131-temporal-bitemporal-tables](131-temporal-bitemporal-tables.md).**

Argus has long-horizon work: 24/7 Ralph loop, periodic decoupled training, COD probe sweeps, marketplace pulls, drift validation, RL micro-batches. Each runs across crashes, deploys, rate limits, and human approvals. Vol. 1 acknowledged "the daemon-style background loop runs on Polaris's heartbeat-scheduler shape" — that's not enough. The right substrate is a workflow engine: code reads as if failure doesn't exist; the engine makes it true.

**The fix.** Argus's long-running loops become Temporal workflows. The split:

- **Workflows (deterministic, replay-safe):**
  - `RalphLoopWorkflow` — the 24/7 catalog hygiene loop. Schedules itself, signals can adjust cadence.
  - `CODProbeBatchWorkflow` — runs N COD probes across catalog pairs, persists results to the witness lattice.
  - `DecoupledTrainingWorkflow` — every K RL steps, executes the periodic-decoupled-training phase. Survives multi-hour reward-disabled exploration.
  - `MarketplacePullWorkflow` — pulls Anthropic/Glama/Smithery/MCPfinder, runs vulnerability scan on each, tier-assigns, persists. May suspend for hours awaiting HITL approval on T-Reviewed promotions.
  - `HRReviewWorkflow` — every 30 invocations or 7 days per Talent, runs the review → PIP → offboarding cycle. Multi-day suspensions while awaiting human approval on borderline cases.
  - `EvolverPersistenceWorkflow` — applies a catalog edit, runs validation, awaits regression-gate clearance, commits or rolls back.

- **Activities (side-effecting, retried by Temporal):**
  - `embed(skill)`, `cross_encode(query, skill)`, `run_simulator_scenario(scenario)`, `vulnerability_scan(skill)`, `pull_marketplace(source)`, `query_llm(prompt)`, `mcp_invoke(server, args)`, `apply_catalog_edit(diff)`.

- **Signals and queries** for live introspection:
  - `signal: pause()`, `signal: resume()`, `signal: tier_change(skill, new_tier)`, `signal: hitl_approve(action_id)`.
  - `query: status() -> {phase, last_review, pending_approvals, cost_to_date}`.

**Concrete consequences.**
- **Crash recovery is free.** A worker dies mid-RL-step; a fresh worker reads the event history and replays to the failure point. Vol. 1's RL trainer would have lost the partial step.
- **Multi-day workflows are native.** A `MarketplacePullWorkflow` waiting 3 days for human approval costs zero compute while suspended. Vol. 1's design implied either polling or bespoke state machines.
- **Idempotency by construction.** Workflow IDs prevent duplicate Ralph loops, duplicate COD batches, duplicate marketplace pulls. The whole class of "did we run this twice?" bugs disappears.
- **Deploy-without-loss.** Argus can be deployed mid-RL-cycle; Temporal replays in-flight workflows on the new code (with `workflow.getVersion()` for backwards compat).
- **Witness lattice integrates naturally.** Every activity result becomes an event-history entry; the witness lattice's signed audit trail is *partially derivable* from the event history. Both can coexist; the witness lattice signs the *workflow-level* decisions, the event history records the *activity-level* details.

**Mapping to Argus structure.**
- `argus/workflows/` — new top-level package.
  - `ralph.py`, `cod_probe.py`, `decoupled_training.py`, `marketplace_pull.py`, `hr_review.py`, `evolver.py`.
- `argus/activities/` — new top-level package.
  - `embed.py`, `cross_encode.py`, `simulator.py`, `vulnerability.py`, `marketplace.py`, `llm.py`, `mcp.py`, `catalog_edit.py`.
- `argus/temporal/client.py` — Temporal client wrapper.
- `argus/temporal/worker.py` — workflow + activity worker registration.
- For dev/test: SQLite Temporal cluster (`temporalite`); for prod: Temporal Cloud or self-hosted.

### Reframe 8 — Horizon-aware skill routing with failure-class attribution

**Source: [27-horizon-long-horizon-degradation](27-horizon-long-horizon-degradation.md). Supporting: [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [38-claw-eval](38-claw-eval.md), [8-context-compaction](08-context-compaction.md), [10-multi-session-continuity](10-multi-session-continuity.md), [16-plan-and-solve](16-plan-and-solve.md).**

Argus v1.0 + Vol. 1 score skills on per-call success/failure plus trajectory-level eval. Neither attributes failures to *classes* of failure. HORIZON's contribution is the taxonomy: **context-forget**, **plan-divergence**, **tool-misuse**, **recovery-failure**, **verification-gap**, **environment-error** — each scaling differently with horizon. Argus should route *with horizon-class awareness* and *attribute failures with horizon-class evidence*.

**The fix.** Three layers of integration:

1. **Skill metadata: per-failure-class effectiveness.** Every catalog entry carries a profile: how often does this skill fail in each class? Per-class effectiveness is part of the capability signature ([191](191-onemancompany-skills-to-talent.md), Vol. 1 Reframe 3). A "context-forget-mitigator" skill has high mitigation score in that class.

2. **Routing: horizon-class-aware tier weights.** When the trajectory is short (≤ 5 steps), HORIZON's research says context-forget is rare; weight tier 0/1 routing more on tool-misuse and verification-gap mitigators. When the trajectory is long (≥ 30 steps), context-forget and plan-divergence dominate; up-weight skills that mitigate those classes (compaction skills, plan-and-solve scaffolds, multi-session-continuity helpers).

3. **Evolution: HORIZON-style failure attribution as L3 evidence.** When a trajectory fails, the L3 evolver runs the [27] LLM-as-Judge attribution pipeline on the trajectory: which step broke, which class of failure, evidence quote. The attribution becomes the *diagnosis* signal for catalog edits: "the routing of skill X at step 12 caused a plan-divergence failure" → demote X for plan-divergence-prone trajectory shapes; rewrite description; potentially split into two skills (one for short-horizon, one with explicit plan scaffolding).

**Concrete consequences.**
- The catalog gains a **failure-class × horizon-bucket effectiveness matrix** per skill: rows = failure classes, columns = horizon buckets (1–5, 6–15, 16–30, 31–100, 100+), values = mitigation effectiveness.
- The routing policy (Vol. 1 Reframe 5's SkillRL) gets new features: predicted-trajectory-length and predicted-dominant-failure-class. These come from the L2 simulator (Reframe 6).
- The L3 evolver's diagnose stage uses HORIZON attribution; the distill stage's catalog edit can be class-targeted.
- Cross-references: skills in [8-context-compaction](08-context-compaction.md), [10-multi-session-continuity](10-multi-session-continuity.md), [16-plan-and-solve](16-plan-and-solve.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) become *first-class horizon-mitigation skills* in Argus's catalog with explicit class-coverage tags.

### Reframe 9 — ReWOO-style decoupled multi-skill planning

**Source: [17-rewoo](17-rewoo.md). Supporting: [16-plan-and-solve](16-plan-and-solve.md), [3-plan-mode](03-plan-mode.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md), [42-langchain-deep-agents](42-langchain-deep-agents.md).**

Vol. 1's R7/R8 (multi-skill activation, skill chaining suggestion) returns top-K skills or a 2-skill compound; the host harness then figures out how to *use* them. That's an interleaved ReAct-style pattern: pick skill, run, observe, pick next, run, observe. ReWOO showed that *one planning call + parallel execution + one solver call* yields 5× fewer LLM calls and matches accuracy. Argus should emit a **SkillPlan** (a DAG of skills with placeholders for outputs), not just an active set.

**The fix.** Argus's output type changes for multi-skill cases:

```
ActiveSkillSet (Vol. 1 default) — flat top-K skills
SkillPlan (Vol. 2 enrichment) — DAG of skill invocations with placeholders
```

A SkillPlan looks like:

```yaml
plan:
  - id: s1
    skill: web-search
    args: {q: "{user_query}"}
  - id: s2
    skill: csv-fetch
    args: {url: "$s1.first_url"}
  - id: s3
    skill: chart-render
    args: {data: "$s2.data", spec: "bar"}
    depends_on: [s2]
solver: synthesize-final-answer
```

**The loop:**

1. **Argus L1 + L2** produces the SkillPlan. The L2 simulator runs the plan as a scenario; if simulated success-probability is high, emit the plan; otherwise iterate.
2. **The host harness's executor** runs the plan: parallel execution of independent steps (s1, s2 if no dependency), serial wait on dependencies (s3 waits on s2). One LLM "solver" call at the end synthesizes from `$s1`, `$s2`, `$s3`.
3. **L2 simulator's predicted trajectory** is *the plan*; predictions on intermediate-state (`$s1.first_url` shape, `$s2.data` shape) become validation gates.

**Concrete consequences.**
- Multi-skill trajectories become 2 LLM calls (planner + solver) instead of N (one per skill turn). 5× cost reduction matches [17]'s benchmark.
- Independent skills run in parallel; wall-clock time drops to critical-path length.
- The plan is *cite-able* — the host harness, the user, and the L3 evolver can all inspect the DAG.
- Plan-mode integration ([3-plan-mode](03-plan-mode.md)): plan mode emits the SkillPlan for approval; execute mode runs it. Natural alignment.
- Edge case: when later steps genuinely depend on reasoning over earlier outputs (where ReAct still wins), Argus emits a partial SkillPlan with a "ReAct branch" leaf — a skill that says "run a ReAct loop with the following sub-catalog."

**Mapping to Argus structure.**
- `argus/output/skill_plan.py` — SkillPlan dataclass + serializer.
- `argus/output/plan_executor.py` — reference DAG executor (host harnesses can use this or write their own).
- `argus/l2/plan_simulator.py` — runs the plan in simulation; predicts intermediate-state shapes.
- `argus/l1/plan_router.py` — generates SkillPlan candidates from cascade.

### Reframe 10 — Bitemporal catalog with chaos-engineered hygiene

**Source: [131-temporal-bitemporal-tables](131-temporal-bitemporal-tables.md), [53-chaos-engineering-next-era](53-chaos-engineering-next-era.md). Supporting: [26-linuxarena-production-agent-safety](26-linuxarena-production-agent-safety.md), [123-robustness-fault-tolerance](123-robustness-fault-tolerance.md), [82-poisonedrag](82-poisonedrag.md), [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md).**

Vol. 1 added temporal validity (`valid_at` / `invalid_at`) and structural anti-poisoning. Two additions sharpen the design: **bitemporal tracking** (system-time + valid-time) for full historical reproducibility, and **chaos engineering** as the verification discipline that exercises the catalog's defenses continuously.

**The fix — Part A: bitemporal catalog.** Every catalog mutation records both:

- **Valid-time:** when the change is *true in the world* (e.g., "skill X became valid on 2026-04-01 when vendor shipped API v2").
- **System-time:** when Argus *learned* about the change (e.g., "Argus ingested the v2 metadata on 2026-04-15").

Every routing decision queries the catalog "as of valid-time T_v, system-time T_s." Replay (witness-lattice replay, Vol. 1 §C9) becomes deterministic: a past decision replays under the catalog state Argus *knew* at the time, not the current state. This solves a subtle bug Vol. 1 didn't address: a witness from 2026-04-10 references skill X v1, but X has since been retracted; without bitemporal tracking, replay is broken.

**The fix — Part B: chaos engineering as continuous catalog hygiene.** [53] frames chaos engineering for agentic systems. Argus operationalizes it:

- **Planted-failure skills.** Every N hours, inject a synthetic skill into the catalog that's deliberately broken (returns wrong type, throws on certain inputs, takes 10× expected time). Argus's drift detector should catch and demote within K cycles. Track time-to-detection as a health metric.
- **Planted-poison skills.** Every N hours, inject a skill with adversarial description (prompt-injection, hidden Unicode, ANSI escapes, embedding-collision attack). Argus's vulnerability scanner should catch on import or at most on first scheduled scan.
- **Planted-substitutables.** Every N hours, inject a skill that's near-duplicate of an existing high-tier skill. COD probes should flag as substitutable within K probe cycles; F6 consolidation should fire.
- **Planted-degradation.** Every N hours, mutate an existing skill's description to be slightly worse (less specific, missing keyword). Description-rewriter (F3) should detect and propose a corrected rewrite.
- **Chaos-budget gate.** All planted skills are tagged in a dedicated "chaos partition" of the catalog; routing logic excludes them from production routing decisions; they only exist to exercise defenses.

**Concrete consequences.**
- **Catalog history is fully reproducible.** Any past witness can be replayed exactly. Compliance, audit, debugging all benefit.
- **Defense systems are continuously tested.** Vol. 1's vulnerability scanner, drift detector, COD probe, description rewriter, and consolidation each get a measurable health signal: time-to-detection on planted failures.
- **Time-to-detection becomes a SLO.** "Argus must detect a planted-poison skill within 6 hours" is a testable promise, not a hope.
- **Adversarial robustness is operationalized.** Vol. 1 had structural anti-poisoning; Vol. 2 has *continuous adversarial pressure* on the system, modeled after [26-linuxarena-production-agent-safety](26-linuxarena-production-agent-safety.md)'s sabotage-task pattern.

**Mapping to Argus structure.**
- `argus/catalog/bitemporal.py` — bitemporal storage layer; every read takes (T_v, T_s) parameters.
- `argus/chaos/planter.py` — planted-failure / planted-poison / planted-substitutable / planted-degradation injectors.
- `argus/chaos/budget.py` — chaos-partition isolation; ensures planted skills never reach production routing.
- `argus/chaos/slo.py` — time-to-detection SLOs; alerts on violations.
- `argus/observability/replay.py` — bitemporal replay UI; pick (T_v, T_s) and replay any past decision.

---

## §3 — Implementation skeleton: Argus Omega-Lite (4–6 weeks)

The strongest combination of Vol. 1 + Vol. 2 ideas that's deployable without huge ML infrastructure investment is **Omega-Lite**: latent cascade + semantic-entropy gating + Temporal substrate + ReWOO planning + bitemporal catalog with chaos engineering + witness lattice. Skips the heavy RL training (Vol. 1 B8–B10) and full L2 simulator (Reframe 6's iterative reasoner).

Concrete file structure to ship:

```
projects/argus/
├── pyproject.toml
├── README.md
├── src/argus_omega/
│   ├── __init__.py
│   ├── catalog/
│   │   ├── __init__.py
│   │   ├── bitemporal.py            # B-temporal storage (Reframe 10A)
│   │   ├── graph.py                 # Typed skill KG (Vol.1 R2 + C.1)
│   │   ├── talent.py                # Talent envelope (Vol.1 R3)
│   │   └── container.py             # Six typed Container interfaces (Vol.1 R3)
│   ├── tiers/
│   │   ├── __init__.py
│   │   ├── tier_0_progressive.py    # Tier 0 with capability signatures
│   │   ├── tier_1_keyword.py        # BM25
│   │   ├── tier_2_embedding.py      # sqlite-vec embeddings
│   │   ├── tier_3_rerank.py         # cross-encoder + KG features
│   │   ├── tier_4_navigate.py       # KG traversal
│   │   └── recursive_link.py        # Inner + Outer Links (Vol.1 R1)
│   ├── gating/
│   │   ├── __init__.py
│   │   ├── semantic_entropy.py      # Vol.1 R12, source: [88]
│   │   └── confidence.py            # learned confidence head (optional)
│   ├── output/
│   │   ├── __init__.py
│   │   ├── active_set.py            # flat top-K (Vol.1 default)
│   │   ├── skill_plan.py            # ReWOO-style DAG (Reframe 9)
│   │   └── plan_executor.py         # reference DAG executor
│   ├── l2/
│   │   ├── __init__.py
│   │   ├── simulator/
│   │   │   ├── backend.py           # cost-model + failure-rate priors
│   │   │   └── reasoner.py          # LLM-driven scenario explorer (Reframe 6)
│   │   ├── scenarios/
│   │   │   ├── cod.py               # counterfactual outcome deviation
│   │   │   ├── intervention.py      # what-if substitutions
│   │   │   ├── recovery.py          # failure-recovery paths
│   │   │   └── composition.py       # multi-skill composition
│   │   ├── plan_simulator.py        # simulates SkillPlans (Reframe 9)
│   │   └── citeable_output.py       # structured rationale
│   ├── horizon/
│   │   ├── __init__.py
│   │   ├── attribution.py           # HORIZON LLM-as-Judge (Reframe 8)
│   │   ├── failure_classes.py       # context-forget / plan-divergence / etc.
│   │   └── matrix.py                # per-skill failure-class × horizon-bucket
│   ├── chaos/
│   │   ├── __init__.py
│   │   ├── planter.py               # injectors (Reframe 10B)
│   │   ├── budget.py                # chaos-partition isolation
│   │   └── slo.py                   # time-to-detection SLOs
│   ├── witness/
│   │   ├── __init__.py
│   │   ├── lattice.py               # signed audit trail
│   │   └── replay.py                # bitemporal replay
│   ├── workflows/                   # Temporal workflows (Reframe 7)
│   │   ├── __init__.py
│   │   ├── ralph.py                 # 24/7 loop
│   │   ├── cod_probe.py             # COD batch
│   │   ├── marketplace_pull.py
│   │   ├── hr_review.py
│   │   └── evolver.py               # L3 catalog edit
│   ├── activities/                  # Temporal activities (Reframe 7)
│   │   ├── __init__.py
│   │   ├── embed.py
│   │   ├── cross_encode.py
│   │   ├── simulator.py
│   │   ├── vulnerability.py
│   │   ├── marketplace.py
│   │   ├── llm.py
│   │   ├── mcp.py
│   │   └── catalog_edit.py
│   ├── temporal/
│   │   ├── __init__.py
│   │   ├── client.py                # Temporal client wrapper
│   │   └── worker.py                # workflow + activity registration
│   ├── governance/
│   │   ├── __init__.py
│   │   ├── budgets.py
│   │   └── bright_lines.py          # 22+ codes (Vol.1 §8 + Vol.2 §7)
│   └── api.py                       # public Python SDK
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── chaos/                       # planted-failure regression tests
│   └── replay/                      # bitemporal replay tests
└── argus/                           # Claude Code skill persona
    ├── SKILL.md
    ├── scripts/
    │   ├── search.py
    │   ├── refine.py
    │   └── audit.py
    └── reference/
        ├── tier-ladder.md
        ├── failure-modes.md
        └── chaos-slo.md
```

### Public API sketch

```python
# argus/api.py — what host harnesses see

from argus_omega.catalog import Talent, BitemporalCatalog
from argus_omega.output import ActiveSkillSet, SkillPlan
from argus_omega.witness import RoutingWitness

class Argus:
    def find(
        self,
        query: str,
        *,
        mode: str = "auto",                    # "keyword" | "semantic" | "auto"
        plan_mode: bool = False,               # if True, returns SkillPlan
        horizon_hint: int | None = None,       # expected trajectory length
        as_of: tuple[Datetime, Datetime] | None = None,  # bitemporal (Tv, Ts)
        cost_cap_usd: float = 0.01,
        latency_cap_ms: int = 1000,
    ) -> ActiveSkillSet | SkillPlan: ...

    def explain(self, witness: RoutingWitness) -> str:
        """Human-readable reasoning trace for a past decision."""

    def replay(self, witness: RoutingWitness) -> ActiveSkillSet | SkillPlan:
        """Bitemporal replay — reproduces the original decision."""

    async def start_ralph(self) -> WorkflowHandle: ...
    async def signal_pause(self, workflow_id: str) -> None: ...
    async def query_status(self, workflow_id: str) -> dict: ...
```

### Minimal Talent dataclass

```python
@dataclass(frozen=True)
class Talent:
    # Identity
    name: str
    role: str
    version: str
    # Body
    system_prompt: str
    working_principles: list[str]
    tool_configurations: dict
    skill_scripts: list[ScriptRef]
    knowledge_files: list[FileRef]
    # Capability signature
    capability_signature: CapabilitySignature
    # Trust + temporal
    trust_tier: TrustTier
    valid_at: Datetime
    invalid_at: Datetime | None
    content_sha: str
    signature_sha: str
    # Modality (Vol.1 R4)
    skill_kind: Literal["K1", "K2", "K3", "K4"]
    modality: Literal["text", "time-series", "tabular", "3d", "molecular", ...]
    # Horizon awareness (Vol.2 Reframe 8)
    failure_class_effectiveness: dict[FailureClass, dict[HorizonBucket, float]]
```

### Witness lattice record

```python
@dataclass(frozen=True)
class RoutingWitness:
    witness_id: str                   # SHA of all fields below
    prev_witness_id: str              # chains to previous witness (immutable history)
    timestamp_system: Datetime
    timestamp_valid: Datetime         # bitemporal
    query_sha: str
    cascade_trace: list[TierEvent]
    selected: ActiveSkillSet | SkillPlan
    capability_signatures_used: list[str]  # SHAs of every Talent's signature at time
    cost_usd: float
    latency_ms: int
    instance_signature: str           # signed by Argus's instance key
    chain_signature: str              # signed prev_witness_id || witness_id
```

### Temporal workflow sketch

```python
@workflow.defn
class RalphLoopWorkflow:
    def __init__(self):
        self._paused = False
        self._cadence_sec = 600       # 10 min default

    @workflow.run
    async def run(self) -> None:
        while True:
            if self._paused:
                await workflow.wait_condition(lambda: not self._paused)
            # cycle 1: refresh telemetry roll-ups
            await workflow.execute_activity(
                refresh_telemetry, schedule_to_close_timeout=timedelta(minutes=5)
            )
            # cycle 2: COD probe batch (handed to child workflow for parallelism)
            await workflow.execute_child_workflow(
                CODProbeBatchWorkflow.run, n_pairs=50
            )
            # cycle 3: vulnerability re-scan
            await workflow.execute_activity(
                vulnerability_rescan_pending, schedule_to_close_timeout=timedelta(minutes=15)
            )
            # cycle 4: chaos planter (every 4 hours)
            if workflow.now().hour % 4 == 0:
                await workflow.execute_activity(plant_chaos_skill)
            # sleep and continue
            await workflow.sleep(self._cadence_sec)

    @workflow.signal
    def pause(self) -> None:
        self._paused = True

    @workflow.signal
    def resume(self) -> None:
        self._paused = False

    @workflow.query
    def status(self) -> dict:
        return {
            "paused": self._paused,
            "cadence_sec": self._cadence_sec,
            "now": str(workflow.now()),
        }
```

This is the *runnable* shape of Argus Omega-Lite. Each module is independently testable; the Temporal layer makes the long-running parts durable; the ReWOO output type unlocks parallel multi-skill execution; the chaos partition continuously exercises defenses.

---

## §4 — Capabilities delta (additive on Vol. 1's 83)

| # | Capability | Source | Layer |
|---|---|---|---|
| **R18** | Iterative L2 simulator-augmented reasoner | [142](142-trajectory-simulation-agents.md) | L2 |
| **R19** | Citeable structured rationale with simulator-evidence set | [142](142-trajectory-simulation-agents.md), [135](135-trustworthy-generation.md) | L2 |
| **R20** | SkillPlan output (DAG with placeholders) | [17](17-rewoo.md) | L1+L2 |
| **R21** | Parallel skill execution via ReWOO planner+solver | [17](17-rewoo.md) | Output |
| **R22** | Plan-mode SkillPlan emission for HITL approval | [3](03-plan-mode.md), [17](17-rewoo.md) | L1 |
| **F23** | HORIZON LLM-as-Judge failure attribution on trajectories | [27](27-horizon-long-horizon-degradation.md), [21](21-llm-as-judge-trajectory-eval.md) | L3 |
| **F24** | Per-skill failure-class × horizon-bucket effectiveness matrix | [27](27-horizon-long-horizon-degradation.md) | L3 |
| **F25** | Horizon-class-aware tier weighting | [27](27-horizon-long-horizon-degradation.md), [88](88-confidence-driven-router.md) | L1 |
| **F26** | Horizon-mitigation-skill tagging (compaction, plan-and-solve, multi-session) | [8](08-context-compaction.md), [10](10-multi-session-continuity.md), [16](16-plan-and-solve.md) | Federation |
| **C16** | Bitemporal catalog (system-time + valid-time) | [131](131-temporal-bitemporal-tables.md), [184](184-strongest-memory-techniques-synthesis-may-2026.md) | L3 |
| **C17** | Witness-lattice bitemporal replay | [186](186-mnema-witness-lattice.md), [188](188-witness-provenance-memory-techniques-synthesis.md), [131](131-temporal-bitemporal-tables.md) | Observability |
| **C18** | Chaos partition + planted-failure injection | [53](53-chaos-engineering-next-era.md), [26](26-linuxarena-production-agent-safety.md) | L3 |
| **C19** | Time-to-detection SLOs (per-defense, per-class) | [53](53-chaos-engineering-next-era.md), [125](125-system-level-production-patterns.md) | L3 |
| **D13** | Temporal as durable execution substrate | [150](150-temporal-durable-execution-substrate.md), [124](124-agent-level-production-patterns.md) | All long-running |
| **D14** | Workflow vs activity split for replay-safety | [150](150-temporal-durable-execution-substrate.md) | All long-running |
| **D15** | Multi-day suspension on HITL approval | [150](150-temporal-durable-execution-substrate.md), [23](23-human-in-the-loop.md) | L3 |
| **O10** | HORIZON-style failure-class dashboard | [27](27-horizon-long-horizon-degradation.md), [24](24-observability-tracing.md) | Observability |
| **O11** | SkillPlan DAG visualizer | [17](17-rewoo.md), [3](03-plan-mode.md) | Observability |
| **O12** | Chaos SLO dashboard (time-to-detection per defense) | [53](53-chaos-engineering-next-era.md), [24](24-observability-tracing.md) | Observability |
| **O13** | Bitemporal catalog snapshot browser | [131](131-temporal-bitemporal-tables.md) | Observability |

**Vol. 2 capabilities total: 20.** Argus Omega Vol. 1 + Vol. 2 = 83 + 20 = **103 capabilities**.

---

## §5 — Failure modes delta (additive on Vol. 1's 60)

| # | Failure mode | Argus Omega Vol. 2 countermeasure |
|---|---|---|
| **F-61** | **Simulator over-confidence on small scenario set** — L2 emits high-confidence prediction from 2-3 scenarios | Minimum-scenario-count gate; uncertainty proportional to coverage; "L2 ran K scenarios; need ≥10 for high-confidence" |
| **F-62** | **Simulator-real divergence** — L2 simulator predictions don't match production outcomes | Continuous calibration: log every (predicted, actual) pair; recalibrate cost model + failure-rate priors weekly; Brier-score SLO |
| **F-63** | **SkillPlan dependency-cycle** — generated DAG has implicit cycles | DAG validator on plan emission; cycle detection; plan rejected with explanation |
| **F-64** | **SkillPlan placeholder mismatch** — `$s1.field_x` doesn't exist on s1's actual output | Plan-time schema check using K3/K4 declared schemas; runtime placeholder validation; fall-back to ReAct branch |
| **F-65** | **HORIZON judge drift** — attribution labels shift over time as judge model upgrades | Version-pin the rubric; periodic recalibration against human annotators; rubric-version tag on every attribution |
| **F-66** | **Horizon-class miscalibration on novel task shapes** — class effectiveness matrix wrong for new task domain | Cold-start with priors; refuse high-stakes routing decisions until ≥N attributions in matrix cell |
| **F-67** | **Temporal workflow non-determinism bug** — workflow code does `random()` or `now()` directly, replay diverges | Static analyzer on workflow code (banned imports list); CI gate; runtime replay-divergence detector |
| **F-68** | **Activity-side thrashing on rate limit** — thousands of parallel COD probes hit OpenAI rate limit | Per-activity concurrency caps; Temporal task-queue rate limiting; backpressure signal to RalphLoop |
| **F-69** | **Bitemporal query confusion** — "as of" parameters reversed (system-time and valid-time swapped) | Type-tagged BTime / VTime classes; mixing them is a type error |
| **F-70** | **Chaos planted skill leaks to production** — partition isolation bug routes a chaos skill | Chaos-partition tag in routing-time filter; integration test plants known-chaos skill, asserts it never appears in production responses |
| **F-71** | **Chaos SLO false alert** — defense detected planted skill but recorder missed event | End-to-end test: plant skill, wait for detection event, assert event recorded; alarm only on 2 consecutive misses |
| **F-72** | **ReWOO planner over-decomposition** — 1-skill task decomposed into a 5-step plan | Cost-aware planner; if SkillPlan has > 1 step but L2 simulator predicts single-skill success, fall back to ActiveSkillSet |
| **F-73** | **Replay non-determinism from external state** — replay uses current catalog, not catalog-as-of timestamp | Bitemporal catalog enforces (Tv, Ts) on every read; replay always uses witness's recorded (Tv, Ts) |
| **F-74** | **Workflow signal storm during chaos planting** — chaos planter signals interrupt RalphLoop too often | Per-signal rate limit; signal coalescing; chaos planter has own dedicated workflow |
| **F-75** | **Verifier-replay mismatch** — surrogate verifier's verdict at training time differs from replay | Verifier version tag in witness; replay uses verifier-as-of-witness-version |

**Vol. 2 failure modes total: 15.** Argus Omega total mapped failure modes: 60 + 15 = **75**.

---

## §6 — Phasing — delta on Vol. 1's B0–B16

Vol. 2 introduces phases C0–C8, each composing with Vol. 1 phases.

| Δ-Phase | Title | Depends on | Effort | Deliverable | Reframe |
|---|---|---|---|---|---|
| **C0** | Temporal substrate scaffold | A0 | ~1 wk | `temporal/`, `workflows/`, `activities/` packages; SQLite Temporal cluster for dev; basic `RalphLoopWorkflow` running | R7 |
| **C1** | Bitemporal catalog | A0 | ~1 wk | `catalog/bitemporal.py`; (Tv, Ts) on every read; type-tagged BTime/VTime | R10A |
| **C2** | Witness lattice with bitemporal replay | C1, B12 | ~1 wk | Replay reproduces past decisions exactly; chain-signature integrity test | R10A, Vol.1 C9 |
| **C3** | L2 simulator-augmented reasoner | B6 | ~2 wk | `l2/simulator/{backend.py, reasoner.py}`; iterative scenario loop; citeable output | R6 |
| **C4** | Scenario library + COD as scenario type | C3 | ~1 wk | `l2/scenarios/{cod, intervention, recovery, composition, failure_mode}.py` | R6 |
| **C5** | SkillPlan output type + DAG executor | A4, B6 | ~1.5 wk | `output/{skill_plan.py, plan_executor.py}`; placeholder schema; parallel execution | R9 |
| **C6** | Plan-simulator (simulate the plan, not just the active set) | C3, C5 | ~1 wk | `l2/plan_simulator.py`; predicts intermediate-state shapes; gates on simulated success-prob | R6, R9 |
| **C7** | HORIZON failure-class attribution + matrix | B7 | ~2 wk | `horizon/{attribution.py, failure_classes.py, matrix.py}`; LLM-as-Judge pipeline; per-skill matrix | R8 |
| **C8** | Chaos planter + SLOs | C0, C1 | ~2 wk | `chaos/{planter.py, budget.py, slo.py}`; planted-failure / planted-poison / planted-substitutable / planted-degradation; per-defense time-to-detection SLOs | R10B |

**Total Vol. 2 delta: ~12.5 weeks** for all of C0–C8. **Vol. 2 strongest subset (Omega-Lite delta):** C0 + C1 + C2 + C5 + C8 = **~6.5 weeks** on top of Vol. 1's MVP delta. Combined Omega-Lite (Vol. 1 B0+B2+B12 + Vol. 2 C0+C1+C2+C5+C8) ≈ **~10 weeks** on top of v1.0 MVP, total **~16 weeks** from greenfield.

**Recommended additional staging on top of Vol. 1's stages:**

8. **Stage 8 (Vol. 2 substrate, weeks 39–43):** C0 + C1 + C2 — Temporal + bitemporal + replay. Foundational; everything else benefits.
9. **Stage 9 (Vol. 2 simulator + plan, weeks 44–48):** C3 + C5 + C6 — simulator-augmented reasoner + SkillPlan + plan simulator.
10. **Stage 10 (Vol. 2 attribution + chaos, weeks 49–52):** C4 + C7 + C8 — scenarios + HORIZON attribution + chaos partition.

Total Vol. 1 + Vol. 2 full path: **~52 weeks (1 year)** for the maximalist deployment. Stop-anywhere remains true: every C-phase delivers user-visible value.

---

## §7 — Bright lines additions (Vol. 2 delta)

In addition to v1.0's 10 + Vol. 1's 12 = 22:

| Code | Trip condition | Default action |
|---|---|---|
| `BL-OMEGA-V2-SIM-COVERAGE` | L2 emits high-confidence prediction with < 10 scenarios run | Demote to medium-confidence; emit warning |
| `BL-OMEGA-V2-SIM-DRIFT` | Simulator-real divergence Brier score worsens > threshold | Block routing decisions that depend on L2 confidence; alert |
| `BL-OMEGA-V2-PLAN-CYCLE` | SkillPlan DAG has cycle | Refuse plan; fall back to top-K ActiveSkillSet |
| `BL-OMEGA-V2-PLAN-PLACEHOLDER` | Placeholder references non-existent field | Refuse plan; emit schema-mismatch error |
| `BL-OMEGA-V2-WORKFLOW-NONDET` | Workflow code uses banned non-determinism | CI gate failure; refuse deploy |
| `BL-OMEGA-V2-CHAOS-LEAK` | Chaos-partitioned skill appears in production routing | Halt routing; emergency alert |
| `BL-OMEGA-V2-CHAOS-SLO` | Time-to-detection exceeds SLO for 2+ consecutive plants | Alert on-call; investigate defense regression |
| `BL-OMEGA-V2-REPLAY-DRIFT` | Bitemporal replay produces different result from witness | Halt; integrity audit; possible witness-tampering |
| `BL-OMEGA-V2-HORIZON-COLD` | Routing high-stakes decision in horizon-class cell with < N attributions | Refuse high-stakes routing; demote to "best-effort" |
| `BL-OMEGA-V2-VERIFIER-VERSION` | Replay uses verifier version not recorded in witness | Halt replay; alert |

Total bright lines: **32** (10 v1.0 + 12 Vol. 1 + 10 Vol. 2).

---

## §8 — Success criteria additions

### Stage 8 (substrate, after C0–C2)

- ✅ **Workflow durability**: kill -9 a worker mid-RalphLoop; fresh worker resumes within 30s; no duplicate work.
- ✅ **Bitemporal replay reproducibility**: 100% of past witnesses replay to identical decision.
- ✅ **Chain-signature integrity**: tampering with any past witness is detected immediately on next read.
- ✅ **Multi-day suspension**: HRReviewWorkflow can suspend for ≥7 days awaiting HITL approval, costing zero compute.

### Stage 9 (simulator + plan, after C3 + C5 + C6)

- ✅ **L2 calibration**: Brier score on (predicted, actual) success outcomes ≤ 0.15.
- ✅ **L2 citation completeness**: 100% of L2 outputs cite ≥3 scenario runs in their evidence set.
- ✅ **SkillPlan parallelism**: multi-skill trajectories execute in critical-path time, not sum-of-skills time (≥2× wall-clock speedup on planted multi-skill bench).
- ✅ **ReWOO cost reduction**: ≥3× fewer LLM calls than ReAct-style on multi-step trajectories at matched accuracy.

### Stage 10 (attribution + chaos, after C4 + C7 + C8)

- ✅ **HORIZON attribution accuracy**: κ ≥ 0.80 against held-out human-annotated failures.
- ✅ **Failure-class matrix coverage**: ≥80% of catalog skills have ≥10 attributions per (failure-class, horizon-bucket) cell.
- ✅ **Chaos SLO compliance**: planted-failure detected within 24h; planted-poison within 6h; planted-substitutable within 72h; planted-degradation within 7d.
- ✅ **Chaos zero-leak**: no chaos-partitioned skill ever appears in production routing across 100K production turns.

---

## §9 — One-paragraph summary

Vol. 2 layers five additional reframes onto Vol. 1: **(6) Argus L2 is a simulator-augmented reasoner with iterative scenario exploration and citeable structured rationales** ([142](142-trajectory-simulation-agents.md)); **(7) Argus runs on Temporal as the durable execution substrate, with workflow/activity split, multi-day suspension, and crash-safe replay** ([150](150-temporal-durable-execution-substrate.md)); **(8) routing is horizon-aware, with HORIZON-style LLM-as-Judge failure attribution feeding a per-skill failure-class × horizon-bucket effectiveness matrix that the L3 evolver consumes** ([27](27-horizon-long-horizon-degradation.md)); **(9) multi-skill outputs are SkillPlans (DAG with placeholders, ReWOO-style), not flat top-K, executing in parallel with one planner + one solver call instead of N ReAct turns** ([17](17-rewoo.md)); **(10) the catalog is bitemporal (system-time + valid-time, enabling exact replay of past decisions) and continuously exercised by chaos engineering — planted-failure, planted-poison, planted-substitutable, planted-degradation skills with time-to-detection SLOs** ([131](131-temporal-bitemporal-tables.md), [53](53-chaos-engineering-next-era.md), [26](26-linuxarena-production-agent-safety.md)). Vol. 2 also ships an implementation skeleton for **Argus Omega-Lite** — the deployable subset of Vol. 1 + Vol. 2 (latent cascade + semantic-entropy gating + Temporal substrate + ReWOO planning + bitemporal catalog + witness lattice + chaos partition) shippable in ~10 weeks on top of v1.0 MVP. Capabilities count rises from Vol. 1's 83 to **103**; mapped failure modes from 60 to **75**; bright lines from 22 to **32**. The combined Argus Omega is no longer a skill router; it is a **production-grade skill federation host with simulator-grounded planning, durable multi-day execution, horizon-aware decisions, bitemporal audit, and continuous chaos-tested catalog hygiene**.

---

## §10 — Decision points (additional, on top of Vol. 1's three)

4. **Approve the additional five reframes** — trajectory simulation, Temporal substrate, HORIZON attribution, ReWOO planning, bitemporal + chaos. Trim if too ambitious.
5. **Pick Vol. 2 staging** — substrate-only (C0–C2, ~3 wk), substrate + plan (C0–C2 + C5, ~5 wk), substrate + simulator + plan (C0–C6, ~7 wk), or full Vol. 2 (C0–C8, ~12.5 wk).
6. **Pick Temporal deployment** — `temporalite` (SQLite, dev), self-hosted (Cassandra/MySQL/Postgres, ops-heavy), or Temporal Cloud (paid, simplest).
7. **Pick chaos cadence** — every 1h (aggressive, dev); every 4h (moderate, recommended); every 24h (conservative, production-cautious).

When ready: "go Vol. 2" or specify which C-phases to start with.

---

## §11 — Reading list additions

Beyond Vol. 1's reading list:

**Reframe 6 (trajectory simulator):** [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [105-agentic-world-modeling-visual-generation](105-agentic-world-modeling-visual-generation.md), [135-trustworthy-generation](135-trustworthy-generation.md).

**Reframe 7 (Temporal substrate):** [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md), [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md), [123-robustness-fault-tolerance](123-robustness-fault-tolerance.md).

**Reframe 8 (HORIZON attribution):** [27-horizon-long-horizon-degradation](27-horizon-long-horizon-degradation.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [38-claw-eval](38-claw-eval.md), [8-context-compaction](08-context-compaction.md), [10-multi-session-continuity](10-multi-session-continuity.md), [16-plan-and-solve](16-plan-and-solve.md).

**Reframe 9 (ReWOO planning):** [17-rewoo](17-rewoo.md), [16-plan-and-solve](16-plan-and-solve.md), [3-plan-mode](03-plan-mode.md), [42-langchain-deep-agents](42-langchain-deep-agents.md).

**Reframe 10 (bitemporal + chaos):** [131-temporal-bitemporal-tables](131-temporal-bitemporal-tables.md), [53-chaos-engineering-next-era](53-chaos-engineering-next-era.md), [26-linuxarena-production-agent-safety](26-linuxarena-production-agent-safety.md), [184-strongest-memory-techniques-synthesis-may-2026](184-strongest-memory-techniques-synthesis-may-2026.md).

---

## §12 — What Argus Omega is NOT (updated)

- **Not a Temporal replacement.** Argus uses Temporal; Argus is not Temporal. The workflow engine remains a separate dependency.
- **Not a simulator framework.** Argus's L2 simulator is purpose-built for skill-trajectory prediction; it does not aspire to be a general-purpose simulation environment (Mu Zero, Genie 2, Dreamer-style world models live elsewhere, see [190](190-agentic-world-modeling-taxonomy.md)).
- **Not a chaos-engineering platform.** Argus's chaos planter targets *its own catalog defenses*; it does not exercise the host harness's other production systems (those belong to the platform's chaos suite).
- **Not a bitemporal database.** Argus's bitemporal catalog is purpose-built for skill mutations; for general bitemporal application data use [130-distributed-sql-as-agent-memory](130-distributed-sql-as-agent-memory.md) or [131-temporal-bitemporal-tables](131-temporal-bitemporal-tables.md) primitives.
- **Not a HORIZON benchmark host.** Argus consumes HORIZON-style attribution; the benchmark suite itself ([27](27-horizon-long-horizon-degradation.md)) is a separate evaluation artifact.

---

## §13 — Cross-references

- Predecessor: [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md) (v1.0).
- Vol. 1 (Reframes 1–5): [194-argus-omega-enhanced-design](194-argus-omega-enhanced-design.md).
- Vol. 2 source canon: [142](142-trajectory-simulation-agents.md), [150](150-temporal-durable-execution-substrate.md), [27](27-horizon-long-horizon-degradation.md), [17](17-rewoo.md), [131](131-temporal-bitemporal-tables.md), [53](53-chaos-engineering-next-era.md).
- Adjacent ecosystems: [104-glm-5v-turbo-native-multimodal-agents](104-glm-5v-turbo-native-multimodal-agents.md) (the cognitive-core counterpart), [191-onemancompany-skills-to-talent](191-onemancompany-skills-to-talent.md) (the organization layer above Argus), [193-recursive-world-organizations-synthesis](193-recursive-world-organizations-synthesis.md) (the architectural map Argus Omega instantiates).
