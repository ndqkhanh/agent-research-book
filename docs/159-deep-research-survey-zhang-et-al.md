# 159 — Deep Research: A Survey of Autonomous Research Agents (Zhang et al., 2025)

**Paper.** Wenlin Zhang, Xiaopeng Li, Yingyi Zhang, Pengyue Jia, Yichao Wang, Huifeng Guo, Yong Liu, Xiangyu Zhao† — *Deep Research: A Survey of Autonomous Research Agents* — arXiv:2508.12752v1 [cs.IR] — submitted 18 August 2025. Affiliations: **City University of Hong Kong** (Wenlin Zhang, Xiaopeng Li, Pengyue Jia, Xiangyu Zhao), **Dalian University of Technology + CityU HK** (Yingyi Zhang), **Huawei Noah's Ark Lab** (Yichao Wang, Huifeng Guo, Yong Liu). Published in the ACM proceedings format.

**One-line definition.** This is the *capability-centric* companion to the Huang et al. system-architecture survey ([158-deep-research-agents-survey-huang-et-al](158-deep-research-agents-survey-huang-et-al.md)) — where Huang et al. organise DR agents by *system architecture* (workflows, planning strategies, single/multi-agent), Zhang et al. organise them by the **four-stage capability pipeline**: **Planning → Question Developing → Web Exploration → Report Generation**. The contribution is a fine-grained, modular decomposition of DR competencies, treating each stage as an independently optimisable capability with its own taxonomy of methods (reward-optimised vs supervision-driven), benchmarks, and bottlenecks. Together with [158](158-deep-research-agents-survey-huang-et-al.md), this provides the canonical *vocabulary and pipeline structure* for the DR-agent field as of 2025.

## Why two surveys are better than one

The two August/June 2025 surveys are not duplicative. They cut the same cake along different axes:

| Survey | Cut | Best for |
|--------|-----|----------|
| Huang et al. ([158](158-deep-research-agents-survey-huang-et-al.md)) | System architecture: workflows, planning strategies, agent composition, tools, tuning | *How the agent is built* |
| Zhang et al. (this) | Capability pipeline: planning, question developing, web exploration, report generation | *What capabilities the agent must have* |

Reading Huang et al. tells you what design decisions to make. Reading Zhang et al. tells you what bottlenecks each stage of the pipeline faces and which optimisation techniques exist for each. Both surveys cite each other; the field treats them as a complementary pair.

## The four-stage pipeline

Zhang et al. define the DR pipeline as four interconnected stages, each with a precise problem definition:

```
User Query
   │
   ▼
┌──────────────┐  Definition 2.1 (Planning):
│  PLANNING    │   P = M_plan(q_0, K; θ)
│              │   where P = [s_1, s_2, ..., s_n] is the sub-goal sequence
└──────────────┘
   │
   ▼
┌──────────────────────┐  Definition 3.1 (Question Developing):
│ QUESTION DEVELOPING  │   Q_i = M_ask(P, s_i, E; θ)
│                      │   for each subgoal s_i, given accumulated evidence E
└──────────────────────┘
   │
   ▼
┌──────────────────┐  Definition 4.x (Web Exploration):
│ WEB EXPLORATION  │   E_t+1 = Retrieve(Q_i, web; environment)
│                  │   iterative; tool-mediated; produces accumulated evidence
└──────────────────┘
   │
   ▼
┌─────────────────────┐  Definition 5.x (Report Generation):
│ REPORT GENERATION   │   R = M_report(E, plan, query)
│                     │   structured + factually integrity-preserving
└─────────────────────┘
   │
   ▼
Structured analytical report
```

This is more than a flowchart. The mathematical definitions matter because the survey shows that *each stage can be independently optimised* with its own training signal. That modular decomposition is the survey's main intellectual contribution.

## Stage 1 — Planning

**Definition 2.1**: planning transforms a user query q_0 and agent context K into a structured plan P = [s_1, …, s_n] of sub-goals or tool-invocation steps via a planning model M_plan parameterised by θ.

The survey's Table 1 categorises planning methods along two axes:

```
Planning with Structured World Knowledge:
    ├── Planning via simulation
    │       WebDreamer, Simulate Before Act
    ├── Planning via modularity
    │       WebPilot (Zhang et al.), WKM, Plan-and-Act (Erdogan et al.)
    ├── Planning via adaptation
    │       Thought of Search, MPO (meta-plan optimisation)
    └── Planning via space exploration
            AgentSquare, Agent-E (Abuelsaad et al.)

Planning as a Learnable Process:
    ├── Planning via self-training
    │       Patel et al., InSTA (Trabucco et al.)
    └── Planning via preference modeling
            MindSearch, SimpleDeepSearcher,
            Search-in-the-Chain, WEPO, MPO
```

