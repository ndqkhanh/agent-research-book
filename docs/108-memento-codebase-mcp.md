# 108 — Memento Codebase Walkthrough: Planner-Executor Loop, MCP Tool Servers, Interpreters, and the Operational Stack

**Repository.** https://github.com/Agent-on-the-Fly/Memento (canonical) — mirror at https://github.com/Memento-Teams/Memento — license MIT — 2.4k stars, 286 forks at survey time — Python 3.11+ — heavy reuse of Camel-AI toolkits and interpreters.

**One-line definition.** The Memento repo is the working reference implementation of the [106](106-memento-paper-theory.md) M-MDP framework: a `client/` package that runs a two-stage planner→executor agent over a `server/` package of MCP tool servers (search, documents, code, image, video, excel, math, web crawl), backed by interchangeable `interpreters/` (Docker, E2B, subprocess, internal-Python) for code execution and a `memory/` package that owns the case bank plus the retriever training pipeline — all wired together with FastMCP as the protocol glue and `uv` as the recommended package manager.

## Why this codebase matters

Most agent-research papers ship reference code that runs the headline benchmark and nothing else. The Memento repo is unusual in two respects. First, it ships a *fully operational* tool stack — search, documents, code, image, video, excel, math, crawler — that is independently useful as an MCP server suite even if you discard the agent entirely. Second, it cleanly separates the four orthogonal concerns of a deep-research agent: **planning** (case-conditioned strategy), **executing** (tool selection), **environment** (MCP servers), and **memory** (case bank + retriever). Each concern has its own directory, its own configuration, and its own lifecycle. That separation is exactly what harness engineers need to copy.

It is also the cleanest published example of an MCP-native agent loop. Most existing agent codebases (LangChain, AutoGen, CrewAI) use bespoke tool registries and adapt to MCP via wrappers. Memento was MCP-first from day one, which means the action space ([106](106-memento-paper-theory.md)) is *literally* the MCP tool registry, with no impedance mismatch.

## Problem it solves

Concrete operational problems the codebase addresses, each one a section below:

1. **MCP-native action space.** Tools are MCP servers, not Python callables glued into the agent. The agent talks to tools the same way Claude Desktop or Cursor would.
2. **Pluggable code execution.** Four interpreter backends (Docker, E2B sandbox, subprocess, internal in-process) cover the security/isolation/latency trade-off space.
3. **Open and closed-source executor.** Default uses GPT-4 (planner) + o3 (executor), but `agent_local_server.py` runs against a local vLLM endpoint, allowing fully open-source reproduction.
4. **Memory ops separated from agent ops.** `memory/` has its own training script, its own checkpoint dir, and its own JSONL store, so the retriever can be retrained without touching the rest of the stack.
5. **Reproducible setup.** `uv sync`, `searxng-docker/`, and a complete `.env` template make the system run on a fresh machine in under an hour.

## Core idea in one paragraph

Two stacks talk over MCP. The **agent stack** (`client/`) is the LLM-side: a planner that consumes a query plus retrieved cases and emits a structured plan, and an executor that consumes plan steps and emits tool calls. The **tool stack** (`server/`) is the environment side: a fleet of FastMCP servers, each owning a domain (search, code, documents, etc.), each independently restartable. The two stacks are wired by FastMCP. The case memory (`memory/`) lives in JSONL on disk; a separate training script periodically retrains a dual-encoder retriever on the accumulated cases. The `interpreters/` package is the code-execution backend the `code_agent.py` server delegates to. Configuration is one `.env` file. That is the entire operational surface.

## Mechanism (step by step)

### 1. Repository layout

```text
Memento/
├── client/                           # the agent-side
│   ├── agent.py                      # main hierarchical agent (planner+executor)
│   ├── no_parametric_cbr.py          # non-parametric retrieval variant
│   ├── parametric_memory.py          # parametric (learned) retrieval variant
│   └── agent_local_server.py         # local vLLM executor variant
│
├── server/                           # the tool-side (MCP servers)
│   ├── code_agent.py                 # code execution + workspace
│   ├── search_tool.py                # SearxNG-backed web search
│   ├── serp_search.py                # SerpAPI-backed web search
│   ├── documents_tool.py             # PDF / Office / image / audio / video parsing
│   ├── image_tool.py                 # image captioning + visual analysis
│   ├── video_tool.py                 # video narration
│   ├── excel_tool.py                 # spreadsheet ops
│   ├── math_tool.py                  # symbolic / numeric math
│   ├── craw_page.py                  # web page crawling
│   └── ai_crawler.py                 # query-aware compression of crawl output
│
├── interpreters/                     # code-execution backends
│   ├── docker_interpreter.py         # docker-isolated python
│   ├── e2b_interpreter.py            # E2B cloud sandbox
│   ├── internal_python_interpreter.py# in-process exec
│   └── subprocess_interpreter.py     # subprocess-isolated python
│
├── memory/                           # the case bank + retriever
│   ├── parametric_memory.py          # inference-time retriever
│   ├── train_memory_retriever.py     # retriever training entry point
│   ├── np_memory.py                  # non-parametric utilities
│   ├── memory.jsonl                  # case storage (one JSON per line)
│   ├── training_data.jsonl           # contrastive training pairs
│   └── ckpts/                        # retriever checkpoints
│
├── searxng-docker/                   # docker-compose for self-hosted search
└── data/                             # sample cases
```

