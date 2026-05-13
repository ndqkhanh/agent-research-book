# 209 — argus Apply Plan: Multi-Hop + Collaborative-AI Techniques into the Skill-Router Agent

> **Disambiguation.** This file is the **argus-specific** deep apply plan, extending the [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md) and [208-lyra-multi-hop-collaborative-apply-plan](208-lyra-multi-hop-collaborative-apply-plan.md) patterns to argus. The cross-project matrix is in [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md); the foundational argus design is [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md), the Omega vols are [194-argus-omega-enhanced-design](194-argus-omega-enhanced-design.md) / [195-argus-omega-vol-2-trajectory-temporal-horizon](195-argus-omega-vol-2-trajectory-temporal-horizon.md) / [197-argus-omega-vol-3-recursive-skills-curator](197-argus-omega-vol-3-recursive-skills-curator.md), and the field comparison is [196-argus-vs-field-skill-loading-comparison](196-argus-vs-field-skill-loading-comparison.md).

## One-line definition

A staged plan to fold multi-hop reasoning + collaborative-AI techniques into argus — the **harness-skill-router** with eleven shipped phases (A0–A10), a five-tier router ladder (BM25 → progressive disclosure → embedding → cross-encoder rerank → hierarchical Corpus2Skill), refinement loop, curator, marketplace adapter, and `HostAdapter` host-integration object — without breaking its **distinctive role as the trust-gating substrate for every other in-tree project**, while levelling argus *up* from a catalog-aware router to a **user-aware, marketplace-backed, trust-verified, self-curating skill-routing service** that the rest of the harness ecosystem consumes.

## §1 — Why apply these techniques to argus

argus is **not a multi-hop QA agent**, **not a research synthesis agent**, **not a coding agent**. It is a **skill-loading agent** with five duties (discovery / routing / curation / refinement / governance) that *every other in-tree project depends on* (cf. [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md) §4 "Shared via argus"). The right apply plan is therefore inverted: argus is mostly the *provider* of techniques, not the *consumer*.

That inversion concentrates argus's adoption priorities on three axes:

- **MCP marketplace as the primary discovery substrate** — argus already indexes 22k+ Claude Code skills and 25k+ MCP servers; LobeHub's curated 169,739-skill index + permission-negotiation-at-install gives argus a higher-trust starting catalog with metadata. Tier 0.
- **Voyager-line skill auto-creation** — argus's mission *is* skill curation; the auto-creator pattern from [167-autoskill-experience-driven-lifelong-learning](167-autoskill-experience-driven-lifelong-learning.md) → [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md) is core, not a side feature. Tier 0.
- **Per-user routing personalisation via ICPEA Preference + Experience layers** — argus's router is currently catalog-aware; making it *user-aware* unlocks materially better routing accuracy. Tier 1.

Multi-hop substrate (HippoRAG, DSPy multi-hop, Plan-on-Graph, Search-R1) is **off-niche** for argus — argus routes to skills that *have* multi-hop substrates, but doesn't itself need to multi-hop-reason over a knowledge graph. The closest argus comes is the **Tier 4 hierarchical navigation** (Corpus2Skill) which is a graph walk but over the skill taxonomy, not over a QA knowledge graph.

Take this plan seriously and three things change. (1) argus's marketplace adapter becomes the trust-gated MCP install gateway every other in-tree project consumes (Polaris, Lyra, Atlas, Helix, Cipher, Mentat, Orion, Syndicate). (2) ICPEA-aware routing closes the gap between "the right skill exists" and "the right skill for *this* user is loaded." (3) The recursive curator (Omega Vol. 3, [197](197-argus-omega-vol-3-recursive-skills-curator.md)) gets fed by every consuming project's trajectories, not just internal ones — turning argus into a cross-project skill commons.

## §2 — The argus × technique mapping table

Each row is one technique, mapped to an argus module + tier slot. argus's existing module structure: `src/harness_skill_router/{tiers/{tier_0..4}, refine/{drift,telemetry,cycle,description,consolidate}, curator/{scan,tiers}, marketplace/{adapter,fetcher}, host.py, router_chain.py}`.

