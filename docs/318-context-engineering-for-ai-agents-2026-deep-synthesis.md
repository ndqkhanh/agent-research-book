# 318 - Context Engineering for AI Agents 2026: Deep Synthesis

**Scope.** A single dense, interactive synthesis of the supplied OpenReview/arXiv context-engineering papers plus adjacent papers, benchmarks, and high-signal open-source systems. This chapter is written for next-paper ideation: scan the taxonomy, compare the systems, then use the research-gap map to design a publishable agent-context paper.

**Thesis.** Context engineering for agents is no longer "prompt engineering with more tokens." It is the engineering of a **budgeted, typed, evolving information state** around a sequential decision process. The strongest 2025-2026 work converges on four verbs:

1. **Write** useful state outside the model window.
2. **Select** only the right memory, examples, tools, documents, and observations.
3. **Compress** transient evidence without erasing causal state.
4. **Isolate** work into subagents, sandboxes, branches, or scoped state so context does not contaminate itself.

The next strong paper is probably not another isolated memory/RAG/compression trick. It is a unified evaluation and system that measures how these four strategies compose, interfere, and trade off cost against accuracy.

---

## 0. Reading Map

If you only have one hour:

1. **Definition / taxonomy:** start with the four-strategy frame: Write, Select, Compress, Isolate.
2. **Best direct context-engineering system:** ACE, which treats context as an evolving playbook.
3. **Best context-memory eval:** MemoryAgentBench and LoCoBench-Agent.
4. **Best experience-memory bridge:** ReasoningBank and Memento.
5. **Best long-horizon compression direction:** Focus / Active Context Compression, CORAL, MemSearcher, FocusAgent.
6. **Best context-isolation direction:** MASAI, AGENT*, Anthropic multi-agent research, LangGraph supervisor patterns.
7. **Best production substrate:** LangGraph, LlamaIndex, Letta, Mem0, Zep/Graphiti, Cognee, DCI-Agent-Lite, OpenHands, SWE-agent.

---

## 1. The Context Engineering Stack

```text
                          CONTEXT ENGINEERING STACK

  PRE-INFERENCE: assemble the next model call
  - system/developer instructions
  - tool loadout
  - retrieved documents
  - retrieved memories
  - selected demonstrations
  - playbook / skill snippets
  - output schema

  IN-LOOP: control context while the agent acts
  - observation pruning
  - tool-result truncation
  - direct corpus interaction
  - scratchpad / think tool
  - subagent dispatch
  - checkpoint + purge

  POST-INFERENCE: update future context
  - reflection extraction
  - memory writes and deletes
  - playbook curation
  - case-bank update
  - skill/prompt mutation
  - bias and drift monitoring
```

The key idea is that the "prompt" is only one component. The context passed to the model is a runtime artifact assembled from instructions, tools, examples, memory, documents, schemas, history, state, and subagent outputs.

---

## 2. Core Taxonomy

| Strategy | Core question | Typical mechanisms | Canonical systems / papers | Failure if done badly |
|---|---|---|---|---|
| **Write** | What should persist outside the current window? | scratchpads, memory files, playbooks, case banks, reflections, skill libraries | MemGPT/Letta, Mem0, Zep, ACE, ReasoningBank, Memento, Voyager | stale memories, bias amplification, privacy leaks |
| **Select** | What should enter this turn's context? | RAG, graph retrieval, demo selection, tool loadout selection, memory retrieval, DCI | LlamaIndex, LangChain/LangGraph, DCI, ICL-for-agents, DynamicRAG, Graphiti | irrelevant evidence, missing constraints, tool confusion |
| **Compress** | What can be shortened without losing state? | summarization, tool-result clearing, checkpointing, prompt compression, observation trimming | LLMLingua, RECOMP, Focus, CORAL, MemSearcher, FocusAgent, OpenDev | context collapse, lost causal details, re-exploration |
| **Isolate** | What should be handled in a separate context? | subagents, worktrees, sandboxes, state shards, lead-worker hierarchy | MASAI, AGENT*, Anthropic multi-agent research, LangGraph supervisors, AutoGen, MetaGPT | context clash, goal drift, weak aggregation |

