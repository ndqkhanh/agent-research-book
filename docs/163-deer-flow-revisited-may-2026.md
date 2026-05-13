# 163 — DeerFlow 2.0 Revisited (May 2026): From Deep Research to Super Agent Harness

**Repository.** https://github.com/bytedance/deer-flow — author: **ByteDance** + community contributors — MIT license — Python 3.12+ backend, Node.js 22+ frontend, Astro for the website — current line on `main` branch is **DeerFlow 2.0** (relaunched February 28, 2026, claimed #1 on GitHub Trending after launch); the previous `1.x` line is maintained on the `main-1.x` branch — the original "Deep Research framework" that file [65-deer-flow-bytedance](65-deer-flow-bytedance.md) covered. This chapter is the **revisited deep-dive** that reflects the ground-up rewrite to 2.0 and the broader architectural ambition of the new line.

**One-line definition.** DeerFlow (**D**eep **E**xploration and **E**fficient **R**esearch **Flow**) 2.0 is an open-source **super agent harness** that orchestrates **sub-agents**, **memory**, and **sandboxes** to do almost anything — powered by **extensible skills**. Where DeerFlow 1.x was a multi-agent deep-research workflow built on LangGraph, DeerFlow 2.0 is a *general-purpose harness* with a LangGraph-compatible Gateway API (`/api/langgraph/*`), an integrated sandbox layer with three execution modes (local, Docker, Kubernetes), six instant-messenger channel integrations (Telegram, Slack, Feishu, WeChat, WeCom, DingTalk), MCP server support with OAuth, Claude-Code-OAuth and Codex-CLI ACP integration, LangSmith + Langfuse tracing, an embedded Python client, and ByteDance Volcengine + BytePlus InfoQuest as the recommended cloud / search backbones. The pivot from "deep research framework" to "super agent harness" is the single biggest signal in 2026 about where the open-source agent-harness category is going.

## Why this chapter is necessary even though file 65 exists

File [65-deer-flow-bytedance](65-deer-flow-bytedance.md) (April 2026) covered DeerFlow 1.x — a multi-agent LangGraph-based deep-research workflow with planner, researcher, coder, and reporter agents, browsing tools, and Python sandbox. Two months later, DeerFlow 2.0 launched as a **ground-up rewrite that shares no code with 1.x**.

This is not an incremental release. The README says it explicitly:

> **DeerFlow 2.0 is a ground-up rewrite.** It shares no code with v1. If you're looking for the original Deep Research framework, it's maintained on the `1.x` branch — contributions there are still welcome. Active development has moved to 2.0.

Three reasons the rewrite warrants a separate chapter:

1. **Different architectural target**. 1.x is a deep-research workflow. 2.0 is a *super agent harness* — a general-purpose substrate for orchestrating sub-agents, memory, sandboxes, and skills. The category has moved.
2. **Different tech stack**. 1.x was Python-only with LangGraph as the orchestrator. 2.0 is Python 3.12+ backend + Node.js 22+ frontend + Astro website + Docker/Kubernetes deployment, with LangGraph as a *compatibility surface* (the Gateway API translates DeerFlow-native routes to LangGraph-compatible paths) rather than the core orchestration.
3. **Different positioning in the landscape**. 1.x competed with other open-source DR frameworks (RAGFlow, Archon, deer-flow itself). 2.0 competes with the broader "open-source agent runtime" category — Claude Code, OpenClaw, Cline, Cursor — by offering a complete harness that can be adapted to many domains, not just research.

For the harness-engineering canon, DeerFlow 2.0 is the most important *general-purpose* open-source harness alongside Claude Code, OpenClaw, and Feynman. It is the open-source-with-MIT-license alternative that combines deep research as a default workflow with the extensibility to be repurposed.

## What's new in 2.0 vs 1.x

A side-by-side comparison:

