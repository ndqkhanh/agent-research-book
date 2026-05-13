# 160 — Deep Researcher Agent: 24/7 Autonomous Deep-Learning Experimentation with Zero-Cost Monitoring

**Paper.** Xiangyue Zhang — *Deep Researcher Agent: An Autonomous Framework for 24/7 Deep Learning Experimentation with Zero-Cost Monitoring* — arXiv:2604.05854v1 [cs.AI] — submitted 7 April 2026 — **The University of Tokyo** (single author). Code: https://github.com/Xiangyue-Zhang/auto-deep-researcher-24x7. The paper is a deeply *operational* artefact: an end-to-end framework that has run in production for 30+ days across 4 concurrent ML research projects, completed 500+ autonomous experiment cycles, and achieved a **52% improvement over baseline metrics** in one project through **200+ automated experiments** at an average LLM cost of **$0.08 per 24-hour cycle**.

**One-line definition.** Deep Researcher Agent is an open-source autonomous-experimentation framework built around three architectural innovations that jointly make 24/7 LLM-driven deep-learning research economically viable: **(1) Zero-Cost Monitoring** — a paradigm that incurs *zero LLM API calls* during the GPU-training phase (which constitutes 90–99% of wall-clock time) by using only OS-level process checks (`kill -0`, `nvidia-smi`, log-tail reads); **(2) Two-Tier Constant-Size Memory** — a memory architecture capped at ~5,000 characters (3K immutable Brief + 2K rolling Log) that prevents unbounded context growth regardless of runtime duration; and **(3) Minimal-Toolset Leader-Worker Architecture** — a multi-agent design where each worker has only 3–5 tools, reducing per-call token overhead by 73% compared to full-toolset approaches. Together with prompt caching, mandatory dry-runs, anti-burn cooldowns, and a HUMAN_DIRECTIVE.md mechanism for asynchronous human-in-the-loop interaction, the framework turns the THINK→EXECUTE→REFLECT loop into a sustainable production substrate.

## Why this paper matters disproportionately

The two surveys ([158](158-deep-research-agents-survey-huang-et-al.md), [159](159-deep-research-survey-zhang-et-al.md)) chart the territory of DR agents that synthesise *information*. This paper takes the same DR-agent framework and applies it to a different problem class: agents that synthesise *experimental knowledge* — running, monitoring, and iterating on actual GPU training runs.

The distinction is operational. Information-DR agents do retrieval, reasoning, and report generation; their work is mostly LLM tokens with web traffic. Experimentation-DR agents must:

- **Launch and manage GPU training jobs** that run for hours.
- **Monitor training progress** without burning tokens on every poll.
- **Detect crashes, divergences, NaN losses, etc.** in real time.
- **Read training logs and decide what to try next.**
- **Maintain coherent state across days and weeks** of experiments.

Naive instantiation of an information-DR agent for this workload doesn't work. A standard polling agent that queries the LLM every 5 minutes during training would cost $50+ per day per project. The whole pipeline becomes economically infeasible. Zhang's contribution is the architectural engineering that makes it feasible: **$0.08 per 24-hour cycle with 24/7 autonomy across 4 concurrent projects = ~$10/month per project**, which is finally cheaper than human researcher time.

For the harness-engineering canon, this paper sits in a unique position. It is one of the very few publications that:

- Reports *real-world long-running deployment* metrics (30+ days, 500+ cycles).
- Quantifies the *cost structure* of autonomous agent operation in detail.
- Shows that *minimal-architecture* agents can outperform full-feature ones at this workload.
- Provides a *single-author, single-paper, open-source artefact* that practitioners can pick up and run.

It is the operational counterpart to the architectural arguments in MEMTIER ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)), Feynman ([155](155-feynman-multi-agent-research-harness.md)), and the synthesis in [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md). Where those papers describe substrates, this one describes a working production system built on equivalent substrates.

## The problem the paper solves

The deep-learning research workflow is fundamentally iterative:

```
1. Researcher designs experiment (hypothesis, model, training config)
2. Launches GPU training (typically hours to days)
3. Analyses results
4. Adjusts hyperparameters / architecture
5. Repeats — often hundreds of times before publication
```

Despite the mechanical nature of the loop, it remains overwhelmingly manual: researchers must be present to check completion, interpret results, and decide next steps. Existing AI research assistants address adjacent territory but not this one:

- **AI Scientist** ([6]): generates papers including experiments — but experiment execution is short-running scripts, no GPU training, no iterative refinement.
- **Claude Scholar** ([10]): research writing workflows with 47 skills + Zotero integration — reactive only, no autonomous execution.
- **SWE-Agent / OpenHands**: software engineering tasks (bug fixing, code review) — not research-iteration capable.
- **AutoML / Optuna / Ray Tune**: efficient hyperparameter search — pre-defined search spaces only, cannot modify model architectures or pipelines.
- **MLAgentBench / ResearchAgent**: ML-agent benchmarks and idea-generation respectively — not full lifecycle.

The gap: **autonomous execution and iteration of GPU experiments over weeks**. Deep Researcher Agent fills it.

## The THINK → EXECUTE → REFLECT loop (Algorithm 1)

The entire framework is structured as a cyclic three-phase loop:

```
Algorithm 1: Deep Researcher Agent Main Loop
  Require: Project brief B, initial memory M_0
  1: t ← 0
  2: while not terminated do
  3:   t ← t + 1
  4:   d ← CONSUME_DIRECTIVE()             # Human override (optional)
  5:   plan_t ← THINK(B, M_{t-1}, d)        # LLM active
  6:   if plan_t.action = "wait" then
  7:     SMART_COOLDOWN()
  8:     continue
  9:   end if
 10:   result_t ← EXECUTE(plan_t)           # LLM → training launch
 11:   if result_t.launched then
 12:     logs_t ← MONITOR(result_t.pid)     # Zero LLM cost
 13:   end if
 14:   M_t ← REFLECT(B, M_{t-1}, result_t, logs_t)  # LLM active
 15: end while
```

Three phases:

1. **THINK**: LLM-active. Reads project brief + memory + (optional) human directive. Produces an experiment plan. Cost: ~$0.05 per cycle.
2. **EXECUTE**: LLM-active to *launch*; then zero-cost monitoring. Implements code changes, runs mandatory dry-run, launches GPU training, monitors to completion. The launch itself is an LLM call (~$0.08 worth of tokens). Monitoring after launch is free.
3. **REFLECT**: LLM-active. Parses training logs, evaluates metrics, updates memory log, decides next action. Cost: ~$0.03 per cycle.

The full cycle averages **$0.08–$0.16 in LLM cost per 24-hour iteration** (Table 2 in the paper), of which the bulk is THINK + EXECUTE token consumption. Compared to a conventional polling agent at $1.60/day, this is a **10–20× cost reduction**.

## Innovation 1 — Zero-Cost Monitoring

This is the central architectural insight of the paper. *During GPU training (90–99% of wall-clock time), the LLM has nothing useful to contribute.* Training follows a predetermined schedule; intermediate metrics are written to log files automatically. Polling the LLM during this period is wasted compute.

The framework replaces LLM polling with three OS-level checks at configurable intervals (default 15 minutes):

```bash
# Check 1: Process liveness
kill -0 $PID    # Single syscall; negligible cost

# Check 2: GPU utilization (catches silent crashes)
nvidia-smi      # Confirms GPU is actually being used

# Check 3: Log tail
tail -50 train.log    # Last 50 lines for local logging
```

The LLM is only invoked when the training process *terminates* (detected by `kill -0` returning non-zero), at which point the accumulated log tail is passed to REFLECT.

### Cost analysis (Table 2)

For a 24-hour cycle with 8 hours of training:

| Component | Conventional polling agent | Deep Researcher Agent |
|-----------|---------------------------|------------------------|
| Active calls (THINK) | 30 min, 15 calls, 50K tokens, $0.16 | 5–10 min, 3–5 calls, ~15K tokens, **$0.05** |
| Execute (launch) | (folded into above) | 10–20 min, 5–10 calls, ~25K tokens, **$0.08** |
| Monitor (during training) | 6–8h, 96 calls, ~192K tokens, $0.50 | 6–8h, **0 calls, 0 tokens, $0.00** |
| Idle poll | 15h, 180 calls, ~360K tokens, $0.94 | (not present) |
| Reflect | (folded into above) | 5–10 min, 2–3 calls, ~10K tokens, **$0.03** |
| **Total per 24h** | **291 calls, ~602K tokens, $1.60** | **10–18 calls, ~50K tokens, $0.08–0.16** |

A single research project running 24/7 for 30 days costs ~$2.40–$4.80 with this architecture. A conventional polling agent would cost ~$48.00 — a 10–20× reduction.

The architectural lesson generalises far beyond ML experimentation: **for any agent running tasks where most wall-clock time is spent in non-LLM-decidable work, replace polling with OS-level event detection.** This same insight applies to:

- Build-system agents waiting for compilation.
- CI/CD agents waiting for pipeline runs.
- Data-pipeline agents waiting for Spark jobs.
- Any agent waiting for a long-running tool to complete.

The pattern: **only invoke the LLM at state-transition events**, not on a polling schedule.

## Innovation 2 — Two-Tier Constant-Size Memory

Long-running LLM agents face a structural problem: accumulated context grows without bound, leading to (a) degraded LLM performance as context length increases, (b) escalating API costs proportional to context size, and (c) eventual context window overflow.

This paper's solution is a memory architecture **bounded at ~5,000 characters (~1,500 tokens)** that remains constant regardless of how long the agent runs.

### Tier 1: Project Brief (B) — 3,000 chars max

A *human-authored, frozen* document describing:
- Research goal
- Codebase structure
- Constraints
- Success criteria

The agent **cannot modify** this tier. This guarantees the research direction stays stable across the entire deployment.

### Tier 2: Memory Log (M) — 2,000 chars max

An *agent-maintained rolling log* with two sub-sections:

- **Key Results** (≤1,200 chars): milestone entries recording significant experimental outcomes. Example: `"Exp003: ViT-B/16, lr=3e-4 + cosine, acc=77.9% — new best!"`. Auto-compacted: when the section exceeds 1,200 chars, the *oldest* entry is removed.
- **Recent Decisions** (no char cap, but limited to 15 entries): a rolling log of the agent's reasoning for each decision. Auto-compacted: only the most recent 15 entries retained, regardless of total character count.

### The bounded-size guarantee

```
|M_t|  ≤  |B|_max + |L|_max  =  3000 + 2000  =  5000 chars,  ∀t
```

The agent has run for 1 day or 6 months — same memory size. This bypasses the long-horizon-context-rot problem entirely.

The compaction is **lossy by design**:

> The agent retains the most valuable information (recent decisions and best results) while discarding routine entries. This mirrors how human researchers maintain a mental model: remembering key milestones and recent context while forgetting routine details.

### Empirical validation (Table 4)

| Time point | Tier 1 (Brief) | Tier 2 (Log) |
|-----------|----------------|---------------|
| Day 1 (cycle 1) | 2,847 chars | 312 chars |
| Day 7 (cycle 25) | 2,847 chars | 1,834 chars |
| Day 14 (cycle 55) | 2,847 chars | 1,956 chars |
| Day 30 (cycle 120) | 2,847 chars | 1,978 chars |

Tier 2 stabilises near its 2,000-char cap within the first week and stays bounded thereafter. The system has reached steady state — and steady state is stable.

### Comparison with MEMTIER

This memory architecture is a *very different design point* from MEMTIER ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)):

| Property | Deep Researcher Agent | MEMTIER |
|----------|----------------------|---------|
| Memory size | Bounded constant (~5K chars) | Unbounded (tiered storage) |
| Retrieval | None (entire memory is in context) | Two-stage scoping with 5-signal scoring |
| Knowledge tier | Single-project specific | Project-shared + agent-private |
| Forgetting policy | LRU (compaction) | Tier-based with cognitive weights |
| Workload | Single-project deep iteration | Multi-session multi-agent recall |

