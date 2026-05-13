# 17 — ReWOO (Reasoning Without Observation)

**Definition.** ReWOO (Xu et al. 2023) decouples reasoning from observation: the planner emits a complete plan of tool calls with *symbolic placeholders* for tool outputs, a worker executes every tool call (often in parallel), and a solver synthesizes the final answer from the plan and the observed evidence. Unlike ReAct — which re-prompts the LLM after every tool call — ReWOO uses the LLM only twice per task.

## Problem it solves

ReAct's interleaved loop pays a steep price: every tool call is followed by a full LLM re-prompt with the entire growing transcript. A 10-tool task is 10 expensive LLM calls, each longer than the last. Worse, many tool calls are independent — searching for capital-of-France and capital-of-Germany don't need to happen sequentially — but ReAct serializes them.

ReWOO rebuilds the interaction pattern to minimize LLM invocations and exploit parallelism. On benchmarks, the paper reports ReWOO matches or exceeds ReAct accuracy at ~5× fewer tokens on HotpotQA and similar multi-hop reasoning tasks.

## Mechanism

Three modules:

1. **Planner.** One LLM call. Produces a structured plan: numbered steps, each with a tool call and a placeholder variable for the result:

   ```
   Plan:
   #E1 = Search[capital of France]
   #E2 = Search[capital of Germany]
   #E3 = Calculator[population(#E1) + population(#E2)]
   ```

2. **Worker.** Executes tool calls. No LLM involved. Independent calls (#E1, #E2) can run in parallel; dependent calls (#E3 depends on #E1 and #E2) wait on their prerequisites.

3. **Solver.** One final LLM call. Given the plan and all observed evidence, produces the answer.

Total LLM cost: 2 calls. Parallel tool execution cuts wall-clock time to the critical path's length, not the sum of all calls.

## Concrete pattern

A ReWOO-style plan and execution for *"Which is larger, the population of France or the square of Germany's GDP in 2022?"*:

```
Planner → Plan:
  #E1 = WebSearch[population of France 2022]
  #E2 = WebSearch[GDP of Germany 2022 in USD]
  #E3 = Calculator[(#E2) ** 2]
  #E4 = Solver[compare #E1 to #E3]

Worker:
  Execute #E1 and #E2 in parallel.
  #E1 = "67.97 million (2022)"
  #E2 = "4.07 trillion USD (2022)"
  Execute #E3 (depends on #E2).
  #E3 = 1.657e25

Solver → Final answer:
  Germany's GDP squared (~1.66 × 10^25) is vastly larger than France's
  population (6.797 × 10^7).
```

Dependency-aware scheduling is the operational ingredient. Build a DAG from placeholder references; execute each layer in parallel.

## Variants & related techniques

- **ReAct** ([13-react.md](13-react.md)) — interleaved, adaptive, expensive. Preferred when later actions genuinely depend on reasoning over earlier observations.
- **Plan-and-Solve** ([16-plan-and-solve.md](16-plan-and-solve.md)) — plans first, executes in order, usually serial. ReWOO is Plan-and-Solve with placeholders and parallel execution.
- **LLMCompiler** (Kim et al. 2024) extends ReWOO with more explicit DAG compilation and has shown further speedups.
- **Toolformer** trains models to generate tool calls inline; orthogonal to ReWOO's invocation structure.
- **Chain-of-Tools / Function-call batching.** Practical adaptations seen in production (OpenAI parallel tool calls, Anthropic parallel tool_use) that recover most of ReWOO's benefits without the full planner/solver split.

## Failure modes & anti-patterns

- **Brittle plans.** Real observations invalidate a plan; ReWOO's single-pass planner has no chance to adapt. Fix: fall back to ReAct when the planner's confidence is low, or add a "re-plan" branch.
- **Placeholder schema drift.** The planner uses a variable `#E1` but the solver references `#1`; worker binds mismatched values. Fix: strict, linter-checked placeholder grammar.
- **Dependency misinference.** Two calls that *look* independent share hidden state (e.g., rate limits, session tokens). Parallelism breaks. Fix: explicit dependency hints; treat unknowns as serial by default.
- **Under-specified tool calls.** Planner emits `Search[...]` with vague queries, gets useless results, solver flounders. Fix: demand the planner produce a specific, self-contained query per tool call.
- **Complex dependency trees.** If #E7 depends on #E3 which depends on #E1, and a later step is conditional, linear ReWOO collapses. Fix: switch to hybrid — ReWOO for independent lookups, ReAct for conditional reasoning.
- **No error recovery.** A tool fails; worker surfaces an error; solver either hallucinates around it or gives up. Fix: structured error handling in the worker layer, optionally re-planning on failure.

## When to use (and when not to)

**Use** ReWOO when:

- Many independent lookups are obvious up-front (research, multi-hop QA, data aggregation).
- Latency matters — parallel tool execution is a big win.
- Budget matters — two LLM calls total instead of N.
- The task's shape is predictable from the user prompt alone.

**Don't** use ReWOO when:

- Tool outputs reshape the plan (e.g., a search result reveals a new subtask). ReAct or LATS handles that.
- The planner isn't strong enough to produce a complete, correct plan from the prompt.
- The tools have complex side effects or ordering constraints not easily expressed as a DAG.

## References

- Xu et al., "ReWOO: Decoupling Reasoning from Observations for Efficient Augmented Language Models", arXiv:2305.18323 — <https://arxiv.org/abs/2305.18323>
- Kim et al., "An LLM Compiler for Parallel Function Calling", arXiv:2312.04511 — <https://arxiv.org/abs/2312.04511>
- LangGraph ReWOO tutorial — <https://langchain-ai.github.io/langgraph/tutorials/rewoo/rewoo/>
- OpenAI, parallel tool calling docs — <https://platform.openai.com/docs/guides/function-calling>
- Anthropic, parallel tool use — <https://docs.claude.com/en/docs/build-with-claude/tool-use>
