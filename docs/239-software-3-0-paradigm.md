# 239 — Software 3.0 as a Paradigm: prompts as code, agents as the installer

**Anchors.** Andrej Karpathy — *Software 2.0* — Medium, 2017. Andrej Karpathy — Y Combinator AI Startup School talk, 2024 (first public airing of the 1.0/2.0/3.0 trichotomy). Andrej Karpathy — IICYMI talk, May 2026, chapters *Software 3.0 Explained* (02:28) and *Agents as the Installer* (03:44) — `https://www.youtube.com/watch?v=96jN2OCOfLs`. Anthropic — *Model Context Protocol* spec, 2024. OpenAI — *Function Calling* and *Agents SDK*, 2024–2026. *llm.c* (Karpathy) — pedagogical 2.0 reference. **High-star repos surveyed:** Significant-Gravitas / **AutoGPT** (ca. 170 K stars), All-Hands-AI / **OpenHands** (formerly OpenDevin, ca. 35 K), **gpt-engineer** (ca. 53 K), KillianLucas / **open-interpreter** (ca. 55 K), paul-gauthier / **aider** (ca. 25 K), **continuedev/continue** (ca. 18 K), **cline/cline** (ca. 30 K), langchain-ai / **langchain** (ca. 95 K) and **langgraph** (ca. 8 K), **microsoft/autogen** (ca. 35 K), **crewAIInc/crewAI** (ca. 28 K), **stanfordnlp/dspy** (ca. 18 K), Pythagora-io / **gpt-pilot** (ca. 30 K), **bolt.new** (ca. 14 K), **stackblitz/bolt.new**, **OpenWebUI** (ca. 80 K), **ggerganov/llama.cpp** (ca. 70 K), **ollama/ollama** (ca. 100 K), princeton-nlp / **SWE-agent** (ca. 14 K), Anthropic — **claude-code** (closed-source CLI; SDK partly open). Star counts as of early 2026; orders of magnitude.

**One-line definition.** Software 3.0 is the programming paradigm in which **the primary unit of source is a natural-language prompt + skill bundle + tool registry + memory**, the primary unit of build is a **harness** that compiles those into agent behavior, and the primary unit of deployment is an **agent acting as the installer** of the resulting software onto real systems — a third rung above Software 1.0 (hand-written code) and Software 2.0 (learned weights), composing recursively with both.

## Why this paradigm matters (3.0 is what 90% of new product code now actually is)

Most "AI features" shipped in 2025–2026 are not 1.0 (the team did not write the algorithm) and not pure 2.0 (the team did not train the weights). They are **3.0 artifacts**: a system prompt, a tool schema, a few skill files, a retrieval pipeline, an eval suite, and an orchestration loop wrapped around a frozen base model. The 3.0 layer is where the differentiation lives — the same Claude/GPT-5/Gemini base model produces wildly different products as a function of the harness around it ([40](40-harness-engineering-principles.md), [62](62-everything-claude-code.md), [144](144-build-your-own-harness.md), [145](145-comparing-coding-harnesses.md)). Calling this rung "prompt engineering" understates it; calling it "AI engineering" muddles it with 2.0; calling it "Software 3.0" gives it a name commensurate with the role it actually plays in the stack.

The rung also matters because **3.0 artifacts have different lifecycle, review, and version-control properties than 1.0 or 2.0**. A prompt is not source code (it is interpreted by a stochastic, version-drifting backend) and it is not a model weight (it is text humans read and edit). The intermediate status forces new tooling: prompt diff viewers, eval-on-prompt-change CI, semantic version pinning of base models, regression suites that catch behavioral drift when only the harness changed. Treating 3.0 as a real rung rather than as "config" or "a string" is the precondition for engineering it properly.