Two important observations from §2.3 (Discussion):

1. **Brittleness**: even with structured formats, plans hallucinate steps and propagate errors downstream. Internal consistency is not guaranteed.
2. **Coarse evaluation**: planning is judged by end-task accuracy rather than plan-quality directly. This makes it hard to diagnose planning failures or compare strategies meaningfully. (DeepResearch Bench is named as an early attempt to address this.)
3. **No transfer**: most systems treat each query as isolated, without reusing transferable planning strategies across tasks.

These three failure modes are the binding constraints for planning quality in 2025. They map onto: (a) the need for verification gates ([155-feynman-multi-agent-research-harness](155-feynman-multi-agent-research-harness.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md)); (b) the need for plan-quality metrics ([115-evaluating-llm-systems](115-evaluating-llm-systems.md)); and (c) the case for skill libraries that *do* transfer ([04-skills](04-skills.md), [154-ctx2skill-self-evolving-context-skills](154-ctx2skill-self-evolving-context-skills.md)).

## Stage 2 — Question Developing

**Definition 3.1**: given subgoal s_i and accumulated evidence E, question developing produces a set of search queries Q_i = M_ask(P, s_i, E; θ).

This is the stage that has seen the most aggressive RL training in 2025. Table 2 categorises by optimisation method:

```
Reward-Optimized Methods:
    ├── Rewards for format and accuracy
    │       DeepResearcher, EvolveSearch, R1-Searcher, Search-R1,
    │       ZeroSearch, MaskSearch, DeepRetrieval
    └── Multi-dimensional reward
            InForage, OTC-PO, IKEA, AutoRefine,
            R-Search, MMSearch-R1, VRAG-RL

Supervision-Driven Methods:
    ├── Multi-agent systems
    │       ManuSearch, Search-o1, SearchAgent-X
    └── Supervision optimization
            ReasonRAG (uses MCTS + DPO)
```

The split between reward-optimised and supervision-driven matters because of *what training data you can construct*:

- **Reward-optimised**: needs an outcome signal (correct/incorrect answer; F1; recall@k). Cheaper data; more variance; better suits noisy or simulated environments.
- **Supervision-driven**: needs labeled query trajectories (human demonstrations or rule-based templates). More controllable; higher quality; but data is expensive and limited.

Several specific contributions worth pulling out:

- **Search-R1**: enforces strict format templates (`<think>`, `<search>`, `<information>`, `<answer>`) and trains with PPO/GRPO. The structured reasoning template has become a de facto standard.
- **ZeroSearch**: replaces real search APIs with a *learned simulator* during training. Eliminates per-query API cost during RL; enables curriculum rollout.
- **OTC-PO** (Optimal-Tool-Cost Policy Optimisation): rewards tool-call *minimisation*, penalising unnecessary external calls. This is the cost-aware reward design that makes RL-trained DR agents economically viable.
- **IKEA**: introduces *knowledge-boundary-aware* reward — rewards solving "easy" questions using internal knowledge alone, penalises unnecessary external searches. Trains the agent's *meta-cognition* about its own knowledge.
- **AutoRefine**: rewards completeness of intermediate refinements from retrieved documents, not just final-answer quality. Process-aware reward.
- **MMSearch-R1, VRAG-RL**: extend the framework to multi-modal (image/video) search.

The trend the survey identifies: from binary outcome rewards in early 2025 (DeepResearcher, R1-Searcher) toward multi-dimensional process-aware rewards by mid-2025 (InForage, OTC-PO, IKEA). The next move (Q4 2025–2026) is heavy-thinking-aware RLVR ([156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md)) and self-evolving skills ([154-ctx2skill-self-evolving-context-skills](154-ctx2skill-self-evolving-context-skills.md)).

## Stage 3 — Web Exploration

This stage handles the actual interaction with external sources. The survey covers two implementation modalities:

### API-based methods

Direct calls to search-engine APIs (Google CSE, Bing, Brave, You.com, Tavily, Exa, Perplexity API). Pros: fast, deterministic, parseable JSON results, easy to rate-limit. Cons: limited to indexed content, no rendered content, no behind-login access.

