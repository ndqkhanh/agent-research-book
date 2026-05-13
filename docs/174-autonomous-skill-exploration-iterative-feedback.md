# 174 — Autonomous Skill Exploration via Iterative Feedback (EXIF)

**Source.** Yongjin Yang, Sinjae Kang, Juyong Lee, Dongjun Lee, Se-Young Yun, Kimin Lee. *Automated Skill Discovery for Language Agents through Exploration and Iterative Feedback*. arXiv:[2506.04287](https://arxiv.org/abs/2506.04287) (Jun 2025).

**One-paragraph thesis.** EXIF (the paper's named system) plugs the gap between *AutoSkill* ([167](167-autoskill-experience-driven-lifelong-learning.md)) — frozen LLMs, dialogue-driven, no execution feedback — and *EvoSkill* ([168](168-evoskill-coding-agent-skill-discovery.md)) — frozen LLMs, ground-truth task scores. EXIF uses a *pair* of agents: an **exploration agent (Alice)** that proposes skills + tasks, and a **target agent (Bob)** that attempts the tasks; Alice evaluates Bob's performance and iterates. The system runs without human intervention on Webshop and Crafter, demonstrating that on-the-go skill discovery is viable in regimes where ground-truth pass/fail is unavailable but real execution is. The most striking result is the **same-model self-evolving** finding: when Alice and Bob share a model, the system improves further than when they're distinct — the loop becomes a one-model self-improvement primitive.

## §1 — Where this paper sits in the design space

`docs/171`'s axis-1 (feedback signal source) had four points:

```
ground-truth ←——→ surrogate ←——→ adversarial ←——→ no signal
```

EXIF lives at a sixth point — **execution-with-iterative-feedback** — softer than ground-truth pass/fail but harder than LLM-as-judge:

```
ground-truth ← offline-sim ← exploration+feedback ← surrogate ← adversarial ← no signal
                              (this paper)
```

The signal is *real environment execution* (Webshop / Crafter rollouts produce concrete observations), but the *grading* is LLM-driven (Alice judges Bob). This is a noisier-but-cheaper substitute for ground-truth scores, and crucially it works in **environments without canonical task definitions** — which is most of the long tail.

## §2 — Mechanism

Two agents on a shared model substrate:

### §2.1 — Alice: the exploration agent

Alice's job is to *expand the skill frontier*:

1. Read the environment description and any existing skill library.
2. Propose **a new candidate skill** (a name + a procedural description) covering an under-served part of the environment.
3. Generate **a task** that requires that skill to solve.
4. Hand the task to Bob.
5. Observe Bob's trajectory.
6. Score Bob's performance. The score becomes feedback that drives Alice's next proposal.

### §2.2 — Bob: the target agent

Bob's job is to *attempt the task using the current skill library*. Each rollout produces:

- A trajectory (sequence of actions + observations).
- A success / fail / partial outcome.
- An error trace if Bob got stuck.

### §2.3 — The iterative feedback loop

Where prior systems decoupled discovery (offline batch) from deployment (online use), EXIF runs the loop *online*:

- Alice's next proposal is conditioned on the *most recent N* of Bob's trajectories.
- Skills that helped Bob across multiple tasks are kept; skills that helped only once are re-evaluated against Alice-generated variants.
- The cycle continues until coverage saturates or the budget is exhausted.

### §2.4 — Same-model self-evolution

The paper's empirical surprise: when **Alice and Bob share a model**, the system improves *more* than when they're distinct families. Two explanations the authors offer:

1. **Shared error modes.** Bob's failures expose blind spots Alice naturally probes; a different-family Alice would generate tasks that don't surface Bob's specific weaknesses.
2. **Self-distillation.** Alice's scoring loop effectively distils Bob's own latent competence into explicit skill files Bob can re-read.

Both are interesting; neither is a full explanation. For harness implementers, the operational takeaway is: **don't enforce a cross-family separation between exploration and target unless ADR-002-style adversarial review is the goal**. AutoSkill / EvoSkill / CoEvoSkills make a different argument; EXIF says the trade-off depends on whether you want robustness (cross-family) or convergence speed (same-family).

## §3 — Results

EXIF demonstrates substantial performance improvements on **Webshop** (e-commerce navigation) and **Crafter** (open-world game). Specific pp gains are not in the abstract excerpt; the qualitative finding is that EXIF *iteratively expands the agent's capabilities without any human intervention*. The same-model self-evolving variant outperforms cross-model variants, which is a structural surprise relative to the [171](171-skill-self-evolution-2026-synthesis.md) wave's general endorsement of cross-family separation.

## §4 — Comparison vs. AutoSkill / EvoSkill / 173 (Offline-Sim)

|  | **AutoSkill** ([167](167-autoskill-experience-driven-lifelong-learning.md)) | **EvoSkill** ([168](168-evoskill-coding-agent-skill-discovery.md)) | **Offline-Sim** ([173](173-offline-sim-skill-discovery.md)) | **EXIF** (2506.04287) |
|---|---|---|---|---|
| Feedback signal | LLM judge | Ground-truth task scores | Offline simulator state diff | **Live execution + LLM judge** |
| Cross-family required? | Yes | Yes (executor ≠ reviewer) | Optional | **No — same-model is preferred** |
| Task generation | User dialogue | Benchmark | GNN-sampled API graph | **Alice (exploration agent)** |
| Promotion bar | Durability marker | Pareto frontier | Multi-variant success | **Iterative usefulness across rollouts** |
| Best regime | Personal-preference / dialogue | Ground-truth-rich coding | Software-automation with stable APIs | **Real environments without canonical tasks** |

## §5 — What harness implementers should steal

Three patterns transfer:

1. **Alice/Bob over a shared model is a viable single-LLM self-improvement primitive.** This is the cleanest pattern for harnesses that don't have a benchmark-grade ground-truth oracle but do have a real environment to execute against. Most CLI / agent harnesses live here.
2. **Iterative-feedback over rollouts is cheaper than Pareto search.** EvoSkill's Pareto frontier requires many parallel runs to estimate the frontier. EXIF's iterative loop converges with fewer runs because each Bob-rollout directly conditions Alice's next proposal — more akin to bandit exploration than population search.
3. **The same-model finding is a knob.** Polaris's ADR-002 cross-family invariant exists for *adversarial review*; EXIF's same-model finding suggests *exploration-driven discovery* is a different regime where the cross-family discipline can be relaxed if convergence speed matters more than robustness. Make the choice explicit per skill type.

## §6 — Polaris integration slot

```text
packages/polaris-skills/src/polaris_skills/research/
  exploration_loop.py        # NEW — Alice/Bob loop. Composes with v2.2
                             # heartbeat scheduler so exploration runs in
                             # the Reflection / Consolidation cycles.
  same_model_policy.py       # NEW — explicit policy: when is same-model
                             # exploration admitted vs. when does cross-
                             # family ADR-002 apply?
```

Bright-line addition (proposed):

```
BL-EXPLORATION-COST       Alice/Bob exploration counts against the
                          program's cost envelope; the loop refuses to
                          start a new round once spend would exceed
                          BL-SPEND-OVER. Auto-defer to next idle cycle.
```

## §7 — Lyra integration slot

```text
packages/lyra-core/src/lyra_core/skills/
  exploration_loop.py        # NEW — Alice/Bob loop driven by the existing
                             # subagent orchestrator. One subagent runs as
                             # Alice; another runs as Bob; both share the
                             # `smart` model slot.
```

This composes naturally with Lyra's existing **arena/elo.py** (already in `lyra-core/arena/`): each Alice-proposed skill enters an Elo tournament against the existing library. Skills that win their bracket promote; losers go back to Alice for refinement. This is the cleanest single integration point — Lyra has all the substrate.

## §8 — Where this fits

Pairs with:

- [167 — AutoSkill](167-autoskill-experience-driven-lifelong-learning.md) — EXIF generalises the dialogue corner to execution-driven settings.
- [168 — EvoSkill](168-evoskill-coding-agent-skill-discovery.md) — EXIF is the cheaper alternative when ground-truth scores aren't available.
- [173 — Offline-Sim Skill Discovery](173-offline-sim-skill-discovery.md) — sister paper at a different feedback point.
- [171 — Skill Self-Evolution Synthesis](171-skill-self-evolution-2026-synthesis.md) — extend the table with this row.
- [178 — Online Skill Discovery & Curation On-the-Go](178-online-skill-discovery-and-curation-on-the-go.md) — EXIF is one of three load-bearing online-discovery patterns.
