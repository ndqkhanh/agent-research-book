# 303 — Sector Verticals Overview: healthcare, finance, legal, education agent specialization

**Anchors.** [149-sector-use-case-catalog](149-sector-use-case-catalog.md), [272-agent-compliance-and-audit](272-agent-compliance-and-audit.md). Sector-specific deep-dives: [127-loan-processing-multi-agent-case-study](127-loan-processing-multi-agent-case-study.md), [28-radagent-agentic-radiology](28-radagent-agentic-radiology.md), [74-kronos-foundation-model-financial-markets](74-kronos-foundation-model-financial-markets.md), [287-helix-bio-seven-layer-stack-apply-plan](287-helix-bio-seven-layer-stack-apply-plan.md).

**One-line definition.** A 2026 picture of **sector-specialized agents** — covering the four largest regulated verticals (healthcare, finance, legal, education) plus three high-volume horizontals (customer service, sales, software) — with **per-sector compliance overlays** (HIPAA + FDA for healthcare, SOX + PCI-DSS + FINRA for finance, attorney-client privilege + state bar rules for legal, FERPA for education), **sector-specific MCP servers + skills**, and **trust-tier-graded verifier ensembles** because regulated-industry agents face the highest scrutiny on accuracy + auditability + bias — staged as a per-sector overlay that drops on top of the seven-layer stack.

## Why sector verticals matter

The 250–296 corpus is sector-agnostic (designed for any production agent). Real production agents are **deployed in specific sectors** with sector-specific compliance, terminology, datasets, and risk profiles. Healthcare agents face HIPAA + FDA; finance agents face SOX + PCI-DSS + FINRA; legal agents face attorney-client privilege + bar rules. Each vertical adds a compliance overlay and sector-specific operational requirements. This file surveys the four largest regulated verticals and three high-volume horizontals.

## Healthcare agents (per [287-helix-bio](287-helix-bio-seven-layer-stack-apply-plan.md))

**Compliance:** HIPAA + FDA QSR + IRB ethics + GDPR (if EU patients) + state-specific privacy laws.

**Sector-specific architecture:**

- **BAA-required model providers** (Anthropic enterprise, OpenAI enterprise, Google Vertex AI HIPAA tier).
- **PHI redactor** at every memory-write + every external-API boundary.
- **Audit log retention 6+ years** ([272](272-agent-compliance-and-audit.md)) in tamper-evident storage.
- **Bright-line gates** on every clinical-decision-support output; human clinician review.
- **Domain-finetuned model** ([287](287-helix-bio-seven-layer-stack-apply-plan.md), [275](275-agent-finetuning-2026.md)) on PubMed + clinical guidelines + UMLS.
- **Specialized MCP servers**: PubMed, UpToDate, Epic / Cerner FHIR, ICD-10 / SNOMED CT.
- **Cross-channel verifier** mandatory on clinical claims; ensemble of 3+ model families.

**Use cases:** clinical-decision support, medical scribing, diagnostic assistance ([28-radagent-agentic-radiology](28-radagent-agentic-radiology.md)), drug discovery research, prior-authorization automation.

## Finance agents

**Compliance:** SOX + PCI-DSS (if cards) + FINRA (if broker-dealer) + KYC/AML + GDPR + EU AI Act high-risk (credit scoring, financial decisions per Annex III).

**Sector-specific architecture:**

- **Tamper-evident audit log** with cryptographic chain-of-custody (Sigstore transparency log).
- **Bright-line gates** on every transaction, position-modify, customer-data action; multi-level approval for high-amount.
- **Cross-channel verifier ensemble** (3+ families) on all consequential outputs.
- **Domain-finetuned** ([74-kronos-foundation-model-financial-markets](74-kronos-foundation-model-financial-markets.md)) on market data + regulations + filings.
- **Specialized MCP servers**: Bloomberg, Refinitiv, internal trading systems (read-only by default), regulatory filings.
- **Real-time anomaly detection** via observability ([264](264-agent-observability-stack-2026.md)).

**Use cases:** loan processing ([127-loan-processing-multi-agent-case-study](127-loan-processing-multi-agent-case-study.md)), trading research, compliance review, customer service in banking, fraud detection.

## Legal agents

**Compliance:** attorney-client privilege + work-product doctrine + state bar rules on AI-assisted practice + GDPR + national jurisdiction-specific rules.

**Sector-specific architecture:**

- **Per-engagement isolated tenancy** — no cross-client data flow ever.
- **Privileged-content classifier** at every output boundary.
- **Citation verifier** mandatory — legal citations are the load-bearing trust signal.
- **Domain-finetuned** on case law + statutes + regulations.
- **Specialized MCP servers**: Westlaw, LexisNexis, Casetext, court filings, regulatory databases.
- **Bright-line gates** on document-filing, client-communication, court-submission.
- **Mandatory human attorney review** on any output going to a client or court.

**Use cases:** legal research, contract review, due-diligence, e-discovery assistance, regulatory compliance review.

## Education agents

