# 284 — Cipher-Sec × Seven-Layer Stack Apply Plan 2026

**Anchors.** Cipher-Sec — security agent (vulnerability triage, threat modeling, security review, incident response) ([projects/cipher-sec](../projects/cipher-sec/)). Per-layer source: [225](225-agent-era-scaling-synthesis.md), [258](258-agent-protocol-stack-synthesis-2026.md), [263](263-production-agent-runtime-synthesis-2026.md), [268](268-agent-operations-synthesis-2026.md), [273](273-agent-security-synthesis-2026.md), [269](269-prompt-injection-2026.md), [270](270-agent-supply-chain-security.md), [271](271-agent-isolation-patterns.md). Companion: [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md), [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [35-malicious-intermediary-attacks](35-malicious-intermediary-attacks.md).

**One-line definition.** A **per-layer apply plan** for cipher-sec — the **security agent** that performs vulnerability triage, threat modeling, security review, and security-incident response — emphasizing the **highest security discipline** (cipher-sec must defend itself against the same attacks it identifies in others), **multi-tier verifier ensemble** for security-finding confidence, **strict tool-output handling** because security-relevant content (CVE feeds, exploit databases, untrusted code samples) is high-risk, and **regulatory compliance** ([272](272-agent-compliance-and-audit.md)) since security agents in regulated industries (finance, healthcare, government) face the strongest audit requirements — staged across four 90-day phases targeting security-team adoption.

## Cipher-Sec shape

Cipher-Sec is a **security-domain specialist agent** running:
- vulnerability triage (CVE analysis, prioritization);
- threat modeling (STRIDE-shape analysis on architectures);
- security code review (SAST findings + LLM-augmented analysis);
- security-incident response (handoff target from aegis-ops);
- compliance audit support (gap analysis vs frameworks).

Critical property: cipher-sec **reads adversarial content** (exploits, CVE descriptions, attacker-controlled samples) as part of normal operation; it must be the **most defended** agent in the ecosystem because compromise of cipher-sec is leverage for compromising everything cipher-sec touches.

## Per-layer plan

### L1 Foundation — security-grade

`harness_core/foundation/`. **Strictest Permission Bridge** — every external tool call gated; default-deny on unknown action kinds. Per-engagement permission contexts; client-specific scope.

### L2 Capability — security-domain specialization

[225-agent-era-scaling-synthesis](225-agent-era-scaling-synthesis.md):

- **Pretraining**: domain-finetuned ([275-agent-finetuning-2026](275-agent-finetuning-2026.md)) on security corpus (CVEs, security-research papers, exploit databases, OWASP, CWE).
- **TTC**: thinking model for hard threat-modeling questions; reasoning lift important.
- **Trajectory**: long-horizon for comprehensive audits.
- **Multi-agent**: STRIDE-shape teams (one specialist per category).
- **Verifier**: cross-channel ensemble (3+ model families) for high-stakes findings.

### L3 Protocol — secure-by-default

[258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md). Stricter trust tiers throughout:

- **MCP**: only `verified-vendor` MCP servers; CVE database, exploit-DB, OWASP, NVD, CISA KEV.
- **A2A**: minimal; mostly receiving incident handoffs from aegis-ops.
- **AGNTCY**: highest trust tier publication.
- **Tailscale + NATS**: strictest ACLs.
- **SKILL.md + marketplace**: vendored skills only initially; marketplace skills require manual review even at `audited` tier.
- **Routines + Agent Teams**: Routines for nightly CVE scans + dependency audits; Agent Teams for STRIDE-shape parallel analysis.

### L4 Runtime — LangGraph for audit workflows

[259-langgraph-deep-dive](259-langgraph-deep-dive.md). Security audits are state-machine workflows (gather → analyze → verify → report); LangGraph fits. Time-travel capability is valuable for security postmortems. Postgres checkpointer mandatory.

### L5 Security — defense-in-depth maximalism

[273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md). Cipher-sec must be the most defended:

- **Prompt-injection defense.** Maximum: full 5-layer defense + adversarial-finetuning (Tülu-3 + adversarial corpus); attack-replay regression suite; multi-turn anomaly detection. Cipher-sec reads attacker-content extensively — defense is structural.
- **Supply chain.** Vendored everything possible; SBOM with cosign / Sigstore signing; revocation routine running every 4 hours instead of nightly.
- **Isolation.** **Highest tier**: micro-VM ([271-agent-isolation-patterns](271-agent-isolation-patterns.md), Firecracker / gVisor); capability-based MAC (SELinux); strict network egress allowlist (CVE / NVD / KEV / OWASP only); per-engagement isolated tenant.
- **Bright-line gates.** Every external action gated; **two-person review** for any client-data-touching action; cross-channel verifier on every high-confidence finding.
- **Memory provenance.** All memory tagged with classification (public / internal / client-confidential).
- **Audit log** in tamper-evident storage with cryptographic chain-of-custody.

### L6 Operations — security-grade SRE

[268-agent-operations-synthesis-2026](268-agent-operations-synthesis-2026.md):

- **Observability.** Full OTel + HIR; security-event-specific spans; SIEM integration.
- **Evaluation.** Per-finding-type accuracy + false-positive rate; eval against known-CVE corpus; adversarial-input robustness.
- **Durability.** LangGraph + Postgres; idempotency keys; saga for multi-step audits.
- **SRE.** Production-grade with security-incident-aware runbooks; on-call rotation including a security engineer; incident severity tiers (P0 = client-data exposed, P1 = compromise detected, P2 = false-positive flood, P3 = degraded).

### L7 Compliance — strongest

[272-agent-compliance-and-audit](272-agent-compliance-and-audit.md). Cipher-sec serves regulated clients; compliance is mandatory:

- **SOC 2 Type II** annual audit covering all five trust services (security, availability, confidentiality, privacy, processing integrity).
- **ISO/IEC 42001** AI Management System certification.
- **ISO/IEC 27001** Information Security Management.
- **EU AI Act** — likely high-risk if used in critical-infrastructure security.
- **Industry-specific**: PCI-DSS for finance clients, HIPAA for healthcare clients, FedRAMP for US-government clients.
- **Audit trail** with cryptographic chain-of-custody (Sigstore transparency log).

## Phased rollout

| Phase | Scope | Duration |
|---|---|---|
| **P1** | L1 Foundation + L2 Capability (CVE + threat-modeling) + L3 (MCP integrations) | 90 days |
| **P2** | L4 Runtime (LangGraph) + **L5 Security maximalism** + adversarial finetuning | 90 days |
| **P3** | L6 Operations (security-grade SRE) + STRIDE-shape Agent Teams | 90 days |
| **P4** | **L7 Compliance** (SOC 2 + ISO 27001 + ISO 42001 + industry-specific) + production hardening | 90 days |

## Cipher-Sec-specific patterns

- **Adversarial-finetuned base model.** Trained on adversarial inputs to recognize and refuse.
- **Multi-model verifier ensemble.** 3+ model families voting on high-stakes findings.
- **STRIDE-shape Agent Teams**: Spoofing / Tampering / Repudiation / Information Disclosure / Denial of Service / Elevation of Privilege as parallel investigators ([250-anthropic-agent-teams](250-anthropic-agent-teams.md)).
- **Cryptographic chain-of-custody** on findings (Sigstore transparency log).
- **Two-person review** for client-data actions.
- **Sandboxed exploit analysis** (micro-VM for parsing exploit databases).
- **Per-engagement tenancy isolation** (no cross-client data flow).
- **Compliance-as-code** maximally adopted.

## Cost economics

[278-agent-unit-economics-2026](278-agent-unit-economics-2026.md). Cipher-sec serves enterprise security teams:

- Per-engagement cost: $50-500 (LLM + compute) for typical audit.
- Pricing model: per-audit or per-month enterprise contract.
- ROI: human security-engineer hour at $200-500; cipher-sec produces output at fraction.

## Cross-project dependencies

- **Argus** ([282](282-argus-seven-layer-stack-apply-plan.md)) for vendored security-skills (with manual override on trust tier).
- **Aegis-Ops** ([283](283-aegis-ops-seven-layer-stack-apply-plan.md)) for security-incident handoff.
- **Lyra patterns** ([280](280-lyra-seven-layer-stack-apply-plan.md)) for memory primitives.

## One-line takeaway

**Cipher-Sec is the security-domain specialist that must be the most defended agent in the ecosystem — across four 90-day phases adopting the seven-layer stack with maximalist L5 Security (full 5-layer defense + adversarial-finetuned base + 3+ model verifier ensemble + micro-VM isolation + capability-based MAC + cryptographic chain-of-custody + two-person review for client data) and strongest L7 Compliance (SOC 2 + ISO 27001 + ISO 42001 + EU AI Act high-risk + industry-specific frameworks); the agent that finds vulnerabilities must not become one.**
