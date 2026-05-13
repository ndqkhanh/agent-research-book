# 176 — Open-Source Skills on the Internet: The Full Landscape (May 2026)

**Scope.** A complete survey of where **open-source skills actually live on the public internet** in May 2026. Two layers, intentionally separated:

* **Layer A — Runtimes & frameworks** (the *engines* that load skills): Anthropic Claude Code, OpenAI Codex, Gemini CLI, smolagents, LlamaIndex routing primitives, etc. *(Compressed; this is the smaller layer.)*
* **Layer B — Skill content sources** (the *catalogs and marketplaces* where actual published skills live): the Agent Skills open standard (agentskills.io), community awesome-lists, single-author production libraries, GitHub-hosted catalogs, MCP server registries (Smithery / Glama / Official MCP Registry / MCPfinder), Hugging Face Spaces, marketplaces (SkillsMP / ClaudeSkills.info). *(The bulk of this doc.)*

**One-paragraph thesis.** The Agent Skills open standard at **[agentskills.io](https://agentskills.io/home)** has been adopted by **26+ platforms** (Claude, Claude Code, OpenAI Codex, Gemini CLI, GitHub Copilot, VS Code, Cursor, Roo Code, Amp, Goose, Mistral AI, Databricks, Antigravity, Hugging Face, etc.) — meaning a single SKILL.md folder format now portably loads across the entire ecosystem. Open-source skill *content* on the internet is concentrated in five distinct surfaces: (1) **the standard's reference catalog at `anthropics/skills`** (130k★, the de-facto canonical bundles); (2) **community awesome-lists** (`VoltAgent/awesome-agent-skills`, `ComposioHQ/awesome-claude-skills`, `sickn33/antigravity-awesome-skills` — 1,000–1,400 skills each); (3) **production libraries from individual maintainers** (`wshobson/agents` — 185 agents + 153 skills + 100 commands across 80 plugins); (4) **MCP server registries** (Smithery, Glama with **21,586 servers**, Official Registry, MCPfinder indexing 25K+ across all three); (5) **dedicated marketplaces** (ClaudeSkills.info with ~140 free skills, SkillsMP with 1.2M+ claimed). The total observable open-source skill population now numbers in the *tens of thousands of MCP servers + thousands of SKILL.md files + tens of thousands of tool-catalog APIs*. Discovery — not authoring — is the bottleneck. Which is the load-bearing question that [docs/179](179-skill-retrieval-routing-and-activation.md) takes over: with this much skill content available, how does the agent know which to load?

---

# Layer A — Runtimes & frameworks

The frameworks that load skills. Compressed; full repo summaries live in the post-`docs/164` canon and in [docs/172](172-polaris-2026-deep-research-roadmap.md).

## A.1 — Skill-native runtimes

| Runtime | Stars | Skill abstraction | Activation |
|---|---:|---|---|
| **Claude Code** ([code.claude.com](https://code.claude.com)) | shipped product | SKILL.md folder + frontmatter (`name`, `description`, `when_to_use`, `allowed-tools`, `disable-model-invocation`, `paths`) | **Progressive disclosure** — name+description in system prompt; full body on demand; supplementary files when referenced |
| **OpenAI Codex** ([developers.openai.com/codex/skills](https://developers.openai.com/codex/skills)) | shipped product | Agent Skills format (cross-compatible with Anthropic's) | Same progressive-disclosure pattern |
| **Gemini CLI** | shipped product | Agent Skills format | Cross-compatible |
| **VS Code Copilot** | shipped product | Agent Skills format ([code.visualstudio.com/docs/copilot/customization/agent-skills](https://code.visualstudio.com/docs/copilot/customization/agent-skills)) | Cross-compatible |
| **Cursor / Roo Code / Amp / Goose** | shipped products | Agent Skills format | Cross-compatible |
| **Mistral AI / Databricks** | shipped products | Agent Skills format | Cross-compatible |

The headline finding for this layer: **the format question has been settled**. SKILL.md folder shape is the open standard; 26+ platforms read it. A skill authored once works across the entire wave.

## A.2 — Skill-aware libraries

| Library | Stars | Role |
|---|---:|---|
| [`huggingface/smolagents`](https://github.com/huggingface/smolagents) | 27.1k | CodeAgent — code-as-tool-call dispatch; sandboxes (E2B / Modal / Docker / WASM); Hub-pull for community tools |
| [`stanfordnlp/dspy`](https://github.com/stanfordnlp/dspy) | 34.3k | Programmatic skill compilation via teleprompt optimizer; modules with typed signatures |
| [`run-llama/llama_index`](https://github.com/run-llama/llama_index) | 49.2k | `RouterQueryEngine`, `ToolRetrieverRouterQueryEngine`, `ObjectIndex` — embedding-then-LLM-rerank routing primitives at scale |
| [`crewAIInc/crewAI`](https://github.com/crewAIInc/crewAI) | large | Crews + Flows; explicit skills marketplace pattern (`crewaiinc/skills`) |
| [`microsoft/autogen`](https://github.com/microsoft/autogen) | 57.8k *(maintenance mode)* | Agent + tool registration; AgentTool wrapper; MCP via McpWorkbench |
| [`microsoft/agent-framework`](https://github.com/microsoft/agent-framework) | newer | AutoGen successor |

## A.3 — Open-ended discovery / lifelong-learning systems

| System | Stars | Domain | Skill primitive |
|---|---:|---|---|
| [`MineDojo/Voyager`](https://github.com/MineDojo/Voyager) | 6.9k | Minecraft | Lifelong skill library + iterative prompting |
| [`eureka-research/Eureka`](https://github.com/eureka-research/Eureka) | 3.2k | RL (IsaacGym, DexterousHands) | LLM-as-reward-designer |
| [`BAAI-Agents/Cradle`](https://github.com/BAAI-Agents/Cradle) | 2.5k | General Computer Control (RDR2, Office, Chrome) | Atomic + composite skills via screenshots |
| [`SakanaAI/AI-Scientist-v2`](https://github.com/SakanaAI/AI-Scientist-v2) | 6.1k | ML research | Tree-search experiment manager |
| [`PeterGriffinJin/Search-R1`](https://github.com/PeterGriffinJin/Search-R1) | 4.6k | Search/retrieval | RL-trained search-using LLM |

For the full architectural treatment of any of these, read [docs/172](172-polaris-2026-deep-research-roadmap.md) and the post-`docs/164` canon.

---

# Layer B — Skill content sources (where the actual skills live)

This is the layer the user originally asked about: not "what loads skills?" but "what *open-source skills* exist on the internet?". Five surfaces.

## B.1 — The open standard: agentskills.io

**[agentskills.io](https://agentskills.io/home)** is the canonical specification site. Originally developed by Anthropic, **released as an open standard**, and now adopted by 26+ platforms (per [the agentskills.io platform list](https://agentskills.io/home)).

### What the standard defines

A skill is a folder containing:

```
my-skill/
├── SKILL.md           # required — frontmatter + body
├── reference.md       # optional reference docs
├── examples/          # optional examples (loaded when needed)
└── scripts/           # optional executable scripts
    └── helper.py
```

`SKILL.md` carries YAML frontmatter — minimum `name` + `description`; optional `when_to_use`, `argument-hint`, `disable-model-invocation`, `user-invocable`, `allowed-tools`, `paths`, `model`, `effort`, `context`, `agent`, `hooks`, `shell`. Body is markdown instructions.

### How agents load it

Three-tier **progressive disclosure**:

1. **Discovery** — agent loads only `name` + `description` (≤1,536 chars per entry, total budget ≈1% of context window or 8,000 chars).
2. **Activation** — when a task matches a description, the full SKILL.md loads.
3. **Execution** — bundled scripts / referenced files load only when the body asks for them.

### Specification + reference repo

- [`agentskills/agentskills`](https://github.com/agentskills/agentskills) — open spec + documentation.
- [`anthropics/skills`](https://github.com/anthropics/skills) — **130k★**, mixed Apache-2.0 / source-available. Bundled categories: Creative & Design, Development & Technical, Enterprise & Communication, Document Skills (PDF, Word, Excel, PowerPoint manipulation). Available via Claude Code (plugin marketplace), Claude.ai (paid plans), Claude Skills API.

This is the **reference catalog**. Every other source-of-skills below is downstream of this format.

---

## B.2 — Awesome-lists: the community catalogs

The skill-discovery equivalent of `awesome-*` repos. These curate *thousands* of skills from the ecosystem.

### `VoltAgent/awesome-agent-skills` — 1,000+ skills

[github.com/VoltAgent/awesome-agent-skills](https://github.com/VoltAgent/awesome-agent-skills) — *"A curated collection of 1000+ agent skills from official dev teams and the community, compatible with Claude Code, Codex, Gemini CLI, Cursor, and more."* Catalogs **official skills published by leading development teams** including Anthropic, Google Labs, Vercel, Stripe, Cloudflare, Netlify, Trail of Bits, Sentry, Expo, Hugging Face, Figma, plus extensive community-built collections. Multi-runtime compatibility is explicit.

### `ComposioHQ/awesome-claude-skills` — 1,000+ skills

[github.com/ComposioHQ/awesome-claude-skills](https://github.com/ComposioHQ/awesome-claude-skills) — *"A comprehensive and curated list of 1000+ production-ready and practical Claude Skills and Plugins for enhancing productivity across use cases on not just Claude.ai, Claude Code, but also across coding agents like Codex, Cursor, Gemini CLI, Antigravity and more."* Composio's curation focus is *productivity workflows*; the list overlaps materially with VoltAgent but emphasises business-process skills.

### `sickn33/antigravity-awesome-skills` — 1,400+ skills

[github.com/sickn33/antigravity-awesome-skills](https://github.com/sickn33/antigravity-awesome-skills) — *"Installable GitHub library of 1,400+ agentic skills for Claude Code, Cursor, Codex CLI, Gemini CLI, Antigravity, and more. Includes installer CLI, bundles, workflows, and official/community skill collections."* Notable: ships an **installer CLI** so skills are not just listed but pulled and installed in one command. This is the largest published collection by raw count.

### `travisvn/awesome-claude-skills` — curated list

[github.com/travisvn/awesome-claude-skills](https://github.com/travisvn/awesome-claude-skills) — narrower, opinionated curation. Better signal-to-noise than the 1,000+ lists for some workflows.

### `alirezarezvani/claude-skills` — 232+ skills

[github.com/alirezarezvani/claude-skills](https://github.com/alirezarezvani/claude-skills) — *"232+ Claude Code skills & agent plugins for Claude Code, Codex, Gemini CLI, Cursor, and 8 more coding agents — engineering, marketing, product, compliance, C-level advisory."* Vertical-focused: not just dev skills, also marketing, product, compliance, advisory.

### `quemsah/awesome-claude-plugins`

[github.com/quemsah/awesome-claude-plugins](https://github.com/quemsah/awesome-claude-plugins) — *"Automated collection of Claude Code plugin adoption metrics across GitHub repositories using n8n workflows."* Meta-curation: tracks which plugins / skills are actually being adopted via crawled GitHub metrics. Useful as a *signal* layer over the raw lists.

### Aggregate scale

Combined, the awesome-lists expose **3,000–4,000 distinct open-source agent skills** before counting MCP servers (next surface). Note the substantial overlap: many "official" skills appear in multiple lists.

---

## B.3 — Production libraries from individual maintainers / orgs

Single-author or small-org skill bundles that ship cohesive end-to-end skill stacks.

### `wshobson/agents`

[github.com/wshobson/agents](https://github.com/wshobson/agents) — Seth Hobson's production-ready system: **185 specialized AI agents + 16 multi-agent workflow orchestrators + 153 agent skills + 100 commands across 80 focused, single-purpose plugins for Claude Code**. Each plugin loads only its specific agents, commands, and skills — progressive disclosure at the *plugin* level. Multi-agent orchestration: parallel code review, hypothesis-driven debugging, coordinated feature development. Preset teams: review, debug, feature, fullstack, research, security, migration.

The complementary [`wshobson/commands`](https://github.com/wshobson/commands) repo ships **52 slash commands** including full-stack development, incident response, ML pipelines, and security hardening workflows. Together these two repos are **the largest single-author open-source skill stack** observably maintained on GitHub today.

### Anthropic-published skills

The [`anthropics/skills`](https://github.com/anthropics/skills) repo ships official bundled categories. Adjacent Anthropic-org repos worth noting:

- [`anthropics/courses`](https://github.com/anthropics/courses) — educational skills.
- [`anthropics/anthropic-quickstarts`](https://github.com/anthropics/anthropic-quickstarts) — quickstart skills.

### Vendor-published official skills

Per VoltAgent's roster, **official skills are published by**: Anthropic, Google Labs, Vercel, Stripe, Cloudflare, Netlify, Trail of Bits, Sentry, Expo, Hugging Face, Figma, and others. Each vendor's skills typically live in their own GitHub org under a `skills/` or `agent-skills/` directory and are aggregated by the awesome-lists.

---

## B.4 — MCP server registries (skills exposed via Model Context Protocol)

MCP servers are the most numerous category of "skill" content on the open internet. Each server exposes one or more typed tools that an agent can discover and invoke.

### Glama — 21,586 servers, daily-updated

[glama.ai/mcp/servers](https://glama.ai/mcp/servers) — *"Open-Source MCP Servers — 21,586+ in the Glama Registry."* Daily-updated index; **2,223 connectors proxied through the Glama gateway** with full call logging, per-tool access control, and managed OAuth. Glama is the production-grade MCP middleware layer: not just a registry but a runtime gateway.

### Smithery — "Hugging Face for MCPs"

[smithery.ai](https://smithery.ai) is *"quickly becoming to MCPs what Hugging Face is to ML models, with developers coming here first when they're looking for a server to solve a specific problem."* Publishing flow: `smithery mcp publish` makes a server "instantly discoverable and installable by anyone running the Smithery CLI". Cursor, Claude Code, Cline, Windsurf integrate Smithery directly.

### Official MCP Registry

[registry.modelcontextprotocol.io](https://registry.modelcontextprotocol.io) — maintained by Anthropic / the MCP Steering Group. The canonical source-of-truth for protocol-conformant servers. Smaller catalog than Glama / Smithery but higher trust bar.

### MCP.so

[mcp.so](https://mcp.so) — community directory; secondary index, useful for discovery cross-checks.

### MCPfinder — meta-discovery

[github.com/mcpfinder/mcpfinder](https://github.com/mcpfinder/mcpfinder) — *"MCP server that finds MCP servers."* AI-first discovery layer indexing **25K+ MCP servers across the Official Registry, Glama, and Smithery** for any AI client (Claude, Cursor, Cline, Windsurf). Search for missing capabilities, inspect trust signals, review required secrets, generate client-specific MCP config snippets. This is the *meta-skill* — the skill whose job is finding other skills.

### Aggregate scale

**~25,000 MCP servers** are publicly indexed across the four registries (with overlap). At an average of 3–10 tools per server, this represents **75,000–250,000 discoverable open-source tools/skills** exposed through MCP.

This is by far the largest open skill population on the internet. The MCP layer dwarfs SKILL.md-format catalogs by an order of magnitude in raw count, though SKILL.md skills tend to be higher-level / more workflow-shaped.

---

## B.5 — Dedicated marketplaces

Web platforms that list, search, and serve skills as a primary product.

### ClaudeSkills.info — 140+ free skills

[claudeskills.info](https://claudeskills.info/skills/) — *"the largest Claude Skills marketplace with 140+ free, open source Claude Code skills and agent skills available to browse."* Browse / download / category-filter UI over open-source skills. Free; community-contributed; light moderation.

### SkillsMP — 1.2M+ agent skills (claimed)

[skillsmp.com](https://skillsmp.com/) — *"Agent Skills Marketplace - Claude, Codex & ChatGPT Skills."* Claims **1.2M+ agent skills with intelligent filtering by occupation, author, and popularity**. Treat the headline number with skepticism — likely includes auto-generated and community-mirrored entries — but the platform is the most aggressive at *cross-runtime* skill aggregation (Claude + Codex + ChatGPT).

### Anthropic Plugin Marketplace (Claude Code)

The Claude Code plugin marketplace exposes vendor-published and community-published skills as installable packages. Currently **no paid skills**; Anthropic has *"mentioned plans for community contributions and a potential marketplace in the future"* but the current model is open-source distribution + vendor governance.

### Aggregate ecosystem signal

Per Mar 2026 Medium reporting: *"The Claude Skills ecosystem on GitHub has exploded with 22,000+ star repos, official Anthropic releases, and communities of developers, designers, and product managers all building and sharing their own .skill files."* The growth curve is steep — most of the ecosystem is younger than 12 months.

---

## B.6 — Adjacent skill-shaped catalogs (not labelled "skills" but functionally equivalent)

Skill-shaped artifacts that predate the agentskills.io standard or live in adjacent ecosystems.

| Catalog | Scale | Skill-equivalent unit |
|---|---:|---|
| **Hugging Face Spaces** | 250K+ | Each Space ≈ a callable skill via the smolagents `import_from_hub` pattern |
| **HuggingFace Hub Tools** | 1K+ | Tool-spec format; smolagents pulls these as skills |
| **OpenBMB/ToolBench** | 16,464 APIs / 3,451 tools | API-as-skill catalog with paired retriever |
| **Gorilla APIBench** | 1,600+ APIs | Earlier-generation API-as-skill catalog |
| **LangChain Hub** | 1K+ | Prompt + agent recipes |
| **n8n / Zapier integrations** | 1K+ each | Workflow-as-skill (different ABI but conceptually similar) |
| **Replicate / fal.ai catalogs** | 10K+ models | Model-as-callable-skill |
| **Vercel AI SDK tool catalogs** | dozens | Typed tool registrations across vendors |

These don't ship `SKILL.md` files but are the *next-best skill-shaped content sources* on the open internet — and tools like smolagents and MCPfinder bridge the formats.

---

# §C — How to actually find skills (the practical view)

If a Polaris or Lyra contributor needs to *find* an existing open-source skill for a given problem, the recommended discovery path in May 2026:

1. **Start at [agentskills.io](https://agentskills.io/home)** for the standard + a vetted tutorial.
2. **Browse [`anthropics/skills`](https://github.com/anthropics/skills)** for the bundled-vendor reference catalog.
3. **Search [`VoltAgent/awesome-agent-skills`](https://github.com/VoltAgent/awesome-agent-skills) and [`ComposioHQ/awesome-claude-skills`](https://github.com/ComposioHQ/awesome-claude-skills)** for the 1,000+ community-curated lists. Cross-reference.
4. **For full installer + bundles, use [`sickn33/antigravity-awesome-skills`](https://github.com/sickn33/antigravity-awesome-skills)** (1,400+ skills with installer CLI).
5. **For workflow / agent stacks, check [`wshobson/agents`](https://github.com/wshobson/agents) + [`wshobson/commands`](https://github.com/wshobson/commands)** as the production reference.
6. **For tools / APIs, query an MCP registry**: [Glama](https://glama.ai/mcp/servers) (21,586 servers), [Smithery](https://smithery.ai), the [Official MCP Registry](https://registry.modelcontextprotocol.io). For AI-driven discovery across all three, run [MCPfinder](https://github.com/mcpfinder/mcpfinder).
7. **For dedicated marketplaces with browseable UI**, try [ClaudeSkills.info](https://claudeskills.info/skills/) (140+ free, well-curated) or [SkillsMP](https://skillsmp.com/) (1.2M+ but noisier).
8. **For adjacent catalogs**, check Hugging Face Spaces / Hub Tools, ToolBench's 16K APIs, LangChain Hub.

The discovery problem itself is fast becoming the load-bearing question — see [docs/179](179-skill-retrieval-routing-and-activation.md) for how runtimes are starting to handle "which skill should I load?" automatically (Anthropic progressive disclosure, SkillRouter at scale, embedding rerank, MCP `tools/list`).

---

# §D — Patterns to steal

Three patterns appear across the strongest content sources and should land in any harness that intends to consume the open-source skill ecosystem.

### Pattern 1 — Adopt the agentskills.io standard verbatim

Don't fork the format. With **26+ platforms** reading the same SKILL.md frontmatter, any custom format isolates you from the entire content layer described above. Polaris's `polaris-skills/` and Lyra's `lyra-skills/` should both target the same frontmatter — name, description, when_to_use, argument-hint, disable-model-invocation, user-invocable, allowed-tools, paths — so skills round-trip across harnesses *and* across the 26+ platforms.

### Pattern 2 — Treat MCP server registries as a first-class skill-content source

The 25K+ MCP servers indexed across Glama / Smithery / Official Registry / MCPfinder are not "tools we might support" — they're **the largest open-source skill population on the internet**. Polaris's `polaris-mcp` package and Lyra's `lyra-mcp` package should both ship a Smithery-CLI-equivalent and Glama-aware retrieval. Skill discovery without MCP-registry awareness misses ~75% of available content.

### Pattern 3 — The vulnerability scanner ([175](175-agent-skills-ecosystem-and-security.md)) runs on *every* content source equally

The 26.1% community-skills vulnerability finding ([docs/175](175-agent-skills-ecosystem-and-security.md)) applies **across every B-layer source above**: agentskills.io community contributions, awesome-list aggregations, MCP registries, marketplaces. Trust differs by source (Anthropic-published > vendor-published > community-published > anonymous-MCP-server) but the scanner runs equally on all of them. The four-tier framework (T-Untrusted → T-Scanned → T-Reviewed → T-Pinned) is the only sustainable governance model at the scale Layer B exposes.

---

# §E — Where this fits

- [173 — Offline-Sim Skill Discovery](173-offline-sim-skill-discovery.md), [174 — EXIF](174-autonomous-skill-exploration-iterative-feedback.md) — paper-side companions; this doc surveys the engineering surface.
- [175 — Agent Skills Ecosystem & Security](175-agent-skills-ecosystem-and-security.md) — the supply-chain argument that grounds Pattern 3 above.
- [177 — Strongest 2026 Techniques Synthesis](177-skills-discovery-curator-strongest-2026-techniques.md) — folds Layer B discovery into the recommended adoption stack.
- [178 — Online Skill Discovery & Curation On-the-Go](178-online-skill-discovery-and-curation-on-the-go.md) — the runtime/live-discovery side; this doc is the static-catalog side.
- [179 — Skill Retrieval, Routing, Activation](179-skill-retrieval-routing-and-activation.md) — *given* the catalogs above, how does the agent know which to load? The activation surface.
