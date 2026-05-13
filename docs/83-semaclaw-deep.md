# 83 — SemaClaw (Paper Deep-Dive): General-Purpose Personal AI Agents via Harness Engineering

**Paper.** Ningyan Zhu, Huacan Wang, Jie Zhou, Feiyu Chen, Shuo Zhang, Ge Chen, Chen Liu, Jiarou Wu, Wangyi Chen, Xiaofeng Mou, Yi Xu — *SemaClaw: A Step Towards General-Purpose Personal AI Agents through Harness Engineering* — Midea AIRC — arXiv:2604.11548 — 13 April 2026 (manuscript date 28 March 2026) — <https://arxiv.org/abs/2604.11548> — code: <https://github.com/midea-ai/SemaClaw> — runtime: <https://github.com/midea-ai/sema-code-core>. This note **complements** the corpus summary in [54-semaclaw-general-purpose-agent](54-semaclaw-general-purpose-agent.md) with **PDF-grounded** implementation numbers, system boundaries, and explicit claims versus evidence.

**One-line definition.** SemaClaw is a **two-layer** open-source harness (runtime `sema-code-core` + application `semaclaw`) that combines **event-facade** execution, **three-tier** context with concrete compaction and hybrid retrieval parameters, a globally scoped **PermissionBridge** for HITL, **DAG Teams** with `create_parent` / `DispatchBridge` scheduling, a **four-mode** scheduled-task router, and a **plain-Markdown wiki** knowledge layer—positioned as infrastructure for *general-purpose personal* agents when model capability is held fixed.

## Why this paper matters

It argues **harness structure** is the differentiation axis as models converge, then backs the claim with: ReAct/MCP/Claude Code–aligned runtime choices; a **concrete** LLM plan → deterministic **execute** cut; and auditable **numbers** (compaction %, hybrid fusion, scheduler cadence, log retention). It also states limits: **no** SemaClaw-native leaderboard; **external** +13.7 pp evidence only; compaction **code-biased**.

## Problem it solves (numbered list)

1. **Orchestration with inspectable structure.** It rejects pure handoff swarms and pure static graphs by composing **LLM-emitted DAGs** with a **deterministic** scheduler, addressing “pseudo-orchestration” and opaque coordinator traces.
2. **Runtime-typed safety for consequential tools.** Permissions are not only configuration or prompts: **PermissionBridge** serializes, multiplexes, and routes approvals at the tool boundary with tiered policies.
3. **Cross-session memory without conflating identity and logs.** A three-source cognitive model (working / external / structured injection) is **instantiated** with soul vs workspace vs MEMORY.md separation and a wiki layer for **knowledge sedimentation** distinct from chat logs.
4. **A reusable personal-agent foundation.** `sema-code-core` is factored for reuse; `semaclaw` adds channels, memory extension, and team coordination—mirroring the paper’s “harness vs runtime” split called out in Section 3.1.

## Core idea in one paragraph

**SemaClaw** treats the personal agent as a **stacked harness**: a shared event-driven runtime enforces the ReAct loop, compaction, and isolation; the application layer adds team orchestration, HITL, memory indexing, and a user-owned wiki. Multi-agent work uses **one-shot structured declaration** (`create_parent` with `tasks[]` and `dependsOn[]`) after which **no further LLM dispatch decisions** interleave with execution—only `DispatchBridge` drives the DAG, turning “dynamic” planning into a **single auditable moment** and execution into **scheduled, state-machine-like** progress. Behavioral and economic constraints are then layered: PermissionBridge, four scheduling modes, and file-system-local corpora the human can edit without a DB round-trip.

## Mechanism (step by step)

**Architecture (Figures 2, 3, 5, 6 in the PDF).**  
- **Layer 0 — `sema-code-core`:** Event-facade pattern; lifecycle transitions (`session init`, `tool`, `compact:*`, `context`, `termination`) are typed events. Consumers (including `semaclaw`) **cannot** reach into raw internal state.  
- **Layer 1 — `semaclaw`:** Channels, `AgentPool`, `MemoryManager`, plugin wiring, `DispatchBridge`, wiki UI—application concerns only.

**Pseudocode: end-to-end harness path (condensed from Sections 3.2–3.5).**

```
on_user_message:
  sema_code_core.append_to_working_memory()
  inject_structured: [SOUL from identityDir, MEMORY.md, workspace context]
  loop ReAct:
    on_tool_invoke:
      if tool is external: PermissionBridge.suspend(id, payload); wait user; resume or deny
    if context_tokens > 0.75 * model_limit - 8000_buffer:
      emit compact:start → summarize() → compact:exec
      append user-role message: generateRulesReminders() [+ generateTodosReminders() if TodoWrite]
      MemoryManager.mark_daily_log_dirty()   # re-index before next hybridSearch
```

**Compaction (novel in detail vs generic “summarize history”).** Trigger at **75%** of configured context length, with an **8,000-token** headroom for the next user turn and injected reminders. Events carry before/after token counts, ratio, and summary text. On summarizer **failure**, truncate to **50%** of limit and prepend a notice—explicit degradation path. The paper states compaction prompts are **code-workflow-optimized**; conversational preference-heavy sessions are a **known weak spot** (Section 3.2.1).

