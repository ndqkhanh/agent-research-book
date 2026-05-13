# 66 — Meta-Harness: The Harness-Builder Landscape (April 2026)

**Definition (working).** A *meta-harness* is a framework whose primary user is not the end-agent but the *harness author*: its abstractions are over loops, tools, hooks, memory, subagents, permissions, recovery paths — the primitives catalogued in [40-harness-engineering-principles.md](40-harness-engineering-principles.md), [43-twelve-harness-patterns.md](43-twelve-harness-patterns.md), [44-four-pillars-harness-engineering.md](44-four-pillars-harness-engineering.md), and [46-components-of-coding-agent.md](46-components-of-coding-agent.md). A meta-harness exists to *produce* harnesses; a plain harness exists to *run* one agent. The corpus so far has been bottom-up (single-harness patterns); this file is the top-down view.

The term has now hit arXiv. Lee et al.'s *Meta-Harness: End-to-End Optimization of Model Harnesses* (arXiv:2603.28052) defines the harness as "the code that determines what information to store, retrieve, and present to the model" and treats the outer loop that *searches over* such harnesses as a first-class object. That paper is one concrete incarnation of a meta-harness; the rest of this file places the frameworks the rest of the industry calls meta-harnesses on a shared map.

This file is deliberately *meta* — it does not repeat the vocabulary those six prerequisite files already introduce; it assumes them.

## 1. What is a meta-harness? The distinction that matters

A single-purpose harness wraps one agent: one loop, one tool set, one memory layout, one set of permission gates. Claude Code is a single-purpose harness. Cursor's agent mode is a single-purpose harness. Devin is a single-purpose harness. Each is *exquisitely tuned* to one task shape and the tuning is the product ([40-harness-engineering-principles.md](40-harness-engineering-principles.md)).

A meta-harness is one level up. It is a **framework for expressing harnesses**, with the following signature properties:

1. **The artifact is another agent**, not a completion. A user "runs" the meta-harness and gets an agent system out. LangChain Deep Agents' `create_deep_agent()` is a concrete example ([42-langchain-deep-agents.md](42-langchain-deep-agents.md)); so is Archon's `.archon/workflows/*.yaml`; so is revfactory/harness' "turn your domain description into an agent team" meta-skill.
2. **Its abstractions are harness primitives, not model primitives.** A meta-harness does not expose `temperature` or `top_p`; it exposes hooks ([05-hooks.md](05-hooks.md)), skills ([04-skills.md](04-skills.md)), permissions ([06-permission-modes.md](06-permission-modes.md)), subagent kinds ([02-subagent-delegation.md](02-subagent-delegation.md)), memory backends ([09-memory-files.md](09-memory-files.md)), recovery paths ([41-product-control-plane.md](41-product-control-plane.md)).
3. **It compiles.** Given a harness description, it produces something runnable — a server, a workflow, a supervisor-plus-subagent graph. This makes the harness a *compilable object*, which is why "harness engineering" keeps rhyming with compiler engineering.
4. **Harnesses are first-class values.** You can fork, diff, version, and search-optimize them. The Meta-Harness paper makes this literal by running an outer loop of *harness search* with full execution-trace feedback (arXiv:2603.28052).

A useful negative definition: LangChain-the-library (bare LCEL) is *not* a meta-harness — it is a plumbing toolkit without opinions about loops, memory consolidation, or recovery. LangChain *plus* Deep Agents is a meta-harness, because Deep Agents ships opinionated versions of those primitives. The distinction is less about name than about whether the abstractions are at the harness layer.

## 2. Two axes to classify builders

After surveying the April-2026 landscape, two orthogonal axes explain most of the variation.

### Axis A — Declarative vs Programmatic

- **Declarative:** the harness is a static description — YAML, JSON, a DAG definition, or a natural-language contract — and the framework interprets it. Archon's `.archon/workflows/`, revfactory/harness' domain description, CrewAI's role/goal/backstory YAML, and Lee et al.'s auto-discovered harness programs sit here. Declarative makes harnesses diffable, reviewable, and *generatable* (an agent can propose new harnesses).
- **Programmatic:** the harness is code — a Python graph, a TypeScript workflow, a function tree. LangGraph, LangChain Deep Agents, Mastra, AutoGen, OpenAI Agents SDK sit here. Programmatic gives maximum flexibility and direct debugger access but trades away diffability.

