# 56 — The Self-Evolving Agents (SEA) Landscape, April 2026

**Scope.** This file is a landscape overview of Self-Evolving Agents (SEA) as the field stands on 2026-04-20. It sits above the individual-paper notes elsewhere in this corpus — Reflexion ([14](14-reflexion.md)), Voyager ([19](19-voyager-skill-libraries.md)), Autogenesis ([36](36-autogenesis-self-evolving-agents.md)), Hyperagents ([45](45-hyperagents-self-modification.md)), the Adaptation of Agentic AI survey ([47](47-adaptation-of-agentic-ai-survey.md)), and Hermes ([55](55-hermes-agent-self-improving.md)) — and tries to answer "where is this whole area going?" rather than "how does paper X work?".

Everything below is anchored to a specific paper (with URL in References), or explicitly flagged as *landscape interpretation*. No benchmark numbers are invented; where a number appears, it is from the source paper's abstract or webpage as cited.

## What "self-evolving" means in 2026

A **Self-Evolving Agent (SEA)** is an agent system that measurably improves *after deployment* through an automated loop — proposing, evaluating, and committing changes to some part of itself — without a human in the inner loop of every change. Two 2026 surveys converge on this framing: Gao et al.'s "Survey of Self-Evolving Agents" (arXiv:2507.21046, last revised January 2026) organises the field along **What / When / How / Where to evolve**, and Fang et al.'s "Comprehensive Survey of Self-Evolving AI Agents" (arXiv:2508.07407) abstracts any SEA into four components: **System Inputs, Agent System, Environment, Optimisers**.

The definition deliberately excludes three things that look adjacent but are not SEA:

- **Stateless agent loops** (plain ReAct, plain ToT) — they improve the current task only, not future tasks.
- **Offline RL fine-tuning** — a human picks when to retrain; the agent is not proposing the update.
- **In-context self-refinement within one task** (Self-Refine, Chain-of-Verification, [18](18-chain-of-verification-self-refine.md)) — the improvement does not outlive the task.

What remains is a spectrum: Reflexion ([14](14-reflexion.md)) updates a short reflection memory; Voyager ([19](19-voyager-skill-libraries.md)) grows a code skill library; Darwin Gödel Machine (arXiv:2505.22954) rewrites its own scaffold; Hermes ([55](55-hermes-agent-self-improving.md)) promotes completed workflows to skills at the harness layer; Hyperagents ([45](45-hyperagents-self-modification.md)) edit the meta-level editing mechanism itself. All count as SEA; they differ in what they change and how fast.

## A taxonomy of SEA systems

Five orthogonal axes have stabilised in the 2026 literature. A given system is a point in this five-dimensional space, not a category.

### Axis 1 — *When* the update fires

- **Online / intra-episode.** Live-SWE-agent (arXiv:2511.13646) modifies its own scaffold during a live SWE-bench attempt.
- **Online / inter-episode.** Reflexion after each episode; Hermes after each completed workflow.
- **Offline / batch.** Darwin Gödel Machine evolves an archive of agents against held-out benchmarks; AlphaEvolve runs evolutionary search on a scored population.

### Axis 2 — *What* the update changes

This is the most useful cut. Two broad families:

- **Parameter updates** — SFT, DPO, RLVR on the base model. SAGE (arXiv:2512.17102) is the 2026 exemplar: GRPO augmented with a skill library.
- **Artifact updates** — the model's weights are frozen, but something *around* the model changes. Artifacts in turn split into:
  - **Prompt / context.** Agentic Context Engineering (ACE, arXiv:2510.04618) evolves a structured playbook.
  - **Skills / tools.** Voyager, Hermes, SkillX (arXiv:2604.04804), SAGE's library side.
  - **Scaffold / agent code.** DGM, Live-SWE-agent, Hyperagents.
  - **Memory schema.** Trajectory-Informed Memory Generation (arXiv:2603.10600); Autogenesis's Memory resource type ([36](36-autogenesis-self-evolving-agents.md)).
  - **Meta-procedure (the update rule itself).** DGM-H / Hyperagents.

This is exactly the split the Adaptation survey ([47](47-adaptation-of-agentic-ai-survey.md)) captures with its agent-side / tool-side dichotomy; Axis 2 refines it for the SEA subset.

