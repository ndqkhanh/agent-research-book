# 240 — From Vibe Coding to Agentic Engineering: a one-year maturation arc

**Anchors.** Andrej Karpathy — X post coining "vibe coding" — February 2025. Andrej Karpathy — IICYMI talk, May 2026, chapter *From Vibe Coding to Agent Engineering* (15:46) — `https://www.youtube.com/watch?v=96jN2OCOfLs`. Andrej Karpathy — X post on LLM coding pathologies (the *Think Before Coding / Simplicity First / Surgical Changes / Goal-Driven Execution* observations) — November 2025. Anthropic Engineering — *Effective Harnesses for Long-Running Agents*, 2024–2026. Cognition AI — *Don't Build Multi-Agents*, 2024. HumanLayer — *Skill Issue: Harness Engineering for Coding Agents*, 2024. Geoffrey Huntley — *The Ralph Loop*, 2024–2025 ([165](165-ralph-autonomous-loop.md)). **Reference repos for the maturation arc:** forrestchang/**andrej-karpathy-skills** (~71 K stars, [71](71-karpathy-skills-single-file-guardrails.md)), **claude-mem** ([72](72-claude-mem-persistent-memory-compression.md)), **multica** ([73](73-multica-managed-agents-platform.md)), **gstack** ([75](75-gstack-garry-tan-claude-code-setup.md)), **everything-claude-code** ([62](62-everything-claude-code.md)), **getcompanion-ai/feynman** ([155](155-feynman-multi-agent-research-harness.md)), **bytedance/deer-flow** ([65](65-deer-flow-bytedance.md), [163](163-deer-flow-revisited-may-2026.md)), **lobehub** ([64](64-lobehub-ai-framework.md), [205](205-lobehub-collaborative-teammate-platform.md)), and the [126](126-frameworks-comparison.md) framework comparison set.

> **Companion.** This file is the *discipline-arc* deep-dive paired with [238](238-karpathy-agentic-engineering-shift.md) (talk-grounded flagship) and [239](239-software-3-0-paradigm.md) (paradigm-grounded). For the broader corpus textbook, see [40](40-harness-engineering-principles.md), [43](43-twelve-harness-patterns.md), [44](44-four-pillars-harness-engineering.md), [144](144-build-your-own-harness.md).

**One-line definition.** **Vibe coding** is the *act* of fluent, prompt-driven coding inside an agent harness; **agentic engineering** is the *discipline* that grows on top of it once eval, verifier, harness, skill, and memory operations are made explicit, named, budgeted, and reviewed — and the May 2026 IICYMI talk's central historical claim is that 2025–2026 is the year that maturation actually happened in public, in repos, in evals, and in product practice.

## Why this maturation matters (the field acquired a real engineering discipline this year)

Before February 2025, two things did not have names. The first was the *fluent prompt-driven coding session* — the experience that, with a good harness around a frontier model, you could complete days of code in hours by *describing what you want* rather than typing what you want. The second was the *discipline of building those harnesses* — the named patterns, evals, and tradecraft. Karpathy's "vibe coding" tweet named the first; the harness-engineering literature ([40](40-harness-engineering-principles.md), [43](43-twelve-harness-patterns.md), [44](44-four-pillars-harness-engineering.md), [144](144-build-your-own-harness.md)) named the second; the May 2026 talk closed the loop by pointing out that **the second is what the first actually depends on**, and that "agentic engineering" is the right name for the second.

The maturation matters because *the same activity* is now sustained by *very different operations* depending on whether the practitioner has crossed the boundary. Pre-discipline vibe coding looks like a magic-trick demo: it works for a session, breaks the next, regresses silently when the base model rotates, and produces code the operator cannot maintain. Post-discipline agentic engineering looks ordinary: it has eval suites, version-pinned base models, harness regression tests, named patterns, on-call rotations, deploy attestations, and a written team manifesto for harness conventions. The difference between the two states is the difference between an engineering discipline and a hobbyist practice — and the entire 2025–2026 evidence record is that the former is winning where it has been adopted.

