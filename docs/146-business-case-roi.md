# 146 — Business Case & ROI for Agentic AI: Cost Models, Payback Windows, Risk-Adjusted Returns

**Sources.** Hodjat, *The Agentic Enterprise*, Chapter 2 (The Business Case for Agentic AI); plus the practitioner ROI literature on enterprise AI (McKinsey, BCG, Gartner reports), the FinOps Foundation's cost-management discipline, and the empirical productivity-gain measurements (GitHub Copilot studies, Cursor user surveys).

**One-line definition.** Building a defensible business case for agentic AI requires *quantified cost models* (LLM spend, infrastructure, engineering, ops), *quantified benefit models* (time saved, errors prevented, revenue lifted, customer experience improved), *risk-adjusted ROI* (probability of success × upside vs cost × probability of failure), and *payback-window discipline* (12 months for pilots, 24 for platforms, 36+ for enterprise transformation) — without all four, agentic-AI investments are bets, not decisions, and leadership eventually withdraws funding when the math is opaque.

## Why this matters

The 2024 enthusiasm for agentic AI made business cases hand-wave-able: "this is transformative, we have to invest." By 2026, finance teams demand the same rigor for AI as for any capital investment. Without a defensible business case, agentic-AI projects get cut at the next budget cycle — even successful ones, because no one has bothered to measure them.

For agent builders, business case discipline is what unlocks sustained funding. It also forces clarity about what success looks like; many failed projects fail because the success criteria were never specified, only assumed.

This chapter is the framework for building a defensible business case: the cost components, the benefit components, the risk adjustment, and the payback discipline. Written for engineers who need to justify their work to non-engineering stakeholders.

## Problem it solves

Five business-case failures common in agentic AI:

1. **Hand-wave benefits.** "It will save time" without quantifying how much, for whom, on what tasks.
2. **Hand-wave costs.** "We'll just use the GPT-4 API" without modelling at production volume.
3. **Optimistic single-point estimates.** Best-case ROI presented as expected ROI; reality disappoints.
4. **Ignored opportunity cost.** Engineers building this could be building something else.
5. **No measurement.** Project ships; no one tracks whether projected benefits materialised.

Each is solved by a structured business-case framework.

## Core idea in one paragraph

A defensible business case has four components. **Cost model**: total cost of the agent over a 24-month horizon, including LLM API spend, infrastructure (compute, storage, observability, eval), engineering FTE (build + maintenance), ops (deployment, monitoring, on-call), and indirect costs (training, change management). **Benefit model**: quantified gains tied to specific metrics — time saved per task × tasks per period × loaded labour cost; error reduction × cost per error; revenue lift × baseline revenue; customer satisfaction lift × retention impact. **Risk adjustment**: probability of achieving the projected benefit (often 30–70% for agentic projects); downside scenarios; the cost of failure. **Payback window**: 12 months for pilots, 24 for platforms, 36+ for enterprise transformation. Layered, this turns "we should do agentic AI" into "this specific project has $X net present value at Y% confidence with Z-month payback." Without the discipline, every project is on borrowed time.

## Mechanism (step by step)

### 1. Cost model components

| Component | Annual cost example |
|---|---|
| LLM API spend | $50K–$500K (volume-dependent) |
| Vector store / DB | $5K–$50K |
| Observability (Datadog, LangSmith) | $10K–$50K |
| Compute / hosting | $10K–$100K |
| Engineering FTE (build) | $300K–$1M (1–3 engineers) |
| Engineering FTE (maintenance) | $200K–$500K (ongoing) |
| Ops / on-call | $50K–$200K |
| Training / change management | $20K–$100K |
| Compliance / legal review | $30K–$200K (regulated industries) |
| **Total Year-1** | **$675K–$2.7M** |

Wide range; depends on scale, complexity, regulatory environment.

### 2. Cost-of-no-action

