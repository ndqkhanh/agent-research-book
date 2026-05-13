# 157 — May 2026 Synthesis: Memory and Skills as the Two Trainable Substrates of the Agent Stack

**Sources synthesised.** [151–153 — MEMTIER trilogy](151-memtier-why-flat-memory-breaks-at-72-hours.md); [154 — Ctx2Skill](154-ctx2skill-self-evolving-context-skills.md); [155 — Feynman multi-agent research harness](155-feynman-multi-agent-research-harness.md); [156 — HeavySkill](156-heavyskill-parallel-reasoning-deliberation.md). Plus the prior canon (chapters 01–150) for grounding.

**One-line thesis.** Read together, the four May-2026 artifacts redraw the harness/model boundary. The harness is now best described as *file-based glue around two trainable substrates — memory and skills — with the LLM as an interchangeable runtime*. Memory architectures (MEMTIER) determine how much an agent can know across long horizons; skill artifacts (Ctx2Skill, HeavySkill) determine how well it can reason within those horizons; multi-agent harness patterns (Feynman) determine how well it can coordinate when single-agent reasoning is insufficient. Of these, MEMTIER demonstrates that **retrieval architecture, not model scale, is the binding ceiling** for memory; HeavySkill demonstrates that **the inner reasoning skill is RLVR-trainable into model weights**; Ctx2Skill demonstrates that **skills can be constructed via adversarial self-play without external feedback**; and Feynman demonstrates that **file-based handoff with mandatory verification is the operational discipline** that ties everything together. The synthesis: the next 12–24 months of agent engineering belong to teams who can produce *high-quality memory architectures and high-quality skill artifacts*, not teams who can spend their way to the biggest model.

This chapter is the *synthesis* — the cross-cutting argument that connects the four artifacts. The individual chapters do the technical work; this chapter does the conceptual work of pulling them together and naming what they collectively imply.

## The shape of the May-2026 evidence

To set up the synthesis, four observations across the four artifacts are worth restating up front:

| Artifact | Domain | Empirical headline | Architectural claim |
|----------|--------|---------------------|----------------------|
| MEMTIER | Long-running agent memory | Tool-success degrades 14pp / 72h on flat memory; architecture lifts LongMemEval-S 0.050 → 0.382 with a 7B model on a 6 GB GPU | BM25 retrieval architecture, not the model and not the weights, is the binding ceiling — a *three-layer invariance* finding |
| Ctx2Skill | Context learning | +5.4 / +4.6 / +3.2 absolute gain on GPT-4.1 / GPT-5.1 / GPT-5.2 on CL-bench; GPT-4.1 + skills (16.5%) beats Gemini 3 Pro alone (15.8%) | Adversarial self-play (Challenger / Reasoner / Judge) with Cross-Time Replay produces portable skills *without external feedback* |
| Feynman | Open-source research harness | 6.6K stars; 4 subagents × 19 skills × 12 prompts; production-deployed; mandatory source-grounding | Multi-agent harness operationalised as *Markdown subagents and Markdown skills with file-based handoffs and a structurally separate Verifier pass* |
| HeavySkill | Test-time scaling skill | GPT-OSS-20B 69.7 → 85.5% on LiveCodeBench; R1-Distill-Qwen-32B 35.7 → 69.3% on IFEval; HM@4 frequently approaches Pass@K | Parallel-reasoning + sequential-deliberation is *one inner skill* — extractable into a Markdown skill file and trainable into model weights via RLVR |

Four different questions, four very different methods, four different empirical communities. But the common thread is *what changes when you stop conflating "harness work" with "orchestration code" and start treating durable artifacts (memory, skills) as the load-bearing pieces*.

## The migration thesis

The deepest thread connecting the four papers is what I will call **the migration thesis**:

> Capabilities continuously migrate from the harness layer down into the model.
>
> Whatever capability the harness handles today by orchestration glue, RLVR will move into the model in 6–18 months.

This is not a new observation — Karpathy framed it pithily ([71](71-karpathy-skills-single-file-guardrails.md)), and the architecture surveys ([66-meta-harness-landscape](66-meta-harness-landscape.md), [56-sea-landscape-2026](56-sea-landscape-2026.md)) document the historical pattern. But the May-2026 papers give it concrete teeth.

