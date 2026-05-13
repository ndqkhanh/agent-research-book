# 179 — Skill Retrieval, Routing & Activation: Which Skills to Load?

**Scope.** Given the open-source skill catalogs surveyed in [docs/176](176-skill-discovery-curator-oss-landscape-may-2026.md) — **22,000+ Claude Code-format skills, 25K+ MCP servers, 16K+ APIs in ToolBench, 1M+ claimed marketplace entries** — the load-bearing question becomes: how does an agent know *which* skill to load on any given turn? This doc surveys native LLM mechanisms, embedding-based retrieval, LLM-as-router patterns, hierarchical / navigation alternatives, and the strongest 2026 paper directly on the problem ([SkillRouter, arXiv:2603.22455](https://arxiv.org/abs/2603.22455)). Closes the activation surface that all of [docs/173–178](173-offline-sim-skill-discovery.md) leave open.

**One-paragraph thesis.** **Yes — native LLM mechanisms for skill discovery and activation exist and are the right starting point**, but they cap out around 50–100 skills. Anthropic Claude's three-tier **progressive disclosure** (name+description in system prompt → full body on demand → bundled files when referenced) is the de-facto standard, mirrored by OpenAI Codex, Gemini CLI, and 23 other platforms via the agentskills.io spec. **MCP `tools/list` + `tools/list_changed`** is the protocol-level analogue. **OpenAI / Anthropic / Gemini native function-calling** does the same thing in JSON-schema form. Beyond the ~50-skill threshold, runtime activation must layer in retrieval: **embedding-based retrieval** (Voyager / LlamaIndex `ObjectIndex`), **LLM-as-router** (call the model first to pick), **hierarchical navigation** (Corpus2Skill — *"Don't retrieve, navigate"* via skill-trees), or the strongest production pattern, **retrieve-and-rerank** (SkillRouter: 1.2B parameters, 74.0% Hit@1 on 80,000-skill registries, 13× fewer parameters and 5.8× faster than baselines). The single most striking empirical finding from the SkillRouter paper: **hiding the skill body and routing on description alone causes a 31-44 percentage-point accuracy drop** — descriptions are not enough at scale, full text matters. Polaris and Lyra both currently rely on description-only routing; layering in retrieve-and-rerank is the highest-leverage activation upgrade either harness can ship.

---

## §1 — The activation problem at scale

### When the system prompt budget runs out

Anthropic's Claude Code allocates **≈1% of the context window or 8,000 chars (whichever is larger)** for the skill-descriptions list, with **≤1,536 chars per skill entry** (`description` + `when_to_use` combined). On a 200K context window, that's a hard cap of ~2,000 chars × 100 skills, or roughly **50–100 skills before description truncation kicks in**. A 1M-context model can carry ~500–1,000 entries before truncation, but per-entry signal still degrades.

Beyond that threshold, you have four options:

1. **Truncate descriptions** (Anthropic's default) — cheap, but the SkillRouter paper's 31-44pp accuracy drop applies *exactly* to this regime.
2. **`skillOverrides: name-only`** (also Anthropic's pattern) — list the skill name without description; manual invocation only.
3. **Embedding retrieval** (LlamaIndex `ObjectIndex`, Voyager skill embeddings) — score query-vs-description offline, surface only the top-k descriptions to the model.
4. **Hierarchical navigation** (Corpus2Skill) — distill the catalog into a tree the agent traverses.

The choice depends on catalog size and per-turn latency budget; §3-§5 below survey each.

### What "native LLM tool to find skills" actually means

Three native mechanisms exist in May 2026:

- **Anthropic progressive disclosure** — built into Claude Code, Claude.ai (paid plans), and the Claude Skills API. Description-driven activation; full body loads on demand.
- **OpenAI / Anthropic / Gemini function calling** — built-in tool selection given a list of function schemas. The model picks among supplied JSON schemas based on the user query's semantic match to each schema's `description`.
- **MCP `tools/list`** — protocol-level discovery. Clients send `tools/list`; servers return name + description + inputSchema; the client / LLM picks via `tools/call`. *"Tools in MCP are designed to be model-controlled, meaning the language model can discover and invoke tools automatically based on its contextual understanding."*

All three native mechanisms share **one structural commitment**: the activation surface is *the description*. The body is only consulted *after* the description-driven router fires. This is the load-bearing engineering choice — and it's also where the failure modes live.

---

## §2 — Native LLM mechanisms

### Anthropic Claude Code — progressive disclosure (the reference)

Three-tier loading, *every turn*:

| Tier | What loads | When |
|---|---|---|
| **1 — Discovery (system prompt)** | `name` + `description` (+ optional `when_to_use`) for every installed skill | At session start; refreshes on `~/.claude/skills/` filesystem change |
| **2 — Activation (on-demand)** | Full `SKILL.md` body | When Claude decides the skill is relevant to the current task |
| **3 — Execution (per-reference)** | Bundled scripts + supplementary files | When the body explicitly references them |

From [Anthropic's official docs](https://code.claude.com/docs/en/skills): *"Like a well-organized manual that starts with a table of contents, then specific chapters, and finally a detailed appendix, skills let Claude load information only as needed."*

**Activation logic** is *implicit semantic match* by the model: Claude reads each `description` from its system prompt and decides which is relevant to the user's prompt. There is no explicit retriever; the model's attention does the work. The `when_to_use` field is appended to `description` for the routing decision and counts toward the 1,536-character per-skill cap.

**The character-budget trap.** When the catalog crosses ~50 skills on a 200K-context model, descriptions truncate. From the docs: *"if you have many skills, descriptions are shortened to fit the character budget, which can strip the keywords Claude needs to match your request."* The official recommendation: *"put the key use case first"* and use `skillOverrides: name-only` to drop low-priority skill descriptions.

This is the strongest reference implementation of the *description-driven activation* family, and it is also what most of the OSS ecosystem has now standardised on (per [docs/176](176-skill-discovery-curator-oss-landscape-may-2026.md) Layer A — 26+ platforms read the same SKILL.md frontmatter).

### MCP `tools/list` and `tools/list_changed`

The [Model Context Protocol tools spec](https://modelcontextprotocol.io/docs/concepts/tools) defines:

```jsonc
// Discovery
client → server: { "method": "tools/list", "params": { "cursor": "..." } }
server → client: { "result": { "tools": [{ "name": ..., "description": ..., "inputSchema": ... }], "nextCursor": ... } }

// Selection (LLM-driven, not protocol-mandated)
LLM → client: "use get_weather with location='New York'"

// Invocation
client → server: { "method": "tools/call", "params": { "name": "get_weather", "arguments": { "location": "New York" } } }

// Live catalog updates
server →| client: { "method": "notifications/tools/list_changed" }
```

The protocol explicitly notes: *"Tools in MCP are designed to be model-controlled, meaning that the language model can discover and invoke tools automatically based on its contextual understanding and the user's prompts."*

Critically, MCP supports **paginated discovery** (`cursor` field), letting servers expose thousands of tools without forcing the client to ingest everything at once. Combined with `listChanged`, this is the protocol-level equivalent of Anthropic's filesystem watcher — runtime catalog evolution without reconnect.

### OpenAI / Anthropic / Gemini function calling

The vendor SDKs (`openai`, `anthropic`, `google-genai`) all expose a `tools=[…]` parameter that takes a list of function schemas. The model picks which to call. Internally these are description-driven the same way Anthropic's progressive disclosure is, just with JSON schemas instead of SKILL.md folders.

**Trade-off vs. progressive disclosure**: function-calling carries *no* deferred-body mechanism — every schema (description + inputSchema) is in context the whole time. A 100-tool catalog over function-calling consumes ~10× more system-prompt budget than the same 100 skills via Anthropic's progressive disclosure (since description+schema is heavier than description alone).

Where function-calling wins: typed inputSchema gives the model better arg-extraction; SKILL.md frontmatter has no equivalent typed-arg surface.

---

## §3 — Embedding-based retrieval

When the catalog crosses the native-mechanism threshold, layer in vector retrieval over descriptions. Three reference implementations:

### Voyager — the simplest version

[github.com/MineDojo/Voyager](https://github.com/MineDojo/Voyager) embeds each skill's description with a sentence-encoder, stores the embeddings, and at each next-task lookup computes query × all-skills cosine similarity. Top-k descriptions are surfaced to the LLM for skill use. This is the canonical *embedding-as-router* pattern; everything below is a refinement.

### LlamaIndex — `ObjectIndex` + `ToolRetrieverRouterQueryEngine`

[github.com/run-llama/llama_index](https://github.com/run-llama/llama_index) (49.2k★) provides three primitives:

| Primitive | What it does |
|---|---|
| `RouterQueryEngine` | LLM picks among query engines from descriptions |
| `ToolRetrieverRouterQueryEngine` | **Embedding retrieval over many tools, then LLM rerank.** The dominant production pattern at scale |
| `ObjectIndex` | Vector index over arbitrary objects, including tool schemas — generalises beyond tools |

The `ToolRetrieverRouterQueryEngine` shape is the cleanest open-source reference for **retrieve-then-rerank** (covered in §5 below).

### Voyage AI / BGE / sentence-transformers

The embedding model itself matters. May 2026 production picks:

- **Voyage AI** (`voyage-3.5`) — strongest cross-domain English embeddings.
- **BGE-M3 / BGE-large-zh** (BAAI) — open-weights leaders.
- **OpenAI text-embedding-3-large** — vendor default, well-tested.
- **sentence-transformers `all-mpnet-base-v2`** — open lightweight baseline.

For most production skill catalogs, embedding model choice contributes <5pp Hit@1; the bigger lever is whether you rerank afterwards. (The SkillRouter paper's main contribution — see §5.)

---

## §4 — LLM-as-router

Skip the embedding step entirely: ask the LLM to read all skill descriptions and pick. Two patterns:

### Single-shot LLM-as-router

Prompt the model with `[user query] + [list of all skill descriptions]` and ask which skill applies. Cheap to implement; expensive to run (every turn pays a full LLM call before any actual work). Acceptable when the catalog is <50 skills *and* per-turn latency budget tolerates the extra round-trip.

### Two-shot: route, then execute

The LLM-as-router call is its own turn (often a smaller / faster model), and its decision becomes the system prompt for the main turn. Polaris's `polaris-domains/ml/experiment_manager.py` pattern. Reduces context bloat at the cost of an extra round-trip.

### When to use vs. avoid

LLM-as-router is *the simplest* solution but **doesn't scale**: every turn pays a full pass over the catalog. The SkillRouter paper benchmarks this approach as the "naive baseline" and beats it at 13× lower parameter count.

Use LLM-as-router when:
- Catalog is small (≤50 entries).
- Latency budget is generous.
- Description quality is high — LLM-as-router is more robust to adversarial / poorly-written descriptions than embedding retrieval.

---

## §5 — The 2026 frontier: retrieve-and-rerank at scale

### SkillRouter — the strongest dedicated paper

**[arXiv:2603.22455](https://arxiv.org/abs/2603.22455)** — *SkillRouter: Skill Routing for LLM Agents at Scale* (Zheng et al., Mar 2026).

The paper directly addresses the user's question: at **80,000 candidate skills**, how does an agent route?

**Architecture**: a 1.2B-parameter retrieve-and-rerank pipeline:

1. **Retriever stage** — full-text retrieval over the skill registry; top-K candidates.
2. **Reranker stage** — cross-encoder rerank of the K candidates against the query.

Crucially, the model uses **full-text** of each skill (not just description) for retrieval. The paper's central empirical finding:

> *"Hiding the skill body causes a 31–44 percentage point drop in routing accuracy."*

This is a structural critique of the description-only paradigm Anthropic / agentskills.io / function-calling all rely on. **At scale, descriptions are not enough.**

**Performance**:

- **74.0% Hit@1 on the SkillsBench-derived benchmark** — strongest among baselines.
- **13× fewer parameters** than the strongest baseline.
- **5.8× faster** at runtime.
- Handles **80,000 candidate skills** with heavy overlap.

**Implications for harness designers**:

1. *At catalog sizes <50, native progressive disclosure wins on simplicity.* Don't add retrieve-and-rerank prematurely.
2. *At catalog sizes 50–10,000, layer in embedding retrieval over descriptions* (Voyager / LlamaIndex shape).
3. *At catalog sizes >10,000 (MCP server registries: 25K+; ToolBench: 16K), full-text retrieve-and-rerank is the production-grade answer.*
4. *Description-only routing trades robustness for compactness.* If your catalog descriptions are well-curated, you keep most of the accuracy. If they're community-contributed and uneven (the [docs/175](175-agent-skills-ecosystem-and-security.md) regime), the 31-44pp drop bites.

### Corpus2Skill — *"Don't retrieve, navigate"*

**[arXiv:2604.14572](https://arxiv.org/abs/2604.14572)** — *Don't Retrieve, Navigate: Distilling Enterprise Knowledge into Navigable Agent Skills for QA and RAG* (Sun, Wei, Hsieh, Apr 2026).

The anti-retrieval school. The paper's argument: traditional RAG treats the model as a *passive consumer* of search results — it never sees how the corpus is organised, so it can't backtrack from unproductive paths or combine evidence across branches. Corpus2Skill *distills the corpus into a hierarchical skill directory offline* (LLM-summarised at each level) and lets the agent *navigate* the tree at serve time:

1. Bird's-eye view of the tree at the top.
2. Drill into topic branches via progressively finer summaries.
3. Retrieve full documents by ID once a leaf is reached.

**Performance**: outperforms dense retrieval, RAPTOR, and agentic RAG baselines on WixQA (enterprise customer-support).

**Implications**: when the skill catalog has *natural hierarchy* (vendor org → product → workflow → skill, or domain → sub-domain → task → skill), navigation can beat retrieval. Polaris's domain-shell architecture (`polaris-domains/{ml,biomed,math,physics,social,eng}`) is exactly this hierarchy; the Corpus2Skill pattern would compose well.

### Hybrid: embedding retrieve-and-rerank

The best-of-both-worlds production pattern:

1. **Embedding retriever** (cheap, fast) returns top-K=20 candidates from the full catalog.
2. **LLM reranker** (expensive but K-bounded) picks top-1 or top-3 from the K.
3. **Native progressive disclosure** then loads the body of the top picks.

This is what LlamaIndex's `ToolRetrieverRouterQueryEngine` ships, what SkillRouter ships at higher quality, and what production stacks consuming the Glama / Smithery MCP catalogs converge on.

---

## §6 — Earlier ancestors (the foundational layer)

For completeness, the earlier-generation skill-routing literature:

### Toolformer / ToolkenGPT

**[arXiv:2305.11554](https://arxiv.org/abs/2305.11554)** (NeurIPS 2023, Hao et al.) — *ToolkenGPT: Augmenting Frozen Language Models with Massive Tools via Tool Embeddings.* Each tool is represented as a special token (a "toolken") with a learned embedding. The model emits the toolken inline; tool execution triggers on the toolken. **Routing is implicit in the token-level generation**: no separate retrieval step. Trade-off: requires training the toolken embeddings (not frozen-weights).

This is the **Tier-1 model-attached** approach from [docs/175](175-agent-skills-ecosystem-and-security.md) §1. Conceptually clean; engineering cost is high. SkillRL is the descendant pattern.

### Gorilla

**[arXiv:2305.15334](https://arxiv.org/abs/2305.15334)** (Patil et al., May 2023) — *Gorilla: Large Language Model Connected with Massive APIs.* LLaMA + document retriever for ~1,600 APIs across HuggingFace, TorchHub, TensorHub. The retrieval-augmented training + retrieval-at-inference pattern; reduces hallucination vs GPT-4 for API calls. **Predecessor to SkillRouter's retrieve-and-rerank**.

### ToolLLM / ToolBench

**[OpenBMB/ToolBench](https://github.com/OpenBMB/ToolBench)** — 16,464 REST APIs, 3,451 tools, 126,486 instances. ToolLLaMA uses a BERT-based retriever trained on instruction-API pairs. Paired retriever + DFSDT (depth-first search decision tree) annotation for multi-tool scenarios. **The largest curated tool catalog in OSS** — when designing a retriever for thousands of skills, this is the baseline to beat.

---

## §7 — Trade-offs (the matrix)

|  | Latency / turn | Accuracy at scale | Cost | Robustness to bad descriptions | Catalog-size ceiling |
|---|---|---|---|---|---|
| **Anthropic progressive disclosure** | low | medium (drops past 50) | low | low (descriptions are everything) | ~50–100 |
| **OpenAI/Gemini function calling** | low–medium | medium | low | medium (typed schema helps) | ~100–500 |
| **MCP `tools/list`** | low | medium | low | medium | ~1K (paginated) |
| **LLM-as-router** | high | medium | high (full LLM pass per turn) | **high** (LLM reads sense, not surface) | ~50 |
| **Embedding retrieval** | low | medium-high | low (cached embeddings) | low (cosine on description) | ~10K |
| **Embedding + LLM rerank (LlamaIndex)** | medium | high | medium | medium | ~10K |
| **SkillRouter (full-text retrieve-rerank)** | medium | **highest (74% Hit@1 at 80K)** | medium | **high (uses body, not just description)** | **~100K** |
| **Corpus2Skill (navigate)** | medium-high | high (when hierarchy exists) | medium | high (LLM summaries each level) | ~100K |

The dominant 2026 stack is **layered**: native progressive disclosure for the small / well-curated locally-installed catalog + embedding-retrieval for the medium / community catalog + retrieve-and-rerank for the large / marketplace catalog. Each tier passes its top picks to the next.

---

## §8 — The 2026 strongest stack (recommended)

### Tier 1 — Native progressive disclosure for ≤50 active skills

Adopt [agentskills.io](https://agentskills.io/home) frontmatter. Description ≤1,536 chars, `when_to_use` for clarification, key use case first. Zero retrieval infrastructure required; the model attention does the work.

### Tier 2 — Embedding retrieval for 50–10,000 skills

Layer in vector retrieval over descriptions:

1. Build description embeddings on every skill at registration time.
2. At session start (or per-turn for fast-changing catalogs), retrieve top-K=20 by cosine to query.
3. Surface the K candidates as the *system-prompt skill list* — drop the rest from the prompt.
4. Native progressive disclosure then loads the body of the top picks.

LlamaIndex `ObjectIndex` is the production reference. Voyager's pattern is the minimum viable.

### Tier 3 — Retrieve-and-rerank for ≥10,000 skills

For MCP-registry-scale catalogs:

1. Full-text retriever over skill bodies — not just descriptions — beats description-only by **31-44pp** ([SkillRouter](https://arxiv.org/abs/2603.22455)).
2. Cross-encoder reranker on top-K from the retriever.
3. Top-1 or top-3 surfaced as the active-context skill set.

SkillRouter is the paper-side reference; LlamaIndex `ToolRetrieverRouterQueryEngine` is the closest OSS shape.

### Tier 4 — Hierarchical navigation when hierarchy exists

For domain-shell-organised catalogs (Polaris's `polaris-domains/`, vendor-org-organised catalogs), Corpus2Skill-style tree navigation can beat retrieval. The agent traverses domain → sub-domain → task → skill rather than pulling top-K from a flat index.

### Tier 5 — Specialised model adapters when weights are open

Toolformer / ToolkenGPT / SkillRL — train tool selection into the model itself. Frozen-weights stacks skip this tier.

---

## §9 — Polaris integration slot

Polaris currently routes via `polaris-skills` description matching. The gap to close in v2.5+:

```text
packages/polaris-skills/src/polaris_skills/router/
  embedding_index.py          # NEW — Tier-2 embedding retrieval over
                              # description + when_to_use, top-K cache.
  full_text_index.py          # NEW — Tier-3 full-body retrieval; only
                              # active when catalog crosses 10K.
  reranker.py                 # NEW — cross-encoder rerank over top-K.
  hierarchical_navigator.py   # NEW — Tier-4 navigation over the
                              # polaris-domains/ tree (Corpus2Skill).
  router_chain.py             # NEW — composes Tiers 1-4 with explicit
                              # catalog-size thresholds.
```

Bright-line additions (proposed):

```
BL-ROUTER-DROP-DESCRIPTION    A skill whose description fails the
                              router on N consecutive turns is auto-
                              flagged for re-authoring; its routing
                              priority drops until reviewed.
BL-ROUTER-FULL-TEXT-PRIVACY   Full-text indexing over a skill's body
                              must respect the skill's privacy tier;
                              private bodies are not indexed.
```

This is the highest-leverage activation upgrade Polaris can ship — all of [docs/173-178](173-offline-sim-skill-discovery.md) feed skills into a SkillBank that currently lacks scalable activation.

---

## §10 — Lyra integration slot

Lyra's `lyra-skills/router.py` already does description-driven routing. Tier-2/3/4 layering:

```text
packages/lyra-core/src/lyra_core/skills/
  embedding_router.py         # NEW — Tier-2 retrieval; reuses Lyra's
                              # existing memory/embeddings substrate.
  rerank_router.py            # NEW — Tier-3 cross-encoder rerank.
  domain_navigator.py         # NEW — Tier-4 navigation over the
                              # lyra-skills/packs/ hierarchy.
```

Existing Lyra primitives that compose:

- `lyra-core/memory/` — already ships SQLite FTS5; Tier-3 full-text index can reuse.
- `lyra-skills/router.py` — Tier-1 description router; extend with the embedding fallback.
- `lyra-skills/packs/` — domain-organised packs; natural Tier-4 hierarchy.

---

## §11 — One-paragraph summary

Skill retrieval and activation in 2026 follow a **layered ladder**: at small scale (≤50 skills), Anthropic's progressive disclosure or OpenAI/Anthropic/Gemini native function-calling solves the problem with zero retrieval infrastructure. At medium scale (50–10K), layer in embedding retrieval over descriptions (LlamaIndex `ObjectIndex` shape). At large scale (≥10K, the regime of MCP server registries and full marketplace consumption), the strongest pattern is full-text retrieve-and-rerank — **SkillRouter shows full-body retrieval beats description-only by 31-44 percentage points** at 80K-skill scale. When natural hierarchy exists, Corpus2Skill-style navigation can replace retrieval entirely. Native LLM mechanisms exist (and are the right answer at small scale), but the 2026 production stack layers retrieval and rerank on top once catalog size crosses the system-prompt budget — a threshold Polaris and Lyra are both rapidly approaching as they consume Layer-B skill content from [docs/176](176-skill-discovery-curator-oss-landscape-may-2026.md). The single most actionable upgrade for either harness is shipping the Tier-2 embedding-retrieval router; the rest of the ladder follows once catalog size demands it.

---

## §12 — Where this fits

- [173 — Offline-Sim Skill Discovery](173-offline-sim-skill-discovery.md), [174 — EXIF](174-autonomous-skill-exploration-iterative-feedback.md) — paper-side discovery papers; their output feeds the catalog this doc routes over.
- [175 — Agent Skills Ecosystem & Security](175-agent-skills-ecosystem-and-security.md) — the **description-injection** attack vector; descriptions drive routing, so routing is part of the attack surface. Run the security scanner ([§5](175-agent-skills-ecosystem-and-security.md#5)) over descriptions before they enter the router's index.
- [176 — Open-Source Skills on the Internet](176-skill-discovery-curator-oss-landscape-may-2026.md) — the catalog this doc routes over. Layer B's 25K+ MCP servers are exactly the regime that demands Tier-3 retrieve-and-rerank.
- [177 — Strongest 2026 Techniques Synthesis](177-skills-discovery-curator-strongest-2026-techniques.md) — the activation surface entered the recommended stack as **Pattern 5**; this doc is its full treatment.
- [178 — Online Skill Discovery & Curation On-the-Go](178-online-skill-discovery-and-curation-on-the-go.md) — runtime catalog evolution (`tools/list_changed`, filesystem watcher, Hub-pull) requires the router to refresh its index without bouncing the session; the patterns here support it.

## §13 — Reading list

In recommended order:

1. **Anthropic Claude Code skills doc** — [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills). The reference implementation of progressive disclosure.
2. **MCP tools spec** — [modelcontextprotocol.io/docs/concepts/tools](https://modelcontextprotocol.io/docs/concepts/tools). Protocol-level discovery + invocation contract.
3. **agentskills.io** — [agentskills.io/home](https://agentskills.io/home). The open standard.
4. **SkillRouter** — [arXiv:2603.22455](https://arxiv.org/abs/2603.22455). The strongest dedicated paper at scale.
5. **Corpus2Skill** — [arXiv:2604.14572](https://arxiv.org/abs/2604.14572). Don't retrieve, navigate.
6. **LlamaIndex** — [run-llama/llama_index](https://github.com/run-llama/llama_index). `ObjectIndex` + `ToolRetrieverRouterQueryEngine` reference.
7. **Voyager** — [MineDojo/Voyager](https://github.com/MineDojo/Voyager). Embedding-as-router minimum viable pattern.
8. **ToolkenGPT** — [arXiv:2305.11554](https://arxiv.org/abs/2305.11554). Toolken-as-embedding ancestor.
9. **Gorilla** — [arXiv:2305.15334](https://arxiv.org/abs/2305.15334). Retrieval-augmented API LLM ancestor.
10. **ToolLLM / ToolBench** — [OpenBMB/ToolBench](https://github.com/OpenBMB/ToolBench). 16K-API scale baseline.