### Browser-based methods

Programmatic browser control via Playwright, Stagehand, or LLM-driven UI agents like browser-use, Skyvern, Steel Browser. Pros: complete web access including dynamic JS, paywalled content, interactive widgets. Cons: brittle, expensive, slow, requires visual reasoning.

The survey notes that production DR systems increasingly use *hybrid* orchestration — APIs for high-volume structured queries, browser for the long-tail of dynamic content. This matches Huang et al.'s observation in [158](158-deep-research-agents-survey-huang-et-al.md).

### The iterative-search pattern

What distinguishes DR-stage web exploration from one-shot RAG retrieval is iteration. The agent issues a query, parses results, identifies gaps, formulates new queries, and continues. The survey emphasises the pattern as *agent-driven* — query strategy is decided by the agent at each step, not pre-templated.

This is the territory where ReAct ([13-react](13-react.md)) lives, but DR exploration is more sophisticated: it tracks accumulated evidence E, prunes redundant retrievals, deduplicates sources, and cross-validates claims across sources.

## Stage 4 — Report Generation

The final stage transforms accumulated evidence into a structured analytical report. The survey identifies two distinct technical concerns:

### Structure Control

The challenge: generate reports with consistent layout, hierarchical sections, balanced coverage across topics, and adherence to the original plan's sub-goals.

Methods include:
- **Layout templates**: pre-defined section structures (e.g., introduction / methods / findings / conclusion) that the agent fills in.
- **Plan-conditioned generation**: the agent generates each section conditioned on the corresponding sub-goal from the planning stage.
- **Hierarchical decomposition**: the agent generates an outline first, then fills in subsections, then verifies coverage.

### Factual Integrity

The bigger challenge: ensure the report is *grounded* in the accumulated evidence and does not hallucinate facts.

