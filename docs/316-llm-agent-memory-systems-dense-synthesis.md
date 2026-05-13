# 316 — LLM Agent Memory Systems: Dense Synthesis (Long-Term Conversational, Episodic, Semantic, MemGPT, A-Mem, Mem0)

**Scope.** A single dense Markdown synthesis focused only on **LLM agent memory systems**: long-term conversational memory, episodic/semantic/procedural memory, MemGPT/Letta-style virtual context, A-Mem/Zettelkasten memory, Mem0-style production memory, Zep/Graphiti temporal memory, Cognee/GraphRAG memory, LightMem efficiency, ReasoningBank/CoPS experience memory, MEM1/ReMemR1 learned memory control, Hu et al.'s forms/functions/dynamics survey taxonomy, and evaluation. Tangential RL-only memory benchmarks are excluded except where they inform benchmark design.

**One-line thesis.** The strongest 2026 LLM-agent memory systems are not "RAG over chat logs"; they are **auditable control loops** that decide what to write, how to organize it, what to retrieve, when to revise or forget it, and when to promote raw episodes into semantic rules, procedural skills, or learned state.

---

## 0. Executive Summary

LLM agent memory has moved through four phases:

1. **Transcript memory** — append chat/tool history, summarize when context overflows.
2. **Retrieval memory** — store chunks in vector/BM25 stores and retrieve top-k.
3. **Structured long-term memory** — split memory into episodic, semantic, procedural, preference, failure, and tool memories with metadata, time, provenance, and update rules.
4. **Controlled / learned memory** — agentic or RL-trained policies decide write/read/update/forget/promote, sometimes operating over graphs, latent KV-cache blocks, or executable artifacts.

The practical frontier is a **hybrid architecture**:

- **Hot working memory:** current task, recent turns, active plan, constraints.
- **Warm conversational memory:** user/session/agent facts and preferences, Mem0/Zep-style.
- **Episodic memory:** concrete events, tool calls, failures, observations.
- **Semantic memory:** consolidated stable facts and abstractions.
- **Procedural memory:** reusable routines, tools, code snippets, workflows.
- **Temporal graph memory:** entities/relations/facts with validity windows.
- **Experience memory:** distilled strategies from successful and failed trajectories.
- **Controller:** decides when to read, write, update, forget, consolidate, or abstain.
- **Evaluator:** tests retrieval, learning, long-range understanding, conflict resolution, cost, latency, privacy, and poisoning resistance.

---

## 1. Unified Taxonomy

| Dimension | Low-complexity version | Strong 2026 version | Representative systems |
|---|---|---|---|
| **Temporal scope** | Current prompt only | working + episodic + semantic + procedural + archival memory | Generative Agents, MemGPT/Letta, Mem0, Zep |
| **Storage object** | raw chat chunks | typed memory records with provenance, validity, confidence, links | A-Mem, ReasoningBank, Cognee, Graphiti |
| **Retrieval** | dense top-k | hybrid BM25+dense+graph+temporal filters+rerank | Mem0, Zep, HippoRAG, Cognee |
| **Update** | append-only | add/update/delete/expire/link/evolve/promote | A-Mem, Cognee, Zep/Graphiti, Letta |
| **Control** | fixed workflow | LLM-agentic or learned memory policy | A-Mem, ReMemR1, MEM1 |
| **Abstraction level** | transcript | reflections, semantic facts, experience rules, executable skills | Reflexion, ReasoningBank, Voyager, SMITH |
| **Efficiency** | full context | sleep-time consolidation, compact state, KV-cache reuse | LightMem, MEM1, MemArt |
| **Governance** | no deletion model | provenance, redaction, validity windows, user-level delete | Zep/Graphiti, production Mem0 patterns |
| **Evaluation** | static QA | multi-turn, conflict, long-range, update, cost, poisoning | MemoryAgentBench, LongMemEval, LoCoMo |

### 1.1 Hu et al. 2026: Forms × Functions × Dynamics

