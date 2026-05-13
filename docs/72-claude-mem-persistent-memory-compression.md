# 72 — thedotmack/claude-mem: Persistent Memory Compression for Claude Code via Progressive-Disclosure Hooks

**Definition.** `thedotmack/claude-mem` ([github.com/thedotmack/claude-mem](https://github.com/thedotmack/claude-mem)) is a **persistent memory compression system** for Claude Code (and Gemini CLI, OpenCode, OpenClaw) that automatically captures everything an agent does during a session, AI-compresses observations into semantic summaries, and injects relevant context back into future sessions — producing *durable cross-session memory* without manual intervention. Built by Alex Newman ([@thedotmack](https://github.com/thedotmack)), the project has **65,040 stars** (April 2026) and is deployed as a Claude Code plugin via `npx claude-mem install`. Architecture consists of **5 lifecycle hooks**, a **Worker Service HTTP API on port 37777** (with 10 search endpoints + web viewer UI), a **SQLite database** for sessions/observations/summaries, a **Chroma vector database** for hybrid semantic + keyword search, and **4 MCP tools** exposing a 3-layer progressive-disclosure retrieval pattern that yields ~10× token savings versus naive dump-everything injection. The project is licensed AGPL-3.0 (with a separate Ragtime subcomponent under PolyForm Noncommercial).

This is the most sophisticated *production* persistent-memory artifact for Claude Code in the community as of April 2026 and sits at the center of the "harness memory layer" conversation that this corpus treats as one of the six coding-agent primitives ([46-components-of-coding-agent.md](46-components-of-coding-agent.md)). Where [55-hermes-agent-self-improving.md](55-hermes-agent-self-improving.md) describes Nous Research's hosted persistent-memory agent, claude-mem is its *open-source, self-hosted, plugin-form* analog — running locally on the developer's machine, integrated into the Claude Code harness, and entirely under user control.

## Problem it solves — the session-boundary context problem

Claude Code sessions begin empty. Every new session, the agent has no memory of:
- Architecture decisions made in prior sessions
- Bugs already investigated
- Code style conventions adopted during project history
- Failed approaches that should not be re-tried
- File layouts, custom tooling, library versions
- The "why" behind design choices

The default remediation is to stuff a CLAUDE.md with everything important. This works up to a point, but CLAUDE.md has three hard limits:

1. **Write discipline.** CLAUDE.md has to be manually updated. Developers rarely update it in real-time during a session, so it always lags the current state of the project.
2. **Growth ceiling.** CLAUDE.md is injected on every prompt. If it grows past a few thousand tokens, it dominates the context window and costs compound on every LLM call.
3. **No recency / relevance weighting.** CLAUDE.md is a flat document — all content is presented as equally relevant to every query. A question about auth doesn't need the deployment config section, but CLAUDE.md gives you both.

Claude-mem addresses all three limits simultaneously by (a) auto-capturing observations from tool use without user intervention, (b) compressing them to semantic summaries rather than raw transcripts, (c) making them retrievable by semantic search rather than always-injected. The net effect: **the agent gets *more* relevant context while consuming *less* token budget**.

## Architecture — the six components

### 1. Five Lifecycle Hooks (6 scripts)

Claude-mem hooks into Claude Code's official lifecycle events to capture and inject at the right moments:

| Hook | Purpose |
|---|---|
| **SessionStart** | Load and inject relevant context from prior sessions based on the starting project. |
| **UserPromptSubmit** | Optionally enrich the prompt with context matching the user's query. |
| **PostToolUse** | Capture the tool-use observation (which tool, what inputs, what results) into the observations log. |
| **Stop** | Trigger AI-compression of recent observations into semantic summaries. |
| **SessionEnd** | Finalize session, persist all observations and summaries. |

A sixth script is a **pre-hook "Smart Install"** — a cached dependency checker that runs before lifecycle hooks to ensure Bun, Chroma, SQLite, and uv are all available.

This hook-based architecture is the standard Claude Code extension pattern ([62-everything-claude-code.md](62-everything-claude-code.md) documents the same hook surface). Claude-mem's insight is that those hooks are *sufficient* for a full memory system — you don't need to modify the Claude Code binary, just register handlers at the right lifecycle points.

### 2. Worker Service (HTTP API on port 37777)

A long-lived Bun-managed worker process that:
- Hosts **10 search endpoints** exposing the SQLite/Chroma query surface.
- Serves a **web viewer UI** at `http://localhost:37777` for browsing sessions, observations, summaries, and searching memory interactively.
- Provides **API access to individual observations** via `http://localhost:37777/api/observation/{id}` — which Claude uses to render citations to prior work.
- Runs in the background across Claude Code sessions, so the memory database is always available.

The choice of Bun over Node.js is worth noting — Bun's faster startup and better SQLite bindings reduce the overhead of spawning query subprocesses, which matters when the agent's search latency is in the per-second hot path.

### 3. SQLite Database

Stores three core entities:
- **Sessions** — one per Claude Code session, with start/end timestamps, project path, model used.
- **Observations** — one per captured tool use: what tool, what parameters, what output, what project, what time. These are the raw memory traces.
- **Summaries** — AI-compressed semantic summaries over groups of observations. These are the "compressed memory" layer.

Full-text search is handled by **FTS5** (SQLite's built-in full-text search extension). This gives fast keyword-based retrieval for when the user's query matches literal content.

### 4. Chroma Vector Database

Stores **semantic embeddings** of summaries and observations, enabling:
- **Hybrid semantic + keyword search.** When the query is "authentication bug", FTS5 catches literal matches; Chroma catches semantic neighbors ("login 401", "token refresh failure", "session expired").
- **Cross-project semantic retrieval.** Even when a past session used different terminology, the embedding match can still surface it.

The hybrid pattern — SQLite FTS5 for exact/structured queries, Chroma for semantic fuzz — is the standard 2026 retrieval architecture. Claude-mem's contribution is integrating both under a single query surface the agent accesses via MCP.

### 5. mem-search Skill (Claude Skills integration)

A native Claude Skill (`SKILL.md`-style, [04-skills.md](04-skills.md)) that lets the agent:
- Accept natural-language memory queries.
- Walk the 3-layer progressive-disclosure retrieval pattern (below).
- Return relevant summaries and observations with citation IDs.

The skill is invoked via standard Claude Code skill-triggering — the user asks a memory question in plain language and Claude routes to the skill.

### 6. MCP Tools — the 3-Layer Progressive-Disclosure Workflow

The four MCP tools expose the memory store to any agent that speaks MCP (not just Claude Code — Gemini CLI, OpenCode, OpenClaw too):

| Tool | Cost per result | Purpose |
|---|---|---|
| **`search`** | ~50-100 tokens/result | Get a compact index of hits: ID, type, project, date, one-line summary. |
| **`timeline`** | ~200-400 tokens/result | Get chronological context *around* a specific observation or query. |
| **`get_observations`** | ~500-1,000 tokens/result | Fetch **full details** for specific IDs. Called only after `search` filters. |

**The 3-layer workflow in practice:**

```
Step 1 (wide, cheap):    search(query="auth bug", type="bugfix", limit=10)
                         → returns compact index, ~500-1000 tokens total

Step 2 (analyze):        agent reads index, identifies IDs #123 and #456 as relevant

Step 3 (narrow, rich):   get_observations(ids=[123, 456])
                         → returns full details only for the 2 relevant items, ~1000-2000 tokens
```

**Total cost: ~1500-3000 tokens** vs. a naive dump of all 10 matches at full detail (~5000-10000 tokens).

**~10× token savings** is the explicit claim in the documentation. The savings compound across sessions because (a) most memory queries are answered by the search index alone, and (b) when full fetches are needed, they are filtered to the handful that actually matter.

This is the **progressive disclosure pattern** applied to retrieval — philosophically identical to [17-progressive-disclosure-context-lifecycle.md](17-progressive-disclosure-context-lifecycle.md) (which describes progressive disclosure applied to context management). Claude-mem exports the pattern as a named retrieval protocol.

## Unique architectural properties

### A. Session-spanning daemonization

Unlike simple memory systems that store to a flat file and re-read on every session, claude-mem runs a **persistent Bun worker** that stays alive across sessions. This gives:
- Sub-millisecond query latency (no JVM-style cold start).
- Shared cache across concurrent agents/sessions.
- Web viewer available even when no Claude Code session is active — the developer can browse memory at `http://localhost:37777` anytime.
- Real-time streaming capability (observations published to Telegram, Discord, Slack via the OpenClaw integration).

### B. `<private>` tags for privacy control

Users can wrap sensitive content in `<private>` tags within prompts or files — claude-mem excludes tagged content from the observation log. This addresses one of the hardest issues in auto-capture memory: *what should NOT be remembered*. By giving users a simple tag-based opt-out, the system preserves user control over what enters long-term storage without requiring complex per-file ACLs.

### C. Multi-IDE delivery via a single installer

```
npx claude-mem install                         # Claude Code (default)
npx claude-mem install --ide gemini-cli        # Gemini CLI
npx claude-mem install --ide opencode          # OpenCode
curl -fsSL https://install.cmem.ai/openclaw.sh | bash   # OpenClaw Gateway
```

Same codebase, four IDEs. This is the multi-surface delivery pattern ([71-karpathy-skills-single-file-guardrails.md](71-karpathy-skills-single-file-guardrails.md) discusses the same pattern for CLAUDE.md principles). The cost is a dispatch layer in the installer that resolves the target IDE's hook/plugin directory and writes the appropriate integration files.

### D. Citations via observation IDs

When Claude references prior work in a conversation, it can cite the observation ID (e.g., "as we saw in observation #123"). The user can then:
- Click through to the web viewer to see the full observation.
- Hit `http://localhost:37777/api/observation/123` programmatically.
- Share the observation with a teammate by URL.

This elevates memory from "the agent vaguely remembers" to "the agent cites specific artifacts you can audit." Directly supports [21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md) and [24-observability-tracing.md](24-observability-tracing.md) by giving memory the same "click to audit" affordance that traces and evaluations have.

### E. Mode & Language configuration

```json
{ "CLAUDE_MEM_MODE": "code--zh" }
```

The `CLAUDE_MEM_MODE` setting controls both:
- **Workflow behavior** (e.g., `code` for default engineering mode, `chill` for more casual exploration, `investigation` for debugging-focused capture).
- **Summary language** — `code--zh` for Simplified Chinese summaries, `code--ja` for Japanese, etc.

This acknowledges that memory is culturally and contextually situated. A team working in Chinese benefits from summaries in Chinese even when the code is English. Unusual for Western open-source tools; suggests the author is actively targeting a non-English developer community (the README is translated into 30+ languages).

### F. Beta channel with Endless Mode

The **beta channel** ships experimental features including **Endless Mode** — described as a "biomimetic memory architecture for extended sessions." While details are gated behind the docs link (and thus not fully described in the README), the name and context suggest:
- Extended context handling that mimics biological memory's consolidation / forgetting patterns.
- Likely includes LRU-like eviction, temporal decay of observation weights, consolidation into longer-term summaries.

Users opt-in via web viewer UI → Settings. This is the **feature flag / opt-in beta** pattern for persistent-memory systems — safe experimentation without breaking stable users' sessions.

## Comparison with other persistent-memory systems in the corpus

| System | File | Deployment | Storage | Retrieval | Scale |
|---|---|---|---|---|---|
| **CLAUDE.md** | [15](15-claude-md-canonical-memory.md) | Flat file in project | Markdown | Always-inject | Tiny (few KB) |
| **Claude Skills** | [04](04-skills.md) | `/skills/*/SKILL.md` | Per-skill MD + scripts | Triggered by description match | Small (<100 skills typical) |
| **Memory Files pattern** | [09](09-memory-files.md) | Flat JSON/MD in repo | Structured files | Keyword | Small |
| **Hermes Agent** | [55](55-hermes-agent-self-improving.md) | Hosted SaaS | Vector DB + proprietary | Semantic | Large, opaque |
| **Voyager skill library** | [19](19-voyager-skill-libraries.md) | Embedded in policy | Code library | Semantic over skill descriptions | Medium, research-scale |
| **claude-mem** | (this file) | Local daemon + plugin | SQLite + Chroma + web viewer | Hybrid semantic + FTS5, 3-layer MCP | Unbounded, production-ready |

Claude-mem's distinguishing profile:
- **Open-source and self-hosted** (vs Hermes' SaaS-only).
- **Local-first, zero cloud dependency** — all data stays on the developer's machine.
- **Explicit token-efficiency focus** (the 3-layer progressive-disclosure pattern is marketed as 10× savings).
- **Multi-IDE support** (Claude Code, Gemini CLI, OpenCode, OpenClaw).
- **MCP-native** — any agent can use it, not just Claude Code.

## Token economics — why progressive disclosure matters

Claude-mem's 10× token savings claim deserves examination. Consider a developer who queries memory 50 times per week about their project:

**Naive approach (dump matched observations at full detail):**
- 50 queries × 10 matches × 800 tokens/match = **400,000 tokens/week**

**Progressive disclosure approach:**
- 50 queries × 10 indexes × 75 tokens/index = 37,500 tokens for indexes
- 50 queries × 2 fetches × 750 tokens/fetch = 75,000 tokens for full details
- Total: **112,500 tokens/week**

**Reduction: 3.5×** (not quite the marketed 10× but directionally correct)

The 10× figure likely assumes the user's top-k is higher (20-30 matches per query) and the fetch-rate is lower (1 per query instead of 2). At production-scale usage with 100+ queries/week and large match pools, the savings compound further. At 2026 Claude 4 pricing (~$3 per million input tokens), a single developer's weekly memory usage costs **~$0.34 vs $1.20** — seemingly small, but:

- Multiplied across a 50-person team: $17 vs $60 per week, **$2,250+ per year just on memory context**.
- At enterprise scale (1000+ developers): **$45K+ per year**.
- At orchestration scale (agents calling agents calling agents): the amplification can hit 10-50×, turning a $45K/year cost into a $500K+ bottleneck.

Progressive disclosure is not a micro-optimization — it is the pattern that makes persistent memory *economically viable* at scale.

## Failure modes

1. **Daemon-dependency fragility.** The worker service has to stay alive. If Bun crashes, or the user's machine restarts without auto-restart configured, memory retrieval silently fails and the agent behaves as if no memory exists. The README does not describe auto-restart / health-check logic clearly.

2. **Observation pollution.** Auto-capture catches everything — including off-task exploration, dead-end investigations, false starts. Without aggressive summarization or pruning, the observation log grows monotonically and retrieval quality degrades over time. Endless Mode (beta) likely addresses this, but stable users bear the cost.

3. **Semantic drift across compression cycles.** Each compression cycle (observations → summaries) loses information. Summaries get summarized into higher-level summaries. Over months, the "remembered" state can diverge meaningfully from the actual historical state. The author clearly understands this (Endless Mode's biomimetic framing) but the stable release currently relies on simpler compression.

4. **Cross-session contamination.** Memory is keyed by project path. If a developer works on multiple projects under the same path (or uses workspaces, submodules, monorepos), observations from one sub-project may surface as context for another. The README does not clarify project-boundary enforcement.

5. **Privacy boundary depends on user discipline.** The `<private>` tag works only if the user remembers to use it. Sensitive content typed without the tag (API keys in a `.env` file viewed during a session, customer PII during a debugging session) enters the permanent observation log. For enterprise deployment this would require additional institutional controls.

6. **AGPL-3.0 network-server requirement.** If an organization deploys claude-mem behind a network service exposed to users, AGPL-3.0 requires making the source code available to those users. This likely blocks some commercial adoptions that would otherwise be interested. Most individual developers don't hit this.

7. **$CMEM token association.** The project README notes that a Solana token ($CMEM) was created "by a 3rd party without prior consent" but was subsequently "officially embraced." For enterprise buyers, the crypto-token association adds non-zero legal / reputational / compliance friction — some corporate IT policies explicitly prohibit crypto-adjacent dependencies.

## Patterns for harness engineers to steal

1. **Progressive-disclosure retrieval is the dominant pattern for production memory.** Three layers (index → context → detail) with escalating token cost and escalating relevance. If your system exposes memory to agents, structure the API this way — not as "search returns everything."

2. **Hybrid SQLite FTS5 + Chroma vector DB** is the right default retrieval substrate. Exact/structured queries go to FTS5; semantic queries go to Chroma. The combined query cost is low and the recall is high.

3. **Lifecycle hooks over modifying the harness binary.** Claude-mem never patches Claude Code — it only registers handlers at published lifecycle events. This is sustainable: the Claude Code team can update the binary without breaking claude-mem as long as the hook contract is stable.

4. **Long-lived daemon for session-spanning state.** A background worker process is cheap and gives you free cross-session affordances (web viewer, REST API for external tools, real-time streaming). The operational cost is ~100MB RAM and negligible CPU at idle.

5. **Privacy-by-user-tag.** The `<private>` tag is simpler than any structured ACL system and shifts the burden to the place it belongs — the user who knows what is sensitive. Don't build complex ACLs; build one simple escape hatch.

6. **Citation IDs create auditability.** Tying every memory retrieval to an observation ID that's navigable via URL makes the system feel *trustworthy*. Without citations, memory feels like a black box. With citations, it feels like a searchable archive.

7. **Multi-IDE support is a 10× adoption lever.** Claude Code, Gemini CLI, OpenCode, OpenClaw — one install target becomes four. The implementation cost is small (a few per-IDE adapters); the adoption gain is large.

8. **Web viewer as the user-facing surface.** Most memory systems hide behind the agent's context. Claude-mem exposes a browsable UI at `localhost:37777`. This lets the user *learn what the agent knows* — a critical trust-building affordance that non-UI memory systems lack.

9. **Token-efficiency as a marketing hook.** "~10× token savings" is a concrete value proposition that decision-makers at companies paying per-token rates can immediately quantify. When you build a memory system, measure the token delta explicitly and lead with it.

10. **Mode-based configuration (workflow + language together).** The `code--zh` pattern is clever — one setting controls two orthogonal dimensions. For internationalization, bundle workflow mode and language to avoid making users configure two independent settings.

## Relationship to the corpus

Claude-mem sits at the intersection of several threads:

- **Memory layer of the six-component model** ([46-components-of-coding-agent.md](46-components-of-coding-agent.md)). This is the most fully-realized open-source implementation of the memory component for Claude Code.
- **Successor to the naive CLAUDE.md approach** ([15-claude-md-canonical-memory.md](15-claude-md-canonical-memory.md)). Where CLAUDE.md gives you static memory, claude-mem gives you dynamic memory. They are complementary — CLAUDE.md for stable facts, claude-mem for session history.
- **Production-ready counterpart to Hermes Agent** ([55-hermes-agent-self-improving.md](55-hermes-agent-self-improving.md)). Hermes is the hosted version; claude-mem is the local version. Hermes optimizes for seamless experience; claude-mem optimizes for user control.
- **Progressive disclosure pattern in retrieval** ([17-progressive-disclosure-context-lifecycle.md](17-progressive-disclosure-context-lifecycle.md)). Claude-mem's 3-layer workflow is the cleanest implementation of progressive disclosure in the corpus.
- **MCP tooling extension** ([07-model-context-protocol.md](07-model-context-protocol.md)). Four MCP tools (`search`, `timeline`, `get_observations`, plus one more) give claude-mem a language-agnostic API surface.
- **Community Claude Code ecosystem** ([62-everything-claude-code.md](62-everything-claude-code.md)). Claude-mem is a crown jewel of the community ecosystem by star count and architectural depth.
- **Observability and trace infrastructure** ([24-observability-tracing.md](24-observability-tracing.md)). The web viewer is essentially a trace viewer with memory-specific affordances — observation tables, session timelines, search-by-query.
- **SKILL.md authoring** ([54-skill-md-authoring-guide.md](54-skill-md-authoring-guide.md)). The `mem-search` skill is an example of a production-quality skill from the community.

## References

- **Main repo.** [github.com/thedotmack/claude-mem](https://github.com/thedotmack/claude-mem) — 65,040 stars, AGPL-3.0.
- **Docs.** [docs.claude-mem.ai](https://docs.claude-mem.ai/) — installation, usage, architecture, configuration, troubleshooting.
- **Architecture overview.** [docs.claude-mem.ai/architecture/overview](https://docs.claude-mem.ai/architecture/overview).
- **Architecture evolution** (v3 to v5). [docs.claude-mem.ai/architecture-evolution](https://docs.claude-mem.ai/architecture-evolution) — useful for understanding the design trajectory.
- **Hooks reference.** [docs.claude-mem.ai/architecture/hooks](https://docs.claude-mem.ai/architecture/hooks) — 7 hook scripts explained.
- **Search architecture.** [docs.claude-mem.ai/architecture/search-architecture](https://docs.claude-mem.ai/architecture/search-architecture) — hybrid search details.
- **Worker service.** [docs.claude-mem.ai/architecture/worker-service](https://docs.claude-mem.ai/architecture/worker-service) — HTTP API / Bun management.
- **Progressive disclosure.** [docs.claude-mem.ai/progressive-disclosure](https://docs.claude-mem.ai/progressive-disclosure) — philosophy.
- **Context engineering.** [docs.claude-mem.ai/context-engineering](https://docs.claude-mem.ai/context-engineering) — general principles.
- **Beta features / Endless Mode.** [docs.claude-mem.ai/beta-features](https://docs.claude-mem.ai/beta-features).
- **OpenClaw integration.** [docs.claude-mem.ai/openclaw-integration](https://docs.claude-mem.ai/openclaw-integration).
- **Author.** Alex Newman ([@thedotmack](https://github.com/thedotmack)).
- **Official X.** [@Claude_Memory](https://x.com/Claude_Memory).
- **Discord.** [discord.com/invite/J4wttp9vDu](https://discord.com/invite/J4wttp9vDu).

## Cross-references in this corpus

- [04-skills.md](04-skills.md) — Claude Skills system underlying `mem-search`.
- [07-model-context-protocol.md](07-model-context-protocol.md) — MCP that exposes the memory tools.
- [09-memory-files.md](09-memory-files.md) — the flat-file pattern claude-mem supersedes.
- [15-claude-md-canonical-memory.md](15-claude-md-canonical-memory.md) — CLAUDE.md as a complementary static-memory surface.
- [17-progressive-disclosure-context-lifecycle.md](17-progressive-disclosure-context-lifecycle.md) — progressive disclosure pattern.
- [19-voyager-skill-libraries.md](19-voyager-skill-libraries.md) — skill-library counterpart to memory library.
- [21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md) — citation-based auditability.
- [24-observability-tracing.md](24-observability-tracing.md) — trace-viewer parallel to the memory web viewer.
- [46-components-of-coding-agent.md](46-components-of-coding-agent.md) — the memory component being implemented.
- [54-skill-md-authoring-guide.md](54-skill-md-authoring-guide.md) — `mem-search` skill as a canonical example.
- [55-hermes-agent-self-improving.md](55-hermes-agent-self-improving.md) — hosted counterpart.
- [62-everything-claude-code.md](62-everything-claude-code.md) — broader Claude Code ecosystem.
- [66-meta-harness-landscape.md](66-meta-harness-landscape.md) — memory as a meta-harness concern.
- [70-voltagent-awesome-ai-agent-papers.md](70-voltagent-awesome-ai-agent-papers.md) — Memory & RAG research bucket informing claude-mem's design space.
- [71-karpathy-skills-single-file-guardrails.md](71-karpathy-skills-single-file-guardrails.md) — sibling community Claude Code artifact.