Techniques include:
- **Inline citation enforcement**: every factual claim must reference a specific evidence source. (See Feynman's mandatory Verifier subagent at [155-feynman-multi-agent-research-harness](155-feynman-multi-agent-research-harness.md).)
- **Evidence-grounded generation**: the generator is constrained to draw from the retrieved evidence rather than parametric knowledge.
- **Cross-source verification**: claims that appear in multiple sources are upweighted; single-source claims are flagged.
- **Citation faithfulness checking**: a separate verification pass confirms that each citation actually supports the claim attached to it. (See [135-trustworthy-generation](135-trustworthy-generation.md).)

The survey's framing ties this to factual hallucination as the central failure mode. Modern DR systems treat factual integrity as a structural concern (a separate verification stage), not a generation guideline.

## Optimisation techniques surveyed

Section 6 of the paper reviews optimisation across the pipeline. Three families:

### RL-based optimisation

- **PPO** (Proximal Policy Optimisation): the most common; used by Agent-R1, Search-R1, SimpleDeepSearcher.
- **GRPO** (Group Relative Policy Optimisation): more compute-efficient than PPO; used by ReSearch, R1-Searcher, DeepResearcher, PANGU DeepDiver, Tool-Star.
- **DAPO**: WebDancer.
- **DUPO**: WebSailor.
- **REINFORCE / REINFORCE++**: Kimi-Researcher, Agent-R1.
- **Online DPO (Iterative)**: WebThinker.

### SFT-based optimisation

Training on labeled trajectories. Used by WebWalker, ReasonRAG (with MCTS-based trajectory sampling and DPO ranking), and many supervised baselines.

### Hybrid optimisation

Multi-stage curricula that begin with SFT for stable behaviour and then switch to RL for outcome-driven refinement. R1-Searcher exemplifies this: phase 1 rewards format compliance and search usage; phase 2 integrates F1-based answer quality.

## Benchmarks specific to DR

The survey reviews DR-specific benchmarks. The most prominent:

- **DeepResearch Bench** [16]: evaluates report fidelity and citation accuracy. The first benchmark designed specifically for DR-style outputs (rather than QA).
- **WebWalkerQA** [121]: web-navigation questions over real web environments.
- **GAIA** [38]: general AI assistant evaluation; widely used as a DR benchmark even though it's not DR-specific.
- **HLE (Humanity's Last Exam)**: extreme-difficulty questions requiring deep research.
- **GPQA, GPQA-Diamond**: graduate-level science questions.
- **MLE-Bench, MLAgentBench**: machine-learning-research-specific benchmarks.

Each benchmark has different strengths. None of them, the survey argues, fully captures *report-quality* DR — they tend to score answer correctness rather than citation fidelity, structural coherence, or evidence grounding.

## How this differs from Huang et al.

Reading the two surveys side by side reveals different choices in framing:

| Aspect | Huang et al. ([158](158-deep-research-agents-survey-huang-et-al.md)) | Zhang et al. (this) |
|--------|-----------|-----------|
| Primary axis | System architecture | Capability pipeline |
| Definition focus | What constitutes a DR agent (the category) | What stages a DR agent must implement |
| Multi-agent treatment | First-class taxonomy axis | Mostly under "Question Developing — multi-agent systems" |
| RL coverage | Tabulated across systems (Table 3) | Stage-specific (PPO/GRPO/DAPO under Question Developing §3) |
| Memory | Section 3.3.4 (extend / compress / external storage) | Less prominent; partial under Web Exploration |
| MCP / A2A | Dedicated section 2.3 | Mentioned but not central |
| Benchmarks | Section 5 with limitations critique | Section 6 with optimisation techniques |
| Industry systems | Section 4 (~15 systems) | Mostly cited inline within stage discussions |

The two surveys are best read as **architecture (Huang) + capability (Zhang) = comprehensive picture**.

## What this survey predicts about the field's evolution

Three evolutionary trends Zhang et al. names in §1 and the closing discussions:

1. **Reasoning-driven retrieval**. Retrieval becomes part of reasoning, not a precursor to it. The agent reasons about *what to retrieve*, *why*, and *what comes next* in an integrated loop. This is the territory of agentic RAG and *deep search*.
2. **Structured report generation**. Reports become structured artefacts (with sections, citations, balance) rather than narrative dumps. This requires layout-aware generation and integrity-preserving citation mechanisms.
3. **Self-evolving agents**. Agents that improve their planning and question-developing strategies over time without human supervision. This anticipates [36-autogenesis-self-evolving-agents](36-autogenesis-self-evolving-agents.md), [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md), [154-ctx2skill-self-evolving-context-skills](154-ctx2skill-self-evolving-context-skills.md).

All three trends have intensified in 2026. The capability-centric framing has aged well precisely because each capability is independently improvable and trainable.

## Practical takeaways for builders

If you are building a DR agent in 2025–2026, the Zhang et al. framing suggests:

### 1. Implement the four stages as separate modules

Even if you start with a monolithic prompt that does everything, factor it into Planning / Question Developing / Web Exploration / Report Generation as soon as you can. This lets you:

- Replace any stage independently.
- Train any stage independently with stage-specific reward signals.
- Compose models across stages (e.g., a strong planner + a fast question developer + a careful generator).

### 2. Train question-developing first

The richest body of training techniques in 2025 is on question developing. PPO and GRPO with format + accuracy rewards is the well-trodden path; OTC-PO and IKEA add cost and knowledge-boundary awareness.

### 3. Treat report generation integrity as a structural concern

Don't bury "make the report faithful" inside a generation prompt. Build a verification stage that *can remove unsourced claims*. Feynman's Verifier subagent ([155](155-feynman-multi-agent-research-harness.md)) is the cleanest open-source instantiation of this discipline.

### 4. Use both API and browser for web exploration

Hybrid orchestration is the production pattern. APIs for the bulk; browser for the long tail.

### 5. Iterate the planning stage from interaction feedback

Plans generated by current LLMs are brittle. Use interaction feedback (which sub-goals lead to dead-ends; which retrievals were useful) to refine planning behaviour. The MPO meta-planning approach in §2.1 generalises.

## Limitations of the survey itself

A few honest critiques:

1. **Web-exploration coverage is thinner than other stages**. Stage 4 (Web Exploration) gets less detailed taxonomic treatment than Stages 2 (Question Developing) and 1 (Planning). API-vs-browser is well covered but the iterative-search pattern's design space is not exhaustively mapped.
2. **Memory architecture is treated lightly**. Unlike Huang et al. §3.3.4, this survey does not give memory a dedicated stage or section. MEMTIER-class architectural arguments ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)) are not anticipated.
3. **Industrial systems are treated more cursorily**. OpenAI DR, Gemini DR, Manus, Kimi-Researcher are mentioned but not deeply analysed. Huang et al. provides better coverage on this dimension.
4. **The pipeline framing can suggest excessive linearity**. In practice, DR loops are *cyclic*: planning informs question developing, exploration results re-trigger planning, report generation reveals gaps that send the agent back to exploration. The survey's discussion acknowledges this but the pipeline diagram emphasises a clean linear flow.