The directory split is the operational decomposition: each subtree corresponds to a separately deployable concern. You can run `client/` on a laptop while `server/` runs on a beefy box; you can swap one MCP server without touching the others; you can retrain the retriever offline without taking the agent down.

### 2. The agent loop — `client/agent.py`

The main entry point is `client/agent.py`, which wires:

```text
1. Load .env (API keys, model names, memory paths, MCP server URLs).
2. Connect to all MCP servers in server/ (FastMCP clients).
3. Initialise memory (non-parametric or parametric, per config).
4. Loop:
     a. Read query from interactive prompt or task source.
     b. cases = memory.retrieve(query, K=4)
     c. plan  = planner_LLM(query, cases)
     d. for step in plan:
          while not step_done:
            tool_call = executor_LLM(step, recent_obs, mcp_registry)
            obs       = mcp_call(tool_call)
            recent_obs.append(obs)
     e. answer = aggregate(plan_outputs)
     f. reward = grade(answer, ground_truth) if available
     g. memory.maybe_write((s_T, a_T, r_T))
```

Three things to note about this loop:

- **The planner is called once.** It produces the entire plan up front. Re-planning is supported but rare; the executor does most of the iteration.
- **The executor is a tight loop.** Per plan step, it can issue many tool calls; only the high-level step boundaries are visible to the planner.
- **The MCP registry is dynamic.** The agent introspects what tools are available at startup, so adding a new MCP server is a matter of pointing the agent at it — no agent-side code change.

### 3. The two retrieval variants — `no_parametric_cbr.py` vs `parametric_memory.py`

The two variants are interchangeable:

- **`no_parametric_cbr.py`** runs the simplest baseline. At startup it loads `memory.jsonl`, encodes every case with an off-the-shelf embedding model, and serves cosine-similarity retrieval. No training, no checkpoints. Used as the cold-start variant and the SimpleQA-grade baseline.
- **`parametric_memory.py`** loads a trained dual-encoder retriever from `RETRIEVER_MODEL_PATH=../memory/ckpts/retriever/best.pt`, encodes the query once per call, scores against the precomputed case bank, and returns the soft-Q top-K. Default `MEMORY_TOP_K=8` (with prompt-side truncation to K=4 for the planner).

Switching between them is a single env var (`MEMORY_BACKEND=parametric` or `non_parametric`) and a different entry-point script.

### 4. The local-LLM variant — `agent_local_server.py`

Runs against a vLLM (or any OpenAI-compatible) endpoint instead of the OpenAI cloud. The interface is identical — same planner contract, same MCP tool calls, same memory layer. The point is reproducibility: HLE / GAIA results in the paper used GPT-4 + o3, but the framework is model-agnostic and the local-server entry point is for verifying the agent works on a fully open stack.

### 5. The MCP tool servers — `server/*.py`

Each server in `server/` is an independent FastMCP process. Highlights:

| Server | Purpose | Notes |
|---|---|---|
| `code_agent.py` | Python execution + workspace | Delegates to `interpreters/` per config; owns a working directory the planner can read/write. |
| `search_tool.py` | SearxNG search | Self-hosted, privacy-friendly. Requires `searxng-docker/` running. |
| `serp_search.py` | SerpAPI search | Cloud alternative; requires `SERPAPI_API_KEY`. |
| `documents_tool.py` | Multi-format ingestion | PDF, Office (Word/Excel/PowerPoint), images (with OCR), audio, video. Uses Chunkr / Jina / AssemblyAI under the hood. |
| `image_tool.py` | Vision | Captioning, visual question-answering, simple analysis. |
| `video_tool.py` | Video narration | Extracts frames + audio, narrates the sequence. |
| `excel_tool.py` | Spreadsheet ops | Read/write/manipulate XLSX. |
| `math_tool.py` | Symbolic + numeric math | Wraps SymPy / NumPy / SciPy. |
| `craw_page.py` | Web page fetch | Plain HTML grab + clean extraction. |
| `ai_crawler.py` | Query-aware compression | Crawls and summarises in one shot, cutting downstream token cost. |

