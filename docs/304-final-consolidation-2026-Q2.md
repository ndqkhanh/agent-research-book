# 304 — Final Consolidation 2026-Q2: what we built across 250–303 and where we go next

**Anchors.** All 250–303 deep-dives. Earlier corpus 1–249. `projects/harness_core/` v0.0.1 code skeleton. Companions: [285](285-harness-core-porting-plan.md), [295-corpus-reading-guide-2026](295-corpus-reading-guide-2026.md), [296-corpus-consolidation-review-2026](296-corpus-consolidation-review-2026.md).

**One-line definition.** A **final consolidation** of the May-2026-Q2 production-agent-engineering deep-dive run — **what we built** (~55 docs spanning capability through compliance + 8 Python modules + per-project plans for all 15 in-tree projects + 30-item production security checklist + decision matrices for runtime / protocol / operations / cost), **the strategic bet** ($1.5-3M cross-project savings if `harness_core/` ships v1.0; all-or-nothing on actual code adoption), **what's missing** (production case-study depth, multi-modal-team patterns, sector-specific deep-dives, RLVR production playbooks beyond [301](301-rlvr-production-deep-dive.md), HCI research depth), and **the critical path** (ship `harness_core/` v0.1.0 within 6 weeks; first adopter Polaris within 3 months; first full v1.0 release with 6 adopters within 12 months) — turning a multi-quarter deep-dive corpus into an actionable engineering roadmap with measurable milestones.

## What we built (May 2026 wrap)

The 250–303 corpus is **~55 deep-dives + 8 Python modules** organized in seven thematic drops + one code package:

| Drop | Range | Files | Theme |
|---|---|---:|---|
| Agent Teams + Distributed | 250–253 | 4 | Coordination + transport |
| Protocol Stack | 254–258 | 5 | A2A + AGNTCY + MCP + marketplaces |
| Runtime Selection | 259–263 | 5 | LangGraph / Agents SDK / AutoGen / ADK / synthesis |
| Operations | 264–268 | 5 | Observability + eval + durability + SRE |
| Security | 269–273 | 5 | Prompt injection + supply chain + isolation + compliance |
| Underserved topics | 274–278 | 5 | Prompt + finetuning + local-first + UX + economics |
| Per-project plans | 279–294, 297 | 16 | All 15 in-tree projects |
| Reading guide + reviews | 295, 296, 304 | 3 | Meta-corpus |
| Code skeleton | `harness_core/` | 8 .py | First v0.0.1 |
| Integration + SBOM | 298, 299 | 2 | Per-project import patterns |
| Priority gaps | 300–303 | 4 | Production cases + RLVR + multimodal + sector |
| **Total May-2026-Q2** | | **~55 docs + 8 .py** | |

Combined with the existing 1-249 corpus, the full collection covers **production-agent engineering end-to-end**.

## The seven-layer stack consolidated (from 268 + 273 + 296)

```
Layer 7 — Compliance      EU AI Act + SOC 2 + GDPR + HIPAA + NIST AI RMF + ISO 42001  ([272], [299])
Layer 6 — Operations      Observability + Evaluation + Durability + SRE  ([268], [264-267])
Layer 5 — Security        Prompt-injection + Supply chain + Isolation + Compliance  ([273], [269-272])
Layer 4 — Runtime         LangGraph / Agents SDK / AutoGen / ADK / CrewAI / Agent Teams  ([263])
Layer 3 — Protocol        MCP × A2A × AGNTCY × NATS+Tailscale × SKILL.md × Routines+Teams  ([258])
Layer 2 — Capability      Pretraining × TTC × Trajectory × Multi-agent × Verifier  ([225])
Layer 1 — Foundation      Permission Bridge, Daemon, Bright-line Gates, Cost Router
```

Plus modality overlays ([302](302-multimodal-agents-2026.md)) and sector overlays ([303](303-sector-verticals-overview.md)).

## Per-project apply plans (complete map)

