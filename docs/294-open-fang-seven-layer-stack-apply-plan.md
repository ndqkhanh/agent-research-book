# 294 — Open-Fang × Seven-Layer Stack Apply Plan 2026

**Anchors.** Open-Fang — autonomous AI research agent specialized for AI / Agent / Harness Engineering literature ([projects/open-fang](../projects/open-fang/)). Per its v8 README: 612 tests green, DAG Teams orchestration, 3-tier progressive-disclosure memory, SQLite+FTS5 KB with weighted citation graph + multi-hop synthesis, hardened 5-tier verifier (lexical → mutation → LLM-judge → executable → cross-channel), 9-specialist research cohort, 7-tool MCP server, agentskills.io-compliant skills, claude-mem tool-name parity, Atropos-compatible trajectory export.

**One-line definition.** A **per-layer apply plan** for Open-Fang — the **most-mature in-tree research agent** — emphasizing **harmonization with the seven-layer stack** (Open-Fang already implements many primitives independently; the apply plan **aligns naming + interfaces** with `harness_core/` rather than re-implementing), **agentskills.io schema alignment** ([257-agent-skill-marketplace-landscape](257-agent-skill-marketplace-landscape.md)), and **continued evolution** of its 5-tier verifier + 9-specialist cohort patterns as **reference implementations** the rest of the ecosystem can learn from — staged across four 90-day phases focused on integration rather than rebuild.

## Open-Fang shape

Open-Fang is the **most-mature** in-tree project as of May 2026 — v8 with 612 green tests, hardened multi-tier verifier, opt-in weekly feed cron, multica runtime adapter, HAFC-lite primitive-level failure attribution, ReBalance confidence-steered halt, Tier 4.5 symbolic claim-number verifier. Many of the seven-layer-stack primitives are already implemented in open-fang's own codebase. The apply plan is **harmonization** with `harness_core/` shared library + alignment with the broader ecosystem rather than ground-up adoption.

## Per-layer plan

### L1 Foundation — harmonize, don't rebuild
Open-Fang's own foundation (Permission Bridge equivalent, Daemon, Bright-line patterns) maps to `harness_core/foundation/`. **Action:** thin adapter layer; keep open-fang internals; expose harness_core-compatible interfaces.

### L2 Capability — already implemented
Multi-hop synthesis ([200](200-graph-grounded-multi-hop-retrieval.md)), citation-graph weighted walks, 3-tier memory (procedural / episodic / semantic), 5-tier verifier all map to L2 capability axes. Already strong.

### L3 Protocol
- **MCP**: open-fang's 7-tool MCP server already production-grade; align with `harness_core/protocols/mcp/` interfaces.
- **A2A**: new — expose research capabilities as A2A endpoints for Polaris / Mentat-Learn consumption.
- **AGNTCY**: publish OASF manifest; trust tier `audited`.
- **SKILL.md**: agentskills.io schema alignment is excellent; integrate with argus marketplace.
- **Routines**: opt-in weekly feed cron is already a Routine pattern.

### L4 Runtime — already custom; consider hybrid
Open-Fang has its own DAG Teams orchestration (Phase-1 LLM planner → Phase-2 deterministic scheduler). **Action:** keep custom runtime as the canonical research-agent pattern; expose A2A endpoint for cross-runtime composition.

### L5 Security — already strong; align
- 5-tier verifier ([Tier 1: lexical, Tier 2: mutation, Tier 3: LLM-judge, Tier 4: executable, Tier 4.5: symbolic claim-number, Tier 5: cross-channel]) is strongest in-tree. **Action:** publish as reference for other projects.
- Security probes + chaos hooks + red-team already in v2.5+.
- Subprocess isolation per specialist via `IsolatedSupervisor`.

### L6 Operations — already strong
- Observability via Gnomon span integration (v4.4).
- Eval: 50-brief decontaminated corpus (v2.7).
- HAFC-lite primitive-level failure attribution (v6.0) — **maps to MAST classification**.
- Trajectory export Atropos-compatible (v3.3) — interop with broader ecosystem.

### L7 Compliance — research-tier
**EU AI Act**: limited / minimal risk for research; transparency obligation. **Citation provenance** is already audit-grade.

## Phased rollout

| Phase | Scope | Duration |
|---|---|---|
| **P1** | Harmonize L1 + expose harness_core-compatible interfaces | 90 days |
| **P2** | L3 (A2A endpoints + AGNTCY publication) + skill marketplace integration | 90 days |
| **P3** | L6 observability convergence with Gnomon ([297](297-gnomon-seven-layer-stack-apply-plan.md)) + L7 alignment | 90 days |
| **P4** | Reference-impl publication + continued v9+ feature work | 90 days |

## Open-Fang as reference for the ecosystem

Several open-fang patterns should be promoted to `harness_core/` reference implementations:

- **5-tier verifier** → `harness_core/operations/evaluation/multi_tier_verifier.py`
- **HAFC-lite failure attribution** → `harness_core/operations/sre/failure_attribution.py`
- **9-specialist research cohort** → reference for `syndicate/` orchestration patterns
- **DAG Teams** → reference for hierarchical orchestration
- **Atropos trajectory export** → standard format for `harness_core/operations/trajectory_export/`

## One-line takeaway

**Open-Fang is the most-mature in-tree project (v8, 612 tests) and many seven-layer-stack primitives are already implemented; the apply plan is **harmonization** with `harness_core/` shared interfaces (not rebuild) plus exposing A2A endpoints for cross-project composition, with several open-fang patterns (5-tier verifier, HAFC-lite, 9-specialist DAG Teams, Atropos export) promoted as **reference implementations** the rest of the ecosystem learns from; the architectural lesson: **a project that took its production discipline seriously from v1 reaches v8 ahead of the field**.**
