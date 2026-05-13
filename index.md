---
title: Home
layout: default
nav_order: 1
description: "114 deep-dives on AI-agent harness engineering — from agent loops and skills to the May-2026 frontier of reasoning models, agentic RL, persistent memory, and computer-use agents."
permalink: /
---

# Agent Research Book
{: .fs-9 }

A curated collection of **114 deep-dives** on building reliable AI agents — drawn from 2024–2026 engineering blogs, arXiv papers, and production frameworks. The focus is balanced between **harness engineering** (the scaffolding around an LLM that turns it into an agent) and **agentic AI** (the reasoning, memory, and evaluation techniques the harness is built for).
{: .fs-6 .fw-300 }

[Read the interactive book](docs/index.html){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[Browse chapters by category](#chapters-by-category){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## What's inside

- **The harness layer** — agent loops, hooks, plan mode, permission modes, MCP, context compaction, memory files, sub-agents, verifier loops, observability.
- **Agentic reasoning** — ReAct, Reflexion, Tree of Thoughts / LATS, ReWOO, Plan-and-Solve, Chain-of-Verification, Voyager-style skill libraries.
- **Production reliability** — LLM-as-Judge, guardrails and prompt-injection defense, human-in-the-loop, agentic RAG.
- **The 2025–2026 research frontier** — 65+ paper-grounded chapters covering benchmarks, claude-code-era harness patterns, frameworks, models, safety, self-improving agents, domain specialists.
- **The May-2026 update (chapters 100–114)** — DeepSeek-R1's pure-RL reasoning recipe, multi-turn agentic RL (RAGEN, ARTIST, Agent Lightning), memory architectures (Mem0, Letta), the OS-level computer-use race, and the OSS-framework consolidation (OpenHands, smolagents, LangGraph, Magentic-One).

## Read it as an interactive book

[`docs/index.html`](docs/index.html) is a **single self-contained interactive book** of all 114 chapters, designed so non-technical readers can follow along. Each chapter opens with a one-sentence TLDR, a real-world analogy, a plain-English ELI5, key-numbers stat cards, a Mermaid diagram, glossary tooltips, and line-by-line code walkthroughs on the core chapters.

[Open the interactive book →](docs/index.html){: .btn .btn-purple }

---

## Chapters by category

### Part I — The harness (1–12)

The scaffolding around an LLM that turns it into an agent.

- [01 — Agent Loop Architecture](docs/01-agent-loop-architecture.md)
- [02 — Subagent Delegation](docs/02-subagent-delegation.md)
- [03 — Plan Mode](docs/03-plan-mode.md)
- [04 — Skills](docs/04-skills.md)
- [05 — Hooks](docs/05-hooks.md)
- [06 — Permission Modes](docs/06-permission-modes.md)
- [07 — Model Context Protocol (MCP)](docs/07-model-context-protocol.md)
- [08 — Context Compaction](docs/08-context-compaction.md)
- [09 — Memory Files](docs/09-memory-files.md)
- [10 — Multi-Session Continuity](docs/10-multi-session-continuity.md)
- [11 — Verifier / Evaluator Loops](docs/11-verifier-evaluator-loops.md)
- [12 — Todo / Scratchpad State](docs/12-todo-scratchpad-state.md)

### Part II — How agents think (13–20)

- [13 — ReAct](docs/13-react.md)
- [14 — Reflexion](docs/14-reflexion.md)
- [15 — Tree of Thoughts / LATS](docs/15-tree-of-thoughts-lats.md)
- [16 — Plan-and-Solve](docs/16-plan-and-solve.md)
- [17 — ReWOO](docs/17-rewoo.md)
- [18 — Chain-of-Verification & Self-Refine](docs/18-chain-of-verification-self-refine.md)
- [19 — Voyager-Style Skill Libraries](docs/19-voyager-skill-libraries.md)
- [20 — MetaGPT Role-Based Workflows](docs/20-metagpt-role-based-workflows.md)

### Part III — Production & reliability (21–25)

- [21 — LLM-as-Judge & Trajectory Evaluation](docs/21-llm-as-judge-trajectory-eval.md)
- [22 — Guardrails & Prompt-Injection Defense](docs/22-guardrails-prompt-injection.md)
- [23 — Human-in-the-Loop Approval](docs/23-human-in-the-loop.md)
- [24 — Observability, Tracing & Cost Attribution](docs/24-observability-tracing.md)
- [25 — Agentic RAG with Self-Critique](docs/25-agentic-rag.md)

### Part IV–VII — Modern research (26–99)

Sixty-plus chapters covering 2025–2026 papers, frameworks, and benchmarks. See the [docs/README.md](docs/README.md) for the full categorised index, including:

- Benchmarks (LinuxArena, HORIZON, ClawBench, OSWorld, GDPval, BrowseComp).
- Claude-Code-era harness patterns (Dive into Claude Code, harness-engineering principles, twelve harness patterns, four pillars, OpenClaw).
- Frameworks (LangChain Deep Agents, RAGFlow, LobeHub, DeerFlow, Archon).
- Self-improving agents (Autogenesis, Hyperagents, Hermes).
- Frontier papers (Lyra set: 22 paper-grounded deep-dives + synthesis).

### **Part VIII — May-2026 update (100–114)**

Fifteen new deep-dives covering the agentic-AI frontier. Read [114](docs/114-may-2026-landscape-update.md) first for the synthesis.

#### Reasoning models and agentic RL
- [100 — DeepSeek-R1: Pure RL for Reasoning](docs/100-deepseek-r1-rl-reasoning.md)
- [101 — RAGEN: Multi-Turn RL and the Echo Trap](docs/101-ragen-multi-turn-rl-agents.md)
- [102 — ARTIST: Agentic Reasoning + Tool RL](docs/102-artist-agentic-rl-tools.md)
- [103 — Agent Lightning: Train Any Agent with RL](docs/103-agent-lightning.md)

#### Memory and statefulness
- [104 — Mem0: Production-Ready Long-Term Memory](docs/104-mem0-production-memory.md)
- [105 — Letta: Stateful Persistent Agents](docs/105-letta-stateful-agents.md)

#### Computer-use and browser surfaces
- [106 — Computer-Use Agents (Anthropic vs OpenAI Operator)](docs/106-computer-use-agents.md)
- [107 — Browser-Use: DOM-Aware OSS Browser Agent](docs/107-browser-use.md)

#### Coding agents and frameworks
- [108 — OpenHands & CodeAct: Code-as-Action](docs/108-openhands-codeact.md)
- [109 — smolagents: HuggingFace's Code-Writing Agents](docs/109-smolagents.md)
- [110 — LangGraph: State-Graph Agent Framework](docs/110-langgraph.md)

#### Multi-agent and evaluation
- [111 — Magentic-One & Magentic-UI: Generalist Multi-Agent](docs/111-magentic-one.md)
- [112 — BrowseComp: Benchmark for Browsing Agents](docs/112-browsecomp.md)
- [113 — Test-Time Compute Scaling for Agents](docs/113-test-time-compute-agents.md)

#### Synthesis
- [114 — May-2026 Landscape Update](docs/114-may-2026-landscape-update.md)

---

## Companion material

The `harness-engineering/` folder contains higher-level synthesis documents that complement the chapter set:

- [SYNTHESIS.md](harness-engineering/SYNTHESIS.md) — cross-paper synthesis essay.
- [knowledge-graph.md](harness-engineering/knowledge-graph.md) — concept graph across the corpus.
- [HARNESS_SKILLS_INTEGRATION_PLAN.md](harness-engineering/HARNESS_SKILLS_INTEGRATION_PLAN.md) — implementation roadmap.
- [AI_FIRST_TRANSFORMATION_PLAN.md](harness-engineering/AI_FIRST_TRANSFORMATION_PLAN.md) — organisational adoption plan.
- [SOUL.md](harness-engineering/SOUL.md) — guiding principles.

## Reading orders

- **Newcomer:** Start with the [interactive book](docs/index.html). Read the cover, then chapters 1, 3, 4, 7 in order. The plain-English mode is on by default.
- **Harness engineer:** [01](docs/01-agent-loop-architecture.md) → [11](docs/11-verifier-evaluator-loops.md) → [40](docs/40-harness-engineering-principles.md) → [44](docs/44-four-pillars-harness-engineering.md) → [62](docs/62-everything-claude-code.md) → [114](docs/114-may-2026-landscape-update.md).
- **Researcher (frontier):** [99](docs/99-papers-deep-dive-synthesis.md) → [76](docs/76-ten-links-synthesis.md) → [114](docs/114-may-2026-landscape-update.md), then drill into the 100s.
- **Production builder:** [110-LangGraph](docs/110-langgraph.md), [108-OpenHands](docs/108-openhands-codeact.md), [104-Mem0](docs/104-mem0-production-memory.md), [22-guardrails](docs/22-guardrails-prompt-injection.md), [24-observability](docs/24-observability-tracing.md).

## License

Prose content is licensed under [CC BY 4.0](LICENSE). The build script (`docs/build_book.py`) is licensed under MIT — see header in the file.
