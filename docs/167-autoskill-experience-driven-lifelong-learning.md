# 167 — AutoSkill: Experience-Driven Lifelong Learning via Skill Self-Evolution

**Paper.** Yutao Yang, Junsong Li, Qianjun Pan, Bihao Zhan, Yuxuan Cai, Lin Du, Jie Zhou, Kai Chen, Liang He; with scientific direction from Xin Li, Bo Zhang, Qin Chen — *AutoSkill: Experience-Driven Lifelong Learning via Skill Self-Evolution* — arXiv:2603.01145v2 [cs.AI]. Affiliations: **Shanghai AI Laboratory** + **School of Computer Science, East China Normal University (ECNU)**. Code: [github.com/ECNU-ICALK/AutoSkill](https://github.com/ECNU-ICALK/AutoSkill) — MIT licence.

**One-line definition.** AutoSkill is a *training-free, model-agnostic* lifelong-learning framework that turns recurring user-dialogue patterns into versioned `SKILL.md` artifacts via online and offline extractors, maintains them in a per-user `SkillBank` with hybrid retrieval, and injects the relevant ones into future requests without touching model parameters — closing the loop between everyday LLM interaction and persistent, transferable, *human-editable* capabilities.

## Why this paper matters

Two trends in agent engineering meet here.

**Trend 1 — Skills as the trainable, portable layer of the agent stack.** [04-skills](04-skills.md) introduced the SKILL.md format. [71-karpathy-skills-single-file-guardrails](71-karpathy-skills-single-file-guardrails.md), [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md), [79-skill-rag](79-skill-rag.md), [89-voyager-deep](89-voyager-deep.md), [154-ctx2skill-self-evolving-context-skills](154-ctx2skill-self-evolving-context-skills.md), and [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md) collectively argue that *skills, not models, are the unit of capability accumulation*.

**Trend 2 — Lifelong personalisation without retraining.** [72-claude-mem-persistent-memory-compression](72-claude-mem-persistent-memory-compression.md), [81-reasoningbank](81-reasoningbank.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md), [69-agent-world-self-evolving-training-arena](69-agent-world-self-evolving-training-arena.md) all attempt to retain experience across sessions; almost all rely either on raw-trajectory memory (token-heavy, noisy) or on parameter-touching fine-tunes (closed-model-incompatible).

AutoSkill is the *cross-product*: **explicit-skill personalisation, training-free, with full artifact-level transparency**. The paper's strongest move is taking what Voyager / Ctx2Skill / EvoSkill / CoEvoSkills do *for tasks* (skill libraries that grow through experience) and bringing it down to the *consumer chat* setting, where the experience signal is dialogue itself — no execution feedback, no ground-truth labels, no rewards. Whether your user wants you to "stop using emojis", "always cite sources", or "rewrite text in formal Vietnamese without commentary", AutoSkill turns those repeating preferences into *versioned files you can read, audit, edit by hand, and share*.

It also matters that the system ships with `AutoSkill4Doc` (paper-and-manual-to-skill) and `AutoSkill4OpenClaw` (trajectory-driven), so the framework spans three input modalities — dialogue, document, agent trajectory — under one universal artifact format.

## Problem it solves

Five concrete gaps:

1. **Stable user preferences are forgotten across sessions.** Users repeatedly express the same constraints (no hallucinations, follow institutional style, avoid jargon, prefer concise answers). Standard LLM stacks treat each session as cold-start; raw-memory systems try to remember, but at the cost of token-heavy, noisy retrieval.
2. **Hidden-state personalisation is opaque.** Fine-tuning, embedding-tweaks, and weight-level personalisation cannot be inspected, audited, or hand-edited. Compliance, safety, and trust break down.
3. **Memory ≠ skills.** Storing every conversation does not yield reusable patterns; what's needed is a *higher-level abstraction* — the "what to do" rule, not the "what was said" trace.
4. **Closed-source models cannot be parameter-tuned.** GPT-5, Claude Opus 4.6, Gemini 3 — the layer between user and model has to be the *prompt-and-context* layer, not weights.
5. **No standard portable format.** Skills authored for one runtime should run on another. Without a shared format the ecosystem fragments. AutoSkill commits to `SKILL.md` (Anthropic-compatible) as the lingua franca.

## Core idea in one paragraph

Run two loops side-by-side: (a) a **serving loop** where the user chats with a base LLM, with relevant skills retrieved from a `SkillBank` and injected as part of the system prompt; (b) a **maintenance loop** where, on a sliding window of recent user turns, an Extractor proposes candidate skills (durable constraints, reusable workflows — *not* one-off requests), a Judge compares each candidate to its top-M nearest existing skills and emits one of `add` / `merge` / `discard`, and a Merger semantically unions the new evidence into the existing skill while bumping its version number. No parameters change. The artifacts are Markdown files on disk. Everything is greppable, editable, exportable.

## Mechanism (step by step)

### 1. The SKILL.md schema

A skill is a tuple `s = (n, d, p, τ, γ, ξ, v)`:

| Field | Symbol | Meaning |
|---|---|---|
| Name | n | Concise, intent-explicit, searchable |
| Description | d | What the skill does + when to use |
| Prompt | p | Executable instruction body (Markdown) |
| Triggers | τ | Activation keyword set |
| Tags | γ | Categorisation labels |
| Examples | ξ | Demonstration cases |
| Version | v | SemVer-ish (`0.1.0` → `0.1.1` → … → `0.1.34`) |

The prompt body uses a fixed three-section structure:

```markdown
# Goal
[task objective]

# Constraints & Style
[format, tone, domain rules]

# Workflow (optional)
[multi-step procedures if explicit]
```

This format is human-readable, model-renderable, and Anthropic-Skills-compatible — see [71-karpathy-skills-single-file-guardrails](71-karpathy-skills-single-file-guardrails.md) for why a *single-file*, header-bounded format is load-bearing.

### 2. The four-layer architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 4 — Serving & Interaction                             │
│   Python SDK · Web UI · OpenAI-compatible reverse proxy     │
├─────────────────────────────────────────────────────────────┤
│ Layer 3 — Storage & Retrieval                               │
│   SkillBank/ JSONL + Markdown files                         │
│   Hybrid retrieval: dense (M_emb) + lexical (BM25), λ-mix   │
├─────────────────────────────────────────────────────────────┤
│ Layer 2 — Skill Management                                  │
│   Extractor → Judge → Merger pipeline                       │
│   Decisions: add | merge | discard                          │
├─────────────────────────────────────────────────────────────┤
│ Layer 1 — Skill Abstraction                                 │
│   The (n, d, p, τ, γ, ξ, v) tuple as on-disk SKILL.md       │
└─────────────────────────────────────────────────────────────┘
```

**Layer 1 — Abstraction.** Skill is the unit. Not memory, not embedding, not weight — *file*.

**Layer 2 — Management.** A small pipeline of LLM calls (templated `P_*` prompts, see §4) decides what to do with each candidate.

**Layer 3 — Storage.** `SkillBank/Users/<user_id>/<skill-slug>/SKILL.md`, plus `scripts/`, `references/`, `assets/`, and a per-user vector cache. Common skills live in `SkillBank/Common/`.

**Layer 4 — Serving.** Three integration surfaces — direct Python SDK, browser Web UI, and an OpenAI-compatible reverse proxy that any client (cursor / claude-code / openai-python) can be pointed at. The proxy is the wedge: *zero changes to the calling code, instant skill injection*.

### 3. The four-stage skill lifecycle

```
                      ┌────────────────────┐
                      │  1. Ingestion      │
                      │  dialogue, traces  │
                      └──────────┬─────────┘
                                 ▼
                      ┌────────────────────┐
                      │  2. Extraction     │
                      │  user-turn window  │   (proposes candidates;
                      │  → candidate skill │    confidence-scored)
                      └──────────┬─────────┘
                                 ▼
                      ┌────────────────────┐
                      │  3. Maintenance    │
                      │  add | merge       │   (top-M neighbour
                      │  | discard         │    comparison; version bump)
                      └──────────┬─────────┘
                                 ▼
                      ┌────────────────────┐
                      │  4. Reuse          │
                      │  retrieve · inject │   (top-K above threshold η,
                      │  · respond         │    rendered into prompt)
                      └────────────────────┘
```

Each stage is a small, testable, prompt-driven module. Stages 1–3 run *behind* the user-facing chat (background or session-end); stage 4 runs in the request critical path with retrieval as the only added cost.

### 4. The five prompt-driven modules

All AutoSkill behaviour is implemented via task-specific prompts paired with a base LLM. There is no model training. The prompts have stable names and roles:

| Symbol | Role | Where it runs |
|---|---|---|
| `P_rw`  | Query rewriting — strip context dependencies, expose constraints | request critical path |
| `P_chat` | Dialogue generation — skill-aware response | request critical path |
| `P_ext` | Skill extraction — surface durable constraints from user turns | maintenance loop |
| `P_judge` | Add/merge/discard decision | maintenance loop |
| `P_merge` | Semantic union of candidate into existing skill | maintenance loop |

Plus `M_emb` for the dense retrieval index.

The paper's design rules for each prompt are worth quoting in compressed form:

- **`P_rw`:** rewrite into a standalone retrieval query; detect topic continuity; preserve task anchor; resolve references; keep only retrieval-relevant constraints.
- **`P_chat`:** treat retrieved skills as *optional* context; use only when matching current intent; **never explicitly mention skill injection** to the user.
- **`P_ext`:** operate on user turns (not assistant output); detect turn boundaries (subject change); extract only *durable* constraints, policies, workflows; avoid one-offs and generic tasks; *remove case-specific entities* (no "rewrite *this* document"; yes "always rewrite documents in formal Vietnamese without commentary").
- **`P_judge`:** four-axis comparison — job-to-be-done, deliverable type, constraints, tools/workflow. *Discard gate first* (generic/low-signal goes away). Merge only on same capability after instance details are removed.
- **`P_merge`:** preserve capability identity; semantic union not concatenation; import only non-conflicting additions; avoid regressions; deduplicate sections.

This is where the methodological discipline lives. A naïve "summarise the conversation and store it" approach produces a noisy memory store. AutoSkill's *durability test* (P_ext) and *four-axis comparison* (P_judge) are the difference between a skill library and a chat-log archive.

### 5. Hybrid retrieval

At inference time:

```
candidates = topK_vector(query_embedding, M_emb)   ∪   topK_bm25(query, lexical_index)
ranked     = sort by  λ * dense_score + (1 - λ) * lexical_score
selected   = { s in ranked  |  score(s) > η }    (top-K cut)
injected   = render(s) for s in selected
```

`λ` is the dense/lexical mix; `η` is the relevance threshold. The paper notes that *lexical hits matter* — many user constraints are keyword-anchored ("never use emojis"), and BM25 catches them where embeddings drift on short queries.

### 6. Versioned merging

When `P_judge` returns `merge`, the existing skill's identity is preserved (same `n`, same slug, same file path) and `P_merge` integrates new constraints/examples/workflow steps. Version increments — `0.1.0` → `0.1.1` → `0.1.2` … . The paper reports skills reaching version `0.1.34` (i.e. 34 distinct merge events) for high-traffic capabilities. This is the empirical evidence that merging actually happens — that the system is not just a candidate-skill *generator* but an actual *evolution* loop.

## Empirical study — WildChat-1M

The paper runs AutoSkill at scale on the WildChat-1M corpus (real-world conversational logs):

| Slice | Conversations | Extracted skills |
|---|---|---|
| English × GPT-3.5 | 10,243 | 631 |
| Chinese × GPT-4 | 1,145 | 224 |

Average conversation length 22–32 messages. Filter: conversations with > 8 turns (signal for repeated patterns).

**Skill distribution by category (English subset):**

| Category | Count |
|---|---|
| Programming & Software Dev | 482 |
| Writing & Content Creation | 363 |
| Data & AI/ML | 354 |
| Systems / DevOps | 194 |

**Top tags:** Python (98), JavaScript (38), Excel (36), C++ (35), Creative Writing (35).

**Versioning evidence:** the `professional_text_rewrite` skill reaches **v0.1.34** — 34 successful merge events. Many other skills remain at `v0.1.0` (low-frequency, no further merging). This bimodal distribution is itself a signal: AutoSkill is not over-merging trivial differences (else everything would be high-version), and it is not failing to merge (else everything would be `0.1.0`).

## Two case studies (verbatim flavours)

### A. 顶级心理咨询师 — "Top-Level Psychological Counselor" (Chinese, GPT-4 stream)

A multi-turn Chinese-language dialogue where the user repeatedly steers the assistant toward warm, empathetic, non-diagnostic conversational style. AutoSkill extracts a skill encoding:

- Conversational *style*: warmth, empathy, non-judgement, patience.
- *Privacy* respect.
- *Anti-pattern*: do not provide medical diagnosis, do not pathologise, do not over-prescribe action items.

Version `0.1.0` — extracted in one pass, no further merges (the user was consistent, the skill was complete). The case shows AutoSkill capturing *interpersonal style*, not just task procedures.

### B. `professional_text_rewrite` (English, GPT-3.5 stream)

A high-traffic skill at version `0.1.34`. The text-rewrite capability:

- Improve fluency & grammar while *strictly preserving meaning*.
- Prohibit commentary, prohibit offering multiple options.
- Anti-pattern enforcement: no explanations of changes, no omissions.

Each merge event integrates an additional constraint or anti-pattern surfaced in a fresh user interaction. The skill *grows in expressivity* while keeping the same identity — exactly the lifelong-learning signal AutoSkill is designed to produce.

## Implementation — the four code surfaces

### `autoskill/`

The core SDK + Web UI + OpenAI-compatible proxy. The proxy is the deployment wedge: any client that talks OpenAI's API can route through AutoSkill and gain skill injection without code changes. Online (live conversation) and offline (post-hoc trajectory) extraction both live here.

### `AutoSkill4Doc/`

Document-to-skill pipeline. Feed it a paper, manual, or domain document; it produces a `SKILL.md` (or several) encoding the procedural knowledge of the document. This generalises the framework beyond dialogue: any *experience* — a reading session, a tutorial walkthrough — can become skills.

The 154-ctx2skill paper benchmarks Ctx2Skill *against* AutoSkill4Doc as a baseline ([154-ctx2skill](154-ctx2skill-self-evolving-context-skills.md) §empirical: GPT-4.1 + AutoSkill4Doc reaches 13.4% on CL-bench vs. 11.1% no-skill baseline; Ctx2Skill goes to 16.5%). AutoSkill4Doc is therefore *positioned as the strong single-pass baseline* in the no-feedback context-learning regime.

### `AutoSkill4OpenClaw/`

Integration with the OpenClaw runtime: trajectory-driven skill evolution and native skill mirroring. Where `autoskill` watches dialogue, this watches *agent action traces*. The skills extracted are tool-use patterns, recovery strategies, navigation heuristics — closer to Voyager's library than to chat-style preferences.

### `SkillEvo/`

The replay / evaluation / mutation / promotion framework. Holds out a corpus, replays it through different skill-set configurations, scores outcomes, and decides whether a candidate skill earns promotion to the active library. This is the *quality gate* — the answer to "how do you know your auto-extracted skills are actually any good?"

In the Polaris (this monorepo's research-harness project) lineage, `SkillEvo` is the closest analogue to the existing `polaris-evals` harness; AutoSkill's promotion gate is the closest analogue to Polaris's `BL-PROMOTE-SKILL` bright-line.

## Storage layout (verbatim shape)

```
SkillBank/
├── Users/<user_id>/
│   ├── <skill-slug>/
│   │   ├── SKILL.md
│   │   ├── scripts/      (optional helpers)
│   │   ├── references/   (optional)
│   │   └── assets/       (optional)
│   └── …
├── Common/
│   └── …                 (cross-user shared skills)
└── vectors/
    └── …                 (persistent embedding cache)
```

Two structural choices worth flagging:

1. **Per-user namespacing** — privacy and personalisation by directory boundary, not by query filter.
2. **`Common/` for shared skills** — promotion path from per-user-private to org-shared lives in a path move, not a database update.

## Three design principles (from the paper)

1. **Explicit skill representation.** Externalised structured artifacts, not hidden state. *Files you can read.*
2. **Continuous but controlled evolution.** The Extractor proposes; the Judge controls; the Merger preserves identity. No skill explosion, no skill staleness.
3. **Low-friction deployment.** Sit atop existing LLM stacks via SDK / Web UI / OpenAI-compatible proxy. Docker Compose support. Don't make the user adopt a new model or runtime.

## Limitations and future directions

The paper does not include a numbered limitations section but flags four follow-ups:

1. **Cross-organisation skill scaling.** A SkillBank shared across thousands of users needs governance (who promotes? who demotes?).
2. **Cross-user transfer.** Today skills are per-user; can lessons from user A safely improve user B without leaking preferences?
3. **Principled skill composition.** Multiple skills in one prompt = potential conflict; today this is heuristic. A formal composition algebra is open.
4. **Skill-quality metrics.** Beyond conversation length × extraction rate, what *intrinsic* quality measure can the system optimise?

Three further limitations that the paper doesn't quite say out loud but are visible:

5. **No execution feedback** — unlike EvoSkill / CoEvoSkills / SkillRL (see [168](168-evoskill-coding-agent-skill-discovery.md), [169](169-coevoskills-co-evolutionary-verification.md), [170](170-skillrl-recursive-skill-augmented-rl.md)), AutoSkill cannot measure *whether the merged skill actually helps*. Merging is judged on textual coherence, not outcome.
6. **No cross-model invariant** — the framework is model-agnostic but does not enforce a *cross-model adversarial-pair* discipline (cf. [Polaris ADR-002](../projects/polaris/docs/concepts/adrs/adr-002-cross-model-adversarial-pair.md)). The Extractor and Judge can be the same model family.
7. **Trust labels are absent** — there's no mechanism distinguishing skills extracted from authoritative sources (peer-reviewed, replicated) versus speculative ones (one user's idiosyncratic preference). Compare [Polaris P24 trust-tiering](../projects/polaris/docs/research/p24-trust-tiering-retractions.md).

## Where AutoSkill sits in the corpus

| Other doc | Relationship |
|---|---|
| [04-skills](04-skills.md), [71-karpathy-skills](71-karpathy-skills-single-file-guardrails.md) | Provides the SKILL.md format AutoSkill consumes. |
| [89-voyager-deep](89-voyager-deep.md) | Voyager's growing skill library is the conceptual ancestor. AutoSkill ports the idea to dialogue + adds versioning + adds a dispatch hierarchy. |
| [79-skill-rag](79-skill-rag.md) | Same retrieval-side idea (hybrid dense+lexical, top-K skill injection). AutoSkill adds the *production* side. |
| [154-ctx2skill](154-ctx2skill-self-evolving-context-skills.md) | Direct competitor for the *no-feedback* regime. Ctx2Skill uses adversarial self-play; AutoSkill uses the four-axis Judge over user-turn sequences. The papers cite each other; CL-bench numbers are directly comparable. |
| [156-heavyskill](156-heavyskill-parallel-reasoning-deliberation.md) | Argues the agent harness itself can become a skill. AutoSkill provides one of the input streams that could feed that skill. |
| [162-paper2agent](162-paper2agent-reimagining-papers-as-agents.md) | `AutoSkill4Doc` is the closest analogue — paper → executable skill. Paper2Agent goes further (skill → MCP server). |
| [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md) | Hermes's outcome-fed skill refinement is a *hot path* analogue of AutoSkill's offline merge loop. |
| [81-reasoningbank](81-reasoningbank.md) | Failure-distillation-as-memory; AutoSkill's positive-pattern-as-skill is the dual. |
| [72-claude-mem](72-claude-mem-persistent-memory-compression.md) | Compresses raw memory; AutoSkill replaces "memory of what was said" with "skill of what to do next time". |
| [168-evoskill](168-evoskill-coding-agent-skill-discovery.md), [169-coevoskills](169-coevoskills-co-evolutionary-verification.md), [170-skillrl](170-skillrl-recursive-skill-augmented-rl.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md) | Sibling and synthesis docs. |

## When to use AutoSkill (decision rule)

Reach for AutoSkill when **all** of the following hold:

- You have a chat-style or dialogue-bound product where users repeat preferences across sessions.
- You cannot fine-tune the underlying LLM (closed model, audit constraints, frequent model swaps).
- Your users are humans who will benefit from *seeing and editing* the captured preferences.
- You can afford a maintenance LLM call per session-end (or batched).
- You don't have a clean execution-feedback signal that would let you use EvoSkill / CoEvoSkills / SkillRL instead.

Reach for **EvoSkill** ([168](168-evoskill-coding-agent-skill-discovery.md)) instead when you have a coding agent and a benchmark with binary success — the failure-driven loop dominates AutoSkill in that regime.

Reach for **CoEvoSkills** ([169](169-coevoskills-co-evolutionary-verification.md)) when you need *multi-file skill packages* (not just a SKILL.md) and you want a surrogate-verifier discipline without ground-truth tests.

Reach for **SkillRL** ([170](170-skillrl-recursive-skill-augmented-rl.md)) when you have weight access and an RL training loop already in motion — recursive skill augmentation will compress trajectories and improve generalisation.

Reach for **Ctx2Skill** ([154](154-ctx2skill-self-evolving-context-skills.md)) when the input is a *single dense document* (paper, manual) and there is no dialogue stream at all.

## Companion plugin — `AutoSkill_Claude` (CatVinci Studio)

A separately-developed Claude Code plugin (MIT, [github.com/CatVinci-Studio/AutoSkill_Claude](https://github.com/CatVinci-Studio/AutoSkill_Claude)) is *named after* AutoSkill but is independent of the ECNU codebase. It uses three Claude Code hooks — `PostToolUse(Skill)`, `SessionEnd`, `SessionStart` — to:

- Track every skill invocation passively.
- Detect repeated workflow patterns (≥ 2 occurrences).
- Optimise skills (`/optimize-skill`) or create new ones (`/new-skill`).
- Notify the user when configurable thresholds are crossed.

Configuration in `~/.local/share/auto-optimize-skills/queue.json`. Two knobs: `notify_after_skill_uses` (default 5) and `notify_after_new_patterns` (default 2). Pattern dedupe uses the first 100 chars of each prompt — a known limitation that misses paraphrased workflows.

The plugin is the *applied flavour* of AutoSkill: same philosophy (passive observation → durable skills → user-approved promotion), implemented as a hook-bus integration into Claude Code rather than as a model-agnostic proxy.

## Bottom line

AutoSkill makes three contributions worth holding onto:

1. **A universal skill representation across input modalities** — dialogue (`autoskill`), document (`AutoSkill4Doc`), trajectory (`AutoSkill4OpenClaw`) — all converging on `SKILL.md`.
2. **A disciplined maintenance pipeline** — `P_ext` durability filter + `P_judge` four-axis comparison + `P_merge` semantic union — that keeps the SkillBank from collapsing into either a chat archive (over-merge) or a one-off-skill graveyard (under-merge).
3. **A deployment story** — Python SDK + Web UI + OpenAI-compatible proxy — that lets the framework drop in front of any existing LLM client with zero changes to the calling code.

Whether or not AutoSkill becomes the dominant lifelong-learning framework, its skill format, four-stage lifecycle, and four-axis Judge are likely to recur. The *next* generation of papers ([168](168-evoskill-coding-agent-skill-discovery.md), [169](169-coevoskills-co-evolutionary-verification.md), [170](170-skillrl-recursive-skill-augmented-rl.md)) extend the same idea into execution-grounded coding domains and into RL-trained agents respectively. The synthesis is in [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md).

---

**Citation:** Yang Y., Li J., Pan Q., Zhan B., Cai Y., Du L., Zhou J., Chen K., He L., et al. *AutoSkill: Experience-Driven Lifelong Learning via Skill Self-Evolution.* arXiv:2603.01145v2, 2026.

**Repository:** [github.com/ECNU-ICALK/AutoSkill](https://github.com/ECNU-ICALK/AutoSkill) — MIT licence — Python 73.5%, TypeScript 16.0%.

**Companion plugin:** [github.com/CatVinci-Studio/AutoSkill_Claude](https://github.com/CatVinci-Studio/AutoSkill_Claude) — MIT licence — independent of the ECNU codebase but inspired by the framework.
