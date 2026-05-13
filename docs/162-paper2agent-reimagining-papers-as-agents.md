# 162 — Paper2Agent: Reimagining Research Papers as Interactive AI Agents

**Repository.** https://github.com/jmiao24/Paper2Agent — Bash entry point with Python sub-tools — author: **Jiacheng Miao** (Stanford-affiliated; first author of the Paper2Agent paper). Companion paper: **Miao, Davis, Pritchard & Zou — *Paper2Agent: Reimagining Research Papers As Interactive and Reliable AI Agents* — arXiv:2509.06917 [cs.AI] — September 2025**. Companion benchmark repo: https://github.com/jmiao24/Paper2AgentBench. Connectable hosted Paper-MCP servers on HuggingFace Spaces: AlphaGenome, Scanpy, TISSUE.

**One-line definition.** Paper2Agent is a **multi-agent AI system that automatically transforms research papers into interactive AI agents** — given a paper's GitHub codebase, the system runs a 5-step pipeline that scans tutorials, executes them in an isolated environment, extracts each tutorial's analytical workflow into a reusable Python tool, packages the tools into an MCP (Model Context Protocol) server, and produces a Claude-Code-loadable "Paper Agent" that exposes the paper's methods as queryable tools — all in 30 minutes to 3+ hours per paper, at roughly $15 per complex repository. The result: instead of reading a paper and re-implementing its method, a researcher can spin up an MCP-backed agent that *runs the paper's actual code* on the researcher's data, with a natural-language query interface.

## Why this is one of the most important agent projects of 2025–2026

Most "research agents" sit on the wrong side of an unproductive divide. Information-DR agents ([158](158-deep-research-agents-survey-huang-et-al.md), [159](159-deep-research-survey-zhang-et-al.md), [155](155-feynman-multi-agent-research-harness.md)) **read** papers and **summarise** them. Experimentation-DR agents ([160](160-deep-researcher-agent-24x7.md)) **run new** experiments. Neither directly **uses** the methods that papers describe.

Paper2Agent occupies a third position: it converts published research papers — which already contain implementations, tutorials, and validated workflows — into *invocable tools* that an LLM can call. The paper's method is no longer something to read about; it is something to *run on your data via natural language*.

The implications for the deep-research-agent stack are substantial:

1. **Transforms papers from passive artefacts into active capability.** A paper's `git clone && pip install && jupyter notebook` workflow gets compressed into "ask the agent to do the analysis."
2. **Generates MCP-compatible tools at scale.** Each paper produces an MCP server. Agents that consume MCP tools (Claude Code, Feynman, OpenClaw, custom builds) inherit the paper's methods automatically.
3. **Brings the research literature into the agent ecosystem.** A literature with N papers, processed through Paper2Agent, becomes a tool registry of N-papers' worth of validated methods.
4. **Demonstrates the maturity of agent-builds-agent self-bootstrapping.** Paper2Agent is itself a multi-agent system whose 9 specialised agents construct *another* agent (the Paper Agent for that specific paper) by reading code and tutorials.

For the harness-engineering canon, this is the operational counterpart to the "skills as portable artefacts" thesis ([157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md)). Where Ctx2Skill ([154](154-ctx2skill-self-evolving-context-skills.md)) constructs context-specific skill *prompts*, Paper2Agent constructs context-specific tool *implementations* (Python code wrapped as MCP). Both are forms of automated capability construction; Paper2Agent's contribution is doing it for *executable* capabilities, not just textual guidance.

## Architecture overview

