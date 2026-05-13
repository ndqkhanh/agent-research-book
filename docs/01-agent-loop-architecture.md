# 01 — Agent Loop Architecture

**Definition.** The agent loop is the core control structure of an LLM agent: the model generates an action, a harness executes it, the result is folded back into context, and the model decides whether to act again or stop. Everything else in this folder (subagents, hooks, permissions, memory) is scaffolding around this loop.

## Problem it solves

A bare LLM is a function from tokens to tokens. To make it *do things* — edit files, call APIs, run tests — you need (a) a way for the model to express intent as a structured call, (b) an executor that runs that call safely, and (c) a feedback channel that returns the result to the model. Without a loop, you have a chatbot. With a naive loop, you get infinite loops, runaway tool calls, and context explosions. The agent loop architecture is the set of design choices that make the loop terminate, stay on-task, and remain debuggable.

## Mechanism

A typical iteration of the loop:

1. **Assemble context.** System prompt + tool schemas + memory + prior turns + last observation.
2. **Generate.** The model emits either a final answer (stop) or one-or-more tool calls (continue).
3. **Authorize.** Permission system decides whether this tool call can run unattended, needs user approval, or must be refused. (See [06-permission-modes.md](06-permission-modes.md).)
4. **Execute.** Run the tool; capture `stdout`, `stderr`, exit code, and metadata.
5. **Reduce observation.** Truncate or summarize the tool result before feeding it back — raw outputs routinely exceed the context window.
6. **Check termination conditions.** Step budget, wall-clock budget, token budget, stop-keyword, or model's own "final answer" signal.
7. **Update state.** Append to transcript, update memory, update scratchpad / todo list.
8. **Loop or return.**

The non-trivial parts are 3, 5, and 6. Authorization is where most safety work happens. Observation reduction is where most context-engineering work happens. Termination is where most reliability work happens.

## Concrete pattern

Skeleton in Python-like pseudo-code:

```python
def agent_loop(task, tools, max_steps=50, max_tokens=200_000):
    transcript = [system_prompt(), user_msg(task)]
    for step in range(max_steps):
        if token_count(transcript) > max_tokens * 0.9:
            transcript = compact(transcript)        # see 08-context-compaction.md

        resp = model.generate(transcript, tools=tool_schemas(tools))
        transcript.append(resp)

        if resp.stop_reason == "end_turn":
            return resp.text

        for call in resp.tool_calls:
            decision = permission.check(call)       # see 06-permission-modes.md
            if decision == "deny":
                transcript.append(tool_result(call, "denied by policy"))
                continue
            if decision == "ask":
                if not user_approves(call):
                    transcript.append(tool_result(call, "user denied"))
                    continue
            result = tools[call.name].run(**call.args)
            result = reduce_observation(result)     # truncate/summarize
            transcript.append(tool_result(call, result))
    return "step budget exhausted"
```

Three knobs deserve highlighting:

- **`max_steps`** — prevents runaway loops. Anthropic's and OpenAI's harnesses both use explicit step budgets; Claude Code uses them per-subagent as well.
- **`reduce_observation`** — critical. Raw `grep -r` or `npm test` output can easily exceed 100k tokens. The harness must summarize before re-entering the loop.
- **`compact`** — when the whole transcript approaches the context window, summarize older turns and continue. See [08-context-compaction.md](08-context-compaction.md).

## Variants & related techniques

- **ReAct** ([13-react.md](13-react.md)) is the prompt-level convention most loops still use: the model emits a `Thought: …` before each `Action: …`.
- **Plan-and-Execute** ([16-plan-and-solve.md](16-plan-and-solve.md)) runs a planner once, then a smaller "executor" loop — fewer tokens per step.
- **ReWOO** ([17-rewoo.md](17-rewoo.md)) pre-commits to a plan of tool calls, executes in parallel, then reasons once over all observations — drastically lower latency for well-structured tasks.
- **Orchestrator-worker** ([02-subagent-delegation.md](02-subagent-delegation.md)) nests loops: the outer loop decides, the inner loops do.
- **Turn-taking harnesses** like Claude Code and Cursor keep the loop interactive — every few steps the user can inject instructions. Fully autonomous harnesses (Devin, Claude Code SDK long-running mode) minimize interrupts in favor of resumable state.

## Failure modes & anti-patterns

- **Infinite loops.** The model proposes a tool call, the tool fails, the model re-tries the same call verbatim. Mitigation: step budgets; repeat-detection that injects "you just tried this, pick a different approach"; let tool errors propagate verbally.
- **Context explosion.** Observations consume the window and the model starts losing earlier instructions. Mitigation: aggressive `reduce_observation`, paginated tool outputs, file-based handoff.
- **Silent drift.** The model forgets the original task after 30+ steps. Mitigation: a persistent "task card" (todo / scratchpad — [12-todo-scratchpad-state.md](12-todo-scratchpad-state.md)) re-injected each turn.
- **Premature termination.** The model says "done" when the test still fails. Mitigation: a verifier step before returning (see [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md)).
- **Tool-use spam.** The model tries eight search queries to find one thing. Mitigation: log observed patterns and nudge in the system prompt ("batch independent searches"), or move to ReWOO for planned fetches.
- **One-giant-turn output.** The model dumps a 10k-line diff in one step with no verification. Mitigation: require tool use for file edits (not raw code dumps) so each edit is an auditable step.

## When to use (and when not to)

Use an agent loop when the task requires (a) more than one tool call, (b) the next action depends on the previous observation, and (c) you can bound success with either a verifier or a step budget. A single-shot LLM call with structured output is almost always cheaper and more reliable for tasks that *don't* meet those criteria — don't turn a classifier into an agent.

## References

- Anthropic Engineering, "Building agents with the Claude Agent SDK" — <https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk>
- Anthropic Engineering, "Effective context engineering for AI agents" — <https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents>
- Lilian Weng, "LLM Powered Autonomous Agents" — <https://lilianweng.github.io/posts/2023-06-23-agent/>
- HumanLayer, "Skill Issue: Harness Engineering for Coding Agents" — <https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents>
- Parallel.ai, "What is an agent harness in the context of large-language models?" — <https://parallel.ai/articles/what-is-an-agent-harness>
