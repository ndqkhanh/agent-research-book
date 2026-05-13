# AI-First Transformation Plan — `harness-engineering/projects/`

> **Goal**: Take the 24 projects under `research/harness-engineering/projects/` from a mix of pure-systems and AI-augmented work to a uniformly AI-first portfolio, where every project either *is* an agent harness or *is operated by* one.
>
> **Reference implementations**: `lyra` (general agent harness) and `argus` (skill router). Both already exemplify the canonical pattern; the rest of the portfolio should converge on it.
>
> **Source corpus**: `research/docs/` (90+ notes on harness engineering, agent loops, memory, skills, verifiers, RAG, guardrails, TTS).

---

## 0. Executive Summary

**What "AI-first" means in this portfolio** (working definition derived from `40-harness-engineering-principles.md`, `44-four-pillars-harness-engineering.md`, `46-components-of-coding-agent.md`, `99-papers-deep-dive-synthesis.md`):

> An AI-first system is one where an LLM agent loop is the primary computational unit, supported by an engineered harness (typed tools, tiered memory, skills, verifier, hooks, permissions, observability) that makes the agent reliable, auditable, and improvable. The harness is as load-bearing as the model — ~1 generation of model-quality lift comes from harness work alone.

**Scope**: 24 projects across 8 domain clusters. Current distribution: 10 AI-native, 5 AI-augmented, 5 AI-adjacent, 4 AI-absent (incl. one placeholder). Target: all 24 land in **AI-native** by end of plan.

**Approach** (3 levers):
1. **Build a shared `harness-core` library** so each project doesn't reinvent the loop, hooks, memory, verifier, permissions, observability.
2. **Apply per-cluster transformation patterns** — agent projects polish; infrastructure projects gain an AI control plane; adjacent projects bolt on agent loops at the right seams.
3. **Roll out in 4 waves** ordered by leverage: foundation → easy wins → infrastructure → specialized.

**Outcome**: Every project ships with the same 16-item AI-first checklist satisfied, observable through a shared HIR-format trace surface, with a uniform Lyra/Argus-style directory skeleton.

---

## Wave 0 Progress (live)

**Shipped & verified** (`harness-core/` workspace alongside this plan):

| Package | LoC | Tests | Notes |
|---|---|---|---|
| `harness-observability` | ~360 | 19/19 | HIR schema (12 primitives, frozen, semver-versioned), JSONL/Multi/Null sinks, `HIREmitter` with parent-child stack + `child()` for subagents, `attribute_failure()` post-mortem |
| `harness-eval` | ~310 | 27/27 | Pass@k & Pass^k (HumanEval estimators), `Task`/`Attempt`/`EvalReport`, `run_corpus`, HORIZON-lite via observability, `ChaosPolicy`, `harness-eval` CLI |
| `harness-core` | ~250 | 18/18 | Bounded `Loop` (step / token / wall-clock budgets), `LoopState`, `Generation`, `LoopResult`, plan-mode `PlanArtifact` + `read_plan`/`write_plan` |

**Total**: 64/64 tests pass. Integration showcase at `harness-core/examples/echo_agent.py` runs end-to-end (3-package composition: Loop ↔ HIR emission ↔ Pass@k aggregation).

**Scaffolded** (dirs only, pyproject pending): `harness-context`, `harness-memory`, `harness-skills`, `harness-verifier`, `harness-permissions`, `harness-hooks`, `harness-providers`, `harness-mcp-tools`, `harness-evolve`.

**Docs**: `harness-core/docs/{architecture,hir-schema,wiring}.md` + `templates/project-template/` with the 16-item checklist as a per-project gate.

**Open follow-ups**:
- pytest-cov not installed on dev machine — coverage rule enforced informally; need `pip install pytest-cov` to wire into CI.
- harness-permissions, harness-hooks, harness-providers are the highest-leverage next packages (any tool-using agent needs them).
- W1 pilot (orion-code) ready to start once permissions+providers ship.

---

## 1. The AI-First Framework

### 1.1 Four Pillars (`44-four-pillars-harness-engineering.md`)

