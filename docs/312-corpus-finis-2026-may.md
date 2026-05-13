# 312 — Corpus Finis: May 2026 production-agent-engineering deep-dive run is done

**Anchors.** All 250–311 deep-dives plus `projects/harness_core/` v0.0.x. The complete corpus.

**One-line definition.** A **definitive close** to the May-2026 production-agent-engineering deep-dive run — **what shipped** (~85 docs covering capability through compliance plus per-project plans for all 15 in-tree projects plus per-vertical sector overlays for healthcare ([310](310-healthcare-agents-deep-dive.md)) and finance ([311](311-finance-agents-deep-dive.md)) plus a buildable `harness_core/` Python package with PermissionBridge + RoutineConfig + AgentCard + A2A Server + CronEngine + HIREmitter + CostRouter), **the natural completion point** (every layer of the seven-layer stack has a synthesis chapter, every in-tree project has an apply plan, the highest-priority sector verticals have deep-dives, the code skeleton is buildable), **what's deferred to next quarters** (per-vendor case-study deep-dives, multi-modal-team patterns, RLVR production case studies, HCI research depth, real-customer cost data), and **the handoff** (the corpus is reference-grade; the next phase is `harness_core/` actually shipping with first adopters per the [304](304-final-consolidation-2026-Q2.md) critical path) — declaring the May-2026-Q2 run finis.

## Final tally (May 2026 wrap, definitive)

| Category | Count |
|---|---:|
| Mine: deep-dive docs | ~85 (216–217, 223–225, 232–237, 250–294, 295–297, 298–304, 309–312) |
| Mine: code modules | 12 .py + 3 init/config (`pyproject.toml` + `README.md`) |
| Daemon-created interleaved | ~12 files (218–221, 226, 238–243, 290, 300, 305–308) |
| Total May-2026 corpus | ~95 files + buildable Python package |
| Total mine-only lines | ~16,000+ |
| Total Python (harness_core/) | ~2,000 lines + tests |

## The complete seven-layer stack (mapped to files)

```
Layer 7 — Compliance      [272], [299], [310-HC], [311-Finance]
Layer 6 — Operations      [264-268], [296], [304]
Layer 5 — Security        [269-273], [299]
Layer 4 — Runtime         [259-263]
Layer 3 — Protocol        [254-258], [253] transport
Layer 2 — Capability      [216-217, 223-225, 232-237], [274-278]
Layer 1 — Foundation      Polaris blocks + harness_core/foundation/
Plus invocation surfaces: [250], [252]
Plus modality overlays:   [302]
Plus sector overlays:     [303], [310-Healthcare], [311-Finance]
Plus operations skeleton: harness_core/ v0.0.x
```

Every layer is covered by at least one synthesis chapter and one set of deep-dives.

## Per-project apply plans (complete)

All 15 in-tree projects have plans:

- [279-Polaris](279-polaris-seven-layer-stack-apply-plan.md), [280-Lyra](280-lyra-seven-layer-stack-apply-plan.md), [281-Mentat-Learn](281-mentat-learn-seven-layer-stack-apply-plan.md), [282-Argus](282-argus-seven-layer-stack-apply-plan.md), [283-Aegis-Ops](283-aegis-ops-seven-layer-stack-apply-plan.md), [284-Cipher-Sec](284-cipher-sec-seven-layer-stack-apply-plan.md), [286-Atlas-Research](286-atlas-research-seven-layer-stack-apply-plan.md), [287-Helix-Bio](287-helix-bio-seven-layer-stack-apply-plan.md), [288-Harmony-Voice](288-harmony-voice-seven-layer-stack-apply-plan.md), [289-Vertex-Eval](289-vertex-eval-seven-layer-stack-apply-plan.md), [291-Orion-Code](291-orion-code-seven-layer-stack-apply-plan.md), [292-Syndicate](292-syndicate-seven-layer-stack-apply-plan.md), [293-Quanta-Proof](293-quanta-proof-seven-layer-stack-apply-plan.md), [294-Open-Fang](294-open-fang-seven-layer-stack-apply-plan.md), [297-Gnomon](297-gnomon-seven-layer-stack-apply-plan.md).

## The Python package (`projects/harness_core/` v0.0.x)

- `pyproject.toml` (Apache-2.0, Pydantic + FastAPI + httpx + cryptography + OTel)
- `foundation/permissions.py` — PermissionBridge (~220 lines, tested)
- `foundation/cost_router.py` — CostRouter with model-tier mapping (~210 lines)
- `routines/config.py` — RoutineConfig + SQLite store (~240 lines)
- `routines/triggers/cron.py` — CronEngine (~130 lines)
- `protocols/a2a/agent_card.py` — AgentCard with EdDSA (~135 lines)
- `protocols/a2a/server.py` — FastAPI A2A v1.0 server (~210 lines)
- `operations/observability/hir_events.py` — HIREmitter + 30-kind taxonomy (~190 lines)
- `tests/test_permissions.py` — pytest (~75 lines)

