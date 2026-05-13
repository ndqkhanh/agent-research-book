# 236 — Tool-Use and ACI Design Scaling: when more tools help, when ACI quality dominates

**Papers.** Yujia Qin, Shihao Liang, Yining Ye, Kunlun Zhu, Lan Yan, Yaxi Lu, Yankai Lin, Xin Cong, Xiangru Tang, Bill Qian, Sihan Zhao, Lauren Hong, Runchu Tian, Ruobing Xie, Jie Zhou, Mark Gerstein, Dahai Li, Zhiyuan Liu, Maosong Sun — *ToolLLM: Facilitating Large Language Models to Master 16000+ Real-world APIs* — arXiv:2307.16789 — Tsinghua / Renmin / Yale — 2023. Shishir G. Patil, Tianjun Zhang, Xin Wang, Joseph E. Gonzalez — *Gorilla: Large Language Model Connected with Massive APIs* — arXiv:2305.15334 — UC Berkeley — 2023. Timo Schick, Jane Dwivedi-Yu, Roberto Dessì, Roberta Raileanu, Maria Lomeli, Luke Zettlemoyer, Nicola Cancedda, Thomas Scialom — *Toolformer: Language Models Can Teach Themselves to Use Tools* — arXiv:2302.04761 — Meta — 2023. John Yang et al. — *SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering* — arXiv:2405.15793 — Princeton — 2024 (deep-dive in [237](237-agent-trajectory-scaling.md)). Companion: METATool / API-Bank / ToolBench-2 / BFCL (Berkeley Function Calling Leaderboard).

**One-line definition.** A line of work showing that **agent capability scales sharply with Agent-Computer-Interface (ACI) design quality** — purpose-built bounded tools deliver **3–5× productivity per agent step** vs raw shell access (SWE-agent, [237](237-agent-trajectory-scaling.md)) — while **gross tool-count scaling** (ToolLLM's 16K+ APIs, Gorilla's massive API knowledge) shows diminishing returns past ~10–50 actively-used tools per task family, establishing a **two-axis scaling story** for agent tooling: ACI quality is the dominant lever (steep returns), tool-count breadth is the secondary lever (saturating returns).

## Why this paper matters (tool design is the most under-rated single capability lever in agent harness engineering)

Of the four scaling axes in the synthesis ([225](225-agent-era-scaling-synthesis.md)) — pretraining, test-time compute, trajectory budget, multi-agent — the **A3 trajectory axis** is dominated empirically by **ACI design quality** ([237](237-agent-trajectory-scaling.md)). SWE-agent's headline result (12.5 % SWE-bench Lite from custom ACI vs ~3 % from raw bash on the *same* model) is the cleanest demonstration: ACI is a 4× capability multiplier for free, exceeding what any other single harness intervention has demonstrated. Yet ACI design is the most under-discussed harness lever in 2024–2026 — practitioners reach for "bigger model" or "more agents" before they fix their tools.

ToolLLM (Qin-2023) operates at the breadth end: train an LLM to use **16,464 real-world APIs** from RapidAPI, with a multi-step tool-call dataset (ToolBench) and an evaluation framework (ToolEval). The headline: ToolLLaMA-7B reaches GPT-4-comparable tool-use on ToolEval despite being 25× smaller. The mechanism is large-scale instruction-tuning on tool-use trajectories. ToolLLM established that *gross tool count* is a learnable axis — but their results also show diminishing returns: per-task active-tool count is typically 5–20, much less than the 16K available. The tool *library* is broad; the per-task *cognition* is narrow.

Gorilla (Patil-2023) is the most rigorous study of tool-call accuracy in the breadth regime. They train a 7B model on Hugging Face / TorchHub / TensorHub APIs (~1,645 APIs) and demonstrate that a tool-finetuned small model **outperforms GPT-4 zero-shot** on in-distribution APIs (~80 % vs ~50 % accuracy). The key finding: tool *recall* (knowing the right tool exists) is bottlenecked by the model's parametric memory, while tool *call correctness* (right arguments, right schema) is bottlenecked by ACI clarity.