The `documents_tool.py` is the unsung hero of GAIA performance: GAIA's task distribution leans on file ingestion (PDFs, spreadsheets, audio clips), and a tool that handles all of those uniformly raises the floor.

The two crawl tools (`craw_page.py` raw, `ai_crawler.py` compressed) embody a concrete cost-control pattern: the raw crawl is for verification and citation, the compressed crawl is for in-context reasoning. Choosing between them is a planner-level decision based on whether the answer needs verbatim source text.

### 6. The interpreters — `interpreters/*.py`

The `code_agent.py` server delegates to one of four interpreters, chosen by config. The four cover the security/latency/portability trade-off space:

| Interpreter | Isolation | Latency | Portability |
|---|---|---|---|
| `internal_python_interpreter.py` | none (in-process) | lowest | trivially portable |
| `subprocess_interpreter.py` | OS-level, single-machine | low | portable |
| `docker_interpreter.py` | container, single-machine | medium | needs Docker |
| `e2b_interpreter.py` | cloud sandbox | network-bound | cloud-only |

The default for benchmark runs is the Docker interpreter — it isolates code execution without the network round-trip of E2B and is the standard choice for hosted deployments.

### 7. The memory training pipeline — `memory/train_memory_retriever.py`

The retriever training entry point. Concrete invocation from the README:

```bash
cd memory
python train_memory_retriever.py \
  --train training_data.jsonl \
  --output_dir ./ckpts/retriever \
  --use_plan --val_ratio 0.1 \
  --batch_size 32 --lr 2e-5 --epochs 10 --save_best
```

Pipeline:

1. Read `training_data.jsonl` (queries with positive and negative case IDs).
2. Initialise a dual-encoder backbone (typically a sentence-transformer or BERT-class model).
3. Optionally condition the query encoder on the planner's plan (`--use_plan`) so the retriever surfaces plan-relevant cases, not just query-relevant ones.
4. Contrastive loss with in-batch negatives plus hard negatives drawn from `memory.jsonl` failed cases.
5. 10% validation split (`--val_ratio 0.1`); save best by validation loss (`--save_best`).
6. Output checkpoint at `./ckpts/retriever/best.pt`, which `parametric_memory.py` loads at agent startup.

### 8. Configuration — `.env`

Single env file owns the operational surface. Reproduced in the README:

```ini
# OpenAI Configuration
USE_AZURE_OPENAI=False
OPENAI_API_KEY=your_key
OPENAI_BASE_URL=https://api.openai.com/v1

# Tool APIs
CHUNKR_API_KEY=your_key       # documents_tool.py
JINA_API_KEY=your_key          # ai_crawler.py
ASSEMBLYAI_API_KEY=your_key    # video / audio transcription

# Memory (parametric mode)
MEMORY_JSONL_PATH=../memory/memory.jsonl
TRAINING_DATA_PATH=../memory/training_data.jsonl
RETRIEVER_MODEL_PATH=../memory/ckpts/retriever/best.pt
MEMORY_TOP_K=8
MEMORY_MAX_POS_EXAMPLES=8
MEMORY_MAX_NEG_EXAMPLES=8
```

`MEMORY_TOP_K` is the *retrieval* top-K (how many candidates to score); the planner-prompt-side truncation to 4 is typically applied separately. The two MAX_*_EXAMPLES caps shape contrastive batches during training.

### 9. Quick-start operational sequence

```bash
# 1. Clone
git clone https://github.com/Agent-on-the-Fly/Memento && cd Memento

# 2. Install (uv recommended)
curl -LsSf https://astral.sh/uv/install.sh | sh
uv sync && source .venv/bin/activate

# 3. PyTorch + CUDA (for parametric memory)
pip install torch torchvision torchaudio \
  --index-url https://download.pytorch.org/whl/cu121

# 4. System deps
brew install ffmpeg                              # macOS
# sudo apt-get install ffmpeg                    # Linux
crawl4ai-setup
playwright install

# 5. Self-hosted search
cd searxng-docker && docker compose up -d && cd ..

# 6. Configure
cp .env.example .env
# fill in OPENAI_API_KEY and tool API keys

# 7. Train retriever (optional, parametric mode)
cd memory
python train_memory_retriever.py \
  --train training_data.jsonl \
  --output_dir ./ckpts/retriever \
  --use_plan --val_ratio 0.1 \
  --batch_size 32 --lr 2e-5 --epochs 10 --save_best
cd ..

# 8. Run
cd client && python parametric_memory.py        # parametric mode
# or: python agent.py                           # interactive default
# or: python no_parametric_cbr.py               # non-parametric mode
# or: python agent_local_server.py              # local-vLLM mode
```

