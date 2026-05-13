# 169 — CoEvoSkills: Self-Evolving Agent Skills via Co-Evolutionary Verification

**Paper.** Hanrong Zhang, Shicheng Fan, Henry Peng Zou, Yankai Chen, Zhenting Wang, Jiayu Zhou, Chengze Li, Wei-Chieh Huang, Yifei Yao, Kening Zheng, Xue Liu, Xiaoxiao Li, Philip S. Yu — *EvoSkills: Self-Evolving Agent Skills via Co-Evolutionary Verification* — arXiv:2604.01687v2 [cs.AI] — submitted 2 Apr 2026 (v1), revised 12 Apr 2026 (v2).

> **Naming note.** Two papers in this corpus carry the name "EvoSkill(s)". To avoid confusion this doc refers to Zhang et al.'s system as **CoEvoSkills** (matching the title's emphasis on co-evolutionary verification) and reserves "EvoSkill" for Alzubi et al. ([168](168-evoskill-coding-agent-skill-discovery.md)).

**One-line definition.** CoEvoSkills is a self-evolving framework where two LLM-backed components — a **Skill Generator** and an **information-isolated Surrogate Verifier** — co-evolve over up to 5 oracle interventions and 15 surrogate retries, producing structured *multi-file* skill packages (SKILL.md + executable Python utilities) that beat human-curated skills by **+17.6 percentage points** on SkillsBench (71.1% vs. 53.5%) and transfer to six additional LLMs without modification.

## Why this paper matters

Three structural moves distinguish CoEvoSkills from prior skill-evolution work.

**Move 1 — Multi-file skill packages, not single-file Markdown.** AutoSkill ([167](167-autoskill-experience-driven-lifelong-learning.md)) produces single SKILL.md files; EvoSkill ([168](168-evoskill-coding-agent-skill-discovery.md)) produces folders with optional helper scripts. CoEvoSkills *requires* multi-file packages: a skill is a SKILL.md + tested Python utility code + supporting assets. The discovered skills (~206 lines: 64 procedural + 142 executable) are bundles, not documents. This matters because a coding agent often needs not just *guidance* but *callable utility code* — and human-authored skills frequently leave the implementation as an exercise.

**Move 2 — Surrogate verification without ground-truth tests.** The dominant evolutionary-skill loop (EvoSkill, AlphaEvolve, GEPA) uses ground-truth task scores as the optimisation signal. CoEvoSkills argues this *teaches the system to overfit the evaluation*. Their answer: keep an LLM-based **surrogate verifier** that observes only the skill output (not ground-truth tests) and emits structured failure diagnostics. The ground-truth oracle returns *only* an opaque pass/fail signal, breaking the over-fitting feedback. The headline ablation: **removing the surrogate verifier drops performance by 30 percentage points** (71.1% → 41.1%). The verifier, not the oracle, is the load-bearing component.

**Move 3 — "Agents create better skills than human-curated ones."** This is one of three explicit takeaways. CoEvoSkills' evolved skills outperform Anthropic-style human-authored skills on SkillsBench by 17.6 absolute points on Claude Opus 4.6. In the *Natural Science* sub-domain, human-curated skills *actively degrade* performance (cognitive misalignment between human writing and agent reasoning patterns). The paper takes seriously the idea that skill-authoring is a job a human is *worse* at than a well-supervised LLM.

## Problem it solves

Anthropic's "Skills" concept (a structured bundle of interdependent multi-file artifacts) is more general than tools, but skill creation today is:

1. **Manually authored** by humans, which is label-intensive at scale.
2. **Cognitively misaligned** between human authors and agent execution patterns. A human writing a SKILL.md envisions human-style instruction; the agent uses tool-call sequences and reasoning chains the human can't fully predict.
3. **Tested poorly** — single-pass generation (GPT writes a SKILL.md from a description) yields skills that pass surface inspection but fail at execution time.

CoEvoSkills targets all three: automate the authoring, align it to agent execution patterns, and ground it in actual skill-execution feedback.

## Core idea in one paragraph

Run two LLM sessions in parallel. The **Skill Generator** maintains a persistent context `C` initialised with the task instruction and a meta-skill template; on each iteration `i`, it samples a refined skill `𝒮(i+1) ∼ π_θ(· | 𝒮(i), C(i+1))` conditioned on accumulated diagnostics from the verifier. The **Surrogate Verifier** runs in an *information-isolated* session, observes only the task instruction `I` and the generator's output files `x(i)`, generates its own test suites `𝒱`, and emits structured failure diagnostics `ℱ` — *without ever seeing ground-truth tests*. A *separate* ground-truth oracle is invoked sparingly (≤ K=5 times); when surrogate-tests pass but oracle-tests fail, the verifier is told to strengthen its tests autonomously. Loop terminates when surrogate + oracle both pass, or when M=15 retries are exhausted, or when the context fills past β=0.7 utilisation. The result is a multi-file skill package with executable Python utilities, tested through agent eyes rather than through a human prompt.

