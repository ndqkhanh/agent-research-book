# 319 - AI Agents 2026 Capstone: Deep Synthesis

**Scope.** A capstone synthesis for the broad AI Agents field, using [masamasa59/ai-agent-papers](https://github.com/masamasa59/ai-agent-papers) as the field-map spine and `references/AI Agents - Capstone Synthesis (Report #6).md` as the local synthesis reference. It integrates the prior memory, self-evolution, research-agent, and context-engineering chapters with high-signal gaps: tool use, computer-use, embodied agents, RL/post-training, safety, operations, and evaluation.

**Thesis.** By 2026, "AI agent" is no longer one architecture. It is a stack:

1. **Foundation model brain** - frozen, post-trained, or RL-trained.
2. **Context/memory middleware** - Write, Select, Compress, Isolate.
3. **Tool and protocol layer** - function calling, MCP, A2A, AGENTS.md, code actions.
4. **Embodiment shell** - text, web, desktop GUI, mobile, API, robot, enterprise workflow.
5. **Evaluation and safety envelope** - benchmarks, telemetry, policy gates, adversarial robustness.

Your existing reports cover the cognitive spine well: memory, self-evolution, AI research agents, and context engineering. The remaining frontier is where those cognitive techniques meet tools, unsafe inputs, computer-use, robotics, and production operations.

---

## 0. Reading Map

If you only have one hour:

1. **Start with the repo spine:** [masamasa59/ai-agent-papers](https://github.com/masamasa59/ai-agent-papers), a selective biweekly AI-agent paper index.
2. **Then the capstone stack:** frozen/post-trained model -> context engineering -> tools/protocols -> embodiment -> eval/safety.
3. **Then the gaps:** tool-use memory, adversarial self-evolving memory, GUI/computer-use context engineering.
4. **Then benchmarks:** AgentBench, GAIA, SWE-bench, OSWorld, WebArena, BFCL, AgentDojo, MemoryAgentBench, LoCoBench-Agent, tau-bench.
5. **Then repos:** AutoGPT, LangChain, browser-use, MetaGPT, OpenHands, AutoGen, Mem0, CrewAI, LlamaIndex, LangGraph, Letta, SWE-agent, OpenVLA/openpi, BrowserGym, OSWorld.

---

## 1. Repo Spine: `masamasa59/ai-agent-papers`

The repo describes an AI agent as an autonomous system powered by LLMs that perceives its environment, reasons through tasks, uses tools, and acts toward goals. Its curation stance is selective rather than exhaustive: papers are added when they introduce a distinctively new approach or concept.

| Field | Snapshot |
|---|---|
| Repo | [masamasa59/ai-agent-papers](https://github.com/masamasa59/ai-agent-papers) |
| Stars at fetch | GitHub page reported **1,375** stars |
| Curation style | weekly arXiv keyword search; novelty-biased rather than complete |
| Markers | 🔥 recommended, 📖 survey, ⚖️ benchmark |
| Most useful role | high-signal map of AI-agent categories and recency trends |

### Category Structure

| Pillar | Categories |
|---|---|
| Agent capabilities | environment, ideation, planning, reasoning, profile, perception, tool use and skills, self-correction, search, memory, self-evolution, safety, agent tuning, agent evaluation |
| Architecture | single-agent, multi-agent, agent-ops / UX |
| Applications | embodied, digital, GUI, web, mobile, software, data, research, API, deep research, agentic AI systems, enterprise, financial, multi-agent simulation/problem-solving |
| Presentations | tutorials and lectures |

### Recency Signal

The 2026 highlight stream is dense around:

- **Self-evolving agents:** CORAL, EvoSkills, SkillX, SkillClaw, SEA-Eval, Frontier-Eng, PolicyBank, EVOAGENT.
- **Skills:** SKILL0, SkillRL, SkillsBench, SkillFlow, SkillLearnBench, skill retrieval, skill structure.
- **Research/scientific agents:** EvoScientist, DeepXiv-SDK, HLER, AwesomeLit, SciVisAgentBench, FlowPIE.
- **Human-AI collaboration:** interactive deep research, clarification, interaction smells, invisible failures.
- **Memory:** SimpleMem, MEMRL, Active Context Compression, event-centric memory, executable/core memory.
- **Coding and terminal agents:** OpenDev, OpenClaw-RL, MetaClaw, tool-use evolution.

---

## 2. Unified AI Agent Stack

```text
L7  Goal / meta-cognition
    Why am I doing this? Should I continue, ask, verify, or stop?

L6  Orchestration
    single-agent, multi-agent, hierarchical, federated via A2A

L5  Context engineering
    Write, Select, Compress, Isolate

L4  Memory and experience
    working, episodic, semantic, procedural, skills, policy banks

L3  Planning and reasoning
    ReAct, Reflexion, ToT, LATS, ReWOO, MCTS, verifiers

L2  Tools and protocols
    function calling, code actions, MCP, A2A, AGENTS.md, API/tool schemas

L1  Perception and embodiment
    text, web DOM, screenshots, desktop GUI, mobile, robot sensors/actions

L0  Foundation model
    frozen, SFT/DPO tuned, RLVR/agentic-RL trained, distilled
```

The older ReAct loop still matters, but it is now the inner loop inside broader harnesses, tool protocols, memory systems, and evaluation envelopes.

---

## 3. Coverage Map Against Prior Reports

| Area | Prior coverage | Current gap |
|---|---|---|
| Memory | strong: MemGPT/Letta, Mem0, Zep, A-Mem, LightMem, ReasoningBank, Memento | memory under tool/API distribution shift |
| Self-evolution | strong: skills, ReasoningBank, ACE, DGM, Memento | adversarial and faithfulness evaluation |
| AI research agents | strong: AI Scientist, Agent Laboratory, PaperQA2, DeepResearch, AgentRxiv | lab-in-loop and production evals |
| Context engineering | strong: Write/Select/Compress/Isolate, ACE, LoCoBench-Agent, DCI | GUI/computer-use context engineering |
| Tool use | partial | BFCL-style long-horizon tool memory and MCP quality |
| Web/GUI/computer-use | thin | OSWorld/WebArena/BrowserGym/Agent-S-style agents |
| Embodied/robotics | thin | VLA memory, skills, and safety |
| RL/post-training | partial | agentic RL beyond math/code into tools and web |
| Safety/adversarial | partial | prompt injection in memory, skills, MCP, self-evolution |
| Agent ops/evaluation | partial | production telemetry, SLOs, cost, reliability |

---

## 4. Master Comparison Table

### 4.1 Surveys and Foundations

| Work | Year | Category | Core idea | Why it matters |
|---|---:|---|---|---|
| [Survey on LLM-based Autonomous Agents](https://arxiv.org/abs/2308.11432) | 2023/2024 | survey | profile, memory, planning, action taxonomy | early field structure |
| [Rise and Potential of LLM Based Agents](https://arxiv.org/abs/2309.07864) | 2023 | survey | brain-perception-action framing | broad conceptual anchor |
| [Agentic Reasoning for LLMs](https://arxiv.org/abs/2601.12538) | 2026 | reasoning survey | CoT -> ToT -> ReAct -> Reflexion -> LATS | reasoning lineage |
| [Memory in the Age of AI Agents](https://arxiv.org/abs/2512.13564) | 2025 | memory survey | forms, functions, dynamics | memory taxonomy anchor |
| [Adaptation of Agentic AI](https://arxiv.org/abs/2512.16301) | 2025 | adaptation survey | post-training, memory, and skills | unifies self-evolution stack |
| [Deep Research: A Systematic Survey](https://arxiv.org/abs/2512.02038) | 2025 | research agents | deep-research systems map | AI research-agent anchor |
| [Measuring Agents in Production](https://arxiv.org/abs/2512.04123) | 2025 | agent-ops | production telemetry and measurement | shift from demos to ops |
| [Towards a Science of Scaling Agent Systems](https://arxiv.org/abs/2512.08296) | 2025 | scaling | scaling laws for agent systems | agent-era scaling frame |

### 4.2 Planning and Reasoning

| Work / repo | Core mechanism | Benchmark / evidence | Role in stack |
|---|---|---|---|
| [ReAct](https://arxiv.org/abs/2210.03629) / [ysymyth/ReAct](https://github.com/ysymyth/ReAct) | interleave reasoning and acting | ALFWorld, WebShop | inner loop primitive |
| [Tree of Thoughts](https://arxiv.org/abs/2305.10601) / [tree-of-thought-llm](https://github.com/princeton-nlp/tree-of-thought-llm) | search over thought tree | Game of 24, puzzle tasks | inference-time search |
| [Reflexion](https://arxiv.org/abs/2303.11366) / [noahshinn/reflexion](https://github.com/noahshinn/reflexion) | verbal feedback into memory | ALFWorld, HumanEval | self-correction ancestor |
| [LATS](https://arxiv.org/abs/2310.04406) | tree search + reflection | HotpotQA, web tasks | expensive but expressive planning |
| [ReWOO](https://arxiv.org/abs/2305.18323) | decouple planning from observations | tool tasks | lowers tool-call waste |
| [Plan-and-Solve](https://arxiv.org/abs/2305.04091) | plan before solving | reasoning benchmarks | basic planning decomposition |

### 4.3 Tool Use, Skills, and Protocols

| Work / repo | Core mechanism | Benchmark / evidence | Why it matters |
|---|---|---|---|
| [Toolformer](https://arxiv.org/abs/2302.04761) | self-supervised API insertion | QA/math/tool tasks | tool-use learning origin |
| [ToolLLM / ToolBench](https://arxiv.org/abs/2307.16789) | large API corpus and tool-use benchmark | 16k+ APIs | long-tail tool use |
| [Gorilla](https://arxiv.org/abs/2305.15334) | retrieval-aware API fine-tuning | API call tasks | API-grounded tool calls |
| [BFCL](https://gorilla.cs.berkeley.edu/leaderboard.html) | function-call benchmark | multi-turn function-calling emphasis | modern tool eval anchor |
| [UniToolCall](https://arxiv.org/abs/2604.11557) | unified Query-Action-Observation-Answer format | 22k+ tools, 390k+ instances | comparable tool-learning experiments |
| [Evolution of Tool Use in LLM Agents](https://arxiv.org/html/2603.22862v2) | survey of single-tool to multi-tool orchestration | literature synthesis | tool-use roadmap |
| [CodeAct](https://arxiv.org/abs/2402.01030) / [code-act](https://github.com/xingyaoww/code-act) | executable code as action language | multi-task agent suite | bridge to programmatic tool use |
| [MCP](https://modelcontextprotocol.io/) | client-server tool protocol | ecosystem standard | vertical tool interface |
| [A2A](https://github.com/a2aproject/A2A) | agent-to-agent protocol | LF ecosystem | horizontal delegation interface |
| [AGENTS.md](https://github.com/openai/agents-md) | project instruction convention | OSS adoption | agent-readable project context |

### 4.4 Multi-Agent Systems

| Work / repo | Core mechanism | Strength | Weakness |
|---|---|---|---|
| [CAMEL](https://github.com/camel-ai/camel) | role-play agents | early multi-agent reference | conversation drift |
| [AutoGen](https://github.com/microsoft/autogen) | conversational multi-agent with code execution | flexible prototyping | group-chat context clash |
| [MetaGPT](https://github.com/FoundationAgents/MetaGPT) | SOP-driven software-company roles | structured artifacts | heavy role machinery |
| [ChatDev](https://github.com/OpenBMB/ChatDev) | software company via cascading chats | accessible demo system | brittle for production |
| [CrewAI](https://github.com/crewAIInc/crewAI) | role-based crews and tasks | popular production-facing framework | quality depends on role/task design |
| [Magentic-One](https://github.com/microsoft/autogen/tree/main/python/packages/autogen-magentic-one) | orchestrator plus specialists | strong research baseline | proprietary model dependence in examples |
| [MASAI](https://openreview.net/forum?id=NSINt8lLYB) | specialized SWE subagents | context isolation by task | SWE-specific |
| [AGENT*](https://openreview.net/forum?id=lifeoGrKRB) | reusable collaboration modules | budget-aware collaboration | submitted / less reproduced |

### 4.5 Web, GUI, Mobile, and Computer-Use

| Work / repo | Domain | Core idea | Benchmark / evidence |
|---|---|---|---|
| [WebArena](https://github.com/web-arena-x/webarena) | web | realistic self-hosted websites | 812 tasks |
| [VisualWebArena](https://arxiv.org/abs/2401.13649) | web+vision | visual web grounding | 910 tasks |
| [BrowserGym](https://github.com/ServiceNow/BrowserGym) | web harness | unified Gym-style web-agent ecosystem | cross-benchmark web eval |
| [AgentLab](https://github.com/ServiceNow/AgentLab) | web experiments | large-scale BrowserGym experiments | model/prompt sweeps |
| [Mind2Web](https://arxiv.org/abs/2306.06070) | web | real-website task dataset | open-web generalization |
| [OSWorld](https://github.com/xlang-ai/OSWorld) | desktop GUI | full OS-in-VM tasks | 369 open-ended tasks |
| [OSWorld-Human](https://arxiv.org/html/2506.16042v1) | GUI eval | efficiency vs humans | time/step metrics |
| [WorldGUI](https://arxiv.org/html/2502.08047v3) | GUI | any-starting-point GUI automation | distribution shift stress |
| [OSUniverse](https://arxiv.org/pdf/2505.03570) | GUI | calibrated desktop tasks | office-realistic eval |
| [browser-use](https://github.com/browser-use/browser-use) | browser SDK | Pythonic browser-agent SDK | high-star practical infra |
| [Agent-S](https://github.com/simular-ai/Agent-S) | GUI | modular planner/reflector | OSWorld-style agents |
| [UFO](https://github.com/microsoft/UFO) | Windows GUI | Windows app automation | desktop workflow focus |

### 4.6 Embodied and Robotic Agents

| Work / repo | Core mechanism | Why it matters |
|---|---|---|
| [SayCan](https://arxiv.org/abs/2204.01691) | LLM planning x affordance value functions | early embodied LLM grounding |
| [PaLM-E](https://arxiv.org/abs/2303.03378) | embodied multimodal LM | large-scale VLM-to-robot bridge |
| [RT-2](https://arxiv.org/abs/2307.15818) | vision-language-action fine-tuning | robotics generalization |
| [OpenVLA](https://arxiv.org/abs/2406.09246) / [openvla](https://openvla.github.io/) | open 7B VLA on Open X-Embodiment | open robot policy anchor |
| [openpi](https://github.com/Physical-Intelligence/openpi) | pi0/pi0.5 VLA stack | contact-rich robot-control frontier |
| [pi0](https://arxiv.org/html/2410.24164v1) | flow-matching VLA | dexterous continuous control |
| [pi0.5](https://arxiv.org/pdf/2504.16054) | open-world VLA generalization | robot deployment in new settings |
| [Voyager](https://github.com/MineDojo/Voyager) | Minecraft skill library | procedural-memory bridge to embodiment |

### 4.7 Software and Coding Agents

| Work / repo | Core idea | Benchmark / evidence | Why it matters |
|---|---|---|---|
| [SWE-bench](https://github.com/SWE-bench/SWE-bench) | real GitHub issues -> patches | Verified/Pro/Live variants | canonical coding-agent eval |
| [SWE-agent](https://github.com/SWE-agent/SWE-agent) | agent-computer interface for code | SWE-bench lineage | clean academic scaffold |
| [OpenHands](https://github.com/All-Hands-AI/OpenHands) | CodeAct-style open software agent platform | leading OSS code agent | production-style harness |
| [Aider](https://github.com/Aider-AI/aider) | terminal pair-programming agent | practical code editing | lightweight baseline |
| [Cline](https://github.com/cline/cline) | IDE coding agent | VS Code ecosystem | user-facing agent UX |
| [Claude Code](https://www.anthropic.com/claude-code) | terminal coding agent | production system | proprietary but field-shaping |
| [OpenDev](https://arxiv.org/abs/2603.05344) | terminal coding-agent architecture | context engineering lessons | open technical report |

### 4.8 RL / Post-Training for Agents

| Work | Core mechanism | Why it matters |
|---|---|---|
| [DigiRL](https://arxiv.org/abs/2406.11896) | offline+online RL for Android VLM agents | mobile GUI agency |
| [Search-R1](https://github.com/PeterGriffinJin/Search-R1) | RL for search-tool use | agentic retrieval training |
| [WebDancer](https://arxiv.org/abs/2505.22648) | SFT cold start + RL for web agents | trained web agency |
| Kimi-Researcher | end-to-end agentic RL for deep research | trained long-horizon research |
| [GRPO analysis](https://arxiv.org/abs/2503.06639) | dynamics of group-relative RL | RLVR methodology |
| [DAPO](https://arxiv.org/html/2503.14476v2) | large-scale RL recipe | post-training infra |
| [Iterative DPO](https://arxiv.org/html/2503.12854v2) | preference iteration | lighter alternative to RL |
| [Agent Lightning](https://github.com/microsoft/agent-lightning) | decoupled RL for agents | train arbitrary agent loops |

### 4.9 Safety and Adversarial Robustness

| Work / repo | Threat model | Why it matters |
|---|---|---|
| [AgentDojo](https://agentdojo.spylab.ai/) | indirect prompt injection under tool use | dynamic agent-security eval |
| [AgentHarm](https://arxiv.org/abs/2410.09024) | malicious-query benchmark | harmful task resistance |
| [OS-Harm](https://arxiv.org/abs/2503.04404) | OSWorld-style harmful tasks | computer-use safety |
| [ST-WebAgentBench](https://arxiv.org/abs/2410.06703) | web policy and safety | trust in web agents |
| [Adaptive attacks on indirect prompt injection defenses](https://arxiv.org/abs/2503.00061) | adaptive attacker defeats defenses | benchmark threat realism |
| [Instruction detection defense](https://arxiv.org/abs/2505.06311) | instruction-like payload detection | practical defense layer |
| [Intent-analysis mitigation](https://arxiv.org/abs/2512.00966) | trusted vs untrusted intent | monitors tool-bound instructions |
| [OWASP MCP Top 10](https://owasp.org/www-project-mcp-top-10/) | MCP-specific risks | production security checklist |

### 4.10 Evaluation and Agent-Ops

| Benchmark / system | Measures | Best use |
|---|---|---|
| [AgentBench](https://arxiv.org/abs/2308.03688) | LLM-as-agent over 8 environments | general agent baseline |
| [GAIA](https://arxiv.org/abs/2311.12983) | real-world multi-step assistant tasks | deep/research agents |
| [TheAgentCompany](https://arxiv.org/abs/2412.14161) | simulated office work | long-horizon work automation |
| [MemoryAgentBench](https://openreview.net/forum?id=DT7JyQC3MR) | memory-agent competencies | memory-as-context eval |
| [LoCoBench-Agent](318-context-engineering-for-ai-agents-2026-deep-synthesis.md) | long-context SWE agents | context-policy eval |
| [tau-bench](https://github.com/sierra-research/tau-bench) | conversational tool+policy support tasks | customer-service eval |
| [tau2-bench](https://github.com/sierra-research/tau2-bench) | dual-control tool+policy agents | ops-shaped reliability |
| [Agent evaluation survey](https://arxiv.org/pdf/2503.16416) | benchmark landscape | eval design |
| [Enterprise eval survey](https://arxiv.org/html/2507.21504) | production roles and reliability | enterprise evaluation |

---

## 5. GitHub Repo Landscape

| Repo | Role | Popularity band | Best use | Limitation |
|---|---|---:|---|---|
| [AutoGPT](https://github.com/Significant-Gravitas/AutoGPT) | early viral autonomous agent | 100k+ | historical baseline | not the production frontier |
| [LangChain](https://github.com/langchain-ai/langchain) | agent/RAG integration layer | 100k+ | broad integrations | wide surface area |
| [browser-use](https://github.com/browser-use/browser-use) | browser-agent SDK | very high | web automation | web-specific |
| [MetaGPT](https://github.com/FoundationAgents/MetaGPT) | SOP multi-agent software company | very high | artifact-driven teams | heavy role system |
| [OpenHands](https://github.com/All-Hands-AI/OpenHands) | open coding-agent platform | very high | SWE agents and server patterns | heavy stack |
| [AutoGen](https://github.com/microsoft/autogen) | multi-agent framework | very high | research prototyping | older line in maintenance/transition |
| [Mem0](https://github.com/mem0ai/mem0) | memory layer | very high | product memory | extraction/privacy risk |
| [CrewAI](https://github.com/crewAIInc/crewAI) | role-based crews | very high | team-style workflows | design quality dominates |
| [LlamaIndex](https://github.com/run-llama/llama_index) | RAG/data context infra | very high | retrieval-heavy agents | tuning required |
| [LangGraph](https://github.com/langchain-ai/langgraph) | stateful graph runtime | high | durable scoped workflows | graph complexity |
| [Letta](https://github.com/letta-ai/letta) | memory-first stateful agents | high | long-lived assistants | runtime overhead |
| [SWE-agent](https://github.com/SWE-agent/SWE-agent) | academic coding-agent scaffold | high | SWE-bench research | code-domain focus |
| [smolagents](https://github.com/huggingface/smolagents) | minimalist agent framework | high | code-as-action agents | limited enterprise features |
| [gpt-researcher](https://github.com/assafelovic/gpt-researcher) | OSS research agent | high | deep-research workflows | citation/eval hardening needed |
| [BrowserGym](https://github.com/ServiceNow/BrowserGym) | web-agent Gym | medium | benchmarked web agents | environment ops |
| [OSWorld](https://github.com/xlang-ai/OSWorld) | OS benchmark | medium | desktop agents | VM flakiness/cost |
| [openpi](https://github.com/Physical-Intelligence/openpi) | VLA robot stack | high for robotics | robot policy fine-tuning | data/compute heavy |
| [Agent-S](https://github.com/simular-ai/Agent-S) | GUI agent | medium-high | computer-use baselines | scaffold-specific |
| [tau2-bench](https://github.com/sierra-research/tau2-bench) | ops-style agent benchmark | medium | support/tool-policy eval | simulator fidelity |

---

## 6. Benchmark Landscape

| Benchmark | Domain | Measures | Layer |
|---|---|---|---|
| AgentBench | general agents | decision making across 8 environments | L3-L6 |
| GAIA | real-world assistant tasks | multi-step reasoning, tools, multimodal | L1-L6 |
| TheAgentCompany | office work | long-horizon business automation | L2-L7 |
| SWE-bench Verified / Pro / Live | code | real issue fixing and patch validation | L2-L5 |
| BFCL | function calling | tool selection, arguments, multi-turn calls | L2 |
| ToolBench | API use | API selection and execution | L2 |
| WebArena / VisualWebArena | web | web task completion and visual grounding | L1-L3 |
| BrowserGym | web harness | unified web benchmark protocol | L1-L6 |
| OSWorld / WorldGUI / OSUniverse | GUI/computer use | desktop automation | L1-L6 |
| AndroidWorld / AitW | mobile | mobile app agency | L1-L3 |
| MemoryAgentBench | memory | recall, update, long-range, forgetting | L4-L5 |
| LoCoBench-Agent | long-context SWE | context policy and tool strategy | L2-L5 |
| AgentDojo | security | utility under indirect prompt injection | L2-L7 |
| AgentHarm / OS-Harm | safety | malicious tasks and computer-use harm | L1-L7 |
| tau-bench / tau2-bench | support ops | tool+policy reliability | L2-L7 |
| AIRS-Bench / MLGym / MLE-bench | research/ML | scientific and ML agent ability | L3-L7 |

---

## 7. Cross-Cutting Findings

### 7.1 The field moved from reasoning patterns to infrastructure

In 2023, the field was ReAct, Reflexion, ToT, and AutoGPT. In 2026, the frontier is tool protocols, memory planes, stateful runtimes, safety benchmarks, GUI environments, and production telemetry.

### 7.2 Tool use is the biggest missing bridge

Memory and context work often evaluate conversation or long-doc tasks. Tool-use work often evaluates single-turn function calls. The missing benchmark is procedural memory across shifting tool distributions.

### 7.3 Computer-use is the commercial frontier

Web/API agents are useful when APIs exist. Computer-use agents matter when they do not. OSWorld, BrowserGym, WorldGUI, and browser-use show the field moving from "call a tool" to "operate a computer."

### 7.4 Embodied agents need memory and skills

VLA systems like OpenVLA and pi0 show action grounding, but agent-memory and skill-library ideas are not yet deeply integrated into robotics. Voyager is the conceptual bridge.

### 7.5 RL is becoming the capability escalator

Prompted agents remain strong for research and analysis. Web, mobile, search, and coding agents increasingly use SFT/RL/post-training because tool-rich environments provide verifiable rewards.

### 7.6 Safety must move from static prompts to persistent state

AgentDojo-style indirect prompt injection becomes worse when bad instructions enter memory, skill libraries, MCP tool outputs, or self-evolving playbooks. Static-agent safety is not enough.

---

## 8. Open Research Gaps and Next-Paper Ideas

| Rank | Gap | Paper idea | First experiment | Why strong |
|---:|---|---|---|---|
| 1 | memory x tool-use under API shift | **MemoryBench-Tool**: procedural memory for tool-augmented agents under distribution shift | BFCL subset with 100/50/0% tool overlap; Mem0/Letta/ReasoningBank baselines | bridges memory/context with tool-use gap |
| 2 | adversarial self-evolving memory | **Adversarial Memory**: prompt-injected recall in self-evolving agents | poison 0.1/1/5% of trajectories in ACE/ReasoningBank-style memory | safety for self-evolution |
| 3 | GUI context engineering | **Computer-Use Context Engineer**: Write/Select/Compress/Isolate for OSWorld | Agent-S baseline on OSWorld subset; ablate quadrant analogs | moves context engineering into CUA |
| 4 | MCP quality | **Smelly Tools**: MCP tool-description antipatterns and auto-repair | crawl public MCP registry, classify schema/tool description smells | protocol-era tooling paper |
| 5 | skill vs tool vs procedural memory | **What Is an Agent Skill?** definition and benchmark | compare Anthropic Skills, Voyager skills, MCP tools | unifies 2026 skill papers |
| 6 | multi-agent memory | **MultiMemBench**: shared/private/federated memory under A2A | two-agent tasks with shared vs private memory | protocol + memory frontier |
| 7 | context strategy ablation | **Which Karpathy quadrant buys what?** | 2^4 ablation over research/code/web agents | turns taxonomy into science |
| 8 | faithful self-evolution | **Faithfulness in Self-Evolving Agents** | compare explained edits vs actual edits | addresses "not always faithful" concern |
| 9 | CUA safety | **CUA-RedTeam** for OSWorld | AgentDojo-style attacks in GUI tasks | safety for computer-use |
| 10 | agent ops | **Agent SLO Bench**: latency/cost/reliability alongside accuracy | tau2-bench + LangGraph/OpenHands telemetry | production relevance |
| 11 | VLA skills | **Robot Skill Memory** for VLA agents | add skill library to OpenVLA/openpi tasks | robotics + memory bridge |
| 12 | RL memory | **Memory for RL-Trained Agents** | Search-R1/WebDancer with external memory replay | joins RL and memory |

### Recommended 3-Paper Roadmap

1. **MemoryBench-Tool** - highest fit with existing memory/context work and the biggest uncovered repo category.
2. **Adversarial Memory** - extends self-evolution into safety with strong novelty.
3. **Computer-Use Context Engineer** - moves context engineering into OS/browser GUI agents.

Together these become a coherent program: **Memory Systems for Real-World Agents**.

---

## 9. Practical Reading Order

1. **Field map:** `masamasa59/ai-agent-papers`, Wang survey, Xi survey.
2. **Inner loop:** ReAct, Reflexion, ToT, LATS, ReWOO.
3. **Tool layer:** Toolformer, ToolLLM, Gorilla, BFCL, UniToolCall, MCP, A2A.
4. **Memory/context layer:** MemGPT/Letta, Mem0, Zep/Graphiti, ReasoningBank, ACE, Context Engineering chapter [318](318-context-engineering-for-ai-agents-2026-deep-synthesis.md).
5. **Embodiment:** WebArena, BrowserGym, OSWorld, OpenVLA, openpi.
6. **Training:** Search-R1, WebDancer, DigiRL, GRPO/DAPO, Agent Lightning.
7. **Safety/eval:** AgentDojo, AgentHarm, OS-Harm, tau2-bench, Measuring Agents in Production.
8. **Next-paper design:** MemoryBench-Tool -> Adversarial Memory -> Computer-Use Context Engineer.

---

## 10. Sources

- [masamasa59/ai-agent-papers](https://github.com/masamasa59/ai-agent-papers)
- [Survey on LLM-based Autonomous Agents](https://arxiv.org/abs/2308.11432)
- [Rise and Potential of LLM Based Agents](https://arxiv.org/abs/2309.07864)
- [ReAct](https://arxiv.org/abs/2210.03629)
- [Tree of Thoughts](https://arxiv.org/abs/2305.10601)
- [Reflexion](https://arxiv.org/abs/2303.11366)
- [LATS](https://arxiv.org/abs/2310.04406)
- [Toolformer](https://arxiv.org/abs/2302.04761)
- [ToolLLM / ToolBench](https://arxiv.org/abs/2307.16789)
- [Gorilla](https://arxiv.org/abs/2305.15334)
- [UniToolCall](https://arxiv.org/abs/2604.11557)
- [Evolution of Tool Use in LLM Agents](https://arxiv.org/html/2603.22862v2)
- [CodeAct](https://arxiv.org/abs/2402.01030)
- [MCP](https://modelcontextprotocol.io/)
- [A2A](https://github.com/a2aproject/A2A)
- [AGENTS.md](https://github.com/openai/agents-md)
- [AutoGen](https://github.com/microsoft/autogen)
- [MetaGPT](https://github.com/FoundationAgents/MetaGPT)
- [ChatDev](https://github.com/OpenBMB/ChatDev)
- [CrewAI](https://github.com/crewAIInc/crewAI)
- [WebArena](https://github.com/web-arena-x/webarena)
- [BrowserGym](https://github.com/ServiceNow/BrowserGym)
- [AgentLab](https://github.com/ServiceNow/AgentLab)
- [OSWorld](https://github.com/xlang-ai/OSWorld)
- [WorldGUI](https://arxiv.org/html/2502.08047v3)
- [OSUniverse](https://arxiv.org/pdf/2505.03570)
- [OpenVLA](https://arxiv.org/abs/2406.09246)
- [openpi](https://github.com/Physical-Intelligence/openpi)
- [pi0](https://arxiv.org/html/2410.24164v1)
- [pi0.5](https://arxiv.org/pdf/2504.16054)
- [SWE-bench](https://github.com/SWE-bench/SWE-bench)
- [SWE-agent](https://github.com/SWE-agent/SWE-agent)
- [OpenHands](https://github.com/All-Hands-AI/OpenHands)
- [DigiRL](https://arxiv.org/abs/2406.11896)
- [Search-R1](https://github.com/PeterGriffinJin/Search-R1)
- [WebDancer](https://arxiv.org/abs/2505.22648)
- [GRPO analysis](https://arxiv.org/abs/2503.06639)
- [DAPO](https://arxiv.org/html/2503.14476v2)
- [Agent Lightning](https://github.com/microsoft/agent-lightning)
- [AgentDojo](https://agentdojo.spylab.ai/)
- [AgentHarm](https://arxiv.org/abs/2410.09024)
- [OS-Harm](https://arxiv.org/abs/2503.04404)
- [Adaptive attacks on indirect prompt injection defenses](https://arxiv.org/abs/2503.00061)
- [Instruction detection defense](https://arxiv.org/abs/2505.06311)
- [OWASP MCP Top 10](https://owasp.org/www-project-mcp-top-10/)
- [AgentBench](https://arxiv.org/abs/2308.03688)
- [GAIA](https://arxiv.org/abs/2311.12983)
- [TheAgentCompany](https://arxiv.org/abs/2412.14161)
- [tau-bench](https://github.com/sierra-research/tau-bench)
- [tau2-bench](https://github.com/sierra-research/tau2-bench)

---

## 11. Final Takeaways

1. **AI Agents are now a stack, not a prompt pattern.**
2. **The repo spine shows the 2026 center of gravity: self-evolution, skills, memory, research agents, coding agents, and benchmarks.**
3. **Your prior docs cover the cognitive layers well, but tool-use, computer-use, embodied agents, RL training, safety, and ops are the gaps.**
4. **The most publishable next step is not another survey. It is a benchmark/system that joins memory, tools, and context engineering under distribution shift.**
5. **The recommended roadmap is MemoryBench-Tool, Adversarial Memory, then Computer-Use Context Engineer.**
