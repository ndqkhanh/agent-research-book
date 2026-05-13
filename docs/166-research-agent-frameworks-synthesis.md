# 166 — Research-Agent Frameworks Synthesis: How Eight 2025–2026 Artifacts Map the Field

**Sources synthesised.** Three surveys + five repositories that together cover the full surface area of the *research-agent framework* category as of mid-2026:

- **Surveys**: [158-deep-research-agents-survey-huang-et-al](158-deep-research-agents-survey-huang-et-al.md) (architecture taxonomy), [159-deep-research-survey-zhang-et-al](159-deep-research-survey-zhang-et-al.md) (capability pipeline), [160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md) (24/7 operational reality).
- **Repositories**: [161-paper-researcher-ai-agent](161-paper-researcher-ai-agent.md) (minimal CrewAI educational), [162-paper2agent-reimagining-papers-as-agents](162-paper2agent-reimagining-papers-as-agents.md) (papers-to-agents transformation), [163-deer-flow-revisited-may-2026](163-deer-flow-revisited-may-2026.md) (super agent harness), [164-crewai-multi-agent-framework](164-crewai-multi-agent-framework.md) (lean Python multi-agent), [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md) (bash-loop autonomous coding).

**One-line thesis.** Together, these eight artifacts do not just describe deep research agents — they *map the entire build-vs-buy-vs-fork decision tree* for anyone constructing a research or coding agent system in 2026. The map has three load-bearing dimensions: **(a) architecture choice** (static vs dynamic workflows, single vs multi-agent, planning strategy — captured by Huang et al.); **(b) capability composition** (planning / question developing / web exploration / report generation as independently optimisable substrates — captured by Zhang et al.); and **(c) operational engineering** (cost discipline, memory bounding, OS-event-driven loops — captured by Deep Researcher Agent and Ralph). The repositories instantiate specific points on this map: Paper-Researcher-AI-Agent (minimal static), Paper2Agent (meta-agent that builds agents), DeerFlow 2.0 (super-harness generalisation), CrewAI (lean Python framework), Ralph (poor-man's durable execution). The synthesis: **memory is files, skills are Markdown, multi-agent is the default, file-based handoffs dominate context dumping, and MCP is the universal tool-server protocol** — and these patterns reproduce independently across every serious system in this set.

This chapter is the *synthesis* across the May/June 2025 → May 2026 research-agent canon. It complements but does not replace [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md) (which synthesised four memory/skill papers) by adding the *framework landscape* dimension.

## The shape of the eight-source evidence

Quick reference to keep the synthesis grounded in source material:

| Source | Type | Year | Distinct contribution |
|--------|------|------|------------------------|
| Huang et al. ([158](158-deep-research-agents-survey-huang-et-al.md)) | Survey | Jun 2025 | Architecture taxonomy: static/dynamic; single/multi-agent; planning strategies |
| Zhang et al. ([159](159-deep-research-survey-zhang-et-al.md)) | Survey | Aug 2025 | Capability pipeline: planning → question dev → web exploration → report |
| Zhang ([160](160-deep-researcher-agent-24x7.md)) | Paper + repo | Apr 2026 | Zero-cost monitoring; constant-size memory; minimal-toolset workers |
| Paper-Researcher-AI-Agent ([161](161-paper-researcher-ai-agent.md)) | Educational repo | 2024 | Minimal CrewAI deep-research MVP (~100 LoC) |
| Paper2Agent ([162](162-paper2agent-reimagining-papers-as-agents.md)) | Production repo + paper | Sep 2025 | Multi-agent system that constructs Paper Agents from research repos |
| DeerFlow 2.0 ([163](163-deer-flow-revisited-may-2026.md)) | Production repo (rewrite) | Feb 2026 | "Super agent harness" with 6 IM channels, 3 sandbox modes, OAuth integrations |
| CrewAI ([164](164-crewai-multi-agent-framework.md)) | Production framework | 2024–ongoing | Crews + Flows paradigms; zero-LangChain; 100K+ certified developers |
| Ralph ([165](165-ralph-autonomous-loop.md)) | Tiny pattern + repo | 2025 | Bash-loop autonomous coding via fresh contexts + git-as-memory |

The diversity is the point: surveys give the taxonomy, papers give the operational reality, repos give the implementation choices.

## The build-vs-buy-vs-fork decision tree

For someone in 2026 deciding how to build a research or coding agent, the eight artifacts collectively yield a clear decision tree:

```
What kind of agent are you building?

│
├── A. EDUCATIONAL / DOMAIN-SPECIFIC TUTORIAL DR AGENT
│       → Fork Paper-Researcher-AI-Agent ([161])
│       → Use CrewAI + your domain's web search tool
│       → Roughly 100–300 LoC of customisation
│
├── B. SOURCE-GROUNDED RESEARCH HARNESS
│       → Use Feynman ([155]) — already exists, MIT-licensed
│       → If you need to extend, fork; structure is clean
│
├── C. SUPER-HARNESS WITH IM CHANNELS / MULTI-RUNTIME OAUTH
│       → Use DeerFlow 2.0 ([163])
│       → Integrates Claude Code OAuth, Codex CLI, MCP, 6 IM channels
│       → Configurable subagent topology
│
├── D. MULTI-AGENT FRAMEWORK FOR ARBITRARY DOMAIN
│       → Use CrewAI ([164])
│       → Crews for autonomy, Flows for control, both composable
│       → Zero LangChain dependency
│
├── E. AUTONOMOUS CODING LOOP WITH FRESH-CONTEXT ITERATIONS
│       → Use Ralph ([165]) pattern
│       → 100 LoC of bash + Markdown prompts
│       → git as memory
│
├── F. 24/7 GPU EXPERIMENTATION AGENT
│       → Use Deep Researcher Agent ([160])
│       → Zero-cost monitoring, constant-size memory
│       → Single-author, single-paper, working open-source
│
├── G. PAPERS-AS-AGENTS PIPELINE
│       → Use Paper2Agent ([162])
│       → 5-step transformation from paper repo to MCP-compatible agent
│       → ~$15/paper amortised over many uses
│
└── H. CUSTOM FRAMEWORK FROM SCRATCH
        → Read all 8 sources first
        → Don't repeat the design space exploration
        → Lean on existing primitives (CrewAI, MCP, MEMTIER)
```

Most teams should *not* be in branch H. The other seven branches cover ~95% of what real research-agent projects need. The exception is when your domain has structural requirements that none of the existing systems handle (high-volume real-time, non-Python language stacks, regulatory compliance constraints incompatible with the existing primitives).

## Five patterns that reproduce independently across every serious system

When the same architectural choice independently appears in four or five separate projects designed by different teams, it is no longer a coincidence — it is *the* right choice for that problem. Here are five patterns that emerge from this set:

### Pattern 1 — Markdown is the agent definition format

Every system in this set defines agents (or skills, or capabilities) as **Markdown files with YAML frontmatter**:

- **Paper2Agent**: 9 agents, all in `agents/*.md` with frontmatter `name / description / model / color`.
- **DeerFlow 2.0**: skills in `skills/` directory, configurable via `DEER_FLOW_SKILLS_PATH`.
- **CrewAI**: agents/tasks in YAML configs (`config/agents.yaml`, `config/tasks.yaml`).
- **Ralph**: agent prompts in `prompt.md` (Amp) and `CLAUDE.md` (Claude Code).
- **Deep Researcher Agent**: agents in Markdown files with frontmatter (Appendix B of [160](160-deep-researcher-agent-24x7.md)).
- **Feynman**: 4 subagents in `.feynman/agents/*.md`, 19 skills in `skills/*/SKILL.md`.
- **Paper-Researcher-AI-Agent**: backstory + role + goal natively, but maps to the same character-as-text pattern.

This convergence is the strongest signal in the field. **Markdown-with-frontmatter is the de facto standard for serialising agent definitions.** It wins because:

- Human-readable.
- Version-controllable.
- Editable by non-developers.
- Portable across runtimes.
- Cheaply parseable.

If you are designing a new framework, do not invent a new format. Use Markdown with YAML frontmatter.

### Pattern 2 — Memory is files, not in-context state

Every system bounds in-context memory and externalises durable state to files:

- **Deep Researcher Agent**: 5K-char two-tier memory (Brief + Log) on disk. Bounded constant.
- **Ralph**: git history + `progress.txt` + `prd.json`. Fresh context per iteration.
- **Feynman**: `outputs/.plans/<slug>.md` + `CHANGELOG.md` + `.provenance.md` sidecars.
- **DeerFlow 2.0**: long-term memory stored externally, loaded selectively into context.
- **Paper2Agent**: extracted tools persist as Python modules; reports persist in `reports/`.
- **CrewAI**: tasks can have `output_file=...`; structured outputs via `output_pydantic`.
- **MEMTIER** ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)): tiered JSONL + semantic facts on disk.

