# 278 — Agent Unit Economics 2026: per-token, per-task, per-feature, per-customer

**Anchors.** Tokens-per-dollar landscape May 2026 — frontier API tiers, OSS distilled tiers, local-first amortized. Companions: [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md), [88-confidence-driven-router](88-confidence-driven-router.md), [146-business-case-roi](146-business-case-roi.md), [147-vendor-lock-in](147-vendor-lock-in.md), [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [235-inference-compression-scaling](235-inference-compression-scaling.md), [275-agent-finetuning-2026](275-agent-finetuning-2026.md), [276-local-first-privacy-first-agents](276-local-first-privacy-first-agents.md).

**One-line definition.** A 2026 picture of **agent unit economics** — the field has converged on **four cost layers** (per-token, per-task, per-feature, per-customer) — with frontier-API per-million-token costs **falling ~10× year-over-year** (GPT-4 $30/M in 2023 → GPT-4o $5/M in 2024 → Claude Sonnet $3/M in 2025 → DeepSeek-V3 $0.27/M in 2025–2026), **distilled OSS models on consumer hardware** at **$0 marginal cost** ([276](276-local-first-privacy-first-agents.md)), **prompt caching** giving 5–10× per-task reduction ([274](274-prompt-and-context-engineering-2026.md)), **routing** giving 3–10× per-task reduction by tier-matching ([86](86-frugalgpt.md), [87](87-routellm.md)), and **distillation** giving 50–100× per-feature reduction by replacing API with local model — combined into a 2026 unit-economics calculus where **the right combination of cheaper models + caching + routing + distillation makes agent products economically viable at scale** that wouldn't have been possible 18 months ago.

## Why this matters (unit economics determine whether agents become products vs demos)

Through 2024 the "is this agent product viable" question was answered by **"will the model-cost-per-task drop enough"** — implicit hope in continued price declines. By 2026 that question has a structured answer: **four cost layers with explicit levers per layer**, and disciplined teams can ship **profitable agent products today** at price points unimaginable 18 months ago. The underlying drivers: (a) **frontier API price compression** — DeepSeek-V3 and Gemini Flash and Claude Haiku at $0.10–0.50/M input tokens, GPT-4-class capability at $0.30–3/M; (b) **OSS distillation** — R1-Distill-Qwen-32B running on a consumer M4 Max at zero marginal cost; (c) **prompt caching** — 5–10× cost cut for repeat-prefix workloads; (d) **routing** — easy queries → cheap tier, hard queries → expensive tier, 3–10× aggregate savings; (e) **finetune + distill** — domain-specialized model at 50–100× cheaper than frontier API.

The four cost layers and their levers:

| Layer | Unit | Levers | Typical reduction |
|---|---|---|---|
| **Per-token** | $/M tokens | Model choice, quantization, caching | 30-90% |
| **Per-task** | $/task | Prompt engineering, output budget, streaming | 30-70% |
| **Per-feature** | $/feature/user/month | Distillation, caching, batch | 50-95% |
| **Per-customer** | $/customer/month | Free-tier design, premium tiering, retention | varies |

Each layer is independently optimizable; the multiplicative effect is what makes products viable.

## Core idea

Agent unit economics in 2026 is **a four-layer optimization** on a stack of cost components. **Layer 1** picks the model — frontier ($3-15/M), mid-tier ($0.30-3/M), small ($0.05-0.30/M), distilled-local ($0). **Layer 2** structures the call — prompt caching, output budgets, structured outputs. **Layer 3** routes by difficulty — cheap tier for easy, mid for medium, frontier for hard. **Layer 4** invests in distillation — train (or distill) a domain-specialized model that replaces frontier on the most-common task family. Combined: frontier-API direct = $1-5/task; with caching + routing = $0.30-1.50/task; with distill = $0.02-0.10/task; with full local-first = $0 marginal. The discipline turns agent products from "venture-capital-burn" into "positive unit economics".

## Mechanism (step by step)

### (a) Layer 1 — frontier API price landscape (May 2026, $/M tokens)

| Tier | Model | Input | Output | Notes |
|---|---|---:|---:|---|
| **Frontier-thinking** | o3 | ~$15 | ~$60 | Highest |
| **Frontier** | Claude Opus 4.6 | $15 | $75 | |
| **Frontier** | GPT-4 Omni | $5 | $15 | |
| **Frontier** | Claude Sonnet 4.6 | $3 | $15 | Sweet spot |
| **Frontier** | Gemini 2.5 Pro | $1.25 | $5 | Long-context strong |
| **Mid-tier** | Claude Haiku | $0.80 | $4 | Fast |
| **Mid-tier** | GPT-4o-mini | $0.15 | $0.60 | Very cheap fast |
| **OSS-frontier** | DeepSeek-V3 (api) | $0.27 | $1.10 | Frontier-class at mid-tier price |
| **OSS-frontier** | DeepSeek-R1 (api) | $0.55 | $2.19 | Thinking class |
| **OSS-distilled** | Qwen-2.5-72B (Together / Fireworks) | $0.60 | $0.60 | |
| **OSS-distilled** | R1-Distill-Qwen-32B | $0.30 | $0.90 | |
| **Local** | R1-Distill-Qwen-32B (M4 Max) | $0 | $0 | Hardware amortized |

The **DeepSeek pricing inflection** (Dec 2024 → 2026) reset the floor: frontier-class at mid-tier price. Cloud providers (Anthropic, OpenAI, Google) responded with aggressive price cuts.

### (b) Layer 1 levers — model choice

| Lever | Saving |
|---|---:|
| Frontier → mid-tier on easy tasks | 5-10× |
| FP16 → INT4 quantization | 2× |
| Hosted → distilled OSS | 5-50× |
| Distilled OSS → local | ∞ (zero marginal) |
| Speculative decoding | 2-4× wall-clock; cost-neutral or favorable |

### (c) Layer 2 — per-task cost structure

```
Cost(task) = input_tokens × $/M_input + output_tokens × $/M_output
           - cache_hits × cache_savings
```

Levers:
- **Prompt caching**: 5–10× cost reduction on cached prefix.
- **Output budget**: cap response tokens to actual need.
- **Structured output**: 2-3× shorter than free-form.
- **Streaming with early termination**: stop generation on first valid output.
- **Tool result truncation**: paginate; don't dump.

Example: task with 50K input + 2K output:
- Naive: 50K × $3/M + 2K × $15/M = $0.15 + $0.03 = $0.18
- + caching (45K cached): 5K × $3/M + 45K × $0.30/M + 2K × $15/M = $0.015 + $0.0135 + $0.03 = **$0.06** (3× cheaper)

### (d) Layer 3 — routing by difficulty

```python
class CostRouter:
    def pick_tier(self, prompt: str, difficulty_hint: float) -> str:
        if difficulty_hint < 0.3:
            return "haiku"        # $0.80/M
        elif difficulty_hint < 0.7:
            return "sonnet"       # $3/M
        elif difficulty_hint < 0.9:
            return "opus"         # $15/M
        else:
            return "thinking"     # $15-60/M
```

[88-confidence-driven-router](88-confidence-driven-router.md) measures difficulty via cheap probe; [87-routellm](87-routellm.md) provides the framework. Empirical: routing typically saves 3-10× vs always-frontier for typical traffic.

### (e) Layer 4 — distillation as cost engineering

Train domain-specialist via [275-agent-finetuning-2026](275-agent-finetuning-2026.md):
- Frontier API for top 5% hardest tasks.
- Distilled local model for 95% of typical traffic.
- Net: ~5% × frontier-cost + 95% × $0 = ~$0.01/task vs $0.18/task frontier-only.

Up-front: $30K finetune + $5K hardware. Payback at $30K + $5K / $0.17 saving × 1M queries = ~200K queries/month. For high-volume agents this is months, not years.

### (f) Per-feature economics

For SaaS agent features, cost per (feature × user × month):

```
Cost = monthly_invocations × cost_per_task
```

A feature invoked 100×/user/month at $0.06/task = $6/user/month. With distillation: $0.01/task = $1/user/month. Pricing tier strategy: $20/user/month consumer SaaS leaves comfortable margin even on multi-feature deployments.

### (g) Per-customer economics

Free tier limits define burn:
- 100 free invocations × $0.06 = $6 LTV cost per free user.
- Conversion to paid at $20/month = $14 net contribution.
- Conversion rate × ($14 net) > free-user cost ⇒ profitable.

Premium-tier features (frontier model + thinking + multi-agent) priced higher to cover their cost.

### (h) Vendor lock-in cost

[147-vendor-lock-in](147-vendor-lock-in.md) — switching vendors is operational cost (re-eval, re-prompt, re-test). LiteLLM proxy + provider-agnostic prompts reduce switching cost to ~1 engineer-week. Distillation to local model is the strongest lock-in defense.

### (i) Cost observability

[264-agent-observability-stack-2026](264-agent-observability-stack-2026.md) — per-span cost attribution; per-routine / per-user / per-feature dashboards. Real-time visibility is required for unit-economics discipline.

### (j) Pricing model alignment

| Pricing model | Cost structure that fits |
|---|---|
| Per-invocation | Fixed cost agents; predictable per-task |
| Per-month subscription | Mixed-volume; amortize with caching |
| Usage-based with cap | Long-tail high-volume; protects margin |
| Freemium | Distill + cache + cheap tier on free; frontier on paid |
| Enterprise | Per-seat with bundled compute; predictable for buyer |

## Empirical results (table — May 2026 product economics)

**Hypothetical "code-review SaaS" agent at $20/user/month:**

| Configuration | Cost per review | Reviews/user/month | Cost/user | Margin |
|---|---:|---:|---:|---:|
| Frontier API direct | $0.50 | 100 | $50 | -$30 (loss) |
| + caching + routing | $0.10 | 100 | $10 | $10 (50%) |
| + finetune + distill | $0.02 | 100 | $2 | $18 (90%) |
| + local-first option | $0 (user runs) | unlimited | $0 | varies |

The discipline turns -150% margin into +90% margin.

## Variants and ablations

- **Hybrid cloud + local.** Frontier API for hard cases; local model for typical.
- **Spot pricing** for batch agent workloads (50% off via cloud-vendor spot tiers).
- **Pre-paid commits** for predictable volume.
- **Caching-as-a-service** (Helicone / Portkey).
- **LiteLLM as cost-router proxy.**
- **Outsource hosting.** Together / Fireworks / Anyscale for OSS-model serving.
- **Customer-supplied keys.** Customer brings own LLM; you bring agent.
- **Edge inference** for ultra-low-cost.
- **Prompt-cost-budget gate.** Refuse tasks projected > $X.
- **Quality-cost frontier curve.** Measure quality at each cost tier; pick Pareto-optimal.

## Failure modes

- **Cost runaway from looping.** [267-agent-sre](267-agent-sre.md) — bright-line cost cap mandatory.
- **Caching thrashing.** Bad prompt design → low cache hit rate.
- **Routing miscalibration.** Easy tier on hard task → bad output → user retries on frontier → 2× cost.
- **Distillation ceiling.** ~85-95% retention; some tasks need frontier.
- **Currency / vendor pricing changes.** Hedge with OSS option.
- **Free-tier abuse.** Limit + rate-limit + verify.
- **Customer-lifetime-value miscalc.** Discounted future revenue against burn-now reality.
- **Hidden cost.** Embeddings, eval runs, observability storage, monitoring all cost too.
- **Sudden price changes** by vendors break unit economics overnight.

## When to use, when not

**Apply full unit-economics discipline** for any commercial agent product, any high-volume internal agent (where compute cost is non-trivial), any cost-sensitive deployment. **Skip detailed cost engineering** for prototypes, low-volume internal tools, and experiments. **Never skip** observability ([264](264-agent-observability-stack-2026.md)) for any production agent — without cost visibility, runaway is invisible.

## Implications for harness engineering

- **Cost router as `harness_core/router/`.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md), [88-confidence-driven-router](88-confidence-driven-router.md).
- **LiteLLM proxy for vendor flexibility.** Avoid lock-in.
- **Prompt caching by default.** [274-prompt-and-context-engineering-2026](274-prompt-and-context-engineering-2026.md).
- **Distillation pipeline.** [275-agent-finetuning-2026](275-agent-finetuning-2026.md), [235-inference-compression-scaling](235-inference-compression-scaling.md).
- **Local-first option.** [276-local-first-privacy-first-agents](276-local-first-privacy-first-agents.md).
- **Cost observability per span.** [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md).
- **Bright-line cost gates.** [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md), [267-agent-sre](267-agent-sre.md).
- **Cost-budget UX.** [277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md) — visible to user.
- **Eval-suite with cost-budget.** [265-agent-evaluation-2026](265-agent-evaluation-2026.md).
- **Per-feature attribution.** Aggregate spans by feature.
- **Pricing model alignment with cost structure.** Engineering work matches pricing tier.
- **Vendor risk management.** [270-agent-supply-chain-security](270-agent-supply-chain-security.md), [147-vendor-lock-in](147-vendor-lock-in.md).
- **Routine cost dashboards.** [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md) — daily cost report.

**One-line takeaway.** **Agent unit economics in 2026 is a four-layer optimization (per-token + per-task + per-feature + per-customer) with explicit levers per layer (model choice + caching + routing + distillation) — combined, the right discipline turns frontier-API-direct $0.50/task into distilled-local-first $0.02/task, making agent products viable at price points that were impossible 18 months ago; the engineering work matches the pricing model, and observability + bright-line cost gates are mandatory because cost runaway is silent without them.**
