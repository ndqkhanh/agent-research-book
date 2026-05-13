# 144 — Building Your Own Harness: A Decision Framework for When to Fork, Wrap, or Write From Scratch

**Source.** book2-comparing-en, Chapter 8 (If You Need to Build Your Own Harness); plus the broader meta-harness landscape ([66-meta-harness-landscape](66-meta-harness-landscape.md)) and the practical experience of teams who have shipped custom harnesses (Anthropic, Cursor, Devin, Replit Agent, Sourcegraph Cody).

**One-line definition.** Most teams should *not* build their own harness — they should adopt LangGraph + DSPy + a model gateway, or extend Claude Code / Cursor / OpenClaw — because building a harness from scratch is 6–12 months of foundational work before user-visible features ship; the framework lays out the four scenarios that *do* justify a custom harness (regulatory, fundamentally novel pattern, deep vertical specialisation, leverage at scale) and the practical decision rule (fork the closest match, wrap the rest, write only what you genuinely differentiate on).

## Why this matters

The instinct to build is strong; the cost is steep. A working harness has dozens of components: agent loop, tool registry, permission system, hooks, memory, observability, evaluation, verification, error recovery, prompt management, retry logic, model gateway, multi-tenancy, deployment infrastructure, UX layer. Each component has been built thousands of times; the marginal value of building it again is rarely positive.

For agent builders, the discipline is honest assessment of when custom is justified. Most teams answer "yes" reflexively; the right answer is "no" most of the time. The teams that ship products are usually the ones that adopted existing harnesses and focused on differentiating the user-facing layer.

This chapter is the decision framework. Four scenarios where building is right; the wrap-vs-fork-vs-build choice; the practical roadmap for teams that have decided to build; and the failure modes specific to home-grown harnesses.

## Problem it solves

Five concrete decision failures:

1. **Reflexive build.** Team builds custom because "we're special"; six months in, they've reinvented LangChain badly.
2. **Reflexive adopt.** Team adopts a framework that doesn't fit; constant friction; eventually rewrites anyway.
3. **Frankenstein stack.** Team mixes seven frameworks, glues them awkwardly; nobody can debug it.
4. **No exit strategy.** Team picks framework; later wants to migrate; lock-in is painful.
5. **Build vs feature.** Team builds harness components instead of features users will see.

Each is solved by deliberate decision-framework application.

## Core idea in one paragraph

The decision space is wider than "build or buy." It's *fork* (start from an existing harness, modify the parts that need to differ), *wrap* (use the harness as-is, build a thin layer on top), *write* (build the parts that differ, adopt the parts that don't), or *adopt* (use the harness wholesale). Most teams should adopt or wrap. A small minority — teams with regulatory requirements that frameworks don't meet, fundamentally novel agent patterns, deep vertical specialisation, or scale that earns the engineering investment — should fork or write. Within the "write" decision, the right approach is to write *only what you differentiate on*: the agent loop, the prompts, the user-facing UX. Don't reinvent permission systems, tool registries, observability, or model gateways unless you have a specific reason. The 80/20 rule applies sharply: 80% of the harness is generic, 20% is your differentiation; only build the 20%.

## Mechanism (step by step)

### 1. The four-option matrix

| Option | Effort | Lock-in | When to use |
|---|---|---|---|
| **Adopt** | Lowest | Moderate | Fits 80% of teams; framework matches needs |
| **Wrap** | Low-medium | Moderate | Need a thin layer over framework |
| **Fork** | High | Low (you own the fork) | Substantial customisation; fits ~10% of teams |
| **Write** | Highest | None (you own everything) | True differentiation; fits ~5% of teams |

The default should be **adopt**; escalate only with evidence.

### 2. The four scenarios that justify build/fork

**Scenario 1: Regulatory requirements.**
- Frameworks don't meet specific compliance needs (audit trail format, on-prem-only deployment, sovereign cloud, regulated-industry certifications).
- Building gives full control over compliance evidence.
- Examples: defence, intelligence, certain healthcare deployments.

**Scenario 2: Fundamentally novel agent pattern.**
- Your agent does something no existing framework supports.
- You're at the research frontier.
- Examples: AlphaEvolve ([85-alphaevolve](85-alphaevolve.md)), some specialised research labs.

