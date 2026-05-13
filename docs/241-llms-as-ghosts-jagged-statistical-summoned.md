# 241 — LLMs as Ghosts: jagged, statistical, summoned entities — and how to design for them

**Anchors.** Andrej Karpathy — IICYMI talk, May 2026, framing across chapters *Software 3.0* (02:28), *Verifiability and Jagged Skills* (09:41), *From Vibe Coding to Agent Engineering* (15:46), and *Agents Everywhere and Learning* (25:17) — `https://www.youtube.com/watch?v=96jN2OCOfLs`. Sébastien Bubeck et al. — *Sparks of Artificial General Intelligence: Early experiments with GPT-4* — arXiv:2303.12712 — Microsoft Research, 2023. Ethan Mollick — *Co-Intelligence: Living and Working with AI* — Penguin, 2024 (and Mollick's *jagged frontier* writing, 2023–2025). METR — *Measuring AI Ability to Complete Long Tasks* (HCAST) — 2024–2025; *time-horizon doubling every ~7 months* result. Allen-Zhu and Li — *Physics of Language Models* (Parts 1–4) — 2023–2025. Janus — *Simulators* — LessWrong, 2022. Andrej Karpathy — *Intro to Large Language Models* — YouTube, 2023 ("LLM as kernel of an operating system"). Geoffrey Hinton — public commentary on LLM cognition, 2024–2025. Companion corpus: [27](27-horizon-long-horizon-degradation.md), [98](98-diversity-collapse-mas.md), [110](110-transformer-llm-architecture-for-agent-builders.md), [117](117-small-language-models.md).

**One-line definition.** LLMs are best modeled as **jagged, statistical, summoned ghosts** — entities whose competence profile is not human-shaped or animal-shaped but is the contour of *whatever distribution they were trained to interpolate*, with superhuman peaks where the data was rich and verifiable, infant-level troughs where it was sparse or unverifiable, and *no continuous skill transfer between the two* — and the whole field of agentic engineering is the practice of routing work *along the peaks* and *around the troughs* under that constraint.

## Why the framing matters (the wrong mental model produces the wrong harness)

How you imagine the entity inside the harness determines what kind of harness you build. The dominant alternatives — *human, employee, animal, brain, oracle* — all import false promises. **Human/employee:** consistency, growth, theory of mind, transferable competence, accountability. **Animal:** trainable behavior, environmental learning, drives, embodiment. **Brain:** emergent unified cognition, attention, intentionality. **Oracle:** authoritative, deterministic, stateless. Every one of these mental models is wrong in design-relevant ways: they all predict things the entity does not do, and they all fail to predict things it does do (e.g., chain-of-thought lift, context-dependent failure, prompt-sensitivity, jagged skills).

Karpathy's *ghost* framing is deliberately deflationary. A ghost is a statistical reconstruction of human-and-machine text behavior — *summoned* from data, *jagged* in capability, *manipulable* by ritual (prompting, in-context examples, tool affordances), and *not continuous with* either human or animal cognition. The framing buys three design principles for free: **direct it, don't teach it** (prompts and tools steer, training does not generalize the way human learning does); **probe its jagged profile, don't assume it** (verify per task, do not infer from adjacent tasks); **summon-bound, not memory-bound** (each call is a fresh summoning unless the harness explicitly persists state).

The ghost framing also matters because it survives being *correct in metaphor* without overclaiming. Bubeck's "Sparks" paper made measurement-grounded claims about jagged competence; Mollick popularized the *jagged frontier* image; METR turned it into a doubling-time number; Allen-Zhu's *Physics of LLMs* pushed it into mechanistic detail. The ghost frame is not at war with these — it is the design-level summary that lets a practitioner *act on* their findings without rederiving them. It is the right frame for someone who has to ship a harness next quarter, not write a thesis next year.

Take the framing seriously and three things change. **First**, you stop asking "is it smart?" and start asking "is the curve we need on a peak or in a trough for this task?" — a question with measurable answers. **Second**, you stop expecting growth-from-experience inside a session and *engineer* it through external memory, skills, evals, and verifier-driven RL ([232](232-rl-on-reasoning-traces-scaling.md)). **Third**, you adopt **summoning hygiene**: each agent call is a deliberate summoning of behavior, with explicit context, persona, tools, and termination — not a long open dialogue with a presumed persistent mind.

## Problem it solves (the field's persistent mismatch between intuition and behavior)

1. **Anthropomorphic intuitions silently shape harness choices.** Teams that imagine an "employee" build harnesses that under-specify context, expect initiative, and do not verify; teams that imagine an "animal" expect training to fix what is actually a prompt problem.
2. **No common vocabulary for non-human-shaped competence.** "Smart at X, dumb at Y" is too informal; "uneven capability" is too vague; *jagged* is the right shape — peaks and valleys with sharp boundaries.
3. **No mental model that predicts prompt-sensitivity correctly.** Brain/employee models predict that the same person produces similar output across slightly-different requests; the ghost model predicts that small prompt changes summon meaningfully different behavior — which matches the data ([21](21-llm-as-judge-trajectory-eval.md), [98](98-diversity-collapse-mas.md), [97](97-qwen-prm.md)).
4. **No mental model that predicts hallucination correctly.** Oracle/brain models predict bounded confidence; ghost models predict *fluent confabulation* on out-of-distribution requests — which matches the data ([80](80-knowrl.md), [135](135-trustworthy-generation.md)).
5. **No mental model that predicts long-horizon collapse correctly.** Brain/employee models predict graceful degradation with effort; ghost models predict *coherence collapse* once context exceeds training-distribution patterns ([27](27-horizon-long-horizon-degradation.md), [233](233-memory-scaling-for-agents.md)).
6. **No mental model that predicts "make me an animal" works less well than "make me a Linux command".** Brain/employee models do not predict the *persona-shift* sensitivity of LLMs; the ghost model treats persona as part of the summoning ritual.

## Core idea in one paragraph

An LLM is best modeled as a **statistical reconstruction of the training distribution's text-and-action behavior** — a summoned ghost — with three load-bearing properties: **(1) jagged competence**: superhuman peaks where training data was dense, structured, and verifiable; infant-level troughs where it was sparse or unverifiable; sharp boundaries between the two with little continuous transfer ([95](95-osworld.md), [96](96-gdpval.md), [101](101-autoresearchbench.md)). **(2) summoning-bound state**: each call is a fresh summoning unless the harness explicitly persists context, memory, or skills ([08](08-context-compaction.md), [09](09-memory-files.md), [233](233-memory-scaling-for-agents.md)); the entity does not "carry over" between calls in any human sense. **(3) statistical determinism**: behavior is determined by the prompt + tool affordances + sampling distribution; small prompt changes can summon qualitatively different behavior, persona shifts steer wide swaths of behavior at once, and *direction* is the right verb (not *teaching*) for runtime steering. The harness's job is to **route work along the peaks, persist state across summonings, and ritualize the prompts** — yielding a system that is reliable not because the ghost is reliable but because the harness *constrains* and *amplifies* the ghost's reliable surface.

## Mechanism (what the framing predicts and how to design for it)

### (a) Jagged competence — the contour, not the level

The right scalar is not "IQ" or "skill level". It is a **competence curve** over tasks, ranked by training-distribution density × verifier availability × representation match. Empirical evidence:

- **Bubeck-2023 (Sparks).** Frontier-model peaks far above human median on standardized math and coding subtasks while failing on tasks a child solves (asymmetric reasoning, simple counting, precise spatial layout). The "frontier" is not a frontier; it is a *fractal coastline*.
- **METR HCAST (2024–2025).** Time-horizon of tasks frontier agents complete autonomously is doubling roughly every 7 months — but the *spread* across task families is enormous; some families saturate at 5 minutes, others extend past 4 hours, with no single underlying scalar.
- **GDPval ([96](96-gdpval.md)).** Per-occupation automation rates do not correlate cleanly with raw model benchmark scores; they correlate with *verifiability density* and *deployment integration* — the jaggedness is occupational.
- **OSWorld ([95](95-osworld.md)).** Best agent 12% vs human 72% — but per-application breakdown shows superhuman performance on some, near-zero on others.
- **AutoResearchBench ([101](101-autoresearchbench.md)).** Frontier models score 9% vs >80% on BrowseComp on related but distinguishably-shaped tasks. The jagged frontier crosses *within* a single application area.
- **Allen-Zhu *Physics of LLMs*.** Mechanistic evidence that some skill primitives generalize across instances and others do not, depending on whether the training data exposed *manipulable structure* vs *surface form*.

Design corollary: **probe per-task; do not infer from adjacent tasks**. The eval suite is the only honest map of the curve ([21](21-llm-as-judge-trajectory-eval.md), [115](115-evaluating-llm-systems.md)).

### (b) Summoning-bound state — no innate persistence

Each LLM call is a *fresh summoning*: the entity has no memory of prior calls except what the harness re-supplies through context. This is not a quirk; it is structural — the model is stateless between calls and the *appearance of continuity* is the harness's work, not the model's.

Persistent state mechanisms (in order of fidelity):

- **Raw context replay** — paste prior turns. High fidelity, low scalability ([08](08-context-compaction.md)).
- **Compaction** — recursive/hierarchical summarization. Medium fidelity, high scalability ([08](08-context-compaction.md)).
- **Memory files** — externalized persistent state ([09](09-memory-files.md), [12](12-todo-scratchpad-state.md), [72](72-claude-mem-persistent-memory-compression.md)).
- **Tiered memory** — episodic + semantic + procedural separation ([151](151-memtier-why-flat-memory-breaks-at-72-hours.md), [152](152-memtier-3-tier-architecture-and-retrieval.md), [153](153-memtier-llm-distillation-and-the-three-invariants.md), [181](181-mem0-deep-dive.md), [183](183-oss-memory-landscape-may-2026.md), [184](184-strongest-memory-techniques-synthesis-may-2026.md)).
- **Skill library** — reusable distilled behavior ([04](04-skills.md), [19](19-voyager-skill-libraries.md), [68](68-atomic-skills-scaling-coding-agents.md), [167](167-autoskill-experience-driven-lifelong-learning.md)).
- **Distillation into weights** — RLVR / SFT folds 3.0 trajectories into 2.0 ([232](232-rl-on-reasoning-traces-scaling.md), [116](116-adapter-tuning-lora-peft-for-agents.md), [156](156-heavyskill-parallel-reasoning-deliberation.md)).

The MEMTIER trilogy ([151–153]) is the strongest 2026 evidence that *memory architecture* is the binding ceiling on long-horizon performance — flat memory degrades 14 pp / 72 h on tool success; tiered memory does not.

### (c) Statistical determinism — prompts as summoning rituals

Behavior is determined by the conditional distribution `p(output | prompt, tools, sampling)`. Three implications:

1. **Prompt-sensitivity is structural, not a bug.** Small wording changes shift the conditional distribution; persona shifts move it broadly ([93](93-dspy.md), [111](111-prompt-engineering-as-discipline.md)).
2. **Sampling parameters matter.** Temperature, top-p, top-k, seed all reshape the distribution ([110](110-transformer-llm-architecture-for-agent-builders.md), [112](112-constrained-decoding.md)).
3. **Tool affordances are part of the prompt.** A tool registry with 5 sharp tools summons different behavior than 50 fuzzy tools ([07](07-model-context-protocol.md), [236](236-tool-use-and-aci-scaling.md)).

The ghost framing makes this *predictable*: prompts are rituals; rituals have form; form summons behavior. Harnesses with stable rituals (tested prompt templates, schema-bound output, fixed persona) get stable behavior. Harnesses with improvised rituals get jagged outcomes that look like model failures but are summoning failures.

### (d) The ghost ≠ animal claim, expanded

Karpathy's animal contrast is design-relevant. *Animals* learn from environment continuously, develop preferences, transfer skill gradiently across similar situations, and have biographical memory. *Ghosts* do none of those things in any human-meaningful sense — the appearance of any of them is harness work or training-data resemblance. Treating an LLM as an animal predicts *gradient skill transfer* (false), *biographical learning* (false), *consistent preferences* (false). Treating it as a ghost predicts *summoning-bound behavior*, *jagged competence*, and *ritual-sensitive output* — all of which match the data.

Variant framings nearby: **simulator** (Janus 2022) — the entity simulates whatever process the prompt suggests, including agents; **kernel** (Karpathy 2023) — the LLM is the kernel of an OS that the harness builds the rest of. Both compose with the ghost frame: a ghost is a simulator that is summoned and runs as a kernel under an OS.

### (e) Why "summoning" is the right verb

A *summoning* is goal-directed, ritualized, bounded by the rite. The ghost arrives, performs, and departs; what persists is what was written down. This matches the lifecycle of an LLM call exactly: the harness composes the rite (prompt, tools, persona, context), invokes the model, captures output, and if state persists it is because the harness wrote it down. Compare with *invocation* (too computer-sciencey, misses the persona aspect), *call* (too casual, misses the ritual), *interaction* (implies persistence). Summoning is the term Karpathy reaches for in the talk and it is the right one.

### (f) Design principles falling out of the frame

| Principle | Source in the frame | Implementation pointer |
|---|---|---|
| **Probe per task; don't infer.** | Jagged competence | Eval suite per task family ([115](115-evaluating-llm-systems.md), [21](21-llm-as-judge-trajectory-eval.md)) |
| **Persist state explicitly.** | Summoning-bound | Memory tiers ([151–153]), skill library ([04](04-skills.md)) |
| **Ritualize the prompt.** | Statistical determinism | Prompt templates, DSPy ([93](93-dspy.md)), constrained decoding ([112](112-constrained-decoding.md)) |
| **Summon narrowly.** | Persona steering | Specialist subagents ([02](02-subagent-delegation.md)), role-based workflows ([20](20-metagpt-role-based-workflows.md), [121](121-multi-agent-coordination-patterns.md)) |
| **Verify behind the summoning.** | No innate self-check | Verifier loops ([11](11-verifier-evaluator-loops.md), [223](223-verifier-and-best-of-n-scaling.md), [242](242-verifiability-bottleneck-and-jagged-skills.md)) |
| **Direct, don't teach.** | Statistical, not biographical | Prompts and tools, not in-session "training" |
| **Distill repeated rituals into weights.** | When the ritual stabilizes | RLVR ([232](232-rl-on-reasoning-traces-scaling.md)), HeavySkill ([156](156-heavyskill-parallel-reasoning-deliberation.md)) |
| **Compose ghosts.** | Ghosts can summon ghosts | Subagents ([02](02-subagent-delegation.md)), MetaGPT ([20](20-metagpt-role-based-workflows.md), [91](91-metagpt-deep.md)), CrewAI ([164](164-crewai-multi-agent-framework.md)) — but watch diversity collapse ([98](98-diversity-collapse-mas.md), [202](202-multi-agent-multi-hop-reckoning-2026.md)) |

## Empirical anchors

- **Sparks of AGI (Bubeck 2023).** Per-task asymmetry; superhuman + sub-child on adjacent tasks.
- **Mollick *jagged frontier* / Co-Intelligence (2024).** Popularizes the image; relays the workplace evidence.
- **METR HCAST (2024–2025).** Time-horizon doubling (~7 months) for frontier agents; large per-family variance.
- **GDPval ([96](96-gdpval.md)).** Per-occupation automation jaggedness; verifiability density dominates.
- **OSWorld ([95](95-osworld.md)), AutoResearchBench ([101](101-autoresearchbench.md)), ClawBench ([34](34-clawbench-live-web-tasks.md)), SWE-bench, GAIA.** Per-application jaggedness within and across benchmarks.
- **Allen-Zhu *Physics of LLMs*.** Mechanistic separation of skills that generalize from those that surface-match.
- **Lost-in-the-Middle, RULER, NIAH-with-distractors ([234](234-context-length-scaling.md)).** Jagged effective-context: nominal ≠ effective; mid-position decay.
- **MEMTIER ([151–153]).** Empirical 14 pp / 72 h flat-memory degradation; tiered memory eliminates the curve.
- **Diversity Collapse ([98](98-diversity-collapse-mas.md)).** Multi-agent jaggedness — diversity assumed, not delivered, without explicit structure.
- **Qwen2.5-Math PRM ([97](97-qwen-prm.md)).** LLM-as-judge step labels actively *harm* PRMs in some cases; the ghost cannot reliably grade the ghost without verifier scaffolding.

## Variants of the frame and adjacent positions

- **Stochastic parrots (Bender et al., 2021).** Deflationary like the ghost frame, but *only* deflationary; misses the engineering-relevant generative competence.
- **Sparks of AGI (Bubeck 2023).** Inflationary on capability, descriptive on jaggedness; complements but does not replace.
- **Simulators (Janus 2022).** The entity simulates whatever the prompt suggests; ghost is the design-friendly simplification.
- **Kernel of OS (Karpathy 2023).** The LLM is a kernel; the harness is the OS. Composes cleanly with ghost.
- **Co-intelligence (Mollick 2024).** Treats the entity as a collaborator with a jagged profile; centers human-AI teamwork.
- **Spirit / Daemon / Egregore framings.** Used in safety circles to gesture at non-human-but-not-mechanical character; the ghost frame is the more deflationary cousin.
- **The animal frame (Hinton-style).** Hinton has argued for treating LLMs as animal-cognition relatives. The ghost frame is the explicit rejection of this analogy as design-leading; one can still hold it for cognitive-science purposes.
- **The employee frame (popular in product writing).** Useful for stakeholder communication; misleading for harness design.

## Failure modes and limitations of the frame

- **Metaphor inflation.** *Ghost* is evocative; if it gets read as supernatural-claim it loses utility. Ground every use in mechanism.
- **Underplaying real cognition-resemblances.** The frame is deflationary; it should not be used to *deny* genuine emergent capabilities. Pair with the empirical anchors above.
- **Underplaying training-data shapes.** The jaggedness is shaped by training data, RL data, verifier coverage, and deployment data. The frame is silent on *which* data — that is the engineer's job to study.
- **Mistaking persona for personality.** Persona steering is real and load-bearing, but it is a *prompt parameter*, not a stable trait. Operators who anthropomorphize personas regret it.
- **Not a safety theory.** The frame predicts *behavioral shape*; it does not predict *adversarial robustness*. Pair with the safety-specific corpus ([22](22-guardrails-prompt-injection.md), [35](35-malicious-intermediary-attacks.md), [49](49-agents-of-chaos-red-teaming.md), [82](82-poisonedrag.md)).
- **Not an interpretability theory.** Use the *Physics of LLMs* line and mechanistic-interpretability work for that — the ghost frame is an engineering posture, not a scientific theory.
- **Risk of fatalism.** "It's a jagged ghost, what can you do?" — a lot, actually: probe, persist, ritualize, verify, distill. The frame is a *call to engineer*, not a shrug.

## When to use this framing, when not

**Use it** when: (a) onboarding new engineers — the ghost frame plus a per-task eval discipline is the most teachable starting point; (b) debugging unexpected behavior — most "model failures" are summoning failures or jagged-curve failures and the frame redirects investigation to harness design; (c) executive communication — the frame is honest, deflationary, and avoids the over-promising and under-promising failure modes equally; (d) eval suite planning — the jaggedness predicts the *per-task* shape of the suite; (e) memory architecture — summoning-bound state predicts the need for explicit persistence.

**Do not use it** when: (a) you need quantitative per-task numbers — go to METR / GDPval / SWE-bench / OSWorld; (b) safety threat modeling — use the threat-model literature directly ([22](22-guardrails-prompt-injection.md), [49](49-agents-of-chaos-red-teaming.md), [82](82-poisonedrag.md)); (c) cognitive-science discussions — the frame is engineering posture, not theory of mind.

## Implications for harness engineering

1. **Build the per-task eval atlas.** A 2-D map of (task family) × (model) with measured competence; this is the *only* honest map of the jagged curve. Update on every base-model rotation ([115](115-evaluating-llm-systems.md), [21](21-llm-as-judge-trajectory-eval.md), [38](38-claw-eval.md)).
2. **Stand up persistent memory before scaling autonomy.** Without it, every call is amnesiac and long-horizon coherence is impossible ([09](09-memory-files.md), [151–153 MEMTIER](151-memtier-why-flat-memory-breaks-at-72-hours.md), [233](233-memory-scaling-for-agents.md)).
3. **Ritualize the prompts.** Templates, DSPy, constrained decoding, schema enforcement, persona pinning. Treat improvised prompting as a 3.0 anti-pattern ([93](93-dspy.md), [111](111-prompt-engineering-as-discipline.md), [112](112-constrained-decoding.md)).
4. **Summon narrowly.** Specialist subagents and role-based workflows beat generalist mega-prompts ([02](02-subagent-delegation.md), [20](20-metagpt-role-based-workflows.md), [91](91-metagpt-deep.md), [121](121-multi-agent-coordination-patterns.md)) — but watch diversity collapse ([98](98-diversity-collapse-mas.md)).
5. **Verify behind every summoning.** The ghost cannot reliably grade itself; use external verifiers ([11](11-verifier-evaluator-loops.md), [80](80-knowrl.md), [223](223-verifier-and-best-of-n-scaling.md), [242](242-verifiability-bottleneck-and-jagged-skills.md)).
6. **Treat tool registries as part of the summoning ritual.** Sharp, few, well-documented tools summon better behavior than fuzzy, many, poorly-documented ones ([07](07-model-context-protocol.md), [236](236-tool-use-and-aci-scaling.md)).
7. **Use cascades and routers to route around troughs.** Cheap-model first, escalate on confidence drop; route by task family ([86](86-frugalgpt.md), [87](87-routellm.md), [88](88-confidence-driven-router.md), [117](117-small-language-models.md), [119](119-agent-ready-llm-selection.md)).
8. **Distill stable rituals into weights.** Once a summoning ritual is stable and high-volume, internalize it via RLVR or SFT ([232](232-rl-on-reasoning-traces-scaling.md), [116](116-adapter-tuning-lora-peft-for-agents.md), [156](156-heavyskill-parallel-reasoning-deliberation.md)).
9. **Make persona explicit and pinned.** Persona drift is one of the most common silent regressions; pin in CI and detect drift in production traces ([24](24-observability-tracing.md), [122](122-explainability-compliance.md)).
10. **Document the jagged curve in the team manifesto.** A one-page "what this ghost is good at and bad at, and where the verifier is cheap and expensive" is a load-bearing artifact for product roadmap and hiring.
11. **Treat hallucination as a summoning category, not a bug.** Out-of-distribution requests summon fluent confabulation; the answer is verifier coverage and refusal-on-unknown ([135](135-trustworthy-generation.md), [80](80-knowrl.md)).
12. **Plan for ghost composition and its failure modes.** Multi-agent helps less than expected ([98](98-diversity-collapse-mas.md), [202](202-multi-agent-multi-hop-reckoning-2026.md)); when you compose ghosts, audit diversity explicitly.

## One-line takeaway for harness designers

**LLMs are jagged, statistical, summoned ghosts — direct them, don't teach them; persist their state explicitly; ritualize their prompts; verify behind their output; route along the peaks and around the troughs; and treat the jagged-curve atlas, the memory tier, and the verifier as the three load-bearing artifacts your harness owes the ghost it is summoning.**