The maturation also matters because **it gives the field a non-pejorative way to talk about both stances**. Vibe coding is not a sin; agentic engineering is not pretentious. They are phases, and the natural workflow is *vibe-then-engineer*: explore in vibe mode, productionize in agentic-engineering mode. Treating them as adversarial categories — "real engineers don't vibe-code", "true vibe coders never engineer" — misses the point. The discipline emerges *from* the act, and the act is *productionized by* the discipline.

Take the maturation seriously and three things change. **First**, you stop staffing 3.0 work with "AI engineers" generically and start staffing it as either *vibe-mode exploration* or *agentic-engineering production*, with explicit handoffs and promotion criteria between them. **Second**, you adopt **eval-driven 3.0 development** as the equivalent of test-driven development — a regression suite that runs on every prompt/skill/tool change ([21](21-llm-as-judge-trajectory-eval.md), [115](115-evaluating-llm-systems.md), [38](38-claw-eval.md)). **Third**, you write a **team harness manifesto** — a one-page document that names the patterns ([43](43-twelve-harness-patterns.md), [44](44-four-pillars-harness-engineering.md)) you commit to, the verifier discipline you require, the menus you ship with raw prompts, and the review rules for 3.0 changes — and you update it quarterly.

## Problem it solves (the field's gap between fluent demo and durable production)

1. **Vibe coding is real, productive, and undertaught.** The act works — when the harness is good. Pre-2025 the field had no honest name for it and underrated the leverage.
2. **But the same act produces wildly different outcomes** depending on the harness around it. Without naming the discipline, the field could not explain why some teams ship and some don't.
3. **The "AI engineer" job title is too generic.** Companies hire "AI engineers" who turn out to be doing 2.0 work (fine-tunes), 1.0 work (RAG plumbing), 3.0 work (skills + prompts + harness), or LLMOps — four distinct disciplines. The vibe / agentic-engineering split helps disambiguate the 3.0 part.
4. **Evals are still treated as optional in many shops.** Eval-driven 3.0 development is the operational definition of "agentic engineering"; without it, you are still vibe-coding (in production, where it does not belong).
5. **The promotion path from prototype to production is unclear.** Vibe-then-engineer gives an explicit promotion checklist: when does a vibe-coded prototype get the harness, the eval suite, the verifier, the HITL gate, the trace surfacing, and the observability budget?
6. **Founders, investors, and hiring managers need the vocabulary.** Without it, "we built it with AI" is uninterpretable. With it, "we vibe-coded the prototype, agentic-engineered the production system" is auditable.

## Core idea in one paragraph

The activity of using LLMs to build software has two complementary modes. **Vibe coding** is the exploratory mode: a fluent prompt-driven session inside an agent harness, optimized for speed of intent-to-prototype, where the harness silently absorbs eval, retry, format, and verification work. **Agentic engineering** is the production mode: the same activity made explicit, named, budgeted, and reviewed — with first-class evals ([115](115-evaluating-llm-systems.md), [21](21-llm-as-judge-trajectory-eval.md)), first-class verifiers ([11](11-verifier-evaluator-loops.md), [223](223-verifier-and-best-of-n-scaling.md), [242](242-verifiability-bottleneck-and-jagged-skills.md)), first-class memory ([09](09-memory-files.md), [233](233-memory-scaling-for-agents.md)), first-class skills ([04](04-skills.md), [19](19-voyager-skill-libraries.md), [68](68-atomic-skills-scaling-coding-agents.md)), first-class observability ([24](24-observability-tracing.md)), and named patterns ([43](43-twelve-harness-patterns.md), [44](44-four-pillars-harness-engineering.md)). The natural workflow is *vibe-then-engineer*: prototype in vibe mode, promote to agentic-engineering mode through an explicit checklist. The 2025–2026 evidence — high-star repos ([71](71-karpathy-skills-single-file-guardrails.md), [62](62-everything-claude-code.md), [75](75-gstack-garry-tan-claude-code-setup.md)), industry essays (Anthropic, Cognition, HumanLayer), eval benchmarks ([95](95-osworld.md), [96](96-gdpval.md), [101](101-autoresearchbench.md), [38](38-claw-eval.md)), and operational reports ([155](155-feynman-multi-agent-research-harness.md), [165](165-ralph-autonomous-loop.md)) — is that the discipline is now defined enough to teach, and that teams that adopt it ship more reliably than teams that stay in pure vibe mode.