| Aspect | DeerFlow 1.x (April 2026) | DeerFlow 2.0 (May 2026+) |
|--------|---------------------------|----------------------------|
| Category | Deep-research workflow | Super agent harness |
| Orchestration | LangGraph (native) | LangGraph-compatible Gateway API + native runtime |
| Frontend | (minimal) | Full Node.js/React frontend, port 2026 |
| Sandbox | Python sandbox | Local + Docker + Kubernetes (provisioner service) |
| Skills | Limited | First-class extensible skills system (`skills/` dir, `DEER_FLOW_SKILLS_PATH`) |
| Sub-agents | Planner / Researcher / Coder / Reporter | Configurable; lead_agent dispatches subagents by name |
| Memory | Workflow state | First-class long-term memory with sample fixtures (`Settings > Memory`) |
| MCP | Not native | Native MCP support with OAuth (`client_credentials`, `refresh_token`) |
| IM channels | None | 6: Telegram, Slack, Feishu, WeChat, WeCom, DingTalk |
| Tracing | Custom | LangSmith + Langfuse with both-providers mode |
| Auth integrations | API keys only | + Claude Code OAuth, Codex CLI ACP, Anthropic auth tokens |
| Cloud backbone | Generic | ByteDance Volcengine + BytePlus InfoQuest as recommended |
| Models recommended | Various | Doubao-Seed-2.0-Code, DeepSeek v3.2, Kimi 2.5 |
| Embedded client | None | Python client embeddable in user code |
| Multi-language docs | English only | English + Chinese + Japanese + French + Russian |
| Deployment options | Single command | Local / Local Daemon / Docker Dev / Docker Prod with comprehensive sizing guidance |

The 2.0 line is *substantially more ambitious*. It is no longer competing with deep-research frameworks; it is competing with general-purpose agent runtimes.

## Architecture overview

```
                                                ┌──────────────────────┐
                                                │      User             │
                                                │ (CLI / Web / IM)     │
                                                └──────────┬───────────┘
                                                           │
                  ┌──────────────────────────────────────┐ │
                  │  IM Channels: Telegram, Slack,      │ │
                  │  Feishu, WeChat, WeCom, DingTalk    │─┘
                  └──────────────────────────────────────┘
                                   │
                                   ▼
                          ┌────────────────┐
                          │ Gateway API    │  ← /api/langgraph/* (compat)
                          │ (nginx + Node) │  ← /api/* (native)
                          └────────┬───────┘
                                   │
                                   ▼
                  ┌────────────────────────────────────┐
                  │  Lead Agent (orchestrator)         │
                  │  + Sub-agents (configurable)       │
                  │  + Memory (long-term)              │
                  │  + Skills (Markdown, extensible)   │
                  │  + Context engineering             │
                  └─────────────┬──────────────────────┘
                                │
            ┌───────────────────┼───────────────────────┐
            ▼                   ▼                       ▼
   ┌─────────────────┐  ┌────────────────┐  ┌──────────────────────┐
   │  Sandbox        │  │  MCP Servers   │  │  External Tools       │
   │  (Local /       │  │  (OAuth flows) │  │  - InfoQuest          │
   │  Docker /       │  │                │  │  - Web search         │
   │  Kubernetes)    │  │                │  │  - File system        │
   └─────────────────┘  └────────────────┘  └──────────────────────┘
            │                   │                       │
            └───────────────────┴───────────────────────┘
                                │
                                ▼
                     ┌──────────────────────┐
                     │  Tracing Layer       │
                     │  - LangSmith         │
                     │  - Langfuse          │
                     │  - Both-providers    │
                     └──────────────────────┘
```

The core abstractions are:

1. **Lead Agent + Sub-agents** — the multi-agent dispatch layer.
2. **Skills** — Markdown skill files in `skills/` that any agent can load (configurable via `DEER_FLOW_SKILLS_PATH`).
3. **Sandbox** — three execution modes for code-running tasks; isolated filesystem.
4. **Memory** — first-class long-term memory with a sample fixture for review (`scripts/load_memory_sample.py`).
5. **MCP** — native Model Context Protocol integration with OAuth.
6. **Channels** — six IM channels for receiving tasks from messaging apps.
7. **Tracing** — LangSmith and/or Langfuse for observability.
8. **Gateway API** — LangGraph-compatible HTTP surface so existing LangGraph clients can drive DeerFlow.

## The four pillars of 2.0

The README's "Core Features" section names four axes of capability — these are the architectural pillars worth pulling out:

### Pillar 1 — Skills & Tools

Skills are *extensible Markdown instructions* that prepend to agent context when relevant. The architecture mirrors what Feynman ([155](155-feynman-multi-agent-research-harness.md)), HeavySkill ([156](156-heavyskill-parallel-reasoning-deliberation.md)), and Paper2Agent ([162](162-paper2agent-reimagining-papers-as-agents.md)) all do — Markdown-as-skill-format is now the de facto standard.

DeerFlow 2.0 adds two integrations:

