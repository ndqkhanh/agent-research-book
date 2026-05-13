# HARNESS-SKILLS — Cross-Project Skill Self-Evolution Integration Plan

> **Goal.** Apply the 2026 skill-self-evolution wave (AutoSkill / EvoSkill / CoEvoSkills / SkillRL — see [docs/167–171](docs/171-skill-self-evolution-2026-synthesis.md)) across **all 14 projects** in `projects/`, preserving each project's existing discipline (cross-model adversarial pair, bright-line gates, structural provenance, file-first persistence) and matching each project to the *correct corner* of the skill-evolution design space.
>
> **This document is a plan, not an implementation.** Read alongside [docs/171-skill-self-evolution-2026-synthesis.md](docs/171-skill-self-evolution-2026-synthesis.md) (the design-space map) and [`projects/polaris/POLARIS_V2_1_SOURCE_FEDERATION_PLAN.md`](projects/polaris/POLARIS_V2_1_SOURCE_FEDERATION_PLAN.md) (the closest precedent).

---

## §0 — Objective and non-goals

### Objective

Integrate skill self-evolution into every project in `projects/` so that:

1. Every project that produces *artifacts* (claims, proofs, runbooks, reports, replies, fixes) extracts repeatable patterns into versioned `SKILL.md` files.
2. Every project that produces *failures* (rejected claims, failing proofs, denied operations, hallucinations, bugs) distils them into negative-space skills (`common_mistakes`-style) so the next attempt reads them first.
3. Skills produced by one project's domain shell can be *consumed* by neighbouring projects where applicable — the trust-tiered, provenance-bound skill substrate built in [Polaris P21–P27](projects/polaris/POLARIS_V2_1_SOURCE_FEDERATION_PLAN.md) is the cross-project federation layer.
4. Bright-line discipline is **preserved verbatim** — no project bypasses its own gates, no skill auto-promotes without HITL, no cross-project skill transfer without explicit consent.

### Non-goals

1. **Not** a unification project. Each project keeps its own runtime, its own tools, its own ethics gates. Skills cross project boundaries via *artifact transfer*, not via shared imports.
2. **Not** a centralised marketplace. Promotion is per-project and gated by project-specific bright-lines.
3. **Not** a model-training project (except SkillRL on open-weights projects). The default integration is frozen-weights skill evolution.
4. **Not** a refactor of existing skill infrastructure. `polaris-skills`, `lyra-skills`, `open-fang/skills/` stay where they are. We add evolution loops, we do not rearrange folders.
5. **Not** a replacement of project-specific safety perimeters. Cipher-Sec keeps its scope-authorisation; Helix-Bio keeps its dual-use perimeter; Aegis-Ops keeps its policy engine. Skills run *inside* the perimeter, never around it.

---

## §1 — Project landscape and design-space mapping

The 14 projects fall into clear categories. Numbers from each project's README:

### A. Already-integrated (skills + evolution exist)