**Compliance:** FERPA (US) + GDPR (EU) + COPPA if under-13 + state privacy laws + accessibility (WCAG 2.2 / Section 508) + AI-in-education state regulations.

**Sector-specific architecture:**

- **Age-gating** + parental-consent flows for minors.
- **Bias auditing** for fair-grading concerns; cross-channel verifier with diverse training corpora.
- **Citation + source-grounding** (avoid hallucinated facts in tutoring).
- **Domain-finetuned** on educational content + Khan-style explanations + pedagogy research.
- **Specialized MCP servers**: SIS (Student Information System), LMS (Canvas / Moodle / Blackboard), Khan Academy, OpenStax.
- **Accessibility-first UX** ([277](277-agent-ux-patterns-2026.md)).

**Use cases:** tutoring ([281-mentat-learn-seven-layer-stack-apply-plan](281-mentat-learn-seven-layer-stack-apply-plan.md) educational variant), grading assistance, curriculum generation, special-education adaptation.

## High-volume horizontals (less-regulated)

### Customer service

- **Validated by Klarna case study** ([309](309-production-case-studies-2026.md)).
- **LangGraph state-machine** workflow.
- **Per-customer memory** + per-issue-type ReasoningBank.
- **LLM-judge + human review** on confidence-band.
- **Bright-line on refunds / account changes**.

### Sales / SDR

- **Lead qualification, outreach personalization, CRM updates**.
- **Bright-line on outbound mass-email** (legal + spam considerations).
- **Per-customer memory + persona modeling**.

### Software engineering ([291-orion-code](291-orion-code-seven-layer-stack-apply-plan.md))

- **Custom ACI** is the dominant lever.
- **Worktree isolation** mandatory.
- **Bright-line on git push to main**, package install, prod-config.

## Per-sector seven-layer stack overlay

| Layer | Healthcare | Finance | Legal | Education |
|---|---|---|---|---|
| L1 Foundation | strict bridge | strict bridge | strict bridge + per-engagement tenant | minor-aware bridge |
| L2 Capability | Bio-PRM | finance-finetuned | legal-citation verifier | pedagogy-finetuned |
| L3 Protocol | FHIR / PubMed MCPs | Bloomberg / Refinitiv | Westlaw / LexisNexis | SIS / LMS |
| L4 Runtime | LangGraph (durable) | LangGraph (audit) | LangGraph (citation) | LangGraph (state) |
| L5 Security | PHI redactor | tamper-evident | privileged-content | accessibility-first |
| L6 Operations | bias + accuracy SLOs | latency + accuracy | citation-correctness | pedagogy-effectiveness |
| L7 Compliance | HIPAA + FDA + IRB | SOX + PCI + FINRA | bar rules + privilege | FERPA + COPPA + bias |

## Cost economics per sector

| Sector | Per-task cost target | Pricing model |
|---|---|---|
| Healthcare clinical | $0.50-$5 (FDA-grade quality) | Per-encounter or subscription |
| Finance trading | $0.10-$1 | Per-trade-decision or subscription |
| Legal research | $0.50-$5 (citation verification expensive) | Per-research-hour-equivalent |
| Education tutoring | $0.05-$0.50 | Per-month subscription |
| Customer service | $0.05-$0.30 (high volume) | Per-ticket |
| SWE | $0.10-$1 (depends on PR complexity) | Per-PR or per-month |
| Sales / SDR | $0.05-$0.50 | Per-lead-touched |

## Sector verticals as future deep-dive expansions

This is an **overview** — each sector deserves a dedicated multi-file deep-dive (per [296-corpus-consolidation-review](296-corpus-consolidation-review-2026.md) "what's missing"). Roadmap:

| Sector | Deep-dive priority | Target files |
|---|---|---|
| Healthcare | P1 | RadAgent, Helix-Bio expansion, FDA QSR specifics, clinical-decision-support |
| Finance | P1 | Loan processing, trading research, regulatory review, fraud detection |
| Legal | P2 | Contract review, e-discovery, citation verification |
| Education | P2 | Tutoring, grading, curriculum, accessibility |
| Customer service | P3 | Klarna deep-dive, escalation patterns |
| Sales | P3 | SDR automation, lead qualification |
| SWE | done | [291](291-orion-code-seven-layer-stack-apply-plan.md), [46](46-components-of-coding-agent.md), [62](62-everything-claude-code.md) |

## One-line takeaway

**Sector verticals overlay sector-specific compliance + MCP servers + skills + verifier ensembles on top of the seven-layer stack — healthcare adds HIPAA + FDA + PHI redactor + clinician review, finance adds SOX + PCI + tamper-evident audit + cross-channel ensemble, legal adds attorney-client privilege + per-engagement tenancy + citation verifier, education adds FERPA + COPPA + bias auditing + accessibility — and each deserves a future dedicated multi-file deep-dive; the architectural lesson: **the seven-layer stack is sector-agnostic by design, sector overlays add per-vertical compliance + tooling + verifier discipline**.**
