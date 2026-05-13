# 158 — Deep Research Agents: A Systematic Examination and Roadmap (Huang et al., 2025)

**Paper.** Yuxuan Huang† (University of Liverpool), Yihang Chen† (Huawei Noah's Ark Lab), Haozheng Zhang† (Huawei), Kang Li (University of Oxford), Huichi Zhou (UCL), Meng Fang (Liverpool), Linyi Yang (UCL), Xiaoguang Li (Huawei), Lifeng Shang (Huawei), Songcen Xu (Huawei), Jianye Hao (Huawei), Kun Shao‡ (Huawei), Jun Wang‡ (UCL) — *Deep Research Agents: A Systematic Examination And Roadmap* — arXiv:2506.18096v2 [cs.AI] — submitted 22 June 2025 (v1), revised 3 September 2025 (v2). Curated companion repository: https://github.com/ai-agents-2030/awesome-deep-research-agent.

**One-line definition.** This is *the* foundational systematisation of "Deep Research (DR) agents" — the new category of autonomous AI systems exemplified by OpenAI Deep Research, Gemini DR, Grok DeepSearch, Perplexity DR, and Manus — that combine **dynamic reasoning, adaptive long-horizon planning, multi-hop information retrieval, iterative tool use, and structured analytical report generation**. The survey provides a unified taxonomy across **(a) search-engine modality (API vs browser-based)**, **(b) workflow type (static vs dynamic)**, **(c) planning strategy (planning-only / intent-to-planning / unified intent-planning)**, **(d) agent composition (single-agent vs multi-agent)**, and **(e) tuning paradigm (prompt-driven, SFT, RL with PPO/GRPO/REINFORCE/DAPO/DUPO)** — and connects these to MCP and A2A protocol stacks, memory mechanisms, benchmark limitations, and a research roadmap.

**Why this paper is the canonical entry point.** Of the dozens of "AI agent survey" papers from 2024–2025, this is the one that names "Deep Research" as a distinct category, provides a definition that generalises across industry systems, and gives a taxonomy that has held up over the subsequent year. Anyone building or researching DR agents will end up referencing the contribution it makes: a *shared vocabulary* for the field. The 2026 papers in this canon ([100-contextual-memory-is-a-memo](100-contextual-memory-is-a-memo.md), [101-autoresearchbench](101-autoresearchbench.md), [109-memento-results-and-harness](109-memento-results-and-harness.md), [155-feynman-multi-agent-research-harness](155-feynman-multi-agent-research-harness.md), [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md), and the second DR survey [159-deep-research-survey-zhang-et-al](159-deep-research-survey-zhang-et-al.md)) all build on the vocabulary introduced here.

## The DR agent definition

The paper's definition is worth reading closely:

> **AI agents powered by LLMs, integrating dynamic reasoning, adaptive planning, and iterative tool use to acquire, aggregate, and analyse external information, culminating in comprehensive outputs for accomplishing open-ended informational research tasks.**

Five operative phrases pull weight:

1. **Powered by LLMs** — the cognitive core is a frontier LM, not symbolic search.
2. **Dynamic reasoning** — the agent re-plans mid-execution; not a fixed chain-of-thought.
3. **Adaptive planning** — task decomposition is generated, not pre-templated.
4. **Iterative tool use** — multiple rounds of tool calls; not a one-shot RAG.
5. **Open-ended informational research tasks** — the answer space is not a single fact; the deliverable is a synthesised analytical artefact.

This crisply distinguishes DR from three predecessor categories:

| Category | Example | DR-distinguishing absence |
|----------|---------|--------------------------|
| Question Answering | GPT-3 zero-shot QA | No retrieval, no planning, no iteration |
| Retrieval-Augmented Generation | Original RAG, FLARE, Self-RAG | Static pipeline; passive consumption of retrieved content; lacks sustained reasoning |
| Tool Use Agents | Toolformer, ReAct | Fixed workflow; no continual reasoning; pre-defined tool catalogue |
| **Deep Research Agents** | OpenAI DR, Gemini DR, Manus | All of: dynamic reasoning + adaptive planning + iterative tool use + open-ended research output |

The boundary between "agentic RAG" and "DR" is the most contested. Huang et al.'s position: agentic RAG sits on the continuum, but DR's *reliance on real-time external interaction* (browsers, APIs, code execution, multi-modal processing) and *generation of structured analytical reports* place it past the threshold.

## The structural contribution: a unified taxonomy

Figure 4 in the paper (and reproduced through Section 3.3) is the contribution that has the longest half-life. It organises DR systems along three independent axes:

```
Axis 1: Workflow type
    ├── Static (predefined sequential pipeline)
    │       Examples: AI Scientist, Agent Laboratory, AgentRxiv
    └── Dynamic (LLM-driven adaptive replanning)

Axis 2 (Dynamic only): Planning strategy — when does the agent ask the user?
    ├── Planning-Only (no clarification; plan from initial prompt)
    │       Examples: Grok DeepSearch, H2O.ai DR, Manus
    ├── Intent-to-Planning (clarify first, then plan)
    │       Example: OpenAI DR
    └── Unified Intent-Planning (plan, then ask user to confirm/revise)
            Example: Gemini DR

Axis 3 (Dynamic only): Agent composition
    ├── Single-Agent (LRM autonomously updates plan and executes)
    │       Examples: Search-o1, R1-Searcher, DeepResearcher,
    │                 WebDancer, WebSailor, PANGU DeepDiver,
    │                 Agent-R1, ReSearch, Search-R1, WebWatcher,
    │                 MiroRL, Memento, Kimi-Researcher
    └── Multi-Agent (specialised agents, hierarchical/centralised planning)
            Examples: OpenManus, Manus, OWL, Alita, AWorld,
                      Webwalker, WebThinker
```

Static workflows are easier to build and easier to reason about; they lose on flexibility. Dynamic workflows are the dominant production design for OpenAI DR, Gemini DR, Manus, and most 2025 systems.

The single-vs-multi-agent split is the most consequential design decision, because it determines *what kind of training is possible*:

- **Single-agent** systems can be optimised end-to-end with reinforcement learning (PPO, GRPO, REINFORCE++, DAPO, DUPO, Online DPO) over the entire trajectory — this is what DeepResearcher, R1-Searcher, Search-R1, WebDancer, WebSailor do.
- **Multi-agent** systems gain modular specialisation but lose the ability to do clean end-to-end RL — coordination complexity defeats the credit assignment.

The paper's commentary (§3.3.3): *"a major current challenge of multi-agent systems lies in the inherent complexity of coordinating multiple independent agents, making it difficult to conduct effective end-to-end reinforcement learning optimisation."* This is the same tension surfaced by [98-diversity-collapse-mas](98-diversity-collapse-mas.md) and central to the orchestration-vs-skills argument in [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md).

## The five technical components of a DR agent

Section 3 of the paper enumerates the five components that constitute a DR agent's architecture. Treat this as the canonical decomposition:

### Component 1 — Search engine: API vs browser

API-based search engines (Google Custom Search, Bing, Brave, scientific database APIs like Semantic Scholar / arXiv) interact with structured data. Pros: efficient, deterministic, easy to rate-limit and cache. Cons: limited to indexed data, no rich UI rendering, no behind-the-login content.

Browser-based search engines simulate human-like web interactions (typically via Playwright or Chromium drivers wrapped by tools like browser-use, Stagehand, Skyvern). Pros: can access any rendered page, can interact with forms and dynamic JS, can extract content from paywalls or interactive widgets. Cons: brittle, expensive, slow, and dependent on visual rendering reasoning by the LLM.

The paper's Table 1 cross-tabulates ~30 DR agents on this axis. The split is roughly: production industrial agents lean browser-first (OpenAI DR, Manus, Gemini DR), while research agents and benchmarks lean API-first because of reproducibility.

### Component 2 — Tool use

Tools fall into four categories per the survey:

1. **Code execution** — Python sandboxes (Jupyter, Docker, e2b.dev) that the agent can invoke for data analysis, plotting, or computation.
2. **File manipulation** — read/write/list operations on a workspace, often with a virtual filesystem to constrain blast radius.
3. **Multimodal processing** — image input understanding, audio transcription, OCR for documents.
4. **MCP integration** — using the Anthropic Model Context Protocol for standardised, interoperable tool access. Reduces per-system tool catalogue maintenance burden.

The paper highlights MCP as the inflection point: before MCP, every DR agent had its own tool registry, with redundant integrations across systems. After MCP, tool servers can be shared across agents (Claude Code, OpenClaw, custom agents) using the same standardised JSON-RPC protocol.

### Component 3 — Architecture and workflow

Already covered above. The static-vs-dynamic split, planning strategies, and single-vs-multi-agent axis.

The paper also covers **memory mechanisms** for handling extended contexts during DR runs — three strategies:

1. **Extending the context window** — Gemini's 1M-token context with RAG fallback. Brute-force; expensive; inefficient.
2. **Compressing intermediate steps** — AI Scientist, CycleResearcher, Search-o1 (Reason-in-Documents), WebThinker. Risks information loss.
3. **External structured storage** — Manus, OWL, Open Manus, Avatar (filesystem-based); AutoAgent (vector DB); Agentic Reasoning (knowledge graphs); Agent-KB and Alita (shared knowledge bases). MEMTIER ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)) is the natural extension of this strategy with a tiered, attribution-aware substrate.

