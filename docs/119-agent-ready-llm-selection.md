# 119 — Agent-Ready LLM Selection & Deployment: Choosing the Model Class, Hosting, Latency Budget, and Cost Envelope

**Sources.** Arsanjani & Bustos, *Agentic Architectural Patterns*, Chapter 2 (Agent-Ready LLMs: Selection, Deployment, and Operationalization); Ozdemir, *Building Agentic AI*, Chapter 9 (Optimizing Production Workloads); plus the public LLM-leaderboard literature (LMSys Chatbot Arena, MMLU, GPQA, HumanEval, AgentBench).

**One-line definition.** Choosing an LLM for an agent stack is a six-axis trade-off — *capability*, *latency*, *cost*, *context*, *tool-use ability*, *deployment posture* — and the wrong starting question is "which is best?"; the right starting question is "for *which subtask* in the stack?", because frontier models, mid-tier models, and SLMs all have a role, and the cheapest reliable agent is one that uses each tier where it earns its place.

## Why this matters

In 2024 the question was binary: GPT-4 or open-source. In 2026 the question is multi-dimensional: GPT-4 or Claude or Gemini or Llama or Qwen or Mistral or DeepSeek, hosted or self-hosted, frontier or mid-tier or small, with or without vision/audio, fine-tuned or not, supporting which tool-use protocol. The wrong choice pays compounding cost: 2× the API bill, 5× the latency, brittle behavior in production, hard to migrate when the model changes.

For agent builders, model selection is not one decision — it's a per-subtask decision repeated across the stack. The planner needs reasoning capacity; the router needs latency; the extractor needs format reliability; the verifier needs cost-efficiency. Treating "the LLM" as a single choice for the whole agent is the most common over-spend.

This chapter is the framework for making per-subtask model decisions, the deployment trade-offs (hosted vs self-hosted vs hybrid), and the operational considerations (rate limits, fallbacks, version pinning) that turn the decision into a sustainable production setup.

## Problem it solves

Five model-selection failures:

1. **Frontier-everywhere.** Use GPT-4 for every LLM call in the stack. Cost balloons; latency is bad; many calls didn't need GPT-4.
2. **Open-source-everywhere.** Self-hosted Llama for everything. Reasoning quality on the planner suffers; the cost saving doesn't materialise after ops overhead.
3. **One-vendor lock-in.** All calls go through OpenAI. When OpenAI rate-limits or has an outage, the agent is down. When pricing changes, you can't switch.
4. **No fallback.** Primary model fails; agent crashes. No second-tier path.
5. **Wrong-axis optimisation.** Optimising for cost when latency is the user-facing constraint; or vice versa.

Each is solved by per-subtask selection plus deployment hygiene.

## Core idea in one paragraph

For every LLM call in your agent, ask six questions: (1) **Capability** — is this a reasoning task, a classification task, a generation task? Frontier vs mid-tier vs SLM has different sweet spots. (2) **Latency** — is this user-facing or background? P95 < 500ms? P95 < 5s? (3) **Cost** — what's the per-call budget at the volume you'll run? (4) **Context** — short prompt or long context? Some models degrade at long context faster than others. (5) **Tool use** — does this call require structured tool output? Some models are better at tool selection than others. (6) **Deployment posture** — hosted, self-hosted, or hybrid? Latency, cost, control, compliance trade-offs differ. The answers point at one or two model candidates per subtask. Validate empirically on your eval set; pick the cheapest that meets the quality bar; ship with a fallback to a second-tier model. Repeat per subtask. The result is an agent stack with three to five different models in it, each placed where it earns its role.

## Mechanism (step by step)

### 1. The six axes

**Capability.** Frontier (GPT-4 / Claude Opus / Gemini 2 Pro / o3) for complex reasoning; mid-tier (GPT-4-mini / Claude Sonnet / Haiku / Gemini Flash) for general tasks; SLMs (Llama-3-8B / Phi-3 / Qwen-2-7B / Mistral) for narrow tasks. See [117-small-language-models](117-small-language-models.md) for the SLM-side decisions.

**Latency.** Frontier models: 1–5s per 1K tokens output. Mid-tier: 0.5–2s. SLMs: 0.05–0.5s. Long-context inputs add latency proportional to context length (see [110-transformer-llm-architecture-for-agent-builders](110-transformer-llm-architecture-for-agent-builders.md)).

