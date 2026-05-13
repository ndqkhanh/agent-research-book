# 149 — Sector Use-Case Catalog: Health, Finance, Legal, Customer Support, Marketing — One Pattern Per Sector with the Constraints That Drive Design

**Sources.** Baker, *Agentic AI For Dummies*, Chapter 6 (Sector Use Cases); Rothman, *Building Business-Ready Generative AI Systems*, Chapter 6 (E-Marketing); Hodjat, *The Agentic Enterprise*, Chapter 3 (High-Impact Enterprise Applications); plus Nagasubramanian's domain examples; the practitioner literature on regulated-industry AI deployment.

**One-line definition.** Each sector imposes specific constraints — regulatory regimes, data sensitivity, stakeholder structure, error tolerance, audit requirements — that shape *which* agentic patterns work and *how* they need to be implemented; this chapter catalogues the dominant pattern per major sector (health, finance, legal, customer support, marketing, manufacturing, education, public sector) so engineers entering a sector know the standard architecture, the regulatory landscape, and the design constraints upfront.

## Why this matters

Agentic AI patterns generalise; sector constraints don't. A research agent built for healthcare must respect HIPAA, audit trails, clinical-decision-support regulations, drug-interaction checks. A research agent built for finance must respect SEC, fair-lending, KYC/AML, recordkeeping. The architecture is similar; the constraints diverge.

For agent builders entering a new sector, knowing the constraints upfront saves quarters. Building an agent and *then* discovering HIPAA requires a redesign is expensive. Knowing on day one shapes the architecture.

This chapter is the cross-sector cheat sheet: what the dominant agentic pattern looks like in each sector, the regulatory regime, the data-sensitivity profile, and the cross-cutting design constraints.

## Problem it solves

Five sector-misalignment failures:

1. **Wrong architecture.** Build a low-stakes-style agent for a sector that requires audit trails.
2. **Compliance afterthought.** Add HIPAA / SOX / fair-lending after the build; expensive redesign.
3. **Wrong stakeholder model.** Assume agent talks to end-user; sector requires agent talks to professional who talks to end-user.
4. **Error tolerance mismatch.** Build a 95%-accurate agent for a sector that requires 99.9%.
5. **Data-residency miss.** Build cloud-only; sector requires on-prem or sovereign cloud.

## Core idea in one paragraph

Each sector has a *dominant pattern* — the architecture that recurs across successful deployments — driven by regulatory and structural constraints. **Healthcare**: clinical-decision-support style; LLM proposes, clinician approves; HIPAA + clinical safety. **Finance**: high-stakes decision automation under threshold, HITL above; SEC / fair-lending / AML / SOX. **Legal**: research-and-draft pattern; agent surfaces evidence, attorney decides; privilege + jurisdictional rules. **Customer support**: tier-1 deflection + tier-2 human escalation; CSAT + privacy. **Marketing**: hybrid (ML personalisation + LLM generation); brand voice + privacy + measurement. **Manufacturing**: simulation + agent reasoning for what-if; OT integration + safety. **Education**: tutoring + assessment generation; pedagogical safety + COPPA. **Public sector**: policy compliance + auditability; explainability + procurement constraints. Knowing the dominant pattern speeds time-to-shippable; knowing the constraints prevents redesign.

## Sector breakdown

### Healthcare

**Dominant pattern**: Clinical decision support. Agent surfaces relevant evidence (literature, patient history, guidelines), drafts a recommendation; clinician reviews, decides.

**Regulations**: HIPAA (privacy), HITECH (breach notification), FDA (clinical-decision-support classification), state-specific telehealth, GDPR for EU patients.

**Constraints**:
- PHI never in prompts to non-BAA-covered services.
- Audit log of every clinician interaction.
- Bias review on training data.
- Explainability (clinician must understand why).
- High accuracy bar (errors → patient harm).

**Common applications**: clinical-decision support, radiology second-read, drug-interaction checking, ICD coding, prior-auth automation, medical record summarisation.

**Cross-references**: [122-explainability-compliance](122-explainability-compliance.md), [127-loan-processing-multi-agent-case-study](127-loan-processing-multi-agent-case-study.md) (similar regulatory architecture), [28-radagent-agentic-radiology](28-radagent-agentic-radiology.md).

### Finance

**Dominant pattern**: Threshold-based automation. Agent decides under a value/risk threshold; HITL above. Always with explainability and audit.

**Regulations**: SEC (advisor rules), Reg BI, FINRA, fair-lending (ECOA / Fair Housing), AML / KYC (BSA, FinCEN), SOX (recordkeeping), GDPR / CCPA, sector-specific (insurance, mortgage, credit unions).

