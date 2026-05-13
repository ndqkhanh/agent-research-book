# 270 — Agent Supply-Chain Security: model, MCP server, marketplace skill, and memory poisoning vectors

**Anchors.** SLSA — *Supply-chain Levels for Software Artifacts* — https://slsa.dev/ — Linux Foundation reference for software supply-chain security. Sigstore / cosign — https://www.sigstore.dev/ — keyless signing for artifacts. OWASP Top 10 for LLM Applications — https://owasp.org/www-project-top-10-for-large-language-model-applications/ — LLM02 (Insecure Output Handling), LLM05 (Supply Chain Vulnerabilities). Anthropic — *Sleeper Agents* — arXiv:2401.05566 — backdoor models that survive safety training. Companions: [269-prompt-injection-2026](269-prompt-injection-2026.md), [82-poisonedrag](82-poisonedrag.md), [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md), [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md), [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md), [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md), [271-agent-isolation-patterns](271-agent-isolation-patterns.md), [273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md), [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md).

**One-line definition.** A 2026 framework for **supply-chain security in agent harnesses** — extending classical software supply chain (npm / PyPI / container registry) to **five agent-specific artifact classes** that all require **provenance, signing, trust-tiering, and revocation**: (1) **base model** (pretrained weights from HF / vendor — sleeper-agent backdoors per Anthropic 2024, training-data poisoning, vendor compromise), (2) **MCP servers** (third-party tool servers — protocol-compliant but malicious or compromised; Smithery + official registry signatures + AGNTCY trust tiers as defense), (3) **marketplace skills** (SKILL.md packages — adversarial system prompts, scope-expansion attacks; signed manifests + min-trust-tier policy + regression-test discipline), (4) **agent memory** (poisoned writes propagating across runs — adversarial inputs planting false facts; redactors + provenance tracking + memory tier isolation), (5) **cross-agent communications** (A2A messages from compromised peer agents — Signed Agent Cards as identity infrastructure); each vector needs **layered controls** — provenance verification, signing, trust tiers, runtime isolation, monitoring, and revocation propagation.

## Why this paper matters (the supply chain became the load-bearing security surface in 2026)

Classical software supply-chain attacks (Solar Winds, ua-parser-js, event-stream) taught the industry that **the artifacts your software depends on are an attack surface**. The 2024-2026 adoption of MCP servers ([256](256-mcp-2025-2026-evolution.md)), marketplace skills ([257](257-agent-skill-marketplace-landscape.md)), persistent memory ([233](233-memory-scaling-for-agents.md)), and cross-agent communications ([254](254-a2a-protocol-deep-dive.md)) created **new artifact classes** specific to agents — and each is an attack surface. By 2026 the supply chain is the dominant security concern for production agents because:

- **Agents read code, prompts, and data from many third parties.** A typical production agent depends on a base model, an MCP server fleet, several marketplace skills, a persistent memory store, and cross-agent A2A endpoints. Each is a potential injection point.
- **Compromised dependencies can persist undetected.** The Anthropic *Sleeper Agents* paper (arXiv:2401.05566) showed that backdoors in pretrained models survive standard safety training; an attacker who poisoned training data could plant behaviors that activate only on specific triggers.
- **Marketplace economies scale attacks.** A single malicious skill installed by 10,000 users compromises 10,000 deployments; signed manifests + trust tiers reduce but don't eliminate.
- **Memory-poisoning is unique to agents.** Classical software doesn't have "memory writes from prior runs"; agent memory does, and it's a propagation vector.

The 2026 mitigation framework borrows from SLSA (Supply-chain Levels for Software Artifacts) and adapts to agents. **Provenance** — every artifact has verifiable origin metadata. **Signing** — cryptographic signatures via cosign / sigstore / EdDSA. **Trust tiering** — graded trust levels (verified-vendor / audited / sandboxed / unsigned) with policies that gate execution. **Runtime isolation** — even compromised dependencies have limited blast radius via [271-agent-isolation-patterns](271-agent-isolation-patterns.md). **Monitoring** — anomaly detection on dependency behavior. **Revocation propagation** — when an artifact is retracted, deployments using it are alerted and auto-rollback.

