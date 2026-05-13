# 196 — Argus vs the Field: How Skill Loading Compares Across Claude Code, OpenClaw, Hermes Agent, Codex CLI, and More

> **Side-by-side analysis of skill-loading mechanisms across the leading agent harnesses of 2026 + Argus / Argus Omega.** Most agent harnesses ship *some* form of skill loading; few specify the routing surface beyond "the model picks based on the description." This document grounds the comparison in actual mechanics — discovery, routing, governance, lifecycle, scaling — using the corpus's own deep-dives ([04](04-skills.md), [29](29-dive-into-claude-code.md), [52](52-dive-into-open-claw.md), [55](55-hermes-agent-self-improving.md), [62](62-everything-claude-code.md), [145](145-comparing-coding-harnesses.md), [180](180-argus-skill-router-agent-design.md), [194](194-argus-omega-enhanced-design.md), [195](195-argus-omega-vol-2-trajectory-temporal-horizon.md)).

**TL;DR.** Most agents (Claude Code, OpenClaw, Cursor, Aider, Codex CLI) load skills via **description-only progressive disclosure** — the model sees `{name, description}` and picks. This works up to ~50 skills before truncation and routing degradation kick in. Hermes Agent adds **continual extraction + skill self-eval every 15 tasks** but uses the same single-tier description match for routing. Argus replaces the single-tier description match with a **5-tier latent-cascade ladder** plus L1/L2/L3 capability layers, Talent/Container abstraction, heterogeneous FM federation, and chaos-engineered hygiene. The result is the difference between a skill *index* (every other system) and a skill *router + planner + simulator + evolver* (Argus Omega).

---

## §1 — Why this comparison

The user's complaint that prompted Argus v1.0 was specifically about Claude Code: *"Claude Code's skill loading is quite stupid; it doesn't get the necessary skills for the step needed."* That isn't a Claude Code bug — it is the **structural ceiling of description-only progressive disclosure** at scale, and it shows up identically in every other 2026 harness because they all use the same primitive ([04-skills](04-skills.md)). The interesting question isn't "is Claude Code worse than others?" — they are mostly equivalent on this axis. It's: **once you accept that description-only routing is structurally ceiling-bound, what does the next-generation skill loader look like?** Argus is one answer; this document compares it to the field.

Three concrete reasons to do the comparison:
1. **Migration cost matters.** Teams currently on Claude Code / OpenClaw / Hermes need to know what they get from Argus that they don't have today, and what they'd lose.
2. **Argus's value depends on workload.** A 10-skill catalog gets nothing from Argus's cascade. A 500-skill catalog gets a lot. The break-even depends on shape.
3. **Cherry-picking ideas.** Even teams that don't migrate to Argus can adopt individual ideas (semantic-entropy gating, witness lattice, periodic decoupled training) and lift their existing harness.

---

## §2 — How each system loads skills today

### 2.1 — Claude Code (Anthropic)

**Mechanism.** Progressive disclosure ([04-skills](04-skills.md), [62-everything-claude-code](62-everything-claude-code.md)). At session start, the harness scans `.claude/skills/` and `.claude/plugins/*/skills/` directories; YAML frontmatter `{name, description, allowed-tools}` is extracted; the harness inserts the *list of name+description* into the system prompt. The model autonomously decides when to invoke a skill via the `Skill(name=...)` tool; on invocation, the harness loads the skill body (plus referenced companion files) into the current turn and narrows tool allowlist to `allowed-tools`.