## Mechanism (the maturation arc step by step)

### (a) Phase 0 — Pre-vibe (pre-Feb 2025): "AI features" as 1.0 + 2.0

Before the vibe-coding tweet, the dominant frame was "add an LLM call to a 1.0 codebase" or "fine-tune for our domain" (2.0). Harnesses existed but were not yet named as a category. ReAct ([13](13-react.md)), AutoGPT ([126](126-frameworks-comparison.md)), BabyAGI, and the early LangChain wave were the prototypes. Adoption was uneven; failure was attributed to "the model is not smart enough" rather than to harness design.

### (b) Phase 1 — Vibe coding named (Feb 2025): the act gets a vocabulary

Karpathy's tweet describes a distinct experience: "fully give in to the vibes, embrace exponentials, and forget that the code even exists" while building real things. The tweet sets off the *fluent prompt-driven coding* discourse. Within weeks, the term is overloaded — used to describe everything from one-shot prototyping to multi-week project building — but the term has the right shape and sticks.

What was true and what was missing in Phase 1: the *act* was real (Cursor, Claude Code, Aider, Continue, Cline already shipped harnesses good enough to make the act productive); the *discipline* was missing (most teams shipping vibe-coded prototypes had no eval suite, no verifier, no regression test for prompt drift, no harness manifesto).

### (c) Phase 2 — The harness-engineering literature crystallizes (Feb–Aug 2025)

Through 2025, the corpus's flagship documents arrive: Anthropic's *Effective Harnesses* essay; HumanLayer's *Skill Issue*; Cognition's *Don't Build Multi-Agents*; the Generative Programmer's *Twelve Harness Patterns* ([43](43-twelve-harness-patterns.md)); the *Four Pillars* analysis ([44](44-four-pillars-harness-engineering.md)); the *Components of a Coding Agent* breakdown ([46](46-components-of-coding-agent.md)); the *Dive into Claude Code* reverse-engineering ([29](29-dive-into-claude-code.md)). The discipline acquires named patterns, primary sources, and a vocabulary independent of any single vendor. The *Build Your Own Harness* decision framework ([144](144-build-your-own-harness.md)) and the *Comparing Coding Harnesses* survey ([145](145-comparing-coding-harnesses.md)) make adopt/wrap/fork/write tradeoffs explicit.

### (d) Phase 3 — High-star community artifacts make the discipline portable (Aug 2025–Apr 2026)

The community ships portable instantiations of the discipline:

- **Single-file guardrails:** Karpathy-skills ([71](71-karpathy-skills-single-file-guardrails.md)) — a 60-line CLAUDE.md codifying *Think Before Coding / Simplicity First / Surgical Changes / Goal-Driven Execution*; >71 K stars makes it the most-starred CLAUDE.md artifact and the canonical *minimum viable harness manifesto*.
- **Memory plugins:** claude-mem ([72](72-claude-mem-persistent-memory-compression.md)) — 5 hooks + worker + Chroma + 3-layer progressive disclosure; >65 K stars — *agentic-engineering memory as a plugin*.
- **Multi-skill setups:** gstack ([75](75-gstack-garry-tan-claude-code-setup.md)) — 23 specialist skills + 7-phase sprint; the canonical *agentic-engineering personal stack* template.
- **Bundled ecosystems:** everything-claude-code ([62](62-everything-claude-code.md)) — the curated set of skills, hooks, MCP servers, and conventions.
- **Managed agents:** Multica ([73](73-multica-managed-agents-platform.md)) — team-scale orchestration.
- **Research-agent harnesses:** Feynman ([155](155-feynman-multi-agent-research-harness.md)), DeerFlow 2.0 ([163](163-deer-flow-revisited-may-2026.md)), Paper2Agent ([162](162-paper2agent-reimagining-papers-as-agents.md)), Paper-Researcher-AI-Agent ([161](161-paper-researcher-ai-agent.md)), CrewAI ([164](164-crewai-multi-agent-framework.md)), Ralph ([165](165-ralph-autonomous-loop.md)).

