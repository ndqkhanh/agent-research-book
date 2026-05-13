# 99 — Lyra Paper-Set Synthesis: What 22 Frontier Papers Tell Us About Agent Engineering in 2026

**Context.** This synthesis ties together the 22 paper-grounded deep-dives in files [77](77-meta-tts-agentic-coding.md) through [98](98-diversity-collapse-mas.md). The papers were selected by the Lyra project team as the primary literature backing the project's design choices. Considered together, they capture the **technical state of agent and harness engineering as of April 2026** — what works, what doesn't, where the open frontiers are, and how the pieces fit. Read this file to orient before drilling into individual papers; or read it after the individual files to consolidate.

## The 22 papers at a glance

| # | Paper | Org | Year | Wave | One-line claim |
|---|---|---|---|---|---|
| [77](77-meta-tts-agentic-coding.md) | Meta TTS for Agentic Coding | Meta SI Labs et al. | 2026 | 1 | Compact rollout summaries + RTV + PDR = +7 to +12 pp on SWE/Terminal-Bench |
| [78](78-ngc-neural-garbage-collection.md) | Neural Garbage Collection | Stanford | 2026 | 1 | RL-train the LM to forget while reasoning; 2–4× KV cache compression at parity |
| [79](79-skill-rag.md) | Skill-RAG | UMich/UBC/Rutgers | 2026 | 1 | Hidden-state prober + 4-skill router (rewrite/decompose/focus/exit) for adaptive RAG |
| [80](80-knowrl.md) | KnowRL | Zhejiang Univ. | 2025 | 1 | Knowledge-aware RL reward for reasoning models; cuts hallucination at training time |
| [81](81-reasoningbank.md) | ReasoningBank + MaTTS | Google Research | 2025 | 1 | Memory of reasoning patterns + memory-aware test-time scaling = self-evolution without weight updates |
| [82](82-poisonedrag.md) | PoisonedRAG | USENIX Security | 2024/25 | 1 | 5 crafted documents in a million-doc corpus flip RAG answers with ~97% success |
| [83](83-semaclaw-deep.md) | SemaClaw (paper-grounded) | Midea AIRC | 2026 | 1 | Generic personal-AI agent built around an event-facade harness (`sema-code-core`) |
| [84](84-swe-search-mcts.md) | SWE-Search MCTS | Antoniades et al. (ICLR 2025) | 2024/25 | 2 | Intra-attempt MCTS over agent action trees; +23% mean relative on SWE-Bench Lite |
| [85](85-alphaevolve.md) | AlphaEvolve | Google DeepMind | 2025 | 2 | Evolutionary coding agent; 4×4 complex matmul in 48 mults (vs 49); real datacenter wins |
| [86](86-frugalgpt.md) | FrugalGPT | Stanford | 2023 | 2 | Cascade routing + completion cache cuts cost up to 98% at quality parity |
| [87](87-routellm.md) | RouteLLM | UC Berkeley / LMSYS | 2024 | 2 | Preference-data router from Chatbot Arena; 75% cost cut vs random at PGR 50% |
| [88](88-confidence-driven-router.md) | Confidence-Driven Router | (RouteLLM follow-up) | 2025 | 2 | Calibration-based escalation; cheaper than preference data, similar quality |
| [89](89-voyager-deep.md) | Voyager (paper-grounded) | NVIDIA / Caltech / UT-Austin (TMLR 2024) | 2023/24 | 2 | Auto-curriculum + iterative code prompting + skill library; 3.3× tech-tree milestones |
| [90](90-reflexion-deep.md) | Reflexion (paper-grounded) | Northeastern / MIT (NeurIPS 2023) | 2023 | 2 | Verbal RL: reflections in episodic memory; HumanEval 91% pass@1 with GPT-4 |
| [91](91-metagpt-deep.md) | MetaGPT (paper-grounded) | Hong et al. (ICLR 2024) | 2023/24 | 2 | SDLC SOPs encoded as multi-agent message routing + structured artifacts |
| [92](92-chatdev.md) | ChatDev | Tsinghua / OpenBMB | 2023/24 | 2 | Chat-chain SDLC: design → code → test → doc; ~$1–2 per software, ~2.5 min |
| [93](93-dspy.md) | DSPy | Stanford (ICLR 2024) | 2023/24 | 2 | Declarative LM pipelines: signatures + modules + teleprompters that compile prompts |
| [94](94-eagle3-spec-decoding.md) | EAGLE-3 | Li et al. | 2025 | 2 | Speculative decoding via training-time test; ~6× single-stream speedup on Llama 3.3 70B |
| [95](95-osworld.md) | OSWorld | Xie et al. (NeurIPS 2024) | 2024 | 2 | 369 real-OS tasks; best agent 12% vs human 72%; the canonical computer-use benchmark |
| [96](96-gdpval.md) | GDPval | OpenAI | 2025 | 2 | 1,320 tasks across 44 occupations × 9 GDP-contributing sectors; pairwise human-vs-AI |
| [97](97-qwen-prm.md) | Qwen2.5-Math PRM Lessons | Qwen team | 2025 | 2 | LLM-as-judge step labels actively harm PRMs; consensus filter MC + LLM is the recipe |
| [98](98-diversity-collapse-mas.md) | Diversity Collapse in MAS | NUS / CUHK-SZ (ACL 2026 Findings) | 2026 | 3 | Multi-agent ≠ multi-perspective; structural coupling collapses diversity at three levels |

