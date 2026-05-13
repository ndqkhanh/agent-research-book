# 311 — Finance Agents Deep-Dive: SOX, PCI-DSS, FINRA, KYC/AML, trading + lending + fraud

**Anchors.** [127-loan-processing-multi-agent-case-study](127-loan-processing-multi-agent-case-study.md), [74-kronos-foundation-model-financial-markets](74-kronos-foundation-model-financial-markets.md), [303-sector-verticals-overview](303-sector-verticals-overview.md), [272-agent-compliance-and-audit](272-agent-compliance-and-audit.md), [310-healthcare-agents-deep-dive](310-healthcare-agents-deep-dive.md). External: SOX (Sarbanes-Oxley) §404, PCI-DSS v4.0, FINRA rules (esp. 3110 supervision, 4511 books and records), Bank Secrecy Act / KYC / AML (FinCEN), Reg-Z (TILA), Reg-CC, Dodd-Frank, EU MiFID II, EU AI Act high-risk Annex III §5 (creditworthiness assessment).

**One-line definition.** A 2026 deep-dive on **finance agents** — trading research, loan processing ([127](127-loan-processing-multi-agent-case-study.md)), KYC/AML, fraud detection, customer service in banking, regulatory review — with **SOX + PCI-DSS + FINRA + AML/KYC + EU AI Act high-risk** as the compliance overlay (creditworthiness assessment is explicitly high-risk per Annex III), **tamper-evident audit log with cryptographic chain-of-custody** as the load-bearing control (financial regulators require irrefutable transaction history), **multi-level approval bright-lines** scaled by transaction-amount, **cross-channel verifier ensemble** on every consequential output, and **domain-finetuning** ([74-kronos-foundation-model-financial-markets](74-kronos-foundation-model-financial-markets.md), [275](275-agent-finetuning-2026.md)) on market data + filings + regulations as the capability differentiator — making finance the **second-highest-stakes / second-highest-margin** agent vertical after healthcare, with production unit economics that justify $30K finetune compute at moderate transaction volume.

## Why finance matters

Finance is the second canonical regulated-vertical agent deployment. Stakes are very high (errors = financial loss + regulatory fines), regulation is strict (SOX + PCI + FINRA + AML), margins are high ($150-500/hour analyst labor displaced), and audit requirements are stringent (7-year retention typical). The 2024-2026 production wave includes: trading research assistants (Bloomberg, Hedgeye, internal at Citadel/Renaissance/D.E. Shaw), loan-processing automation ([127](127-loan-processing-multi-agent-case-study.md)), KYC/AML transaction monitoring, fraud detection (Stripe Radar agentic enhancements per [300](309-production-case-studies-2026.md)), customer service in banking (Bank of America, Klarna), regulatory-filing review (compliance assistants).

## Compliance overlay (load-bearing)

| Regulation | Scope | Implementation |
|---|---|---|
| **SOX** §404 (US public co.) | Financial reporting controls | Audit log + access control + segregation of duties + management attestation |
| **PCI-DSS v4.0** | Card data | Encryption + tokenization + network segmentation + audit |
| **FINRA** rules (US broker-dealers) | Supervision (3110), books-and-records (4511), communications (2210) | Audit log 6+ years + supervisory review + communication archiving |
| **BSA / KYC / AML** | Customer ID + suspicious activity | KYC pipelines + SAR (Suspicious Activity Report) generation + audit |
| **Reg-Z (TILA)** | Consumer credit | Disclosure accuracy + fair-lending |
| **Dodd-Frank** | Consumer protection (CFPB) | Fair-lending + bias audit |
| **EU MiFID II** | EU markets | Best-execution + transaction reporting |
| **EU AI Act high-risk** Annex III §5 | Creditworthiness assessment | Risk management + bias audit + human oversight |
| **GDPR Art. 22** | Automated decisions w/ legal effect | Explanation right + opt-out |

**Key tamper-evident audit requirements:** SEC Rule 17a-4 + FINRA 4511 require **WORM (write-once-read-many) storage** with audit trail; modern implementations use **AWS S3 Object Lock** or **Sigstore transparency log** for cryptographic chain-of-custody.

## Finance-specific seven-layer overlay

### L1 Foundation

- Permission Bridge with **finance action kinds**: `MARKET_DATA_READ`, `POSITION_READ`, `POSITION_MODIFY`, `TRADE_SUBMIT`, `LOAN_APPROVE`, `LOAN_DENY`, `KYC_REVIEW`, `SAR_FILE`, `CUSTOMER_DATA_READ`, `CUSTOMER_DATA_WRITE`, `BILLING_CHARGE`, `FUND_TRANSFER`.
- **Multi-level approval scaled by amount**:
  - <$100 — single approval
  - $100-$10K — supervisor approval
  - $10K-$1M — manager approval
  - $1M+ — executive approval + risk committee
