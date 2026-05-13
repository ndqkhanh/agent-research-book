# 170 — SkillRL: Evolving Agents via Recursive Skill-Augmented Reinforcement Learning

**Paper.** Peng Xia, Jianwen Chen, Hanyang Wang, Jiaqi Liu, Kaide Zeng, Yu Wang, Siwei Han, Yiyang Zhou, Xujiang Zhao, Haifeng Chen, Zeyu Zheng, Cihang Xie, Huaxiu Yao — *SkillRL: Evolving Agents via Recursive Skill-Augmented Reinforcement Learning* — arXiv:2602.08234 [cs.LG] — submitted 9 February 2026. Code: [github.com/aiming-lab/SkillRL](https://github.com/aiming-lab/SkillRL) — MIT licence.

**One-line definition.** SkillRL is the *only* skill-evolution framework in the 2026 wave that **trains the agent's policy** alongside the skill library: a hierarchical `SkillBank` (general / task-specific / common-mistakes) is distilled from successful and failed RL trajectories, retrieved adaptively at inference time, and **recursively co-evolved with the policy** during RL training — yielding +15.3% over strong baselines on ALFWorld + WebShop + 7 search-augmented tasks while compressing the policy's effective context by 10–20% relative to raw-trajectory memory.

## Why this paper matters

Three structural decisions distinguish SkillRL from the rest of the 2026 skill-evolution corpus.

**Decision 1 — Skills as a *training-time* artifact, not just an inference-time one.** AutoSkill ([167](167-autoskill-experience-driven-lifelong-learning.md)), EvoSkill ([168](168-evoskill-coding-agent-skill-discovery.md)), CoEvoSkills ([169](169-coevoskills-co-evolutionary-verification.md)) all freeze the underlying model and evolve skills around it. SkillRL does the opposite: while RL is updating the policy weights, the SkillBank is *also* updating, and they co-evolve. The skills inform the policy's exploration; the policy's failures inform the skills.

**Decision 2 — Three-tier hierarchy: general / task-specific / common-mistakes.** Most skill libraries are flat. SkillRL splits skills along *generality × valence*:

- **General skills** — universal strategic patterns (e.g., "Systematic Exploration"). Apply across tasks.
- **Task-specific skills** — heuristics organised by task type (`pick_and_place`, `clean`, `heat`, `cool`, `examine` for ALFWorld; product-search, attribute-filter, etc. for WebShop).
- **Common mistakes** — failure lessons with descriptions, causes, and avoidance strategies. The negative space.

The `common_mistakes` tier is the SkillRL twist on [81-reasoningbank](81-reasoningbank.md): failure-distillation as a first-class memory layer, but here it co-trains with the policy.

**Decision 3 — Adaptive retrieval at inference, two modes.** General-purpose retrieval (template keyword matching or embedding similarity); task-specific retrieval gated by the task header. Two retrieval `top_k` knobs (general and task-specific) keep the injected context bounded. The token-compression metric (10–20% reduction over raw-trajectory memory) is the case for skills as memory primitives.

In the 2026 corpus, SkillRL is the **RL-coupled** corner of the design space. Where the others ask *"how do I evolve skills given a frozen model?"*, SkillRL asks *"how do I co-train a policy and a skill library to converge faster than either alone?"*

## Problem it solves

Three concrete gaps in agent training:

1. **Raw-trajectory memory is token-heavy.** Standard memory-based RL agents replay past trajectories as context, which (a) blows up the prompt budget and (b) feeds the policy redundant low-signal tokens.
2. **Skills evolved against a frozen model don't co-evolve with the policy.** AutoSkill / EvoSkill / CoEvoSkills assume the policy is fixed. In RL, the policy *is changing every batch*; skills extracted yesterday may be stale tomorrow.
3. **No clean signal source for skills in dense-reward environments.** Most skill-evolution loops require execution feedback that is sparse for many real RL tasks (delayed reward, partial observability). SkillRL distils both successes and failures into the SkillBank during RL training itself, using the validation-failure rate as the recursion trigger.

## Core idea in one paragraph

Run two coupled processes during RL training: (a) the standard PPO/RLHF/GRPO policy update, and (b) a **SkillBank update** that reads the validation rollouts, identifies failure patterns, and recursively edits the SkillBank — adding new skills, refining existing ones, and adding common-mistake entries. At each rollout, retrieve the top-K most-relevant general skills and top-K task-specific skills, render them as context, prepend to the policy's prompt. As the policy improves, easier skills become redundant and harder ones get added; as the SkillBank improves, the policy's exploration is better-shaped. After convergence, the deployed agent uses the final SkillBank as a static prompt-prefix; no further training is needed.

## Mechanism (step by step)

### 1. The SkillBank schema

Three top-level keys in JSON:

```json
{
  "general_skills": {
    "GS-001": {
      "skill_id": "GS-001",
      "title": "Systematic Exploration",
      "principle": "When the goal location is unknown, ...",
      "when_to_apply": "Tasks with no fixed object location."
    },
    "GS-002": { ... }
  },
  "task_specific_skills": {
    "pick_and_place": [ {skill_id, title, principle, when_to_apply}, ... ],
    "clean":          [ ... ],
    "heat":           [ ... ],
    "cool":           [ ... ],
    "examine":        [ ... ]
  },
  "common_mistakes": [
    {
      "title": "Premature commitment to wrong room",
      "description": "Agent picks a room based on partial observation ...",
      "cause": "Insufficient exploration before committing.",
      "avoidance": "Visit ≥ 3 rooms before opening containers."
    }
  ]
}
```

A skill entry is `(skill_id, title, principle, when_to_apply)` — four fields, deliberately under-specified to keep skills compact.

A common-mistake entry is `(title, description, cause, avoidance)` — four fields, parallel structure but with diagnostic semantics.

### 2. Distillation from trajectories

After each rollout batch, SkillRL runs an *experience-based distillation*:

```
for each successful trajectory τ ∈ batch:
    pattern = extract_pattern(τ)
    candidate = (skill_id, title, principle, when_to_apply)
    if candidate is novel and generalisable:
        add to general_skills
    elif candidate matches a task family:
        add to task_specific_skills[family]

for each failed trajectory τ ∈ batch (validation set):
    failure_mode = analyse_failure(τ)
    mistake = (title, description, cause, avoidance)
    if failure_mode is novel:
        add to common_mistakes
```

The "novelty / generalisability" judgement uses an LLM call (similar to AutoSkill's `P_judge`). Skills duplicating existing entries are merged or dropped; common-mistakes get added if the failure mode is not yet covered.

### 3. Recursive evolution trigger

Co-evolution does not run every step (too expensive, too noisy). It triggers on **validation-failure rate**:

```python
if validation_success_rate < update_threshold:    # default 0.4
    new_skills = distill_from_validation_failures(...)
    add up to max_new_skills (default 3) to SkillBank
    log update event for tracing
```

The `update_threshold` (default 0.4) and `max_new_skills` (default 3) are the two hyperparameters that govern co-evolution rate. Below threshold = the policy is struggling, time to add skills; above threshold = the policy is fine, leave the SkillBank alone.

### 4. Adaptive retrieval at inference

Two retrieval modes, selectable per environment:

- **`template`** — keyword matching against `when_to_apply` strings. Fast, transparent, brittle.
- **`embedding`** — semantic similarity using `Qwen3-Embedding-0.6B`. Slower, more flexible.

Two `top_k` knobs:

- `top_k = 6` — general skills injected per episode.
- `task_specific_top_k` — task-specific skills, gated by the task family header.

The injected context for one rollout looks like:

```
# Skills

## General (top-6 retrieved by relevance to the query)
- GS-001: Systematic Exploration — When goal is unknown, ...
- GS-008: Sub-goal decomposition — When ...
- ...

## Task-specific (pick_and_place, top-3)
- TS-PP-002: Verify object location before pickup ...
- ...

## Common mistakes to avoid
- Premature commitment to wrong room — Visit ≥ 3 rooms ...
- ...

# Task
[task description]
```

The whole skill-prelude is bounded by `top_k × avg_skill_size` — typically a few hundred tokens, vs. multi-thousand-token raw-trajectory memory.

### 5. The token-compression result

> "10-20% token compression compared to raw trajectory storage"

This is the structural argument for skills as a memory primitive: the same predictive signal in 80–90% of the tokens. For RL training, where every rollout is many forward passes, this is meaningful compute savings.

## Empirical results

### Headline

| Benchmark | SkillRL | Baseline | Δ |
|---|---|---|---|
| ALFWorld | reported state-of-the-art | strong baselines | **+15.3%** average |
| WebShop | reported state-of-the-art | strong baselines | included |
| 7 search-augmented tasks | reported state-of-the-art | strong baselines | included |

The +15.3% is the average lift over the strongest comparable baselines. The paper claims robustness as task complexity increases — i.e., the gap grows on the harder tasks.

### Token efficiency

Same predictive signal at 80–90% of the tokens, vs. raw-trajectory memory replay. The SkillBank is the compression.

### Robustness as task complexity scales

> "Maintaining robustness as task complexity increases"

This is the structural claim that *recursive evolution* matters: as task difficulty scales, the SkillBank can grow new skills to cover harder cases; a static skill library cannot.

## Implementation notes (from the repo)

### Install

```bash
git clone https://github.com/aiming-lab/SkillRL.git
cd SkillRL
pip install -r requirements.txt
pip install vllm==0.11.0
pip install flash-attn==2.7.4.post1
pip install -e .
```

Plus environment-specific setup for ALFWorld, WebShop, and Search.

### Repo structure

```
SkillRL/
├── agent_system/        # Core agent implementation
├── skill_generation/    # Scripts for generating skill banks
├── memory_data/         # Pre-generated skills and training data
├── gigpo/               # Supporting components (GIGPO = Generalised Iterated GRPO?)
├── verl/                # RL training framework integration (Volcano Engine RL)
├── examples/            # Training scripts for different environments
└── …
```

The `verl/` integration is interesting: SkillRL plugs into a production RL framework rather than reimplementing PPO/GRPO. The choice means SkillRL is *production-friendly* for teams already using `verl`.

### Key hyperparameters (config keys)

| Key | Default | Meaning |
|---|---|---|
| `enable_dynamic_update` | true | Activate recursive evolution |
| `update_threshold` | 0.4 | Validation success-rate below which evolution triggers |
| `max_new_skills` | 3 | Per-cycle cap on additions |
| `top_k` | 6 | General skills retrieved per episode |
| `task_specific_top_k` | varies | Task-specific skills retrieved |
| `retrieval_mode` | template / embedding | Retrieval strategy |

### Dependencies

- `vllm==0.11.0` — fast inference for rollouts.
- `flash-attn==2.7.4.post1` — efficient attention.
- `LLaMA-Factory` — for the SFT stage (warm-start before RL).
- `Qwen3-Embedding-0.6B` — embedding-mode retrieval.
- Environment-specific: `alfworld`, `gymnasium`, etc.

## Limitations

The paper does not include an explicit limitations section in the abstract; from the architecture and reproducibility constraints I see:

1. **Parameter access required.** Unlike AutoSkill / EvoSkill / CoEvoSkills, SkillRL is *not* applicable to closed-source models. The recursive evolution mechanism is coupled to RL gradient updates.
2. **Three benchmarks, all sequential-decision-making.** ALFWorld + WebShop + 7 search tasks are all sequential-action environments. The framework's applicability to single-shot generation tasks (e.g., code generation, text rewriting) is untested.
3. **Two-tier skill hierarchy may not generalise to all domains.** The general / task-specific split works because ALFWorld tasks naturally fall into 5–10 families. For domains without a clean task-family taxonomy (e.g., open-ended dialogue), the structure may not apply.
4. **`update_threshold = 0.4` is a heuristic.** Different environments likely need different triggers; the paper does not characterise sensitivity to this hyperparameter.
5. **Co-evolution can chase its tail.** If the policy and SkillBank update simultaneously, the policy may be optimising against a SkillBank that is also changing, which can destabilise training. The paper claims robustness but does not show convergence-rate ablations across `update_threshold` values.

## Comparison to neighbouring systems

| Dimension | **SkillRL** | AutoSkill ([167](167-autoskill-experience-driven-lifelong-learning.md)) | EvoSkill ([168](168-evoskill-coding-agent-skill-discovery.md)) | CoEvoSkills ([169](169-coevoskills-co-evolutionary-verification.md)) | Ctx2Skill ([154](154-ctx2skill-self-evolving-context-skills.md)) |
|---|---|---|---|---|---|
| Parameter access | **Required** | Frozen | Frozen | Frozen | Frozen |
| Skill artifact | JSON 3-tier SkillBank | SKILL.md (single file) | SKILL.md folder + scripts | Multi-file package | Markdown system-prompt prefix |
| Feedback signal | RL reward + validation failure rate | Textual judgement (P_judge) | Ground-truth task scores | Surrogate-verifier diagnostics | Adversarial self-play |
| Co-evolution with policy | **Yes** (the loop's identity) | No | No | No | No |
| Common-mistakes tier | **Yes** (first-class) | Implicit (anti-patterns) | No | No | No |
| Token compression | 10–20% over trajectories | n/a | n/a | n/a | n/a |
| Transfer across models | Same family only | Untested | Yes (BrowseComp +5.3) | Yes (6 LLMs +35–44pp) | Yes (cross-model) |

SkillRL's distinguishing trait: **the only one that uses the RL loop itself as the optimisation engine for the SkillBank**. This is power and constraint at once — power because policy and skills can co-converge; constraint because closed models are out.

## Where SkillRL sits in the corpus

| Doc | Relationship |
|---|---|
| [04-skills](04-skills.md), [71-karpathy-skills](71-karpathy-skills-single-file-guardrails.md) | The Anthropic SKILL.md format; SkillRL's JSON schema is the RL-friendly variant of the same idea. |
| [89-voyager-deep](89-voyager-deep.md) | Voyager grew a skill library through curiosity-driven RL; SkillRL grows it through reward-driven RL with explicit recursion. |
| [81-reasoningbank](81-reasoningbank.md) | ReasoningBank is *failures-as-memory*; SkillRL's `common_mistakes` is the same primitive, integrated into the RL training loop. |
| [69-agent-world](69-agent-world-self-evolving-training-arena.md) | Agent-World is the broader self-evolving arena; SkillRL is one specific shape (RL + skill library co-training). |
| [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md) | Hermes outcome-fed refinement is the API analogue; SkillRL is the RL-grounded version. |
| [79-skill-rag](79-skill-rag.md) | Same retrieval-side idea; SkillRL adds the training-side. |
| [Polaris ADR-002](../projects/polaris/docs/concepts/adrs/adr-002-cross-model-adversarial-pair.md) | SkillRL does *not* enforce cross-family adversarial review (single policy, single SkillBank-distiller); a follow-up could harden this. |
| [167-autoskill](167-autoskill-experience-driven-lifelong-learning.md), [168-evoskill](168-evoskill-coding-agent-skill-discovery.md), [169-coevoskills](169-coevoskills-co-evolutionary-verification.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md) | Sibling and synthesis docs. |

## When to use SkillRL (decision rule)

Reach for SkillRL when **all** of the following hold:

- You have **parameter access** to the model — open-weights or self-hosted.
- You have an **active RL training pipeline** (PPO / GRPO / RLHF) you can integrate with.
- Your task is **sequential decision-making** with a clean reward signal.
- You can afford the *training-time* compute cost (several days of GPU-hours minimum).
- You want the deployed agent to use a **compact prompt-prefix** rather than verbose memory replay.

Reach for **CoEvoSkills** ([169](169-coevoskills-co-evolutionary-verification.md)) instead if your model is closed-source.

Reach for **EvoSkill** ([168](168-evoskill-coding-agent-skill-discovery.md)) if you want frozen-model failure-driven discovery without RL.

Reach for **AutoSkill** ([167](167-autoskill-experience-driven-lifelong-learning.md)) if your input is dialogue, not RL trajectories.

## Bottom line

SkillRL makes three contributions worth holding onto:

1. **A three-tier SkillBank schema** (general / task-specific / common-mistakes) that decomposes the skill library along the *generality × valence* axes. The `common_mistakes` tier as a first-class citizen is the most underrated idea here.
2. **Recursive co-evolution gated by validation failure rate** — skills update when the policy struggles, idle when it doesn't. Two hyperparameters (`update_threshold`, `max_new_skills`) and an `enable_dynamic_update` flag are the entire control surface.
3. **A demonstrated 10–20% token compression over raw-trajectory memory replay** — *skills are the right compression of experience*, not memory replay buffers.

The hard constraint — parameter access required — is the real cost. For teams running open-weights models in production, SkillRL is the obvious framework to slot into the RL loop. For teams stuck on closed-source frontier models, CoEvoSkills or AutoSkill are the right adjacent choices.

---

**Citation:** Xia P., Chen J., Wang H., Liu J., Zeng K., Wang Y., Han S., Zhou Y., Zhao X., Chen H., Zheng Z., Xie C., Yao H. *SkillRL: Evolving Agents via Recursive Skill-Augmented Reinforcement Learning.* arXiv:2602.08234, 2026.

**Repository:** [github.com/aiming-lab/SkillRL](https://github.com/aiming-lab/SkillRL) — MIT licence.