What does *not* doing the project cost?
- Lost productivity (status quo time spent on the task).
- Competitive disadvantage (others adopt; you don't).
- Recruiting cost (engineers want to work on AI; not having it is a recruiting headwind).
- Tech debt (manual processes don't scale; eventually break).

Often comparable to the project cost; this matters for the comparison.

### 3. Benefit model — quantification framework

For each benefit, specify:

```text
Benefit type: <time saved, error reduction, revenue lift, satisfaction>
Mechanism: <how the agent produces this benefit>
Measurement: <what metric, how often, by whom>
Baseline: <current value of the metric>
Target: <projected value with the agent>
Confidence: <high / medium / low>
```

Example:

```text
Benefit type: time saved
Mechanism: Agent handles 70% of tier-1 customer-support tickets without human intervention
Measurement: avg minutes per ticket, before/after, monthly
Baseline: 8 min/ticket × 50K tickets/mo = 400K min/mo = 6700 hr/mo
Target: 3 min/ticket × 50K tickets/mo = 150K min/mo = 2500 hr/mo
Saved: 4200 hr/mo × $40/hr loaded = $168K/mo = $2M/year
Confidence: medium (depends on automation rate)
```

Quantified, measurable, traceable.

### 4. Risk adjustment

Probability-adjusted NPV:

```text
Expected value = Σ (probability_i × outcome_i)

Scenarios:
  Best case (70% benefits realised): probability 30%, NPV $5M
  Base case (50% benefits realised): probability 50%, NPV $2M
  Worst case (20% benefits realised): probability 20%, NPV -$500K

Expected NPV = 0.3 × $5M + 0.5 × $2M + 0.2 × -$500K = $2.4M
```

This is the number to present.

### 5. Payback window discipline

| Project type | Acceptable payback |
|---|---|
| Pilot (one team, one workflow) | < 12 months |
| Platform (multiple teams) | < 24 months |
| Enterprise transformation | < 36 months |

Payback longer than this signals over-investment for current maturity ([118-genai-maturity-models](118-genai-maturity-models.md)).

### 6. ROI metrics

The right metrics:
- **Net Present Value (NPV)**: discount future cash flows to present.
- **Internal Rate of Return (IRR)**: implied annual return.
- **Payback period**: when does cumulative benefit = cost?
- **Risk-adjusted NPV**: probability-weighted.
- **Cost per outcome**: e.g. cost per ticket resolved.

Present multiple metrics; finance teams have preferences.

### 7. Common benefit categories

**Time saved**: tasks completed faster; engineers / analysts / customer service.
**Error reduction**: fewer mistakes × cost per mistake (often the largest in regulated industries).
**Revenue lift**: better recommendations, conversion, upsell.
**Customer experience**: NPS lift, retention improvement.
**Capacity unlocked**: handle more volume without hiring.
**Cost avoidance**: replace external vendor, avoid hire, defer system upgrade.

Quantify each; not all apply to every project.

### 8. Common cost surprises

**LLM cost at production volume**: 10× pilot estimate.
**Engineering maintenance**: ongoing cost often 30–50% of initial build.
**Compliance**: especially in regulated industries; can dwarf engineering.
**Change management**: training users, updating processes; often underestimated.
**Eval data curation**: ongoing.
**Vendor outages**: secondary costs (downtime, customer impact).

Surface these in the cost model.

### 9. Measurement infrastructure

Without measurement, ROI is fiction:
- **KPIs defined upfront**: per benefit category.
- **Dashboards**: live tracking.
- **Quarterly reviews**: actual vs projected.
- **Attribution clear**: this gain came from this agent.

Without this, leadership cuts the project at the next budget cycle.

### 10. Presenting the case

Structure for leadership:

```text
1. The opportunity (business problem).
2. The proposal (what we'll build).
3. The investment (cost model, year by year).
4. The return (benefit model, year by year, by category).
5. The risk (probability-adjusted scenarios).
6. The payback (cumulative cash flow over 24 months).
7. The ask (funding, headcount, timeline).
8. The measurement plan (KPIs, dashboards, review cadence).
```

Land all eight; leadership understands and funds.

## Empirical anchors

- **GitHub Copilot productivity studies**: ~30% faster on common coding tasks.
- **Cursor user surveys**: 1–3 hour daily savings per developer.
- **Customer-support automation**: 50–70% deflection rates achievable.
- **ROI of AI projects**: median 2–3× in 24 months among successful; many fail to break even.
- **Failure rate**: 30–50% of AI projects don't reach production; budget for this.
- **Payback windows**: 12–24 months typical; > 36 months projects often get cut.

## Variants and counter-arguments addressed

- **"AI is too important to wait for ROI."** Wrong. Without ROI, projects get cut.
- **"The benefits are intangible."** Then quantify them via proxies (NPS, retention).
- **"Models change too fast for long-term ROI."** Even more reason for short payback.
- **"Just use vendor RAG / Copilot."** Sometimes right; the make-vs-buy is part of the case.
- **"Engineering productivity is hard to measure."** It is; use proxies (cycle time, defect rate, satisfaction).

## Failure modes and limitations

1. **Optimism bias.** Projected benefits at best-case; actual at base-case.
2. **Hidden costs.** Maintenance, change management, compliance often missing.
3. **Single-axis benefit.** Only counting time saved; ignoring error reduction.
4. **No counter-factual.** What would have happened without the project?
5. **Attribution gaps.** Which benefit came from which intervention?
6. **Goodhart's law on metrics.** Optimising the metric until it stops measuring what mattered.
7. **Stale ROI.** Project succeeds; nobody updates the projection; benefits unmeasured.
8. **Sunk-cost rationalisation.** Project failing; team rationalises continuation.

## When to use, when not

**Use a defensible business case for** any agentic-AI project requesting funding, headcount, or sustained engineering investment.

**Skip elaborate ROI for** experimental prototypes (stage 1, [118-genai-maturity-models](118-genai-maturity-models.md)) where the goal is learning, not value.

**Re-validate annually.** Projections go stale; refresh.

## Implications for harness engineering

- **Build measurement infrastructure with the project.** Without it, you cannot demonstrate ROI.
- **FinOps from day one.** [125-system-level-production-patterns](125-system-level-production-patterns.md). Per-task cost attribution.
- **Eval-data curation** is a measurable activity ([115-evaluating-llm-systems](115-evaluating-llm-systems.md)) — count toward project cost.
- **Maintenance budget**: 30–50% of build budget, ongoing.
- **Document baseline rigorously.** Without baseline, you can't measure lift.
- **Share the case with engineers.** Engineers who understand the case build the right thing.
- **Quarterly review cadence.** Project lives or dies on these.
- **Outcome attribution.** Track which agent / feature drove which gain. Without attribution, future cases don't have data.

The one-sentence takeaway: **a defensible business case has cost model, benefit model, risk adjustment, and payback window — without all four, agentic-AI projects are bets that get cut at the next budget cycle, even successful ones.**

## See also

- [118-genai-maturity-models](118-genai-maturity-models.md) — maturity stage shapes ROI horizon.
- [125-system-level-production-patterns](125-system-level-production-patterns.md) — FinOps as engineering discipline.
- [115-evaluating-llm-systems](115-evaluating-llm-systems.md) — measurement infrastructure.
- [122-explainability-compliance](122-explainability-compliance.md) — compliance as a cost component.
- [123-robustness-fault-tolerance](123-robustness-fault-tolerance.md) — production reliability as cost-of-quality.
- [144-build-your-own-harness](144-build-your-own-harness.md) — build vs adopt has ROI implications.
- [147-vendor-lock-in](147-vendor-lock-in.md) — strategic risk component of ROI.
- [149-sector-use-case-catalog](149-sector-use-case-catalog.md) — sector-specific ROI patterns.