### Component 4 — Tuning methodologies

The paper's Table 3 catalogues tuning strategies across DR agents:

- **Prompt-driven structured generation**: most production systems (OpenAI DR, Gemini DR, Grok, Manus) — no model fine-tuning.
- **SFT (Supervised Fine-Tuning)**: e.g., WebWalker.
- **RL via PPO**: Agent-R1, Search-R1, SimpleDeepSearcher.
- **RL via GRPO**: ReSearch, R1-Searcher, Search-R1, DeepResearcher, PANGU DeepDiver, Tool-Star.
- **RL via DAPO**: WebDancer.
- **RL via DUPO**: WebSailor.
- **RL via Online DPO**: WebThinker.
- **RL via REINFORCE++**: Agent-R1, R1-Searcher.
- **RL via REINFORCE**: Kimi-Researcher.
- **Offline RL**: SWIRL.

Reward designs split between:
- **Rule-based outcome reward** (binary correct/incorrect; majority of systems)
- **Process-based reward** (intermediate-step quality; SimpleDeepSearcher)

The paper anticipates the trend now playing out across [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md): heavy-thinking-aware RLVR will increasingly shift work from harness orchestration into trained model behaviour.

### Component 5 — Non-parametric continual learning

The newest section (added in v2, Sept 2025). Rather than updating model weights, agents *self-evolve* by adapting external tools, memory, and workflows. This is exactly the territory of [36-autogenesis-self-evolving-agents](36-autogenesis-self-evolving-agents.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md), [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md), [81-reasoningbank](81-reasoningbank.md), [109-memento-results-and-harness](109-memento-results-and-harness.md), and [154-ctx2skill-self-evolving-context-skills](154-ctx2skill-self-evolving-context-skills.md). The paper's framing connects them as a coherent research direction.