| Pillar | Concrete asks per project |
|---|---|
| **State Management** | Resumable sessions; plan/state files in `.{project}/state/`; checkpoint hand-offs between phases. |
| **Context Architecture** | Index-before-content; stable prompt prefix for caching; tokens-per-turn budget with alerts; lazy load of large artifacts. |
| **Deterministic Guardrails** | Pre/post tool hooks; lifecycle hooks (SessionStart/Stop); tests/lints in Stop hooks; code enforces invariants — prompts only advise. |
| **Entropy Management** | Memory consolidation daemon (Dream pattern); dead-skill GC; periodic re-indexing; trace rotation. |

### 1.2 Twelve Patterns (`43-twelve-harness-patterns.md`) — checklist

**Memory & Context**: (1) Persistent Instruction File · (2) Scoped Context Assembly · (3) Tiered Memory · (4) Dream Consolidation · (5) Progressive Compaction.
**Workflow**: (6) Explore-Plan-Act · (7) Context-Isolated Subagents · (8) Fork-Join Parallelism.
**Tools & Permissions**: (9) Progressive Tool Expansion · (10) Command Risk Classification · (11) Single-Purpose Tool Design.
**Automation**: (12) Deterministic Lifecycle Hooks.

### 1.3 Reference Architecture (4-plane stack)

```
                 ┌────────────────────────────────────┐
                 │   USER / EVENT / UPSTREAM CALLER   │
                 └─────────────────┬──────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                    CONTROL PLANE  (Agent Loop)                   │
│  assemble_context → generate → permission_check → execute_tool   │
│  → reduce_observation → check_termination → update_state → loop  │
└──────────┬─────────────┬──────────────────┬──────────────┬───────┘
           │             │                  │              │
           ▼             ▼                  ▼              ▼
    ┌──────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐
    │ DATA     │  │  MEMORY      │  │   SAFETY     │  │ OBSERVABIL. │
    │ PLANE    │  │  PLANE       │  │   PLANE      │  │ PLANE       │
    │          │  │              │  │              │  │             │
    │ • tools  │  │ • persistent │  │ • hooks      │  │ • HIR traces│
    │ • skills │  │ • episodic   │  │ • permissions│  │ • OTel spans│
    │ • RAG    │  │ • in-cache   │  │ • sandbox    │  │ • cost+token│
    │ • MCP    │  │ • TTL/decay  │  │ • verifier   │  │ • attribution│
    └──────────┘  └──────────────┘  └──────────────┘  └─────────────┘
                                                            │
                                                            ▼
                                                ┌──────────────────────┐
                                                │  IMPROVEMENT LOOP    │
                                                │  (Reflexion ▸ patch) │
                                                └──────────────────────┘
```

Citations: `01-agent-loop-architecture.md`, `44-four-pillars-harness-engineering.md`, `46-components-of-coding-agent.md`, `66-meta-harness-landscape.md`, `67-recommended-breakthrough-project.md`.

### 1.4 The 16-Item AI-First Checklist (per-project gate)

A project is AI-first when it can answer **yes** to all 16:

- [ ] **L1.** Bounded agent loop (step / token / wall-clock budget)
- [ ] **L2.** Plan-mode artifact (read-only exploration before write)
- [ ] **L3.** Typed tools (no raw shell to model)
- [ ] **L4.** Tiered memory (persistent + episodic + in-cache)
- [ ] **L5.** Skill library (indexed, retrievable, post-task extractor)
- [ ] **L6.** Two-phase verifier (objective checks + LLM judge with rubric)
- [ ] **L7.** Pre/post-tool & lifecycle hooks (formatters, secrets scan, destructive-pattern blocker)
- [ ] **L8.** Permission bridge (Ask/Deny/Auto) with risk classifier
- [ ] **L9.** Context pipeline with progressive compaction & stable cache prefix
- [ ] **L10.** Agentic RAG (when retrieval is on the path)
- [ ] **L11.** Subagent orchestrator (fork-join, worktree-isolated)
- [ ] **L12.** ReAct-style reasoning trace
- [ ] **L13.** HIR-format observability + OTel spans
- [ ] **L14.** Reflexion → memory loop on failures
- [ ] **L15.** Test-time scaling (model-routing or rollout summaries on hard queries)
- [ ] **L16.** Prompt-injection defense (untrusted-content boundaries, instruction hierarchy)

### 1.5 Anti-Patterns to Avoid (`40`, `01`, `08`, `11`, `22`)

