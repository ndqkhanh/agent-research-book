# 268 — Agent Operations Synthesis 2026: the four legs (observability + evaluation + durability + SRE) and the operations stack

**Synthesis of:** [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md), [265-agent-evaluation-2026](265-agent-evaluation-2026.md), [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md), [267-agent-sre](267-agent-sre.md). Cross-references: [258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md), [263-production-agent-runtime-synthesis-2026](263-production-agent-runtime-synthesis-2026.md), [225-agent-era-scaling-synthesis](225-agent-era-scaling-synthesis.md), [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md), [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [97-qwen-prm](97-qwen-prm.md), [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md), [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md), [115-evaluating-llm-systems](115-evaluating-llm-systems.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md).

**One-line definition.** A unified **operations stack** for production agents in 2026 — **four legs** of equal load-bearing weight (**Observability** for visibility, **Evaluation** for quality measurement, **Durability** for crash safety, **SRE** for operating discipline) layered above the protocol stack ([258](258-agent-protocol-stack-synthesis-2026.md)) and runtime selection ([263](263-production-agent-runtime-synthesis-2026.md)) — composed deliberately so that **observability spans feed evaluation pipelines feed SLO measurements feed SRE runbooks feed rollback decisions feed back into observability**, closing the production-quality feedback loop and giving operators the toolkit to ship agents that survive contact with paying users, regulated industries, and adversarial environments.

## Why this synthesis matters (the operations stack is what turns "the agent works in demos" into "the agent ships to production")

The hardest lesson of the 2024–2025 production-agent wave was that **shipping a working agent ≠ operating a production agent**. The same agent that scored 65% on GAIA in a demo can fail badly in production because: cost runs away (no observability + no SLO + no runbook), quality regresses silently (no eval pipeline + no production-eval feedback + no rollback), crashes lose work (no durability + no idempotency), incidents pile up (no on-call + no postmortems). The four legs of the operations stack — observability ([264](264-agent-observability-stack-2026.md)), evaluation ([265](265-agent-evaluation-2026.md)), durability ([266](266-agent-durability-and-idempotency.md)), SRE ([267](267-agent-sre.md)) — are not optional add-ons; they are the engineering disciplines that distinguish prototype-grade from production-grade.

The 2026 operations stack composes deliberately. Observability ([264](264-agent-observability-stack-2026.md)) provides the **visibility** — OTel-shape spans with HIR-shape typed events, cross-runtime correlation via `traceparent`, cost / latency / quality drill-down, eval-link annotations on production traces. Evaluation ([265](265-agent-evaluation-2026.md)) provides the **quality measurement** — eval-driven development as a discipline, canonical benchmarks for capability claims, production-eval feedback loops via observability, statistical methodology for confident comparisons. Durability ([266](266-agent-durability-and-idempotency.md)) provides the **crash safety** — durable-execution layer (LangGraph checkpointer / Temporal / Restate / Inngest / Cloudflare Workflows), idempotency keys on every effectful tool, saga/outbox patterns for multi-step external transactions, replay safety via non-determinism isolation. SRE ([267](267-agent-sre.md)) provides the **operating discipline** — SLOs across availability+quality+cost+latency+escalation-rate, runbooks for agent-specific incidents (cost runaway, quality regression, escalation flood, memory poisoning, supply-chain compromise), four-axis rollback (model+prompt+skill+runtime), MAST-classified postmortems, capacity planning for tokens.

The composition is the high-leverage architectural move. Each leg standalone has value; the four together close a feedback loop: observability → eval → SLO → SRE-driven rollback → improved observability. Production agent platforms that ship the four legs deliberately outperform those that don't, regardless of model or runtime choice. This synthesis is the cheat sheet for that composition.

Take this synthesis seriously and three things change. **First**, you stop thinking of operations as a "we'll add it later" concern and start treating it as **part of the architecture from day one** — every project in `projects/` gets all four legs from `harness_core/`. **Second**, you understand that **the four legs feed each other** — observability feeds eval (production traces with eval-link annotations); eval feeds SLO (eval-pass-rate is a SLI); SLO feeds SRE (breach triggers runbook); SRE feeds rollback (which is an action the durability layer supports). **Third**, you treat **production-grade as a measurable threshold** — your agent platform is production-grade when it has the four legs deployed, the SLOs defined, and the runbooks rehearsed; before that it's a sophisticated prototype.

## The four legs

### Leg 1 — Observability ([264](264-agent-observability-stack-2026.md))

- **Wire format:** OpenTelemetry spans with `traceparent` propagation across A2A and MCP.
- **Typed taxonomy:** HIR-shape events — `Tool.call`, `LLM.call`, `Verifier.verdict`, `Permission.decision`, `BrightLine.escalation`, `Memory.*`, `Skill.invoke`, `Routine.fire_*`, `Team.*`.
- **Consumers:** LangSmith (LangGraph shops), Phoenix (OSS), Helicone (LLM-cost), Honeycomb / Datadog / Cloud Trace (general APM).
- **Capabilities:** cost attribution per span, latency drill-down, cross-runtime correlation, eval-link annotations.

### Leg 2 — Evaluation ([265](265-agent-evaluation-2026.md))

- **Canonical benchmarks:** GAIA, OSWorld, GDPval, SWE-bench Verified, AgentBench, MultiAgentBench, MAST (failure taxonomy).
- **Discipline:** eval-driven development (EDD); CI runs evals; regressions block merge.
- **Production-eval feedback loop:** observability spans with eval-link annotations → eval dataset → re-run.
- **Statistical methodology:** sample sizing, confidence intervals, paired comparisons, Bonferroni correction, MAST-cluster-rate breakdown.
- **Verifier hierarchy:** hard-rule > simulator > PRM > LLM-judge > human spot-check.

### Leg 3 — Durability ([266](266-agent-durability-and-idempotency.md))

- **Durable execution:** LangGraph checkpointer (in-runtime), Temporal (workflow-as-code), Restate / Inngest / Cloudflare Workflows (SaaS-shape), custom event-sourcing.
- **Idempotency:** per-tool idempotency keys; dedup at receiver; safe retry semantics.
- **Saga / outbox:** multi-step transactional patterns with compensating actions; exactly-once delivery.
- **Replay safety:** non-determinism isolated at workflow boundary; `workflow.uuid4()`, `workflow.now()` instead of raw.

### Leg 4 — SRE ([267](267-agent-sre.md))

- **SLOs/SLIs:** availability, quality (eval-pass-rate), cost, latency, escalation-rate, memory integrity, skill drift.
- **Runbooks:** cost runaway, quality regression, escalation flood, memory poisoning, marketplace-skill compromise.
- **Four-axis rollback:** model, prompt, skill, runtime.
- **On-call rotation** with agent-specific tooling (replay, time-travel, eval re-run).
- **MAST-classified postmortems** for multi-agent incidents.
- **Capacity planning** for tokens, query volume, eval cost, observability storage.

## How the four legs compose (the feedback loop)

```
[Production agent run]
        │
        ▼
[Observability spans]──────────────────────────►[OTel-shape store]
        │                                                │
        │                                                │
        ├──► [Cost attribution]                          │
        │     │                                          │
        │     └──► [SLO: cost SLI]                       │
        │                                                │
        ├──► [Eval-link annotations]                     │
        │     │                                          │
        │     └──► [Eval dataset growth]                 │
        │           │                                    │
        │           └──► [Eval-pass-rate SLI]            │
        │                                                │
        ├──► [Bright-line escalation events]             │
        │     │                                          │
        │     └──► [Escalation-rate SLI]                 │
        │                                                │
        └──► [Latency P99]                               │
              │                                          │
              └──► [Latency SLI]                         │
                                                         │
                                                         ▼
                                                  [SLO breach detection]
                                                         │
                                                         ▼
                                                  [SRE runbook execution]
                                                         │
                                                         ▼
                                          [Rollback action via four axes]
                                                         │
                                                         ▼
                                          [Durability layer applies rollback]
                                                         │
                                                         ▼
                                          [Observability captures recovery]
                                                         │
                                                         ▼
                                          [Postmortem with MAST classification]
                                                         │
                                                         ▼
                                          [Action items: eval cases, runbook updates,
                                                         SLO recalibration, instrumentation gaps]
                                                         │
                                                         ▼
                                                  [Next iteration]
```

Each arrow is a contract between adjacent legs. The full loop closes within hours-to-days.

## The operations stack stacked above protocol + runtime

```
Layer 6 — Operations    Observability + Evaluation + Durability + SRE  ([268] — this file)
Layer 5 — Runtime       LangGraph / Agents SDK / AutoGen / ADK / CrewAI / Agent Teams ([263])
Layer 4 — Protocol      MCP × A2A × AGNTCY × NATS+Tailscale × SKILL.md × Routines+Teams ([258])
Layer 3 — Capability    Pretraining × TTC × Trajectory × Multi-agent × Verifier ([225])
Layer 2 — Foundation    Permission Bridge, Daemon, Bright-line Gates, Cost Router
Layer 1 — Hardware      LLM provider, compute, storage
```

Production agent engineering is full-stack: capability axes for height, protocol stack for composition, runtime for execution, operations for production-quality.

## Per-regime adoption matrix

| Regime | Observability | Evaluation | Durability | SRE |
|---|---|---|---|---|
| **Prototype / one-shot** | Skip or stdout | Spot-check | Skip | Skip |
| **Personal agent (Lyra-class)** | Phoenix local | Basic eval suite | LangGraph checkpointer | Cost-runaway alert |
| **Team-internal agent** | Phoenix / LangSmith | EDD with CI | LangGraph + Postgres | Runbooks + on-call |
| **Customer-paying SaaS agent** | Full stack | Full EDD + production-eval loop | Temporal / LangGraph | Full SRE + game days |
| **Regulated agent** | Full + audit retention | Full + compliance evals | Full + audit logging | Full + compliance docs |
| **Multi-team org platform** | Multi-tenant Phoenix / LangSmith | Shared eval suite + per-team subset | Cross-team durability infra | Shared SRE + per-team SLOs |

## The cost of the operations stack

| Leg | Setup cost | Ongoing cost | ROI |
|---|---|---|---|
| **Observability** | 1–2 engineer-weeks | $0–500/mo (free tier sufficient for many) | High (debug + cost attribution) |
| **Evaluation** | 2–4 engineer-weeks | $50–500/mo (eval costs) | Very high (quality regression prevention) |
| **Durability** | 1–3 engineer-weeks | $0–200/mo (Postgres or Temporal Cloud) | High (production crash recovery) |
| **SRE** | 2–4 engineer-weeks initial; ongoing | On-call burden | Critical at scale |

Total: ~6–13 engineer-weeks for full deployment; ~$50–1500/month ongoing. Pays back the first time a production incident is averted or detected early.

## Empirical results (table — what production agent shops have shipped)

| Shop | Stack |
|---|---|
| Anthropic Claude Code | Native HIR + observability; internal tooling; experimental Agent Teams |
| OpenAI Agents SDK users | Auto-uploaded tracing to OpenAI dashboard; OpenAI Evals |
| LangChain shops (Klarna, Replit, Elastic, etc.) | LangSmith + LangGraph checkpointer + custom SRE |
| Google ADK users | Cloud Trace + Logging + Vertex Agent Engine |
| Microsoft AutoGen users | OTel + Azure Monitor + custom |
| Personal-agent open source (Lyra, Mentat-Learn) | Phoenix + SQLite checkpointer + manual SRE |

## Variants and ablations

- **Phoenix-only.** OSS, free, sufficient for many use cases.
- **LangSmith-only.** Polished UX, LangGraph-native.
- **Multi-consumer.** OTel Collector forwards to Phoenix + Honeycomb + Helicone.
- **Eval-only canary deploys.** Promote to prod when eval-pass-rate exceeds threshold.
- **Chaos-engineering for agents.** [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md), [53-chaos-engineering-next-era](53-chaos-engineering-next-era.md) — game days.
- **Per-routine SLOs.** Different routines have different cost / quality budgets.
- **Multi-agent SLOs.** Per-team SLOs aggregated to platform SLOs.
- **A/B testing infrastructure.** Built on cost router + eval pipeline.
- **Trust-tier monitoring.** [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md) — auto-alert on retraction events.
- **Production replay.** Save production traces; replay against new versions; compare quality.

## Failure modes when the four legs are imbalanced

- **Observability without eval:** you see the slow request but can't tell if quality regressed.
- **Eval without observability:** you know quality dropped but not where in the trace.
- **Eval without durability:** you can measure quality but a crash mid-run loses the eval data.
- **Durability without SRE:** you have crash safety but no on-call to handle escalations.
- **SRE without observability:** you have runbooks but no visibility to know when to execute them.
- **All four without protocol stack ([258](258-agent-protocol-stack-synthesis-2026.md)):** you have ops in a silo; cross-runtime correlation fails.

## When to use, when not

**Adopt the full operations stack** for production deployments with paying users / regulatory requirements / quality SLAs / multi-team coordination. The strongest cases are **enterprise agent platforms**, **paying-customer SaaS agents**, **regulated agents** (healthcare, finance), **multi-team org platforms**, and **safety-critical agents**.

**Skip components selectively** for personal agents (basic observability + cost alerts suffice), prototypes (defer until production), or single-user low-stakes deployments. The minimum viable production stack is: cost-runaway alert + eval CI + idempotency keys on tool calls + one runbook for the top failure mode.

## Implications for harness engineering

- **`harness_core/operations/` package.** Shared observability + eval + durability + SRE primitives across all 14 agents.
- **OTel SDK as first-class import.** [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md), [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), [24-observability-tracing](24-observability-tracing.md).
- **EDD via shared eval framework.** [265-agent-evaluation-2026](265-agent-evaluation-2026.md), [115-evaluating-llm-systems](115-evaluating-llm-systems.md).
- **Durability via LangGraph checkpointer or Temporal.** [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md), [259-langgraph-deep-dive](259-langgraph-deep-dive.md), [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md).
- **SRE primitives as `harness_core/sre/`.** [267-agent-sre](267-agent-sre.md) — SLOs, runbooks, rollback infrastructure.
- **Per-project SLOs.** [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md), [208-lyra-multi-hop-collaborative-apply-plan](208-lyra-multi-hop-collaborative-apply-plan.md), [209-argus-multi-hop-collaborative-apply-plan](209-argus-multi-hop-collaborative-apply-plan.md), [210-mentat-learn-collaborative-apply-plan](210-mentat-learn-collaborative-apply-plan.md), [211-cross-project-power-up-plan-with-tradeoffs](211-cross-project-power-up-plan-with-tradeoffs.md).
- **Observability + eval feedback loop.** Production traces with eval-link annotations; nightly eval-set growth.
- **MAST classification in postmortems.** [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md), [98-diversity-collapse-mas](98-diversity-collapse-mas.md).
- **Trust-tier monitoring.** [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md), [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md), [p24-trust-tiering-retractions](../projects/polaris/docs/research/p24-trust-tiering-retractions.md).
- **Cross-channel verifier as quality gate.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md).
- **Distributed durability via Tailscale + NATS.** [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md).
- **Bright-line gates as SRE alert source.** [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md).
- **Cost router with operations feedback.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md), [88-confidence-driven-router](88-confidence-driven-router.md).
- **Memory ops alignment.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md) — memory writes are idempotent, observable, eval-able, recoverable.

**One-line takeaway for harness designers.** **The 2026 agent operations stack is four legs of equal weight (observability + evaluation + durability + SRE) layered above the protocol stack and runtime selection — they feed each other in a closed feedback loop (observability → eval → SLO → SRE-driven rollback → durability-supported recovery → improved observability) and shipping the four together is the difference between a sophisticated prototype and a production-grade agent that survives paying users, regulated industries, and adversarial environments; pull the four into a shared `harness_core/operations/` package and every project in `projects/` becomes production-ready by import.**