## The protocol stack — MCP and A2A

Section 2.3 introduces two complementary protocols that the paper argues are foundational to the future of DR agents:

- **MCP (Model Context Protocol, Anthropic)**: a standardised tool-server interface. Solves the data-silo and redundant-development problems of bespoke tool integrations. MCP servers expose tools via JSON-RPC; agents discover and invoke them through a uniform interface. Covered in detail at [07-model-context-protocol](07-model-context-protocol.md).
- **A2A (Agent-to-Agent, Google)**: a standardised inter-agent communication layer. Abstracts agent discovery into "Agent Cards" and task coordination into "Tasks and Artefacts". Enables agents from different vendors / model families to collaborate as equals.

The paper's framing: **MCP and A2A are complementary, not competing**. MCP is the interface to *external tools*; A2A is the interface to *other agents*. Together, they form the modular foundation for open, interoperable agent ecosystems. By 2026, we see this play out in DeerFlow 2.0 ([163-deer-flow-revisited](163-deer-flow-revisited.md)), CrewAI ([164-crewai](164-crewai-multi-agent-framework.md)), and Feynman ([155-feynman-multi-agent-research-harness](155-feynman-multi-agent-research-harness.md)) all integrating MCP-compatible tool servers and (more cautiously) A2A-style multi-agent orchestration.