1. LLM bolted on, harness ignored.
2. No verifier; agent declares victory on failure.
3. Memory bloat or no memory.
4. Raw shell as the main tool.
5. No compaction or context architecture.
6. Prompt-injection-exposed tool output.
7. Test-time scaling ignored (single fixed model+prompt).

---

## 2. Reference Implementation Pattern (Lyra + Argus)

### 2.1 Five Invariants (every AI-first project must satisfy)

| # | Invariant | Lyra evidence | Argus evidence |
|---|---|---|---|
| I1 | **LLM is composed, not infrastructure** | `_resolve_model_for_role()` per call; `build_llm` factory across 16 providers | Router does ranking deterministically; LLM only invoked by host harness |
| I2 | **Code-level governance > prompt** | `PermissionBridge`, TDD gate hook, secrets scan, destructive-pattern hook | Curator scans for vulnerabilities in code; SHA256 drift detection |
| I3 | **Observability built in, not bolted on** | OTel spans + JSONL turn log + `STATE.md` + git commits | `telemetry.py` on every routing decision (cost/latency/success) |
| I4 | **Modular plug-ins, not monolith** | Single-agent / three-agent / dag-teams strategies; opt-in TDD; user hooks | 5-tier router (any subset); pluggable marketplace adapters |
| I5 | **Replay & cross-channel verification** | Verifier Phase 1 + Phase 2 + (trace ↔ diff ↔ env) | Decision objects with full trace + alternatives |

### 2.2 Canonical Directory Skeleton (target for all 24)

```
<project>/
├── README.md                       # one-sentence "what does the AI do here?"
├── docs/{architecture,system-design,threat-model,roadmap}.md
├── src/<project>_core/
│   ├── loop.py | Loop.kt | Loop.java        # agent loop (the heart)
│   ├── gateway.py                            # session lifecycle + entry
│   ├── context/{pipeline,compactor,grid}.py  # 5-layer context
│   ├── memory/{store,capture,decay,
│   │           reasoning_bank,consolidator}.py
│   ├── skills/{loader,router,extractor,curator}.py
│   ├── permissions/{bridge,risk_classifier}.py
│   ├── hooks/{lifecycle,tdd_gate,secrets_scan,
│   │          destructive_pattern}.py
│   ├── tools/{base,core_tools}.py            # typed, scoped
│   ├── verifier/{objective,subjective}.py
│   ├── observability/{hir,otel_export,retro}.py
│   ├── providers/build_llm.py                # multi-provider factory
│   ├── harnesses/{single,three,dag}.py       # strategy plug-ins
│   └── subagent/worktree.py                  # parallel isolation
├── tests/                                    # RED-test the architecture
├── pyproject.toml | build.gradle | pom.xml
└── .<project>/{config.yaml,state/,traces/,plans/,memory/}
```

For JVM projects (Java 21 / Kotlin), substitute `src/main/{java,kotlin}/...` with the same module names — the architecture is language-portable.

---

## 3. Common Library — `harness-core` (build once, reuse across 24)

To avoid reinventing the loop in every project, extract these as shared packages. Each is small (one well-defined concern) and language-paired (Python + JVM).

| Package | Responsibility | Lyra/Argus origin |
|---|---|---|
| `harness-core` (py + jvm) | Agent loop, gateway, plan-mode, ReAct trace | Lyra `agent/loop.py` |
| `harness-context` | 5-layer context pipeline, NGC compactor, stable prefix | Lyra `context/` |
| `harness-memory` | Tiered store (FTS5/Chroma/file), TTL decay, consolidator | Lyra `memory/` |
| `harness-skills` | SKILL.md loader, router cascade, extractor | Argus router + Lyra `skills/` |
| `harness-verifier` | Objective phase + subjective LLM-judge | Lyra `verifier/` |
| `harness-permissions` | PermissionBridge + risk classifier + sandbox shim | Lyra `permissions/bridge.py` |
| `harness-hooks` | Pre/post-tool + lifecycle hook protocol | Lyra `hooks/` |
| `harness-observability` | HIR trace schema, OTel exporter, JSONL writer, attribution | Lyra `observability/` |
| `harness-providers` | Multi-provider LLM factory (16 providers) | Lyra `gateway/adapter.py` |
| `harness-mcp-tools` | Typed tool catalog with MCP transport | Lyra `tools/` + MCP |
| `harness-evolve` | Reflexion + Autogenesis patch loop | Lyra `evolve/` |
| `harness-eval` | Pass@k, Pass^k, HORIZON attribution | `vertex-eval` extracted |

