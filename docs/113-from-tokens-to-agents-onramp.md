# 113 — From Tokens to Agents: How an LLM Becomes Agentic, Without the Jargon

**Sources.** Baker, *Agentic AI For Dummies*, Part 1; Stewart & Huang, *Agentic AI Data Architectures*, Chapter 1 (the perceive/reason/act/learn framing); Hodjat, *The Agentic Enterprise*, Chapter 1 (From Automation to Agents); plus the foundational definitions in Russell & Norvig's *AIMA* and the autonomous-agents literature.

**One-line definition.** This is the lay-friendly entry chapter the book never had: it walks the reader from "an LLM is a text completion engine" to "an agent is an LLM in a loop with tools, memory, and goals" without assuming any prior AI background — useful both as the on-ramp for new readers and as the chapter you point a non-technical stakeholder at when they ask "what is agentic AI, really?".

## Why this chapter matters

The rest of this book assumes the reader knows what an agent is, what tool-use means, and why memory matters. That assumption excludes a real audience: managers deciding whether to fund agentic work, lawyers writing policies for it, designers building UX around it, and engineers from neighbouring fields (mobile, frontend, infra) joining an agent team for the first time. This chapter brings them in.

It also serves a second purpose: it forces the rest of the book to be *consistent* about terminology. When chapter 1 of the book is the agent-loop architecture (technical) and the casual reader is dropped straight into "ReAct, Reflexion, ToT", the dropout rate is high. With this chapter as a bridge, the book is teachable to a wider audience without lowering the technical bar elsewhere.

## Problem it solves

Five concrete confusions a casual reader has on first contact with "agentic AI":

1. **"Is this just ChatGPT?"** No — ChatGPT is a chat. An agent is ChatGPT in a loop, with the ability to *do* things, not just say them.
2. **"What's an agent vs an AI?"** AI is the broad field; an agent is one specific use of AI — a system that perceives, reasons, acts, and (sometimes) learns over a multi-step task on behalf of a user.
3. **"Why is everyone talking about tools?"** Because the bottleneck for an LLM acting in the world is not its language ability — it's its access to do things (read files, search the web, run code). Tools are how it gets that access.
4. **"What does memory have to do with anything?"** Because an LLM forgets everything between calls. If you want an assistant that learns your preferences over weeks, the memory is *outside* the LLM, and that outside is what we mean by "agentic memory".
5. **"Why is this hard? It's just GPT in a loop."** Because the loop has to handle errors, hallucinations, infinite loops, ambiguous goals, malicious inputs, cost overruns, and the limits of what the model knows. The loop is most of the work.

Each confusion gets a clean answer below.

## Core idea in one paragraph

An LLM by itself is a text-completion engine: text in, text out, no memory, no tools, no actions. To turn it into an *agent*, you wrap it in four things: (1) a **goal** the user gives the agent ("book a flight to Tokyo under $800"); (2) a **tool set** the agent can use to interact with the world (search the web, query a database, send an email); (3) a **loop** that lets the agent *think → act → observe → think again*, repeating until the goal is met or the budget runs out; (4) a **memory** that survives across loop iterations and (sometimes) across sessions so the agent can learn from prior work. Add these four wrappers around a frozen LLM and you have an agent. Everything else in this book — patterns, frameworks, benchmarks, papers — is engineering on top of those four pieces.

## Mechanism (step by step)

### 1. What an LLM does on its own

A large language model takes a sequence of words (well, tokens — see [110-transformer-llm-architecture-for-agent-builders](110-transformer-llm-architecture-for-agent-builders.md)) and predicts the next word, one at a time. That's it. No memory between calls, no ability to do anything in the world, no goals, no plans. If you ask an LLM "book me a flight to Tokyo," it will *describe* how to book a flight to Tokyo. It will not actually book one.

### 2. The four wrappers that turn an LLM into an agent

**Goal.** You give the agent a target: "summarise these PDFs", "fix this bug", "research and write a memo on X". The goal is the success criterion the agent tries to satisfy.

