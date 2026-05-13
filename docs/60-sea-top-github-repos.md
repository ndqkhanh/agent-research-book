# 60 — Top-Starred Self-Evolving-Agent GitHub Repositories (April 2026)

**Definition.** A *self-evolving agent* (SEA) is one whose capability grows across time without a fresh training run — by writing skills, editing its own prompts, revising its own code, or curating its own experience. This document catalogues the open-source SEA implementations that people actually star and run as of 2026-04-20. It is the empirical counterpart to the more theoretical entries in this corpus — [19 — Voyager skill libraries](19-voyager-skill-libraries.md), [36 — Autogenesis protocol](36-autogenesis-self-evolving-agents.md), [45 — Hyperagents self-modification](45-hyperagents-self-modification.md), [47 — Adaptation of Agentic AI survey](47-adaptation-of-agentic-ai-survey.md), [55 — Hermes Agent](55-hermes-agent-self-improving.md) — mapped onto shipped code.

## Scope

**Inclusion criteria.**

1. The project is a working, publicly installable codebase (not a survey, not a wishlist, not a paper-only release). The sole "awesome-list" entry is flagged as such and noted for navigation value only.
2. It implements a mechanism by which the agent's behavior changes *across episodes* without a fresh pre-training pass — skill accumulation, prompt evolution, code self-edits, self-play curriculum, or experience-guided RL.
3. It has non-trivial traction — meaningful star count relative to its niche — or it is the canonical implementation of a widely-cited technique (e.g., Voyager, even though it is no longer top-of-leaderboard).
4. Star counts and activity signals verified via WebFetch against GitHub on 2026-04-20. Where a number could not be verified, it is labeled "unverified".

**Explicit non-goals.**

- Proprietary agents (Claude Code, Devin, Cursor) — no source to study. Covered in [29](29-dive-into-claude-code.md).
- General agent frameworks without an evolution loop (LangChain core, AutoGen, CrewAI). These route and orchestrate; they do not accumulate capability across runs on their own.
- Single-shot reflection techniques with no persistent memory across tasks (classical Reflexion, Self-Refine). Covered as patterns in [14](14-reflexion.md), [18](18-chain-of-verification-self-refine.md).
- Training-time RL-only work that ships model weights but no runtime self-evolution harness.