```
INPUT:  --project_dir <DIR>  --github_url <URL of paper's GitHub repo>
                                                    │
                                                    ▼
       ┌─────────────────────────────────────────────────────────────┐
       │ Paper2Agent.sh (orchestrator entry point)                    │
       └─────────────────────────────────────────────────────────────┘
                                                    │
        ┌──────────────┬──────────────┬─────────────┼───────────────┐
        ▼              ▼              ▼             ▼               ▼
   step1_prompt  step2_prompt   step3_prompt   step4_prompt   step5_prompt
   (Tutorial    (Tutorial      (Tool          (MCP server    (Coverage +
    Scanner)    Executor)      Extraction)    Creation)      Quality
                                                              Analysis)
        │              │              │             │               │
        ▼              ▼              ▼             ▼               ▼
   tutorial-    tutorial-      tutorial-tool  test-verifier  benchmark-
   scanner.md   executor.md    extractor-      improver.md    extractor.md
                               implementor.md
                               environment-                   benchmark-
                               python-                        judge.md
                               manager.md
                                                              benchmark-
                                                              reviewer.md

                                                              benchmark-
                                                              solver.md

OUTPUT:
  src/<repo_name>_mcp.py            ← Generated MCP server
  src/tools/<tutorial>.py           ← Extracted tool implementations
  <repo_name>-env/                  ← Isolated Python environment
  reports/coverage/                 ← Coverage analysis
  reports/quality/pylint/           ← Code quality analysis
  notebooks/<tutorial>/             ← Executed tutorials
  tests/                            ← Test suite for tools
  reports/benchmark_*.csv           ← Optional benchmark results
```

**9 specialised agents** in `Paper2Agent/agents/`:

1. `tutorial-scanner.md` — discovers tutorial materials in the repo
2. `tutorial-executor.md` — runs each tutorial in an isolated env
3. `tutorial-tool-extractor-implementor.md` — converts tutorial workflows into reusable tools
4. `environment-python-manager.md` — manages Python envs and dependencies
5. `test-verifier-improver.md` — verifies and improves the generated test suite
6. `benchmark-extractor.md` — extracts benchmark questions from tutorials
7. `benchmark-judge.md` — judges benchmark answers
8. `benchmark-reviewer.md` — reviews benchmark assessments
9. `benchmark-solver.md` — solves benchmark questions using the generated agent

Each agent is a Markdown file with frontmatter (name, description, model, color) and a detailed system prompt. The same `agents-as-Markdown` pattern as Feynman ([155](155-feynman-multi-agent-research-harness.md)), Deep Researcher Agent ([160](160-deep-researcher-agent-24x7.md)), and HeavySkill ([156](156-heavyskill-parallel-reasoning-deliberation.md)).

## The 5-step pipeline in detail

### Step 1 — Tutorial Scanner

Agent: `tutorial-scanner`. Mission: identify tutorials in the codebase whose code is *valuable enough to be wrapped as a tool*.

Core principles (from the agent definition):

> 1. **Complete Evaluation**: Read each file end-to-end before making determinations
> 2. **Conservative Classification**: When uncertain, lean toward "exclude-from-tools"
> 3. **Quality Standards**: Only include tutorials with runnable, self-contained, reusable functionality
> 4. **Documentation Accuracy**: Document reasoning clearly
> 5. **Python Script Priority**: Include `.py` only when no `.ipynb` or `.md` tutorials exist
> 6. **Template Exclusion**: Never scan `templates/` directory
> 7. **Legacy Filtering**: Exclude tutorials with "legacy", "deprecated", "outdated", or "old" in title/filename
> 8. **Systematic Approach**: Start with `docs/**` for authoritative content

Output: `tutorial-scanner.json` with each tutorial classified as `include-in-tools` or `exclude-from-tools` plus reasoning.

The conservative-classification bias is operationally critical. Including tutorials of marginal quality leads to broken tools downstream; excluding them limits coverage. The agent is told: when in doubt, exclude.

### Step 2 — Tutorial Executor

Agent: `tutorial-executor`. Runs each `include-in-tools` tutorial in the isolated `<repo_name>-env` Python environment. Captures execution outputs, plots, errors. Validates that the tutorial works end-to-end before tools are extracted.

This step is essential because the next step (tool extraction) requires the tutorial to be *demonstrably runnable*. Tools extracted from non-runnable tutorials would be dead weight.