- **Claude Code Integration**: works with Claude Code OAuth so that existing Claude Code skills can be invoked via DeerFlow.
- **Custom skills directory**: configurable via `DEER_FLOW_SKILLS_PATH`; defaults to `skills/` under project root.

The skills system also bridges to MCP: skills can declare MCP server dependencies, and the harness ensures the relevant MCP servers are loaded before invoking the skill.

### Pillar 2 — Sub-Agents

The `lead_agent` is the orchestrator. It can dispatch to *named* sub-agents:

```yaml
channels:
  session:
    assistant_id: lead_agent  # or a custom agent name
    config:
      recursion_limit: 100
    context:
      thinking_enabled: true
      is_plan_mode: false
      subagent_enabled: false
```

The configuration shows three important context flags:

- `thinking_enabled`: turns on thinking-mode reasoning (for models that support it — Doubao-Seed-2.0-Code, DeepSeek v3.2 Thinking, Kimi 2.5).
- `is_plan_mode`: routes the agent into [03-plan-mode](03-plan-mode.md)-style planning before execution.
- `subagent_enabled`: enables sub-agent dispatch.

These flags can be set globally, per channel, or per user. The config in the README example shows VIP users getting `subagent_enabled: true` with a higher recursion limit while default users get a slimmer config.

### Pillar 3 — Sandbox & File System

Three sandbox execution modes:

- **Local Execution**: runs sandbox code directly on the host. Fast; no isolation; for trusted code only.
- **Docker Execution**: runs sandbox code in isolated Docker containers. The default for production-grade isolation.
- **Docker Execution with Kubernetes**: routes via a `provisioner` service for Kubernetes pod-based execution. For multi-tenant or large-scale deployments.

The provisioner service is only started when `config.yaml` uses `sandbox.use: deerflow.community.aio_sandbox:AioSandboxProvider` with `provisioner_url` — i.e., explicit opt-in to Kubernetes mode.

This three-mode design is *the right architecture*. Most production agent runtimes ship with one sandbox mode and force users to swap if they need a different one. DeerFlow 2.0 makes the choice configurable. Compare with [125-system-level-production-patterns](125-system-level-production-patterns.md).

### Pillar 4 — Context Engineering & Long-Term Memory

The README references "Context Engineering" as a distinct feature, alongside long-term memory. The implementation details are in `backend/docs/MEMORY_SETTINGS_REVIEW.md`.

DeerFlow 2.0's long-term memory is configured via `Settings > Memory` in the web UI. The sample-fixture loader (`python scripts/load_memory_sample.py`) lets reviewers populate a fresh install with realistic memory state for testing. This is the right operational convenience for a memory feature: showing reviewers what populated memory looks like.

The architecture is compatible with MEMTIER-style external storage ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)) — the long-term memory layer can be backed by external substrates, and the harness's plug-in architecture supports custom memory providers.

## The six IM channels

DeerFlow 2.0's most distinctive operational feature: **six IM channels** for receiving tasks from messaging apps with no public IP required.

| Channel | Transport | Difficulty |
|---------|-----------|------------|
| Telegram | Bot API (long-polling) | Easy |
| Slack | Socket Mode | Moderate |
| Feishu / Lark | WebSocket | Moderate |
| WeChat | Tencent iLink (long-polling) | Moderate |
| WeCom (WeChat Work) | WebSocket | Moderate |
| DingTalk | Stream Push (WebSocket) | Moderate |

The breadth here is unique. Most open-source agent runtimes support Slack and maybe Telegram. DeerFlow 2.0 covers the full Chinese enterprise messenger stack (Feishu, WeCom, WeChat, DingTalk) plus the Western options. This is direct evidence of ByteDance's positioning toward the Chinese enterprise market — but the IM-channel infrastructure also benefits non-Chinese users who want their agent reachable from anywhere.

The channel system is unified: the same agent runs behind Telegram and Slack and DingTalk; each channel adapter handles its specific messaging API. The harness sees only "incoming task" events.

Per-user session configuration is supported:

```yaml
session:
  users:
    "123456789":
      assistant_id: vip-agent
      config:
        recursion_limit: 150
      context:
        thinking_enabled: true
        subagent_enabled: true
```

VIP users get a different agent, a higher recursion limit, and thinking + subagent enabled. This *per-user routing* is operationally important for any harness that will be exposed to many users with varying permission levels.

## OAuth integrations and the agent-runtime ecosystem