| Project | Existing skill infra | Notes |
|---|---|---|
| **polaris** | `polaris-skills` package, P10 auto-creator, P22-P27 source federation (this turn's work) | Full skill lifecycle; v2.1 ships federation. Reference implementation. |
| **lyra** | `lyra-skills` package (extractor, curator, loader, activation, installer, router, review, packs) | Substantial, no evolution loop yet. Static skill library. |
| **open-fang** | `skills/` dir with citation-extraction, claim-localization, counter-example-generation, peer-review, reproduction-script | 5 hand-authored skills, MCP-served, no auto-creation. |

### B. Candidate for AutoSkill (frozen-weights, dialogue/preference)

| Project | One-line shape | Why AutoSkill fits |
|---|---|---|
| **mentat-learn** | Self-improving personal assistant (Hermes-style closed skill loop in design) | Per-user dialogue; consent-gated memory; literally describes Hermes-style skill extraction. *Direct AutoSkill target.* |
| **harmony-voice** | Dual-agent voice (Slow Thinker + Fast Talker + Semantic Cache) | Per-user voice preferences (verbosity, formality, language); short turns; high session count. |
| **cipher-sec** | Security agent under scope-authorisation | Analyst preferences (technique selection, evidence-style, reporting cadence) — but skill drift is a safety risk; needs trust-tier gating. |

### C. Candidate for EvoSkill / CoEvoSkills (frozen-weights, coding/operational, ground-truth)

| Project | One-line shape | Why EvoSkill / CoEvoSkills fits |
|---|---|---|
| **orion-code** | Autonomous coding agent (MVP) | Has tests as ground truth; benchmark-driven; failure-mode-rich. *Direct EvoSkill target.* |
| **lyra** | General-purpose CLI coding harness | Same shape as Orion but more mature. CoEvoSkills' multi-file packages fit the existing `lyra-skills/packs/`. |
| **aegis-ops** | SRE / Ops runbooks | Failure-driven (incident traces); deterministic verifier (audit chain); EvoSkill loop natural. |
| **cipher-sec** | Security techniques | Some pen-test patterns are ground-truth-verifiable (CVE replay, capture-the-flag); EvoSkill for those. |
| **quanta-proof** | Formal proof agent | Lean/Coq verifier IS the ground-truth oracle; tactic-sequence skills evolve via failure analysis. *Strong EvoSkill fit.* |

### D. Candidate for Ctx2Skill / AutoSkill4Doc (document-derived)

| Project | One-line shape | Why document-derived fits |
|---|---|---|
| **atlas-research** | Long-horizon research agent (ReWOO + verifier) | Reads papers, retrieves docs; AutoSkill4Doc converts each absorbed paper into a procedural skill. |
| **helix-bio** | Biomedical research, ontology-bound | Protocols, regulatory documents, clinical-trial registries — Ctx2Skill is the right shape. |
| **quanta-proof** | Math + Lean | Theorem statements + textbook proofs as Ctx2Skill input. Pairs with EvoSkill side. |

### E. Candidate for SkillRL (RL-trained, open-weights)

| Project | One-line shape | Caveat |
|---|---|---|
| **open-fang** | DAG Teams research agent, 9-specialist cohort, weekly cron | Atropos-compatible trajectory export is already there. SkillRL fits *if* a fine-tuning loop is desired (currently unspecified). |
| **vertex-eval** | Third-party evaluation platform | *Hosts* the SkillBench-style benchmark, doesn't consume skills directly. Pairs with the others as the *measurement* layer. |

### F. Substrate / orchestration (skill-aware but not skill-consumer)

| Project | One-line shape | Role in the skill ecosystem |
|---|---|---|
| **gnomon** | Harness-aware evaluator + closed evolution loop, portable Harness IR | *The trace substrate.* Already attributes failures to primitives; this is the data source skill-extractors read. Bundles committed patches as Claude-Code SKILL.md in MVP — meaning gnomon already produces skills. |
| **syndicate** | Multi-agent platform (Permissions, Handoffs, Visibility) | *The skill-exchange protocol.* When skills move between projects, syndicate is the right transport for permission + audit. |
| **vertex-eval** | Evaluator | *The benchmark.* Pass@k / Pass^k / HORIZON attribution measures skill quality across projects. |

### Design-space corners (each project plotted)

```
                     dialogue / preference        coding / operational        document-derived
                     ───────────────────────      ───────────────────────     ────────────────────────
frozen-weights      mentat-learn                  orion-code                   atlas-research
                    harmony-voice                 lyra                         helix-bio
                    cipher-sec*                   aegis-ops                    quanta-proof (Ctx side)
                                                  cipher-sec*
                                                  quanta-proof (Evo side)

RL-trained          —                             open-fang (if SFT/RL)       —
                                                  
substrate / orch.   gnomon (trace) · syndicate (transport) · vertex-eval (benchmark)
already integrated  polaris (full pipeline) · lyra (skills, no evolution) · open-fang (5 static skills)
```

\* Cipher-Sec appears in two corners because some techniques are dialogue-driven (analyst prefs) and some are ground-truth-verifiable (CVE replay).

---

## §2 — Shared infrastructure: `harness-skills`

### The decision

The existing repo convention is **vendored shared code, not a single shared package** — `harness_core/` is duplicated across atlas-research, helix-bio, gnomon, vertex-eval, orion-code, quanta-proof, open-fang. This plan respects that convention. Each project gets its own copy of `harness-skills/` with project-specific overrides.

### What `harness-skills` ships

A reference implementation distilled from `polaris-skills.auto_creator` + the four 2026 papers:

```
harness-skills/                           # ship in: research/harness-engineering/packages/
├── pyproject.toml
├── README.md
└── src/harness_skills/
    ├── __init__.py
    ├── extract/
    │   ├── dialogue.py                   # AutoSkill-style: P_ext / P_judge / P_merge
    │   ├── failure.py                    # EvoSkill-style: failure-driven proposer
    │   ├── document.py                   # AutoSkill4Doc / Ctx2Skill paper-to-skill
    │   └── trace.py                      # gnomon-HIR-trace-to-skill
    ├── verify/
    │   ├── surrogate.py                  # CoEvoSkills surrogate verifier
    │   ├── ground_truth.py               # EvoSkill ground-truth oracle adapter
    │   └── triangulate.py                # P25-style cross-source triangulation
    ├── store/
    │   ├── skill_bank.py                 # JSONL + Markdown layout
    │   ├── versioning.py                 # SemVer-ish merge bookkeeping
    │   └── trust_tier.py                 # P24-style tier labels (T1/T2/T3/RED/legacy)
    ├── retrieve/
    │   ├── hybrid.py                     # dense + BM25, λ-mix, threshold η
    │   └── domain_priority.py            # P26-style per-domain source priority
    ├── promote/
    │   ├── pareto_frontier.py            # EvoSkill k=3 frontier
    │   ├── coevolutionary.py             # CoEvoSkills generator-verifier loop
    │   └── eval_gate.py                  # held-out eval before BL-PROMOTE-SKILL
    ├── hooks/
    │   ├── claim_gate.py                 # composes with project's BEFORE_CLAIM hook
    │   └── trust_gate.py                 # P24 RED-tier refusal hook
    └── adapters/
        ├── claude_code.py                # PostToolUse / SessionEnd / SessionStart hooks
        ├── openai_proxy.py               # AutoSkill-style OpenAI-compatible proxy
        └── mcp.py                        # MCP-server adapter (skill = MCP tool)
```

### Six adapters per project

Each project picks 1-3 of these adapters depending on its corner:

| Adapter | Used by |
|---|---|
| `extract.dialogue` | mentat-learn, harmony-voice, cipher-sec (dialogue side) |
| `extract.failure` | orion-code, lyra, aegis-ops, quanta-proof, cipher-sec (technique side) |
| `extract.document` | atlas-research, helix-bio, quanta-proof |
| `extract.trace` | gnomon (it produces traces), all others (they consume) |
| `promote.pareto_frontier` | EvoSkill-corner projects |
| `promote.coevolutionary` | CoEvoSkills-corner projects |

### Cross-cutting invariants `harness-skills` enforces

1. **Cross-model adversarial pair** — `extract.*` and `verify.*` always run on different model families (Polaris ADR-002 lifted into the shared library).
2. **Trust tier on every skill** — every skill produced has a `trust_tier` field; the default is `T2-PREPRINT` (auto-extracted, not yet verified) or `legacy` (pre-existing static skill).
3. **No silent promotion** — skills enter the active library only via `promote.eval_gate` + project's `BL-PROMOTE-SKILL`-style HITL gate.
4. **Provenance on every skill** — extraction trace, source artefact (dialogue id, paper DOI, failure trace id), reviewer verdict.
5. **Filesystem-first** — `SKILL.md` + sidecar JSONL versioning ledger; no DB in the critical path.

---

## §3 — Per-project integration plan

Each section: *current state → target capability → integration steps → bright-lines added → success criteria → effort estimate*.

### 3.1 polaris (already integrated — touch-up only)

**Current state.** P10 auto-creator + P21–P27 source federation, trust-tiering, cross-source triangulation, domain-aware routing, author/lab memory all designed and partially implemented.

**Target.** Lift the auto-creator logic into the shared `harness-skills` reference. No functional change.

**Steps.**
1. Extract `polaris-skills.auto_creator.*` into `harness-skills.extract.dialogue` (since polaris's auto-creator is the closest existing analogue).
2. Re-import from `harness-skills` in `polaris-skills`; deprecation-shim the old import path.

**Bright-lines added.** None.

**Success criteria.** All existing 469 polaris tests still pass; new vendored `harness-skills` package has identical behaviour to the in-place auto-creator.

**Effort.** ~3 days (refactor + re-test).

### 3.2 lyra (skills exist, evolution missing)

**Current state.** `lyra-skills/` has extractor, curator, loader, activation, installer, router, review, packs. Static library; no evolution loop.

**Target.** Wire EvoSkill-style failure-driven evolution + CoEvoSkills-style multi-file packages. Lyra's coding-harness shape and existing TDD plugin make this the strongest fit in the corpus.

**Steps.**
1. Vendor `harness-skills` into `projects/lyra/packages/lyra-harness-skills/`.
2. Bind `extract.failure` to lyra's verifier output (Phase 1 / Phase 2 cross-channel rejection from [block 11](projects/lyra/docs/blocks/11-verifier-cross-channel.md)).
3. Add `promote.pareto_frontier` (k=3) over lyra's session branches — already has git-worktree-per-subagent ([block 10](projects/lyra/docs/blocks/10-subagent-worktree.md)), so EvoSkill's git-backed versioning maps directly.
4. Add `promote.coevolutionary` for the harder skill packages (e.g. domain-specific refactor patterns).
5. New CLI: `lyra skills evolve --benchmark <bench> --max-generations 10`.

**Bright-lines added.**
- `BL-LYRA-SKILL-PROMOTE` — promotion to the shared library requires HITL.
- `BL-LYRA-SKILL-COST` — evolution loop hits a per-program budget envelope.

**Success criteria.**
- One reference benchmark (e.g. SWE-bench Lite subset) shows ≥ 5pp gain from evolved skills.
- Cross-family executor + reviewer invariant preserved (ADR-002 lifted from polaris).
- 100% of evolved skills carry trust tier and provenance.

**Effort.** ~3 weeks (vendoring + benchmark wiring + 2-week evaluation).

### 3.3 orion-code (coding agent, no skills yet)

**Current state.** AgentLoop, typed sandboxed tools, Plan Mode, permission policy, hooks. No skill library at all.

**Target.** Direct EvoSkill instantiation. Smallest delta to demonstrate — orion-code is the sibling to lyra but simpler.

**Steps.**
1. Vendor `harness-skills.extract.failure` + `harness-skills.promote.pareto_frontier`.
2. Define an Orion-Code-specific skill schema (`orion-code/skills/<slug>/SKILL.md` + scripts).
3. Hook into the existing PostToolUse → if test fails, log the failure with context for `extract.failure` to consume.
4. Run EvoSkill loop offline (between sessions, not inline) on accumulated failures.
5. Activate evolved skills via prompt-prefix injection at session start.

**Bright-lines added.**
- `BL-ORION-SKILL-PROMOTE` — same shape as Lyra's.

**Success criteria.**
- Smoke benchmark (~20 tasks) shows ≥ 3pp gain.
- All evolved skills are folder-shaped (SKILL.md + scripts), not just text.

**Effort.** ~2 weeks (Orion is simpler than Lyra).

### 3.4 aegis-ops (SRE runbooks, failure-rich)

**Current state.** Hand-authored runbooks; PolicyEngine; AuditLog; mock target system.

**Target.** EvoSkill-style runbook *evolution* — when an incident is resolved through a sequence of operator interventions, distil into a runbook candidate; promote on second occurrence and HITL approval.

**Steps.**
1. Vendor `harness-skills.extract.failure`.
2. Subscribe to AuditLog events; correlate by incident-id.
3. When an incident closes successfully (e.g. service-recovered = true), run failure-driven extraction over the action sequence to build a candidate runbook.
4. Hold candidate in `aegis-ops/runbooks/candidates/`; surface to operator via `GET /v1/runbooks/candidates`.
5. On HITL approval (signature + scope match), promote to `aegis-ops/runbooks/active/`.

**Bright-lines added.**
- `BL-AEGIS-RUNBOOK-PROMOTE` — irreversible (a wrong runbook can hurt production); HITL with cryptographic signing required.

**Success criteria.**
- 5 incident-derived runbooks promoted across a 30-day pilot.
- Zero promotion bypasses bright-line.
- Audit chain unbroken across all promotion events.

**Effort.** ~2.5 weeks. The bright-line discipline is the dominant cost.

### 3.5 cipher-sec (security; dual mode)

**Current state.** Scope-authorisation, deny engine, sandboxed runner, safety monitor, audit log. Hand-authored techniques.

**Target.** Two skill streams:
- **Dialogue side (AutoSkill).** Analyst preferences (e.g. evidence-format, prose style) — opt-in per analyst, governed by the analyst's own `BL-CIPHER-PREF-EXPORT` (preferences cannot leak across analysts).
- **Technique side (EvoSkill).** CTF-style ground-truth-verifiable techniques — discover patterns from successful exploit chains; promote only under explicit scope.

**Steps.**
1. Vendor `harness-skills.extract.dialogue` + `harness-skills.extract.failure`.
2. Dialogue extraction binds to analyst-session traces; output goes to `cipher-sec/skills/analyst-prefs/<analyst-id>/`.
3. Technique extraction binds to engagement-completion events; trust tier defaults to `T2-PREPRINT`; cannot promote until reviewed by engagement-lead.
4. Both streams pass through the existing scope-authoriser before any cross-engagement use.

**Bright-lines added.**
- `BL-CIPHER-SKILL-CROSS-ENGAGEMENT` — a skill from engagement A cannot be used in engagement B without explicit scope re-authorisation.
- `BL-CIPHER-SKILL-EXFIL` — skills cannot include credentials, scope artefacts, or HMAC keys (regex + entropy gate at extraction time).
- `BL-CIPHER-PREF-EXPORT` — analyst preferences cannot leave the analyst's namespace.

**Success criteria.**
- Zero engagement-cross-contamination on a red-team test.
- 10 analyst-pref skills, 10 technique skills surface in pilot.
- Audit chain unbroken.

**Effort.** ~3.5 weeks (security review is the bulk).

### 3.6 quanta-proof (formal proofs; dual mode)

**Current state.** Lemma index, LATS-lite proof search, verifier-first gate, ribbon extractor, lemma memory.

**Target.** Two skill streams:
- **Document side (Ctx2Skill / AutoSkill4Doc).** Convert textbook / paper proofs into reusable tactic-sequence skills.
- **Failure side (EvoSkill).** When LATS-lite fails on a conjecture, run failure-driven extraction over the search tree to identify missing tactics.

**Steps.**
1. Vendor `harness-skills.extract.document` + `harness-skills.extract.failure`.
2. The Lean/Coq verifier IS the ground-truth oracle for EvoSkill — already in place.
3. Document side reads textbook proofs (Lean `mathlib` modules, paper LaTeX) and emits tactic skills.
4. Failure side reads search-tree dumps and emits "tactic to try when X fails" skills.
5. Lemma index becomes the shared retrieval layer for both streams.

**Bright-lines added.**
- `BL-QUANTA-SKILL-PROOF-IMPORT-CLOSURE` — promoted skills must close their proof-import dependencies (no skill that references private lemmas).

**Success criteria.**
- 20 tactic skills derived from `mathlib` modules.
- 10 failure-derived skills.
- Smoke benchmark (5 toy miniF2F-class) ≥ +1 proved task.

**Effort.** ~3 weeks. Lean/Coq integration is the hard part.

### 3.7 atlas-research (long-horizon research)

**Current state.** ReWOO QueryPlanner, Retriever, SynthesisWriter, CitationVerifier.

**Target.** AutoSkill4Doc instantiation — every absorbed paper becomes a candidate procedural skill. Plus AutoSkill on user reports (e.g. preferred section structure, citation style).

**Steps.**
1. Vendor `harness-skills.extract.document` + `harness-skills.extract.dialogue`.
2. Hook AutoSkill4Doc into the Retriever's fetch pipeline: each fetched + verified paper triggers a candidate-skill extraction.
3. Hook AutoSkill into the report-rendering output: user feedback (revise tone, change format, include/exclude sections) becomes preference skills.
4. P22-P27 federation (already designed for polaris) is shared — Atlas can use the same OpenAlex / S2 backbone.

**Bright-lines added.**
- `BL-ATLAS-SKILL-PROMOTE` — paper-derived skills require human verification of the source paper's tier (peer-reviewed vs. preprint).

**Success criteria.**
- 50 paper-derived skills across pilot reports.
- User-pref skills observed reaching v0.1.5+ on heavy users.

**Effort.** ~2.5 weeks. Atlas's existing CitationVerifier dovetails well with `harness-skills.verify.triangulate`.

### 3.8 helix-bio (biomedical, ontology-bound)

**Current state.** OntologyGrounder, 7 mock tools, KnowledgeRouter, FaithfulnessGate, DualUseSafetyLayer, ResearcherMemory.

**Target.** Ctx2Skill / AutoSkill4Doc for protocols + clinical guidelines; AutoSkill for researcher preferences. *Strict trust-tiering required* — biomedical claims cannot ship from RED-tier evidence.

**Steps.**
1. Vendor `harness-skills.extract.document` + `harness-skills.extract.dialogue` + `harness-skills.store.trust_tier`.
2. Bind trust-tier population to ontology-grounding result: skills derived from PubMed-indexed sources start at `T1-PEER-REVIEWED`; bioRxiv at `T2-PREPRINT`; web sources at `T3-DISCUSSION`.
3. Wire `harness-skills.hooks.trust_gate` *upstream* of the existing FaithfulnessGate — RED-tier rejections happen first.
4. Researcher preferences run AutoSkill-style; consent-gated by ResearcherMemory.

**Bright-lines added.**
- `BL-HELIX-SKILL-PHI` — skills cannot include PHI (regex + entropy at extraction).
- `BL-HELIX-SKILL-CLINICAL-CLAIM` — skills emitting clinical claims require Tier-1 source AND ethics-review tag.
- `BL-HELIX-SKILL-DUAL-USE` — proposed skills run through DualUseSafetyLayer at extraction time, not just at use time.

**Success criteria.**
- Zero RED-tier-only claims survive on a held-out test set.
- 20 protocol-derived skills, 10 preference skills in pilot.
- Dual-use trip rate at extraction matches at-use rate (no shift).

**Effort.** ~4 weeks. The ethics gates dominate.

### 3.9 mentat-learn (personal assistant — direct AutoSkill target)

**Current state.** Architecture explicitly describes Hermes-style closed skill-learning loop. Tests for skills exist; full implementation likely partial.

**Target.** Full AutoSkill instantiation — this project's architecture *is* AutoSkill in spirit. Cross-channel (Slack, WhatsApp, Telegram, email, iMessage, web) skill aggregation per-user.

**Steps.**
1. Vendor full `harness-skills.extract.dialogue` pipeline (P_ext / P_judge / P_merge).
2. Per-channel session adapters route incoming messages into the unified extraction stream.
3. SkillBank shape: `mentat/SkillBank/Users/<user_id>/<skill-slug>/SKILL.md` (matches polaris exactly — opportunity for shared SkillBank format).
4. Cross-channel persona — SOUL.md anchored, skill-set is per-user not per-channel.
5. AutoSkill's OpenAI-compatible proxy is the deployment surface.

**Bright-lines added.**
- `BL-MENTAT-SKILL-CONSENT` — every per-user skill requires opt-in capture; any deletion request purges all derived skills (right-to-erasure).
- `BL-MENTAT-SKILL-CROSS-CHANNEL` — skills extracted from one channel cannot leak metadata about that channel's existence to another channel.
- `BL-MENTAT-SKILL-MULTI-USER-EXPORT` — preventing one user's skills from informing another user's responses without explicit consent.

**Success criteria.**
- WildChat-1M-shape pilot: ≥ 200 conversations → ≥ 30 extracted skills.
- v0.1.5+ skills observed on heavy users.
- Zero cross-user leakage on red-team test.
- Persona-fidelity score ≥ 0.9 across ≥ 5 sessions per user.

**Effort.** ~4 weeks. This is the largest single integration because mentat-learn is closest to AutoSkill's core target.

### 3.10 harmony-voice (voice agent)

**Current state.** SemanticCache, SlowThinker, FastTalker. Text mode MVP.

**Target.** AutoSkill on dialogue with voice-specific fields — verbosity preference (short / standard / detailed), formality, language fallback policy, interrupt handling.

**Steps.**
1. Vendor `harness-skills.extract.dialogue` with a voice-shaped `P_ext` template (extract *delivery* preferences, not just content preferences).
2. Skills cache compatibly with the SemanticCache — at session-start, retrieved skills warm the cache via SlowThinker.
3. Fast-path (FastTalker) reads the cached skill prelude inline.

**Bright-lines added.**
- `BL-HARMONY-SKILL-LATENCY` — skill injection cannot push critical-path latency past p95 < 1.2s (the existing target).
- `BL-HARMONY-SKILL-VOICE-CONSENT` — voice-derived skills (tone, pace) require opt-in; default off.

**Success criteria.**
- Latency-budget compliant on a 1000-turn pilot.
- 10 voice-pref skills surface; 5 reach v0.1.3+.

**Effort.** ~2 weeks.

### 3.11 open-fang (research agent, has skills + DAG teams)

**Current state.** 5 hand-authored skills (citation-extraction, claim-localization, counter-example-generation, peer-review, reproduction-script). Atropos-compatible trajectory export. 9-specialist research cohort. 612 tests.

**Target.** EvoSkill-style failure evolution over the 5 existing skills + new-skill discovery. Optional: SkillRL if open-fang adopts SFT (Atropos-compatible export suggests this is on the roadmap).

**Steps.**
1. Vendor `harness-skills.extract.failure` + `harness-skills.promote.pareto_frontier`.
2. Bind to the 5-tier verifier's rejection events; failure-driven proposals against the existing skill set.
3. Held-out eval gate uses the existing `cross-channel verification` discipline.
4. *Optional v2:* Vendor `harness-skills`-with-SkillRL when SFT/RL pipeline lands.

**Bright-lines added.**
- `BL-OPENFANG-SKILL-LITERATURE-DRIFT` — skills derived from the weekly-feed cron tagged with the feed-week timestamp; auto-decay after N weeks unless renewed.

**Success criteria.**
- Existing 5 skills evolved (versioned); ≥ 2 new skills discovered.
- 612 tests still pass.

**Effort.** ~2 weeks.

### 3.12 vertex-eval (evaluator, not consumer)

**Current state.** Trace ingestion, rubric registry, cross-channel evidence, judge pool, Pass@k / Pass^k, HORIZON attribution, LaStraj federation.

**Target.** *Host* a SkillsBench-style benchmark for skill-evolution evaluation. Add rubric primitives that score skills, not just trajectories.

**Steps.**
1. New rubric category: `skill_quality` with checks: `trust_tier_correct`, `provenance_present`, `cross_model_audited`, `eval_gate_passed`.
2. Pass@k metric extension: per-skill Pass@k (does this skill help across k attempts?).
3. LaStraj-style federation already exists — add skill-trace-share for cross-tenant skill benchmarking (opt-in).

**Bright-lines added.**
- `BL-VERTEX-SKILL-LEAK` — skill content not included in shared traces unless tenant explicitly opts in.

**Success criteria.**
- Skill-quality rubric runs against polaris's auto-creator output.
- Skill-Pass@k computed for the lyra and orion-code pilots.

**Effort.** ~2 weeks.

### 3.13 gnomon (substrate, produces skills already)

**Current state.** HIR Harness IR, primitive-level attribution, chaos injection, evolution loop with Pass^k + mesa-guard, Claude-Code-SKILL.md bundling.

**Target.** Gnomon already produces skills as patches. The integration is *standardising the format* so other projects' `extract.trace` adapters can read gnomon's HIR traces directly.

**Steps.**
1. Add `harness-skills.extract.trace` adapter that consumes gnomon's HIR JSONL.
2. Standardise gnomon's SKILL.md output schema to match `harness-skills.store.skill_bank` layout.
3. Bidirectional: skills evolved by other projects can be tested via gnomon's chaos-injection + replay.

**Bright-lines added.** None new (gnomon's existing replay + Pass^k + mesa-guard are the discipline).

**Success criteria.**
- gnomon-emitted skills consumable by lyra and orion-code without translation.
- Cross-project skill replay succeeds on at least one test pair.

**Effort.** ~1.5 weeks.

### 3.14 syndicate (skill-exchange transport)

**Current state.** AgentRegistry, PermissionPlane, HandoffProtocol, Orchestrator, Workflow / Step, FastAPI.

**Target.** Define a *skill handoff* protocol — a typed handoff class for "agent A is sharing skill X with agent B". HMAC-signed, scope-validated, audit-logged.

**Steps.**
1. New handoff class: `SkillTransfer` with required keys `(skill_id, source_agent, target_agent, trust_tier, source_provenance, content_hash)`.
2. PermissionPlane rule template: `operation=skill_transfer × target=<agent> × condition=<scope_check>`.
3. Visibility events: `skill.transfer.proposed`, `skill.transfer.allowed`, `skill.transfer.denied`, `skill.transfer.applied`.
4. Per-tenant skill-marketplace optional add-on (out of scope for v1).

**Bright-lines added.**
- `BL-SYNDICATE-SKILL-CROSS-TENANT` — skill transfer across tenants requires both tenants' explicit opt-in.

**Success criteria.**
- Round-trip: lyra-evolved skill → syndicate transfer → orion-code consumes. Audit chain unbroken.

**Effort.** ~2 weeks.

---

## §4 — Cross-cutting invariants

These apply across every project, lifted into the shared `harness-skills` package:

### 4.1 Cross-model adversarial pair

Every `extract.*` and `verify.*` pair must use different model families. If the project does not currently enforce this, vending `harness-skills` adds the constraint at the type level — `Extractor` and `Verifier` have a `family` field; constructor refuses if equal. (Polaris ADR-002 lifted into the shared lib.)

### 4.2 Trust tier on every skill

Every skill produced by `harness-skills` carries:
- `source_kind` (closed taxonomy: dialogue, document, failure-trace, hir-trace).
- `source_id` (DOI / session-id / trace-id).
- `source_version`.
- `trust_tier` (T1 / T2 / T3 / RED / legacy).
- `content_sha256` of the skill body.
- `extraction_session_id`.
- `reviewer_verdict` (if cross-model review ran).

This is the [Polaris P21 Evidence schema](projects/polaris/POLARIS_V2_1_SOURCE_FEDERATION_PLAN.md#3-architecture-how-this-slots-in) lifted to skills.

### 4.3 No silent promotion

A skill enters the project's *active* library only via:
1. `harness-skills.promote.eval_gate` (held-out evaluation passes).
2. Project-specific bright-line (`BL-<PROJECT>-SKILL-PROMOTE`) HITL approval.

A skill produced but not promoted lives in `<project>/skills/candidates/` indefinitely (or until decay).

### 4.4 Provenance travels

Every active skill links back to its provenance ledger row. Every claim *grounded in* a skill links forward to the skill in its own evidence. Bidirectional references are file-on-disk JSONL (no DB required).

### 4.5 Per-user / per-program isolation

Skills are scoped by default. Cross-project skill transfer requires syndicate's `SkillTransfer` handoff. Cross-tenant transfer requires explicit opt-in (BL-VERTEX-SKILL-LEAK / BL-SYNDICATE-SKILL-CROSS-TENANT).

### 4.6 Filesystem-first

`SKILL.md` + sidecar `versions.jsonl` (merge ledger) + sidecar `provenance.jsonl`. Optional vector cache. Storage layout matches polaris's `SkillBank/` format so cross-project consumption is trivial.

---

## §5 — Sequencing and waves

The 14 projects + shared infra split into four waves. Each wave reuses the previous wave's output.

### Wave 0 — Shared infra (3 weeks)

**Goal.** Build `harness-skills` reference module, lift polaris's auto-creator into it, ship to a pilot.

**Deliverables.**
- `research/harness-engineering/packages/harness-skills/` — full package per §2.
- `polaris-skills` updated to import from `harness-skills` with a deprecation shim.
- All 469 existing polaris tests still green.

**Critical path.** Wave 0 ships before any other project starts. Subsequent waves vendor `harness-skills` as code-copy; bug fixes propagate through PRs.

### Wave 1 — Easiest wins (5 weeks, parallelisable)

**Goal.** The three projects with the smallest delta to working implementation.

**In parallel:**
- **orion-code** (~2 weeks) — clean coding-agent EvoSkill instantiation.
- **harmony-voice** (~2 weeks) — AutoSkill with latency budget.
- **gnomon** (~1.5 weeks) — standardise existing skill output to `harness-skills` schema.

**Critical path.** Each is independent; wave completes when last finishes. Wave 1 produces the *first cross-project skill exchange* — gnomon's HIR-derived skills consumable by orion-code.

### Wave 2 — Substantial integrations (8 weeks, parallelisable)

**Goal.** Five projects with non-trivial existing infrastructure.

**In parallel:**
- **lyra** (~3 weeks) — EvoSkill + CoEvoSkills over existing `lyra-skills`.
- **atlas-research** (~2.5 weeks) — AutoSkill4Doc + user-pref AutoSkill.
- **aegis-ops** (~2.5 weeks) — runbook evolution.
- **open-fang** (~2 weeks) — failure-driven evolution over 5 existing skills.
- **vertex-eval** (~2 weeks) — skill-quality rubrics + Pass@k for skills.

**Critical path.** Vertex-eval should land before the others' pilots so they have a benchmark to score against. Otherwise parallel.

### Wave 3 — Safety-critical / largest deltas (10 weeks, partial parallel)

**Goal.** Four projects with heavy ethics / security / dual-stream complexity.

**In sequence (security review needs serial review):**
- **mentat-learn** (~4 weeks) — full AutoSkill, multi-channel, consent gates.
- **cipher-sec** (~3.5 weeks) — dual-mode (dialogue + technique), strict scope.
- **helix-bio** (~4 weeks) — ethics gates dominate, dual-use perimeter at extraction time.

**In parallel after wave 2:**
- **quanta-proof** (~3 weeks) — Lean/Coq integration; not safety-critical but technically heavy.
- **syndicate** (~2 weeks) — skill-handoff protocol; can land any time after Wave 1.

**Critical path.** Mentat-Learn → Cipher-Sec → Helix-Bio in series for security review continuity. Quanta-proof + syndicate parallel.

### Total wall-clock

| Wave | Wall-clock | Effort | Notes |
|---|---|---|---|
| 0 — Shared infra | 3 weeks | 3 weeks | Single engineer |
| 1 — Easy wins | 5 weeks | 5.5 person-weeks | Parallel |
| 2 — Substantial | 8 weeks | 12 person-weeks | Parallel |
| 3 — Safety-critical | 10 weeks | 16.5 person-weeks | Partial parallel |
| **Total** | **~26 weeks** (~6 months) | **~37 person-weeks** | 1.5 engineers steady |

---

## §6 — Effort summary table

| # | Project | Corner | Effort | Wave |
|---|---|---|---|---|
| 0 | Shared `harness-skills` | substrate | 3 weeks | 0 |
| 1 | polaris | already done — touch-up | 3 days | 0 |
| 2 | lyra | EvoSkill + CoEvoSkills | 3 weeks | 2 |
| 3 | orion-code | EvoSkill | 2 weeks | 1 |
| 4 | aegis-ops | EvoSkill (runbooks) | 2.5 weeks | 2 |
| 5 | cipher-sec | AutoSkill + EvoSkill (dual) | 3.5 weeks | 3 |
| 6 | quanta-proof | Ctx2Skill + EvoSkill (dual) | 3 weeks | 3 |
| 7 | atlas-research | AutoSkill4Doc + AutoSkill | 2.5 weeks | 2 |
| 8 | helix-bio | Ctx2Skill + AutoSkill (ethics-heavy) | 4 weeks | 3 |
| 9 | mentat-learn | Full AutoSkill | 4 weeks | 3 |
| 10 | harmony-voice | AutoSkill (latency-bound) | 2 weeks | 1 |
| 11 | open-fang | EvoSkill | 2 weeks | 2 |
| 12 | vertex-eval | benchmark host (rubrics) | 2 weeks | 2 |
| 13 | gnomon | substrate (trace adapter) | 1.5 weeks | 1 |
| 14 | syndicate | skill-handoff transport | 2 weeks | 3 |

---

## §7 — Risk register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Vendoring drift — `harness-skills` copies diverge across projects | High | Medium | A single canonical reference at `research/harness-engineering/packages/harness-skills/`; per-project copies tracked in CI via `make harness-skills-sync`; PRs to canonical version flagged for downstream sync. |
| Bright-line bypass via shared infra | Medium | **High** | Each project's bright-line registry stays in-project; `harness-skills` is *passive* — it produces candidates, projects' own gates control promotion. No path from `harness-skills` to active use that bypasses the project gate. |
| Cross-model invariant violated under cost pressure | Medium | High | Type-level enforcement: `Extractor.family != Verifier.family` checked at session boot, refuses to start otherwise. Lifted directly from polaris ADR-002. |
| Skill explosion — too many skills, retrieval breaks | High | Medium | Held-out eval gate as the floor; per-project `max_active_skills` cap; periodic decay (skills not retrieved in N sessions go to candidates). |
| Cipher-sec skill leakage across engagements | Low | **High** | Engagement-id namespace + cryptographic scope check at retrieval; red-team test before pilot. |
| Helix-bio dual-use bypass via skill content | Low | **High** | DualUseSafetyLayer runs at *extraction* time, not just at use time. Same regex + heuristic perimeter applied to skill text. |
| Mentat-learn cross-user contamination | Low | **High** | Per-user namespace + cryptographic key scoping; right-to-erasure deletes derived skills. Same isolation as AutoSkill's `SkillBank/Users/<user_id>/`. |
| Aegis-ops bad runbook promotes via auto-merge | Low | **High** | Two-signature HITL on `BL-AEGIS-RUNBOOK-PROMOTE`; no automated path to active runbook. |
| Quanta-proof skills reference private lemmas | Medium | Medium | `BL-QUANTA-SKILL-PROOF-IMPORT-CLOSURE` — skills must close their imports; fail-fast at promotion. |
| Open-fang weekly-feed skill drift | Medium | Medium | Skills tagged with feed-week; auto-decay after N weeks unless renewed. |
| Vertex-eval skill content leaks via federation | Low | High | Skill content excluded from federated traces by default; opt-in tenant flag; PII-scrubber extended to skill bodies. |
| Syndicate cross-tenant skill transfer abuse | Low | High | `BL-SYNDICATE-SKILL-CROSS-TENANT` requires both tenants' opt-in; HMAC-signed handoffs; full audit. |
| Wave 0 takes longer than 3 weeks (delays everything) | Medium | High | Aggressive scope discipline on `harness-skills` — the reference module must be small enough to land in 3 weeks; reduce to extract+store+verify+promote if needed; defer adapters to project-side. |
| Wave 3 ethics review delays mentat/cipher/helix | Medium | High | Schedule security/ethics review reviewers in parallel with implementation; do not let waiting-for-review block merge. |

---

## §8 — Migration approach

### 8.1 Backwards compatibility

Each project's existing skill infrastructure (where present) keeps working:
- **polaris** — `polaris-skills.auto_creator` continues to work; new code lives in `harness-skills`; deprecation shim aliases the old paths for one minor version.
- **lyra** — `lyra-skills` keeps its existing API; new evolution loops are additive; old skills auto-tagged `trust_tier="legacy"` until reviewed.
- **open-fang** — 5 existing skills auto-tagged `trust_tier="legacy"`; first pilot run promotes to T1/T2 based on evidence quality.

### 8.2 Test hygiene

- Wave 0 must show all existing polaris tests (469 today) still passing.
- Each subsequent wave: project-local test count must not regress.
- New tests for evolution loops are hermetic — no live model calls in CI; injectable `Extractor` / `Verifier` follow the polaris-mcp `HttpTransport` pattern.

### 8.3 Documentation discipline

Every project gets a `docs/skills.md` written before code lands, in the existing `docs/architecture.md` style. Sections:
- Which corner of the design space.
- What skills exist today (legacy).
- What evolution loop runs (when triggered, what extracts).
- What bright-lines protect the loop.
- How to use the resulting skills.

### 8.4 Versioning

`harness-skills` follows SemVer-ish:
- `0.1.x` — internal use, polaris + 1 pilot project.
- `0.2.x` — Wave 1 complete.
- `0.3.x` — Wave 2 complete.
- `1.0.0` — Wave 3 complete and stable.

---

## §9 — Success criteria for "skill self-evolution applied to all projects"

Final acceptance is satisfied when **all** of the following hold:

### Coverage

- ✅ All 14 projects have either *implemented* a skill-evolution loop OR explicitly *opted out* with documented reasoning.
- ✅ Each project has a `docs/skills.md` describing its corner and integration.
- ✅ At least 1 cross-project skill transfer round-trip demonstrated via syndicate.

### Discipline

- ✅ Cross-model adversarial-pair invariant preserved across all integrations (no extractor-verifier pair shares a model family).
- ✅ Every active skill has `trust_tier`, `source_kind`, `source_id`, `content_sha256`, `extraction_session_id`.
- ✅ No skill enters active use without project-specific bright-line HITL approval.
- ✅ Provenance bidirectional — every skill links to its source artefact; every claim grounded in a skill links forward.

### Ethics / safety

- ✅ Mentat-Learn passes a cross-user red-team test (zero leakage).
- ✅ Cipher-Sec passes a cross-engagement red-team test (zero leakage).
- ✅ Helix-Bio passes a dual-use red-team test (extraction-time gate matches use-time gate).
- ✅ Aegis-Ops audit chain unbroken across all promotion events.

### Performance

- ✅ Harmony-Voice latency budget compliant (p95 < 1.2s) under skill injection.
- ✅ Vertex-Eval skill-quality rubrics run on outputs from at least 5 projects.
- ✅ Lyra benchmark (SWE-bench Lite subset) shows ≥ 5pp gain.
- ✅ Orion-Code smoke benchmark shows ≥ 3pp gain.

### Sustainability

- ✅ `harness-skills` reference module versioned 1.0.0 and stable.
- ✅ Per-project copies sync via `make harness-skills-sync`; CI flags drift.
- ✅ Each project's new bright-line registry codified and tested.

---

## §10 — Reading list (for engineers picking up this plan)

In order:

1. [docs/171-skill-self-evolution-2026-synthesis.md](docs/171-skill-self-evolution-2026-synthesis.md) — the design-space map.
2. [docs/167-autoskill-experience-driven-lifelong-learning.md](docs/167-autoskill-experience-driven-lifelong-learning.md) — the dialogue corner.
3. [docs/168-evoskill-coding-agent-skill-discovery.md](docs/168-evoskill-coding-agent-skill-discovery.md) — the failure-driven coding corner.
4. [docs/169-coevoskills-co-evolutionary-verification.md](docs/169-coevoskills-co-evolutionary-verification.md) — the surrogate-verifier corner.
5. [docs/170-skillrl-recursive-skill-augmented-rl.md](docs/170-skillrl-recursive-skill-augmented-rl.md) — the RL-coupled corner.
6. [docs/154-ctx2skill-self-evolving-context-skills.md](docs/154-ctx2skill-self-evolving-context-skills.md) — the document corner.
7. [projects/polaris/POLARIS_V2_1_SOURCE_FEDERATION_PLAN.md](projects/polaris/POLARIS_V2_1_SOURCE_FEDERATION_PLAN.md) — the closest precedent.
8. [projects/polaris/docs/architecture.md](projects/polaris/docs/architecture.md) — the cross-model + bright-line + provenance discipline being lifted.
9. The relevant project's own `docs/architecture.md` and `docs/architecture-tradeoff.md`.

---

## §11 — One-paragraph summary

This plan integrates skill self-evolution into all 14 projects in `projects/` by (1) building a small reference module `harness-skills` that distils the four 2026 papers into pluggable extractors / verifiers / promoters / hooks; (2) vendoring it into each project as code-copy (matching the existing `harness_core/` convention); (3) wiring each project to *the right corner of the design space* — AutoSkill for dialogue projects, EvoSkill / CoEvoSkills for coding / operational projects, Ctx2Skill / AutoSkill4Doc for document-driven projects, SkillRL for any open-weights training-loop projects, with gnomon as substrate, syndicate as transport, vertex-eval as benchmark; (4) preserving each project's existing discipline verbatim — bright-lines, cross-model pairs, scope authorisation, dual-use perimeters, audit chains; (5) shipping in four waves over ~26 weeks (~6 months) of ~37 person-weeks of effort. Wave 0 ships shared infra; Waves 1–3 ship parallel project integrations of increasing complexity. The plan does not duplicate existing skill infrastructure — `polaris-skills`, `lyra-skills`, `open-fang/skills/` keep their homes; we add evolution loops, not a refactor. The acceptance criterion is a cross-project skill transfer round-trip, demonstrated by Wave 1's gnomon → orion-code path and Wave 3's syndicate-mediated cross-tenant exchange.

---

## §12 — What "proceed later" means

When you say "go", we'll:
1. **Open a tracking task per project** (14 tasks + 1 Wave-0 task = 15 tasks).
2. **Start Wave 0** — build `research/harness-engineering/packages/harness-skills/`. ~3 weeks.
3. **Schedule Waves 1–3** as parallel and serial slices per §5.
4. **Open per-project pilot evaluations** before Wave 3 starts (Vertex-Eval rubrics from Wave 2).

Until then, this plan is the contract. Specific changes to scope, sequencing, or success criteria should land here as edits before code moves.