Output: `executed_notebooks.json` summarising which tutorials ran successfully. For each successful run, the executed notebook is saved with all outputs preserved.

### Step 3 — Tool Extraction & Implementation

Agent: `tutorial-tool-extractor-implementor`. The most consequential step. Transforms each runnable tutorial into a Python module of reusable tools.

The agent's core principles (verbatim, because they encode the design philosophy):

> 1. **Applied to new inputs**: Every function must accept user-provided input. No hardcoded values.
> 2. **User-Centric Design**: The function should be designed for real-world usage, not just tutorial reproduction.
> 3. **Exact Reproduction**: When run with tutorial data, tools must produce identical results to the original tutorial.
> 4. **Clear Boundaries**: Each tool performs one well-defined scientific analysis task.
> 5. **Production Quality**: All code must be immediately usable without modification.
> 6. **No Mock**: Never use mock data or mocks in the code.
> 7. **Each tutorial file should be converted to exactly one Python file.**
> 8. **The order of tools should match the order of sections in the tutorial.**
> 9. **Primary Use Case Focus**: Tools should be designed for the intended real-world use case, not restricted to tutorial demonstration.
> 10. **NEVER ADD PARAMETERS NOT IN TUTORIAL**: Function calls must exactly match the tutorial. If the tutorial shows `sc.tl.pca(adata)`, DO NOT add parameters like `n_comps`. Only parameterize values that were explicitly set in the tutorial code.
> 11. **PRESERVE EXACT TUTORIAL STRUCTURE**: Do not create generalized patterns or artificial logic.

Two things are striking:

**Discipline #10 — "NEVER ADD PARAMETERS NOT IN TUTORIAL"** — is the architectural constraint that makes the tool-extraction *trustworthy*. If the agent could invent parameters, the resulting tool would diverge from the paper's validated workflow. By forcing the agent to preserve exactly what the tutorial specified, the tool retains the paper's empirical guarantees.

**Discipline #6 — "No Mock"** — is the same kind of integrity rule as Feynman's "URL or it didn't happen" (at [155](155-feynman-multi-agent-research-harness.md)). The whole pipeline collapses if mock data is introduced anywhere.

Output for each tutorial: one Python file in `src/tools/<tutorial_file_name>.py` with the extracted tool functions. Each function:
- Has a docstring describing its scientific purpose.
- Accepts user-provided data as parameters.
- Calls the underlying paper's library functions exactly as the tutorial did.
- Returns a structured output (often a dict, dataframe, or trained model).

### Step 4 — MCP Server Creation

Agent: implicit (no dedicated `step4-*.md` agent shown — but the system prompt orchestrates this step). Wraps the extracted tools into a Model Context Protocol server.

The result: `src/<repo_name>_mcp.py` — a single file that exposes every extracted tool over the MCP JSON-RPC protocol. This file is what gets loaded into Claude Code (or any MCP-compatible runtime) as the "Paper MCP".

```bash
# To use in Claude Code:
claude mcp list                # Verify the MCP is registered
\mcp                           # Inside Claude Code; shows MCP tools
```

The MCP server can be hosted locally or deployed to HuggingFace Spaces (the AlphaGenome / Scanpy / TISSUE examples are deployed publicly). Once deployed, *any* MCP-compatible agent — Claude Code, Feynman, OpenClaw, custom — can use the paper's methods.

### Step 5 — Coverage and Quality Analysis

Agent: `test-verifier-improver`. Generates and runs a test suite against the extracted tools. Produces:

- **Coverage reports**: `reports/coverage/` (XML, JSON, HTML) — code coverage metrics from `pytest-cov`.
- **Code quality reports**: `reports/quality/pylint/` — pylint analysis with per-file scores and detailed issue breakdown.
- **Combined report**: `reports/coverage_and_quality_report.md`.

