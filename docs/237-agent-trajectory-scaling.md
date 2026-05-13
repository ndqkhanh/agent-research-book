# 237 — Agent Trajectory Scaling: how performance scales with steps, tools, and context

**Anchors.** John Yang, Carlos E. Jimenez, Alexander Wettig, Kilian Lieret, Shunyu Yao, Karthik Narasimhan, Ofir Press — *SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering* — arXiv:2405.15793 — Princeton — 2024. Grégoire Mialon, Clémentine Fourrier, Craig Swift, Thomas Wolf, Yann LeCun, Thomas Scialom — *GAIA: A Benchmark for General AI Assistants* — arXiv:2311.12983 — Meta / HuggingFace — 2023. Tianbao Xie, Danyang Zhang, Jixuan Chen, Xiaochuan Li, Siheng Zhao, Ruisheng Cao, Toh Jing Hua, Zhoujun Cheng, Dongchan Shin, Fangyu Lei, Yitao Liu, Yiheng Xu, Shuyan Zhou, Silvio Savarese, Caiming Xiong, Victor Zhong, Tao Yu — *OSWorld: Benchmarking Multimodal Agents for Open-Ended Tasks in Real Computer Environments* — arXiv:2404.07972 (deep-dive in [95](95-osworld.md)). GDPval (deep-dive in [96](96-gdpval.md)). METR's *measuring AI ability to complete long tasks* (Apr 2024 → 2025 long-horizon studies). Companion: SWE-bench Verified, agent-leaderboard time-series across 2024–2025.

**One-line definition.** The empirical scaling story of LLM agents: holding model fixed, **agent task accuracy scales as a smooth function of allowed trajectory budget** — number of steps, tool-call count, context length, and per-call test-time compute — with **model-specific plateaus** that move with pretraining capability ([216](216-pretraining-scaling-laws-foundation.md)) and per-step test-time-compute spend ([217](217-test-time-compute-scaling.md)), and **where the plateau lives** is the single most predictive number for whether a harness will solve a benchmark.

## Why this paper matters (the agent-loop budget is the third scaling axis, and benchmark progress 2024–2026 is mostly explained by it)

Once pretraining buys you the floor ([216](216-pretraining-scaling-laws-foundation.md)) and per-call test-time compute buys you the in-call ceiling ([217](217-test-time-compute-scaling.md)), the **agent loop** is the third axis: how many tool calls, browser actions, file edits, observations, and reflections does the harness allow before forced termination? Almost every benchmark improvement narrative on SWE-bench, GAIA, OSWorld, WebArena, AgentBench, and GDPval between mid-2024 and 2026 is, when measured carefully, the *same model* solving more tasks because the *harness lets it run longer with better tools and more state*. SWE-agent's 12.5 % → 33 % → 48 % → 62 %+ trajectory on SWE-bench Verified across 2024–2025, with the underlying base models mostly stable in raw capability, is the textbook proof.

SWE-agent (Yang-2024) is the clearest single source on how agent **trajectory budget** translates to capability. The paper introduces an **Agent-Computer Interface (ACI)** — a *purpose-built* set of tools and observations the agent can use to interact with a code repository — and shows that the same Claude/GPT-4 base model, given the right ACI plus a bounded trajectory length, jumps from a few percent on SWE-bench (raw shell access) to **12.5 %** on SWE-bench-Lite. Subsequent work on SWE-agent + verifier ensembles, OpenHands, Aider, Roo Code, and agentless approaches all sit on top of the same underlying observation: **the trajectory is the unit of compute, and the harness's job is to maximize useful steps inside it**.

GAIA (Mialon-2023) operationalizes the scaling story for general assistants. It defines three difficulty levels ranging from "human takes 5 minutes" to "human takes 30 minutes with tools," and benchmark scores climb steeply with allowed budget: a base model with no tools scores ~5 %; the same model with web-browsing + code-execution + multi-step planning scores 30–60 %. The Level 1 → 3 difficulty axis maps closely to **how many sequential subgoals the agent must execute correctly** — a trajectory-length proxy.

OSWorld ([95](95-osworld.md)) and GDPval ([96](96-gdpval.md)) extend this to long-horizon, multi-application, real-environment trajectories — and confirm that agent-task accuracy curves climb monotonically with allowed steps until a model-specific plateau. METR's long-task benchmark (2024 → 2025) measures the *time-horizon* of tasks an agent can complete, and reports the well-known result that this horizon *doubled approximately every 7 months* for frontier agents over 2024–2025.

