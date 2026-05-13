# 109 — smolagents: HuggingFace's Code-Writing Agents

**Project.** smolagents — the lightweight agent framework from Hugging Face. Repo: https://github.com/huggingface/smolagents. Documentation: https://huggingface.co/docs/smolagents. Released early 2025; ~14.8k GitHub stars within ~15 months.

**One-line definition.** smolagents is a **minimal, code-first agent library** that defaults to **CodeAgent** — agents emit Python code instead of JSON tool calls — and ships in **a few hundred lines of core**, deliberately rejecting the abstraction layers of LangChain and LangGraph in favor of "an LLM in a `while` loop with a code interpreter."

## Why this paper matters

While [110-langgraph](110-langgraph.md), AutoGen, and CrewAI race to add features, smolagents is the **subtractive** counterpoint: how *little* framework do you actually need? The library is a pedagogical artifact and a production option simultaneously — small enough to read end-to-end in an hour, used in real Hugging Face products. Its rapid star growth and the related HF blog post *"How code agents are unlocking the next level of LLM agents"* shaped how a generation of practitioners think about the *minimum* agent stack.

## Problem it solves

1. **Framework bloat.** Multi-thousand-line agent frameworks make it hard to debug, hard to deploy, and hard to reason about correctness.
2. **JSON tool call ceiling.** smolagents adopts the [108-openhands-codeact](108-openhands-codeact.md) thesis: Python-as-action beats JSON for multi-step tasks. But where OpenHands ships a heavy runtime, smolagents ships a tiny one.
3. **Tooling-as-Python-functions.** Every tool is a `@tool`-decorated Python function with type hints; the agent gets a full Python environment to compose them.
4. **Agnostic to model provider.** smolagents abstracts model calls behind a `Model` interface that supports HF Inference, OpenAI, Anthropic, Ollama, vLLM — making BYO-model trivial.

## Core idea in one paragraph

A `CodeAgent(tools=[...], model=...)` accepts a task, calls the LLM, parses out a Python code block from the response, executes it in a restricted local Python interpreter (or via E2B / Docker for sandboxed remote execution), feeds back the output, and loops until the agent calls `final_answer(...)`. Tools are plain Python functions decorated to expose their docstring and signature to the agent. The whole thing is small enough to fit in a few files; users routinely fork and extend it.

## Mechanism (step by step)

### (a) Defining a tool

```python
from smolagents import tool

@tool
def search_web(query: str) -> str:
    """Search the web for the given query and return the top result."""
    ...
```

The decorator extracts the type hints and docstring and exposes a structured description to the LLM.

### (b) Running the agent

```python
from smolagents import CodeAgent, HfApiModel

agent = CodeAgent(
    tools=[search_web, calculator],
    model=HfApiModel("meta-llama/Llama-3.3-70B-Instruct"),
)
agent.run("What's the current population of the largest city in Japan?")
```

The agent emits something like:

```python
result = search_web("largest city Japan population")
print(result)
final_answer(result)
```

### (c) The restricted Python interpreter

By default, smolagents uses a **whitelisted local interpreter** that allows imports only from a pre-approved list (math, statistics, etc.) and bans `os`, `subprocess`, `sys.modules` manipulation, file writes outside a workspace, and network calls except via authorized tools. This is *not* a hardened sandbox — for untrusted inputs you must use the **E2B** or **Docker** executor instead, which run code in proper isolation.

### (d) The action loop

```text
while step < max_steps and not finished:
    response = model(prompt + history)
    code = extract_code_block(response)
    output = interpreter.run(code)
    history.append((response, output))
    if "final_answer" was called: finished = True
```

The loop is ~20 lines. Reading it once explains the framework.

### (e) Multi-agent

smolagents supports a `ManagedAgent` wrapper that lets one agent invoke another as a tool — the orchestrator-worker pattern ([02-subagent-delegation](02-subagent-delegation.md)) without new abstractions, just nested function calls.

## Empirical results

The HF blog accompanying smolagents reports CodeAgent outperforming a comparable JSON-tool-call agent on **GAIA** and **SimpleQA**-style benchmarks by margins consistent with the original CodeAct paper ([108-openhands-codeact](108-openhands-codeact.md)). The framework deliberately doesn't compete on benchmark SOTA — it competes on **simplicity per capability point**.