### 10. Operational dependencies (for reference)

Core packages from `pyproject.toml`:

- `fastmcp` — Model Context Protocol server/client framework.
- `openai` — GPT-4 + o3 integration (planner + executor by default).
- `anthropic` — optional Claude integration.
- `torch` — neural retriever backbone.
- `crawl4ai` — web extraction.
- `ffmpeg-python` — video/audio processing.
- `serpapi` — search backend.
- `pandas` — data manipulation.
- `pydantic` — configuration management.

External services typically connected: SearxNG (self-hosted) or SerpAPI (cloud) for search, Chunkr for document parsing, Jina for embeddings/reranking, AssemblyAI for transcription, OpenAI for planner + executor models.

## Empirical anchors

The repo's own evaluation table (reproduced from the README):

| Benchmark | Metric | Result |
|---|---|---|
| GAIA | Val (Pass@3 Top-1) | **87.88%** |
| GAIA | Test | **79.40%** |
| DeepResearcher | F1 / PM | **66.6% / 80.4%** |
| DeepResearcher | OOD gain | **+4.7 to +9.6 absolute** |
| SimpleQA | Accuracy | **95.0%** |
| HLE | Performance Metric | **24.4%** |

Combined with the K=4 memory-size sweet spot and the "small high-quality memory beats large banks" finding ([107](107-memento-cbr-memory.md)), these are the operational targets a fresh deployment should aim for.

## Variants and counter-arguments addressed

- **"FastMCP is one of several MCP runtimes."** True. The architecture is runtime-agnostic; replace `fastmcp` with another MCP client/server pair and the agent stack is unchanged.
- **"Why GPT-4 + o3?"** The planner is reasoning-heavy (cases + query → plan); o3 was the strongest reasoning model at submission time. The executor is tool-call-heavy and benefits less from heavyweight reasoning. The split is configurable.
- **"Why Docker for code execution?"** Strikes the best isolation/latency balance for benchmark runs. E2B is preferred for production cloud deployments. Subprocess is sufficient for trusted internal scripts.
- **"SearxNG vs SerpAPI?"** SearxNG is privacy-friendly and free; SerpAPI is more reliable for production loads. Both are pluggable.
- **"Where is the planner's prompt?"** Inlined in `client/agent.py`. The repo deliberately keeps prompts editable as code rather than abstracting them behind a config layer.
- **"Why JSONL for memory storage?"** Append-only, human-readable, trivially backed up, easily transformed by Unix tools. Heavyweight DBs are overkill for the bank sizes that matter.

## Failure modes and limitations

1. **OpenAI dependency in defaults.** Out-of-the-box runs require `OPENAI_API_KEY`. The `agent_local_server.py` variant exists for open-source reproduction but is less battle-tested than the cloud path.
2. **External tool keys.** Chunkr, Jina, AssemblyAI, optionally SerpAPI — each adds an operational dependency. Self-hosting (SearxNG, local docling, etc.) reduces this but multiplies setup time.
3. **Documents tool fragility.** Multi-format parsing is intrinsically lossy; PDF tables and complex layouts are common failure points and contribute disproportionately to GAIA Level-3 errors.
4. **Memory consistency under concurrent writes.** `memory.jsonl` is append-only but two concurrent agent processes appending simultaneously will produce no atomicity guarantee. Single-writer assumption holds.
5. **Retriever staleness.** The retriever is offline-trained; cases added between retraining passes are not reflected in retrieval quality (though they are stored). The harness must schedule retraining on a cadence.
6. **No native multi-agent orchestration.** Memento is a single-agent system. Multi-agent extensions are downstream work.
7. **Limited fully-open-source coverage.** The repo's own README acknowledges that open-source executor pipelines are less validated.
8. **GAIA Level 3 long-horizon failures.** Compounding errors on multi-step Level 3 tasks remain unsolved; the (s_T, a_T, r_T) compression discards the multi-step trajectory shape needed for those tasks.

