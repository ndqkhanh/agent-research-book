# 102 — ClawGym: A Scalable Framework for Building Effective Claw Agents

**Paper.** Fei Bai, Huatong Song, Shuang Sun, Daixuan Cheng, Yike Yang, Chuan Hao, Renyuan Li, Feng Chang, Yuan Wei, Ran Tao, Bryan Dai, Jian Yang, Wayne Xin Zhao — *ClawGym: A Scalable Framework for Building Effective Claw Agents* — arXiv:2604.26904v1 — Gaoling School of AI (Renmin University of China) / IQuest Research / Beihang — April 29, 2026. Code: https://github.com/ClawGym (resources to be released).

**One-line definition.** ClawGym is a complete lifecycle for *Claw-style* (workspace-grounded computer-using) agents: dual-route task synthesis (persona-driven top-down + skill-grounded bottom-up over 30K OpenClaw skills) → resource generation + hybrid verification (code-checks 0.7 + rubric 0.3) → black-box trajectory rollout via a proxy layer that *never* modifies the encapsulated agent runtime → SFT with multi-turn loss masking + GRPO with sandbox-parallel rollouts → ClawGym-Bench (200 tasks, 6 categories, std < 1%); the resulting ClawGym-30B-A3B beats Qwen3-235B-A23B on PinchBench (86.0 vs 60.6), demonstrating that high-quality agent-specific data eats parameter scale.

## Why this paper matters

Computer-using agents — the class Claude Code, Devin, Cursor Agent, OpenClaw, AutoClaw all belong to — operate through *opaque system interfaces*, not clean text APIs. They read files, edit them, run shell commands, observe outputs, and close the loop on a persistent workspace. Almost every recent benchmark for this class (PinchBench, ClawBench, OSWorld) measures *outcomes* but tells you nothing about how to *train* the agent that produced them. The training-data problem for Claw agents is harder than for code agents (SWE-Bench has tests as ground truth) or text-reasoning agents (AIME has answer keys): a Claw task can succeed with a heterogeneous pile of artefacts (modified files, structured outputs, generated reports) and verifying *correct workspace state change* requires both deterministic checks and qualitative judgement.

ClawGym is the first paper that closes the full loop end-to-end: synthesize tasks → generate workspaces → verify with hybrid scoring → roll out trajectories *without modifying* the encapsulated agent → filter by reward threshold → SFT then RL → benchmark with discriminative, stable evaluation. Five things change if the loop works as advertised. (1) Smaller open-weight models can compete with frontier closed models on workspace-grounded tasks via targeted fine-tuning — Qwen3-8B jumps +43% on ClawGym-Bench and +39% on PinchBench after the recipe. (2) The "black-box rollout via proxy" pattern dissolves the awkwardness of training data collection on closed agent runtimes: you can train against any harness whose model calls go through a network. (3) Hybrid (code + rubric) verification, with the explicit 0.7/0.3 weighting, becomes a transferable production-grade scoring template for any environment-grounded agent. (4) Sandbox-parallel GRPO with code-only reward (no auxiliary reward model) demonstrates that reinforcement learning for computer-use can be done with *cheap* infrastructure. (5) ClawGym-Bench's discriminative stability (std ≤ 1% across five repeated runs) raises the bar for what "reliable Claw evaluation" means.

## Problem it solves

Three concrete blockers in prior work, each addressed by one stage of ClawGym:

1. **Data scarcity.** Generating diverse, verifiable Claw-style training tasks at scale is hard: tasks must capture *personalized* workflows, must be *long-horizon* (sequences of file ops + tool calls + intermediate validation), and must be *grounded in local workspaces* with realistic mock files (real user files break privacy and reproducibility). ClawGym's dual-route synthesis attacks the diversity problem from both ends.
2. **Trajectory collection opacity.** OpenClaw is an encapsulated black box — opaque internal execution, context management, subagent sessions. Modifying it to log trajectories breaks execution semantics. ClawGym's proxy layer captures all model-server traffic without touching the runtime.
3. **Evaluation unreliability.** Environment-grounded outputs are heterogeneous; existing benchmarks either ignore qualitative dimensions or apply overly strict/lenient verification. ClawGym's hybrid verifier with the 0.7/0.3 split makes both sides explicit and tunable.

## Core idea in one paragraph