The *act* of vibe coding is now reproducibly assisted by *artifacts* of agentic engineering — a sign that the discipline has crossed from essay to repo.

### (e) Phase 4 — Karpathy's pathologies post (Nov 2025): negative space gets named

The November 2025 X post observes the *failure modes of LLM coding* — wrong-assumption silence, overcomplication, drive-by edits, goal-fluent looping. The Karpathy-skills repo ([71](71-karpathy-skills-single-file-guardrails.md)) operationalizes the four observations into four principles. The discipline now has named anti-patterns to match its named patterns ([43](43-twelve-harness-patterns.md)), the way that *code smells* (Fowler) named anti-patterns to match design patterns (GoF).

### (f) Phase 5 — Memory + skills as trainable substrates (May 2026)

The MEMTIER trilogy ([151](151-memtier-why-flat-memory-breaks-at-72-hours.md), [152](152-memtier-3-tier-architecture-and-retrieval.md), [153](153-memtier-llm-distillation-and-the-three-invariants.md)), Ctx2Skill ([154](154-ctx2skill-self-evolving-context-skills.md)), HeavySkill ([156](156-heavyskill-parallel-reasoning-deliberation.md)), and the May 2026 synthesis ([157](157-may-2026-synthesis-memory-and-skills.md)) jointly establish that **memory and skills are the two trainable substrates of the agent stack**, with retrieval architecture as the binding ceiling and Markdown skill files as the portable artifact. This phase fills in the *what we engineer* answer to "agentic engineering": skills + memory, with prompts/tools/loop as the substrate. The discipline now has a *clear primary work surface*.

### (g) Phase 6 — The talk closes the loop (May 2026)

Karpathy's IICYMI interview names *agentic engineering* as the discipline that has emerged on top of vibe coding. The talk does not introduce new mechanisms; it gives the field its native vocabulary for what it has spent a year doing. The two terms are now in productive contrast: the act is *vibe coding*; the discipline is *agentic engineering*; the workflow is *vibe-then-engineer*; the artifacts are *prompts + skills + tools + memory + evals + verifier + harness*; the deploy unit is *agent-as-installer* ([239](239-software-3-0-paradigm.md)).

### (h) The promotion checklist (vibe → agentic engineering)

A vibe-coded prototype is *agentic-engineering ready* when it has acquired all of the following:

1. **Eval suite** — golden traces + judge prompts + regression probes; runs on prompt/skill/tool change ([115](115-evaluating-llm-systems.md), [21](21-llm-as-judge-trajectory-eval.md), [38](38-claw-eval.md)).
2. **Verifier** — automated cheaper-than-LLM check that the agent can self-call ([11](11-verifier-evaluator-loops.md), [80](80-knowrl.md), [97](97-qwen-prm.md), [223](223-verifier-and-best-of-n-scaling.md), [242](242-verifiability-bottleneck-and-jagged-skills.md)).
3. **Memory** — first-class persistent state with retention rules ([09](09-memory-files.md), [72](72-claude-mem-persistent-memory-compression.md), [151–153 MEMTIER](151-memtier-why-flat-memory-breaks-at-72-hours.md), [181](181-mem0-deep-dive.md)).
4. **Skills** — Markdown skill bundles with progressive disclosure ([04](04-skills.md), [19](19-voyager-skill-libraries.md), [68](68-atomic-skills-scaling-coding-agents.md), [167](167-autoskill-experience-driven-lifelong-learning.md), [168](168-evoskill-coding-agent-skill-discovery.md)).
5. **Trace surfacing + observability** — every step recorded with cost attribution ([24](24-observability-tracing.md), [122](122-explainability-compliance.md)).
6. **HITL gates** — explicit approval points for high-stakes actions ([23](23-human-in-the-loop.md), [143](143-ux-design-for-agentic-systems.md)).
7. **Guardrails + permission modes** — pre/post-tool hooks, prompt-injection defense, scoped permissions ([05](05-hooks.md), [06](06-permission-modes.md), [22](22-guardrails-prompt-injection.md)).
8. **Version-pinned base model** + cross-vendor fallback ([119](119-agent-ready-llm-selection.md), [86](86-frugalgpt.md), [87](87-routellm.md), [147](147-vendor-lock-in.md)).
9. **Written harness manifesto** — one page naming patterns committed to, verifier discipline, eval cadence, deploy gate.
10. **Deploy attestation** — agent-installer ([239](239-software-3-0-paradigm.md)) brings up environment, runs smoke evals, signs report.
11. **On-call rotation** for prompt-driven outages, with runbook ([124](124-agent-level-production-patterns.md), [125](125-system-level-production-patterns.md)).
12. **Compaction and long-horizon strategy** ([08](08-context-compaction.md), [10](10-multi-session-continuity.md), [27](27-horizon-long-horizon-degradation.md), [233](233-memory-scaling-for-agents.md)).