Take this seriously and three things change. **First**, you treat **every agent dependency as a supply-chain artifact** — base model, MCP server, marketplace skill, memory partition, A2A peer agent — each with provenance, signature, trust tier, and revocation status. **Second**, you adopt **layered controls** — no single control suffices; provenance + signing + trust tiering + isolation + monitoring + revocation all apply. **Third**, you build **revocation propagation** — when an upstream vendor publishes a retraction (CVE in a model, vulnerability in MCP server, malicious skill), your platform auto-detects and rolls back; this is the operational closure of supply-chain security.

## Problem it solves (extending software supply-chain security to agent-specific artifacts)

1. **Pretrained model backdoors.** Anthropic *Sleeper Agents* (arXiv:2401.05566) demonstrated trigger-activated backdoors surviving safety training. A vendor-poisoned model is undetectable by application-layer measures.
2. **Training-data poisoning.** Adversarial training data injects behaviors activated by specific inputs. Hard to detect post-training; defense via vendor reputation + provenance audits.
3. **MCP server compromise.** A trusted MCP server is updated with malicious code; subsequent tool calls leak data or execute unauthorized actions. Signing + behavior monitoring.
4. **Marketplace skill compromise.** SKILL.md package updated with adversarial system prompt that overrides agent behavior. Signed manifests + scope-expansion detection + min-trust-tier.
5. **Memory poisoning.** Adversarial input planted in episodic memory; later runs read it as fact. Redactor + provenance tracking + memory tier isolation.
6. **Cross-agent compromise.** Trusted peer agent compromised; sends malicious A2A messages. Signed Agent Cards + trust tiering.
7. **Distillation supply chain.** Distilled student models inherit teacher backdoors. Provenance + audit.
8. **Skill-with-MCP-bundle attacks.** Skill bundles MCP server; install one, attacker controls both.
9. **Lock-file vs floating dependency.** Pinned dependencies are auditable; floating "latest" versions invisible.
10. **Revocation latency.** Days between CVE publication and deployment update; window of vulnerability.

## Core idea in one paragraph

Agent supply-chain security extends classical software supply-chain practices to **five agent-specific artifact classes**: (1) **base model** — provenance via vendor + training-data audit + checkpoint signing; (2) **MCP servers** — signed manifests via Smithery + trust tiering via AGNTCY ([255](255-agntcy-oasf-acp-deep-dive.md)) + behavior monitoring + min-trust-tier policy; (3) **marketplace skills** — SKILL.md signing + signed dependency-graphs + scope-expansion detection + min-trust-tier + regression-test discipline ([257](257-agent-skill-marketplace-landscape.md)); (4) **agent memory** — write-time redactor + provenance metadata per memory item + tier isolation (untrusted memory cannot promote to trusted) + write-attribution + retraction propagation; (5) **cross-agent communications** — A2A Signed Agent Cards + OAuth 2.1 tokens + trust tiering of peer agents + audit logging. Each artifact class needs **six layered controls**: **provenance** (origin metadata), **signing** (cryptographic verification), **trust tiering** (graded trust with policies), **runtime isolation** (limit blast radius), **behavioral monitoring** (anomaly detection), and **revocation propagation** (retraction events propagate to deployments). The 2026 production stack adopts SLSA-shape practices adapted to agents: **SBOM-equivalent for agents** (Software Bill of Materials enumerates every dependency), **cosign-shape signing** (keyless via OIDC, fast verification), **trust-tier-gated install** (refuse below threshold), **behavior baselining** (alert on deviation from expected), **revocation-as-routine** ([252](252-routines-pattern-for-self-hosted-agents.md)) (scheduled scan of advisories + auto-rollback). Without this framework, supply-chain attacks succeed silently; with it, they are detected, contained, and recovered.

## Mechanism (step by step)

### (a) Five artifact classes and their controls