## The benchmark critique

Section 5 of the paper makes a substantive critique of current DR benchmarks. Three failures it identifies:

1. **Restricted external knowledge access**. Most benchmarks operate over a frozen corpus snapshot (Wikipedia, GAIA tasks, GPQA-Diamond). DR agents in production work over the *live* web. Benchmarks underestimate the benefit of real-time browsing and over-credit static-corpus retrieval.
2. **Sequential execution inefficiency**. Benchmarks score correctness, not parallelism. A DR agent that issues 20 sequential queries scores the same as one that issues 20 parallel queries, despite vastly different real-world performance.
3. **Misalignment between metrics and practical objectives**. End-task accuracy on QA-style benchmarks is a poor proxy for the actual deliverable of a DR agent — a structured, citation-rich, faithfully-grounded report. Citation accuracy, structural coherence, and source diversity are not measured.

The paper's recommendation: future DR benchmarks should evaluate (a) live-corpus interaction, (b) async parallel execution efficiency, (c) report-quality dimensions (citation faithfulness, structure, balance). [101-autoresearchbench](101-autoresearchbench.md) and DeepResearch Bench (cited in [159](159-deep-research-survey-zhang-et-al.md)) are early attempts to address (c).

## The roadmap

Section 6 names four research directions:

1. **Expanding retrieval scope** — beyond text-on-the-web. Multi-modal retrieval (video, audio, scientific datasets, structured data sources) is the obvious next territory.
2. **Asynchronous parallel execution** — most current DR agents run sequential tool calls. Async execution can compress wall-clock time by 5–10× on retrieval-heavy tasks but requires coordination primitives (the paper points at [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md)-style durable execution as a candidate).
3. **Comprehensive multi-modal benchmarks** — addressing the limitations above, especially for image-heavy or video-heavy research tasks.
4. **Optimising multi-agent architectures** — improving robustness and efficiency of multi-agent DR systems, particularly through hybrid centralised-distributed coordination (closer to A2A's vision than current implementations).

## Industry implementations covered

Section 4 walks through ~15 representative industry DR systems. Highlights worth pulling out for the harness-engineering canon:

| System | Provenance | Distinctive feature |
|--------|-----------|----------------------|
| **OpenAI DR** | OpenAI, Feb 2025 | GPT-o3 backbone; intent-to-planning; end-to-end RL |
| **Gemini DR** | Google, Dec 2024 | 1M-token context; unified intent-planning; user-confirmable plan |
| **Grok DeepSearch** | xAI, Feb 2025 | Grok 3; real-time X integration; planning-only |
| **Perplexity DR** | Perplexity, Feb 2025 | Flexible model selection; SimoleQA benchmark focus |
| **Manus** | Manus AI, Mar 2025 | Hierarchical planner-toolcaller; Claude 3.5 + GPT-4o |
| **Genspark Super Agent** | Apr 2025 | Mixture-of-agents architecture |
| **Towards an AI Co-Scientist** | Google, Feb 2025 | Gemini 2.0; multi-agent scientist collaboration |
| **AgentRxiv** | Mar 2025 | arXiv-mimicking shared knowledge repository for agents |
| **AutoGLM Rumination** | Mar 2025 | GLM-Z1-Air; Chinese-market DR |
| **AutoAgent** | Feb 2025 | Self-managing modules with vector DB memory |
| **Manus + OpenManus** | Mar 2025 | Hierarchical planner + tool-caller architecture |
| **OWL** | 2025 | Workforce-oriented; central manager + execution agents |
| **Alita** | 2025 | Self-evolution with online MCP server instantiation |
| **AWorld** | 2025 | Open-source build/orchestrate/train framework |
| **Kimi-Researcher** | Moonshot, Jun 2025 | Kimi k1.5/k2 backbone; REINFORCE training |

The pattern across these systems: **dynamic workflow + multi-agent + browser-based search + RL training is becoming the default architecture for production DR**. Static workflows survive in research settings (AI Scientist, Agent Laboratory) where structure aids reproducibility.

## What this paper changes about how to build DR agents

Five practical takeaways for builders, derived from the survey:

### 1. Choose your workflow type early — it determines your training options

Static workflows are easy to start with but hard to scale. Dynamic workflows are harder to build but enable end-to-end RL training. If you intend to train your agent (rather than only prompt-engineer it), commit to a dynamic single-agent architecture from day one — that's what gets you GRPO/DAPO/DUPO-trainable.

### 2. Pick your planning strategy based on user trust

- *Planning-Only* (Grok-style) when the user trusts the system and wants speed.
- *Intent-to-Planning* (OpenAI DR-style) when the user's request is ambiguous and clarification is cheap.
- *Unified Intent-Planning* (Gemini DR-style) when the cost of a wrong plan is high and the user wants approval rights.

The choice has UX consequences as much as technical ones. See [143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md).

### 3. Browser AND API; not browser OR API

The strongest production systems (OpenAI DR, Gemini DR, Manus) use both modalities. APIs for structured high-volume retrieval; browser for paywalled, dynamic, or behind-login content. The paper's Table 1 shows that the most-capable systems do not pick one — they pick a hybrid orchestration.

### 4. Memory is a load-bearing architectural concern, not an afterthought

The three-strategy decomposition in §3.3.4 (extend context / compress / external storage) corresponds directly to the architectural debate that MEMTIER ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)) settles. *External structured storage wins for long-running DR agents.*