### Axis 3 — *Reward source*

- **Self-signal.** Agent grades itself (A2 in [47](47-adaptation-of-agentic-ai-survey.md)). Fragile: Reflexion's "shared-blind-spot recursion" appears here.
- **Verifier / environment.** Unit tests, compilers, code executors. Absolute Zero Reasoner (arXiv:2505.03335) is the purest 2026 case — a code executor is the *only* external signal, with zero training data.
- **LLM-as-judge.** A different model scores trajectories ([21](21-llm-as-judge-trajectory-eval.md)). Cheap, plausible, and reward-hackable.
- **Human.** Slowest, strongest. Usually reserved for meta-level changes (Hyperagents recommend human review on meta-policy edits).

### Axis 4 — *Search operator*

How candidate updates are generated:

- **Verbal / reflective.** "Write down what went wrong." Reflexion, ERL (arXiv:2603.24639).
- **Gradient-like textual.** TextGrad / EvoAgentX — backpropagate a natural-language critique across a pipeline.
- **Evolutionary.** Population + mutation + selection. DGM, AlphaEvolve, Digital Red Queen, Imbue's Darwinian Evolver.
- **RL with verifiable rewards.** SAGE, Absolute Zero, GLOVE.
- **Protocol-level patching.** Autogenesis ([36](36-autogenesis-self-evolving-agents.md)) — typed, versioned, rollbackable resource edits.

### Axis 5 — *Granularity of commit*

- **Append-only** (Reflexion memory, Voyager skill additions). Easy to reason about; library rot is the hazard.
- **Atomic version bump** (Autogenesis: `prompt://researcher@v4 → v5`). Rollback is a primitive.
- **Destructive rewrite** (DGM edits scaffold in place in a child archive node). The archive preserves history; the child may be incompatible.

## Landmark papers, 2023–2026 (chronological)

A one-line linkage between each paper and the existing corpus. TL;DRs are anchored to each paper's own abstract.