## Mechanism (step by step)

### 1. The two components

**Skill Generator (G).** A frozen LLM with a persistent context window `C` initialised with `I` (the task instruction) and a meta-skill template (the SKILL.md schema + scaffolding for `scripts/`). At iteration `i`:

```
context  C(i+1) = C(i) ⊕ ℱ(i)
skill    𝒮(i+1) ∼ π_θ(· | 𝒮(i), C(i+1))
output   x(i+1) = run(𝒮(i+1), task)
```

The generator produces both procedural Markdown text *and* executable Python utility functions; the runtime invokes the Python where the SKILL.md instructs.

**Surrogate Verifier (V).** A *separate* frozen LLM in an *information-isolated* session — it never sees ground-truth tests. Inputs:

- `I`: the task instruction (verbal description, no test cases).
- `x(i)`: the skill's output files.

Outputs:

- A test suite `𝒱` the verifier itself synthesised.
- Structured diagnostics `ℱ` describing failures *qualitatively*: which behaviours, which inputs, which constraints.

`ℱ` is the feedback signal the Generator consumes. It is text, not a number. This is the key bandwidth choice — opaque scalar feedback (the trap EvoSkill exposes itself to) cannot tell the generator *why* something failed; structured diagnostics can.

### 2. The escalation rule

Two oracles in CoEvoSkills:

| Oracle | What it sees | What it returns | How often invoked |
|---|---|---|---|
| Surrogate Verifier | task instruction + skill output files | structured diagnostics + verifier-synthesised tests | every iteration |
| Ground-truth oracle | ground-truth test contents | opaque pass/fail | up to K = 5 times per skill |

When the surrogate verifier passes but the ground-truth oracle fails, the verifier receives feedback (without leaking the ground-truth content) to *strengthen its own test suite*. This is the co-evolution: the verifier gets better at predicting what the oracle will reject, the generator gets better at passing both.

The paper's argument: in production, the agent will not have a ground-truth oracle. The *deployed* loop is generator + surrogate. The training-time oracle is needed only to ground the surrogate; in steady state it can be retired.

### 3. Termination

The loop stops on **first** of:

- Surrogate verifier passes AND oracle passes (i.e., skill is complete).
- K = 5 oracle interventions exhausted.
- M = 15 surrogate retries exhausted.
- Context utilisation reaches β = 0.7 (prevent overflow).

Average per task: **4.1 surrogate verification cycles, 2.4 oracle rounds**. **Over 60% of tasks resolve within 2 oracle rounds** — most of the heavy lifting is the surrogate.

### 4. SkillsBench

The paper introduces SkillsBench:

- **87 tasks** across 11 professional domains.
- **Deterministic verifiers** — primary metric is pass-rate (proportion of tasks reaching reward = 1.0).
- **No subjective judgement** required — every task has a unambiguous pass/fail.

The 11 domains span natural science, engineering, finance, business, etc. The deterministic-verifier constraint is critical: it makes skill-quality measurable as a *number*, which is what enables the ablation arithmetic.

### 5. The five baselines

CoEvoSkills compares against five reference systems on the same SkillsBench:

1. **No-Skill Baseline.** Agent runs without any skill access. Floor.
2. **Self-Generated Skills.** One-pass generation: agent writes a SKILL.md from a description in a single LLM call. No iteration.
3. **CoT-Guided Self-Generation.** Five-step chain-of-thought prompt walks the agent through skill creation; still single-pass.
4. **Skill-Creator.** Anthropic's official `skill-creator` tool, run autonomously without human review.
5. **Human-Curated Skills.** Pre-installed skills from the SkillsBench release; authored by humans for the benchmark.

## Empirical results

### Primary on Claude Opus 4.6 + Claude Code

| System | Pass-rate | vs. no-skill | vs. human |
|---|---:|---:|---:|
| No-Skill | 30.6% | 0 | −22.9 |
| Self-Generated (1-pass) | ~30–34% | ~0 | ~−20 |
| CoT-Guided Self-Gen | ~30–34% | ~0 | ~−20 |
| Skill-Creator (autonomous) | ~30–34% | ~0 | ~−20 |
| Human-Curated | 53.5% | +22.9 | 0 |
| **CoEvoSkills** | **71.1%** | **+40.5** | **+17.6** |

Two stories.

**Story 1 — Single-pass skill generation barely helps.** All four baselines that don't iterate cluster within 4 points of the no-skill floor. The take-away: *generating a SKILL.md in one shot is barely better than no skill at all*. This is the case against `skill-creator`-style autonomous tools used naively.

