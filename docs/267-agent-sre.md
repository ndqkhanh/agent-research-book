# 267 — Agent SRE: on-call, runbooks, cost-runaway alerts, quality regression, and the operations discipline production agents need

**Anchors.** Site Reliability Engineering — Google's *Site Reliability Engineering* (Beyer, Jones, Petoff, Murphy 2016) — the canonical SRE reference. SRE-for-LLMs / SRE-for-agents — emerging discipline 2024–2026. Companions: [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md), [265-agent-evaluation-2026](265-agent-evaluation-2026.md), [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md), [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md), [123-robustness-fault-tolerance](123-robustness-fault-tolerance.md), [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md), [53-chaos-engineering-next-era](53-chaos-engineering-next-era.md).

**One-line definition.** A 2026 picture of **Site Reliability Engineering for production agents** — the discipline that takes Google-SRE primitives (SLOs, SLIs, error budgets, runbooks, incident response, postmortems, capacity planning, rollback strategies) and adapts them to the **agent-specific failure modes** (cost-runaway from looping LLM calls, quality regression from prompt drift or model swap, bright-line escalation floods, MAST-cluster-2 multi-agent failures, verifier mis-calibration, memory poisoning, supply-chain compromise via marketplace skills) — making "the agent had a bad day" something with a runbook, an on-call rotation, and a postmortem template, not a Slack DM and a manual fix.

## Why this paper matters (without SRE practices, agents fail in production-shaped ways the field is unprepared for)

The first generation of production agent deployments (2023–2024) failed predictably: cost runaways from looping LLM calls burned through monthly budgets in hours; prompt updates silently degraded quality 10pp without anyone noticing for days; bright-line escalations piled up unanswered because no one was on-call; multi-agent debates collapsed into wrong-answer agreement (MAST cluster-2) without any alert; production memory got poisoned by an adversarial input that propagated to all subsequent runs. **None of these failure modes have analogs in pre-LLM software**, which is why classical SRE practices need to be **adapted, not adopted**, for agents.

The 2026 picture: SRE-for-agents is an emerging discipline with a converged shape. **SLOs adapted to quality + cost + latency + escalation-rate** rather than just availability. **SLIs include eval-suite pass rate, per-routine cost, P99 latency, bright-line-escalation rate** ([16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md)). **Error budgets** include both classical down-time and quality-regression budget (you can ship updates if eval-pass-rate stays above the SLO). **Runbooks** for agent-specific incidents: cost runaway, quality regression, escalation flood, memory poisoning, marketplace-skill compromise. **On-call** rotates engineers as it does for any service, but with agent-specific tooling (replay, time-travel, eval re-run). **Postmortems** include MAST-cluster classification ([251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md)) for multi-agent incidents. **Capacity planning** factors in token-cost growth, not just CPU/memory. **Rollback strategies** cover four dimensions — model, prompt, skill, runtime — each with its own rollback mechanism.

The economic and trust stakes are real. A single cost runaway can cost $10K-$100K in hours; a quality regression on a customer-facing agent shows up in NPS and churn within days; a memory-poisoning incident is a security event with audit-trail consequences. Agents at production scale need the SRE primitives the rest of distributed-system software has had for two decades, plus the agent-specific extensions.

Take this seriously and three things change. **First**, you adopt **agent-specific SLOs/SLIs/error-budgets** — quality, cost, latency, escalation-rate as first-class metrics with thresholds and budgets. **Second**, you write **agent-specific runbooks** — cost-runaway, quality-regression, escalation-flood, memory-poisoning, supply-chain-compromise — and rehearse them. **Third**, you build **rollback infrastructure** for four orthogonal axes (model, prompt, skill, runtime) — pinning to known-good versions, fast deployment of rollbacks, observability into which axis a degradation came from.

## Problem it solves (operating production agents at quality, cost, and trust SLAs)

1. **No agent-specific SLO vocabulary.** Classical "99.9% uptime" doesn't capture quality, cost, escalation-rate.
2. **Cost runaways are silent.** Without budget alerts, a looping agent burns through $10K in hours.
3. **Quality regressions are invisible without eval.** Pre-eval-driven-development, prompt or model changes can degrade quality 10pp silently.
4. **Bright-line escalations need on-call.** Without rotation, escalations queue up.
5. **MAST cluster-2 multi-agent failures need detection.** Production-eval feedback loop catches them.
6. **Memory poisoning has no parallel in classical software.** Adversarial inputs propagate via memory; needs runbook.
7. **Marketplace-skill compromise is a supply-chain risk.** Signed manifests + auto-rollback.
8. **Rollback is multi-axis.** Model, prompt, skill, runtime — each rollback path differs.
9. **Capacity planning includes token cost.** Pre-LLM SRE never had to forecast tokens-per-month.
10. **Postmortem culture for agent failures.** Without templates, incidents repeat.

