# 285 — `harness_core/` Porting Plan: skeleton tree, public APIs, migration phases

**Anchors.** All seven layers across [225](225-agent-era-scaling-synthesis.md), [258](258-agent-protocol-stack-synthesis-2026.md), [263](263-production-agent-runtime-synthesis-2026.md), [268](268-agent-operations-synthesis-2026.md), [273](273-agent-security-synthesis-2026.md). Per-project plans: [279](279-polaris-seven-layer-stack-apply-plan.md), [280](280-lyra-seven-layer-stack-apply-plan.md), [281](281-mentat-learn-seven-layer-stack-apply-plan.md), [282](282-argus-seven-layer-stack-apply-plan.md), [283](283-aegis-ops-seven-layer-stack-apply-plan.md), [284](284-cipher-sec-seven-layer-stack-apply-plan.md). Existing canonical implementations: polaris-core, polaris-daemon, lyra-core, lyra-cli, lyra-mcp, lyra-skills.

**One-line definition.** A **concrete porting plan** for `harness_core/` — the shared library every project in `projects/` imports — with the **skeleton tree** (eleven sub-packages: `foundation/` + `capability/` + `protocols/` + `runtime/` + `security/` + `operations/` + `compliance/` + `routines/` + `memory/` + `skills/` + `ux/`), **public-API signatures for the most critical interfaces** (Permission Bridge, HIR Emitter, Daemon, Bright-line Gates, A2A server / client, MCP server registry, Routine engine, OTel observability, SLO + Runbook framework, Compliance checks), **migration phases** from the current per-project implementations (consolidate from polaris-core + lyra-core + lyra-mcp + lyra-skills), and **per-project rollout sequencing** that gets all 14+ projects onto the shared library in ~6 months without breaking deployed agents — turning the 35-file design corpus from docs into an actual Python package.

## Why we need `harness_core/`

The corpus from 250 onward repeatedly calls for "shared `harness_core/` package" — observability, routines, protocols, security, ops, eval, finetuning, sre. By 285 the case is overdetermined: **fourteen projects** (polaris, lyra, mentat-learn, argus, aegis-ops, cipher-sec, atlas-research, helix-bio, harmony-voice, vertex-eval, gnomon, orion-code, syndicate, quanta-proof, open-fang) all need the same primitives; without `harness_core/`, each ports independently — 14 implementations of Permission Bridge, 14 implementations of HIR observability, 14 implementations of A2A endpoints, 14 implementations of cost router. **The shared library is the high-leverage architectural move of 2026** for the in-tree ecosystem.

The good news: **canonical implementations already exist**. `polaris-core` ships Permission Bridge ([07](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md)), Hooks ([08](../projects/polaris/docs/blocks/08-hooks-and-claim-gate.md)), Daemon ([13](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md)), Bright-line Gates ([14](../projects/polaris/docs/blocks/14-bright-line-gates.md)), Cost Router ([15](../projects/polaris/docs/blocks/15-cost-router-and-budget.md)), HIR Observability ([16](../projects/polaris/docs/blocks/16-observability-hir.md)), MCP Adapter ([17](../projects/polaris/docs/blocks/17-mcp-adapter.md)), Subagent + Worktree ([18](../projects/polaris/docs/blocks/18-subagent-and-worktree.md)). `lyra-core` ships memory primitives ([Lyra Block 07](../projects/lyra/docs/blocks/07-memory-three-tier.md), 13 modules including auto_memory, consolidator, decay, distillers, fusion, memory_tools, procedural, progressive, reasoning_bank, redactor). The porting work is **promote shared code from polaris-core / lyra-core into `harness_core/`**, with thin per-project wrappers.

## Skeleton tree