### 5. MCP is the right interoperability bet

The paper's framing of MCP as the standardised interface to tools — and A2A as the complementary inter-agent protocol — is the architectural vision that matters. Build your DR agent on MCP-compatible tool servers from day one; it's the easiest way to make your agent interoperable with the rest of the ecosystem.

## How this paper relates to the rest of the May-2026 canon

This survey was written before the May-2026 papers landed. Reading it with hindsight:

- The survey *anticipates* MEMTIER's architectural argument (§3.3.4 on memory mechanisms): structured external storage is named as one of three strategies, but the relative-merit comparison is unsettled. MEMTIER ([153-memtier-llm-distillation-and-the-three-invariants](153-memtier-llm-distillation-and-the-three-invariants.md)) settles it: structured storage with LLM-distilled tiers wins.
- The survey *frames* the problem space that HeavySkill addresses (§3.4 on tuning): RL-based DR-agent training is reviewed across PPO/GRPO/DAPO. HeavySkill ([156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md)) provides the next move: distil the inner reasoning skill into a portable artefact.
- The survey *predicts* Ctx2Skill's contribution (§3.5 on non-parametric continual learning): self-evolving agents that adapt external memory and skills are highlighted as a major direction. Ctx2Skill ([154-ctx2skill-self-evolving-context-skills](154-ctx2skill-self-evolving-context-skills.md)) provides the concrete adversarial-self-play mechanism for skill construction without external feedback.
- The survey *previews* Feynman's design (§3.3 on multi-agent + memory + structured output): file-based handoffs, source-grounded outputs, multi-agent isolation — Feynman ([155-feynman-multi-agent-research-harness](155-feynman-multi-agent-research-harness.md)) is the operationalisation.

In other words: Huang et al. is the *map*. The May-2026 papers are specific *territory* covered by that map.

## Limitations of the survey itself

A few honest critiques:

1. **Snapshot dating**. The survey was submitted in June 2025 and revised in September. It was already partially out of date when published — Memento ([106-109](106-memento-paper-theory.md)), MEMTIER, Ctx2Skill, HeavySkill are all unrepresented.
2. **Industrial-system bias toward Western frontier labs**. OpenAI, Google, xAI, Anthropic, Perplexity, Manus get extensive coverage. Chinese systems (AutoGLM, Kimi-Researcher, GLM, ByteDance DeerFlow) are noted but less deeply analysed.
3. **Single-agent vs multi-agent treatment is asymmetric**. The single-agent section is much richer, with detailed RL-tuning analysis. The multi-agent section is more cataloguing than analysis. This reflects the field's evolution — single-agent systems with good RL are easier to study rigorously than multi-agent systems with opaque coordination dynamics.
4. **Benchmark critique is incisive but limited in scope**. The paper diagnoses what's wrong with current benchmarks but does not propose a rigorous replacement. DeepResearch Bench ([16] cited in the second survey [159](159-deep-research-survey-zhang-et-al.md)) and AutoResearchBench ([101](101-autoresearchbench.md)) make initial proposals.