**Story 2 — Iteration with a surrogate verifier dominates human authoring.** CoEvoSkills' +17.6 over human-curated is the entire point of the paper. The verifier-grounded co-evolutionary loop produces skills that *the human author would not have written* — and they work better.

### Cross-model transfer (the load-bearing experiment)

Take the skills evolved by Claude Opus 4.6, attach them to six other LLMs, run SkillsBench:

| Model | Pass-rate | vs. no-skill |
|---|---:|---:|
| GPT-5.2 (Codex) | 65.0% | +35.4 |
| Claude Sonnet 4.5 | 63.1% | +43.1 |
| Claude Haiku 4.5 | 54.5% | +44.1 |
| Qwen3-Coder-480B | 50.8% | +42.4 |
| DeepSeek V3 | 48.8% | +35.8 |
| Mistral Large 3 | 43.1% | +38.2 |

Same-family transfer (Opus → Sonnet/Haiku) is best, but cross-family is robust: every model gains +35 to +44 absolute points. The paper's interpretation: **evolved skills encode reusable task structures, not model-specific artifacts**. This is the strongest evidence in the corpus that *skills, not weights, are the right unit of capability transfer*.

### Ablation table

| Condition | Pass-rate | Δ vs. full |
|---|---:|---:|
| **Full CoEvoSkills** | **71.1%** | — |
| Without surrogate verifier | 41.1% | **−30.0** |
| Without evolution (single pass with verifier) | 48.6% | −22.5 |
| No-skill baseline | 30.6% | −40.5 |

Reading this table:

- The verifier is more important than evolution alone (−30 vs. −22.5).
- Evolution-without-verifier (relying on opaque oracle) gets you *less than half* the gain.
- Both together compound — the loop that drives improvement is verifier-feedback × iterative refinement.

### Convergence dynamics (Figure 2)

Skills converge within 5 iterations on average:

- Round 0 — matches no-skill baseline (30.6%).
- Round 2 — reaches 44%.
- Round 3 — surpasses human-curated (63%).
- Round 5 — converges at ~75%.

Most skills are *complete* by round 5; pushing past round 5 produces marginal gains and risks overfitting to the verifier's quirks.

## Three explicit takeaways (verbatim flavour)

The paper closes on three:

1. **"Agents create better skills than human-curated ones."** Captured agent reasoning patterns and tool-use strategies that human authors did not anticipate.
2. **"Skills are portable across model families."** Same-family is a +4.8 lift over cross-family on average, but transferability is strong (65–69% on diverse vendors).
3. **"Human–machine cognitive misalignment."** In the Natural Science sub-domain, human-curated skills *actively degrade* performance; self-evolved skills produce substantial gains. This is a counterintuitive result: on technically sophisticated content, human-authored guidance can hurt the agent.

## Skill complexity (the bundle picture)

| Skill source | SKILL.md lines | Python lines | Total |
|---|---:|---:|---:|
| Average human-curated (5 SKILL.md docs) | 1,096 (across 5) | 0 | 1,096 |
| Average CoEvoSkills-evolved | 64 procedure | 142 executable | 206 |

The evolved skills are **shorter and more executable**. Human authors write longer prose; the verifier-grounded loop converges on tight procedure + tested utility code. From a maintenance and audit perspective this is preferable: ~60 lines of procedural Markdown is reviewable; ~150 lines of tested Python is more reliable than ~1000 lines of prose-only instruction.

## Limitations

The paper is explicit about three:

1. **Single benchmark.** SkillsBench only. No tasks beyond it. Generalisation untested.
2. **Evolution cost.** 3000s timeout per task with up to 5× retry multiplier. Skill evolution is GPU-hour-class, not seconds. The paper does not quantify dollar cost.
3. **Context-overflow protection at β = 0.7.** A heuristic; aggressive context-tiering work (see [151–153 MEMTIER](151-memtier-why-flat-memory-breaks-at-72-hours.md)) could relax this.

Three further limitations visible from the architecture:

4. **Two-LLM dependency.** Generator + Verifier need two API access points (or two model families if cross-family discipline is enforced). This raises operational cost.
5. **Verifier hallucination** — if the surrogate verifier is *systematically wrong* in a domain (it passes invalid skills), the loop converges on garbage. The K = 5 oracle interventions are the safeguard but the bandwidth is narrow.
6. **No skill-decay or skill-pruning** — once a skill is in the library it persists; if domain drift makes it stale there is no removal mechanism.

## Related systems CoEvoSkills explicitly distinguishes from

The paper names six neighbouring systems and explains the gap:

- **SkillCraft, Yunjue, Live-SWE-Agent, Voyager** ([89](89-voyager-deep.md)) — focus on *tools*, not multi-file skill packages.
- **AutoSkill** ([167](167-autoskill-experience-driven-lifelong-learning.md)), **AutoRefine** — extract *prompt heuristics*, not executable artifacts.
- **SEAgent** — internalises into model weights (non-transferable).
- **EvoSkill (Alzubi et al.)** ([168](168-evoskill-coding-agent-skill-discovery.md)) — relies on **ground-truth failure diagnosis**; CoEvoSkills' surrogate-verifier discipline is the explicit alternative.

The taxonomy CoEvoSkills articulates: *(tool vs. skill) × (prompt-only vs. executable) × (parameter-frozen vs. parameter-updated) × (ground-truth-grounded vs. surrogate-grounded)*. CoEvoSkills occupies the (skill, executable, frozen, surrogate-grounded) corner — and the empirical claim is that this corner dominates.

## Where CoEvoSkills sits in the corpus

| Doc | Relationship |
|---|---|
| [04-skills](04-skills.md), [71-karpathy-skills](71-karpathy-skills-single-file-guardrails.md) | Multi-file skills are an extension of the SKILL.md format; CoEvoSkills makes the case that the *bundle* is the unit, not the document. |
| [89-voyager-deep](89-voyager-deep.md) | Voyager's growing skill library is conceptual ancestor; CoEvoSkills replaces Voyager's curiosity-driven-exploration with verifier-driven optimisation. |
| [154-ctx2skill](154-ctx2skill-self-evolving-context-skills.md) | Sibling: also uses an LLM-as-judge in a no-feedback regime. Ctx2Skill operates on dense documents; CoEvoSkills on coding tasks with deterministic verifiers. |
| [167-autoskill](167-autoskill-experience-driven-lifelong-learning.md), [168-evoskill](168-evoskill-coding-agent-skill-discovery.md), [170-skillrl](170-skillrl-recursive-skill-augmented-rl.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md) | Sibling and synthesis docs. |
| [156-heavyskill](156-heavyskill-parallel-reasoning-deliberation.md) | Argues the entire harness can be a skill; CoEvoSkills shows multi-file skill packages can match that ambition. |
| [69-agent-world](69-agent-world-self-evolving-training-arena.md) | Self-evolving arena meta-frame; CoEvoSkills is one shape (verifier co-evolution) of the arena dynamic. |
| [Polaris ADR-002](../projects/polaris/docs/concepts/adrs/adr-002-cross-model-adversarial-pair.md) | The cross-family adversarial-pair invariant is *exactly* what makes a verifier different from the generator; CoEvoSkills' surrogate-verifier discipline is the empirical case for ADR-002. |

## When to use CoEvoSkills (decision rule)

Reach for CoEvoSkills when **all** of the following hold:

- You have a coding/agent benchmark with **deterministic verifiers** (pass/fail per task).
- You can afford **two LLM endpoints** (generator + verifier; ideally different families).
- You want **multi-file skill packages** — procedural Markdown plus executable Python.
- You expect the resulting skills to **transfer across model families**.
- You're willing to invest GPU-hours per skill.

Reach for **EvoSkill** ([168](168-evoskill-coding-agent-skill-discovery.md)) instead if you only need single-folder skills and you're comfortable with ground-truth-grounded fitness.

Reach for **AutoSkill** ([167](167-autoskill-experience-driven-lifelong-learning.md)) if your input is dialogue rather than coding tasks.

Reach for **SkillRL** ([170](170-skillrl-recursive-skill-augmented-rl.md)) if you have parameter access and an active RL training loop.

## Bottom line

CoEvoSkills makes three contributions worth holding onto:

1. **Multi-file skill packages as the unit.** SKILL.md + executable Python is more reliable than SKILL.md alone.
2. **Surrogate-verifier-without-ground-truth.** The +30 ablation gap proves the verifier is more important than the evolution loop alone. *Structured textual feedback dominates opaque scalar reward.*
3. **A demonstrated cross-model transfer** — same skills, six LLM families, +35–44 absolute points. The clearest evidence in the 2026 corpus that *skills* (not weights) are the portable layer.

The naming collision with Alzubi et al.'s EvoSkill is unfortunate; both papers are real and complementary. CoEvoSkills lives at (skill, multi-file, frozen-weights, surrogate-grounded); EvoSkill at (skill, single-folder, frozen-weights, ground-truth-grounded). Together with AutoSkill and SkillRL they cover the four corners of the 2026 skill-evolution design space — synthesised in [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md).

---

**Citation:** Zhang H., Fan S., Zou H. P., Chen Y., Wang Z., Zhou J., Li C., Huang W.-C., Yao Y., Zheng K., Liu X., Li X., Yu P. S. *EvoSkills: Self-Evolving Agent Skills via Co-Evolutionary Verification.* arXiv:2604.01687v2, 2026.

**Repository:** Code release planned ("Code will be released" per arXiv abstract); not yet public at time of paper.