Third, the rung composes recursively with the lower two. **3.0 in 1.0:** DSPy compiles a 3.0 declarative program into deterministic 1.0 control flow with optimized prompts ([93](93-dspy.md)); constrained decoding compiles a 3.0 schema into 1.0 logits-mask code ([112](112-constrained-decoding.md)). **3.0 in 2.0:** RL on 3.0 trajectories updates 2.0 weights (RLVR, [232](232-rl-on-reasoning-traces-scaling.md)); skill libraries are distilled into adapter weights ([116](116-adapter-tuning-lora-peft-for-agents.md)) or into base-model behavior ([156](156-heavyskill-parallel-reasoning-deliberation.md)). The rung is not a sealed layer; it is a *projection* of the same engineering work into a different surface.

Take the paradigm seriously and three things change. **First**, you stop calling 3.0 work "prompt engineering" and start calling it engineering — with code review, CI, regression evals, on-call rotations for prompt-driven outages, and explicit owners. **Second**, you adopt **agents-as-installer** as a deployment primitive — every 3.0 artifact ships with an installer-agent that brings up the runtime, registers the skills, configures the tools, and runs the smoke evals. **Third**, you pick the high-star ecosystem repo (or fork) that closest matches your harness shape and *commit to its conventions* rather than reinventing them — the cost of homegrown harness deviation has surpassed the cost of adopting a community convention.

## Problem it solves (a clean rung structure for what was a vocabulary mush)

1. **"Prompt engineering" undersells the surface area.** Prompts are 5–10% of a 3.0 artifact; the rest is tool schemas, skills, memory, evals, retrieval. A new word is needed.
2. **"AI engineering" confuses 3.0 with 2.0.** AI engineering as practiced often means *fine-tuning + serving*, which is 2.0; 3.0 is harness + skills + prompts on top of a frozen base model. Different skills, different lifecycles.
3. **"LLMOps" too narrow.** LLMOps is the *ops* layer of 3.0, not the rung itself; the rung also includes design, taste, and product affordances.
4. **No clean place to put **agents-as-installer** as a primitive.** Agents that bring up systems, configure them, and self-test ([164](164-crewai-multi-agent-framework.md), [165](165-ralph-autonomous-loop.md), [102](102-clawgym-scalable-claw-agents.md)) are the natural deployment mechanism for 3.0 artifacts; the rung makes that explicit.
5. **No accepted vocabulary for *menus over raw prompts*.** The Software 3.0 frame names the menu/raw-prompt distinction as a first-class design choice ([04](04-skills.md), [205](205-lobehub-collaborative-teammate-platform.md)) rather than as a UI afterthought.
6. **Cross-rung composition rules need a vocabulary.** The recursive composition of 1.0/2.0/3.0 (DSPy, constrained decoding, RLVR, skill distillation) is the actual stack and needs a top-down narrative.

## Core idea in one paragraph

Software 3.0 is the programming paradigm whose *primary source artifact* is a prompt + tool registry + skill library + memory, whose *primary build artifact* is a harness that loops a base LLM through that source under a verifier, and whose *primary deploy artifact* is an agent that installs the result onto target systems. The paradigm is recursive: 3.0 source compiles down to 1.0 (DSPy, constrained decoding, generated code) and trains 2.0 (RLVR, skill distillation, fine-tunes). The high-star ecosystem instantiates the paradigm at three levels — *runtime* (ollama, llama.cpp, OpenWebUI), *agent harness* (Claude Code, Cursor, OpenHands, Aider, Continue, Cline, AutoGPT, OpenInterpreter, gpt-engineer, gpt-pilot, bolt.new, SWE-agent), and *orchestration framework* (LangChain/LangGraph, AutoGen, CrewAI, DSPy, Semantic Kernel, OpenAI Agents SDK) — with **MCP** ([07](07-model-context-protocol.md)) emerging as the cross-cutting protocol that makes 3.0 portable across them. The harness-engineering literature ([40](40-harness-engineering-principles.md), [43](43-twelve-harness-patterns.md), [44](44-four-pillars-harness-engineering.md)) is the textbook for engineering at this rung; the discipline of doing it well is **agentic engineering** ([240](240-vibe-coding-to-agentic-engineering-discipline.md)).

## Mechanism (the rung structure step by step)

### (a) The trichotomy