- **Reflexion** (Shinn et al., arXiv:2303.11366, 2023) — verbal reinforcement across episodes; the prototype of A2-paradigm SEA. See [14-reflexion.md](14-reflexion.md).
- **Voyager** (Wang et al., arXiv:2305.16291, 2023) — curriculum + code skill library in Minecraft; canonical artifact-update SEA. See [19-voyager-skill-libraries.md](19-voyager-skill-libraries.md).
- **TextGrad** (Yuksekgonul et al., arXiv:2406.07496, 2024) — natural-language "gradients" backpropagated through an agent pipeline; the optimiser substrate for many 2026 systems. *Landscape interpretation:* TextGrad is to SEA optimisers what PyTorch was to deep learning: a useful abstraction, not yet the winner.
- **Absolute Zero Reasoner** (Zhao et al., arXiv:2505.03335, 2025; NeurIPS 2025 spotlight) — self-play reasoning with zero external data, verified by a code executor; the most extreme point on Axis 3 (verifier-only).
- **Darwin Gödel Machine** (Zhang et al., arXiv:2505.22954, 2025) — evolutionary archive of self-rewriting coding agents; SWE-bench 20.0%→50.0%, Polyglot 14.2%→30.7% per the abstract.
- **AlphaEvolve** (Novikov et al., arXiv:2506.13131, 2025) — Gemini-driven evolutionary code search; found a 48-multiplication algorithm for 4×4 complex matrix multiplication, improving on Strassen 1969. Not strictly an *agent* SEA but an important reference for the evolutionary-operator family.
- **Survey of Self-Evolving Agents** (Gao et al., arXiv:2507.21046, 2025; rev. Jan 2026) — the What/When/How/Where taxonomy used throughout this file.
- **Comprehensive Survey of Self-Evolving AI Agents** (Fang et al., arXiv:2508.07407, 2025) — four-component framework (Inputs/Agent/Environment/Optimisers); maintained list at *github.com/EvoAgentX/Awesome-Self-Evolving-Agents*.
- **Agentic Context Engineering (ACE)** (Lin et al., arXiv:2510.04618, 2025; ICLR 2026) — evolving-playbook approach with Generator/Reflector/Curator roles; +10.6% on agent tasks, +8.6% on finance per abstract.
- **Continual Learning, Not Training (ATLAS)** (arXiv:2511.01093, 2025) — dual-agent orchestration as gradient-free continual learning; reframes CL as system design rather than retraining.
- **Live-SWE-agent** (Xia et al., arXiv:2511.13646, 2025) — scaffold self-evolution at runtime; the paper's headline is 77.4% on SWE-bench Verified without test-time scaling per abstract.
- **Trajectory-Informed Memory Generation** (arXiv:2603.10600, 2026) — extracts actionable learnings from trajectories; reports up to 14.3pp gain on AppWorld per abstract.
- **Agent Skills for LLMs** (arXiv:2602.12430, Feb 2026) — four-axis survey of the skills ecosystem; empirical finding: 26.1% of community-contributed skills contain vulnerabilities (per abstract).
- **Hyperagents / DGM-H** (FAIR, arXiv:2603.19461, Mar 2026) — unifies task-agent and meta-agent in one editable program; the meta-procedure becomes editable. See [45-hyperagents-self-modification.md](45-hyperagents-self-modification.md).
- **Experiential Reflective Learning (ERL)** (arXiv:2603.24639, Mar 2026) — heuristic extraction + selective retrieval; +7.8% over ReAct on Gaia2 per abstract.
- **Adaptation of Agentic AI survey** (arXiv:2512.16301, Mar 2026) — the four-paradigm taxonomy in [47](47-adaptation-of-agentic-ai-survey.md).
- **Autogenesis** (NTU, arXiv:2604.15034, Apr 2026) — RSPL + SEPL protocol layers for resource-typed self-evolution with rollback. See [36-autogenesis-self-evolving-agents.md](36-autogenesis-self-evolving-agents.md).
- **Hermes Agent** (Nous Research, Apr 2026) — production-leaning single-agent harness that turns completed workflows into SKILL.md procedures. See [55-hermes-agent-self-improving.md](55-hermes-agent-self-improving.md).
- **Frontier-Eng** (arXiv:2604.12290, Apr 2026) — 47-task engineering benchmark with continuous reward and hard feasibility constraints; documents a dual power-law decay in improvement frequency (~1/iteration) and magnitude per abstract.
- **SAGE** (arXiv:2512.17102, 2026) — GRPO + skill library; +8.9% Scenario Goal Completion, 26% fewer steps, 59% fewer tokens on AppWorld per abstract.

## What changed in 2026

A few shifts are clear-cut:

1. **The field got its first surveys.** Before mid-2025, "self-evolving agents" was a tag that meant different things to different authors. The Gao survey (arXiv:2507.21046) and the Fang survey (arXiv:2508.07407) now anchor a shared vocabulary, and the Adaptation-of-Agentic-AI survey ([47](47-adaptation-of-agentic-ai-survey.md)) makes adjacent territory tractable.

2. **Runtime evolution overtook offline evolution for coding agents.** DGM was the 2025 high-water mark and depended on costly offline evolutionary training. Live-SWE-agent (arXiv:2511.13646) flipped the script: modify your own scaffold *during* the episode, starting from a minimal shell-only agent. Its abstract reports 77.4% on SWE-bench Verified without test-time scaling — i.e. runtime evolution matched or beat an offline-evolved scaffold.

3. **"Context" became first-class.** ACE (arXiv:2510.04618) treats the system prompt, playbook, and memory as evolving artifacts governed by Generator/Reflector/Curator; this is in direct methodological contrast to Reflexion, which treated memory as a list of lessons. ACE explicitly tries to defeat *context collapse* (iterative rewriting eroding detail) and *brevity bias*.

4. **The meta-level got editable.** Hyperagents ([45](45-hyperagents-self-modification.md)) is the clearest statement: not only can the task agent improve, but so can the procedure that improves it. This is still research-grade; the practical implication for harness designers is humility about which level gets rewritten and which doesn't.

5. **Protocols arrived.** Autogenesis ([36](36-autogenesis-self-evolving-agents.md)) is the first production-shaped framing: SEA as versioned resources with rollback, not as a monolithic training loop. Hermes is the product-shaped parallel at the personal-agent scale.