## How this survey grounds the May-2026 papers

Reading Zhang et al. with hindsight from the May-2026 papers:

- The capability-centric pipeline framing predicts that *each capability becomes a separately trainable substrate*. This is exactly what HeavySkill ([156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md)) does for the parallel-reasoning capability and what Ctx2Skill ([154-ctx2skill-self-evolving-context-skills](154-ctx2skill-self-evolving-context-skills.md)) does for the context-learning capability.
- The reward-optimised vs supervision-driven taxonomy in Question Developing anticipates the reward-design contributions of OTC-PO and IKEA — and forecasts the trajectory toward verifiable rewards (RLVR) that culminates in HeavySkill's training of inner skills.
- The structure-control + factual-integrity framing in Report Generation aligns directly with Feynman's mandatory Verifier and slug-keyed provenance discipline ([155](155-feynman-multi-agent-research-harness.md)).
- The "self-evolving agents" trend mentioned in §1 is exactly the territory of [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md)'s "two trainable substrates" thesis.

The survey is six months early in pointing where the field is heading.

## Where this fits in the canon

- **Read alongside**: [158-deep-research-agents-survey-huang-et-al](158-deep-research-agents-survey-huang-et-al.md) — the architecture-centric companion.
- **Read after**: [25-agentic-rag](25-agentic-rag.md), [13-react](13-react.md), [16-plan-and-solve](16-plan-and-solve.md), [17-rewoo](17-rewoo.md).
- **Stage-specific deep-dives**:
  - Planning: [03-plan-mode](03-plan-mode.md), [16-plan-and-solve](16-plan-and-solve.md).
  - Question Developing: [25-agentic-rag](25-agentic-rag.md), [79-skill-rag](79-skill-rag.md).
  - Web Exploration: [34-clawbench-live-web-tasks](34-clawbench-live-web-tasks.md), [95-osworld](95-osworld.md).
  - Report Generation: [135-trustworthy-generation](135-trustworthy-generation.md), [155-feynman-multi-agent-research-harness](155-feynman-multi-agent-research-harness.md).
- **Optimisation**: [80-knowrl](80-knowrl.md), [97-qwen-prm](97-qwen-prm.md), [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md).
- **Self-evolving agents**: [36-autogenesis-self-evolving-agents](36-autogenesis-self-evolving-agents.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md), [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md), [81-reasoningbank](81-reasoningbank.md), [109-memento-results-and-harness](109-memento-results-and-harness.md), [154-ctx2skill-self-evolving-context-skills](154-ctx2skill-self-evolving-context-skills.md).
- **Synthesis**: [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md), [166-research-agent-frameworks-synthesis](166-research-agent-frameworks-synthesis.md).

## References

1. Zhang et al. 2025. *Deep Research: A Survey of Autonomous Research Agents*. arXiv:2508.12752.
2. Huang et al. 2025. *Deep Research Agents: A Systematic Examination And Roadmap*. arXiv:2506.18096v2 — companion architectural survey; deep-dive at [158](158-deep-research-agents-survey-huang-et-al.md).
3. Planning methods cited: WebDreamer [20], Simulate Before Act [21], WebPilot [96], WKM [52], Plan-and-Act [17], Thought of Search [32], MPO [83], AgentSquare [62], Agent-E [4], MindSearch [11], SimpleDeepSearcher [69], Search-in-the-Chain [85], WEPO [37], Patel et al. [45], InSTA [71].
4. Question developing methods cited: DeepResearcher [98], EvolveSearch [93], R1-Searcher [66], Search-R1 [31], ZeroSearch [68], MaskSearch [80], DeepRetrieval [30], InForage [51], OTC-PO [74], IKEA [28], AutoRefine [65], R-Search [97], MMSearch-R1 [77], VRAG-RL [75], ManuSearch [25], Search-o1 [35], SearchAgent-X [89], ReasonRAG [95].
5. Web exploration methods cited: API-based [1, 2, 6, 7] and browser-based [24, 41, 90, 101].
6. Benchmarks: DeepResearch Bench [16], WebWalkerQA, GAIA [38], HLE, GPQA, MLE-Bench, MLAgentBench.
7. Companion: [158-deep-research-agents-survey-huang-et-al](158-deep-research-agents-survey-huang-et-al.md), [160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md).