**Constraints**:
- Explainability for adverse decisions.
- Audit trail per decision.
- Bias / disparate-impact review.
- AML / fraud screening.
- Records retention (7 years typical).
- Data residency for cross-border.

**Common applications**: loan processing ([127-loan-processing-multi-agent-case-study](127-loan-processing-multi-agent-case-study.md)), fraud detection, AML screening, claims processing, customer onboarding (KYC), trading-decision support, risk reporting.

**Cross-references**: [122-explainability-compliance](122-explainability-compliance.md), [123-robustness-fault-tolerance](123-robustness-fault-tolerance.md).

### Legal

**Dominant pattern**: Research-and-draft. Agent does research, surfaces evidence, drafts initial work; attorney reviews and decides.

**Regulations**: Attorney-client privilege, ABA model rules, jurisdictional bar rules, professional-responsibility, conflict-of-interest, e-discovery rules (FRCP).

**Constraints**:
- Privilege preservation (no AI training on privileged content).
- Hallucination is malpractice (citations must be real and accurate).
- Jurisdictional accuracy (state vs federal vs international).
- Conflict screening.
- Discoverability of agent traces.

**Common applications**: legal research, contract review, e-discovery, drafting, due diligence, brief writing.

**Cross-references**: [135-trustworthy-generation](135-trustworthy-generation.md), [122-explainability-compliance](122-explainability-compliance.md).

### Customer support

**Dominant pattern**: Tier-1 deflection. Agent handles common queries; escalates complex / emotional / high-stakes to humans.

**Regulations**: Consumer-protection, recording-consent, GDPR / CCPA, sector-specific (healthcare = HIPAA, finance = SEC).

