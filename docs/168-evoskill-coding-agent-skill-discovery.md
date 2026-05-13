# 168 — EvoSkill: Automated Skill Discovery for Multi-Agent Systems

**Paper.** Salaheddin Alzubi, Noah Provenzano, Jaydon Bingham, Weiyuan Chen, Tu Vu — *EvoSkill: Automated Skill Discovery for Multi-Agent Systems* — arXiv:2603.02766v1 [cs.AI / cs.MA] — submitted 3 March 2026.

**One-line definition.** EvoSkill is a *failure-driven* automated-skill-discovery framework that maintains a Pareto frontier of agent programs (Claude Code with attached skill folders) and improves the frontier through a three-agent loop — Executor, Proposer, Skill-Builder — that reads execution failures, proposes new or edited skills, materialises them as SKILL.md folders with optional helper scripts, and evicts dominated frontier members. It posts +7.3 / +12.1 / +5.3 absolute-percentage gains on OfficeQA / SealQA / BrowseComp respectively, with the underlying LLM frozen throughout.

## Why this paper matters

Three observations stack up.

**Observation 1 — Failure analysis beats positive imitation for skill discovery.** Traditional skill-library work (Voyager, [89-voyager-deep](89-voyager-deep.md); AutoSkill, [167](167-autoskill-experience-driven-lifelong-learning.md)) builds skills by imitating successful trajectories. Failures are signal-rich: they show where current capability ends. EvoSkill commits exclusively to failure-driven proposals — every iteration looks at the worst-scoring tasks, not the best.

**Observation 2 — Skills generalise where prompts and code don't.** AlphaEvolve and GEPA optimise codebases or prompts but produce artifacts tightly coupled to a specific model and task. EvoSkill's claim is that when you optimise at the *skill* level (a SKILL.md folder + scripts, model-frozen), the result transfers — and the paper's BrowseComp transfer experiment is the proof point: a skill discovered on SealQA improves a different benchmark by +5.3% with no further training.

**Observation 3 — Pareto frontiers > single-best.** The EvoSkill loop maintains `k=3` programs on a Pareto frontier rather than a single best-so-far. This is borrowed from evolutionary computation and prevents the system from getting stuck on a local optimum where one cluster of failure modes dominates the proposal stream.

In the broader skill-evolution map, EvoSkill is the **execution-grounded coding-agent** corner of the design space. Compare:

| | Dialogue | Document | Coding agent | Multi-file skills | RL |
|---|---|---|---|---|---|
| AutoSkill ([167](167-autoskill-experience-driven-lifelong-learning.md)) | ✓ | ✓ (via 4Doc) | partial (4OpenClaw) | – | – |
| **EvoSkill** | – | – | **✓** | partial (folder + scripts) | – |
| CoEvoSkills ([169](169-coevoskills-co-evolutionary-verification.md)) | – | – | ✓ | **✓** | – |
| SkillRL ([170](170-skillrl-recursive-skill-augmented-rl.md)) | – | – | partial | – | **✓** |
| Ctx2Skill ([154](154-ctx2skill-self-evolving-context-skills.md)) | – | **✓** | – | – | – |

## Problem it solves