## Core idea in one paragraph

SRE-for-agents adapts Google-SRE primitives to agent-specific failure modes. **SLOs/SLIs include four dimensions** — availability (classical), quality (eval-pass-rate ≥ X% over rolling N runs), cost (per-routine spend < $Y), latency (P99 < Z seconds), escalation-rate (bright-line-escalations < W per hour). **Error budgets** combine these — you can ship updates if all four stay within budget; pause feature work if any breaches. **Runbooks** for agent-specific incidents: (a) **cost runaway** — kill the runaway routine, lower per-task budget, rotate to cheaper model, alert on root cause; (b) **quality regression** — rollback to last-known-good prompt/model/skill, replay production traces against rolled-back version, file regression-bug; (c) **escalation flood** — suppress non-critical escalations, escalate to specialist, write postmortem; (d) **memory poisoning** — flush poisoned memory partition, audit dependent runs, file CVE-shape report; (e) **marketplace-skill compromise** — auto-rollback to vendored fallback, trust-tier downgrade, audit dependent runs. **On-call rotation** with agent-specific tooling (replay, time-travel, eval re-run, observability). **Rollback infrastructure** for four orthogonal axes — model (pin and re-pin), prompt (versioned templates with bisect), skill (marketplace + vendored fallback), runtime (canary deploy with auto-revert). **Postmortems** include MAST-cluster classification for multi-agent incidents and route-cause analysis across the four axes. **Capacity planning** forecasts tokens-per-month, query-volume growth, eval-suite cost, observability storage. The discipline turns "the agent had a bad day" from a Slack DM into a runbook execution with measurable SLOs and a postmortem-driven improvement loop.

## Mechanism (step by step)

### (a) Agent-specific SLOs and SLIs

| Dimension | SLI (measurement) | SLO (target) | Error budget |
|---|---|---|---|
| **Availability** | % of routine fires that succeed | 99.5% / month | 0.5% / month |
| **Quality** | Eval-pass-rate over rolling 7 days | ≥ 92% | drop > 2pp |
| **Cost** | Per-routine spend P95 | < $0.50 | budget overrun |
| **Latency** | P99 wall-clock per routine | < 30s | P99 > 60s |
| **Escalation rate** | Bright-line escalations / hour | < 1.0 | > 5/hr |
| **Memory integrity** | % memory writes passing redactor | 100% | any failure |
| **Skill drift** | % installed skills passing regression | 100% | any failure |

### (b) Runbook structure (template)

```markdown
# Runbook: <Incident type>

## Symptoms
- ...

## Detection
- alert source: <Datadog / Phoenix / custom>
- threshold: <SLO breach>

## Triage
1. Check observability for trace_id of failing request.
2. Check eval suite for recent regression.
3. Check cost dashboard for spike.
4. Check bright-line escalation log.

## Mitigation
1. Immediate: <kill-switch, rollback>
2. Short-term: <fix-forward>
3. Long-term: <postmortem action items>

## Rollback procedure
- Model: ...
- Prompt: ...
- Skill: ...
- Runtime: ...

## Communication
- Page on-call: ...
- Status page update: ...
- User notification: ...
```

### (c) Cost-runaway runbook

```markdown
# Runbook: Cost Runaway

## Symptoms
- Per-routine cost > $5 (vs SLO $0.50)
- Token-rate climbing across multiple agents

## Detection
- Datadog alert: agent_cost_per_routine_p95 > 5
- Helicone budget breach

## Triage
1. Identify offending routine_id from observability.
2. Check trace for tool-call loop pattern.
3. Verify bright-line cost gate state — should have fired.

## Mitigation
1. Kill running routine: harness_core routines kill <id>
2. Tighten cost cap: harness_core routines update <id> --cost-cap=1.00
3. Rotate to cheaper model: harness_core router pin --tier=cheap

## Postmortem
- Why didn't bright-line fire?
- Was the cap set correctly?
- Did the agent loop on a tool call? File bug.
```

### (d) Quality-regression runbook

```markdown
# Runbook: Quality Regression

## Symptoms
- Eval-pass-rate dropped > 2pp on rolling 7-day window.

## Detection
- LangSmith / Phoenix alert; CI eval comparison.

## Triage
1. Identify which subset of evals dropped.
2. Bisect: prompt change? model change? skill update? runtime upgrade?
3. Check production-eval feedback loop for recent failure cluster.

## Mitigation
1. Rollback offending axis (model/prompt/skill/runtime).
2. Re-run eval suite to confirm recovery.
3. Replay production traces against rolled-back version.

## Postmortem
- File regression bug.
- Add eval case for the regression pattern.
- MAST-cluster classification if multi-agent.
```

