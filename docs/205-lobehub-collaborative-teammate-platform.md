# 205 — LobeHub in May 2026: The Reference Design for Teammates That Grow With You

> **Disambiguation.** This file is a **May-2026 refresh** of LobeHub focused on its consolidation as the **reference design for "teammates that grow with you"** — the explicit framing the project uses on its README. It refreshes [64-lobehub-ai-framework](64-lobehub-ai-framework.md) (April 2026) by treating LobeHub as a *design surface*, not a competitor — i.e., what patterns to lift into the in-tree projects (Polaris, Lyra, argus, and the rest). The broader 2026 collaborative-AI canon is in [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md); the apply plan is in [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md).

## One-line definition

LobeHub is the open-source **collaborative-teammate platform** (76.5k★, Apache-derived license, TypeScript/Next.js + Postgres, mobile + desktop + PWA + web) whose 2025-2026 evolution from "lobe-chat" to "lobehub" formalised five distinctive primitives — **ICPEA five-layer memory** (RFC-144), **Agent Groups + Pages**, **branching conversations**, **MCP-first marketplace** (10,000+ skills, 169,739 indexed), and **client-side IndexedDB mode** — each of which is independently copy-worthy for any in-tree harness aiming to feel like a teammate that grows with the user.

## Why the refresh matters

The April-2026 deep-dive ([64-lobehub-ai-framework](64-lobehub-ai-framework.md)) already established LobeHub's surface area. The May-2026 refresh shifts perspective: LobeHub is no longer *a competitor* to the in-tree projects (it's not a deep-research agent like Polaris, not a research synthesis surface like Lyra, not a skill router like argus); it is the **reference UI/UX layer** for collaborative AI — a *complement* to those projects, not a substitute. Its primitives are independently copy-worthy. Treating LobeHub as the consumer-shell layer that would *wrap* an in-tree research agent (Polaris) or a skill router (argus) reframes the discussion from "build vs buy LobeHub" to "lift these specific primitives into our harnesses."

The May-2026 evidence supporting this reframing: **RFC-144** (the Universal User Memory spec, formalising the I-C-P-E-A five-layer schema) was published in March 2026 and is now the most-cited OSS reference for typed personal memory. **Agent Gateway** (server-resident agents streaming via WebSocket) shipped in early 2026 and made multi-agent sessions scale beyond per-tab Web Workers. **Lobe Pages** generalised co-authored documents into a Notion-style timeline-versioned surface. The **MCP marketplace** crossed 10,000+ skills with one-click install + permission negotiation at install. The **branching chat** UX, originally a 2024 changelog feature, has become the de-facto pattern for "explore alternatives" across the OSS chat-harness landscape (LibreChat, OpenWebUI, AnythingLLM all adopted variants).

Take this refresh seriously and three things change. (1) You evaluate every in-tree project against LobeHub's five distinctive primitives and ask which ones each project should adopt verbatim. (2) You start treating typed personal memory (ICPEA) as a *base substrate* every agent should sit on, not a feature of one product. (3) You stop trying to build your own consumer chat UI from scratch — LobeHub is good enough that even closed in-tree projects should consider it as the consumer-facing shell, with a private backend that does the actual agent work.

## Problem this design surface solves

- How do you make an agent feel like a teammate that *knows the user* across sessions, projects, and devices, without violating privacy boundaries?
- How do you let a user explore alternatives the agent considered without forcing them to compare 5 walls of text?
- How do you make a skill catalog of >10,000 items navigable at the speed of a one-click install?
- How do you compose multiple agents into a session without each browser tab spawning its own runtime worker?
- How do you give a user a co-authored document surface where multiple agents and the human edit concurrently?
- How do you offer privacy-preserving local-first storage *and* a sync-when-you-want-to upgrade path?

## §1 — Project state at May 2026