```
harness_core/
├── pyproject.toml
├── README.md
├── src/harness_core/
│   ├── __init__.py
│   ├── foundation/                    # L1
│   │   ├── permissions.py             # PermissionBridge, ResearchAction, ActionKind
│   │   ├── hooks.py                   # Pre/post-tool-use hooks; claim gate
│   │   ├── bright_lines.py            # BrightLineGate, BrightLineCode
│   │   ├── daemon.py                  # Daemon, IdlePolicy, Heartbeat
│   │   └── cost_router.py             # CostRouter, ModelTier, BudgetGate
│   ├── capability/                    # L2
│   │   ├── ttc_router.py              # Difficulty-aware test-time compute
│   │   ├── trajectory.py              # TrajectoryBudget, plateau detection
│   │   └── verifier.py                # CrossChannelVerifier, PRM interface
│   ├── protocols/                     # L3
│   │   ├── mcp/
│   │   │   ├── server.py              # MCP server (Streamable HTTP)
│   │   │   ├── client.py              # MCP client
│   │   │   └── registry.py            # MCP server discovery
│   │   ├── a2a/
│   │   │   ├── server.py              # A2A server (FastAPI-based)
│   │   │   ├── client.py              # A2A client
│   │   │   ├── agent_card.py          # AgentCard, sign + verify
│   │   │   └── task.py                # Task, Message, Artifact lifecycle
│   │   ├── agntcy/
│   │   │   ├── oasf.py                # OASF schema models
│   │   │   ├── acp.py                 # ACP REST client + server
│   │   │   └── registry.py            # Discovery layer
│   │   ├── nats_transport/
│   │   │   ├── leaf.py                # NATS leaf node config
│   │   │   └── jetstream.py           # Durable stream helpers
│   │   └── tailscale.py               # Tailscale ACL helpers
│   ├── runtime/                       # L4 (adapters)
│   │   ├── langgraph_adapter.py       # LangGraph wrapper
│   │   ├── agents_sdk_adapter.py      # OpenAI Agents SDK wrapper
│   │   ├── autogen_adapter.py         # AutoGen v0.4 wrapper
│   │   ├── adk_adapter.py             # Google ADK wrapper
│   │   ├── crewai_adapter.py          # CrewAI wrapper
│   │   └── claude_code_teams.py       # Anthropic Agent Teams wrapper
│   ├── security/                      # L5
│   │   ├── classifiers/
│   │   │   ├── prompt_shield.py       # Microsoft Prompt Shield client
│   │   │   ├── constitutional.py      # Anthropic Constitutional Classifier
│   │   │   └── ensemble.py            # Multi-classifier voting
│   │   ├── instruction_hierarchy.py   # System prompt structure helpers
│   │   ├── spotlight.py               # Spotlight prompting
│   │   ├── isolation/
│   │   │   ├── worktree.py            # Git worktree per agent run
│   │   │   ├── container.py           # Docker/Podman runner
│   │   │   ├── microvm.py             # Firecracker / gVisor runner
│   │   │   └── network_policy.py      # Egress allowlist
│   │   ├── supply_chain/
│   │   │   ├── sbom.py                # SBOM generation + audit
│   │   │   ├── signing.py             # Cosign / Sigstore wrappers
│   │   │   ├── trust_tier.py          # Trust tier enforcement
│   │   │   └── revocation.py          # Revocation feed + propagation
│   │   └── memory_redactor.py         # Promoted from lyra-core
│   ├── operations/                    # L6
│   │   ├── observability/
│   │   │   ├── otel.py                # OTel SDK setup
│   │   │   ├── hir_events.py          # Typed HIR-shape events
│   │   │   └── trace_propagation.py   # traceparent across A2A / MCP
│   │   ├── evaluation/
│   │   │   ├── eval_runner.py         # Eval suite framework
│   │   │   ├── llm_judge.py           # LLM-as-judge with calibration
│   │   │   ├── benchmarks.py          # GAIA / OSWorld / SWE-bench wrappers
│   │   │   └── statistical.py         # Power analysis, paired t-test, Bonferroni
│   │   ├── durability/
│   │   │   ├── langgraph_checkpoint.py # Postgres / SQLite checkpointer
│   │   │   ├── temporal_adapter.py    # Temporal workflow wrapper
│   │   │   ├── idempotency.py         # Idempotency keys + dedup table
│   │   │   └── saga.py                # Saga / outbox patterns
│   │   └── sre/
│   │       ├── slo.py                 # SLO/SLI definitions
│   │       ├── runbooks/              # Canonical runbooks
│   │       ├── rollback.py            # Four-axis rollback (model/prompt/skill/runtime)
│   │       └── postmortem.py          # MAST-classified postmortem template
│   ├── compliance/                    # L7
│   │   ├── eu_ai_act.py               # Risk tier classification + obligations
│   │   ├── soc2.py                    # SOC 2 controls + evidence collection
│   │   ├── gdpr.py                    # Right-to-erasure cascade
│   │   ├── hipaa.py                   # PHI handling
│   │   └── audit_trail.py             # Tamper-evident log (S3 Object Lock / Sigstore)
│   ├── routines/                      # Invocation surface
│   │   ├── config_store.py            # Routine config CRUD
│   │   ├── triggers/
│   │   │   ├── cron.py                # Daemon-driven cron
│   │   │   ├── api.py                 # HTTPS POST /routines/{id}/fire
│   │   │   └── webhook.py             # GitHub / Slack / generic
│   │   └── executor.py                # Isolated execution per fire
│   ├── memory/                        # Promoted from lyra-core
│   │   ├── auto_memory.py
│   │   ├── auto_capture.py
│   │   ├── consolidator.py
│   │   ├── contradictions.py
│   │   ├── decay.py
│   │   ├── distillers.py              # Heuristic + LLM distillers
│   │   ├── fusion.py                  # FTS5 + Chroma RRF fusion
│   │   ├── memory_tools.py            # MCP-shape API
│   │   ├── procedural.py              # Skill records
│   │   ├── progressive.py             # Progressive disclosure
│   │   ├── reasoning_bank.py          # Lessons + trajectories
│   │   └── reasoning_bank_store.py    # SQLite + Chroma persistence
│   ├── skills/                        # SKILL.md primitives
│   │   ├── skill_md.py                # Parse + serialize + sign
│   │   ├── skill_engine.py            # Load + run skills
│   │   ├── skill_auto_creator.py      # Pattern extraction → SKILL.md draft
│   │   ├── marketplace_client.py      # Smithery / subagents.cc / LobeHub
│   │   └── vendored/                  # Vendored fallback skills
│   └── ux/                            # User-facing patterns
│       ├── plan_mode.py               # Plan emit + review + execute
│       ├── streaming.py               # Stage delineation
│       ├── permission_review.py       # Three-tier review UI helpers
│       └── team_panel.py              # Multi-agent UX helpers
└── tests/
    └── (mirror src/ structure)
```