GitHub-star trajectory tells the popularity story: from 0 to ~14.8k stars in ~15 months, the steepest curve among 2025 OSS agent frameworks.

## Variants and ablations

- **CodeAgent vs ToolCallingAgent**: smolagents ships both; CodeAgent is the default and recommended for non-trivial tasks.
- **Local Python interpreter vs E2B vs Docker**: only E2B/Docker are safe for untrusted code; the local one is for trusted, dev-time use.
- **Different model backbones**: small open models (Llama 3.x 8B/70B, Qwen2.5-Coder) work surprisingly well via CodeAgent because they only need to emit syntactically valid Python — a capability open models have caught up on.

## Failure modes and limitations

1. **Local interpreter is not a real sandbox.** Treat it as such only for trusted inputs. Misreading this leads to security incidents.
2. **No built-in long-term memory** — bring your own (e.g., Mem0 — [104-mem0-production-memory](104-mem0-production-memory.md)).
3. **Limited orchestration primitives.** No state graph, no built-in retries, no cycle detection. For complex flows, [110-langgraph](110-langgraph.md) is more appropriate.
4. **Code-emission failure with weak models.** Models that don't reliably emit valid Python derail the loop; ToolCallingAgent is safer with weaker models.
5. **Trace volume.** Code-action traces are larger than JSON tool-call traces (full code + outputs); plan accordingly for [24-observability-tracing](24-observability-tracing.md).

## When to use, when not

**Use smolagents when** you want a tight, readable, auditable agent layer; when your team is small and prefers Python primitives over framework abstractions; or when you want to learn the mechanics of agent loops by reading actual production-quality code. Especially compelling as a pedagogical baseline to compare against [110-langgraph](110-langgraph.md) or AutoGen.

**Don't reach for it** when you need built-in observability/state-graph features, when you must orchestrate complex multi-agent workflows with cycle handling, or when your task suite requires the heavyweight sandboxing of OpenHands.

## Implications for harness engineering

- **Less framework, more harness.** smolagents is the existence proof that the agent loop itself is small; *most* of the value lives in the surrounding harness (tools, sandbox, observability) — exactly the [40-harness-engineering-principles](40-harness-engineering-principles.md) thesis.
- **Tool definition can be `@tool`-decorated functions** — clean, typed, no schema files.
- **Local-vs-remote interpreter choice is a security decision** that the framework makes explicit. Don't conflate them.
- **Code-action models are now small enough** — Llama 3.3 70B class models are sufficient — that on-prem, BYO-model agents are within reach.

## Connections to other work in this corpus

- **[108-openhands-codeact](108-openhands-codeact.md):** the same code-as-action thesis at a heavier scale; smolagents is the minimalist instantiation.
- **[110-langgraph](110-langgraph.md):** the explicit state-graph alternative; the design tradeoff is explicit-vs-implicit control flow.
- **[42-langchain-deep-agents](42-langchain-deep-agents.md):** more abstractions, more features, more dependencies — the antipode.
- **[02-subagent-delegation](02-subagent-delegation.md), [04-skills](04-skills.md):** smolagents' `ManagedAgent` wrapper implements these patterns minimally.
- **[40-harness-engineering-principles](40-harness-engineering-principles.md):** smolagents is, in effect, principles 1–3 ("loop, tool, context") in code form.

## Key takeaways

1. **The agent loop is genuinely small** — smolagents ships it in a few files.
2. **CodeAgent is the default**; JSON tool-calling is a fallback.
3. **Model abstraction layer** keeps the framework portable across HF Inference, OpenAI, Anthropic, local Ollama, vLLM.
4. **Use E2B / Docker executors** for untrusted code — never the local interpreter.
5. **Bring your own memory and observability** — smolagents' minimalism is the feature.

## References

- smolagents repository: https://github.com/huggingface/smolagents
- smolagents documentation: https://huggingface.co/docs/smolagents
- HF blog: *Introducing smolagents — How code agents are unlocking the next level of LLM agents.*
- Origin paper for code-as-action: Wang et al. (2024), *Executable Code Actions Elicit Better LLM Agents*, arXiv:2402.01030 — see [108-openhands-codeact](108-openhands-codeact.md).
- Comparative landscape: Langfuse — *Comparing Open-Source AI Agent Frameworks* (2025-03-19).