The pattern: **anything that must persist beyond a single LLM call belongs on disk**. In-context memory is for the current decision; durable memory is for everything else.

For coding agents specifically, the *git-as-memory* sub-pattern (Ralph; many of the others if implicitly) is the killer move. git provides content-addressed durable storage with rich query semantics for free. Reuse it.

### Pattern 3 — Multi-agent decomposition is the default

Of the eight systems:

- **Multi-agent**: Paper-Researcher-AI-Agent (2 agents), Paper2Agent (9 agents), DeerFlow 2.0 (configurable), CrewAI (Crew = team), Deep Researcher Agent (Leader + 3 Workers), Feynman ([155](155-feynman-multi-agent-research-harness.md), 4 subagents).
- **Single-agent**: Ralph (single coding instance per iteration).

Even Ralph, the single-agent system, is single-agent *per iteration* — across iterations, it's effectively the same agent re-instantiated, which is closer to "a single role played repeatedly" than truly single-agent.

The convergence: **multi-agent with specialised roles dominates monolithic agents** for any non-trivial workload. The reasons (from across the chapters):

- Specialised agents have smaller, more focused prompts.
- Tool sets per agent shrink, reducing per-call token overhead (Deep Researcher Agent's minimal-toolset insight at [160](160-deep-researcher-agent-24x7.md)).
- Verification can be a separate agent (Feynman's Verifier at [155](155-feynman-multi-agent-research-harness.md)).
- Different roles can use different LLMs (HeavySkill's deliberator-strength insight at [156](156-heavyskill-parallel-reasoning-deliberation.md)).

The exception — single-agent with end-to-end RL trainability (Search-R1, R1-Searcher per [158](158-deep-research-agents-survey-huang-et-al.md), [159](159-deep-research-survey-zhang-et-al.md)) — is a research-frontier choice, not a default for production.

### Pattern 4 — File-based handoffs over context dumping

When subagents communicate, they write artifacts to disk and pass paths, not full content:

- **Feynman**: subagents write `<slug>-research-*.md`; lead agent reads selectively.
- **Paper2Agent**: each step writes JSON outputs (`step1_output.json`, `step2_output.json`, etc.); next step reads them.
- **Deep Researcher Agent**: workers report PIDs and log file paths; leader reads logs at zero LLM cost.
- **DeerFlow 2.0**: skills + memory + sandbox all communicate via filesystem.
- **Ralph**: progress.txt + prd.json are the handoff mechanism between iterations.

The pattern: **subagent communication via files dominates inline result-dumping** for any system that does meaningful multi-agent work. Inline result-dumping pollutes the parent's context window with potentially large tool outputs, accumulating attention pressure across rounds. Files outlive the call; the parent reads selectively.

This is one of the operational insights from [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md) confirmed by every system in this set.

### Pattern 5 — MCP is the universal tool-server protocol

References to Model Context Protocol (MCP) thread through this set:

- **Huang et al. ([158](158-deep-research-agents-survey-huang-et-al.md))** — MCP given a dedicated section (§2.3) as the standardised tool-access layer.
- **DeerFlow 2.0 ([163](163-deer-flow-revisited-may-2026.md))** — native MCP server support with OAuth flows.
- **Paper2Agent ([162](162-paper2agent-reimagining-papers-as-agents.md))** — generates *MCP servers* as its primary output. Every Paper Agent is an MCP-compatible tool surface.
- **Feynman ([155](155-feynman-multi-agent-research-harness.md))** — Pi runtime supports MCP-compatible skill installation.
- **CrewAI ([164](164-crewai-multi-agent-framework.md))** — ships its docs as an MCP server (`docs.crewai.com/mcp`).

The convergence: **MCP has won as the protocol for agent ↔ tool communication**. New systems should ship MCP-compatible tool servers, not bespoke Python tool registries. Existing systems should add MCP adapters.

This is consistent with the broader canon's prediction at [07-model-context-protocol](07-model-context-protocol.md): MCP is the right interoperability bet.

## Three structural insights from the surveys

The two surveys (Huang et al. and Zhang et al.) provide structural insights that the repositories operationalise. Three are worth pulling out:

### Insight 1 — Capability pipelines decompose orthogonally

Zhang et al.'s four-stage pipeline (Planning → Question Developing → Web Exploration → Report Generation) maps cleanly onto every system in this set. Here's the cross-tabulation:

| System | Planning | Question Developing | Web Exploration | Report Generation |
|--------|----------|---------------------|-----------------|---------------------|
| Paper-Researcher-AI-Agent | Implicit (2 fixed tasks) | Researcher's Serper queries | Serper API | Writer's markdown |
| Paper2Agent | step1_prompt + scanner | step2 (executor) | step3 (extractor) | step4 (MCP) + step5 (verify) |
| DeerFlow 2.0 | Lead Agent | Sub-agents | Sandbox + InfoQuest | Skills-driven generation |
| Deep Researcher Agent | THINK | (folded into Code Agent) | EXECUTE | REFLECT |
| CrewAI | Flow's structured planning | Crew's autonomous reasoning | Tools | Tasks with structured output |
| Ralph | PRD acceptance criteria | Implicit | Implicit | git commits + progress.txt |
| Feynman | Plan artifact | Researcher subagent | Researcher's web search | Writer + Verifier |

The pipeline is universal. Every research/coding agent system has these four stages; the differences are in **how richly each stage is implemented** and **how much of each stage is end-to-end-RL-trainable**. The pipeline framing from [159](159-deep-research-survey-zhang-et-al.md) is the right one for understanding any system in the field.

### Insight 2 — Static vs dynamic workflow is the load-bearing axis

Huang et al.'s static-vs-dynamic distinction shows up empirically:

- **Static** (predefined sequential pipeline): Paper-Researcher-AI-Agent, Paper2Agent, Ralph (per-iteration), and any Flow in CrewAI.
- **Dynamic** (LLM-driven adaptive replanning): DeerFlow 2.0, Deep Researcher Agent, Feynman's deepresearch workflow, Crew in CrewAI.

Static workflows dominate when:
- The structure is well-known.
- Auditability matters.
- Cost predictability matters.
- The work decomposes naturally into atomic units.

Dynamic workflows dominate when:
- The work is open-ended.
- Multi-step replanning adds value.
- The agent should adapt to intermediate results.

The right systems offer *both*: CrewAI's Crew + Flow paradigms, DeerFlow 2.0's configurable subagent topology with `is_plan_mode` flags. The choice should be per-task, not per-system.

### Insight 3 — Memory architecture is workload-specific

The memory architectures across this set are radically different — and each is correct for its workload:

| System | Memory architecture | Why |
|--------|---------------------|-----|
| Deep Researcher Agent | Bounded constant ~5K chars (Brief + Log) | Single-project deep iteration over weeks |
| Ralph | git history + progress.txt + prd.json | Coding sessions where git is already memory |
| MEMTIER ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)) | Tiered JSONL + LLM-distilled semantic facts | Multi-session multi-agent recall |
| Feynman ([155](155-feynman-multi-agent-research-harness.md)) | Per-run plan + lab-notebook CHANGELOG | Research workflows with provenance |
| DeerFlow 2.0 | Long-term memory provider (configurable) | General-purpose super-harness |
| CrewAI | In-crew memory + Pydantic outputs | Multi-agent collaboration within tasks |
| Paper2Agent | None (pipeline output is durable artifact) | Paper-to-agent transformation is one-shot |

The insight from [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md) reproduces here: **there is no universal best memory architecture**. The right architecture depends on:

- How long does the agent run?
- How many concurrent projects?
- Is recall across sessions important?
- Is git already capturing the work product?
- What's the failure cost of memory loss?

For builders: don't pick a memory architecture in the abstract. Pick it after answering these workload questions.

## What's missing from this set

What the eight artifacts in this set don't cover well:

1. **Cost-control discipline as a system-level concern**. Deep Researcher Agent ([160](160-deep-researcher-agent-24x7.md)) covers it brilliantly for its specific workload, but the surveys and other systems treat cost as secondary. For production deployments, cost is often the binding constraint, and this set under-emphasises it.
2. **Multi-tenancy and isolation between users**. DeerFlow 2.0 has per-user session config; CrewAI has the AMP Suite for enterprise; the others mostly assume single-user deployment. Multi-tenant agent systems are a different design space.
3. **Compliance and auditability**. Feynman's Verifier ([155](155-feynman-multi-agent-research-harness.md)) is the closest to compliance-grade discipline, but most of these systems don't address regulated-industry use cases.
4. **Cross-language stacks**. All eight systems are Python-or-bash centric. Node.js / TypeScript / Rust / Go agent stacks are emerging but not represented here.
5. **Privacy-sensitive deployments**. None of the systems address running deep research on private data with strong privacy guarantees.

These gaps suggest where the next year of research-agent framework development will go: cost discipline (more "Deep Researcher Agent" style work), multi-tenancy (SaaS deployments of these systems), compliance (verifier-by-default architectures), polyglot (Mastra-like efforts in JS), private (local-LLM-friendly architectures).

## How to use this synthesis

For builders, the practical reading order:

1. **Start with the surveys** ([158](158-deep-research-agents-survey-huang-et-al.md), [159](159-deep-research-survey-zhang-et-al.md)) — get the vocabulary and mental map.
2. **Read [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md)** — get the substrate-level argument (memory + skills as the two trainable layers).
3. **Pick a starting point** — fork Paper-Researcher-AI-Agent ([161](161-paper-researcher-ai-agent.md)) for educational, fork Feynman ([155](155-feynman-multi-agent-research-harness.md)) for source-grounded research, fork DeerFlow 2.0 ([163](163-deer-flow-revisited-may-2026.md)) for a super-harness, use CrewAI ([164](164-crewai-multi-agent-framework.md)) for a Python framework, copy Ralph ([165](165-ralph-autonomous-loop.md)) for autonomous coding loops, or run Paper2Agent ([162](162-paper2agent-reimagining-papers-as-agents.md)) on the papers you care about.
4. **Read [160](160-deep-researcher-agent-24x7.md)** for the cost-control playbook *before* deploying.
5. **Layer in MEMTIER** ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)) once your agent's memory becomes a load-bearing concern.
6. **Add HeavySkill / Ctx2Skill skill artifacts** ([154](154-ctx2skill-self-evolving-context-skills.md), [156](156-heavyskill-parallel-reasoning-deliberation.md)) as your capability list grows.