**Cost.** Approximate per-million-tokens pricing in 2026:
- Frontier: $3–15 input / $15–60 output
- Mid-tier: $0.15–1 input / $0.5–3 output
- Hosted SLM: $0.05–0.3 input / $0.1–0.5 output
- Self-hosted SLM: compute cost only (typically $0.01–0.1 per million effective tokens at 1M+ tokens/day volume)

**Context window.** Frontier: 128K–2M nominal. Mid-tier: 32K–128K. SLMs: 8K–128K. *Effective* context is much shorter; budget conservatively.

**Tool use.** Provider-native tool calling on Anthropic, OpenAI, Gemini is mature. Open-source models vary: Llama-3 / Qwen-2 are competent; smaller models often fail at multi-tool selection.

**Deployment posture.** Hosted (provider API) is the default; self-hosted (vLLM, TensorRT-LLM) for cost at scale or compliance; hybrid (mix) for fallback or for matching subtasks to optimal hosts.

### 2. Per-subtask defaults — the starting point

For an agent stack, a reasonable default mapping:

| Subtask | Default model class | Reason |
|---|---|---|
| **Planner** | Frontier | Reasoning capacity is the bottleneck |
| **Executor (tool calls)** | Mid-tier | Reasoning lighter; cost matters more |
| **Router / classifier** | SLM (PEFT) | Narrow, high-volume, latency-sensitive |
| **Extractor** | SLM (PEFT) | Narrow, structured output |
| **Reranker** | SLM (cross-encoder) | Pure scoring, high volume |
| **Verifier / judge** | Mid-tier or frontier | Quality matters; volume moderate |
| **Synthesiser (final output)** | Mid-tier or frontier | User-facing quality matters |

This default is a starting point; per-subtask validation (next step) refines it.

### 3. The validation loop

For each subtask:

```text
1. Build a 200-example eval set for the subtask.
2. Run all candidate models on the eval set with appropriate prompts.
3. Measure: accuracy, p95 latency, cost per call, format reliability.
4. Pick the cheapest model that clears the quality bar.
5. Note the next-cheapest as the fallback.
6. Document the choice + rationale.
```

This is the same evaluation discipline as [115-evaluating-llm-systems](115-evaluating-llm-systems.md), applied at the subtask level.

### 4. Deployment posture decision

**Hosted (provider API)**:
- Pros: zero ops, latest models, scale handled.
- Cons: per-token cost, rate limits, vendor lock-in, less control.
- Default for: most teams, most subtasks.

**Self-hosted (vLLM / TensorRT-LLM)**:
- Pros: cost at scale, latency control, compliance, version pinning.
- Cons: ops overhead, GPU planning, fault tolerance.
- Default for: high-volume narrow subtasks (SLM rerankers, classifiers) at >1M tokens/day.