| Artifact class | Provenance | Signing | Trust tier | Isolation | Monitoring | Revocation |
|---|---|---|---|---|---|---|
| **Base model** | Vendor + training data | Checkpoint signing | Vendor trust tier | Sandboxed inference | Behavior baselining | CVE feeds |
| **MCP server** | Repository + maintainer | Manifest signing | AGNTCY trust tier | Containerized | Tool-call audit | Marketplace retraction events |
| **Marketplace skill** | Author + version | SKILL.md signing | Min trust tier | Scoped tools | Regression tests | Marketplace retraction events |
| **Agent memory** | Write attribution | Per-item signing (optional) | Memory tier | Tier isolation | Redactor + content classifier | Retraction propagation |
| **Cross-agent (A2A)** | Vendor identity | Signed Agent Cards | Vendor trust tier | OAuth scopes | Behavior baselining | Retraction events |

### (b) Base-model supply chain

```
Vendor (Anthropic / OpenAI / Google / Meta / Qwen / DeepSeek)
    ↓ (training-data audit, checkpoint signing)
Model checkpoint (HF / vendor API / local file)
    ↓ (deployment, fingerprint verification)
Production inference
```

Defenses:
- **Vendor reputation tier** — frontier vendors > smaller open-weight > random fork.
- **Checkpoint fingerprint verification** — SHA-256 of weights matches vendor-published.
- **Behavior baselining** — eval suite ([265-agent-evaluation-2026](265-agent-evaluation-2026.md)) detects anomalous outputs.
- **CVE feed monitoring** — subscribe to vendor security advisories.
- **Sleeper-agent detection** — adversarial prompts probe for trigger-activated backdoors.

### (c) MCP server supply chain

```
MCP server author / org
    ↓ (Smithery / official registry submission)
Marketplace listing (signed manifest)
    ↓ (OAuth-mediated install)
Production deployment
    ↓ (runtime container)
Tool calls
```

Defenses:
- **Smithery / official registry verification** — only install from verified marketplaces.
- **Manifest signing** — cosign / EdDSA signature; verify against published JWK.
- **AGNTCY trust tiering** ([255](255-agntcy-oasf-acp-deep-dive.md)) — refuse below `audited` tier.
- **Containerized execution** — MCP server runs in sandbox ([271-agent-isolation-patterns](271-agent-isolation-patterns.md)).
- **Tool-call audit** — log every tool call with args + result; anomaly detection.
- **Behavior baselining** — eval suite for the MCP server's behavior.

### (d) Marketplace skill supply chain

```
Skill author
    ↓ (subagents.cc / buildwithclaude / LobeHub / Smithery)
Marketplace listing (signed SKILL.md + dependency graph)
    ↓ (OAuth-mediated install)
Local skill installation (~/.harness_core/skills/installed/<skill>/)
    ↓ (skill engine loads)
Agent invocation
```

Defenses:
- **SKILL.md frontmatter signing** — `signed_by`, `audit_date`, `vulnerability_scan_passing` fields verified.
- **Min trust tier policy** — `harness_core install policy: require trust_tier >= audited`.
- **Scope-expansion detection** — install of v1.1 with new tool grants prompts user approval.
- **Regression-test discipline** — pinned skill version; eval suite runs on update; auto-rollback on regression.
- **Vendored fallback** — `harness_core/skills/vendored/<skill>/` overrides marketplace.
- **Dependency-graph audit** — skills with MCP server deps audited recursively.

### (e) Memory supply chain

```
Agent run
    ↓ (memory.write_observation)
Write-time redactor (PII, secrets, prompt-injection patterns)
    ↓ (passes redaction)
Memory store (SQLite + Chroma + .md files)
    ↓ (TTL eviction, decay)
Future agent run (memory.search / memory.get)
    ↓ (consulted as memory)
Agent decision
```

Defenses:
- **Write-time redactor** ([233-memory-scaling-for-agents](233-memory-scaling-for-agents.md)) — strip injection patterns at boundary.
- **Provenance metadata** — every memory item tagged with source, trust level, write-time agent identity.
- **Tier isolation** — untrusted memory partition cannot promote to semantic / fact tier without explicit approval.
- **Content classifier** at write boundary — refuses adversarial content.
- **Write-attribution audit** — who wrote what, when, from what source.
- **Retraction propagation** — when a write-time error is discovered post-hoc, retroactive flush of dependent items.

