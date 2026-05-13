# 171 — Skill Self-Evolution in 2026: A Synthesis

**Scope.** A landscape view of the 2026 wave of skill-self-evolution work, comparing **AutoSkill** ([167](167-autoskill-experience-driven-lifelong-learning.md)), **EvoSkill** (Alzubi et al., [168](168-evoskill-coding-agent-skill-discovery.md)), **CoEvoSkills** (Zhang et al., [169](169-coevoskills-co-evolutionary-verification.md)), and **SkillRL** ([170](170-skillrl-recursive-skill-augmented-rl.md)) — all submitted to arXiv between February and April 2026 — alongside the existing harness-engineering corpus (Voyager, Ctx2Skill, HeavySkill, ReasoningBank, Karpathy Skills, Atomic Skills, Skill-RAG, Hermes, Voyager-deep), applied examples (the AutoSkill_Claude plugin and [Self-Improving Agent Skills](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/awesome_agent_skills/self-improving-agent-skills)), and the broader self-modifying-agent frontier ([Darwin Gödel Machine](https://arxiv.org/abs/2505.22954)).

**One-paragraph thesis.** *Skills* — externalised, structured, file-on-disk capability artifacts — are emerging as the *trainable, transferable, auditable* unit of agent-system improvement. Four 2026 papers stake out the four corners of the design space: AutoSkill (frozen-model, dialogue-driven, single-file), EvoSkill (frozen-model, coding-agent, ground-truth-grounded), CoEvoSkills (frozen-model, multi-file, surrogate-verifier-grounded), SkillRL (RL-trained, three-tier hierarchy, recursive co-evolution). Together they answer four very different questions but share three structural commitments: skills are files, evolution beats one-shot generation, and skills are more portable than weights. The synthesis is in the table; the design rules follow.

## The four corners — at a glance

| | **AutoSkill** ([167](167-autoskill-experience-driven-lifelong-learning.md)) | **EvoSkill** ([168](168-evoskill-coding-agent-skill-discovery.md)) | **CoEvoSkills** ([169](169-coevoskills-co-evolutionary-verification.md)) | **SkillRL** ([170](170-skillrl-recursive-skill-augmented-rl.md)) |
|---|---|---|---|---|
| arXiv | 2603.01145 | 2603.02766 | 2604.01687 | 2602.08234 |
| Affiliation | Shanghai AI Lab + ECNU | Alzubi/Provenzano/Bingham/Chen/Vu | Zhang et al. (UIC + UBC + others) | aiming-lab |
| Input modality | Dialogue + docs + traces | Coding-agent traces | Coding-agent traces | RL trajectories |
| Skill artifact | Single SKILL.md + assets | SKILL.md folder + scripts | Multi-file package | JSON 3-tier SkillBank |
| Feedback signal | LLM judge (no ground truth) | Ground-truth task scores | **Surrogate verifier** (no ground truth) | RL reward + validation-failure rate |
| Parameter access | None (frozen) | None (frozen) | None (frozen) | **Required** |
| Co-evolution | None (one-way) | Pareto-frontier search | Generator ↔ Verifier | **Policy ↔ SkillBank** |
| Headline result | WildChat-1M scale demo | OfficeQA +7.3, SealQA +12.1, BrowseComp +5.3 | SkillsBench 71.1% (vs. 53.5% human) | ALFWorld + WebShop avg +15.3% |
| Cross-model transfer | Untested | +5.3 on BrowseComp | **+35–44pp on 6 LLMs** | Same-family only |
| Code public | Yes (MIT, ECNU-ICALK) | Pending | Pending | Yes (MIT, aiming-lab) |

## Three design axes (with all systems plotted)

### Axis 1 — Feedback signal source

```
ground-truth ←——→ surrogate ←——→ adversarial ←——→ no signal
oracle           verifier        self-play        (judgement only)

EvoSkill, SkillRL    CoEvoSkills      Ctx2Skill           AutoSkill
```

- **Ground-truth oracle.** Pass/fail per task. EvoSkill consumes it directly; SkillRL absorbs it through RL reward.
- **Surrogate verifier.** A separate LLM session, information-isolated from the ground-truth, that emits structured diagnostics. CoEvoSkills' contribution.
- **Adversarial self-play.** A Challenger and a Reasoner with co-evolving rubrics; the Judge is itself an LLM. Ctx2Skill's mechanism ([154](154-ctx2skill-self-evolving-context-skills.md)).
- **No signal.** Only LLM-as-judge of textual coherence (P_judge in AutoSkill).

The deeper-left a system sits, the more brittle to gaming and the better at transfer. The deeper-right a system sits, the more applicable to settings without ground truth.

### Axis 2 — Skill artifact form

```
single SKILL.md ←——→ folder + scripts ←——→ multi-file bundle ←——→ JSON tiered

AutoSkill / Ctx2Skill   EvoSkill              CoEvoSkills          SkillRL
```

The artifact form matters because it determines what the runtime can *do* with the skill:

- **Single SKILL.md** — readable, hand-editable, easy to audit. Best for human-edited preferences.
- **Folder + scripts** — agent can call helper code. Useful when guidance alone is insufficient.
- **Multi-file bundle** — full executable package: tested utilities, multiple entry points. CoEvoSkills' format.
- **JSON tiered** — programmatic retrieval over a structured schema. SkillRL's choice for RL-loop-friendliness.

### Axis 3 — Parameter access

```
frozen weights ←——————————————→ trained weights

AutoSkill, EvoSkill, CoEvoSkills,    SkillRL
Ctx2Skill, Voyager
```

This is binary and decisive. Most of the 2026 wave is *frozen*: they assume closed-source frontier models. SkillRL is the dissenting voice — it argues that if you have weight access, you should *train* the policy alongside the skills. The performance numbers (+15.3% over baselines) suggest the dissent has merit when applicable.

## The "feedback signal × artifact form" matrix

|  | **No signal / LLM judge** | **Surrogate verifier** | **Ground-truth oracle** |
|---|---|---|---|
| **Single SKILL.md** | AutoSkill ([167](167-autoskill-experience-driven-lifelong-learning.md)), Ctx2Skill ([154](154-ctx2skill-self-evolving-context-skills.md)) | — | — |
| **Folder + scripts** | — | Self-Improving Agent Skills uses Analyst/Mutator diagnostics plus Executor scoring before keeping changes | EvoSkill ([168](168-evoskill-coding-agent-skill-discovery.md)) |
| **Multi-file package** | — | CoEvoSkills ([169](169-coevoskills-co-evolutionary-verification.md)) | — |
| **JSON tiered (RL)** | — | — | SkillRL ([170](170-skillrl-recursive-skill-augmented-rl.md)) (via RL reward) |

## Applied additions: self-improving skill apps and self-modifying agents

| System | Artifact improved | Loop | What it adds to this synthesis | Main caveat |
|---|---|---|---|---|
| [Self-Improving Agent Skills](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/awesome_agent_skills/self-improving-agent-skills) | `SKILL.md` folder following agentskills.io | Executor generates/scored scenarios, Analyst diagnoses failures, Mutator applies one surgical prompt edit, keep if score improves | A concrete app-level implementation of skill evolution with Google ADK, Gemini, FastAPI, Next.js, structured Pydantic outputs, SSE progress, and downloadable improved skills | Eval quality is user/app-defined; prompt overfitting remains possible |
| [Darwin Gödel Machine](https://arxiv.org/abs/2505.22954) | entire coding-agent codebase | select parent agent from archive, self-modify code, evaluate on SWE-bench/Polyglot, add viable variants to archive | Extends "skills evolve" into "the agent runtime evolves"; reports SWE-bench 20.0%→50.0% and Polyglot 14.2%→30.7% | Requires sandboxing, traceability, and strong evals; recursive self-modification can optimize the wrong objective |

Empty cells are *open opportunities*. A surrogate-verifier-grounded single-SKILL.md system would land in the upper-middle and target the dialogue / context-learning regime that AutoSkill and Ctx2Skill currently dominate but with a stronger feedback signal. A surrogate-verifier-grounded JSON-tiered RL system would be the natural extension of SkillRL into the closed-source regime.

## Three structural commitments shared across all four systems

Despite the differences, every paper in this wave commits to:

### 1. **Skills are files, not state.**

Every system materialises skills as on-disk artifacts: SKILL.md files (AutoSkill, EvoSkill, CoEvoSkills), JSON SkillBank (SkillRL), Markdown system-prompt prefix (Ctx2Skill). None of them store skills in model weights or in opaque vector indexes. *Auditability is non-negotiable.* The reason to externalise skills as files is the same reason filesystem-first persistence is the right default for memory ([Polaris architecture trade-off 6](../projects/polaris/docs/architecture-tradeoff.md#trade-off-6-filesystem-first-jsonlmd-vs-database)) — `cat`, `grep`, `git` are the tools that make a skill library trustworthy.

### 2. **Evolution beats one-shot generation.**

Across every paper, single-pass skill generation barely beats the no-skill baseline:

- **CoEvoSkills ablation** ([169](169-coevoskills-co-evolutionary-verification.md)): "Without evolution" = 48.6% vs. 71.1% full = −22.5pp.
- **EvoSkill** ([168](168-evoskill-coding-agent-skill-discovery.md)) explicitly compares against single-pass `Skill-Creator`, which lands at the no-skill floor.
- **AutoSkill** ([167](167-autoskill-experience-driven-lifelong-learning.md)) shows 34 merge events on `professional_text_rewrite` — clear evidence that iterative refinement is doing real work.
- **SkillRL** ([170](170-skillrl-recursive-skill-augmented-rl.md)) couples evolution directly to the RL loop.

The implication for practitioners: any skill library with no evolution mechanism is *strictly weaker* than the equivalent no-skill baseline plus a few minutes of human authoring.

### 3. **Skills are more portable than weights.**

Cross-model transfer is the strongest empirical claim in the wave:

- **CoEvoSkills**: skills evolved on Claude Opus 4.6 transfer to GPT-5.2, Sonnet 4.5, Haiku 4.5, Qwen3-Coder, DeepSeek V3, Mistral Large 3 — *every* model gains +35 to +44 absolute points.
- **EvoSkill**: skills evolved on SealQA transfer to BrowseComp without modification — +5.3 points.
- **AutoSkill** and **SkillRL** do not test cross-model transfer in the headlines but the artifact-format makes it possible.

The deeper claim is that **skills encode reusable task-structure rather than model-specific artifacts**. Where a fine-tuned weight matrix is by definition coupled to one model, a SKILL.md captures the procedural knowledge in *a form the next model can read*. This is the main reason the 2026 wave focuses on frozen-model skills: weights belong to the foundation-model labs; skills belong to the application layer.

## A taxonomy of evolution loops

Reading across the four papers, evolution loops fall into four shapes:

### Shape A — One-way maintenance loop (AutoSkill)

```
session ─┬─→ extract candidates (P_ext)
         │     ↓
         │   judge vs. neighbours (P_judge)
         │     ↓
         └─→ add | merge | discard
```

A small set of LLM calls per session-end. No fitness signal beyond textual coherence. Cheap, scalable, but cannot *test* whether merged skills actually work better.

### Shape B — Pareto-frontier search (EvoSkill)

```
frontier G = { p₀, p₁, …, p_k }
each iter: parent = round_robin(G)
           failures F = score(parent) < τ
           proposal π = Proposer(F)
           candidate p̃ = SkillBuilder(parent, π)
           if score(p̃, V) > min(G): admit
```

Evolutionary search at the *agent-program* level. Frontier prevents single-point failure. Ground-truth fitness signal — strong but tied to having labelled tasks.

### Shape C — Co-evolutionary loop (CoEvoSkills + Ctx2Skill)

```
generator ← context ← diagnostics ← verifier
verifier ← strengthen-tests ← oracle (occasional)
```

Two LLMs locked in a feedback dance. The verifier evolves alongside the generator; the oracle is consulted sparingly to keep the verifier honest. The most expressive feedback signal — structured textual diagnostics — but the most expensive.

### Shape D — RL-coupled co-evolution (SkillRL)

```
policy ← gradient ← rollouts ← SkillBank
SkillBank ← distill ← validation-failures ← rollouts
```

Two coupled processes during training: policy update and SkillBank update. Triggered on validation-failure rate. Requires weight access; only applies during training.

## When to use which (decision tree)

```
Do you have weight access?
├── Yes → Are you actively running RL?
│         ├── Yes → SkillRL ([170])
│         └── No  → Use frozen-weights options below
└── No  → What is your input?
          ├── Dialogue / preferences        → AutoSkill ([167])
          ├── Single dense document         → Ctx2Skill ([154])
          ├── Coding-agent + ground truth   → EvoSkill ([168])
          ├── Coding-agent + no ground truth → CoEvoSkills ([169])
          └── Just want a skill format spec → Karpathy Skills ([71]) / Anthropic Skills ([04])
```

A second cut, by *what you're optimising for*:

| Goal | Reach for |
|---|---|
| Personalisation across user sessions | AutoSkill ([167](167-autoskill-experience-driven-lifelong-learning.md)) |
| Domain expertise on a coding benchmark | EvoSkill ([168](168-evoskill-coding-agent-skill-discovery.md)) |
| Reusable skill packages portable across models | CoEvoSkills ([169](169-coevoskills-co-evolutionary-verification.md)) |
| Better RL training with reusable patterns | SkillRL ([170](170-skillrl-recursive-skill-augmented-rl.md)) |
| Skill from a single technical document | Ctx2Skill ([154](154-ctx2skill-self-evolving-context-skills.md)) |
| Just a pluggable Claude Code productivity boost | AutoSkill_Claude plugin (see below) |

## The applied example — `AutoSkill_Claude` plugin

[github.com/CatVinci-Studio/AutoSkill_Claude](https://github.com/CatVinci-Studio/AutoSkill_Claude) — MIT — independent of the ECNU codebase but inspired by AutoSkill's philosophy.

A practical Claude Code plugin that registers three hooks:

- `PostToolUse(Skill)` — silently records every skill invocation.
- `SessionEnd` — scans full transcripts, updates the optimisation queue.
- `SessionStart` — checks thresholds, notifies user.

Storage at `~/.local/share/auto-optimize-skills/queue.json`. Two configurable thresholds: `notify_after_skill_uses` (default 5), `notify_after_new_patterns` (default 2). Two slash commands: `/optimize-skill` (refines an existing skill from observed signals — missed triggers, correction patterns, incomplete outputs) and `/new-skill` (creates from current conversation, repeated patterns ≥ 2 occurrences, or user description). All drafts user-approved before write. Original skills backed up.

Known limitation: pattern dedupe uses first 100 chars only — paraphrased workflows missed.

This plugin is the *"can it run in production today?"* answer for the AutoSkill philosophy. It runs today. It uses the existing Claude Code skill format. It auto-optimises. It does not require any of the academic systems above; it is the simplest deployable shape of the same idea.

## Existing harness-engineering corpus connections

The 2026 skill-evolution wave doesn't appear in a vacuum. Connecting back to the corpus:

| Existing doc | Relationship |
|---|---|
| [04-skills](04-skills.md) | The Anthropic SKILL.md format — the lingua franca every system in this synthesis adopts. |
| [71-karpathy-skills-single-file-guardrails](71-karpathy-skills-single-file-guardrails.md) | The case for single-file, header-bounded skill format. AutoSkill, EvoSkill, CoEvoSkills, Ctx2Skill all comply. |
| [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md) | Atomic-skills argues for scale via decomposition; CoEvoSkills' multi-file packages and SkillRL's tiering are atomic-skills-compatible. |
| [79-skill-rag](79-skill-rag.md) | The retrieval side. AutoSkill's hybrid dense+lexical retrieval is the production version. |
| [89-voyager-deep](89-voyager-deep.md) | Voyager's growing skill library is the conceptual ancestor of all four 2026 systems. |
| [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md) | Hermes outcome-fed refinement is a *single-program* analogue of EvoSkill's frontier search. |
| [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md) | The broader arena meta-frame; each of the four 2026 systems is one shape of self-evolving arena. |
| [81-reasoningbank](81-reasoningbank.md) | Failure-distillation as memory; SkillRL's `common_mistakes` tier is the RL-loop integration of this primitive. |
| [85-alphaevolve](85-alphaevolve.md) | Code-level evolutionary search; EvoSkill argues skill-level transfer better. |
| [154-ctx2skill](154-ctx2skill-self-evolving-context-skills.md) | Sibling — the no-feedback dense-document corner of the same design space. |
| [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md) | Argues the harness itself can become a skill — the logical endpoint of skill-as-portable-layer thinking. |
| [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md) | Earlier synthesis post; this doc extends it with the four newly-published systems. |
| [162-paper2agent-reimagining-papers-as-agents](162-paper2agent-reimagining-papers-as-agents.md) | Paper → MCP server pipeline. AutoSkill4Doc is the closest analogue — paper → SKILL.md. |
| [Polaris ADR-002](../projects/polaris/docs/concepts/adrs/adr-002-cross-model-adversarial-pair.md) | The cross-family adversarial-pair invariant. CoEvoSkills' surrogate-verifier discipline is the empirical case for ADR-002. |
| [Polaris P24 trust-tiering](../projects/polaris/docs/research/p24-trust-tiering-retractions.md) | Trust-tier labels for evidence; analogous structure for skills (which skills are battle-tested vs. speculative) is open work. |

## Surveys and broader landscape

Two recent survey efforts cover adjacent territory:

- **A Survey of Self-Evolving Agents** (arXiv:2507.21046, TMLR 2026 — "What, When, How, and Where to Evolve on the Path to Artificial Super Intelligence"). Maps the broader self-evolution literature; skill-evolution is one branch among several (memory, planning, tool-use evolution).
- **A Comprehensive Survey of Self-Evolving AI Agents** (arXiv:2508.07407 — "A New Paradigm Bridging Foundation Models and Lifelong Agentic Systems"). Frames self-evolution as the bridge between static foundation models and adaptive agentic systems.

The curated GitHub list [XMUDeepLIT/Awesome-Self-Evolving-Agents](https://github.com/XMUDeepLIT/Awesome-Self-Evolving-Agents) tracks the rolling literature.

Adjacent applied projects worth flagging:

- **Hermes Agent Self-Evolution** ([github.com/NousResearch/hermes-agent-self-evolution](https://github.com/NousResearch/hermes-agent-self-evolution), ICLR 2026 Oral) — DSPy + GEPA (Genetic-Pareto Prompt Evolution) over Hermes Agent's skills, prompts, and code. Same family as EvoSkill but at the DSPy level.
- **EvoAgentX** ([github.com/EvoAgentX/EvoAgentX](https://github.com/EvoAgentX/EvoAgentX)) — self-evolving ecosystem of agents; different scope but shared philosophy.

## Five open problems

The 2026 wave leaves five problems that no single paper has cracked:

### 1. Surrogate-grounded skill evolution for non-coding domains

CoEvoSkills' surrogate verifier is brilliant but assumes a coding-task-with-deterministic-verifier structure. For dialogue, document, or open-ended generation, the surrogate verifier needs different bandwidth. Adapting it to AutoSkill's regime is the obvious follow-up.

### 2. Cross-organisation skill transfer with provenance

AutoSkill's per-user namespace can move to per-org but skill provenance (where did this skill come from, has it been audited, has it been replicated) is unaddressed. Polaris's [P24 trust-tiering](../projects/polaris/docs/research/p24-trust-tiering-retractions.md) is the structural answer; integrating it with the skill-evolution loop is open.

### 3. Skill composition algebra

Multiple skills loaded simultaneously can conflict. AutoSkill flags this as future work; none of the others address it directly. A formal composition operator (does `SkillA ⊕ SkillB` always succeed? when does it conflict?) is missing.

### 4. Skill-quality decay and pruning

Skills that helped at training-time may hurt at test-time after data drift. Only EvoSkill has eviction (Pareto-frontier capacity); the others append-only. A general decay model is open.

### 5. Verifier hallucination and adversarial robustness

CoEvoSkills' surrogate verifier is the load-bearing component (−30pp without it) — but if the verifier hallucinates systematically, the loop converges on garbage. Hardening the verifier (cross-family adversarial-pair? ground-truth-oracle re-grounding cadence?) is open.

## Bottom line

The 2026 skill-self-evolution wave converges on three structural truths:

1. **Skills are the right unit of capability accumulation** — files, not state; portable, not parameter-bound; auditable, not implicit.
2. **Evolution dominates one-shot generation** — every paper that compares the two finds the gap is large (−22pp in CoEvoSkills' ablation, similar in EvoSkill's `Skill-Creator` baseline).
3. **The four corners of the design space are now mapped** — frozen-vs-trained × ground-truth-vs-surrogate × single-vs-multi-file × dialogue-vs-coding-vs-document — and four canonical systems hold the corners (AutoSkill, EvoSkill, CoEvoSkills, SkillRL plus Ctx2Skill at the document corner).

For practitioners: **pick by axis**. Closed model + dialogue → AutoSkill. Closed model + coding + labels → EvoSkill. Closed model + coding + no labels → CoEvoSkills. Open model + RL → SkillRL. Single document → Ctx2Skill. The papers are not competitors; they are different points on a Pareto frontier of design choices, and the right one is determined by what you can pay (parameter access? ground truth? compute?) and what you have (dialogue stream? code benchmark? technical document? RL pipeline?).

For researchers: the *empty cells* in the matrix are the next moves. Surrogate-verifier dialogue evolution. Cross-org skill transfer with provenance. Skill composition algebra. Verifier-hardening against hallucination. Each is a paper. Each is mid-2026 work.

For the harness-engineering canon: the *cumulative* claim — that skills are the trainable, transferable, auditable unit of agent improvement — is now empirically supported across dialogue, coding, document, and RL settings. The case for skills-as-first-class-artifact is closed; the open work is the *infrastructure* (storage, evolution, retrieval, governance) that makes skill libraries production-grade.

---

**Citations.**

- [167] Yang Y. et al., *AutoSkill*, arXiv:2603.01145v2, 2026. Code: [github.com/ECNU-ICALK/AutoSkill](https://github.com/ECNU-ICALK/AutoSkill).
- [168] Alzubi S., Provenzano N., Bingham J., Chen W., Vu T., *EvoSkill: Automated Skill Discovery for Multi-Agent Systems*, arXiv:2603.02766v1, 2026.
- [169] Zhang H. et al., *EvoSkills: Self-Evolving Agent Skills via Co-Evolutionary Verification*, arXiv:2604.01687v2, 2026.
- [170] Xia P. et al., *SkillRL: Evolving Agents via Recursive Skill-Augmented Reinforcement Learning*, arXiv:2602.08234, 2026. Code: [github.com/aiming-lab/SkillRL](https://github.com/aiming-lab/SkillRL).
- [154] Si S. et al., *Ctx2Skill: From Context to Skills*, arXiv:2604.27660v2, 2026.
- *A Survey of Self-Evolving Agents*, arXiv:2507.21046, TMLR 2026.
- *A Comprehensive Survey of Self-Evolving AI Agents*, arXiv:2508.07407.
- AutoSkill_Claude plugin: [github.com/CatVinci-Studio/AutoSkill_Claude](https://github.com/CatVinci-Studio/AutoSkill_Claude).
- Hermes Agent Self-Evolution: [github.com/NousResearch/hermes-agent-self-evolution](https://github.com/NousResearch/hermes-agent-self-evolution).
- Awesome list: [github.com/XMUDeepLIT/Awesome-Self-Evolving-Agents](https://github.com/XMUDeepLIT/Awesome-Self-Evolving-Agents).