DeerFlow 2.0 integrates with three major agent-runtime ecosystems via OAuth:

### Claude Code OAuth

```yaml
display_name: Claude Sonnet 4.6 (Claude Code OAuth)
use: deerflow.models.claude_provider:ClaudeChatModel
model: claude-sonnet-4-6
max_tokens: 4096
supports_thinking: true
```

DeerFlow can use a Claude Code OAuth token to call Anthropic models, inheriting the user's Claude Code subscription. This means a user with Claude Code Pro/Max access can drive DeerFlow without managing a separate Anthropic API key.

### Codex CLI / Codex ACP

OpenAI's Codex CLI uses an Anthropic Connector Protocol (ACP) adapter (`npx -y @zed-industries/codex-acp`). DeerFlow 2.0 supports this as an `acp_agents.codex` config entry. The ACP layer provides a uniform protocol for calling external coding CLIs as if they were sub-agents.

### LangSmith + Langfuse

For observability, DeerFlow supports both LangSmith (LangChain ecosystem) and Langfuse (independent open-source observability). Notably, both can be enabled simultaneously — the README documents a "Using Both Providers" mode that sends traces to both. This is the right discipline: tracing should never be vendor-locked.

## Recommended models — and why they matter

The README's "Recommended Models" guidance is direct:

> We strongly recommend using **Doubao-Seed-2.0-Code**, **DeepSeek v3.2** and **Kimi 2.5** to run DeerFlow.

The choice is significant for two reasons:

1. **Doubao-Seed-2.0-Code** is ByteDance's own coding-optimised model, served via Volcengine. Recommending your own first-party model is rational vendor strategy, but it works because Doubao-Seed-2.0-Code is genuinely competitive with frontier coding models.
2. **DeepSeek v3.2** and **Kimi 2.5** are Chinese open-weight models. The recommendation reflects the Chinese frontier-model ecosystem's maturity in late 2025 / early 2026.

For non-Chinese users, the harness supports OpenAI, Anthropic, Gemini, and OpenRouter via the standard config patterns. The recommendation is not a constraint — it is a *baseline* that ByteDance has tested.

## InfoQuest as the search backbone

DeerFlow 2.0 integrates **BytePlus InfoQuest** as a search and crawling toolset. The README:

> DeerFlow has newly integrated the intelligent search and crawling toolset independently developed by BytePlus — InfoQuest (supports free online experience).

This is functionally a hybrid API+browser search system, similar in shape to Tavily, Exa, or Perplexity API. For DR workflows, InfoQuest is the recommended primary search tool; alternative backends (Tavily, Bing, Google CSE) are supported via standard config.

## Deployment sizing guidance

One of the most operationally useful sections of the README is the Deployment Sizing table:

| Target | Starting point | Recommended | Notes |
|--------|----------------|-------------|-------|
| Local evaluation / `make dev` | 4 vCPU, 8 GB RAM, 20 GB SSD | 8 vCPU, 16 GB RAM | Good for one developer, light session |
| Docker development / `make docker-start` | 4 vCPU, 8 GB RAM, 25 GB SSD | 8 vCPU, 16 GB RAM | Image builds + bind mounts + sandbox containers |
| Long-running server / `make up` | 8 vCPU, 16 GB RAM, 40 GB SSD | 16 vCPU, 32 GB RAM | Shared use, multi-agent runs, heavier sandbox |

Most agent-harness READMEs gloss over hardware requirements, leaving builders to discover them through OOM crashes. DeerFlow 2.0's explicit sizing guidance is the kind of operational maturity that distinguishes production-grade frameworks from research demos.

The note "If CPU or memory usage stays pinned, reduce concurrent runs first, then move to the next sizing tier" is good operational hygiene — capacity planning before vertical scaling.

## Setup wizard and `make doctor`

The setup experience is strikingly polished:

```bash
git clone https://github.com/bytedance/deer-flow.git
cd deer-flow
make setup    # Interactive wizard: LLM provider, search, execution preferences
              # Generates minimal config.yaml + writes keys to .env
              # Takes ~2 minutes
make doctor   # Verifies setup, gives actionable fix hints
make dev      # Start in dev mode
```

`make doctor` is the diagnostic tool that most agent runtimes lack. It checks Node.js 22+, pnpm, uv, nginx, config files, and provides explicit fix steps for missing dependencies. Operationally, it saves users from the "I don't know why it doesn't work" failure mode.

