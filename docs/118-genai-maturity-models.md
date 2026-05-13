# 118 — GenAI Maturity Models: From Pilots to Production, and the Five-Stage Ladder Organisations Climb

**Sources.** Arsanjani & Bustos, *Agentic Architectural Patterns for Building Multi-Agent Systems*, Chapter 1 (GenAI in the Enterprise: Landscape, Maturity, and Agent-Readiness) — the canonical source of the GenAI Maturity Model — and Chapter 12 (Implementation Maturity Levels); Hodjat, *The Agentic Enterprise*, Chapter 5 (Core Capabilities and Technical Foundations); plus the broader maturity-model literature (Capability Maturity Model Integration, IT4IT, Cloud Adoption Frameworks).

**One-line definition.** A maturity model is a five-stage ladder — *experimentation → controlled pilots → scaled production → integrated platform → enterprise-wide agentic operation* — that names where an organisation is on the GenAI/agentic journey, what capabilities it must build to advance, and which classes of project it can responsibly take on at each stage; treating maturity explicitly turns "should we build agents?" from a yes/no question into a *which stage are we at, and what's the next investment?*.

## Why this chapter matters

In 2024–2025, almost every organisation that experimented with LLMs hit the same wall: a Friday-afternoon prototype impresses leadership, then takes 18 months to ship to production with monitoring, security, governance, evaluation, and reliability. The wall is the same one ML platforms hit a decade earlier and that microservices hit before that. Maturity models exist because the wall is real, predictable, and structurally identical across organisations.

For agent builders, the maturity model is two things: (1) a diagnostic that tells you why your project is stuck — because you're trying to do stage-3 work in a stage-1 organisation — and (2) a roadmap that tells you what to invest in next. Without the model, every team reinvents the same staircase, falling off the same steps.

This chapter is the canonical five-stage ladder, what each stage requires, what mistakes are typical at each stage, and how to use the ladder to scope what your team can responsibly ship.

## Problem it solves

Five recurring organisational dysfunctions the maturity model diagnoses:

1. **Pilot purgatory.** The org has built 20 prototypes; none have shipped. Diagnosis: stage-1 capability, attempting stage-3 deployment.
2. **Reliability cliff.** A demo agent that worked in dev fails in production. Diagnosis: missing stage-2 disciplines (eval, monitoring, error recovery).
3. **Scaling pain.** One agent works; trying to support 50 use cases falls over. Diagnosis: missing stage-3 platform (shared services, governance, reusable components).
4. **Governance whiplash.** Compliance freezes the project after launch. Diagnosis: stage-3 deploy with no stage-3 governance — explainability, audit, policy enforcement.
5. **Strategic drift.** Each team picks a different framework, vendor, model. Diagnosis: missing stage-4 architectural alignment.

Each dysfunction is fixable, but only by identifying the stage gap and investing in the missing capabilities.

## Core idea in one paragraph

Organisations adopting agentic AI move through five stages. **Stage 1 (Experimentation)** runs prototypes; success criterion is "did the demo work?" **Stage 2 (Controlled Pilots)** ships small projects to limited users; success requires evaluation discipline, basic observability, and a rollback path. **Stage 3 (Scaled Production)** runs a portfolio of agents in production; success requires shared platform services (model gateway, observability, eval, security, governance) and standardised patterns. **Stage 4 (Integrated Platform)** treats agentic capability as enterprise infrastructure; success requires architectural alignment, vendor-neutrality, and reuse across teams. **Stage 5 (Agentic Operation)** has agents in the operational loop of the business; success requires trust, regulatory compliance, and auditable autonomy. Most organisations in 2026 are at stage 1 or 2; ambitious ones at stage 3; very few at stage 4+. Each stage requires the prior stage's foundations; skipping is the most common failure mode.

## Mechanism (step by step)

### 1. The five stages in detail

**Stage 1 — Experimentation.** Individual contributors or small teams build prototypes. No production deploy. Success = "the demo works." Capabilities required: model API access, basic prompt engineering, dev sandboxes. Typical artifacts: notebooks, demos, slide decks. Time horizon: weeks. Risk: low (no production exposure). Common stuck point: prototypes pile up, none ship.

**Stage 2 — Controlled Pilots.** A single agent ships to a limited user group (internal beta, friendly customer). Success = "real users accept results, with HITL." Capabilities required: evaluation harness, basic observability, HITL approval flows, rollback. Typical artifacts: shipped pilot, eval dashboards, incident runbooks. Time horizon: 1–6 months per pilot. Risk: moderate. Common stuck point: pilot succeeds but cannot scale because each subsequent pilot rebuilds the same plumbing.

**Stage 3 — Scaled Production.** A portfolio of agents runs in production across teams. Success = "platform supports many agents reliably." Capabilities required: shared model gateway, central observability, agent harness library, security review, governance policy. Typical artifacts: platform team, agent catalogue, golden-path templates. Time horizon: 1–2 years to reach. Risk: significant (production exposure × portfolio). Common stuck point: governance cannot keep up with proliferation; projects slow to crawl.

