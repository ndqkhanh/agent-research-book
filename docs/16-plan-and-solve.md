# 16 — Plan-and-Solve

**Definition.** Plan-and-Solve (Wang et al. 2023) is a two-stage prompting pattern: first produce an explicit plan (a numbered list of subtasks), then execute each subtask in order, using the plan as a persistent scaffold. Unlike ReAct's interleaved thoughts and actions, Plan-and-Solve front-loads the planning phase and keeps execution focused.

## Problem it solves

Chain-of-thought and ReAct make plans implicitly, one step at a time. When the task is complex — multi-step math, long workflows — the implicit plan drifts: the model forgets earlier subtasks, skips steps, or reorders them incoherently. A structured upfront plan gives the model something durable to refer back to, and lets a human or an evaluator review the plan before any compute is spent on execution.

Plan-and-Solve also surfaces misunderstandings cheaply. If the plan is wrong, the user sees it at step 0 instead of step 20.

## Mechanism

Two-phase prompt:

1. **Plan phase.** The prompt instructs the model: *"Let's first understand the problem and devise a plan. Then let's carry out the plan and solve the problem step by step."* The model outputs a numbered plan.
2. **Solve phase.** The model executes plan items in order, either as a continuation of the same prompt or as a second prompt that includes the plan verbatim.

The paper introduces two variants:

- **PS.** Basic version — plan and solve in one pass, with the prompt asking the model to think step by step after planning.
- **PS+.** Adds explicit sub-prompts: "pay attention to calculation", "extract relevant variables", "calculate intermediate results" — stronger on arithmetic and logical reasoning.

At the agent level, Plan-and-Solve generalizes to **Plan-and-Execute**:

- A *planner* LLM generates a plan (often as a list of tool-call intentions, not just reasoning).
- An *executor* loop handles each step, using a smaller/faster model.
- The executor can re-prompt the planner for re-planning mid-execution when observations invalidate the plan.

This split is the reason "agent with a separate planner" is a common LangChain/LlamaIndex pattern — it lets you use a strong, expensive model once (for the plan) and a cheap model per step.

## Concrete pattern

Text-only PS prompt on a math word problem:

```
Q: A train leaves station A at 9am at 60 km/h heading east. Another leaves
B (200 km east of A) at 10am at 40 km/h heading west. When do they meet?

Let's first understand the problem and devise a plan. Then carry out the
plan and solve step by step.

Plan:
1. At 10am, the eastbound train has traveled 60 km; 140 km remain between them.
2. Combined closing speed = 60 + 40 = 100 km/h.
3. Time to close 140 km = 1.4 h = 1h 24m.
4. They meet at 10am + 1h 24m = 11:24am.

Answer: 11:24am.
```

Agentic Plan-and-Execute (pseudo-code):

```python
plan = strong_model.plan(task)          # 1 expensive call
steps = parse_steps(plan)

for step in steps:
    while not step.done:
        result = cheap_model.execute(step, tools)
        if not result.success:
            revised = strong_model.replan(task, plan, steps_done, failure=result)
            steps = parse_steps(revised)
            break
        step.done = True

return aggregate(steps)
```

## Variants & related techniques

- **ReAct** ([13-react.md](13-react.md)) interleaves reasoning and action; Plan-and-Solve front-loads reasoning. ReAct adapts better mid-task; PS is cheaper and more auditable.
- **ReWOO** ([17-rewoo.md](17-rewoo.md)) takes Plan-and-Execute further: the plan includes variables for tool outputs, executed in parallel.
- **Plan Mode (harness)** ([03-plan-mode.md](03-plan-mode.md)) is the operational version: plan phase is permission-restricted, plan is user-reviewed, execution phase is separately permissioned.
- **Tree of Thoughts** ([15-tree-of-thoughts-lats.md](15-tree-of-thoughts-lats.md)) can be used to *generate* multiple candidate plans and pick the best.
- **Hierarchical Task Networks (HTN)** — classical AI planner that Plan-and-Solve loosely echoes.
- **LangChain Plan-and-Execute agents** — off-the-shelf implementation.

## Failure modes & anti-patterns

- **Overambitious plans.** The model writes a 20-step plan it can't actually execute. Fix: bound plan length; require each step to cite a tool or clearly named subtask.
- **Plan-execution mismatch.** Planner says "search for X with tool A"; executor uses tool B with slightly different semantics. Fix: planner outputs tool calls, not prose; executor validates each against the real tool schema.
- **No replanning.** The plan was based on wrong assumptions that observations now contradict, but the executor marches on. Fix: explicit "re-plan" triggers — e.g., every tool failure, every N steps, or when any precondition fails.
- **Replanning thrash.** Every observation triggers a full replan; progress stalls. Fix: replan only on material surprise; otherwise continue.
- **Plan as rubber stamp.** A plan is generated but nobody (user or evaluator) reads it before execution. Fix: gate execution on approval when the cost of misunderstanding is high (see [03-plan-mode.md](03-plan-mode.md)).
- **Planner-executor model mismatch.** Cheap executor can't follow the strong planner's plan. Fix: planner tailors verbosity and vocabulary to the executor.
- **Hidden state between steps.** Step 3 depends on a value produced in step 1, but the plan doesn't make that explicit. Fix: variable-based plan formats (ReWOO).

## When to use (and when not to)

**Use** Plan-and-Solve when:

- The task is multi-step and the steps are mostly predictable from the prompt.
- You want auditability — a human should be able to read the plan.
- You want a cost-efficient split: one strong model call + N cheap executor calls.
- Re-planning is rare (stable environment, deterministic tools).

**Don't** use it when:

- The plan changes after almost every observation — ReAct is simpler.
- The task is short and trivially planned inline.
- You can't afford the single large plan call up-front (plan can be long for complex tasks).

## References

- Wang et al., "Plan-and-Solve Prompting: Improving Zero-Shot Chain-of-Thought Reasoning by Large Language Models", arXiv:2305.04091 — <https://arxiv.org/abs/2305.04091>
- LangChain "Plan-and-Execute" docs — <https://blog.langchain.dev/plan-and-execute-agents/>
- Lilian Weng, "LLM Powered Autonomous Agents" — <https://lilianweng.github.io/posts/2023-06-23-agent/>
- Anthropic Engineering, "Harness design for long-running application development" — <https://www.anthropic.com/engineering/harness-design-long-running-apps>
- LlamaIndex "Agents & Workflows" planning patterns — <https://docs.llamaindex.ai/en/stable/module_guides/workflow/>
