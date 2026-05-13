# Harness Engineering & Agentic AI — Deep-Dive Index

A curated collection of 99 deep-dives on techniques for building reliable AI agents, drawn from 2024–2026 engineering blogs, arXiv papers, and production frameworks. The focus is balanced between **harness engineering** — the scaffolding around an LLM (loop, tools, context, permissions, verification) — and **agentic AI** — the reasoning and evaluation techniques the harness is built for.

Files **77–99** are paper-grounded deep-dives of 22 frontier papers selected by the Lyra project (mirrored in `projects/lyra/papers/`), plus a synthesis. Read [99](99-papers-deep-dive-synthesis.md) first if you want the consolidated view of where the field sits in April 2026.

> **Read it as a book.** `index.html` in this folder is a single self-contained interactive book of all 99 chapters, designed so non-technical readers can follow along. Each chapter opens with:
>
> - a **chapter icon** + **one-sentence TLDR** for instant orientation,
> - a **real-world analogy card** (chef during dinner rush, chess engine, bouncer at the door…) that grounds the concept in something familiar,
> - a **plain-English ELI5 metaphor + explanation** (toggleable via the sidebar; on by default),
> - **key-numbers stat cards** for the impactful metrics ("+23% on SWE-Bench", "98% cost reduction"),
> - a **live diagram** — either a hand-crafted Mermaid flowchart or an auto-generated concept card,
> - **glossary tooltips** that pop up when you hover any technical term (RAG, MCP, KV cache, PRM, RTV…),
> - **line-by-line code walkthroughs** — the pseudo-code on the core chapters (agent loop, MCP, Reflexion, Voyager, guardrails, agentic RAG, RTV/Meta TTS, DSPy, Deep Agents, verifier loop, HITL, hooks) is paired with a plain-English panel: what each line does, a worked example, and the one thing to remember.
>
> Plus: cover page, sidebar TOC, full-text search, dark mode, reading progress, bookmarks. Just double-click `index.html` (or run `python3 -m http.server` in this folder and visit `http://localhost:8000`). Re-build after edits with `python3 build_book.py`.

## How to read this folder

Every deep-dive follows the same structure:

1. One-line definition
2. Problem it solves
3. Mechanism (step-by-step)
4. Concrete patterns / pseudo-code
5. Variants & related techniques
6. Failure modes & anti-patterns
7. When to use (and when not to)
8. References

Files are numbered 01–25. Cross-references between files use relative links. If you are new to the space, read in order; if you are picking solutions to specific problems, scan the one-line descriptions below.

## Harness & Scaffolding (the "how" of agents)