6. **New benchmarks that actually stress SEA.** SWE-bench Verified and Pro are still useful but saturate on static agents; Frontier-Eng (arXiv:2604.12290) specifically measures the shape of the improvement curve under a fixed interaction budget, surfacing the dual-power-law decay. SWE-Skills-Bench (arXiv:2603.15401) isolates the marginal utility of agent skills on real-world SWE tasks. LiveCodeBench keeps the contamination window small.

7. **New failure modes are now documented, not hypothetical.** "Reward Hacking as Equilibrium under Finite Evaluation" (arXiv:2603.28063) formalises the observation that evaluation coverage drops to zero as tool count grows — an explicit argument that *SEA without an evaluation investment proportional to capability is a design error*. "Monitoring Emergent Reward Hacking During Generation via Internal Activations" (arXiv:2603.04069) shows reward-hacking is often undetectable from outputs alone. 26.1% of community-contributed agent skills carrying vulnerabilities (arXiv:2602.12430) is the supply-chain instance of the same problem.

## Open problems (2026-current)

- **Catastrophic forgetting, re-framed.** ATLAS (arXiv:2511.01093) argues that continual learning is about orchestration, not gradient updates. *Landscape interpretation:* this dissolves the classical problem in one direction but does not eliminate it — skill libraries and playbooks can still rot, drift, or get overwritten. The failure modes in [19](19-voyager-skill-libraries.md) and [55](55-hermes-agent-self-improving.md) are the new face of the old problem.
- **Reward hacking scales faster than evaluation.** The equilibrium paper (arXiv:2603.28063) is blunt: evaluation costs grow linearly, gameable dimensions grow combinatorially. No current SEA system claims a principled answer.
- **Cost of the evaluation loop.** Frontier-Eng (arXiv:2604.12290) documents a dual power-law decay — for each problem, improvement frequency and magnitude both fall as ~1/iteration. The practical consequence is that self-evolution loops are expensive *per marginal improvement*. This is an important corrective to the framing in which SEA is "free" because it runs after deployment.
- **Cross-task / cross-domain skill transfer.** Voyager and Hermes accumulate skills for a single domain; there is no 2026-current paper I found that convincingly demonstrates learned skills from one domain transferring to a materially different one. AlphaEvolve transfers *algorithms* but not *agents*.
- **Safety of self-modification at the meta-level.** Hyperagents foregrounds this explicitly; the suggested posture (containment + human review on meta changes) is a stance, not a solution. The "Your Agent, Their Asset" OpenClaw analysis (arXiv:2604.04759) shows the supply-chain side: a self-evolving agent in an ecosystem with untrusted skills is an attack surface that traditional red-teaming does not cover. Cross-link [26-linuxarena-production-agent-safety.md](26-linuxarena-production-agent-safety.md) and [49-agents-of-chaos-red-teaming.md](49-agents-of-chaos-red-teaming.md).
- **Evaluation-proof SEA.** *Landscape interpretation:* no system I am aware of has solved the "who watches the watcher" problem where the SEA's own evaluator is subject to the same improvement loop. Hyperagents' answer — human review at the meta-level — is realistic but limits scalability.

## Design-pattern catalogue

Six patterns recur across the 2026 literature. Each is either explicit in a paper or a clean distillation across several.

### 1. Closed Learning Loop (propose / assess / commit / rollback)

Canonicalised by Autogenesis's SEPL ([36](36-autogenesis-self-evolving-agents.md)) but implicit in Reflexion, Voyager, DGM, Hermes. The invariant: every change passes through an evaluator before it becomes the default, and every change is revertable. When this pattern is absent, "self-evolution" is indistinguishable from "self-drift".

### 2. Skill Library with Verifier

Introduced by Voyager ([19](19-voyager-skill-libraries.md)), productised by Hermes ([55](55-hermes-agent-self-improving.md)), RL-ified by SAGE. The library accumulates *verified* procedures and is retrieved by the agent on new tasks. The pattern fails the moment the verifier fails: skill poisoning is the dominant failure mode. Cross-link [04-skills.md](04-skills.md).

### 3. Evolving Playbook (ACE-style)

