# 272 — Agent Compliance and Audit: EU AI Act, SOC2, GDPR, HIPAA adapted to production agents

**Anchors.** EU AI Act — *Regulation (EU) 2024/1689* — entered force 1 Aug 2024; major obligations on **General-Purpose AI** (GPAI) effective 2 Aug 2025; **high-risk system** obligations effective 2 Aug 2026. SOC 2 Type II — AICPA Trust Services Criteria adapted for AI systems by 2025–2026 audit standards. GDPR — Regulation (EU) 2016/679. HIPAA — 45 CFR Parts 160 and 164. NIST AI Risk Management Framework — *AI RMF 1.0* (2023) + Generative AI Profile (2024). ISO/IEC 42001 — *AI Management System* (2023). Companions: [122-explainability-compliance](122-explainability-compliance.md), [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md), [148-beginner-onramp-what-is-agentic-ai](148-beginner-onramp-what-is-agentic-ai.md), [149-sector-use-case-catalog](149-sector-use-case-catalog.md), [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md), [267-agent-sre](267-agent-sre.md), [269-prompt-injection-2026](269-prompt-injection-2026.md), [270-agent-supply-chain-security](270-agent-supply-chain-security.md), [271-agent-isolation-patterns](271-agent-isolation-patterns.md), [273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md).

**One-line definition.** A 2026 picture of **regulatory compliance and audit for production agents** — the **EU AI Act** (highest-impact, 2 Aug 2026 high-risk obligations; GPAI obligations live since Aug 2025; defines four risk tiers including **prohibited / high-risk / limited-risk / minimal-risk**; agents in HR, lending, healthcare, education, infrastructure, justice, migration are typically high-risk) plus **classical compliance frameworks** adapted to agents — **SOC 2 Type II** (now requires AI-specific controls including model-version tracking, prompt-injection defense evidence, eval pipelines), **GDPR** (memory tier == personal data; right-to-erasure cascades to memory + traces + skill caches), **HIPAA** (PHI handling in healthcare agents requires BAA, audit logs, encryption, access controls), **NIST AI RMF** (govern + map + measure + manage cycle as the operational frame), **ISO/IEC 42001** (AI Management System certification, gaining adoption as the agent-platform certification standard); each requires **provenance + audit trail + access control + retention policy + retraction-propagation** — exactly the primitives observability ([264](264-agent-observability-stack-2026.md)) + supply-chain security ([270](270-agent-supply-chain-security.md)) + isolation ([271](271-agent-isolation-patterns.md)) provide, but with documentation, certification, and ongoing evidence collection that make compliance auditable rather than aspirational.

## Why this paper matters (compliance is now a deployment gate, not an afterthought)

Through 2024 the regulatory landscape for AI was speculative — drafts, proposals, voluntary frameworks. By 2026 it is binding: the **EU AI Act** prohibitions are in force (2 Feb 2025); **GPAI obligations** entered force 2 Aug 2025 (transparency, copyright compliance, technical documentation, incident reporting); **high-risk system obligations** begin 2 Aug 2026 (risk management, data governance, technical documentation, transparency, human oversight, accuracy/robustness/cybersecurity, post-market monitoring). SOC 2 auditors now ask AI-specific questions on model-version tracking, prompt-injection defenses, eval evidence. GDPR enforcement actions reference agent memory as personal data. HIPAA covered entities deploying healthcare agents require BAA + encryption + audit. **Compliance moved from "nice to have" to "deployment gate"** for the agent classes most subject to it: enterprise SaaS, healthcare, finance, HR, education, government.

The good news: **compliance primitives map cleanly to the engineering primitives** the rest of this corpus has covered. **Observability** ([264](264-agent-observability-stack-2026.md)) is your audit trail. **Eval pipelines** ([265](265-agent-evaluation-2026.md)) are your accuracy + robustness evidence. **Durability** ([266](266-agent-durability-and-idempotency.md)) is your incident-response tooling. **SRE** ([267](267-agent-sre.md)) is your post-market monitoring. **Supply-chain security** ([270](270-agent-supply-chain-security.md)) is your data-governance + technical-documentation. **Isolation** ([271](271-agent-isolation-patterns.md)) is your security control. **Permission bridge** ([07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md)) is your access control. **Bright-line gates** ([14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md)) are your human-oversight points. The compliance work in 2026 is **mapping engineering primitives to regulatory requirements + collecting evidence**, not building new infrastructure from scratch.