### (e) Escalation-flood runbook

```markdown
# Runbook: Escalation Flood

## Symptoms
- Bright-line escalations > 5/hr.

## Detection
- HIR alert: bright_line.escalation rate spike.

## Triage
1. Group escalations by code (PERMISSION_DENY, COST_CAP, VERIFIER_VETO, ...)
2. Identify dominant code.
3. Check whether code is correctly tuned.

## Mitigation
1. If false-positive flood: lower sensitivity for that code (with audit).
2. If true-positive flood: page specialist; pause affected routines.
3. If verifier-veto flood: re-calibrate verifier; check for adversarial input.
```

### (f) Memory-poisoning runbook

```markdown
# Runbook: Memory Poisoning

## Symptoms
- Eval failures correlated with shared memory usage.
- Adversarial-content patterns in memory writes.

## Detection
- Memory.write redactor failure.
- Eval-link annotations show poisoned-memory pattern.

## Triage
1. Identify poisoned partition + first poisoned write.
2. Audit downstream reads.
3. Check ingress source.

## Mitigation
1. Flush poisoned partition.
2. Rebuild from last known good snapshot.
3. Audit dependent runs; flag for human review.
4. Update redactor rules.

## Postmortem
- Why did redactor miss?
- Add adversarial input to red-team suite ([49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md)).
```

### (g) Marketplace-skill compromise runbook

```markdown
# Runbook: Marketplace-Skill Compromise

## Symptoms
- CVE published for skill / OASF retraction event.
- Anomalous behavior from a marketplace skill.

## Detection
- AGNTCY trust-tiering retraction event ([p24-trust-tiering-retractions](../projects/polaris/docs/research/p24-trust-tiering-retractions.md)).
- Eval regression after skill auto-update.

## Triage
1. Identify affected skill + version.
2. Audit dependent runs.

## Mitigation
1. Auto-rollback to last-known-good version pinned in `harness_core/skills/vendored/`.
2. Trust-tier downgrade for the vendor.
3. Audit affected runs.

## Postmortem
- File CVE if novel.
- Update skill-install policy (require minimum trust tier).
```

### (h) Four-axis rollback infrastructure

| Axis | Rollback mechanism |
|---|---|
| **Model** | Pin to specific model version (e.g., `claude-sonnet-4-5` not `claude-sonnet-latest`); `harness_core router pin <model_id>` |
| **Prompt** | Versioned prompt templates with git-backed bisect; `harness_core prompts rollback <agent> <version>` |
| **Skill** | Vendored `harness_core/skills/vendored/` fallback overrides marketplace; `harness_core skills pin <skill> <version>` |
| **Runtime** | Canary deploy with auto-revert; blue-green deployments; `harness_core runtime rollback <version>` |

### (i) Capacity planning

```python
# Forecast monthly tokens per agent
historical_tokens_per_month = [...]
trend = numpy.polyfit(range(len(historical_tokens_per_month)), historical_tokens_per_month, 1)
forecast = trend[0] * 12 + trend[1]  # 12 months out

# Forecast cost
avg_cost_per_token = ...
forecast_cost = forecast * avg_cost_per_token

# Plan capacity:
# - LLM provider quotas
# - Postgres for checkpointer
# - Phoenix / LangSmith span retention
# - JetStream for durable replay
```

### (j) On-call rotation

| Tier | Trigger | Response |
|---|---|---|
| **Tier 0** | Page (SLO breach, escalation flood, security event) | 15-min response, on-call engineer |
| **Tier 1** | Email (eval regression, cost spike, capacity warning) | Same-day, business hours |
| **Tier 2** | Dashboard (drift, capacity forecast) | Weekly review |

### (k) Postmortem template

```markdown
# Postmortem: <date> <incident name>

## Summary
- <one-line>

## Timeline
- T+0: ...
- T+5m: ...

## Impact
- Users affected: ...
- Cost impact: ...
- Quality impact: ...

## Root cause
- Axis: model / prompt / skill / runtime / multi-agent / memory / supply-chain
- Mechanism: ...
- MAST classification (if multi-agent): cluster-1 / cluster-2 / cluster-3 / failure-mode-N

## What worked
- ...

## What didn't
- ...

## Action items
- [ ] ...
- [ ] ...

## SLO impact
- Error budget consumed: ...
```

## Empirical results (table — emerging SRE-for-agents practices May 2026)