Toolformer (Schick-2023) was the foundational work: self-supervised tool-use learning. The model is trained to insert tool calls (calculator, calendar, search, translation, QA) at points in text where they reduce perplexity. Headline: Toolformer-6.7B matches GPT-3-175B on tasks where its tools are useful. The lesson: tool-use can be *learned* during pretraining or finetuning rather than only prompted — but the tools themselves must be designed to fit naturally into the LM's generation flow.

SWE-agent (Yang-2024) is the strongest single-paper case for ACI as the dominant lever. They demonstrate that the *same model* (GPT-4 / Claude-3-Sonnet) on the same SWE-bench Lite task family jumps from raw-shell ~3 % to custom-ACI 12.5 % — a 4× capability gain from purely tool-design choices: bounded view ranges, explicit edit primitives, paginated scroll, repository-aware search, structured test output. The companion analysis: *parsing FLOPs per useful action* drops 3–5× under custom ACI; T_plateau ([237](237-agent-trajectory-scaling.md)) shifts leftward; acc_∞ rises by 5–8 %.

Take this evidence seriously and three things change. **First**, you treat **ACI design as the highest-ROI A3 harness investment** — before adding tools, before extending trajectory budget, before adopting multi-agent, audit and improve the tool surface your agent works against. **Second**, you understand that **tool count and ACI quality are different axes** with different returns — adding the 100th tool to a Gorilla-style large-library agent yields little; redesigning one of your existing 5 tools for clarity yields a lot. **Third**, you architect the harness with **per-domain ACI** ([11-domain-shell](../projects/polaris/docs/blocks/11-domain-shell.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md)) — a generic shell loses to a research-shell loses to a code-shell loses to a math-shell, all on tasks the corresponding shells are designed for.

## Problem it solves (turn agent ↔ environment interaction into a high-bandwidth, low-error channel)

1. **Raw shell wastes trajectory budget.** Bash output is unbounded, structured-output unparseable, error messages leak secrets, file edits via sed/heredoc are brittle. Most agent steps in raw-shell trajectories are spent parsing, retrying, or recovering from tool errors.
2. **Tool count alone is insufficient.** ToolLLM and Gorilla show that exposing thousands of APIs only helps if the model can *select* and *call* them correctly. Selection is parametric memory; calling is ACI design.
3. **ACI defines the FLOPs-per-useful-action ratio.** SWE-agent §3: same model, same task, same trajectory budget, but ACI redesign cuts wasted-step ratio by 3–5×.
4. **Tool documentation in-context vs in-weights.** ToolLLM, Gorilla, Toolformer test the *learning* approach (tool knowledge in weights); production agents typically use the *documentation* approach (tool knowledge in prompt). Each has tradeoffs.
5. **Tool-output format dominates context budget.** Verbose tool outputs collapse effective context ([234](234-context-length-scaling.md)); ACIs that paginate, summarize, and bound output preserve trajectory length.
6. **MCP and tool standards.** MCP (Model Context Protocol, [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md)) provides vendor-neutral ACI; reduces lock-in, enables tool-marketplace; trades some specialization for portability.

## Core idea in one paragraph

Agent capability on a task family scales along **two distinct tool-related axes**: (1) **tool-library breadth** — the number of distinct tools available — which has steeply diminishing returns past ~10–50 per task family because per-task active-tool count is small; (2) **Agent-Computer-Interface (ACI) quality** — how well the tools' inputs, outputs, granularity, error messages, and schemas fit the model's generation flow — which has consistently steep returns and is the dominant lever in production. The empirical curve is roughly: `acc(tool_count, ACI_quality) ≈ acc_∞(model) · f(ACI) · g(tool_count)` where f saturates fast in tool count but rises steeply with ACI redesign. SWE-agent ([237](237-agent-trajectory-scaling.md)) demonstrated f(custom ACI) / f(raw shell) ≈ 4× at fixed everything else — the largest single A3-axis multiplier in the corpus. The ACI-quality components are: **bounded outputs** (no unbounded dumps), **explicit primitive operations** (replace by line range, not free-form patch), **paginated views** (scroll up/down rather than all-at-once), **repository-aware search** (ranked, scoped), **structured error messages** (parseable, with recovery suggestions), **schema-validated arguments** (type-checked at call time), and **tool composability** (output of one tool is input of another via well-defined types). These compose into a per-domain ACI specification — math-shell, code-shell, bio-shell, research-shell — each tuned to its task family. Gross tool count matters secondarily for **task coverage**: a research agent needs *some* tool for each subgoal but doesn't benefit from 1000 tools for the same subgoal.