The agent goes beyond "did the tests pass?" — it identifies tests that need improvement and rewrites them.

## Optional benchmark mode

The `--benchmark` flag activates four additional agents:

- `benchmark-extractor`: extracts benchmark questions from the executed tutorials. Output: `reports/benchmark_questions.csv`.
- `benchmark-solver`: uses the *generated agent* to answer the benchmark questions. Tests whether the Paper Agent reproduces the paper's results.
- `benchmark-judge`: judges the answers' correctness against tutorial expectations.
- `benchmark-reviewer`: reviews the judging for fairness and quality.

Output: `reports/benchmark_results.csv`. This is the answer to "does the Paper Agent actually work as a useful interface to the paper?"

The benchmark methodology, with companion repo Paper2AgentBench, is a substantive contribution: it provides an empirical foundation for evaluating *the quality of automated paper-to-agent translation* — not just whether the pipeline runs, but whether the resulting agent faithfully reproduces the paper's capabilities.

## Hosted Paper-MCP examples

Three pre-built Paper Agents are publicly accessible:

- **AlphaGenome MCP** (https://Paper2Agent-alphagenome-mcp.hf.space) — genomic data interpretation. Example query: *"Analyze heart gene expression data with AlphaGenome MCP to identify the causal gene for the variant chr11:116837649:T>G, associated with Hypoalphalipoproteinemia."*
- **TISSUE MCP** (https://Paper2Agent-tissue-mcp.hf.space) — uncertainty-calibrated single-cell spatial transcriptomics analysis. Example query: *"Calculate the 95% prediction interval for the spatial gene expression prediction of gene Acta2 using TISSUE MCP."*
- **Scanpy MCP** (https://Paper2Agent-scanpy-mcp.hf.space) — single-cell preprocessing and clustering. Example query: *"Use Scanpy MCP to preprocess and cluster the single-cell dataset pbmc_all.h5ad."*

These are not toy examples. AlphaGenome (DeepMind's variant-effect predictor), Scanpy (the dominant single-cell-analysis library), and TISSUE (a Stanford uncertainty-calibrated transcriptomics method) are real production tools used by computational biology researchers. The Paper Agent versions of these tools wrap the paper's actual code — the analyses produced are the paper's analyses.

## Cost and runtime

From the README's stated estimates:

- **Time**: 30 minutes to 3+ hours per repository, depending on complexity.
- **Cost**: ~$15 per complex repository, using **Claude Sonnet 4** as the LLM backbone (one-time cost per paper; the resulting MCP server is then free to call).

Compare with the cost of a researcher manually setting up a paper's environment, running its tutorials, understanding its API, and writing wrapper code: typically several hours of expensive human time per paper. Paper2Agent compresses this to fifteen dollars and a coffee break.

For institutions that want to operationalise hundreds of papers (e.g., a pharmaceutical company indexing all relevant computational-biology papers as queryable tools), the economics are favourable: $15 × 1000 papers = $15,000, vs ~hundred-thousand-dollar engineer time for the same coverage.

## How this fits in the agent stack

The most important architectural observation: **Paper2Agent is an agent that builds agents**. It is one of the cleanest implementations of the *meta-agent* pattern — agents whose job is to construct other agents.

This connects to the broader "self-evolving agents" thread in the canon:

- [36-autogenesis-self-evolving-agents](36-autogenesis-self-evolving-agents.md) — the Resource Specification Protocol Library (RSPL) for agent self-construction.
- [45-hyperagents-self-modification](45-hyperagents-self-modification.md) — DGM-H metacognitive self-modification.
- [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md) — agent self-improving across episodes.
- [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md) — adversarial self-play training arena.
- [109-memento-results-and-harness](109-memento-results-and-harness.md) — case-bank-driven agent improvement.
- [154-ctx2skill-self-evolving-context-skills](154-ctx2skill-self-evolving-context-skills.md) — adversarial-self-play skill construction.

Paper2Agent's contribution: *reading existing scientific code and tutorials* and converting them into running agents. This is mechanically distinct from the other self-evolving systems (which evolve their own behaviour based on feedback) — it is more like *translation* than evolution. But the spirit is the same: agents are not handwritten anymore; they are constructed by other agents.

## Why MCP is the right output format

Paper2Agent generates MCP servers, not raw Python libraries. Why?

Three reasons MCP wins for this output:

1. **Interoperability**. Any MCP-compatible runtime can use the resulting agent. Claude Code, Feynman, OpenClaw, custom MCP clients — all see the same tools through the same JSON-RPC interface.
2. **Discoverability**. MCP defines a standard `tools/list` endpoint where clients enumerate available tools with name, description, and parameter schema. The agent doesn't need to know the tool exists in advance.
3. **Composability**. Multiple MCP servers can be loaded simultaneously. A researcher's Claude Code instance can have AlphaGenome MCP, TISSUE MCP, and Scanpy MCP all active — and the LLM choreographs cross-tool workflows ("use AlphaGenome to predict the variant effect, then run Scanpy to find expressing cells, then use TISSUE to get uncertainty bounds").

The MCP standard ([07-model-context-protocol](07-model-context-protocol.md)) is the right substrate for this kind of automated tool generation. Without MCP, every Paper Agent would be a bespoke integration; with MCP, every Paper Agent is automatically usable everywhere.

## Practical takeaways

For builders, the architectural lessons:

### 1. Multi-agent pipelines as automated artifact production

The 5-step pipeline (Scanner → Executor → Extractor → MCP → Verifier) is a *production line* for agents. Each step has a defined input, output, and quality gate. This is the same discipline as Feynman's `Plan → Research → Draft → Cite → Verify` ([155](155-feynman-multi-agent-research-harness.md)) — multi-stage pipelines with separation of concerns dominate single-pass approaches.

### 2. Discipline #10 (NEVER ADD PARAMETERS) is a generalisable rule

When converting between representations (tutorial → tool, paper → agent, prompt → skill), the temptation is to "improve" the source by adding generality, parameters, abstractions. Resist this temptation. Faithfulness to the source is what makes the conversion trustworthy. Same lesson as Feynman's Verifier rule "verify meaning, not topic overlap" — accuracy of representation is paramount.

### 3. Test the generated agent, not just the pipeline

Step 5 (Coverage + Quality Analysis) and the optional benchmark mode are essential. A pipeline that produces broken tools is worse than no pipeline. Auto-generated artefacts must be auto-tested.

### 4. Conservative inclusion bias for source materials

The Tutorial Scanner's "when uncertain, exclude" bias is the right one. Including marginal sources poisons downstream tool quality; excluding them limits coverage. For automated curation, *high-precision low-recall* dominates *low-precision high-recall* in nearly every case.

### 5. Agents-as-Markdown is the de facto standard

This is the third or fourth project in the canon to use Markdown frontmatter for agent definitions (after Feynman, Deep Researcher Agent, HeavySkill, the Memento case bank). The pattern has clearly won.

### 6. MCP as the universal output format

If you are building any kind of capability that should be agent-invocable, generate an MCP server. This makes your output composable with the broader agent ecosystem.

## Limitations

A few honest constraints:

1. **Quality depends on tutorial quality**. If a paper has poorly-written tutorials, Paper2Agent has nothing to work with. The system can't synthesise a tutorial from the paper text alone.
2. **Domain bias toward computational biology**. The hosted examples (AlphaGenome, TISSUE, Scanpy) are all bioinformatics. The architecture should generalise to other domains (NLP, computer vision, robotics) but is unproven there.
3. **Single-paper agents, not cross-paper composition**. Each Paper Agent is per-paper. There is no automatic mechanism to build a meta-agent that orchestrates multiple Paper MCPs for combined workflows. (You can do it manually in Claude Code.)
4. **Cost amortises only at scale**. $15/paper is cheap *per use* if you ask the agent many questions afterward. For a one-off lookup, manually reading the paper is cheaper.
5. **Failure modes are not always recoverable**. Tutorials that depend on proprietary data, paywalled APIs, or specific hardware can't be fully captured. The pipeline produces partial agents in these cases.
6. **Claude Sonnet dependency**. The README states ~$15 cost using Claude Sonnet 4. The pipeline is not LLM-agnostic; results may differ with other models.

## When to use Paper2Agent

Use Paper2Agent when:

- You have a paper whose method you want to apply to your data, and the paper has a reasonable GitHub repo with tutorials.
- You expect to invoke the method many times (the per-use amortised cost gets very small).
- You want the method to be MCP-accessible from any agent runtime.
- You are operationalising many papers and want a production pipeline.

Don't use Paper2Agent when:

- The paper has no public code or only fragmentary implementations.
- You only need to apply the method once or twice (manual setup is faster).
- The paper requires proprietary data or hardware that can't be mocked.
- You want a fundamentally different interface than tool calls (e.g., a chat-style consultation about the paper).

## Where this fits in the canon

- **Read alongside**: [155-feynman-multi-agent-research-harness](155-feynman-multi-agent-research-harness.md) (multi-agent research with file-based handoffs); [07-model-context-protocol](07-model-context-protocol.md) (MCP); [04-skills](04-skills.md) (skills as Markdown artifacts).
- **Self-evolving agents family**: [36-autogenesis-self-evolving-agents](36-autogenesis-self-evolving-agents.md), [45-hyperagents-self-modification](45-hyperagents-self-modification.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md), [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md), [109-memento-results-and-harness](109-memento-results-and-harness.md), [154-ctx2skill-self-evolving-context-skills](154-ctx2skill-self-evolving-context-skills.md).
- **Multi-agent coordination**: [02-subagent-delegation](02-subagent-delegation.md), [20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md).
- **Domain-specific agents**: [28-radagent-agentic-radiology](28-radagent-agentic-radiology.md), [30-gpt-rosalind-domain-specialized](30-gpt-rosalind-domain-specialized.md), [33-dnahnet-genomic-foundation](33-dnahnet-genomic-foundation.md).
- **Survey context**: [158-deep-research-agents-survey-huang-et-al](158-deep-research-agents-survey-huang-et-al.md), [159-deep-research-survey-zhang-et-al](159-deep-research-survey-zhang-et-al.md).
- **Synthesis**: [166-research-agent-frameworks-synthesis](166-research-agent-frameworks-synthesis.md).

## References

1. Repository: https://github.com/jmiao24/Paper2Agent.
2. Companion paper: Miao, Davis, Pritchard, Zou. 2025. *Paper2Agent: Reimagining Research Papers As Interactive and Reliable AI Agents*. arXiv:2509.06917 [cs.AI].
3. Benchmark companion repo: https://github.com/jmiao24/Paper2AgentBench.
4. Hosted MCP examples:
   - AlphaGenome: https://Paper2Agent-alphagenome-mcp.hf.space
   - Scanpy: https://Paper2Agent-scanpy-mcp.hf.space
   - TISSUE: https://Paper2Agent-tissue-mcp.hf.space
5. Source papers wrapped:
   - AlphaGenome (DeepMind variant effect predictor)
   - Scanpy (Wolf et al., the standard single-cell analysis toolkit)
   - TISSUE (Sun et al., Stanford uncertainty-calibrated spatial transcriptomics)
6. Model Context Protocol (Anthropic): https://modelcontextprotocol.io/
7. Adjacent canon: [04-skills.md](04-skills.md), [07-model-context-protocol.md](07-model-context-protocol.md), [154-ctx2skill-self-evolving-context-skills.md](154-ctx2skill-self-evolving-context-skills.md), [155-feynman-multi-agent-research-harness.md](155-feynman-multi-agent-research-harness.md).