The 22 papers span ~2.5 years (2023–2026) and four scales of contribution: **breakthrough algorithms** (Meta TTS, NGC, AlphaEvolve, Diversity Collapse), **canonical references** (Reflexion, Voyager, MetaGPT, ChatDev, DSPy, FrugalGPT), **benchmarks** (OSWorld, GDPval), and **production lessons** (Qwen PRM, RouteLLM, EAGLE-3). They cover a wide enough surface that — if you internalize all 22 — you have read most of what defines the agent-engineering frontier as of April 2026.

## Ten cross-cutting themes

### Theme 1 — Test-time scaling is the new training-time scaling

Five papers explicitly attack the test-time scaling (TTS) problem from different angles:
- **Meta TTS ([77](77-meta-tts-agentic-coding.md))** — RTV + PDR over rollout summaries; +7 to +12 pp on SWE/Terminal-Bench.
- **SWE-Search ([84](84-swe-search-mcts.md))** — MCTS over action trees within a single attempt; +23% mean relative.
- **AlphaEvolve ([85](85-alphaevolve.md))** — population-based evolutionary search over programs; cracks 56-year-old algorithmic frontiers.
- **Reflexion ([90](90-reflexion-deep.md)) and ReasoningBank ([81](81-reasoningbank.md))** — sequential refinement via reflection; Reflexion is single-agent, ReasoningBank is memory-extended.

The pattern: **representation, selection, reuse**. Meta TTS makes this thesis explicit. Each TTS paper boils down to (a) a representation of prior compute (summary, tree node, program, reflection), (b) a selection mechanism over those representations (RTV, UCT, evolutionary fitness, evaluator), and (c) a way to reuse selected items in the next round (PDR refinement context, MCTS expansion, mutation prompt, reflection memory).

The implication for harness engineering: **TTS is now a harness-level lever, not a model-level lever.** A harness that exposes a "thinking harder" knob can spend 2–10× compute for 5–30 pp of quality lift on hard tasks, regardless of which frontier model is loaded. This is closer to the impact of a base-model generation jump than to a typical prompt tweak.

### Theme 2 — Memory is now a *trainable* substrate, not just a file

Three sharply different memory architectures appear in the corpus:

| Paper | Memory location | Memory unit | How it's updated |
|---|---|---|---|
| **NGC ([78](78-ngc-neural-garbage-collection.md))** | Inside the model's KV cache | Token activations | RL during training (eviction policy is itself a learned action) |
| **ReasoningBank ([81](81-reasoningbank.md))** | External database | Reasoning patterns / lessons | Append on episode end, retrieve by embedding |
| **Reflexion ([90](90-reflexion-deep.md))** | In-context (episodic) | Verbal reflections | Append after each episode, prune to last k |

