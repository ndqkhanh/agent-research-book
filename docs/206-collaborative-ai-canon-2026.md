# 206 — The Collaborative-AI Canon: Teammates That Grow With the User (May 2026)

> **Disambiguation.** This file synthesises the **2024–2026 collaborative-AI canon** under the framing LobeHub uses on its README — *agent teammates that grow with you*. It treats personal memory, multi-agent collaboration, MCP-first skill catalogs, branching UX, co-evolution, and local-first privacy as **one design surface** rather than six independent topics. Companion files: [204-metagpt-foundationagents-2026-refresh](204-metagpt-foundationagents-2026-refresh.md), [205-lobehub-collaborative-teammate-platform](205-lobehub-collaborative-teammate-platform.md). The cross-project apply plan is in [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md).

## One-line definition

A **collaborative-AI agent that grows with the user** is one that combines (1) **typed personal memory** that survives sessions and projects (Letta, Mem0, LobeHub ICPEA, OpenAI/Anthropic/Google Memory), (2) **multi-agent collaboration primitives** at workflow / role / supervisor / handoff / graph-node level (AutoGen → MAF, CrewAI, MetaGPT, LangGraph, Magentic-One, deepagents, smolagents), (3) **MCP-first skill catalogs with one-click install + permission negotiation** (LobeHub, Smithery, Glama, FastMCP, Anthropic Skills), (4) **branching / Pages collaborative UX** (LobeHub, Claude Artifacts, Notion AI), (5) **co-evolution loops** (Voyager → AutoSkill → EvoSkill → CoEvoSkills → SkillRL → HyperAgents) and (6) **local-first privacy** (IndexedDB, Ollama, OpenWebUI, confidential computing) — and as of May 2026 these six axes have crystallised into a coherent design surface that any harness aiming to feel like a teammate must address.

## Why this canon matters

Through 2024 and most of 2025, "AI personalisation," "multi-agent frameworks," "skill marketplaces," "branching UX," "self-improving agents," and "local-first privacy" were treated as independent research / product threads. By May 2026 they have visibly coalesced. The same product (LobeHub) ships ICPEA typed memory, Agent Groups, MCP marketplace, branching chat, Pages, and IndexedDB local mode. The same research lab (Anthropic) ships persistent Memory + Memory Spaces + Skills + Subagents + Live Artifacts. The same paper line (Voyager → AutoSkill → CoEvoSkills) is now read as a single co-evolution canon rather than five disconnected papers. The convergence is the story.

The unifying theme is **persistent identity**. A teammate grows with you only if it remembers you (memory), can collaborate with peer specialists who also remember (multi-agent), can reach the tools you use (MCP marketplace), can let you explore alternatives without losing the trail (branching), can improve at things you do repeatedly (co-evolution), and can do all this without exfiltrating your data (local-first). Removing any one axis collapses the teammate metaphor. This is why the six-axis lens, not any single axis, is the right way to evaluate harness designs in 2026.

Take this canon seriously and three things change. (1) You stop building memory or marketplace or branching as standalone features and start designing them as a coordinated substrate. (2) You stop measuring harnesses on a single benchmark axis and start scoring them on six-axis maturity. (3) You start treating an agent's *teammate-feel* as a measurable product property, not a vibes-based marketing line — the six axes give you concrete pass/fail criteria.

## Problem this canon characterises

- An agent that doesn't remember the user across sessions is a chatbot, not a teammate.
- An agent that can't dispatch to peer specialists collapses on tasks bigger than its context window.
- An agent that can't reach external tools is a fluent encyclopedia, not a collaborator.
- An agent whose UX is a single linear chat thread can't surface alternatives without forcing the user to re-prompt.
- An agent that doesn't grow at things the user does repeatedly silently caps the relationship.
- An agent that exfiltrates everything to a vendor's server fails the privacy test for any sensitive use.

## §1 — Axis 1: typed personal memory (the identity layer)

The 2023–2026 personal-memory canon collapses into four reference patterns:

### MemGPT / Letta — virtual context as OS