HeavySkill is the clearest example. Three years ago, "use multiple LLM calls and aggregate" was a bespoke pipeline you wrote in Python. Two years ago, "spawn parallel subagents and deliberate" was orchestration code in Claude Code, OpenClaw, Hermes. Today, HeavySkill says: *write the workflow as a Markdown skill file*, and any sufficient LM follows it natively. Tomorrow (HeavySkill's RLVR section), the skill is internalised into model weights — and the orchestration disappears into the model.

Ctx2Skill is the same migration applied to context learning. Yesterday: build a custom RAG pipeline for each new document. Today: run Ctx2Skill once, get a Markdown skill file, prepend to the prompt. Tomorrow: skills as RL training data, internalised into model weights.

The migration runs in only one direction. Every capability that gets cleanly distilled into a skill file becomes a candidate for RLVR internalisation. Every capability that gets internalised stops being a harness differentiator.

So the question for harness builders is not "what cleverness can I bake into my orchestration code?" It is: *what capabilities are not yet skills, and can I be the one to skillify them first?*

## What does *not* migrate

Three things, plausibly, do not migrate down into the model:

1. **State across long horizons.** A model cannot durably remember a project's facts from a week ago. Memory architectures live in the harness layer (or the substrate layer below it). MEMTIER explicitly addresses this. Even with bigger models, you cannot internalise *project-specific durable knowledge* into weights — that knowledge is too volatile and too specific to be a training target.
2. **External system integration.** Tool calls, MCP servers, file I/O, network requests, APIs. The model can choose to call a tool, but the tool itself lives outside the model. Skills can specify tool-use protocols, but the tools remain external.
3. **Operational discipline.** Source-grounding (Feynman's Verifier), verification gates, plan artifacts, slug-keyed filesystem, lab notebooks, audit trails, permission layers. These are organisational/policy commitments enforced by the harness, not capabilities of the model.

The *non-migrating* layer is what real harness products will be made of. Memory architecture, integration glue, operational discipline — that's the sustained moat. Reasoning lift is leased from whatever model you happen to be wrapping this season.

## The two-substrate thesis

If we take the migration thesis seriously, what survives in the harness layer? Two things:

### Substrate 1 — Memory

Memory is the *durable record of what has happened and what is true*. It is the substrate that lets agents survive across sessions, across compactions, across crashes, across model upgrades. MEMTIER is the canonical formal statement.

Properties of memory as a substrate:

- **Durable**: persists past the model's context window and past process restarts.
- **Structured**: encodes relations (entity vs incidental co-occurrence — Mode 3 from MEMTIER), time (timestamps, decay), causality (cognitive weight from outcomes).
- **Tiered**: episodic (per-session events) vs semantic (durable facts) vs procedural (skills).
- **Multi-agent-aware**: agent-private episodic, project-shared semantic.
- **Closed-loop**: outcomes update memory quality (cognitive weight; consolidation daemon).

MEMTIER's three-layer invariance is the most important diagnostic in the paper precisely because it tells you *which axis of memory work matters*. Scaling the generator doesn't help. Tuning retrieval weights doesn't help. *Architecture* helps. Specifically: tiered structure, two-stage scoping, LLM-distilled semantic facts, attribution loops.

### Substrate 2 — Skills

Skills are the *portable specifications of how to do things*. They are Markdown files that prepend to the system prompt, encoding rules, procedures, decision criteria, output formats.

Properties of skills as a substrate:

- **Portable across models**: a skill works across Claude / GPT / DeepSeek / Qwen with some quality variation. Ctx2Skill empirically demonstrates this transfer (Table 5 in the paper: GPT-4.1 with GPT-5.1 skills vs same-model skills).
- **Editable as Markdown**: human-readable, version-controllable, peer-reviewable. Feynman's `.feynman/agents/*.md` files demonstrate this in production.
- **Composable**: multiple skills can be loaded into the same prompt; they layer.
- **Constructible automatically**: Ctx2Skill from a context document; HeavySkill distilled from a workflow; Voyager-style skills from interaction traces.
- **Trainable into weights**: HeavySkill's RLVR experiments show skills can be internalised into model parameters with verifiable rewards.

Skills are the asset class of the next phase. Today's harness wins are tomorrow's skill files.

### Why both substrates matter

A common mistake is to think one substrate is a substitute for the other. Memory and skills are *complements*, not alternatives:

- Skills tell the agent *how to do* something.
- Memory tells the agent *what is true* and *what has happened*.

A coding agent without memory cannot recall the project's coding conventions from last week. A coding agent without skills cannot follow the test-then-fix protocol you have specified. Both are needed.

The combination shows up directly in Feynman: skills for *capabilities* (deep-research, peer-review, replicate); memory for *continuity* (CHANGELOG.md as lab notebook, slug-keyed plan artifacts, session logs). Same pattern in production-grade Claude Code deployments: SKILL.md files for capabilities; CLAUDE.md plus memory directories for project state.

## Architecture, not model, is the binding constraint — at multiple layers

MEMTIER's three-layer invariance — *neither model scale nor learned weights nor RL training moves performance significantly within BM25 retrieval architecture* — is a diagnostic that, when you look across the four papers, generalises.

Each paper finds an analogous *architecture-not-model* result in its own domain:

| Paper | Architecture that bound performance | What did NOT move performance |
|-------|--------------------------------------|--------------------------------|
| MEMTIER | BM25 retrieval (Phase 1/2) | Model size (Qwen 7B ≈ DeepSeek 284B); learned retrieval weights |
| Ctx2Skill | Self-play loop with Cross-Time Replay | Skipping CTR (drops 1.8–2.8 pp); single-pass extraction (Prompting baseline) |
| HeavySkill | Two-stage parallel + deliberation | Best-of-N alone (V@K < HM@K consistently); the choice of deliberator model relies more on instruction-following than reasoning |
| Feynman | Multi-agent + Verifier as separate pass | A single super-prompt for everything would skip the verification gate |

The pattern: across these four substrate-level papers, *the architectural choice of what mechanism to use* produces a much larger lift than *the model used to power that mechanism*.

This is the second-order implication of the migration thesis. Capabilities move into the model; *architectures stay in the harness*. Choose your architecture deliberately because that's where the leverage is.

## The artifact lifecycle: skill construction → skill use → skill internalisation

Putting Ctx2Skill and HeavySkill together produces a clean three-stage lifecycle for any procedural capability:

```
Stage 1 — Discovery
    Adversarial self-play (Ctx2Skill)
    ─OR─
    Workflow distillation (HeavySkill workflow → skill)
    ─OR─
    Interaction-trace extraction (Voyager, [19](19-voyager-skill-libraries.md))
    ↓
    Skill file (Markdown, frontmatter, prepended to system prompt)

Stage 2 — Use
    Skill is loaded into prompt at inference time.
    Model follows the skill in-context.
    Cost: per-query token overhead (skill file size).

Stage 3 — Internalisation (optional)
    Skill traces become RLVR training data.
    Reward = verifiable outcome of the skill (correctness, format, tool-call success).
    Model is trained to follow the skill *without the file* in prompt.
    Cost: training run; benefit: lower per-query overhead, baseline capability.
```

This lifecycle is the productionisation pattern for skill-based agent engineering. Today, most teams operate in Stage 2: write skills by hand, deploy. Frontier labs are starting to operate in Stage 3 for high-value skills (GPT-5 Thinking, Claude 4.5 Thinking, Gemini 3 Thinking, DeepSeek V3.2 Thinking — these are not just "better at reasoning", they are *parameterised heavy-thinking-aware* models).

Stage 1 is where Ctx2Skill and HeavySkill innovate. Ctx2Skill closes the *no external feedback* gap with adversarial self-play; HeavySkill closes the *workflow-not-skill* gap with explicit distillation of the workflow into a skill file plus RLVR validation.

For builders, the practical implication is: **start writing skill files**. Do not wait for skill-file authoring tools — Markdown is the tool. Treat skill files as first-class engineering artifacts, version-controlled in git, peer-reviewed, tested.

## File-based handoffs and the harness as glue

Feynman's defining operational pattern — *file-based handoffs between subagents* — is the answer to a question the harness-engineering canon has been wrestling with for two years: *how do you coordinate multiple subagents without choking the parent's context?*

The pattern:
- Subagents write artifacts to disk under a per-run slug.
- Subagents return one-line summaries plus paths to artifacts.
- The parent reads selectively from those paths.
- Plan artifacts (`outputs/.plans/<slug>.md`) are *the working memory of the run*, not a static outline.
- A workspace lab notebook (`CHANGELOG.md`) records chronological state.

This is the multi-agent equivalent of [12-todo-scratchpad-state](12-todo-scratchpad-state.md): externalise your state to durable files so the agent's attention does not have to carry it.

When you combine file-based handoffs with MEMTIER-style memory architecture, you get the substrate-level architecture for a long-horizon multi-agent system:

```
┌─────────────┐
│ Lead agent  │  reads plan, dispatches subagents, synthesises
└─────────────┘
       ↓ subagent dispatch with paths
┌─────────────────────────────────────────────────┐
│ Subagent ledgers (agent-private episodic JSONL) │
└─────────────────────────────────────────────────┘
       ↓ async consolidation
┌─────────────────────────────────────────────────┐
│ Project semantic tier (LLM-distilled facts)     │
└─────────────────────────────────────────────────┘
       ↓ retrieval at next prompt build
       ↑ updated cognitive weights at agent_end
       ↑ provenance + verification at every output
```

Feynman has the *file-based handoff* layer. MEMTIER has the *durable structured memory* layer. Together they cover the persistent-state needs of a long-horizon multi-agent harness. Today, no single open-source project ships both — but the recipe is in plain sight.

## Verification as a structural primitive

Three of the four papers exhibit *verification as a separate structural pass* — not a guideline, not a self-check, but a distinct phase with the authority to gate or modify outputs.

- **Feynman**: the Verifier subagent is a separate role with its own output file (`cited.md`), with the explicit authority to remove unsourced claims. *"Refuse fake certainty. Do not use words like 'verified', 'confirmed', or 'reproduced' unless underlying evidence exists."*
- **Ctx2Skill**: the Judge is a separate frozen-LM role that produces per-rubric binary verdicts and routes outcomes to per-side skill updates. The whole self-play loop is *driven* by the Judge's verdicts.
- **HeavySkill**: the deliberator is a *re-reasoner* that critically evaluates each parallel trajectory, identifies logical errors, and synthesises a final answer — explicitly forbidden from naive concatenation or majority voting.

The pattern: when you want output integrity, *do not bundle generation and verification into a single LLM call*. Generate first; verify in a separate pass; allow the verification pass to alter or remove generation outputs.

This is the operational discipline that closes the "LLMs confidently say wrong things" failure mode. Not by making the LLM less confident — that does not work — but by inserting a structural step whose only job is to check.

For harness builders: *make verification a subagent, not a guideline*. Whatever your domain (coding, research, customer support, ops), there is a verification step that should be separated out. That separation is the discipline.

## Empirical patterns that recur across the four papers

A few empirical patterns that show up in multiple papers and seem to generalise:

### 1. Precision over coverage

MEMTIER: LLM-distilled semantic facts (3.1 facts/q) beat heuristic-extracted (509 facts/q) by 51× on F1 — *fewer facts, higher precision, dramatic gain*.

HeavySkill: Max-Length trajectory selection performs worst; Max-Answer-Number (consensus pre-filtering) performs best — *higher precision in the candidate set, better deliberation outcomes*.

Ctx2Skill: cross-case Proposer synthesis (one diagnosis covering many failures) beats per-case patches — *precision compounds, redundancy rots*.

Feynman: the Verifier removes unsourced claims rather than adding speculative citations — *precision is preserved by removing, not by inflating*.

The cross-paper takeaway: **for memory, retrieval, skill construction, and output verification, precision compounds and coverage without precision is poison.** Default to a small, precise artifact and grow only when growing keeps precision.

### 2. Architecture is the ceiling

Already discussed under the migration thesis. Each paper finds an architecture-level constraint that cannot be remediated by scaling models, tuning weights, or training harder.

The cross-paper takeaway: **diagnose your architecture before you diagnose your model.** The most expensive mistake in agent engineering is throwing model upgrades at architectural ceilings.

### 3. File-based externalisation beats in-context tracking

MEMTIER: episodic JSONL files, semantic tier, daily files. State lives in files.

Feynman: plan artifacts, research files, drafts, cited files, provenance sidecars, lab notebook. State lives in files.

Ctx2Skill: skills are Markdown files. State lives in files.

HeavySkill: the skill file is a Markdown document. The deliberation is an LLM call against a serialised cache. State lives in files.

The cross-paper takeaway: **for any state that must persist beyond a single LLM call, externalise it to a file**. Do not rely on context tracking. The cost of writing a file is negligible; the cost of context loss is potentially total.

### 4. Multi-pass over single-pass

Ctx2Skill: N=5 self-play iterations beat single-pass Prompting.

HeavySkill: Stage 1 + Stage 2 (parallel + deliberate) beats single-pass.

Feynman: Plan → Research → Draft → Cite → Final beats one-shot.

MEMTIER: two-stage retrieval (semantic facts → episodic entries) beats one-stage flat BM25.

The cross-paper takeaway: **multi-pass architectures with separated concerns dominate single-pass architectures.** Each pass has a defined input, a defined output, and a defined purpose. Single-pass tries to do everything at once, conflating concerns and amplifying errors.

## A practical recipe for builders, May 2026

Tying the four papers together into a buildable system:

### Layer 1 — Memory substrate (MEMTIER-like)

- Episodic JSONL store, daily files, append-only, structured schema with cognitive weight.
- Async consolidation daemon that LLM-distils episodic clusters into a semantic tier.
- Two-stage retrieval (semantic facts → episodic entries).
- Five-signal scoring (BM25, time decay, cognitive weight, tier boost, [reserved] semantic similarity).
- Two lifecycle hooks: `before_prompt_build`, `agent_end`.
- Multi-agent isolation: episodic agent-private, semantic project-shared.
- Plan to migrate ϕ_sem to a dense retriever (Phase 3) once initial system is stable.

### Layer 2 — Skill artifacts (Ctx2Skill / HeavySkill-like)

- One Markdown skill file per capability.
- `Activation conditions` declared per skill so the orchestrator knows when to load.
- Skills constructed via:
  - Adversarial self-play (Ctx2Skill) for context-specific capabilities,
  - Workflow distillation (HeavySkill) for procedural capabilities (parallel reason-and-deliberate, code review, planning, debugging),
  - Interaction-trace extraction (Voyager-style) for tool-use capabilities.
- Skills are *version-controlled* in git, peer-reviewed, tested.
- Each skill has a *quality evaluation* (a held-out task set on which we measure skill quality before and after edits).

### Layer 3 — Multi-agent harness (Feynman-like)

- Lead agent + 2–4 specialised subagents per workflow.
- Subagents are Markdown files with frontmatter.
- File-based handoffs only — no inline context dumping.
- Per-run slug; every artifact prefixes with the slug.
- Plan artifact (`outputs/.plans/<slug>.md`) as the run's working memory.
- Workspace `CHANGELOG.md` as the project's lab notebook.
- Mandatory Verifier subagent for any externalisable output.
- Standard provenance sidecars (`<slug>.provenance.md`) recording source accounting.

### Layer 4 — Runtime

- Pi or OpenClaw or a custom Pi-compatible runtime.
- LLM-provider-agnostic (multi-key, multi-model).
- Local model support (LM Studio, Ollama, vLLM) for cost-sensitive paths.
- Optional Modal / RunPod integration for compute-bound replication or training.
- Optional Temporal substrate ([150](150-temporal-durable-execution-substrate.md)) for crash-safe long-running workflows.

### Layer 5 — Operational discipline

- Plan-first, verify-after.
- Slug-keyed everything.
- Provenance tracked.
- Verification states honest (`verified`, `unverified`, `blocked`, `inferred`).
- No silent task skipping (subagents must record skipped tasks).
- Adversarial verification on critical claims.

This is roughly two months of solo engineering work (or two weeks for a small team) to assemble. Most of the components exist as open-source today; the missing piece is the *integration architecture* — and the May-2026 papers describe exactly the right architecture.

## Where each paper falls short, and what fills the gap

No paper is complete; each leaves something to the others:

- **MEMTIER's Phase 3 (dense retrieval) is unimplemented.** That is where the work goes next: hybrid BM25 + dense, multi-vector retrieval, absolute date resolution. Adjacent: [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md), [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md), [132-vector-cdc-pipelines](132-vector-cdc-pipelines.md), [133-reranking-for-agentic-retrieval](133-reranking-for-agentic-retrieval.md). MEMTIER's PPO infrastructure is also reusable for the dense-retrieval policy.
- **Ctx2Skill is per-context.** Skill composition across multiple contexts is unsolved. The natural extension: cross-context skill composition with per-context Cross-Time Replay, then a meta-replay across contexts. No paper has done this.
- **Feynman is research-domain-specific.** The patterns generalise but the concrete codebase is opinionated. The general-purpose harness that combines Feynman's verification discipline with Claude Code's coding focus and OpenClaw's openness does not exist as a single project yet.
- **HeavySkill's RLVR is preliminary.** The first 100 steps show ~10% lift, but full-scale heavy-thinking-aware post-training is still future work for the open community (frontier labs may have already done it). Expect 2026 H2 papers reporting full RLVR results.

The next 6 months of papers will fill these gaps. A safe bet for what shows up:

1. *MEMTIER-Dense*: a Phase 3 paper turning on dense retrieval + recall-first ranking, addressing multi-session synthesis (0.180 → 0.50+) and temporal reasoning (0.316 → 0.50+).
2. *Ctx2Skill++*: cross-context skill composition with hierarchical Cross-Time Replay; a meta-replay over a "skill library" rather than a single skill set.
3. *General-purpose Feynman*: an MIT-licensed harness with mandatory Verifier, file-based handoffs, slug-keyed filesystem, integrated MEMTIER-style memory, and a coding-focused skill library.
4. *Heavy-X-RLVR*: extension of HeavySkill to other inner skills (planning, debugging, browsing, tool composition) trained via RLVR.

## What this synthesis means for harness vendors

The four papers, read together, articulate an existential question for harness vendors:

> If skills are portable across models and the inner reasoning skill is RLVR-trainable into model weights, what is the durable value of your harness?

Three honest answers:

### Answer 1 — Operational discipline

The durable value is *the operational discipline you enforce* — verification gates, file-based handoffs, slug-keyed filesystems, provenance sidecars, lab notebooks. Anyone can ship a wrapper around an LLM. Few teams ship a harness that *forbids unsourced output and enforces it*.

This is what Feynman bets on. It is also where the most defensible product moats live for the next 12–24 months.

### Answer 2 — Memory architecture

The durable value is *the memory substrate*. Models do not have project-specific durable memory; they have parametric knowledge plus context windows. Anyone who builds a memory substrate that survives 72-hour, 7-day, 30-day operation windows owns a layer that the model cannot encroach on.

This is what MEMTIER's architectural commitments imply. Vendors who get the structure right (tiered, distilled, attribution-loop-equipped) own a layer that scales with usage and gets *better* over time as cognitive weights and semantic facts accumulate.

### Answer 3 — Domain skills, not generic capabilities

The durable value is *domain-specific skills* — skills that encode legal compliance, medical procedure, financial regulation, customer-specific workflow. Generic reasoning skills will be commoditised as models internalise them. Domain-specific skills require domain knowledge that can be acquired only by working in the domain.

This is the upgrade path: from being a "generic agent harness vendor" to being a "vertical-specific agent platform." The harness is plumbing; the skills are the asset.

## What this synthesis means for individual builders

For someone building an agent today (a coding agent, a research agent, a customer-support agent), the four papers suggest a concrete priority order:

1. **First: durable structured memory.** Episodic JSONL + semantic tier + two-stage retrieval. This will close the largest single source of long-horizon failure (the 14pp/72h drop or its analogue in your stack). Without it, every other improvement is fighting an entropy gradient.
2. **Second: a skill library.** Even if it is hand-written initially. Five to ten Markdown skills covering your most-common procedural capabilities. This is the substrate you will train against, hand off across model upgrades, and version-control.
3. **Third: file-based multi-agent coordination if you need it.** If your tasks are big enough that one agent loop cannot finish, partition into subagents with file-based handoffs. Slug-keyed filesystem.
4. **Fourth: a Verifier subagent.** Whatever your domain, output integrity is at stake. Bundle generation; un-bundle verification. Make the Verifier structural, not optional.
5. **Fifth and last: model upgrades.** Only after the above four. Bigger models will *not* fix problems caused by missing memory architecture, missing skill library, missing coordination, missing verification. They will only fix the *raw reasoning quality* layer, which is rarely the binding constraint for production agents.

This priority list is, frankly, the inverse of what most teams actually do. Most teams pick a frontier model first, glue some prompts around it, and ship. The May-2026 papers are evidence that this approach has run out of room — diminishing returns from model scaling, structural problems untouched by it.

## Where this synthesis leads

The next few moves I expect in the field, conditional on the May-2026 evidence:

### Near-term (Q3–Q4 2026)

- **MEMTIER-style memory plugins** for Claude Code, Cursor, OpenClaw, Cline, and other coding agents. The 14pp/72h figure is too actionable for builders to ignore.
- **LongMemEval-S becomes the standard memory benchmark**. Field consensus shifts away from LoCoMo within 6 months.
- **Heavy-thinking-aware models from frontier labs**. Already happening (Kimi K2 Thinking, GPT-5 Thinking, etc.); the explicit skill-as-RLVR-target framing accelerates it.
- **Skill libraries as version-controlled assets**. Teams begin treating their `skills/` directory the way they treat their `lib/` directory — first-class engineering artefact.
- **Verifier subagents in every serious harness**. Feynman's pattern is too obviously right not to copy.

### Medium-term (Q1–Q2 2027)

- **Hybrid retrieval architectures** for memory: dense + BM25 + reranking, replacing the BM25 bottleneck MEMTIER diagnoses.
- **Cross-context skill composition** as a research topic, with Cross-Time Replay-like mechanisms operating across multi-document skill libraries.
- **Skill marketplaces** — open, MIT-licensed Markdown skill collections curated by domain (legal, medical, finance, ops, coding). The Hugging Face for skills.
- **Memory-as-a-service offerings** from infrastructure vendors. The MEMTIER architecture as a managed substrate, with the consolidation daemon and PPO trainer as service primitives.

### Long-term (H2 2027 and beyond)

- **The harness layer becomes thin glue.** Most reasoning capabilities are internalised into models. Most skill artifacts are RL training data, not inference-time prompts. The harness mostly does memory, integration, and policy.
- **The two-substrate architecture (memory + skills) crystallises as the dominant pattern.** New harnesses are evaluated on substrate quality, not orchestration cleverness.
- **The generic agent platform consolidates around a few vendors.** The vertical agent platforms (legal, medical, etc.) proliferate; their differentiation is *domain skill libraries plus domain-specific memory architecture*.

These predictions could be wrong in detail. The structural claim — *memory and skills as the two trainable substrates, with the harness as glue and the model as runtime* — is harder to escape. The four May-2026 papers, read together, make the structural claim hard to argue with.

## Closing: what this changes about how to read the canon

A re-read of earlier chapters in this canon, with the May-2026 lens, reveals which chapters *aged well* and which look more dated:

**Aged well** (substrate-level chapters, still load-bearing):
- [09-memory-files](09-memory-files.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md), [128-knowledge-graphs-as-substrate](128-knowledge-graphs-as-substrate.md), [130-distributed-sql-as-agent-memory](130-distributed-sql-as-agent-memory.md), [134-semantic-indexing](134-semantic-indexing.md) — memory architectures.
- [04-skills](04-skills.md), [19-voyager-skill-libraries](19-voyager-skill-libraries.md), [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md), [71-karpathy-skills-single-file-guardrails](71-karpathy-skills-single-file-guardrails.md), [79-skill-rag](79-skill-rag.md) — skill artifacts.
- [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [38-claw-eval](38-claw-eval.md), [115-evaluating-llm-systems](115-evaluating-llm-systems.md) — verification as structural primitive.
- [02-subagent-delegation](02-subagent-delegation.md), [42-langchain-deep-agents](42-langchain-deep-agents.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md) — multi-agent file-based handoff patterns.
- [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md) — durable execution as substrate.

**Aged less well** (model-centric chapters, less load-bearing now):
- Chapters arguing for specific model architectures (recurrent depth, etc.) as agent enablers — useful, but architecture-not-model finding makes them less central.
- Chapters about specific reasoning techniques as orchestration glue (ReAct, ToT, plan-and-solve) — still important to know, but their content is increasingly being internalised by models. Read as historical context for what is now an inner skill.

**Becoming more important**:
- Anything operationalising the *file-based handoff* pattern.
- Anything addressing *multi-agent isolation* and *cross-agent knowledge transfer*.
- Anything providing *durable structured memory* substrates.
- Anything about *operational discipline* — provenance, verification, lab notebooks, plan artifacts.

The intellectual centre of gravity of the canon is shifting from *what reasoning techniques can the model do* to *what substrates does the agent stand on*. The May-2026 papers mark the shift.

## Where this fits

This is a synthesis chapter — it should be read after the four sources it synthesises:

1. [151-memtier-why-flat-memory-breaks-at-72-hours](151-memtier-why-flat-memory-breaks-at-72-hours.md)
2. [152-memtier-3-tier-architecture-and-retrieval](152-memtier-3-tier-architecture-and-retrieval.md)
3. [153-memtier-llm-distillation-and-the-three-invariants](153-memtier-llm-distillation-and-the-three-invariants.md)
4. [154-ctx2skill-self-evolving-context-skills](154-ctx2skill-self-evolving-context-skills.md)
5. [155-feynman-multi-agent-research-harness](155-feynman-multi-agent-research-harness.md)
6. [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md)

It complements but does not replace the prior synthesis chapters in the canon: [56-sea-landscape-2026](56-sea-landscape-2026.md), [60-sea-top-github-repos](60-sea-top-github-repos.md), [66-meta-harness-landscape](66-meta-harness-landscape.md), [67-recommended-breakthrough-project](67-recommended-breakthrough-project.md), [76-ten-links-synthesis](76-ten-links-synthesis.md), [99-papers-deep-dive-synthesis](99-papers-deep-dive-synthesis.md). Each of those took a different cross-section of the canon and named what was emerging. This chapter focuses specifically on the four May-2026 artifacts and what they jointly imply about substrate-level architecture.

For a builder reading this synthesis with the goal of acting on it: the practical recipe in the *Practical recipe for builders* section is the operational checklist. Start there. The conceptual sections (migration thesis, two-substrate thesis, file-based handoffs, verification as structural primitive) are the framing that makes the recipe make sense.

## References

Primary sources (the four artifacts synthesised):

1. Sidik & Rokach. 2026. *MEMTIER: Tiered Memory Architecture and Retrieval Bottleneck Analysis for Long-Running Autonomous AI Agents*. arXiv:2605.03675.
2. Si et al. 2026. *From Context to Skills: Can Language Models Learn from Context Skillfully?* arXiv:2604.27660v2.
3. getcompanion-ai team. 2026. *Feynman — open-source AI research agent*. https://github.com/getcompanion-ai/feynman.
4. Wang et al. 2026. *HeavySkill: Heavy Thinking as the Inner Skill in Agentic Harness*. arXiv:2605.02396.

Adjacent canon chapters (substrate level):

5. [09-memory-files.md](09-memory-files.md), [10-multi-session-continuity.md](10-multi-session-continuity.md), [12-todo-scratchpad-state.md](12-todo-scratchpad-state.md), [72-claude-mem-persistent-memory-compression.md](72-claude-mem-persistent-memory-compression.md), [78-ngc-neural-garbage-collection.md](78-ngc-neural-garbage-collection.md), [81-reasoningbank.md](81-reasoningbank.md), [128-knowledge-graphs-as-substrate.md](128-knowledge-graphs-as-substrate.md), [129-kg-rag-hybrid-retrieval.md](129-kg-rag-hybrid-retrieval.md), [130-distributed-sql-as-agent-memory.md](130-distributed-sql-as-agent-memory.md), [131-temporal-bitemporal-tables.md](131-temporal-bitemporal-tables.md), [132-vector-cdc-pipelines.md](132-vector-cdc-pipelines.md), [133-reranking-for-agentic-retrieval.md](133-reranking-for-agentic-retrieval.md), [134-semantic-indexing.md](134-semantic-indexing.md), [135-trustworthy-generation.md](135-trustworthy-generation.md).
6. [04-skills.md](04-skills.md), [19-voyager-skill-libraries.md](19-voyager-skill-libraries.md), [68-atomic-skills-scaling-coding-agents.md](68-atomic-skills-scaling-coding-agents.md), [71-karpathy-skills-single-file-guardrails.md](71-karpathy-skills-single-file-guardrails.md), [79-skill-rag.md](79-skill-rag.md), [89-voyager-deep.md](89-voyager-deep.md).
7. [02-subagent-delegation.md](02-subagent-delegation.md), [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md), [21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md), [38-claw-eval.md](38-claw-eval.md), [42-langchain-deep-agents.md](42-langchain-deep-agents.md), [73-multica-managed-agents-platform.md](73-multica-managed-agents-platform.md), [115-evaluating-llm-systems.md](115-evaluating-llm-systems.md), [121-multi-agent-coordination-patterns.md](121-multi-agent-coordination-patterns.md), [127-loan-processing-multi-agent-case-study.md](127-loan-processing-multi-agent-case-study.md).
8. [40-harness-engineering-principles.md](40-harness-engineering-principles.md), [44-four-pillars-harness-engineering.md](44-four-pillars-harness-engineering.md), [144-build-your-own-harness.md](144-build-your-own-harness.md), [150-temporal-durable-execution-substrate.md](150-temporal-durable-execution-substrate.md).
9. Prior synthesis chapters: [56-sea-landscape-2026.md](56-sea-landscape-2026.md), [60-sea-top-github-repos.md](60-sea-top-github-repos.md), [66-meta-harness-landscape.md](66-meta-harness-landscape.md), [67-recommended-breakthrough-project.md](67-recommended-breakthrough-project.md), [76-ten-links-synthesis.md](76-ten-links-synthesis.md), [99-papers-deep-dive-synthesis.md](99-papers-deep-dive-synthesis.md).

Adjacent papers (LongMemEval, LoCoMo, MemGPT, Voyager, Reflexion, etc.) are listed in the individual chapter references.