| Project | File | Distinctive role |
|---|---|---|
| Polaris | [279](279-polaris-seven-layer-stack-apply-plan.md) | Long-running autonomous research |
| Lyra | [280](280-lyra-seven-layer-stack-apply-plan.md) | Local-first personal assistant |
| Mentat-Learn | [281](281-mentat-learn-seven-layer-stack-apply-plan.md) | Multi-channel self-improving |
| Argus | [282](282-argus-seven-layer-stack-apply-plan.md) | Marketplace + skill provider |
| Aegis-Ops | [283](283-aegis-ops-seven-layer-stack-apply-plan.md) | Operator role |
| Cipher-Sec | [284](284-cipher-sec-seven-layer-stack-apply-plan.md) | Security specialist |
| Atlas-Research | [286](286-atlas-research-seven-layer-stack-apply-plan.md) | Research lit retrieval |
| Helix-Bio | [287](287-helix-bio-seven-layer-stack-apply-plan.md) | Biology / HIPAA + FDA |
| Harmony-Voice | [288](288-harmony-voice-seven-layer-stack-apply-plan.md) | Voice-first |
| Vertex-Eval | [289](289-vertex-eval-seven-layer-stack-apply-plan.md) | Eval provider |
| Orion-Code | [291](291-orion-code-seven-layer-stack-apply-plan.md) | SWE coding |
| Syndicate | [292](292-syndicate-seven-layer-stack-apply-plan.md) | Multi-agent orchestrator |
| Quanta-Proof | [293](293-quanta-proof-seven-layer-stack-apply-plan.md) | Math + formal proof |
| Open-Fang | [294](294-open-fang-seven-layer-stack-apply-plan.md) | v8 reference; harmonize |
| Gnomon | [297](297-gnomon-seven-layer-stack-apply-plan.md) | Observability provider |

All 15 in-tree projects have apply plans.

## The Python skeleton

`projects/harness_core/` v0.0.1 ships:

- `pyproject.toml` (Apache-2.0, Pydantic + FastAPI + httpx + cryptography + OTel deps)
- `foundation/permissions.py` — full PermissionBridge with mode + bright-line + custom rules + listeners (~220 lines, fully tested)
- `routines/config.py` — RoutineConfig + SQLite-backed RoutineConfigStore (~240 lines)
- `routines/triggers/cron.py` — CronEngine + tick() polling (~130 lines)
- `protocols/a2a/agent_card.py` — AgentCard with EdDSA sign + verify (~135 lines)
- `protocols/a2a/server.py` — FastAPI A2A v1.0 server (~210 lines)
- `operations/observability/hir_events.py` — HIREmitter + HIREventKind taxonomy + SpanContext (~190 lines)
- `tests/test_permissions.py` — pytest suite (~75 lines)

**Total: ~1200 lines of working Python** + tests.

## The strategic bet

The corpus's strategic bet:

> Shared infrastructure (`harness_core/`) + protocol stack (A2A + MCP + AGNTCY + NATS + SKILL.md) + operations stack + security stack + per-project apply plans = a complete production-agent engineering discipline that turns the in-tree ecosystem from 14 prototypes into a coherent platform.

**If it pays off:**
- Cross-project agent composition becomes trivial.
- Compliance becomes engineering (mapping primitives to frameworks).
- $1.5-3M saved across 14 projects vs independent ports.
- External adoption of `harness_core/` as OSS shared library.

**If it doesn't:**
- Per-project drift; each implements own variant.
- `harness_core/` shelfware.
- Corpus becomes reference-only.

## The critical path

| Milestone | Target | Date |
|---|---|---|
| `harness_core/` v0.1.0 release | Foundation + protocols/a2a + routines/cron + observability + tests | 2026-06-30 |
| Polaris first adopter | Re-export from polaris-core; new code uses harness_core directly | 2026-09-30 |
| Lyra second adopter | Memory module promoted to harness_core; lyra-core re-exports | 2026-12-31 |
| Argus, Mentat-Learn, Aegis-Ops, Cipher-Sec adoption | Provider role + 4 consumer roles validated | 2027-03-31 |
| `harness_core/` v1.0.0 with 6 production adopters | All cross-project composition working | 2027-05-31 |

