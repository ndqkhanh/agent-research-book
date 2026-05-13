# 110 — Transformer & LLM Architecture for Agent Builders: Only the Parts That Change How You Build Agents

**Sources.** Nagasubramanian, *Agentic AI for Engineers* (Apress, 2026), Chapter 3 (Transformer Models and LLM Architecture); Devlin, *Building LLM Agents with RAG Knowledge Graphs*, Chapter 2 (How LLMs Think: The Transformer and Beyond); plus the canonical Vaswani et al. 2017 *Attention Is All You Need* and the GPT/Llama/Gemini family papers as background.

**One-line definition.** This is the minimum-viable transformer chapter for agent builders — attention, KV cache, autoregressive decoding, positional encodings, context window, tokenization, sampling — covered only at the depth where decisions about *agent design* actually depend on the answers (KV-cache eviction, context budget, structured-output decoding, latency under long contexts, why some prompt patterns work and others don't), and skipping the parts (multi-head math, layer-norm placement, gradient flow, RoPE derivations) that don't.

## Why this chapter matters

Most agent harnesses are built by engineers who treat the LLM as a black box: text in, text out, occasional tool calls. That's enough to ship a demo. It is *not* enough to debug why a 100K-context prompt gets slower per token, why structured-output decoding fails halfway through a JSON, why the agent forgets what was on the first page, or why two queries with the same words produce different answers. Those are all consequences of how the transformer actually executes.

The agent-builder's working model of a transformer needs to answer six questions:
1. **What does context cost in time and memory?** (KV cache; quadratic vs linear regimes.)
2. **Why does the middle of a long context get ignored?** (Attention dispersion; positional bias.)
3. **Why does structured output sometimes break?** (Sampling probabilities vs grammar constraints.)
4. **Why do "the same" tokens sometimes mean different things?** (Tokenizer boundaries.)
5. **Why does one prompt template beat another that says "the same thing"?** (Position effects, format anchoring.)
6. **What can a frozen model never do?** (No state across calls without external memory.)

Once you can answer these six, every other architecture topic is optional. This chapter gives the answers.

## Problem it solves

Agent designers without a transformer mental model make five recurring mistakes:

1. **Stuff the context.** Believing "longer context = more knowledge available", they push 100K-token prompts into the model and are surprised when accuracy drops below the 8K-token baseline.
2. **Trust the JSON.** They prompt for JSON output and don't add structured-decoding fallback, then debug downstream parser errors.
3. **Reuse "the same" prompt across models.** Tokenizer differences (GPT-4 BPE vs Llama SentencePiece vs Gemini SentencePiece) make seemingly identical prompts behave differently.
4. **Assume the model "remembers".** They build a multi-turn agent without an external memory layer, then debug why turn 5 has lost what turn 1 said.
5. **Underbudget latency.** They quote single-token latency and forget that long-context decoding scales with context length, not token count.

Each mistake traces back to a missing piece of the transformer model.

## Core idea in one paragraph

A transformer is a stack of layers, each containing **self-attention** (every token can look at every other token in the prompt + so far in the output) followed by a **feed-forward** network. **Attention** is content-based: each token computes a similarity ("query") against every other token's "key", uses that to weight every other token's "value", and produces a new representation. **Autoregressive decoding** generates the output one token at a time, feeding each new token back in. The cost of each new token grows with how much context already exists, but **KV caching** stores the keys and values of all prior tokens so the per-token cost stays predictable. Sampling at the output is a softmax over the vocabulary, modulated by **temperature**, **top-p**, **top-k**, and any external constraints (logits masking, grammar). Position is encoded either as added positional embeddings or — in modern models — as **RoPE** rotations applied inside attention. Tokenization is **subword-level**, so words split unevenly and "the same text" can have different token counts across models. The model has *no* state between calls; everything it "remembers" must be in the prompt or in an external memory.

That's the architecture. Everything else is engineering.

## Mechanism (step by step)

### 1. Self-attention — content-addressable lookup

For each position *i* in the input, the transformer computes:

```text
Q_i = x_i · W_Q       (query)
K_j = x_j · W_K       (key, for every j)
V_j = x_j · W_V       (value, for every j)
attention_i = softmax(Q_i · K_j^T / sqrt(d)) · V_j     (sum over all j)
```

Three things matter for agent builders:

- **Every output token attends to every prior token.** This is the source of the famous quadratic cost. With *n* tokens in context and *m* tokens of output, attention compute is roughly O(n × m), and memory for the KV cache is O(n × layers × heads × d).
- **The softmax disperses attention.** With long contexts, attention spreads thin — mass that would have gone to one critical token at *n* = 1K is split across hundreds of distractor tokens at *n* = 100K. This is the mechanism behind "lost in the middle" (Liu et al. 2023).
- **Multi-head means multiple parallel attention computations.** Most agent design decisions are head-count-agnostic, but you should know the cache scales linearly with heads.

### 2. KV cache — the agent-relevant memory artifact

The KV cache is the LLM's per-conversation working memory. Each new token computes a fresh Q, but the K and V values for all prior tokens are stored and reused. This is what makes long-conversation latency tolerable: the cost of generating token *t+1* is O(t) for the attention over prior K/V (which is read, not recomputed) instead of O(t²) (which would be re-attention).

For agent builders:
- **Prompt caching** at the API level (Anthropic, OpenAI) is exactly KV-cache reuse across requests with the same prefix. Stable system prompts → cache hits → 5–10× cost reduction.
- **Context truncation** invalidates the cache from the truncation point forward. Context-compaction strategies that rewrite the early prompt break caching.
- **The cache is finite.** Eviction policies (FIFO, LRU, learned — see [78-ngc-neural-garbage-collection](78-ngc-neural-garbage-collection.md)) decide what gets dropped when capacity is exceeded.

### 3. Autoregressive decoding — one token at a time

The transformer produces a probability distribution over the vocabulary at each step. The next token is sampled from that distribution, appended, and the process repeats. This has three consequences:

- **Long outputs are slow.** Latency scales with output length × per-token compute. A 4000-token response takes ~4× longer than a 1000-token response — even if the prompt is identical.
- **Errors compound.** A wrong token at step 3 conditions every subsequent token. Self-correction mid-generation is impossible without external loops ([18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md)).
- **Speculative decoding** generates many tokens at once via a draft model and verifies them in parallel ([94-eagle3-spec-decoding](94-eagle3-spec-decoding.md)), bringing throughput up.

### 4. Sampling — temperature, top-p, top-k, structured

The output distribution is shaped by:
- **Temperature** *T*: rescales logits by 1/T before softmax. T=0 is greedy (deterministic given KV cache state), T=1 is the raw distribution, T>1 is exploration.
- **Top-p (nucleus)**: keep the smallest set of tokens whose cumulative probability exceeds *p*; sample within.
- **Top-k**: keep the *k* highest-probability tokens; sample within.
- **Logits masking**: zero out tokens that violate a constraint (e.g. "must be a digit"). See [112-constrained-decoding](112-constrained-decoding.md) for the full pattern.

For agent builders: **lower temperatures for tool selection, higher for creative generation, T=0 for deterministic regression-testable outputs**.

### 5. Positional encoding — why position matters

Transformers are permutation-equivariant by default; positional information must be added explicitly. Two regimes:
- **Absolute positions** (original transformer, GPT-2) — added to embeddings; limited by the maximum position seen during training.
- **RoPE / ALiBi / sliding-window** (modern models) — applied inside attention; extrapolate to longer contexts but not infinitely.

For agent builders: **the model's effective context is shorter than its advertised context**. Paulsen 2026 reports utilization saturating around 20K tokens even on 128K-context models. Plan your prompt budget accordingly.

### 6. Tokenization — words don't equal tokens

The tokenizer splits text into subword units. Important consequences:
- "Hello world" might be 2 tokens in one model and 3 in another.
- Numbers, code, and non-Latin scripts tokenize unevenly.
- A "10K-token prompt" in GPT-4 BPE is roughly 7-8K words; in Llama tokenizer it's slightly different.
- Whitespace, punctuation, and trailing characters can change token count by 5-10%.

For agent builders: when comparing context budgets across models, count tokens with the model's own tokenizer, not by word or character.

### 7. Statelessness — the model has no memory between calls

The model's parameters never change during inference. Each API call is independent unless the prior context is sent again. This is the foundational truth that motivates the entire agent-memory layer: every persistent agent capability ([09-memory-files](09-memory-files.md), [72-claude-mem-persistent-memory-compression](72-claude-mem-persistent-memory-compression.md), [100-contextual-memory-is-a-memo](100-contextual-memory-is-a-memo.md), [106-memento-paper-theory](106-memento-paper-theory.md)) is a workaround for this single fact.

## Empirical anchors

- **Quadratic attention cost.** Doubling context roughly doubles per-token latency at long context. Caching softens this but does not remove it.
- **"Lost in the middle"** (Liu et al. 2023). Information placed in the middle of a long context is recalled with much lower accuracy than at the start or end. Rate-limiting your prompt length is the cheapest fix.
- **Effective utilization saturation** (Paulsen 2026). Even on 128K-context models, retrieval performance saturates around 20K tokens of relevant content. Bigger context windows do not linearly improve recall.
- **Prompt-caching wins.** Anthropic and OpenAI both report 5–10× cost reduction and 2–3× latency reduction on cached prefixes for stable system prompts.
- **Tokenizer differences matter.** Same text, different tokenizers: GPT-4 ≈ 1.3 tokens/word, Llama ≈ 1.5 tokens/word, Gemini varies. This shifts your context budget meaningfully.

## Variants and counter-arguments addressed

- **"Linear / sparse attention will fix the quadratic cost."** Architectures like Mamba, Linear Attention, and sparse attention have promise but in 2026 production systems still use full attention with caching. The agent-builder treatment is unchanged.
- **"Mixture of Experts changes the math."** MoE affects parameter count and compute path but not the user-facing context behaviour. Agents see MoE as a black-box LLM.
- **"You can fine-tune your way out of statelessness."** Fine-tuning shifts the prior; it does not give the model session memory. Even a fine-tuned model is stateless across calls.

## Failure modes and limitations

1. **Treating context as free.** Every token costs compute, latency, and dollars; budget accordingly.
2. **Ignoring token boundaries.** Code with weird tokenization, languages with rare scripts, and content with many numbers all consume more tokens than expected.
3. **Mid-context information loss.** Without prompt-engineering tricks (anchoring, repetition, summarization), middle-of-context information is unreliable.
4. **Cache invalidation by accident.** A timestamp in the system prompt invalidates the cache for every call. Keep system prompts stable.
5. **Sampling drift.** Temperature > 0 means non-determinism; integration tests that expect exact strings will be flaky. T=0 is reproducible *given* the same KV cache state — but server-side cache misses can still vary.

## When to dive deeper, when this is enough

**This chapter is enough when** you are building an agent harness, choosing a prompt template, sizing context budget, picking a sampling temperature, or debugging "why did the model not see X?".

**Dive deeper when** you are training models, designing inference servers, building speculative-decoding systems, optimising for specific hardware, or implementing novel attention variants.

## Implications for harness engineering

- **Budget context, not just prompt.** Track the cumulative KV-cache cost of long-running agents; evict aggressively. See [08-context-compaction](08-context-compaction.md), [78-ngc-neural-garbage-collection](78-ngc-neural-garbage-collection.md).
- **Stable system prompts unlock prompt caching.** Move all dynamic content to the user-message tail. Anthropic and OpenAI both reward this.
- **Place critical instructions at start *and* end of long prompts.** Information in the middle is unreliable; bookending compensates.
- **Use T=0 for deterministic agent steps**, T>0 for creative or exploratory steps. Treat temperature as a per-step harness knob.
- **Always have an external memory layer.** The model is stateless; persistence is your problem. See [09-memory-files](09-memory-files.md), [106-memento-paper-theory](106-memento-paper-theory.md).
- **Count tokens with the right tokenizer** when comparing context budgets across models or migrating prompts.
- **Pair structured-output requests with grammar enforcement** ([112-constrained-decoding](112-constrained-decoding.md)). Probability ≠ correctness.

The one-sentence takeaway: **the transformer is content-addressable lookup followed by autoregressive sampling, with no memory between calls — and every constraint you'll hit as an agent builder traces back to one of those three facts.**

## See also

- [08-context-compaction](08-context-compaction.md), [09-memory-files](09-memory-files.md), [78-ngc-neural-garbage-collection](78-ngc-neural-garbage-collection.md) — managing the KV cache and memory.
- [94-eagle3-spec-decoding](94-eagle3-spec-decoding.md) — speculative decoding, the throughput escape hatch from autoregressive cost.
- [112-constrained-decoding](112-constrained-decoding.md) — making structured output reliable.
- [100-contextual-memory-is-a-memo](100-contextual-memory-is-a-memo.md) — the deeper consequence of statelessness.
- [32-recurrent-depth-implicit-reasoning](32-recurrent-depth-implicit-reasoning.md) — variant architecture that loops layers for depth.