**Tools.** You give the agent a set of capabilities: read a file, search the web, run code, send an email, query a database, manipulate an image. Each tool has a name, a description (so the model knows when to use it), inputs (what it needs), and outputs (what it returns). The agent doesn't *know* how to do these things — it just knows when to ask for them.

**Loop.** The control structure is roughly:
```
while not done:
    thought = LLM("given goal G, history H, what should I do next?")
    action  = parse_tool_call(thought)
    if action is None:                  # agent decided to answer
        return final_answer(thought)
    observation = execute(action)        # the tool actually runs
    H.append((thought, action, observation))
```

The loop iterates until the agent has reached its goal or exhausted a budget (steps, tokens, seconds, dollars).

**Memory.** Two kinds:
- **Within-task memory** is just the running history H — what the agent has thought, done, and observed so far. It lives in the LLM's prompt across iterations.
- **Across-task memory** is what survives after the task ends — preferences, prior solutions, learned strategies. It lives outside the LLM, in files or databases, and is read back in on future tasks. See [09-memory-files](09-memory-files.md) and [100-contextual-memory-is-a-memo](100-contextual-memory-is-a-memo.md) for the depth.

### 3. The perceive → reason → act → learn cycle

A common framing (from Stewart & Huang, *Agentic AI Data Architectures*; also AIMA) labels the four moves an agent makes per iteration:

- **Perceive.** Read the world: tool results, environment state, user input.
- **Reason.** Think about what to do next given the goal and what you've perceived.
- **Act.** Pick a tool call and execute it; or decide you're done and return an answer.
- **Learn** (optional). Update memory with what worked or didn't.

This is the same loop as #2 above, just relabelled in cognitive terms. Useful for explaining to non-engineers; less useful for building.

### 4. What makes an agent "agentic" — autonomy gradient

There is no single line where "AI" ends and "agentic AI" begins. There is a *gradient* of autonomy:

| Level | Description | Example |
|---|---|---|
| 0 | Single-shot Q&A | ChatGPT answering a question |
| 1 | Single tool call | "Search this and answer" |
| 2 | Short loop with tools | A 5-step research assistant |
| 3 | Long loop with planning | A 50-step coding agent |
| 4 | Multi-agent with delegation | An orchestrator + workers |
| 5 | Persistent learning over weeks | An assistant that improves with use |

Most production "agents" today are level 2 or 3. Level 4 is where multi-agent frameworks ([20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md), [42-langchain-deep-agents](42-langchain-deep-agents.md)) live. Level 5 is the frontier ([55-hermes-agent-self-improving](55-hermes-agent-self-improving.md), [56-sea-landscape-2026](56-sea-landscape-2026.md)).

### 5. Why this is hard

A reasonable software person looks at the loop in #2 and thinks "that's twenty lines of code, what's the fuss?" The fuss is that the loop fails in a thousand ways:

- The model **hallucinates** a tool that doesn't exist.
- The model **mis-parses** the tool's output and acts on the wrong fact.
- The model **loops infinitely**, asking the same thing over and over.
- The model **gives up too early**, returning "I cannot do this" when it could.
- The model **takes a malicious action** because the input was poisoned ([22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [82-poisonedrag](82-poisonedrag.md)).
- The model **forgets** a critical constraint from earlier in the conversation.
- The agent **costs more than expected** because it ran 200 iterations instead of 10.

Almost every chapter in this book is a pattern that solves one of these failure modes.

### 6. The four wrappers map to the four pillars

The four wrappers (goal, tools, loop, memory) map directly to the four pillars of harness engineering ([44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md)):

| Wrapper | Pillar |
|---|---|
| Goal | Context (what does the agent know?) |
| Tools | (the action space) |
| Loop | State + Entropy (controlling iteration) |
| Memory | State (what survives across calls) |

The wrappers are the *user-facing* description; the pillars are the *engineering-facing* description. Both are correct.

## Empirical anchors

- **A working "Hello World" agent is ~50 lines of code.** Tools = a calculator and a search function; goal = a math word problem; loop = 5 iterations.
- **Production agents are 5,000–50,000 lines** because they handle the failure modes in #5 above.
- **Most "agentic" demos in 2024–2025 were level 2.** The level-3+ harnesses (Claude Code, Cursor, Devin) emerged in 2025–2026.
- **The naming is unsettled.** "Agent" vs "agentic AI" vs "AI agent" vs "autonomous agent" all overlap. This book uses "agent" for an instance and "agentic" for the design philosophy.

## Variants and counter-arguments addressed

- **"Isn't this just function calling?"** Function calling is one form of tool use. Agents add the *loop*, the *goal*, and the *memory* on top of function calling.
- **"Aren't agents just chatbots?"** Chatbots respond to messages. Agents pursue goals across multi-step tasks, often without further user input.
- **"Will agents replace developers?"** Some narrow tasks (boilerplate, well-specified bug fixes), already partly. Most multi-step engineering work, no — for reasons covered in this entire book.
- **"Why is autonomy good?"** It's not, automatically. Autonomy is useful when the cost of constant human oversight exceeds the risk of agent error. The trade-off is what permission modes ([06-permission-modes](06-permission-modes.md)) and HITL ([23-human-in-the-loop](23-human-in-the-loop.md)) navigate.

## Failure modes and limitations

1. **Mistaking demo for product.** A 5-iteration agent on a clean task looks magical; the same agent on a dirty task fails embarrassingly. Real agents need engineering for the dirty case.
2. **Treating the LLM as the bottleneck.** Often the bottleneck is the *tools*, the *memory*, or the *loop's failure handling* — not the model. Upgrading the model rarely fixes a broken loop.
3. **Confusing autonomy with intelligence.** A more autonomous agent is not a smarter one; it just acts on its (possibly wrong) beliefs without supervision. Calibrate autonomy to risk.
4. **Underestimating cost.** A 50-iteration agent costs ~50× a single LLM call. Budget accordingly.

## When to use this framing, when not

**This chapter is the right starting point for** new readers, non-technical stakeholders, designers, lawyers, product managers, and engineers from neighbouring fields. It is also the framing to use in slide decks, executive summaries, and onboarding docs.

**It is too coarse for engineers building agents.** Engineers should jump to [01-agent-loop-architecture](01-agent-loop-architecture.md) and the rest of Part I.

## Implications for harness engineering

- **Document the four wrappers explicitly.** Every agent should have a one-page doc naming its goal, tools, loop limits, and memory layer. This is the contract for the agent.
- **Pick an autonomy level deliberately.** Level 2/3 most production work; level 4 only when multi-agent makes sense; level 5 only when the deployment is long-lived enough to amortise the engineering.
- **Keep the four wrappers separable.** Mixing tool definitions into the loop, or memory into the goal, makes the agent un-debuggable. Each wrapper is its own module.
- **Pair this chapter with [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md)** for new team members. User-facing framing first, engineering-facing framing second.
- **Use this chapter for HITL UX design.** When designing the human-facing interface ([143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md)), the four-wrapper mental model is what the user is implicitly building; surface goal/tools/loop/memory in the UI.

The one-sentence takeaway: **an agent is an LLM with a goal, a toolbox, a loop, and a memory — and every chapter in this book is engineering on top of those four pieces.**

## See also

- [01-agent-loop-architecture](01-agent-loop-architecture.md) — the technical version of the loop.
- [04-skills](04-skills.md), [07-model-context-protocol](07-model-context-protocol.md) — what tools really look like in production.
- [09-memory-files](09-memory-files.md), [100-contextual-memory-is-a-memo](100-contextual-memory-is-a-memo.md) — the depth on memory.
- [44-four-pillars-harness-engineering](44-four-pillars-harness-engineering.md) — the engineering-facing description of the same four pieces.
- [148-beginner-onramp-what-is-agentic-ai](148-beginner-onramp-what-is-agentic-ai.md) — even gentler on-ramp for non-technical audiences.
- [143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md) — UX implications of the four-wrapper model.
