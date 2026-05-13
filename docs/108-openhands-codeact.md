# 108 — OpenHands & CodeAct: Code-as-Action for Coding Agents

**Project.** OpenHands (formerly OpenDevin). Repo: https://github.com/All-Hands-AI/OpenHands. **CodeAct paper.** Wang et al., *Executable Code Actions Elicit Better LLM Agents* — arXiv:2402.01030 — ICML 2024. CodeAct is OpenHands' default action representation.

**One-line definition.** OpenHands is the **most active open-source coding agent project of 2025–2026**, built around the **CodeAct** scaffold — agents emit Python code as their action format (instead of JSON tool calls), and execute it in a sandboxed runtime, gaining the full expressive power of a programming language inside the agent loop.

## Why this paper matters

JSON-schema function calling has been the industry-standard agent action format since 2023. CodeAct is the principled rejection of that pattern. The paper showed (and OpenHands' SWE-bench performance has confirmed) that **letting agents write Python code as their primary action consistently outperforms structured tool-call JSON** on multi-step tasks — because Python composes (loops, conditionals, function reuse) where JSON tool calls don't. OpenHands' rapid trajectory on the SWE-bench Verified leaderboard makes this the OSS coding-agent reference architecture.

## Problem it solves

1. **JSON tool calls don't compose.** A multi-step plan in JSON is N independent calls; each result handled separately. Python lets the model loop, branch, and reuse intermediate values in a single emitted block.
2. **Expressing complex tool sequences in JSON is verbose.** "Read all `.py` files, count TODOs, sort by file" is many turns in JSON; one Python loop in CodeAct.
3. **Agents need a real environment.** SWE-bench tasks involve writing tests, running them, reading stack traces, editing — a sandbox terminal + filesystem is the natural surface.
4. **Open-source SWE-bench performance had stalled.** CodeAct + OpenHands moved the OSS frontier from ~27% (Oct 2024) to north of 60% on Verified, closing much of the gap to closed agents.

## Core idea in one paragraph

The agent's **action vocabulary is Python**. On each step, the LLM emits a Python code block (alongside optional natural-language thinking) which is executed in a Jupyter-kernel-backed sandbox container. Stdout, stderr, and exceptions stream back as observations. The agent's "tools" are simply Python functions/CLIs available in that sandbox: `bash` for shell commands, `str_replace_editor` for file edits, `browser` for web actions, etc. The model's expressive freedom — loops, list comprehensions, intermediate variables — turns multi-step JSON tool dances into single coherent code blocks.

## Mechanism (step by step)

### (a) The CodeAct action format

```python
# What the agent emits:
import subprocess
result = subprocess.run(["pytest", "tests/test_auth.py", "-v"], capture_output=True, text=True)
print(result.stdout[-2000:])
print("STDERR:", result.stderr[-500:])
```

The runtime executes this in a stateful Jupyter kernel; subsequent emissions can reference variables defined in earlier steps. Statefulness is the multiplier — JSON tool calls force the agent to re-fetch context every step.

### (b) The OpenHands runtime

A Docker-isolated sandbox per session containing:

- A Linux user with file-system, git, pip, bash.
- A Jupyter kernel for code execution.
- Optional: a headless browser, an MCP server for additional tools.
- Resource limits (CPU, memory, network egress allowlist).

State persists for the session; teardown is automatic on session end.

### (c) The default agent: `CodeActAgent`

The reference agent is a ReAct-style loop ([13-react](13-react.md)) where the action format is restricted to:

- A Python code block (fenced) — executed.
- A `finish` directive — terminates the session.
- A natural-language message to the user — surfaces output without execution.

Everything else (file reads, edits, web browsing, git operations) is just Python the agent writes.

### (d) Specialized agents (planner, etc.)

Beyond `CodeActAgent`, OpenHands ships agents specialized for delegation, browser tasks, and structured planning. The runtime contract is the same; only the system prompt and toolset differ.

### (e) Multi-agent and microagents

OpenHands supports a **microagent** pattern: small specialist sub-agents triggered by keywords or context (e.g., a "GitHub microagent" activates when the task mentions a PR). This is the OSS counterpart of [02-subagent-delegation](02-subagent-delegation.md) and [04-skills](04-skills.md) skill-style auto-loading.

## Empirical results

**SWE-bench Verified (500-task subset)** progression (per the paper, OpenHands releases, and SWE-bench leaderboard tracking):

| Date | System | Verified % |
|------|--------|-----------:|
| 2024-08 | AutoCodeRover | 19% |
| 2024-10 | OpenHands + Claude 3.5 Sonnet | 27% |
| 2025-04 | OpenHands + Claude 4 Sonnet | ~50% |
| 2025-12 | Sonar Foundation Agent (built on AutoCodeRover) + Claude Opus 4.5 | **79.2%** |

The Sonar 79.2% peak combines AutoCodeRover-style indexing with Claude Opus 4.5 in a CodeAct-style scaffold; OpenHands itself remains the OSS reference and is in active development by All-Hands-AI.

The original CodeAct paper showed code-action agents beat JSON-tool-call agents by **~20 percentage points** on subsets of API-Bench and similar — a robust effect that has held up at scale.

## Variants and ablations

- **CodeAct vs JSON tool calls**: per the paper, CodeAct wins on tasks with >2 sequential tool uses; ties or trails on single-call tasks. The break-even is fast.
- **Stateful kernel vs stateless code-runner**: stateful is decisively better — it's how the agent reuses parsed data without serializing.
- **Sandboxed bash + Python vs Python only**: bash adds value for ops-heavy tasks (build, deploy); pure-Python is enough for most editing.
- **Microagents on/off**: domain microagents (Git, Docker) measurably improve task-specific performance and reduce token use by pre-loading relevant context.

## Failure modes and limitations

1. **Sandbox escape risk**. A Python sandbox is famously hard to fully contain. Network allowlists, no-root containers, ephemeral file systems are mandatory.
2. **Long-running execution**. Agents can launch infinite loops or slow processes; runtime needs hard timeouts and OOM kills.
3. **Hidden side effects**. `pip install` can fetch arbitrary code; `git push` can publish; file writes are real. Permission modes ([06-permission-modes](06-permission-modes.md)) and HITL ([23-human-in-the-loop](23-human-in-the-loop.md)) gates apply.
4. **Quality of stack traces in context**. Long Python tracebacks can blow context budgets; truncation strategies matter.
5. **Backbone dependence**. CodeAct works best with strong code-trained models (Claude Opus/Sonnet 4+, GPT-4o+, Qwen2.5-Coder 32B+). Smaller models produce code that fails to execute as often as not.

## When to use, when not

**Use OpenHands / CodeAct when** the task is genuinely software-engineering (codebases, tests, builds), you can run a sandboxed runtime, and you want OSS. The default for "I need a coding agent in my stack" in 2026.

**Don't reach for it** when tasks don't need code execution (plain Q&A, research summaries) — JSON-tool-call agents are leaner. Also avoid when sandbox isolation is non-negotiable and untrusted (regulated environments may forbid arbitrary Python execution).

## Implications for harness engineering

- **Action format is a first-order design choice.** Code-as-action vs JSON-tool-call is the most consequential decision after model selection. Default to code where the runtime allows.
- **The sandbox *is* the harness.** All the harness primitives ([05-hooks](05-hooks.md), [06-permission-modes](06-permission-modes.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md)) wrap the sandbox; nothing else.
- **Stateful kernels need state observability.** Tracing must include kernel state at each step, not just code+output.
- **Microagents = lazy-loaded skills.** OpenHands' microagents and Anthropic Skills ([04-skills](04-skills.md)) are convergent patterns; both lazy-load specialized context to the harness.
- **Resource limits are part of the safety story.** Without CPU/RAM caps, an agent looping on a sort can exhaust the host.

## Connections to other work in this corpus

- **[29-dive-into-claude-code](29-dive-into-claude-code.md), [62-everything-claude-code](62-everything-claude-code.md):** Claude Code is the closed cousin; OpenHands is the OSS analogue.
- **[42-langchain-deep-agents](42-langchain-deep-agents.md):** related OSS coding-agent stack with planning emphasis; both descend from the Claude Code/Devin design space.
- **[52-dive-into-open-claw](52-dive-into-open-claw.md):** another OSS harness; OpenClaw is more meta-harness, OpenHands is more task-runner.
- **[109-smolagents](109-smolagents.md):** smolagents extends code-as-action to the lightweight, runtime-pluggable case.
- **[02-subagent-delegation](02-subagent-delegation.md), [04-skills](04-skills.md):** OpenHands microagents implement these patterns concretely.

## Key takeaways

1. **Code as action wins** on multi-step tasks; the empirical advantage over JSON tool calls is large and stable.
2. **Stateful Jupyter sandbox** is the right substrate — variables persist, the agent composes.
3. **OpenHands moved the OSS SWE-bench frontier dramatically** from late 2024 to late 2025.
4. **Microagents are how OpenHands ships skill-style specialization** within an OSS framework.
5. **Sandboxing, resource caps, and tracing are non-negotiable** when the agent gets a Python REPL.

## References

- Wang, X. et al. (2024). *Executable Code Actions Elicit Better LLM Agents.* arXiv:2402.01030. https://arxiv.org/abs/2402.01030
- OpenHands repository: https://github.com/All-Hands-AI/OpenHands (formerly OpenDevin).
- SWE-bench leaderboards: https://www.swebench.com/ and https://www.swebench.com/verified.html
- *Sonar Claims Top Spot on SWE-bench leaderboard* (Sonar Foundation Agent reaches 79.2%): https://www.sonarsource.com/company/press-releases/sonar-claims-top-spot-on-swe-bench-leaderboard/
- AutoCodeRover (related index-and-edit baseline).