| Practice | Adoption (production agent shops) |
|---|---|
| Eval-pass-rate as SLI | High |
| Per-routine cost SLO | High |
| Bright-line escalation rate alert | Mature shops |
| Four-axis rollback infrastructure | Mature shops |
| MAST-classified postmortems | Multi-agent shops |
| Capacity planning for tokens | Mid-large shops |
| On-call rotation | Production-tier deployments |
| Runbooks for cost / quality / escalation | High |
| Runbooks for memory-poisoning / supply-chain | Security-mature shops |

## Variants and ablations

- **SRE-as-code.** Runbooks executable; one-click mitigation.
- **Game days.** Chaos engineering ([53-chaos-engineering-next-era](53-chaos-engineering-next-era.md)) for agents.
- **Red-team rotation.** [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md) — periodic adversarial exercises.
- **Eval-driven canary deploys.** New version runs in shadow; eval-pass-rate compared before promotion.
- **Cost-budget guardrails per user.** Per-user routine budgets prevent abuse.
- **Trust-tier policy enforcement.** [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md) — refuse below-threshold skills automatically.
- **Quality-as-code.** Eval suites checked into the repo; CI runs them.
- **Postmortem-driven action item tracking.** JIRA / Linear / GitHub issues; due dates; closure.

## Failure modes and limitations

- **Alert fatigue.** Too many alerts → ignored; tune thresholds carefully.
- **SLO mis-calibration.** Too strict = constant breaches; too loose = users notice degradation first.
- **Runbook rot.** Outdated runbooks worse than none; review quarterly.
- **Postmortem theater.** Going through motions without action items.
- **Rollback collateral damage.** Rolling back model affects all dependent agents.
- **Capacity forecast inaccuracy.** Token use is bursty; forecasts noisy.
- **Cross-team coordination.** Multi-team agent platforms need shared SLOs / runbooks.
- **Lag in production-eval feedback.** New failure modes appear in production before evals exist for them.
- **Game-day cost.** Chaos exercises consume tokens.

## When to use, when not

**Adopt SRE-for-agents practices** for any production deployment with paying users; any deployment with quality / cost / latency SLAs; any multi-agent platform; any agent in a regulated industry. The strongest cases are **paying-customer SaaS agents**, **enterprise internal agent platforms**, **regulated agents** (healthcare, finance), and **multi-team org agent platforms**.

**Skip detailed SRE practices** for personal-use agents (overhead exceeds value), prototypes (premature), or low-stakes single-user deployments. Maintain at minimum: cost-runaway alert, eval regression CI, runbook for the top three failure modes.

## Implications for harness engineering

- **`harness_core/sre/` package.** SLOs, alerting, runbooks, rollback infrastructure shared across 14 agents.
- **Per-project SLO definitions.** [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md), [208-lyra-multi-hop-collaborative-apply-plan](208-lyra-multi-hop-collaborative-apply-plan.md).
- **Bright-line escalation as alert.** [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — escalation triggers page.
- **Cost-router as SLO enforcer.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md) — route to cheaper tier when SLO breach approaches.
- **Permission Bridge as security boundary.** [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [05-hooks](05-hooks.md), [08-hooks-and-claim-gate](../projects/polaris/docs/blocks/08-hooks-and-claim-gate.md).
- **Daemon as health-check engine.** [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [p7-daemon](../projects/polaris/docs/research/p7-daemon.md) — periodic health probes.
- **Observability as SLI source.** [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md), [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), [24-observability-tracing](24-observability-tracing.md).
- **Eval as SLO source.** [265-agent-evaluation-2026](265-agent-evaluation-2026.md), [115-evaluating-llm-systems](115-evaluating-llm-systems.md).
- **Durability as recovery primitive.** [266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md), [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md).
- **Trust tiering as supply-chain defense.** [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md), [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md), [p24-trust-tiering-retractions](../projects/polaris/docs/research/p24-trust-tiering-retractions.md).
- **Cross-channel verifier as quality gate.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md).
- **Memory redactor as security primitive.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md).
- **MAST classification in postmortems.** [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md), [98-diversity-collapse-mas](98-diversity-collapse-mas.md).
- **Trajectory simulation for chaos exercises.** [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md), [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md), [53-chaos-engineering-next-era](53-chaos-engineering-next-era.md).
- **Routine pause/resume on incident.** [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md) — kill switch.

**One-line takeaway for harness designers.** **SRE-for-agents adapts Google-SRE primitives to agent-specific failure modes — SLOs across availability + quality + cost + latency + escalation-rate, runbooks for cost-runaway / quality-regression / escalation-flood / memory-poisoning / marketplace-compromise, four-axis rollback infrastructure (model + prompt + skill + runtime), MAST-classified postmortems, capacity planning for tokens — and the discipline turns "the agent had a bad day" into a runbook execution with measurable SLOs and a postmortem-driven improvement loop, the difference between a prototype and a production-grade deployment.**
