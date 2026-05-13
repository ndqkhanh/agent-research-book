# 314 — OpenReview Memory Paper Atlas (2025–2026)

**Scope.** One compact deep-dive card per user-supplied OpenReview memory paper. Read this with the master synthesis in [313](313-memory-research-2026-master-synthesis.md).

**Correction note.** `https://openreview.net/pdf?id=jHc8dCx6DDr` returns empty content through the PDF endpoint, but the corresponding forum page is valid: [Memory Gym: Partially Observable Challenges to Memory-Based Agents](https://openreview.net/forum?id=jHc8dCx6DDr).

---

## 1. MemoryAgentBench — Evaluating Memory in LLM Agents via Incremental Multi-Turn Interactions

**Source:** [OpenReview](https://openreview.net/forum?id=DT7JyQC3MR), [PDF](https://openreview.net/pdf?id=DT7JyQC3MR), [GitHub search result](https://github.com/HUST-AI-HYZ/MemoryAgentBench)

**Problem.** Existing agent benchmarks mostly evaluate reasoning, planning, and execution, while memory quality is under-evaluated.

**Mechanism.** The paper converts long-context and newly built datasets into incremental multi-turn settings. It defines four competencies:

1. Accurate retrieval.
2. Test-time learning.
3. Long-range understanding.
4. Selective forgetting / conflict resolution.

**Key findings.**

- Current memory agents fail to master all four competencies.
- Smaller chunks can improve retrieval but damage long-range integration.
- Stronger backbones help agentic memory more than simple RAG once the base model is already strong.

**Why it matters.** This should become the default benchmark suite for any new memory paper.

**New-paper angle.** Add adversarial memory poisoning and privacy deletion tests to MemoryAgentBench.

---

## 2. A-Mem — Agentic Memory for LLM Agents

**Source:** [OpenReview](https://openreview.net/forum?id=FiM0M8gcct), [PDF](https://openreview.net/pdf?id=FiM0M8gcct), [AgenticMemory](https://github.com/WujiangXu/AgenticMemory), [A-mem-sys](https://github.com/WujiangXu/A-mem-sys)

**Problem.** Fixed memory schemas and fixed workflow insertion points make existing memory systems brittle in new tasks.

**Mechanism.** A-Mem uses Zettelkasten-style atomic notes. Each note contains raw content, timestamp, keywords, tags, contextual description, embedding, and links. New memories trigger:

- note construction,
- link generation,
- memory evolution,
- retrieval over the evolving graph.

**Strength.** Memory organization becomes adaptive rather than predetermined.

**Weakness.** The LLM is trusted to build and evolve the topology, so provenance and verifier gating are necessary for production.

**New-paper angle.** Temporal/verifier-gated A-Mem: every note evolution must cite evidence and pass contradiction checks.

---

## 3. LightMem — Lightweight and Efficient Memory-Augmented Generation

**Source:** [OpenReview](https://openreview.net/forum?id=dyJ0GWpjJB), [PDF](https://openreview.net/pdf?id=dyJ0GWpjJB), [zjunlp/LightMem](https://github.com/zjunlp/LightMem)

**Problem.** Many memory systems improve quality but add too much latency, token use, and API-call cost.

**Mechanism.** Three-stage cognitive memory:

1. **Sensory memory** filters and compresses quickly.
2. **Short-term memory** consolidates topic groups.
3. **Long-term memory** performs offline sleep-time update.

**Reported result.** OpenReview reports up to 10.9% accuracy gain, 117× token reduction, 159× API-call reduction, and 12× runtime reduction.

**Strength.** Turns memory efficiency into a first-class research objective.

**Weakness.** Less focused on deep graph/experience abstraction than A-Mem, ReasoningBank, or SMITH.

**New-paper angle.** Cost-aware memory routing: learn when to use no memory, short-term summary, graph memory, or expensive reasoning memory.

---

## 4. SMITH — Shared Memory Integrated Tool Hub

**Source:** [OpenReview](https://openreview.net/forum?id=JnwClln80Q)

**Problem.** Agents either rely on predefined tools or create tools from scratch without reusing prior task experience.

**Mechanism.** SMITH organizes memory into:

- procedural memory,
- semantic memory,
- episodic memory.

It integrates dynamic tool creation through sandboxed code generation and cross-task experience sharing through episodic retrieval.

**Reported result.** 81.8% Pass@1 on GAIA, ahead of Alita 75.2% and Memento 70.9%.

**Strength.** Connects memory with capability expansion.

**Weakness.** Complex architecture makes it hard to isolate which memory component drives the win.

**New-paper angle.** "Memory-to-tool promotion": when should an episode become executable code?

---

## 5. CoPS — Provable Cross-Task Experience Sharing

**Source:** [OpenReview](https://openreview.net/forum?id=9W6Z9IeLzc), [arXiv](https://arxiv.org/abs/2410.16670), [uclaml/COPS](https://github.com/uclaml/cops)

**Problem.** Experience-assisted reasoning often retrieves prior experiences without principled distribution-shift control.

**Mechanism.** CoPS selects distribution-matched experiences using a pessimism-based strategy.

**Benchmarks.** AlfWorld, WebShop, HotPotQA.

**Strength.** Adds theory to experience retrieval and directly targets negative transfer.

**Weakness.** It is an experience-selection algorithm, not a full memory lifecycle system.

**New-paper angle.** Add CoPS-style conservative retrieval to A-Mem or ReasoningBank.

---

## 6. ReMemR1 — Look Back to Reason Forward

**Source:** [OpenReview](https://openreview.net/forum?id=1cymflI2Lh), [arXiv](https://arxiv.org/html/2509.23040v1), [syr-cn/ReMemR1](https://github.com/syr-cn/ReMemR1)

**Problem.** Forward-only "memorize while reading" loses early evidence and cannot revisit prior memory non-linearly.

**Mechanism.**

- Callback-enhanced memory allows selective retrieval from the full memory history.
- Reinforcement Learning with Multi-Level Rewards combines final-answer and step-level memory-use rewards.

**Strength.** Trains memory navigation, not just memory storage.

**Weakness.** Long-document QA setting may not capture tool-use side effects and real multi-session agent memory.

**New-paper angle.** Callback memory for tool-using agents: train when to revisit tool outputs, errors, and plans.

---

## 7. MEMENTO — Embodied Agents Meet Personalization

**Source:** [OpenReview](https://openreview.net/forum?id=E5L43l5EIu), [arXiv](https://arxiv.org/html/2505.16348v2), [Connoriginal/MEMENTO](https://github.com/Connoriginal/MEMENTO)

**Problem.** Embodied agents need personalized memory, not just generic object rearrangement skill.

**Mechanism.** The benchmark tests object semantics and user patterns through single-memory and joint-memory tasks. The proposed module is a hierarchical user-profile KG.

**Key bottlenecks.**

- Information overload.
- Coordination failure across multiple memories.

**Strength.** Moves memory evaluation into embodied personalization.

**Weakness.** High environment/dependency cost.

**New-paper angle.** Multi-memory composition benchmark for general agents.

---

## 8. HexMachina — Self-Evolving Multi-Agent System for Catan

**Source:** [OpenReview](https://openreview.net/forum?id=V0Fb4pwhS4)

**Status.** Withdrawn ICLR 2026 submission.

**Problem.** Prompt-centric agents fail to maintain coherent long-horizon strategy in adversarial stochastic environments.

**Mechanism.** Separate environment discovery from strategy improvement; preserve executable artifacts so the LLM becomes a high-level strategy designer rather than a per-turn decider.

**Reported result.** The abstract reports 54% win rate against AlphaBeta in best runs.

**Strength.** Demonstrates artifact-centric memory for strategy learning.

**Weakness.** Narrow domain and withdrawn status.

**New-paper angle.** Generalize artifact memory from games to coding, science, or ops workflows.

---

## 9. EgoMem — Lifelong Memory Agent for Full-Duplex Omnimodal Models

**Source:** [OpenReview](https://openreview.net/forum?id=9QYA3DiPl8), [arXiv](https://arxiv.org/html/2509.11914v2)

**Problem.** Real-time omnimodal agents need to identify users, retrieve preferences, and update memory from raw AV streams.

**Mechanism.** Three asynchronous processes:

1. Retrieval: identify user via face/voice and gather context.
2. Dialog: generate personalized audio response.
3. Memory management: detect dialog boundaries and update long-term memory.

**Reported result.** >95% module accuracy for retrieval/memory management and >87% fact consistency.

**Strength.** Extends memory from text to real-time embodied multimodal interaction.

**Weakness.** Raises large privacy, identity, and deletion-governance questions.

**New-paper angle.** Privacy-preserving multimodal memory with cryptographic consent and revocation.

---

## 10. MEM1 — Learning to Synergize Memory and Reasoning

**Source:** [OpenReview](https://openreview.net/forum?id=jJ6F1sDn9i), [arXiv](https://arxiv.org/html/2506.15841), [MIT-MI/MEM1](https://github.com/MIT-MI/MEM1)

**Problem.** Full-context prompting grows unbounded and degrades out-of-distribution long-horizon performance.

**Mechanism.** Train an agent to maintain a compact shared internal state that jointly supports reasoning and consolidation.

**Reported result.** 3.5× performance improvement and 3.7× memory reduction in a reported 16-objective multi-hop QA task.

**Strength.** Shows learned constant memory can beat larger stateless baselines.

**Weakness.** Compact state is less inspectable than explicit memory.

**New-paper angle.** Interpretable MEM1: distill compact state into auditable textual/graph anchors.

---

## 11. GraphMind — Dynamic Knowledge Builders for Sequential Decision-Making

**Source:** [OpenReview](https://openreview.net/forum?id=XromAiEaE3)

**Status.** Withdrawn ICLR 2026 submission.

**Problem.** Partially observable environments require long-term memory and exploration under incomplete information.

**Mechanism.** LLM agent incrementally constructs a knowledge graph from interactions and retrieves graph information for high-level planning.

**Strength.** KG memory is natural for POMDP-like navigation.

**Weakness.** Public information is limited due to withdrawn status.

**New-paper angle.** Add causal edges and temporal validity to planning KGs.

---

## 12. ChemAgent — Self-Updating Memories Improve Chemical Reasoning

**Source:** [OpenReview](https://openreview.net/forum?id=kuhIqeVg0e), [arXiv](https://arxiv.org/abs/2501.06590), [gersteinlab/ChemAgent](https://github.com/gersteinlab/chemagent)

**Problem.** Chemistry reasoning requires precise calculations and domain-specific procedures.

**Mechanism.** Decompose tasks into subtasks and compile them into three memory types:

- plan memory,
- execution memory,
- knowledge memory.

**Reported result.** Up to 46% GPT-4 gain on four SciBench chemical reasoning datasets.

**Strength.** Excellent example of domain-shaped procedural memory.

**Weakness.** Domain transfer remains open.

**New-paper angle.** Procedural memory benchmark across chemistry, math, biology, and law.

---

## 13. Intrinsic Memory Agents

**Source:** [OpenReview](https://openreview.net/forum?id=UbSUxAK3BI), [PDF](https://openreview.net/pdf?id=UbSUxAK3BI)

**Problem.** Multi-agent systems lose memory consistency, role adherence, and procedural integrity under context limits.

**Mechanism.** Each agent keeps role-aligned memory that evolves intrinsically with its outputs, using a generic memory template.

**Benchmarks.** PDDL, FEVER, ALFWorld, plus data-pipeline design.

**Strength.** Strong direction for heterogeneous multi-agent systems.

**Weakness.** Role-aligned memory can harden bias or silo facts unless there is shared consensus memory.

**New-paper angle.** Dual memory for teams: private role memory + shared verified factual memory.

---

## 14. Memory Gym — Partially Observable Challenges to Memory-Based Agents

**Source:** [OpenReview forum](https://openreview.net/forum?id=jHc8dCx6DDr), [PDF endpoint](https://openreview.net/pdf?id=jHc8dCx6DDr)

**Status.** Valid OpenReview forum page; the PDF endpoint returned empty content during lookup.

**Problem.** Many reinforcement-learning agents claim memory capability but are tested in environments where memory is not the bottleneck.

**Mechanism.** Memory Gym contributes partially observable 2D/discrete-control environments that are designed to be unsolvable by memory-less agents:

- Mortar Mayhem,
- Mystery Path,
- Searing Spotlights.

**Key finding.** PPO + GRU experiments underline strong memory dependency, and Searing Spotlights remains especially challenging because moving spotlights reveal ground truth while also perturbing recurrent-memory agents.

**Why it matters for LLM-agent memory.** It is not an LLM-memory paper, but it is useful as a benchmark-design ancestor: memory should be tested under partial observability, noise, long sequences, and repeated memory interactions, not just static recall.

**New-paper angle.** Port Memory Gym principles into LLM tool-agent environments: partial observability, distractor observations, delayed recall, and action consequences that require retained state.

---

## Cross-paper ranking by research leverage

| Rank | Paper | Why it is high leverage |
|---|---|---|
| 1 | MemoryAgentBench | Defines what to measure |
| 2 | A-Mem | Strong explicit memory architecture |
| 3 | LightMem | Best cost-performance framing |
| 4 | MEM1 | Strong learned-control direction |
| 5 | ReasoningBank | Strong experience-abstraction direction |
| 6 | ReMemR1 | Strong long-context revisit direction |
| 7 | CoPS | Theoretical negative-transfer control |
| 8 | SMITH | Links memory to dynamic capability creation |
| 9 | EgoMem | Multimodal/embodied frontier |
| 10 | MEMENTO embodied | Personalized embodied benchmark |

---

## Sources

- [MemoryAgentBench — OpenReview](https://openreview.net/forum?id=DT7JyQC3MR)
- [A-Mem — OpenReview](https://openreview.net/forum?id=FiM0M8gcct)
- [LightMem — OpenReview](https://openreview.net/forum?id=dyJ0GWpjJB)
- [SMITH — OpenReview](https://openreview.net/forum?id=JnwClln80Q)
- [CoPS — OpenReview](https://openreview.net/forum?id=9W6Z9IeLzc)
- [ReMemR1 — OpenReview](https://openreview.net/forum?id=1cymflI2Lh)
- [MEMENTO — OpenReview](https://openreview.net/forum?id=E5L43l5EIu)
- [HexMachina — OpenReview](https://openreview.net/forum?id=V0Fb4pwhS4)
- [EgoMem — OpenReview](https://openreview.net/forum?id=9QYA3DiPl8)
- [MEM1 — OpenReview](https://openreview.net/forum?id=jJ6F1sDn9i)
- [GraphMind — OpenReview](https://openreview.net/forum?id=XromAiEaE3)
- [ChemAgent — OpenReview](https://openreview.net/forum?id=kuhIqeVg0e)
- [Intrinsic Memory Agents — OpenReview](https://openreview.net/forum?id=UbSUxAK3BI)
- [Memory Gym — OpenReview](https://openreview.net/forum?id=jHc8dCx6DDr)
