# 310 — Healthcare Agents Deep-Dive: HIPAA, FDA QSR, clinical workflows, diagnostic assistance

**Anchors.** [287-helix-bio-seven-layer-stack-apply-plan](287-helix-bio-seven-layer-stack-apply-plan.md), [303-sector-verticals-overview](303-sector-verticals-overview.md), [28-radagent-agentic-radiology](28-radagent-agentic-radiology.md), [33-dnahnet-genomic-foundation](33-dnahnet-genomic-foundation.md), [149-sector-use-case-catalog](149-sector-use-case-catalog.md), [272-agent-compliance-and-audit](272-agent-compliance-and-audit.md). External: HIPAA — 45 CFR Parts 160 + 164. FDA QSR — 21 CFR Part 820 + Software as Medical Device (SaMD) guidance. EU AI Act high-risk Annex III §5 (medical devices, healthcare access).

**One-line definition.** A 2026 deep-dive on **healthcare agents** — clinical-decision support, medical scribing, prior-authorization, drug-discovery research, radiology assistance — with the **strongest compliance overlay in the agent ecosystem** (HIPAA + FDA QSR + IRB + EU AI Act high-risk + state-specific privacy + GDPR if EU patients), **PHI handling as load-bearing security control** (BAA-required model providers, redactor at every memory + external-API boundary, 6+ year audit retention with cryptographic tamper-evidence), **clinical-grade verifier ensembles** (3+ model families on every clinical claim, plus human clinician sign-off as bright-line), and **domain-finetuning** ([287](287-helix-bio-seven-layer-stack-apply-plan.md), [275](275-agent-finetuning-2026.md)) on PubMed + UMLS + clinical guidelines + UpToDate as the capability differentiator — making healthcare the **highest-stakes, highest-regulated, highest-margin** sector for production agent deployment.

## Why healthcare matters

Healthcare is the canonical regulated-vertical agent deployment. Stakes are highest (clinical decisions affect lives), regulation is strictest (HIPAA + FDA + IRB), margins are highest ($200-500/hour clinician labor displaced), and unit economics work even at $5-10/task. The 2024-2026 production wave includes: clinical scribing (Abridge, Suki, Nuance/Microsoft DAX Copilot), prior-authorization automation (Cohere Health), diagnostic assistance (PathAI radiology — see [28-radagent-agentic-radiology](28-radagent-agentic-radiology.md)), drug-discovery research (Insilico, Recursion). Each requires the seven-layer stack with healthcare-specific overlays.

## Compliance overlay (load-bearing)

| Regulation | Scope | Implementation |
|---|---|---|
| **HIPAA** (45 CFR 160/164) | PHI handling | BAA with model providers; encryption at-rest + in-transit (FIPS 140-2/3); audit log 6+ years; access controls; breach notification ≤60 days |
| **FDA QSR** (21 CFR 820) | Medical device software (SaMD) | Quality management system; design controls; verification + validation; complaint handling |
| **IRB / ethics** | Human-subjects research | Informed consent; protocol review; data de-identification |
| **EU AI Act high-risk** (Annex III §5) | Medical access decisions | Risk management; technical documentation; human oversight; conformity assessment |
| **GDPR Art. 9** | EU patient data | Special-category lawful basis; DPIA mandatory |
| **State laws** | Varies (CCPA, NY SHIELD, etc.) | Per-state mapping |

**HIPAA-eligible model providers (May 2026):** Anthropic enterprise, OpenAI enterprise, Google Vertex AI HIPAA tier, Microsoft Azure OpenAI HIPAA, AWS Bedrock with BAA, IBM watsonx. Always verify current BAA terms.

## Healthcare-specific seven-layer overlay

### L1 Foundation (extends [279](279-polaris-seven-layer-stack-apply-plan.md))

- Permission Bridge with **healthcare action kinds**: `PHI_READ`, `PHI_WRITE`, `CLINICAL_RECOMMENDATION`, `EHR_WRITE`, `PRESCRIBE_MEDICATION`, `ORDER_LAB`, `ORDER_IMAGING`, `BILLING_CHARGE`.
- **Two-person review** (clinician + agent) for high-stakes; bright-line on `CLINICAL_RECOMMENDATION`.
- Daemon with PHI-tier scheduling (encrypted queue, no PHI in logs).