| Technique | Source | argus module | Tier | Effort | Payoff |
|---|---|---|---|---|---|
| **Marketplace + trust-gate substrate (argus's primary axis)** | | | | | |
| LobeHub MCP marketplace integration | [205](205-lobehub-collaborative-teammate-platform.md) §2.3, [206](206-collaborative-ai-canon-2026.md) §3 | `marketplace/lobehub_adapter.py` | 0 | M | 169,739-skill catalog with metadata + reviews |
| Smithery / Glama / FastMCP Cloud as upstream sources | [206](206-collaborative-ai-canon-2026.md) §3 | `marketplace/{smithery,glama,fastmcp_cloud}.py` | 0 | M | Multi-source federation behind one API |
| One-click install + permission negotiation at install | [205](205-lobehub-collaborative-teammate-platform.md) §2.3 | `marketplace/install_flow.py` | 0 | S | Surfaces over-broad permissions when user is paying attention |
| Argus trust verdict on every install | [180](180-argus-skill-router-agent-design.md) | `marketplace/trust_verdict.py` (extends existing curator) | 0 | M | Closes the supply-chain gap LobeHub leaves open |
| MCP capability schema validation | [206](206-collaborative-ai-canon-2026.md) §3 | `marketplace/capability_validator.py` | 0 | S | Rejects malformed MCP servers at install |
| **Voyager-line skill auto-creation (argus's primary axis)** | | | | | |
| Trajectory-driven skill extractor | [167](167-autoskill-experience-driven-lifelong-learning.md), [197](197-argus-omega-vol-3-recursive-skills-curator.md) | `curator/auto_creator/extract.py` | 0 | M | Successful trajectories become skill candidates |
| Surrogate-verifier gating (info-isolated) | [169](169-coevoskills-co-evolutionary-verification.md) | `curator/auto_creator/surrogate_verifier.py` | 0 | M | +30pp ablation lift when surrogate is enforced |
| Held-out eval gate | [167](167-autoskill-experience-driven-lifelong-learning.md), [171](171-skill-self-evolution-2026-synthesis.md) | `curator/auto_creator/eval_gate.py` | 0 | S | Promote-skill bright-line gate |
| Pareto-frontier candidate selection (k=3) | [168](168-evoskill-coding-agent-skill-discovery.md) | `curator/auto_creator/pareto.py` | 1 | S | Best-of-k skill variants |
| Common-mistakes negative-space tier | [170](170-skillrl-recursive-skill-augmented-rl.md) | `curator/auto_creator/negative_space.py` | 1 | S | Skills know what they *don't* do |
| Self-referential meta-modification | [197](197-argus-omega-vol-3-recursive-skills-curator.md) | `curator/auto_creator/meta.py` | 2 | M | Curator improves its own curation rules |
| **ICPEA-aware routing (argus's user-aware axis)** | | | | | |
| ICPEA Preference + Experience layers as routing signal | [205](205-lobehub-collaborative-teammate-platform.md) §2.1, [206](206-collaborative-ai-canon-2026.md) §1 | `tiers/tier_2_5_user_preference.py` (between embedding and rerank) | 1 | M | Per-user routing ("this user prefers code-emitting skills") |
| Per-user skill-affinity model | [206](206-collaborative-ai-canon-2026.md) §5 | `refine/user_affinity.py` | 1 | M | Affinity scores update from edit/rollback |
| Per-user constitution as routing constraint | [206](206-collaborative-ai-canon-2026.md) §5 | `refine/user_constitution.py` | 2 | M | Constitution limits which skills are eligible |
| **Branching / UX as alternative-loadout surface** | | | | | |
| Alternative-skill-loadout branching | [206](206-collaborative-ai-canon-2026.md) §4 | `host.py` (extend HostAdapter) + UI | 2 | M | When router is uncertain, present alternatives as branches |
| Skill-loadout review timeline (Pages-style) | [205](205-lobehub-collaborative-teammate-platform.md) §2.2 | new optional UI surface | 2 | M | Per-week review of curator decisions |
| **Cross-cutting discipline** | | | | | |
| Pure-function curator agents | [202](202-multi-agent-multi-hop-reckoning-2026.md) §4 | `curator/pure_function.py` | 0 | S | Replayable curator decisions |
| Equal-budget benchmark harness | [202](202-multi-agent-multi-hop-reckoning-2026.md) §3 | `tests/benchmarks/equal_budget.py` | 0 | S | Argus-vs-LobeHub-vs-OpenAI honest comparisons |
| Active-parameter accounting | [202](202-multi-agent-multi-hop-reckoning-2026.md) §2 | `tests/benchmarks/active_params.py` | 0 | S | MoE comparisons on retrieval ranking |
| **The Tier 4 hierarchical navigation (already shipped, augmented)** | | | | | |
| Plan-on-Graph backtracking on skill taxonomy | [200](200-graph-grounded-multi-hop-retrieval.md) §"Plan-on-Graph" | `tiers/tier_4_navigate.py` (existing — augment) | 1 | S | Recovers from wrong-subdomain navigation |
| AnchorRAG-style ambiguous-skill prediction | [200](200-graph-grounded-multi-hop-retrieval.md) §"AnchorRAG" | `tiers/tier_2_embedding.py` (extend) | 1 | M | Robust to imperfect skill-name matching |

## §3 — Tier 0 (days 1-14): the marketplace + auto-creator hardening

argus's existing eleven phases (A0–A10) are shipped; Tier 0 here builds **on top** of those, not in them.

### 0.1 — LobeHub MCP marketplace integration

**File.** `src/harness_skill_router/marketplace/lobehub_adapter.py`

**What.** Implement a `LobeHubAdapter` that conforms to argus's existing `MarketplaceAdapter` interface (already used by `marketplace/adapter.py`). Pulls from:
- [lobehub.com/skills](https://lobehub.com/skills) — 169,739 indexed skills, including 10,000+ MCP-compatible.
- [lobehub.com/mcp](https://lobehub.com/mcp) — curated MCP servers with one-click metadata.
- [lobehub.com/agent](https://lobehub.com/agent) — 505+ community agents (informational; argus doesn't install agent personas).

**Schema mapping.** LobeHub's `index.json` schema → argus's typed `Skill` model. Permission scopes → argus's bright-line gates. Reputation / download counts → argus's prior on the trust verdict.

### 0.2 — Multi-marketplace federation

**File.** `src/harness_skill_router/marketplace/{smithery,glama,fastmcp_cloud}.py`

**What.** Adapters for Smithery, Glama (23,212 servers), FastMCP Cloud, and the official MCP registry ([registry.modelcontextprotocol.io](https://registry.modelcontextprotocol.io/)). All conform to the same `MarketplaceAdapter` interface; argus's existing `marketplace/fetcher.py` federates them.

**Why all four.** Different marketplaces have different curation philosophies and overlapping but non-identical catalogs. Federation means an argus user gets the union of all four indexes, with deduplication on canonical server URL.

### 0.3 — One-click install + permission negotiation at install

**File.** `src/harness_skill_router/marketplace/install_flow.py`

**What.** Adopt LobeHub's pattern: pick a server, argus writes a config block (or ABI call to the host harness's MCP config), the server is reachable on next message. Critical: **OAuth scopes / credential injection happen at install, not at first use** — surfaces over-broad permissions when the user is paying attention.

### 0.4 — Argus trust verdict on every install

**File.** `src/harness_skill_router/marketplace/trust_verdict.py`

**What.** A typed `TrustVerdict` returned for every marketplace lookup. Sources of evidence:
- LobeHub / Smithery / Glama metadata (downloads, ratings, reports).
- argus's own curator scan (CVE feed, vulnerability scan via `bandit` for Python, `npm audit` for TS).
- Permission-scope audit (does this server request more scopes than it advertises?).
- Source-code lineage check (is this a fork of a known-malicious server?).
- ICPEA Identity layer of the calling user (regulated industry users get stricter thresholds).

**Verdict tiers.** TRUSTED / SUSPICIOUS / REJECTED. Bright-line gate `BL-SUPPLY-CHAIN-VERDICT` enforces.

### 0.5 — MCP capability schema validation

**File.** `src/harness_skill_router/marketplace/capability_validator.py`

**What.** Strict validation of the MCP server's `initialize` capability manifest. Rejects servers with malformed schemas, undeclared tools, or capability claims that don't match the actual exposed surface.

### 0.6 — Trajectory-driven skill extractor

**File.** `src/harness_skill_router/curator/auto_creator/extract.py`

**What.** Consumes successful trajectories from any host (Polaris, Lyra, Mentat, etc.) via a stable `argus.curator.submit_trajectory(trajectory)` API. An LLM-side extractor proposes skill candidates from recurring patterns ([167-autoskill-experience-driven-lifelong-learning](167-autoskill-experience-driven-lifelong-learning.md) — extractor / judge / merger).

**Surface.** Each calling project's harness submits trajectories at session-end (Lyra does this on `lyra` exit; Polaris does this on report-finalize; Mentat does this on workflow completion).

### 0.7 — Surrogate-verifier gating

**File.** `src/harness_skill_router/curator/auto_creator/surrogate_verifier.py`

**What.** From [169-coevoskills-co-evolutionary-verification](169-coevoskills-co-evolutionary-verification.md) — an *info-isolated* surrogate verifier that grades the candidate skill *without seeing the gold trajectory*. Empirically yields **+30pp ablation lift** when enforced. Catches "skill that memorises the example" regressions.

### 0.8 — Held-out eval gate

**File.** `src/harness_skill_router/curator/auto_creator/eval_gate.py`

**What.** Bright-line gate `BL-PROMOTE-SKILL`: the candidate skill must improve the held-out eval by ≥X% before promotion. The held-out eval set is per-domain (code, research, ops, biomed) and shipped with argus.

### 0.9 — Pure-function curator agents

**File.** `src/harness_skill_router/curator/pure_function.py`

**What.** Yenugula's pattern ([202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md) §4): every curator decision is a pure function of (catalog state, trajectory, eval result). Replayable, testable, auditable. Necessary precondition for the Tier 2 self-referential meta-modification.

### 0.10 — Equal-budget benchmark + active-parameter accounting

**File.** `tests/benchmarks/{equal_budget,active_params}.py`

**What.** Argus-vs-LobeHub-vs-OpenAI-skills routing accuracy comparisons must control for compute. Add the harnesses; report numbers on the argus README.

## §4 — Tier 1 (days 15-45): user-aware routing

After Tier 0, the marketplace and curator surfaces are in place. Tier 1 makes the router *user-aware*.

### 1.1 — ICPEA Preference + Experience layers as routing signal

**File.** `src/harness_skill_router/tiers/tier_2_5_user_preference.py`

**What.** A new tier between Tier 2 (embedding) and Tier 3 (cross-encoder rerank) that re-scores embedding candidates using the calling user's ICPEA Preference + Experience layers. Concretely:

- **Preference layer.** "This user prefers Python skills over Bash; prefers code-emitting skills over UI skills; prefers `pytest` over `unittest`." Skills tagged as matching get a Preference boost.
- **Experience layer.** "This user has used the `pdf-extract` skill 12 times with success; never accepted the `web-scraper-ai` skill." Skills with positive Experience get an affinity boost.

The combined re-score is `(1−α)·embedding_score + α·(preference_score + experience_score)`, with α tunable (default 0.3).

### 1.2 — Per-user skill-affinity model

**File.** `src/harness_skill_router/refine/user_affinity.py`

**What.** Affinity scores update on edit / rollback / explicit user feedback. PRELUDE/CIPHER's edit-as-preference-signal ([206](206-collaborative-ai-canon-2026.md) §1) is the canonical reference. Sits next to argus's existing `refine/drift.py` and `refine/telemetry.py`.

### 1.3 — Tier 4 hierarchical navigation: Plan-on-Graph backtracking

**File.** `src/harness_skill_router/tiers/tier_4_navigate.py` (extend the existing module)

**What.** When the hierarchical Corpus2Skill walk visits the wrong subdomain, the Reflection mechanism backtracks. This is the only multi-hop technique that maps cleanly into argus's existing architecture — Tier 4 is itself a graph walk, so PoG ([200-graph-grounded-multi-hop-retrieval](200-graph-grounded-multi-hop-retrieval.md) §"Plan-on-Graph") fits naturally.

### 1.4 — AnchorRAG-style ambiguous-skill prediction

**File.** `src/harness_skill_router/tiers/tier_2_embedding.py` (extend)

**What.** When the user's query maps to multiple equally-plausible skill anchors (e.g. "summarise this document" could be `pdf-summary`, `doc-summary`, or `text-summary`), use AnchorRAG's anchor-prediction pattern ([200](200-graph-grounded-multi-hop-retrieval.md) §"AnchorRAG") to enumerate candidates rather than commit.

### 1.5 — Pareto-frontier + common-mistakes tier

**File.** `src/harness_skill_router/curator/auto_creator/{pareto,negative_space}.py`

**What.** From [168-evoskill-coding-agent-skill-discovery](168-evoskill-coding-agent-skill-discovery.md) and [170-skillrl-recursive-skill-augmented-rl](170-skillrl-recursive-skill-augmented-rl.md) — keep the top-3 candidate skill variants on the Pareto frontier (cost vs accuracy), and explicitly document common-mistakes for each skill (negative space).

## §5 — Tier 2 (days 46-90): meta-modification + UX

### 2.1 — Self-referential meta-modification

**File.** `src/harness_skill_router/curator/auto_creator/meta.py`

**What.** [197-argus-omega-vol-3-recursive-skills-curator](197-argus-omega-vol-3-recursive-skills-curator.md) — the curator improves its own curation rules. Pure-function agents (Tier 0) make this safe; the meta-modification trajectory is replayable. Sandbox via shadow-deploy: meta-modified rules run in shadow against the past month of curation decisions before being promoted.

### 2.2 — Per-user constitution as routing constraint

**File.** `src/harness_skill_router/refine/user_constitution.py`

**What.** ICAI/C3AI ([206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md) §5) — a 3-principle per-user constitution that constrains which skills are eligible. Examples:
- A regulated-industry user's constitution might exclude all `web-scrape-*` skills by default.
- A privacy-sensitive user's constitution might exclude all skills with cloud upload.
- A speed-prioritising user's constitution might exclude all skills with >5s latency.

### 2.3 — Alternative-skill-loadout branching

**File.** `src/harness_skill_router/host.py` (extend `HostAdapter`)

**What.** When the router is uncertain between two skill activations, return both alternatives in the `RouteDecision`. The host harness (Lyra / Polaris / etc.) decides whether to present them as branches to the user. argus doesn't render the UI; it provides the decision shape.

### 2.4 — Skill-loadout review timeline (Pages-style)

**File.** new optional UI surface (e.g. `argus-review/`)

**What.** A weekly review surface where the user inspects argus's curation decisions: which skills were promoted, which were rejected, which got higher Experience affinity. Lobe Pages-style timeline preserves prior versions.

## §6 — How this maps onto argus's existing A0-A10 phasing

| argus phase | Existing scope | This plan adds |
|---|---|---|
| A0–A4 | Tiers (BM25, progressive, embedding, rerank) + router_chain | Tier 2.5 (ICPEA preference re-score) at Tier 1 |
| A5 | Tier 4 hierarchical navigate | + Plan-on-Graph backtracking + AnchorRAG anchor-prediction at Tier 1 |
| A6 | refine: drift + telemetry + cycle | + user_affinity + user_constitution at Tier 1/2 |
| A7 | refine: description + consolidate | (no additions; existing surface is enough) |
| A8 | curator: scan + tiers | + auto_creator/{extract, surrogate_verifier, eval_gate, pareto, negative_space, meta, pure_function} at Tier 0/1/2 |
| A9 | marketplace: adapter + fetcher | + lobehub_adapter + smithery + glama + fastmcp_cloud + install_flow + trust_verdict + capability_validator at Tier 0 |
| A10 | host.py + SKILL.md persona | + alternative-loadout branching at Tier 2 |
| A11 (proposed) | User-awareness | ICPEA + per-user constitution (Tier 1/2) |
| A12 (proposed) | Marketplace federation + trust | LobeHub + Smithery + Glama + FastMCP + trust_verdict (Tier 0) |
| A13 (proposed) | Auto-creator + meta | Skill auto-creation + recursive curator (Tier 0/2) |

## §7 — Five sequencing decisions worth defending

### Decision 1 — Marketplace federation before auto-creator

argus's existing curator (`curator/scan.py`, `curator/tiers.py`) operates on whatever skills are in the catalog. Without the LobeHub / Smithery / Glama / FastMCP federation, argus's catalog is roughly the user's local skill repository — small. Federate first, then the auto-creator runs against a 169,739-skill substrate from day one.

### Decision 2 — ICPEA *augments* embedding score, not *replaces* it

Tier 2.5 in §4.1 is *re-scoring*, not *replacing*. The embedding score remains the primary signal; ICPEA preference + experience are weights on top. This avoids cold-start failure when a new user has no preference history yet.

### Decision 3 — Pure-function curator agents are Tier 0, not Tier 1

You cannot replay a non-deterministic curator decision, and you cannot RL-train against non-replayable trajectories. The self-referential meta-modification in Tier 2 *requires* this. So pure-function agents land in Tier 0 alongside the marketplace adapters.

### Decision 4 — Trust verdicts compose with LobeHub trust, not replace it

LobeHub already has reputation (downloads, ratings, reports). argus's trust verdict adds CVE / vulnerability / scope-audit / lineage / ICPEA-Identity inputs. The composition is the win — neither alone is sufficient.

### Decision 5 — Defer alternative-loadout branching to Tier 2

Branching at the routing level is meaningful only when (a) ICPEA is in place (Tier 1) and (b) the alternative loadouts are user-aware. Without both, branching is a confusing UX with no clear axis for the user to choose along. So Tier 2.

## §8 — Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| LobeHub catalog poisoning | Med | High | argus trust verdict gates every install; CVE feed integration; scope-audit |
| Marketplace federation latency | Med | Med | Cache marketplace queries with TTL; degrade gracefully when one upstream is slow |
| Trust verdict false-positives reject legitimate skills | Low-Med | Med | TRUSTED / SUSPICIOUS / REJECTED tiers; SUSPICIOUS allows install with explicit user consent |
| ICPEA cross-contaminates routing across tenants | Low | High | ICPEA per-user-namespace; argus enforces tenant isolation in the routing path |
| Auto-creator over-promotes noise | Med | Med | Surrogate verifier (+30pp ablation lift) + held-out eval + 2-occurrence threshold |
| Meta-modification of curator rules introduces regressions | Med | High | Shadow-deploy meta-rules against past month of decisions; promote only on no-regression |
| Per-user constitution conflicts with marketplace policy | Low | Med | Constitution + marketplace policy compose with documented precedence (constitution wins on user-scoped decisions; marketplace wins on supply-chain) |
| User-affinity model gets gamed by repeated rollback | Low | Low-Med | Affinity updates damped; rollback within 30s ignored as accidental |
| Equal-budget benchmark misrepresents argus's tiered architecture | High | Low-Med | Report per-tier compute separately; document the 5-tier ladder in benchmarks |
| Recursive curator infinite-loop on degenerate trajectories | Low | High | Bounded recursion depth + budget cap; alarm on cycle detection |

## §9 — Concrete day-by-day Tier 0 checklist

A 14-day Tier 0 plan:

- **Day 1-2** — `marketplace/lobehub_adapter.py` skeleton; first `index.json` pull from LobeHub.
- **Day 3** — `marketplace/{smithery,glama,fastmcp_cloud}.py` adapters.
- **Day 4-5** — `marketplace/install_flow.py` + `marketplace/capability_validator.py`.
- **Day 6** — `marketplace/trust_verdict.py` first cut (downloads + ratings + CVE feed + scope-audit).
- **Day 7** — `curator/pure_function.py` base class; refactor existing curator components.
- **Day 8-9** — `curator/auto_creator/extract.py` + `curator/auto_creator/surrogate_verifier.py`.
- **Day 10** — `curator/auto_creator/eval_gate.py` with held-out eval set per domain.
- **Day 11-12** — Cross-project trajectory submission API: `argus.curator.submit_trajectory()`. Polaris and Lyra subscribe.
- **Day 13** — `tests/benchmarks/{equal_budget,active_params}.py`; first numbers vs LobeHub-routing-only and OpenAI-skills-routing-only.
- **Day 14** — Tier 0 retro: how many trajectories? how many promoted skills? how many trust-rejected installs? sign-off for Tier 1.

## §10 — How argus serves every other in-tree project after this plan

Once Tier 0 ships, argus exposes a stable API every other in-tree project consumes (cf. [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md) §4):

- `argus.HostAdapter.route(query, mode)` — the existing routing call.
- `argus.HostAdapter.install_mcp(server_name, user_context)` — Tier 0 trust-gated install.
- `argus.HostAdapter.list_marketplace(filter)` — federated catalog browse.
- `argus.curator.submit_trajectory(trajectory, outcome)` — auto-creator submission endpoint.
- `argus.curator.list_promoted_skills(user_context)` — list skills the curator promoted from this user's trajectories.

Polaris (file 172, 203), Lyra (file 208, V3.8), Atlas-Research, Mentat-Learn (210), Aegis-Ops, Cipher-Sec, Helix-Bio, Orion-Code all subscribe to these APIs.

## §11 — Cross-references

- [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md) — foundational design (50-capability matrix, 44 failure modes, bright-line gates).
- [194-argus-omega-enhanced-design](194-argus-omega-enhanced-design.md), [195-argus-omega-vol-2-trajectory-temporal-horizon](195-argus-omega-vol-2-trajectory-temporal-horizon.md), [196-argus-vs-field-skill-loading-comparison](196-argus-vs-field-skill-loading-comparison.md), [197-argus-omega-vol-3-recursive-skills-curator](197-argus-omega-vol-3-recursive-skills-curator.md) — Omega vols.
- [167-autoskill-experience-driven-lifelong-learning](167-autoskill-experience-driven-lifelong-learning.md), [168-evoskill-coding-agent-skill-discovery](168-evoskill-coding-agent-skill-discovery.md), [169-coevoskills-co-evolutionary-verification](169-coevoskills-co-evolutionary-verification.md), [170-skillrl-recursive-skill-augmented-rl](170-skillrl-recursive-skill-augmented-rl.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md) — skill self-evolution canon.
- [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md), [176-skill-discovery-curator-oss-landscape-may-2026](176-skill-discovery-curator-oss-landscape-may-2026.md), [177-skills-discovery-curator-strongest-2026-techniques](177-skills-discovery-curator-strongest-2026-techniques.md), [178-online-skill-discovery-and-curation-on-the-go](178-online-skill-discovery-and-curation-on-the-go.md), [179-skill-retrieval-routing-and-activation](179-skill-retrieval-routing-and-activation.md) — skill ecosystem context.
- [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md), [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md), [208-lyra-multi-hop-collaborative-apply-plan](208-lyra-multi-hop-collaborative-apply-plan.md) — sibling apply plans.

## §12 — One-paragraph summary

argus's multi-hop + collaborative-AI subsystem ships in three tiers: Tier 0 (14 days) hardens the marketplace + auto-creator surface with LobeHub / Smithery / Glama / FastMCP federation + one-click install with permission negotiation + argus trust verdict + capability schema validation + trajectory-driven skill extractor + surrogate-verifier gating + held-out eval gate + pure-function curator agents + equal-budget benchmarks; Tier 1 (30 days) makes the router user-aware with ICPEA Preference + Experience layers + per-user affinity model + Plan-on-Graph backtracking on Tier-4 navigation + AnchorRAG anchor-prediction + Pareto-frontier + negative-space tiers; Tier 2 (60 days) closes the meta-modification loop with self-referential curator + per-user constitution + alternative-loadout branching + Pages-style review timeline. The five sequencing decisions defended in §7 turn argus from a catalog-aware router into a **user-aware, marketplace-backed, trust-verified, self-curating skill-routing service** that every other in-tree project consumes. Multi-hop QA techniques are mostly off-niche for argus — argus's role is to route to the projects that *have* multi-hop QA substrates, not to multi-hop-reason itself, except at the Tier-4 hierarchical navigation tier where Plan-on-Graph backtracking is a clean fit.

**The one-line takeaway for harness designers:** argus's apply plan is **inverted** — it's mostly a *provider* of marketplace + trust + skill-auto-creation services, with ICPEA-aware routing the only major *consumer* lift; do these three correctly and every other in-tree project gets a teammate-grade skill substrate for one engineering investment.