## Public-API signatures (most critical)

### Permission Bridge

```python
# src/harness_core/foundation/permissions.py
from enum import Enum
from dataclasses import dataclass

class ActionKind(str, Enum):
    READ_FILE = "read_file"
    WRITE_FILE = "write_file"
    EXEC_CODE = "exec_code"
    SEND_EMAIL = "send_email"
    A2A_SUBMIT = "a2a_submit"
    A2A_RECEIVE = "a2a_receive"
    ROUTINE_FIRE = "routine_fire"
    DAEMON_START = "daemon_start"
    TEAM_SPAWN = "team_spawn"
    BRIGHT_LINE = "bright_line"
    # ... extensible per project

@dataclass
class ResearchAction:
    kind: ActionKind
    summary: str
    target: str | None = None
    metadata: dict = None

class PermissionBridge:
    def decide(self, action: ResearchAction) -> Decision: ...
    def hook(self, before: callable = None, after: callable = None): ...
```

### HIR Emitter

```python
# src/harness_core/operations/observability/hir_events.py
from enum import Enum

class HIREventKind(str, Enum):
    AGENT_LOOP_START = "AgentLoop.start"
    AGENT_LOOP_STEP = "AgentLoop.step"
    AGENT_LOOP_END = "AgentLoop.end"
    TOOL_CALL = "Tool.call"
    TOOL_RESULT = "Tool.result"
    LLM_CALL = "LLM.call"
    VERIFIER_VERDICT = "Verifier.verdict"
    PERMISSION_DECISION = "Permission.decision"
    BRIGHT_LINE_ESCALATION = "BrightLine.escalation"
    MEMORY_WRITE = "Memory.write"
    MEMORY_READ = "Memory.read"
    SKILL_INVOKE = "Skill.invoke"
    ROUTINE_FIRE_START = "Routine.fire_start"
    ROUTINE_FIRE_END = "Routine.fire_end"
    TEAM_SPAWN = "Team.spawn"
    TEAM_MESSAGE = "Team.message"
    # ... full taxonomy

class HIREmitter:
    def emit(self, kind: HIREventKind, payload: dict) -> None: ...
    def with_traceparent(self, traceparent: str) -> "HIREmitter": ...
```