Both architectures are correct for their respective workloads. Deep Researcher Agent's domain is *vertically deep within one project* — the agent doesn't need cross-project recall, it needs stable working memory across days of one project. MEMTIER's domain is *horizontally broad across many sessions* — the agent needs to recall facts from sessions weeks ago.

The lesson: **memory architecture should match workload structure**. There is no universal best answer.

## Innovation 3 — Minimal-Toolset Leader-Worker Architecture

The framework uses a Leader-Worker pattern with deliberately *minimal* tool sets per agent.

### Leader Agent (3 tools)

The strategic decision-maker. Maintains a persistent conversation *within each cycle* for coherent multi-step reasoning. Conversation history **is reset between cycles** to prevent unbounded growth.

Tools: `log_memory`, `write_file`, `read_file` (3 tools).

### Worker Agents (3–5 tools each)

Three specialised workers, each with a minimal tool set:

| Worker | Role | Tools |
|--------|------|-------|
| **Idea Agent** | Literature search and hypothesis formation | `search_papers`, `get_paper`, `write_file`, `read_file` (4 tools) |
| **Code Agent** | Experiment implementation and execution | `run_shell`, `launch_experiment`, `write_file`, `read_file`, `list_files` (5 tools) |
| **Writing Agent** | Report and analysis generation | `write_file`, `read_file`, `list_files` (3 tools) |

**Only one worker runs at a time**; others are completely idle at zero token cost. The Leader dispatches at most 3 worker tasks per cycle.

### Why minimal tool sets matter

Each tool definition adds approximately 200 tokens to every API call (name, description, parameter schema). A typical agent framework provides 15+ tools to every agent, adding ~3,000 tokens of overhead per call.

This framework averages 4 tools per agent (~800 tokens) — a 73% reduction. Over 100 API calls per day, this saves ~220K tokens, translating to meaningful cost savings and faster response times.

The lesson: **tool sprawl is a hidden cost**. If you add 10 tools "in case the agent needs them", you pay 2,000 extra tokens on every single call. For a high-frequency agent, this dominates the actual reasoning cost.

The cure is *agent specialisation*: instead of one agent with all tools, decompose into multiple agents each with the minimum tool set for their task. Coordination cost is trivial compared to per-call token waste.

## Safety mechanisms

The framework includes four safety mechanisms — operational details that matter for production use:

### Mandatory Dry-Run

Before any real training launch, the Code Agent must execute a short dry-run (typically 2 forward-backward steps) to verify the code runs without errors. **Catches 18% of planned experiments before they are launched** (Table 1) — preventing wasted GPU hours from configuration mistakes, missing imports, tensor shape mismatches.

### Protected Files

Critical state files cannot be overwritten by worker agents:
- `state.json`
- `MEMORY_LOG.md`
- `PROJECT_BRIEF.md`

Prevents accidental corruption of the agent's memory or configuration.

### Human Override (3 mechanisms)

1. **HUMAN_DIRECTIVE.md** — a file consumed at the start of each cycle with highest priority. Asynchronous human-in-the-loop. Archived with timestamp after consumption to prevent re-reading.
2. **--directive flag** — command-line one-time instructions.
3. **Direct memory modification** — edit MEMORY_LOG.md for permanent behavioural changes.

The HUMAN_DIRECTIVE mechanism is particularly elegant. It enables *mobile human-in-the-loop* through companion apps like Happy Coder — the researcher can push directives from their phone, and the agent picks them up at the next cycle boundary.

### Anti-Burn Protection

If consecutive cycles produce no meaningful output (repeated errors, empty results), the cooldown interval is exponentially increased (up to 30 minutes) to prevent wasteful token consumption. Failure cases don't compound into runaway cost.

## Operational results (Table 1)

The deployment evidence is the paper's strongest argument:

| Metric | Value |
|--------|-------|
| Total autonomous experiment cycles | **500+** |
| Longest continuous operation | **30+ days** |
| Concurrent projects managed | **4** |
| GPU servers utilised | **4** |
| Best single-project improvement | **52% over baseline** |
| Experiments in best project | **200+** |
| Average LLM cost per 24h cycle | **$0.08** |
| Average experiments per day per project | **2–4** |
| Dry-run failure rate (caught pre-training) | **18%** |
| Post-dry-run training crash rate | **<3%** |

Hardware: 4 GPU servers with NVIDIA L20X 144GB GPUs (Chinese-domestic GPUs). LLM backbone: Claude Sonnet with prompt caching enabled.

**The autonomous-improvement story (200+ experiments → 52% improvement)** is the headline. The agent autonomously explored 200+ configurations over weeks, with diminishing returns: most gains in the first 50 experiments, fine-grained optimisation thereafter. **Human intervention frequency: ~once every 3–5 days**, primarily for major direction changes.

This means the agent makes day-to-day decisions about hyperparameter exploration, learning rate scheduling, and regularisation strategies *fully autonomously*. The human researcher sets project direction and intervenes on architectural pivots; the agent runs the search.

## The 8 cost-control strategies

Table 3 enumerates the cost-control strategies that compose to make the framework economically viable:

| # | Strategy | Mechanism |
|---|---------|-----------|
| 1 | Zero-LLM monitoring | No API calls during training |
| 2 | Constant-size memory | Fixed at ~1.5K tokens |
| 3 | Within-cycle persistence | Brief sent once per cycle (cached) |
| 4 | Prompt caching | System/tool schemas cached |
| 5 | Minimal tool sets | 3–5 tools vs 15+ (73%↓) |
| 6 | Slim prompts | Agent prompts <500 tokens |
| 7 | State trimming | Redundant context removed |
| 8 | Single-worker execution | No parallel LLM costs |

Each individually saves 10–30% of token cost. *Together they multiply* to a 10–20× reduction. This is the engineering principle that makes the framework distinctive: cost control is not a single design choice but a **stack of compositional engineering decisions**.

## Configuration reference (from Appendix A)

The framework's full YAML configuration:

```yaml
project:
  name: "my-research"
  brief: "PROJECT_BRIEF.md"
  workspace: "./workspace"

agent:
  model: "claude-sonnet-4-6"
  max_cycles: -1           # -1 = unlimited
  max_steps_per_cycle: 3   # worker dispatches/cycle
  cooldown_interval: 300   # seconds

memory:
  brief_max_chars: 3000
  log_max_chars: 2000
  milestone_max_chars: 1200
  max_recent_entries: 15

gpu:
  auto_detect: true
  reserve_last: true       # last GPU for keep-alive

monitor:
  poll_interval: 900       # 15 minutes
  zero_llm: true

experiment:
  mandatory_dry_run: true
  max_parallel: 1
```

The configuration is admirably small. Most production agent frameworks have dozens of tunables; this one has ~15. Defaults are sensible and the headline parameters (`zero_llm: true`, `mandatory_dry_run: true`) are operational invariants rather than tuning knobs.

## Agent prompt structure (Appendix B)

Each agent is defined as a **Markdown file with YAML frontmatter** specifying name, description, and model. Example for the Code Agent:

```yaml
---
name: code_agent
description: Experiment implementation
model: inherit
---
# Code Agent
You are the Code agent. Your role is to implement and run experiments.

## Mandatory Workflow
1. Understand the Leader's task
2. Implement code/config changes
3. Dry-run (MANDATORY - abort if fails)
4. Launch via launch_experiment tool
5. Report PID and log file path

## Constraints
- NEVER skip dry-run
- ALWAYS use launch_experiment for training
- Do NOT modify protected files
```

This is the *same skill-as-Markdown* pattern that Feynman ([155](155-feynman-multi-agent-research-harness.md)), HeavySkill ([156](156-heavyskill-parallel-reasoning-deliberation.md)), and the broader "skills as portable artefacts" thesis ([157](157-may-2026-synthesis-memory-and-skills.md)) consolidate. Independent confirmation, in a different production system, that Markdown is the right serialisation format for agent definitions.

