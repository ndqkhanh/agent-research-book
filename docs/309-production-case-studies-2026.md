# 309 — Production Case Studies 2026: Klarna, Replit, Elastic, NVIDIA, Snowflake, Stripe

**Anchors.** Disclosed production deployments — Klarna AI customer-service (LangGraph), Replit Agent (LangGraph), Elastic Security Investigation Agent, NVIDIA AgentIQ, Snowflake Cortex Agents, Stripe operations agents, plus public case studies from Anthropic / OpenAI customers.

**One-line definition.** A 2026 picture of **what production agents look like in deployment** — six well-documented case studies (Klarna's customer service agent at 700+ FTE-equivalent capacity, Replit's coding agent at scale, Elastic's security investigation agent, NVIDIA AgentIQ multi-agent orchestrator, Snowflake Cortex Agents for data, Stripe's ops automation) — covering **architectural choices** (which runtime, which protocol, which memory pattern, which isolation), **operational realities** (cost per task, P99 latency, eval-pass-rate, on-call burden), and **honest postmortems** (what failed, what was rebuilt) — making the abstract design corpus concrete by anchoring to deployments practitioners can study.

## Why production case studies matter

The 250–296 corpus is **design-grounded** — patterns, primitives, decision matrices. Production deployments are what test those designs against reality. The 2024-2026 wave of public agent deployments lets the field study **what actually works at scale** vs what's theoretically clean. This file consolidates six well-documented cases into actionable lessons.

## Case 1: Klarna AI Customer Service (LangGraph)

**Disclosed:** 2024 → ongoing scaling; reportedly handling work equivalent to ~700 FTE customer-service agents.

- **Runtime:** LangGraph for state-machine ticket workflows.
- **Memory:** ReasoningBank-shape per-customer + per-issue-type.
- **Verifier:** LLM-judge ensemble + human review on high-confidence-low-precision band.
- **UX:** Human handoff via `interrupt()` when confidence drops or escalation rules trigger.
- **Observability:** LangSmith.
- **Lessons:** state-machine + checkpoint critical for 7-figure-monthly-volume; eval-pass-rate is the load-bearing SLI; quality regression detection within 48h is the difference.

## Case 2: Replit Agent (LangGraph)

**Disclosed:** 2024 → 2026 with the Agent v3 release.

- **Runtime:** LangGraph for stateful coding workflows.
- **ACI:** Custom file editor + sandbox + preview deploy + DB integration.
- **Memory:** Per-user project-context + skill library.
- **Verifier:** Test-suite + linter + LLM-as-judge for code quality.
- **Isolation:** Replit's sandbox VMs (Firecracker-shape).
- **Lessons:** custom ACI is the dominant lever (per [236](236-tool-use-and-aci-scaling.md)); ~3-5× productivity gain over generic shell.

## Case 3: Elastic Security Investigation Agent

**Disclosed:** Elastic's 2024-2026 security-incident-response agent.

- **Runtime:** LangGraph with hybrid LLM-judge + traditional SIEM rules.
- **Use case:** correlate alerts → produce investigation timeline → suggest remediation.
- **Verifier:** cross-channel (Anthropic + OpenAI verifier) + analyst sign-off.
- **Bright-line gates:** every prod-modifying suggestion gated.
- **Compliance:** SOC 2 + ISO 27001.
- **Lessons:** for security-domain agents, false-positive rate matters more than recall (analysts already drown in alerts).

## Case 4: NVIDIA AgentIQ

**Disclosed:** NVIDIA's open-source multi-agent orchestrator (2025).

- **Runtime:** Custom event-driven (AutoGen v0.4 inspiration).
- **Use cases:** RAG, code generation, customer service.
- **Profiling-aware:** built-in token-cost + latency tracking per agent.
- **Lessons:** profiling primitives in the runtime save hours of post-hoc analysis; bundle observability with orchestration.

## Case 5: Snowflake Cortex Agents

**Disclosed:** Snowflake's 2024-2026 data-question-answering agents.

- **Runtime:** Snowflake-internal; equivalent to LangGraph for state-machine SQL+text workflows.
- **Memory:** Per-warehouse schema + query-history.
- **Verifier:** SQL syntax + result-shape + business-rule classifiers.
- **Lessons:** structured-output + grounded-citation is the difference between "the agent answered" and "the analyst trusts the answer."

## Case 6: Stripe Ops Automation

**Disclosed:** Stripe's 2024-2026 internal ops agents (incident triage, dispute review).

- **Runtime:** Custom + LangGraph for newer.
- **Bright-line discipline:** every customer-money-touching action human-approved.
- **Cost economics:** $0.05-0.50 per task; 100x cheaper than human equivalent.
- **Lessons:** for financial workflows, bright-line-first design is non-negotiable; even high-confidence agent suggestions go through human queue.

## Cross-cutting lessons

| Pattern | Validated by |
|---|---|
| LangGraph for production state-machine workflows | All six |
| Custom ACI is the dominant productivity lever | Replit |
| LLM-judge + human review on confidence-band | Klarna, Elastic |
| Bright-line gates for money/prod-modify | Stripe, Replit |
| Cross-channel verifier for high-stakes | Elastic |
| Profiling-aware orchestration | NVIDIA AgentIQ |
| Structured output + grounded citation for trust | Snowflake |
| Memory tiering per-customer / per-project | Klarna, Replit |
| Eval-pass-rate as the load-bearing SLI | All |
| Quality regression detection within 48h | Klarna |

## What didn't work (postmortems)

- **AutoGen v0.2 GroupChat in production:** failure rates per [251](251-multi-agent-teams-2026-synthesis.md) MAST audit; teams migrated to structured channels.
- **Per-project bespoke observability:** teams built proprietary stacks; LangSmith / Phoenix / OTel-shape adoption replaced.
- **Implicit-state agents:** AgentExecutor pre-LangGraph; lost 30+ minutes of work per crash.
- **No cost cap:** runaway loops cost $10K+ in hours; bright-line cost gates added.
- **No eval pipeline:** 10pp quality regressions silent for days.

## What's missing from public disclosure

- **Honest cost numbers** at production scale.
- **Honest false-positive rates** in safety-critical domains.
- **Detailed multi-agent failure-mode breakdown** (MAST cluster classification).
- **Reward-hacking case studies** in RLVR-trained models.
- **Post-incident root-cause analyses** with full trace replays.

This is the gap a more transparent industry would fill.

## One-line takeaway

**Production agent deployments in 2026 validate the corpus's design choices — LangGraph + checkpointer + custom ACI + LLM-judge ensemble + bright-line gates + cross-channel verifier + memory tiering + eval-pass-rate as load-bearing SLI — across Klarna / Replit / Elastic / NVIDIA / Snowflake / Stripe; the cross-cutting lesson is that **eval discipline + bright-line gates + observability** are the three non-negotiables that distinguish a production deployment from a demo, and the two industry gaps are honest cost / quality numbers and detailed multi-agent failure-mode breakdowns.**