### L2 Capability — domain-finetuned

- Base model: Claude Sonnet (HIPAA BAA) or Anthropic medical-tuned variant or domain-finetuned per [275](275-agent-finetuning-2026.md).
- **Domain corpus**: PubMed (35M+ abstracts), UpToDate, UMLS (concept hierarchy), ICD-10 / SNOMED CT (coding), clinical guidelines (CDC, AHA, NCCN), drug interactions (DrugBank), FDA labels.
- **Clinical-PRM** trained on (clinical-context, candidate-recommendation, expert-judgment) tuples; augments L5 verifier.

### L3 Protocol

- **MCP servers** (HIPAA-aligned where applicable):
  - PubMed MCP, UpToDate MCP (paid), UMLS MCP
  - Epic / Cerner FHIR MCPs (clinical EHR integration)
  - ICD-10 / SNOMED CT MCP (medical coding)
  - DrugBank / RxNav MCP (drug info / interactions)
  - Clinical guidelines MCP (NCCN / AHA)
  - PACS MCP (imaging)
- **A2A peers**: domain-specialist agents (radiology / pathology / pharmacy specialists) callable via A2A from generalist clinical assistant.
- **AGNTCY OASF** with `audit_date`, `bias_audit_passing`, `clinical_sign_off`, `regulatory_status` (FDA cleared / not).

### L4 Runtime — LangGraph + durable

LangGraph state-machine (intake → evaluate → query → recommend → review → document → bill) with **Postgres checkpointer** for full audit reproducibility. `interrupt()` for clinician review at every consequential gate.

### L5 Security — strongest in the corpus

- **PHI redactor** at every boundary (memory write, external API call, log emission, observability span). Use Microsoft Presidio + custom medical-NER + UMLS redaction rules.
- **Encryption** (FIPS 140-2/3 validated): at-rest (AES-256) + in-transit (TLS 1.3 + mTLS).
- **Role-based access control** (RBAC) at the agent level; per-clinician scope.
- **Isolation** ([271](271-agent-isolation-patterns.md)): per-encounter container; per-patient memory partition; **micro-VM for any third-party-tool execution**.
- **Bright-line gates** on every clinical-recommendation, EHR-write, prescription, order, billing-charge.
- **Cross-channel verifier ensemble** (3+ model families, ideally including a domain-finetuned model + a general frontier + a clinical-PRM).
- **Human clinician sign-off** is mandatory for any output going to patient or chart.

### L6 Operations

- **Observability** ([264](264-agent-observability-stack-2026.md)) with PHI-redacted spans; tamper-evident audit log (Sigstore transparency log) for HIPAA + FDA evidence.
- **Eval** ([265](265-agent-evaluation-2026.md)): clinical-correctness eval suite (USMLE-shape, MedQA, MedMCQA, custom institutional cases); bias-audit eval; clinician-rated quality.
- **Durability** ([266](266-agent-durability-and-idempotency.md)): every clinical recommendation idempotent; chart updates via outbox pattern; saga compensation for any multi-step intervention.
- **SRE** ([267](267-agent-sre.md)): 99.9% availability for clinical-decision-support; runbooks for PHI-leak, mis-recommendation cluster, drug-interaction-miss; quarterly chaos exercise.

### L7 Compliance

- **HIPAA SRA** (Security Risk Assessment) annually.
- **FDA SaMD classification** if used in clinical decision-making; conformity assessment per IEC 62304 software-lifecycle.
- **EU AI Act** Annex III high-risk obligations effective 2026-08-02.
- **DPIA** for GDPR EU patients.
- **Bias auditing** quarterly; demographic-parity + equalized-odds across patient demographics.
- **Audit log retention 6+ years** in tamper-evident storage.

## Use-case deep-dives

### Clinical scribing (highest-volume use case)

Replaces clinician documentation effort. Listens to clinician-patient interaction → transcribes → structures into SOAP note → submits to EHR for clinician review.

**Architecture:** voice agent ([288](288-harmony-voice-seven-layer-stack-apply-plan.md)) → text agent → EHR via FHIR MCP. Bright-line gate before EHR write; clinician review mandatory.

**Production examples:** Abridge ($2B+ valuation 2024), Suki, DAX Copilot.