This is not a binary. LangGraph has a declarative graph-definition layer underneath its Python API; CrewAI has Python bindings that emit the YAML-equivalent structure. But the *default user-facing artifact* falls clearly on one side.

### Axis B — Harness-aware vs Harness-agnostic

- **Harness-aware:** the framework has first-class primitives for loops, tools, hooks, memory, permissions, subagents. Deep Agents ships `write_todos`, virtual filesystem, `task` tool, auto-compaction out of the box ([42-langchain-deep-agents.md](42-langchain-deep-agents.md)). DeerFlow ships sandboxes, memory, skills, subagents, and a message gateway as named primitives. OpenClaw, SemaClaw ([54-semaclaw-general-purpose-agent.md](54-semaclaw-general-purpose-agent.md)), and revfactory/harness expose the twelve-pattern vocabulary directly.
- **Harness-agnostic:** the framework is a general execution substrate that happens to run agents — Temporal, Restate, Argo Workflows, Kubernetes Operators. These don't *know* about hooks or compaction; you implement harness primitives on top of them. Harness-agnostic platforms provide durability and scheduling that harness-aware frameworks lack, which is why the hybrid pattern (Temporal + OpenAI Agents SDK, GA March 2026) is hot.

Putting the axes together gives four quadrants:

```
                 Harness-aware                    Harness-agnostic
Declarative  |  Archon, CrewAI,             |  Argo Workflows,
             |  revfactory/harness,         |  GitHub Actions
             |  DeerFlow preset workflows   |  (used for agents)
-------------+------------------------------+------------------------------
Programmatic |  Deep Agents, SemaClaw,      |  Temporal, Restate,
             |  LangGraph, AutoGen, OpenAI  |  Ray, LangChain core
             |  Agents SDK, Mastra, LobeHub |
```

The most interesting frameworks straddle — Archon is declarative on the workflow surface but programmatic in the executor layer it wraps around Claude Code / Codex. This straddling is itself a design pattern (§6, Pattern M-3).

## 3. Landscape map (April 2026)

Twelve frameworks, placed on the plane. One paragraph each. Every placement is anchored to a primary URL.