**Scenario 3: Deep vertical specialisation.**
- Your agent is so domain-specific that the harness benefits from deep integration with domain primitives.
- Generic harnesses force compromise.
- Examples: medical imaging agents, drug-discovery loops, scientific simulation pipelines.

**Scenario 4: Leverage at scale.**
- You will run millions of agent tasks; small per-task efficiency gains compound to big savings.
- Custom harness optimisation > framework overhead.
- Examples: hyperscalers (Anthropic, OpenAI), companies with billions of agent tasks/year.

If none of these apply: don't build.

### 3. Within "build" — the 80/20 rule

For teams that decided to build, build *only what differentiates*:

| Component | Build? |
|---|---|
| Agent loop | Build (core logic) |
| Prompts | Build (core IP) |
| User-facing UX | Build (the product) |
| Tool registry | Adopt (MCP) |
| Permission system | Adopt or extend |
| Hooks | Adopt or extend |
| Memory store | Adopt (Postgres / vector / distributed SQL) |
| Observability | Adopt (LangSmith / Datadog / OpenTelemetry) |
| Evaluation harness | Build (your eval set is yours; harness is generic) |
| Model gateway | Build (multi-vendor, your routing logic) |
| Deployment | Adopt (Kubernetes / serverless) |

Differentiation is in the loop, prompts, and UX. The rest is infrastructure.

### 4. Wrap pattern — the most common right answer

```text
[your differentiation layer]
   ├── domain-specific prompts
   ├── domain-specific tools
   ├── product-specific UX
   ↓
[wrapper module]
   ↓
[adopted framework: LangGraph + DSPy + LangSmith + ...]
   ↓
[infrastructure: K8s, Postgres, vector DB, etc.]
```

The wrapper exposes your domain abstractions; everything below is the framework.

### 5. Fork pattern — own the modifications

```text
[your fork of OpenClaw / LangChain / Memento]
   ├── added: your domain primitives
   ├── removed: features you don't need
   ├── modified: behaviors that don't fit
   ↓
[your differentiation on top]
```

Fork costs:
- Maintenance burden (rebase against upstream).
- Loss of community support for modifications.
- Skill silo (your engineers know your fork, not the original).

Fork wins when modifications are substantial and ongoing.

### 6. Write pattern — green-field

```text
[your harness]
   ├── agent loop
   ├── tool registry (likely MCP-compatible)
   ├── permission system
   ├── hooks
   ├── memory
   ├── observability
   ├── evaluation
   ├── model gateway
   ├── prompts
   └── UX
```

Costs:
- 6–12 months minimum before user-visible features.
- All the maintenance.
- All the corner cases.

Wins when your differentiation is in the harness itself, not the application.

### 7. The exit-strategy question

For any choice, ask: "If we wanted to switch in 2 years, what's the cost?"
- Adopt with abstraction: low.
- Adopt without abstraction: high.
- Wrap: low.
- Fork: high (you own the fork).
- Write: low to swap out, but rarely justified to switch.

Build the abstraction layer ([147-vendor-lock-in](147-vendor-lock-in.md)) regardless of choice.

### 8. Hybrid pattern — mix-and-match

The 2026 production pattern in mature teams:

```text
[differentiation layer: agent loop + prompts + UX]
[orchestration: LangGraph]
[narrow components: DSPy]
[memory: Postgres or distributed SQL]
[vector: managed (Pinecone) or co-located (pgvector)]
[observability: LangSmith + Datadog]
[deployment: K8s + ArgoCD]
[CI/CD: GitHub Actions or similar]
```

Each layer adopted from its best-in-class. Custom only where genuinely needed.

### 9. Practical roadmap for "we're going to build"

Month 1–2: agent loop + tool registry + basic prompts. Get a single-task end-to-end.
Month 3–4: permission system + hooks + observability. Make it shippable.
Month 5–6: evaluation + memory + model gateway. Make it production-ready.
Month 7+: differentiation layer; UX; iteration.

If your milestones lag this by 50%+: stop and re-evaluate the build decision.

### 10. When to migrate from build to adopt