## Limitations the paper acknowledges

Four named limitations:

1. **Single-GPU scope**. The current open-source release supports single-GPU experiments only. Multi-GPU distributed training (DDP) and multi-server orchestration are planned for future releases.
2. **Metric extraction by regex**. Log parsing relies on regex pattern matching, which may miss custom metric formats. Structured logging (e.g., JSON Lines) would improve robustness.
3. **No formal exploration strategy**. Experiment planning relies on the LLM's reasoning capabilities without formal search methods like Bayesian optimisation. Integrating structured search could improve sample efficiency.
4. **Evaluation methodology is open**. Unlike software-engineering agents that can be tested on fixed benchmarks (SWE-Bench), research agents operate in open-ended domains where the "correct" next experiment is undefined. Standardised evaluation protocols for long-running research agents are an open problem.

A fifth implicit limitation worth naming: **the framework assumes the underlying training scripts are well-behaved**. If a training script doesn't produce parseable logs, doesn't write metrics in a recognisable format, or hangs without error, the agent will struggle. The framework's success rests on the existence of robust scripts to monitor.

## Why Zero-Cost Monitoring is the most reusable contribution

Of the three innovations, **Zero-Cost Monitoring is the one that generalises most broadly**. The Two-Tier Memory and Minimal-Toolset patterns are standard architectural moves; the Leader-Worker structure is mainstream multi-agent design. But the *paradigm* of replacing LLM polling with OS-level event detection — and of structuring agent loops so the LLM is *only invoked at state transitions* — is the architectural lesson that scales to many other domains:

- **CI/CD orchestration agents**: poll process exit codes and webhook events instead of LLM-querying for build status.
- **Long-running data-pipeline agents**: monitor airflow/dagster/temporal events instead of LLM-checking dataset materialisation.
- **Multi-day research-watch agents**: poll RSS feeds and arxiv listings; only invoke LLM when new content arrives.
- **Customer-support escalation agents**: subscribe to ticket-event streams; LLM-process at decision points.

The general pattern: **build the agent's loop around the underlying system's events, not a fixed time interval**. This single insight reduces operational cost by an order of magnitude in any domain where the underlying work is non-LLM-decidable.

This is the same insight that durable-execution substrates ([150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md)) embed: workflows wait on signals, not on polls. Deep Researcher Agent applies the principle within an agent loop.

## How this paper relates to the rest of the canon

- **Memory architecture contrast**: Deep Researcher Agent's bounded-constant memory vs MEMTIER's tiered structure ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)). Both correct, for different workloads. The synthesis at [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md) covers the choice.
- **Skill-as-Markdown convergence**: agents-as-Markdown-files independently appears here, in Feynman ([155](155-feynman-multi-agent-research-harness.md)), in HeavySkill ([156](156-heavyskill-parallel-reasoning-deliberation.md)), in DeerFlow 2.0 ([163-deer-flow-revisited](163-deer-flow-revisited.md)). The format has won.
- **Cost analysis discipline**: most agent papers don't quantify per-cycle cost in detail. This paper's Table 2 cost breakdown is one of the cleanest published. Useful as a reference for any builder doing operational economics.
- **Single-author single-paper open-source pattern**: alongside Memento ([106-109](106-memento-paper-theory.md)), MEMTIER ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)), HeavySkill ([156](156-heavyskill-parallel-reasoning-deliberation.md)), this is the rising paper format — a focused architectural contribution + working code, often single-institution or single-author. The PhD-thesis-as-paper-and-codebase pattern.

## Practical takeaways for builders

If you are operating an agent that runs for many hours and waits on long-running external work (training, builds, pipelines, data jobs), this paper's design choices apply:

### 1. Build your agent loop around state transitions, not polling

`kill -0 $PID` runs in microseconds. An LLM poll runs in seconds and costs $0.001+. If you can detect the state you care about with an OS primitive, do that. Reserve LLM calls for *decision points*, not status checks.