- **Bright-line gates** on every consequential action.

### L2 Capability

- Base model: domain-finetuned per [74-kronos-foundation-model-financial-markets](74-kronos-foundation-model-financial-markets.md) or [275](275-agent-finetuning-2026.md); finetune on market data + 10-K/10-Q filings + research notes + regulations.
- **Finance-PRM** trained on (decision-context, recommendation, expert-judgment) with focus on regulatory-compliance; cross-channel ensemble for high-stakes.

### L3 Protocol

- **MCP servers**:
  - Bloomberg Terminal MCP (paid, BBL/BBR/BBV)
  - Refinitiv (formerly Reuters) MCP
  - Internal trading systems (read-only by default)
  - SEC EDGAR MCP (filings)
  - SEC Rule 605/606 reporting MCP
  - Regulatory filings MCP (FINRA Form U4/U5, etc.)
  - Internal customer database MCP (with strong access control)
- **A2A peers**: cipher-sec for fraud-detection handoff; aegis-ops for trade-system monitoring.
- **AGNTCY OASF** with regulatory attributes.

### L4 Runtime

LangGraph for state-machine workflows (typical: customer-context → policy-check → recommendation → multi-level approval → execution → reporting). Postgres checkpointer mandatory for SOX audit. `interrupt()` at every approval level.

### L5 Security — strongest tamper-evidence

- **Tamper-evident audit log** with cryptographic chain-of-custody (Sigstore transparency log + S3 Object Lock).
- **Bright-line gates** on every transaction, position-modify, customer-data action.
- **Cross-channel verifier ensemble** (3+ model families, including domain-finetuned + frontier + finance-PRM).
- **Encryption** (FIPS 140-3 validated): AES-256 at-rest, TLS 1.3 + mTLS in-transit.
- **Network segmentation** (PCI-DSS): isolated VLAN for cardholder-data-environment.
- **Tokenization** for card numbers (PCI-DSS).
- **Per-engagement isolation** ([271](271-agent-isolation-patterns.md)): per-customer container; per-trade-desk memory partition.
- **Real-time anomaly detection** via observability ([264](264-agent-observability-stack-2026.md)).
- **Memory provenance + classification** (public market data / internal proprietary / customer PII).

### L6 Operations

- **Observability** with PII / cardholder-data-redacted spans; SOX-grade audit log retention 7 years.
- **Eval**: regulatory-compliance eval; trading-decision quality (Sharpe / IR if applicable); KYC false-positive / false-negative rates; bias audit on lending.
- **Durability**: every transaction idempotent; saga + outbox for multi-step trades; reconciliation routines.
- **SRE**: 99.99% availability for trading; 99.9% for back-office; sub-100ms latency for trade routing; runbooks for compliance breach + position-reconciliation-mismatch + AML-alert-flood.

### L7 Compliance

- **SOX 404** annual external audit + management attestation.
- **PCI-DSS v4.0** annual assessment (QSA-led if Level 1 merchant).
- **FINRA 3110/4511** quarterly supervisory review + annual exam preparedness.
- **BSA/KYC/AML** annual SAR rate review + FinCEN reporting.
- **Reg-Z + Dodd-Frank fair-lending** quarterly bias audit.
- **EU AI Act Annex III §5** conformity assessment for any creditworthiness-assessment use.
- **GDPR Art. 22** explanation right for any automated credit decision.

## Use-case deep-dives

### Trading research assistants (Renaissance / Citadel-shape)

Reads market data + research notes + filings → generates trade ideas + supporting analysis → presents to PM for review.

**Architecture:** Polaris-shape long-running research daemon ([279](279-polaris-seven-layer-stack-apply-plan.md)) with finance-domain MCPs; bright-line on any trade-submission; cross-channel verifier on every recommendation.

**Production:** internal at top quant funds; commercial offerings emerging (Bloomberg AI, Hedgeye AI).

### Loan processing ([127-loan-processing-multi-agent-case-study](127-loan-processing-multi-agent-case-study.md))

Reads borrower documents + credit-bureau data → applies underwriting policy → recommends approve/deny + terms → human underwriter reviews.

**Architecture:** multi-agent team — document reader + policy applicator + risk scorer + bias auditor + writer. Bright-line on every approve/deny; bias audit mandatory; EU AI Act high-risk if EU borrowers.