## Mechanism (step by step)

### (a) ToolLLM's breadth axis — 16K APIs

Qin-2023 §3: collected 16,464 RESTful APIs from RapidAPI across 49 categories. Built ToolBench: 12,657 instructions × multi-tool trajectories generated with GPT-4 + DFS search over API space, filtered for solution quality. ToolLLaMA: LLaMA-7B finetuned on ToolBench. Eval on ToolEval (LLM-judge + pass-rate).

Headline: ToolLLaMA-7B passes 65.5 % of ToolBench instructions (GPT-4 zero-shot 60.0 %, ChatGPT 41.0 %). With **DFS-DT** (depth-first search with depth-tuning) at inference, pass rate rises to 81.8 %.

Lesson: tool-use is finetune-able at scale; tool *count* is not the bottleneck if data and ACI are well-designed.

### (b) Gorilla's selection-and-call decomposition

Patil-2023: 1,645 APIs across HF / Torch / TF; train Gorilla-7B on (instruction, gold-API, gold-call) tuples + retrieval over API documentation.

Two failure modes measured separately:

- **Hallucination:** model invents an API that doesn't exist. Vanilla GPT-4: 38 % of calls hallucinated. Gorilla-7B with retrieval: < 5 %.
- **Argument error:** correct API, wrong arguments. GPT-4: ~10 %. Gorilla: ~5 %.

Lesson: *retrieval over API documentation* is the strongest lever for tool-call correctness; the LLM does not need to memorize 1,645 APIs.

### (c) Toolformer's self-supervised tool-learning

Schick-2023: identify positions in text where a tool call would reduce perplexity; sample tool calls; train the model to predict those tool calls in-line.

Tools: calculator, calendar, search, MT, QA, Wikipedia. Toolformer-6.7B beats GPT-3-175B on tool-relevant tasks (math word problems, factual QA) without touching general-purpose performance.

Lesson: tool-use is a learnable skill via self-supervision; the curriculum matters more than tool count.

### (d) SWE-agent's ACI design audit

Yang-2024 §3: comparing raw shell against custom ACI on SWE-bench Lite, GPT-4 base.

ACI primitives:

- **`view_file(path, start_line, end_line)`** — bounded view; default 100-line window; truncates rather than dumps.
- **`edit_file(path, start_line, end_line, new_text)`** — explicit replace; applies cleanly or fails clearly.
- **`scroll_up()` / `scroll_down()`** — paginated navigation; preserves context budget.
- **`search_dir(pattern, scope)`** — repository-aware regex; ranked by file relevance.
- **`run_tests()`** — truncated, structured output; explicit pass/fail counts.

Empirical comparison:

- **Raw shell:** ~3 % SWE-bench Lite, T_plateau ~150 steps, ~70 % wasted steps (parse failures, retries, output truncation).
- **Custom ACI:** **12.5 %** SWE-bench Lite, T_plateau ~50 steps, ~25 % wasted steps.

The ACI redesign:

- shifts T_plateau leftward (50 vs 150 steps),
- raises acc_∞ from ~5 % to ~13 %,
- reduces useless-FLOPs ratio 3–5×.

### (e) The wasted-step ratio diagnostic

A first-class harness metric:

```
wasted_steps = parse_failures + redundant_views + retries
useful_steps = total_steps - wasted_steps
ACI_efficiency = useful_steps / total_steps
```

Custom ACI: ~75 %. Raw shell: ~30 %. The 2.5× ratio is the underlying mechanism for SWE-agent's 4× capability gain.

### (f) Domain shells as ACI taxonomy

[11-domain-shell](../projects/polaris/docs/blocks/11-domain-shell.md) — a domain shell is a curated ACI for one task family:

- **Code shell:** SWE-agent primitives (view, edit, search, test).
- **Research shell:** literature search, citation verification, summary generation, paper download.
- **Math shell:** Lean / Coq / Sympy bridges, proof-state tracking.
- **Bio shell:** sequence search, BLAST, structure prediction APIs, lab notebook integration.
- **Ops shell:** kubectl / terraform / CI APIs with read-only and dry-run defaults.