The bad news: **evidence collection is non-trivial**, ongoing, and audit-driven. SOC 2 Type II requires observation periods of 6+ months. EU AI Act requires technical documentation maintained throughout the lifecycle. GDPR requires impact assessments + DPIAs. HIPAA requires audit logs retained for years with tamper-evident storage. The discipline is **continuous evidence collection + periodic external audit + ongoing remediation**, mirroring SOC 2 / ISO discipline from classical software but adapted to the agent-specific surfaces.

Take this seriously and three things change. **First**, you map your agent platform to the **applicable regulatory framework(s)** explicitly — EU AI Act risk tier, SOC 2 audit scope, GDPR processor / controller role, HIPAA covered-entity status. **Second**, you build **continuous evidence collection** into the platform — every observability span, every eval result, every supply-chain SBOM update, every isolation policy is logged with retention policies appropriate to the regulatory regime. **Third**, you treat **compliance as engineering** — not as a separate audit-team function — with compliance dashboards alongside SLO dashboards, compliance regression tests in CI, and compliance-aware bright-line gates.

## Problem it solves (regulatory compliance for production agents in the post-EU-AI-Act era)

1. **No standard regulatory framework existed pre-2024.** EU AI Act + NIST AI RMF + ISO 42001 fill this; field convergence in progress.
2. **SOC 2 didn't have AI controls.** AICPA published AI-system extensions 2024-2026.
3. **GDPR application to agent memory was unclear.** Regulator guidance + enforcement actions clarified through 2024-2025.
4. **HIPAA + AI agents** required BAA-shape contracts, encryption, audit; enterprise patterns emerged.
5. **Audit-trail tampering risk.** Cryptographic tamper-evident logs (hash chains, Sigstore) became standard.
6. **Right-to-erasure cascades** to memory, traces, skill caches — engineering work to implement.
7. **High-risk classification ambiguity.** EU AI Act Annex III lists high-risk use cases; mapping your agent is a legal call.
8. **Cross-border data transfers.** GDPR + Schrems II implications for memory storage location.
9. **GPAI provider vs deployer distinction.** EU AI Act splits obligations; vendor relationships clarified.
10. **Continuous compliance.** One-time audits insufficient; continuous monitoring + evidence required.

## Core idea in one paragraph

Agent compliance in 2026 is a **mapping exercise** between **regulatory frameworks** and **engineering primitives**. The regulatory side: **EU AI Act** classifies your agent into one of four tiers (prohibited / high-risk / limited / minimal) with progressively stronger obligations; **SOC 2 Type II** with AI extensions audits internal controls over a 6-12 month observation period; **GDPR** governs personal data including agent memory and traces; **HIPAA** governs PHI in healthcare agents; **NIST AI RMF** provides the govern-map-measure-manage operational frame; **ISO/IEC 42001** certifies AI management systems. The engineering side maps to **eight primitives**: (1) **risk classification** — written analysis of which regulatory regimes apply; (2) **technical documentation** — architecture diagrams, model cards, eval reports, SBOM; (3) **audit trail** — observability ([264](264-agent-observability-stack-2026.md)) with tamper-evident storage and regulatory-aligned retention; (4) **access control** — permission bridge ([07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md)) with audit logs; (5) **human oversight** — bright-line gates ([14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md)) on consequential actions; (6) **data governance** — provenance + supply-chain ([270](270-agent-supply-chain-security.md)) + retraction-propagation; (7) **post-market monitoring** — SRE ([267](267-agent-sre.md)) + drift detection + incident reporting; (8) **right-to-erasure** — memory + trace + skill-cache deletion that cascades. The discipline is **continuous evidence collection** with retention policies aligned to the regulatory regime (typically 5-7 years for HIPAA + EU AI Act high-risk), **periodic external audit** (SOC 2 Type II annually, ISO 42001 every 3 years), and **ongoing remediation** of findings. Compliance dashboards alongside SLO dashboards; compliance regression tests in CI; compliance-aware bright-line gates. The 2026 production agent platform that ships compliance-by-construction is the one that wins regulated-industry deployments.

## Mechanism (step by step)

### (a) EU AI Act — risk tier classification

Map your agent to the four tiers per Annex III:

| Tier | Examples | Obligations |
|---|---|---|
| **Prohibited** (Art. 5) | Social scoring, predictive policing, biometric mass surveillance | Cannot deploy |
| **High-risk** (Art. 6 + Annex III) | HR (recruiting, promotion), credit scoring, healthcare diagnostics, education assessment, critical infrastructure, law enforcement, justice, migration, biometric ID | Full obligations: risk management, data governance, technical documentation, logging, transparency, human oversight, accuracy/robustness/cybersecurity, post-market monitoring, conformity assessment, registration |
| **Limited risk** | Chatbots, deepfakes, emotion recognition | Transparency obligations (disclose AI use) |
| **Minimal risk** | Most other AI | Voluntary code of conduct |

**GPAI providers** (e.g., Anthropic, OpenAI shipping foundation models) have separate obligations: technical documentation, copyright compliance, transparency about training data summary, incident reporting.

**GPAI with systemic risk** (>10^25 FLOPs training compute or specific designation) has additional obligations: model evaluations, adversarial testing, serious incident reporting, cybersecurity protections.

### (b) Technical documentation requirements (high-risk)

Per Article 11 + Annex IV, maintain throughout lifecycle:

- General description (intended purpose, deployer's instructions)
- Detailed system description (architecture, components, design choices)
- Detailed development information (datasets, training methods, validation procedures)
- Risk management system (Article 9 — identify, analyse, evaluate, mitigate)
- Data governance plan (Article 10 — quality criteria, bias detection, gap-filling)
- Logging capabilities (Article 12 — automatic record-keeping, traceability)
- Transparency to users (Article 13)
- Human oversight measures (Article 14)
- Accuracy / robustness / cybersecurity (Article 15)
- Quality management system (Article 17)
- Conformity assessment (Article 43 — internal or third-party)
- Post-market monitoring plan (Article 72)

### (c) Audit-trail requirements

| Regulation | Retention | Tamper-evidence | Granularity |
|---|---|---|---|
| EU AI Act high-risk | Minimum throughout deployment | Required (Art. 12) | Each interaction |
| SOC 2 Type II | 6-12 month observation; multi-year retention | Required | Per-control |
| GDPR | Until data deleted (right to erasure) | Recommended | Per-personal-data-touchpoint |
| HIPAA | 6 years minimum | Required (45 CFR § 164.530) | Each PHI access |
| Financial (SEC, FINRA) | 3-7 years depending | Required | Each transaction |

Implementation: HIR observability ([264](264-agent-observability-stack-2026.md)) with **append-only storage** (S3 with object lock, Sigstore transparency log, hash-chained event log).

### (d) GDPR for agent memory

Memory tiers ([233-memory-scaling-for-agents](233-memory-scaling-for-agents.md)) often contain personal data:

| Tier | GDPR class | Obligations |
|---|---|---|
| Working (context) | Personal data if user input | Lawful basis, transparency |
| Episodic | Personal data | Data minimization, retention limit |
| Semantic / facts | Often special-category if profile data | Article 9 lawful basis, DPIA |
| Procedural / skills | Usually not | Standard |

**Right to erasure (Art. 17)** cascades:

```python
async def gdpr_erase(user_id: str):
    # Memory tiers
    await memory.delete_user(user_id)
    # Traces
    await observability.delete_user_traces(user_id)
    # Eval datasets
    await eval_db.delete_user_cases(user_id)
    # Skill caches
    await skill_cache.delete_user(user_id)
    # Cross-region replicas
    await sync.propagate_deletion(user_id)
    # Confirmation
    return ErasureCertificate(user_id, completed_at=now())
```

### (e) HIPAA for healthcare agents

Required controls per 45 CFR Parts 160 + 164:

- **BAA** (Business Associate Agreement) with model providers if PHI touches their infrastructure
- **Encryption at rest + in transit** (FIPS 140-2 / 140-3 validated)
- **Access controls** (role-based, audit-logged)
- **Audit logs** (who accessed what PHI, when, why)
- **Breach notification** (within 60 days)
- **Workforce training**
- **Risk analysis** (formal HIPAA SRA)

**Anthropic, OpenAI, Google, Microsoft** all offer HIPAA-eligible enterprise tiers as of 2025-2026; verify BAA availability.

### (f) SOC 2 Type II AI extensions

AICPA's 2025 audit guidance adds AI-specific controls:

- **Model lifecycle management** — version tracking, deprecation policy
- **Prompt-injection defense** — evidence of [269-prompt-injection-2026](269-prompt-injection-2026.md) controls
- **Eval pipelines** — accuracy + robustness measurements
- **Supply-chain security** — SBOM + signature verification
- **Drift detection** — production-eval feedback
- **Incident response** — runbooks executed + postmortems
- **Vendor risk management** — model providers under BAA / DPA

### (g) NIST AI RMF — operational frame

Four functions, applied continuously:

- **Govern** — policies, accountabilities, risk tolerance
- **Map** — context, intended use, stakeholders, risks
- **Measure** — quantitative + qualitative risk assessment
- **Manage** — risk treatment, monitoring, response

Maps to the engineering primitives:

- Govern → permission bridge + bright-line gates + escalation policy
- Map → risk classification + threat model
- Measure → eval pipelines + observability + SRE SLOs
- Manage → SRE runbooks + supply-chain revocation + incident response

### (h) ISO/IEC 42001 certification

AI Management System standard analogous to ISO 27001 for security. Adoption increasing through 2026 as the agent-platform certification standard. Requires:

- Documented AI policy
- Risk assessment + treatment
- AI lifecycle management
- Performance monitoring
- Continual improvement
- Internal + external audit

### (i) Compliance-as-code

```python
# harness_core/compliance/checks.py
class ComplianceCheck(ABC):
    @abstractmethod
    async def evaluate(self) -> ComplianceResult: ...

class ProvenanceCompleteness(ComplianceCheck):
    """Every span has source provenance metadata (EU AI Act Art. 12)."""
    async def evaluate(self):
        spans = await get_recent_spans(window="24h")
        missing = [s for s in spans if not s.has_provenance()]
        return ComplianceResult(passing=len(missing)==0, ...)

class HumanOversightOnHighRisk(ComplianceCheck):
    """Bright-line gates enabled for high-risk action kinds (EU AI Act Art. 14)."""
    ...

# Run in CI + scheduled routine
```

### (j) Compliance dashboard alongside SLO dashboard

```
┌─────────────────────────────────────────┐
│ Compliance Dashboard                    │
├─────────────────────────────────────────┤
│ EU AI Act Art. 12 logging:    ✅ 100%  │
│ Art. 14 human oversight:      ✅ 100%  │
│ Art. 15 cyber/accuracy:       ⚠ 92%   │
│                                         │
│ SOC 2 Type II controls:       ✅ 47/47 │
│ GDPR Art. 17 erasure SLA:     ✅ <72h  │
│ HIPAA audit log retention:    ✅ 6yr   │
│                                         │
│ Last external audit: Q1 2026 (clean)   │
│ Next: Q1 2027                          │
└─────────────────────────────────────────┘
```

## Empirical results (table — May 2026 regulatory state)

| Regulation | Status | Effective | Affects |
|---|---|---|---|
| EU AI Act prohibitions | In force | Feb 2 2025 | Banned use cases |
| EU AI Act GPAI | In force | Aug 2 2025 | Foundation-model providers |
| EU AI Act high-risk obligations | Effective | Aug 2 2026 | High-risk system providers + deployers |
| EU AI Act fines | Up to 7% global revenue or €35M | Live | Non-compliance |
| SOC 2 Type II AI ext | Standard practice | Live | Enterprise SaaS |
| ISO/IEC 42001 | Adoption growing | Live | Voluntary certification |
| NIST AI RMF | Operational frame | Live | US federal + voluntary |
| US AI Executive Order | EO 14110 / 14179 | Live (varies by admin) | US deployments |
| UK AI Bill | Drafted | TBD | UK deployments |
| China Generative AI rules | In force | Aug 2023 | China deployments |

## Variants and ablations

- **Compliance-by-construction.** Engineering primitives map directly to controls; new primitives auto-comply.
- **Compliance-as-code in CI.** Regression tests for compliance.
- **Cross-region data residency.** GDPR + Schrems II; data stored in EU only.
- **BAA-only model providers.** Healthcare deployments; verify BAA.
- **Conformity assessment internal vs third-party.** EU AI Act allows both for many high-risk; third-party for biometric ID.
- **Compliance dashboards.** Real-time view of control status.
- **Vendor-tier compliance metadata.** AGNTCY trust attributes include compliance status.
- **Audit trail in tamper-evident storage.** S3 Object Lock, Sigstore transparency log.
- **Prompt + response retention policies.** Per-user retention; right-to-erasure cascades.

## Failure modes and limitations

- **Compliance theater.** Going through motions without substance.
- **Evidence collection lag.** Spans-of-record collection has gaps.
- **Cross-border data transfer issues.** Schrems II implications for US-cloud-stored memory.
- **GPAI vs deployer ambiguity.** Who's responsible for what in the value chain.
- **Audit fatigue.** Multi-framework audits (SOC 2 + ISO 42001 + EU AI Act) compounding burden.
- **Right-to-erasure incompleteness.** Distributed traces hard to fully erase.
- **Model version churn.** Vendor model updates; compliance evidence per-version.
- **Risk classification mistakes.** Misclassifying high-risk as limited-risk = enforcement action.
- **Vendor BAA changes.** Model providers update terms; healthcare deployers must re-validate.
- **Cross-jurisdiction conflicts.** EU vs US vs China vs UK divergence.

## When to use, when not

**Adopt full compliance discipline** for any deployment in regulated industries (healthcare, finance, HR, education, government, critical infrastructure); for any deployment in the EU subject to AI Act high-risk classification; for any enterprise SaaS product (SOC 2 Type II is increasingly required for enterprise sales). The strongest cases are **regulated-vertical SaaS**, **government deployments**, **healthcare agents**, and **financial agents**.

**Skip detailed compliance** for personal-use prototypes, internal-only research tools, or non-regulated low-risk consumer agents. **Never skip** GDPR-aware right-to-erasure if processing EU personal data — enforcement actions are real.

## Implications for harness engineering

- **`harness_core/compliance/` package.** Compliance checks as code; CI + scheduled routine.
- **Compliance dashboard alongside SLO dashboard.** [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md), [267-agent-sre](267-agent-sre.md).
- **Tamper-evident audit log.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — hash-chained event log to S3 Object Lock or Sigstore.
- **Right-to-erasure cascades.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md) — implement deletion across memory + traces + skill caches.
- **Provenance metadata mandatory.** [269-prompt-injection-2026](269-prompt-injection-2026.md), [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md).
- **Bright-line gates as human-oversight evidence.** [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [23-human-in-the-loop](23-human-in-the-loop.md).
- **SBOM as data-governance evidence.** [270-agent-supply-chain-security](270-agent-supply-chain-security.md).
- **Eval pipeline as accuracy evidence.** [265-agent-evaluation-2026](265-agent-evaluation-2026.md), [115-evaluating-llm-systems](115-evaluating-llm-systems.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md).
- **Isolation as cybersecurity evidence.** [271-agent-isolation-patterns](271-agent-isolation-patterns.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md).
- **SRE runbook execution as post-market monitoring.** [267-agent-sre](267-agent-sre.md).
- **Memory tier classification per GDPR class.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md).
- **Cross-channel verifier as accuracy/robustness evidence.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [97-qwen-prm](97-qwen-prm.md).
- **Vendor BAA tracking.** Per-model-provider compliance status in SBOM.
- **DPIA / risk assessment as routine.** [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md) — annual DPIA review.
- **Cross-border data residency policy.** [253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md) — NATS clusters per region.
- **Per-project compliance scope.** [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md), [208-lyra-multi-hop-collaborative-apply-plan](208-lyra-multi-hop-collaborative-apply-plan.md), [210-mentat-learn-collaborative-apply-plan](210-mentat-learn-collaborative-apply-plan.md), [211-cross-project-power-up-plan-with-tradeoffs](211-cross-project-power-up-plan-with-tradeoffs.md).

**One-line takeaway for harness designers.** **Agent compliance in 2026 is engineering, not legal — the EU AI Act + SOC 2 + GDPR + HIPAA + NIST AI RMF + ISO 42001 obligations map cleanly to engineering primitives (observability + eval + supply-chain security + isolation + permission bridge + bright-line gates), so the work is mapping + continuous evidence collection + periodic audit, not building new infrastructure; treat compliance as engineering discipline, build compliance dashboards alongside SLOs, run compliance regression tests in CI, and ship compliance-by-construction so regulated-industry deployments become a deployment-gate-passed instead of a quarter-long fire drill.**