### Five Orthogonal Axes

| Axis | Low end | Middle | High end | Why it matters |
|---|---|---|---|---|
| **Lifecycle** | pre-inference packing | in-loop updates | post-inference consolidation | Agents need all three, not one-shot prompts |
| **State duration** | one turn | one session | lifelong | Context engineering subsumes memory once state persists |
| **Representation** | raw text | structured notes / schemas | graph, latent, executable artifacts | Representation determines editability and trust |
| **Control policy** | static heuristic | agentic tool call | learned/RL policy | Learned control is emerging but harder to audit |
| **Trust boundary** | all context trusted | provenance-tagged | quarantined / isolated | Retrieved or user-supplied context is an attack surface |

---

## 3. Master Comparison Table - Supplied Primary Sources

| Work | Status / venue | Main strategy | Core mechanism | Evidence / benchmark | Why it matters for context engineering | Main limitation |
|---|---|---|---|---|---|---|
| [ACE / Agentic Context Engineering](https://openreview.net/forum?id=eC4ygDs02R) | ICLR 2026 Poster | Write + Select | evolving playbook with generate-reflect-curate loop and grow-and-refine updates | AppWorld, FiNER, Formula; author-reported gains over GEPA/dynamic cheatsheets | Best direct formulation of context as an evolving artifact, not a static prompt | playbook growth, curation cost, possible collapse over long horizons |
| [MemoryAgentBench](https://openreview.net/forum?id=DT7JyQC3MR) | ICLR 2026 Poster | Eval for Write + Select | incremental multi-turn benchmark with retrieval, test-time learning, long-range, selective forgetting | memory-agent benchmark suite | Defines what memory-as-context must be able to do | text-centric benchmark; not full production harness |
| [Opponent Simulation / inference-time scaling](https://openreview.net/forum?id=ywXgFYs44d) | ICLR 2026 submitted | Select + Isolate | opponent-model LLM plus best-of-N simulated rollouts | repeated negotiation games | Shows inference-time context can include simulated social state | narrow strategic-game domain |
| [ReasoningBank](https://openreview.net/forum?id=jL7fwchScm) | ICLR 2026 Poster | Write + Select | extracts reusable reasoning memories from successes/failures; memory-aware test-time scaling | web browsing and SWE-style tasks | Turns experience into reusable context rather than raw transcript | memory pollution and bad self-judgment risk |
| [AGENT*](https://openreview.net/forum?id=lifeoGrKRB) | ICLR 2026 submitted | Isolate + Write | self-play extracts reusable collaboration modules; dual-level planner budgets module calls | complex agent benchmarks, author-reported | Treats collaboration patterns as callable context modules | submitted; external reproducibility unclear |
| [CORAL / Don't Lose the Thread](https://openreview.net/forum?id=NBGlItueYE) | ICLR 2026 withdrawn | Compress + Write | callable checkpoint tool, task-progress memory, purge/resume, multi-episode ARPO | GAIA-style long-horizon claims | Strong idea: working memory should be actively reset, not just summarized | withdrawn; needs rigorous replication |
| [MASSE](https://openreview.net/forum?id=CuYto2s2Kd) | ICLR 2026 submitted / withdrawn signal in prior reference | Isolate | multi-agent structural-engineering workflow | domain case studies | Shows context isolation beyond SWE/research domains | narrow domain and limited accessible detail |
| [ReMemR1](https://openreview.net/forum?id=1cymflI2Lh) | ICLR 2026 Poster | Write | revisitable memory and RL-trained callbacks into prior evidence | long-document QA | Breaks forward-only memory; relevant to agents that must revisit old facts | mostly long-doc QA, less tool-heavy |
| [MemSearcher](https://openreview.net/forum?id=EWIAx3NgvA) | ICLR 2026 withdrawn | Compress + Write | compact per-turn external memory trained with multi-context GRPO | seven search QA benchmarks, author-reported gains | Clean formulation of search-agent memory as compact state | withdrawn; RL pipeline and code availability unclear |
| [CaTS](https://openreview.net/forum?id=jrSc4RJXy1) | OpenReview access partially restricted | Test-time scaling | calibrated confidence for adaptive sampling | multiple LLMs and datasets per public classification | Adjacent: compute budget competes with context budget | not primarily a context artifact paper |
| [Bias Amplification in LM Evolution](https://openreview.net/forum?id=BSYn7ah4KX) | NeurIPS 2024 Poster | Risk model for Write loops | Bayesian iterated-learning analysis of repeated model outputs | experiments across LMs | Any self-evolving context/playbook can amplify subtle bias | theory/risk paper, not an engineering recipe |
| [HCAM / mental time travel](https://openreview.net/forum?id=wfiVgITyCC_) | older RL memory paper / URL access fragile | Selective memory | hierarchical chunk attention | RL memory tasks | Conceptual ancestor for hierarchical context selection | not LLM-agent specific |
| [LatentEvolve](https://openreview.net/forum?id=QTOYA4PFiJ) | ICLR 2026 submitted | Write + Select in latent space | daytime latent retrieval plus nighttime latent consolidation | 8 benchmarks, 5 backbones, author-reported | Context evolution can happen in latent state, not only text | low interpretability and harder governance |
| [Verlog](https://openreview.net/forum?id=GmodkWwMV3) | NeurIPS 2025 MTI-LLM Workshop | Compress | customizable memory length, dual-discount GAE, long multi-turn RL | BabyAI, BabaIsAI, Crafter; 400+ turn trajectories | Empirical warning that shorter memory can beat longer memory | RL environments, not web/code agents |
| [Test-Time Adaptation / GTTA](https://openreview.net/forum?id=OH4PE0TDo0) | ICLR 2026 Poster | Write + Select | syntactic alignment vector plus dynamics grounding by exploration | WebArena multi-site, function calling | Agents can build deployment-specific contextual knowledge without full retraining | can overfit probes or miss adversarial dynamics |
| [FocusAgent](https://openreview.net/forum?id=DQlOn3pVvL) | ICLR 2026 submitted | Select + Compress | small LLM extracts goal-relevant web observation lines from AxTree | WorkArena/WebArena, injection mitigation variant | Direct attack on bloated web observations | extractor can remove rare but crucial UI facts |
| [LoCoBench-Agent](https://openreview.net/forum?id=cc18pvWqb9) / [arXiv](https://arxiv.org/abs/2511.13998) | CoRR 2025 | Benchmark | long-context interactive SWE benchmark, 8 tools, 9 metrics, 10k-1M token scenarios | 8,000 scenarios | Best benchmark candidate for long-context agent context policies | synthetic/simulated repos differ from production mess |
| [Leveraging ICL for LM Agents](https://openreview.net/forum?id=BzBqRAbdFC) | NeurIPS 2025 MTI-LLM Workshop | Select | annotate trajectories, select similar demos, insert snippets per step | AppWorld, agent ICL experiments | Shows demonstration selection can rival heavier adaptation | annotation and retrieval cost |
| [Opponent Shaping / ShapeLLM](https://openreview.net/forum?id=yJoHTqUNry) | ICLR 2026 Poster | Isolate / multi-agent dynamics | model-free opponent shaping for transformer agents | iterated matrix games | Subagents are not neutral; contexts can shape other agents | toy strategic environments |
| [MASAI](https://openreview.net/forum?id=NSINt8lLYB) | NeurIPS 2024 | Isolate | five specialized SWE subagents with scoped input-strategy-output contracts | SWE-bench Lite | Strong example of context isolation by decomposition | SWE-only; no broad context-policy ablations |

---

## 4. Adjacent Papers and Techniques to Cite

| Cluster | Must-cite works | Core idea | Where it fits |
|---|---|---|---|
| Context engineering surveys | [A Survey of Context Engineering for LLMs](https://arxiv.org/abs/2507.13334), Context Engineering 2.0 | CE as an assembly function over information payloads | definition and taxonomy |
| Long-context pathology | [Lost in the Middle](https://arxiv.org/abs/2307.03172), context-length degradation work, context rot essays | long windows degrade through position, distraction, poisoning, confusion, clash | failure-mode section |
| Prompt/context compression | [LLMLingua](https://github.com/microsoft/LLMLingua), LongLLMLingua, RECOMP, Selective Context, ICAE, Provence | compress retrieved or historical context before inference | Compress quadrant |
| Retrieval selection | DynamicRAG, dynamic passage selection, graph RAG, DCI | top-k retrieval is too rigid for agents | Select quadrant |
| Memory systems | MemGPT/Letta, Mem0, Zep/Graphiti, Cognee, A-Mem, LightMem, Memento | persistent state as an external context plane | Write + Select |
| Test-time scaling | self-consistency, BoN, budget forcing, CaTS, s1, scaling-over-scaling | allocate tokens/rollouts between thinking and environment context | compute/context budget |
| Multi-agent isolation | MASAI, AutoGen, MetaGPT, Anthropic multi-agent researcher, DACS, SagaLLM | isolate detailed work and return compact summaries | Isolate quadrant |
| Long-context evals | LongBench, RULER, InfiniteBench, LongMemEval, LoCoMo, LoCoBench-Agent | measure usable context, not advertised context length | evaluation |

---

## 5. Expandable Paper Cards

<details>
<summary><strong>ACE - Agentic Context Engineering</strong></summary>

**Source:** [OpenReview](https://openreview.net/forum?id=eC4ygDs02R)

**Core idea.** Context should evolve like a playbook. Instead of repeatedly rewriting a short summary, the system generates attempts, reflects on failures/successes, and curates durable context entries that can be selected later.

**Why it matters.** ACE names two failure modes: **brevity bias**, where optimizers converge to terse prompts that lose heuristics, and **context collapse**, where repeated summarization erodes useful detail. It is the most direct paper in this batch for "context engineering" as a self-improving discipline.

**Paper angle.** ACE is a strong base for a follow-up on playbook decay, cache-aware playbook ordering, or bias-aware curation.

</details>

<details>
<summary><strong>MemoryAgentBench - memory as context evaluation</strong></summary>

**Source:** [OpenReview](https://openreview.net/forum?id=DT7JyQC3MR)

**Core idea.** Evaluate memory agents through incremental multi-turn interactions rather than static long-context QA. The four skills are accurate retrieval, test-time learning, long-range understanding, and selective forgetting.

**Context-engineering interpretation.** MemoryAgentBench is the Write+Select evaluation harness: it asks whether what the agent wrote to memory and later selected into context was actually useful and current.

</details>

<details>
<summary><strong>ReasoningBank - experience memory as reusable context</strong></summary>

**Source:** [OpenReview](https://openreview.net/forum?id=jL7fwchScm)

**Core idea.** Distill successes and failures into reusable reasoning memories, then retrieve them during future tasks.

**Why it matters.** The context object is not a document or fact. It is a strategy. That moves context engineering from "retrieve facts" to "retrieve procedures and failure lessons."

</details>

<details>
<summary><strong>CORAL, MemSearcher, Focus / FocusAgent - compression with control</strong></summary>

**Sources:** [CORAL](https://openreview.net/forum?id=NBGlItueYE), [MemSearcher](https://openreview.net/forum?id=EWIAx3NgvA), [Active Context Compression](https://arxiv.org/abs/2601.07190), [FocusAgent](https://openreview.net/forum?id=DQlOn3pVvL)

**Shared idea.** Full history is often worse than compact state. CORAL checkpoints and purges working memory, MemSearcher maintains compact search memory, Focus compresses intra-trajectory logs into a Knowledge block, and FocusAgent trims web observations before the action model sees them.

**Design rule.** Compress transient observations aggressively; preserve verified causal state conservatively.

</details>

<details>
<summary><strong>LoCoBench-Agent - long-context agent evaluation</strong></summary>

**Sources:** [OpenReview](https://openreview.net/forum?id=cc18pvWqb9), [arXiv](https://arxiv.org/abs/2511.13998)

**Core idea.** A long-context interactive SWE-agent benchmark with scenarios spanning 10k to 1M tokens, multiple tools, and metrics for comprehension and efficiency.

**Why it matters.** This is the best supplied benchmark for measuring whether context-engineering policies actually help long-running agents rather than just improving static QA.

</details>

<details>
<summary><strong>MASAI and AGENT* - context isolation</strong></summary>

**Sources:** [MASAI](https://openreview.net/forum?id=NSINt8lLYB), [AGENT*](https://openreview.net/forum?id=lifeoGrKRB)

**Core idea.** Do not let one monolithic trajectory carry every fact, action, and hypothesis. MASAI decomposes SWE tasks into specialized subagents; AGENT* treats collaboration modules as reusable compute/context units.

**Production lesson.** Isolate only when the task naturally decomposes. For write-heavy tasks with tight state coupling, subagents can create context clash.

</details>

---

## 6. OSS and Systems Landscape

| Repo / system | Role | Best for | Limitation | Strategy |
|---|---|---|---|---|
| [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph) | stateful graph runtime with checkpoints and durable execution | production agents with replay, branching, human-in-loop, scoped state | graph complexity can exceed task complexity | Select + Isolate |
| [langchain-ai/langchain](https://github.com/langchain-ai/langchain) | broad integrations for tools, retrievers, and agents | fast plumbing for RAG/tool agents | large surface and version churn | Select |
| [langchain-ai/langmem](https://github.com/langchain-ai/langmem) | memory tooling for LangGraph-style stores | agent-controlled memory tools | tied to framework choices | Write + Select |
| [run-llama/llama_index](https://github.com/run-llama/llama_index) | data and agent framework for ingestion/indexing/retrieval | RAG-heavy context assembly | retrieval tuning remains hard | Select |
| [letta-ai/letta](https://github.com/letta-ai/letta) | MemGPT lineage stateful agents and managed memory | long-lived assistants with explicit memory | heavier runtime than simple memory APIs | Write |
| [mem0ai/mem0](https://github.com/mem0ai/mem0) | universal memory layer | product personalization and user/session memory | extraction/privacy risks | Write + Select |
| [getzep/graphiti](https://github.com/getzep/graphiti) | temporal knowledge graph memory | changing facts, provenance, graph expansion | graph ops overhead | Write + Select |
| [topoteretes/cognee](https://github.com/topoteretes/cognee) | graph/vector cognitive memory pipeline | messy corpora and private memory graphs | backend/perf tuning | Write + Select |
| [DCI-Agent-Lite](https://github.com/DCI-Agent/DCI-Agent-Lite) | direct corpus interaction agent | local/private corpora and exact evidence search | slower than pre-indexed retrieval | Select |
| [OpenHands](https://github.com/OpenHands/OpenHands) | software-engineering agent harness | rich tool/file/sandbox context | heavy system | Isolate + Compress |
| [SWE-agent](https://github.com/princeton-nlp/SWE-agent) | academic SWE-bench coding-agent harness | reproducible coding-agent context studies | benchmark-oriented | Select + Compress |
| [microsoft/autogen](https://github.com/microsoft/autogen) | multi-agent orchestration | prototyping conversations/roles | group chat can create context clash | Isolate |
| [crewAI](https://github.com/crewAIInc/crewAI) | role-based crews and memory | workflow teams with role scopes | crew design quality dominates | Isolate + Write |
| [microsoft/LLMLingua](https://github.com/microsoft/LLMLingua) | prompt compression | cost/window reduction | can delete task-critical details | Compress |
| [NVIDIA/RULER](https://github.com/NVIDIA/RULER) | effective long-context benchmark | measuring usable context length | synthetic | Eval |
| [LongBench](https://github.com/THUDM/LongBench) | long-context benchmark suite | broad long-input comparisons | not agent-specific | Eval |
| [LoCoMo](https://github.com/snap-research/LoCoMo) | long conversational memory | memory-as-context eval | small/domain-limited | Eval |

---

## 7. Failure Modes

| Failure mode | Symptom | Most relevant sources | Mitigation |
|---|---|---|---|
| **Context poisoning** | bad observation or malicious instruction persists and steers future actions | prompt-injection research, FocusAgent safe variant, memory poisoning work | quarantine, provenance, trust scoring, instruction/data separation |
| **Context distraction** | long history makes model repeat or miss current goal | Verlog, Focus, LoCoBench-Agent | active compression, shorter memory, task-state checkpoints |
| **Context confusion** | too many tools/examples/memories cause wrong selection | tool-loadout work, Anthropic guidance, LangGraph/LlamaIndex practice | dynamic tool selection, schema clarity, progressive disclosure |
| **Context clash** | conflicting shards or subagent outputs collide | ShapeLLM, MASAI, multi-agent failure studies | scoped subagents, verifier aggregation, source-ranked summaries |
| **Context collapse** | repeated summaries/playbook updates erase useful heuristics | ACE, CORAL, MemSearcher | append-mostly structured playbooks, high-recall compaction, audit logs |
| **Bias amplification** | self-improving loops amplify subtle patterns | Bias Amplification in LM Evolution | diversity-preserving curation, drift metrics, reservoir sampling |
| **Negative transfer** | retrieved prior strategy hurts new task | CoPS, ReasoningBank, Memento | conservative retrieval, verifier-gated application, OOD detection |
| **Cost explosion** | every turn reprocesses huge state | prompt caching, Focus, OpenDev, Anthropic context engineering | cache-aware ordering, tool-result clearing, subagent summaries |

---

## 8. Benchmark Map

| Benchmark | Context dimension tested | Best use in a paper |
|---|---|---|
| MemoryAgentBench | memory retrieval, test-time learning, long-range understanding, forgetting | Write+Select evaluation |
| LoCoBench-Agent | long-context interactive SWE agents, 10k-1M tokens | full-stack context-policy eval |
| AppWorld | tool-use planning and application workflows | ACE/ICL-style context adaptation |
| WebArena / WorkArena | web observations, UI state, tool choice | FocusAgent/GTTA/observation pruning |
| GAIA | tool-use research and multi-hop reasoning | long-horizon agent context |
| SWE-bench / SWE-bench Verified | codebase context, tool traces, patch validation | coding-agent harness context |
| LongBench / LongBench-v2 | long-input understanding | base model capability baseline |
| RULER | effective context length and retrieval/aggregation | stress-testing advertised windows |
| InfiniteBench | ultra-long input tasks | high-context packing baseline |
| LongMemEval / LoCoMo | conversational long-term memory | memory product comparison |
| BabyAI / BabaIsAI / Crafter | long-horizon RL trajectories | context length in training loops |
| Berkeley Function Calling / tool evals | tool loadout confusion | Select quadrant for tools |

---

## 9. Patterns for Builders

### Pattern A - Playbook Context

Use when the agent repeatedly solves related tasks. Store structured lessons, examples, and constraints. Retrieve only relevant playbook sections.

**Best sources:** ACE, ReasoningBank, ICL-for-agents, Memento.

### Pattern B - Checkpoint and Purge

Use when the trace grows but only verified task state matters. Save progress/facts, then clear noisy observations.

**Best sources:** CORAL, Focus, OpenDev compaction, Claude Code auto-compaction.

### Pattern C - Observation Pruning

Use when tool observations are large, noisy, or adversarial. Extract relevant spans before the action model sees them.

**Best sources:** FocusAgent, DCI, LLMLingua/RECOMP, Anthropic just-in-time retrieval.

### Pattern D - Direct Corpus Interaction

Use when exact lexical constraints, local evidence checking, or evolving corpora matter more than embedding similarity.

**Best sources:** DCI-Agent-Lite, Anthropic context engineering, coding-agent grep/glob patterns.

### Pattern E - Isolated Specialist Contexts

Use when subproblems are separable and summaries are sufficient for aggregation.

**Best sources:** MASAI, AGENT*, Anthropic multi-agent researcher, LangGraph supervisors.

### Pattern F - Context Governance

Use for production agents. Every context item should carry type, source, trust, timestamp, token cost, and deletion/audit behavior.

**Best sources:** Zep/Graphiti, LangGraph persistence, prompt-injection defense, memory governance docs.

---

## 10. Research Gaps and Next-Paper Ideas

| Gap | Why unsolved | Concrete paper direction |
|---|---|---|
| **Unified four-quadrant evaluation** | papers test Write/Select/Compress/Isolate separately | *ContextStack-Bench*: one harness over AppWorld, LoCoBench-Agent, WebArena, GAIA with ablations for each strategy |
| **Strategy interference** | composing memory, compression, retrieval, and subagents can hurt | *When Context Engineering Hurts*: measure negative interactions between strategies |
| **Context rot benchmark** | poisoning/distraction/confusion/clash are mostly anecdotal | *ContextRot-Bench*: perturb agents with each failure family |
| **Playbook decay** | ACE-style contexts grow and can collapse | cache-aware append-mostly playbooks with decay, dedupe, and diversity metrics |
| **Bias-aware self-evolving context** | self-improvement loops amplify bias | add iterated-learning monitors to ACE/ReasoningBank-style writes |
| **Long-context vs RAG routing** | no per-turn policy decides "load more" vs "retrieve less" | train a router from task features, model context threshold, and retrieval uncertainty |
| **Tool-loadout learning** | too many tool schemas confuse agents | learn per-task tool subsets using AppWorld / function-calling rewards |
| **Subagent pollution detection** | lead agents aggregate low-quality worker outputs | verifier-weighted aggregation with provenance and contradiction scoring |
| **Demonstrations from playbooks** | ACE stores trajectories; ICL-for-agents selects demos | build demo selection over self-evolved playbook entries |
| **Latent + symbolic context** | LatentEvolve is fast but opaque; ACE is interpretable but long | dual-store architecture: latent cache for speed, symbolic playbook for audit |
| **Cache-locality-preserving context updates** | rewriting prompts/playbooks breaks prompt/KV cache | append-only context logs with stable hot prefix and rolling compaction |
| **Intrinsic context-quality metrics** | accuracy is delayed and expensive | define relevance density, redundancy, contradiction risk, recency utility, and trust score |

### Best Next Paper

**Title candidate:** *ContextStack: A Four-Quadrant Benchmark and Runtime for Context Engineering in LLM Agents*

**Core claim:** Agent performance improves when context is managed as a typed runtime state with explicit Write, Select, Compress, and Isolate policies, but these policies can also interfere. Measuring the interference is as important as measuring the individual gains.

**System:**

- Write: ACE/ReasoningBank-style playbook and failure memory.
- Select: DCI/RAG/tool-loadout/demonstration selection.
- Compress: Focus/FocusAgent/LLMLingua-style observation and trace compaction.
- Isolate: MASAI/LangGraph-style scoped subagents.
- Governance: provenance, trust level, token cost, timestamp, and verifier score on every context item.

**Ablations:**

1. no persistent playbook,
2. no memory retrieval,
3. no observation pruning,
4. no subagent isolation,
5. no verifier-gated aggregation,
6. no prompt-cache-aware ordering,
7. no context-rot defense.

**Benchmarks:**

- AppWorld for tool-use context adaptation.
- LoCoBench-Agent for long-context SWE agents.
- WebArena / WorkArena for web observation pruning.
- GAIA for research/tool-use tasks.
- MemoryAgentBench for memory competencies.

---

## 11. Practical Reading Order

1. **Start with definitions:** Context Engineering survey, Anthropic context engineering, ACE.
2. **Then memory-as-context:** MemoryAgentBench, ReasoningBank, ReMemR1, Memento, MemGPT/Letta, Mem0.
3. **Then compression:** LLMLingua, RECOMP, Focus, CORAL, MemSearcher, FocusAgent.
4. **Then selection:** DCI, LlamaIndex, LangGraph, ICL-for-agents, dynamic passage selection.
5. **Then isolation:** MASAI, AGENT*, Anthropic multi-agent research, AutoGen, MetaGPT.
6. **Then evaluation:** LoCoBench-Agent, RULER, LongBench, LongMemEval, LoCoMo.
7. **Then risks:** Bias Amplification, prompt injection, context clash, memory poisoning.

---

## 12. Sources

- [ACE / Agentic Context Engineering](https://openreview.net/forum?id=eC4ygDs02R)
- [MemoryAgentBench](https://openreview.net/forum?id=DT7JyQC3MR)
- [Opponent Simulation as Inference-time Scaling](https://openreview.net/forum?id=ywXgFYs44d)
- [ReasoningBank](https://openreview.net/forum?id=jL7fwchScm)
- [AGENT*](https://openreview.net/forum?id=lifeoGrKRB)
- [CORAL / Don't Lose the Thread](https://openreview.net/forum?id=NBGlItueYE)
- [MASSE](https://openreview.net/forum?id=CuYto2s2Kd)
- [ReMemR1](https://openreview.net/forum?id=1cymflI2Lh)
- [MemSearcher](https://openreview.net/forum?id=EWIAx3NgvA)
- [CaTS](https://openreview.net/forum?id=jrSc4RJXy1)
- [Bias Amplification in Language Model Evolution](https://openreview.net/forum?id=BSYn7ah4KX)
- [HCAM / mental time travel](https://openreview.net/forum?id=wfiVgITyCC_)
- [LatentEvolve](https://openreview.net/forum?id=QTOYA4PFiJ)
- [Verlog](https://openreview.net/forum?id=GmodkWwMV3)
- [GTTA / Test-Time Adaptation for LLM Agents](https://openreview.net/forum?id=OH4PE0TDo0)
- [FocusAgent](https://openreview.net/forum?id=DQlOn3pVvL)
- [LoCoBench-Agent OpenReview](https://openreview.net/forum?id=cc18pvWqb9)
- [LoCoBench-Agent arXiv](https://arxiv.org/abs/2511.13998)
- [ICL for LM Agents](https://openreview.net/forum?id=BzBqRAbdFC)
- [ShapeLLM / Opponent Shaping](https://openreview.net/forum?id=yJoHTqUNry)
- [MASAI](https://openreview.net/forum?id=NSINt8lLYB)
- [Anthropic Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [A Survey of Context Engineering for LLMs](https://arxiv.org/abs/2507.13334)
- [Lost in the Middle](https://arxiv.org/abs/2307.03172)
- [RECOMP](https://arxiv.org/abs/2310.04408)
- [LLMLingua](https://github.com/microsoft/LLMLingua)
- [LangGraph](https://github.com/langchain-ai/langgraph)
- [LangChain](https://github.com/langchain-ai/langchain)
- [LlamaIndex](https://github.com/run-llama/llama_index)
- [Letta](https://github.com/letta-ai/letta)
- [Mem0](https://github.com/mem0ai/mem0)
- [Graphiti](https://github.com/getzep/graphiti)
- [Cognee](https://github.com/topoteretes/cognee)
- [DCI-Agent-Lite](https://github.com/DCI-Agent/DCI-Agent-Lite)
- [OpenHands](https://github.com/OpenHands/OpenHands)
- [SWE-agent](https://github.com/princeton-nlp/SWE-agent)
- [AutoGen](https://github.com/microsoft/autogen)
- [CrewAI](https://github.com/crewAIInc/crewAI)
- [RULER](https://github.com/NVIDIA/RULER)
- [LongBench](https://github.com/THUDM/LongBench)
- [LoCoMo](https://github.com/snap-research/LoCoMo)

---

## 13. Final Takeaways

1. **Context engineering is the control plane for agent intelligence.** It decides what the model sees, when, why, and under what trust assumptions.
2. **The field is converging on Write / Select / Compress / Isolate.** Most papers are one quadrant; the opportunity is composition.
3. **More context is not always better.** Verlog, Focus, FocusAgent, and long-context pathology work all show that shorter, better state can beat longer state.
4. **Memory is context over time.** Memory papers belong in context engineering once retrieval and update policies affect the next model call.
5. **Subagents are context isolation, not magic.** They help read-mostly decomposable tasks and hurt when shared state is tight.
6. **Evaluation is the bottleneck.** LoCoBench-Agent, MemoryAgentBench, AppWorld, WebArena, and GAIA should be combined into a unified CE benchmark.
7. **The next publishable idea is a stack, not a component.** A typed context runtime with ablations across Write, Select, Compress, and Isolate would connect memory, self-evolution, research agents, and harness engineering in one paper.
