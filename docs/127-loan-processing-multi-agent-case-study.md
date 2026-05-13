# 127 — Loan-Processing Multi-Agent Case Study: Tying the Pattern Catalog Together with an End-to-End Example

**Source.** Arsanjani & Bustos, *Agentic Architectural Patterns for Building Multi-Agent Systems*, Chapters 13 (Single Agent for Loan Processing) and 14 (Multi-Agent System for Loan Processing); the canonical book-long worked example that demonstrates how the previous chapters' patterns combine in a realistic regulated-domain agentic workflow.

**One-line definition.** This chapter walks the loan-processing case study end-to-end as a working illustration of how the abstract patterns from chapters 118–126 — maturity-staged deployment, LLM selection per role, RAG+PEFT for skill+knowledge, hierarchical multi-agent coordination, explainability/compliance, fault tolerance, agent-level production, system-level platform, framework choice — combine into a single shipped agent system; loan processing is chosen because it is *regulated, multi-step, multi-stakeholder*, and *high-stakes*, exposing every pattern category at once.

## Why this matters

Pattern catalogs are abstract until you see them composed. Reading [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md) tells you orchestrator-specialist exists; reading [122-explainability-compliance](122-explainability-compliance.md) tells you decision records matter; reading [123-robustness-fault-tolerance](123-robustness-fault-tolerance.md) tells you circuit breakers help. But the *integration* — how these decisions interact, where they conflict, how the team makes the trade-offs — only becomes visible in a worked example.

Loan processing is the right case study because:
- **Regulated.** Fair-lending laws, GDPR Article 22, sector-specific compliance — every constraint from [122](122-explainability-compliance.md) applies.
- **Multi-step.** Application intake → identity verification → credit assessment → fraud check → underwriting → approval → notification — multiple coordination points.
- **Multi-stakeholder.** Applicant, loan officer, underwriter, compliance officer, customer support — different stakeholder views.
- **High-stakes.** A wrong approval is bad debt; a wrong rejection is a regulatory complaint.
- **Concrete metrics.** Approval rate, default rate, processing time, customer satisfaction — all measurable.

Reading the case study makes the rest of Part X actionable.

## Problem it solves

Five concrete operational gaps the case study addresses:

1. **Where do the patterns connect?** Without a worked example, each pattern feels independent.
2. **Where do trade-offs surface?** Compliance vs latency, cost vs quality — visible in context.
3. **What does a stage-3 implementation look like?** Maturity progression mapped to specific decisions.
4. **How do you explain this to leadership?** A concrete narrative makes the abstract concrete.
5. **What goes wrong in production?** The case study includes the realistic failure modes.

## Core idea in one paragraph

A loan-processing agent system has six interacting subagents (intake, identity-verification, credit-assessment, fraud-check, underwriting, notification) coordinated hierarchically by a LoanOrchestratorAgent. Knowledge comes from a RAG layer over policies and historical decisions; skill comes from PEFT-tuned specialists for fraud and underwriting; explainability is baked in via decision records and right-to-explanation surfaces; robustness is provided by circuit breakers on each external service (credit bureau, fraud DB) and a verifier loop on the underwriting decision; production patterns include cold start with policy preload, idempotency on application submission, multi-tenant isolation across banks, FinOps tagging per loan-decision, eval-gated deploys, and quarterly compliance audits. The choice of framework is LangGraph for orchestration + DSPy-compiled specialists + custom code at the regulatory boundary. The system runs at stage 3 maturity for a year, accumulates audit-grade evidence, advances to stage 4 only after demonstrated reliability and trust, and never reaches stage 5 because regulatory requirements mandate human oversight on every approval over a threshold.

## Mechanism (step by step)

### 1. The agent decomposition

```text
[USER: applicant submits loan application]
   ↓
[LoanOrchestratorAgent — hierarchical orchestrator]
   ├── [IntakeAgent] — parse application, validate completeness
   ├── [IdentityVerificationAgent] — KYC, document verification
   ├── [CreditAssessmentAgent] — pull credit bureau data, compute score
   ├── [FraudCheckAgent] — fraud-pattern detection (PEFT-tuned)
   ├── [UnderwritingAgent] — final decision (PEFT-tuned + verifier)
   └── [NotificationAgent] — send decision, with right-to-explanation
   ↓
[USER: receives decision, explanation, and contest path]
```

This is hierarchical orchestrator-specialist ([121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md)). Each specialist is an agent in its own right with its own model, prompts, and evaluation metrics.

### 2. Per-agent model selection (chapter 119 in practice)

| Agent | Model | Reason |
|---|---|---|
| LoanOrchestratorAgent | Frontier (GPT-4 or Claude Opus) | Reasoning over the orchestration logic |
| IntakeAgent | Mid-tier (Claude Haiku / GPT-4-mini) | Form parsing; cost-sensitive |
| IdentityVerificationAgent | SLM + OCR specialist | Document understanding; high volume |
| CreditAssessmentAgent | Mid-tier + deterministic logic | Most logic is rule-based; LLM for narrative |
| FraudCheckAgent | PEFT-tuned 7B + classical ML | Pattern recognition; cost-sensitive |
| UnderwritingAgent | Frontier with verifier loop | Highest stakes; verifier catches errors |
| NotificationAgent | Mid-tier | Generation, not reasoning |