A Claw task is a tuple τ = ⟨p, s₀, A, F, V_τ⟩ — instruction, initial workspace state, action space, transition function, task-specific verifier — and a trajectory is ξ = (A₁, O₁, …, A_K, O_K) where actions and observations *batch* (multiple tool calls before observation processing, the way real Claw runtimes actually behave). To produce training data for this object, run two synthesis pipelines (persona-driven top-down for user-realism, skill-grounded bottom-up for capability-realism), generate self-contained mock files for each task, score completion with code-checks + rubric judgement (weighted 0.7/0.3), assess task quality via novelty/plausibility/difficulty filters and verifier-quality alignment checks, then deploy hundreds of OpenClaw containers behind a logging proxy and let them solve tasks. Aggregate the proxied messages into multi-turn trajectories, threshold by final score ≥ 0.5, and use the surviving 24.5K trajectories for SFT (with multi-turn loss masking that excludes environment-feedback tokens) followed by GRPO with sandbox-parallel rollouts and the code-checker as direct reward. Build a 200-task benchmark by retaining only tasks where strong agents materially outperform weak ones, with humans in the final review loop. The whole pipeline is reproducible because the only deeply integrated component is the proxy — everything else is data engineering.

## Mechanism (step by step)

### Stage 1 — Dual-route task synthesis

**Persona-driven (top-down).** Seed = (persona u from a 1B-combination space, scenario c from 9 major / 43 sub-categories, atomic operations G from 7 categories / 26 operations). LLM task-generator M_task (GPT-5) emits instruction p. Distribution over scenarios is balanced (max 12.5% of any single category, Figure 2). Strength: realistic, diverse user-facing intents.

**Skill-grounded (bottom-up).** 30K raw OpenClaw skills from ClawHub get LLM-annotated (MiniMax-M2.5) for skill summary, core content, usage constraints, I/O properties. 16K survive the synthesizable filter (no missing credentials, compatible formats). Each task pairs 1 primary skill + up to 3 supporting skills, composed into an instruction by M_task. Strength: anchors tasks in concrete executable capabilities.

Mixed synthesis beats either alone (Table 7): Qwen3-8B reaches 50.24% on ClawGym-Bench with mixed vs 49.44% (persona-only) / 49.06% (skill-only). The two pipelines bridge abstract user needs and concrete tool execution.

### Stage 2 — Resource generation + hybrid verification

For each instruction p, LLM identifies file requirements f = {(l_i, t_i, d_i)} (path, type, content spec) and generates lightweight, task-specific mock files (text/markdown/JSON/CSV/YAML). Self-contained, reproducible, no real-user data.

**Hybrid verifier:**

```text
s_code   = (1/m) Σ_i b_i,        b_i ∈ {0,1}   over deterministic code-checks
s_rubric = (Σ_j w_j q_j) / (Σ_j w_j),    q_j ∈ {0, .25, .5, .75, 1.0}
s_task   = 0.7 · s_code  +  0.3 · s_rubric
```

Why 0.7? Workspace-grounded completion is most reliably evidenced by concrete artefacts (files, outputs, schema validity); rubric handles the residual qualitative dimensions (clarity, organization, faithfulness) without dominating. The split is empirical but robust.

### Stage 3 — Quality filters

Two layers, both automatic, with human spot-checks:

- **Task quality.** Novelty (cosine similarity below threshold), plausibility (GPT-5.4 binary judge for clarity/consistency/realism + tool availability), difficulty (LLM estimate of step count, op diversity, reasoning depth — keep a balanced simple/moderate/hard mix).
- **Verifier quality.** Code-checker executability sanity (reconstruct s₀ with input files only; reject checkers that already pass non-trivially); LLM review for requirement coverage and over-strictness; rubric overlap check (remove rules already covered by code).

Human sample of 50 tasks (Table 2) on a 1–5 scale: task reasonableness 4.46, execution feasibility 3.50, resource consistency 4.36, verification quality 3.92, overall **4.06**.

### Stage 4 — Black-box rollout

Deploy many OpenClaw Docker containers on a cluster; each container is a sealed executable. For each task, the runtime solves it; a **proxy layer** intercepts all model-server traffic (inputs, outputs, tool calls, feedback) without modifying agent logic. Teacher models: MiniMax-M2.5 + GLM-5.1, mixed. After rollout:

- Group requests by identical prefixes → concatenate to recover coherent multi-turn sequences.
- Strip auxiliary prompts (cron, heartbeats) and unsupported tools.
- **Reward-threshold filter** at τ = 0.5 (Figure 5): below 0.5 introduces noisy supervision; above 0.5 prunes valuable behavioral diversity (recovery, partial strategies). 

Survivors: **24.5K high-fidelity trajectories** (Table 3). Average rounds 13.00, tokens 18.67K, tool calls 15.82, tool types 3.25 — rich multi-turn supervision spanning planning, inspection, execution, and adjustment.

### Stage 5 — Agentic training

**Supervised fine-tuning** of Qwen3 (4B / 8B / 30B-A3B). YaRN extends Qwen3-8B from 32K to 64K tokens. **Multi-turn loss masking** excludes environment-feedback tokens from the SFT loss — the model is supervised on its *own* reasoning, decisions, and tool invocations, not on mimicking environment responses. Result: ClawGym-{4B, 8B, 30B-A3B}.