**LangGraph** (<https://github.com/langchain-ai/langgraph>) — Programmatic + Harness-aware. A graph-of-nodes runtime where nodes are agents or tools and edges are transitions. Ships checkpointing, streaming, time-travel debugging, and LangSmith observability ([24-observability-tracing.md](24-observability-tracing.md)). The most production-mature of the harness-aware programmatic builders — the default when a team has explicit branching logic and compliance requirements.

**LangChain Deep Agents** (<https://github.com/langchain-ai/deepagents>, v0.5 April 2026) — Programmatic + Harness-aware. The most on-nose meta-harness in the ecosystem: `write_todos`, virtual filesystem with swappable backends, `task`-tool subagents, auto-summarization, and (v0.5) async subagents. Operationalizes most of the twelve patterns ([43-twelve-harness-patterns.md](43-twelve-harness-patterns.md)) with opinionated defaults. See [42-langchain-deep-agents.md](42-langchain-deep-agents.md) for the full treatment.

**LangChain (core)** (<https://github.com/langchain-ai/langchain>) — Programmatic + Harness-agnostic. The toolkit layer: LLM adapters, retrievers, output parsers. Not opinionated about loops or memory consolidation, so teams assemble their own harness on top. 47M+ PyPI downloads; the canonical substrate rather than the canonical harness.

**LlamaIndex** (<https://github.com/run-llama/llama_index>) — Programmatic + Harness-agnostic, drifting toward Harness-aware. Originally retrieval-centric ([25-agentic-rag.md](25-agentic-rag.md)); its AgentWorkflow / ReAct / function-calling agents now expose loop and memory abstractions. RAG primitives are its differentiator — it is effectively a specialized meta-harness for retrieval-heavy agents.

**AutoGen** (<https://github.com/microsoft/autogen>) — Programmatic + Harness-aware, conversational slant. Agents are modeled as participants in a structured conversation; coordination is via messages between named roles rather than graph edges. Strong fit for prototyping and human-in-the-loop research ([23-human-in-the-loop.md](23-human-in-the-loop.md)). Less opinionated about context architecture than Deep Agents — its harness emerges from the conversation, which is both a strength and a debuggability tax.

**CrewAI** (<https://github.com/crewAIInc/crewAI>) — Declarative + Harness-aware, with Python escape hatches. Agents are YAML-ish objects with `role`, `goal`, `backstory`, `tools`; crews orchestrate them. Lowest learning curve; popular for non-engineers. Relatively thin on the deterministic-guardrails pillar ([44-four-pillars-harness-engineering.md](44-four-pillars-harness-engineering.md)) — role descriptions are still prompts, not code-enforced invariants.

**Archon (coleam00)** (<https://github.com/coleam00/Archon>) — Declarative + Harness-aware. Explicitly self-describes as "the first open-source harness builder for AI coding." Workflows are YAML DAGs with sequential nodes, loop nodes, deterministic nodes (bash/tests/git), and human-approval gates. Adapters to Claude Code, Codex, Pi as executors. The clearest current answer to "how would you write a Dockerfile for an agent harness?" and the cleanest declarative surface in the map.

**DeerFlow 2.0 (ByteDance)** (<https://github.com/bytedance/deer-flow>) — Straddles Declarative/Programmatic + Harness-aware. A "long-horizon SuperAgent harness" with sandboxes, memory, skills, subagents, and a message gateway as first-class primitives. 2.0 (early 2026) generalized beyond deep research into research / code / slide / web generation. The most primitive-rich open-source harness today, at the cost of heavier runtime surface than Deep Agents.

**SemaClaw** (arXiv:2604.11548, <https://arxiv.org/abs/2604.11548>) — Programmatic + Harness-aware; academic formalization. Treats harness engineering as a named paradigm shift and formalizes the components a general-purpose personal agent needs. See [54-semaclaw-general-purpose-agent.md](54-semaclaw-general-purpose-agent.md). Less of a shipping product and more of a reference-implementation-plus-paper — important for naming what the others are converging on.

**RAGFlow** (<https://github.com/infiniflow/ragflow>) — Programmatic + Harness-aware, retrieval domain. A RAG engine that fused with agent capabilities; defines a "superior context layer for LLMs" with agent loops on top. Available as an OpenClaw skill (March 2026), so it functions as a harness *component* as often as a meta-harness in its own right.

**LobeHub** (<https://github.com/lobehub/lobehub>) — Declarative + Harness-aware, multi-agent-collaboration slant. Self-describes as "taking agent harness to the next level — enabling multi-agent collaboration, effortless agent team design, and introducing agents as the unit of work interaction." Emphasis on user-visible agent teams (close to the Visibility pillar of [41-product-control-plane.md](41-product-control-plane.md)), which is under-served elsewhere.

**OpenClaw** (<https://github.com/openclaw/openclaw>, see [52-dive-into-open-claw.md](52-dive-into-open-claw.md)) — Programmatic + Harness-aware. Open-source Claude-Code-style harness; ships the twelve-pattern primitives as first-class constructs. Functions as both a ready-to-run harness *and* a meta-harness when its skill/permission/hook layers are configured fresh per project.

**OpenAI Agents SDK** (<https://github.com/openai/openai-agents-python>) — Programmatic + Harness-aware. The core abstraction is the *handoff* — agents explicitly transfer control. Provider-agnostic (compatible with 100+ LLMs); 10M+ monthly downloads. Thinner on memory consolidation and permission semantics than Deep Agents; strong on tracing and guardrails. March 2026 GA of the Temporal integration lets you run SDK agents on durable execution.

**Mastra** (<https://github.com/mastra-ai/mastra>) — Programmatic + Harness-aware, TypeScript-native. From the ex-Gatsby team; ships model routing across 40+ providers, workflows, RAG, memory. The dominant TS-side meta-harness, filling the niche Deep Agents fills on the Python side.

**Temporal / Restate** (<https://temporal.io/solutions/ai>, <https://restate.dev>) — Programmatic + Harness-agnostic. Durable-execution engines. Not agent frameworks per se, but AI workloads increasingly land on them for crash-safe long-running coordination. Temporal's OpenAI Agents SDK integration (GA March 2026) exemplifies the pattern: harness-aware framework *inside*, harness-agnostic durability *around*.

**revfactory/harness** (<https://github.com/revfactory/harness>) — Declarative + Harness-aware, Claude-Code-plugin-shaped. A meta-skill that takes a domain description and generates an agent team plus their skills, choosing from six architectural patterns (Pipeline, Fan-out/Fan-in, Expert Pool, Producer-Reviewer, Supervisor, Hierarchical Delegation). Includes an evolution mechanism that folds runtime deltas back into the factory — an early-but-real instance of self-modification ([36-autogenesis-self-evolving-agents.md](36-autogenesis-self-evolving-agents.md), [45-hyperagents-self-modification.md](45-hyperagents-self-modification.md), [55-hermes-agent-self-improving.md](55-hermes-agent-self-improving.md)).

**Meta-Harness (Lee et al.)** (arXiv:2603.28052) — Declarative + Harness-aware, research prototype. The outer loop: propose a harness, score it on held-out tasks, learn from the full execution trace, iterate. Shows 7.7-point improvement with 4× fewer tokens on online text classification. The clearest demonstration that harnesses are searchable objects — not just designed ones.

## 4. What unifies a good meta-harness — seven traits

Across the above, the meta-harnesses that engineers actually ship on share seven traits. These are *landscape interpretation* anchored in the existing principles files.

1. **Loop-as-software, not prompt-as-loop.** The loop has invariants, tests, and review surface. Deep Agents, LangGraph, and OpenAI Agents SDK all expose loops as structured objects. ([40-harness-engineering-principles.md](40-harness-engineering-principles.md) principle 1.)
2. **Explicit memory backend choice.** Not "we'll serialize to Redis if needed" but a swappable, named backend — Deep Agents' StateBackend/FilesystemBackend/StoreBackend/CompositeBackend is the exemplar. ([09-memory-files.md](09-memory-files.md), [44-four-pillars-harness-engineering.md](44-four-pillars-harness-engineering.md) pillar 1.)
3. **Permission semantics, not permission RBAC.** Operation × data × condition — Adaline's framing ([41-product-control-plane.md](41-product-control-plane.md)). Archon's approval gates and OpenAI Agents SDK's guardrails both move in this direction.
4. **Subagent kinds as declared types.** Subagents are named, typed, and have allowlisted tools — not anonymous recursive calls. Deep Agents' subagent spec, revfactory/harness' six patterns, DeerFlow's subagent registry. ([02-subagent-delegation.md](02-subagent-delegation.md).)
5. **Handoff contracts as artifacts.** A handoff is a first-class thing with a transferred-context schema and a failure-attribution rule, not an implicit edge. OpenAI Agents SDK explicitly promotes handoffs to its core abstraction; Archon's phase-to-phase transitions similarly. ([41-product-control-plane.md](41-product-control-plane.md) primitive 2.)
6. **Determinism wherever possible.** Hooks, validators, formatters run as code at lifecycle points, not via prompt. ([05-hooks.md](05-hooks.md), [44-four-pillars-harness-engineering.md](44-four-pillars-harness-engineering.md) pillar 3.) Archon is most aggressive here; LangGraph's conditional edges also count.
7. **Harnesses are versioned and diffable.** YAML (Archon), graph definitions (LangGraph), domain descriptions (revfactory/harness), full source trees (Meta-Harness paper). Anything that can't be diffed can't be reviewed or optimized.

A meta-harness missing trait 1 is a prompt library. Missing 2 is a toy. Missing 3–4 is a demo. Missing 5 crashes in production. Missing 6 regresses on every model swap. Missing 7 cannot compound learning.

## 5. What is missing in April 2026 — gap catalogue

These are the opportunities. Each is labeled *landscape interpretation* because the absence is inferred from the survey, not quoted from any one source.

### G1. A harness IL (intermediate language)

*Landscape interpretation.* Every meta-harness uses its own surface — YAML dialects, Python graphs, TypeScript workflows, domain-description prose. There is no *intermediate representation* for harnesses. The closest analogue is what LLVM IR is to compilers — a shared, machine-readable form that multiple front-ends lower to and multiple back-ends execute. Its absence means: (a) Archon workflows don't transpile to Deep Agents graphs, even though they express similar things; (b) the Meta-Harness paper's search runs over source code rather than over a canonical IR, losing optimization affordances; (c) cross-framework evaluation is apples-to-oranges. An open harness IR would let frameworks compete on front-end ergonomics and back-end execution while sharing the middle.

### G2. Harness-aware evaluators

*Landscape interpretation.* Every mature framework ships tracing ([24-observability-tracing.md](24-observability-tracing.md)); almost none ship harness-level evals. LangSmith, AgentOps, Pi can grade trajectories — but "did this harness fail because of the loop or the memory or the permissions?" is not a question today's evaluators answer directly. ClawBench ([34-clawbench-live-web-tasks.md](34-clawbench-live-web-tasks.md)) and Claw-Eval ([38-claw-eval.md](38-claw-eval.md)) grade agents; Meta-Harness grades harnesses end-to-end but only via task scores. A meta-harness evaluator would attribute failure to a *specific primitive* — "the compaction step dropped the relevant fact" — and surface it as a diff. Without this, harness improvement is guesswork. This is arguably the highest-leverage gap because improvement requires attribution.

### G3. A chaos-engineering layer for agents

*Landscape interpretation.* [53-chaos-engineering-next-era.md](53-chaos-engineering-next-era.md) argues for injecting faults into agent runs; [49-agents-of-chaos-red-teaming.md](49-agents-of-chaos-red-teaming.md) and [26-linuxarena-production-agent-safety.md](26-linuxarena-production-agent-safety.md) motivate why. No meta-harness in the survey ships chaos-mode out of the box. Temporal's durability means crashes can be replayed, but the framework does not *inject* crashes, tool-latency spikes, or partial tool failures to stress-test the harness. A "gremlins mode" that faults 5% of tool calls, delays subagent returns, or drops memory backends would expose recovery-path gaps in the Adaline sense ([41-product-control-plane.md](41-product-control-plane.md)).

### G4. First-class prompt-cache topology

*Landscape interpretation.* [46-components-of-coding-agent.md](46-components-of-coding-agent.md) identifies cache reuse as a 50–90% cost lever. Frameworks treat cache as an implementation detail — none expose *cache topology* as a declared harness property (stable prefix schema, cache breakpoints, invalidation events). This means users who want Raschka's benefit hand-engineer it per harness. A declared cache topology, validated at build time, would be cheap and universal.

### G5. A recovery-path DSL

*Landscape interpretation.* Adaline's fourth primitive (Recovery) is named in every article and implemented by ~no framework. Retries, fallback prompts, workflow degradation, HITL escalation are all ad-hoc. The meta-harness that ships `recovery:` as a grammar (triggers, transitions, named degradations) will own production. Close starts: Archon's approval gates are a special case; Temporal's workflow retries are a substrate. Neither is a DSL at the harness layer.

### G6. Entropy-management as a platform service

*Landscape interpretation.* Pillar 4 of [44-four-pillars-harness-engineering.md](44-four-pillars-harness-engineering.md) calls for automated cleanup of memory files and code drift. The Dream Consolidation pattern ([43-twelve-harness-patterns.md](43-twelve-harness-patterns.md) pattern 4) names the technique. No meta-harness operationalizes it as a platform service the author simply enables. This gap shows up as silent 90-day performance degradation in long-lived agents.

### G7. Cross-session safety posture

*Landscape interpretation.* Permissions ([06-permission-modes.md](06-permission-modes.md)), hooks ([05-hooks.md](05-hooks.md)), and HITL ([23-human-in-the-loop.md](23-human-in-the-loop.md)) are expressed per-session. A user running the same agent daily has no coherent way to say "loosen permissions after 10 clean runs, tighten again after any policy violation." Trust adapts; frameworks don't. A temporal safety posture — RL-lite for permissions — is absent.

Synthesis: **G1 (harness IR), G2 (harness-aware evaluators), and G3 (agent chaos)** are the three with the largest impact-per-unit-work and the cleanest founding-team problem statements. The others are meaningful but narrower.

## 6. Design-pattern catalogue for *meta*-harnesses

These are patterns that appear *at the meta layer* — design decisions of the framework itself, distinct from the twelve harness-level patterns in [43-twelve-harness-patterns.md](43-twelve-harness-patterns.md). Each named, so the community can argue about them.

### M-1. Pattern-library-as-generator

Bundle a small library of architectural patterns (Pipeline, Fan-out/Fan-in, Expert Pool, Producer-Reviewer, Supervisor, Hierarchical Delegation — revfactory/harness' six) and let the framework *choose* from them at build time based on the domain description. The meta-harness is then an interpreter over named patterns, not a blank canvas. Trade-off: fewer footguns, less expressivity.

### M-2. Harness-as-code-artifact

Emit the harness as source the user can read, not as a runtime-only object. Archon's YAML files, Mastra's TypeScript, Deep Agents' Python are all artifacts. Contrast with frameworks where the harness exists only as the running process — those can't be diffed. Closely related to trait 7 above.

### M-3. Bring-your-own-executor straddle

Separate the *harness description* from the *execution substrate*. Archon describes workflows; executes via Claude Code / Codex / Pi. OpenAI Agents SDK + Temporal executes SDK agents on Temporal durability. This pattern future-proofs the meta-harness — executors come and go; the description survives. It is also the only viable path when the model vendor and the durability vendor differ.

### M-4. Harness-search outer loop

Treat the harness as a searchable object. Lee et al.'s Meta-Harness is the pure version: propose, evaluate, learn. Partial versions: revfactory/harness' evolution mechanism that folds runtime deltas into the factory; Hermes ([55-hermes-agent-self-improving.md](55-hermes-agent-self-improving.md)) for the per-agent case. The outer loop is what turns harness engineering from craft into capability.

### M-5. Primitive-kit-plus-opinionated-defaults

Ship a full set of primitives (twelve patterns worth) *and* a set of defaults so trivial cases trivially work. Deep Agents is the exemplar: the minimal `create_deep_agent(tools, instructions)` call silently gives you todos + VFS + subagents + compaction; advanced users configure. The alternative (LangGraph-like: primitives but no defaults) is more flexible but has a steeper cliff.

### M-6. Skill-marketplace coupling

A meta-harness that integrates with an external skill registry (OpenClaw's skills marketplace, LobeHub's agent teammates) scales its capability surface faster than one that doesn't. The meta-harness stops being the source of all functionality and becomes the *assembler* of functionality third parties ship. Risk: trust and permission semantics (G7) become harder.

### M-7. Declarative-handoff graph

Make handoffs — not just nodes — first-class. OpenAI Agents SDK's promotion of the handoff to core abstraction is a clean example; Archon's phase transitions are a special case. A declarative handoff graph admits static analysis: "is any handoff's transferred-context schema inconsistent with the receiver's expected context?" is a compile-time question if the graph is declarative.

### M-8. Trace-as-feedback

Feed full execution traces (not scores, not summaries) back to whatever layer is improving the harness — a human reviewer, an auto-evaluator, or (Meta-Harness) a proposer agent. Lee et al.'s central result is that richer feedback beats compressed feedback for harness search. Any meta-harness whose improvement loop is score-only is leaving signal on the floor.

## 7. Cross-links and synthesis

The corpus built bottom-up — primitives ([01](01-agent-loop-architecture.md)–[12](12-todo-scratchpad-state.md)), techniques ([13](13-react.md)–[25](25-agentic-rag.md)), case studies ([26](26-linuxarena-production-agent-safety.md)–[39](39-ai-and-mathematics-structure.md)), syntheses ([40](40-harness-engineering-principles.md)–[46](46-components-of-coding-agent.md), [52](52-dive-into-open-claw.md), [54](54-semaclaw-general-purpose-agent.md), [55](55-hermes-agent-self-improving.md)). This file is the view from the opposite end of the telescope: how the *frameworks that embody those primitives* stack up against each other.

Three observations fall out of that view that no single-framework file could expose:

- **Convergence on harness-aware + programmatic is nearly complete** (quadrant bottom-left in §2). Seven of the fourteen surveyed frameworks live there. The interesting divergence is on the *other* axis: declarative frameworks are newer, fewer, and growing.
- **The academic literature is a leading indicator.** Meta-Harness (arXiv:2603.28052), SemaClaw (arXiv:2604.11548), Natural-Language Agent Harnesses (arXiv:2603.25723), and the Claude Code formalization ([29-dive-into-claude-code.md](29-dive-into-claude-code.md)) all landed inside a six-month window. The industry-framework wave (Archon, Deep Agents v0.5, DeerFlow 2.0) is the downstream reflection.
- **The biggest open gaps are shared across all frameworks** — no one ships a harness IR, harness-aware evaluators, or agent chaos-mode. That is an opportunity profile for a new entrant, not an established-player strength to copy.

The next synthesis file in the corpus could usefully pick one of G1/G2/G3 and go deep. That is the pointer this file leaves behind.

## References

- Lee, Y., Nair, R., Zhang, Q., Lee, K., Khattab, O., & Finn, C. "Meta-Harness: End-to-End Optimization of Model Harnesses." arXiv:2603.28052. <https://arxiv.org/abs/2603.28052>
- Natural-Language Agent Harnesses. arXiv:2603.25723. <https://arxiv.org/abs/2603.25723>
- SemaClaw. arXiv:2604.11548. <https://arxiv.org/abs/2604.11548>
- Building AI Coding Agents for the Terminal. arXiv:2603.05344. <https://arxiv.org/abs/2603.05344>
- Agent Harness for Large Language Model Agents: A Survey. Preprints.org 202604.0428. <https://www.preprints.org/manuscript/202604.0428/v1>
- Archon (coleam00). <https://github.com/coleam00/Archon>
- DeerFlow (ByteDance). <https://github.com/bytedance/deer-flow>
- revfactory/harness. <https://github.com/revfactory/harness>
- LangChain Deep Agents. <https://github.com/langchain-ai/deepagents>
- LangGraph. <https://github.com/langchain-ai/langgraph>
- AutoGen (Microsoft). <https://github.com/microsoft/autogen>
- CrewAI. <https://github.com/crewAIInc/crewAI>
- OpenAI Agents SDK. <https://github.com/openai/openai-agents-python>
- Mastra. <https://github.com/mastra-ai/mastra>
- LlamaIndex. <https://github.com/run-llama/llama_index>
- RAGFlow. <https://github.com/infiniflow/ragflow>
- LobeHub. <https://github.com/lobehub/lobehub>
- Temporal for AI. <https://temporal.io/solutions/ai>
- Temporal + OpenAI Agents SDK GA (March 2026). <https://temporal.io/blog/announcing-openai-agents-sdk-integration>
- Restate. <https://restate.dev>
- Awesome Harness Engineering (community list). <https://github.com/ai-boost/awesome-harness-engineering>
- BDTechTalks, "The art of AI harness engineering." <https://bdtechtalks.com/2026/04/06/ai-harness-engineering-claude-code-leak/>
- Generative Programmer, "12 Agentic Harness Patterns." <https://generativeprogrammer.com/p/12-agentic-harness-patterns-from>
- Morph LLM, "Agent Engineering 2026." <https://www.morphllm.com/agent-engineering>
- StackOne, "120+ Agentic AI Tools Mapped Across 11 Categories [2026]." <https://www.stackone.com/blog/ai-agent-tools-landscape-2026/>
- Cobus Greyling, "The Rise of AI Harness Engineering." <https://cobusgreyling.medium.com/the-rise-of-ai-harness-engineering-5f5220de393e>