1. [Agent Loop Architecture](01-agent-loop-architecture.md) — the think/act/observe loop, step budgets, and termination conditions that define what "being an agent" even means.
2. [Subagent Delegation](02-subagent-delegation.md) — orchestrator-worker pattern with isolated context windows; when it wins and when it fragments coherence.
3. [Plan Mode](03-plan-mode.md) — a read-only planning phase that produces an artifact the user approves before any mutation.
4. [Skills (SKILL.md)](04-skills.md) — model-invocable capability packages with progressive disclosure of instructions and tools.
5. [Hooks](05-hooks.md) — deterministic pre/post-tool-use guardrails that run shell, HTTP, LLM, or agent actions on events.
6. [Permission Modes](06-permission-modes.md) — safe-autonomy gradients (plan / default / acceptEdits / bypass) and why they matter for trust.
7. [Model Context Protocol (MCP)](07-model-context-protocol.md) — a standardized client-server tool interface decoupling agents from integrations.
8. [Context Compaction](08-context-compaction.md) — recursive and hierarchical summarization to survive context-window limits without losing ground truth.
9. [Memory Files](09-memory-files.md) — persistent semantic/episodic memory as first-class files (CLAUDE.md, memory/*.md).
10. [Multi-Session Continuity](10-multi-session-continuity.md) — initializer + coding-agent split, artifact handoffs, multi-hour autonomy.
11. [Verifier/Evaluator Loops](11-verifier-evaluator-loops.md) — GAN-inspired planner/generator/evaluator harnesses for long-running coding.
12. [Todo / Scratchpad State](12-todo-scratchpad-state.md) — externalizing the plan as a file/tool the agent can reread, so attention degradation doesn't kill progress.

## Agentic Reasoning Techniques (the "what" agents do with the harness)

13. [ReAct](13-react.md) — the interleaved reasoning + acting pattern that most modern agent loops descend from.
14. [Reflexion](14-reflexion.md) — verbal self-reflection written into episodic memory to improve across episodes without fine-tuning.
15. [Tree of Thoughts / LATS](15-tree-of-thoughts-lats.md) — search over a tree of candidate reasoning steps with explicit value functions.
16. [Plan-and-Solve](16-plan-and-solve.md) — separate planning from execution so the plan is criticized before resources are burned.
17. [ReWOO](17-rewoo.md) — reason about information needs first, then fetch; decouples reasoning from observations to cut tool-call waste.
18. [Chain-of-Verification & Self-Refine](18-chain-of-verification-self-refine.md) — structured self-critique loops for factuality.
19. [Voyager-Style Skill Libraries](19-voyager-skill-libraries.md) — curriculum learning plus a growing library of reusable skills.
20. [MetaGPT Role-Based Workflows](20-metagpt-role-based-workflows.md) — imposing SDLC roles and artifacts on multi-agent collaboration.

## Production & Reliability (making agents survive contact with users)

21. [LLM-as-Judge & Trajectory Evaluation](21-llm-as-judge-trajectory-eval.md) — scoring whole trajectories, not just final answers, at scale.
22. [Guardrails & Prompt-Injection Defense](22-guardrails-prompt-injection.md) — structural defenses against adversarial inputs and unsafe outputs.
23. [Human-in-the-Loop Approval](23-human-in-the-loop.md) — routing high-stakes actions for human sign-off without killing autonomy.
24. [Observability, Tracing & Cost Attribution](24-observability-tracing.md) — making the inside of a black-box agent visible and billable.
25. [Agentic RAG with Self-Critique](25-agentic-rag.md) — dynamic retrieval with relevance verification loops, not one-shot RAG.

## Extended Research Set — 2025–2026 papers, industry essays, benchmarks

One file per source in the user-supplied reading list. Sources synthesized from primary links where retrievable; files with insufficient primary access note that explicitly.

### Benchmarks & evaluation
26. [LinuxArena — Production-Agent Safety](26-linuxarena-production-agent-safety.md) — real Linux prod environments + sabotage tasks (arXiv:2604.15384).
27. [HORIZON — Long-Horizon Failure Attribution](27-horizon-long-horizon-degradation.md) — why agents degrade as horizons grow (arXiv:2604.11978).
34. [ClawBench — Live Web Tasks](34-clawbench-live-web-tasks.md) — 153 tasks × 144 production sites; 33.3% best-model success (arXiv:2604.08523).
38. [Claw-Eval — Trajectory-Aware Evaluation](38-claw-eval.md) — 3 evidence channels, Pass^k, 44% safety-violation recovery (arXiv:2604.06132).

### Harness & architecture — Claude Code era
29. [Dive into Claude Code — Architecture Under the Hood](29-dive-into-claude-code.md) — reverse-engineered source; 13 principles (arXiv:2604.14228).
40. [The Art of AI Harness Engineering — Industry Principles](40-harness-engineering-principles.md) — BDTechTalks framing from the Claude Code leak.
43. [Twelve Agentic Harness Patterns — Catalog](43-twelve-harness-patterns.md) — Generative Programmer's named-patterns catalog.
44. [Four Pillars of Harness Engineering](44-four-pillars-harness-engineering.md) — State, Context, Guardrails, Entropy.
46. [Components of a Coding Agent (Raschka)](46-components-of-coding-agent.md) — six components every coding harness needs.

### Frameworks & product layer
41. [Product Control Plane for Multi-Agent Systems](41-product-control-plane.md) — Permissions / Handoffs / Visibility / Recovery (Adaline Labs).
42. [LangChain Deep Agents — Planning, Virtual FS, Async Subagents](42-langchain-deep-agents.md) — the open-source distillation of the Claude Code / Devin pattern.
52. [Dive Into OpenClaw — The Open-Source Agent Harness](52-dive-into-open-claw.md) — Gateway + pluggable harness; the MIT-licensed foil to Claude Code.

### Model & reasoning advances
31. [GLM-5 — Training for Agentic Engineering](31-glm-5-agentic-engineering.md) — async RL, long-horizon trajectories (arXiv:2602.15763).
32. [Recurrent-Depth Transformers — Implicit Reasoning](32-recurrent-depth-implicit-reasoning.md) — iterate layers for reasoning depth (arXiv:2604.07822).
51. [ReBalance — Confidence-Guided Balanced Thinking](51-rebalance-efficient-reasoning.md) — training-free overthink/underthink control (arXiv:2603.12372).

### Self-improving & self-modifying agents
36. [Autogenesis — Self-Evolving Agent Protocol](36-autogenesis-self-evolving-agents.md) — RSPL / SEPL resource protocols (arXiv:2604.15034).
45. [Hyperagents — Metacognitive Self-Modification](45-hyperagents-self-modification.md) — DGM-H, editing the editing mechanism (arXiv:2603.19461).
47. [Adaptation of Agentic AI — Survey](47-adaptation-of-agentic-ai-survey.md) — 4-paradigm taxonomy over post-training, memory, skills (arXiv:2512.16301).

### Domain-specialized agents
28. [RadAgent — Tool-Using Radiology Agent](28-radagent-agentic-radiology.md) — inspectable stepwise CT interpretation (arXiv:2604.15231).
30. [GPT-Rosalind — Domain-Specialized Reasoning](30-gpt-rosalind-domain-specialized.md) — OpenAI's life-sciences model, workflow-trained.
33. [dnaHNet — Hierarchical Genomic Foundation Model](33-dnahnet-genomic-foundation.md) — tokenizer-free, dynamic chunking (arXiv:2602.10603).
39. [AI and the Structure of Mathematics](39-ai-and-mathematics-structure.md) — proof hypergraphs, AI-driven discovery (arXiv:2604.06107).

### Safety, red-teaming & supply chain
35. [Malicious Intermediary Attacks on the LLM Supply Chain](35-malicious-intermediary-attacks.md) — compromised API routers (arXiv:2604.08407).
49. [Agents of Chaos — Red-Teaming Autonomous Agents](49-agents-of-chaos-red-teaming.md) — 11 failure case studies (arXiv:2602.20021).

### Specialized architectures
48. [VoiceAgentRAG — Dual-Agent Real-Time Voice](48-voiceagentrag-dual-agent.md) — Slow Thinker + Fast Talker (arXiv:2603.02206).

### Neuro-symbolic & reasoning
37. [Neuro-Symbolic AI — Marcus & Belle AAAI 2026](37-neuro-symbolic-ai.md) — the case against scaling-alone.
50. [METCL — Neuro-Symbolic Metaphor Reasoning](50-metcl-metaphor-reasoning.md) — typicality-based compositional logic (IJCAI 2025).

## April-2026 Landscape — Ten-link deep-dive set

This set of nine new deep-dives + two pre-existing files covers the April-2026 state of the harness-engineering landscape across research papers, community artifacts, curation projects, managed-agent platforms, domain foundation models, and persistent-memory plugins. Read the synthesis ([76](76-ten-links-synthesis.md)) first to orient, then drill into the individual files.

### Research papers (April 2026 arXiv)
68. [Scaling Coding Agents via Atomic Skills](68-atomic-skills-scaling-coding-agents.md) — five atomic skills + joint RL + 10K-concurrent sandboxes, +18.7% average (arXiv:2604.05013).
69. [Agent-World — Self-Evolving Training Arena](69-agent-world-self-evolving-training-arena.md) — 1,978 environments + 19,822 tools + closed-loop co-evolution (arXiv:2604.18292).

### Research curation
70. [VoltAgent — Awesome-AI-Agent-Papers Ledger](70-voltagent-awesome-ai-agent-papers.md) — 363+ papers across Multi-Agent / Memory & RAG / Eval / Tooling / Security buckets.

### Community artifacts — skills, memory, orchestration
71. [Karpathy-Skills — Single-File Guardrails](71-karpathy-skills-single-file-guardrails.md) — 60-line CLAUDE.md codifying four principles; 71K stars.
72. [Claude-Mem — Persistent Memory Compression](72-claude-mem-persistent-memory-compression.md) — 5 hooks + worker + Chroma + 3-layer progressive disclosure; 65K stars.
73. [Multica — Managed-Agents Platform](73-multica-managed-agents-platform.md) — team-scale orchestration across 8 agent CLIs; 18K stars.
75. [gstack — Garry Tan's Claude-Code Setup](75-gstack-garry-tan-claude-code-setup.md) — 23 specialist skills + 7-phase sprint + 810× productivity claim.

### Domain foundation models
74. [Kronos — Financial-Markets Foundation Model](74-kronos-foundation-model-financial-markets.md) — first open FM for candlestick data; 4.1M-499M params; 20K stars.

### Pre-existing files in the ten-link set
55. [Hermes Agent — Nous Research Self-Improving Agent](55-hermes-agent-self-improving.md) — hosted self-improving agent.
62. [Everything-Claude-Code — Community Artifact Bundle](62-everything-claude-code.md) — curated Claude-Code ecosystem.

### Synthesis
76. [Ten-Link Synthesis — April 2026 Landscape](76-ten-links-synthesis.md) — ten cross-cutting themes + three emergent stack patterns + two open questions + recommended reading order.

## Lyra Paper-Set Deep-Dives — 22 frontier papers, paper-grounded

This set of 22 deep-dives + a synthesis covers the Lyra project's reference papers (mirrored locally in `projects/lyra/papers/`). Each file is grounded directly in the actual paper PDF (not secondary sources): full citation, mechanism with equations and pseudocode, experimental results with concrete numbers, ablations, honest limitations, and harness-engineering implications. Read the synthesis ([99](99-papers-deep-dive-synthesis.md)) first to orient, then drill into individual files. Where a paper was already covered by an earlier summary file (Voyager #19, Reflexion #14, MetaGPT #20, SemaClaw #54), the new file is a paper-grounded complement, not a replacement.

### Wave 1 — Capabilities (test-time scaling, memory, skills, factuality, security)
77. [Meta TTS — Scaling Test-Time Compute for Agentic Coding](77-meta-tts-agentic-coding.md) — RTV + PDR over rollout summaries, +7 to +12 pp on SWE/Terminal-Bench (arXiv:2604.16529).
78. [NGC — Neural Garbage Collection](78-ngc-neural-garbage-collection.md) — RL-train the LM to forget while reasoning; 2–4× KV cache compression (arXiv:2604.18002).
79. [Skill-RAG — Failure-State-Aware Retrieval via Hidden-State Probing](79-skill-rag.md) — 4-skill router for adaptive RAG (arXiv:2604.15771).
80. [KnowRL — Knowledgeable RL for Factuality](80-knowrl.md) — knowledge-aware reward for reasoning models (arXiv:2506.19807).
81. [ReasoningBank + MaTTS — Self-Evolving Agents via Reasoning Memory](81-reasoningbank.md) — Google Research; memory-aware test-time scaling (arXiv:2509.25140).
82. [PoisonedRAG — Knowledge Corruption Attacks on RAG](82-poisonedrag.md) — 5 docs flip 97% of NQ answers (USENIX 2025; arXiv:2402.07867).
83. [SemaClaw (paper-grounded)](83-semaclaw-deep.md) — Midea AIRC's general-purpose personal-AI harness; complements [54](54-semaclaw-general-purpose-agent.md) (arXiv:2604.11548).

### Wave 2 — Performance edges (search, evolution, routing, evaluation, multi-agent foundations)
84. [SWE-Search — MCTS-Augmented Software Agents](84-swe-search-mcts.md) — intra-attempt MCTS, +23% mean relative on SWE-Bench Lite (ICLR 2025; arXiv:2410.20285).
85. [AlphaEvolve — Evolutionary Coding Agent for Scientific Discovery](85-alphaevolve.md) — DeepMind; 4×4 complex matmul in 48 mults, real datacenter wins (arXiv:2506.13131).
86. [FrugalGPT — Cost-Aware LLM Cascades and Routing](86-frugalgpt.md) — Stanford; up to 98% cost reduction at quality parity (arXiv:2305.05176).
87. [RouteLLM — Learning to Route with Preference Data](87-routellm.md) — UC Berkeley / LMSYS; 75% cost cut at PGR 50% (arXiv:2406.18665).
88. [Confidence-Driven LLM Router](88-confidence-driven-router.md) — calibration-based escalation, third-generation routing (arXiv:2502.11021).
89. [Voyager (paper-grounded)](89-voyager-deep.md) — auto-curriculum + skill library; the canonical reference, complements [19](19-voyager-skill-libraries.md) (TMLR 2024; arXiv:2305.16291).
90. [Reflexion (paper-grounded)](90-reflexion-deep.md) — verbal RL; HumanEval 91% pass@1 with GPT-4; complements [14](14-reflexion.md) (NeurIPS 2023; arXiv:2303.11366).
91. [MetaGPT (paper-grounded)](91-metagpt-deep.md) — SDLC SOPs as multi-agent comms; complements [20](20-metagpt-role-based-workflows.md) (ICLR 2024; arXiv:2308.00352).
92. [ChatDev — Communicative Agents for Software Development](92-chatdev.md) — Tsinghua/OpenBMB; chat-chain SDLC (arXiv:2307.07924).
93. [DSPy — Compiling Declarative LM Pipelines](93-dspy.md) — Stanford; signatures + modules + teleprompters (ICLR 2024; arXiv:2310.03714).
94. [EAGLE-3 — Speculative Decoding via Training-Time Test](94-eagle3-spec-decoding.md) — ~6× single-stream speedup on Llama 3.3 70B (arXiv:2503.01840).
95. [OSWorld — Computer-Use Agents in Real OS Environments](95-osworld.md) — 369 tasks; best agent 12% vs human 72% (NeurIPS 2024; arXiv:2404.07972).
96. [GDPval — Evaluating AI on Real-World Economic Tasks](96-gdpval.md) — OpenAI; 1,320 tasks × 44 occupations × 9 GDP sectors (arXiv:2510.04374).
97. [Qwen2.5-Math PRM — Lessons in Process Reward Models](97-qwen-prm.md) — LLM-as-judge step labels actively harm PRMs (arXiv:2501.07301).

### Wave 3 — Hardening
98. [Diversity Collapse in Multi-Agent LLM Systems](98-diversity-collapse-mas.md) — NUS/CUHK-SZ; structural coupling collapses MAS diversity at three levels (ACL 2026 Findings; arXiv:2604.18005).

### Synthesis
99. [Lyra Paper-Set Synthesis](99-papers-deep-dive-synthesis.md) — ten cross-cutting themes + five emergent stack patterns + three open questions + reading orders.

## Part VIII — May-2026 Update (chapters 100–114)

Fifteen new deep-dives covering the agentic-AI frontier as of May 2026. Themes: pure-RL reasoning recipes, multi-turn agentic RL with diagnostic vocabulary, persistent-memory architectures, the OS-level computer-use race, and the OSS-framework consolidation. Read [114](114-may-2026-landscape-update.md) first for the synthesis, then drill in.

### Reasoning models and agentic RL
100. [DeepSeek-R1 — Pure RL for Reasoning](100-deepseek-r1-rl-reasoning.md) — GRPO + verifiable reward elicits reasoning without CoT data; foundational for the open reasoning-model era (arXiv:2501.12948; Nature 2025).
101. [RAGEN — Multi-Turn RL and the Echo Trap](101-ragen-multi-turn-rl-agents.md) — StarPO for interactive agents; names the diversity-collapse failure mode (arXiv:2504.20073).
102. [ARTIST — Agentic Reasoning + Tool RL](102-artist-agentic-rl-tools.md) — couples reasoning, RL, and tool use; tool-result masking is non-negotiable (arXiv:2505.01441).
103. [Agent Lightning — Train Any Agent with RL](103-agent-lightning.md) — proxy-based RL retrofit for existing agent codebases (Microsoft Research; arXiv:2508.03680).

### Memory and statefulness
104. [Mem0 — Production-Ready Long-Term Memory](104-mem0-production-memory.md) — extract → consolidate → retrieve; +26% over OpenAI built-in memory on LOCOMO (ECAI 2025; arXiv:2504.19413).
105. [Letta — Stateful Persistent Agents](105-letta-stateful-agents.md) — the MemGPT lineage; self-managed memory tools and persistent agent identity (arXiv:2310.08560).

### Computer-use and browser surfaces
106. [Computer-Use Agents — Anthropic vs OpenAI Operator](106-computer-use-agents.md) — OSWorld 22% → 61.4% in 12 months; HITL gates as the safety contract.
107. [Browser-Use — DOM-Aware Open-Source Browser Agent](107-browser-use.md) — element-index actions; surpasses Operator on WebVoyager.

### Coding agents and frameworks
108. [OpenHands & CodeAct — Code-as-Action](108-openhands-codeact.md) — most active OSS coding-agent project; Python actions in a Jupyter sandbox (arXiv:2402.01030).
109. [smolagents — HuggingFace's Code-Writing Agents](109-smolagents.md) — minimal code-first framework; ~14.8k stars in 15 months.
110. [LangGraph — State-Graph Agent Framework](110-langgraph.md) — explicit graphs, persistence, time-travel, HITL interrupts; surpassed CrewAI in stars in early 2026.

### Multi-agent and evaluation
111. [Magentic-One & Magentic-UI — Generalist Multi-Agent](111-magentic-one.md) — Microsoft Research; orchestrator + 4 specialists; HITL companion UI (arXiv:2411.04468).
112. [BrowseComp — Benchmark for Browsing Agents](112-browsecomp.md) — OpenAI; 1,266 hard-to-find-fact questions; Deep Research at ~50% (arXiv:2504.12516).
113. [Test-Time Compute Scaling for Agents](113-test-time-compute-agents.md) — parallel / sequential / verifier-guided / diversification; the four levers and when each wins.

### Synthesis
114. [May-2026 Landscape Update](114-may-2026-landscape-update.md) — synthesizes 100–113; what changed since file 99; the four axes; what's still open.

## Intermediate files (2025–2026 ecosystem and regional coverage)

Additional deep-dives in the 53-67 range cover chaos engineering, SemaClaw, SEA-region landscape, Archon harness-builder, RAGFlow, LobeHub, DeerFlow ByteDance, meta-harness landscape, and breakthrough-project recommendations.

53-54. Chaos engineering + SemaClaw general-purpose agent.
56-60. SEA (South-East Asia) agent landscape + SEA-specific arXiv papers + top SEA GitHub repos.
61, 63-65. Archon, RAGFlow, LobeHub, DeerFlow-ByteDance production frameworks.
66-67. [Meta-Harness Landscape](66-meta-harness-landscape.md) + [Recommended Breakthrough Project — Gnomon](67-recommended-breakthrough-project.md).

## Key sources

Across the collection, the most heavily-cited primary sources are:

- Anthropic Engineering blog — harness design, context engineering, Claude Agent SDK, effective harnesses for long-running agents.
- Cognition AI — "Don't Build Multi-Agents" (context-sharing primacy).
- HumanLayer — "Skill Issue: Harness Engineering for Coding Agents".
- Chroma Research — "Context Rot".
- arXiv papers for academic techniques (ReAct, Reflexion, ToT, LATS, Self-Refine, Plan-and-Solve, ReWOO, MetaGPT, Voyager, SWE-agent, ACE, Memory in the Age of AI Agents).
- LangSmith / LangChain docs, LlamaIndex docs, Nemo Guardrails for production patterns.
- Lilian Weng, Hamel Husain, Eugene Yan, Shreya Shankar, Chip Huyen for applied guidance.

Individual files cite the specific URLs and paper IDs.
