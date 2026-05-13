# 211 — Cross-Project Power-Up Plan with Tradeoffs (Strategy C + Option 1)

> **Disambiguation.** This file is the **executable cross-project power-up plan** with tradeoffs explicit, complementing [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md) (the matrix), [203](203-polaris-multi-hop-reasoning-apply-plan.md) (Polaris-specific), [208](208-lyra-multi-hop-collaborative-apply-plan.md) (Lyra-specific), [209](209-argus-multi-hop-collaborative-apply-plan.md) (argus-specific), [210](210-mentat-learn-collaborative-apply-plan.md) (Mentat-specific). Where [207] was a matrix, this file is a sequenced 90-day plan with the strategic choices, sequencing, and tradeoffs committed.

> **Defaults assumed (open to redirection).** This plan adopts **Strategy C** (hub services + vendored library) and **Option 1** (foundation-first) — the two leans from the planning conversation. The six decision points in §10 are answered with the leans; if any answer changes, the affected paragraphs are flagged with `[DECISION-X]` so you can see exactly what shifts.

## One-line definition

A 90-day, six-decision, eight-tradeoff cross-project power-up that adds **shared `harness_core/` library modules** + **three hub services** (argus marketplace + curator, gnomon HIR, vertex-eval benchmarks) **without breaking the per-project independently-buildable property** every in-tree project's pyproject relies on, and that levels every project up by one to two grades on the [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md) §9 maturity matrix.

## §1 — Goals + non-goals

**Goals.**

- Power up all 16 in-tree projects (`aegis-ops`, `argus`, `atlas-research`, `cipher-sec`, `gnomon`, `harmony-voice`, `helix-bio`, `lyra`, `mentat-learn`, `open-fang`, `orion-code`, `polaris`, `quanta-proof`, `syndicate`, `vertex-eval`) on the [206] six-axis maturity matrix.
- Avoid duplication of pure-function-agent / equal-budget / multi-hop / chain-of-note primitives across 16 projects.
- Establish three hub services (argus / gnomon / vertex-eval) with stable APIs every other project consumes.
- Respect the [projects/README.md](../projects/README.md) honesty discipline: design targets vs measured numbers, vendored library per project, "nothing here is expected to ship to production without further hardening."

**Non-goals.**

- Productising hosted SaaS for any project.
- Replacing per-project CLIs with one unified shell.
- Abandoning per-project vendored `harness_core/` (we extend it, not replace it).
- Forcing every project to adopt every technique — niche-mismatch (e.g., ICPEA into Aegis-Ops) is documented as off-niche.

## §2 — Strategy committed: C (hubs + vendored)

**Default: Strategy C** — most of A's cross-project leverage with most of B's independence. Stand up `argus` / `gnomon` / `vertex-eval` as services with stable APIs (Strategy A's hub-and-spoke), but keep `harness_core/` vendored per-project (Strategy B's library shape). Each project pulls hub services lazily and adds shared library modules on its own schedule.

**Rationale.** argus / gnomon / vertex-eval are already in-tree projects; turning them into services is incremental work, not new infra. The hubs are the irreducible network-effect piece — argus's curator gets better with more trajectory submissions, gnomon's HIR gets better with more harness adapters, vertex-eval's benchmarks compound across consumers. The library side stays per-project to preserve the independently-buildable property.

**Cost accepted.** Two integration patterns (vendored library + service subscriptions). Hub-service API stability discipline. Versioned schemas with deprecation windows.

`[DECISION-1]` *If you change to Strategy A:* drop the per-project vendored copy of `harness_core/` and centralise. Add ~50 days of refactor + coordination tax. *If you change to Strategy B:* drop the hub services. Lose argus/gnomon/vertex-eval as cross-project services; each project re-implements skill curation, eval discipline, harness-IR. Add ~150 days of cumulative duplication tax across projects. **C is the recommended path.**

## §3 — Sequencing committed: Option 1 (foundation-first)

**Default: Option 1** — pure-function agents + equal-budget + ttc-curve + chain-of-note land first across all 16 projects, *before* any user-visible feature lands.