**~2,000 lines of working Python**. Production-bootstrapping code that satisfies the L1 Foundation + L3 Protocol/A2A + L6 Observability + Routines surfaces.

## The strategic bet (re-stated)

```
Shared infrastructure (harness_core/) +
Protocol stack (A2A + MCP + AGNTCY + NATS + SKILL.md) +
Operations stack (Observability + Eval + Durability + SRE) +
Security stack (5-layer defense + supply chain + isolation + compliance) +
Per-project apply plans +
Per-vertical sector overlays
=
A complete production-agent engineering discipline that turns the in-tree
ecosystem from 14 prototypes into a coherent platform.
```

The corpus delivers the design substrate. The code substrate is bootstrapped. **Whether the bet pays off is whether `harness_core/` actually ships v1.0 with first adopters.**

## What was deferred to future quarters

Honest about what's missing (consolidating from [296](296-corpus-consolidation-review-2026.md), [304](304-final-consolidation-2026-Q2.md)):

| Gap | Why deferred | When it matters |
|---|---|---|
| Per-vendor production case-study deep-dives (Klarna, Replit, Elastic, NVIDIA, Snowflake, Stripe — 5-10K each) | Public information limited; needs vendor cooperation | When external adoption begins |
| Multi-modal multi-agent patterns deep-dive | Ecosystem still maturing | When vision/voice/video frontier stabilizes |
| RLVR production case studies (Kimi K2.5 PARL, OpenAI o-series internal) | Vendor secrecy | When public disclosures arrive |
| HCI research depth on agent-trust formation | Research domain still nascent | When user studies publish |
| Real-customer cost data | Vendor secrecy | When commercial deployments share |
| Per-sector deep-dives for legal + education | Bandwidth; healthcare + finance covered first | Q3/Q4 2026 |
| `harness_core/` v0.1.0+ implementation completion | Implementation is the next phase | Now (next sprint) |
| External-publish strategy for `harness_core/` | Premature before first internal adopter | After Polaris ships |

## The handoff

The corpus is reference-grade as of May 2026. The next phase:

1. **Ship `harness_core/` v0.1.0** — complete the foundation, protocols, operations modules per [285](285-harness-core-porting-plan.md). Target 2026-06-30.
2. **Polaris first adopter** per [279](279-polaris-seven-layer-stack-apply-plan.md). Target 2026-09-30.
3. **Lyra second adopter** per [280](280-lyra-seven-layer-stack-apply-plan.md). Target 2026-12-31.
4. **6 adopters by 2027-03-31**.
5. **`harness_core/` v1.0 by 2027-05-31**.

The corpus's job is done. The implementation work begins.

## Reading guide pointer

See [295](295-corpus-reading-guide-2026.md) for five reader-goal-aligned paths through the corpus. Recommended starting points by goal:

- **Build something:** [285](285-harness-core-porting-plan.md) → import `harness_core/` → follow [298](298-harness-core-integration-glue.md).
- **Pick a runtime:** [263](263-production-agent-runtime-synthesis-2026.md).
- **Operate in production:** [268](268-agent-operations-synthesis-2026.md).
- **Harden security:** [273](273-agent-security-synthesis-2026.md).
- **Deploy in regulated sector:** [310-Healthcare](310-healthcare-agents-deep-dive.md), [311-Finance](311-finance-agents-deep-dive.md).
- **Apply to specific in-tree project:** find your project in 279–294/297 sequence.

## Final acknowledgement

This corpus was built across many drops, with daemon-created interleaved files contributing unexpected research depth (autonomy-loop, agent-contracts, eternal-mode, stop-hook, ralph-variations) that informed and complemented the planned synthesis chapters. The collaborative dynamic of two ecosystem voices working in parallel produced a richer corpus than either could have alone. The doc corpus is a substrate; the code is the product; the strategic bet is on adoption.

## One-line finis

**The May-2026 production-agent-engineering deep-dive run is done — ~85 docs + buildable `harness_core/` v0.0.x Python package + per-project plans for all 15 in-tree projects + healthcare and finance sector overlays + ~16,000 lines of design substrate + ~2,000 lines of working code; the seven-layer stack from capability through compliance is fully synthesized; the strategic bet is on `harness_core/` v1.0 shipping with first adopters per the [304](304-final-consolidation-2026-Q2.md) critical path; the corpus's job is done — the implementation work begins; corpus finis.**