**Stage 4 — Integrated Platform.** Agentic capability is enterprise infrastructure. Success = "any team can ship an agent in weeks, with safety and observability built in." Capabilities required: vendor-neutral abstractions ([147-vendor-lock-in](147-vendor-lock-in.md)), framework standardisation, FinOps for AI cost control, comprehensive eval-as-a-service, reusable patterns ([121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md)). Typical artifacts: internal developer portal, agent SDK, golden patterns library. Time horizon: 2–3 years. Risk: managed. Common stuck point: organisational alignment — different business units want different stacks.

**Stage 5 — Agentic Operation.** Agents are in the operational loop of the business — making decisions, taking actions, learning over time. Success = "the business runs partly via agents, with humans in oversight roles." Capabilities required: continuous evaluation in production, regulatory compliance, auditable autonomy, automated rollback, human-AI teaming protocols. Typical artifacts: agent-augmented business processes, AI ethics committees, agentic-operation playbooks. Time horizon: 3–5+ years. Risk: high (AI-driven business outcomes). Common stuck point: trust is the bottleneck, not technology.

### 2. The capability matrix per stage

| Capability | Stage 1 | Stage 2 | Stage 3 | Stage 4 | Stage 5 |
|---|---|---|---|---|---|
| Model API access | ✓ | ✓ | ✓ | ✓ | ✓ |
| Prompt engineering | basic | disciplined | optimised | platform | continuous |
| Evaluation | manual | offline | offline+online | full + judge | continuous in prod |
| Observability | none | basic | central | enterprise | regulatory |
| Security | dev-only | pilot-grade | prod-grade | platform | compliance-grade |
| Governance | none | informal | policy | committee | auditable |
| Platform | individual | per-project | shared | enterprise | integrated |
| HITL | none | always | configurable | optional | exception-only |
| Cost management | unmanaged | tracked | optimised | FinOps | optimised at edge |

Each row is a capability that must reach a stage-appropriate maturity before the org can responsibly attempt that stage's projects.

### 3. The diagnostic — where are we?

Three questions name your stage:

1. **Have you shipped agentic projects to production?** No → stage 1; yes → stage 2+.
2. **How many agentic projects in production, and do they share infrastructure?** Few, no shared infra → stage 2; many, shared infra → stage 3+.
3. **Can a new team ship an agent in weeks using golden-path patterns?** No → stage 3; yes → stage 4+.

The three questions take 5 minutes. The investment-roadmap conversation that follows is what the organisation actually needs.

### 4. Stage-appropriate project scoping

Each stage has projects it can responsibly take on:

- **Stage 1**: prototypes, demos, internal experiments. *Not* production.
- **Stage 2**: customer-facing pilots with explicit user opt-in and HITL.
- **Stage 3**: production agents at scale with monitoring and governance.
- **Stage 4**: platform-level features (e.g., AI assistant in every product).
- **Stage 5**: business-critical agentic operations (e.g., autonomous trading, autonomous customer support).

A stage-1 organisation that scopes a stage-3 project is the most common failure pattern in agentic AI.

### 5. Advancement requires capabilities, not pilots

The single biggest organisational mistake is *building more pilots* when stuck at stage 1 or 2. More pilots don't advance maturity; *capabilities* advance maturity. To go from stage 1 → 2, build evaluation and observability. To go from stage 2 → 3, build a platform team and shared services. To go from stage 3 → 4, build vendor-neutral abstractions and golden-path templates.

The progression is investment in *non-feature work*: platform, evaluation, governance, infrastructure. Most organisations underfund this.

### 6. Per-stage anti-patterns

**Stage 1 anti-patterns**: shipping prototypes to production; promising demos as products.
**Stage 2 anti-patterns**: scaling without platform; rebuilding plumbing per project.
**Stage 3 anti-patterns**: governance bypass; vendor lock-in via accident.
**Stage 4 anti-patterns**: over-standardisation that kills team innovation; central platform team becomes a bottleneck.
**Stage 5 anti-patterns**: agentic operation without explicit human oversight; autonomy creep.

### 7. The Agentic AI Maturity Spectrum (Arsanjani Ch.3)

Arsanjani also names a separate *agentic AI* maturity spectrum within stages 3–5:

- **Manual orchestration**: human chains LLM calls.
- **Workflow with LLM steps**: deterministic pipeline with LLM nodes ([114-workflows-vs-agents](114-workflows-vs-agents.md)).
- **Bounded agents**: LLM picks next step within a fixed tool set and budget.
- **Adaptive agents**: agents learn over time ([14-reflexion](14-reflexion.md), [106-memento-paper-theory](106-memento-paper-theory.md)).
- **Autonomous multi-agent systems**: agents coordinate, learn, and operate without human-in-the-loop.