**External memory and hybrid retrieval (specific numbers).** Corpus: `MEMORY.md` plus `memory/YYYY-MM-DD.md` logs, **50-day** FIFO; `memory_search` supports `source ∈ {memory, session, all}`. **hybridSearch** when embeddings exist: **BM25 (FTS5) + vector (sqlite-vec)**. Merge rule: vector-only and FTS-only docs get **0.7** base; overlap uses **`0.7 * vec + 0.3 * fts`** to avoid under-ranking cross-lingual FTS-only hits. Fallback chain: **Vector+FTS → FTS-only → token keyword scan**. Indexing: documents **without** stopword removal; queries **with** stopword filtering (recall vs precision trade). Retrieval is **agent-initiated** via MCP; daily logs store **user prompt only**, not injected retrieval (trace fidelity).

**Persona / workspace (semantic action representation in practice).** Not a separate “semantic action DSL” in the ReAct sense—the paper’s *structured* contract is the **MCP `create_parent` payload**: each task has `agentName`, `prompt`, `dependsOn` labels. **Binding** is *not* neural: `resolveAgent()` is **case-insensitive exact string match** on **registered names and folder ids**; orchestrator *reasons* about who to assign, runtime *confirms* identity. `list_agents` exposes roster metadata to the LLM. **Workspace switching** can occur without session reset; **soul** is fixed per agent instance; multi-agent instances have **isolated** namespaces (no cross-agent retrieval or injection leakage).

**DAG execution — DispatchBridge (deterministic).** `processPending()` every **300 ms**: timeout checks, then dispatch tasks whose dependencies are terminal (**done, error, or timeout**—failed upstream does **not** dead-lock downstream). `startTask()` builds prompts including `<parent_goal>`, `<prerequisites>`, `<other_tasks>`. IPC: dispatch MCP (stdio child) ↔ main process via **JSON state file + lock**; on worker idle, `AgentPool` → `notifyReply` / `notifyError` → `processNextPending()`. Polling: orchestrator’s `dispatch_task` polls every **500 ms**; dynamic deadline switches once execution starts. Heartbeat **every 2 min** while blocked on `dispatch_task` poll to avoid a **30-minute** inactivity timeout. **Only one** active parent group per admin agent; further `create_parent` calls **queue**. `detectCycle()` rejects cyclic graphs at **declaration** time. Startup recovery: parents left active/queued → marked **done** with tasks **interrupted**.

**PermissionBridge.** Single global instance; request IDs demux concurrent suspensions. Two pause modes: **tool permission** and **user question** (clarification returned like a tool result). **Two tiers:** internal tools pre-authorized; external tools (user MCP, FS, APIs) require **per-invocation** default consent. UI: Telegram inline buttons; Web UI; **“last active”** timeout so approvals don’t kill sessions.

**Four scheduled modes (Section 3.6).** (1) Pure notification—**zero tokens**; (2) Pure script—**zero tokens**; (3) Pure agent—full ReAct; (4) **Hybrid** script→agent: deterministic gather then interpret.

**Wiki (§3.7).** Markdown+YAML; search API ≠ `memory_search`. **Organize:** user body fixed; agent edits **frontmatter** only. Fig. 1: CLI+Web, same files.

**Novel vs adapted.** *Prior art:* ReAct, MCP, MemGPT/GA concepts, LangGraph DAGs, OpenClaw-inspired patterns. *Claimed:* `sema-code-core`∥`semaclaw` split; **DispatchBridge** + pre-exec `create_parent` DAG; multiplexed **PermissionBridge**; numeric hybrid fusion; **four** schedule modes; wiki as parallel corpus.

## Empirical results (concrete benchmark numbers from the paper)

- **External controlled harness study (LangChain blog, 2026, cited in the PDF):** on **Terminal Bench 2.0**, changing **only the harness** (model held constant) moved task completion from **52.8% to 66.5%** — a **+13.7 percentage point** gain. This is the paper’s **only** quantitative benchmark table-style result.  
- **SemaClaw-specific:** the authors **do not** report a proprietary SWE-bench or GAIA-style score for the SemaClaw stack. Section 4.2.4 states mid-tier+SemaClaw **≈** frontier w/o harness is a **working hypothesis** “**not yet** produced rigorous empirical validation” in their deployment; they **invite** community benchmarks. Treat leaderboard claims as **non-present** in this paper.

## Variants and ablations

The PDF does not publish **SemaClaw A/B** ablations (e.g., DAG vs ad-hoc dispatch). It instead contrasts **architectural variants** in Section 2.4: stateless swarms (OpenAI Swarm), static LangGraph-style DAGs, and orchestrator-dynamics—then **composes** dynamic planning + explicit DAG. Section 4.1 discusses **virtual** vs **persistent** personas; SemaClaw **commits** to persistent identities and flags **ephemeral+core hybrid** as future. Section 2.2.1–2.2.3 frames alternative retrieval schemes (BM25 vs dense vs hybrid) and positions their **degradation ladder** as an engineering response—not an ablation table.