**Rationale.** Pure-function agents are the prerequisite for replayability, RL training, branching UX, and the gnomon HIR. Equal-budget + TTC + active-params benchmarks are the only honest way to evaluate everything else. Chain-of-Note is the cheapest cross-cutting quality gate. Without these, any feature shipped in days 15+ has a soft floor that will need re-doing.

**Cost accepted.** Days 1–14 ship nothing user-visible. Slow optics. Mitigated by communicating the discipline win clearly.

`[DECISION-2]` *If you change to Option 2 (showcase-first):* pick Mentat-Learn or Lyra as the showcase, ship its full Tier 0 + Tier 1 + Tier 2 in 90 days, accept that other projects copy patterns later. Visible progress in week 2; portfolio leverage delayed by ~60 days. *If you change to Option 3 (parallel cells):* abandon shared-library moves; each project owner picks 2–3 techniques from [207] independently. Maximum autonomy; ~600 days cumulative engineering vs ~450 for Option 1. **Option 1 is the recommended path.**

## §4 — The 90-day plan

### Days 1–14 — foundation discipline

**Owner.** Platform team (or whoever owns `harness_core/` and shared eval).

**Deliverables in `harness_core/`:**

- `harness_core.orchestration.pure_function` — base class enforcing side-effect-free agent semantics. Side-effecting tools (writes, posts, payments) go through gated APIs that log + require explicit BL-* permission.
- `harness_core.evals.equal_budget` — token-budget controller for fair single-vs-multi-agent comparisons. Closes the [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md) §3 critique portfolio-wide.
- `harness_core.evals.ttc_curve` — test-time-compute curve plotter; flags SealQA-style plateau-or-decline.
- `harness_core.evals.active_params` — active-parameter cost accounting for MoE comparisons.
- `harness_core.gates.chain_of_note` — per-doc relevance gate before reasoning injection.
- `harness_core.memory.decomposition_cache` — normalised-key sub-question cache.
- `harness_core.routing.bell_router` — BELLE-style question-type router (without debate; just routing).

**Per-project work in days 1–14.** Each project refactors its agent loop to extend `pure_function`. Side-effect inventory done first; gated API wrapper added for every side-effecting tool. **All 16 projects must complete this** — pure-function agents are the prerequisite for everything in days 15+.

**Day-14 milestone.** Every project's CI runs the equal-budget + ttc-curve + active-params benchmark suite over its existing functionality, baselines published.

### Days 15–30 — hub services API stabilisation

**Owners.** argus team, gnomon team, vertex-eval team.

**argus stabilises:**

- `argus.HostAdapter.route(query, mode)` — existing API, now versioned `v1`.
- `argus.HostAdapter.install_mcp(server_name, user_context)` — new, trust-gated.
- `argus.HostAdapter.list_marketplace(filter)` — federated catalog browse (LobeHub + Smithery + Glama + FastMCP Cloud).
- `argus.curator.submit_trajectory(trajectory, outcome)` — new, hosts skill auto-creation submission.
- `argus.curator.list_promoted_skills(user_context)` — new, list curator-promoted skills.

**gnomon stabilises:**

- `gnomon.hir_emit(trace_bundle)` — emit a HIR-compliant trace from any host harness.
- `gnomon.hir_attribute(trace_bundle, eval_outcome)` — primitive-attributed failure attribution.
- `gnomon.hir_bench(trace_bundle, suite)` — run benchmark suite over HIR traces.

**vertex-eval stabilises:**

- `vertex_eval.benchmarks.{musique_full, frames, seal0}` — multi-hop triple as importable suites.
- `vertex_eval.benchmarks.equal_budget(suite, budget)` — equal-budget enforcer.
- `vertex_eval.benchmarks.ttc_curve(suite, max_compute)` — TTC plotter.

**Critical discipline.** Each hub service must commit to a SemVer + deprecation-window policy. Consumers pin the major version. Breaking changes require a 30-day deprecation window with both versions live.

**Day-30 milestone.** Three hub services have v1 APIs published, with versioned schemas and stability commitments documented in each project's README.