**Hybrid**:
- Frontier models hosted (you can't self-host them anyway); SLMs self-hosted.
- Or: primary path hosted, fallback path self-hosted (or vice versa).

The break-even point for self-hosting an SLM is usually 1M–10M tokens/day. Below that, hosted wins on TCO.

### 5. Fallback strategy

Every primary model needs a fallback path:

```text
try:
    response = primary_model(prompt)
except (RateLimited, Outage, Timeout):
    response = fallback_model(prompt)
```

The fallback should be:
- A different vendor (or self-hosted) so a vendor outage doesn't take you down.
- Comparable quality (within 5%) so the user-facing experience degrades gracefully.
- Documented in the agent's contract; users should know when fallback is in effect.

Without a fallback, your agent's uptime is bounded by the worst-case uptime of your primary vendor.

### 6. Version pinning and migration

Provider models change without notice. Three pinning strategies:

- **Latest alias** (e.g. `gpt-4o-latest`): get improvements automatically; risk silent regressions.
- **Date-pinned** (e.g. `gpt-4o-2024-08-06`): stable; must opt in to upgrades.
- **Locked checkpoint**: only available for self-hosted.

For production agents, **date-pinned** is the recommendation. Schedule re-evaluation against the latest version quarterly; migrate when the new version benchmarks better.

### 7. Operational hygiene

- **Rate-limit headroom.** Provision 2× peak; spikes are real.
- **Retry with backoff.** Exponential backoff, max 3 retries, then fallback.
- **Timeout contract.** Per-call timeout matched to the subtask's latency budget.
- **Cost dashboard.** Per-subtask cost tracked daily; alert on 20% week-over-week drift.
- **Circuit breaker.** If primary model fails > 5% of calls in 5 minutes, fail-over to fallback automatically.
- **Provider health monitoring.** Subscribe to status pages; integrate with on-call.

## Empirical anchors

- **Cost spread is 30–100×** between frontier and SLM tiers; mis-targeting is the easiest source of cost overrun.
- **Latency spread is 10–30×** between frontier and SLM tiers; latency-sensitive subtasks must match.
- **Mid-tier models** (GPT-4-mini, Sonnet, Haiku, Flash) hit a sweet spot for ~70% of subtasks in a typical agent stack.
- **Vendor outages** average a few hours per year per major provider; without fallback, your agent inherits this floor.
- **Multi-vendor agents** in 2026 are common; teams report 30–50% TCO improvement vs single-vendor setups.

## Variants and counter-arguments addressed

- **"Just use frontier everywhere."** Easier to operate; expensive; misses the cost-efficiency lever.
- **"Self-host everything."** Cost wins at scale but loses on frontier-model quality (you can't self-host GPT-4) and on ops overhead.
- **"One vendor for simplicity."** Until they outage, rate-limit, raise prices, or kill the model.
- **"Pinning is risky — newer is better."** Newer is sometimes better. Quarterly re-evaluation captures gains without surprise regressions.
- **"Fallbacks add complexity."** They add operational maturity. The complexity is mechanical and the failure mode is well-understood.

## Failure modes and limitations

1. **Vendor outage propagation.** Without fallback, primary vendor's uptime is your ceiling.
2. **Rate-limit collisions.** Multiple agents sharing one API key burst together; throttle at the harness layer.
3. **Cost surprise.** Volume miscalculation × frontier model × long context = unexpectedly large bill. Always project annual cost on real volume.
4. **Latency creep.** Long-context calls slow down silently; track p95 by context-length bucket.
5. **Model deprecation.** Provider sunsets a model; if you weren't pinned, behavior changes silently.
6. **Format-reliability variance.** Different models format JSON differently; constrained decoding ([112-constrained-decoding](112-constrained-decoding.md)) is non-optional in production.
7. **Context-window mis-trust.** "128K context" doesn't mean 128K of useful context. Test at production prompt lengths.
8. **Self-host fault tolerance.** GPU OOM, model loading failures, autoscaling issues — all the standard ML-ops problems.

## When to use, when not

**Use per-subtask model selection** for any production agent with > 100 LLM calls per task or > 10K tasks per month. The TCO savings amortise the engineering quickly.

**Stick with one-model defaults** for early prototypes (stage 1, see [118-genai-maturity-models](118-genai-maturity-models.md)). Optimise later.

**Self-host** when volume × cost-spread justifies ops overhead.

**Always have a fallback** for primary models in production.

## Implications for harness engineering

- **Build a model gateway abstraction.** The agent calls `model.complete(...)`; the gateway picks the actual model. This is the lever for fallback, A/B, version-pin migration. See [126-frameworks-comparison](126-frameworks-comparison.md) for which frameworks ship this.
- **Per-subtask model registry.** Document which model is used where; review quarterly.
- **Eval per subtask, not per agent.** Aggregate metrics hide subtask regressions. See [115-evaluating-llm-systems](115-evaluating-llm-systems.md).
- **Cost dashboard at subtask granularity.** Per-call cost × volume; alert on drift.
- **Date-pin in prod, latest in dev.** Catch regressions before users do.
- **Multi-vendor by design.** Hard requirement for any production stack with uptime SLAs.
- **Multi-LoRA serving for SLMs.** One base, many adapters, many subtasks. See [116-adapter-tuning-lora-peft-for-agents](116-adapter-tuning-lora-peft-for-agents.md), [117-small-language-models](117-small-language-models.md).
- **Context-budget per call type.** Different subtasks have different sensible context lengths; enforce limits at the gateway.

The one-sentence takeaway: **agent-ready LLM selection is per-subtask, multi-vendor, with a model gateway that abstracts the choices and a fallback path that keeps the agent up when one provider has a bad day.**

## See also

- [110-transformer-llm-architecture-for-agent-builders](110-transformer-llm-architecture-for-agent-builders.md) — the underlying model behavior.
- [115-evaluating-llm-systems](115-evaluating-llm-systems.md) — the eval that drives selection.
- [116-adapter-tuning-lora-peft-for-agents](116-adapter-tuning-lora-peft-for-agents.md), [117-small-language-models](117-small-language-models.md) — the SLM tier.
- [126-frameworks-comparison](126-frameworks-comparison.md) — frameworks that ship model gateways.
- [147-vendor-lock-in](147-vendor-lock-in.md) — the strategic side of multi-vendor.
- [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md), [88-confidence-driven-router](88-confidence-driven-router.md) — cost-aware routing patterns.