### KYC/AML transaction monitoring

Streams transactions → flags suspicious patterns → enriches with context → drafts SAR → human compliance officer reviews.

**Architecture:** event-driven AutoGen-shape ([261](261-autogen-v04-deep-dive.md)) actor receiving transaction events; multi-classifier ensemble for false-positive reduction; bright-line on SAR filing.

**Production:** ComplyAdvantage, Hummingbird, internal at major banks.

### Fraud detection (Stripe Radar enhancement per [309](309-production-case-studies-2026.md))

Scores incoming transactions in real-time; flags suspicious; explains reasoning to customer service.

**Architecture:** low-latency (P99 < 100ms) with cached classifier + agentic post-hoc explanation; bright-line on auto-decline above threshold.

### Regulatory-filing review

Reads 10-K/10-Q + comparable filings + regulations → generates compliance review + flags issues → human compliance officer reviews.

**Architecture:** atlas-research-shape ([286](286-atlas-research-seven-layer-stack-apply-plan.md)) with SEC EDGAR + regulatory MCPs; long-context for full-filing read; citation verifier.

## Cost economics

| Use case | Per-task cost | Pricing model | Margin |
|---|---|---|---|
| Trading research idea generation | $5-50 | Per-month seat $5K-50K | 90%+ |
| Loan processing | $1-5 | Per-application markup $25-100 | 80%+ |
| KYC/AML transaction monitor | $0.001-0.01 (high volume) | Per-transaction $0.005-0.05 | 50%+ at scale |
| Fraud detection | $0.0001 (latency-bound) | Per-transaction $0.001-0.01 | varies |
| Regulatory-filing review | $5-50 | Per-review $200-2000 | 95%+ |

Finance is high-margin because analyst/compliance labor is $150-500/hour and audit-grade quality is mandatory.

## Failure modes (finance-specific)

- **Hallucinated facts** in research → wrong trade idea → loss + reputational damage.
- **Bias in lending decisions** → CFPB action + Reg-Z violation.
- **Missed AML pattern** → BSA violation + fines.
- **PII leak via logs** → GDPR fine + customer notification.
- **Wrong KYC** → onboarding fraud customer.
- **Trade entry error** → market impact + costly reversal.
- **Compliance drift** → SOX / SEC enforcement.
- **Adversarial input** in customer service → social-engineering attack.
- **Audit trail tampering** (insider threat) → regulatory investigation.

## Production checklist (finance-specific extensions to [273](273-agent-security-synthesis-2026.md))

- [ ] **F1.** Tamper-evident audit log (S3 Object Lock + Sigstore) with 7-year retention.
- [ ] **F2.** Multi-level approval bright-lines scaled by amount.
- [ ] **F3.** Cross-channel verifier ensemble on every consequential output.
- [ ] **F4.** Domain-finetuned base on market data + filings + regulations.
- [ ] **F5.** PII / cardholder-data redactor at every boundary.
- [ ] **F6.** PCI-DSS network segmentation if cardholder data touched.
- [ ] **F7.** Bias audit quarterly for lending / creditworthiness decisions.
- [ ] **F8.** AML pattern eval on production traffic; SAR filing rate sanity check.
- [ ] **F9.** SOX 404 quarterly internal audit; annual external.
- [ ] **F10.** FINRA supervisory review monthly.
- [ ] **F11.** EU AI Act conformity assessment for credit-decision use cases.
- [ ] **F12.** Real-time anomaly detection on trade-system writes.
- [ ] **F13.** Saga + outbox for multi-step trades; reconciliation routine.
- [ ] **F14.** Explanation right (GDPR Art. 22) for every automated decision affecting EU customer.
- [ ] **F15.** Disaster recovery + business continuity plan validated quarterly (SOX expectation).

## One-line takeaway

**Finance agents are the second-highest-stakes / second-highest-margin sector — SOX + PCI-DSS + FINRA + BSA/AML + EU AI Act high-risk overlay drives tamper-evident audit log + multi-level approval bright-lines + cross-channel verifier ensemble as load-bearing controls; domain-finetuning on market data + filings + regulations is the capability differentiator; production examples (loan processing, KYC/AML monitoring, trading research, fraud detection, regulatory review) validate the seven-layer stack with finance overlay; the 15-item finance-specific production checklist extends [273](273-agent-security-synthesis-2026.md) for SEC + FINRA + CFPB-grade evidence collection; second only to healthcare in stakes and exceeds it in audit-trail rigor.**