Three signals:
- Your custom harness is 80% generic infrastructure, 20% differentiation.
- Maintaining the harness is taking more engineering than building features.
- Hiring is harder because your custom harness has no community.

At those signals: migrate. Adopt LangGraph; preserve only your differentiation.

## Empirical anchors

- **6–12 months minimum** to build a working harness from scratch.
- **80% of harness functionality** is generic infrastructure across teams.
- **Most successful agent products** (Cursor, Replit Agent, Devin, Claude Code) wrote their harness; their differentiation is in the loop and the UX, not the infrastructure.
- **Most failed agent products** rebuilt infrastructure that frameworks already provide.
- **Teams that fork** typically maintain their fork ~3–5 years before re-adopting upstream.

## Variants and counter-arguments addressed

- **"Frameworks are too restrictive."** Most aren't, in 2026. LangGraph is highly flexible.
- **"We need control."** You have control of your differentiation layer; you don't need control of every primitive.
- **"Custom is faster."** It's slower. Benchmark: a 5-person team with LangGraph ships in months; with a custom harness ships in years.
- **"Adopting locks us in."** Less than custom locks you in to your own bugs.
- **"Frameworks don't perform."** They mostly do; the bottleneck is rarely framework overhead.

## Failure modes and limitations

1. **Reinvention.** Building something LangChain has had for 2 years.
2. **Maintenance debt.** Custom code accumulates; community has none of it.
3. **Hiring difficulty.** Engineers prefer to work with mature frameworks.
4. **Slow iteration.** Every harness improvement requires custom engineering.
5. **Gap with frontier.** Community frameworks adopt new patterns fast; your custom lags.
6. **Misjudged scale.** "We'll need leverage at scale" — but you don't actually run millions of tasks.
7. **Sunk-cost lock-in.** Once you've built, hard to abandon even when you should.

## When to use, when not

**Adopt** when there's no clear reason to build.

**Wrap** when you have differentiation but most components are generic.

**Fork** when modifications are substantial and ongoing; you accept the maintenance burden.

**Write** only with a clear case: regulatory, novel pattern, vertical specialisation, leverage at scale.

**Migrate to adopt** when the signals above appear.

## Implications for harness engineering

- **Default to LangGraph for the loop.** [126-frameworks-comparison](126-frameworks-comparison.md). It's the right starting point in 2026.
- **Build the differentiation layer.** Loop, prompts, UX.
- **Don't build infrastructure.** Adopt observability, deployment, model gateway.
- **Build evaluation.** Your eval set is your own; the harness can be generic.
- **Build with abstraction.** [147-vendor-lock-in](147-vendor-lock-in.md). Even with framework lock-in, abstract enough that migration is feasible.
- **Document the build decision.** Why we forked; why we wrapped. Future maintainers need this.
- **Re-evaluate annually.** Build decisions go stale. Was it worth it?
- **Track maintenance cost.** Engineering hours on harness vs features. Trends matter.

The one-sentence takeaway: **most teams should adopt or wrap an existing harness; building is justified only by regulatory requirements, fundamentally novel patterns, deep vertical specialisation, or true leverage at scale — and within "build", differentiate on loop+prompts+UX while adopting infrastructure.**

## See also

- [62-everything-claude-code](62-everything-claude-code.md), [29-dive-into-claude-code](29-dive-into-claude-code.md) — Claude Code as a custom harness reference.
- [52-dive-into-open-claw](52-dive-into-open-claw.md), [61-archon-harness-builder](61-archon-harness-builder.md) — alternative open-source harnesses.
- [66-meta-harness-landscape](66-meta-harness-landscape.md), [76-ten-links-synthesis](76-ten-links-synthesis.md) — landscape view.
- [126-frameworks-comparison](126-frameworks-comparison.md) — framework choice.
- [108-memento-codebase-mcp](108-memento-codebase-mcp.md) — Memento as a build reference.
- [40-harness-engineering-principles](40-harness-engineering-principles.md), [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md) — what a harness must provide.
- [145-comparing-coding-harnesses](145-comparing-coding-harnesses.md) — head-to-head of leading custom harnesses.
- [147-vendor-lock-in](147-vendor-lock-in.md) — abstraction strategy.