Coding agents like Claude Code, Codex, and Cursor ship strong general-purpose capability. They underperform on **domain-specialised** tasks: financial analysis over US Treasury data, medical-record extraction, regulatory-document interpretation, semi-structured spreadsheet reasoning. Adding capability requires either (a) human-authored skills (slow, expensive, doesn't cover the long tail) or (b) fine-tuning (cost, latency, model lock-in).

EvoSkill targets the gap: **automatically discover the small number of high-leverage skills that bridge a frontier model into a domain niche, frozen weights, with a few dozen training examples**.

## Core idea in one paragraph

Treat skill discovery as evolutionary search over agent *programs* (a coding agent + its attached SKILL.md folder set). Each generation: pick a parent from the frontier (round-robin), score it on the training set, harvest the failures, hand them to a Proposer LLM that writes either a new skill or an edit to an existing one, hand the proposal to a Skill-Builder LLM that materialises it as a folder of files (SKILL.md + scripts/ + trigger metadata), evaluate the candidate on a validation split, admit it to the frontier if it dominates the weakest member or there's room. Repeat. Score the final frontier on a held-out test split, never seen during evolution. Optionally merge the unique skills from independent runs (skill-merge) for a final lift.

## Mechanism (step by step)

### 1. The three-agent loop

Three LLM-backed roles, each frozen:

- **Executor (A).** Runs an agent program on a task. The program is the underlying coding agent + its current skill folder.
- **Proposer (P).** Reads execution failures (scoring below threshold τ), reads accumulated feedback history H, proposes a textual *intervention*: either a new skill or an edit to an existing skill.
- **Skill-Builder (S).** Materialises the proposal into a structured folder:

```
skills/<slug>/
├── SKILL.md      # trigger metadata + instructions
├── scripts/      # optional helper code (Python or TypeScript)
└── …
```

### 2. The Pareto-frontier algorithm (Algorithm 1, paraphrased)

```
Initialise frontier G = { p_0 }                # p_0 = base agent, no extra skills
For t = 1 to T:
    parent p   = round_robin(G)
    failures F = { task ∈ training_set | score(p, task) < τ }
    proposal π = Proposer(failures F, history H)
    candidate p̃ = Skill_Builder(parent p, proposal π)
    score      = evaluate(p̃, validation_set V)
    If score > min(G) or |G| < k:
        G ← G ∪ { p̃ }
        if |G| > k:    evict argmin(G)
    H ← H ∪ { (π, score) }
Return G
```

`k = 3` by default. Round-robin parent selection ensures each frontier member contributes equally. The `min(G)` admission rule means the frontier is monotonically non-decreasing in worst-case score.

### 3. Stratified data splits

Three disjoint subsets, sized ~5–15% / ~7% / rest:

- **Training set.** Failure detection during evolution. The Proposer reads only failures from this split.
- **Validation set.** Frontier admission decisions. ~7% on OfficeQA.
- **Test set.** Reported numbers only. Never exposed to evolution.

This is a standard ML hygiene practice but worth flagging: skill-evolution papers that don't split out a never-seen test split tend to over-claim transfer. EvoSkill is explicit and reports test-set numbers throughout.

### 4. Git-backed versioning

Each candidate program is a *git branch* containing the skill folders + metadata. This is small but consequential:

- Reproducible lineage — `git log` of any frontier member shows the exact sequence of proposals that produced it.
- Cheap branching — frontier maintenance is `git branch <new>` not file-system copy.
- Rollback — if the test-set score drops, `git reset` is a one-line undo.

### 5. Scoring with fuzzy tolerance

OfficeQA scoring uses fuzzy matching with five tolerance levels: `0.0`, `0.01`, `0.025`, `0.05`, `0.10`. Each level is weighted; stricter thresholds dominate the score. The reported "exact-match" headline is the `0.0`-tolerance score.

### 6. Skill-merge (the final lift)

Run EvoSkill `n` times independently with different random seeds; each run produces a frontier `G_i`. The *unique* skills across all frontiers are merged into a single skill folder; the final agent uses this superset. The skill-merge run on OfficeQA is the strongest configuration (67.9% test accuracy) — the single-run best is 65.8%. Merging across runs captures complementary failure modes that a single evolutionary trajectory missed.

## Empirical results

### OfficeQA — grounded reasoning over US-Treasury data

Baseline: Claude Code with Claude Opus 4.5, 60.6% exact-match accuracy on the test split.

| Training split | 0%-tolerance accuracy | Δ vs. baseline |
|---|---|---|
| 5% (12 examples) | 63.4% | +2.8 |
| 10% (24 examples) | 65.8% | +5.2 |
| 15% (36 examples) | 64.5% | +3.9 (plateau) |
| **Skill-merge (across runs)** | **67.9%** | **+7.3** |

Two skills carry most of the lift:

- **Data Extraction Verification.** Rigorous protocol for numerical extraction from tables — addresses adjacent-cell misreads and unit/metric confusion (a common failure on financial tables).
- **Quantitative Analysis Methodology.** Structured guidance for financial analysis with mandatory validation checkpoints (sanity-check ranges, cross-reference reporting periods).

The plateau at 15% training is interesting: more data does not always help. The system saturates on the failure modes representable in the corpus; additional examples re-cover the same ground.

### SealQA — search-augmented QA with noisy retrieval

Baseline: 26.6% test accuracy.

After EvoSkill: 38.7%. Gain: **+12.1 percentage points**.

The dominant evolved skill: **search-persistence-protocol**. It encodes:

- Term-interpretation expansion — list every reasonable interpretation of an ambiguous query before searching.
- Three-source minimum verification — never report on the first hit alone.
- Enumeration-completeness checks — when asked for "all X", verify the list closes (no missing rows).
- Data-source follow-through — exhaust the search budget before reporting "unable to find".

This is the kind of failure-mode-bundled skill that human authors rarely write. It's procedural, defensive, and specific to the failure pattern of *retrieval-noise + agent-impatience*.

### BrowseComp — zero-shot transfer

Take the `search-persistence-protocol` evolved on SealQA, attach it to a fresh Claude Code agent, point it at BrowseComp (a *different* benchmark, no further training):

| Configuration | Test accuracy |
|---|---|
| Baseline (no skill) | 43.5% |
| With transferred skill | 48.8% |
| Δ | +5.3 |

This is the load-bearing experiment. The claim "skills transfer" is testable, and the test passes. Compare:

- Prompt-level optimisation (FunSearch, GEPA, AlphaEvolve) — historically transfers poorly across tasks.
- Skill-level optimisation (EvoSkill) — transfers without modification on this corpus.

If the result generalises, the implication is large: domain skill libraries become *capital* that compound across benchmarks, not consumables tied to one task.

## Ablations and observations

The paper's ablation set is lighter than CoEvoSkills' or SkillRL's. Key observations:

- **Variance across seeds is unmeasured** — the authors flag this as an explicit follow-up. The single-run vs. skill-merge gap (65.8% vs. 67.9% on OfficeQA) implies non-trivial seed sensitivity.
- **Frontier size `k`** — the default `k=3` is empirically chosen; no sweep reported.
- **Failure threshold `τ`** — the score below which a task counts as "failure" for proposal-generation; not reported in detail.
- **Proposer / Skill-Builder model identity** — same family as the Executor in the headline runs; cross-family ablation absent (compare CoEvoSkills' surrogate-verifier discipline at [169](169-coevoskills-co-evolutionary-verification.md)).

## Limitations

The authors are explicit about three:

1. **Single-domain experiments.** Three benchmarks (OfficeQA, SealQA, BrowseComp), all English, all primarily-text. Generalisation to multimodal coding tasks (vision + code) is open.
2. **Computational cost.** Each iteration is expensive — full agent run × validation-set evaluation. The paper does not give wall-clock numbers per generation.
3. **Single-run variance unstudied.** The skill-merge result implies that any one seed is missing some failure modes; how much variance across seeds remains an open question.

Three further limitations visible from the architecture:

4. **Same-family executor + proposer.** No cross-family adversarial discipline ([Polaris ADR-002](../projects/polaris/docs/concepts/adrs/adr-002-cross-model-adversarial-pair.md)). A failure-mode that the executor and proposer share is invisible to the loop.
5. **No skill-quality decay** — once a skill is on the frontier it stays unless dominated; a skill that helps early but harms later is hard to evict.
6. **Validation-set leakage risk** — the proposer reads training-set failures but the frontier is decided on the validation set; if the proposer's interventions implicitly target validation-style failures, the test-set transfer story weakens. The paper does not test for this.

## Discovered-skill flavours (worked examples)

### `Data Extraction Verification` (OfficeQA)

Generated by EvoSkill on Treasury-data tasks. Sketch (paper-paraphrased):

```markdown
# Goal
Extract a specific numeric value from a tabular financial document, returning
only that value with no surrounding commentary.

# Constraints & Style
- Identify the exact row by header AND row label, not by row index.
- Verify units (millions vs. billions, percent vs. basis points).
- Cross-reference reporting period explicitly.
- If two adjacent cells could match, refuse and ask for disambiguation.

# Workflow
1. Locate the table by section header.
2. Identify the row by exact-match label.
3. Identify the column by exact-match header.
4. Read the cell value.
5. Verify units and period.
6. Return only the value with units.
```

This skill was discovered, not authored. It addresses the specific failure mode — adjacent-cell misreads, metric confusion — that the proposer surfaced from training-set failures.

### `search-persistence-protocol` (SealQA, transfers to BrowseComp)

```markdown
# Goal
Answer search-augmented questions completely and accurately even when
retrieval is noisy.

# Constraints & Style
- Never accept the first search result as definitive.
- Require evidence from at least three independent sources.
- For "list all X" questions, do not stop until enumeration is verifiably
  complete.
- Before reporting "unable to find", explicitly enumerate the search
  strategies attempted and confirm none yielded results.

# Workflow
1. Expand the query into all reasonable interpretations.
2. Issue parallel searches across interpretations.
3. Triangulate facts across ≥ 3 sources before committing.
4. For enumeration questions, verify completeness via an explicit closure check.
5. Only report failure after a documented exhaustion of the search budget.
```

Note this skill encodes *agent discipline*, not domain knowledge. That's why it transfers — the underlying problem (retrieval noise + agent impatience) is shared across benchmarks.

## Comparison to neighbouring systems

| Dimension | EvoSkill | AutoSkill ([167](167-autoskill-experience-driven-lifelong-learning.md)) | CoEvoSkills ([169](169-coevoskills-co-evolutionary-verification.md)) | SkillRL ([170](170-skillrl-recursive-skill-augmented-rl.md)) | Ctx2Skill ([154](154-ctx2skill-self-evolving-context-skills.md)) |
|---|---|---|---|---|---|
| Input modality | Coding-agent traces | Dialogue + docs + traces | Coding-agent traces | RL training trajectories | Single dense document |
| Feedback signal | Ground-truth task scores (training split) | Textual judgement (P_judge) | Surrogate verifier + opaque oracle | RL reward | Adversarial self-play |
| Skill artifact | SKILL.md folder + scripts | Single SKILL.md | Multi-file skill package | JSON SkillBank | Markdown system-prompt prefix |
| Parameters | Frozen | Frozen | Frozen | Trained (RL) | Frozen |
| Frontier mechanism | Pareto k=3 | Top-K retrieval | Iterative refinement | RL co-evolution | Cross-time replay |
| Evidence transfer? | Yes (BrowseComp +5.3) | Untested | Yes (6 LLMs +35–43pp) | Cross-task on 7 search tasks | Cross-model demonstrated |

EvoSkill's distinguishing trait: **the only one of the four that uses ground-truth task scores explicitly as the evolutionary fitness signal**. AutoSkill cannot (no labels in chat). CoEvoSkills won't (it explicitly avoids ground-truth dependence). SkillRL absorbs the signal into RL reward instead.

## Where EvoSkill sits in the corpus

| Doc | Relationship |
|---|---|
| [04-skills](04-skills.md), [71-karpathy-skills](71-karpathy-skills-single-file-guardrails.md) | EvoSkill produces SKILL.md folders compatible with the Anthropic format. |
| [89-voyager-deep](89-voyager-deep.md) | Voyager is the conceptual ancestor; EvoSkill replaces curiosity-driven exploration with failure-driven optimisation and adds a Pareto frontier. |
| [85-alphaevolve](85-alphaevolve.md) | Same evolutionary search frame; AlphaEvolve targets code, EvoSkill targets skills. The paper explicitly distinguishes skill-level from code-level optimisation as the source of better transfer. |
| [154-ctx2skill](154-ctx2skill-self-evolving-context-skills.md) | Sibling: skill-level adversarial self-play in the *no-feedback* regime. |
| [167-autoskill](167-autoskill-experience-driven-lifelong-learning.md), [169-coevoskills](169-coevoskills-co-evolutionary-verification.md), [170-skillrl](170-skillrl-recursive-skill-augmented-rl.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md) | Sibling and synthesis docs. |
| [69-agent-world](69-agent-world-self-evolving-training-arena.md) | Agent-World is the broader self-evolving-arena; EvoSkill is one specific shape of the loop. |
| [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md) | Hermes outcome-fed refinement is the single-program version of EvoSkill's Pareto-frontier search. |

## When to use EvoSkill (decision rule)

Reach for EvoSkill when **all** of the following hold:

- You have a coding (or coding-adjacent) agent and a benchmark with **binary or scalar success per task**.
- You have at least a few dozen training examples (12–36 worked in the paper).
- You can afford full agent runs as the evaluation step (the search is expensive — budget GPU-hours, not seconds).
- You want the resulting skills to **transfer** to neighbouring benchmarks without further training.
- You can attach the skills as files (`skills/<slug>/SKILL.md`) — i.e., your runtime supports the Anthropic Skills format or equivalent.

Reach for **CoEvoSkills** ([169](169-coevoskills-co-evolutionary-verification.md)) instead if you want multi-file packages with a surrogate-verifier discipline (no ground-truth dependence).

Reach for **AutoSkill** ([167](167-autoskill-experience-driven-lifelong-learning.md)) if your input is dialogue and there is no ground-truth signal.

Reach for **SkillRL** ([170](170-skillrl-recursive-skill-augmented-rl.md)) if you have parameter access and an RL training loop.

## Bottom line

EvoSkill makes three contributions worth holding onto:

1. **Failure-driven proposal generation** — the right signal source for skill discovery is failures, not successes.
2. **Pareto-frontier search at the skill-set level** — `k=3` programs in flight, evolved by round-robin, evicted by domination.
3. **A demonstrated transfer result** — `search-persistence-protocol` evolved on SealQA improves BrowseComp by +5.3% with no further training. *Skills are capital that compound, not consumables that one-shot.*

The paper's caveats — small benchmark set, single-domain experiments, unstudied seed variance — are real but bounded. The structural design (three-agent loop, Pareto frontier, git-backed versioning) is reusable in any execution-grounded coding-agent setting today.

---

**Citation:** Alzubi S., Provenzano N., Bingham J., Chen W., Vu T. *EvoSkill: Automated Skill Discovery for Multi-Agent Systems.* arXiv:2603.02766v1, 2026.

**Repository:** Code release status not announced at time of paper. Watch the arXiv listing for updates.