For framework authors:

1. **Default to Markdown-with-frontmatter** for all agent / skill / task definitions.
2. **Externalise memory** to files; never make in-context the only memory.
3. **Generate MCP-compatible tool surfaces** by default.
4. **Provide both static and dynamic workflow primitives** (CrewAI's Flow + Crew is the canonical example).
5. **Ship official skills for coding agents** (CrewAI's `crewaiinc/skills` marketplace is the template).
6. **Provide a docs MCP server** (CrewAI's `docs.crewai.com/mcp` is the template).
7. **Make the runtime's hardware requirements explicit** (DeerFlow 2.0's deployment-sizing table is the template).
8. **Provide both human-attended and autonomous modes**, with the autonomous mode having explicit safety mechanisms (Deep Researcher Agent's anti-burn protection, Ralph's `<promise>COMPLETE</promise>` signal).

## Where this fits

This is a synthesis chapter — read after the eight sources it synthesises:

- [158-deep-research-agents-survey-huang-et-al](158-deep-research-agents-survey-huang-et-al.md)
- [159-deep-research-survey-zhang-et-al](159-deep-research-survey-zhang-et-al.md)
- [160-deep-researcher-agent-24x7](160-deep-researcher-agent-24x7.md)
- [161-paper-researcher-ai-agent](161-paper-researcher-ai-agent.md)
- [162-paper2agent-reimagining-papers-as-agents](162-paper2agent-reimagining-papers-as-agents.md)
- [163-deer-flow-revisited-may-2026](163-deer-flow-revisited-may-2026.md)
- [164-crewai-multi-agent-framework](164-crewai-multi-agent-framework.md)
- [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md)

Complementary syntheses in this canon: [56-sea-landscape-2026](56-sea-landscape-2026.md), [60-sea-top-github-repos](60-sea-top-github-repos.md), [66-meta-harness-landscape](66-meta-harness-landscape.md), [67-recommended-breakthrough-project](67-recommended-breakthrough-project.md), [76-ten-links-synthesis](76-ten-links-synthesis.md), [99-papers-deep-dive-synthesis](99-papers-deep-dive-synthesis.md), [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md). Each takes a different cross-section; this one takes the *research-agent framework landscape* cross-section.

## References

1. Huang et al. 2025. *Deep Research Agents: A Systematic Examination And Roadmap*. arXiv:2506.18096v2. Deep-dive: [158](158-deep-research-agents-survey-huang-et-al.md).
2. Zhang et al. 2025. *Deep Research: A Survey of Autonomous Research Agents*. arXiv:2508.12752. Deep-dive: [159](159-deep-research-survey-zhang-et-al.md).
3. Zhang. 2026. *Deep Researcher Agent: An Autonomous Framework for 24/7 Deep Learning Experimentation*. arXiv:2604.05854v1. Deep-dive: [160](160-deep-researcher-agent-24x7.md).
4. Repository: https://github.com/manpreet171/Paper-Researcher-AI-Agent. Deep-dive: [161](161-paper-researcher-ai-agent.md).
5. Repository: https://github.com/jmiao24/Paper2Agent. Companion paper: arXiv:2509.06917. Deep-dive: [162](162-paper2agent-reimagining-papers-as-agents.md).
6. Repository: https://github.com/bytedance/deer-flow. Deep-dive: [163](163-deer-flow-revisited-may-2026.md).
7. Repository: https://github.com/crewAIInc/crewAI. Deep-dive: [164](164-crewai-multi-agent-framework.md).
8. Repository: https://github.com/snarktank/ralph. Original pattern: https://ghuntley.com/ralph/. Deep-dive: [165](165-ralph-autonomous-loop.md).
9. Adjacent canon: [04-skills.md](04-skills.md), [07-model-context-protocol.md](07-model-context-protocol.md), [09-memory-files.md](09-memory-files.md), [12-todo-scratchpad-state.md](12-todo-scratchpad-state.md), [25-agentic-rag.md](25-agentic-rag.md), [42-langchain-deep-agents.md](42-langchain-deep-agents.md), [126-frameworks-comparison.md](126-frameworks-comparison.md), [144-build-your-own-harness.md](144-build-your-own-harness.md), [145-comparing-coding-harnesses.md](145-comparing-coding-harnesses.md), [150-temporal-durable-execution-substrate.md](150-temporal-durable-execution-substrate.md), [151-memtier-why-flat-memory-breaks-at-72-hours.md](151-memtier-why-flat-memory-breaks-at-72-hours.md), [154-ctx2skill-self-evolving-context-skills.md](154-ctx2skill-self-evolving-context-skills.md), [155-feynman-multi-agent-research-harness.md](155-feynman-multi-agent-research-harness.md), [156-heavyskill-parallel-reasoning-deliberation.md](156-heavyskill-parallel-reasoning-deliberation.md), [157-may-2026-synthesis-memory-and-skills.md](157-may-2026-synthesis-memory-and-skills.md).