**Where it lives**: a new top-level `research/harness-engineering/harness-core/` workspace (multi-package monorepo). Consumed by the 24 projects via path/version dep.

**Why centralize**: the maturity matrix (§4) shows every project needs ~6 of these. Without a common library, each rebuilds the loop, hooks, and verifier — that's the #1 anti-pattern from §1.5.

---

## 4. Project Maturity Matrix

| # | Project | Current | Target | Gap | Wave | Effort |
|---|---|---|---|---|---|---|
| 1 | aegis-ops | augmented | native | hooks, verifier, mem, skills | W3 | M |
| 2 | agent-forge | native | native+ | observability parity, evolve loop | W1 | S |
| 3 | argus | native | native+ | already canonical — extract to lib | W0 | S |
| 4 | atlas-research | native | native+ | reflexion loop, TTS routing | W1 | S |
| 5 | cipher-sec | augmented | native | tiered mem, verifier, skills | W3 | M |
| 6 | flash-cache | absent | native (control-plane) | RL eviction agent + control plane | W3 | L |
| 7 | gnomon | native | native+ | LLM-judge migration, HIR self-trace | W1 | S |
| 8 | grabflow | native | native+ | unify ML ops behind agent loop | W4 | M |
| 9 | harmony-voice | augmented | native | swap to learned cache + planner | W2 | M |
| 10 | helix-bio | augmented | native | judge ensemble, claim verifier | W3 | M |
| 11 | inferx | native | native+ | bandit posterior persistence, trace | W1 | S |
| 12 | lyra | native | reference | already canonical — extract to lib | W0 | S |
| 13 | mentat-learn | absent | native | scope it (placeholder today) | W4 | L |
| 14 | nexus-db | adjacent | native (control-plane) | workload-aware tuner agent | W3 | L |
| 15 | open-fang | native | native+ | self-wiring → reflexion gating | W1 | S |
| 16 | orion-code | native | native+ | align loop with `harness-core` | W1 | S |
| 17 | polaris | native | native+ | program-graph mem persistence | W1 | S |
| 18 | quanta-proof | native | native+ | LATS reuse to `harness-core` | W1 | M |
| 19 | streamforge | adjacent | native | neural ABR + steering agent | W2 | M |
| 20 | syndicate | augmented | native | three-agent + observability | W3 | M |
| 21 | triggerflow | adjacent | native | bandit budget + LLM rule synth | W2 | M |
| 22 | turbo-mq | absent | native (control-plane) | broker forecast + rebalance agent | W3 | L |
| 23 | vector-forge | adjacent | native | query-rewrite + learned RRF | W2 | S |
| 24 | vertex-eval | native | reference | extract eval primitives to lib | W0 | S |

**Effort**: S=≤1wk, M=2–4wk, L=1–2mo (rough estimates; assumes single engineer).

---

## 5. Per-Project Transformation Plans

> Format per project: **Current → Target** (one line each), then specific changes mapped to the canonical skeleton (§2.2) and AI-first checklist (§1.4).

### Cluster A — Agent Harnesses & Orchestration

#### 5.1 lyra (reference)
- **Current → Target**: canonical agent harness → reference donor for `harness-core`.
- **Changes**:
  - Extract `agent/`, `context/`, `memory/`, `verifier/`, `permissions/`, `hooks/`, `skills/`, `observability/`, `providers/`, `harnesses/`, `evolve/` packages into `harness-core/` workspace; pin lyra to consume via path dep.
  - Document the canonical wiring in `harness-core/docs/wiring.md` so other projects copy-paste.
  - Add `.lyra/REFERENCE.md` declaring it the donor.
- **Owner of**: L1, L2, L3, L4, L5, L6, L7, L8, L9, L11, L12, L13, L14, L15.

#### 5.2 argus (reference)
- **Current → Target**: canonical skill router → donor for `harness-skills`.
- **Changes**:
  - Move `router_chain.py`, `tiers/`, `curator/`, `refine/`, `marketplace/`, `governance/bright_lines.py` into `harness-skills/`.
  - Keep argus as the *thick host adapter* showcase (3-line integration); other projects pull from `harness-skills` directly.