Cost is dominated by LoanOrchestratorAgent + UnderwritingAgent; SLMs and specialists handle volume.

### 3. Knowledge + skill layering (chapter 120 in practice)

**RAG layer** indexes:
- Lending policies (current versioned)
- Historical decisions (anonymised, similarly-situated cases)
- Fraud-pattern library

Each subagent retrieves what it needs; the LoanOrchestratorAgent retrieves policy context.

**PEFT layer** tunes:
- FraudCheckAgent: 7B base + LoRA on fraud-pattern dataset (10K cases).
- UnderwritingAgent: 13B base + LoRA on underwriting decisions (50K decisions, expert-labelled).

Re-trained quarterly; A/B-tested before promotion.

### 4. Coordination pattern (chapter 121)

Hierarchical with clear boundaries:
- LoanOrchestrator decides delegation order; specialists work in isolation.
- Specialists return structured outputs (Pydantic schemas, constrained decoding per [112-constrained-decoding](112-constrained-decoding.md)).
- LoanOrchestrator aggregates and produces final decision.

No specialist-to-specialist communication; no peer-mesh. Reasons: accountability, debuggability, regulatory clarity.

### 5. Explainability + compliance (chapter 122)

Per loan application:
- **Trace** every step (what subagent ran, with what input, what output, what model, what cost).
- **Decision record** for the final approval/denial: structured (option, rationale, evidence, policies-applied, signature).
- **Audit log**: immutable, signed, retained per regulatory regime.
- **Right-to-explanation surface**: applicant sees a sanitised explanation; can request human review.
- **Regulatory map**: documents which agent step satisfies which clause of which regulation; maintained by compliance team.

### 6. Robustness + fault tolerance (chapter 123)

- **Circuit breakers** on credit-bureau API, fraud-DB, KYC service. Trip → fall back to graceful degradation (manual review queue).
- **Adaptive retry** on transient errors (rate limits, timeouts).
- **Verifier loop** on UnderwritingAgent decision: a separate verifier model checks for fair-lending pattern violations; on flag → escalate to human.
- **Cost circuit breaker**: per-application max budget; hits → fall back to cheaper path.
- **Loop detection**: orchestrator can't call the same specialist twice with the same input; force-diversify or escalate.
- **Graceful degradation**: full agent fails → manual-review path with operator queue.

### 7. Agent-level production (chapter 124)

- **Cold start**: load PEFT adapters, RAG indexes, policy bundles. Smoke test before serving.
- **Checkpointing**: per-application state saved at every subagent transition. Restart picks up mid-application.
- **Idempotency**: each application has a unique key; resubmission is detected, not duplicated.
- **Dead-letter queue**: applications that fail repeatedly → operator queue with diagnostic context.
- **Multi-tenant isolation**: each bank's data, traces, policies isolated.

### 8. System-level platform (chapter 125)

- **Tenancy**: serves multiple banks from one platform.
- **Quotas**: per-bank LLM budget, per-bank rate limits.
- **Multi-region**: EU bank's data stays in EU.
- **Blue/green deploy**: every release passes eval gate (offline + trajectory); rollback path active.
- **FinOps**: per-bank, per-application, per-subagent cost tracking. Top cost drivers identified, optimised quarterly.
- **Observability backplane**: traces aggregated; per-bank dashboards.

### 9. Framework choice (chapter 126)

For this stack:
- **LangGraph** for the orchestrator and specialist coordination.
- **DSPy** for compiled specialists (fraud-pattern classifier, narrative generators).
- **Custom code** at the regulatory boundary (decision records, audit log, right-to-explanation surfaces) — no framework abstraction here; these need full control.

### 10. Maturity progression (chapter 118)

- **Year 1**: Stage 1–2. Pilot with one bank, HITL on every decision, internal eval.
- **Year 2**: Stage 3. Production with three banks, automated decisions under threshold, HITL above. Platform team established.
- **Year 3**: Stage 4. Vendor-neutral abstractions, golden-path templates for new banks. Eval-gated deploys.
- **Year 4+**: Stage 5 explicitly *not* attempted. Regulatory requirements mandate human oversight on high-value decisions; full agentic operation is not the goal.

### 11. Failure scenarios

- **Credit bureau outage**: circuit breaker trips; fallback to manual queue. Customers see "decision pending; we'll reach out within 24 hours" via the right-to-explanation surface.
- **PEFT model regresses**: post-quarterly-retrain, the new fraud-tuner is worse on a subset. Eval gate catches it; release blocked; team investigates dataset drift.
- **Compliance update**: new fair-lending rule; regulatory map updated; verifier loop's flag list expanded; deploy.
- **Cost spike**: one bank's volume triples unexpectedly; FinOps alert fires; quotas reduced or capacity added.
- **Multi-tenancy bug**: bank A's traces leak into bank B's dashboard. Severe incident. Rollback, post-mortem, isolation hardening.