## When to use, when not

**Use the codebase as-is when** you want a working deep-research agent with state-of-the-art GAIA / DeepResearcher / SimpleQA performance, you have OpenAI access (or are willing to replicate against vLLM), and you can host SearxNG + the small retriever locally. Total setup time on a fresh machine: ~1 hour. It is also the cleanest reference for "how to wire MCP into an agent harness" if you are building a new harness from scratch.

**Use it as a reference architecture but not as production code when** you need multi-tenant, audit-logged, security-hardened deployment. The repo is research-grade — single-user, single-writer, minimal observability. Production deployment requires wrapping in your own auth + tracing + persistence layer.

**Skip it when** your domain doesn't benefit from CBR (i.i.d. tasks, no reward signal), or your tool requirements diverge significantly from the bundled set, or you cannot host the retriever offline.

## Implications for harness engineering

- **Treat MCP as the agent's action space.** Memento's `client/` ↔ `server/` split is the cleanest published example of this. Existing harnesses ([07-model-context-protocol](07-model-context-protocol.md), [42-langchain-deep-agents](42-langchain-deep-agents.md)) should adopt the same boundary: agent code talks only MCP, never imports tool code directly.
- **Separate planner from executor at the model level.** GPT-4 (planning) + o3 (executing) is a clean factoring: reasoning model upstream, tool-call model downstream. Most current harnesses use one model for both. The Memento split is the upgrade path. ([16-plan-and-solve](16-plan-and-solve.md), [02-subagent-delegation](02-subagent-delegation.md) cover the abstract pattern.)
- **Memory has its own deploy lifecycle.** The `memory/` package is a separately deployable artefact: training pipeline, checkpoint dir, JSONL store. Production harnesses should give memory the same operational maturity as any model artefact — versioned, A/B-able, rollbackable. See [09-memory-files](09-memory-files.md), [72-claude-mem-persistent-memory-compression](72-claude-mem-persistent-memory-compression.md).
- **Pluggable interpreters as a pattern.** The four-interpreter ladder (in-process → subprocess → docker → e2b) is a useful template for any harness that runs untrusted code. Defaulting to subprocess for dev and Docker for prod is the operational sweet spot. ([04-skills](04-skills.md), [05-hooks](05-hooks.md) intersect this concern.)
- **Self-hosted search matters more than you think.** SearxNG running locally removes a major rate-limit + cost variable from the agent loop. Production harnesses should consider self-hosting search rather than depending on SerpAPI for every query.
- **`documents_tool.py` is a candidate atomic skill.** The multi-format ingestion server should be a *standalone* MCP server in any deep-research harness, reusable across agents. This aligns with the [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md) thesis: build durable, composable tool servers, not bespoke per-agent integrations.
- **AI-aware crawling cuts cost.** The `ai_crawler.py` query-aware compression pattern (crawl → summarise inline → return compressed) is a 5–10× token-cost reduction on document-heavy traces. Worth porting independently to any harness that does heavy web crawling.
- **`.env` is the API.** A single env file containing every operational knob (model names, API keys, memory paths, top-K) is the right level of configuration for an agent harness. Hierarchical configs (yaml + jinja + secrets manager) are over-engineering at this scale.
- **Prompts as code.** Memento inlines prompts in `client/agent.py` rather than externalising them. This is a controversial choice but sound for research code: prompt edits are reviewable in git diff, and there is no out-of-band "prompt store" to fall out of sync. Production harnesses should adopt the same convention until prompt complexity demands otherwise.

The one-sentence takeaway: **Memento's repo is the reference implementation for an MCP-native, planner-executor, case-bank-driven agent — copy the directory layout, adapt the prompts, and you have a deep-research harness.**

## See also

- [106-memento-paper-theory](106-memento-paper-theory.md) — the M-MDP framework this codebase implements.
- [107-memento-cbr-memory](107-memento-cbr-memory.md) — the case-based reasoning memory mechanism.
- [109-memento-results-and-harness](109-memento-results-and-harness.md) — benchmarks, ablations, integration patterns.
- [07-model-context-protocol](07-model-context-protocol.md) — MCP as the standardised tool interface.
- [42-langchain-deep-agents](42-langchain-deep-agents.md) — the closest open-source comparable.
- [52-dive-into-open-claw](52-dive-into-open-claw.md) — alternative MIT-licensed agent harness.
- [62-everything-claude-code](62-everything-claude-code.md) — Claude Code's harness for comparison.
- [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md) — atomic-skill composition matches Memento's MCP server design.