## Failure modes and limitations

- **Compaction mis-summarization** in non-code conversational flows (authors’ caveat). **Silent** corruption risk whenever summaries replace verbatim history.  
- **Routing mismatch:** orchestrator can pick a **name** that semantically misfits a **drifted** `SOUL.md`—binding still succeeds (Section 4.1.3).  
- **Roster gap:** no on-the-fly specialist spawn—forced fit or “no agent” (Section 4.1.3).  
- **Wiki vs memory** split: **no unified** cross-corpus retrieval yet (Limitations 5.3).  
- **Channel coverage** narrow vs messaging/email/voice; **hook parity** with Claude Code not complete; permission/context **drift** vs upstream.  
- **Session-export JSON** “anticipated” but **not** indexed in current pipeline (Section 3.2.2).  
- **Marketing vs validated:** the opening “millions of users” *OpenClaw* adoption line is **societal framing**, not a measurement of SemaClaw’s user base. **Figure 6** is an architectural **schematic**, not a performance proof.

## When to use, when not

**Use** when you need: inspectable **DAG-shaped** team runs with **deterministic** dispatch; **file-local** memory + wiki for **user ownership**; HITL at **scale** (multiplexed bridge); strict **internal vs external** tool tiering; scheduled jobs where **most work** should bypass the LM (modes 1–2 or hybrid 4). **Avoid** when you need: proven **SemaClaw** scores on your benchmark (not in paper); **fully managed** SaaS; **seamless** omnichannel; **tight** upstream lockstep with **Claude Code** hooks day-one; or **one-shot** Q&A where DAG + bridge overhead adds little.

## Implications for harness engineering

- [54](54-semaclaw-general-purpose-agent.md) = concepts; this file = **75% / 8k / 50% / 300 & 500 ms / 0.7–0.3 / 50 d**-style **constants**.  
- [29-dive-into-claude-code](29-dive-into-claude-code.md): CC is the **explicit** alignment target (§3.1.2).  
- [40-harness-engineering-principles](40-harness-engineering-principles.md), [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md): four plugin layers + hooks-as-clamps.  
- [52-dive-into-open-claw](52-dive-into-open-claw.md): predecessor narrative; **no** head-to-head score here. **+13.7 pp** = **LangChain** TB2 harness study, *not* SemaClaw A/B.

## Connections to other work in this corpus

- [17-rewoo](17-rewoo.md), [02-subagent-delegation](02-subagent-delegation.md): two-phase plan/execute; here **MCP `create_parent` + DispatchBridge +** persistent roster.  
- [06-permission-modes](06-permission-modes.md), [05-hooks](05-hooks.md): global PermissionBridge; hooks may **lag** Claude Code (§5.3).  
- [25-agentic-rag](25-agentic-rag.md): hybrid retrieval **plus** soul/MEMORY/wiki injection—not RAG only.  
- [54](54-semaclaw-general-purpose-agent.md) points to *Externalization* (arXiv:2604.08224) and **M⋆** (arXiv:2604.11811); this paper’s wiki = **file-native** sedimentation.

## Key takeaways

1. Core **empirical** anchor: **external** LangChain **52.8% → 66.5%** (**+13.7 pp** TB2, harness-only)—not a SemaClaw table.  
2. Differentiators: **DispatchBridge** + **hybridSearch** fusion, not new models.  
3. **String-match** worker binding vs **LLM** pick—drift/ misname risk.  
4. Sedimentation = **wiki** corpus + separate search, not log growth alone.  
5. Mid-tier **parity** = **hypothesis** (§4.2.4).

## References

- Zhu, N., Wang, H., Zhou, J., Chen, F., Zhang, S., Chen, G., Liu, C., Wu, J., Chen, W., Mou, X., Xu, Y. *SemaClaw: A Step Towards General-Purpose Personal AI Agents through Harness Engineering.* arXiv:2604.11548. April 2026. <https://arxiv.org/abs/2604.11548> — HTML: <https://arxiv.org/html/2604.11548>  
- SemaClaw: <https://github.com/midea-ai/SemaClaw> — sema-code-core: <https://github.com/midea-ai/sema-code-core>  
- LangChain (2026) — *Improving deep agents with harness engineering* (Terminal Bench 2.0 / **52.8% → 66.5%** harness-only result, cited in SemaClaw Section 1 & 4.2.2) — <https://blog.langchain.com/improving-deep-agents-with-harness-engineering/>  
- Steinberger, P. et al. OpenClaw (2026) — <https://github.com/openclaw/openclaw>  
- Yao, S. et al. ReAct, 2023 — <https://arxiv.org/abs/2210.03629>  
- Anthropic — MCP, Claude Code, Agent Skills (documentation cited in PDF references)  
- Corpus cross-links: [54-semaclaw-general-purpose-agent](54-semaclaw-general-purpose-agent.md), [29-dive-into-claude-code](29-dive-into-claude-code.md), [40-harness-engineering-principles](40-harness-engineering-principles.md), [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md), [52-dive-into-open-claw](52-dive-into-open-claw.md)