- **Owner of**: L5 (full lifecycle: load/route/extract/refine/govern).

#### 5.3 agent-forge
- **Current → Target**: Java agent runtime with adaptive compaction → JVM twin of `harness-core` (parity surface).
- **Changes**:
  - Mirror `harness-core` package boundaries in Java 21 (`com.harness.core.{loop,context,memory,verifier,permissions,hooks,observability}`).
  - Wire entropy-scored compaction strategies behind `Compactor` interface so Python lyra and JVM agent-forge share the same HIR trace shape.
  - Add `evolve/` reflexion loop (currently absent).
  - Bring observability to HIR parity (currently OTel-only).
- **Adds**: L14 (reflexion), parity on L13.

#### 5.4 orion-code
- **Current → Target**: coding-specific agent → consumer of `harness-core`, specialized in code-edit tools.
- **Changes**:
  - Replace internal loop with `harness-core` loop; keep the orion-specific *tool catalog* (`tools/code_tools.py`: ast-aware Edit, Patch, Run-tests).
  - Add `verifier/code_verifier.py` (compile + lint + test triple).
  - Expose three-agent harness for refactors (Planner→Generator→Evaluator).
- **Adds**: L6, L11, L15.

#### 5.5 syndicate
- **Current → Target**: multi-agent platform (MVP) → orchestrator built on `harness-core` + canonical observability.
- **Changes**:
  - Replace ad-hoc handoff with `harness-core/subagent/worktree.py` for fork-join.
  - Add `verifier/` (audit handoff contracts at boundaries).
  - Add HIR-format trace stream so cross-agent flows are debuggable.
  - Ship a `dag-teams` sample built on top.
- **Adds**: L6, L11, L13, L14.

### Cluster B — Long-Horizon Research Agents

#### 5.6 atlas-research
- **Current → Target**: ReWOO-style research agent → research agent with reflexion loop + TTS routing.
- **Changes**:
  - Add `evolve/` Reflexion: failed citation verification → memory note → next-run query rewrite.
  - Route hard queries (judged by complexity classifier) to a stronger model slot via `harness-providers`.
  - Persist program-graph memory (à la polaris) so multi-session research compounds.
- **Adds**: L14, L15.

#### 5.7 open-fang
- **Current → Target**: 9-specialist autonomous research → reflexion-gated, attribution-driven.
- **Changes**:
  - Migrate primitive attribution (HAFC-lite) into `harness-observability/attribution.py` so all projects can reuse.
  - Add Reflexion gate: failed verification triggers re-plan with the failure note pinned.
  - Move self-wiring (regex + cascade) into a `harness-skills` recipe.
- **Adds**: L14, polish on L13.

#### 5.8 polaris
- **Current → Target**: long-running polymath → durable program-graph memory + auto-shell selection via skills.
- **Changes**:
  - Externalize program-graph store via `harness-memory` (replace bespoke storage).
  - Move shell auto-selection into `harness-skills` (each shell becomes a skill manifest).
  - Add `harnesses/dag-teams` for the writing/rebuttal phases.
- **Adds**: L4 parity, L5 parity.

#### 5.9 helix-bio
- **Current → Target**: bio agent w/ ontology grounding → claim-verified with judge ensemble.
- **Changes**:
  - Implement two-phase verifier with **judge ensemble** (different model families) for dual-use safety claims.
  - Train a small classifier on the existing red-team corpus → integrated into permission `risk_classifier`.
  - Expand entity linking from regex to LLM with confidence gating.
- **Adds**: L6 (real verifier), L8 (real risk classifier).

### Cluster C — AI-Native Specialized Agents

#### 5.10 quanta-proof
- **Current → Target**: neuro-symbolic prover → LATS-search reusable component.
- **Changes**:
  - Lift LATS-lite proof search into `harness-core/search/lats.py` so other planning-heavy projects can reuse.
  - Add Lean verifier as a real `verifier/objective.py` plugin (currently mock).
  - Expose lemma retrieval as a skill in `harness-skills`.
- **Adds**: parity on L6, L15 (rollout summaries).