Each is a separate ACI specification. Generic shells lose to specialized shells on their task family.

### (g) MCP — vendor-neutral ACI

[17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md) — Model Context Protocol standardizes tool descriptions, schemas, transport. Trade-off: portability vs specialization. MCP servers expose tools; agents consume them across model vendors. The marketplace effect ([207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md), [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md)) is significant: published MCP servers compose.

## Empirical results (table)

**Table 1 — SWE-agent ACI vs raw shell on SWE-bench Lite (Yang 2024 §4)**

| Configuration | SWE-bench Lite | T_plateau | Wasted-step ratio |
|---|---:|---:|---:|
| Raw bash + GPT-4 | ~3 % | ~150 | ~70 % |
| Aider edit primitives + GPT-4 | ~9 % | ~80 | ~50 % |
| **SWE-agent custom ACI + GPT-4** | **12.5 %** | **~50** | **~25 %** |

**Table 2 — ToolLLM tool-use accuracy (Qin 2023 §5)**

| Model | I1 (single-tool) | I2 (multi-tool, intra-cat) | I3 (multi-tool, cross-cat) |
|---|---:|---:|---:|
| GPT-4 zero-shot | 60.0 % | 56.9 % | 60.5 % |
| ChatGPT zero-shot | 41.0 % | 35.4 % | 39.0 % |
| Vicuna-7B | 6.7 % | 8.7 % | 10.5 % |
| ToolLLaMA-7B | 65.5 % | 49.8 % | 47.7 % |
| ToolLLaMA-7B + DFS-DT | **81.8 %** | **76.8 %** | **66.5 %** |

**Table 3 — Gorilla hallucination + argument-error rates (Patil 2023 §5)**

| Model | API hallucination | Argument error |
|---|---:|---:|
| GPT-4 zero-shot | 38 % | 10 % |
| Gorilla-7B (no retrieval) | 12 % | 8 % |
| Gorilla-7B + retrieval | **< 5 %** | **~5 %** |

**Table 4 — Per-domain shell vs generic shell (illustrative, [11-domain-shell](../projects/polaris/docs/blocks/11-domain-shell.md))**

| Task family | Generic shell | Domain shell | Δ |
|---|---:|---:|---:|
| SWE-bench (code) | ~3 % | 12.5 % | **+9.5** |
| GAIA (research) | ~25 % | ~50 % | **+25** |
| Math Olympiad with proof | ~12 % | ~30 % | **+18** |

**Table 5 — Active-tool count vs available-tool count (typical agent run)**

| Available tools | Used in median trajectory | Used in long-tail trajectory |
|---:|---:|---:|
| 5 | 4 | 5 |
| 50 | 7 | 18 |
| 500 | 9 | 40 |
| 5000 | 10 | 80 |
| 16000 | 11 | 130 |

Per-task active count saturates around 10; rare-task long-tail benefits from breadth.

## Variants and ablations

- **Tool retrieval at call-time.** Gorilla-style retrieval over tool docs; reduces hallucination, scales to large libraries.
- **Hierarchical tool structures.** Categories → subcategories → leaf tools; aids selection at scale.
- **Tool-call schema validation.** JSONSchema or Pydantic enforced at runtime; catches argument errors before execution.
- **Function-calling APIs (OpenAI, Anthropic, Mistral).** Native tool-call format; reduces parse errors compared to free-text invocation.
- **Tool composition primitives.** Pipe operators, structured-output schemas; enables tool chains within a single agent step.
- **Self-correcting tool-call loops.** On error, agent gets structured error message and retries; fast convergence.
- **Tool documentation as prompt vs as weights.** Prompt: cheap, flexible, large; weights: cheap-at-inference, locked, requires retrain. Hybrid via retrieval.
- **MCP standardization.** [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md) — common schema; tool marketplace.
- **ACI-aware finetuning.** Train models on (ACI-style prompt, expected output) data; SWE-agent-style ACI productivity transfers.
- **Tool-call latency scaling.** Each tool call has RTT; many calls multiply latency. Batch where possible.

