# 273 — Agent Security Synthesis 2026: layered defense, threat model, and the production security checklist

**Synthesis of:** [269-prompt-injection-2026](269-prompt-injection-2026.md), [270-agent-supply-chain-security](270-agent-supply-chain-security.md), [271-agent-isolation-patterns](271-agent-isolation-patterns.md), [272-agent-compliance-and-audit](272-agent-compliance-and-audit.md). Cross-references: [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [35-malicious-intermediary-attacks](35-malicious-intermediary-attacks.md), [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md), [53-chaos-engineering-next-era](53-chaos-engineering-next-era.md), [82-poisonedrag](82-poisonedrag.md), [122-explainability-compliance](122-explainability-compliance.md), [123-robustness-fault-tolerance](123-robustness-fault-tolerance.md), [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md), [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md), [267-agent-sre](267-agent-sre.md), [268-agent-operations-synthesis-2026](268-agent-operations-synthesis-2026.md), [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md), [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md), [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md), [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md), [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md), [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md).

**One-line definition.** A **unified threat model + layered defense + production security checklist** for agents in 2026 — the threat model identifies **five attack surfaces** (user input, retrieved content, tool outputs, memory writes, cross-agent messages) and **four artifact-supply-chain classes** (base model, MCP servers, marketplace skills, A2A peers); the layered defense composes **eight controls** (input/output classifiers, instruction hierarchy, spotlight prompting, isolation, bright-line gates, supply-chain signing/trust-tiering, memory provenance, audit-trail-with-tamper-evidence) into defense in depth; and the production security checklist gives operators a 30-item gating list spanning prompt-injection, supply-chain, isolation, compliance, and operational discipline — making "is this agent ready for production?" a question with a **deterministic yes/no answer** rather than a vibes-based call.

## Why this synthesis matters (the security stack finally has a coherent shape, and the checklist makes it auditable)

The agent security landscape in 2024 was fragmented: prompt-injection papers in one corner, supply-chain attacks in another, isolation patterns in a third, compliance frameworks in a fourth. By 2026 the field has enough engineering experience and enough regulatory pressure that a **unified shape** has emerged — one threat model encompassing all attack surfaces, one layered-defense framework, one production checklist. This synthesis is that unification.

The 2026 unified threat model has two dimensions. **Attack surfaces** — where adversarial input enters: (1) **direct user input** ([269-prompt-injection-2026](269-prompt-injection-2026.md), Layer-1 classifiers); (2) **retrieved content** (web pages, emails, docs — Layer-2 spotlight + content classifier); (3) **tool outputs** (MCP server responses, executed code output — Layer-3 trust + isolation); (4) **memory writes** (persistent across runs, propagation vector — Layer-4 redactor + provenance); (5) **cross-agent messages** (A2A peer agents — Layer-5 Signed Cards + trust tiering). **Artifact supply chain** — where adversarial dependencies enter ([270-agent-supply-chain-security](270-agent-supply-chain-security.md)): (a) **base model** (vendor + training-data + checkpoint signing); (b) **MCP servers** (signed manifests + AGNTCY trust tiers); (c) **marketplace skills** (SKILL.md signing + min-trust-tier); (d) **A2A peers** (Signed Agent Cards + OAuth scopes). The five attack surfaces × four artifact classes = the full attack matrix; the layered defense addresses each cell.

The layered defense composes **eight controls**: (1) **input/output classifiers** (Anthropic Constitutional, Microsoft Prompt Shield, custom — ensemble); (2) **instruction hierarchy** (training-time + system-prompt structure); (3) **spotlight prompting** (markup untrusted content as data); (4) **isolation** ([271-agent-isolation-patterns](271-agent-isolation-patterns.md), worktree + container + micro-VM + capability-MAC); (5) **bright-line gates on consequential actions** ([14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md)); (6) **supply-chain controls** (provenance, signing, trust-tiering, monitoring, revocation propagation); (7) **memory provenance + redactor** ([233-memory-scaling-for-agents](233-memory-scaling-for-agents.md)); (8) **audit-trail with tamper-evidence** (HIR observability + S3 Object Lock / Sigstore transparency log; [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md)). The eight controls are **multiplicative** — each catches independent attacks; combined they close most of the attack matrix at <2% false-positive rate.

The compliance overlay ([272-agent-compliance-and-audit](272-agent-compliance-and-audit.md)) adds the regulatory gate — EU AI Act, SOC 2 Type II, GDPR, HIPAA, NIST AI RMF, ISO/IEC 42001 — turning the engineering controls into auditable evidence. The compliance frameworks don't add new engineering primitives; they require **documentation, evidence collection, and external audit** of the same controls. This is why **compliance-by-construction** works: the engineering primitives map directly to regulatory controls.

Take this synthesis seriously and three things change. **First**, your security architecture has a **unified shape** — five attack surfaces × four artifact classes × eight controls — that lets you reason about coverage and gaps systematically. **Second**, your **production security checklist** ([Production Checklist](#production-checklist) below) gives a deterministic gate for "is this agent ready" — 30 items, each with engineering primitive + evidence source. **Third**, you adopt **assume-compromise design** — the eight controls drive prevention; observability + SRE + durability drive detection-and-response when prevention fails; isolation bounds the blast; recovery is fast.

## The unified threat model

### Attack surfaces × artifact classes matrix

|  | Direct input | Retrieved content | Tool outputs | Memory writes | Cross-agent messages |
|---|---|---|---|---|---|
| **Base model** | Sleeper agent triggered | Sleeper agent triggered | Sleeper agent triggered | Sleeper agent triggered | Sleeper agent triggered |
| **MCP servers** | n/a | n/a | Compromised server returns malicious | n/a | n/a |
| **Marketplace skills** | Skill prompt overrides on input | Skill prompt overrides on retrieved | Skill calls malicious tools | Skill writes poisoned memory | Skill sends malicious A2A |
| **A2A peers** | n/a | n/a | n/a | n/a | Compromised peer sends injection |

Each cell is a distinct attack pattern; the layered defense addresses each.

### Eight controls × attack surfaces coverage

|  | Direct input | Retrieved content | Tool outputs | Memory writes | Cross-agent messages |
|---|---|---|---|---|---|
| **1. Classifiers** | ✅ | ✅ | ✅ | ✅ at write | ✅ at receive |
| **2. Instruction hierarchy** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **3. Spotlight prompting** | n/a | ✅ | ✅ | ✅ | ✅ |
| **4. Isolation** | n/a | n/a (limits blast) | ✅ (blast) | ✅ (tier isolation) | ✅ (peer isolation) |
| **5. Bright-line gates** | ✅ on consequential | ✅ | ✅ | ✅ | ✅ |
| **6. Supply-chain controls** | n/a | provenance | signing, trust-tier | source attribution | Signed Agent Cards |
| **7. Memory provenance** | n/a | n/a | n/a | ✅ | n/a |
| **8. Audit + tamper-evidence** | ✅ logging | ✅ | ✅ | ✅ | ✅ |

Each cell with ✅ is a coverage point; gaps in the matrix are residual risks documented in the threat model.

## Threat-tier × isolation-tier mapping

| Threat tier | Examples | Recommended isolation |
|---|---|---|
| **Low (read-only)** | Research agent reading public web | Worktree + container with read-only network |
| **Medium (read + memory)** | Personal assistant with memory | Worktree + container + memory redactor |
| **High (code-executing)** | Agent runs untrusted code | Container + gVisor or micro-VM |
| **Critical (consequential actions)** | Email / financial / prod-config | Micro-VM + MAC + bright-line gates |
| **Cross-tenant SaaS** | Multi-customer platform | Micro-VM per tenant + strict capability-MAC |

## The 30-item production security checklist

### A. Prompt injection (8 items)

- [ ] **A1.** Input classifier deployed at user-input boundary (Anthropic / Microsoft / custom; ensemble of two)
- [ ] **A2.** Output classifier deployed at user-output boundary
- [ ] **A3.** Instruction hierarchy in all system prompts (system > developer > user > tool > retrieved-content)
- [ ] **A4.** Spotlight prompting on all retrieved content / tool outputs / memory reads
- [ ] **A5.** Encoding-based spotlight for adversarial-prone content (base64 / structured markup)
- [ ] **A6.** Attack-replay regression suite in CI; blocks merge on regression
- [ ] **A7.** Multi-turn anomaly detection for slow-build attacks
- [ ] **A8.** Cross-channel verifier on consequential outputs ([02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md))

### B. Supply chain (7 items)

- [ ] **B1.** SBOM-equivalent maintained automatically; lists base model + MCP servers + skills + A2A peers + dependencies
- [ ] **B2.** Base-model fingerprint verification; vendor-tier reputation check
- [ ] **B3.** MCP server signature verification mandatory before install
- [ ] **B4.** Marketplace skill signature verification + min-trust-tier policy
- [ ] **B5.** Vendored fallback for critical-path skills
- [ ] **B6.** Revocation routine ([252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md)) running daily; auto-rollback on critical CVE
- [ ] **B7.** Eval-suite regression on every dependency update

### C. Isolation (5 items)

- [ ] **C1.** Worktree per agent run for any agent that modifies files
- [ ] **C2.** Container isolation for code-executing agents (Docker/Podman with seccomp + cap_drop:ALL + read-only root)
- [ ] **C3.** Micro-VM (Firecracker / gVisor) for high-trust execution (untrusted-input code)
- [ ] **C4.** Network egress allowlist; default-deny
- [ ] **C5.** Cgroup resource limits (CPU + memory + disk + processes + network)

### D. Permission + bright-line (4 items)

- [ ] **D1.** Permission bridge ([07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md)) deployed; all tool calls gated
- [ ] **D2.** Bright-line gates ([14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md)) on consequential action kinds (send-email, transfer-money, delete-file, modify-prod-config, install-dep, exec-code, cross-agent-invoke)
- [ ] **D3.** Action-kind-to-OAuth-scope mapping documented and enforced
- [ ] **D4.** Per-routine bearer-token scoping; single-view secrets

### E. Memory (3 items)

- [ ] **E1.** Memory write-time redactor ([233-memory-scaling-for-agents](233-memory-scaling-for-agents.md))
- [ ] **E2.** Memory provenance metadata per item (source, trust level, write-time agent identity)
- [ ] **E3.** Memory tier isolation (untrusted memory cannot promote without explicit approval)

### F. Audit + observability (3 items)

- [ ] **F1.** OTel-shape spans with HIR-shape typed events ([264-agent-observability-stack-2026](264-agent-observability-stack-2026.md))
- [ ] **F2.** Audit log in tamper-evident storage (S3 Object Lock / Sigstore / hash-chain)
- [ ] **F3.** Per-regulation retention policy applied (HIPAA 6yr, EU AI Act lifecycle, SOC 2 multi-year)

### Compliance (frameworks-as-applicable)

- [ ] **Compliance:** Risk-tier classification documented (EU AI Act); SOC 2 audit scope defined; GDPR processor/controller role documented; HIPAA covered-entity status confirmed; NIST AI RMF cycle in place.

## How the security stack composes with operations + protocol + runtime

```
Layer 7 — Compliance      EU AI Act + SOC 2 + GDPR + HIPAA + NIST AI RMF + ISO 42001  ([272])
Layer 6 — Operations      Observability + Evaluation + Durability + SRE  ([268])
Layer 5 — Security        Prompt-injection defense + Supply chain + Isolation  ([273] — this file)
Layer 4 — Runtime         LangGraph / Agents SDK / AutoGen / ADK / Agent Teams  ([263])
Layer 3 — Protocol        MCP × A2A × AGNTCY × NATS+Tailscale × SKILL.md × Routines+Teams  ([258])
Layer 2 — Capability      Pretraining × TTC × Trajectory × Multi-agent × Verifier  ([225])
Layer 1 — Foundation      Permission Bridge, Daemon, Bright-line Gates, Cost Router
Layer 0 — Hardware        LLM provider, compute, storage
```

Each layer has a synthesis chapter; the layers are independent in implementation but composed in deployment.

## The closed feedback loop

```
[Compromise attempt]
        │
        ▼
[Layer 5 — Security controls block]  ←── Most attacks stopped here
        │
        ├── Some succeed
        ▼
[Layer 6 — Observability detects]
        │
        ▼
[Layer 6 — SRE runbook executes]
        │
        ▼
[Layer 6 — Durability supports rollback]
        │
        ▼
[Layer 5 — Isolation contains blast]
        │
        ▼
[Layer 7 — Compliance audit captured]
        │
        ▼
[Postmortem feeds back to Layer 5 controls]
        │
        ▼
[Layer 6 — Eval suite expanded with attack replay]
        │
        └─→ [Layer 5 — Classifier re-trained]
```

## Empirical results (table — May 2026 production agent security state)

| Capability | Adoption (production shops) |
|---|---|
| Input/output classifiers ensemble | High |
| Instruction hierarchy in prompts | High |
| Spotlight prompting | Mid (growing) |
| MCP signing verification | Mature shops |
| Marketplace skill min-trust-tier | Mature shops |
| Worktree isolation | High |
| Container isolation (Docker + seccomp + cap_drop) | High |
| Micro-VM isolation | Mature SaaS |
| Bright-line gates on consequential actions | High |
| Memory redactor | Lyra-class deployments |
| OTel + HIR observability | High |
| Tamper-evident audit log | Compliance-driven shops |
| EU AI Act high-risk compliance | Active prep (Aug 2026 deadline) |
| SOC 2 Type II AI extensions | Enterprise SaaS |
| Right-to-erasure cascading | EU-deployed agents |

## Variants and ablations

- **Tiered checklist by deployment class.** Personal agent: A1, A6, C1, D1, E1, F1. Enterprise SaaS: full 30. Regulated agent: full 30 + compliance overlay.
- **Compliance-by-construction.** Engineering primitives auto-comply; new primitives audited at design time.
- **Red-team rotation.** Quarterly chaos exercises ([49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md), [53-chaos-engineering-next-era](53-chaos-engineering-next-era.md)).
- **Bug bounty for agents.** External researchers test the platform.
- **Threat model per agent.** Each agent in `projects/` has its own STRIDE-shape threat model.
- **Continuous compliance scoring.** Compliance dashboard alongside SLO dashboard.
- **Vendor risk scoring.** Per-model-provider security posture tracked.
- **Insurance.** Cyber-insurance policies for agent platforms.

## Failure modes when the security stack is incomplete

- **Classifier without hierarchy:** subtle injections via lower-trust source.
- **Hierarchy without spotlight:** untrusted content read as instruction in some prompt structures.
- **Spotlight without isolation:** prompt-injection succeeds; no blast-radius limit.
- **Isolation without bright-line:** compromised agent makes legal-but-harmful action within isolation envelope.
- **Bright-line without supply chain:** trusted dependency compromised; gate respects but agent acts on.
- **Supply chain without observability:** compromise undetected.
- **Observability without retention:** compliance audit fails.
- **Retention without tamper-evidence:** audit trail discredited if tampered.

## When to use, when not

**Adopt the full 30-item checklist** for any production agent with paying customers, regulatory obligations, consequential tool grants, or high-trust requirements. The strongest cases are **enterprise SaaS**, **regulated agents** (healthcare, finance, government), **multi-tenant platforms**, and **agent-platform-as-a-service**.

**Adopt subset** for personal-use prototypes (top 8: A1, A6, B1, C1, C2, D1, E1, F1); for internal-only research tools (top 15); for single-user low-stakes deployments (top 12). **Never skip** A1 (input classifier), C1 (worktree), D1 (permission bridge), F1 (observability) — these are the minimum any agent that touches files or external systems should have.

## Implications for harness engineering

- **`harness_core/security/` package consolidates Layer 5.** Five sub-packages: `classifiers/`, `instruction_hierarchy/`, `spotlight/`, `isolation/`, `bright_lines/`.
- **Per-project security checklist scored.** [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md), [208-lyra-multi-hop-collaborative-apply-plan](208-lyra-multi-hop-collaborative-apply-plan.md), [209-argus-multi-hop-collaborative-apply-plan](209-argus-multi-hop-collaborative-apply-plan.md), [210-mentat-learn-collaborative-apply-plan](210-mentat-learn-collaborative-apply-plan.md), [211-cross-project-power-up-plan-with-tradeoffs](211-cross-project-power-up-plan-with-tradeoffs.md) — each project lists its score.
- **Compliance-as-code in CI.** Pass-rate measured on every commit.
- **Threat model per agent.** STRIDE-shape document.
- **Red-team routine.** [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md) — chaos exercises scheduled quarterly.
- **Permission bridge action-kind taxonomy.** [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — covers all consequential actions.
- **Memory redactor + provenance.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md), [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md).
- **Cross-channel verifier.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md).
- **HIR observability with provenance.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md), [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md), [24-observability-tracing](24-observability-tracing.md).
- **Tamper-evident audit log.** S3 Object Lock or Sigstore transparency log.
- **Right-to-erasure cascade.** Implementation across all storage layers.
- **Vendor-tier compliance metadata.** AGNTCY trust attributes ([255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md)) include compliance status.
- **Skill marketplace policy.** [257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md), [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md) — min-trust-tier per project.
- **MCP server policy.** [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md) — signature verification mandatory.
- **A2A peer policy.** [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md), [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md) — Signed Agent Cards required.
- **Bug bounty + responsible disclosure.** [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md) — process documented.

**One-line takeaway for harness designers.** **The 2026 agent security stack is a unified threat model (5 attack surfaces × 4 artifact classes), 8 layered defenses (classifiers + hierarchy + spotlight + isolation + bright-line gates + supply-chain controls + memory provenance + tamper-evident audit), 30-item production checklist, and a closed feedback loop (security blocks most attacks → observability detects the rest → SRE runbooks execute → durability supports rollback → isolation contains blast → compliance captures evidence → eval suite expands → classifiers re-trained); compliance-by-construction maps engineering primitives to regulatory frameworks (EU AI Act + SOC 2 + GDPR + HIPAA + NIST AI RMF + ISO 42001) so the work is mapping + evidence + audit, not new infrastructure; the checklist gives "is this agent ready for production" a deterministic yes/no answer.**