### Days 31–45 — per-project consumption + HippoRAG promotion

**HippoRAG-2 promotion** (`[DECISION-3]`). Move the implementation in `polaris-core/src/polaris_core/memory/ppr_fusion.py` (already shipped under v2.5 P42) up to `harness_core.multi_hop.hipporag`. Polaris re-imports from the new location; Lyra / Atlas-Research / Helix-Bio / Cipher-Sec adopt as new consumers.

**Default decision.** Promote *now* (days 31–35), accepting that future Polaris v2.5 work risks breaking consumers. Mitigation: pin consumers to `harness_core.multi_hop.hipporag@1.x`; Polaris evolves on `2.x` until ready to migrate consumers.

`[DECISION-3]` *If you defer until Polaris v2.5 stabilises:* shift this to days 75–90, push HippoRAG consumption in Lyra / Atlas / Helix / Cipher to v2 of this plan. Adds 30–60 days delay to ~5 projects. **Default is promote now.**

**Per-project consumption (days 36–45).** Each project pulls hub services on its own schedule:

- **Polaris.** Already consumes ppr_fusion locally; switches to argus.HostAdapter.install_mcp for MCP installs; subscribes to gnomon.hir_emit for trace export; uses vertex-eval suites for benchmark CI.
- **Lyra.** Pulls hipporag for the deep-dive corpus; subscribes to argus marketplace for tool installs; emits HIR traces.
- **Mentat-Learn.** Submits trajectories to argus curator (cross-project skill commons); pulls promoted skills back; subscribes to argus marketplace for channel-scoped tools.
- **Aegis-Ops.** Pulls argus marketplace for ops tools (k8s, AWS, GCP MCP servers); emits HIR traces for incident replay.
- **Cipher-Sec.** Pulls argus marketplace with strict trust threshold; submits offensive-playbook trajectories with HITL gating.
- **Orion-Code.** Pulls argus marketplace for coding tools (linters, formatters, scanners); subscribes to gnomon for SWE-Bench trace attribution.

**Day-45 milestone.** All six "active consumer" projects (Polaris, Lyra, Mentat, Aegis, Cipher, Orion) successfully call hub APIs in CI. Cross-project skill commons populated by argus's curator with at least 50 promoted skills.

### Days 46–90 — niche-fit per-project lifts

**Decision: ICPEA only into person-as-user projects** (`[DECISION-4]`).

**Default split:**

- **Person-as-user (ICPEA on):** Polaris, Lyra, Atlas-Research, Helix-Bio, Mentat-Learn, Orion-Code, Syndicate.
- **System-as-user (ICPEA off):** Aegis-Ops, Cipher-Sec, Quanta-Proof, Vertex-Eval, Gnomon.
- **Edge cases:** Harmony-Voice (person-as-user but voice-channel makes ICPEA Identity layer the centre of gravity), open-fang (placeholder).

**Per-project tier-2 lifts (days 46–90):**

- **Polaris.** ICPEA + branching + per-user constitution; subscribes to argus auto-creator submitting research trajectories.
- **Lyra.** ICPEA + branching as alternative-implementation UX (via existing git-worktree subagents) + Lobe Pages-style design doc.
- **Mentat-Learn.** Tier 2 from [210]: real-time voice (Realtime / Gemini Live) + screen-share + confidential-VM deployment for high-stakes users.
- **argus.** Self-referential meta-modification + per-user constitution as routing constraint + alternative-loadout branching.
- **Atlas-Research.** ICPEA + multi-hop substrate + branching as research-tree UX.
- **Helix-Bio.** ICPEA per researcher + biomed-specific gates + voice + screen-share for "show the agent this gel image."
- **Aegis-Ops.** Per-operator constitution + Voyager-line runbook auto-creation (no ICPEA).
- **Cipher-Sec.** Per-operator constitution + offensive-playbook auto-creation with HITL gating (no ICPEA).
- **Orion-Code.** ICPEA + DSPy multi-hop program for cross-file refactor + branching for alternative-implementation UX.
- **Syndicate.** ICPEA + Pages co-authored + per-user constitution + role-catalog tournament.
- **Vertex-Eval.** No ICPEA; benchmarks-as-a-service stabilisation.
- **Gnomon.** No ICPEA; HIR + cross-harness eval discipline.
- **Quanta-Proof.** No ICPEA; branching for proof-strategy exploration; Pages for proof-document surface.
- **Harmony-Voice.** ICPEA Identity layer; voice channel matures.

