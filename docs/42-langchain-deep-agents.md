# 42 — LangChain Deep Agents: Planning, Virtual Filesystem, Async Subagents

**Definition.** LangChain Deep Agents (March–April 2026) is an agent harness built on LangChain and LangGraph that ships a built-in **planning tool**, a **virtual filesystem**, **subagent delegation** via a `task` tool, auto-summarization, and (since v0.5) **asynchronous subagents** that run on separate servers without blocking the parent. It is the clearest open-source distillation of the patterns pioneered in Claude Code and Devin.

## Problem it solves

Building a competent agent used to require assembling many primitives yourself: a plan-write-todos tool, a filesystem abstraction, a subagent dispatch mechanism, context compaction. Each team reinvented subtly-different versions and hit the same gotchas. Deep Agents packages these primitives into one runtime with opinionated defaults, so teams can build sophisticated multi-step agents in hours rather than weeks. The async subagent addition addresses a real production pain: long-running subtasks (2-hour research jobs, multi-service deploys) shouldn't block the supervisor agent's main loop.

## Mechanism

### Core primitives

1. **Planning tool — `write_todos`.** The agent decomposes the task into a structured todo list and maintains it as work progresses. Same pattern as [12-todo-scratchpad-state.md](12-todo-scratchpad-state.md).
2. **Virtual filesystem.** A filesystem abstraction backed by one of several implementations — **StateBackend** (default; ephemeral, scoped to a LangGraph thread), **FilesystemBackend** (real disk), **LocalShellBackend** (shell access), **StoreBackend** (database), **CompositeBackend** (mix). The agent uses familiar tools (`read_file`, `write_file`, `edit_file`, `ls`, `glob`, `grep`) without caring about the backing store.
3. **Subagent spawn — `task` tool.** Delegate work to a specialized subagent with its own system prompt, tool allowlist, and context. Context isolation protects the parent from observation floods.
4. **Shell — `execute`.** Sandboxed shell access when real execution is needed.
5. **Built-in context management.** Auto-summarization, spilling of large outputs to the virtual filesystem so they don't pollute the LLM context.

### Async subagents (v0.5, April 2026)

The supervisor spawns a subagent on a separate server; receives a **task ID** immediately; continues working while the subagent runs independently. When the subagent finishes, the result is retrievable via the task ID. This is the harness-level analogue of promises/futures, and it unlocks long-running parallel work (research, deploy pipelines, batch processing) without wasting supervisor latency.

Inline subagents remain available for shorter tasks where the supervisor genuinely needs the result to proceed.

## Concrete pattern

Minimal Deep Agent with custom subagent types:

```python
from deepagents import create_deep_agent

planner_subagent = {
    "name": "planner",
    "description": "Draft research plans.",
    "prompt": "You produce 5-step research plans, no execution.",
    "tools": ["read_file", "write_file"],
}

agent = create_deep_agent(
    tools=[read_file, write_file, edit_file, execute],
    instructions="You are a senior research engineer.",
    subagents=[planner_subagent],
)

result = agent.invoke({"messages": [{"role": "user", "content": task}]})
```

Async subagent usage (v0.5):

```python
task_id = agent.spawn_async(subagent="researcher",
                            prompt="Survey recent agentic AI papers; write report.md")
# continue doing other work...
report = agent.await_task(task_id)
```

## Variants & related techniques

- **Subagent delegation** ([02-subagent-delegation.md](02-subagent-delegation.md)) — Deep Agents is a specific implementation.
- **Virtual filesystem as context offload** ([08-context-compaction.md](08-context-compaction.md), [09-memory-files.md](09-memory-files.md)) — writing observations to files keeps the context lean.
- **Claude Agent SDK / OpenClaw / Deepwisdom Owl** — peer frameworks with overlapping design points.
- **Deep Agents + SKILL.md** — community patterns adopt the [04-skills.md](04-skills.md) format for invokable capabilities.
- **Multi-session continuity** ([10-multi-session-continuity.md](10-multi-session-continuity.md)) — the filesystem backend lets a session's state persist into the next.

## Failure modes & anti-patterns

- **Default StateBackend surprise.** `StateBackend` is ephemeral per thread; users expect disk. Fix: pick a backend deliberately per deployment; document it.
- **Subagent sprawl.** Defining ten specialized subagents for each task is overhead. Start with one or two; add only when a clean subtask justifies the isolation.
- **Async misuse.** Awaiting the task ID immediately defeats async. Use async only when the supervisor has genuine work to do in parallel.
- **Filesystem as free-form memory.** Writing unbounded artifacts clutters the context that eventually reads them. Put structure on filenames and paths.
- **Over-reliance on auto-summarization.** Automated compaction makes mistakes; pair with explicit memory files for durable facts.
- **Poor boundary between `execute` and tools.** Defaulting to shell instead of typed tools reintroduces the risks harnesses exist to mitigate.

## When to use (and when not to)

**Use** Deep Agents when:

- You're building an agent in the LangChain/LangGraph ecosystem.
- The task is multi-step with potential for parallel subtasks.
- You want the planning + filesystem + subagent + compaction primitives without reinventing them.
- Async subagents unlock workflows (research, batch, long pipelines) you couldn't build before.

**Don't** use it when:

- You're outside the LangChain ecosystem and adoption cost is high.
- The task is a single-shot LLM call or simple Q&A — overhead outweighs benefit.
- You need features not yet in Deep Agents (specific MCP integrations, vendor SDK features) and can't wait.

## References

- LangChain, "Deep Agents" blog post. <https://blog.langchain.com/deep-agents/>
- Deep Agents docs. <https://docs.langchain.com/oss/python/deepagents/overview>
- GitHub langchain-ai/deepagents. <https://github.com/langchain-ai/deepagents>
- Deep Agents v0.5 release notes (async subagents, April 2026). <https://blockchain.news/news/langchain-deep-agents-v05-async-subagents-multimodal>
- A B Vijay Kumar, "Building Deep Agents + SKILL.md with LangChain". <https://abvijaykumar.medium.com/building-deep-agents-skill-md-with-langchain-074176c66dec>