| Rung | Source artifact | Build artifact | Deploy artifact | Engineering literature |
|---|---|---|---|---|
| **1.0** | Hand-written code (`.py`, `.ts`, `.go`) | Compiler / interpreter output | Binary, container, package | Decades of CS textbooks |
| **2.0** | Data + training loop + loss | Weights (`.safetensors`, `.gguf`, LoRA) | Inference server | [216-pretraining](216-pretraining-scaling-laws-foundation.md), [110](110-transformer-llm-architecture-for-agent-builders.md), [116-PEFT](116-adapter-tuning-lora-peft-for-agents.md) |
| **3.0** | Prompts + skills + tool schemas + memory | Harness (loop + tools + verifier + state) | Agent-installer | This corpus (40, 43, 44, 46, 62, 144, 145, 238) |

The rungs are *complementary* not exclusive: most production systems mix all three. A coding agent like Claude Code is 3.0 (skills, prompts, harness) calling a 2.0 base model (Claude weights), itself running atop 1.0 infrastructure (Python CLI, terminal, file system, git).

### (b) The 3.0 source artifact bundle

A complete 3.0 source bundle has six parts:

1. **System prompt / persona / role** — the steering text. Often versioned per environment.
2. **Skill bundle** — Markdown skill files with progressive disclosure ([04](04-skills.md), [19](19-voyager-skill-libraries.md), [68](68-atomic-skills-scaling-coding-agents.md), [156](156-heavyskill-parallel-reasoning-deliberation.md), [167](167-autoskill-experience-driven-lifelong-learning.md)).
3. **Tool registry / MCP servers** — typed function surfaces ([07](07-model-context-protocol.md), [236](236-tool-use-and-aci-scaling.md)).
4. **Memory files** — CLAUDE.md, scratchpads, episodic JSONL, semantic facts ([09](09-memory-files.md), [12](12-todo-scratchpad-state.md), [72](72-claude-mem-persistent-memory-compression.md), [151](151-memtier-why-flat-memory-breaks-at-72-hours.md), [181](181-mem0-deep-dive.md)).
5. **Eval suite** — golden traces, regression probes, judge prompts ([21](21-llm-as-judge-trajectory-eval.md), [115](115-evaluating-llm-systems.md)).
6. **Verifier** — automated checker (compile, test, schema, simulator, theorem prover) ([11](11-verifier-evaluator-loops.md), [80](80-knowrl.md), [223](223-verifier-and-best-of-n-scaling.md), [242](242-verifiability-bottleneck-and-jagged-skills.md)).

A 3.0 PR is a diff over this bundle. Prompts touch CI; skill changes touch eval CI; tool changes touch contract tests; memory changes touch retrieval evals; verifier changes touch the whole end-to-end loop.

### (c) The harness as the 3.0 build artifact

The harness is the 3.0 analog of a compiler + linker + runtime: it takes the source bundle and produces *agent behavior* on a stream of user inputs. The corpus's *Four Pillars* ([44](44-four-pillars-harness-engineering.md)) — State, Context, Guardrails, Entropy — names the four jobs a harness must do; the *Twelve Patterns* ([43](43-twelve-harness-patterns.md)) names the canonical implementations; the *Components of a Coding Agent* ([46](46-components-of-coding-agent.md)) lists the six concrete sub-systems every coding harness contains.

A 3.0 harness has the following minimum viable surface:

- **Loop** — think/act/observe with step budget ([01](01-agent-loop-architecture.md), [13](13-react.md)).
- **Tools** — function calls or MCP ([07](07-model-context-protocol.md)).
- **Context manager** — compaction, retrieval, scratchpad ([08](08-context-compaction.md), [12](12-todo-scratchpad-state.md), [25](25-agentic-rag.md)).
- **Memory** — persistent state across sessions ([09](09-memory-files.md), [10](10-multi-session-continuity.md)).
- **Verifier / evaluator** — gate before commit ([11](11-verifier-evaluator-loops.md), [18](18-chain-of-verification-self-refine.md), [21](21-llm-as-judge-trajectory-eval.md)).
- **Guardrails** — input/output filters, permission gates, hooks ([05](05-hooks.md), [06](06-permission-modes.md), [22](22-guardrails-prompt-injection.md)).
- **Observability** — trace, cost, attribution ([24](24-observability-tracing.md)).
- **HITL** — approval and intervention affordances ([23](23-human-in-the-loop.md), [143](143-ux-design-for-agentic-systems.md)).