**Day-90 milestone.** Every project that should be at L2+ on the [206] §9 maturity matrix is there. The portfolio averages 2 grade-bumps per project per axis.

## §5 — Per-project tier table (with shared-infra dependencies marked)

Derived from [207] §2 with shared-infra dependencies highlighted. **Bold** = blocked on shared `harness_core/`; *italics* = blocked on a hub service.

| Project | Tier 0 (days 1-14) | Tier 1 (days 15-45) | Tier 2 (days 46-90) |
|---|---|---|---|
| Polaris | **pure_function**, **equal_budget**, **ttc_curve**, *gnomon.hir_emit* | **hipporag** consume, *argus.install_mcp*, *vertex_eval suites* | ICPEA, branching, per-user constitution, *argus.curator submit* |
| Lyra | **pure_function**, **equal_budget**, **ttc_curve** | **hipporag** consume, *argus.install_mcp*, ICPEA, *argus.curator submit* | branching (git-worktree), Pages, per-user constitution |
| argus | **pure_function**, **equal_budget** | **own service stabilisation** (HostAdapter v1, curator v1, marketplace v1) | meta-modification, per-user constitution, alternative-loadout branching |
| Atlas-Research | **pure_function**, **equal_budget**, **ttc_curve** | **hipporag**, ICPEA, *argus.install_mcp* | branching as research tree, Pages, per-user constitution |
| Aegis-Ops | **pure_function**, **equal_budget** | *argus.install_mcp* (ops tools), Pages-as-runbook | per-operator constitution, runbook auto-creation via *argus.curator* |
| Harmony-Voice | **pure_function**, **equal_budget**, **ttc_curve** | ICPEA Identity, **chain_of_note**, voice channel | Realtime API, screen-share, confidential VM |
| Helix-Bio | **pure_function**, **equal_budget**, **ttc_curve** | **hipporag**, ICPEA, *argus.install_mcp* (biomed tools), retraction gate | branching, Pages, voice + screen-share |
| Cipher-Sec | **pure_function**, **equal_budget** | *argus.install_mcp* (strict trust), per-agent layer access | per-operator constitution, offensive-playbook auto-creation |
| Mentat-Learn | **pure_function**, ICPEA, IndexedDB local-first | **chain_of_note**, *argus.curator submit*, *argus.install_mcp* | Realtime voice, screen-share, confidential VM (per [210]) |
| Orion-Code | **pure_function**, **equal_budget** | DSPy multi-hop, ICPEA, *argus.install_mcp* | branching alternative-impl, Pages design doc, per-user constitution |
| Syndicate | **pure_function**, **equal_budget** | ICPEA, *argus.install_mcp*, Pages | per-user constitution, role-catalog tournament |
| Quanta-Proof | **pure_function**, **equal_budget** | Pages-as-proof, branching as proof strategy | confidential VM (formal-method secrets) |
| Vertex-Eval | **pure_function**, **equal_budget**, **ttc_curve** | **own service stabilisation** (vertex_eval suites v1) | benchmark CI for all consumers |
| Gnomon | **pure_function**, **equal_budget** | **own service stabilisation** (HIR API v1) | HIR adapter library, cross-harness eval CI |
| open-fang | (placeholder) | (placeholder) | (placeholder) |

## §6 — The eight tradeoffs explicit

These are the costs you accept by adopting this plan. Each one is a real tradeoff, not a non-issue.

