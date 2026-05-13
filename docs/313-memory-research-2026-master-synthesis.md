# 313 — Memory Research 2026: Master Synthesis for Next-Paper Ideation

**Scope.** Deep synthesis of the user-supplied OpenReview memory papers plus the strongest adjacent 2023–2026 canon, surveys, and high-star open-source repositories. This chapter is designed as an **interactive reading dashboard**: expand the `<details>` blocks, scan the comparison tables, then use the gap map and research-idea prompts to design the next paper.

**Thesis.** The 2026 agent-memory field is converging on one answer: **memory is not a vector database; it is a control system**. The best systems couple (1) typed memory objects, (2) learned or agentic write/update/delete policies, (3) retrieval that is aware of temporal validity and task distribution, (4) consolidation from raw episodes into reusable experience, and (5) evaluation that tests memory inside interactive multi-turn agents rather than static QA.

---

## 0. Reading map

Start here if you only have one hour:

1. **Benchmark ground truth** — [MemoryAgentBench](https://openreview.net/forum?id=DT7JyQC3MR): four competencies for memory agents.
2. **Best explicit memory architecture** — [A-Mem](https://openreview.net/forum?id=FiM0M8gcct): Zettelkasten-style dynamic linking and evolution.
3. **Best efficiency architecture** — [LightMem](https://openreview.net/forum?id=dyJ0GWpjJB): three-stage sensory/short-term/long-term memory with offline sleep updates.
4. **Best learned control direction** — [MEM1](https://openreview.net/forum?id=jJ6F1sDn9i) and [ReMemR1](https://openreview.net/forum?id=1cymflI2Lh): train the agent to decide what to keep and when to revisit.
5. **Best experience-abstraction direction** — [ReasoningBank](https://openreview.net/forum?id=jL7fwchScm), [CoPS](https://openreview.net/forum?id=9W6Z9IeLzc), and [SMITH](https://openreview.net/forum?id=JnwClln80Q): memory as reusable strategies, not transcript storage.
6. **Best production substrate direction** — [MemArt / KVCache-Centric Memory](https://openreview.net/forum?id=YolJOZOGhI), [Mem0](https://github.com/mem0ai/mem0), [Letta](https://github.com/letta-ai/letta), [Zep / Graphiti](https://github.com/getzep/graphiti), [Cognee](https://github.com/topoteretes/cognee), [AgentMemory](https://github.com/rohitg00/agentmemory) for coding-agent trace/session memory, and [Anthropic context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) for production compaction/note-taking/subagent patterns.
7. **Best new retrieval/context interface direction** — [DCI / Direct Corpus Interaction](https://arxiv.org/abs/2605.05242) and [DCI-Agent-Lite](https://github.com/DCI-Agent/DCI-Agent-Lite): raw-corpus tool interaction as a higher-resolution alternative to fixed top-k retrievers.

---

## 1. The master comparison table

| Paper / system | Core memory object | Write policy | Retrieval policy | Update / forget | Evaluation | Main win | Main weakness |
|---|---|---|---|---|---|---|---|
| [MemoryAgentBench](https://openreview.net/forum?id=DT7JyQC3MR) | Benchmark chunks + injected facts | n/a | Measures systems rather than proposing one | Tests conflict / forgetting via FactConsolidation | Accurate Retrieval, Test-Time Learning, Long-Range Understanding, Selective Forgetting | Defines the field's missing eval axis | Benchmark still mostly text-centric |
| [A-Mem](https://openreview.net/forum?id=FiM0M8gcct) | Atomic Zettelkasten note with content, timestamp, keywords, tags, contextual description, links | LLM constructs enriched note | Semantic + linked-note graph traversal | New memories can evolve old memories | Long-term conversation datasets across six foundation models | Agentic organization, not fixed schema | LLM write/update cost; risk of self-reinforcing bad links |
| [LightMem](https://openreview.net/forum?id=dyJ0GWpjJB) | Sensory groups → topic short-term summaries → long-term consolidated store | Lightweight online filtering + offline sleep consolidation | Topic-aware retrieval | Sleep-time update | LongMemEval with GPT/Qwen | Strong cost-performance: up to 10.9% accuracy gain, up to 117× token reduction, 159× API-call reduction, 12× runtime reduction | Less expressive than graph / experience systems |
| [SMITH](https://openreview.net/forum?id=JnwClln80Q) | Procedural + semantic + episodic memories; generated tools | Tool creation loop + experience storage | Episodic similarity + hierarchical memory | Curriculum-driven capability expansion | GAIA | 81.8% Pass@1 vs Alita 75.2 and Memento 70.9 | Complex system; harder to isolate memory contribution |
| [CoPS](https://openreview.net/forum?id=9W6Z9IeLzc) | Prior task experiences | Selects distribution-matched experiences | Pessimism-based experience selection | Reuse, not full lifecycle mgmt | AlfWorld, WebShop, HotPotQA | Provable selection under distribution shift | Mostly experience selection, not full memory OS |
| [ReMemR1](https://openreview.net/forum?id=1cymflI2Lh) | Callback-enhanced memory history | RL-trained memory actions | Selectively revisits entire memory history | RLMLR dense rewards guide memory use | Long-document QA | Breaks forward-only "memorize while reading" limit | Training cost; long-doc QA focus |
| [MEMENTO embodied](https://openreview.net/forum?id=E5L43l5EIu) | Personalized episodic/user-profile KG | Records object semantics + user routines | Hierarchical KG retrieval | Separates personalized knowledge | Embodied single-memory / joint-memory tasks | Identifies overload + coordination failures | Embodied benchmark-specific |
| [HexMachina](https://openreview.net/forum?id=V0Fb4pwhS4) | Executable artifacts / evolved strategies | Preserve and refine code players | Retrieve/use compiled strategy artifacts | Continual artifact evolution | Catanatron | Artifact memory beats prompt-per-turn memory | Withdrawn; narrow Catan setting |
| [EgoMem](https://openreview.net/forum?id=9QYA3DiPl8) | Audiovisual identity + user facts/preferences/relations | Async dialog-boundary extraction | Face+voice user ID plus long-term context retrieval | Async memory manager | Real-time omnimodal dialog | >95% retrieval/management module accuracy; >87% fact consistency | Specialized hardware/data assumptions |
| [MEM1](https://openreview.net/forum?id=jJ6F1sDn9i) | Compact shared internal state | RL updates constant memory | Uses state for reasoning + consolidation | Learns to discard irrelevant/redundant info | Retrieval QA, web QA, web shopping | 3.5× performance and 3.7× memory reduction vs Qwen2.5-14B baseline in reported task | Less interpretable than explicit memory |
| [GraphMind](https://openreview.net/forum?id=XromAiEaE3) | Incremental knowledge graph | LLM builds graph from environment interactions | Graph retrieval for planning | Incremental KG construction | Partially observable navigation | Good fit for POMDP-like planning | Withdrawn; limited public detail |
| [ChemAgent](https://openreview.net/forum?id=kuhIqeVg0e) | Plan, execution, and knowledge memory libraries | Decompose chemical tasks into reusable subtask memory | Retrieve/refine library entries | Self-updating domain library | SciBench chemistry tasks | Up to 46% GPT-4 gain on chemical reasoning | Domain-specific; relies on task decomposability |
| [Intrinsic Memory Agents](https://openreview.net/forum?id=UbSUxAK3BI) | Role-aligned contextual memory per agent | Evolve from each agent's outputs | Agent-specific memory template | Maintains role/procedural consistency | PDDL, FEVER, ALFWorld + data-pipeline design | Strong consistency in multi-agent systems | Needs safeguards against role-locked blind spots |
| [MemArt](https://openreview.net/forum?id=YolJOZOGhI) | KV-cache blocks | Store turns as reusable latent blocks | Attention-score retrieval in latent KV space | Position-encoding-safe reuse | LoCoMo | +11% accuracy, 91–135× prefill-token reduction | Model/runtime coupling; harder to inspect |
| [ReasoningBank](https://openreview.net/forum?id=jL7fwchScm) | Distilled reasoning strategies from success + failure | Self-judge → extract memory → consolidate | Retrieve relevant reasoning memory | Memory-aware test-time scaling (MaTTS) | Web browsing, SWE benchmarks | Turns experience into scaling dimension | Quality depends on self-judgment and failure attribution |
| [Memento](https://arxiv.org/abs/2508.16153) | Episodic Case Bank | Store trajectories as cases; rewrite memory from feedback | Learned case-selection policy over Memory-Augmented MDP | Online memory-based RL without LLM fine-tuning | GAIA, DeepResearcher, SimpleQA, HLE | Reports GAIA validation 87.88% Pass@3 and strong OOD gains from case memory | Case-bank curation and negative transfer remain hard |
| [Focus / Active Context Compression](https://arxiv.org/abs/2601.07190) | Persistent Knowledge block plus pruned history | Agent decides focus boundaries and compression moments | Current context + retained Knowledge block | Drops raw trace between checkpoints | SWE-bench Lite N=5 | Reports 22.7% token reduction at equal success | Small eval; aggressive pruning can add overhead |
| [DCI / Direct Corpus Interaction](https://arxiv.org/abs/2605.05242) | Raw corpus files as inspectable memory | No offline write; corpus is searched directly | Terminal tools (`rg`, reads, scripts) instead of fixed retriever | Context management via truncation/compaction/summarization in DCI-Agent-Lite | BrowseComp-Plus, BRIGHT/BEIR, multi-hop QA | High-resolution evidence access without embeddings/indexing | Slower and tool-policy dependent |

---

## 2. The seven-axis taxonomy

| Axis | Low end | Middle | High end | Best exemplars |
|---|---|---|---|---|
| **Representation** | Raw transcript | Structured note / summary | Graph / KV / parameter / executable artifact | A-Mem, GraphMind, MemArt, HexMachina |
| **Control** | Append-only | Heuristic filter/update | Agentic or RL-learned policy | A-Mem, ReMemR1, MEM1 |
| **Abstraction** | Episodes | Reflections | Cross-trajectory experience / reusable strategy | ReasoningBank, CoPS, SMITH |
| **Temporal awareness** | Timestamp only | Recency decay | Validity, contradiction, selective forgetting | MemoryAgentBench FactConsolidation, Zep/Graphiti |
| **Modality** | Text | Tool traces / code | Audio-video embodied memory | EgoMem, MEMENTO embodied |
| **Efficiency** | Full context | RAG top-k | Constant state / KV-cache reuse / sleep-time consolidation | MEM1, MemArt, LightMem |
| **Evaluation** | Static QA | Multi-turn recall | Interactive task success under memory stress | MemoryAgentBench, GAIA/SMITH, embodied MEMENTO |

### The key shift

Earlier memory systems asked: **"What should I retrieve?"**

The 2026 frontier asks:

1. **What should become memory at all?**
2. **What should be updated, contradicted, expired, or forgotten?**
3. **Which memory object should be promoted from episode to strategy?**
4. **Should retrieval happen in natural language, graph space, latent KV space, or executable artifact space?**
5. **Can the memory policy itself be learned?**

---

## 3. Expandable paper cards

<details>
<summary><strong>MemoryAgentBench — the benchmark that defines what "memory agent" should mean</strong></summary>

**Source:** [OpenReview](https://openreview.net/forum?id=DT7JyQC3MR), [arXiv / repo search result](https://github.com/HUST-AI-HYZ/MemoryAgentBench)

**Core idea.** Memory must be evaluated as incremental multi-turn interaction, not as static long-context QA. The authors define four competencies:

1. **Accurate retrieval** — can the agent find the right fact?
2. **Test-time learning** — can it learn new skills or preferences during interaction?
3. **Long-range understanding** — can it integrate globally across long histories?
4. **Selective forgetting / conflict resolution** — can it update stale facts and avoid outdated recall?

**Important results.**

- The paper reports that current systems fail to master all four competencies.
- Chunk-size ablations show a tradeoff: smaller chunks help accurate retrieval, but can hurt long-range understanding because the task requires integration over coherent context.
- Backbone-model ablations suggest stronger LLMs marginally help RAG systems once the backbone is strong, but agentic memory methods can benefit more from stronger backbones.
- FactConsolidation remains hard under longer contexts: models solve shorter 6K versions far better than 32K versions.

**Research implication.** A new memory paper should report results on at least the four MemoryAgentBench axes, otherwise it risks optimizing only retrieval while missing update, forgetting, and long-range integration.

</details>

<details>
<summary><strong>A-Mem — Zettelkasten as agentic memory</strong></summary>

**Source:** [OpenReview](https://openreview.net/forum?id=FiM0M8gcct), [paper PDF text via OpenReview](https://openreview.net/pdf?id=FiM0M8gcct), [WujiangXu/AgenticMemory](https://github.com/WujiangXu/AgenticMemory), [WujiangXu/A-mem-sys](https://github.com/WujiangXu/A-mem-sys)

**Core idea.** Replace fixed memory schema with **agentic note construction, link generation, and memory evolution**. Each note contains original content, timestamp, LLM-generated keywords, tags, contextual description, embedding, and links. New memories can cause older memories to update their contextual descriptions or attributes.

**Why it matters.** It upgrades memory from "retrieve similar chunk" to "grow a network of ideas". This matches how researchers use Zettelkasten: memories are atomic, linked, and reinterpreted as the corpus grows.

**Best use case.** Research assistants, personal knowledge bases, coding agents, and scientific agents where relationships between memories matter more than raw chronological recall.

**Failure mode.** If the LLM writes wrong links or bad contextual descriptions, the system can create a self-reinforcing false topology.

</details>

<details>
<summary><strong>LightMem — efficient cognitive memory with sleep-time updates</strong></summary>

**Source:** [OpenReview](https://openreview.net/forum?id=dyJ0GWpjJB), [zjunlp/LightMem](https://github.com/zjunlp/LightMem)

**Core idea.** Inspired by Atkinson–Shiffrin: sensory memory filters quickly, short-term memory consolidates topic groups, long-term memory updates offline during "sleep". The decisive move is **decoupling expensive consolidation from online inference**.

**Reported win.** OpenReview abstract reports up to **10.9% accuracy gains**, **117× token reduction**, **159× API-call reduction**, and **12× runtime reduction** on LongMemEval with GPT/Qwen backbones.

**Research implication.** Many memory papers over-optimize accuracy without a serious online cost model. LightMem makes latency/API-call/token budgets part of the core contribution.

</details>

<details>
<summary><strong>SMITH — memory plus dynamic tools</strong></summary>

**Source:** [OpenReview](https://openreview.net/forum?id=JnwClln80Q)

**Core idea.** Organize memory into procedural, semantic, and episodic components, then unify **dynamic tool creation** with **cross-task experience sharing**. Tool generation happens in a controlled sandbox; experience retrieval supplies prior patterns.

**Reported win.** The OpenReview abstract reports **81.8% Pass@1 on GAIA**, ahead of Alita **75.2%** and Memento **70.9%**.

**Research implication.** The next frontier is not memory alone but **memory as capability expansion**: memories become tools, routines, and executable code.

</details>

<details>
<summary><strong>CoPS — provable cross-task experience selection</strong></summary>

**Source:** [OpenReview](https://openreview.net/forum?id=9W6Z9IeLzc), [arXiv](https://arxiv.org/abs/2410.16670), [uclaml/COPS](https://github.com/uclaml/cops)

**Core idea.** Select past experiences using a distribution-matched, pessimism-based strategy to maximize utility while minimizing distribution-shift risk.

**Why it matters.** Most memory retrieval assumes similarity is good. CoPS says: similar but distribution-shifted experience can hurt, so retrieval needs conservative selection.

**Research implication.** Memory retrieval should report **negative-transfer rate**, not just recall accuracy.

</details>

<details>
<summary><strong>ReMemR1 — revisitable memory for long-context reasoning</strong></summary>

**Source:** [OpenReview](https://openreview.net/forum?id=1cymflI2Lh), [arXiv](https://arxiv.org/html/2509.23040v1), [syr-cn/ReMemR1](https://github.com/syr-cn/ReMemR1)

**Core idea.** Forward-only "memorize while reading" loses evidence. ReMemR1 adds callback-enhanced memory so the agent can selectively revisit earlier evidence, then trains memory use with **Reinforcement Learning with Multi-Level Rewards**.

**Why it matters.** Memory is not just compression; it is a **navigation policy over prior evidence**.

**Research implication.** The next benchmark should track "evidence revisit correctness": did the agent look back at the right moment?

</details>

<details>
<summary><strong>MEMENTO embodied — personalized memory in physical environments</strong></summary>

**Source:** [OpenReview](https://openreview.net/forum?id=E5L43l5EIu), [arXiv](https://arxiv.org/html/2505.16348v2), [Connoriginal/MEMENTO](https://github.com/Connoriginal/MEMENTO)

**Core idea.** Evaluate embodied personalization through object semantics and user-pattern memory. The paper identifies two bottlenecks: **information overload** and **coordination failure** when multiple memories must jointly inform planning.

**Proposed direction.** Hierarchical knowledge-graph user profile memory separates personalized knowledge and improves both single-memory and joint-memory tasks.

**Research implication.** Memory systems need tests for **multi-memory coordination**, not only single fact recall.

</details>

<details>
<summary><strong>EgoMem — full-duplex omnimodal lifelong memory</strong></summary>

**Source:** [OpenReview](https://openreview.net/forum?id=9QYA3DiPl8), [arXiv](https://arxiv.org/html/2509.11914v2)

**Core idea.** A real-time memory agent for raw audiovisual streams. It runs three async processes: user identification + retrieval, omnimodal dialog generation, and memory management via dialog-boundary detection and fact extraction.

**Reported win.** The abstract reports **>95% accuracy** for retrieval and memory-management modules and **>87% fact consistency** in real-time personalized dialogs.

**Research implication.** Text-only memory is no longer sufficient for embodied agents. The memory write path must handle identity, modality alignment, and privacy.

</details>

<details>
<summary><strong>MEM1 — constant-memory learned consolidation</strong></summary>

**Source:** [OpenReview](https://openreview.net/forum?id=jJ6F1sDn9i), [arXiv](https://arxiv.org/html/2506.15841), [MIT-MI/MEM1](https://github.com/MIT-MI/MEM1)

**Core idea.** Train a 7B agent to update a compact shared internal state across turns, jointly supporting memory consolidation and reasoning while discarding irrelevant/redundant information.

**Reported win.** The abstract reports **3.5× performance improvement** and **3.7× memory reduction** compared to Qwen2.5-14B-Instruct on a 16-objective multi-hop QA task.

**Research implication.** Long-horizon agents may not need ever-growing stores if the memory controller learns a sufficient statistic.

</details>

<details>
<summary><strong>ChemAgent — domain memory as self-updating task library</strong></summary>

**Source:** [OpenReview](https://openreview.net/forum?id=kuhIqeVg0e), [arXiv](https://arxiv.org/abs/2501.06590), [gersteinlab/ChemAgent](https://github.com/gersteinlab/chemagent)

**Core idea.** Chemical reasoning tasks are decomposed into reusable subtasks and compiled into three memory types: plan, execution, and knowledge memory.

**Reported win.** The abstract reports up to **46% GPT-4 performance gain** on four SciBench chemistry datasets.

**Research implication.** In high-structure domains, memory should preserve **procedural decomposition**, not just final answers.

</details>

</details>

---

## 4. Benchmark landscape

| Benchmark | What it tests | Why old memory systems fail | Best suited papers |
|---|---|---|---|
| **MemoryAgentBench** | Accurate retrieval, test-time learning, long-range understanding, selective forgetting | Static RAG can retrieve but not update/forget/integrate | MemoryAgentBench, LightMem, A-Mem, MEM1 |
| **LongMemEval / LoCoMo** | Long-term conversational memory | Full context is expensive; naive retrieval misses temporal/user facts | LightMem, Mem0, Zep, Letta, MemArt |
| **GAIA** | General agent problem solving | Memory must become tools and reusable experience | SMITH, Memento-CBR, ReasoningBank |
| **WebArena / SWE-Bench** | Web/coding agent trajectories | Raw trajectory storage repeats errors; needs strategy distillation | ReasoningBank, CoPS |
| **Embodied MEMENTO** | Personalized embodied object/routine memory | Single-memory retrieval fails under multi-memory coordination | MEMENTO embodied, EgoMem |
| **PDDL / FEVER / ALFWorld** | Multi-agent role consistency and procedural memory | Agents lose role adherence and procedural integrity | Intrinsic Memory Agents |
| **SciBench chemistry** | Domain-specific multi-step reasoning | General memory lacks procedural/knowledge decomposition | ChemAgent |

---

## 5. Current techniques, grouped by memory lifecycle

### 5.1 Construction: what becomes memory?

| Technique | Mechanism | Papers |
|---|---|---|
| Raw trajectory logging | Store every observation/action | Reflexion, Generative Agents baseline, early MemGPT |
| Importance-scored observation | LLM scores importance before retrieval / reflection | Generative Agents |
| Structured note construction | Convert event to typed note with metadata | A-Mem, ReasoningBank |
| Domain decomposition | Convert task into plan/execution/knowledge units | ChemAgent |
| Multimodal extraction | Detect dialog boundaries and extract user facts from AV stream | EgoMem |
| Learned state update | Train policy to compress current observation into compact state | MEM1 |
| KV-cache storage | Store latent attention KV blocks rather than text | MemArt |

### 5.2 Organization: how is memory arranged?

| Organization | Strength | Weakness | Exemplars |
|---|---|---|---|
| Chronological stream | Simple and auditable | Retrieval burden grows | Generative Agents |
| Vector store | Cheap semantic lookup | Temporal contradiction blind | Mem0, many RAG systems |
| Hierarchical tiers | Good cost/latency control | Promotion/demotion policy hard | MemGPT, Letta, LightMem |
| Knowledge graph | Multi-hop / relational structure | Schema and extraction errors | GraphMind, MEMENTO, Zep/Graphiti |
| Zettelkasten network | Flexible evolving relationships | LLM-linking quality risk | A-Mem |
| Executable artifacts | Turns experience into capability | Requires sandbox/eval | SMITH, HexMachina, Voyager |
| Latent KV blocks | Maximum inference efficiency | Opaque and model-coupled | MemArt |

### 5.3 Retrieval: how is memory selected?

| Retrieval strategy | Where it shines | Where it fails |
|---|---|---|
| BM25 / lexical | Exact entity/fact lookup | Paraphrases and abstraction |
| Dense embedding | Semantic paraphrase | Contradictions, stale facts |
| Hybrid RRF | Robust production default | Still shallow for multi-hop |
| Entity-link graph traversal | Multi-hop and relationship QA | Entity extraction errors |
| PPR graph fusion | Best for graph + dense multi-hop retrieval | Requires KG construction |
| Pessimism / distribution matching | Avoiding negative transfer | Needs uncertainty/coverage estimates |
| Learned callback | Long-context evidence revisiting | Training cost |
| KV attention retrieval | Efficient latent reuse | Poor interpretability |

### 5.4 Update, consolidation, and forgetting

| Technique | Description | Missing piece |
|---|---|---|
| Reflection | Write critique after attempts | Can hallucinate lessons |
| Sleep-time consolidation | Offline updates outside online inference | Scheduling and stale-state races |
| Memory evolution | New notes update old notes | Needs provenance and rollback |
| Selective forgetting | Drop outdated or irrelevant memory | Hard to prove safe deletion |
| Temporal validity | Preserve history but mark valid/invalid intervals | Needs query-time temporal filters |
| Experience abstraction | Cluster trajectories into reusable strategy | Needs generalization tests |
| Learned compression | Train compact state update | Interpretability and safety |

---

## 6. The strongest 2026 architecture: recommended hybrid

If you were building a new memory system today, the strongest design is not any one paper. It is this combination:

1. **Hot working memory** — recent turn, current plan, active goals, active constraints.
2. **Warm explicit memory** — A-Mem-style structured notes with tags, contextual descriptions, links, provenance, validity intervals.
3. **Cold graph memory** — Zep/Graphiti/HippoRAG-style temporal entity graph for multi-hop and contradiction-aware retrieval.
4. **Experience bank** — ReasoningBank/CoPS-style distilled strategies from success and failure, with negative-transfer safeguards.
5. **Procedural/artifact memory** — SMITH/Voyager/ChemAgent-style executable tools, plans, and domain routines.
6. **Efficiency layer** — LightMem sleep-time consolidation and optional MemArt KV-cache reuse for high-volume conversational agents.
7. **Learned controller** — MEM1/ReMemR1-style policy for when to read, write, revisit, compress, and forget.
8. **Evaluation harness** — MemoryAgentBench + LongMemEval/LoCoMo + domain task success + cost/latency + memory-poisoning tests.

### Pseudocode shape

```python
def agent_step(observation, goal):
    hot = working_context.recent(goal)
    query = controller.formulate_memory_query(observation, goal, hot)

    explicit = warm_notes.retrieve(query, filters={"valid_at": now(), "scope": scope})
    graph = temporal_graph.retrieve(query, mode="ppr_fusion")
    experience = reasoning_bank.retrieve(query, require_distribution_match=True)
    tools = procedural_memory.retrieve(query)

    action = llm.act(observation, goal, hot, explicit, graph, experience, tools)
    outcome = environment.apply(action)

    write_decision = controller.should_write(observation, action, outcome)
    if write_decision:
        note = note_constructor.to_structured_note(observation, action, outcome)
        warm_notes.add(note)
        warm_notes.evolve_neighbors(note)
        temporal_graph.upsert(note)

    if controller.should_consolidate():
        reasoning_bank.distill_recent(successes=True, failures=True)
        procedural_memory.promote_verified_routines()

    return action, outcome
```

---

## 7. Deep gap map: where the next paper can win

| Gap | Why it matters | Existing partial solution | Paper idea |
|---|---|---|---|
| **Memory write correctness** | Bad writes poison all future turns | A-Mem, ReasoningBank self-judgment | Verifier-gated memory writes with calibrated abstention |
| **Negative transfer** | Similar memories can hurt new tasks | CoPS pessimism | Benchmark negative-transfer rate under shifted tasks |
| **Temporal contradiction** | Old facts remain semantically relevant | MemoryAgentBench FactConsolidation, Graphiti | Temporal validity + contradiction-aware retrieval in agent loops |
| **Multi-memory coordination** | Several correct memories can conflict in planning | MEMENTO embodied | Memory composition benchmark: solve tasks requiring 3–5 memories jointly |
| **Learned forgetting** | Stores grow, noise accumulates | MEM1, LightMem | RL policy for safe delete/update with audit log |
| **Interpretability of latent memory** | KV-cache memory is efficient but opaque | MemArt | Hybrid latent+text memory with inspectable anchors |
| **Cross-modal identity memory** | Embodied assistants need person-specific recall | EgoMem | Privacy-preserving multimodal memory with revocation |
| **Memory security** | Retrieved content can prompt-inject or poison | Existing docs 269–273 | Memory firewall: classify, quarantine, and expire suspicious memories |
| **Cost-aware routing** | Memory retrieval can exceed latency budgets | LightMem | Learned router among hot/warm/cold/KV stores |
| **Evaluation saturation** | Static recall benchmarks are insufficient | MemoryAgentBench | End-to-end MemoryArena for tool-using agents with cost + safety |

---

## 8. Twenty next-paper ideas

### Idea 1 — Verifier-Gated Agentic Memory

Combine A-Mem's note evolution with a verifier that checks each proposed write/update/link against source evidence. Report memory precision, recall, contradiction rate, and downstream task success.

### Idea 2 — Memory Negative Transfer Benchmark

Extend CoPS: create tasks where semantically similar past episodes are subtly wrong. Measure whether memory retrieval helps or harms.

### Idea 3 — Temporal Zettelkasten

Add Graphiti-style validity intervals to A-Mem's Zettelkasten notes. Every link has a validity window and confidence, so old memories remain auditable but not always active.

### Idea 4 — Learned Store Router

Train a policy to route each memory query to hot context, vector notes, graph memory, KV cache, or experience bank under a latency budget.

### Idea 5 — Memory Write Unit Tests

For coding agents, every procedural memory must include a small executable test. Promotion from episode to skill requires passing tests.

### Idea 6 — Multimodal Memory Revocation

Build on EgoMem: user can revoke "forget my face/voice/preference" and the system cascades deletion across embeddings, facts, links, and summaries.

### Idea 7 — Multi-Memory Composition Benchmark

Inspired by embodied MEMENTO, design tasks that require combining multiple facts/preferences/routines, where each single memory is insufficient.

### Idea 8 — Failure-First ReasoningBank

ReasoningBank learns from success and failure. Push further: failure memory is first-class, with "do-not-repeat" retrieval and counterfactual repair.

### Idea 9 — Interpretability Layer for KV Memory

MemArt stores KV blocks. Attach a textual "anchor card" to each KV block so latent retrieval can be audited and edited.

### Idea 10 — Sleep-Time Graph Consolidation

Combine LightMem sleep-time updates with graph consolidation: batch merge duplicate entities, resolve contradictions, and promote repeated patterns to semantic memory.

### Idea 11 — Memory Poisoning Red Team Suite

Create adversarial memories that look relevant but instruct the agent to violate policy, use stale API docs, or reveal secrets. Score quarantine rate and safe fallback.

### Idea 12 — Agent-Specific Role Memory with Cross-Agent Fact Consensus

Extend Intrinsic Memory Agents: each agent keeps role memory, but factual claims go through shared consensus memory to prevent role-specific hallucinations.

### Idea 13 — Procedural Memory Distillation for Science Agents

Generalize ChemAgent: task decomposition memories for chemistry, biology, math, and law with domain-specific validators.

### Idea 14 — Memory-Aware Test-Time Scaling Budget Law

Extend ReasoningBank/MaTTS: quantify how much extra trajectory sampling improves memory quality and where diminishing returns begin.

### Idea 15 — Causal Memory Graphs for Planning

GraphMind builds KGs; add causal edges: action → delayed outcome. Evaluate in partially observable environments.

### Idea 16 — Sufficient-State Memory for Tool Agents

MEM1 learns compact state. Adapt it to tool-using agents where state must preserve tool outputs, errors, permissions, and side effects.

### Idea 17 — Memory Provenance as First-Class Token Budget

Every recalled memory includes provenance. Study how much provenance is needed before recall becomes trustworthy without overloading context.

### Idea 18 — Forgetting Without Deleting

Temporal soft deletion: stale memories remain in audit storage but are excluded from normal retrieval unless the user asks historical questions.

### Idea 19 — Cross-Project Coding-Agent Memory

Design memory that transfers across repos without leaking secrets: extract generic debugging strategies, not proprietary code snippets.

### Idea 20 — Unified Memory Control Language

Define a minimal DSL for `remember`, `recall`, `revise`, `expire`, `promote`, `link`, `unlink`, `verify`, and `audit`, compatible with Mem0/Letta/Zep/Cognee.

---

## 9. What to cite in the next paper

### Foundational memory agents

- [Generative Agents](https://arxiv.org/abs/2304.03442) — memory stream with recency, importance, relevance, reflection, planning.
- [Reflexion](https://arxiv.org/abs/2303.11366) — verbal self-reflection as episodic memory.
- [MemGPT / Letta](https://github.com/letta-ai/letta) — OS-inspired tiered context management.
- [Voyager](https://arxiv.org/abs/2305.16291) — executable skill library as procedural memory.

### 2025–2026 explicit memory systems

- [A-Mem](https://openreview.net/forum?id=FiM0M8gcct) — dynamic Zettelkasten memory.
- [LightMem](https://openreview.net/forum?id=dyJ0GWpjJB) — efficient cognitive-stage memory.
- [Mem0](https://github.com/mem0ai/mem0) — production long-term memory layer.
- [AgentMemory](https://github.com/rohitg00/agentmemory) — MCP/REST/hook-based persistent memory for coding agents; repo-reported 5.3K★ and 95.2% LongMemEval-S R@5.
- [Zep / Graphiti](https://github.com/getzep/graphiti) — temporal knowledge graph memory.
- [Cognee](https://github.com/topoteretes/cognee) — GraphRAG memory with remember/recall/forget/improve.

### Survey / taxonomy anchors

- [Memory in the Age of AI Agents](https://arxiv.org/abs/2512.13564) — 107-page survey that reframes agent memory through **forms** (token-level / parametric / latent), **functions** (factual / experiential / working), and **dynamics** (formation / evolution / retrieval); useful for positioning a new memory paper beyond the older short-term/long-term vocabulary.

### Learned and latent memory

- [MEM1](https://openreview.net/forum?id=jJ6F1sDn9i) — RL constant-memory state.
- [ReMemR1](https://openreview.net/forum?id=1cymflI2Lh) — callback-enhanced revisitable memory.
- [MemArt](https://openreview.net/forum?id=YolJOZOGhI) — KV-cache-centric memory.

### Experience and self-evolution

- [ReasoningBank](https://openreview.net/forum?id=jL7fwchScm) — distilled reasoning memory + MaTTS.
- [CoPS](https://openreview.net/forum?id=9W6Z9IeLzc) — provable cross-task experience selection.
- [SMITH](https://openreview.net/forum?id=JnwClln80Q) — memory + dynamic tool creation.
- [ChemAgent](https://openreview.net/forum?id=kuhIqeVg0e) — domain self-updating memory.

### Surveys

- [From Storage to Experience](https://arxiv.org/html/2605.06716) — Storage → Reflection → Experience evolutionary framing.
- [Memory for Autonomous LLM Agents](https://www.arxiv.org/pdf/2603.07670) — write/manage/read loop taxonomy.
- [LLM Agent Memory: Unified Representation–Management Perspective](https://openreview.net/forum?id=KPs1EgGKcT) — natural language / intermediate representation / parameter paradigms.
- [Anatomy of Agentic Memory](https://arxiv.org/abs/2602.19320v1) — taxonomy and empirical limitations.

---

## 10. Recommended thesis for a new paper

> **A next-generation memory agent should treat memory as a verifiable, temporally-aware, cost-constrained control policy over heterogeneous stores, not as a passive RAG database.**

The strongest concrete contribution would combine four currently separated threads:

1. **A-Mem** dynamic note/link evolution.
2. **Graphiti/Zep** temporal validity and contradiction handling.
3. **CoPS/ReasoningBank** experience abstraction with negative-transfer safeguards.
4. **LightMem/MEM1** cost-aware learned memory control.

The paper could be titled:

> **VeriMem: Verifier-Gated Temporal Agentic Memory with Negative-Transfer-Aware Experience Retrieval**

Minimum experiments:

| Experiment | Dataset | Metric |
|---|---|---|
| MemoryAgentBench | AR / TTL / LRU / conflict | Accuracy + cost |
| LongMemEval / LoCoMo | Conversational memory | F1 / exact match / latency |
| WebArena or SWE-Bench subset | Experience transfer | Task success + repeated-error reduction |
| Adversarial memory poisoning | Synthetic + real docs | Poison block rate + safe fallback |
| Ablation | no verifier / no temporal / no CoPS / no sleep update | Contribution isolation |

---

## 11. Cross-links inside this corpus

Read these existing local chapters next:

- [09 — Memory Files](09-memory-files.md)
- [72 — Claude-Mem Persistent Memory Compression](72-claude-mem-persistent-memory-compression.md)
- [100 — Contextual Memory Is a Memo](100-contextual-memory-is-a-memo.md)
- [106–109 — Memento series](106-memento-paper-theory.md)
- [151–153 — MEMTIER trilogy](151-memtier-why-flat-memory-breaks-at-72-hours.md)
- [157 — May 2026 Memory + Skills Synthesis](157-may-2026-synthesis-memory-and-skills.md)
- [182 — Memory Frontiers 2026](182-memory-frontiers-2026.md)
- [183 — OSS Memory Landscape](183-oss-memory-landscape-may-2026.md)
- [184 — Strongest Memory Techniques](184-strongest-memory-techniques-synthesis-may-2026.md)
- [187 — Multi-Agent Shared Memory Landscape](187-multi-agent-shared-memory-landscape.md)
- [188 — Witness / Provenance Memory](188-witness-provenance-memory-techniques-synthesis.md)
- [233 — Memory Scaling for Agents](233-memory-scaling-for-agents.md)

---

## Sources

- [MemoryAgentBench — OpenReview](https://openreview.net/forum?id=DT7JyQC3MR)
- [A-Mem — OpenReview](https://openreview.net/forum?id=FiM0M8gcct)
- [LightMem — OpenReview](https://openreview.net/forum?id=dyJ0GWpjJB)
- [SMITH — OpenReview](https://openreview.net/forum?id=JnwClln80Q)
- [CoPS — OpenReview](https://openreview.net/forum?id=9W6Z9IeLzc)
- [ReMemR1 — OpenReview](https://openreview.net/forum?id=1cymflI2Lh)
- [MEMENTO embodied — OpenReview](https://openreview.net/forum?id=E5L43l5EIu)
- [HexMachina — OpenReview](https://openreview.net/forum?id=V0Fb4pwhS4)
- [EgoMem — OpenReview](https://openreview.net/forum?id=9QYA3DiPl8)
- [MEM1 — OpenReview](https://openreview.net/forum?id=jJ6F1sDn9i)
- [GraphMind — OpenReview](https://openreview.net/forum?id=XromAiEaE3)
- [ChemAgent — OpenReview](https://openreview.net/forum?id=kuhIqeVg0e)
- [Intrinsic Memory Agents — OpenReview](https://openreview.net/forum?id=UbSUxAK3BI)
- [KVCache-Centric Memory / MemArt — OpenReview](https://openreview.net/forum?id=YolJOZOGhI)
- [ReasoningBank — OpenReview](https://openreview.net/forum?id=jL7fwchScm)
- [LLM Agent Memory Survey — OpenReview](https://openreview.net/forum?id=KPs1EgGKcT)
- [From Storage to Experience — arXiv](https://arxiv.org/html/2605.06716)
- [Memory for Autonomous LLM Agents — arXiv](https://www.arxiv.org/pdf/2603.07670)
- [Anatomy of Agentic Memory — arXiv](https://arxiv.org/abs/2602.19320v1)
- [Memory in the LLM Era — arXiv](https://arxiv.org/html/2604.01707v2)