Take this evidence seriously and three things change. **First**, you stop benchmarking on "model X scores Y" and start benchmarking on **`accuracy(model, harness, max_steps)` curves**, treating max-steps as a co-equal axis. **Second**, you architect the harness around **plateau-aware** budgeting: do not waste compute past the plateau; do not under-budget below it. **Third**, you take the **Agent-Computer Interface design** as seriously as model selection — the ACI is the multiplier between raw model capability and trajectory-budget productivity, and a poorly designed ACI burns all your trajectory budget on parsing.

## Problem it solves (turning the agent loop into a scalable, measurable axis)

1. **No quantitative agent-loop scaling curve before SWE-agent.** Pre-2024 work on agents (ReAct, AutoGPT, BabyAGI) lacked controlled FLOP-equal trajectory-length sweeps. SWE-agent provides the first such curve in a real software-engineering setting — accuracy as a function of allowed trajectory budget at fixed base model.
2. **Tool-design dominates trajectory productivity.** Yang-2024 §3: replacing raw bash with a small custom edit/search/navigate ACI multiplies productivity per step by ~3–5×; same model, same step budget. The lesson: tool-design is FLOPs.
3. **Long-horizon coherence breaks before model capability does.** METR and OSWorld show that agents fail not from missing knowledge but from **state-management collapse** — losing track of what they've done. Memory blocks ([05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md), [151-153 MEMTIER]) and trajectory-length scaling [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md), [195-argus-omega-vol-2](195-argus-omega-vol-2-trajectory-temporal-horizon.md) directly attack this.
4. **Context-length scaling has its own curve.** Effective context (the portion of context the model attends to without "lost in the middle" decay) caps useful trajectory length even when nominal context is large. Yang-2024 and the long-context evaluation literature show empirical decay sets in well before nominal limits.
5. **Step-cap selection is a load-bearing harness decision.** Set max_steps too low and easy tasks fail; too high and hard tasks bleed compute past the plateau. Plateau-aware budgeting from accuracy(steps) curves is the principled choice.
6. **Subagent decomposition is a trajectory-axis trick.** [02-subagent-delegation](02-subagent-delegation.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md) — splitting one long trajectory into N short sub-trajectories with independent context windows shifts the curve favourably; the orchestrator pays the integration cost.

## Core idea in one paragraph

For a fixed base model M and harness H with Agent-Computer Interface ACI, the **task-accuracy of an agent** is a smooth, monotone-increasing, plateauing function of the allowed trajectory budget T (measured as step count, tool-call count, total tokens, or wall-clock time): `acc(M, H, ACI, T)` rises sharply for small T, has an inflection point near `T ≈ T_plateau(M, ACI)`, and saturates at `acc_∞(M, H, ACI) ≤ pretraining-implied ceiling(M)`. The plateau height moves with **base model capability** (Chinchilla curve, [216](216-pretraining-scaling-laws-foundation.md)) and per-call **test-time compute spend** ([217](217-test-time-compute-scaling.md)); the plateau location moves with **ACI quality** (better tools → fewer wasted steps → plateau hits earlier and higher); and the slope to the plateau moves with **memory and state-management quality** (better memory → less coherence loss → steeper climb). This three-parameter description — height, location, slope — is the operational scaling law of agent harnesses: pick a base model to set height, design an ACI to move location, install memory and verifiers to steepen slope. SWE-agent demonstrates the location-shift (custom ACI moves plateau leftward). HeavySkill ([156](156-heavyskill-rlvr.md)) and tree-search inner skills ([p18](../projects/polaris/docs/research/p18-tree-search-thinking.md)) demonstrate height-shift via per-call TTC. ReasoningBank ([81](81-reasoningbank.md)) and MEMTIER trilogy ([151–153]) demonstrate slope-shift via memory.

## Mechanism (step by step)

### (a) Trajectory budget as the unit of agent compute

Define `T` as the allowed budget for one agent attempt, in any of:

- **Step count.** Number of tool calls / agent-loop iterations (most common; SWE-agent default 50–100).
- **Tool-call FLOPs.** Counts non-LLM compute (executing test suites, browser-render, sandbox runs).
- **Total tokens.** Cumulative context tokens consumed across the trajectory.
- **Wall-clock time.** What the user actually pays for; the budget that matters in production.

The four are *correlated but not interchangeable* — wall-clock especially decouples for tool-bound trajectories.

### (b) The accuracy(T) curve and its three parameters

Empirically (SWE-agent §4, OSWorld §6, GAIA §3):