From Agentic Context Engineering (arXiv:2510.04618). The system prompt / reasoning playbook is not a static document but a structured, incrementally-updated object, edited by a Generator / Reflector / Curator triad. Distinguished from Reflexion by its *structure* — ACE specifically resists context collapse and brevity bias, which a flat list of reflections does not.

### 4. Evolutionary Archive

DGM, AlphaEvolve, Digital Red Queen, Imbue Darwinian Evolver. A population of variants is maintained; mutation is an LLM call; selection is a score. This pattern is the only one with a credible story for *open-ended* improvement — the archive preserves dead-end-looking variants that turn out to enable a later jump. The costs are: compute, evaluator load, and the safety overhead of running untrusted code.

### 5. Runtime Scaffold Self-Modification

Live-SWE-agent (arXiv:2511.13646), partially Hyperagents ([45](45-hyperagents-self-modification.md)). Unlike the archive pattern, the agent modifies *itself in place* during an episode. Requires strict sandboxing ([06-permission-modes.md](06-permission-modes.md)) and a way to undo a change if the new scaffold is worse than the old one. The theoretical appeal is zero offline-training cost; the practical risk is scaffold trashing mid-episode.

### 6. Resource Protocol Layer (Autogenesis)

RSPL + SEPL ([36](36-autogenesis-self-evolving-agents.md)). Instead of a single loop over "the agent", every component is its own versioned resource with a URI, a schema, and a lifecycle. This is the pattern most aligned with software-engineering discipline and is probably the right default for production teams who want to ship self-improvement without shipping chaos. Cross-link [40-harness-engineering-principles.md](40-harness-engineering-principles.md) and [44-four-pillars-harness-engineering.md](44-four-pillars-harness-engineering.md).

### 7. Metacognitive Self-Modification

Hyperagents / DGM-H ([45](45-hyperagents-self-modification.md)). The *procedure that proposes improvements* is itself subject to improvement. Very powerful in principle, very research-grade in practice. For harness designers, the take-away is usually homeopathic: have a place to record which part of your self-evolution loop *is itself being improved* and guard it with stricter evaluation than the task-level.

### 8. Verifier-Only Self-Play

Absolute Zero Reasoner (arXiv:2505.03335). A code executor is the only external signal; the agent both proposes tasks and solves them. The cleanest Axis-3 stance ("verifier, nothing else"). Limited to domains with a free, fast, ground-truth verifier (code, math); opens the question of what the analogue looks like for, say, legal reasoning.

## Where this leaves the practitioner (landscape interpretation)

Two stances are defensible in April 2026 and most of the corpus files implicitly pick one:

- **Research-frontier stance.** If you're trying to push capability, Hyperagents / DGM / AlphaEvolve-style evolutionary self-modification is where the frontier is. Most teams should read, not build.
- **Production stance.** Pick one artifact (skills, playbook, prompt), wrap it in a Closed Learning Loop with versioning and rollback (Autogenesis-shape), invest more than you think in evaluation, and do not try to edit the meta-level until the object-level is stable. Hermes and ACE are proof this works at production polish.

This is the same dichotomy as the Hyperagents file's "research use / not-production" split ([45](45-hyperagents-self-modification.md)), generalised across the SEA field.

## References

All URLs verified as of 2026-04-20.