| Property | Value |
|---|---|
| Canonical repo | [github.com/lobehub/lobehub](https://github.com/lobehub/lobehub) |
| Stars | 76.5k |
| Forks | 15.1k |
| Open issues / PRs | 561 / 197 |
| Total commits | 10,363 (canary) |
| Sister repo | [github.com/lobehub/lobe-chat](https://github.com/lobehub/lobe-chat) — ~69.3k★ |
| Agents repo | [github.com/lobehub/lobe-chat-agents](https://github.com/lobehub/lobe-chat-agents) — 1.1k★, JSON `index.json` driven |
| UI library | [@lobehub/ui](https://www.npmjs.com/package/@lobehub/ui) |
| Icons | [@lobehub/icons](https://www.npmjs.com/package/@lobehub/icons) — 1.7k★ |
| TTS / STT | [@lobehub/tts](https://www.npmjs.com/package/@lobehub/tts) — v5.1.2 (Mar 2 2026) |
| Lint config | [@lobehub/lint](https://www.npmjs.com/package/@lobehub/lint) |
| License | LobeHub Community License (Apache 2.0 text + commercial-distribution clause) |
| Hosted product | [lobehub.com](https://lobehub.com/) |
| Marketplace | 505+ agents, 10,000+ MCP skills, 169,739 total indexed |
| Deployment | Vercel one-click, Docker Compose, Sealos, Zeabur, Alibaba Cloud, RepoCloud, desktop Electron app |

**Mission statement (verbatim from README).** *"the ultimate space for work and life — to find, build, and collaborate with agent teammates that grow with you."* This single sentence is the operating thesis the whole product is shaped to embody.

## §2 — The five distinctive primitives

### 2.1 ICPEA — typed five-layer personal memory (RFC-144)

[RFC-144 Universal User Memory](https://lobehub.com/blog/rfc-144). Memory is *typed* into five layers, each with its own schema, retention policy, and injection budget:

- **I**dentity — stable self-descriptions ("I'm a radiologist at a tertiary hospital"). Always present in context.
- **C**ontext — environmental facts (timezone, current project, organisation). Rotated by recency.
- **P**reference — UI / output preferences (tone, language, citation style). Promoted by edit-evidence.
- **E**xperience — episodic "this happened, this worked" traces. Compressed via Ebbinghaus-style decay.
- **A**ctivity — what the user did (tool invocations, queries). FIFO-rotated; the highest-volume layer.

**Async extractor** (Upstash workflow: gate → extract → validate → insert). Reads completed sessions, prompts an extractor model, writes typed rows. Conversation latency is decoupled from memory write quality — the chat path stays fast, memory is eventually consistent.

**Per-agent layer access control.** An agent declares which layers it can read; e.g., a "travel booking" agent sees Preference + Identity but not Experience.

**Composable Context Engine.** Providers run in stable I → C → P → E → A order before message-time injection; budget allocation per layer is configurable.

**Why this matters as a reference.** OpenAI, Anthropic, Google all converged on similar typed-memory shapes (preferences, projects, facts, separate memory spaces) in 2025-2026. ICPEA is the OSS canonicalisation; it's the cleanest published spec for typed personal memory and the easiest to copy verbatim.

**Cross-link.** [09-memory-files](09-memory-files.md), [185-memory-integration-playbook](185-memory-integration-playbook.md), [184-strongest-memory-techniques-synthesis-may-2026](184-strongest-memory-techniques-synthesis-may-2026.md), [186-mnema-witness-lattice](186-mnema-witness-lattice.md).

### 2.2 Agent Groups + Pages + Agent Gateway

**Agent Groups.** Multi-agent channels where N agents share a thread and can address each other. Implemented over the **Agent Gateway** (added 2026) — a WebSocket service that lets agents run server-resident and stream responses to multiple subscribers. Solves the "every browser tab spawns its own worker" scaling pain.

**Lobe Pages.** Co-authored Markdown document where multiple agents *and* the human edit concurrently. Notion-style timeline preserves prior versions per source. The **shared-context document** is a peer of the chat thread, not a downstream artifact.

**Branching conversations.** Any message can spawn a sibling branch. Tree-structured navigation supersedes linear chat. From the [forkable-chat changelog](https://lobehub.com/changelog/2024-11-27-forkable-chat) — first shipped Nov 2024, now widely imitated.

**Why this matters as a reference.** These three primitives — channels, documents, branches — turn agents from request-response chatbots into **collaborators with state**. A research workflow looks like: branch the chat → fan out to multiple agents in a Group → consolidate findings into a Page → version the Page over time. The composition of three primitives is more expressive than any of them alone.

**Cross-link.** [02-subagent-delegation](02-subagent-delegation.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md), [187-multi-agent-shared-memory-landscape](187-multi-agent-shared-memory-landscape.md).

### 2.3 MCP-first skill catalog with one-click install

**Marketplace.** [lobehub.com/mcp](https://lobehub.com/mcp), [lobehub.com/skills](https://lobehub.com/skills) — 10,000+ MCP-compatible plugins, 169,739 skills indexed.

**One-click install.** Pick a server in the UI, the host writes a config block, the server is reachable on the next message.

**Permission negotiation at install (not first-use).** OAuth scopes / credential injection at install time, surfacing over-broad permissions when the user is paying attention rather than mid-task.

**Hosted MCP integration.** Cloudflare Workers, Smithery, Mintlify, Glama, FastMCP Cloud — LobeHub's marketplace is a directory layer over these underlying providers.

**Convergence with Anthropic Skills.** Anthropic's `SKILL.md` standard ([anthropics/skills](https://github.com/anthropics/skills)) is converging with MCP — a skill can declare an MCP server as a dependency, and an MCP server can ship a skill. LobeHub straddles both.

**Why this matters as a reference.** The marketplace is **not just discovery** — it is a contributor-onboarding ramp. Each agent is a PR to `index.json`, no code, schema-validated by CI, auto-deployed. The network effect compounds without LobeHub having to staff curators. This pattern is the cheapest way to grow a skill ecosystem.

**Cross-link.** [04-skills](04-skills.md), [07-model-context-protocol](07-model-context-protocol.md), [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md), [176-skill-discovery-curator-oss-landscape-may-2026](176-skill-discovery-curator-oss-landscape-may-2026.md).

### 2.4 Client-side IndexedDB mode

**Source.** [LobeHub local DB docs](https://lobehub.com/docs/usage/features/database).

**Default storage.** **IndexedDB + Dexie ORM** in the browser. Image binaries stored directly; experimental **CRDT** sync for multi-device.

**Switch to server-DB.** Postgres + Drizzle when teams need multi-user or admin features. Same ORM-level API, no app-code changes.

**Why this matters as a reference.** Local-first by default *with* opt-in cloud sync is the best of both: privacy-preserving for casual users, server-DB upgrade path when teams scale. Very few chat harnesses support this — most assume server storage. For privacy-sensitive deployments (healthcare, legal, regulated industries) this is the difference between deployable and not.

**Cross-link.** [125-system-level-production-patterns](125-system-level-production-patterns.md), [122-explainability-compliance](122-explainability-compliance.md), [147-vendor-lock-in](147-vendor-lock-in.md).

### 2.5 Multi-modal voice + vision surface

**TTS/STT.** [@lobehub/tts](https://www.npmjs.com/package/@lobehub/tts) v5.1.2 (Mar 2 2026). Server + browser, supports OpenAI / EdgeSpeech / Microsoft / others. React Hooks + audio components.

**Vision.** Image upload with VLM-aware models (Claude/GPT-4o/Gemini/Qwen3-VL/GLM-4.5V). Screen-share is on the roadmap.

**Generative UI / Artifacts.** A `<lobe-artifact>` custom element any agent can emit; SVG, HTML, React-in-iframe, mermaid, code, markdown documents.

**Why this matters as a reference.** The voice/vision/artifact stack is **separable** from the chat app — `@lobehub/tts` is published independently. You can lift the voice stack into your own harness without adopting LobeHub.

**Cross-link.** [137-voice-agents](137-voice-agents.md), [48-voiceagentrag-dual-agent](48-voiceagentrag-dual-agent.md), [136-multimodal-rag](136-multimodal-rag.md), [105-agentic-world-modeling-visual-generation](105-agentic-world-modeling-visual-generation.md).

## §3 — Architecture quickref (refreshed)

```
┌─────────────────────────────────────────────────────────┐
│ Next.js App Router (chat, admin, marketplace, pages)     │
├─────────────────────────────────────────────────────────┤
│ agent-runtime  │  model-runtime  │  plugin gateway       │
│ (orchestration) │ (provider ABI) │ (MCP + legacy)        │
├─────────────────────────────────────────────────────────┤
│ model-bank (50+ provider metadata)  │  Agent Gateway      │
│                                      │  (WebSocket)       │
├─────────────────────────────────────────────────────────┤
│ Postgres+Drizzle ORM   |   IndexedDB+Dexie (client mode)  │
│ Redis (cache)          |   S3 (blob storage)              │
│ Upstash workflow (memory extractor)                       │
└─────────────────────────────────────────────────────────┘
```

The architecture is a **shared-database modular monolith** — not a microservice fan-out. The pnpm workspace splits agent-runtime (orchestration), model-runtime (provider adapters), and model-bank (static metadata). Adding a new model provider is a model-runtime contribution; adding a new agent is a `lobe-chat-agents/index.json` PR.

## §4 — Empirical adoption signals (May 2026)

- 76.5k stars on the umbrella; 69.3k on lobe-chat. Top-3 OSS chat harness.
- 10,363 commits on canary — actively developed.
- 24M+ container pulls aggregated across LobeHub Docker images.
- 505+ community-contributed agents in the Agent Market.
- 169,739 skills indexed (10,000+ MCP-compatible).
- One-click install completes in median <10s for hosted MCP servers.
- Adopted as deployment target by Sealos, Zeabur, RepoCloud — three OSS PaaS layers.

## §5 — Failure modes and limitations

- **License risk.** The LobeHub Community License's commercial-distribution clause limits white-label resale. Self-hosting for your own org is fine; reselling a fork is not. Track the [v1 license update](https://lobehub.com/blog/lobe-chat-v1-license-update) for any movement back to plain Apache 2.0.
- **Tracing/observability is thin.** No built-in OpenTelemetry, no per-trajectory logs, no eval harness. Operators rely on provider-side logs. **First-party gap** for any production deployment.
- **No prompt-injection defence.** A real issue for Agent Groups where one agent can feed another a poisoned document. See [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [82-poisonedrag](82-poisonedrag.md).
- **Coarse permission model.** Per-plugin scopes exist; no hook-style intercept point. Cf. [05-hooks](05-hooks.md), [06-permission-modes](06-permission-modes.md) — those primitives are stronger than LobeHub's.
- **Trust verdicts are reputation-only.** No Argus-style trust-tier verification, no retraction-aware gates, no two-axis CiteGuard / Contradiction-to-Consensus checks. See [172-polaris-2026-deep-research-roadmap](172-polaris-2026-deep-research-roadmap.md) §3 Gap 5.
- **Memory injection budget is heuristic.** ICPEA's typed layers are clean, but per-layer budget allocation in real workloads is still hand-tuned.
- **Async extractor reliability.** When the extractor fails (rate-limit, model error), the agent silently misses the new memory. Needs a retry-and-alarm path that today is operator-managed.
- **Agent Gateway scaling unproven.** WebSocket fan-out for many concurrent Group sessions stresses the gateway; reference deployments are still small (<100 concurrent groups).
- **Branching conversations don't replay sub-agent state.** Forking a branch forks the chat but not the agent's memory state at that turn — a subtle semantic gap when the user expects to "rewind."

## §6 — When to use LobeHub, when to lift just the patterns

**Use LobeHub directly** when (a) you need a polished consumer-facing chat UI on day one, (b) you don't have license restrictions on the LobeHub Community License, (c) your in-tree project's value is the *backend* (a research agent, a deep-research agent, a domain expert) and you want LobeHub as the user shell, (d) you can accept the thin observability / permission / guardrail surface and layer your own checks on top.

**Lift the patterns instead** when (a) you have license restrictions, (b) your harness needs first-class observability / permissions / guardrails (cf. [05-hooks](05-hooks.md), [06-permission-modes](06-permission-modes.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md)), (c) you want a different visual identity, (d) you need the patterns inside a non-Next.js stack.

The two approaches are not exclusive — a common pattern is "LobeHub as the consumer shell + in-tree backend exposed as MCP servers."

## §7 — Implications for harness engineering

- **Adopt ICPEA verbatim.** The five-layer typed memory schema (Identity / Context / Preference / Experience / Activity) plus async extractor is the cleanest open spec for personal memory. Bake it into [09-memory-files](09-memory-files.md) defaults.
- **Add per-agent layer access control.** When mounting personal memory, every agent declares which layers it may read. This is the access-control story that prompt-stuffing-with-everything misses. Cf. [04-skills](04-skills.md).
- **Async memory extractor.** Decouple memory write quality from chat-path latency. Cf. [10-multi-session-continuity](10-multi-session-continuity.md), [184-strongest-memory-techniques-synthesis-may-2026](184-strongest-memory-techniques-synthesis-may-2026.md).
- **Branching chat as universal UX.** Any harness exposing trajectories should let the user fork. Branch = retry the user can compare. Cf. [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md).
- **Pages as collaborative-document primitive.** Co-authored markdown with timeline-versioning is the right shape for research write-ups, runbooks, and deliverables. Cf. [12-todo-scratchpad-state](12-todo-scratchpad-state.md).
- **MCP-first skill catalog.** One-click install + permission negotiation at install is now table stakes for any agent platform. Cf. [07-model-context-protocol](07-model-context-protocol.md), [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md).
- **Agent-as-JSON publishing pipeline.** PR to `index.json` → CI lint → marketplace deploy. Zero code; schema does the work. Cf. [02-subagent-delegation](02-subagent-delegation.md).
- **Client-side IndexedDB mode.** Privacy-by-default for sensitive deployments. Cf. [122-explainability-compliance](122-explainability-compliance.md), [147-vendor-lock-in](147-vendor-lock-in.md).
- **Server-resident agents over WebSocket.** The Agent Gateway pattern scales multi-agent sessions without per-tab workers. Generalisable to any shared-state agent. Cf. [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md).
- **Pure-function agents underneath.** LobeHub doesn't enforce this; in-tree projects should layer Yenugula's pure-function-agent pattern ([202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md) §4) on top for replayability.
- **Voice + vision as separable libraries.** `@lobehub/tts` is published independently; lift it without adopting the chat app. Cf. [137-voice-agents](137-voice-agents.md).
- **Add a trust gate over the marketplace.** LobeHub trusts community contributions; in-tree projects should layer Argus-style trust verdicts before install. Cf. [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md).
- **Add observability and per-trajectory logs.** Don't deploy LobeHub-as-shell without an external trace aggregator. Cf. [24-observability-tracing](24-observability-tracing.md).
- **Marketplace as contributor onboarding, not just discovery.** The PR-to-`index.json` shape is the cheapest way to grow a skill ecosystem. Cf. [177-skills-discovery-curator-strongest-2026-techniques](177-skills-discovery-curator-strongest-2026-techniques.md).

**The one-line takeaway for harness designers:** Treat LobeHub as the **reference design** for collaborative-teammate UX — adopt ICPEA + branching + Pages + MCP-first marketplace verbatim, and decide deployment-by-deployment whether to use the LobeHub shell or just the patterns.

## §8 — Cross-references

- [64-lobehub-ai-framework](64-lobehub-ai-framework.md) — April-2026 deep-dive (architecture, plugin/agent extension, license discussion).
- [09-memory-files](09-memory-files.md), [184-strongest-memory-techniques-synthesis-may-2026](184-strongest-memory-techniques-synthesis-may-2026.md), [185-memory-integration-playbook](185-memory-integration-playbook.md), [186-mnema-witness-lattice](186-mnema-witness-lattice.md) — memory-substrate context.
- [02-subagent-delegation](02-subagent-delegation.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md), [187-multi-agent-shared-memory-landscape](187-multi-agent-shared-memory-landscape.md) — multi-agent coordination context.
- [04-skills](04-skills.md), [07-model-context-protocol](07-model-context-protocol.md), [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md), [176-skill-discovery-curator-oss-landscape-may-2026](176-skill-discovery-curator-oss-landscape-may-2026.md), [177-skills-discovery-curator-strongest-2026-techniques](177-skills-discovery-curator-strongest-2026-techniques.md) — skill ecosystem context.
- [12-todo-scratchpad-state](12-todo-scratchpad-state.md), [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md) — branching / scratchpad context.
- [137-voice-agents](137-voice-agents.md), [48-voiceagentrag-dual-agent](48-voiceagentrag-dual-agent.md) — voice surface.
- [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md), [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md) — adjacent and downstream files.