1. **Shared substrate vs per-project copies.** Shared = leverage; per-project = independence. Strategy C splits the difference by sharing only the highest-leverage primitives (`harness_core/{orchestration, evals, gates, multi_hop, memory}`) while keeping per-project vendoring of the agent loop, CLI, and project-specific glue. Cost: two patterns to maintain.
2. **Hub-and-spoke vs library-only.** Hubs = network effects (skill commons, benchmark commons, cross-harness eval); library-only = simpler deploy, no SPOF. Cost: hub-service stability discipline + versioning rigor.
3. **HippoRAG promotion timing.** Promote now = fast leverage for ~5 projects; wait for v2.5 = avoid promoting unstable code. Default: promote now with consumer-pinning to `1.x`. Cost: deprecation discipline on Polaris's evolution.
4. **Pure-function portfolio-wide vs project-by-project.** Portfolio-wide (default) = consistency for replayability + RL + branching; project-by-project = ship before consensus. Cost: every project must refactor in days 1–14, before any visible feature lands.
5. **MCP marketplace via argus vs each project picks.** Centralise (default) = trust gating consolidated, supply-chain attacks gated once; decentralise = per-project autonomy. Cost: argus is now a SPOF for tool installation; mitigated by graceful-degradation + local caching.
6. **Speed vs maturity.** Adopt all six axes from [206] (default) vs ship the highest-leverage three first. Cost: 90-day plan vs ~45-day plan; trade short-term visibility for portfolio coherence.
7. **In-tree services vs external.** argus / gnomon / vertex-eval already in-tree as projects; promoting them to services = new versioning + SLA discipline. Cost: those project owners take on platform-team responsibilities.
8. **ICPEA only on person-as-user projects.** Strict on/off split (default) = prevents personal-data contamination in system-as-user projects; lax = simpler taxonomy. Cost: per-project decision discipline; document the split clearly so future contributors don't introduce ICPEA into Aegis or Cipher by accident.

## §7 — Risk register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Day 1–14 pure-function refactor blocks every project simultaneously | Med | High | Per-project refactor in parallel branches; merge when each project's CI passes; only block downstream tier work, not unrelated work |
| Hub service API churn breaks consumers | Med | High | SemVer + 30-day deprecation window; consumers pin major version; nightly cross-project smoke tests in vertex-eval |
| HippoRAG promotion before Polaris v2.5 stabilisation | Med | Med | Pin consumers to `harness_core.multi_hop.hipporag@1.x`; Polaris evolves on `2.x` until consumers migrate |
| ICPEA cross-contaminates between projects | Low-Med | High | Per-project namespace + per-agent layer access control + per-user scope keys |
| MCP marketplace supply-chain attack via argus | Med | High | Argus trust verdict before install; sandboxed MCP execution; CVE feed integration; capability-scope audit |
| argus / gnomon / vertex-eval team velocity insufficient | Med | High | Each hub service has explicit Day-30 milestone; if missed, cascade-shift downstream tier work; document fallback (vendored-library-only path) |
| Skill auto-creator over-promotes noise | Med | Med | Surrogate verifier + held-out eval + 2-occurrence threshold + cross-project judge isolation |
| Equal-budget enforcement is hard across heterogeneous architectures | High | Med | Token-counting at the harness boundary; document Gemini-style accounting caveats per [202] §3; report per-architecture variance |
| Dependency loops (argus depends on gnomon, gnomon depends on argus) | Low | High | Define minimal stable APIs; break circular dependencies via traits/interfaces; weekly dependency-graph review |
| Per-user constitution drift over time | Med | Low-Med | Versioned constitutions; drift detection at extraction time; user prompted to confirm major changes |
| Confidential VM deployment cost overruns budget | Low | Med | Optional deployment tier; default users on shared infra; opt-in for SEV-SNP / TDX |
| Day-1 to Day-14 has no user-visible upgrade | High | Low-Med | Communicate the discipline win clearly; surface per-project equal-budget baselines as the visible milestone |

## §8 — Quantified leverage estimate

A rough portfolio-level estimate of the three-strategy comparison:

| Metric | Strategy A | Strategy B | **Strategy C (default)** |
|---|---|---|---|
| Time to first user-visible upgrade | 30–45 days | 5–10 days per project | 14–30 days |
| Cumulative engineering cost (all 16 projects) | ~350 days | ~600 days | ~450 days |
| Risk of duplication | Low | High | Medium |
| Risk of SPOF | High (one substrate breaks all) | Low | Medium (only hub services) |
| Coordination tax | High | Low | Medium |
| Network effects | Yes | No | Yes |
| Honors README "independently buildable" | No | Yes | Mostly |
| Reversibility | Hard | Easy | Medium |
| Grade bumps on [206] §9 averaged across portfolio | +2.0 | +1.0 | +1.7 |
| Cumulative engineering cost per grade bump | ~22 days | ~38 days | ~16 days |

**Key takeaway.** Strategy C delivers grade-bumps at lower cost-per-bump than either A or B, despite shipping less aggressive shared substrate than A. The reason: hub services concentrate the network-effect work where it matters (skill commons, benchmark commons, cross-harness eval) and leave per-project work alone.

## §9 — Milestones

| Milestone | Day | Deliverable |
|---|---|---|
| **M1 — Foundation discipline shipped** | 14 | All 16 projects refactored to `pure_function`; equal-budget + TTC + active-params benchmark suite running in every CI; per-project baselines published |
| **M2 — Hub services v1** | 30 | argus / gnomon / vertex-eval each ship versioned APIs; SemVer + deprecation policy documented; first cross-project smoke test passes |
| **M3 — HippoRAG promoted + active consumers** | 45 | `harness_core.multi_hop.hipporag@1.0` published from polaris-core; Polaris / Lyra / Atlas / Helix / Cipher all consume from new location; cross-project skill commons populated with ≥50 promoted skills |
| **M4 — Person-as-user ICPEA shipped** | 60 | Polaris / Lyra / Atlas / Helix / Mentat / Orion / Syndicate / Harmony all have ICPEA layers populated and per-user routing/personalisation live |
| **M5 — Branching + Pages + voice live** | 90 | Branching UX in ~9 projects; Lobe-Pages-style co-author surface in research/ops/proof projects; Realtime voice + screen-share in Mentat / Harmony / Helix |

## §10 — Decision log

The six decisions from the planning conversation, with the leans committed as defaults. Each is open to redirection — flip a default and the affected paragraphs in this doc change as marked.

| # | Decision | Default (lean) | Alternatives + cost |
|---|---|---|---|
| 1 | Strategy | **C — hubs + vendored** | A: +50 days refactor + coordination tax. B: +150 days cumulative duplication |
| 2 | Sequencing | **Option 1 — foundation-first** | Option 2: visible win in week 2; portfolio leverage delayed 60 days. Option 3: max autonomy; ~600 days cumulative cost |
| 3 | HippoRAG promotion timing | **Now (days 31–45)** | Defer to v2.5: 30–60 day delay to ~5 projects |
| 4 | ICPEA scope | **Person-as-user only** (Polaris, Lyra, Atlas, Helix, Mentat, Orion, Syndicate, Harmony) | All projects: personal-data contamination risk in Aegis / Cipher / Quanta-Proof |
| 5 | Hub services live in-tree (argus / gnomon / vertex-eval as services) | **Yes — promote in-tree projects to service status** | External services: lose in-tree control + customisation; +infra cost |
| 6 | Visible-progress budget for days 1–14 | **Accept invisible progress; communicate discipline win** | Parallel showcase track on Mentat or Lyra: +20% engineering cost; visible progress in week 2 |

## §11 — How to redirect

If any default in §10 is wrong for your priorities, here's how each changes the rest of this doc:

- **Flip Decision 1 to Strategy A.** §2 commits, §4 days 15–30 collapses (hub services become library modules), §5 loses the *italics* dependency markers, §6 tradeoff #2 disappears, §7 risk row "API churn" → "shared-substrate breakage."
- **Flip Decision 1 to Strategy B.** §2 commits, §4 days 15–45 disappears (no hub services), §5 loses the *italics* markers, §6 tradeoff #2 disappears, §7 risk row "team velocity" disappears, §8 cumulative cost rises to ~600 days.
- **Flip Decision 2 to Option 2.** §4 days 1–14 shrinks to one project; the showcase project (default Mentat) ships full Tier 0 + 1 + 2 in 90 days; other 15 projects copy patterns from days 91+. Cumulative timeline extends to ~150 days.
- **Flip Decision 2 to Option 3.** §4 collapses entirely; each project owner picks 2–3 techniques from [207] independently; no shared substrate; no hub services.
- **Flip Decision 3 to defer.** §4 days 31–45 shrinks; HippoRAG consumption in Lyra / Atlas / Helix / Cipher slips to v2 of this plan; ~60-day delay to those four projects.
- **Flip Decision 4 to "all projects."** §5 ICPEA cells expand to Aegis / Cipher / Quanta-Proof / Vertex / Gnomon; §6 tradeoff #8 changes; §7 adds row "personal-data contamination in system-as-user projects."
- **Flip Decision 5 to external.** Hub services move out-of-tree; §4 days 15–30 changes to "subscribe to external services"; §6 tradeoff #7 inverts; +infra cost; -in-tree control.
- **Flip Decision 6 to parallel showcase.** §4 days 1–14 adds a parallel Mentat-Learn track; cumulative engineering cost +20%; visible progress in week 2.

## §12 — What success looks like at day 90

If this plan executes as written, on day 90 the portfolio has:

- **All 16 projects on pure-function agents** with replayable trajectories, equal-budget benchmarks, TTC curves, and active-parameter accounting in CI.
- **Three stable hub services** (argus marketplace + curator, gnomon HIR + cross-harness eval, vertex-eval benchmarks) consumed by the other 13 projects via versioned APIs.
- **HippoRAG-2 multi-hop substrate** in `harness_core` consumed by Polaris (existing), Lyra, Atlas-Research, Helix-Bio, Cipher-Sec.
- **ICPEA typed personal memory** in 8 person-as-user projects (Polaris, Lyra, Atlas, Helix, Mentat, Orion, Syndicate, Harmony).
- **Cross-project skill commons** populated by argus's curator with ≥200 promoted skills sourced from Mentat, Lyra, Polaris, Aegis, Cipher, Orion trajectories.
- **Branching UX + Pages co-authored surface** in ~9 projects.
- **Realtime voice + screen-share** in Mentat-Learn, Harmony-Voice, Helix-Bio.
- **L2+ maturity** on all six [206] §9 axes for at least 4 projects (Mentat, Polaris, Lyra, argus); L2+ on at least four axes for the other 11.

That's the win. The cost is ~450 cumulative engineering days across the portfolio, six committed decisions, and ~14 days of invisible-progress communication discipline.

## §13 — Cross-references

- [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md), [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md), [208-lyra-multi-hop-collaborative-apply-plan](208-lyra-multi-hop-collaborative-apply-plan.md), [209-argus-multi-hop-collaborative-apply-plan](209-argus-multi-hop-collaborative-apply-plan.md), [210-mentat-learn-collaborative-apply-plan](210-mentat-learn-collaborative-apply-plan.md) — apply-plan family.
- [198-multi-hop-qa-datasets-canon](198-multi-hop-qa-datasets-canon.md) → [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md) — multi-hop block.
- [204-metagpt-foundationagents-2026-refresh](204-metagpt-foundationagents-2026-refresh.md), [205-lobehub-collaborative-teammate-platform](205-lobehub-collaborative-teammate-platform.md), [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md) — collaborative-AI block.
- [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md) — Polaris's V2.x phasing this plan tracks.
- [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md), [194-argus-omega-enhanced-design](194-argus-omega-enhanced-design.md), [197-argus-omega-vol-3-recursive-skills-curator](197-argus-omega-vol-3-recursive-skills-curator.md) — argus design substrate.
- [projects/README.md](../projects/README.md) — the honesty discipline this plan respects.

**The one-line takeaway for harness designers:** Adopt **Strategy C + Option 1** as defaults, accept the eight tradeoffs explicit in §6, hit the five milestones in §9, and the portfolio levels up by 1.7 grades on the [206] §9 maturity matrix at ~16 cumulative-engineering-days per grade bump — better than either Strategy A (~22) or Strategy B (~38).