### A2A Server

```python
# src/harness_core/protocols/a2a/server.py
from fastapi import FastAPI

class A2AServer:
    def __init__(self, agent_card: AgentCard, signer: Signer, oauth_provider: OAuthProvider):
        ...
    
    def expose_capability(self, capability_id: str, handler: callable):
        """Register a capability handler."""
        ...
    
    def app(self) -> FastAPI:
        """Get the underlying FastAPI app for deployment."""
        ...
```

### Routine engine

```python
# src/harness_core/routines/executor.py
@dataclass
class RoutineConfig:
    routine_id: str
    owner_user_id: str
    name: str
    prompt: str
    workspace: WorkspaceSpec
    connectors: list[ConnectorSpec]
    triggers: list[TriggerSpec]
    bearer_token_hash: bytes
    permission_mode: PermissionMode
    cost_cap_per_run: int

class RoutineExecutor:
    def fire(
        self,
        routine: RoutineConfig,
        trigger_kind: TriggerKind,
        payload: dict,
    ) -> str:  # returns session_id
        ...
```

### LangGraph adapter

```python
# src/harness_core/runtime/langgraph_adapter.py
from langgraph.graph import StateGraph
from langgraph.checkpoint.postgres import PostgresSaver

class HarnessLangGraph:
    """Wraps a LangGraph StateGraph with harness_core primitives."""
    
    def __init__(self, graph: StateGraph, *, checkpointer=None, observability=None, permissions=None):
        self.graph = graph.compile(checkpointer=checkpointer or PostgresSaver(...))
        ...
    
    async def run(self, input, *, thread_id, traceparent=None):
        ...
```

### SLO + Runbook framework

```python
# src/harness_core/operations/sre/slo.py
@dataclass
class SLO:
    name: str
    sli: callable   # function returning current value
    target: float
    error_budget: float
    
class SLOMonitor:
    def check_all(self) -> list[Breach]: ...
    def alert_on_breach(self, slo: SLO, severity: str): ...

# src/harness_core/operations/sre/runbooks/cost_runaway.py
class CostRunawayRunbook(Runbook):
    def detect(self) -> bool: ...
    def triage(self, incident: Incident) -> TriageResult: ...
    def mitigate(self, incident: Incident) -> MitigationResult: ...
    def postmortem_template(self) -> str: ...
```

### Compliance check

```python
# src/harness_core/compliance/eu_ai_act.py
class EUAIActComplianceCheck:
    def classify_risk_tier(self, agent_metadata: AgentMetadata) -> RiskTier: ...
    def list_obligations(self, tier: RiskTier) -> list[Obligation]: ...
    def evidence_for_obligation(self, obligation: Obligation) -> Evidence: ...
```

## Migration plan

### Source consolidation