NGC is the most radical: the model **learns to evict its own KV cache** as a discrete action, jointly with the reasoning process, from outcome-only RL reward. This collapses the separation between "what to remember" (a heuristic written by an engineer) and "what to compute" (a learned function). The implication is that future LMs will not need external context-compaction harnesses — they'll do it natively.

ReasoningBank and Reflexion remain external because (a) most teams cannot retrain frontier models, and (b) externalization makes memory inspectable, shareable across users/sessions, and persistable across model versions. The three approaches are not competitors — they are layered (in-cache for transient, in-context for episode, external for persistent). See also [09-memory-files](09-memory-files.md), [72-claude-mem-persistent-memory-compression](72-claude-mem-persistent-memory-compression.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md).

### Theme 3 — Routing is the new caching

Four papers (FrugalGPT, RouteLLM, Confidence-Driven Router, EAGLE-3) define a **three-generation routing arc**:

1. **Heuristic cascades** ([86](86-frugalgpt.md), 2023). Run cheap → expensive in sequence; escalate when scoring confidence drops. ~98% cost reduction at quality parity in batch QA.
2. **Learned routers** ([87](87-routellm.md), 2024). Train a binary classifier on Chatbot Arena preferences; pick weak vs strong per query. ~75% cost cut at PGR 50% vs random.
3. **Calibration-based** ([88](88-confidence-driven-router.md), 2025). Use weak model's own uncertainty as the routing signal; no preference data needed.

Plus a fourth, orthogonal lever:
4. **Speculative decoding** ([94](94-eagle3-spec-decoding.md), 2025). Train a draft head against the target model's hidden states; ~6× single-stream speedup on Llama 3.3 70B.

These compose multiplicatively. A harness using **routing + spec decoding + caching** can ship the same quality at 10× lower cost than naive single-model serving. For self-hosted deployments, this is the difference between "viable" and "infeasible" at production scale.

The implication for harness engineering: routing is no longer optional. Every harness in 2026 is implicitly or explicitly an LLM router. The papers above standardize the patterns (cascade, learned, confidence) so harness designers don't have to reinvent.

### Theme 4 — Skills are the unit of capability composition

Four papers center on skills as the abstraction:

- **Voyager ([89](89-voyager-deep.md))** — automatic curriculum + iterative code prompting + skill library; the canonical reference.
- **Skill-RAG ([79](79-skill-rag.md))** — four retrieval skills (rewrite, decompose, focus, exit) chosen by a hidden-state prober.
- **DSPy ([93](93-dspy.md))** — modules as composable units, signatures as type contracts, teleprompters that compile prompts/weights from data.
- Implicitly: **Atomic Skills (existing [68](68-atomic-skills-scaling-coding-agents.md))** — five RL-trained atomic policies for coding.

These differ in what a "skill" *is*: code function (Voyager), retrieval action (Skill-RAG), declarative module (DSPy), trained policy (Atomic Skills). But they share the same architectural commitment: **a skill is a named, scoped, composable, callable capability with a contract.** Modern agent harnesses (Claude Skills, gstack 23-specialist, Multica team workspace) all reify this.

The unification: skills + memory + routing + verification = the four substrates of every modern agent harness. None of these existed as named first-class concepts in 2022; all four are mature design vocabulary by 2026.

### Theme 5 — RAG is an active attack surface

Three papers triangulate the RAG-security frontier:

- **PoisonedRAG ([82](82-poisonedrag.md))** — the attack: 5 crafted documents in a million-doc corpus flip 97% of NQ answers with PaLM2.
- **Skill-RAG ([79](79-skill-rag.md))** — partial defense via failure-aware routing (the "exit" skill avoids confidently wrong answers).
- **KnowRL ([80](80-knowrl.md))** — orthogonal defense: train the model to know its knowledge boundary, reducing reliance on retrieved (potentially poisoned) content.