### Prior-authorization automation

Reads insurance criteria + patient chart → drafts prior-auth request → submits via payer portal.

**Architecture:** read-only chart access via FHIR MCP → criteria-matching skill → request-drafting skill → submission via payer-portal MCP. Bright-line on submission.

### Diagnostic assistance ([28-radagent-agentic-radiology](28-radagent-agentic-radiology.md))

Vision agent reads imaging → flags anomalies → cites reference cases → presents to radiologist for review.

**Architecture:** vision-capable model (Claude 3.7 vision or domain-finetuned) + PACS MCP + radiology PRM + bright-line on report submission. **Never autonomous; always advisory.**

### Drug-discovery research

Multi-step research pipeline: target identification → compound screening → property prediction → toxicity simulation → synthesis recommendation.

**Architecture:** Polaris-shape long-running daemon ([279](279-polaris-seven-layer-stack-apply-plan.md)) with bio-MCPs (BLAST, AlphaFold, PubChem, ChEMBL); IRB review on any experimental work; runs over weeks.

## Cost economics

| Use case | Per-task cost | Pricing model | Margin |
|---|---|---|---|
| Clinical scribing | $0.50-2 | Per-encounter, $0.10-1 markup | 50%+ |
| Prior-auth | $0.30-1 | Per-request, $5-20 markup | 90%+ |
| Diagnostic assist | $1-5 | Per-study, $20-50 markup | 80%+ |
| Drug-discovery | $50-500 | Per-program, $10K-100K markup | varies |

Healthcare is the highest-margin agent vertical because clinician labor displaced is $200-500/hour.

## Failure modes (healthcare-specific)

- **PHI leakage** via memory or logs — redactor failures.
- **Hallucinated clinical claims** — citation requirement + cross-channel verifier mandatory.
- **Bias in recommendations** — quarterly audit + diverse training data.
- **Drug-interaction miss** — explicit lookup + verifier check.
- **Coding errors** (ICD-10 / CPT) — verifier check + clinician review.
- **Documentation drift** — eval-pass-rate continuously tracked.
- **Adversarial attack** via patient-supplied content — spotlight + classifier defense.
- **FDA non-compliance** if SaMD classification missed — legal exposure.

## Production checklist (healthcare-specific extensions to [273](273-agent-security-synthesis-2026.md))

- [ ] **HC1.** BAA verified with every model provider touching PHI.
- [ ] **HC2.** PHI redactor passing on >99.99% of writes (audit subsample).
- [ ] **HC3.** Audit log retention 6+ years in tamper-evident storage.
- [ ] **HC4.** Bright-line gate on every clinical-recommendation / EHR-write / prescription / order / billing.
- [ ] **HC5.** Two-person review (clinician + agent) for high-stakes.
- [ ] **HC6.** Cross-channel verifier ensemble (3+ model families) on clinical claims.
- [ ] **HC7.** Domain-PRM ([97-qwen-prm](97-qwen-prm.md)-shape) trained on clinical data.
- [ ] **HC8.** FDA SaMD classification documented if applicable.
- [ ] **HC9.** EU AI Act Art. 11 technical documentation maintained.
- [ ] **HC10.** Bias audit quarterly with demographic-parity check.
- [ ] **HC11.** HIPAA SRA annually; finding remediation tracked.
- [ ] **HC12.** Chaos exercise quarterly including PHI-leak red-team.
- [ ] **HC13.** Clinician sign-off on every output to patient or chart.
- [ ] **HC14.** State-law mapping for every deployment region.
- [ ] **HC15.** Patient consent flows for any agentic interaction.

## One-line takeaway

**Healthcare agents are the highest-stakes / highest-regulated / highest-margin sector — HIPAA + FDA SaMD + IRB + EU AI Act high-risk overlay drives PHI redactor + tamper-evident audit + cross-channel verifier ensemble + clinician sign-off as load-bearing controls; domain-finetuned base on PubMed + UMLS + UpToDate + clinical guidelines is the capability differentiator; production examples (Abridge, Suki, DAX Copilot, Cohere Health, RadAgent, Insilico) validate the seven-layer stack with healthcare overlay; the 15-item healthcare-specific production checklist extends [273](273-agent-security-synthesis-2026.md) for FDA-grade evidence collection.**