**Constraints**:
- Quality of service (response time, satisfaction).
- Escalation paths must be obvious.
- Persistent context across channels (chat, voice, email).
- Sentiment-aware (don't send a chipper agent to a frustrated customer).
- Brand voice consistency.

**Common applications**: customer-service deflection, ticket routing, knowledge-base Q&A, conversational commerce, churn rescue.

**Cross-references**: [137-voice-agents](137-voice-agents.md), [141-e-commerce-marketing-agents](141-e-commerce-marketing-agents.md), [143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md).

### Marketing

**Dominant pattern**: Hybrid ML + LLM. Classical for personalisation; LLM for generation and reasoning.

**Regulations**: Advertising standards, consumer protection, CAN-SPAM, TCPA (telephony), GDPR / CCPA, sector-specific (financial advertising, healthcare claims, COPPA).

**Constraints**:
- Brand voice consistency.
- Personalisation respects consent.
- Truth-in-advertising; LLM hallucination is liability.
- A/B testing for measurement.

**Common applications**: campaign generation, personalised email, ad copy variation, content marketing, customer-journey orchestration, churn rescue.

**Cross-references**: [141-e-commerce-marketing-agents](141-e-commerce-marketing-agents.md), [115-evaluating-llm-systems](115-evaluating-llm-systems.md).

### Manufacturing / industrial

**Dominant pattern**: Simulation-augmented operations. Agent reasons over simulator outputs to recommend interventions; operator approves.

**Regulations**: Industry-specific safety (FDA for med devices, ISO for general), OSHA, environmental, sector-specific.

**Constraints**:
- OT (operational technology) integration; air-gapped systems.
- Safety-critical: errors can hurt people or damage equipment.
- Latency: real-time control loops.
- Determinism in critical paths.

**Common applications**: predictive maintenance, supply-chain optimisation, quality inspection (CV + LLM), production scheduling, what-if scenario planning.

**Cross-references**: [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md), [140-traditional-ml-genai-hybrid](140-traditional-ml-genai-hybrid.md).

### Education

**Dominant pattern**: Tutor + assessment. Agent provides personalised tutoring; assessments generated and validated.

**Regulations**: COPPA (children under 13), FERPA (educational records), state-specific student data privacy.

**Constraints**:
- Pedagogical safety (no harmful, age-inappropriate, or biased content).
- Assessment integrity (anti-cheating).
- Personalisation respects developmental level.
- Long-context (track student progress).

**Common applications**: 1:1 tutoring, assignment grading, content generation, study plans, language learning.

**Cross-references**: [107-memento-cbr-memory](107-memento-cbr-memory.md) (long-term memory of student progress), [115-evaluating-llm-systems](115-evaluating-llm-systems.md).

### Public sector

**Dominant pattern**: Compliance-first decision support. Agent recommends; civil servant decides; everything auditable.

**Regulations**: Administrative procedure, public records, FOIA, equal-protection, sector-specific (immigration, tax, benefits).

**Constraints**:
- Procurement: vendors must clear extensive review.
- Citizens' rights (right to explanation, right to appeal).
- Anti-discrimination.
- Public-records retention.
- Open-source / on-prem preferences.

**Common applications**: benefits processing, document analysis, citizen Q&A, compliance review, infrastructure planning.

**Cross-references**: [122-explainability-compliance](122-explainability-compliance.md), [147-vendor-lock-in](147-vendor-lock-in.md).

### HR / talent

**Dominant pattern**: Sourcing + screening + scheduling. Agent surfaces candidates; humans interview and decide.

**Regulations**: EEOC (US), GDPR (EU), state-specific anti-discrimination, NYC AEDT (automated employment decision tools).

**Constraints**:
- Bias auditing required by law in some jurisdictions.
- Explainability for adverse decisions.
- Candidate data privacy.

**Common applications**: resume screening (with bias audit), candidate matching, scheduling automation, interview-prep tools.

**Cross-references**: [122-explainability-compliance](122-explainability-compliance.md).

## Cross-cutting design constraints by sector

| Constraint | Health | Finance | Legal | CS | Marketing | Mfg | Edu | Public |
|---|---|---|---|---|---|---|---|---|
| Audit trail | High | High | High | Med | Low-Med | Med-High | Med | High |
| Explainability | High | High | High | Low-Med | Low | Med | Med | High |
| HITL required | Often | Threshold | Always | Escalation | Rarely | Often | Often | Often |
| Data residency | Strict | Strict | Strict | Med | Med | Strict | Strict | Strict |
| Error tolerance | Very low | Very low | Very low | Med | Med-High | Very low | Med | Low |
| Privacy | PHI | PII+Fin | Privilege | PII | PII | OT | Student | Citizen |
| Vendor screening | High | High | High | Med | Med | Med | High | High |

This is the cheat sheet for entering a new sector.

## When to use, when not

**Use this catalog as a starting point** when entering a new sector; deepen with sector-specific consultation.

**Skip cross-sector generalisations.** Each sector has gotchas; the catalog is the map, not the territory.

**Match your stack to the dominant pattern.** Building a manufacturing agent like a customer-support agent is wasted effort.

## Implications for harness engineering

- **Sector-aware architecture.** Match patterns to sector constraints from day one.
- **Compliance from day one.** [122-explainability-compliance](122-explainability-compliance.md). Retrofit is expensive.
- **HITL pattern matched to risk.** [23-human-in-the-loop](23-human-in-the-loop.md). High-stakes sectors → always HITL above threshold.
- **Audit trail granularity matched to sector.** [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md).
- **Bias audits where required by law.** Especially HR, finance, public sector.
- **Domain expert in the loop.** Especially during build and eval.
- **Sector-tuned eval sets.** Generic eval doesn't capture domain edge cases.
- **Vendor screening matched to sector.** [147-vendor-lock-in](147-vendor-lock-in.md).

The one-sentence takeaway: **each sector has a dominant agentic pattern shaped by regulatory and structural constraints — match the pattern to the sector before building, because retrofitting compliance is the most expensive mistake in agentic-AI projects.**

## See also

- [113-from-tokens-to-agents-onramp](113-from-tokens-to-agents-onramp.md), [148-beginner-onramp-what-is-agentic-ai](148-beginner-onramp-what-is-agentic-ai.md) — beginner framing.
- [118-genai-maturity-models](118-genai-maturity-models.md) — how sector + maturity interact.
- [122-explainability-compliance](122-explainability-compliance.md), [123-robustness-fault-tolerance](123-robustness-fault-tolerance.md) — cross-sector compliance and reliability patterns.
- [127-loan-processing-multi-agent-case-study](127-loan-processing-multi-agent-case-study.md) — finance worked example.
- [137-voice-agents](137-voice-agents.md), [141-e-commerce-marketing-agents](141-e-commerce-marketing-agents.md), [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md) — sector-specific patterns.
- [28-radagent-agentic-radiology](28-radagent-agentic-radiology.md), [30-gpt-rosalind-domain-specialized](30-gpt-rosalind-domain-specialized.md), [33-dnahnet-genomic-foundation](33-dnahnet-genomic-foundation.md) — domain-specialised existing chapters.
- [146-business-case-roi](146-business-case-roi.md) — sector-specific ROI patterns.