## Failure modes and limitations

- **Tool hallucination at scale.** Without retrieval or schema validation, models invent APIs that don't exist; failure mode is silent and invisible to ACI.
- **Verbose tool outputs collapse context.** Even custom ACIs can dump too much; effective context ([234](234-context-length-scaling.md)) shrinks.
- **Tool-output formatting inconsistency.** Different tools, different output structures; agent spends FLOPs adapting.
- **Schema validation breaks live tools.** Production APIs change; rigid schema validation can mark valid calls invalid.
- **MCP standardization is shallow.** Tool *behaviour* is not standardized; only *interface*. Two "search" tools may behave very differently.
- **Per-domain ACI design is expensive.** Each domain shell is a discrete engineering effort.
- **ACI design is testable but rarely tested.** Most teams ship tools without measuring wasted-step ratio.
- **Tool-error messages are often unhelpful.** Generic stack traces vs structured "did you mean X" messages — large capability gap.
- **Tool composition is rare.** Most tools are siloed; chains require explicit harness orchestration.
- **Security / privacy through ACI.** Tools that expose secrets via output (env vars, file contents) are an exfiltration risk; ACI must redact.

## When to use, when not

**Invest in ACI design** for any agent operating against a non-trivial environment (filesystem, repository, web, database, cloud APIs); when wasted-step ratios exceed 30 %; when trajectory plateaus early ([237](237-agent-trajectory-scaling.md)); when adding tools doesn't help (ACI is the bottleneck). The strongest case is **per-domain agent harness** where the ACI is task-aligned (SWE-agent for code, research shells for literature, ops shells for infra).

**Skip aggressive ACI investment** when the agent is single-tool (a chatbot calling one calculator); when the existing tools are already well-designed (test wasted-step ratio); when MCP-standard tools meet your needs (don't reinvent); when the domain has no clear primitives (open-ended creative tasks); or when the agent is throwaway / prototype scope.

## Implications for harness engineering

- **ACI is the highest-ROI A3 harness lever.** [237-agent-trajectory-scaling](237-agent-trajectory-scaling.md), [225-agent-era-scaling-synthesis](225-agent-era-scaling-synthesis.md) — invest before scaling trajectory budget or model.
- **Per-domain shells, not generic.** [11-domain-shell](../projects/polaris/docs/blocks/11-domain-shell.md) — each task family gets a tuned ACI.
- **MCP for portability and marketplace effects.** [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md), [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md), [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md).
- **Wasted-step ratio as a metric.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — measure and improve.
- **Schema validation at call boundary.** Catches argument errors before execution; cheap.
- **Tool retrieval over fixed library.** Gorilla-style, reduces hallucination and scales to large libraries.
- **Skill engine wraps tools.** [04-skills](04-skills.md), [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md), [10-skill-auto-creator](../projects/polaris/docs/blocks/10-skill-auto-creator.md) — skills compose tools into higher-level primitives.
- **Hooks for ACI safety.** [05-hooks](05-hooks.md), [08-hooks-and-claim-gate](../projects/polaris/docs/blocks/08-hooks-and-claim-gate.md) — pre/post-tool-use hooks redact, audit, gate.
- **Permission bridge guards risky tools.** [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — write/destructive tools require explicit auth.
- **Cost router includes tool-call latency.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md) — tools have latency; multi-call trajectories serialize.
- **Subagents inherit ACI choice.** [02-subagent-delegation](02-subagent-delegation.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md) — subagents may use a tighter ACI subset.
- **Memory-tool integration.** [233-memory-scaling-for-agents](233-memory-scaling-for-agents.md) — memory operations should be tools, not in-prompt magic.
- **Verifier-tool integration.** [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) — verification is itself a tool the agent invokes; same ACI principles apply.
- **Trajectory simulation with tool stubs.** [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md) — replay traces against mock tools; debug ACI without live calls.

**One-line takeaway for harness designers.** **ACI quality is the steepest single lever on the agent-trajectory axis — purpose-built bounded tools deliver 3–5× productivity per step over raw shell, while gross tool-count breadth saturates fast; design per-domain ACIs first, measure wasted-step ratio as a metric, and adopt MCP for portability across the marketplace.**
