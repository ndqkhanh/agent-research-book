# 65 — DeerFlow (ByteDance): From Deep Research Graph to SuperAgent Harness

**Definition.** DeerFlow ("Deep Exploration and Efficient Research Flow") is ByteDance's open-source agent framework, released under MIT. Version 1.x (May 2025) was a LangGraph-based **deep-research workflow** with a fixed Coordinator → Planner → (Researcher | Coder) → Reporter pipeline, a human-in-the-loop plan editor, and novel deliverable modes (podcast, slides, prose). Version 2.0 (February 27, 2026) is a ground-up rewrite into a **long-horizon SuperAgent harness** built around a lead agent, Docker-sandboxed subagents, a Markdown-based Skills system, long-term memory, MCP tool servers, and a multi-IM-channel gateway. The 2.0 rewrite shares no code with 1.x; the 1.x branch (`main-1.x`) is still maintained for teams who want the deterministic research-pipeline abstraction.

This doc treats DeerFlow as a *harness-builder case study*: what primitives they shipped, how they map onto the corpus vocabulary, and what ideas are worth stealing.

## What DeerFlow is

- **Repo:** [github.com/bytedance/deer-flow](https://github.com/bytedance/deer-flow)
- **License:** MIT
- **Stars / forks:** ~62.9k / ~8.1k (April 2026); #1 on GitHub Trending within 24 hours of the 2.0 launch.
- **Last commit observed:** `6dce26a` on 2026-04-20 ("fix: resolve tool duplication and skill parser YAML inconsistencies").
- **Languages:** Python 69% (backend, LangGraph, gateway) / TypeScript 19% (web UI) / HTML 4.6%; 243 contributors.
- **Tag line (2.0):** "An open-source long-horizon SuperAgent harness that researches, codes, and creates. With the help of sandboxes, memories, tools, skills, subagents and message gateway, it handles different levels of tasks that could take minutes to hours."
- **Tag line (1.x):** "A community-driven Deep Research framework."

Two products, same repo. The 1.x graph is the more didactic artifact (clean, small, fully pipeline-shaped); 2.0 is the more ambitious one (general-purpose harness competing with Claude Code / Manus / OpenHands). Both are instructive for different reasons.

## Architecture — 1.x: the LangGraph deep-research pipeline

DeerFlow 1.x is a LangGraph `StateGraph` with six logical nodes, executed largely in a fixed order with conditional edges for the research loop:

1. **Coordinator.** The entry node. Small-talk filter, routes trivial queries back to the user without invoking the rest of the graph, and hands research intent to the planner. Responsible for lifecycle control and the user-facing chat surface.
2. **Planner.** Decomposes the user question into a structured **research plan**: an ordered list of steps, each tagged `research` or `processing` (code). Emits a typed `Plan` object — the system's core artifact. The planner is also the replanning node: after observations come back, it decides whether the plan is complete (`has_enough_context: true`) or needs more steps.
3. **Human feedback.** A LangGraph interrupt node. The plan is rendered to the user; the user replies `[ACCEPTED]` to continue or `[EDIT PLAN] <natural-language edits>` to have the planner regenerate. Auto-accept is available via an API flag for non-interactive runs. This is the cleanest productised HITL plan-edit loop I have seen in an open-source agent.
4. **Research team (router).** A dispatcher: for each unexecuted plan step, route to Researcher or Coder based on the step's type tag.
5. **Researcher.** A ReAct-style subagent with search and crawl tools: Tavily (default), BytePlus InfoQuest, Brave, DuckDuckGo, Arxiv, Searx/SearxNG, plus Jina for web fetch, plus any MCP-provided tools the user has wired in. Writes back an `observation` field on its plan step.
6. **Coder.** A ReAct subagent with a Python REPL for numeric work, quick data wrangling, and verification.
7. **Reporter.** After all plan steps complete, the Reporter synthesises observations into a long-form report with citations. Template-selectable: academic, business briefing, tech report. Supports a **Notion-style block editor** on the frontend for AI-assisted post-edits.

Graph shape (simplified):

```
START → coordinator → planner → human_feedback ──accepted──► research_team
                        ▲                                   │
                        └──────── replan ◄───────────────┐  │
                                                         │  ▼
                                           research_team ─► researcher ─┐
                                                       └─► coder ───────┤
                                                                        ▼
                                                                observations
                                                                        │
                                                    (more steps? loop ──┘
                                                     else → reporter → END)
```

State is a typed `AgentState` with `messages`, `plan`, `observations`, `locale`, `current_plan_iteration`, and a `research_topic`. All nodes mutate the same state dict; the graph is deterministic and re-entrant (you can resume from any checkpoint because LangGraph persists state).

## Architecture — 2.0: the SuperAgent harness

DeerFlow 2.0 drops the fixed pipeline and adopts a **lead-agent + dynamic-subagent** topology, closer to Claude Code / Manus / Deep Agents than to the 1.x graph.

Top-level components:

- **Frontend.** TypeScript/React web UI. Chat + skill picker + Notion-style block editor for outputs.
- **Gateway API.** REST entry point, OAuth for MCP (supports `client_credentials` and `refresh_token` flows), IM-channel webhooks.
- **LangGraph Platform server.** Hosts the lead agent and spawns subagent graphs. Still LangGraph under the hood — they did not replace the runtime, only the graph topology.
- **Sandbox layer.** Docker (local / compose) by default, with Kubernetes and bare-local modes. Each task or subagent gets its own container with a persistent filesystem view: `skills/`, `workspace/`, `uploads/`, `outputs/`.
- **Skill store.** `skills/public/` ships built-in skills (research, report, slides, web page, image/video generation, Claude Code integration). Skills are Markdown files with optional YAML frontmatter describing trigger conditions, allowed tools, and supporting resources; user skills live in `.skill` archives.
- **Long-term memory.** A persistent KV/vector store keyed by user that records profile, preferences, writing style, project structures, and accumulated knowledge. Survives across sessions.
- **MCP hub.** HTTP and SSE MCP clients with OAuth, three transport types unified via `_get_tools_from_client_session()` (per the DeepWiki index of `src/server/mcp_utils.py`). Tools are attached to agents at runtime via a `mcp_settings` parameter on the chat stream API.
- **IM gateway.** Telegram, Slack, Feishu/Lark, WeChat, WeCom. No public IP required — outbound-initiated long-poll / webhook tunnels. You can drive a long-horizon task from your phone.
- **Observability.** LangSmith and Langfuse tracing integrated out of the box.
- **Default port:** `http://localhost:2026`. (Cute.)

Lead-agent loop: "breaks the prompt into structured sub-tasks, decides which tasks can run in parallel, spawns sub-agents to handle them, and then synthesises the results." Each subagent runs with **scoped context, tools, and termination conditions** — context isolation protects the lead from observation floods, which matches the Deep Agents / Claude Code `task` tool pattern (see [42-langchain-deep-agents.md](42-langchain-deep-agents.md)). Context compaction is described in the docs as "aggressive summarisation and intermediate-result offloading" — large artifacts spill to the sandbox filesystem rather than living in context, same play as [08-context-compaction.md](08-context-compaction.md) and Deep Agents' virtual FS.

## Harness primitives (corpus-vocab mapping)

| Corpus primitive | DeerFlow 1.x | DeerFlow 2.0 |
|---|---|---|
| Plan mode ([03](03-plan-mode.md)) | Explicit `Planner` node with typed `Plan` artifact and `[EDIT PLAN]` HITL gate | Lead agent emits sub-tasks into the shared state; no separate "plan review" gate by default |
| Subagent delegation ([02](02-subagent-delegation.md)) | Researcher / Coder — specialised ReAct loops inside the fixed graph | Dynamic: lead spawns N subagents with per-task system prompts and tool allowlists |
| HITL ([23](23-human-in-the-loop.md)) | First-class `human_feedback` interrupt on the plan; user edits in natural language; replan loop | HITL is opt-in via the chat UI; less structured than 1.x but richer action surface (can pause mid-task) |
| Context compaction ([08](08-context-compaction.md)) | Step-scoped observations in state; reporter reads the whole observation set | Aggressive summarisation + offload to sandbox FS + per-subagent isolated context |
| Tool / verifier loop ([11](11-verifier-evaluator-loops.md)) | Coder's Python REPL doubles as a numeric verifier; reporter has no verifier step | Skills can encode verifiers; no built-in judge loop |
| Memory ([09](09-memory-files.md)) | Thread-scoped LangGraph checkpoint state only | Persistent long-term memory across sessions: profile, preferences, style, accumulated knowledge |
| MCP ([07](07-model-context-protocol.md)) | MCP servers attachable per-agent via `mcp_settings` | Full MCP hub with OAuth, HTTP/SSE/stdio transports |
| Skills ([04](04-skills.md)) | Implicit — prompt templates only | Explicit Markdown + frontmatter skill format, hot-loadable `.skill` archives |
| Sandbox / shell | Python REPL only | Docker-sandboxed shell + FS per subagent |
| Observability ([24](24-observability-tracing.md)) | LangSmith tracing | LangSmith + Langfuse, out-of-the-box IM channel traces |

The 1.x → 2.0 jump is, in corpus terms, the transition from a **fixed-graph artifact-passing harness** (MetaGPT-family; see [20](20-metagpt-role-based-workflows.md)) to a **dynamic subagent-dispatch harness** (Claude-Code / Deep-Agents family; see [29](29-dive-into-claude-code.md), [42](42-langchain-deep-agents.md)). Both versions remain in the repo, which is itself a pedagogical artifact — you can diff the two topologies directly.

## Multi-agent pattern — how coordination actually works

**1.x: artifact-passing pipeline.** The `Plan` object is the system's coordination protocol. Each researcher/coder subagent does one step, writes one observation, and exits. No researcher ever sees another researcher's output directly; all cross-agent communication goes through the shared `observations` list in state, mediated by the planner's decision to replan or hand to the reporter. This is the same "structured artifacts, not free-form chat" discipline [MetaGPT](20-metagpt-role-based-workflows.md) preaches, but thinner — DeerFlow 1.x has one artifact (`Plan` + `observations`) instead of MetaGPT's five (PRD, design, task list, code, tests).

**2.0: lead-agent fan-out.** The lead agent acts as a supervisor, though the DeerFlow team explicitly rejects pure LangGraph "supervisor" routing (see issue #270); they prefer an explicit shared-state pattern over dynamic routing because it makes reproducibility, checkpointing, and HITL interception easier. A task fan-out looks like: lead emits N subtasks → each gets a container + prompt + tool allowlist → subagents write outputs to the shared sandbox FS → lead reads back the files (not the chat transcripts) → lead synthesises or iterates.

**Handoff mechanism.** In both versions, handoff is **state-mediated, not message-mediated**. Parents do not forward chat transcripts to children; they forward the *task* and (in 2.0) point children at specific files in the sandbox. Children write results back as files or typed state fields. This is what [43-twelve-harness-patterns.md](43-twelve-harness-patterns.md) calls the "shared scratchpad" pattern, and it is the right call — transcript-forwarding blows up token budgets in long horizons.

**Contrast with sibling frameworks.**

- **Syndicate (your MVP).** Syndicate dispatches subagents via an explicit `task` tool and treats the supervisor's todo list as the coordination object. DeerFlow 2.0 has converged on a very similar shape, but adds (a) persistent long-term memory per user, (b) Docker-sandbox-per-subagent rather than shared workspace, and (c) a first-class Skills format. If Syndicate wants to scale to hours-long runs, the persistent-memory + per-subagent-sandbox combination is the upgrade path DeerFlow has de-risked.
- **MetaGPT ([20](20-metagpt-role-based-workflows.md)).** MetaGPT fixes the roles (PM, Architect, Engineer, QA) to the SDLC and the artifacts (PRD, design, code, tests) to that SDLC. DeerFlow 1.x generalises this: the pipeline stages are fixed (plan → research → report) but the domain is not tied to software; the artifact is just `Plan + observations`. DeerFlow 2.0 abandons role-fixing entirely — subagents are parameterised, not named.
- **LangGraph supervisor pattern.** The upstream LangGraph pattern has a supervisor LLM routing between named workers on each step. DeerFlow chose deterministic pipelines (1.x) or shared-state fan-out (2.0) instead, citing reproducibility. This is a defensible call; dynamic supervisor routing is harder to trace and checkpoint.

## Notable novel contributions

Things DeerFlow shipped that are uncommon in open-source harnesses:

1. **Podcast mode.** A dedicated pipeline that takes a completed research report, generates a two-host podcast script (writer → host-A / host-B dialog), and synthesises audio via **Volcengine TTS** (ByteDance's cloud TTS). Customisable speed, volume, pitch per voice. It is the first open-source harness I am aware of that ships end-to-end *audio-as-a-deliverable* built directly into the research graph. See [48-voiceagentrag-dual-agent.md](48-voiceagentrag-dual-agent.md) for related dual-agent voice patterns.
2. **PPT mode.** Reports → Marp-CLI → PowerPoint/PDF slide decks. Implementation is a thin skill that takes the Reporter output, reformats it with slide-break heuristics, and shells out to Marp.
3. **Prose mode.** Short-form writing pipeline (Notion-style blocks) separate from the full report pipeline — useful for quick summaries without invoking the full planner.
4. **MCP integration with OAuth.** Most open-source harnesses support stdio MCP only; DeerFlow ships HTTP + SSE with OAuth `client_credentials` and `refresh_token` flows, which is what you need to plug into real enterprise MCP servers (Atlassian, Okta-gated internal services).
5. **IM-channel gateway.** Telegram / Slack / Feishu / WeChat / WeCom with no public IP requirement. You can kick off a 2-hour research run from a phone chat and get the deliverable (report + podcast + slides) pushed back into the same channel. This is a *product* feature more than a research one, but it is a useful existence proof of what "agent as long-horizon worker" looks like in practice.
6. **Knowledge-base integrations.** First-class adapters for RAGFlow, Qdrant, Milvus, VikingDB, MOI, and Dify — so the Researcher can query a company's internal KB alongside web search.
7. **Dual-topology repo.** Keeping 1.x and 2.0 co-maintained is unusual; it gives teams a choice between "deterministic research pipeline I can audit" and "open-ended superagent I can extend."

## Production-readiness

- **Docker-first deployment.** `make docker-init && make docker-start` brings up Gateway API + LangGraph server + frontend + Nginx + sandbox. `make up` is the production variant.
- **Config.** `config.yaml` for model providers (any OpenAI-compatible API), `.env` for secrets, `make setup` wizard for guided first-run.
- **Runtime requirements.** Python 3.12+, Node 22+.
- **Observability.** LangSmith and Langfuse tracing are wired through the LangGraph runtime, so every node invocation, token count, tool call, and subagent run is traced. IM-gateway events are traced too, which matters for long runs where you need to debug a task that spanned hours.
- **Security posture.** The apidog guide's baseline — "keep deployment local/trusted by default, add IP allowlists for cross-network access, reverse-proxy with strong auth, isolate network segments" — is appropriate; the harness intentionally has high-privilege capabilities (shell execution, arbitrary file writes inside the sandbox). A CVE (CVE-2026-40518) was filed against a pre-`2176b2b` commit for a path-traversal in bootstrap-mode custom-agent creation; fixed in main. This is a useful data point: even a well-engineered sandboxed harness will have agent-boundary escape bugs; treat the sandbox as a soft perimeter, not a hard one. See [26-linuxarena-production-agent-safety.md](26-linuxarena-production-agent-safety.md), [35-malicious-intermediary-attacks.md](35-malicious-intermediary-attacks.md).
- **Benchmarks.** DeerFlow is cited in multiple 2026 deep-research benchmark papers (DEER, LiveResearchBench); "only DeerFlow+ and Gemini Deep Research exceed ODR" on at least one benchmark. Vanilla DeerFlow has been reported brittle on very long search-heavy queries, terminating with `max_total_tokens` — a useful reminder that aggressive compaction is necessary, not optional, at long horizons ([08](08-context-compaction.md), [27](27-horizon-long-horizon-degradation.md)).

## Comparison table

| | **DeerFlow 1.x** | **DeerFlow 2.0** | **Deep Agents** ([42](42-langchain-deep-agents.md)) | **CrewAI** | **AutoGen** | **MetaGPT** ([20](20-metagpt-role-based-workflows.md)) | **Syndicate (MVP)** | **Atlas-Research (MVP)** |
|---|---|---|---|---|---|---|---|---|
| Topology | Fixed graph | Lead + dynamic subagents | Lead + `task` tool | Role-based crew | Conversational group chat | SDLC roles + SOP | Lead + `task` tool | Research pipeline |
| Runtime | LangGraph | LangGraph | LangGraph | Custom | Custom | Custom | Custom | Custom |
| HITL | `[EDIT PLAN]` interrupt | Opt-in pause | Interrupt-based | Minimal | Minimal | Minimal | Limited | Limited |
| Sandbox | Python REPL | Docker per subagent | Shell backend | None | None | None | Shared workspace | Shared workspace |
| MCP | Yes | Yes + OAuth | Yes | Partial | Partial | No | Planned | No |
| Long-term memory | No | Yes (per-user KV) | No | No | No | No | No | No |
| Deliverables | Report, podcast, PPT, prose | + code, web pages, images, video | Code only | Varies | Varies | Code | Varies | Report |
| Observability | LangSmith | LangSmith + Langfuse | LangSmith | Basic | Basic | Basic | TBD | TBD |
| IM gateway | No | 5 channels | No | No | No | No | No | No |

Interpretation for your MVPs:

- **Atlas-Research** is closest in spirit to DeerFlow 1.x. If you want to ship faster, 1.x's `Plan`-object + `[EDIT PLAN]` HITL gate + template-selectable reporter is a ~300-line LangGraph that you can implement in an afternoon and which will produce audit-ready research reports. Steal the plan artifact shape and the `research|processing` step tagging; it is the smallest viable plan schema.
- **Syndicate** should look at DeerFlow 2.0's Skills format. Your current skill surface is prompt-only; DeerFlow's Markdown-with-frontmatter format (trigger conditions, allowed-tools allowlist, referenced resources) is a small amount of structure that unlocks non-code-contributors writing skills — which is the actual adoption unlock.

## Takeaways — copy-worthy ideas

1. **Typed `Plan` artifact as coordination object.** `{steps: [{type: research|processing, description, expected_outcome, observation?}], iterations}`. Cheap, auditable, trivially HITL-editable. Atlas-Research should ship this as v1.
2. **`[EDIT PLAN]` natural-language interrupt.** The HITL gate is not a form; it is a chat message with a sentinel token, and the planner is prompted to interpret the free-form edits and regenerate. This is the lowest-friction plan-edit UX I have seen in an OSS agent. Generalisable to any harness with a plan node.
3. **Deliverable diversity as a product lever.** Shipping podcast + PPT + prose on top of the same research graph turns one pipeline into three products. The compute cost is marginal (TTS and Marp are cheap); the perceived value is not.
4. **Dual-topology co-maintenance.** Keeping the pedagogical 1.x graph alive next to the 2.0 harness is a deliberate choice. For internal corpora like this one, it is worth advocating for: the simple version documents the hard version.
5. **MCP with OAuth.** Enterprise adoption requires more than stdio MCP. If we are serious about our harnesses plugging into production tools, HTTP+SSE+OAuth is the bar.
6. **State-mediated handoff, not transcript-forwarding.** Subagents read from and write to files / typed state fields. Parents never get the children's chat transcripts. This is the single most important architectural choice for long-horizon runs — it keeps token budgets bounded.
7. **Per-subagent Docker sandbox.** If your harness is shared-workspace today, moving each subagent into its own container is a one-week upgrade that unlocks parallel execution + blast-radius isolation + clean per-task observability traces.

**The single most distinguishing harness feature:** DeerFlow's **typed, natural-language-editable Plan artifact with an `[EDIT PLAN]` HITL interrupt** is the cleanest productised plan-mode-with-human-edit loop in open-source agents. It is what a Claude Code "plan mode" looks like when you actually ship it as a user-facing product instead of a debugger feature.

## References

- [github.com/bytedance/deer-flow](https://github.com/bytedance/deer-flow) — main 2.0 repo, README, MIT license.
- [github.com/bytedance/deer-flow/tree/main-1.x](https://github.com/bytedance/deer-flow/tree/main-1.x) — 1.x branch (deep-research pipeline).
- [github.com/bytedance/deer-flow/blob/main/README.md](https://github.com/bytedance/deer-flow/blob/main/README.md) — 2.0 README.
- [github.com/bytedance/deer-flow/issues/270](https://github.com/bytedance/deer-flow/issues/270) — supervisor-pattern design discussion.
- [deepwiki.com/bytedance/deer-flow](https://deepwiki.com/bytedance/deer-flow) — auto-generated architecture index, incl. `2-system-architecture`, `2.1-multi-agent-workflow`, `4.3-prompt-system`, `5.5-subagent-system`.
- [deerflow.tech](https://deerflow.tech/) — official marketing site.
- [dev.to/arshtechpro/deerflow-20-what-it-is-how-it-works-and-why-developers-should-pay-attention-3ip3](https://dev.to/arshtechpro/deerflow-20-what-it-is-how-it-works-and-why-developers-should-pay-attention-3ip3) — 2.0 architecture walkthrough.
- [apidog.com/blog/deer-flow-guide-2026](https://apidog.com/blog/deer-flow-guide-2026/) — 2026 setup & security guide.
- [medium.com/data-science-in-your-pocket/bytedance-deerflow-multi-ai-agent-framework-for-deep-research-acfbc4d90fbd](https://medium.com/data-science-in-your-pocket/bytedance-deerflow-multi-ai-agent-framework-for-deep-research-acfbc4d90fbd) — 1.x role breakdown.
- [redpacketsecurity.com/cve-alert-cve-2026-40518-bytedance-deer-flow](https://www.redpacketsecurity.com/cve-alert-cve-2026-40518-bytedance-deer-flow/) — path-traversal CVE pre-`2176b2b`.
- [arxiv.org/abs/2512.17776](https://arxiv.org/abs/2512.17776) — DEER benchmark citing DeerFlow performance.
- [arxiv.org/html/2510.14240v3](https://arxiv.org/html/2510.14240v3) — LiveResearchBench citing DeerFlow.
- Internal cross-refs: [02](02-subagent-delegation.md), [03](03-plan-mode.md), [04](04-skills.md), [07](07-model-context-protocol.md), [08](08-context-compaction.md), [09](09-memory-files.md), [20](20-metagpt-role-based-workflows.md), [23](23-human-in-the-loop.md), [24](24-observability-tracing.md), [26](26-linuxarena-production-agent-safety.md), [27](27-horizon-long-horizon-degradation.md), [29](29-dive-into-claude-code.md), [35](35-malicious-intermediary-attacks.md), [42](42-langchain-deep-agents.md), [43](43-twelve-harness-patterns.md), [48](48-voiceagentrag-dual-agent.md).