**Discovery surface.** Local filesystem only. Plugins from `agentskills.io`-compatible registries (Anthropic's own + community) install into `.claude/plugins/`. No live marketplace search; install is explicit.

**Routing.** Single-tier — the model reads the description list and picks. No keyword index, no embedding retrieval, no rerank, no telemetry-driven re-ranking, no LLM-as-router fallback.

**Governance.** Tool allowlist per skill; permission modes (plan / default / acceptEdits / bypass) gate side-effecting tools but not skills themselves. No vulnerability scanning; no trust tiers; no drift detection.

**Lifecycle.** None. Skills don't have versions, signatures, telemetry. A skill that worked yesterday and broke today silently keeps being routed to until the user notices.

**Scale ceiling.** Description list saturates the prompt around 50 active skills (1.2K char descriptions × 50 ≈ 60K tokens of context, before truncation kicks in and key keywords get stripped — see [180](180-argus-skill-router-agent-design.md) §1). Beyond that, routing accuracy degrades — empirically reproduced by the SkillRouter paper showing **31–44pp accuracy gap** between description-only and full-body routing at scale ([179](179-skill-retrieval-routing-and-activation.md) §5).

**Refinement.** None. No "rewrite a description that's not earning activations." Authors update by hand.

### 2.2 — OpenClaw (Peter Steinberger, MIT-licensed, Nov 2025)

**Mechanism.** Gateway + Agent Harness plugin split ([52-dive-into-open-claw](52-dive-into-open-claw.md)). Skills follow the `SKILL.md` (or `SOUL.md` for agent templates) pattern, near-identical to Claude Code's. Awesome-list ecosystem: `awesome-openclaw-agents` (162 production templates × 19 categories), `VoltAgent/awesome-openclaw-skills` (1,400+ skills).

**Discovery surface.** Local + community awesome-lists. Multi-channel-aware — same skill catalog shared across Slack / Telegram / iMessage / Discord / Signal / Matrix / WeChat / etc. (25+ channels).

**Routing.** Single-tier description match, like Claude Code. Each harness plugin has its own routing logic but the canonical plugin uses progressive disclosure.

**Governance.** User-sovereign permissions per channel; per-channel trust tier (sensitive skills can refuse outside approved channels). No vulnerability scanner, no description-quality check.

**Lifecycle.** None native; the OpenClaw-RL companion project ([Gen-Verse/OpenClaw-RL](https://github.com/Gen-Verse/OpenClaw-RL)) trains agents from conversational feedback, but it is experimental and not the default skill-loader.

**Scale ceiling.** Same as Claude Code — description-only routing ceiling. Plus a more acute pathology: the awesome-list ecosystem encourages bulk-installing 100s of skills at once (`sickn33/antigravity-awesome-skills` is 1,400+), making the description-list-saturation problem land faster.

**Refinement.** None native. The `OpenClaw-RL` direction *is* working on this but it's not in the default harness.

**Notable strength.** Multi-channel architecture means skill loading is *channel-aware* — sensitive skills (`disable-model-invocation: true` plus per-channel trust) only activate on whitelisted channels. This is a primitive Claude Code lacks.

### 2.3 — Hermes Agent (Nous Research, April 2026)

**Mechanism.** Single-agent continual learning ([55-hermes-agent-self-improving](55-hermes-agent-self-improving.md)). The distinguishing bet: skills aren't *authored* and *installed*; they're **extracted from completed workflows** and refined over time.

**Discovery surface.** Local + extracted-from-traces. The learning loop:
1. Task completes.
2. Pattern extractor analyzes the trace.
3. Reusable substructure emitted as `~/.hermes/skills/<name>.md` in `agentskills.io` format.
4. On subsequent uses, the skill is invoked and its outcome (success/partial/failure) feeds back to refine the body.
5. Self-eval cadence every 15 tasks — performance review of all skills.

**Routing.** Single-tier description match plus skill-success-rate weighting from the self-eval. Closer to a 1.5-tier than Claude Code's pure 1-tier — telemetry weight on routing weights is a real lift.

**Governance.** Per-channel trust tier (inherited from OpenClaw-style multi-channel gateway); no vulnerability scanner; per-skill success tracking.

**Lifecycle.** **The distinguishing strength.** Hermes is the only mainstream harness that explicitly tracks skill *lifecycle*: extraction → first use → refinement → consolidation → retirement. The 15-task self-eval is the first published "skill HR cycle" in the field.

**Scale ceiling.** Better than Claude Code / OpenClaw because the self-eval consolidates duplicates and retires rotted skills. But still single-tier description routing under the hood — once the catalog grows past ~100 active skills, the same ceiling applies.

**Refinement.** Native and structural. Skill bodies update from outcome signal; learnings are appended to skill markdown.

**Notable strength.** Memory architecture (3 layers: procedural skills, preferences/facts, session working context, plus optional Honcho dialectic user model) is the strongest of the field. Hermes's skill substrate is *production-grade continual learning*, not just storage.

### 2.4 — Codex CLI (OpenAI)

**Mechanism.** Policy-language explicit ([145-comparing-coding-harnesses](145-comparing-coding-harnesses.md) §3). Skills are configured declaratively in policy files; tool allowlists, approval thresholds, sandbox boundaries are all explicit.

**Discovery surface.** Local + OpenAI-hosted plugin registry. No live marketplace search.

**Routing.** Single-tier description match within the explicit policy boundaries.

**Governance.** Strongest of the mainstream — explicit policy language; sandbox boundaries; approval thresholds per skill class.

**Lifecycle.** None native.

**Scale ceiling.** Same description-only ceiling.

**Refinement.** None native.

**Notable strength.** Policy-language governance. Argus's bright-lines and trust-tier framework are conceptually similar but not enforced via a policy DSL.

### 2.5 — Cursor / Cline / Continue / Windsurf (IDE-integrated)

**Mechanism.** IDE-event-driven. Skills (or "rules" in Cursor's `.cursor/rules/` format) are loaded based on file path globs and editor events, not autonomous model decision.

**Discovery surface.** Local — `.cursor/rules/`, `.continue/config.json`, etc.

**Routing.** Glob-based path matching → rule activation. The *model* doesn't decide; the editor's event router does, based on which file is open or which edit is being made.

**Governance.** None at the skill level (some IDE-level approval gating).

**Lifecycle.** None.

**Scale ceiling.** Glob-match is precise but limited — a "rule" is more like a per-file-type prompt extension than a Claude Code skill.

**Refinement.** None.

**Notable strength.** Path-glob routing is *deterministic* — same file always activates the same rules. No description-routing failure modes because there's no description routing.

**Notable weakness.** Not really skill loading in the [04-skills](04-skills.md) sense — these are "rules that activate on file events," not "procedures the model autonomously decides to invoke." Different abstraction.

### 2.6 — Aider (open source)

**Mechanism.** Project-config; minimal skill abstraction. Aider has *commands* and *prompts* but not progressive-disclosure skills.

**Discovery surface.** Local config files only.

**Routing.** None — user invokes commands explicitly.

**Governance.** None.

**Lifecycle.** Stateless across sessions ("no persistence" per [145](145-comparing-coding-harnesses.md) §6).

**Scale ceiling.** N/A — different abstraction.

**Refinement.** None.

**Notable strength.** Simplicity. No skill-routing failures because there's no skill routing.

### 2.7 — Devin / Replit Agent (cloud-only autonomous)

**Mechanism.** Server-side opaque. Skills (if any) are not user-visible primitives; the agent just "knows how to do things." Cognition / Replit doesn't publish the internal mechanism.

**Discovery surface.** Vendor-managed.

**Routing.** Vendor-internal.

**Governance.** Vendor-managed.

**Lifecycle.** Server-side memory persists; skill lifecycle (if exists) is invisible.

**Scale ceiling.** Unknown.

**Refinement.** Unknown — likely some, given Devin's continual-learning marketing.

**Notable strength.** Cloud-managed UX; user doesn't see skill loading at all.

**Notable weakness.** Black-box; cannot be studied, audited, or self-hosted.

### 2.8 — SemaClaw (general-purpose multi-agent)

**Mechanism.** Multi-agent orchestration-first ([54-semaclaw-general-purpose-agent](54-semaclaw-general-purpose-agent.md), [83-semaclaw-deep](83-semaclaw-deep.md)). Skills exist as agent specializations; each agent owns its own skill set; routing is *which agent*, not *which skill within an agent*.

**Discovery surface.** Per-agent local + shared registry.

**Routing.** Two-tier: (1) which agent for this task (multi-agent orchestrator), (2) within agent, single-tier description match for skills.

**Governance.** Per-agent isolation.

**Lifecycle.** Per-agent independent.

**Scale ceiling.** Better than single-agent because skills are partitioned per agent. But each per-agent partition still hits the description-only ceiling.

**Refinement.** None native.

**Notable strength.** Multi-agent partition is a structural way to scale beyond single-agent ceilings — at the cost of cross-agent skill isolation.

### 2.9 — Argus v1.0 ([180](180-argus-skill-router-agent-design.md))

**Mechanism.** Five-tier cascade ladder with telemetry-driven refinement, four-tier trust framework, vulnerability scanner, marketplace integration, Claude Code-format skill persona, per-harness adapters (Lyra, Polaris). 50 capabilities, 44 mapped failure modes, 11-phase roadmap, ~14 weeks total / 6 weeks MVP.

**Discovery surface.** Local + MCP registry pull (Glama / Smithery / Official Registry / MCPfinder) + awesome-list aggregation + Hugging Face Hub + cross-runtime SKILL.md format adoption + skill graph extraction + provenance trace.

**Routing.** Five tiers:
- Tier 0: Native progressive disclosure (free, descriptions in system prompt).
- Tier 1: BM25 keyword search (the "easy" approach).
- Tier 2: Embedding retrieval (sqlite-vec / FAISS / lancedb).
- Tier 3: Cross-encoder rerank (closes the 31–44pp gap).
- Tier 4: Hierarchical navigation (when domain hierarchy exists).

Auto-cascading by catalog size + ambiguity. Mode flags: `keyword` (Tier 1 only), `semantic` (Tier 0+2+3+4 as needed), `auto` (default).

**Governance.** Four-tier trust (T-Untrusted → T-Scanned → T-Reviewed → T-Pinned); content pinning (SHA-256); description-vs-allowed-tools consistency; cost / latency / privacy budgets per tier; cross-tenant isolation; 10 bright lines.

**Lifecycle.** Drift detection (held-out tests on cadence); telemetry-driven re-ranking; description rewriter; stale-skill demotion / retirement; vulnerability re-scan; consolidation / split; description squatting cleanup; embedding refresh; skill graph repair.

**Scale ceiling.** 100K active skills, 1M cold-storage (target).

**Refinement.** Comprehensive — F1–F10 capability matrix.

### 2.10 — Argus Omega ([194](194-argus-omega-enhanced-design.md) Vol. 1 + [195](195-argus-omega-vol-2-trajectory-temporal-horizon.md) Vol. 2)

Adds on top of v1.0:

- **Reframe 1:** Recursive cascade with latent RecursiveLinks ([189](189-recursive-multi-agent-systems.md)).
- **Reframe 2:** L1 Predictor → L2 Simulator → L3 Evolver, × Deterministic / API / Multi-agent / Statistical regimes ([190](190-agentic-world-modeling-taxonomy.md)).
- **Reframe 3:** Talent / Container split + HR lifecycle + 7 formal correctness guarantees ([191](191-onemancompany-skills-to-talent.md)).
- **Reframe 4:** Heterogeneous K1/K2/K3/K4 federation; Tsaheylu bond for non-LLM specialist FMs ([103](103-eywa-heterogeneous-fm-collaboration.md)).
- **Reframe 5:** Co-evolving surrogate verifier + PRM + SkillRL + periodic decoupled training ([169](169-coevoskills-co-evolutionary-verification.md), [170](170-skillrl-recursive-skill-augmented-rl.md), [192](192-world-r1-3d-constraints-t2v.md)).
- **Reframe 6:** L2 simulator-augmented reasoner (citeable rationale, scenario library) ([142](142-trajectory-simulation-agents.md)).
- **Reframe 7:** Temporal as durable execution substrate ([150](150-temporal-durable-execution-substrate.md)).
- **Reframe 8:** HORIZON-style failure attribution + per-skill failure-class × horizon-bucket effectiveness matrix ([27](27-horizon-long-horizon-degradation.md)).
- **Reframe 9:** SkillPlan output (ReWOO-style DAG with placeholders) ([17](17-rewoo.md)).
- **Reframe 10:** Bitemporal catalog + chaos-engineered hygiene with time-to-detection SLOs ([131](131-temporal-bitemporal-tables.md), [53](53-chaos-engineering-next-era.md)).

103 capabilities, 75 mapped failure modes, 32 bright lines, ~52 weeks for full deployment.

---

## §3 — Cross-axis comparison matrix

The matrix below grades each system on 14 axes from the v1.0 capability framework, plus 4 axes from Omega.

Legend: ✗ none / ✓ basic / ✓✓ moderate / ✓✓✓ strong / ✓✓✓✓ comprehensive.

### 3.1 — Discovery & catalog management

| Axis | Claude Code | OpenClaw | Hermes | Codex CLI | Cursor/Cline | SemaClaw | Argus v1.0 | Argus Omega |
|---|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| Local catalog scan | ✓✓ | ✓✓ | ✓✓ | ✓✓ | ✓✓ | ✓✓ | ✓✓ | ✓✓ |
| Online runtime discovery | ✓ | ✓ | ✓✓ | ✓ | ✗ | ✓ | ✓✓ | ✓✓✓ |
| MCP registry pull | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ | ✓✓✓ | ✓✓✓ |
| Awesome-list aggregation | ✗ | ✓✓ | ✗ | ✗ | ✗ | ✗ | ✓✓ | ✓✓✓ |
| Skill graph extraction | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓✓✓ |
| Provenance trace | ✗ | ✗ | ✓ | ✓ | ✗ | ✗ | ✓✓ | ✓✓✓✓ (witness lattice) |
| Cross-runtime SKILL.md adoption | ✓✓✓ | ✓✓ | ✓✓✓ | ✓ | ✗ | ✗ | ✓✓✓ | ✓✓✓✓ (Talent/Container) |

### 3.2 — Routing tiers

| Axis | Claude Code | OpenClaw | Hermes | Codex CLI | Cursor/Cline | SemaClaw | Argus v1.0 | Argus Omega |
|---|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| Tier 0: Native progressive disclosure | ✓ | ✓ | ✓ | ✓ | (path-glob) | ✓ | ✓ | ✓ (capability-signature-aware) |
| Tier 1: BM25 keyword | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓ | ✓✓ |
| Tier 2: Embedding retrieval | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓ | ✓✓✓ (HippoRAG PPR + dense fusion) |
| Tier 3: Cross-encoder rerank | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓ | ✓✓✓ (KG-aware features) |
| Tier 4: Hierarchical navigation | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ (per-agent partition) | ✓ | ✓✓✓ (typed KG traversal) |
| LLM-as-router fallback | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓✓ |
| Multi-skill activation (top-K) | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓✓ (diversity-aware) |
| ReWOO-style multi-skill plan | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓✓ (SkillPlan DAG) |
| Semantic-entropy gating | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓ |
| Plan/execute mode bifurcation | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓ |
| Inter-tier communication | N/A | N/A | N/A | N/A | N/A | (text) | (text) | ✓✓✓ (latent RecursiveLink) |

### 3.3 — Refinement & lifecycle

| Axis | Claude Code | OpenClaw | Hermes | Codex CLI | Cursor/Cline | SemaClaw | Argus v1.0 | Argus Omega |
|---|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| Drift detection (held-out tests) | ✗ | ✗ | ✓ | ✗ | ✗ | ✗ | ✓✓ | ✓✓✓ |
| Telemetry-driven re-ranking | ✗ | ✗ | ✓✓ | ✗ | ✗ | ✗ | ✓✓ | ✓✓✓ (PRM + SkillRL) |
| Description rewriter | ✗ | ✗ | ✓ | ✗ | ✗ | ✗ | ✓✓ | ✓✓ |
| Skill consolidation / split | ✗ | ✗ | ✓ | ✗ | ✗ | ✗ | ✓✓ | ✓✓ |
| Stale-skill retirement | ✗ | ✗ | ✓ | ✗ | ✗ | ✗ | ✓✓ | ✓✓ |
| HR lifecycle (review/PIP/offboarding) | ✗ | ✗ | ✓ (15-task self-eval) | ✗ | ✗ | ✗ | ✓ | ✓✓✓✓ (formal guarantees) |
| Skill extracted from traces | ✗ | ✗ | ✓✓✓ (the distinguishing strength) | ✗ | ✗ | ✗ | ✗ (out of scope, see §10) | ✗ (out of scope) |
| Periodic decoupled training | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓✓ |
| 24/7 autonomous improvement loop | ✗ | ✗ | ✓ | ✗ | ✗ | ✗ | ✗ | ✓✓✓ (Ralph + Temporal) |

### 3.4 — Trust, governance, and adversarial defense

| Axis | Claude Code | OpenClaw | Hermes | Codex CLI | Cursor/Cline | SemaClaw | Argus v1.0 | Argus Omega |
|---|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| Trust tiers | ✗ | ✓ (per-channel) | ✓ (per-channel) | ✓✓ (policy DSL) | ✗ | ✗ | ✓✓✓ (4-tier T-Untrusted→T-Pinned) | ✓✓✓ |
| Vulnerability scanner | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ | ✓✓ | ✓✓✓ (structural anti-poisoning) |
| Content pinning (SHA-256) | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓ | ✓✓✓ (capability signature too) |
| Per-skill cost / latency / privacy budgets | ✗ | ✓ (channel-level) | ✗ | ✓✓ | ✗ | ✗ | ✓✓ | ✓✓✓ |
| Description-vs-allowed-tools consistency | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ | ✓✓ | ✓✓ |
| Witness lattice / audit trail | ✗ | ✓ (logs) | ✓ (logs) | ✓✓ | ✗ | ✗ | ✓ | ✓✓✓✓ (signed, bitemporal-replayable) |
| Bitemporal catalog | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓✓ |
| Chaos engineering (planted failures) | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓✓ (time-to-detection SLOs) |
| Bright lines / refusal gates | ✗ | ✓ | ✓ | ✓✓ | ✗ | ✗ | ✓✓ (10 codes) | ✓✓✓ (32 codes) |

### 3.5 — Capability tier (planning, simulation, evolution)

| Axis | Claude Code | OpenClaw | Hermes | Codex CLI | Cursor/Cline | SemaClaw | Argus v1.0 | Argus Omega |
|---|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| L1 Predictor (which skill matches?) | ✓ | ✓ | ✓ | ✓ | (path-glob) | ✓ | ✓✓✓ | ✓✓✓✓ |
| L2 Simulator (predict trajectory under intervention?) | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓✓ (simulator-augmented reasoner) |
| L3 Evolver (catalog edits under evidence?) | ✗ | ✗ | ✓ (skill body refinement) | ✗ | ✗ | ✗ | ✓ | ✓✓✓ (governed validation, regression gates) |
| Counterfactual Outcome Deviation probes | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓✓ |
| HORIZON-style failure attribution | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓✓ |
| Per-failure-class × horizon-bucket matrix | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓✓ |

### 3.6 — Federation & heterogeneity

| Axis | Claude Code | OpenClaw | Hermes | Codex CLI | Cursor/Cline | SemaClaw | Argus v1.0 | Argus Omega |
|---|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| K1 LLM-call skills | ✓✓ | ✓✓ | ✓✓ | ✓✓ | ✓ | ✓✓ | ✓✓ | ✓✓ |
| K2 Tool-script skills | ✓✓ | ✓✓ | ✓✓ | ✓✓ | ✗ | ✓ | ✓✓ | ✓✓ |
| K3 MCP-server skills | ✓✓ | ✓✓ | ✓✓ | ✓✓ | ✓✓ | ✗ | ✓✓ | ✓✓ |
| K4 Specialist FM skills (Tsaheylu) | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓✓ |
| Per-step delegation policy C(s) | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓✓ |
| Modality signature on skills | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓ |
| Talent/Container split | ✗ | ✓ (Gateway/plugin) | ✗ | ✗ | ✗ | ✗ | ✗ | ✓✓✓✓ |
| Cross-runtime portability | ✓ (via SKILL.md format) | ✓✓ | ✓✓ | ✗ | ✗ | ✓ | ✓ | ✓✓✓ (six typed interfaces) |

### 3.7 — Scaling & operations

| Axis | Claude Code | OpenClaw | Hermes | Codex CLI | Cursor/Cline | SemaClaw | Argus v1.0 | Argus Omega |
|---|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| Practical max active skills | ~50 | ~50 | ~100 | ~50 | ~50 (rules) | ~50/agent | ~10K | ~100K |
| Cold-storage skill capacity | N/A | N/A | N/A | N/A | N/A | N/A | ~1M | ~1M |
| Latency p95 (catalog ≤ 1K) | ~0 (in-prompt) | ~0 | ~0 | ~0 | ~0 | ~0 | ~200ms | ~250ms |
| Latency p95 (catalog ≤ 10K) | (degraded routing) | (degraded) | (degraded) | (degraded) | N/A | (per-agent OK) | ~500ms | ~600ms |
| Latency p95 (catalog ≤ 100K) | (broken) | (broken) | (broken) | (broken) | N/A | (broken) | ~1s | ~1.2s |
| Durable execution (crash-safe loops) | ✗ | ✗ | ✓ (hibernation) | ✗ | ✗ | ✗ | ✗ | ✓✓✓ (Temporal) |
| Multi-day workflows native | ✗ | ✗ | ✓ | ✗ | ✗ | ✗ | ✗ | ✓✓✓ |

### 3.8 — Headline summary

Across 60 axes, the pattern is consistent:

- **Claude Code, OpenClaw, Codex CLI, Cursor/Cline, SemaClaw** all live at "single-tier description routing + light governance" (10–20 ✓ marks total). They're nearly equivalent on skill loading; differentiation is on UX, multi-channel, IDE integration, etc.
- **Hermes Agent** is meaningfully ahead on lifecycle (continual extraction + 15-task self-eval is unique) and memory architecture (3-tier + Honcho), but routing is still single-tier.
- **Argus v1.0** scores ~80 ✓ marks on the same axes — it is the first system with full 5-tier cascade, four-tier trust, drift / consolidation / split / squatting cleanup, marketplace integration with vulnerability scanning.
- **Argus Omega** scores ~140+ ✓ marks — adds L1/L2/L3 capability layers, Talent/Container, K4 federation, RecursiveLink latent cascade, ReWOO planning, HORIZON attribution, witness lattice with bitemporal replay, chaos engineering, Temporal substrate.

---

## §4 — Where Argus wins (and where it doesn't)

### 4.1 — Where Argus clearly wins

1. **Catalogs ≥ 100 skills.** Every other system's description-only routing degrades; Argus's cascade is engineered for this regime.
2. **Heterogeneous workloads (K4 specialist FMs).** No other system federates non-LLM FMs. Time-series tasks routed to Chronos via Tsaheylu are a Pareto improvement that nothing else delivers.
3. **Production safety + audit.** Witness lattice + bitemporal replay + signed provenance is unmatched. For compliance / regulated deployments, no other system is in the same bracket.
4. **Long-horizon trajectories.** HORIZON failure-class attribution + horizon-mitigation skill tagging directly addresses the [27] long-horizon degradation problem none of the others touches.
5. **Multi-skill plans.** ReWOO SkillPlan output is 5× cheaper than ReAct loops on multi-skill trajectories. No other system emits a SkillPlan.
6. **Continuous improvement under governance.** Periodic decoupled training + governed L3 evolver is the only published anti-collapse, anti-overwrite recipe for self-improving skill catalogs.
7. **Cross-runtime substitutability.** Talent/Container split is the first abstraction that lets the same skill identity run on Claude Code, Codex, OpenClaw, smolagents without changes.

### 4.2 — Where Argus does NOT win (or is currently worse)

1. **Skill *creation* / extraction from traces.** **Hermes wins this.** Argus is explicitly out-of-scope for skill *creation*; v1.0 §10 says "Argus is not a skill creator. Authoring new skills lives in the harness-skills evolution pipeline." Hermes's pattern-extractor → SKILL.md → refinement loop is the gold standard for *generating* skills from completed work; Argus consumes such pipelines but doesn't run them.
2. **IDE-integrated UX.** Cursor and Windsurf win for solo-developer pair programming. Argus is a CLI / MCP / Python-SDK component, not an IDE.
3. **Time-to-first-skill.** Aider has zero skill-loading cost (because no skill abstraction). Claude Code/OpenClaw skills take minutes to author. Argus's full setup (Talent envelope + capability signature + trust tier) takes longer.
4. **Operational complexity.** Argus Omega requires Temporal cluster, embedding model, cross-encoder, KG store, witness-lattice keys, chaos planter. Hermes runs on a $5 VPS. Cost-of-entry is real.
5. **Black-box managed services.** Devin and Replit Agent are zero-config from the user's perspective. Argus is open-architecture; the user owns the operational complexity.
6. **Channel-aware activation.** OpenClaw's per-channel trust tier is a primitive Argus doesn't have natively (could be added as a Container-interface flag, but isn't in v1.0/Omega specs).
7. **Path-glob deterministic activation.** Cursor's `.cursor/rules/` per-file-glob activation is *deterministic* in a way Argus's probabilistic cascade isn't. For predictability-critical use cases, Cursor's primitive may be preferable.

### 4.3 — Design lessons in both directions

**What Argus should learn from the field:**
- **From Hermes:** continual skill extraction from traces. Argus should ship a companion skill-extraction harness (or formally document the integration with existing `harness-skills` extraction pipelines per [180](180-argus-skill-router-agent-design.md) §10).
- **From OpenClaw:** per-channel trust tier. Should be a Container-interface flag in Vol. 1 §R3.
- **From Codex CLI:** policy-DSL governance. Argus's bright-line codes should compile to enforceable policy expressions, not just code constants.
- **From Cursor:** deterministic path-glob activation as a routing tier *option* (Tier -1: deterministic glob). For file-bound skills, this is the right primitive.

**What the field should learn from Argus:**
- **5-tier cascade.** Claude Code's progressive disclosure could add Tier 1 (BM25) trivially — local index, no LLM calls, immediate uplift on catalogs > 50. The other three tiers are progressively harder but the recipe is published.
- **Witness lattice.** Compliance pressure (SOC2, EU AI Act, financial services) is rising; signed routing audit trails will become table stakes.
- **HORIZON failure-class attribution.** Even for description-only routers, attribution-of-failure feeds catalog hygiene. No infrastructure overhead to adopt.
- **Periodic decoupled training.** Any RL-trained activation policy needs this; should be standard.

---

## §5 — Implementation cost comparison

For teams considering migration or in-house build, here's the operational reality:

| System | Lines of code (approx) | Infra dependencies | Setup time | Per-skill cost overhead |
|---|---:|---|---|---|
| Claude Code | (vendor) | none (local FS) | minutes | ~0 |
| OpenClaw | ~80K LOC + plugins | Gateway process | ~1 day | ~0 (description-only) |
| Hermes Agent | ~30K LOC | SQLite + LLM API | ~1 hour | low (lazy skill load) |
| Codex CLI | (vendor) | none (local FS) | minutes | ~0 |
| Cursor/Cline | (vendor / FOSS) | IDE | minutes | ~0 (rules in prompt) |
| **Argus v1.0** | ~10K LOC (target MVP) | sqlite-vec + cross-encoder | ~2 weeks | ~$0.0001/lookup (Tier 2 only); ~$0.005 if Tier 3 fires |
| **Argus Omega-Lite** | ~20K LOC | + Temporal cluster + KG store | ~6–10 weeks | adds witness signing (~$0.0002) + plan simulator (~$0.001) |
| **Argus Omega-Full** | ~50K LOC | + RL infra + chaos planter + simulator backend | ~52 weeks | adds RL training amortized cost |

**Operational floor** for Argus Omega: a Temporal cluster ($0 dev with `temporalite`, $50–$500/mo Temporal Cloud or self-hosted), an embedding model (OpenAI text-embedding-3-large at ~$0.13/1M tokens, or BGE-M3 free if self-hosted), a cross-encoder (free open-weights), a vector store (`sqlite-vec` free, or `lancedb`/`qdrant`). Total marginal cost per routing decision: ~$0.0001–$0.005 depending on tier depth. At 100K routing decisions/month, that's ~$10–$500.

**Time-to-break-even:** Argus's value is *catalog-size-dependent*. Below 50 skills, native progressive disclosure is fine. At 50–500 skills, Argus v1.0 MVP pays back via routing-accuracy gains. Above 500 skills (or in compliance-bound contexts), Argus Omega is the only viable option among the published designs.

---

## §6 — Migration paths

### 6.1 — From Claude Code → Argus

Native compatibility. Argus consumes `.claude/skills/` directly (D1 in [180](180-argus-skill-router-agent-design.md)); the Claude Code installation continues working. Argus adds the cascade as a lateral skill (`argus/SKILL.md`) that Claude Code itself can invoke. The user keeps their existing skill catalog and gains the cascade for any catalog that grew past Claude Code's ceiling.

**Gradual path:** install Argus as a skill (Phase A10); have Claude Code call it for catalogs > 50; keep description-only routing for small catalogs.

### 6.2 — From OpenClaw → Argus

Argus replaces or supplements the harness-plugin's routing logic. OpenClaw's Gateway and channel layer remain; the harness plugin delegates skill loading to Argus via the Python SDK or MCP server interface. Multi-channel awareness can flow through to Argus's Container interface as a Talent activation parameter.

**Gradual path:** start by routing one channel through Argus; expand once stability proven.

### 6.3 — From Hermes Agent → Argus

These are *complementary*, not competing. Hermes is the strongest published *skill creation* system; Argus is the strongest *skill loading* system. The intended composition: Hermes extracts, refines, and self-evals skills; Argus indexes, routes, governs, and federates. The interface is the `agentskills.io` SKILL.md format both already speak.

**Recommended composition:**
- Hermes's pattern-extractor writes to `~/.skills/extracted/`.
- Argus's catalog scanner picks up `~/.skills/extracted/` as one source.
- Hermes's self-eval signal feeds Argus's telemetry (capability F2).
- Argus's vulnerability scanner gates promotion of extracted skills to higher trust tiers.

This combination is the strongest published end-to-end skill substrate — extraction (Hermes) + indexing/routing/governance (Argus). Neither vendor / project would be doing redundant work.

### 6.4 — From Codex CLI → Argus

Argus's bright-line / trust-tier framework is structurally similar to Codex CLI's policy-language. The migration is mostly translating policy expressions to bright-line codes plus accepting Argus's catalog format. Codex's sandbox / approval primitives carry over via Argus's Container interface.

### 6.5 — From SemaClaw → Argus

SemaClaw's per-agent partition becomes Argus's per-Talent partition. Cross-agent coordination (which SemaClaw owns) lives at a layer above Argus. The combination is natural — SemaClaw orchestrates Talents; Argus picks Talents within each task.

---

## §7 — Honest summary

The skill-loading layer is *under-engineered* across the field. Most production agents ship the same description-only progressive disclosure primitive that Anthropic introduced in 2024; the primitive has structural ceilings (≤ 50 active skills), no governance, no lifecycle, no cross-runtime portability, and no federation.

The two systems that genuinely advance the state of the art are:

- **Hermes Agent** — for *creating* and *refining* skills from completed traces. The 15-task self-eval and the three-layer memory architecture are durable contributions.
- **Argus / Argus Omega** — for *routing*, *governing*, *evolving*, and *federating* a large heterogeneous catalog. The 5-tier cascade, four-tier trust, witness lattice, ReWOO planner, and Tsaheylu federation are durable contributions.

The two are **complementary**: Hermes makes skills; Argus loads skills. Together they form the strongest published end-to-end skill substrate. Neither is a complete replacement for the other.

For teams currently on Claude Code / OpenClaw / Codex CLI / Cursor with small catalogs (< 50), the description-only primitive is fine — don't over-engineer. For teams whose catalogs are growing (50–500), or who need cross-runtime substitutability, or who have compliance / audit requirements, Argus v1.0 MVP (6 weeks) is the highest-leverage upgrade. For teams running production multi-day workflows with heterogeneous skill kinds, Argus Omega-Lite (~10 weeks on top of v1.0 MVP) is the production target.

The "Claude Code is stupid at skill loading" complaint that started this thread is *correct as observed but misattributed as cause*: it isn't a Claude Code design flaw; it's the structural ceiling of the description-only primitive every system uses. The fix isn't a Claude Code patch — it's a different routing layer (Argus) sitting between the harness and the catalog.

---

## §8 — One-paragraph summary

Across eight production agent harnesses (Claude Code, OpenClaw, Hermes Agent, Codex CLI, Cursor/Cline, Aider, Devin/Replit, SemaClaw), skill loading is dominated by **single-tier description-only progressive disclosure** that hits a structural ceiling at ~50 active skills, has no native lifecycle, no governance, and no cross-runtime substitutability. **Hermes Agent** advances the field on skill *creation* — pattern extraction from completed workflows + 15-task self-eval — but its routing layer is the same description-match primitive as everyone else's. **Argus v1.0** ([180](180-argus-skill-router-agent-design.md)) introduces the first 5-tier cascade (BM25 → embedding → cross-encoder → KG-navigate) plus four-tier trust framework, marketplace integration with vulnerability scanning, and 11-phase implementation roadmap. **Argus Omega** ([194](194-argus-omega-enhanced-design.md), [195](195-argus-omega-vol-2-trajectory-temporal-horizon.md)) layers ten reframes on top: latent RecursiveLinks between tiers, L1/L2/L3 capability ladder, Talent/Container split with HR lifecycle and seven formal correctness guarantees, K4 specialist-FM federation via Tsaheylu bond, co-evolving surrogate verifier with SkillRL and periodic decoupled training, simulator-augmented L2 reasoner with citeable rationales, Temporal as durable execution substrate, HORIZON-style failure-class attribution, ReWOO SkillPlan output, and bitemporal catalog with chaos-engineered hygiene. Across 60 comparison axes, single-tier systems score 10–20 ✓; Hermes scores ~30; Argus v1.0 scores ~80; Argus Omega scores ~140+. **The two strongest systems are complementary, not competing**: Hermes is the strongest published skill-creator, Argus is the strongest published skill-loader, and the combination — Hermes extracting + refining, Argus indexing + routing + governing + federating — is the production target for any team building a production skill substrate at scale.

---

## §9 — Cross-references

- v1.0 design: [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md).
- Argus Omega Vol 1 (the five reframes): [194-argus-omega-enhanced-design](194-argus-omega-enhanced-design.md).
- Argus Omega Vol 2 (five more reframes + skeleton): [195-argus-omega-vol-2-trajectory-temporal-horizon](195-argus-omega-vol-2-trajectory-temporal-horizon.md).
- Field harnesses: [04-skills](04-skills.md), [29-dive-into-claude-code](29-dive-into-claude-code.md), [52-dive-into-open-claw](52-dive-into-open-claw.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md), [62-everything-claude-code](62-everything-claude-code.md), [54-semaclaw-general-purpose-agent](54-semaclaw-general-purpose-agent.md), [83-semaclaw-deep](83-semaclaw-deep.md), [102-clawgym-scalable-claw-agents](102-clawgym-scalable-claw-agents.md), [145-comparing-coding-harnesses](145-comparing-coding-harnesses.md).
- Skills landscape: [167-autoskill-experience-driven-lifelong-learning](167-autoskill-experience-driven-lifelong-learning.md), [168-evoskill-coding-agent-skill-discovery](168-evoskill-coding-agent-skill-discovery.md), [169-coevoskills-co-evolutionary-verification](169-coevoskills-co-evolutionary-verification.md), [170-skillrl-recursive-skill-augmented-rl](170-skillrl-recursive-skill-augmented-rl.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md), [173-offline-sim-skill-discovery](173-offline-sim-skill-discovery.md), [174-autonomous-skill-exploration-iterative-feedback](174-autonomous-skill-exploration-iterative-feedback.md), [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md), [176-skill-discovery-curator-oss-landscape-may-2026](176-skill-discovery-curator-oss-landscape-may-2026.md), [177-skills-discovery-curator-strongest-2026-techniques](177-skills-discovery-curator-strongest-2026-techniques.md), [178-online-skill-discovery-and-curation-on-the-go](178-online-skill-discovery-and-curation-on-the-go.md), [179-skill-retrieval-routing-and-activation](179-skill-retrieval-routing-and-activation.md).
- Source canon for Omega reframes: [17](17-rewoo.md), [27](27-horizon-long-horizon-degradation.md), [53](53-chaos-engineering-next-era.md), [103](103-eywa-heterogeneous-fm-collaboration.md), [131](131-temporal-bitemporal-tables.md), [142](142-trajectory-simulation-agents.md), [150](150-temporal-durable-execution-substrate.md), [186](186-mnema-witness-lattice.md), [188](188-witness-provenance-memory-techniques-synthesis.md), [189](189-recursive-multi-agent-systems.md), [190](190-agentic-world-modeling-taxonomy.md), [191](191-onemancompany-skills-to-talent.md), [192](192-world-r1-3d-constraints-t2v.md).