**Reinforcement learning** with a lightweight sandbox-parallel pipeline. Each task spins up an isolated sandbox (filesystem + workspace + gateway + verifier). Parallel rollouts without interference; low infrastructure dependency (Docker or Docker-free). Reward is the **code-verifier output directly** — no auxiliary reward model. Algorithm: GRPO with LR=1e-6, batch=8, 8 rollouts/prompt, 100 steps, temp=0.7, max_response=64K. RL gains stack on both vanilla 4B (no prior SFT) and SFT 30B-A3B — complementary signals.

### Stage 6 — Benchmark construction

Difficulty-aware filtering: run n=4 rollouts per candidate task with strong (GPT-5.4) and small agents; retain tasks satisfying  s̄_strong ≥ 0.2,  s̄_small ≤ 0.6, and  s̄_strong > s̄_small — drop trivially easy and impossibly hard tasks; preserve the discriminative gap. LLM-assisted human review (GPT-5.4 diagnostic + human accept/revise/reject). Final benchmark: **200 instances across 6 categories** — Productivity & collaboration (44), Systems & automation (42), Analysis & reasoning (35), Content & domain support (28), Planning & knowledge (26), Software development (25). Stability (Table 5): std ≤ 1% across 5 repeated runs.

## Empirical results

### Headline (Table 6, condensed)

| Model | PinchBench | ClawGym Avg |
|---|---:|---:|
| Claude-4.7-Opus | 79.40 | 77.81 |
| Gemini-3-Flash | 88.70 | 73.50 |
| Qwen3-235B-A23B (untrained) | 60.60 | — |
| Qwen3-8B (untrained) | 54.50 | 35.02 |
| Qwen3-30B-A3B (untrained) | 55.60 | 45.11 |
| **ClawGym-8B** | **75.70** (+39%) | **50.24** (+43%) |
| **ClawGym-30B-A3B** | **86.00** (+55%) | **56.82** (+26%) |

ClawGym-30B-A3B's 86.00 on PinchBench *exceeds* untrained Qwen3-235B-A23B's 60.60 — high-quality agent-specific data outweighs an order-of-magnitude parameter difference. PinchBench is *external* to the synthesis distribution, so the gain is genuine generalization, not benchmark overfit.

Discriminative spread across the 6 ClawGym-Bench categories is broad (35.02 → 77.81), and no single agent dominates all categories — Claude-4.7-Opus overall, GPT-5.4 in Productivity, Gemini-3-Flash in Software Dev. The variation makes the benchmark useful as a *probe* of capability composition, not just a leaderboard.

### Synthesis-strategy ablation (Table 7)

| Base | Source | ClawGym-Bench | PinchBench |
|---|---|---:|---:|
| Qwen3-8B | Persona only | 49.44 | 73.51 |
| Qwen3-8B | Skill only | 49.06 | 68.23 |
| Qwen3-8B | **Mixed** | **50.24** | **75.68** |
| Qwen3-30B-A3B | Persona only | 53.65 | 84.92 |
| Qwen3-30B-A3B | Skill only | 52.27 | 80.05 |
| Qwen3-30B-A3B | **Mixed** | **56.82** | **86.00** |

Mixed is consistently best by 1–3 pp; the gain is larger on PinchBench (external transfer) than on ClawGym-Bench (in-distribution).

### Other ablations

- **Training dynamics (Figure 4).** Performance peaks at the end of epoch 3 (step 309); beyond that, overfitting on the synthesized distribution causes steady decline. Net gain at peak: +5.8 on ClawGym-Bench, +10.5 on PinchBench.
- **Reward threshold (Figure 5).** τ = 0.5 is optimal on both benchmarks; ±0.1 reduces performance. Below 0.5: noisy supervision degrades training; above 0.5: behavioral diversity is over-pruned.
- **RL starting point.** GRPO improves both vanilla 4B (without prior SFT) and SFT-30B-A3B (with prior SFT). RL is complementary to supervised training, not a replacement.

## Failure modes and limitations

The behavioral analysis in Section 7 is unusually candid:

- **Tool-use appropriateness (Figure 6).** Weaker agents invoke valid tools but fail to compose them into coherent **discovery → inspection → computation → verification** pipelines. They recover from local errors but produce brittle solutions that only partially satisfy the verifier.
- **Long-horizon execution (Figure 7).** Agents accumulate unresolved errors instead of treating failures as recoverable feedback. Dead-ends (e.g., approval requests that never come) block progress instead of triggering fallback strategies. No idempotency tracking across reruns.
- **Fine-grained instruction following (Figure 8).** Plausible artefacts violate core filtering rules ("Quantity ≤ ReorderPoint"); errors propagate downstream (bad rows → bad totals → bad per-supplier outputs).