#### 5.11 gnomon
- **Current → Target**: harness evaluator → owner of attribution + LLM-judge migration path.
- **Changes**:
  - Promote 12-primitive Harness IR to `harness-observability/hir.py` (already cited in target).
  - Implement rule-based-HAFC → LLM-judge bridge as `harness-eval/judge_bridge.py`.
  - Wire chaos injection into `tests/integration/chaos.py` shared across projects.
- **Adds**: drives L13 across portfolio.

#### 5.12 vertex-eval
- **Current → Target**: third-party eval harness → eval primitives library.
- **Changes**:
  - Extract Pass@k / Pass^k / HORIZON / LaStraj into `harness-eval/`.
  - Provide a CLI: `harness-eval run <project> --suite=<name>`; every project gets a `tests/eval/` directory wired to it.
  - Add federated corpus PII-aware aggregation.
- **Adds**: makes L13 + L14 measurable everywhere.

#### 5.13 inferx
- **Current → Target**: ML inference gateway → bandit-state persisted + traced.
- **Changes**:
  - Persist Thompson-Sampling posteriors in `harness-memory` so the gateway warm-starts on restart.
  - Emit HIR traces per arm-pull for inspection.
  - Auto-tune PID batcher gains via background optimizer (`harness-evolve`).
- **Adds**: L4 + L13 (already L1/L2-equivalent in spirit; just needs to be shaped through harness-core).

### Cluster D — Application / ML-Ops

#### 5.14 grabflow
- **Current → Target**: ride-share ML stack → AI-control-plane unifier across NexusDB/TurboMQ/FlashCache/AgentForge.
- **Changes**:
  - Build a top-level `grabflow/control-plane/` agent that observes the four custom infra projects and tunes them via the AI-control-plane primitives (§5.15, §5.18, §5.20, §5.22).
  - Wrap LSTM surge forecast + RL dispatch as skills inside `harness-skills`.
  - Add agentic dispatcher narration → HIR traces for every match.
- **Adds**: portfolio-level integration showcase; L11 + L13 + L15.

#### 5.15 mentat-learn (placeholder)
- **Current → Target**: stub → "learning lab" companion to `harness-skills` (Voyager-style skill curriculum).
- **Changes**:
  - Define scope: this is the *skill-evolution playground* — generates skills, runs them in sandbox, scores, promotes/demotes.
  - Build CoEvoSkills harness on top of `harness-skills/extractor.py`.
  - Output: a curated skill catalog consumed by the rest of the portfolio.
- **Adds**: L5 lifecycle automation.

### Cluster E — Voice

#### 5.16 harmony-voice
- **Current → Target**: dual-agent voice cache → learned cache + planner.
- **Changes**:
  - Replace bag-of-words with embedding-based semantic cache (use `harness-memory` Chroma adapter).
  - LLM-driven topic prefetch → skill in `harness-skills`.
  - Adaptive hedge timeout via online latency predictor (small online model).
- **Adds**: L4 (real semantic memory), L5, L15.

### Cluster F — Security / Ops

#### 5.17 aegis-ops
- **Current → Target**: SRE runbooks → agent-driven incident response with verifier.
- **Changes**:
  - Add `loop.py` driving runbook execution; existing policy engine becomes `permissions/policy.py`.
  - Add `verifier/` (objective: did the runbook land in the green? subjective: judge the diagnosis).
  - Adaptive policy learning: Reflexion on runbook outcomes → patches to `policies/*.yaml`.
- **Adds**: L1, L6, L14.

#### 5.18 cipher-sec
- **Current → Target**: pentest agent → tiered-memory + verifier + judge.
- **Changes**:
  - Add tiered memory (campaign log → episodic per-engagement → persistent vendor knowledge).
  - Add `verifier/safety.py` to gate adversarial intent (judge ensemble + risk classifier).
  - Move recon technique catalog into `harness-skills` with vendor-trust gating.
- **Adds**: L4, L5, L6, L8 polish.

### Cluster G — Infrastructure (with AI Control Plane)

> Pattern for all infra projects: keep the deterministic data plane, add an **AI control plane** that observes telemetry, decides adaptive policies, and writes them back through a typed-tool interface. The control plane is itself a `harness-core` agent.