A vibe-coded prototype that lacks more than three of the above is *not yet productionized*; treating it as such causes outages.

### (i) The eval-driven 3.0 development cycle

Eval-driven 3.0 development is to agentic engineering what test-driven development is to 1.0 engineering. The cycle:

1. **Write a failing eval** — a trace, a judge prompt, a verifier check that captures the desired behavior or its absence ([21](21-llm-as-judge-trajectory-eval.md), [115](115-evaluating-llm-systems.md)).
2. **Vibe-code the change** — prompt/skill/tool edit; iterate fluently until the eval passes locally.
3. **Run the regression suite** — full eval pack against the change.
4. **Promote** — once eval-clean, promote to merged with deploy attestation.
5. **Monitor in prod** — every production trace eligible for sample-and-judge.

The cycle preserves the speed of vibe coding (step 2) while gating it on the discipline of agentic engineering (steps 1, 3, 4, 5). This is what "agentic engineering" actually buys.

### (j) Taste and judgment as the new craft

Karpathy's talk treats *taste* as load-bearing. In the discipline, taste shows up as: which prompt structure is robust across model rotations; which skill belongs as Markdown vs as fine-tune; which sub-tree to compile to 1.0 ([93](93-dspy.md), [112](112-constrained-decoding.md)); when to introduce a verifier vs accept the noise; when a multi-agent split is genuine separation vs ceremonial diversity ([98](98-diversity-collapse-mas.md)); when to ship a menu over a raw prompt ([04](04-skills.md), [205](205-lobehub-collaborative-teammate-platform.md)). Taste is not learnable from documentation alone; it is learned by reading and writing harnesses. The corpus is the textbook; the high-star repos are the worked examples; the team manifesto is the practitioner's apprenticeship artifact.

## Empirical anchors and the 2025–2026 evidence record

| Phase signal | Source |
|---|---|
| The act named (Feb 2025) | Karpathy "vibe coding" tweet |
| Industry essays crystallize the discipline | Anthropic *Effective Harnesses*; Cognition *Don't Build Multi-Agents*; HumanLayer *Skill Issue*; BDTechTalks summaries ([40](40-harness-engineering-principles.md)) |
| Pattern catalog | [43](43-twelve-harness-patterns.md), [44](44-four-pillars-harness-engineering.md), [46](46-components-of-coding-agent.md) |
| Decision frameworks | [144](144-build-your-own-harness.md), [145](145-comparing-coding-harnesses.md), [126](126-frameworks-comparison.md) |
| Single-file manifesto | [71](71-karpathy-skills-single-file-guardrails.md) (>71 K stars) |
| Memory plugins | [72](72-claude-mem-persistent-memory-compression.md), [181](181-mem0-deep-dive.md), [183](183-oss-memory-landscape-may-2026.md), [184](184-strongest-memory-techniques-synthesis-may-2026.md) |
| Skill ecosystem | [04](04-skills.md), [19](19-voyager-skill-libraries.md), [68](68-atomic-skills-scaling-coding-agents.md), [154](154-ctx2skill-self-evolving-context-skills.md), [156](156-heavyskill-parallel-reasoning-deliberation.md), [167](167-autoskill-experience-driven-lifelong-learning.md), [168](168-evoskill-coding-agent-skill-discovery.md), [171](171-skill-self-evolution-2026-synthesis.md), [177](177-skills-discovery-curator-strongest-2026-techniques.md) |
| Production patterns | [124](124-agent-level-production-patterns.md), [125](125-system-level-production-patterns.md), [127](127-loan-processing-multi-agent-case-study.md), [150](150-temporal-durable-execution-substrate.md) |
| Observability | [24](24-observability-tracing.md), [122](122-explainability-compliance.md) |
| Evals | [21](21-llm-as-judge-trajectory-eval.md), [38](38-claw-eval.md), [95](95-osworld.md), [96](96-gdpval.md), [101](101-autoresearchbench.md), [115](115-evaluating-llm-systems.md), [34](34-clawbench-live-web-tasks.md) |
| Anti-patterns named | [71](71-karpathy-skills-single-file-guardrails.md) (Karpathy Nov 2025 post) |
| Memory + skills as substrate | [157](157-may-2026-synthesis-memory-and-skills.md) |
| Talk closes the loop | IICYMI May 2026 chapter 15:46 |

