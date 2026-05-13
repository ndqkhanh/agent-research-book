# 126 — Frameworks Comparison: LangChain / LangGraph / Semantic Kernel / AutoGen / CrewAI / OpenAI Agents SDK / DSPy in 2026

**Sources.** Arsanjani & Bustos, *Agentic Architectural Patterns*, Chapter 15 (Agent Frameworks Use Case); Ozdemir, *Building Agentic AI*, Chapter 5; the *State of AI Agents* surveys from LangChain (2024–2025); plus public framework documentation as of 2026 — LangChain/LangGraph (Python+JS), Microsoft Semantic Kernel (Python+C#), Microsoft AutoGen, CrewAI, OpenAI Agents SDK, DSPy (Stanford), Llama-Index, Haystack.

**One-line definition.** This is the honest framework-comparison chapter — what each major agent framework is good at, where it fails, what the lock-in cost looks like, and which framework to default to in 2026 for which kind of project — written from the perspective of an engineer choosing for production rather than from any framework's marketing.

## Why this matters

Framework choice is a 2–5 year commitment. Migration cost at year 2 is high — not just code rewrite but also operational tooling, monitoring integrations, prompt-template repositories, and team mental models. Choosing well at year 0 saves a quarter of engineering effort over the agent's lifetime. Choosing poorly costs the same quarter, in remediation.

For agent builders in 2026, the question is no longer "should we use a framework?" — most teams should — but "which framework matches our team's constraints?" The answer depends on language preference, deployment posture (hosted vs self-hosted), agent complexity (workflow vs agent), team size, and how much control you want vs how much abstraction you want.

This chapter is the trade-off matrix without the marketing.

## Problem it solves

Five framework-choice mistakes:

1. **Pick by popularity.** Choose LangChain because everyone has heard of it; discover it's overkill for a workflow.
2. **Pick by vendor preference.** Choose OpenAI Agents SDK because the org uses OpenAI; lock yourself in.
3. **Pick by capability list.** Choose AutoGen because it has multi-agent; spend months fighting it for a single-agent use case.
4. **Pick by absence.** Reject all frameworks; build everything from scratch; reinvent every primitive.
5. **Mix everything.** Use LangChain for retrieval, LangGraph for orchestration, AutoGen for multi-agent, OpenAI SDK for tools — operational mess.

Each is solved by treating framework choice as a deliberate trade-off across known axes.

## Core idea in one paragraph

The 2026 agent framework landscape has converged into seven first-class options, each with a distinct fit profile. **LangChain / LangGraph** is the broad ecosystem — biggest community, most integrations, most patterns; LangGraph is the modern flow-graph layer that supersedes LangChain's older agent abstractions. **Semantic Kernel** is Microsoft's enterprise-grade option — strong .NET story, Microsoft ecosystem integration, conservative versioning. **AutoGen** is Microsoft's multi-agent research codebase — innovative but volatile. **CrewAI** is the role-based-pipeline-friendly option — opinionated, fast to ship, narrower in use case. **OpenAI Agents SDK** is OpenAI's first-party offering — simplest, most polished, locked to OpenAI. **DSPy** is the declarative-compilation option — best for narrow LLM-step optimisation, weaker for full agents. **Llama-Index / Haystack** are RAG-specialised but extending into agent territory. Default recommendation: LangGraph for flexibility, OpenAI SDK for OpenAI-only simplicity, DSPy for narrow optimised components, custom code for the agent loop. Most production stacks combine 2–3 of these; pure single-framework deployments are rare.

## Mechanism (per-framework breakdown)

### 1. LangChain / LangGraph

**What it is.** LangChain (2022+): the OG framework — components for prompts, models, tools, retrievers, memory, agents, callbacks. LangGraph (2024+): the modern flow-graph layer — explicit state machines, deterministic execution, async support.

**Strengths.**
- Largest community, most integrations (100+ models, 50+ vector stores, 100+ tools).
- LangGraph: the right abstraction for stateful agent workflows in 2026.
- LangSmith: production observability with strong agent-trace support.
- Ecosystem inertia: most patterns have a LangChain implementation.

**Weaknesses.**
- LangChain's older agent abstractions (`AgentExecutor`, `Tool`) are dated; LangGraph is the path forward but migration is non-trivial.
- API surface is huge; learning curve is real.
- Performance overhead vs custom code; usually negligible, sometimes meaningful.
- Frequent breaking changes historically; stabilising in 2025–2026.

**Right for.** Most production agents in 2026 — workflow + bounded agent regions, multi-vendor, observability-heavy.

**Wrong for.** Tiny prototypes (overkill); pure C# / .NET shops (use Semantic Kernel).

### 2. Microsoft Semantic Kernel

**What it is.** Microsoft's enterprise-grade orchestration framework — Python + C# + Java; integrates with Azure, Microsoft Graph, M365, Copilot Studio.

**Strengths.**
- Conservative versioning, enterprise-friendly.
- Strong .NET / C# support; Python is co-equal.
- Tight Microsoft ecosystem integration.
- Plugin model is clean; tools-as-plugins.

**Weaknesses.**
- Smaller community than LangChain.
- Microsoft-shaped opinions; less flexible if you're not in the MS ecosystem.
- Innovation pace slower than research-driven frameworks.

**Right for.** Enterprise .NET / Azure shops; teams that value stability over the leading edge.

**Wrong for.** Python-mostly shops where LangGraph is a more natural fit; teams wanting the latest research patterns.

### 3. Microsoft AutoGen

**What it is.** Microsoft Research's multi-agent framework — agents-as-conversational-actors, group-chat patterns, orchestrator + worker setups.

**Strengths.**
- Strong multi-agent abstractions; conversational agents work well.
- Active research output; new patterns shipped frequently.
- AutoGen Studio: visual designer.

**Weaknesses.**
- Volatile API; breaking changes frequent.
- Multi-agent abstractions can encourage diversity collapse without role asymmetry ([121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md), [98-diversity-collapse-mas](98-diversity-collapse-mas.md)).
- Less production-hardened than LangChain.

**Right for.** Multi-agent research, prototyping role-based pipelines.

**Wrong for.** Single-agent production work; teams wanting stability.

### 4. CrewAI

**What it is.** A role-based-pipeline-first framework — agents as "crew members" with roles, tools, goals; tasks orchestrated by a Crew object.

**Strengths.**
- Opinionated; fast to ship for the patterns it covers.
- Clean role-based-pipeline abstractions ([121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md)).
- Good docs and examples.

**Weaknesses.**
- Narrower use case; not great for hierarchical or peer-mesh patterns.
- Less mature than LangChain; smaller community.
- Performance overhead for simple cases.

**Right for.** Role-based-pipeline projects (analyst → architect → engineer → reviewer); teams wanting opinion not flexibility.

**Wrong for.** Single-agent work; complex multi-agent topologies.

### 5. OpenAI Agents SDK

**What it is.** OpenAI's first-party agent SDK (2024+) — clean abstractions for agents, tools, handoffs, tracing; tight integration with OpenAI features (function calling, Assistants API, structured outputs).

**Strengths.**
- Simplest, most polished SDK in the space.
- Integrates with all OpenAI features natively.
- Tracing built in.

**Weaknesses.**
- Locked to OpenAI; cross-vendor support is poor.
- Less flexibility than LangChain or DSPy.
- Younger; less battle-tested.

**Right for.** Teams committed to OpenAI; simple-to-medium agents; rapid prototyping.

**Wrong for.** Multi-vendor strategies; teams concerned about lock-in ([147-vendor-lock-in](147-vendor-lock-in.md)).

### 6. DSPy

**What it is.** Stanford's declarative LM-pipeline compiler — write task signatures and metrics, let the compiler optimise prompts (BootstrapFewShot, MIPROv2, etc.). Detailed in [93-dspy](93-dspy.md).

**Strengths.**
- Declarative: specify *what*, not *how*.
- Automatic prompt optimisation.
- Strong for narrow, well-specified subtasks.
- Pythonic, composable.

**Weaknesses.**
- Less suited for full agents (the planner needs flexibility DSPy doesn't compile well).
- Smaller community than LangChain.
- Optimization compute is real cost.

**Right for.** Narrow subtasks (classifiers, extractors, scorers) where automatic prompt optimisation is high-leverage. The right component for [111-prompt-engineering-as-discipline](111-prompt-engineering-as-discipline.md).

**Wrong for.** Full agent loops; ad-hoc prototyping.

### 7. Llama-Index / Haystack

**What it is.** Llama-Index (originally GPT Index): RAG-first framework with strong indexing, retrieval, and query patterns. Haystack: similar focus, more production-oriented from deepset.

**Strengths.**
- Best-in-class RAG abstractions.
- Strong indexing, hybrid retrieval, reranking.
- Now extending into agents (Llama-Index Agents, Haystack Agents).

**Weaknesses.**
- Agent abstractions less mature than dedicated agent frameworks.
- RAG-first DNA shows: agent loops feel bolted on.

**Right for.** RAG-heavy applications; document-Q&A platforms; knowledge-base agents.

**Wrong for.** Pure agent work where retrieval is incidental.

### 8. The decision matrix

| Use case | Framework |
|---|---|
| Stateful agent workflow, Python-shop | LangGraph |
| Enterprise .NET / Azure | Semantic Kernel |
| Multi-agent research / prototype | AutoGen |
| Role-based pipeline | CrewAI or LangGraph |
| OpenAI-only, simple | OpenAI Agents SDK |
| Narrow LLM-step optimisation | DSPy |
| Document Q&A / RAG-heavy | Llama-Index or Haystack |
| Custom production agent | LangGraph + DSPy + custom code |

For most production stacks in 2026: **LangGraph for the agent loop + DSPy for narrow components + custom code for the harness specifics**.

### 9. The hidden axis — observability

Frameworks differ in observability story:

| Framework | Native observability |
|---|---|
| LangChain/LangGraph | LangSmith (excellent) |
| Semantic Kernel | OpenTelemetry + Azure Monitor |
| AutoGen | minimal native; bring-your-own |
| CrewAI | Crew tracing + bring-your-own |
| OpenAI Agents SDK | OpenAI dashboard + traces |
| DSPy | minimal native; bring-your-own |
| Llama-Index/Haystack | bring-your-own |

For production: **observability is the deciding factor** as often as the agent abstractions themselves.

### 10. The lock-in axis

| Framework | Lock-in cost |
|---|---|
| LangChain/LangGraph | Moderate (broad abstractions; migration painful but feasible) |
| Semantic Kernel | Moderate (tied to MS but multi-language) |
| AutoGen | Moderate (Python only; abstractions migrate) |
| CrewAI | Moderate-high (opinionated abstractions) |
| OpenAI Agents SDK | High (provider locked) |
| DSPy | Low (compilation output is just prompts; portable) |
| Llama-Index/Haystack | Moderate |

DSPy has the lowest lock-in (its output is portable prompts); OpenAI Agents SDK has the highest (provider-bound).

## Empirical anchors

- **LangChain has 2–3× the community / integration footprint** of any competitor.
- **Most production stacks combine 2–3 frameworks** rather than committing to one.
- **DSPy gains** of 10–30% on narrow subtasks vs hand-engineered prompts.
- **OpenAI Agents SDK** lock-in is real; teams migrating cite 2–6 weeks of rework.
- **AutoGen breaking-change frequency** is high — 2024 stacks needed substantial 2025 rewrites.
- **Custom code for the agent loop** is the choice of mature teams who've outgrown framework abstractions; about 30% of stage-3+ agent platforms.

## Variants and counter-arguments addressed

- **"Frameworks slow you down."** Custom-everything is slower at production. Frameworks accelerate the 80% you don't differentiate on.
- **"All frameworks converge."** Surface API converges; opinions about state, control flow, observability remain different.
- **"Pick the most popular."** Popular ≠ right. LangChain is popular and right for many; not for all.
- **"Avoid lock-in by avoiding frameworks."** You're locked into your custom abstractions instead. Pick the lock-in deliberately.
- **"Use the simplest possible thing."** For prototypes, yes. For production, the framework's production features earn their complexity.
- **"DSPy will replace prompt engineering."** It replaces it for narrow tasks; the agent loop's prompt engineering remains manual.

## Failure modes and limitations

1. **Framework lock-in via accident.** Using framework-specific tracing IDs, prompt templates, tool schemas in ways that resist migration.
2. **Version churn.** Major framework upgrades mid-project break things. Pin versions; upgrade on cadence.
3. **Incomplete features.** Marketing pages oversell; some features only work for one model class. Read carefully.
4. **Performance overhead.** Naive framework usage adds 10–30% latency; profile and optimise hot paths.
5. **Community drift.** A "lively community" today can be stale tomorrow. Look at commit cadence + issue resolution.
6. **Multi-framework friction.** Combining frameworks needs careful boundary design; tracing across frameworks is non-trivial.
7. **Marketing-driven misconceptions.** Every framework claims "production-ready"; most aren't, by stage-3 standards.

## When to use, when not

**Use a framework** for nearly all stage-2+ agent work. The trade-off is real but framework benefits compound.

**Mix frameworks deliberately**: LangGraph for control flow + DSPy for narrow LLM steps is the dominant production stack.

**Custom code** for the agent loop when you've outgrown framework abstractions and have the team to maintain bespoke code.

**Avoid framework cargo-culting.** Match the framework to the project; "everyone uses LangChain" is not a project-fit argument.

## Implications for harness engineering

- **Treat framework choice as a 2–5 year commitment.** Migration is real cost; pick deliberately.
- **Prefer frameworks with strong observability.** [24-observability-tracing](24-observability-tracing.md) — without it, debugging is forensic.
- **Default to LangGraph for the agent loop** in Python shops in 2026. The abstractions are mature; ecosystem is strong.
- **Use DSPy for narrow optimised components.** [93-dspy](93-dspy.md), [111-prompt-engineering-as-discipline](111-prompt-engineering-as-discipline.md). Stack on top of LangGraph.
- **Multi-vendor strategy disqualifies OpenAI Agents SDK** as the sole framework. Use it for OpenAI-only paths, fall back to LangGraph for cross-vendor work.
- **Build the model gateway separately.** [119-agent-ready-llm-selection](119-agent-ready-llm-selection.md) — don't rely on the framework's vendor abstractions for production multi-vendor.
- **Pin framework versions.** Upgrade on a cadence (quarterly), test the upgrade in staging.
- **Document framework boundaries.** When mixing frameworks, where does control transfer? What's the data contract?
- **Consider vendor-neutral abstractions on top.** [147-vendor-lock-in](147-vendor-lock-in.md) — frameworks themselves can be the lock-in vector.

The one-sentence takeaway: **for most production stacks in 2026, the answer is LangGraph for the agent loop, DSPy for narrow LLM components, custom code where you differentiate, and a model gateway abstracting vendors — not a single framework that does everything.**

## See also

- [42-langchain-deep-agents](42-langchain-deep-agents.md) — LangChain's deep-agents pattern in detail.
- [52-dive-into-open-claw](52-dive-into-open-claw.md) — alternative open-source agent harness.
- [62-everything-claude-code](62-everything-claude-code.md) — Claude Code as a custom-built harness reference.
- [63-ragflow-agent-patterns](63-ragflow-agent-patterns.md), [64-lobehub-ai-framework](64-lobehub-ai-framework.md), [65-deer-flow-bytedance](65-deer-flow-bytedance.md) — additional framework analyses.
- [66-meta-harness-landscape](66-meta-harness-landscape.md) — landscape view.
- [93-dspy](93-dspy.md) — DSPy in depth.
- [108-memento-codebase-mcp](108-memento-codebase-mcp.md) — Memento as a custom-code reference implementation.
- [119-agent-ready-llm-selection](119-agent-ready-llm-selection.md), [147-vendor-lock-in](147-vendor-lock-in.md) — vendor strategy.