#### 5.19 flash-cache
- **Current → Target**: Redis-compatible cache → AI-tuned cache w/ control-plane agent.
- **Changes**:
  - Data plane untouched (RESP/HTTP/2/WebSocket, SWIM gossip).
  - Add `flash-cache-control/` package: agent loop that observes hit-rate / hot-key telemetry and updates eviction policy + replication topology via typed tools (`SetEvictionPolicy`, `RebalanceShard`).
  - RL eviction policy (state: access patterns, action: evict candidate, reward: hit rate); policy persisted in `harness-memory`.
  - Predictive prefetch agent (small transformer → next-N keys).
- **Adds**: L1, L4, L7, L13.

#### 5.20 turbo-mq
- **Current → Target**: per-partition Raft MQ → MQ with broker-forecasting + adaptive rebalancing agent.
- **Changes**:
  - Data plane untouched.
  - Add `turbo-mq-control/`: time-series model on broker metrics → preemptive rerouting; RL rebalancing on partition hotness; learned retention per topic.
  - Tool surface for control: `Rebalance`, `Drain`, `SetRetention`.
- **Adds**: L1, L13, L15.

#### 5.21 nexus-db
- **Current → Target**: adaptive-lock-elision DB → workload-aware DB w/ tuning agent.
- **Changes**:
  - Data plane (MVCC, SSI, ARIES) untouched.
  - Add `nexus-db-control/` agent: proposes/drops indexes, picks isolation level per query class, predicts contention.
  - Plan-cache with embedding-based query similarity (use `harness-memory` Chroma).
- **Adds**: L1, L4, L7.

#### 5.22 streamforge
- **Current → Target**: video streaming → neural-ABR + steering agent.
- **Changes**:
  - Replace EWMA + buffer-based ABR with Pensieve-style learned policy.
  - Per-content bitrate ladder (offline trainer in `mentat-learn`, online inference in stream path).
  - CDN steering agent (small online RL).
- **Adds**: L1 (control loop), L13.

#### 5.23 triggerflow
- **Current → Target**: rule-based marketing automation → bandit-budget + LLM-rule-synth.
- **Changes**:
  - LLM rule synthesizer: natural language → AST (skill in `harness-skills`).
  - Thompson Sampling for campaign budget allocation.
  - Predictive segmentation via online clustering.
- **Adds**: L1, L5, L13.

#### 5.24 vector-forge
- **Current → Target**: hybrid search → query-rewrite + learned RRF.
- **Changes**:
  - LLM query rewriter as a skill (cached).
  - Learned RRF fusion weights per query class (offline + online update).
  - Adaptive PQ bit-width via recall feedback.
  - Agentic RAG entry-point exposed as `tools/AgenticSearch`.
- **Adds**: L1, L5, L10.

---

## 6. Phased Rollout

### Wave 0 — Foundations (≈4 weeks)
- Stand up `harness-core/` monorepo with the 12 packages from §3.
- Donor extraction from lyra (Python) + agent-forge (JVM) + argus (skills) + vertex-eval (eval) + gnomon (HIR/attribution).
- Publish package registry (path/version pinned).
- **Exit criteria**: lyra and argus fully consume `harness-core` with green test suites; HIR trace schema versioned 1.0.

### Wave 1 — Polish AI-Native Projects (≈3 weeks)
- 9 projects already AI-native (lyra, argus, agent-forge, atlas-research, gnomon, inferx, open-fang, orion-code, polaris, quanta-proof, vertex-eval) align to `harness-core`.
- Backfill missing checklist items (mostly L14/L15 + observability parity).
- **Exit criteria**: all 9 pass the 16-item checklist; cross-project HIR traces visible in a single dashboard.

### Wave 2 — Easy Adjacent Wins (≈4 weeks)
- streamforge, triggerflow, vector-forge, harmony-voice.
- These have rich telemetry/data already; AI primitives are additive (not invasive).
- **Exit criteria**: each emits HIR traces for AI decisions; A/B vs. heuristic baseline ≥ neutral.

### Wave 3 — Infrastructure Control Planes (≈6 weeks)
- flash-cache, turbo-mq, nexus-db (the L-effort projects).
- Build control-plane sidecars; data plane untouched.
- aegis-ops, cipher-sec, helix-bio, syndicate also land here (M-effort, agentic upgrades).
- **Exit criteria**: each control plane shows ≥1 measurable improvement (hit-rate, rebalance-recovery, or query-latency) over baseline.