## Where this fits in the canon

- **Read alongside**: [159-deep-research-survey-zhang-et-al](159-deep-research-survey-zhang-et-al.md) — the complementary capability-centric survey, with planning/question-developing/web-exploration/report-generation pipeline framing.
- **Read after**: [25-agentic-rag](25-agentic-rag.md) — the RAG ancestor of DR agents.
- **Practical implementations**: [29-dive-into-claude-code](29-dive-into-claude-code.md), [42-langchain-deep-agents](42-langchain-deep-agents.md), [52-dive-into-open-claw](52-dive-into-open-claw.md), [61-archon-harness-builder](61-archon-harness-builder.md), [63-ragflow-agent-patterns](63-ragflow-agent-patterns.md), [65-deer-flow-bytedance](65-deer-flow-bytedance.md), [66-meta-harness-landscape](66-meta-harness-landscape.md), [155-feynman-multi-agent-research-harness](155-feynman-multi-agent-research-harness.md), [163-deer-flow-revisited](163-deer-flow-revisited.md), [164-crewai-multi-agent-framework](164-crewai-multi-agent-framework.md).
- **Memory architectures**: [09-memory-files](09-memory-files.md), [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md), [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md), [130-distributed-sql-as-agent-memory](130-distributed-sql-as-agent-memory.md), [151-memtier-why-flat-memory-breaks-at-72-hours](151-memtier-why-flat-memory-breaks-at-72-hours.md), [152-memtier-3-tier-architecture-and-retrieval](152-memtier-3-tier-architecture-and-retrieval.md), [153-memtier-llm-distillation-and-the-three-invariants](153-memtier-llm-distillation-and-the-three-invariants.md).
- **Self-evolving agents**: [36-autogenesis-self-evolving-agents](36-autogenesis-self-evolving-agents.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md), [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md), [81-reasoningbank](81-reasoningbank.md), [109-memento-results-and-harness](109-memento-results-and-harness.md), [154-ctx2skill-self-evolving-context-skills](154-ctx2skill-self-evolving-context-skills.md).
- **Skills as portable artefacts**: [04-skills](04-skills.md), [19-voyager-skill-libraries](19-voyager-skill-libraries.md), [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md), [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md).
- **Synthesis**: [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md), [166-research-agent-frameworks-synthesis](166-research-agent-frameworks-synthesis.md).

## References

1. Huang et al. 2025. *Deep Research Agents: A Systematic Examination And Roadmap*. arXiv:2506.18096v2.
2. Curated repository: https://github.com/ai-agents-2030/awesome-deep-research-agent.
3. Zhang et al. 2025. *Deep Research: A Survey of Autonomous Research Agents*. arXiv:2508.12752 — companion survey with capability-centric pipeline framing; deep-dive at [159](159-deep-research-survey-zhang-et-al.md).
4. Zhang. 2026. *Deep Researcher Agent: An Autonomous Framework for 24/7 Deep Learning Experimentation*. arXiv:2604.05854 — operational deep-dive at [160](160-deep-researcher-agent-24x7.md).
5. Industrial DR systems referenced: OpenAI DR [78], Gemini DR [33], Grok DeepSearch [124], Perplexity DR [81], Manus [66], OpenManus [60], OWL [12], Alita [84], AWorld [8], Kimi-Researcher [70], DeepResearcher [135], Search-o1 [58], R1-Searcher [96], WebDancer [120], WebSailor [57], PANGU DeepDiver [94], WebThinker [59], WebWalker [121], MindSearch [11], Memento [138], etc.
6. Protocols: MCP (Anthropic), A2A (Google).
7. RAG ancestors: Lewis et al. 2020, FLARE [133], Self-RAG [7], IAG [134].