The architectural lesson: **the harness layer is the only place to defend against PoisonedRAG-class attacks.** Models cannot detect that retrieved documents are crafted (the documents look semantically perfect). Retrievers cannot detect this (the documents are designed to score high). Defense lives in the harness — corpus integrity (provenance, signatures), retrieval-time perplexity filters, ensemble retrievers, and verifier loops that disagree with retrieved content. See [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [25-agentic-rag](25-agentic-rag.md), [35-malicious-intermediary-attacks](35-malicious-intermediary-attacks.md), [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md).

### Theme 6 — Multi-agent ≠ multi-perspective: the diversity collapse problem

Three papers expose the dark side of multi-agent systems:

- **MetaGPT ([91](91-metagpt-deep.md))** — encodes SOPs as agent communication; succeeds by **constraining** rather than expanding the solution space.
- **ChatDev ([92](92-chatdev.md))** — chat-chain SDLC; same compositional pattern, different protocol.
- **Diversity Collapse ([98](98-diversity-collapse-mas.md))** — proves empirically across 10K+ proposals that the dominant assumption "more agents = more diversity" is **false**. Structural coupling at three levels (model intelligence, agent cognition, system dynamics) collapses diversity below single-model output.

The synthesis: MetaGPT and ChatDev work for *convergent* tasks (build this app, write this test) where coordination + role specialization actually helps. They fail for *divergent* tasks (generate ideas, propose hypotheses, brainstorm) where the SOP-induced structural coupling drives premature consensus. **Picking MAS vs single-agent is a function of task divergence, not task complexity.**

The Diversity Collapse paper's prescriptions (Nominal Group Technique with private writing, subgroup decomposition, vertical persona mix, Leader/Explorer/Judge architecture) are now table stakes for any harness designer building MAS for creative or open-ended tasks. See also [02-subagent-delegation](02-subagent-delegation.md), [73-multica-managed-agents-platform](73-multica-managed-agents-platform.md).

### Theme 7 — Evaluation has caught up to the frontier

Two benchmarks matter most as 2026's evaluation infrastructure:

- **OSWorld ([95](95-osworld.md))** — 369 real-OS tasks; **best agent 12% vs human 72%**. Massive headroom; the canonical benchmark for computer-use agents.
- **GDPval ([96](96-gdpval.md))** — 1,320 tasks across 44 occupations × 9 GDP-contributing sectors; pairwise human-vs-AI judgments. The new bar for "AI doing real work humans get paid for."

Plus a third infrastructure piece:
- **Qwen PRM Lessons ([97](97-qwen-prm.md))** — when training process reward models for math, **LLM-as-judge step labels actively harm performance.** Consensus filter (MC sampling + LLM voting) is the recipe.

The implication: harness designers can no longer treat evaluation as an afterthought. Every production harness needs (a) trajectory-level eval (see [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md)), (b) sector-grounded value targets (GDPval-style), (c) execution-based scoring (OSWorld-style), and (d) careful PRM design if it uses verifier loops. The Qwen lessons in particular are bitter — naïve evaluator pipelines are silently worse than no pipeline.

### Theme 8 — Self-improving agents move from research to deployment

Three papers stake out this frontier:

- **AlphaEvolve ([85](85-alphaevolve.md))** — Gemini-based evolutionary coding agent; cracks open algorithmic problems, real datacenter wins.
- **ReasoningBank + MaTTS ([81](81-reasoningbank.md))** — agents that grow a memory of reasoning patterns across episodes.
- **Voyager ([89](89-voyager-deep.md))** — automatic curriculum + skill library; the original self-improvement template.

Adjacent existing files: [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md), [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md), [45-hyperagents-self-modification](45-hyperagents-self-modification.md), [36-autogenesis-self-evolving-agents](36-autogenesis-self-evolving-agents.md).

The pattern: **self-improvement = curriculum + memory + verification + composition.** Take any of these four legs out and the agent stops improving. AlphaEvolve has all four (evolutionary curriculum, program archive, evaluator ensemble, mutation composition). Voyager has all four. ReasoningBank focuses on memory but plugs into existing curriculum + verification.

### Theme 9 — Inference economics are now a first-class harness concern

Combine the routing papers (FrugalGPT, RouteLLM, Confidence-Router, EAGLE-3) with the TTS papers (Meta TTS, SWE-Search, AlphaEvolve) and a clear picture emerges:

- TTS adds 2–10× compute per task to lift quality 5–30 pp.
- Routing + spec decoding cuts compute 5–10× at quality parity.

The net: in 2026, **a well-built harness can deliver ~1 generation of base-model quality lift at ~1× the cost of a naive single-model deployment**, by combining cost-cutters and quality-lifters. This is roughly the magnitude of a frontier-model release, achieved without retraining.

For deployments with a fixed quality target, the harness designer's job is to find the routing + TTS combination that minimizes cost. For deployments with a fixed cost budget, the job is to find the combination that maximizes quality. **The harness is now the cost-quality optimization surface.** This was not true in 2024.

### Theme 10 — Declarative pipelines vs free-form agents are not opposites

DSPy ([93](93-dspy.md)) represents the declarative camp: explicit signatures, composable modules, compiled prompts. Free-form agents (the Claude Code / Cursor camp) work in the opposite direction: a single agent loop with tools, prompted in natural language, no compilation step. Both work; both are deployed in production.

The synthesis: declarative pipelines win when the task is **stable, repeated, evaluable** (extraction pipelines, classification, multi-hop QA). Free-form agents win when the task is **novel, exploratory, hard to spec** (debugging, research, creative). The two are increasingly **stacked** — DSPy compiles the inner predicates that a free-form agent calls as tools.

See [42-langchain-deep-agents](42-langchain-deep-agents.md), [13-react](13-react.md), [02-subagent-delegation](02-subagent-delegation.md).

## Five emergent stack patterns

### Pattern 1 — The TTS stack
```
Frontier model
  ↓ (multiple parallel rollouts, fresh containers)
Compact rollout summaries (Meta TTS)
  ↓ (RTV selection)
Top-K selected summaries
  ↓ (PDR refinement context)
Iteration-2 rollouts
  ↓ (Final RTV)
Single best output
```
Adopters: Lyra (per project plans), gstack-style 7-phase sprints, frontier coding agents.

### Pattern 2 — The economy-aware stack
```
Query
  ↓ (RouteLLM / Confidence-Router)
[Route to weak] → spec-decoded weak model → response
[Route to strong] → spec-decoded frontier model → response
  ↓
[If completed cache hit] → cached response (FrugalGPT)
```
Adopters: any production deployment serving >10K requests/day.

### Pattern 3 — The skill-library stack (Voyager-descendant)
```
Task
  ↓ (curriculum proposes)
Skill retrieval (similar past skills)
  ↓ (compose)
Code generation
  ↓ (execute, verify)
[Success] → save to skill library
[Failure] → reflect, retry
```
Adopters: gstack (23 specialist skills), Atomic Skills, Voyager itself, Multica skill workspaces.

### Pattern 4 — The MAS-with-diversity-protection stack
```
Open-ended task
  ↓ (Nominal Group Technique: private writing first)
Multiple independent proposals
  ↓ (subgroup decomposition by topic)
Subgroup deliberation (with vertical persona mix)
  ↓ (Leader / Explorer / Judge architecture)
Cross-subgroup synthesis
```
Adopters: Diversity-collapse-aware MAS (still rare in 2026, but growing).

### Pattern 5 — The evaluation harness
```
Trajectory
  ↓ (LM judge with consensus filter — Qwen PRM)
Step-level scores
  ↓ (PRM aggregates to trajectory score)
[Score above threshold] → ship
[Below threshold] → reflect (Reflexion) → retry
```
Adopters: any production harness with verifier loops; Lyra's evaluator pipeline; OpenAI/Anthropic internal evals.

## Three open questions left by these papers

1. **How do TTS, MAS, and skill libraries compose?** Each paper takes one axis as primary. No paper answers what happens when you stack RTV (across-rollout TTS) on top of MCTS (within-rollout TTS) on top of MetaGPT-style MAS over a Voyager-style skill library. The combinatorics of harness components is now the open frontier.

2. **Where is the diversity-collapse mitigation actually validated?** Diversity Collapse ([98](98-diversity-collapse-mas.md)) prescribes NGT + Subgroups + Vertical persona mix. The paper validates these at the proposal level. **Nobody has yet shown an end-to-end MAS-coding-agent that uses these mitigations and beats a single-agent baseline on a real task.** The prescriptions are theoretically sound but operationally unproven.

3. **Can self-improvement compound?** AlphaEvolve, ReasoningBank, and Voyager all show one round of self-improvement. None show that **N rounds of self-improvement** continue to compound. Curriculum saturation, memory bloat, and skill-library decay are real effects. The paradigm of "set it running and come back in 6 months" is still aspirational.

## Recommended reading order

If you have **2 hours**:
1. [77-meta-tts-agentic-coding](77-meta-tts-agentic-coding.md) — the cleanest TTS paper, sets the frame
2. [98-diversity-collapse-mas](98-diversity-collapse-mas.md) — the most consequential MAS finding
3. [81-reasoningbank](81-reasoningbank.md) — the principled memory architecture
4. [82-poisonedrag](82-poisonedrag.md) — the canonical RAG threat model

If you have **a weekend**: the four above, plus the canonical references — [89-voyager-deep](89-voyager-deep.md), [90-reflexion-deep](90-reflexion-deep.md), [91-metagpt-deep](91-metagpt-deep.md), [92-chatdev](92-chatdev.md), [93-dspy](93-dspy.md).

If you have **a week**: all 22, in this order: capabilities (77, 78, 79, 80, 81, 82, 83), edges (84, 85, 86, 87, 88, 94), benchmarks (95, 96, 97), foundations (89, 90, 91, 92, 93), hardening (98). End with this synthesis to consolidate.

## How this paper-set relates to other corpus material

- **Files 01–25** — foundational harness/agentic AI techniques. Most paper-deep-dives in this set extend or update those techniques with 2024–2026 evidence. Read 01–25 first if new to the space.
- **Files 26–50** — extended research set covering 2025–2026 papers, industry essays, benchmarks. Heavy overlap in topics with this paper set (e.g. 27 long-horizon, 31 GLM-5 agentic, 36 autogenesis); the paper-set is more rigorous, the 26–50 set is broader.
- **Files 53–67** — 2025-2026 ecosystem and regional coverage; SemaClaw ([54](54-semaclaw-general-purpose-agent.md)) is the same paper as [83](83-semaclaw-deep.md) but seen from the Lyra-architecture angle.
- **Files 68–76** — April 2026 landscape ten-link set; [76-ten-links-synthesis](76-ten-links-synthesis.md) is the analog of *this* file for community projects rather than papers. Read both syntheses for the full landscape.

## Bottom-line takeaways

1. **Harness engineering is now the dominant cost-quality lever.** A well-built harness in 2026 delivers ~1 generation of base-model lift at ~1× cost via TTS + routing + spec decoding + memory + skills.
2. **Test-time scaling for long-horizon agents is fundamentally a representation problem** (Meta TTS thesis). Compact summaries are the substrate.
3. **Memory is the substrate that turns one-shot agents into durable agents.** Three layers: in-cache (NGC), in-context (Reflexion), external (ReasoningBank, claude-mem).
4. **Multi-agent systems collapse diversity by default.** Adopt NGT + Subgroups + Vertical persona mix or stay single-agent.
5. **RAG is an active attack surface.** The harness layer is the only place to defend.
6. **Skills are the unit of capability composition** — code (Voyager), retrieval action (Skill-RAG), declarative module (DSPy), trained policy (Atomic Skills).
7. **Routing is the three-generation cost lever** (heuristic → preference → calibration), and spec decoding is its inference-side complement.
8. **Evaluation has caught up:** OSWorld and GDPval are the new bars; Qwen PRM lessons are mandatory reading for evaluator design.
9. **Self-improvement = curriculum + memory + verification + composition.** Take one leg out, the agent stops improving.
10. **The frontier is composition, not single-paper innovation.** The most interesting 2027 work will stack TTS over MAS over skills over evaluation, not invent a new fifth axis.

## References

All 22 paper-grounded deep-dives in this corpus, files [77](77-meta-tts-agentic-coding.md) through [98](98-diversity-collapse-mas.md), plus the original arXiv PDFs in `projects/lyra/papers/`. Adjacent corpus syntheses: [76-ten-links-synthesis](76-ten-links-synthesis.md), [66-meta-harness-landscape](66-meta-harness-landscape.md).
