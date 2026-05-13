# 315 — Memory Canon + OSS Landscape (2026)

**Scope.** Practical map of the papers, surveys, and GitHub repositories that matter for building and researching agent memory in 2026. This is the implementation companion to [313](313-memory-research-2026-master-synthesis.md) and [314](314-memory-openreview-paper-atlas-2026.md).

---

## 1. Canonical papers to anchor the literature review

| Work | Year | Memory primitive | Why it still matters |
|---|---:|---|---|
| [Generative Agents](https://arxiv.org/abs/2304.03442) | 2023 | Memory stream + reflection + planning | The canonical observation/reflection/planning memory loop |
| [Reflexion](https://arxiv.org/abs/2303.11366) | 2023 | Verbal self-reflection | Converts failed attempts into future guidance without fine-tuning |
| [Voyager](https://arxiv.org/abs/2305.16291) | 2023 | Executable skill library | Shows procedural memory can be code, not text |
| [MemGPT / Letta](https://github.com/letta-ai/letta) | 2023–2026 | OS-like tiered virtual context | Defines core/recall/archival-style self-managed memory |
| [HippoRAG / HippoRAG 2](https://arxiv.org/abs/2502.14802) | 2024–2025 | Graph + dense retrieval | Strong multi-hop memory retrieval via graph traversal/fusion |
| [Mem0](https://github.com/mem0ai/mem0) | 2025–2026 | Production memory layer | Popular long-term memory API for user/session/agent memory |
| [Zep / Graphiti](https://github.com/getzep/graphiti) | 2025–2026 | Temporal knowledge graph | Strong answer to temporal validity and changing facts |
| [Cognee](https://github.com/topoteretes/cognee) | 2025–2026 | Cognitive GraphRAG memory | Practical remember/recall/forget/improve control plane |
| [AgentMemory](https://github.com/rohitg00/agentmemory) | 2026 | Coding-agent persistent memory | MCP/REST memory server for coding agents, with hooks, hybrid search, replay, and local-first SQLite/iii-engine storage |
| [ReasoningBank](https://openreview.net/forum?id=jL7fwchScm) | 2026 | Reasoning strategy memory | Treats memory as self-evolution from successes and failures |
| [MemArt](https://openreview.net/forum?id=YolJOZOGhI) | 2026 | KV-cache memory | Moves memory below text into LLM-native latent blocks |

---

## 2. Survey papers and how to use them

| Survey | Link | Best conceptual frame |
|---|---|---|
| Memory in the Age of AI Agents: Forms, Functions and Dynamics | [arXiv](https://arxiv.org/abs/2512.13564) | Broadest 2026 survey; unifies memory by **forms** (token-level / parametric / latent), **functions** (factual / experiential / working), and **dynamics** (formation / evolution / retrieval) |
| From Storage to Experience | [arXiv](https://arxiv.org/html/2605.06716) | Evolution from raw storage → reflection → abstract experience |
| Memory for Autonomous LLM Agents | [arXiv PDF](https://www.arxiv.org/pdf/2603.07670) | Write–manage–read loop; temporal scope × substrate × control policy |
| LLM Agent Memory: Unified Representation–Management | [OpenReview](https://openreview.net/forum?id=KPs1EgGKcT) | Natural-language tokens vs intermediate representations vs parameters |
| Anatomy of Agentic Memory | [arXiv](https://arxiv.org/abs/2602.19320v1) | Taxonomy plus evaluation/system limitations |
| Memory in the LLM Era | [arXiv](https://arxiv.org/html/2604.01707v2) | Modular architecture and benchmark comparison |

### Recommended literature-review structure

1. **Pre-agent neural memory:** Memory Networks, Neural Turing Machines, Differentiable Neural Computers, RAG, RETRO.
2. **Agent memory v1:** ReAct trajectories, Reflexion, Generative Agents.
3. **Form/function/dynamics survey layer:** Hu et al. 2026 — token-level / parametric / latent forms; factual / experiential / working functions; formation / evolution / retrieval dynamics.
4. **Virtual context / memory OS:** MemGPT, Letta, context compression, tiered stores.
5. **Structured memory:** Knowledge graphs, temporal KGs, Zettelkasten, graph RAG.
6. **Experience memory:** CoPS, ReasoningBank, SMITH, ChemAgent, Voyager.
7. **Learned memory control:** MEM1, ReMemR1, Memento case-selection, Focus active compression, INFMEM-style system-2 memory control.
8. **Latent memory:** KV-cache storage, recurrent memory, parameter/internalized memory.
9. **Retrieval interface as memory:** DCI / Direct Corpus Interaction, just-in-time corpus navigation, file-system memory, tool-mediated evidence search.
10. **Evaluation:** MemoryAgentBench, LongMemEval, LoCoMo, MemoryArena-like interactive tests.

---

## 3. OSS repository landscape

| Repo | Role | Best for | Avoid when |
|---|---|---|---|
| [mem0ai/mem0](https://github.com/mem0ai/mem0) | Plug-in long-term memory layer | Chat/product personalization, user/session memory | You need explicit temporal validity or deep graph reasoning |
| [letta-ai/letta](https://github.com/letta-ai/letta) | Stateful agent runtime | Agent-managed tiered memory and auditability | You only need a lightweight memory API |
| [supermemoryai/supermemory](https://github.com/supermemoryai/supermemory) | Hosted/OSS memory service | Cross-app personal memory and MCP-style integrations | Strict local-first/private deployments |
| [rohitg00/agentmemory](https://github.com/rohitg00/agentmemory) | Coding-agent memory engine + MCP server | Cross-session memory for Claude Code, Cursor, Gemini CLI, Codex CLI, OpenCode, Cline/Roo, Goose, Aider via REST, and other MCP/HTTP agents | You need a general app memory API rather than coding-agent trace/session memory |
| [getzep/zep](https://github.com/getzep/zep) | Memory platform | Temporal conversational memory | You need fully embedded local-only memory |
| [getzep/graphiti](https://github.com/getzep/graphiti) | Temporal KG engine | Changing facts and validity intervals | Simple short-lived personalization |
| [topoteretes/cognee](https://github.com/topoteretes/cognee) | Cognitive GraphRAG | Local/private graph memory with improve/forget | Pure chat memory where graph overhead is too high |
| [zjunlp/LightMem](https://github.com/zjunlp/LightMem) | Efficient memory framework | Token/API/runtime constrained systems | You need maximal expressive graph/experience memory |
| [WujiangXu/AgenticMemory](https://github.com/WujiangXu/AgenticMemory) | A-Mem research code | Reproducing Zettelkasten memory experiments | Production deployment without hardening |
| [WujiangXu/A-mem-sys](https://github.com/WujiangXu/A-mem-sys) | A-Mem system code | Production-oriented agentic note memory | Strict deterministic/provenance-heavy workflows |
| [google-research/reasoning-bank](https://github.com/google-research/reasoning-bank) | Reasoning memory | SWE/web agents learning from success/failure | Basic user preference memory |
| [MIT-MI/MEM1](https://github.com/MIT-MI/MEM1) | Learned constant memory | Research on RL memory control | Immediate production deployment |
| [syr-cn/ReMemR1](https://github.com/syr-cn/ReMemR1) | Revisitable memory RL | Long-doc/nonlinear evidence revisiting | Simple RAG applications |
| [gersteinlab/ChemAgent](https://github.com/gersteinlab/chemagent) | Domain procedural memory | Chemistry/science decomposition memory | General chat personalization |
| [Connoriginal/MEMENTO](https://github.com/Connoriginal/MEMENTO) | Embodied personalization benchmark | Robot/embodied memory research | Lightweight software-only benchmarks |
| [HUST-AI-HYZ/MemoryAgentBench](https://github.com/HUST-AI-HYZ/MemoryAgentBench) | Benchmark | Measuring memory-agent competencies | Serving memory in production |
| [DCI-Agent/DCI-Agent-Lite](https://github.com/DCI-Agent/DCI-Agent-Lite) | Direct-corpus interaction agent | Local corpora where terminal search/read/script tools give higher-resolution evidence than fixed retrievers | Simple chat memory or low-latency hosted retrieval |

---

## 4. Build-vs-buy decision table

| Goal | Best starting point | Add these ideas |
|---|---|---|
| Personal assistant memory | Mem0 or Zep | temporal validity, privacy deletion, contradiction handling |
| Long-running coding agent | AgentMemory or Letta + ReasoningBank | verifier-gated procedural memory, test-backed skill promotion, replayable sessions, hook-based auto-capture |
| Research assistant | A-Mem + Cognee + DCI | Zettelkasten links, graph consolidation, source provenance, raw-corpus tool search |
| Low-latency chatbot | LightMem | sleep-time consolidation, hot/warm routing |
| Embodied multimodal agent | EgoMem concepts + MEMENTO benchmark | identity privacy, multi-memory coordination |
| Multi-agent team | Intrinsic Memory Agents + shared KG | role-specific private memory + shared factual consensus |
| Scientific reasoning agent | ChemAgent | domain validators, plan/execution/knowledge memories |
| Long-horizon QA | MEM1 or ReMemR1 | callback memory, learned compression, evidence revisit metrics |
| Deep-research continual learning | Memento | episodic case bank, learned case-selection, case-bank anti-swamping |
| Context-bloat control | Focus-style active compression | explicit focus regions, persistent Knowledge block, compression evals by task type |
| Production temporal memory | Zep/Graphiti | temporal QA, fact supersession, audit trails |
| Frontier systems research | MemArt + MEM1 + A-Mem | latent memory with interpretable text anchors |

---

## 5. Minimal architecture for a paper prototype

### New OSS note: AgentMemory

[AgentMemory](https://github.com/rohitg00/agentmemory) is a high-star coding-agent memory implementation rather than a general chat-memory API. The repository page reports about **5.3k stars**, Apache-2.0 licensing, `packages/mcp`, agent integrations, a real-time viewer, replay support, 51 MCP tools, 12 auto hooks, 827 passing tests, and benchmark claims of **95.2% LongMemEval-S R@5**, **98.6% R@10**, and roughly **92% token savings**. Treat these as repository-reported numbers until independently reproduced, but the design is important because it targets the exact "coding agent forgets the repo between sessions" problem that Mem0/Zep/Letta do not specialize around.

Where it fits:

- **Compared with Mem0:** less of a generic application-memory API, more of an auto-captured coding-session memory engine.
- **Compared with Letta/MemGPT:** less of a full stateful-agent runtime, more of an external memory server that can be shared by many existing coding agents through MCP/REST.
- **Compared with Cognee/Zep:** less focused on temporal business facts or graph-heavy document reasoning, more focused on project/session/tool-call traces and coding context injection.
- **Best paper-use angle:** benchmark it on coding-agent memory tasks alongside ReasoningBank, Agent Workflow Memory, Mem0, Letta, and Cognee, with verifier-gated promotion of codebase facts and procedural lessons.

For a strong publishable prototype, implement only these pieces first:

1. **Memory object schema**
   - `id`
   - `type`: episodic / semantic / procedural / preference / failure / tool
   - `content`
   - `source_span`
   - `created_at`
   - `valid_from`
   - `valid_until`
   - `confidence`
   - `links`
   - `verifier_status`

2. **Write pipeline**
   - candidate extraction,
   - deduplication,
   - contradiction check,
   - verifier approval,
   - write/update/expire.

3. **Read pipeline**
   - query routing,
   - hybrid retrieval,
   - temporal filter,
   - graph expansion,
   - reranking,
   - provenance attachment.

4. **Consolidation pipeline**
   - sleep-time clustering,
   - repeated-pattern abstraction,
   - failure lesson extraction,
   - procedural promotion with tests.

5. **Evaluation**
   - MemoryAgentBench,
   - LongMemEval / LoCoMo,
   - one agentic task benchmark,
   - cost/latency,
   - memory-poisoning tests.

---

## 6. A practical research roadmap

### Week 1 — Reproduce baselines

- Run Mem0, LightMem, and A-Mem on LongMemEval/LoCoMo.
- Run MemoryAgentBench if compute permits.
- Record token, latency, and API-call costs.

### Week 2 — Implement one novel mechanism

Best high-leverage choice: **verifier-gated temporal note evolution**.

- Start from A-Mem-style notes.
- Add temporal validity.
- Add contradiction checking.
- Add verifier status.

### Week 3 — Add experience abstraction

- Extract lessons from success/failure.
- Retrieve lessons with CoPS-style conservative matching.
- Track negative transfer.

### Week 4 — Evaluate and ablate

Ablations:

1. no verifier,
2. no temporal validity,
3. no experience memory,
4. no graph links,
5. no sleep-time consolidation.

Report:

- task accuracy,
- contradiction rate,
- repeated-error rate,
- memory precision,
- memory recall,
- cost per turn,
- p95 latency.

---

## Sources

- [Mem0 GitHub](https://github.com/mem0ai/mem0)
- [Letta GitHub](https://github.com/letta-ai/letta)
- [Supermemory GitHub](https://github.com/supermemoryai/supermemory)
- [AgentMemory GitHub](https://github.com/rohitg00/agentmemory)
- [Memory in the Age of AI Agents](https://arxiv.org/abs/2512.13564)
- [Memento](https://arxiv.org/abs/2508.16153)
- [Active Context Compression / Focus](https://arxiv.org/abs/2601.07190)
- [Direct Corpus Interaction](https://arxiv.org/abs/2605.05242)
- [DCI-Agent-Lite GitHub](https://github.com/DCI-Agent/DCI-Agent-Lite)
- [Zep GitHub](https://github.com/getzep/zep)
- [Graphiti GitHub](https://github.com/getzep/graphiti)
- [Cognee GitHub](https://github.com/topoteretes/cognee)
- [LightMem GitHub](https://github.com/zjunlp/LightMem)
- [A-Mem GitHub](https://github.com/WujiangXu/AgenticMemory)
- [ReasoningBank GitHub](https://github.com/google-research/reasoning-bank)
- [MEM1 GitHub](https://github.com/MIT-MI/MEM1)
- [ReMemR1 GitHub](https://github.com/syr-cn/ReMemR1)
- [ChemAgent GitHub](https://github.com/gersteinlab/chemagent)
- [MEMENTO GitHub](https://github.com/Connoriginal/MEMENTO)
- [MemoryAgentBench GitHub](https://github.com/HUST-AI-HYZ/MemoryAgentBench)