The single best aggregate signal is the rate of *pattern-named-and-published* in 2025–2026: more named harness patterns shipped in 18 months than in the prior decade, which is the clearest indicator that a discipline crystallized.

## Variants of the maturation reading

- **The "vibe coding is just hacking" reading.** Treats the term as derisive shorthand for ill-disciplined LLM use. The talk explicitly rejects this — the act is a real productive mode that the discipline should preserve, not eliminate.
- **The "agentic engineering is just LLMOps" reading.** Conflates the discipline with its ops layer. Agentic engineering includes design, taste, skill curation, and verifier choice — *not* just deployment monitoring.
- **The "AI engineering" reading.** Lumps 2.0 (fine-tuning), 3.0 (harness/skills/prompts), and ops together. The vibe / agentic-engineering split is *narrower*: it is the 3.0-rung discipline only ([239](239-software-3-0-paradigm.md)).
- **The "post-discipline vibe coding" trap.** A practitioner who reaches agentic engineering and stops vibe coding has ossified the discipline; the cycle requires fluent exploration to feed the production loop.
- **Cross-cultural variants.** Asian and European harness ecosystems ([56–60 SEA landscape](56-sea-landscape-2026.md), [64-lobehub](64-lobehub-ai-framework.md), [65-deer-flow](65-deer-flow-bytedance.md), [104-glm-5v](104-glm-5v-turbo-native-multimodal-agents.md)) instantiate the same discipline with different vocabulary; the underlying patterns are convergent.

## Failure modes and limitations of the framing

- **Premature productionization.** Forcing all 12 promotion-checklist items on a one-week prototype kills exploration speed. The discipline is for production, not for sketches.
- **Eternal vibe mode.** Teams that never promote prototypes accumulate fragility; outages eventually force the discipline retroactively, at much higher cost.
- **Manifesto without enforcement.** A written harness manifesto with no CI, no review, no on-call runbook is theatre. The discipline lives in CI gates and on-call response, not in documents.
- **Verifier-free agentic engineering.** A team that has memory, skills, traces, observability, and HITL but no verifier ([11](11-verifier-evaluator-loops.md), [223](223-verifier-and-best-of-n-scaling.md), [242](242-verifiability-bottleneck-and-jagged-skills.md)) is doing *vibe coding with monitoring*. Verifier presence is the load-bearing diagnostic.
- **Discipline taken as ladder rather than cycle.** Treating agentic engineering as a one-way "level up" from vibe coding misses the *vibe-then-engineer* cycle. Fluent exploration is required permanently, not phased out.
- **Confusing "agentic" with "multi-agent".** Agentic engineering can be single-agent ([01](01-agent-loop-architecture.md), [13](13-react.md)); multi-agent ([121](121-multi-agent-coordination-patterns.md)) is a *technique inside* the discipline, not its definition. The 2025–2026 evidence ([98](98-diversity-collapse-mas.md), [202](202-multi-agent-multi-hop-reckoning-2026.md), [224](224-multi-agent-parallel-scaling.md)) is that multi-agent helps less than naively expected.

## When to use this framing, when not

**Use it** when: (a) onboarding new engineers — the vibe / agentic-engineering split is the most teachable boundary; (b) building a team manifesto — the promotion checklist is a ready-made template; (c) executive communication — explains why "we built it with AI" is not enough; (d) post-mortem analysis — most prompt-driven outages are *vibe code in production*, and naming that explicitly is the right diagnosis; (e) hiring — clarifies whether the role is exploration (vibe) or productionization (agentic engineering) or both.