[Memory in the Age of AI Agents: A Survey — Forms, Functions and Dynamics](https://arxiv.org/abs/2512.13564) is now the broadest survey anchor for this file. Its core value is not a new system, but a cleaner map that separates three questions that older memory docs often mix together:

| Axis | Categories | How to use it in this synthesis |
|---|---|---|
| **Forms: what carries memory?** | token-level, parametric, latent | separates external text/graph stores from learned weights and KV/hidden-state style memory |
| **Functions: why agents need memory?** | factual, experiential, working | separates user/environment facts from case/strategy/skill memory and active task state |
| **Dynamics: how memory operates?** | formation, evolution, retrieval | aligns with the write/manage/read loop: summarize/distill/construct/internalize, consolidate/update/forget, retrieve/rerank/post-process |

This survey also explicitly distinguishes **agent memory** from adjacent ideas such as LLM memory, RAG, and context engineering, which is useful when positioning a new paper: if the contribution is "better retrieval over documents," it is RAG; if it changes what the agent forms, evolves, retrieves, or internalizes across interactions, it is agent memory.

---

## 2. Memory Types That Matter for LLM Agents

| Type | Definition | Example | Storage shape | Failure mode |
|---|---|---|---|---|
| **Working memory** | What is actively in the current context | current plan, last tool result, active user request | prompt/context state | overflow, distraction, lost goal |
| **Episodic memory** | Specific past events | "On May 10, pytest failed because env var X was missing" | timestamped event record | too many noisy episodes |
| **Semantic memory** | Stable abstracted knowledge | "This repo uses uv, not pip" | claim/fact record with evidence | stale facts, overgeneralization |
| **Procedural memory** | Reusable method or skill | "To release Lyra, run A then B then verify C" | checklist, code, workflow, tool | outdated procedures |
| **Preference memory** | User or team preferences | "User prefers dense Markdown" | scoped user fact | privacy risk, over-personalization |
| **Failure memory** | Mistakes and counterexamples | "Do not use API v1; it silently drops metadata" | lesson with trigger conditions | pessimistic blockage |
| **Social / role memory** | Multi-agent identity and responsibilities | "Reviewer agent checks tests, Planner agent writes specs" | role profile | role lock-in, siloed facts |
| **Latent memory** | Hidden-state or KV representation | retrieved KV-cache blocks | model/runtime-native block | opacity, poor editability |
| **Artifact memory** | Executable persistent output | generated tools, scripts, skill files | code + tests + metadata | unsafe or brittle generated code |

---

## 3. Lifecycle: Write → Manage → Read → Act → Learn

### 3.1 Write Path

Naive systems write every turn. Strong systems write selectively:

```text
observation/tool_result/user_turn
  -> candidate extraction
  -> salience scoring
  -> scope assignment: user/session/agent/project/global
  -> type assignment: episodic/semantic/procedural/preference/failure
  -> evidence/provenance attachment
  -> contradiction check
  -> privacy/security filter
  -> add/update/expire/reject
```

**Best practice:** every durable memory should answer:

- What is the claim/event/procedure?
- Who/what is it about?
- When was it true?
- Why was it written?
- What evidence supports it?
- What can supersede or delete it?
- Which future tasks should retrieve it?

### 3.2 Manage Path

Memory management is where most systems differ.

| Management operation | Purpose | Systems / papers |
|---|---|---|
| **Summarize** | compress many turns | context compaction, LightMem |
| **Deduplicate** | reduce repeated facts | Mem0-style production pipelines |
| **Link** | create relationship topology | A-Mem, graph memory |
| **Evolve** | update old memory when new memory arrives | A-Mem |
| **Expire** | mark old fact no longer valid | Graphiti/Zep-style temporal KG |
| **Forget** | remove or hide memory | Cognee, privacy-aware production memory |
| **Consolidate** | promote episodes to semantic/procedural memory | Generative Agents, LightMem, ReasoningBank |
| **Distill** | extract general reasoning strategies | ReasoningBank, CoPS |
| **Promote to artifact** | convert pattern into executable skill/tool | Voyager, SMITH, ChemAgent |

### 3.3 Read Path

Strong retrieval is a pipeline, not a single vector search:

```text
task/query/current_state
  -> decide whether memory is needed
  -> route to store(s): hot/warm/graph/procedural/experience
  -> retrieve: BM25 + dense + graph + temporal filters
  -> rerank with task relevance and confidence
  -> check recency/validity/contradiction
  -> attach provenance
  -> pack into prompt under token budget
  -> act
```

---

## 4. Main Systems and Papers

### 4.1 Generative Agents

**Primitive:** memory stream + reflection + planning.  
**Why it matters:** established the canonical memory formula: retrieve by **recency + importance + relevance**, periodically reflect into higher-level summaries, then plan from retrieved memory.  
**Limit:** mostly simulation/social behavior; not a production memory control plane.

### 4.2 Reflexion

**Primitive:** verbal self-reflection.  
**Why it matters:** showed that natural-language postmortems can improve future attempts without fine-tuning.  
**Limit:** reflections can hallucinate lessons, overfit to a prior failure, or be retrieved in the wrong setting.

### 4.3 Voyager

**Primitive:** executable skill library.  
**Why it matters:** procedural memory can be code, not text. Verified routines become reusable capability.  
**Limit:** safe promotion requires tests, sandboxing, and versioning.

### 4.4 MemGPT / Letta

**Primitive:** OS-inspired virtual context and tiered memory.  
**Core idea:** model context is like RAM; archival/recall memory is external storage; the agent uses tools to page memory in/out.  
**Strength:** makes memory operations explicit and traceable.  
**Weakness:** memory tool calls add latency and require strong policies for when to page.

**Design lesson:** expose memory operations as first-class tool calls:

```python
memory.search_archival(query)
memory.write_archival(text, metadata)
memory.update_core_block(name, content)
memory.recall_recent(filter)
```

### 4.5 Mem0

**Primitive:** production-ready long-term user/session/agent memory.  
**Core idea:** extract durable facts/preferences from conversations, store them by scope, retrieve them for personalization.  
**Strength:** simple integration model and strong product fit for assistants, support bots, and conversational apps.  
**Weakness:** a pure personalization layer is not enough for deep temporal reasoning, procedural memory, or causal task memory.

**Best use:** user preference memory, cross-session chat memory, customer-support context.  
**Needs extension:** temporal validity, contradiction handling, memory poisoning defense, explicit provenance.

### 4.6 Zep / Graphiti

**Primitive:** temporal knowledge graph memory.  
**Core idea:** facts change; memory needs validity intervals and entity/relation structure.  
**Strength:** handles stale facts better than embedding-only memory; useful for "what is true now?" versus "what happened before?"  
**Weakness:** graph extraction, entity resolution, and schema drift can create brittle edges.

**Best use:** long-running assistants, CRM/support memory, dynamic user/project facts, multi-hop personal or organizational memory.

### 4.7 Cognee

**Primitive:** cognitive GraphRAG with `remember`, `recall`, `forget`, `improve`.  
**Core idea:** memory should have lifecycle operations, not only store/retrieve.  
**Strength:** practical local-first graph memory with explicit improvement/forgetting.  
**Weakness:** graph overhead can be unnecessary for simple chat memory.

### 4.8 A-Mem

**Primitive:** Zettelkasten-style agentic memory.  
**Core idea:** each memory is an atomic note enriched with LLM-generated keywords, tags, contextual description, embedding, and links. When new memories arrive, the system generates links and can evolve old memories.  
**Strength:** memory becomes a self-organizing knowledge network rather than a fixed schema.  
**Weakness:** if the LLM creates wrong links or updates, errors become structural.

**Best use:** research agents, coding agents, knowledge workers, long-running project assistants.

### 4.9 LightMem

**Primitive:** cognitive three-stage efficient memory.  
**Core idea:** sensory memory filters, short-term memory groups/summarizes by topic, long-term memory consolidates offline during sleep-time updates.  
**Reported win:** up to 10.9% accuracy gains, 117x token reduction, 159x API-call reduction, and 12x runtime reduction on reported evaluations.  
**Strength:** treats token/API/runtime cost as first-class.  
**Weakness:** less expressive than graph/experience memory for multi-hop changing knowledge.

### 4.10 MemoryAgentBench

**Primitive:** evaluation framework for memory agents.  
**Core competencies:**

1. Accurate retrieval.
2. Test-time learning.
3. Long-range understanding.
4. Selective forgetting / conflict resolution.

**Why it matters:** it prevents overclaiming from systems that only improve top-k recall.

### 4.11 ReasoningBank

**Primitive:** distilled reasoning memory.  
**Core idea:** learn from both success and failure; extract generalizable reasoning strategies; retrieve them for future tasks; use memory-aware test-time scaling (MaTTS) to produce more diverse experiences and better memories.  
**Strength:** memory becomes a self-evolution and scaling axis.  
**Weakness:** depends on self-evaluation quality and correct failure attribution.

### 4.12 CoPS

**Primitive:** provable cross-task experience selection.  
**Core idea:** not every similar memory helps; retrieve distribution-matched experiences conservatively to avoid negative transfer.  
**Strength:** brings theory to experience retrieval.  
**Weakness:** does not solve full memory lifecycle by itself.

### 4.13 MEM1

**Primitive:** learned constant memory state.  
**Core idea:** train the model to maintain a compact internal state across turns, integrating observations while discarding irrelevant information.  
**Reported win:** 3.5x performance and 3.7x memory reduction in the reported long-horizon setting.  
**Strength:** challenges the assumption that external memory must grow forever.  
**Weakness:** compact state is hard to inspect, edit, or delete for governance.

### 4.14 ReMemR1

**Primitive:** revisitable memory with RL.  
**Core idea:** forward-only memory loses early evidence; allow callback retrieval from memory history and train when to revisit using multi-level rewards.  
**Strength:** directly targets nonlinear long-context reasoning.  
**Weakness:** long-doc QA does not fully cover tool-using agent side effects.

### 4.15 MemArt / KVCache-Centric Memory

**Primitive:** KV-cache memory blocks.  
**Core idea:** store memories in LLM-native KV-cache form and retrieve blocks with latent attention scores, preserving prefix-cache efficiency.  
**Reported win:** OpenReview abstract reports +11% accuracy and 91–135x prefill-token reduction.  
**Strength:** extremely attractive for high-throughput conversational agents.  
**Weakness:** hard to inspect, edit, redact, or port across models.

### 4.16 SMITH

**Primitive:** hierarchical memory + dynamic tool creation.  
**Core idea:** procedural, semantic, and episodic memory support tool generation and cross-task experience sharing.  
**Reported win:** 81.8% Pass@1 on GAIA in the abstract.  
**Strength:** connects memory to capability expansion.  
**Weakness:** complex architecture makes attribution difficult.

### 4.17 ChemAgent

**Primitive:** domain procedural/knowledge memory.  
**Core idea:** decompose chemistry tasks into subtasks; store plan, execution, and knowledge memories; retrieve/refine for new problems.  
**Reported win:** up to 46% GPT-4 gain on SciBench chemistry tasks.  
**Strength:** shows memory should be domain-shaped.  
**Weakness:** transfer outside chemistry needs validation.

### 4.18 EgoMem and Embodied MEMENTO

**Primitive:** personalized embodied/multimodal memory.  
**Core idea:** memory must include user identity, object semantics, routines, audiovisual signals, and multi-memory coordination.  
**Strength:** pushes memory beyond text.  
**Weakness:** privacy, consent, and deletion become much harder.

### 4.19 Intrinsic Memory Agents

**Primitive:** role-aligned memory for multi-agent systems.  
**Core idea:** each agent maintains task-relevant memory aligned to its role, preserving consistency and procedural integrity.  
**Strength:** useful for heterogeneous agent teams.  
**Weakness:** role memory can become siloed or overfit unless shared facts are verified globally.

### 4.20 AgentMemory

**Primitive:** persistent memory server for AI coding agents.  
**Core idea:** auto-capture coding-agent sessions through hooks/MCP/REST, compress them into searchable memory, and inject relevant repo/session context into future runs across Claude Code, Cursor, Gemini CLI, Codex CLI, OpenCode, Cline/Roo, Goose, Aider, and other MCP/HTTP clients.  
**Mechanism:** repository-reported hybrid BM25 + vector + graph retrieval with RRF fusion, 4-tier consolidation, decay/auto-forget, SQLite + iii-engine local storage, 51 MCP tools, 12 auto hooks, real-time viewer, session replay, and no external DB requirement.  
**Reported numbers:** GitHub page reports about 5.3k stars, Apache-2.0 licensing, 827 passing tests, 95.2% LongMemEval-S R@5, 98.6% R@10, 88.2% MRR, and roughly 92% token savings; treat these as repo-reported until independently reproduced.  
**Best use:** long-running coding agents where memory is mostly project facts, architectural decisions, tool-call traces, bugs, fixes, tests, and user preferences.  
**Weakness:** less obviously suited than Zep/Graphiti for temporal business facts, less general than Mem0 for app-level user memory, and less runtime-native than Letta/MemGPT if you want the agent itself to manage memory as part of its reasoning trace.

### 4.21 Memento

**Primitive:** memory-based online reinforcement learning for deep-research agents.  
**Core idea:** freeze the base LLM and adapt the agent through an episodic Case Bank plus a learned case-selection policy, formalized as a Memory-Augmented MDP.  
**Reported numbers:** arXiv:2508.16153 reports 87.88% GAIA validation Pass@3, 79.40% GAIA test, and +4.7 to +9.6 absolute points from case memory on out-of-distribution tasks.  
**Best use:** deep-research or tool-use agents where prior trajectories are reusable as cases.  
**Weakness:** case-bank growth creates a swamping/negative-transfer problem unless retrieval and memory rewriting are strongly governed.

### 4.22 Focus / Active Context Compression

**Primitive:** intra-trajectory active compression.  
**Core idea:** the agent explicitly starts focus regions, explores, writes a compact learning into a persistent Knowledge block, and prunes raw history, turning context growth into a sawtooth pattern rather than append-only bloat.  
**Reported numbers:** arXiv:2601.07190 reports 22.7% token reduction on 5 context-heavy SWE-bench Lite instances with equal 3/5 success, but one task saw +110% token overhead.  
**Best use:** exploration-heavy coding/research tasks with clean phase boundaries.  
**Weakness:** aggressive pruning can force re-exploration on iterative-refinement tasks.

### 4.23 DCI / Direct Corpus Interaction

**Primitive:** retrieval-as-tool-use rather than retrieval-as-top-k.  
**Core idea:** let the agent search raw corpora directly with terminal tools (`rg`, file reads, scripts) instead of forcing all evidence through a fixed vector/sparse retriever interface.  
**Reported numbers:** arXiv:2605.05242 and DCI-Agent-Lite report strong BrowseComp-Plus, BRIGHT, BEIR, and multi-hop QA results without embeddings or offline indexing.  
**Best use:** local research corpora, codebases, and evolving document stores where exact constraints and multi-step evidence refinement matter.  
**Weakness:** slower and more dependent on tool hygiene than pre-indexed retrieval.

### 4.24 Production Context-Engineering Memory

**Primitive:** context as a finite attention budget.  
**Core idea:** Anthropic's context-engineering guidance treats memory, compaction, message history, tool results, system prompts, examples, and subagent summaries as parts of one curated context state; the optimal policy is "smallest high-signal working set," not "maximum tokens."  
**Best use:** production agents that need long-horizon coherence with predictable cost.  
**Weakness:** guidance is architecture-level; each implementation still needs task-specific evals for compaction, note-taking, and just-in-time retrieval.

---

## 5. Comparison Matrix: What Each System Actually Solves

| System | Conversational long-term memory | Episodic memory | Semantic memory | Procedural memory | Temporal validity | Learned control | Production-ready tendency |
|---|---:|---:|---:|---:|---:|---:|---:|
| Generative Agents | medium | high | medium via reflection | low | low | low | low |
| Reflexion | low | medium | medium | low | low | low | medium pattern |
| Voyager | low | medium | low | high | low | low | medium research |
| MemGPT / Letta | high | high | medium | medium | medium | medium-agentic | high |
| Mem0 | high | medium | medium | low | low-medium | medium-agentic | high |
| Zep / Graphiti | high | high | high | low-medium | high | medium | high |
| Cognee | medium-high | high | high | medium | medium | medium | medium-high |
| A-Mem | high | high | high | medium | low-medium | high-agentic | medium |
| LightMem | high | medium | high summaries | low | medium | medium | medium-high |
| ReasoningBank | medium | high | high strategies | medium | low-medium | high-agentic | medium |
| CoPS | low-medium | high | medium | low | low | high-selection | research |
| MEM1 | medium | implicit | implicit | low | implicit | high-RL | research |
| ReMemR1 | medium | high | medium | low | low | high-RL | research |
| MemArt | high | latent | latent | low | low | medium | research/systems |
| SMITH | medium | high | high | high | low-medium | high-agentic | research |
| ChemAgent | low | medium | high-domain | high-domain | low | medium | research |
| EgoMem | high-multimodal | high | high-user | low | medium | medium | research |
| Intrinsic Memory Agents | medium | medium | medium | medium-high role | low-medium | medium | research |
| AgentMemory | medium-high coding | high session/tool traces | medium-high repo facts | medium-high hooks/skills | low-medium | medium-agentic | high for coding agents |
| Memento | medium research | high case bank | medium-high case abstractions | high tool-use cases | low-medium | high learned selection | research frontier |
| Focus / Active Context Compression | medium | high recent trajectory | medium knowledge block | medium focus workflow | low | medium-agentic | experimental harness |
| DCI / Direct Corpus Interaction | low conversational | high evidence traces | high raw-corpus facts | medium search procedures | medium via corpus metadata | medium-agentic | high for local research |
| Anthropic context engineering | high pattern | medium notes/history | medium-high curated context | high compaction/subagent patterns | medium | medium-human-guided | production guidance |

---

## 6. Architecture Patterns

### Pattern A — Personal Assistant Memory

Best stack:

- Mem0-style user/session/agent memory.
- Zep/Graphiti temporal validity for changing facts.
- Lightweight dedupe/update/delete pipeline.
- Privacy controls and user-visible memory editor.

Use when:

- The main need is remembering user preferences, facts, relationships, projects, and prior conversations.

Avoid:

- Heavy graph/procedural systems if you only need simple personalization.

### Pattern B — Research Agent Memory

Best stack:

- A-Mem-style Zettelkasten notes.
- Cognee/HippoRAG-style graph retrieval.
- Source provenance on every note.
- Sleep-time consolidation into semantic claims.

Use when:

- The agent must build a long-lived map of papers, ideas, claims, and contradictions.
- Research-agent harnesses such as [The Agentic Researcher](https://arxiv.org/abs/2603.15914) and trained deep-search agents such as [Tongyi DeepResearch](https://arxiv.org/abs/2510.24701) are useful downstream stress tests: they need persistent project state, experiment logs, citations, negative results, TODO/report continuity, and evidence provenance, even though they are not themselves memory frameworks.

Key risk:

- False links and unsupported abstractions.

### Pattern C — Coding Agent Memory

Best stack:

- AgentMemory-style hook/MCP auto-capture for cross-session coding traces.
- Letta/MemGPT-style explicit memory tools.
- Episodic failure memory.
- Semantic repo facts.
- Procedural memory as tested scripts/skills.
- ReasoningBank-style failure/success strategy distillation.

Use when:

- The agent works on the same codebase across days/weeks.

Key rule:

- No procedural memory becomes durable unless it has a verifier: test, lint, typecheck, or reproduction command.

### Pattern D — Low-Latency Conversational Memory

Best stack:

- LightMem-style online filtering.
- Short-term topic summaries.
- Offline sleep-time consolidation.
- Optional MemArt KV-cache memory for high-throughput repeated contexts.

Use when:

- Cost and latency are as important as accuracy.

Key risk:

- Over-compression loses rare but important facts.

### Pattern E — Multi-Agent Team Memory

Best stack:

- Role-specific memory for each agent.
- Shared verified factual memory.
- Consensus/arbiter for conflicts.
- Trace-level provenance for who wrote what.

Use when:

- Agents have different roles, tools, or perspectives.

Key risk:

- Private role memory hardens errors unless shared facts are reconciled.

### Pattern F — Science / Domain Agent Memory

Best stack:

- ChemAgent-style plan/execution/knowledge memory.
- Domain validators.
- Structured equations/entities/procedures.
- Failure memory for invalid reasoning paths.

Use when:

- The domain has repeatable procedures and objective validation.

Key risk:

- Domain memory can encode wrong formulas or stale assumptions.

---

## 7. Reference Memory Schema

```yaml
MemoryRecord:
  id: string
  scope: user | session | agent | project | org | global
  type: episodic | semantic | procedural | preference | failure | tool | social | latent_anchor
  content: string
  normalized_claim: string?
  source:
    conversation_id: string?
    turn_ids: [string]
    tool_call_ids: [string]
    document_refs: [string]
  temporal:
    created_at: datetime
    observed_at: datetime?
    valid_from: datetime?
    valid_until: datetime?
    supersedes: [memory_id]
    superseded_by: [memory_id]
  confidence:
    extractor: float
    verifier: float
    user_confirmed: boolean
  retrieval:
    embedding_id: string?
    lexical_terms: [string]
    entities: [string]
    tags: [string]
    links: [memory_id]
  governance:
    pii_class: none | low | sensitive | secret
    retention_policy: string
    delete_with_user: boolean
  lifecycle:
    status: candidate | active | expired | quarantined | deleted
    write_reason: string
    verifier_status: unverified | verified | rejected
```

---

## 8. Reference Control Loop

```python
def memory_augmented_agent_step(user_input, task_state):
    # 1. Decide whether memory is useful.
    need = memory_controller.classify_need(user_input, task_state)
    if need == "none":
        return llm.act(user_input, task_state)

    # 2. Route query across heterogeneous stores.
    query = memory_controller.make_query(user_input, task_state)
    stores = memory_router.select_stores(query, budget_ms=800, token_budget=3000)

    # 3. Retrieve with temporal and provenance constraints.
    candidates = []
    for store in stores:
        candidates += store.retrieve(
            query=query,
            filters={"status": "active", "valid_at": "now"},
            include_provenance=True,
        )

    # 4. Rerank and check for contradictions.
    memories = memory_ranker.rerank(candidates, task_state)
    memories = contradiction_filter.remove_superseded(memories)

    # 5. Act with selected memory.
    action, rationale = llm.act_with_memory(user_input, task_state, memories)
    outcome = tools_or_environment.apply(action)

    # 6. Write only if the event is durable.
    proposed = memory_writer.extract_candidates(user_input, action, outcome)
    for record in proposed:
        if memory_verifier.approve(record):
            memory_store.upsert_or_expire(record)
        else:
            memory_store.quarantine(record)

    # 7. Periodic consolidation.
    if scheduler.should_run("sleep_consolidation"):
        memory_consolidator.promote_episodes_to_semantics()
        memory_consolidator.extract_failure_lessons()
        memory_consolidator.promote_verified_procedures()

    return action, outcome
```

---

## 9. Evaluation Checklist

| Capability | Minimum test | Strong test |
|---|---|---|
| Accurate retrieval | single fact recall | retrieval under distractors and paraphrases |
| Test-time learning | remember new preference | apply new preference across sessions and tasks |
| Long-range understanding | retrieve old fact | synthesize facts spread across long history |
| Conflict resolution | update old fact | ignore superseded fact despite semantic relevance |
| Selective forgetting | delete memory by ID | cascading deletion from summaries/embeddings/graphs |
| Negative transfer | similar prior task | detect and avoid harmful near-neighbor memory |
| Multi-memory composition | 2 facts | 3–5 memories jointly required for plan |
| Procedural reuse | recall checklist | execute verified procedure successfully |
| Failure learning | recall previous error | avoid repeated error in shifted environment |
| Cost | average tokens | p95 latency, API calls, storage, retrieval fanout |
| Privacy | PII detection | user-visible memory editing and deletion audit |
| Security | benign retrieval | memory poisoning and prompt-injection stress test |

---

## 10. Research Gaps Worth a Paper

| Gap | Why unsolved | Concrete paper direction |
|---|---|---|
| **Verifier-gated writes** | Memory systems trust LLM extractors too much | Add evidence-grounded verification before durable writes |
| **Temporal Zettelkasten** | A-Mem links lack strong validity semantics | Add valid_from/valid_until and contradiction-aware links |
| **Negative-transfer retrieval** | Similar memories can hurt | Combine CoPS with conversational/agent memory |
| **Memory poisoning defense** | Retrieved memories can carry malicious instructions | Build quarantine, trust scoring, and memory firewall |
| **Inspectable latent memory** | KV-cache memory is efficient but opaque | Attach textual anchors and deletion handles to KV blocks |
| **Learned forgetting** | Delete/update policies are heuristic | RL policy trained on utility, privacy, and contradiction metrics |
| **Procedural promotion** | Agents store workflows without proof | Promote only memories with executable validators |
| **Multi-memory coordination** | Retrieval often returns individually relevant but jointly inconsistent facts | Benchmark and solve memory composition |
| **Cross-agent consensus memory** | Role-specific memories diverge | Shared verified fact store plus private role memories |
| **Memory cost laws** | Papers under-report latency/token/storage cost | Derive cost-performance curves for memory tiers |

---

## 11. Most Promising Next Paper

**Title candidate:** *VeriMem: Verifier-Gated Temporal Agentic Memory for Long-Term LLM Agents*

**Core claim:** LLM-agent memory fails less often when every durable memory is (1) typed, (2) evidence-backed, (3) temporally scoped, (4) contradiction-checked, and (5) updated through a lifecycle controller rather than appended blindly.

**Architecture:**

- A-Mem note construction and linking.
- Zep/Graphiti temporal validity.
- Mem0-style scope: user/session/agent/project.
- ReasoningBank-style success/failure lesson extraction.
- LightMem-style sleep-time consolidation.
- CoPS-style negative-transfer-aware retrieval.
- Verifier-gated write/update/delete.

**Ablations:**

1. no verifier,
2. no temporal validity,
3. no graph links,
4. no failure memory,
5. no sleep consolidation,
6. no conservative retrieval.

**Metrics:**

- MemoryAgentBench four-competency accuracy.
- LongMemEval/LoCoMo conversational QA.
- repeated-error reduction on web/coding tasks.
- contradiction rate.
- memory precision/recall.
- stale-fact retrieval rate.
- token/API/p95 latency.
- memory poisoning block rate.
- deletion completeness.

---

## 12. Dense Source Map

| Category | Must-read works |
|---|---|
| Foundational agent memory | [Generative Agents](https://arxiv.org/abs/2304.03442), [Reflexion](https://arxiv.org/abs/2303.11366), [Voyager](https://arxiv.org/abs/2305.16291), [MemGPT/Letta](https://github.com/letta-ai/letta) |
| Production conversational memory | [Mem0](https://github.com/mem0ai/mem0), [Zep](https://github.com/getzep/zep), [Graphiti](https://github.com/getzep/graphiti), [Cognee](https://github.com/topoteretes/cognee), [Supermemory](https://github.com/supermemoryai/supermemory) |
| Coding-agent memory | [AgentMemory](https://github.com/rohitg00/agentmemory), [ReasoningBank](https://openreview.net/forum?id=jL7fwchScm), [Agent Workflow Memory](https://openreview.net/forum?id=NTAhi2JEEE), [Letta](https://github.com/letta-ai/letta) |
| Research-agent memory stress tests | [The Agentic Researcher](https://arxiv.org/abs/2603.15914), [Tongyi DeepResearch](https://arxiv.org/abs/2510.24701), [Memento](https://arxiv.org/abs/2508.16153), [Agent Laboratory](http://hf.co/papers/2501.04227), [AgentRxiv](https://arxiv.org/html/2503.18102v1) |
| Agentic/structured memory | [A-Mem](https://openreview.net/forum?id=FiM0M8gcct), [LightMem](https://openreview.net/forum?id=dyJ0GWpjJB), [SMITH](https://openreview.net/forum?id=JnwClln80Q), [ChemAgent](https://openreview.net/forum?id=kuhIqeVg0e) |
| Learned memory control | [MEM1](https://openreview.net/forum?id=jJ6F1sDn9i), [ReMemR1](https://openreview.net/forum?id=1cymflI2Lh), [MemArt](https://openreview.net/forum?id=YolJOZOGhI), [Focus](https://arxiv.org/abs/2601.07190) |
| Experience memory | [ReasoningBank](https://openreview.net/forum?id=jL7fwchScm), [CoPS](https://openreview.net/forum?id=9W6Z9IeLzc), [Memento](https://arxiv.org/abs/2508.16153) |
| Context engineering and retrieval interface | [Anthropic context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents), [DCI](https://arxiv.org/abs/2605.05242), [DCI-Agent-Lite](https://github.com/DCI-Agent/DCI-Agent-Lite), [OpenDev](https://arxiv.org/abs/2603.05344), [Dive into Claude Code](https://arxiv.org/abs/2604.14228) |
| Multimodal / embodied | [EgoMem](https://openreview.net/forum?id=9QYA3DiPl8), [MEMENTO embodied](https://openreview.net/forum?id=E5L43l5EIu) |
| Multi-agent memory | [Intrinsic Memory Agents](https://openreview.net/forum?id=UbSUxAK3BI) |
| Evaluation | [MemoryAgentBench](https://openreview.net/forum?id=DT7JyQC3MR), LongMemEval, LoCoMo, MemoryArena-style interactive tests |
| Surveys | [Memory in the Age of AI Agents](https://arxiv.org/abs/2512.13564), [From Storage to Experience](https://arxiv.org/html/2605.06716), [Memory for Autonomous LLM Agents](https://www.arxiv.org/pdf/2603.07670), [LLM Agent Memory: Unified Representation–Management](https://openreview.net/forum?id=KPs1EgGKcT), [Anatomy of Agentic Memory](https://arxiv.org/abs/2602.19320v1), [Memory in the LLM Era](https://arxiv.org/html/2604.01707v2) |

---

## 13. Final Takeaways

1. **Memory is a lifecycle, not a store.** Write/update/delete/promote matter as much as retrieve.
2. **Episodic memory is not enough.** Long-running agents need semantic and procedural consolidation.
3. **Temporal validity is mandatory.** Old facts are often semantically relevant and behaviorally wrong.
4. **Experience memory is the bridge to self-improvement.** Store lessons, not only transcripts.
5. **Efficiency is a first-class metric.** LightMem, MEM1, and MemArt show that cost can dominate.
6. **Learned memory control is the frontier.** The next wave will train when to read, write, revisit, compress, and forget.
7. **Production memory needs governance.** Provenance, privacy, deletion, poisoning defense, and audit trails are not optional.
8. **The best next paper is hybrid.** A-Mem structure + temporal KG + verifier-gated writes + conservative experience retrieval + sleep-time consolidation.

