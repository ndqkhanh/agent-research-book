# 112 — Constrained Decoding: Logits Masking, Grammar, and Structured Outputs That Cannot Be Malformed

**Sources.** Lakshmanan & Hapke, *Generative AI Design Patterns*, Chapter 2 (Pattern 1: Logits Masking; Pattern 2: Grammar); Willard & Louf 2023 (*Efficient Guided Generation for Large Language Models*); Microsoft TypeChat, OpenAI structured-outputs API, Anthropic tool-use schema enforcement; the *outlines* and *guidance* libraries.

**One-line definition.** Constrained decoding intervenes between the model's logits and the sampler to *zero out* tokens that would violate a declared grammar (regular expression, context-free grammar, JSON schema, finite-state machine), so the output is *structurally guaranteed* to parse — converting "the model usually returns valid JSON" from a probabilistic prayer into a typed contract.

## Why this matters

The single most common failure mode in production agent harnesses is parsing the model's output. The agent asks for JSON; the model returns JSON 99.5% of the time and natural-language prose plus three trailing periods 0.5% of the time; the parser crashes; the agent loop terminates. With 1000 tool calls per day, that's five crashes per day on a single subagent.

Standard mitigations don't solve the problem. Asking nicely ("output only JSON") works most of the time. Few-shot examples nudge the distribution. Temperature 0 narrows variance but doesn't eliminate failure modes. Retry-on-parse-fail throws compute and latency at a problem that should not exist. The *real* fix is to make malformed output structurally impossible by intercepting the decoding loop and masking out any token that cannot extend a valid output. After this, the parser cannot fail because the bytes the parser sees are guaranteed by construction.

For agent builders, constrained decoding is the difference between a flaky agent and a reliable one. It is also the cheapest reliability win available — no model swap, no fine-tuning, no prompt engineering, just a small wrapper around the sampler.

## Problem it solves

Six concrete failure modes that constrained decoding eliminates:

1. **Trailing prose.** The model returns valid JSON followed by "I hope this helps!" — JSON parser crashes.
2. **Truncated output.** The model hits max-tokens mid-JSON; output is invalid.
3. **Missing required fields.** Schema requires `{name, age}`; model returns `{name}`; downstream code fails.
4. **Wrong types.** Schema requires `age: int`; model returns `age: "thirty"`.
5. **Invalid enum values.** Schema requires `status ∈ {"open", "closed"}`; model returns `"opened"`.
6. **Malformed nested structures.** Unbalanced braces, wrong array vs object, escaped quotes inside strings.

All six are eliminated *by construction* under properly-applied constrained decoding.

## Core idea in one paragraph

The transformer produces a probability distribution over the vocabulary at each step. A constraint engine maintains a *state machine* that knows what tokens are valid next given the output produced so far. Before sampling, multiply the distribution by a *mask* that zeroes out invalid tokens. Re-normalise; sample. Append; advance the state machine; repeat. The constraint can be a regular expression (numeric ID, date), a context-free grammar (balanced parens, code), a JSON schema (typed object), or a custom finite-state machine. The model's distribution over *valid* tokens is preserved (relative ordering, sampling temperature); the only effect is that *invalid* tokens become impossible. The model's content quality is unchanged; the structural correctness becomes 100%.

## Mechanism (step by step)

### 1. The decoding loop with a constraint engine

```text
constraint = build_constraint(grammar)         # FSM, CFG, regex, or JSON-schema
state      = constraint.initial_state
output     = ""
while not done:
    logits = model.forward(prompt + output)    # raw distribution
    mask   = constraint.mask(state)            # 1.0 for valid tokens, 0.0 for invalid
    masked = logits * mask                     # invalid tokens get -inf logit
    token  = sample(softmax(masked / T))        # standard sampling
    output += token
    state  = constraint.transition(state, token)
    done   = constraint.is_terminal(state) or len(output) > max_tokens
```

The wrapper is ~50 lines of code; the constraint engine is the substantive work.

### 2. Constraint flavours

Four common forms, in increasing expressiveness:

- **Regular expression.** Generate text matching `^\d{4}-\d{2}-\d{2}$` (a date). FSM-based; small memory; fast. Sufficient for IDs, dates, phone numbers, structured strings.
- **JSON schema / Pydantic model.** Generate JSON conforming to a typed schema. The constraint engine compiles the schema to a context-free grammar over JSON tokens. Most agent use cases.
- **Context-free grammar.** Generate text matching a CFG (e.g. valid Python expressions, valid SQL). More expressive than JSON schema; needed for code generation with type guarantees.
- **Custom finite-state machine.** Hand-written transitions for domain-specific patterns. Useful when the natural specification is a state machine (e.g. "first an HTTP method, then a URL, then optional headers, then a body").

### 3. JSON-schema constraint — the dominant agent-builder use case

Given a Pydantic model:

```python
class ToolCall(BaseModel):
    tool_name: Literal["search", "code", "calculator"]
    args: dict
    rationale: str
```

The constraint engine compiles this to a CFG over JSON tokens that:
- Requires `{` start.
- Requires the keys `tool_name`, `args`, `rationale`.
- Restricts `tool_name`'s value to one of three string literals.
- Allows `args` to be any JSON object.
- Allows `rationale` to be any JSON string.
- Requires `}` end.

Sample under this constraint: the model can choose freely *within* the structure — which tool, what args, what rationale — but cannot ever produce malformed JSON, missing keys, or invalid enum values.

### 4. Performance considerations

Constrained decoding adds two costs:
- **Mask computation.** O(|vocab|) per step. With vocabularies of 50K–200K tokens, this is non-trivial but small relative to the model forward pass.
- **State-machine memory.** Context-free grammars need a stack; complex grammars can blow up.

Three optimisations that real implementations use:
- **Cached transitions.** Precompute valid-next-token sets per FSM state and cache them.
- **Speculative decoding compatibility.** The mask must be applied to both draft and verifier; otherwise speculative decoding breaks the constraint.
- **Batch parallelism.** Per-stream masks are independent; batch over streams for GPU efficiency.

Reported overhead: 5–15% throughput hit on a typical schema-constrained workload. Worth it for 100% structural correctness.

### 5. Provider-native vs library-based

Three integration paths:

- **Provider-native.** OpenAI's structured-outputs API, Anthropic's tool-use schema enforcement, Gemini's function-calling-with-schema. Easiest path; fully supported. Limited grammar expressiveness (mostly JSON-schema).
- **Library-based.** *outlines*, *guidance*, *lm-format-enforcer*, *llguidance* — open-source libraries that wrap any LLM (open-source or via logits-bias APIs) with a constraint engine. More expressive (CFG, custom FSMs).
- **Inference-server-native.** vLLM, TensorRT-LLM, and friends ship grammar enforcement as a server flag. The right path for self-hosted deployments.

For agent builders: **default to provider-native when available; fall back to library-based for self-hosted or when you need CFG/regex beyond JSON-schema**.

### 6. Composing with other patterns

Constrained decoding composes orthogonally with:

- **Tool calling.** Tool args are guaranteed-valid JSON, so the agent never sees malformed tool inputs.
- **Structured planning.** A planner that emits `{steps: [...]}` is structurally safe to consume.
- **Self-critique.** A judge that emits `{score: int, rationale: string}` cannot return un-parseable scores.
- **Prompt caching.** Constrained decoding is per-step output; the prompt prefix is unaffected.

It does *not* compose with:

- **Free-text creative generation.** Don't constrain a creative-writing prompt; the constraint flattens variance and makes output dull.
- **Adversarial robustness.** Constrained decoding does not stop the model from picking the *wrong* tool — only ensures the output *parses*. Semantic correctness needs other defences ([22-guardrails-prompt-injection](22-guardrails-prompt-injection.md)).

## Empirical anchors

- **100% parse success.** Reported by every major implementation when grammar matches schema.
- **5–15% throughput overhead.** Reported across vLLM, outlines, guidance.
- **Latency parity at low constraint complexity.** JSON-schema constraints add < 5% latency on most agent workloads.
- **Quality preservation.** Structured-outputs benchmarks show no significant content-quality degradation when constraints are correctly specified — the model picks the same values, just within the structure.
- **Provider parity.** OpenAI structured-outputs and Anthropic tool-use schemas perform indistinguishably on structured tasks.