- *MemGPT: Towards LLMs as Operating Systems* (Packer et al., UC Berkeley, 2023, [arXiv:2310.08560](https://arxiv.org/abs/2310.08560)). Productionised as **Letta** ([github.com/letta-ai/letta](https://github.com/letta-ai/letta), 22.4k★).
- **Idea.** LLM context as virtual memory; explicit page-in / page-out via `archival_memory_insert/search` tool calls. Sleep-time compute consolidates between turns.
- **Teammate insight.** Memory is a hierarchy with explicit primitives the agent itself controls; the OS abstraction is the first credible treatment of *persistent identity*.

### Mem0 — production-ready scalable memory

- *Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory* (Chhikara et al., Mem0 Inc., Apr 2025, [arXiv:2504.19413](https://arxiv.org/abs/2504.19413)). Repo [github.com/mem0ai/mem0](https://github.com/mem0ai/mem0) — 48k+★.
- **Idea.** Two-phase architecture: extract → consolidate/update; **Mem0g** variant adds graph-memory; three parallel retrieval scorers (semantic + keyword + entity).
- **Headline numbers.** **+26% over OpenAI Memory** on LoCoMo benchmark (LLM-as-Judge); **~91% lower p95 latency** vs full-context baselines.
- **Teammate insight.** Memory is *user-scoped by default* (USER_ID filter on every search); consolidation prevents duplicate / contradictory raw extractions.

### LobeHub ICPEA — typed five-layer schema

- [RFC-144 Universal User Memory](https://lobehub.com/blog/rfc-144). **I**dentity / **C**ontext / **P**reference / **E**xperience / **A**ctivity. Async extractor via Upstash workflow. Per-agent layer access control.
- **Teammate insight.** Typed layers let injection budgets be allocated per layer (Identity always present, Activity FIFO-rotated, Preference promoted by edit-evidence). The async extractor decouples conversation latency from memory write quality.
- See [205-lobehub-collaborative-teammate-platform](205-lobehub-collaborative-teammate-platform.md) §2.1.

### Vendor-product convergence — OpenAI / Anthropic / Google

- **OpenAI ChatGPT Memory.** Saved-memories 2024 → reference-past-conversations Apr 2025 → free-tier Jun 2025. Account-scoped, opt-out, excluded from training.
- **Anthropic Claude Memory.** Max/Team/Enterprise Aug 2025 → Pro/Max automatic Oct 2025 → free tier Mar 2 2026. Stores **preferences, project context, facts**; *distinct memory spaces* wall off work vs personal; April 2026 added **"dreaming"** — pattern-finding across past sessions ([Claude Code Dreams](https://claudefa.st/blog/guide/mechanics/auto-dream)).
- **Google Gemini.** Gemini Live carries personal preferences via Google account; cross-product Workspace personalisation.
- **Convergence.** All three vendors converged on the same shape: typed slots (preferences, projects, facts) + temporary chat mode + distinct memory spaces. ICPEA is the OSS canonicalisation of this vendor convergence.

### Adjacent / supporting

- **MemoryBank / SiliconFriend** (Zhong et al. 2023, AAAI 2024, [arXiv:2305.10250](https://arxiv.org/abs/2305.10250)) — Ebbinghaus forgetting curve as the memory-strength function (recency × access × salience).
- **PRELUDE / CIPHER** (Gao et al. 2024, NeurIPS) — user *edits* as the latent preference signal. Edits are dense, free-of-effort, and unambiguous.
- **Cross-link.** [184-strongest-memory-techniques-synthesis-may-2026](184-strongest-memory-techniques-synthesis-may-2026.md), [185-memory-integration-playbook](185-memory-integration-playbook.md), [181-mem0-deep-dive](181-mem0-deep-dive.md), [182-memory-frontiers-2026](182-memory-frontiers-2026.md), [183-oss-memory-landscape-may-2026](183-oss-memory-landscape-may-2026.md), [186-mnema-witness-lattice](186-mnema-witness-lattice.md), [188-witness-provenance-memory-techniques-synthesis](188-witness-provenance-memory-techniques-synthesis.md), [72-claude-mem-persistent-memory-compression](72-claude-mem-persistent-memory-compression.md).

## §2 — Axis 2: multi-agent collaboration primitives

A taxonomy of the dominant 2026 patterns:

| Pattern | Reference | Repo + stars | When |
|---|---|---|---|
| **GroupChat (manager + roles)** | AutoGen → MAF | [microsoft/autogen](https://github.com/microsoft/autogen) 50.4k★ (maintenance), [microsoft/agent-framework](https://github.com/microsoft/agent-framework) production successor | Baseline; turn-taking by speaker-selection |
| **Role catalogs (typed contracts)** | CrewAI | [crewAIInc/crewAI](https://github.com/crewAIInc/crewAI) 50.8k★ | Tasks with clear role decomposition |
| **SDLC-SOP roleplay** | MetaGPT, ChatDev | [FoundationAgents/MetaGPT](https://github.com/FoundationAgents/MetaGPT) 67.8k★, [OpenBMB/ChatDev](https://github.com/OpenBMB/ChatDev) 33k★ | Software / data pipelines |
| **Agent-as-graph-node** | LangGraph (supervisor + swarm) | [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph) 31.3k★ | Persistent state, parallel branches, HITL interrupts |
| **Orchestrator + ledger** | Magentic-One | Microsoft Research, in autogen tree | Generalist task with planning, re-planning |
| **Handoffs (stateless transfer)** | OpenAI Swarm → Agents SDK | [openai/swarm](https://github.com/openai/swarm) — superseded | Educational; production via Agents SDK |
| **Code-as-action dispatch** | smolagents CodeAgent | [huggingface/smolagents](https://github.com/huggingface/smolagents) 26k+★ | Code-emitting agents with full Python control |
| **Filesystem-backed sub-agents** | deepagents | [langchain-ai/deepagents](https://github.com/langchain-ai/deepagents) 22.5k★ | Multi-step, multi-hour deep tasks |
| **Long-horizon SuperAgent** | DeerFlow 2.0 | [bytedance/deer-flow](https://github.com/bytedance/deer-flow) ~45k★ | Sandboxes + memory + skills + sub-agents |
| **Production runtime** | Agno (formerly Phidata) | [agno-agi/agno](https://github.com/agno-agi/agno) ~39k★ | Built-in monitoring, memory, deployment |
| **Open-governance fork** | AG2 | [ag2ai/ag2](https://github.com/ag2ai/ag2) | When vendor-led framework moves to maintenance |

**Important institutional note.** AutoGen → MAF (Microsoft Agent Framework) consolidation in Oct 2025 means new production work should target MAF; AG2 is the community continuation of AutoGen v0.3 for those who don't want vendor lock-in.

**Cross-link.** [02-subagent-delegation](02-subagent-delegation.md), [13-react](13-react.md), [20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md), [42-langchain-deep-agents](42-langchain-deep-agents.md), [43-twelve-harness-patterns](43-twelve-harness-patterns.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md), [126-frameworks-comparison](126-frameworks-comparison.md), [164-crewai-multi-agent-framework](164-crewai-multi-agent-framework.md), [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md), [166-research-agent-frameworks-synthesis](166-research-agent-frameworks-synthesis.md), [187-multi-agent-shared-memory-landscape](187-multi-agent-shared-memory-landscape.md), [189-recursive-multi-agent-systems](189-recursive-multi-agent-systems.md).

## §3 — Axis 3: MCP-first skill catalogs

The pattern as it has crystallised by May 2026:

1. **One-click install** from a marketplace UI (LobeHub, Smithery, Glama).
2. **Permission negotiation at install**, not at first-use — surfaces over-broad permissions when the user is paying attention.
3. **Hosted MCP providers** — Cloudflare Workers, Smithery (cloud + Docker), Mintlify (docs + MCP), Glama (registry + gateway), FastMCP Cloud (Python, free tier with built-in OAuth + monitoring + Git CI/CD).
4. **Standard MCP schemas** — three primitives (resources / tools / prompts), discovery via `initialize` capability manifest.
5. **Convergence with Anthropic Skills** — `SKILL.md` and MCP servers cross-reference; a skill can declare an MCP dependency, an MCP server can ship a skill.
6. **Scale.** ~12,000+ MCP servers across directories (May 2026); 23,212 in Glama's registry alone; 2,000+ appeared on GitHub within weeks of MCP open-sourcing in 2024.

### Key repos / providers

- [github.com/modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) — official reference (filesystem, GitHub, Postgres, Slack…).
- [Smithery](https://smithery.ai/) — cloud-hosted MCP at scale; CLI + Docker.
- [Glama](https://glama.ai/mcp/servers) — 23,212 servers; multi-transport gateway.
- [registry.modelcontextprotocol.io](https://registry.modelcontextprotocol.io/) — official Anthropic-maintained registry, preview Sept 2025.
- [github.com/prefecthq/fastmcp](https://github.com/prefecthq/fastmcp) — FastMCP Python (1M+ downloads/day, ~70% of MCP servers across all languages).
- [github.com/punkpeye/fastmcp](https://github.com/punkpeye/fastmcp) — FastMCP TypeScript v4.0.1.
- [github.com/anthropics/skills](https://github.com/anthropics/skills) — `SKILL.md` reference.

**Cross-link.** [04-skills](04-skills.md), [07-model-context-protocol](07-model-context-protocol.md), [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md), [79-skill-rag](79-skill-rag.md), [134-semantic-indexing](134-semantic-indexing.md), [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md), [176-skill-discovery-curator-oss-landscape-may-2026](176-skill-discovery-curator-oss-landscape-may-2026.md), [177-skills-discovery-curator-strongest-2026-techniques](177-skills-discovery-curator-strongest-2026-techniques.md), [178-online-skill-discovery-and-curation-on-the-go](178-online-skill-discovery-and-curation-on-the-go.md), [179-skill-retrieval-routing-and-activation](179-skill-retrieval-routing-and-activation.md), [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md).

## §4 — Axis 4: branching / Pages / collaborative-document UX

**Branching conversations.** LobeHub forkable chat (Nov 2024), now imitated by LibreChat, OpenWebUI, AnythingLLM. Tree-structured navigation supersedes linear chat. The right substrate for parallel agent rollouts (each branch = an agent's exploration the user picks from). Composes with [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md) (the *method*) — branching makes ToT a UX primitive.

**Claude Artifacts → Generative UI → Live Artifacts → Claude Design.**

- *Artifacts* — generated content deliverables (code, docs, SVG); free-tier Feb 2026.
- *Generative UI* — interactive HTML/JS subset, runs inside the conversation.
- *Live Artifacts* (April 2026) — refresh on each open with current data; persistent across sessions; per-user private storage *or* shared.
- *Claude Design* (April 17 2026) — visual tool on Opus 4.7, research preview at `claude.ai/design` for Pro/Max/Team/Enterprise.

**Lobe Pages.** Notion-style timeline-versioned shared document where multiple agents and the human edit concurrently. ([LobeHub Pages docs](https://lobehub.com/docs/usage/getting-started/page).)

**Notion AI.** Full agent suite inside Notion ([notion.com/product/ai](https://www.notion.com/product/ai)) — agents inside an existing knowledge-graph product.

**Cross-link.** [12-todo-scratchpad-state](12-todo-scratchpad-state.md), [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md).

## §5 — Axis 5: co-evolution / human–agent learning loop

The 2023–2026 co-evolution canon:

- **Voyager** (Wang et al. 2023, TMLR 2024, [arXiv:2305.16291](https://arxiv.org/abs/2305.16291)) — automatic curriculum + ever-growing skill library + iterative prompting; **3.3× more unique items, 15.3× faster tech-tree milestones** in Minecraft. The canonical "skill-as-code that compounds" reference. See [89-voyager-deep](89-voyager-deep.md), [19-voyager-skill-libraries](19-voyager-skill-libraries.md).
- **HyperAgents** (Meta AI, March 2026, [arXiv:2603.19461](https://arxiv.org/abs/2603.19461)) — self-referential agents combining task + meta agent in one editable program; the meta-modification procedure is itself editable. **DGM-Hyperagents** instantiate across coding, paper review, robotics reward design, Olympiad math grading.
- **AutoGenesis** ([arXiv:2604.15034](https://arxiv.org/pdf/2604.15034)) — two-layer protocol decoupling Resource Substrate (what may evolve) from Self-Evolution (how updates are proposed/assessed/committed).
- **AutoSkill / EvoSkill / CoEvoSkills / SkillRL / Ctx2Skill** — corpus deep-dives at [167-autoskill-experience-driven-lifelong-learning](167-autoskill-experience-driven-lifelong-learning.md), [170-skillrl-recursive-skill-augmented-rl](170-skillrl-recursive-skill-augmented-rl.md), [169-coevoskills-co-evolutionary-verification](169-coevoskills-co-evolutionary-verification.md), [168-evoskill-coding-agent-skill-discovery](168-evoskill-coding-agent-skill-discovery.md), [154-ctx2skill-self-evolving-context-skills](154-ctx2skill-self-evolving-context-skills.md), with synthesis at [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md).
- **R1-Searcher++** (Song et al., May 2025, [arXiv:2505.17005](https://arxiv.org/abs/2505.17005)) — outcome-only RL with **memorization mechanism** that continuously assimilates retrieved info into internal knowledge.
- **PromptBreeder** (Fernando et al., DeepMind 2023, [arXiv:2309.16797](https://arxiv.org/abs/2309.16797)) — self-referential prompt evolution; mutates both task-prompts and the mutation-prompts.
- **GEPA** ([arXiv:2507.19457](https://arxiv.org/abs/2507.19457), ICLR 2026 oral) — genetic prompt evolution + natural-language reflection + Pareto candidate selection. **Outperforms GRPO by +6% avg, up to +20%, with 35× fewer rollouts**; beats MIPROv2 by +10%. Repo [github.com/gepa-ai/gepa](https://github.com/gepa-ai/gepa).
- **OPRO** (Yang et al., DeepMind 2023) — *Optimization by PROmpting* — earlier zero-shot prompt optimiser.
- **Inverse Constitutional AI** (ICAI, [arXiv:2406.06560](https://arxiv.org/html/2406.06560v1)) — formulates constitution-derivation from preference pairs as compression; per-user 3-principle constitutions work best for the user they were generated for.
- **C3AI** ([arXiv:2502.15861](https://arxiv.org/html/2502.15861v1), ACM Web 2025) — Crafting and Evaluating Constitutions; positively-framed behaviour-based principles align best.

**Teammate insight.** A user's personal constitution is the cleanest serialisable form of "what working with you should feel like" — far smaller than a preference dataset, far more interpretable than a reward model.

**Cross-link.** [14-reflexion](14-reflexion.md), [19-voyager-skill-libraries](19-voyager-skill-libraries.md), [36-autogenesis-self-evolving-agents](36-autogenesis-self-evolving-agents.md), [45-hyperagents-self-modification](45-hyperagents-self-modification.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md), [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md), [167-171] cluster, [191-onemancompany-skills-to-talent](191-onemancompany-skills-to-talent.md), [197-argus-omega-vol-3-recursive-skills-curator](197-argus-omega-vol-3-recursive-skills-curator.md).

## §6 — Axis 6: voice / multimodal collaboration surfaces

- **`@lobehub/tts`** (v5.1.2 Mar 2026) — server + browser TTS/STT with React Hooks; supports OpenAI / EdgeSpeech / Microsoft / others.
- **OpenAI Realtime API** — `/v1/realtime`; **gpt-realtime** + **gpt-realtime-mini** (cost-sensitive). Pricing ~$0.06/min input / ~$0.24/min output (May 2026, 20% drop from launch). Eight voices.
- **Google Live API (Gemini Live)** — real-time voice + video understanding via screen-share / camera; multimodal turn-taking with persistent personalisation via Google account.
- **ElevenLabs Conversational AI / ElevenAgents** — TTS in 70+ languages, fine-tuned STT, proprietary turn-taking model. Channel-agnostic (phone via Twilio, WhatsApp, web). IBM partnership Mar 2026 to bring TTS/STT into watsonx Orchestrate.
- **Pipecat** ([github.com/pipecat-ai/pipecat](https://github.com/pipecat-ai/pipecat)) — pipeline framework: VAD → STT → LLM → TTS as frame stages. More control, more turn-taking work.
- **LiveKit Agents** — Go/Python/Node primitives over LiveKit's WebRTC SFU; shines at multi-participant rooms (AI agent in a 5-human Zoom-style call).

**Vision.** **GLM-4.5V** (106B / 12B-active) SOTA on 42 benchmarks across image/video/document/GUI/grounding; **Qwen3-VL** unified vision+text+video; **Gemma 4** Apr 2026 (4 sizes, native bounding-box output). With OSS VLMs reaching frontier-equivalent on 40+ benchmarks, **screen-share + voice** becomes the right teammate substrate for desktop work — not text-only chat.

**Cross-link.** [137-voice-agents](137-voice-agents.md), [48-voiceagentrag-dual-agent](48-voiceagentrag-dual-agent.md), [136-multimodal-rag](136-multimodal-rag.md), [104-glm-5v-turbo-native-multimodal-agents](104-glm-5v-turbo-native-multimodal-agents.md), [105-agentic-world-modeling-visual-generation](105-agentic-world-modeling-visual-generation.md), [190-agentic-world-modeling-taxonomy](190-agentic-world-modeling-taxonomy.md).

## §7 — Axis 7: privacy + local-first / confidential agents

- **LobeHub IndexedDB client mode** — local-first by default, opt-in cloud sync via experimental CRDT.
- **OpenWebUI + Ollama** — [open-webui/open-webui](https://github.com/open-webui/open-webui) 124k★ + Ollama 164k★ (the de-facto local LLM runtime).
- **LM Studio / Jan / AnythingLLM** — desktop GUIs with varying polish vs flexibility tradeoffs.
- **LibreChat** — [danny-avila/LibreChat](https://github.com/danny-avila/LibreChat) 36.4k★, v0.8.5 Apr 2026; self-hosted multi-user.
- **Sealos / Zeabur** — AI-native cloud OS / PaaS deployment.
- **Confidential computing.** AWS Nitro Enclaves (proprietary), **AMD SEV-SNP** (hardware-enforced integrity, page validation, guest attestation), **Intel TDX** (Trust Domains, memory + state encryption isolating from host OS / hypervisor), Red Hat OpenShift confidential clusters on Azure with SEV-SNP, **SecretVM** (Zen-4 SEV-SNP CVM, Feb 2026 update).

**Teammate insight.** Once an agent stores user identity + secrets + persisted skills, the threat model promotes from "data at rest" to **"data in use."** Confidential VMs are the only real answer at the operator-doesn't-trust-cloud-provider level.

**Cross-link.** [122-explainability-compliance](122-explainability-compliance.md), [125-system-level-production-patterns](125-system-level-production-patterns.md), [147-vendor-lock-in](147-vendor-lock-in.md), [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [35-malicious-intermediary-attacks](35-malicious-intermediary-attacks.md).

## §8 — Marketplace economics (a quick aside)

- **OpenAI GPT Store.** Live on Plus/Enterprise; **revenue share still pending** as of May 2026 (announced 2024, US-only initial eligibility, Spotify-style payouts based on usage). Two years on, no creator has been paid.
- **Anthropic Skills.** Free, open, no monetisation built in; positioned as a developer-collaboration ecosystem rather than a commerce surface.
- **Hugging Face Spaces.** Free distribution, no direct payouts; monetisation indirect through Pro subs and paid services.
- **LobeHub Agent Market.** Community contribution model (PR to `index.json`); no direct monetisation for contributors but no friction either.
- **Coze Studio + Coze Loop** (ByteDance, open-sourced July 2025) — Studio = builder, Loop = ops. Average **146 issues/month** despite smaller star count vs Dify.
- **Dify.** [langgenius/dify](https://github.com/langgenius/dify) 111k+★, plugin-first marketplace from v1.0 (Feb 2025) — *retrofitted* a marketplace into a mature platform; better to design for it from day one.
- **Flowise.** Node-based no-code; the natural alternative if your team already lives in LangChain.

**Lesson.** Monetisation must ship with the marketplace, not be promised later. Community-contribution models without direct payouts (LobeHub, HF Spaces) are *more durable* than promised-payouts-pending models (GPT Store).

## §9 — A maturity matrix for "teammate-feel"

A scorecard you can apply to any harness:

| Axis | Level 0 (none) | Level 1 (basic) | Level 2 (production) | Level 3 (frontier) |
|---|---|---|---|---|
| Personal memory | None | Flat blob | Typed slots + access control | ICPEA + async extractor + per-agent layers |
| Multi-agent | Single agent | Manual handoffs | Supervisor + worker | Graph-state + branching + ledger replanning |
| Skill catalog | Hand-coded tools | Static catalog | Marketplace + browse | MCP-first + one-click install + permission-at-install |
| Branching UX | Linear chat | Regenerate button | Branch tree | Branch tree + Pages + Live Artifacts |
| Co-evolution | None | Manual prompt updates | Skill auto-creation | Constitution + GEPA + outcome-RL + meta-modification |
| Local-first | Server-only | Optional self-host | Local-first DB + sync | Confidential VM + per-tenant isolation + CRDT sync |

A harness scoring **L2+ on all six axes** has earned the "teammate that grows with you" framing. As of May 2026, only LobeHub (axis-balanced) and Anthropic Claude with Skills+Memory+Subagents+Live Artifacts (richer on co-evolution and branching) clear this bar across all six. Most OSS harnesses score L2 on 2–3 axes and L0–L1 on the rest.

## §10 — Failure modes and limitations

- **Memory privacy.** A typed personal-memory layer is a higher-leverage target than a flat embedding store — one node link can expose a whole identity. Cf. [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [82-poisonedrag](82-poisonedrag.md).
- **Multi-agent drift / collapse.** Without diversity discipline, role catalogs and debate frameworks collapse on homogeneous outputs ([98-diversity-collapse-mas](98-diversity-collapse-mas.md)).
- **Marketplace supply-chain attacks.** A trusted MCP server can ship a poisoned tool; trust verdicts at install (Argus-style) are the mitigation. Cf. [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md).
- **Branching state inconsistency.** Forking a branch forks the chat but not always the agent's memory state — a subtle semantic gap.
- **Co-evolution reward hacking.** Outcome-only RL can teach the agent to reward-hack the eval. AgentPRM-style process rubrics are the mitigation.
- **Local-first sync conflicts.** CRDT-style sync handles many cases but not all (especially binary attachments, encrypted memory).
- **Constitution stability.** Per-user constitutions drift as the user's preferences drift; stability is an open problem.
- **Maturity-matrix gaming.** Self-scoring can be optimistic; co-evaluation by a peer team is the safest signal.

## §11 — When to pursue all six axes vs subset

**Pursue all six** when (a) the agent will be used daily over months / years by a small set of users, (b) you control the deployment and can amortise the engineering cost, (c) the user's expectation is "this gets to know me" rather than "I prompt this fresh each time."

**Pursue a subset** when (a) the agent is one-shot or short-horizon, (b) the user surface is API-only and there's no consumer chat to ship, (c) the deployment is a B2B integration where the user *isn't* the same person across sessions.

Polaris, Lyra, argus all fall in the first category — see [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md) for the staged plan.

## §12 — Implications for harness engineering

- **Design the six axes together, not in sequence.** A memory schema that ignores the multi-agent axis (no per-agent layer access control) needs reworking when Agent Groups arrive.
- **Adopt ICPEA verbatim** as the typed personal-memory schema. Cf. [205-lobehub-collaborative-teammate-platform](205-lobehub-collaborative-teammate-platform.md) §2.1.
- **Pick a multi-agent shape per task.** GroupChat for baseline, role catalog for SDLC-shaped tasks, graph-as-node for HITL / parallel branches, ledger-orchestrator for generalist tasks.
- **MCP-first skill catalog.** One-click install + permission negotiation at install + Argus-style trust gate. Cf. [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md).
- **Branching as universal UX.** Any trajectory exposed to the user should be forkable. Cf. [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md).
- **Pages / shared documents as collaborative-output primitive.** Co-authored markdown with timeline-versioning. Cf. [12-todo-scratchpad-state](12-todo-scratchpad-state.md).
- **Co-evolution as a first-class concern.** Bake skill-auto-creation into the agent runtime; don't bolt it on later. Cf. [167-autoskill-experience-driven-lifelong-learning](167-autoskill-experience-driven-lifelong-learning.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md).
- **Per-user constitution as a serialisable preference object.** Smaller, more interpretable, and more transferable than a reward model. Cf. ICAI / C3AI.
- **Local-first storage is a competitive moat for sensitive deployments.** Don't retrofit — design for it from day one. Cf. [125-system-level-production-patterns](125-system-level-production-patterns.md).
- **Voice + screen-share + Pages as the desktop-work substrate.** Text-only chat caps the productivity gain. Cf. [137-voice-agents](137-voice-agents.md).
- **Skill marketplace as contributor-onboarding, not just discovery.** PR-to-`index.json` is the cheapest network-effect machine.
- **Async memory extractor.** Decouple chat-path latency from memory-write quality. Cf. [184-strongest-memory-techniques-synthesis-may-2026](184-strongest-memory-techniques-synthesis-may-2026.md).
- **Per-agent layer access control on memory.** No agent gets blanket access to all memory — declared layers only. Cf. [06-permission-modes](06-permission-modes.md).
- **Equal-budget benchmarks for multi-agent claims.** Don't take "multi-agent helps" at face value. Cf. [202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md).
- **Confidential computing for high-stakes deployments.** Once you store user identity + secrets + persisted skills, the threat model is "data in use."
- **Score harnesses on the maturity matrix.** Use the §9 grid as the regression yardstick.

**The one-line takeaway for harness designers:** Treat **typed personal memory + multi-agent + MCP marketplace + branching + co-evolution + local-first** as a *single design surface*, not six independent features — the teammate metaphor only emerges when all six axes mature together.

## §13 — Cross-references

- Memory: [09-memory-files](09-memory-files.md), [184-strongest-memory-techniques-synthesis-may-2026](184-strongest-memory-techniques-synthesis-may-2026.md), [185-memory-integration-playbook](185-memory-integration-playbook.md), [181-mem0-deep-dive](181-mem0-deep-dive.md), [186-mnema-witness-lattice](186-mnema-witness-lattice.md), [188-witness-provenance-memory-techniques-synthesis](188-witness-provenance-memory-techniques-synthesis.md).
- Multi-agent: [02-subagent-delegation](02-subagent-delegation.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md), [126-frameworks-comparison](126-frameworks-comparison.md), [164-crewai-multi-agent-framework](164-crewai-multi-agent-framework.md), [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md), [166-research-agent-frameworks-synthesis](166-research-agent-frameworks-synthesis.md), [187-multi-agent-shared-memory-landscape](187-multi-agent-shared-memory-landscape.md), [189-recursive-multi-agent-systems](189-recursive-multi-agent-systems.md).
- Skills: [04-skills](04-skills.md), [07-model-context-protocol](07-model-context-protocol.md), [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md), [176-skill-discovery-curator-oss-landscape-may-2026](176-skill-discovery-curator-oss-landscape-may-2026.md), [177-skills-discovery-curator-strongest-2026-techniques](177-skills-discovery-curator-strongest-2026-techniques.md), [180-argus-skill-router-agent-design](180-argus-skill-router-agent-design.md).
- Co-evolution: [19-voyager-skill-libraries](19-voyager-skill-libraries.md), [36-autogenesis-self-evolving-agents](36-autogenesis-self-evolving-agents.md), [45-hyperagents-self-modification](45-hyperagents-self-modification.md), [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md), [167-autoskill-experience-driven-lifelong-learning](167-autoskill-experience-driven-lifelong-learning.md), [168-evoskill-coding-agent-skill-discovery](168-evoskill-coding-agent-skill-discovery.md), [169-coevoskills-co-evolutionary-verification](169-coevoskills-co-evolutionary-verification.md), [170-skillrl-recursive-skill-augmented-rl](170-skillrl-recursive-skill-augmented-rl.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md), [197-argus-omega-vol-3-recursive-skills-curator](197-argus-omega-vol-3-recursive-skills-curator.md).
- UX: [12-todo-scratchpad-state](12-todo-scratchpad-state.md), [15-tree-of-thoughts-lats](15-tree-of-thoughts-lats.md), [143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md).
- Voice / vision: [48-voiceagentrag-dual-agent](48-voiceagentrag-dual-agent.md), [104-glm-5v-turbo-native-multimodal-agents](104-glm-5v-turbo-native-multimodal-agents.md), [136-multimodal-rag](136-multimodal-rag.md), [137-voice-agents](137-voice-agents.md).
- Privacy: [122-explainability-compliance](122-explainability-compliance.md), [125-system-level-production-patterns](125-system-level-production-patterns.md), [147-vendor-lock-in](147-vendor-lock-in.md).
- Direct neighbours: [204-metagpt-foundationagents-2026-refresh](204-metagpt-foundationagents-2026-refresh.md), [205-lobehub-collaborative-teammate-platform](205-lobehub-collaborative-teammate-platform.md), [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md).