**Do not use it** when: (a) you need the per-pattern mechanics (use [43](43-twelve-harness-patterns.md), [44](44-four-pillars-harness-engineering.md), [144](144-build-your-own-harness.md) directly); (b) the conversation is at the 2.0 layer (fine-tuning, RL, distillation — different discipline); (c) safety modeling — agentic-engineering discipline does not by itself address adversarial threat models ([22](22-guardrails-prompt-injection.md), [82](82-poisonedrag.md), [49](49-agents-of-chaos-red-teaming.md)).

## Implications for harness engineering

1. **Adopt vibe-then-engineer as the default workflow.** Two-mode operation: explore in vibe mode, productionize in agentic-engineering mode, with explicit promotion criteria. Document the criteria.
2. **Write a one-page harness manifesto.** Name 5–10 patterns ([43](43-twelve-harness-patterns.md), [44](44-four-pillars-harness-engineering.md)) you commit to, the verifier discipline you require, the menus you ship with raw prompts, and the review rules for 3.0 changes. Update quarterly.
3. **Adopt the 12-item promotion checklist.** Make it a literal CI / pre-merge gate for prompt/skill/tool changes that touch production paths.
4. **Treat eval-driven development as the primary loop.** Tests-first discipline migrated to 3.0 — every new behavior begins as a failing eval ([21](21-llm-as-judge-trajectory-eval.md), [115](115-evaluating-llm-systems.md)).
5. **Stand up the verifier first.** Before adding skills, before tuning prompts, build the verifier. The verifier is the binding axis ([242](242-verifiability-bottleneck-and-jagged-skills.md), [223](223-verifier-and-best-of-n-scaling.md), [232](232-rl-on-reasoning-traces-scaling.md)).
6. **Borrow the high-star manifestos.** The Karpathy-skills CLAUDE.md ([71](71-karpathy-skills-single-file-guardrails.md)) is a 60-line starting point; everything-claude-code ([62](62-everything-claude-code.md)) and gstack ([75](75-gstack-garry-tan-claude-code-setup.md)) are templates for personal stacks. Fork rather than rewrite.
7. **Stand up trace surfacing and observability before scaling.** The discipline depends on visibility; without it, the team is back to vibe coding ([24](24-observability-tracing.md), [122](122-explainability-compliance.md)).
8. **Pair menus with raw prompts.** The discipline includes UX. Every recurring raw prompt grows into a slash-command, a skill, or a UI affordance ([04](04-skills.md), [143](143-ux-design-for-agentic-systems.md), [205](205-lobehub-collaborative-teammate-platform.md)).
9. **Stand up on-call for prompt-driven outages.** Treat prompt drift, base-model rotation, MCP server outage, and skill regression as first-class incident classes with runbooks ([124](124-agent-level-production-patterns.md), [125](125-system-level-production-patterns.md)).
10. **Hold a quarterly "obvious by NOW" review.** The talk's discipline of inventorying contested-then-obvious claims is a hygiene practice — apply it to your team's beliefs about harness shape ([225](225-agent-era-scaling-synthesis.md)).
11. **Track taste explicitly.** Identify which decisions have been *taste-driven* (no eval ground truth) and *log them* — over time the eval suite catches up to the taste, but in the meantime the log is the audit trail ([122](122-explainability-compliance.md)).
12. **Promote skills into 2.0 when they stabilize.** Once a skill is stable, high-volume, and well-evaled, internalize it via fine-tune or RLVR ([116](116-adapter-tuning-lora-peft-for-agents.md), [232](232-rl-on-reasoning-traces-scaling.md), [156](156-heavyskill-parallel-reasoning-deliberation.md)). The 3.0 source becomes the regression suite.

## One-line takeaway for harness designers

**Vibe coding is the act, agentic engineering is the discipline, the workflow is vibe-then-engineer, and the 12-item promotion checklist (eval, verifier, memory, skills, trace, HITL, guardrails, model pinning, manifesto, deploy attestation, on-call, compaction) is what separates a fluent prototype from a system you can run in production without losing sleep.**