The interactive wizard generates a minimal `config.yaml` rather than the full template — most users never need 90% of the config options. Advanced users run `make config` to copy the full template.

## "From Deep Research to Super Agent Harness" — the architectural narrative

The README has a section by this name. The narrative arc:

1. **DeerFlow 1.x** focused on deep research. The agent took a research question, planned a multi-step investigation, browsed the web, and produced a structured report. This was the *vertical* — a complete workflow for one task type.
2. **DeerFlow 2.0** generalises. The same multi-agent + sandbox + memory + skills primitives that made deep research work also work for: coding tasks, customer support workflows, data analysis pipelines, internal automation, IM-driven assistants. Deep research becomes one of many workflows, not the entire framework.

This is the same evolutionary arc that Claude Code went through: from a coding-specialised assistant to a general-purpose agent runtime that happens to have strong coding skills. The arc seems to be the natural maturation path: domain-specialised → general-purpose with domain skills.

For the broader open-source ecosystem, DeerFlow 2.0's pivot is the strongest signal yet that the "deep research framework" category is consolidating into the broader "agent harness" category. Future open-source DR projects are likely to be skill libraries on top of agent harnesses, not standalone frameworks.

## Comparison with adjacent harnesses

How DeerFlow 2.0 sits in the open-source-harness landscape:

| Harness | Domain | Open Source | License | Distinctive feature |
|---------|--------|-------------|---------|---------------------|
| **DeerFlow 2.0** | General | Yes | MIT | 6 IM channels; LangGraph-compatible Gateway; OAuth integrations with Claude Code + Codex |
| Feynman ([155](155-feynman-multi-agent-research-harness.md)) | Research | Yes | MIT | Mandatory Verifier; slug-keyed file-based handoffs; provenance discipline |
| OpenClaw ([52](52-dive-into-open-claw.md)) | General | Yes | MIT | MEMTIER-style memory plugin architecture; Lobster engine for multi-agent |
| Claude Code ([29](29-dive-into-claude-code.md)) | Coding | Closed | Proprietary | First-mover; tight CLI UX; subagent SDK |
| RAGFlow ([63](63-ragflow-agent-patterns.md)) | RAG-heavy | Yes | Apache | RAG-first architecture; document parsing pipeline |
| LobeHub ([64](64-lobehub-ai-framework.md)) | Chat | Yes | MIT | Web-UI-first; plugin marketplace |
| Archon ([61](61-archon-harness-builder.md)) | Builder | Yes | MIT | Builds-other-agents pattern |
| crewAI ([164](164-crewai-multi-agent-framework.md)) | General | Yes | MIT | Lean abstractions; Crew + Flow paradigms; enterprise AMP suite |

The differentiation matrix:

- **DeerFlow 2.0 wins on**: IM channel coverage (6 channels, especially Chinese enterprise), Claude Code + Codex OAuth integrations, sandbox flexibility (Local/Docker/K8s), deployment sizing guidance, multi-language docs.
- **Feynman wins on**: source-grounded research discipline, Verifier as structural primitive, slug-keyed filesystem.
- **OpenClaw wins on**: plugin architecture for memory and other concerns, simpler embedding into existing systems.
- **Claude Code wins on**: out-of-box UX, model integration, day-one Anthropic support.
- **crewAI wins on**: framework simplicity, ease of getting started, Python-only stack.

The right choice depends on the use case. DeerFlow 2.0 is the strongest pick when you need (a) IM-driven agents, (b) flexible sandbox deployment, (c) integration with multiple existing agent runtimes (Claude Code, Codex), or (d) Chinese-market deployment.

## Practical takeaways

For builders evaluating DeerFlow 2.0:

### 1. Use the LangGraph compatibility surface for migration

The Gateway API exposes `/api/langgraph/*` paths that are compatible with LangGraph clients. If you already have a LangGraph-based agent and want to migrate to DeerFlow's runtime, this is the on-ramp. Rewrite the agent inside DeerFlow's framework, but keep your existing LangGraph clients pointing at the new server.

### 2. Treat IM channels as the production interface

Most open-source agents are CLI-first or web-first. DeerFlow 2.0's IM-channel architecture lets you ship the *same agent* via Slack to your team and via Telegram to external users — without rebuilding the integration layer. For internal-tools or automation use cases, this is a significant win.

### 3. Use the sandbox configuration progressively