**Corpus selection.** Candidates surfaced via WebSearch, cross-checked against the two active survey lists ([EvoAgentX](https://github.com/EvoAgentX/Awesome-Self-Evolving-Agents), [XMUDeepLIT](https://github.com/XMUDeepLIT/Awesome-Self-Evolving-Agents)), triaged down to eight primary entries spanning the design space: skill-library agents, self-play reasoners, code-self-editing agents, prompt-optimizer frameworks, and personal-agent harnesses.

---

## 1. Hermes Agent (NousResearch)

- **URL.** <https://github.com/nousresearch/hermes-agent>
- **Owner.** Nous Research
- **Stars.** 104k (verified 2026-04-20 via GitHub page).
- **Forks.** 14.9k.
- **Last activity.** v0.10.0 released 2026-04-16 (four days before this survey). Extremely active.
- **License.** MIT.

**What it does.** A single-agent personal harness — "the agent that grows with you" — that extracts completed multi-step workflows into Markdown `SKILL.md` procedures, then retrieves and executes those skills on similar future tasks. It runs as a unified messaging gateway across Telegram, Discord, Slack, WhatsApp, Signal, and CLI, with a cache-aware three-layer memory architecture (procedural skills, preference facts, session buffer) and optional Honcho dialectic user modeling.

**Architecture.** *Loop shape.* Completion-triggered: after every finished task, an extractor pass analyzes the trace and emits a skill candidate. A self-eval cadence every ~15 tasks decides promotions, consolidations, and retirements. *Skill store.* Markdown files compatible with the [agentskills.io](https://agentskills.io) standard (same shape as [04 — Skills](04-skills.md)), retrieved by FTS5 + semantic summary. *Reward signal.* Task outcome (success / partial / failure) folded back into skill body updates; no RL, no gradients. *Verifier.* Task-specific; the associated [hermes-agent-self-evolution](https://github.com/NousResearch/hermes-agent-self-evolution) repo (2k stars, MIT) layers DSPy + GEPA optimization with test-suite and size gates.

**Novel techniques.** The gateway-as-identity pattern (one coherent personality across 6+ channels); skill resolution JIT-inlined so token cost scales with task, not library size; the $2-10 API-only GEPA evolution loop with no GPU.

**Production-readiness.** Shipping v0.10; deployed on $5 VPS through serverless GPU; enormous third-party ecosystem ([awesome-hermes-agent](https://github.com/0xNyk/awesome-hermes-agent)); commercially friendly MIT. **Ready.**

**Relationship to papers.** A direct engineering descendant of Voyager (skill library + verification core, minus the curriculum). The companion self-evolution repo implements GEPA (ICLR 2026 Oral). Covered in detail in [55 — Hermes Agent](55-hermes-agent-self-improving.md).

**Copy-these-patterns pros.** Clean separation of gateway from harness; skill format is portable and human-readable; cache-aware memory design solves the token-bill-as-learning-tax problem. **Cons.** The skill extractor is LLM-driven and occasionally mis-generalizes; large skill libraries still need curation; messaging-gateway operational surface is wider than a CLI tool.

---

## 2. DSPy (Stanford NLP)

- **URL.** <https://github.com/stanfordnlp/dspy>
- **Owner.** stanfordnlp
- **Stars.** 33.8k (verified).
- **Forks.** 2.8k.
- **Last activity.** v3.1.3 released 2026-02-05; actively maintained.
- **License.** MIT.

**What it does.** A declarative framework for programming (rather than prompting) language models — pipelines are expressed as Python modules with signatures, and a *compiler* optimizes prompts and few-shot examples against a validation metric. It is the parent technology of the modern prompt-evolution ecosystem (MIPROv2, GEPA, BootstrapFewShot, COPRO).

**Architecture.** *Loop shape.* Offline: compile over a train set with a metric, get an improved program; no online per-episode loop by default. *Skill store.* The compiled program itself — prompts, few-shot demos, instructions — serialized to disk. *Reward signal.* Any user-supplied metric (LLM-as-judge, exact-match, task-specific). *Verifier.* Metric is the verifier.

**Novel techniques.** Signatures as typed interfaces separating "what" from "how"; teacher-student bootstrapping; MIPRO's joint instruction + demo search; GEPA's reflective prompt evolution with Pareto fronts over a trace.

**Production-readiness.** Deeply battle-tested — countless production pipelines, heavy enterprise use, stable API. **Ready.**

**Relationship to papers.** Implements its own paper (DSPy, 2023) and a growing family of optimizer papers (MIPROv2, GEPA). Many downstream SEA repos (EvoAgentX, hermes-agent-self-evolution) consume DSPy rather than re-implement.

**Copy-these-patterns pros.** Signatures are a clean contract to hang evolution on — you can swap prompts without touching orchestration. **Cons.** Not itself an agent — no tool loop, no persistent memory, no channels; you build the agent and invoke DSPy to evolve the prompt components. Best used *inside* a harness, not as one.

---

## 3. Voyager (MineDojo)

- **URL.** <https://github.com/MineDojo/Voyager>
- **Owner.** MineDojo (NVIDIA / Caltech alumni)
- **Stars.** 6.8k (verified).
- **Forks.** 666.
- **Last activity.** Substantially dormant — canonical reference implementation rather than a live project. Last significant commits in late 2023.
- **License.** MIT.

**What it does.** The original LLM-powered lifelong-learning Minecraft agent: automatic curriculum proposes tasks, iterative code-prompting writes Mineflayer JS to solve them, and verified code snippets land in a growing skill library indexed by docstring embeddings. It unlocked 3.3× more unique items and progressed up the tech tree 15× faster than prior Minecraft agents.

**Architecture.** *Loop.* Curriculum → code-write → execute → self-verify → (if pass) skill-library add. *Skill store.* Embedding-indexed code snippets, retrieved top-5 per new task. *Reward signal.* Self-verification LLM checks task completion from observations. *Verifier.* A second LLM instance running a task-completion classifier.

**Novel techniques.** The three-part formula (curriculum + iterative prompting + skill library) became the canonical SEA blueprint; code-as-memory; docstring embedding retrieval.

**Production-readiness.** Research artifact — Mineflayer-coupled, not trivially generalizable. **Not production.** But the template is everywhere: Hermes, OpenSpace, GenericAgent, AgentEvolver all descend from it.

**Relationship to papers.** Canonical implementation of Wang et al. 2023 "Voyager". Extended conceptually in [19 — Voyager-style skill libraries](19-voyager-skill-libraries.md).

**Copy-these-patterns pros.** Small, legible codebase — you can actually read the loop end to end in one sitting; the skill-library abstraction lifts to any domain with code + executor. **Cons.** Minecraft-specific glue; no multi-tenant concerns; dormant.

---

## 4. AI Scientist (Sakana AI)

- **URL.** <https://github.com/SakanaAI/AI-Scientist> (v1); <https://github.com/SakanaAI/AI-Scientist-v2>
- **Owner.** Sakana AI
- **Stars.** v1 — 13.3k; v2 — 5.7k (verified).
- **Forks.** v1 — 1.9k; v2 — 794.
- **Last activity.** v2 active; v1 paused (canonical v1 release published in *Nature*, 2026).
- **License.** "AI Scientist Source Code License" — a derivative of the Responsible AI License. **Not OSI-approved; commercial use is restricted.**

**What it does.** An end-to-end autonomous research agent: given a broad direction, it generates hypotheses, reads literature, writes and runs experiments, analyzes results, and produces a full LaTeX paper. v2 replaces v1's human-authored templates with progressive agentic tree search guided by an experiment-manager agent.

**Architecture.** *Loop.* Idea → literature → code → experiment → analysis → write-up, with tree-search over experiment designs in v2. *Skill store.* Not a general skill library — per-project codebases accumulate within a run. *Reward signal.* Automated reviewer agents score the paper; v2's manager prunes the tree on expected value. *Verifier.* Compile-and-run + automated reviewer + (in v2) an accepted workshop paper as the end-to-end proof.

**Novel techniques.** Progressive agentic tree search; automated peer-reviewer-as-reward; the first AI-only paper to pass human peer review (ICBINB workshop, 2025).

**Production-readiness.** Usable for research-automation use cases with careful supervision. The license materially restricts adoption. **Research / narrow.**

**Relationship to papers.** Self-implementing — each repo is the Nature paper's artifact. v1 inspired a wave of science-agent papers (Chain-of-Scientists, Research-Town, AutoReview).

**Copy-these-patterns pros.** Clear module boundaries (idea-gen, coding-agent, reviewer) you can copy individually; the reviewer-as-reward is portable. **Cons.** License; tree-search in v2 is expensive ($20-50/paper in compute); not a general harness.

---

## 5. Darwin Gödel Machine (Sakana AI / Jenny Zhang)

- **URL.** <https://github.com/jennyzzt/dgm>
- **Owner.** jennyzzt (first author; Sakana AI)
- **Stars.** 2.0k (verified).
- **Forks.** 404.
- **Last activity.** Moderate — core research release, community forks active.
- **License.** Apache-2.0.

**What it does.** A self-improving coding agent that iteratively modifies its own source code, validates each modification against coding benchmarks (SWE-bench, Polyglot), and keeps or discards the variant based on score. Reported 20.0% → 50.0% on SWE-bench under its own self-modification regime.

**Architecture.** *Loop.* Parent agent → propose-and-apply patch to self → child agent runs benchmark → score delta gates commit to the population. *Skill store.* An archive of agent variants (code + score), treated as a population à la open-ended evolution. *Reward signal.* Benchmark score on held-out tasks. *Verifier.* The benchmark harness itself.

**Novel techniques.** Population-based open-ended evolution applied to *agent source code*, not just prompts; empirically validated self-modification loop; the "Gödelian" framing of agents that improve their ability-to-improve.

**Production-readiness.** **Research only.** The README explicitly warns about executing model-generated code; there is no sandbox baked in beyond what you bring.

**Relationship to papers.** Implements Zhang et al. 2025 "Darwin Gödel Machine: Open-Ended Evolution of Self-Improving Agents". The [Huxley-Gödel Machine](https://github.com/metauto-ai/HGM) (361 stars, Apache-2.0, ICLR 2026 Oral) extends it with an explicit approximation lens.

**Copy-these-patterns pros.** Cleanest reference implementation of "agent edits its own code" pattern; population archive is a useful abstraction when you want diversity. **Cons.** Safety posture is DIY; compute-hungry (every iteration runs a benchmark); benchmark-score-as-reward narrows what you evolve toward.

---

## 6. OpenSpace (HKUDS)

- **URL.** <https://github.com/HKUDS/OpenSpace>
- **Owner.** HKU Data Intelligence Lab
- **Stars.** 5.5k (verified).
- **Forks.** 680.
- **Last activity.** Commit 2026-04-16 (four days ago) on evolution-candidate lifecycle tracking. Actively developed.
- **License.** MIT.

**What it does.** A self-evolving "skill engine" that sits *alongside* any host agent (OpenClaw, nanobot, Claude Code, Codex, Cursor via MCP) and contributes learned, token-efficient skills — while also exposing a community cloud at open-space.cloud where evolved skills are shared across users. Reports 46% token reduction and a 4.2× "income" multiplier on GDPVal vs baseline.

**Architecture.** *Loop.* Cold-start → task execution → post-hoc evolution with three modes (**FIX** repairs broken skills, **DERIVED** specializes generic skills for recurring contexts, **CAPTURED** extracts new skills from successful traces) → warm rerun. *Skill store.* Local SQLite + optional cloud sync. *Reward signal.* Task success + token cost + reuse frequency; Qwen 3.5-Plus drives the evolution judgments. *Verifier.* Per-mode validators, plus GDPVal as the headline benchmark.

**Novel techniques.** The tri-modal evolution taxonomy (FIX / DERIVED / CAPTURED) is sharper than Voyager's monolithic "add skill" primitive; the MCP-as-engine pattern lets a single evolving skill base serve multiple host harnesses at once; community sync with skill-level provenance.

**Production-readiness.** Shipping and integrated with multiple popular hosts. MIT. **Production-viable** for the agent-skill-backend role, though the community cloud is young.

**Relationship to papers.** Own-paper ("OpenSpace: Make Your Agents Smarter, Low-Cost, Self-Evolving", HKUDS 2026). Related conceptually to [04 — Skills](04-skills.md), [19 — Voyager](19-voyager-skill-libraries.md), and the T2 (agent-supervised tool-side) slot from [47 — Adaptation survey](47-adaptation-of-agentic-ai-survey.md).

**Copy-these-patterns pros.** The three-mode taxonomy is useful even if you don't adopt the rest; the MCP-backend model is a clean alternative to embedding skill logic inside the harness; GDPVal coverage gives you real task economics. **Cons.** Qwen-tuned — your mileage varies with other backbones; community skill sharing has obvious trust/safety surface.

---

## 7. GenericAgent (lsdefine)

- **URL.** <https://github.com/lsdefine/GenericAgent>
- **Owner.** lsdefine (Fudan-affiliated)
- **Stars.** 4.8k (verified).
- **Forks.** 523.
- **Last activity.** Recent, very active through early 2026.
- **License.** MIT.

**What it does.** A minimalist (~3.3K LoC seed) autonomous agent that grants any LLM system-level control over a local computer via 9 atomic tools, and automatically "crystallizes" every solved task into a reusable skill — growing a personal skill tree from the seed. Claims 6× lower token consumption than comparable agents by keeping the context window under 30K.

**Architecture.** *Loop.* ReAct-style inner loop + post-task crystallization pass that promotes the trace to a callable skill. *Skill store.* Skill-tree on disk, loaded into context on demand. *Reward signal.* Task completion + operator feedback. *Verifier.* Tool-execution success + LLM-as-judge on skill quality. *Tools.* 9 atomic primitives (browser, terminal, file ops, input, mobile) plus a `code_run` that can install packages and create new tools at runtime.

**Novel techniques.** Extreme minimalism as a design constraint (3K seed); dynamic tool creation at runtime via `code_run`; aggressive token-efficiency discipline that forces the skill-tree path to pay off quickly.

**Production-readiness.** Solo personal-computer agent; works, but the default execution model is *your own machine* with few guardrails — suitable for power users, not multi-tenant deploys. **Personal-use-ready.**

**Relationship to papers.** Not tied to a specific paper — implementation-first project that sits in the Voyager → Hermes lineage with a harder minimalism bias.

**Copy-these-patterns pros.** The seed-code size is the whole point: you can read it top to bottom and port the crystallization loop into your own harness. **Cons.** Thin security story; docs are partly Chinese-first; broad LLM compatibility is aspirational on edge models.

---

## 8. EvoAgentX (EvoAgentX)

- **URL.** <https://github.com/EvoAgentX/EvoAgentX>
- **Owner.** EvoAgentX (Zaiqiao Meng et al., Glasgow)
- **Stars.** 2.9k (verified).
- **Forks.** 247.
- **Last activity.** Active; v0.1.0 release 2025-09-06; ongoing commits through 2026.
- **License.** MIT.

**What it does.** A framework for building and *evolving* multi-agent workflows — takes a natural-language goal plus a dataset and produces an optimized multi-agent pipeline using state-of-the-art self-evolving algorithms (integrates TextGrad, MIPRO, AFlow out of the box). Ships memory modules, toolkits, and HITL checkpoints.

**Architecture.** *Loop.* Workflow synthesis from NL → execute on dataset → score → invoke evolution algorithm (TextGrad / MIPRO / AFlow) → commit variant if better. *Skill store.* Workflow graph + per-node prompts; memory modules for both short- and long-term recall. *Reward signal.* User-supplied metric on user-supplied dataset. *Verifier.* Dataset metric + optional human reviewer at HITL checkpoints.

**Novel techniques.** Explicitly menu-of-optimizers — rather than one evolution algorithm, it unifies TextGrad / MIPRO / AFlow under a single workflow abstraction; HITL first-class; memory module is not bolted on.

**Production-readiness.** Published, documented, stable enough for internal platforms. **Production-cautious** — API still evolving.

**Relationship to papers.** Owns a 2025 arXiv paper and a 2025 self-evolving-agents survey; is effectively the "reference platform" for EvoAgentX's own Awesome-Self-Evolving-Agents list ([2.1k stars](https://github.com/EvoAgentX/Awesome-Self-Evolving-Agents)).

**Copy-these-patterns pros.** Best one-stop shop for comparing TextGrad vs MIPRO vs AFlow head to head; workflow-graph abstraction is clean. **Cons.** Heavier than Hermes/GenericAgent; the "evolve the workflow" framing is powerful when you already have a dataset, less useful for personal-assistant use.

---

## Also notable (briefly)

- **[TextGrad (zou-group)](https://github.com/zou-group/textgrad).** 3.5k stars, MIT. Automatic "differentiation" via text — LLM-produced textual gradients back-propagated through a PyTorch-like computation graph. Published in *Nature* 2025. Not an agent itself, but the optimization substrate under many SEA systems.
- **[AgentEvolver (modelscope)](https://github.com/modelscope/AgentEvolver).** 1.4k stars, Apache-2.0. End-to-end RL training framework built on three "self" mechanisms (self-questioning, self-navigating, self-attributing). 7B model reportedly beats most 14B baselines on in-domain evals. Training-heavy; if you have GPU budget and want to *train* self-evolving agents rather than evolve-at-inference, this is the main option.
- **[Absolute Zero Reasoner (LeapLabTHU)](https://github.com/LeapLabTHU/Absolute-Zero-Reasoner).** 1.8k stars, MIT. Self-play reasoning with *zero* external data — the model proposes tasks, solves them, and uses a code executor as the ground truth. Relevant as a training-time counterpart to runtime skill evolution; narrow in scope (reasoning-on-code).
- **[Huxley-Gödel Machine (metauto-ai)](https://github.com/metauto-ai/HGM).** 361 stars, Apache-2.0, ICLR 2026 Oral. An approximation-theoretic refinement of DGM.
- **[EvoAgentX/Awesome-Self-Evolving-Agents](https://github.com/EvoAgentX/Awesome-Self-Evolving-Agents).** 2.1k stars, MIT. Survey list — useful for navigation, not an implementation.
- **[XMUDeepLIT/Awesome-Self-Evolving-Agents](https://github.com/XMUDeepLIT/Awesome-Self-Evolving-Agents).** Second survey list. Star count unverified at fetch time.

Two repos I tried to surface but could not treat as first-class: any hypothetical "Nous Research Hermes" non-agent repo (Nous's Hermes-the-LLM is separate from hermes-agent, which is what's in this survey) and a "self-refine agent github 2026" that resolved only to individual example notebooks rather than a single canonical repo.

---

## Cross-repo synthesis

**What is converging.**

1. **Skills as the unit of evolution.** Hermes, OpenSpace, GenericAgent, Voyager, and (implicitly) EvoAgentX all settle on *named, Markdown-or-code skills retrieved on demand* as the storage primitive — not episodic memory, not vectors-of-tokens. This is the pattern that won. See [04 — Skills](04-skills.md) for why the shape is so stable.
2. **Trace-driven post-hoc promotion.** The dominant loop is: finish task → analyze the trace → decide whether to write, update, or retire a skill. Voyager's per-task verification, Hermes's 15-task cadence, OpenSpace's three-mode classifier, and GenericAgent's crystallization pass are all variants of this.
3. **Executor-grounded rewards.** Self-verification via code execution (Absolute Zero), benchmark score (DGM), or tool-call success (Hermes, GenericAgent) has replaced pure LLM-judge scoring as the default. LLM judges remain for quality gating, not for ground-truth.
4. **External optimizer libraries are winning over bespoke evolution code.** DSPy (with MIPROv2 and GEPA) and TextGrad are the substrate under EvoAgentX and hermes-agent-self-evolution. Rolling your own genetic loop is no longer the norm.

**What is still experimental.**

1. **Source-code self-edits** (DGM, HGM). Impressive benchmark gains; scary to operate; no one ships this to end users. This is where the research frontier remains hottest but the production frontier is coldest.
2. **Community skill sharing** (OpenSpace's open-space.cloud, implicitly the awesome-hermes-agent catalog). Trust, provenance, and injection attacks across a multi-user skill marketplace are unsolved — see [22 — Guardrails / prompt injection](22-guardrays-prompt-injection.md) and [35 — Malicious intermediary attacks](35-malicious-intermediary-attacks.md). Compare the threat-model surface to the concerns raised in [49 — Agents of chaos](49-agents-of-chaos-red-teaming.md).
3. **Zero-data self-play** (Absolute Zero). Works on code reasoning; generalization to open-ended domains is TBD.
4. **Population-based open-ended evolution** outside coding benchmarks. DGM is the canonical case; non-SWE-bench domains are lightly explored.

**Which patterns win for a new harness.**

- If you want a **personal agent**: copy Hermes's loop and memory split, consume OpenSpace as your skill-engine MCP backend.
- If you want a **coding agent**: copy DGM's population archive, but keep source-code edits in a worktree with strict tests (see [52 — OpenClaw](52-dive-into-open-claw.md) for worktree patterns).
- If you want an **evaluable workflow**: build on EvoAgentX and let DSPy/TextGrad/AFlow compete on your metric.
- If you want a **single-machine assistant**: read GenericAgent's 3K seed first, understand the crystallization pass, then graft it onto your own loop.

The template-that-works is Voyager + verifier + MCP-skill-engine + DSPy/GEPA offline optimizer. Everything else is flavor.

## References

All star counts and URLs verified via WebFetch on 2026-04-20 unless marked "unverified".

- [Hermes Agent](https://github.com/nousresearch/hermes-agent) — 104k stars, 14.9k forks, MIT, v0.10.0 2026-04-16.
- [Hermes Agent Self-Evolution](https://github.com/NousResearch/hermes-agent-self-evolution) — 2k stars, 194 forks, MIT.
- [DSPy](https://github.com/stanfordnlp/dspy) — 33.8k stars, 2.8k forks, MIT, v3.1.3 2026-02-05.
- [Voyager](https://github.com/MineDojo/Voyager) — 6.8k stars, 666 forks, MIT.
- [AI Scientist](https://github.com/SakanaAI/AI-Scientist) — 13.3k stars, 1.9k forks, AI Scientist Source Code License (non-OSI).
- [AI Scientist v2](https://github.com/SakanaAI/AI-Scientist-v2) — 5.7k stars, 794 forks, AI Scientist Source Code License.
- [Darwin Gödel Machine](https://github.com/jennyzzt/dgm) — 2.0k stars, 404 forks, Apache-2.0.
- [Huxley-Gödel Machine](https://github.com/metauto-ai/HGM) — 361 stars, 57 forks, Apache-2.0.
- [OpenSpace](https://github.com/HKUDS/OpenSpace) — 5.5k stars, 680 forks, MIT, last commit 2026-04-16.
- [GenericAgent](https://github.com/lsdefine/GenericAgent) — 4.8k stars, 523 forks, MIT.
- [EvoAgentX](https://github.com/EvoAgentX/EvoAgentX) — 2.9k stars, 247 forks, MIT.
- [TextGrad](https://github.com/zou-group/textgrad) — 3.5k stars, 287 forks, MIT (published *Nature* 2025).
- [AgentEvolver](https://github.com/modelscope/AgentEvolver) — 1.4k stars, 161 forks, Apache-2.0.
- [Absolute Zero Reasoner](https://github.com/LeapLabTHU/Absolute-Zero-Reasoner) — 1.8k stars, 298 forks, MIT.
- [EvoAgentX Awesome-Self-Evolving-Agents](https://github.com/EvoAgentX/Awesome-Self-Evolving-Agents) — 2.1k stars, 147 forks, MIT.
- [XMUDeepLIT Awesome-Self-Evolving-Agents](https://github.com/XMUDeepLIT/Awesome-Self-Evolving-Agents) — star count unverified.

**Cross-references within this corpus.**

- [04 — Skills](04-skills.md) — the shared `SKILL.md` standard.
- [11 — Verifier evaluator loops](11-verifier-evaluator-loops.md) — verifier patterns cited throughout.
- [19 — Voyager skill libraries](19-voyager-skill-libraries.md) — theoretical backbone of #3.
- [22 — Guardrails / prompt injection](22-guardrails-prompt-injection.md), [35 — Malicious intermediary attacks](35-malicious-intermediary-attacks.md), [49 — Agents of chaos](49-agents-of-chaos-red-teaming.md) — safety surface for community-shared skills.
- [36 — Autogenesis protocol](36-autogenesis-self-evolving-agents.md) — protocol-level framework these repos approximate informally.
- [45 — Hyperagents self-modification](45-hyperagents-self-modification.md) — theoretical precursor to DGM/HGM.
- [47 — Adaptation of Agentic AI survey](47-adaptation-of-agentic-ai-survey.md) — T1/T2 classification used above.
- [52 — OpenClaw](52-dive-into-open-claw.md) — host harness commonly paired with OpenSpace.
- [55 — Hermes Agent](55-hermes-agent-self-improving.md) — long-form treatment of #1.
