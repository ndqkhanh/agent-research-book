# 274 — Production Prompt + Context Engineering 2026

**Anchors.** Anthropic — *Engineering with Claude* + prompt caching (Aug 2024 GA) and [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) (Sep 2025). OpenAI — automatic prompt caching (Oct 2024). Google — Gemini context caching (2024). Karpathy — *Software 3.0 / agentic engineering* (May 2026). Recent agent-context papers: [Active Context Compression / Focus](https://arxiv.org/abs/2601.07190), [Direct Corpus Interaction](https://arxiv.org/abs/2605.05242), [OpenDev](https://arxiv.org/abs/2603.05344), [Dive into Claude Code](https://arxiv.org/abs/2604.14228). Companions: [111-prompt-engineering-as-discipline](111-prompt-engineering-as-discipline.md), [112-constrained-decoding](112-constrained-decoding.md), [234-context-length-scaling](234-context-length-scaling.md), [235-inference-compression-scaling](235-inference-compression-scaling.md), [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md), [269-prompt-injection-2026](269-prompt-injection-2026.md).

**One-line definition.** A 2026 picture of **production prompt + context engineering** as a unified discipline — context being the **whole working set** the model attends to (system prompt + developer prompt + user input + retrieved docs + tool outputs + memory reads + conversation history + structured-output schema), engineered as a **versioned artifact** with **prompt caching** (5–10× cost reduction for repeat prefixes), **context-window budgeting** (effective vs nominal per [234](234-context-length-scaling.md)), **structured output** (JSON/Pydantic/XML), **content provenance + spotlight prompting** ([269](269-prompt-injection-2026.md)), and **prompt-as-code** (templates in repo, CI'd, reviewed) — the difference between a prototype and a production agent is partly which model you pick and partly **how disciplined the context-engineering practice is**.

## Why this matters (context is the whole input the model sees; engineering it is a discipline)

The 2024 framing of prompt engineering as "magic phrases that work" became the 2026 framing of **context engineering as system design**: you're building the data structure the model reasons over. Context = system prompt + developer prompt + user message + retrieved docs + tool results + memory reads + conversation history + structured-output schema. Each component has a trust level, a cost in tokens, an ordering relative to others, and a cache strategy. Production-grade work treats context as an engineered artifact, not a side effect.

The 2024–2026 platform changes that drove this discipline: **prompt caching** (Anthropic Aug 2024, OpenAI Oct 2024, Gemini 2024) gives 5–10× cost reduction for repeat prefixes — system prompts, retrieved docs, conversation history can be cached if they precede the variable suffix; **structured output** (Anthropic JSON-mode, OpenAI structured outputs with schemas, Gemini constrained decoding) makes "the model returns JSON conforming to this Pydantic schema" production-trivial; **effective vs nominal context length** ([234](234-context-length-scaling.md)) became a measurable discipline via RULER. These primitives enable a production discipline that wasn't possible in 2023.

## Core idea

Engineer context as a **typed, ordered, versioned, cache-aware data structure**. Order: most-stable-first (cacheable) → least-stable-last (variable). Trust: provenance metadata per item ([269](269-prompt-injection-2026.md)). Output: schema-constrained where possible. Versioning: prompts in repo, reviewed in PRs, regression-tested. Cost: token budget per request, with per-component allocation; cached components effectively free. Structure: XML tags for sectioning, instruction hierarchy, spotlight on untrusted content.

## Mechanism (step by step)

### (a) The seven context components

| Component | Trust | Cache | Notes |
|---|---|---|---|
| System prompt | Highest | ✅ usually | Hierarchy declaration; constants |
| Developer prompt | High | ✅ usually | Task-specific; semi-stable |
| Tool definitions | High | ✅ when stable | Pre-cache for repeat use |
| Retrieved docs | Variable | Sometimes | Spotlight wrap; provenance |
| Memory reads | Variable | Sometimes | Tier-classified |
| Conversation history | Variable | ✅ prefix | Append-only; cache prefix |
| User input | Lowest | ❌ | Fresh per call |

### (b) Prompt caching mechanics

```python
# Anthropic
messages = [
    {
        "role": "system",
        "content": [
            {"type": "text", "text": SYSTEM_PROMPT, "cache_control": {"type": "ephemeral"}},
            {"type": "text", "text": TOOLS_DOC, "cache_control": {"type": "ephemeral"}},
            {"type": "text", "text": MEMORY_TIER_1, "cache_control": {"type": "ephemeral"}},
        ],
    },
    {"role": "user", "content": user_input},  # not cached
]
```

Cache hit: ~10% cost vs uncached; ~50% faster prefill; 5-min TTL (Anthropic); auto-extend on hit. **Order matters** — caches everything *before* the first uncached block; hot prefix design is the engineering work.

### (c) Structured output

```python
from pydantic import BaseModel
from anthropic import Anthropic

class Finding(BaseModel):
    severity: Literal["low", "medium", "high", "critical"]
    file_path: str
    line_number: int
    description: str

response = client.messages.create(
    model="claude-sonnet-4-6",
    messages=[...],
    response_format={"type": "json_schema", "json_schema": Finding.model_json_schema()},
)
findings = [Finding.model_validate(f) for f in json.loads(response.content[0].text)]
```

Constrained decoding ([112-constrained-decoding](112-constrained-decoding.md)) eliminates JSON parse failures; reduces output tokens; production-default for any structured output.

### (d) XML-tagged sectioning (Anthropic-recommended)

```
<system_instructions>
You are a code reviewer. Be concise.
</system_instructions>

<retrieved_docs source="repo_search" trust="medium">
{docs}
</retrieved_docs>

<untrusted_content source="user_email" trust="low">
{email_body}
</untrusted_content>

<conversation_history>
{history}
</conversation_history>

<user_request>
{request}
</user_request>
```

The model is trained to respect XML structure; sections become first-class addressable.

### (e) Context-budget per request

```python
class ContextBudget:
    system: int = 2_000
    developer: int = 1_000
    tools: int = 3_000
    retrieved: int = 16_000
    memory: int = 4_000
    history: int = 8_000
    user: int = 2_000
    headroom: int = 4_000

    @property
    def total(self): return sum(asdict(self).values())  # 40K — well under effective context
```

Budgets per component; allocator trims aggressively (summarize, paginate, or page out to memory) when over.

### (f) Prompt-as-code

```
prompts/
  ├── system/
  │   └── researcher.v3.md      (system prompt for researcher agent)
  ├── developer/
  │   └── lit_review.v1.md
  ├── tools/
  │   └── search_tools.json
  └── tests/
      └── test_researcher.py     (golden cases, regression suite)
```

CI runs eval suite ([265](265-agent-evaluation-2026.md)) on every prompt change; PR review for prompt updates; semantic versioning per prompt.

### (g) Memory + retrieval as context layers

[233-memory-scaling-for-agents](233-memory-scaling-for-agents.md) memory tiers map to context layers: hot (working) = current message, warm (episodic) = retrieved on demand, cold (archival) = retrieved rarely. Each tier has its own cache + budget allocation.

### (h) Effective context budget

[234-context-length-scaling](234-context-length-scaling.md) — RULER-effective length is the real budget; nominal is marketing. Plan against effective.

### (i) 2025–2026 agent-context updates

| Work | Context-engineering lesson | Why it matters |
|---|---|---|
| [Anthropic effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | Context is the whole inference state: prompts, tools, examples, message history, retrieved data, memory, and subagent summaries; keep the smallest high-signal working set | Production guidance for just-in-time retrieval, compaction, structured note-taking, and subagents |
| [Focus / Active Context Compression](https://arxiv.org/abs/2601.07190) | Let the agent explicitly compress and prune its own trajectory into a persistent Knowledge block | Reports 22.7% token reduction on 5 hard SWE-bench Lite tasks at equal accuracy, but warns that aggressive pruning can add overhead |
| [Direct Corpus Interaction](https://arxiv.org/abs/2605.05242) / [DCI-Agent-Lite](https://github.com/DCI-Agent/DCI-Agent-Lite) | Retrieval interface resolution matters; raw-corpus terminal tools can outperform fixed top-k retrievers for agentic search | Pushes context engineering from "which chunks to retrieve" to "which information interface should the agent control" |
| [OpenDev](https://arxiv.org/abs/2603.05344) | Context management belongs in the runtime loop, alongside model routing, tool dispatch, memory, and reminders | Concrete terminal-agent architecture with adaptive compaction, lazy tool discovery, and cross-session memory |
| [Dive into Claude Code](https://arxiv.org/abs/2604.14228) | Production coding agents are mostly surrounding systems: permission modes, compaction pipeline, MCP/plugins/skills/hooks, subagents, and session storage | Useful design-space map for deciding what must be prompt, memory, tool, hook, or durable state |

## Empirical results (table)

| Optimization | Cost reduction | Latency reduction |
|---|---:|---:|
| Prompt caching (Anthropic 5-min) | ~90% on cached portion | ~50% prefill |
| OpenAI auto-cache | Up to 50% | Variable |
| Structured output (vs free-form parse) | Output tokens 2-3× shorter | Parse failures → 0 |
| XML sectioning | Negligible cost; quality up | n/a |
| Context budget (vs naive dump) | 30-70% | Proportional |
| Memory tier hot-only on common path | 60-80% | Faster |
| Focus-style active compression | paper reports 22.7% total token reduction on N=5 SWE-bench Lite hard instances | mixed; can add overhead on iterative tasks |
| DCI raw-corpus interaction | no offline index; can avoid embedding/vector-db cost | slower than precomputed retrieval; depends on tool loop |

## Variants and ablations

- **Few-shot examples in cached prefix.** Examples stable → cache.
- **Hard-coded refusal patterns.** Pre-output guardrails.
- **Self-consistency at output.** Ensemble for hard cases.
- **Output-length budgets.** Cap response tokens.
- **Tool-result truncation.** Long outputs paginated.
- **Compression of conversation history.** Summarize old turns.
- **Active agent-controlled compression.** Focus-style `start_focus` / `complete_focus` regions let the agent decide when to preserve learnings and drop raw traces.
- **Just-in-time corpus navigation.** DCI/Claude-Code-style file references, `rg`, and targeted reads keep large corpora out of context until needed.
- **Per-user persona caching.** SOUL.md-shape persona ([Lyra Block 07](../projects/lyra/docs/blocks/07-memory-three-tier.md)) cached per user.
- **DSPy-shape compiled prompts.** [93-dspy](93-dspy.md) — prompts as compiled programs.

## Failure modes

- **Cache thrashing** — variable bytes early in the sequence kill hit-rate.
- **Stale cache** — TTL expiry on hot path.
- **Schema mismatch** — structured output schema drifts from downstream parser.
- **XML tag injection in user input.** Strip closing tags.
- **Effective-context overrun.** Plan against RULER, not nominal.
- **Memory poisoning into prompts.** [269](269-prompt-injection-2026.md), [270](270-agent-supply-chain-security.md), [233](233-memory-scaling-for-agents.md).
- **Prompt drift without versioning.**

## When to use

**Apply full discipline** for any production agent — caching alone pays back the engineering work in a week. **Skip** for one-shot scripts. **Never skip** structured output for any schema-able response and cached prefix for any repeat-prefix workload.

## Implications for harness engineering

- **`harness_core/prompts/` package.** Versioned templates; CI'd; regression-tested.
- **Context allocator with budget.** Per-component limits; auto-trim.
- **Prompt-cache-aware ordering.** Stable → variable; spec'd in template.
- **Structured output by default.** Pydantic schemas everywhere.
- **Spotlight wrapping.** [269](269-prompt-injection-2026.md) — automatic for retrieved/memory/tool content.
- **Provenance metadata.** [269](269-prompt-injection-2026.md), [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md).
- **Effective-context tracking.** [234](234-context-length-scaling.md) — RULER per model.
- **Memory-tier integration.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md).
- **Cost router by cache hit rate.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md).
- **Eval regression on prompt changes.** [265](265-agent-evaluation-2026.md).
- **HIR observability of prompts.** [264-agent-observability-stack-2026](264-agent-observability-stack-2026.md).
- **DSPy alignment for prompt compilation.** [93-dspy](93-dspy.md).
- **Cross-channel verifier with cached system prompt.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md).

**One-line takeaway.** **Production prompt engineering in 2026 is context engineering — typed, ordered, versioned, cache-aware data structures with structured output, XML sectioning, prompt-as-code in repo, RULER-effective budget allocation per component, and 5–10× cost reduction from prompt caching alone; the discipline turns "the agent works in demos" into "the agent ships at production cost".**