Start with **Local** sandbox for trusted code in development. Switch to **Docker** sandbox before deploying anywhere with users you don't trust. Move to **Kubernetes** sandbox if you need multi-tenant isolation or large-scale parallel execution. The configuration surface is the same; only the backing implementation changes.

### 4. Enable both LangSmith and Langfuse for observability

Both are good; both have failure modes. Running both gives you redundancy and lets you compare. The DeerFlow config supports both-providers mode out of the box.

### 5. Use `make setup` and `make doctor` even for evaluation

Most "let me try this framework for an hour" evaluations fail because of dependency issues. Run `make doctor` first; it will tell you exactly what's missing. Then the wizard takes 2 minutes.

### 6. Skills are portable across runtimes

DeerFlow 2.0's skills directory supports the same Markdown-with-frontmatter format as Feynman, Claude Code, OpenClaw. If you write skills for one, they (mostly) work in the others. Build skills as portable artefacts.

## Limitations and concerns

Honest enumeration of DeerFlow 2.0's limitations:

1. **Heavy stack**. Python 3.12+ + Node.js 22+ + Docker + nginx is a lot of moving parts. For a simple deep-research workflow, simpler frameworks (Feynman, Paper-Researcher-AI-Agent) ship faster.
2. **ByteDance-flavoured defaults**. Doubao + DeepSeek + Kimi as recommended models, BytePlus InfoQuest as recommended search, Volcengine as recommended cloud. Non-Chinese-market users will need to swap defaults.
3. **2.0 is new**. Released February 28, 2026 — very recent at time of writing. Expect rough edges, bug fixes, and API churn.
4. **Documentation is uneven**. The English README is detailed (760 lines), but some backend docs are still being translated. Chinese docs are fuller; Japanese / French / Russian translations exist but lag.
5. **Multi-agent orchestration is configurable but not opinionated**. Unlike Feynman (which has 4 specific subagents and a clear workflow), DeerFlow 2.0 expects you to define your subagent topology. More flexible; more setup required.
6. **No mandatory verification gate**. Source-grounding and verification are *configurable* through skills, not *enforced* like Feynman's Verifier.

## Where this fits in the canon

- **Updated context**: [65-deer-flow-bytedance](65-deer-flow-bytedance.md) — the 1.x deep-dive (now historical reference).
- **Read alongside**: [155-feynman-multi-agent-research-harness](155-feynman-multi-agent-research-harness.md), [29-dive-into-claude-code](29-dive-into-claude-code.md), [52-dive-into-open-claw](52-dive-into-open-claw.md), [61-archon-harness-builder](61-archon-harness-builder.md), [62-everything-claude-code](62-everything-claude-code.md), [63-ragflow-agent-patterns](63-ragflow-agent-patterns.md), [64-lobehub-ai-framework](64-lobehub-ai-framework.md), [66-meta-harness-landscape](66-meta-harness-landscape.md), [144-build-your-own-harness](144-build-your-own-harness.md), [145-comparing-coding-harnesses](145-comparing-coding-harnesses.md), [164-crewai-multi-agent-framework](164-crewai-multi-agent-framework.md).
- **Substrate-level concerns**: [07-model-context-protocol](07-model-context-protocol.md), [09-memory-files](09-memory-files.md), [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md), [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md), [151-memtier-why-flat-memory-breaks-at-72-hours](151-memtier-why-flat-memory-breaks-at-72-hours.md).
- **Synthesis**: [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md), [166-research-agent-frameworks-synthesis](166-research-agent-frameworks-synthesis.md).

## References

1. Repository: https://github.com/bytedance/deer-flow.
2. Official site: https://deerflow.tech.
3. v1.x branch (preserved): https://github.com/bytedance/deer-flow/tree/main-1.x.
4. ByteDance Volcengine: https://www.volcengine.com/.
5. BytePlus InfoQuest: https://docs.byteplus.com/en/docs/InfoQuest/What_is_Info_Quest.
6. LangGraph: https://langchain-ai.github.io/langgraph/.
7. Codex ACP adapter: `@zed-industries/codex-acp` on npm.
8. Claude Code OAuth: https://docs.anthropic.com/en/docs/claude-code.
9. Recommended models: Doubao-Seed-2.0-Code (Volcengine), DeepSeek v3.2, Kimi 2.5.
10. Adjacent canon chapters listed above.
11. Sample memory fixture loader: `python scripts/load_memory_sample.py`.
