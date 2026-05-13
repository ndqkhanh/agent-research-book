# 287 — Helix-Bio × Seven-Layer Stack Apply Plan 2026

**Anchors.** Helix-Bio — biology / bioinformatics agent (sequence analysis, structure prediction, lab notebook integration) ([projects/helix-bio](../projects/helix-bio/)). Companion: [219-helix-bio-multi-hop-collaborative-apply-plan](219-helix-bio-multi-hop-collaborative-apply-plan.md), Biopython skill, [33-dnahnet-genomic-foundation](33-dnahnet-genomic-foundation.md), [28-radagent-agentic-radiology](28-radagent-agentic-radiology.md).

**One-line definition.** A **per-layer apply plan** for Helix-Bio — the **biology-domain specialist** with sequence-search / BLAST / structure-prediction / lab-notebook integration — emphasizing the **regulated-domain compliance overlay** (HIPAA for clinical work, FDA QSR for medical-device-adjacent, IRB/ethics for human research), **specialized MCP servers** for bioinformatics (BLAST, AlphaFold, NCBI, UniProt, PubMed), **citation-verifier rigor** for scientific claims, and **chain-of-custody** for any clinical / regulated data — staged across four 90-day phases.

## Per-layer plan

### L1 Foundation
Standard. Strict Permission Bridge for any clinical-data action. Daemon for long-running structure-prediction jobs.

### L2 Capability
**Pretraining**: domain-finetuned base on PubMed + UniProt + biology corpus ([275](275-agent-finetuning-2026.md)). **TTC**: thinking model for hypothesis generation. **Trajectory**: long-horizon for multi-stage analyses. **Multi-agent**: STRIDE-shape biology team (sequence specialist + structure specialist + literature specialist + verifier). **Verifier**: cross-channel + domain-tuned (Bio-PRM).

### L3 Protocol
- **MCP**: BLAST MCP, AlphaFold MCP, NCBI / GenBank MCP, UniProt MCP, PubMed MCP, lab-notebook MCP, Biopython MCP.
- **A2A**: helix-bio exposes `sequence-analysis`, `structure-prediction`, `literature-context` capabilities.
- **AGNTCY**: published OASF with trust attributes (audit_date, vulnerability_scan_passing).
- **Tailscale + NATS**: lab-notebook sync across instruments.
- **SKILL.md**: bioinformatics skills via argus, with strict trust tier.
- **Routines**: scheduled corpus updates, periodic re-analysis.

### L4 Runtime
LangGraph for state-machine workflows (load → analyze → synthesize → cite). Workflow durability mandatory (some jobs run for hours).

### L5 Security
- **Prompt injection**: defense critical for PubMed / web-scraped content.
- **Supply chain**: vendored bio-skills; SBOM with biology-tool versions.
- **Isolation**: container per analysis; clinical data in dedicated VM with capability MAC.
- **Bright-line**: `PATIENT_DATA_ACCESS`, `EXTERNAL_DATA_SHARE`, `CLINICAL_DECISION_SUPPORT_OUTPUT`.

### L6 Operations
- **Observability**: per-step provenance for analytical reproducibility.
- **Eval**: domain-specific eval suite (sequence-prediction accuracy, structure-quality scores, literature-grounding).
- **Durability**: LangGraph + Postgres + saga for multi-step analyses.
- **SRE**: per-experiment cost cap, regulatory-aware quality SLO.

### L7 Compliance
- **HIPAA** for any clinical / patient data (BAA required with model providers; audit logs 6yr+).
- **EU AI Act** — high-risk if used in clinical decision support.
- **FDA QSR** for medical-device-adjacent.
- **IRB / ethics review** for human research integration.

## Phased rollout

| Phase | Scope | Duration |
|---|---|---|
| **P1** | L1-L2 + MCP integrations + LangGraph | 90 days |
| **P2** | L3 (A2A) + L5 Security (regulated-domain) + bio-PRM | 90 days |
| **P3** | L6 Operations + per-domain eval | 90 days |
| **P4** | **L7 Compliance** (HIPAA / FDA / IRB) + production | 90 days |

## One-line takeaway

**Helix-Bio adopts the seven-layer stack with **regulated-domain overlay** — HIPAA + FDA QSR + IRB compliance as load-bearing — bioinformatics-specialized MCP servers (BLAST / AlphaFold / NCBI / UniProt / PubMed), domain-finetuned reasoning model, multi-agent STRIDE-bio team, durable LangGraph workflows for hour-long analyses, and chain-of-custody audit for all clinical data; the regulatory burden is the binding constraint, but the seven-layer stack maps cleanly to FDA-grade evidence collection.**