- Shinn et al., "Reflexion", arXiv:2303.11366 — <https://arxiv.org/abs/2303.11366>
- Wang et al., "Voyager", arXiv:2305.16291 — <https://arxiv.org/abs/2305.16291>
- Yuksekgonul et al., "TextGrad", arXiv:2406.07496 — <https://arxiv.org/abs/2406.07496>
- Zhao et al., "Absolute Zero: Reinforced Self-play Reasoning with Zero Data", arXiv:2505.03335 — <https://arxiv.org/abs/2505.03335>
- Zhang et al., "Darwin Gödel Machine", arXiv:2505.22954 — <https://arxiv.org/abs/2505.22954>
- Novikov et al., "AlphaEvolve", arXiv:2506.13131 — <https://arxiv.org/abs/2506.13131>
- Gao et al., "A Survey of Self-Evolving Agents", arXiv:2507.21046 — <https://arxiv.org/abs/2507.21046>
- Fang et al., "A Comprehensive Survey of Self-Evolving AI Agents", arXiv:2508.07407 — <https://arxiv.org/abs/2508.07407>
- Lin et al., "Agentic Context Engineering (ACE)", arXiv:2510.04618 — <https://arxiv.org/abs/2510.04618>
- "Continual Learning, Not Training: Online Adaptation For Agents" (ATLAS), arXiv:2511.01093 — <https://arxiv.org/abs/2511.01093>
- Xia et al., "Live-SWE-agent", arXiv:2511.13646 — <https://arxiv.org/abs/2511.13646>
- "Adaptation of Agentic AI: A Survey of Post-Training, Memory, and Skills", arXiv:2512.16301 — <https://arxiv.org/abs/2512.16301>
- "SAGE: Reinforcement Learning for Self-Improving Agent with Skill Library", arXiv:2512.17102 — <https://arxiv.org/abs/2512.17102>
- "Agent Skills for Large Language Models", arXiv:2602.12430 — <https://arxiv.org/abs/2602.12430>
- "Trajectory-Informed Memory Generation for Self-Improving Agent Systems", arXiv:2603.10600 — <https://arxiv.org/abs/2603.10600>
- "SWE-Skills-Bench", arXiv:2603.15401 — <https://arxiv.org/abs/2603.15401>
- "Hyperagents" (FAIR), arXiv:2603.19461 — <https://arxiv.org/abs/2603.19461>
- "Experiential Reflective Learning for Self-Improving LLM Agents", arXiv:2603.24639 — <https://arxiv.org/abs/2603.24639>
- "Reward Hacking as Equilibrium under Finite Evaluation", arXiv:2603.28063 — <https://arxiv.org/abs/2603.28063>
- "Monitoring Emergent Reward Hacking During Generation via Internal Activations", arXiv:2603.04069 — <https://arxiv.org/abs/2603.04069>
- "Your Agent, Their Asset: A Real-World Safety Analysis of OpenClaw", arXiv:2604.04759 — <https://arxiv.org/abs/2604.04759>
- "SkillX: Automatically Constructing Skill Knowledge Bases for Agents", arXiv:2604.04804 — <https://arxiv.org/abs/2604.04804
- "Frontier-Eng", arXiv:2604.12290 — <https://arxiv.org/abs/2604.12290>
- "Autogenesis: A Self-Evolving Agent Protocol", arXiv:2604.15034 — <https://arxiv.org/abs/2604.15034>
- Hermes Agent (Nous Research) — <https://github.com/NousResearch/hermes-agent> and <https://hermes-agent.nousresearch.com/>
- EvoAgentX / Awesome-Self-Evolving-Agents list — <https://github.com/EvoAgentX/Awesome-Self-Evolving-Agents>
- AlphaEvolve blog — <https://deepmind.google/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/>
- Absolute Zero Reasoner project site — <https://andrewzh112.github.io/absolute-zero-reasoner/>
- Darwin Gödel Machine (Sakana) — <https://sakana.ai/dgm/>

### Related files in this corpus

[04-skills.md](04-skills.md) · [06-permission-modes.md](06-permission-modes.md) · [09-memory-files.md](09-memory-files.md) · [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md) · [14-reflexion.md](14-reflexion.md) · [18-chain-of-verification-self-refine.md](18-chain-of-verification-self-refine.md) · [19-voyager-skill-libraries.md](19-voyager-skill-libraries.md) · [21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md) · [26-linuxarena-production-agent-safety.md](26-linuxarena-production-agent-safety.md) · [36-autogenesis-self-evolving-agents.md](36-autogenesis-self-evolving-agents.md) · [40-harness-engineering-principles.md](40-harness-engineering-principles.md) · [44-four-pillars-harness-engineering.md](44-four-pillars-harness-engineering.md) · [45-hyperagents-self-modification.md](45-hyperagents-self-modification.md) · [47-adaptation-of-agentic-ai-survey.md](47-adaptation-of-agentic-ai-survey.md) · [49-agents-of-chaos-red-teaming.md](49-agents-of-chaos-red-teaming.md) · [55-hermes-agent-self-improving.md](55-hermes-agent-self-improving.md)