### 2. Bound your memory, regardless of run duration

Either with hard caps (this paper) or with tiered structure + retrieval (MEMTIER). Unbounded context is a guaranteed degradation curve.

### 3. Audit your tool overhead

Count your tool count. Multiply by ~200 tokens per tool definition. Multiply by tool-call frequency. Many agent frameworks have 15+ tools per agent for marginal capabilities. If your agent makes 100 tool calls per day, that's 300K tokens of tool definition overhead alone. Specialise into multiple narrow agents.

### 4. Add anti-burn protection

If your agent loops on errors, exponential backoff prevents runaway costs. This is operational hygiene, but most agent frameworks ship without it.

### 5. Use prompt caching aggressively

If your project brief or system prompt is long and stable, cache it. The savings compound over hundreds of calls per day. This paper relies on it for the $0.08/24h cost figure.

### 6. Make every agent definition a Markdown file

Don't bury agent prompts in code. The agent_code_agent example in the appendix shows the discipline: YAML frontmatter for metadata, Markdown body for the system prompt. Easy to audit, easy to version, easy to share.

### 7. HUMAN_DIRECTIVE.md is a beautiful pattern

A file the human writes; the agent consumes at the next cycle boundary; archived with timestamp. Asynchronous human-in-the-loop without a real-time interface. This is the right primitive for autonomous-but-supervised systems.

## Where this fits in the canon

- **Read alongside**: [157-may-2026-synthesis-memory-and-skills](157-may-2026-synthesis-memory-and-skills.md), [151-memtier-why-flat-memory-breaks-at-72-hours](151-memtier-why-flat-memory-breaks-at-72-hours.md), [155-feynman-multi-agent-research-harness](155-feynman-multi-agent-research-harness.md).
- **Operational concerns**: [10-multi-session-continuity](10-multi-session-continuity.md), [24-observability-tracing](24-observability-tracing.md), [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md), [144-build-your-own-harness](144-build-your-own-harness.md), [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md).
- **Multi-agent architecture**: [02-subagent-delegation](02-subagent-delegation.md), [42-langchain-deep-agents](42-langchain-deep-agents.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md), [127-loan-processing-multi-agent-case-study](127-loan-processing-multi-agent-case-study.md).
- **Cost economics**: [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md), [88-confidence-driven-router](88-confidence-driven-router.md).
- **Companion DR surveys**: [158-deep-research-agents-survey-huang-et-al](158-deep-research-agents-survey-huang-et-al.md), [159-deep-research-survey-zhang-et-al](159-deep-research-survey-zhang-et-al.md).
- **Synthesis**: [166-research-agent-frameworks-synthesis](166-research-agent-frameworks-synthesis.md).

## References

1. Zhang. 2026. *Deep Researcher Agent: An Autonomous Framework for 24/7 Deep Learning Experimentation with Zero-Cost Monitoring*. arXiv:2604.05854v1.
2. Code: https://github.com/Xiangyue-Zhang/auto-deep-researcher-24x7.
3. Akiba et al. 2019. *Optuna: A next-generation hyperparameter optimization framework*. KDD 2019. Adjacent AutoML.
4. Liaw et al. 2018. *Tune: A research platform for distributed model selection and training*. arXiv:1807.05118. Ray Tune.
5. Lu et al. 2024. *The AI Scientist*. arXiv:2408.06292. Adjacent autonomous research framework.
6. Wang et al. 2024. *OpenHands*. arXiv:2407.16741. Software-engineering agent.
7. Yang et al. 2024. *SWE-agent*. arXiv:2405.15793. Adjacent SE-agent.
8. Huang et al. 2024. *MLAgentBench*. ICML 2024. Benchmark for ML agents.
9. Baek et al. 2024. *ResearchAgent*. arXiv:2404.07738. Idea-generation agent.
10. Zhang. 2026. *Claude Scholar*. https://github.com/Galaxy-Dawn/claude-scholar.
11. Slopus. 2025. *Happy: Mobile and web client for codex and Claude Code*. https://github.com/slopus/happy.