### (d) The agent-as-installer as the 3.0 deploy artifact

The "installer" claim is the most operationally consequential of the IICYMI talk's framings. Concretely, an agent-installer is responsible for:

- **Bringing up runtime** — provisioning containers, registering MCP servers, mounting secrets, configuring environment.
- **Skill installation** — fetching skill bundles from a marketplace ([175](175-agent-skills-ecosystem-and-security.md), [205](205-lobehub-collaborative-teammate-platform.md)), verifying signatures, registering with the harness.
- **Tool wiring** — connecting MCP / function-call interfaces to actual systems (DBs, queues, APIs).
- **Smoke evals** — running the eval suite against the freshly-installed environment to confirm behavior matches the dev / staging baseline ([115](115-evaluating-llm-systems.md), [101](101-autoresearchbench.md), [102](102-clawgym-scalable-claw-agents.md)).
- **Post-install attestation** — emitting a signed report of what was installed, with what versions, against what evals, for audit ([122](122-explainability-compliance.md), [188](188-witness-provenance-memory-techniques-synthesis.md)).

Reference instances of agent-installer behavior in the wild: OpenHands ([github.com/All-Hands-AI/OpenHands](https://github.com/All-Hands-AI/OpenHands)) literally runs in a sandbox and brings up its own working environment; Aider ([github.com/Aider-AI/aider](https://github.com/Aider-AI/aider)) installs git hooks for its commit workflow; Claude Code installs MCP servers and skills; bolt.new spins up StackBlitz WebContainers; gpt-engineer instantiates whole project scaffolds; gpt-pilot ([github.com/Pythagora-io/gpt-pilot](https://github.com/Pythagora-io/gpt-pilot)) goes further and installs dependencies through a multi-step debugger loop. The pattern is converging.

### (e) Cross-rung composition rules

| From | To | Mechanism | Example |
|---|---|---|---|
| 3.0 | 1.0 | Compile prompts into deterministic code | DSPy ([93](93-dspy.md)), constrained decoding ([112](112-constrained-decoding.md)), Outlines, Instructor |
| 3.0 | 2.0 | RL or distillation from 3.0 trajectories | RLVR ([232](232-rl-on-reasoning-traces-scaling.md)), KnowRL ([80](80-knowrl.md)), HeavySkill internalization ([156](156-heavyskill-parallel-reasoning-deliberation.md)) |
| 1.0 | 3.0 | Wrap deterministic code as a tool | MCP servers ([07](07-model-context-protocol.md)), function-calling APIs |
| 2.0 | 3.0 | Use a fine-tuned model as the base of a harness | Devin/SWE-agent on tuned coder models, Memento ([106-109](106-memento-paper-theory.md)) |
| 3.0 | 3.0 | Compose harnesses; agent-of-agents | Subagents ([02](02-subagent-delegation.md)), MetaGPT ([20](20-metagpt-role-based-workflows.md), [91](91-metagpt-deep.md)), CrewAI ([164](164-crewai-multi-agent-framework.md)), AutoGen, supervisor patterns ([121](121-multi-agent-coordination-patterns.md)) |

The take-home: **3.0 is not a closed layer**; it leaks into 1.0 via compilation and into 2.0 via RL. Treat it as the *user-facing projection of a multi-layer system*.

## Empirical anchors and the 2026 high-star ecosystem

The Software 3.0 paradigm exists *because the ecosystem instantiated it*. The strongest empirical evidence is the convergent shape of the most-starred repos in the agent space. Three tiers:

### Tier A — Runtime layer (model serving, local-first)

- **ollama/ollama** (~100 K stars) — local LLM runtime; agents-as-installer ergonomics for 2.0 weights.
- **ggerganov/llama.cpp** (~70 K) — CPU/GPU inference; reference implementation for quantization and the GGUF format.
- **OpenWebUI** (~80 K) — chat surface over local/remote backends; ships a plug-in system that *is* a 3.0 skill registry.
- **vLLM**, **LiteLLM**, **TabbyAPI** — serving-layer projects that the harness sits on top of. LiteLLM specifically standardizes the *cross-vendor* call layer, an underrated 3.0 portability primitive ([147](147-vendor-lock-in.md)).

### Tier B — Coding-agent harnesses (the canonical 3.0 application family)

- **AutoGPT** (~170 K) — the original autonomous coding-agent attempt; lessons in [126-frameworks-comparison](126-frameworks-comparison.md), [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md).
- **gpt-engineer** (~53 K) — scaffold-an-entire-codebase wedge.
- **OpenHands** (~35 K, formerly OpenDevin) — sandbox-first, eval-grounded, SWE-bench-driven; the closest open-source analog to Devin.
- **gpt-pilot** (~30 K) — structured multi-step harness; explicit role separation ([20](20-metagpt-role-based-workflows.md), [91](91-metagpt-deep.md)).
- **cline/cline** (~30 K) — VS Code-embedded autonomous coder; first-class diff approval flow ([23](23-human-in-the-loop.md)).
- **aider** (~25 K) — repo-aware diff-driven CLI; pioneered repo-map context engineering ([08](08-context-compaction.md)).
- **continuedev/continue** (~18 K) — IDE-embedded copilot; configurable model + tool stack.
- **SWE-agent** (~14 K, princeton-nlp) — paper + reference harness for SWE-bench; the canonical Agent-Computer-Interface design ([237](237-agent-trajectory-scaling.md), [236](236-tool-use-and-aci-scaling.md)).
- **OpenInterpreter** (~55 K) — natural-language → local-machine action; the most direct expression of "agent as installer".
- **bolt.new / stackblitz** (~14 K) — full-stack web app generation in browser-sandbox; agent-installer at the project level.
- **Claude Code** (Anthropic, partly closed) — the de-facto reference 3.0 harness; deep-dives in [29](29-dive-into-claude-code.md), [62](62-everything-claude-code.md), [40](40-harness-engineering-principles.md).
- **Cursor** (closed-source product) — IDE-native pair-programmer; not a repo but the dominant commercial reference.
- **forks/derivatives:** roo-code, kilo-code, void, opencodex — the family is now too large to enumerate; see [66-meta-harness-landscape](66-meta-harness-landscape.md), [145-comparing-coding-harnesses](145-comparing-coding-harnesses.md).

### Tier C — Orchestration / framework layer (multi-agent, declarative)

- **langchain-ai/langchain** (~95 K) and **langgraph** (~8 K) — the dominant Python orchestration framework; LangGraph reframes agents as state-machines ([42-langchain-deep-agents](42-langchain-deep-agents.md), [126-frameworks-comparison](126-frameworks-comparison.md)).
- **microsoft/autogen** (~35 K) — multi-agent conversation framework; influential on MetaGPT, ChatDev, CrewAI lineages.
- **crewAIInc/crewAI** (~28 K) — role-based multi-agent crews ([164](164-crewai-multi-agent-framework.md)).
- **stanfordnlp/dspy** (~18 K) — declarative LM programs with optimization compilers ([93](93-dspy.md)); the most explicitly *programming-language-shaped* 3.0 framework.
- **bytedance/deer-flow** — research-agent harness ([65](65-deer-flow-bytedance.md), [163](163-deer-flow-revisited-may-2026.md)).
- **All-Hands-AI/OpenHands** straddles tiers B and C with subagent + memory.
- **mem0**, **letta** (formerly MemGPT), **Zep** — memory-substrate frameworks ([181](181-mem0-deep-dive.md), [183](183-oss-memory-landscape-may-2026.md), [184](184-strongest-memory-techniques-synthesis-may-2026.md)).

### Cross-cutting protocol

- **Model Context Protocol (MCP)** — Anthropic-led, 2024 spec; now adopted by every harness in tiers B–C ([07](07-model-context-protocol.md)). MCP is the **3.0 ABI**: tools and skills are portable across harnesses if and only if they expose MCP. This is the closest analog to the C ABI on Unix.

## Variants and ablations

- **Static workflow vs dynamic agent.** Workflows ([114](114-workflows-vs-agents.md)) sit at the 3.0/1.0 boundary — declarative pipelines with deterministic control flow. Agents ([01](01-agent-loop-architecture.md), [13](13-react.md)) sit deeper in 3.0 — let the LM choose control flow at every step. Pick by verifier density: where verification is cheap, prefer agents; where not, prefer workflows.
- **Single-agent vs multi-agent.** The 2025–2026 evidence ([98](98-diversity-collapse-mas.md), [202](202-multi-agent-multi-hop-reckoning-2026.md), [224](224-multi-agent-parallel-scaling.md)) is that multi-agent helps *less than its proponents claim* and degrades when diversity collapses. Default to single-agent; promote to multi-agent only with measured benefit.
- **Local vs hosted.** Tier-A repos (ollama, llama.cpp) make Software 3.0 viable on a laptop; Tier-B/C frameworks span both. Local-first 3.0 is a privacy and cost wedge ([206](206-collaborative-ai-canon-2026.md)).
- **Open-weights base vs frontier API.** The 3.0 layer is mostly model-portable thanks to LiteLLM and MCP; specific harnesses (Claude Code skills) lean on per-vendor extensions ([147](147-vendor-lock-in.md)).
- **Compiled 3.0 (DSPy, Outlines, Instructor) vs interpreted 3.0 (raw prompts).** Compiled wins for stable production; interpreted wins for exploration. The discipline is choosing which sub-trees to compile.

## Failure modes and limitations

- **Treating 3.0 as "config".** Configuration-grade review processes (no PR review, no eval, no version) cause production outages from prompt drift and tool-schema breakage. 3.0 needs full SDLC.
- **Treating 3.0 as 1.0.** Applying deterministic-test thinking to stochastic outputs misses the eval discipline 3.0 requires ([21](21-llm-as-judge-trajectory-eval.md), [115](115-evaluating-llm-systems.md)).
- **Treating 3.0 as 2.0.** Demanding fine-tunes for every behavioral change burns weeks where a prompt + skill + verifier change would have shipped in a day. 3.0 is the *fast iteration layer*.
- **Lock-in to a single vendor's 3.0 dialect.** Skills and prompts written for one harness rarely move cleanly to another without MCP. Build vendor-neutral where you can ([147](147-vendor-lock-in.md)).
- **Treating "agent" as the deploy artifact when "workflow" would do.** Many production systems are workflows misrepresented as agents because "agent" sells better. The Software 3.0 frame should encourage the more honest classification.
- **Ignoring the recursive composition rules.** Teams that don't compile down to 1.0 (DSPy, constrained decoding) leave latency, cost, and reliability gains on the table; teams that don't distill into 2.0 (RLVR, skill internalization) leave durable capability gains on the table.

## When to use this paradigm framing, when not

**Use it** when: (a) bootstrapping team vocabulary; (b) explaining the stack to executives or non-technical stakeholders ([148](148-beginner-onramp-what-is-agentic-ai.md)); (c) onboarding new engineers ([113](113-from-tokens-to-agents-onramp.md)); (d) doing 6-month roadmap planning that needs to allocate effort across rungs; (e) auditing a system for which rung each artifact lives at and who owns it.

**Do not use it** when: (a) you need precise per-task implementation guidance (use the harness pattern catalogs in [40](40-harness-engineering-principles.md), [43](43-twelve-harness-patterns.md), [44](44-four-pillars-harness-engineering.md), [144](144-build-your-own-harness.md)); (b) the conversation is about base-model selection ([119](119-agent-ready-llm-selection.md)) — that lives at 2.0; (c) you are arguing about safety models — those need the threat-model-specific framings in [22](22-guardrails-prompt-injection.md), [35](35-malicious-intermediary-attacks.md), [82](82-poisonedrag.md), [49](49-agents-of-chaos-red-teaming.md).

## Implications for harness engineering

1. **Tag every artifact with its rung.** Source tree organization should make 1.0/2.0/3.0 visible at a glance — `code/` (1.0), `weights/` or `adapters/` (2.0), `skills/`, `prompts/`, `mcp/`, `memory/`, `evals/` (3.0). Review and CI rules differ per directory.
2. **Treat the 3.0 source bundle as a shipping artifact.** Versioned, signed, distributable. Skill marketplaces ([175](175-agent-skills-ecosystem-and-security.md), [205](205-lobehub-collaborative-teammate-platform.md)) are the package registry equivalent.
3. **Adopt MCP as your 3.0 ABI.** Even if you only run one harness today, exposing tools as MCP keeps you portable when the harness rotates ([07](07-model-context-protocol.md), [147](147-vendor-lock-in.md)).
4. **Build agent-installers for production deploy.** Not Helm + manual steps; an agent that brings up the environment, installs skills, runs smoke evals, and signs a deploy attestation. See [165-ralph-autonomous-loop](165-ralph-autonomous-loop.md), [124-agent-level-production-patterns](124-agent-level-production-patterns.md), [125-system-level-production-patterns](125-system-level-production-patterns.md).
5. **Compile down to 1.0 where the curve flattens.** DSPy ([93](93-dspy.md)) for declarative pipelines; constrained decoding ([112](112-constrained-decoding.md)) for schema-bound output; codegen-then-execute for hot paths.
6. **Distill into 2.0 where 3.0 plateaus.** Once a skill is stable and high-volume, internalize it via fine-tune or RLVR ([116](116-adapter-tuning-lora-peft-for-agents.md), [232](232-rl-on-reasoning-traces-scaling.md), [156](156-heavyskill-parallel-reasoning-deliberation.md)). The 3.0 source becomes an *eval and regression suite* for the 2.0 internalization.
7. **Pick a Tier-B harness and align your conventions to it.** Adopt Claude Code, OpenHands, Cline, or Aider conventions; do not invent your own. The cost of homegrown deviation now exceeds the cost of adoption ([66](66-meta-harness-landscape.md), [144](144-build-your-own-harness.md), [145](145-comparing-coding-harnesses.md)).
8. **Model the eval suite as part of the source bundle, not as an external test pack.** When the prompt changes, the eval pack runs in the same CI step ([115](115-evaluating-llm-systems.md), [21](21-llm-as-judge-trajectory-eval.md)). Coupling them is what makes 3.0 *engineerable*.
9. **Pair menus with raw prompts.** Every raw-prompt path that recurs should grow a slash-command, a skill, or a UI affordance ([04](04-skills.md), [62](62-everything-claude-code.md)). Raw prompts are the IR; menus are the user-facing artifact.
10. **Plan rung-aware roadmaps.** Quarterly planning should explicitly allocate "3.0 work" (skills, prompts, harness), "2.0 work" (fine-tunes, distillations, eval-data labeling), and "1.0 work" (tools, infra, sandboxes). Confused rungs cause confused capacity planning.
11. **Keep the cross-rung composition rules visible.** Architecture decision records ([122](122-explainability-compliance.md)) should explicitly note when a 3.0 sub-system was compiled to 1.0 (and why) or distilled into 2.0 (and why). Future-you needs that breadcrumb.
12. **Treat the high-star ecosystem as a moving textbook.** A weekly read of new skills/agents on the top repos is now the equivalent of reading conference proceedings. The Argus-style skills curator ([177](177-skills-discovery-curator-strongest-2026-techniques.md), [194](194-argus-omega-enhanced-design.md)) is the long-run automation of that read.

## One-line takeaway for harness designers

**Software 3.0 is the rung where prompts + skills + tools + memory + evals + verifier + harness compose into agent behavior, and the agent itself is the installer that puts that behavior on the target system — engineer it as a discipline, ship it through MCP, compile down to 1.0 where the curve flattens, distill into 2.0 where it plateaus, and pick a Tier-B harness convention rather than inventing your own.**