### (f) Cross-agent (A2A) supply chain

```
Peer agent (your runtime or external)
    ↓ (A2A message: signed task or response)
Signed Agent Card verification
    ↓ (verify EdDSA signature against published JWK)
OAuth 2.1 token verification
    ↓ (scope check)
Trust tier check (refuse below threshold)
    ↓ (proceed)
Inbound message processed
```

Defenses:
- **Signed Agent Cards** ([254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md), v1.0) — cryptographic identity.
- **OAuth 2.1 scopes** — fine-grained permissions per peer.
- **AGNTCY trust tiering** ([255](255-agntcy-oasf-acp-deep-dive.md)) — verified vendor / audited tiers.
- **Behavior baselining** — anomaly detection on peer agent outputs.
- **Audit logging** — every cross-agent call logged with peer identity.

### (g) SBOM-equivalent for agents

A Software Bill of Materials for the agent platform:

```yaml
# harness_core/sbom.yaml
agents:
  polaris:
    base_model:
      vendor: anthropic
      model_id: claude-sonnet-4-6
      checkpoint_sha: "..."
    mcp_servers:
      - id: papers-mcp
        version: "1.2.0"
        signature_verified: true
        trust_tier: audited
    skills:
      - id: literature-review
        version: "2.1.0"
        signature_verified: true
        trust_tier: audited
        source: marketplace
    a2a_peers:
      - id: argus-skill-curator
        trust_tier: verified-vendor
```

Maintained automatically; consumed by SRE for revocation propagation.

### (h) Revocation-as-routine

A scheduled routine ([252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md)) checks vendor advisories nightly:

```python
@routine(schedule="0 2 * * *")  # nightly 02:00
async def check_supply_chain_advisories():
    # Pull latest CVE / retraction feeds
    advisories = await pull_advisories(["anthropic", "openai", "smithery", "agntcy", ...])
    sbom = load_sbom()
    affected = match_advisories_to_sbom(advisories, sbom)
    for item in affected:
        await alert_oncall(item)
        if item.severity >= CRITICAL:
            await auto_rollback(item)
```

Closes the supply-chain loop operationally.

## Empirical results (table — May 2026)

| Vector | Real-world incidents 2024-2026 | Mitigation maturity |
|---|---|---|
| Sleeper-agent backdoors | Anthropic 2024 paper (lab demo); no public production incident | Research-stage detection; vendor-tier reputation |
| MCP server compromise | Smithery advisories monthly | Signing + trust tiering production |
| Marketplace skill compromise | subagents.cc occasional advisories | Signing + min-trust-tier production |
| Memory poisoning | Documented in Lyra red-team [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md) | Redactor + provenance production |
| A2A peer compromise | Hypothetical (limited multi-vendor production) | Signed Cards production |

| Defense | Adoption (production agent shops) |
|---|---|
| Vendored fallback for skills | High |
| MCP signing verification | Mature shops |
| AGNTCY trust tier policy | Emerging |
| Memory redactor | High (Lyra-class deployments) |
| SBOM for agents | Mature shops |
| Revocation routine | Mature shops |
| Behavior baselining for MCP | Mid-maturity shops |
| Eval-suite regression on skill update | Mature shops |

## Variants and ablations

- **Vendored-only skills** — no marketplace; deterministic but breadth-limited.
- **Per-tier policies.** Different agents have different trust thresholds.
- **Two-person review for trust-tier upgrades.** Audit + sign-off when promoting a skill from `signed` to `audited`.
- **Memory tier promotion gates.** Untrusted → trusted requires explicit approval.
- **Reproducible builds for MCP servers.** Build artifacts deterministic; verifiable via hash.
- **Sigstore / cosign keyless signing.** OIDC-tied identity, no key management.
- **Software Heritage archival.** Long-term archive of agent dependencies.
- **EU AI Act registry compliance** ([272-agent-compliance-and-audit](272-agent-compliance-and-audit.md)).