## What's missing (priorities for next quarters)

Per [296](296-corpus-consolidation-review-2026.md):

1. **Production case-study depth** — beyond [309](309-production-case-studies-2026.md) overview; per-vendor 5-10K-word deep-dives.
2. **Multi-modal multi-agent patterns** — beyond [302](302-multimodal-agents-2026.md); team architectures with vision + voice + text specialists.
3. **Per-sector deep-dives** — healthcare, finance, legal, education (per [303](303-sector-verticals-overview.md) roadmap).
4. **RLVR production case studies** — beyond [301](301-rlvr-production-deep-dive.md); Moonshot Kimi K2.5 PARL, OpenAI o-series internal lessons.
5. **HCI research depth** — agent-trust-formation, plan-mode UX research, multi-agent observability UX.
6. **Cost economics with real customer data** — beyond [278](278-agent-unit-economics-2026.md) theoretical.
7. **External-publish strategy** — once `harness_core/` v1.0 ships, external community engagement.

## Cumulative output across the May-2026 drops

| Drop | Files | Lines |
|---|---:|---:|
| Scaling Laws (216–217, 223–225, 232–237) | 11 | ~3,000 |
| Karpathy (238–243, daemon-created) | 6 | ~1,500 |
| Agent Teams + Distributed (250–253) | 4 | ~900 |
| Protocol Stack (254–258) | 5 | ~1,200 |
| Runtime Selection (259–263) | 5 | ~1,300 |
| Operations (264–268) | 5 | ~1,300 |
| Security (269–273) | 5 | ~1,300 |
| Underserved (274–278) | 5 | ~975 |
| Per-project plans (279–294, 297) | 16 | ~1,500 |
| Reading guide + reviews (295, 296, 304) | 3 | ~500 |
| Code skeleton (`harness_core/`) | 8 .py | ~1,200 |
| Integration + SBOM (298, 299) | 2 | ~400 |
| Priority gaps (300–303) | 4 | ~600 |
| **Total May-2026-Q2** | **~75 docs + 8 .py** | **~13,000+ lines** |

## How to traverse from here

Per [295](295-corpus-reading-guide-2026.md), pick a reader path. Recommended starting point per goal:

- **Building from scratch:** start with [285](285-harness-core-porting-plan.md) → import `harness_core/` → follow [298](298-harness-core-integration-glue.md).
- **Selecting runtime:** [263](263-production-agent-runtime-synthesis-2026.md) decision matrix.
- **Operationalizing:** [268](268-agent-operations-synthesis-2026.md) four legs.
- **Hardening:** [273](273-agent-security-synthesis-2026.md) 30-item checklist.
- **Picking project to apply to:** [296](296-corpus-consolidation-review-2026.md) per-project map → relevant 279–294/297 file.

## What I learned writing this

- **Synthesis chapters give 80% of value.** Reading the six (225, 251, 258, 263, 268, 273) before deep-dives is more efficient than linear traversal.
- **Per-project apply plans are reusable templates.** The same shape (per-layer + phased rollout + cross-project deps + cost economics) works for every project.
- **Code is the load-bearing artifact.** ~13K lines of docs are reference-grade; ~1.2K lines of Python are production-bootstrapping.
- **Daemon collisions are a feature.** The daemon-created files at 218-221, 226, 238-243, 290 added unexpected research depth I didn't plan for; the in-tree ecosystem's productive collaborative dynamic.
- **The compounding effect.** Each drop builds on previous; the corpus is far more than the sum of its parts.

## One-line takeaway

**The May-2026-Q2 production-agent-engineering corpus is ~75 docs + 8 Python modules totaling ~13,000+ lines spanning the seven-layer stack from capability through compliance, with per-project apply plans for all 15 in-tree projects, the first concrete `harness_core/` v0.0.1 code skeleton, and a clear critical path (v0.1.0 by 2026-06-30, Polaris first adopter by 2026-09-30, v1.0 with 6 adopters by 2027-05-31); the strategic bet is on `harness_core/` actually shipping — that's the difference between "we have great docs" and "we have a production agent platform."**