### Wave 4 — Application & Stub (≈4 weeks)
- grabflow becomes the integration showcase pulling all 4 infra control planes.
- mentat-learn scoped and shipped as the skill-evolution lab.
- **Exit criteria**: portfolio-level demo where grabflow narrates dispatch decisions across the 4 custom infra agents in real time.

**Total elapsed**: ~17 weeks of focused work. Parallelizable into ~10 weeks if multiple engineers.

---

## 7. Verification & Success Criteria

### 7.1 Per-project gate
Project is "AI-first complete" when:
- 16/16 checklist items satisfied (§1.4).
- Canonical directory skeleton (§2.2) followed.
- Tests in `tests/eval/` run via `harness-eval run <project>` and pass.
- HIR traces from a 30-step run are loadable by gnomon for attribution.

### 7.2 Portfolio-level metrics
- **Coverage**: 24/24 projects in AI-native bucket.
- **Reuse**: ≥80% of LoC for loop/context/memory/verifier/hooks/perm/observability comes from `harness-core` (not local).
- **Observability**: every AI decision across the portfolio addressable in one HIR trace browser.
- **Eval**: `harness-eval` Pass@k over the portfolio published quarterly.

### 7.3 Verification roles (separate from authoring; per CLAUDE.md guidance)
- Authoring: each project owner.
- Review: `code-reviewer` + `verifier` agents in CI.
- Final gate: `oh-my-claudecode:verify` on the project's eval suite.

---

## 8. Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | `harness-core` becomes a god-package | M | H | Strict package boundaries; no cross-package internal imports; ADR per addition. |
| R2 | Infrastructure control planes destabilize data planes | M | H | Sidecar architecture; control plane only writes via typed tools with permission gates; canary rollout. |
| R3 | LLM costs blow up under always-on agents | H | M | Test-time scaling: cheap model on inner loop, smart model for hard cases; cache prefix; per-project budget alarms. |
| R4 | Drift from canonical lyra/argus pattern | M | M | Quarterly skeleton audit (`harness-audit` skill); ADRs for any deviation. |
| R5 | Prompt-injection through tool output / RAG | M | H | Untrusted-content boundary in `harness-permissions`; tool output tagged as data not instruction; instruction hierarchy enforced in prompt builder. |
| R6 | Mentat-learn stays a placeholder | H | L | Time-box scoping in W4; if not productive, fold its purpose into argus refinement loop. |

---

## 9. Open Questions / Decisions Needed

1. **Mono-repo vs. multi-repo for `harness-core`?** Recommend mono-repo with workspace tooling (yarn/uv workspaces) — easier shared evolution.
2. **Python-only `harness-core`, or first-class JVM twin?** JVM twin needed because 6 projects are Java/Kotlin (agent-forge, flash-cache, grabflow, inferx, streamforge, vector-forge) + 1 Kotlin (turbo-mq). Recommend yes.
3. **LLM provider default for the portfolio?** Lyra supports 16; pick 2-3 (Anthropic + DeepSeek + a local Ollama) as the portfolio default to avoid sprawl.
4. **Eval budget**: pass@k requires k samples; agree on k=5 for now, k=10 for high-stakes (cipher-sec, helix-bio).
5. **Observability sink**: local file (JSONL) only, or also push to LangSmith / Phoenix? Recommend file + optional LangSmith/Phoenix exporter via `harness-observability`.

---

## 10. Next Actions (immediate)

1. Approve this plan or flag changes (especially Wave order & scope of `harness-core`).
2. Create `research/harness-engineering/harness-core/` workspace skeleton (Wave 0 kickoff).
3. Pick 1 pilot project per wave to validate the pattern before scaling:
   - W1 pilot: **orion-code** (smallest AI-native consumer; surfaces friction in the lib).
   - W2 pilot: **vector-forge** (rich telemetry; least invasive change).
   - W3 pilot: **flash-cache** (most aggressive — data-plane sanctity test).
   - W4: grabflow is its own pilot (integration).
4. Stand up `harness-eval` CLI + `harness-observability` HIR schema first; everything else depends on these.

---

*Sources*: synthesized from `research/docs/01,03,05,06,08,09,11,13,14,18,19,22,25,40,43,44,46,66,67,99,77`-style notes, plus reverse-engineering of `lyra/` and `argus/`. See §1 citations for traceability.