## Empirical anchors

- **Hierarchical orchestrator-specialist** is the dominant pattern in regulated production agents in 2026.
- **PEFT** typically lifts fraud-detection 5–15% over prompt-based; the labelled dataset is the binding asset.
- **Verifier loops** catch ~10% of decisions that would otherwise be wrong on regulatory grounds.
- **Decision records** are the artifact regulators inspect; teams that ship without them fail compliance audits.
- **Cost dominated by orchestrator + underwriter**: usually 70–80% of total LLM cost.
- **Explainability infrastructure** typically takes 30% of initial build effort; pays back in incident response time.

## Variants and counter-arguments addressed

- **"Why not single-agent?"** Loan processing has natural decomposition (intake / KYC / credit / fraud / underwriting); single-agent LLM can't reliably hold all of those in one prompt.
- **"Why hierarchical, not blackboard?"** Accountability and regulatory clarity require a single decision-maker per application; blackboard's implicit coordination is harder to audit.
- **"Why not full automation?"** Regulation mandates human oversight on high-value decisions; the system is automated under threshold, escalated above.
- **"Couldn't this be a workflow?"** It is, mostly. The agentic regions are intake (input parsing) and underwriting (judgement). The rest is workflow with LLM steps.
- **"Why so much compliance overhead?"** Because the cost of skipping it is regulatory enforcement action and brand damage. The overhead is insurance.

## Failure modes and limitations

1. **Eval set bias.** Underwriting eval set drawn from past decisions inherits past biases; fair-lending audit requires explicitly de-biased eval.
2. **PEFT-set drift.** Fraud patterns shift; quarterly retraining is the minimum cadence.
3. **Verifier mis-calibration.** Verifier rejects valid decisions or accepts invalid ones; periodic human-labelled re-calibration is essential.
4. **Right-to-explanation theatre.** Generic explanations that don't reflect the actual decision logic fail GDPR.
5. **Multi-bank prompt drift.** Each bank wants its own underwriting tone; multi-LoRA serving keeps prompts isolated.
6. **Audit log retention cost.** At scale, regulatory retention windows × volume = significant storage. Tier (hot vs warm vs cold) and budget.
7. **Cross-region data flow.** Even metadata can violate data residency; design for it from day one.
8. **Stage-5 temptation.** Pressure to remove human oversight for cost reduction; regulatory cost of doing so is high.

## When to use, when not

**Use this case study as a reference** when designing any regulated multi-step agentic system: KYC/AML, claims processing, medical-decision support, hiring, mortgage processing, insurance underwriting, fraud investigation.

**Adapt it for** non-regulated multi-step systems by relaxing compliance requirements (decision records, audit logs) while keeping the architecture (hierarchical, multi-stage, verified).

**Don't read it as a recipe**: every system has its own constraints, but the pattern composition is invariant.

## Implications for harness engineering

- **The patterns compose.** Chapters 118–126 are interlocking, not independent. Build the harness with all of them in mind, not one at a time.
- **Compliance shapes architecture.** Decision records, audit logs, right-to-explanation surfaces are not afterthoughts; they're load-bearing.
- **Verifier loops are the safety net.** [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) — every high-stakes decision should pass through one.
- **Per-subagent eval.** Each specialist has its own metrics; aggregate metrics hide subagent regressions. See [115-evaluating-llm-systems](115-evaluating-llm-systems.md).
- **Multi-tenancy from day one** when serving regulated customers.
- **Stage progression deliberately.** [118-genai-maturity-models](118-genai-maturity-models.md). Don't ship stage-4 ambitions on stage-2 capability.
- **Document the coordination contract.** Every subagent's input/output is a Pydantic schema; constrained decoding ([112-constrained-decoding](112-constrained-decoding.md)) at every boundary.
- **Plan for the regulatory audit.** A full audit drill annually; if the system can't survive it, fix the system, not the audit.

The one-sentence takeaway: **a regulated multi-agent system is the patterns of chapters 118–126 *integrated*, not optional features added on; loan processing is the canonical worked example because every pattern earns its place at once.**

## See also

- [02-subagent-delegation](02-subagent-delegation.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md) — hierarchical pattern in detail.
- [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) — the verifier-loop component.
- [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md) — adversarial scenarios any production system must consider.
- [120-rag-to-finetuning-spectrum](120-rag-to-finetuning-spectrum.md) — RAG+PEFT layering.
- [122-explainability-compliance](122-explainability-compliance.md), [123-robustness-fault-tolerance](123-robustness-fault-tolerance.md), [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md) — the pattern catalog this case study composes.
- [126-frameworks-comparison](126-frameworks-comparison.md) — framework choice.
- [148-beginner-onramp-what-is-agentic-ai](148-beginner-onramp-what-is-agentic-ai.md), [149-sector-use-case-catalog](149-sector-use-case-catalog.md) — sector-level framing.