Authors flag for future work: finer-grained supervision on intermediate decisions, cross-file consistency, trajectory-level safety/efficiency/recovery (current verification scores final state only), and a more principled rule for the reward threshold.

Pipeline limitations:
- Mock files may not capture real-domain complexity.
- ClawGym focuses single-agent (or simple multi-agent) topologies; complex multi-agent coordination is out of scope.
- Verification weighting (0.7) is empirical; production deployments may need to retune.

## When to use, when not

**Use** when building personal/workplace agents that operate on local files, tools, and persistent workspaces (document automation, file organization, data transformation); training compact open-weight models (4B–30B) for computer-use; evaluating Claw-style agent quality on realistic workspace-grounded tasks; iterating on agent capabilities via synthesized data + black-box rollout instead of expensive real-user workflows.

**Don't use** for real-time safety-critical workflows where first-attempt idempotency is required; pure-reasoning tasks without structured tool interfaces; privacy-sensitive environments where mock files don't capture domain complexity; or coordination-heavy multi-agent settings beyond simple hierarchies.

## Implications for harness engineering

- **Black-box rollout via proxy is a transferable harness pattern.** Most production harnesses (Claude Code, OpenClaw, AutoClaw, internal copilots) are encapsulated and not designed to emit training data. ClawGym's proxy interception is generic: as long as model calls go over HTTP, you can collect authentic trajectories without touching agent logic. See [29-dive-into-claude-code](29-dive-into-claude-code.md) and [52-dive-into-open-claw](52-dive-into-open-claw.md) for the harness internals this pattern has to coexist with.
- **Hybrid (code + rubric) verification with 0.7/0.3 split is the right default.** Pure code-check scoring is too brittle; pure LLM-judge scoring is too soft and leaks. The 0.7/0.3 weighting privileges artefacts (the actual workspace change) while still supervising qualitative dimensions. Drop this directly into [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md)-style harnesses.
- **Multi-turn loss masking is non-obvious and important.** Excluding environment-feedback tokens from the SFT loss focuses optimization on *what the model decides*, not on *what the environment said*. Without it, the model overfits to specific tool-output formats and degrades on out-of-distribution observations. Should be the default for any computer-use SFT recipe.
- **Sandbox-parallel GRPO with code-only reward.** No reward model, no preference data — direct verifier output is enough signal. Cheap to run, robust to hyperparameter changes, and lets the same verifier be reused across SFT and RL. Mirrors the simplicity advocated in [69-agent-world](69-agent-world-self-evolving-training-arena.md) for self-evolving training arenas.
- **Dual-route synthesis vs single-route.** Persona-driven (top-down from user intent) covers the diversity gap; skill-grounded (bottom-up from capability inventory) covers the feasibility gap. Either alone produces brittle distributions. The 1–3 pp consistent uplift from mixing is small per-paper but compounds across the recipe. See [04-skills](04-skills.md) for the SKILL.md primitive that the bottom-up route distills, and [68-atomic-skills](68-atomic-skills-scaling-coding-agents.md) for how atomic-skills scale the same pattern up to coding agents.
- **Difficulty-aware benchmark filtering.** Strong-vs-weak gap retention (s̄_strong ≥ 0.2, s̄_small ≤ 0.6, s̄_strong > s̄_small) is the cleanest known method for keeping a benchmark *discriminative* and *solvable*. Under-used in agent eval. See [38-claw-eval](38-claw-eval.md) for adjacent trajectory-aware Claw evaluation, and [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md) for the broader judge-trajectory framing this slots into.
- **High-quality data eats parameter scale.** ClawGym-30B-A3B (86.0 on PinchBench) > untrained Qwen3-235B-A23B (60.6) is the headline lesson for any team facing budget vs quality choices. The expensive piece is the synthesis + verification + rollout pipeline; once it exists, the model size choice becomes secondary. Pairs with [55-hermes](55-hermes-agent-self-improving.md) and [19-voyager-skill-libraries](19-voyager-skill-libraries.md) on the broader self-improving agent thread.
- **Agent-loop discipline.** The behavioral analysis (recovery, idempotency, instruction fidelity) maps directly onto [01-agent-loop-architecture](01-agent-loop-architecture.md). The paper's open problems are open *agent-loop design* problems, not data problems: the loop has to expose error-recovery and cross-file-consistency state to the model, not bury them in observations.

The one-line takeaway for harness designers: **ClawGym shows the full data → training → eval loop is now an *engineering problem*, not a research problem — and the 0.7-code/0.3-rubric verifier plus proxy-based black-box rollout are the two patterns to copy first.**