- **height (`acc_∞`):** the asymptote, set by base model + per-call TTC + ACI quality.
- **location (`T_plateau`):** the step count at which 90 % of `acc_∞` is reached. Better ACIs reduce this — fewer steps wasted on syntax / parsing / discovery.
- **slope (`d(acc)/d(T) at T_plateau/2`):** the climb rate. Better memory / planning / verifier loops increase this.

A first-cut model: `acc(T) ≈ acc_∞ · (1 − exp(−T / T_plateau))`. Real curves have a slight S-shape (warm-up where the agent is establishing context) but the exponential approximation works for budgeting.

### (c) ACI design dominates trajectory productivity

Yang-2024 §3: SWE-agent's ACI exposes:

- **`view_file(path, line_range)`** — bounded snippets, not full-file dumps that explode context.
- **`edit_file(path, start, end, new_text)`** — explicit replace ranges, not free-form patches that mis-apply.
- **`search_dir(pattern)`** — repository-aware, ranked.
- **`run_tests()`** — feedback-bearing tool with truncated output.
- **`scroll_up`/`scroll_down`** — paginated view to prevent context collapse.

Compared to raw shell, this ACI:

- reduces *parsing FLOPs per useful action* by ~3–5×,
- shifts `T_plateau` from ~150 steps (raw shell) to ~50 steps (ACI),
- raises `acc_∞` by ~5–8 % absolute on SWE-bench Lite, holding model fixed.

The lesson generalizes: every harness should have a purpose-built ACI per task family, not a generic shell. See [11-domain-shell](../projects/polaris/docs/blocks/11-domain-shell.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md).

### (d) Context-length scaling within a trajectory

A trajectory consumes context monotonically: each step appends observations and reasoning. The effective-context curve (Lost-in-the-Middle, RULER, NIAH-with-distractors) caps useful trajectory length:

- **Frontier models (Claude 3.5/3.7 Sonnet, GPT-4o, Gemini 2.0/2.5):** effective context to ~64 K – 128 K tokens before noticeable decay.
- **Open-weights 70 B–72 B:** typically ~16 K – 32 K effective.
- **Long-context specialist models (Gemini 2 1M+, Claude 200K, MiniMax-Text-01):** push the boundary but with non-uniform attention — early/late tokens still attended, mid-tokens degrade.

This caps trajectory length at fixed-context. Mitigations: **subagent decomposition** ([02](02-subagent-delegation.md)), **memory blocks** ([05](../projects/polaris/docs/blocks/05-reasoning-bank.md), [151–153]), **summarization-and-rewind**, **fresh-context loops** ([p16-ralph-fresh-context](../projects/polaris/docs/research/p16-ralph-fresh-context.md)).

### (e) Long-horizon coherence collapse

METR (2024 → 2025) reports that agent task-completion-time horizon **roughly doubles every 7 months** at the frontier; OSWorld and GAIA confirm that the failure mode at long horizon is rarely "model didn't know" and almost always **"agent lost track."** The fixes:

- **Episodic memory.** ReasoningBank ([81](81-reasoningbank.md)) stores compressed past experiences keyed for retrieval.
- **Scratchpads / structured plans.** [03-plan-mode](03-plan-mode.md), [p4-pre-research](../projects/polaris/docs/research/p4-pre-research.md).
- **Verifier checkpoints.** [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [19-verifier-cross-channel](../projects/polaris/docs/blocks/19-verifier-cross-channel.md) — verify subgoals, not just final answer.
- **Subagent delegation with context budgets per subagent.** [02](02-subagent-delegation.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md).
- **Trajectory simulation / replay.** [142](142-trajectory-simulation-agents.md), [195](195-argus-omega-vol-2-trajectory-temporal-horizon.md).

### (f) Time-horizon doubling: METR's empirical claim

METR's time-horizon benchmark assigns tasks at varying human-time-to-complete (5 min → 4 hours) and measures agent success rate. The 50 %-success-rate task duration was tracked through 2024–2025 across frontier agents and *roughly doubled every 7 months* — independent of the underlying base-model parameter count. The driver was: better harnesses, better ACIs, better memory, longer context. This is the empirical signature of trajectory-axis scaling.

### (g) Trajectory plateau as harness diagnostic

The location of `T_plateau` is a diagnostic: if your benchmark scores plateau at T=20 steps, you have an ACI problem (tools too coarse) or a planning problem (agent gets stuck). If they plateau at T=200 with a flat tail, you have a memory problem (longer trajectories don't help because state is lost). If they plateau at T=50 cleanly with the asymptote near `acc_∞(model)`, your harness is well-tuned and the bottleneck is now base model.

## Empirical results (table)

**Table 1 — SWE-bench Lite progress 2024–2025 (same Claude / GPT-4-class models, different harnesses)**

| Harness | Year | SWE-bench Lite | Note |
|---|---:|---:|---|
| Raw shell + Claude-3.5 | 2024 Q1 | ~3 % | No ACI |
| SWE-agent + Claude-3.5 | 2024 Q2 | 12.5 % | Custom ACI |
| Aider + Claude-3.5 | 2024 Q3 | ~26 % | Repo-map + edits |
| OpenHands / SWE-Search | 2024 Q4 | ~30–35 % | + tree-search inner skill |
| Agentless + GPT-4o | 2024 Q3 | ~27 % | Localization-first |
| Frontier 2025 (Claude 3.7 + harness) | 2025 H1 | 60 %+ | Long-context + verifiers + RL |

**Table 2 — Effective trajectory budget vs accuracy (illustrative; SWE-bench-Verified, Claude-3.5)**

| max_steps | accuracy | comments |
|---:|---:|---|
| 5 | ~8 % | Insufficient for repo navigation |
| 25 | ~28 % | Approach plateau |
| 50 | ~33 % | Near plateau |
| 100 | ~34 % | Plateau; diminishing returns |
| 200 | ~33 % | Slight decay (context overflow) |

**Table 3 — Agent benchmark scores 2024 → 2025 (frontier harnesses)**

| Benchmark | 2024 baseline | 2025 frontier | Δ |
|---|---:|---:|---:|
| SWE-bench Verified | 12.5 % | 62 %+ | **~5×** |
| GAIA Avg | ~30 % | ~65 % | **~2.2×** |
| OSWorld | ~12 % | ~38 % | **~3.2×** |
| WebArena | ~14 % | ~40 % | **~2.9×** |

**Table 4 — Failure-mode breakdown at long horizon (OSWorld §6 + analyses)**

| Failure | Share of long-horizon errors |
|---|---:|
| State-tracking / coherence loss | ~40 % |
| Tool misuse (wrong arg, parsing failure) | ~25 % |
| Missing knowledge (model ceiling) | ~20 % |
| Verifier mis-call | ~10 % |
| Other | ~5 % |

## Variants and ablations

- **Custom-ACI vs raw-shell.** ~3–5× productivity gain at fixed model; biggest single harness lever. [11-domain-shell](../projects/polaris/docs/blocks/11-domain-shell.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md).
- **MCP-bridged ACI.** [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md) — generic, vendor-neutral; trades some productivity for portability.
- **Subagent decomposition.** Splits one trajectory of length T into k of length T/k each; bypasses long-context decay; pays orchestration overhead. [02](02-subagent-delegation.md).
- **Worktree isolation.** [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md) — subagents act on isolated repo copies; eliminates state-bleed and enables parallel exploration.
- **Reflective revisions.** [90-reflexion-deep](90-reflexion-deep.md) — sequential test-time compute applied per agent step; raises slope.
- **Tree-search inner skill.** [p18-tree-search-thinking](../projects/polaris/docs/research/p18-tree-search-thinking.md), [84-swe-search-mcts](84-swe-search-mcts.md) — per-step parallel test-time compute; raises height.
- **Memory blocks.** [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md), [151-153 MEMTIER] — slope-raisers via cross-trajectory state.
- **Cost-aware step-cap selection.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md) — pick max_steps per task type from the curve.
- **Long-context substrate.** Gemini 2.5 1M+, MiniMax-Text-01 — push T_plateau outward but with non-uniform attention; cannot replace memory.
- **RL-trained agent policies.** [80-knowrl](80-knowrl.md), DeepSeek-R1, OpenAI o3 — push the policy itself toward better trajectory shaping.

## Failure modes and limitations

- **Plateau ≠ ceiling.** Past the plateau, additional steps usually do not help; sometimes they actively hurt (context overflow, accumulated noise). Detect and stop.
- **Tool fragility.** A flaky tool (intermittent test failure, brittle parser) causes accidental loops; budget vanishes on retries. Audit tool-call success rates per ACI.
- **Coherence collapse without memory.** Long trajectories with no episodic memory lose ~40 % of long-horizon performance to state-tracking errors.
- **Subagent integration debt.** Decomposing a trajectory into subagents shifts cost from agent-loop to orchestrator-merge; if the merge step is fragile, gains evaporate. [98-diversity-collapse-mas](98-diversity-collapse-mas.md), [224](224-multi-agent-parallel-scaling.md).
- **Context budget misallocation.** Naïve agents dump full file contents and tool outputs; effective context shrinks rapidly. ACIs must paginate and summarize.
- **Verifier-induced premature termination.** Aggressive verifiers can terminate trajectories too early; lax verifiers waste budget. [19-verifier-cross-channel](../projects/polaris/docs/blocks/19-verifier-cross-channel.md).
- **Benchmark contamination.** SWE-bench Lite leaked into pretraining corpora by 2024–2025; SWE-bench Verified and held-out evaluations are necessary.
- **Wall-clock vs FLOP optimum.** Sequential trajectories serialize; parallel subagents parallelize. Wall-clock-optimal harnesses look different from FLOP-optimal ones.
- **Plateau location is unstable across task distributions.** A harness tuned for SWE-bench plateaus differently on GAIA. Re-measure per benchmark.

## When to use, when not

**Treat trajectory budget as a primary scaling axis** when you ship a long-running agent (SWE-bench, OSWorld, GAIA, GDPval, deep-research, autonomous-coding); when you can pre-compute or estimate a difficulty bucket per task; when memory and verifiers are available to steepen slope; and when wall-clock latency tolerates 50–200 step trajectories. The trajectory axis dominates the ROI of harness investment for any task whose human-completion-time exceeds ~5 minutes.

**Do not** over-invest in trajectory length when the underlying ACI is poor (fix the ACI first; you'll get the same gain with fewer steps); when context-management is unsolved (long trajectories collapse); when the base model lacks priors for the task (no trajectory length recovers missing knowledge — see [216](216-pretraining-scaling-laws-foundation.md)); or when latency is the binding constraint (sequential trajectories serialize).

## Implications for harness engineering

- **Maintain and publish accuracy(T) curves per harness.** Without the curve you cannot pick step caps; without step caps you cannot enforce cost discipline ([p13-cost-discipline](../projects/polaris/docs/research/p13-cost-discipline.md), [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md)).
- **Custom ACI per domain shell is the highest-ROI harness investment.** [11-domain-shell](../projects/polaris/docs/blocks/11-domain-shell.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md). The ACI shifts T_plateau leftward by 2–5× — the same gain as a 2× larger model in trajectory-budget terms.
- **Memory is a slope-multiplier, not a luxury.** [05-reasoning-bank](../projects/polaris/docs/blocks/05-reasoning-bank.md), [81-reasoningbank](81-reasoningbank.md), [151-153 MEMTIER] — without it, the long-horizon curve flattens; with it, ~40 % of long-horizon failures are recoverable.
- **Subagent decomposition for context-bound tasks.** [02](02-subagent-delegation.md), [18-subagent-and-worktree](../projects/polaris/docs/blocks/18-subagent-and-worktree.md) — extends effective trajectory length beyond the long-context decay cap.
- **Verifier checkpoints on subgoals, not only final answers.** [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [19-verifier-cross-channel](../projects/polaris/docs/blocks/19-verifier-cross-channel.md) — catch coherence collapse early.
- **HIR observability with trajectory metadata.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — log step number, tool used, context size, verifier verdict; this is the data that fits the curves.
- **Per-difficulty step caps from a router.** [88-confidence-driven-router](88-confidence-driven-router.md), [87-routellm](87-routellm.md), combined with TTC allocation from [217](217-test-time-compute-scaling.md).
- **Bright-line cost gates that respect plateaus.** [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — terminate past the plateau, not at fixed budget.
- **Recover from coherence collapse with fresh-context loops.** [p16-ralph-fresh-context](../projects/polaris/docs/research/p16-ralph-fresh-context.md) — when the trajectory gets stuck, restart with summarized state.
- **Skill memory amortizes across trajectories.** [04-skills](04-skills.md), [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md), [156-heavyskill-rlvr](156-heavyskill-rlvr.md) — skills moved from per-trajectory invention to harness-resident library.
- **Daemon-mode agents shift wall-clock economics.** [13-daemon-and-scheduler](../projects/polaris/docs/blocks/13-daemon-and-scheduler.md), [p7-daemon](../projects/polaris/docs/research/p7-daemon.md) — long horizons are tolerable when not user-blocking.
- **Trajectory simulation reduces deployment risk.** [142-trajectory-simulation-agents](142-trajectory-simulation-agents.md), [195-argus-omega-vol-2](195-argus-omega-vol-2-trajectory-temporal-horizon.md) — simulate before paying for real-environment compute.

**One-line takeaway for harness designers.** **The agent loop is a scaling axis: model sets height, ACI sets location, memory sets slope, and the harness's job is to *measure* the accuracy(T) curve, *budget* against the plateau, and *invest* in whichever lever is steepest at the current frontier — usually the ACI first, memory second, model last.**