## Failure modes and limitations

- **Provenance metadata can be forged at lower-tier.** Verified-vendor tier requires registry attestation.
- **Sleeper-agent detection is research-stage.** No production-grade detector for trigger-activated backdoors.
- **Marketplace racing attacks.** Attacker publishes lookalike skill name; users install wrong one.
- **Trust-tier inflation.** Vendors self-attest; without third-party verifier, tiers drift.
- **Revocation propagation latency.** Daily routine has up to 24h gap; critical CVEs need real-time alerts.
- **Cross-marketplace gaps.** A skill banned on one marketplace may stay live on a fork.
- **Compromised vendor identity.** EdDSA private key stolen → attacker signs malicious updates.
- **Memory-poisoning persistence.** Even after retraction, dependent decisions remain biased.
- **A2A supply chain at distance.** Peer agent's dependencies are not visible to caller; trust transitive.
- **SBOM staleness.** Manual maintenance; auto-update tooling required.

## When to use, when not

**Adopt full supply-chain security framework** for any production agent with external dependencies (MCP servers, marketplace skills, persistent memory, A2A peers); for any compliance-regulated deployment; for any deployment where supply-chain compromise would be high-impact. The strongest cases are **enterprise platforms**, **regulated agents** (healthcare, finance, government), **paying-customer SaaS**, and **multi-tenant marketplaces**.

**Skip detailed supply-chain controls** for fully-vendored prototypes (no external deps); for single-user low-stakes agents; for short-lived experiments. **Never skip** vendor-tier reputation check on base model and signature verification on installed dependencies — these are the cheapest controls with highest yield.

## Implications for harness engineering

- **`harness_core/security/supply_chain/` package.** SBOM + signature verification + trust-tier policy + revocation routine.
- **SBOM as runtime artifact.** Auto-generated; readable from observability.
- **Vendored skills as deterministic floor.** [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md), [04-skills](04-skills.md), [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md).
- **MCP signing verification mandatory.** [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md), [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md).
- **Memory redactor as supply-chain primitive.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [82-poisonedrag](82-poisonedrag.md).
- **A2A Signed Agent Cards mandatory.** [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md), [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md).
- **Trust tier policy enforcement.** [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [p24-trust-tiering-retractions](../projects/polaris/docs/research/p24-trust-tiering-retractions.md).
- **Revocation routine.** [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md) — scheduled advisory check + auto-rollback.
- **Eval-suite regression on update.** [265-agent-evaluation-2026](265-agent-evaluation-2026.md), [265](265-agent-evaluation-2026.md) — installed-skill regression.
- **Behavior baselining via observability.** [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md), [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md).
- **Compliance audit trail.** [272-agent-compliance-and-audit](272-agent-compliance-and-audit.md) — SBOM + provenance + retraction history.
- **SRE runbook for supply-chain compromise.** [267-agent-sre](267-agent-sre.md) — auto-rollback to vendored fallback.
- **Cross-channel verifier across runtime boundaries.** [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md).
- **Isolation as defense in depth.** [271-agent-isolation-patterns](271-agent-isolation-patterns.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md).
- **Pretrained model audit per Anthropic Sleeper-Agents.** [212-pretraining-scaling-laws-foundation](216-pretraining-scaling-laws-foundation.md), [216-pretraining-scaling-laws-foundation](216-pretraining-scaling-laws-foundation.md) — vendor reputation as primary control.

**One-line takeaway for harness designers.** **Agent supply-chain security extends classical software supply-chain practices to five new artifact classes (base model, MCP servers, marketplace skills, persistent memory, cross-agent A2A) — each requiring six layered controls (provenance, signing, trust tiering, isolation, monitoring, revocation propagation); maintain an SBOM-equivalent enumerating every dependency, run a nightly revocation routine, vendor your critical-path skills as deterministic fallback, and accept that the supply chain is now the dominant security surface that requires SRE-grade operational discipline.**