## Variants and counter-arguments addressed

- **"Retry on parse fail is good enough."** It is not. Retry adds latency and cost; constrained decoding adds neither (or 5–15% on throughput) and is deterministic.
- **"The model knows JSON; constraints are unnecessary."** The model knows JSON 99.5% of the time. Production at 1000 calls/day means 5 daily failures. Constraints take that to 0.
- **"Constraints make the model less capable."** Constraints reduce *malformed* output to 0; they do not reduce semantically valid output. The model picks the same content, just in a guaranteed shape.
- **"Grammar engineering is hard."** For JSON, it's free (auto-derived from schema). For CFG/regex, the work is one-time.
- **"This won't work for free-text fields."** It works fine: a JSON-schema string is unconstrained content; the *position* of that content is constrained.

## Failure modes and limitations

1. **Wrong constraint.** A bug in your schema means the model can't produce valid output; debug by relaxing constraints incrementally.
2. **Constraint conflict with tokenizer.** Some grammars cannot be efficiently represented over a given tokenizer; library implementations vary in handling this.
3. **Empty allowed set.** If the constraint produces no valid next tokens (e.g. a regex with no matches), the engine either errors or hangs. Implementations differ in behavior.
4. **Speculative decoding pitfall.** Constraints applied to the verifier but not the draft model produce misaligned masks. Use library/server implementations that handle this.
5. **Free-text content quality.** When a free-text field is wrapped in a JSON-schema string, the model still has full creative freedom, but token boundaries (e.g. quote escaping) can subtly shape output. Test the wrapped form, not just the unwrapped.
6. **Streaming partial validation.** Streaming output requires partial-grammar awareness; not all libraries support this cleanly for arbitrary CFGs.
7. **Schema sprawl.** Production agents accumulate dozens of schemas; manage them like API contracts (versioned, reviewed).

## When to use, when not

**Use constrained decoding for any agent output that will be machine-parsed:** tool calls, structured plans, classifier outputs, scoring outputs, extracted entities, configuration objects.

**Skip it for free-text outputs to humans:** chat replies, summaries, creative content, explanations.

**Use library-based decoding for self-hosted models** where provider-native isn't available; **use provider-native for hosted models** where it's a one-line API change.

## Implications for harness engineering

- **Mandate constrained decoding for all tool-call schemas.** This is the single largest reliability win available in an agent harness; it costs ~5–10 lines of code per tool definition.
- **Treat schemas as harness assets.** Every tool, every structured output, every classifier should have a Pydantic model checked into the repo. See [04-skills](04-skills.md), [07-model-context-protocol](07-model-context-protocol.md).
- **Test schemas at agent-init time.** Validate that every tool's schema compiles to a valid grammar; fail fast.
- **Never wrap a tool call in `try/except json.JSONDecodeError`.** If you need that try/except, you don't have constrained decoding. Add it.
- **Pair with provider-native tool-calling.** OpenAI / Anthropic / Gemini all support schema-enforced tool calls; use the native path.
- **For open-source models, ship the inference server with grammar enforcement on.** vLLM, TensorRT-LLM, sglang all support this.
- **Guard against semantic errors separately.** Constrained decoding ensures output parses; it does not ensure output is *correct*. Pair with [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) and [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md) for semantic safety.

The one-sentence takeaway: **constrained decoding turns "the model usually returns valid JSON" into a typed contract — pay 5–15% throughput, eliminate parse-fail forever.**

## See also

- [110-transformer-llm-architecture-for-agent-builders](110-transformer-llm-architecture-for-agent-builders.md) — sampling, logits, autoregressive decoding background.
- [04-skills](04-skills.md), [07-model-context-protocol](07-model-context-protocol.md) — where tool schemas live in the harness.
- [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) — semantic verification beyond structural correctness.
- [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md) — defenses constrained decoding does NOT provide.
- [93-dspy](93-dspy.md) — DSPy uses typed signatures that compose naturally with constrained decoding.
- [111-prompt-engineering-as-discipline](111-prompt-engineering-as-discipline.md) — format anchoring, the prompt-side complement.