This spectrum is *within* the GenAI maturity model, not orthogonal. A stage-3 organisation is typically running bounded agents; stage-5 organisations attempt adaptive multi-agent.

## Empirical anchors

- **Most organisations are at stage 1 or 2** in 2026; surveys (LangChain *State of AI Agents* 2024–2025; McKinsey 2025) consistently report < 30% have agents in production.
- **Pilots-to-production ratio** is typically 10:1 or worse — for every shipped agent, 10 prototypes died.
- **Governance is the #1 stuck point at stage 3**; security review and explainability requirements halt projects.
- **Cost surprises** at stage 3 are typical when FinOps practices haven't been built.
- **Time to stage-4 maturity** is usually 2–3 years of deliberate investment; organisations that try to shortcut typically end up at "stage 3 with stage 1 governance" — not safer, just disorganised.

## Variants and counter-arguments addressed

- **"Maturity models slow innovation."** They name reality. Organisations move through these stages whether or not they call them that; naming makes the path explicit.
- **"Our org is special."** Some details vary; the structural progression is universal.
- **"We can skip stages."** No team has demonstrated this. The capabilities are cumulative; a stage-3 deployment without stage-2 evaluation is not deployment, it's gambling.
- **"This is too generic."** The generality is the point — it applies across teams, projects, and verticals.
- **"Too prescriptive."** The model names the *capabilities* you need, not how to build them. Many implementations possible.

## Failure modes and limitations

1. **Mis-diagnosing the stage.** Self-assessment is biased toward higher stages. Use external benchmarking when possible.
2. **Cargo-culting capabilities.** Building "platform" without users is wasted; capability investment must follow demand, not lead it too far.
3. **Over-standardisation early.** Stage-2 organisations standardising prematurely kill innovation; stay flexible until volume justifies platform investment.
4. **Under-standardisation late.** Stage-4 organisations without standardised patterns are paying for the same plumbing many times.
5. **Skipping evaluation.** The most common shortcut; the most expensive in retrospect.
6. **Compliance afterthought.** Adding governance after launch is more expensive than designing for it.
7. **Treating stages as years.** A small team might cover stage 1–2 in months; an enterprise will take longer. Calibrate to organisation size.

## When to use, when not

**Use the model for** strategic planning, investment prioritisation, project scoping, organisational diagnosis, communicating to leadership.

**Skip it when** you are an individual developer or two-person team — the model is overkill at that scale.

**Use specific stages' capabilities lists** as a checklist when planning the next investment.

## Implications for harness engineering

- **Build capability for the next stage, not three stages out.** Premature platform investment kills momentum; under-investment caps growth. Match capability building to demonstrated demand.
- **Prioritise evaluation and observability above almost everything else** in stage 1 → 2 transition. They unlock everything downstream. See [115-evaluating-llm-systems](115-evaluating-llm-systems.md), [24-observability-tracing](24-observability-tracing.md).
- **A central platform team appears at stage 3.** Plan for it; don't hire reactively. The team owns the model gateway, the eval-as-a-service, the harness library, the agent SDK.
- **Standardise patterns at stage 4, not earlier.** [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md), [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md) become enforceable golden paths.
- **Vendor-neutral abstractions emerge at stage 4.** See [147-vendor-lock-in](147-vendor-lock-in.md). Investing earlier is over-engineering; investing later is rework.
- **Stage-5 trust requires regulatory-grade audit.** Plan for it from stage 3 if your domain is regulated. See [122-explainability-compliance](122-explainability-compliance.md).
- **Maturity is asymmetric across capabilities.** An org may be stage-3 on infrastructure but stage-1 on evaluation; the lagging capability is the binding constraint.
- **Use stage as a project gating tool.** Match project ambition to organisational stage; don't let leadership commitments outrun capability.

The one-sentence takeaway: **maturity advances through five stages, each requiring capability investment in non-feature areas (platform, evaluation, governance), and the most expensive mistake is scoping projects two stages above where your organisation actually is.**

## See also

- [40-harness-engineering-principles](40-harness-engineering-principles.md), [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md), [66-meta-harness-landscape](66-meta-harness-landscape.md) — engineering disciplines that mature stage by stage.
- [115-evaluating-llm-systems](115-evaluating-llm-systems.md) — the evaluation discipline gating stage 1→2.
- [122-explainability-compliance](122-explainability-compliance.md) — governance for stages 3–5.
- [125-system-level-production-patterns](125-system-level-production-patterns.md) — the platform patterns of stages 3–4.
- [126-frameworks-comparison](126-frameworks-comparison.md), [147-vendor-lock-in](147-vendor-lock-in.md) — vendor-neutral abstractions emerging at stage 4.
- [146-business-case-roi](146-business-case-roi.md) — the financial framing leadership uses to fund stage transitions.
