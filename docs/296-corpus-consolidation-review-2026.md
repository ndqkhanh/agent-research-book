# 296 — Corpus Consolidation Review 2026: what we built, what's missing, what's next

**Anchors.** All 250-295 deep-dives plus earlier corpus. Companion: [README.md](README.md), [285-harness-core-porting-plan](285-harness-core-porting-plan.md), [295-corpus-reading-guide-2026](295-corpus-reading-guide-2026.md).

**One-line definition.** A **meta-review** of the May-2026 production-agent-engineering deep-dive corpus — **what we built** (47-file production-agent toolkit covering capability through compliance, plus per-project apply plans for all 14 in-tree projects, plus the harness_core/ Python skeleton), **what's load-bearing** (six synthesis chapters that give 80% of the corpus's value alone), **what's missing** (deeper finetuning recipes, real production deployment case studies, formal-verification integration depth, multi-modal agent depth), and **what's next** (the actual `harness_core/` v1.0 implementation, external-publish to broader community, and the next 50 deep-dives extending into agent training + agent UX + sector-specific verticals).

## What we built (May 2026 wrap)

The corpus from 250 onward is a **production-agent engineering toolkit**:

| Range | Theme | Files | What it answers |
|---|---|---:|---|
| 216-217, 223-225, 232-237 | Capability scaling | 11 | "How do I make my agent better?" |
| 238-243 | Karpathy Software 3.0 | 6 | (daemon-created) "What's the post-2025 vocabulary?" |
| 250-253 | Agent Teams + Distributed | 4 | "How do agents coordinate?" |
| 254-258 | Protocol stack | 5 | "How do agents talk?" |
| 259-263 | Runtime selection | 5 | "Which framework?" |
| 264-268 | Operations | 5 | "How do I run in production?" |
| 269-273 | Security | 5 | "How do I keep adversaries out?" |
| 274-278 | Underserved topics | 5 | "Prompt engineering, finetuning, local-first, UX, economics" |
| 279-294 | Per-project apply plans | 16 | "How does this apply to project X?" |
| 285 | `harness_core/` porting plan + Python skeleton | 1 + code | "What do I import?" |
| 295-296 | Reading guide + this review | 2 | "How do I traverse this?" |

**Total: 65+ files in the May-2026 production-agent drop.** Combined with the existing 1-249 corpus, the full collection is ~10,000 pages.

## The seven-layer stack consolidated

```
Layer 7 — Compliance      EU AI Act + SOC 2 + GDPR + HIPAA + NIST AI RMF + ISO 42001  ([272])
Layer 6 — Operations      Observability + Evaluation + Durability + SRE  ([268])
Layer 5 — Security        Prompt-injection + Supply chain + Isolation + Compliance  ([273])
Layer 4 — Runtime         LangGraph / Agents SDK / AutoGen / ADK / CrewAI / Agent Teams  ([263])
Layer 3 — Protocol        MCP × A2A × AGNTCY × NATS+Tailscale × SKILL.md × Routines+Teams  ([258])
Layer 2 — Capability      Pretraining × TTC × Trajectory × Multi-agent × Verifier  ([225])
Layer 1 — Foundation      Permission Bridge, Daemon, Bright-line Gates, Cost Router  (existing blocks)
```

Each layer has a synthesis chapter; the layers are independent in implementation but composed in deployment. The **closed feedback loop** (security blocks attacks → observability detects rest → SRE runbooks execute → durability supports rollback → isolation contains blast → compliance captures evidence → eval suite expands → classifiers re-trained) closes the production-quality cycle.

## Per-project map

All 15 in-tree projects now have apply plans:

| Project | File | Role |
|---|---|---|
| Polaris | [279](279-polaris-seven-layer-stack-apply-plan.md) | Long-running autonomous research |
| Lyra | [280](280-lyra-seven-layer-stack-apply-plan.md) | Local-first personal assistant |
| Mentat-Learn | [281](281-mentat-learn-seven-layer-stack-apply-plan.md) | Multi-channel self-improving |
| Argus | [282](282-argus-seven-layer-stack-apply-plan.md) | Marketplace + skill-curator provider |
| Aegis-Ops | [283](283-aegis-ops-seven-layer-stack-apply-plan.md) | Operator role |
| Cipher-Sec | [284](284-cipher-sec-seven-layer-stack-apply-plan.md) | Security specialist |
| Atlas-Research | [286](286-atlas-research-seven-layer-stack-apply-plan.md) | Research-lit retrieval |
| Helix-Bio | [287](287-helix-bio-seven-layer-stack-apply-plan.md) | Biology / bioinformatics |
| Harmony-Voice | [288](288-harmony-voice-seven-layer-stack-apply-plan.md) | Voice-first |
| Vertex-Eval | [289](289-vertex-eval-seven-layer-stack-apply-plan.md) | Eval provider |
| Gnomon | [297](297-gnomon-seven-layer-stack-apply-plan.md) | Observability provider |
| Orion-Code | [291](291-orion-code-seven-layer-stack-apply-plan.md) | SWE coding |
| Syndicate | [292](292-syndicate-seven-layer-stack-apply-plan.md) | Multi-agent orchestration substrate |
| Quanta-Proof | [293](293-quanta-proof-seven-layer-stack-apply-plan.md) | Math / formal proof |
| Open-Fang | [294](294-open-fang-seven-layer-stack-apply-plan.md) | Most-mature research agent (v8 reference) |

## What's load-bearing (read these if nothing else)

Six synthesis chapters cover ~80% of the corpus's value:

1. **[225-agent-era-scaling-synthesis](225-agent-era-scaling-synthesis.md)** — capability axes
2. **[251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md)** — multi-agent
3. **[258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md)** — protocols
4. **[263-production-agent-runtime-synthesis-2026](263-production-agent-runtime-synthesis-2026.md)** — runtimes
5. **[268-agent-operations-synthesis-2026](268-agent-operations-synthesis-2026.md)** — operations
6. **[273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md)** — security

Plus this consolidation review (296) for the meta-view.

## What's missing

Despite the corpus's breadth, gaps remain:

### 1. Deeper finetuning recipes
[275-agent-finetuning-2026](275-agent-finetuning-2026.md) summarizes Tülu-3 + R1; **a multi-file deep-dive** on production RLVR (data curation, GRPO ablations, reward hacking detection, distillation studies) is missing.

### 2. Real production deployment case studies
The corpus is design-grounded but light on **post-mortems from actual deployments**. Klarna / Replit / Elastic / NVIDIA / Snowflake have public LangGraph deployments; deeper case-study coverage would strengthen practitioner intuition.

### 3. Formal-verification integration depth
[293-quanta-proof](293-quanta-proof-seven-layer-stack-apply-plan.md) sketches Lean / Coq integration; **dedicated coverage of formal-verifier-backed agents** (AlphaProof, AlphaProver) deserves its own synthesis.

### 4. Multi-modal agent depth
The corpus is text-centric. **Vision agents, audio agents, video agents, multi-modal multi-agent teams** need deeper coverage as Claude / GPT / Gemini multi-modal capabilities mature.

### 5. RL on agent traces production case studies
[232-rl-on-reasoning-traces-scaling](232-rl-on-reasoning-traces-scaling.md) covers DeepSeek-R1; production deployment lessons (Moonshot Kimi K2.5 PARL, OpenAI o-series production) deserve explicit deep-dives.

### 6. Cost economics with real production data
[278-agent-unit-economics-2026](278-agent-unit-economics-2026.md) is theoretical; **per-vertical cost case studies** (customer-service, code-review, research, voice) with real-customer numbers would strengthen.

### 7. Sector-specific deep-dives
[149-sector-use-case-catalog](149-sector-use-case-catalog.md) lists sectors; **dedicated multi-file drops on healthcare-agents, finance-agents, legal-agents, education-agents** with sector-specific compliance details would extend.

### 8. Agent UI/UX research depth
[277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md) summarizes; **HCI research on agent-trust formation** deserves deeper treatment.

## What's next (priorities for the next 50 deep-dives)

| Priority | Drop | Target files | Why |
|---|---|---|---|
| **P0** | `harness_core/` v0.1.0 release | code, not docs | Ship the Python; corpus is design |
| **P1** | Production case studies | 5-10 files | Practitioner intuition |
| **P2** | Sector verticals (healthcare / finance / legal) | 5-10 files | Sector-specific compliance |
| **P3** | RLVR production deep-dives | 3-5 files | Deeper than 232/275 |
| **P4** | Multi-modal agents | 5-7 files | Field is moving here |
| **P5** | Formal verification depth | 3-5 files | AlphaProof-class |

## What we'd do differently

Reflecting on the May-2026 drop:

- **Earlier consolidation.** Six synthesis chapters could have been written first, with deep-dives slotted in.
- **More worked examples.** Each pattern needs 2-3 concrete code examples beyond pseudocode.
- **Real customer numbers.** Cost economics theoretical; needs production data.
- **Earlier code skeleton.** [285](285-harness-core-porting-plan.md) should have shipped earlier with the synthesis chapters.

## The strategic bet of the corpus

The bet: **shared infrastructure (`harness_core/`) + protocol stack (A2A + MCP + AGNTCY + NATS + SKILL.md) + operations stack (observability + eval + durability + SRE) + security stack (5-layer defense + supply chain + isolation + compliance) + per-project apply plans = a complete production-agent engineering discipline that turns the in-tree ecosystem from 14 independent prototypes into a coherent platform**.

If the bet pays off:
- **In-tree projects ship faster** (12-14 projects share infrastructure).
- **Cross-project agent composition** becomes trivial via A2A.
- **Compliance becomes engineering** (mapping primitives to frameworks).
- **External adoption** of `harness_core/` as the OSS shared library.

If the bet doesn't pay off:
- Per-project drift (each implements their own variant).
- `harness_core/` becomes shelfware.
- The corpus becomes a reference but not used.

The deciding factor is **whether `harness_core/` v1.0 actually ships** with the first 2-3 adopters validating the promotion. [285](285-harness-core-porting-plan.md) outlines the 12-month plan; this is the critical path.

## Cross-references back to the broader corpus

- [99-papers-deep-dive-synthesis](99-papers-deep-dive-synthesis.md) — earlier (file 99) cross-paper synthesis.
- [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md) — memory + skills synthesis.
- [193-recursive-world-organizations-synthesis](193-recursive-world-organizations-synthesis.md) — recursive multi-agent + world models + AI orgs synthesis.
- [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md) — collaborative AI canon.
- [211-cross-project-power-up-plan-with-tradeoffs](211-cross-project-power-up-plan-with-tradeoffs.md) — strategic frame.

## One-line takeaway

**The May-2026 production-agent engineering corpus is 65+ files (~10K pages) covering capability through compliance, with six synthesis chapters that give 80% of the value alone, per-project apply plans for all 15 in-tree projects, and a `harness_core/` Python skeleton; the strategic bet is that shared infrastructure + protocol stack + operations stack + security stack + per-project plans turn the ecosystem from 14 prototypes into a coherent production-grade platform; the deciding factor is whether `harness_core/` v1.0 actually ships with first adopters validating the promotion — that's the critical path from corpus to production.**