| `harness_core/` package | Source |
|---|---|
| `foundation/permissions.py` | promote `polaris-core/permissions/` |
| `foundation/hooks.py` | promote `polaris-core/hooks/` |
| `foundation/bright_lines.py` | promote `polaris-core/bright_lines/` |
| `foundation/daemon.py` | promote `polaris-daemon/daemon.py` |
| `foundation/cost_router.py` | promote `polaris-core/cost_router/` |
| `operations/observability/hir_events.py` | promote `polaris-core/observability/hir.py` |
| `protocols/mcp/` | promote `polaris-core/mcp_adapter/` + extend |
| `memory/` | promote `lyra-core/memory/` (13 modules) |
| `skills/skill_engine.py` | promote `lyra-skills/` core |
| `protocols/a2a/`, `protocols/agntcy/`, `protocols/nats_transport/` | new |
| `runtime/*_adapter.py` | new (thin wrappers) |
| `security/classifiers/`, `security/spotlight.py`, `security/isolation/` | new |
| `operations/evaluation/`, `operations/durability/`, `operations/sre/` | new |
| `compliance/` | new |
| `routines/` | new |
| `ux/` | new |

### Migration phases

| Phase | Duration | Scope |
|---|---|---|
| **M1** | 30 days | Bootstrap repo, set up build / CI / docs / package skeleton |
| **M2** | 60 days | Promote `foundation/` from polaris-core; promote `memory/` from lyra-core; tests |
| **M3** | 60 days | Build `protocols/` (MCP refactor + new A2A + AGNTCY + NATS) |
| **M4** | 60 days | Build `security/` (classifiers + isolation + supply-chain) |
| **M5** | 60 days | Build `operations/` (observability + eval + durability + SRE) |
| **M6** | 60 days | Build `routines/`, `compliance/`, `ux/`, `runtime/` adapters |
| **M7** | 60 days | Migrate Polaris first (validate); then Lyra; then others in parallel |
| **M8** | 60 days | Documentation polish; external-publish; first external adopter |

**Total: ~12 months from M1 start to v1.0 release with all 14 in-tree projects on the shared library.**

## Per-project rollout sequencing

| Project | When | Notes |
|---|---|---|
| **Polaris** | First (M7) | Most existing primitives; validates promotion |
| **Lyra** | Second | Memory module is the source; refactor to import own promoted code |
| **Argus** | Third | Provider role; needs marketplace + AGNTCY first |
| **Mentat-Learn** | Fourth | Multi-channel; needs UX + voice |
| **Aegis-Ops** | Fifth | Operator role; needs full L6 + L7 |
| **Cipher-Sec** | Sixth | Most security-rigorous; needs full L5 + L7 |
| **Atlas / Helix / Harmony / Vertex / Gnomon / Orion / Syndicate / Quanta-Proof / Open-Fang** | Parallel | After core 6 stabilize |

## Public versioning + release

- **Semver**: 0.x during M1-M7; 1.0 at first stable cross-project deployment.
- **Breaking changes** in 0.x; deprecation warnings; migration guides.
- **License**: Apache-2.0 (matches NATS, Tailscale, Phoenix, A2A; permissive for in-tree + external use).
- **Distribution**: PyPI + GitHub releases; Docker images for `harness_core/` services.

## Compute + cost estimate

- **Engineering effort**: 3-4 senior engineers × 12 months = ~$700K-1.2M loaded.
- **Infrastructure**: $1-5K/month for CI / artifact storage / dev infra during build.
- **External audit** (when seeking external adopters): SOC 2 prep ~$50K; ISO 42001 ~$30K.

## ROI

- **14 projects × ~$100-200K port savings each** vs independent implementation = $1.5-3M saved.
- **Cross-project capability sharing**: marketplace-published skills work for all consumers.
- **Maintenance cost reduction**: one maintained codebase vs 14.
- **External adoption potential**: shared library for the broader agent ecosystem.

Net ROI: positive within the first 2-3 projects ported.

## One-line takeaway

**`harness_core/` is the shared library that consolidates 35 chapters of design corpus into Python — eleven sub-packages (foundation + capability + protocols + runtime + security + operations + compliance + routines + memory + skills + ux), promoted from canonical implementations in polaris-core + lyra-core + lyra-skills, built across eight 60-day migration phases (~12 months total), with Polaris and Lyra as the first two adopters validating the promotion, then Argus / Mentat / Aegis / Cipher in sequence, then the remaining nine projects in parallel — turning the deep-dive corpus from docs into the Python package that makes the in-tree ecosystem internally consistent and externally reusable.**
