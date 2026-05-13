# 155 — Feynman: A Multi-Agent Research Harness with Source-Grounded Outputs

**Repository.** https://github.com/getcompanion-ai/feynman — MIT license — 6.6 K stars / 809 forks — 32 releases (latest v0.2.43) — TypeScript (65.1%), JavaScript (20.1%), Astro (4.9%), Shell (4.5%) — 169 commits on `main` — distributed under `@companion-ai/feynman` on npm — built on **Pi** (https://github.com/badlogic/pi-mono) for the agent runtime and **alphaXiv** (https://www.alphaxiv.org/) for paper search and analysis — installable via a single curl/PowerShell line that fetches a standalone native bundle with its own Node.js runtime.

**One-line definition.** Feynman is an open-source command-line research agent that operationalises *source-grounded multi-agent academic research* through (a) four bundled subagents — **Researcher**, **Reviewer**, **Writer**, **Verifier** — each defined as a Markdown skill file at `.feynman/agents/`, (b) nineteen *Pi skills* covering the academic research lifecycle from literature review to replication to peer review, (c) ten user-facing slash-commands (`/deepresearch`, `/lit`, `/audit`, `/replicate`, `/review`, `/draft`, `/compare`, `/autoresearch`, `/watch`, `/outputs`), and (d) a strict *file-based handoff* discipline that externalises plans, research notes, drafts, citations, provenance sidecars, and a chronological lab notebook (`CHANGELOG.md`) — all keyed off a per-run **slug** so concurrent research jobs cannot collide. The defining commitment is operational integrity: every factual claim must trace to a verified source URL, and the Verifier subagent enforces this as a separate post-processing pass.

## Why this repository matters

Most agent harnesses we have catalogued so far ([29-dive-into-claude-code](29-dive-into-claude-code.md), [42-langchain-deep-agents](42-langchain-deep-agents.md), [52-dive-into-open-claw](52-dive-into-open-claw.md), [61-archon-harness-builder](61-archon-harness-builder.md), [63-ragflow-agent-patterns](63-ragflow-agent-patterns.md), [65-deer-flow-bytedance](65-deer-flow-bytedance.md)) are *general-purpose*: coding, research, web tasks, ops, customer support, anything. Feynman is the first widely-adopted MIT-licensed harness that is *opinionated for one domain* — academic research — and as a consequence it ends up demonstrating, more clearly than any general-purpose harness can, what *opinionation* buys at the harness layer.

What Feynman is opinionated about:

1. **Sources are first-class.** Every claim links to a paper, repo, or URL with a direct, checkable link. Unsourced claims are removed by a dedicated Verifier subagent.
2. **Plans are externalised.** Long-running workflows write a plan artifact to `outputs/.plans/<slug>.md` and treat it as the working memory of the run, updated as evidence accumulates.
3. **Subagents handoff via files, not context.** The Researcher writes `<slug>-research-*.md` files; the Writer reads them; the Verifier reads the draft and the research files. Inline context dumping is a code smell.
4. **Slugs prevent collision.** Every run derives a 5-word-max hyphenated slug from its topic; every artifact in that run prefixes with the slug. Concurrent runs cannot stomp on each other.
5. **The session log is a lab notebook.** `CHANGELOG.md` is updated chronologically with what was done, what failed, what was verified, what is next. This is the "long-running workflow continuity" pattern from [10-multi-session-continuity](10-multi-session-continuity.md), made standard.

These are not abstract opinions; they are encoded into the agent skill files themselves. The Verifier's `cited.md` is structurally separate from the Writer's `draft.md` because the verification pass is not a polish step — it is a *gate* with the authority to delete claims.

For harness builders, Feynman is the first project I would point a colleague at if they asked "show me what production-grade source-grounding and multi-agent file-handoff actually look like in code." The patterns generalise far beyond research workflows.

## Repository at a glance

```
feynman/
├── .feynman/
│   ├── SYSTEM.md              # repo-level system prompt synced to runtime
│   ├── settings.json
│   ├── themes/
│   └── agents/
│       ├── researcher.md      # Pi subagent definition
│       ├── reviewer.md
│       ├── writer.md
│       └── verifier.md
├── skills/                    # 19 Pi skills, each with a SKILL.md
│   ├── alpha-research/
│   ├── autoresearch/
│   ├── contributing/
│   ├── deep-research/
│   ├── docker/
│   ├── eli5/
│   ├── jobs/
│   ├── literature-review/
│   ├── modal-compute/
│   ├── paper-code-audit/
│   ├── paper-writing/
│   ├── peer-review/
│   ├── preview/
│   ├── replication/
│   ├── runpod-compute/
│   ├── session-log/
│   ├── session-search/
│   ├── source-comparison/
│   └── watch/
├── prompts/                   # 12 slash-command prompts
│   ├── audit.md
│   ├── autoresearch.md
│   ├── compare.md
│   ├── deepresearch.md
│   ├── draft.md
│   ├── jobs.md
│   ├── lit.md
│   ├── log.md
│   ├── replicate.md
│   ├── review.md
│   ├── summarize.md
│   └── watch.md
├── src/
│   ├── bootstrap/
│   ├── cli.ts
│   ├── config/
│   ├── index.ts
│   ├── model/
│   ├── pi/
│   ├── search/
│   ├── setup/
│   ├── system/
│   ├── ui/
│   └── web-search.ts
├── extensions/                # tool integrations (alpha CLI, etc.)
├── outputs/                   # research artifacts go here
├── papers/                    # paper-style drafts
├── notes/                     # session logs
├── experiments/               # experiment configs
├── tests/                     # test suite
├── AGENTS.md                  # repo-level agent contract
└── CHANGELOG.md               # workspace lab notebook
```

The dual structure of `skills/` (model-invocable capabilities) and `prompts/` (user-facing slash commands) mirrors the Claude Code skills/commands distinction ([04-skills](04-skills.md), [62-everything-claude-code](62-everything-claude-code.md)). Skills are loaded into the model's context when relevant; prompts are user-typed entry points that expand into structured workflows.

## The four subagents

All four subagent definitions live in `.feynman/agents/<name>.md`. They are Pi *subagent* definitions — Markdown files with frontmatter — synced into the Pi agent directory at runtime. Editing these files is the canonical way to change subagent behaviour.

### Researcher — the evidence-gathering subagent

Frontmatter:
```yaml
name: researcher
thinking: high
tools: read, write, edit, bash, grep, find, ls, web_search,
       fetch_content, get_search_content
output: research.md
```

The Researcher's prompt is unusually *prescriptive about integrity*. The "Integrity commandments" section reads like a code of conduct:

1. Never fabricate a source.
2. Never claim a project exists without checking.
3. Never extrapolate details you haven't read.
4. URL or it didn't happen.
5. Read before you summarise.
6. Mark status honestly.

The search strategy is explicit: *start wide* (2–4 varied-angle queries simultaneously via `web_search.queries[]`), *evaluate availability*, *progressively narrow*, *cross-source* (web + alphaXiv `alpha search`).

The output contract is a numbered evidence table:

| # | Source | URL | Key claim | Type | Confidence |
|---|--------|-----|-----------|------|------------|
| 1 | …      | …   | …         | primary / secondary / self-reported | high / medium / low |

Findings use inline `[1]`, `[2]` references; every factual claim cites at least one numbered source. Inferences (vs directly stated source claims) are explicitly labelled as such.

A `Coverage Status` section lists what was checked directly, what remains uncertain, and tasks not completed. Skipped tasks must be recorded — *subagents may not silently skip assigned tasks.*

The deepest design choice in the Researcher: **context hygiene as an explicit rule.** "Write findings to the output file progressively. Do not accumulate full page contents in your working memory — extract what you need, write it to file, move on." This is an enforced pattern of writing *to disk* rather than holding state in context — exactly the externalised-state pattern that [12-todo-scratchpad-state](12-todo-scratchpad-state.md) and [09-memory-files](09-memory-files.md) describe.

### Verifier — the citation and source-verification subagent

Frontmatter:
```yaml
name: verifier
thinking: medium
tools: read, bash, grep, find, ls, write, edit, web_search,
       fetch_content, get_search_content
output: cited.md
```

The Verifier is the gate. Its job is to take a Writer-produced draft and the Researcher's evidence files and:

1. **Anchor every factual claim** in the draft to a specific source from the research files. Insert inline `[1]`, `[2]` citations directly after each claim.
2. **Verify every source URL** — `fetch_content` to confirm each URL resolves and contains the claimed content. Flag dead links.
3. **Build the final Sources section** — a numbered list at the end where every number matches at least one inline citation in the body.
4. **Remove unsourced claims** — if a factual claim cannot be traced to a source in the research files, either find a source or remove it. *Do not leave unsourced factual claims.*
5. **Verify meaning, not just topic overlap** — a citation is valid only if the source actually supports the specific number, quote, or conclusion attached to it.
6. **Refuse fake certainty** — do not use words like "verified," "confirmed," or "reproduced" unless underlying evidence exists.
7. **Enforce the system prompt's provenance rule** — unsupported results, figures, charts, tables, benchmarks, and quantitative claims must be removed or converted to TODOs.

Notice point 5 — *meaning, not topic overlap*. This is the difference between "yes there is a paper about X, citation valid" and "yes there is a paper that *says specifically* what we attributed to it." The Verifier is required to check the second.

The "Result provenance audit" sub-section is a checklist scan for: numeric scores, benchmark names, figure/image references, claims of improvement, dataset sizes, charts. Each item must map to a source URL, research note, raw artifact path, or script path. If not, *remove it or TODO it*. A `Removed Unsupported Claims` section is added when material is removed.

This Verifier specification is one of the strongest articulations of *operational integrity for research output* I have seen in any open-source agent project. It directly addresses one of the chronic failure modes of LLM-generated research artifacts — confidently citing sources whose content does not actually support the claim attached to them.

### Reviewer — the simulated peer-review subagent

The Reviewer behaves like a skeptical but fair AI/ML peer reviewer. Output: `review.md`. Required sections:

```markdown
## Summary
1-2 paragraph summary of contributions.

## Strengths
- [S1] ...
- [S2] ...

## Weaknesses
- [W1] **FATAL:** ...
- [W2] **MAJOR:** ...
- [W3] **MINOR:** ...

## Questions for Authors
- [Q1] ...

## Verdict
Overall assessment and confidence score.

## Revision Plan
...
```

Severity tagging is mandatory. If the parent agent frames the task as a verification pass rather than venue-style peer review, the Reviewer behaves like an *adversarial auditor* prioritising evidence integrity over novelty commentary.

The review checklist is concrete and demanding: missing/weak baselines, missing ablations, evaluation mismatches, unclear novelty, weak related-work positioning, insufficient statistical evidence, benchmark leakage, under-specified implementation details, claims that outrun experiments, sections/figures that survive from earlier drafts without support, notation drift, "verified"/"confirmed" statements without underlying evidence.

Two principles worth pulling out:

- *Do not praise vaguely.* Every positive claim should be tied to specific evidence.
- *Keep looking after you find the first major problem.* Do not stop at one issue if others remain.

These are the disciplines of competent peer review applied to LLM-generated reviews — directly addressing the "LLM-as-a-judge praises everything" failure mode.

### Writer — the drafting subagent

Frontmatter:
```yaml
name: writer
thinking: medium
tools: read, bash, grep, find, ls, write, edit
output: draft.md
```

Notice the Writer does *not* have web search tools. By design, the Writer can only work from the Researcher's evidence files. The Integrity commandments:

1. *Write only from supplied evidence.*
2. *Preserve caveats and disagreements.*
3. *Be explicit about gaps.*
4. *Do not promote draft text into fact.*
5. *No aesthetic laundering.*
6. *Follow the provenance rule.*

The Writer does **not** add inline citations — that is the Verifier's job in a separate post-processing pass. This separation of concerns is the key insight: *if writing and citing are the same step, the writer will paper over uncited claims.* If they are separate, the Verifier can scan for and remove uncited claims as a structural check.

The Writer is allowed to generate visuals (Mermaid diagrams, `pi-charts`, `pi-generative-ui`), but only when source-backed. Decorative visuals are forbidden. Each visual must reference its source data, URL, research file, raw artifact, or script.

## Multi-agent coordination — the lead/subagent pattern

The lead agent is the user-facing CLI. Subagents are dispatched via Pi's `subagent` tool. The coordination rules from `AGENTS.md`:

- **The lead agent plans, delegates, synthesises, and delivers.** It owns the final output.
- **Use subagents when work is meaningfully decomposable.** Do not spawn them for trivial work.
- **Prefer file-based handoffs.** Subagents write artifacts to disk; the lead agent reads the artifacts. *Do not return large intermediate results inline.*
- **The lead agent reconciles task completion.** Subagents may not silently skip assigned tasks; skipped or merged tasks must be recorded in the plan artifact.
- **Adversarial verification.** For critical claims, require at least one adversarial verification pass after synthesis. Fix fatal issues before delivery or surface them explicitly.

This is the pattern from [02-subagent-delegation](02-subagent-delegation.md), [42-langchain-deep-agents](42-langchain-deep-agents.md), and [73-multica-managed-agents-platform](73-multica-managed-agents-platform.md), formalised into a contract — and *adversarial verification* as a hard requirement is the non-trivial addition.

## The slug-based artifact convention

Every workflow that produces artifacts derives a short **slug** from the topic — lowercase, hyphenated, no filler words, ≤5 words. Example: `cloud-sandbox-pricing`. All files in a single run prefix with the slug:

```
outputs/.plans/cloud-sandbox-pricing.md
cloud-sandbox-pricing-research-web.md
cloud-sandbox-pricing-research-papers.md
outputs/.drafts/cloud-sandbox-pricing-draft.md
outputs/.drafts/cloud-sandbox-pricing-cited.md
outputs/cloud-sandbox-pricing.md
outputs/cloud-sandbox-pricing.provenance.md
```

The convention forbids generic names: never `research.md`, never `draft.md`, never `brief.md`, never `summary.md`. Concurrent runs cannot collide.

Why this matters: it makes the entire research filesystem *self-describing*. You can grep for a slug and see every artifact for a run. You can list the `outputs/` directory and immediately see which projects are in flight, which have completed final outputs, which have provenance sidecars. The slug is the run's natural primary key.

## The plan artifact and CHANGELOG.md as externalised memory

For long-running workflows (`/deepresearch`, `/lit`, `/replicate`, `/audit`, `/autoresearch`, `/watch`), the plan artifact is **the run's working memory**:

```
outputs/.plans/<slug>.md
```

It contains:
- Key questions
- Evidence needed
- Scale decision
- Task ledger
- Verification log
- Decision log

The plan is updated as the run evolves — task statuses progress, verification states are marked (`verified`, `unverified`, `blocked`, `inferred`), decisions and their justifications are logged. The plan is the single durable record of what the agent intended, what it tried, what worked, and what is left.

`CHANGELOG.md` at the workspace root is the *chronological lab notebook*:
- Read it before resuming substantial work.
- Append concise entries after meaningful progress, failed approaches, major verification results, or new blockers.
- Each entry identifies the active slug or objective and ends with the next recommended step.
- Mark verification state honestly.

This is the long-horizon-continuity pattern from [10-multi-session-continuity](10-multi-session-continuity.md) made first-class. Feynman doesn't just enable resumability; it *requires* it for any non-trivial task.

## The deepresearch workflow walkthrough

The `prompts/deepresearch.md` workflow is the most fully-specified entry point in the repository. The execution sequence:

### Step 1 — Plan

Create `outputs/.plans/<slug>.md` *immediately*, before any other action. Include key questions, evidence needed, scale decision, task ledger, verification log, decision log.

The scale decision is mandatory before assigning owners. For narrow "what is X" explainers, the plan must use lead-owned direct search tasks only — *do not allocate researcher subagents in the task ledger*. This prevents over-engineering small lookups into multi-agent orchestrations.

After writing the plan, the agent **stops and asks for explicit user confirmation** before gathering evidence. This is the [03-plan-mode](03-plan-mode.md) pattern: read-only planning before write-side execution.

### Step 2 — Scale

Two paths:

- **Direct search** for single facts, narrow questions, work answerable with 3–10 tool calls. The lead handles it without spawning subagents.
- **Multi-agent** for broad investigations requiring evidence triangulation across many sources.

### Step 3 — Researcher subagents

For multi-agent runs, dispatch one or more Researcher subagents in parallel, each owning a partition of the question space. Each writes to its own `<slug>-research-<topic>.md` file. The lead reads the resulting files (not inline returns).

### Step 4 — Writer drafts

The Writer reads all `<slug>-research-*.md` files and writes `outputs/.drafts/<slug>-draft.md`. Source citations are *not* added at this stage.

### Step 5 — Verifier cites

The Verifier reads the draft and the research files, adds inline `[N]` citations, verifies every URL via `fetch_content`, removes unsourced claims, and writes `outputs/.drafts/<slug>-cited.md`.

### Step 6 — Final output and provenance

The lead promotes the cited file to `outputs/<slug>.md` (or `papers/<slug>.md`) and writes `<slug>.provenance.md` recording source accounting and verification status.

### Step 7 — Lab notebook entry

The lead appends an entry to `CHANGELOG.md`: what was done, what was verified, what's next.

The whole pipeline is *strict-mode by default*. After plan approval, if any capability fails, the workflow continues in degraded mode and *still writes a final output and provenance sidecar* — never ends with chat-only output. `Verification: BLOCKED` is the honest status when verification could not be completed.

## The 19 skills

Skills are the model-invocable layer — capability-tagged Markdown files that the orchestrator loads into context when relevant. The 19 Feynman skills cover the academic research lifecycle:

| Category               | Skill                  | Purpose                                                                     |
|------------------------|------------------------|-----------------------------------------------------------------------------|
| **Research**           | `alpha-research`       | Search, read, query papers via the `alpha` CLI (alphaXiv-backed).           |
|                        | `deep-research`        | Multi-agent source-heavy investigation; produces a cited brief.             |
|                        | `literature-review`    | Lit review with consensus, disagreements, open questions.                   |
|                        | `source-comparison`    | Compare multiple sources; produce grounded comparison matrix.               |
| **Writing**            | `paper-writing`        | Polished paper-style drafts with sections, equations, citations.            |
|                        | `eli5`                 | Plain-English explanations with concrete analogies.                         |
| **Review & Audit**     | `peer-review`          | Tough but constructive AI research peer review.                             |
|                        | `paper-code-audit`     | Compare paper claims against public codebase.                               |
| **Replication**        | `replication`          | Plan or execute a replication of a paper, claim, or benchmark.              |
| **Compute**            | `docker`               | Isolated container execution for safe experiments.                          |
|                        | `modal-compute`        | Modal serverless GPU.                                                       |
|                        | `runpod-compute`       | RunPod persistent GPU pods with SSH.                                        |
| **Autonomy**           | `autoresearch`         | Autonomous experiment loop — try, measure, keep what works.                 |
|                        | `watch`                | Recurring research watch; alerts on a topic, company, paper area.           |
|                        | `jobs`                 | Inspect active background research work.                                    |
| **Memory & Continuity**| `session-log`          | Durable session logs.                                                       |
|                        | `session-search`       | Search past Feynman session transcripts.                                    |
| **UX**                 | `preview`              | Browser/PDF rendering of artifacts.                                         |
| **Meta**               | `contributing`         | Contribute changes to the Feynman repo.                                     |

The taxonomy itself is instructive. The skills are organised around *a research scientist's workflow* — gather evidence, write up, get reviewed, audit a paper against its code, replicate, watch a topic over time. Every step has a dedicated skill.

The compute-tier skills (`docker`, `modal-compute`, `runpod-compute`) acknowledge that real research increasingly involves running experiments on GPUs. Feynman's stance is to make *isolated* compute the default — every replication runs in a Docker container or a serverless Modal job — and to support *persistent* GPU pods (RunPod) for long-running experiments. This is a substantive harness commitment to *running real code* as part of research, not just summarising it.

## The 12 prompts (slash commands)

| Command          | Description                                                                 |
|------------------|-----------------------------------------------------------------------------|
| `/deepresearch`  | Source-heavy multi-agent investigation                                      |
| `/lit`           | Literature review                                                           |
| `/review`        | Simulated peer review                                                       |
| `/audit`         | Paper vs. codebase mismatch audit                                           |
| `/replicate`     | Replicate experiments on local or cloud GPUs                                |
| `/compare`       | Source comparison matrix                                                    |
| `/draft`         | Paper-style draft from research findings                                    |
| `/autoresearch`  | Autonomous experiment loop                                                  |
| `/watch`         | Recurring research watch                                                    |
| `/summarize`     | Quick summary                                                               |
| `/log`           | Session log entry                                                           |
| `/jobs`          | List active background work                                                 |

Each prompt is a Markdown file in `prompts/` that expands into the full workflow when invoked. The `/deepresearch` prompt walked through above is representative — these are not one-shot commands but *workflow templates* that orchestrate plans, subagents, and verification passes.

## The Pi runtime

Feynman is built on **Pi** (https://github.com/badlogic/pi-mono) — a lightweight agent runtime. The relationship:

- Feynman provides the *capability layer* (skills, agents, prompts).
- Pi provides the *runtime* (subagent dispatch, tool integration, model routing, session management).

Capabilities are delivered as *Pi skills* — Markdown instruction files synced to `~/.feynman/agent/skills/` on startup. The user-facing CLI (`feynman`) is a thin wrapper around the Pi runtime configured for research.

This separation matters: the same skill files can be installed into other Pi-compatible environments (`~/.codex/skills/feynman` for Codex compatibility, `.agents/skills/feynman` for repo-local installs, `.opencode/skills/feynman` for OpenCode). The capability layer is *portable across runtimes*.

## Local model support

Feynman's setup flow supports local models — LM Studio (`http://localhost:1234/v1` default), LiteLLM Proxy (`http://localhost:4000/v1`), Ollama and vLLM via custom provider config (`openai-completions` driver pointed at `/v1`).

The `.env.example` lists ~15 LM provider keys: OpenAI, Anthropic, Gemini, OpenRouter, Z.AI, Kimi, Minimax (US + CN), Mistral, Groq, X.AI, Cerebras, HuggingFace, OpenCode, AI Gateway, Azure OpenAI. Plus Modal and RunPod for compute.

This is a multi-provider stance — Feynman does not lock you into a single LM vendor. Skills are model-portable; the user chooses the model.

## Three patterns Feynman crystallises

Reading the repository in its entirety, three patterns stand out as durable contributions to the harness-engineering canon:

### 1. Verification as a structurally separate subagent pass

In most agent harnesses, "verify outputs" is a guideline. In Feynman, it is a *subagent with its own output file* — a Verifier that takes a draft and produces `cited.md`, where the very act of citation is the verification. This separation is what makes the integrity rules enforceable.

Generalises beyond research: *whenever you have a long-form generative output where claims must be traceable to evidence, run verification as a separate post-processing subagent that has the authority to remove unsupported claims.* This is a stronger pattern than "ask the model to double-check itself" because it disentangles writing from verification at the architectural level.

See related: [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [38-claw-eval](38-claw-eval.md), [115-evaluating-llm-systems](115-evaluating-llm-systems.md).

### 2. File-based handoffs as the multi-agent disciplinary primitive

Feynman's coordination rule — *prefer file-based handoffs over dumping large intermediate results back into parent context* — is the single most important practical lesson for multi-agent system designers.

The reason is simple: parent context is bounded. If subagent A returns 8 KB of research, subagent B returns 6 KB, and subagent C returns 12 KB, the parent has 26 KB of fresh tokens to wrangle on top of its own context. After three rounds of subagent calls, the parent is choking. With file-based handoffs, subagents return *paths*; the parent reads selectively.

The pattern shows up across mature multi-agent harnesses (Claude Code subagents return summaries plus paths; Memento ([108-memento-codebase-mcp](108-memento-codebase-mcp.md)) writes case bank entries to disk; OpenClaw plugins use the workspace as durable state). Feynman makes it normative.

See related: [02-subagent-delegation](02-subagent-delegation.md), [42-langchain-deep-agents](42-langchain-deep-agents.md), [73-multica-managed-agents-platform](73-multica-managed-agents-platform.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md).

### 3. The plan as externalised working memory, not a static outline

The plan artifact in Feynman *evolves throughout the run*. Task statuses change. Verification states get marked. Decisions and their justifications accumulate. The plan is the *living state* of the run, not a document written once at the start.

This is the [12-todo-scratchpad-state](12-todo-scratchpad-state.md) pattern formalised. Long-running workflows externalise the working memory to disk because in-context working memory degrades — the agent re-reads the plan to recover where it was, rather than relying on attention over a long context window. Compaction kills working memory; files survive.

See related: [09-memory-files](09-memory-files.md), [10-multi-session-continuity](10-multi-session-continuity.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md), [144-build-your-own-harness](144-build-your-own-harness.md).

## How Feynman fits into the broader harness landscape

Compared to other open-source agent harnesses:

| Harness                           | Domain         | Multi-agent?   | Verification?        | Source-grounded?   |
|-----------------------------------|----------------|----------------|----------------------|--------------------|
| **Feynman**                       | Research       | Yes (4 roles)  | Dedicated Verifier   | Mandatory          |
| Claude Code ([29](29-dive-into-claude-code.md))                       | Coding         | Yes (subagents)| Optional             | Optional           |
| OpenClaw ([52](52-dive-into-open-claw.md))                              | General        | Yes (Lobster)  | Plugin-based         | Plugin-based       |
| LangChain Deep Agents ([42](42-langchain-deep-agents.md))               | General        | Yes (async)    | Optional             | Optional           |
| Archon ([61](61-archon-harness-builder.md))                              | Builder        | Yes            | Yes                  | Optional           |
| RagFlow ([63](63-ragflow-agent-patterns.md))                              | RAG            | Yes            | Yes                  | Yes (at retrieval) |
| Deer-Flow ([65](65-deer-flow-bytedance.md))                                  | General        | Yes            | Yes                  | Optional           |

Feynman's distinguishing posture: *source-grounding is mandatory*, the Verifier is not optional, and the slug-based filesystem discipline is enforced. Most harnesses give you the *tools* to be source-grounded; Feynman makes it the default and removes the option to be loose.

## What Feynman demonstrates about skill-based agents

Feynman is one of the cleanest examples in the wild of the *skill is the portable artifact* thesis — the same thesis that [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md) formalises and that [04-skills](04-skills.md), [19-voyager-skill-libraries](19-voyager-skill-libraries.md), [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md), [71-karpathy-skills-single-file-guardrails](71-karpathy-skills-single-file-guardrails.md), [79-skill-rag](79-skill-rag.md) discuss in different ways.

Three concrete demonstrations:

1. **Capability portability across runtimes.** The same `skills/` directory installs into Pi (default), Codex (`~/.codex/skills/feynman`), repo-local (`.agents/skills/feynman`), and OpenCode (`.opencode/skills/feynman`). Skills are not tied to Pi.
2. **Subagents as Markdown files.** The four subagent definitions (researcher, reviewer, writer, verifier) are *editable Markdown* with frontmatter. To change subagent behaviour, you edit the file. There is no compiled prompt template, no embedded SDK config, no proprietary glue. This is the maximal expression of skills-as-portable-artifacts.
3. **Composition via slash commands.** Each prompt expands to invoke skills + subagents in a particular sequence. The composition layer is itself just Markdown.

The result is that *contributing to Feynman is editing Markdown files*, not learning a framework. New skills are new directories with a `SKILL.md`. New workflows are new prompt files. The core code (TypeScript, ~169 commits) is the runtime glue; the *capability* is in the Markdown.

## Limitations and tensions

A few honest limitations:

1. **Domain-specific.** Feynman is opinionated for research. Repurposing it as a coding harness or customer-support harness is possible but loses the alignment between subagent design and workflow. For non-research domains, the patterns transfer; the codebase is less directly useful.
2. **Pi-coupling.** While skills install into multiple runtimes, the full Feynman experience (CLI, slash commands, settings) requires Pi. Other Pi-compatible runtimes are nascent.
3. **alphaXiv as a primary search source.** alphaXiv is excellent for arXiv but doesn't cover non-arXiv academic sources well (proceedings papers behind paywalls, books, technical reports). The web search backends (Exa, Perplexity, Gemini) fill some of this in.
4. **Cost can spiral.** Multi-agent runs with verification, web fetches, and replication on cloud GPUs add up fast. The slash commands give no built-in cost ceiling.
5. **No memory tier.** Feynman has session search ([session-log], [session-search] skills) but no first-class persistent memory architecture in the MEMTIER ([151-153](151-memtier-why-flat-memory-breaks-at-72-hours.md)) sense. For very-long-horizon research watching ([watch] skill), this could become a degradation issue.

The MEMTIER + Feynman combination is interesting — Feynman could plausibly use MEMTIER as its substrate for persistent research memory, with the slug acting as the project key, and the four agents reading/writing structured episodic + semantic tiers instead of free-form research files. That is a project waiting to happen.

## Implementation lessons for harness builders

Five concrete takeaways if you are designing your own harness:

### 1. Make subagents editable Markdown

If your subagent definitions live in code (Python classes, TypeScript modules, dictionary configs), you have raised the floor for contribution. Markdown subagents with frontmatter are the lowest-friction primitive: anyone reading the file can understand what the subagent does and edit it.

### 2. Treat verification as a structural pass, not a guideline

Build a Verifier subagent. Give it the *authority to remove* unsupported claims. Run it after the Writer. Do not bundle writing and citation into the same step.

### 3. Externalise plans and decisions

For workflows that span more than ~3 tool calls, write a plan artifact at the start, evolve it during the run, and treat it as the working memory of record. Don't rely on in-context tracking.

### 4. Slug-key everything

Pick a deterministic per-run identifier. Prefix every artifact with it. Forbid generic names. Concurrent runs and resumability both come for free.

### 5. File-based subagent handoffs

Subagents return paths and one-line summaries. Parent agents read selectively. Inline result-dumping is a tax on parent context and a source of failure.

## When to use Feynman

Use Feynman directly when:
- You are doing academic-style research (literature reviews, paper audits, replication studies, watch lists).
- You want source-grounded outputs as a default.
- You are comfortable installing a Node-bundled CLI and configuring LM provider keys.
- You want a stable artifact format (slug-keyed Markdown) for long-running work.

Use Feynman *as a reference design* when:
- You are building a different-domain harness and want to study source-grounding patterns.
- You are migrating away from a "just dump the LLM output" workflow toward something rigorous.
- You want to see what well-factored multi-agent + Markdown-skill harness looks like in production.

Do not use Feynman when:
- You need a coding agent (use Claude Code, Cursor, or OpenClaw).
- You need real-time interactive UX (Feynman is CLI-first).
- You need fine-grained control over agent lifecycle that the four-subagent model doesn't fit.

## Where this fits in the harness arc

- **Read alongside**: [29-dive-into-claude-code](29-dive-into-claude-code.md), [42-langchain-deep-agents](42-langchain-deep-agents.md), [52-dive-into-open-claw](52-dive-into-open-claw.md), [61-archon-harness-builder](61-archon-harness-builder.md), [62-everything-claude-code](62-everything-claude-code.md), [63-ragflow-agent-patterns](63-ragflow-agent-patterns.md), [64-lobehub-ai-framework](64-lobehub-ai-framework.md), [65-deer-flow-bytedance](65-deer-flow-bytedance.md).
- **Multi-agent coordination patterns**: [02-subagent-delegation](02-subagent-delegation.md), [20-metagpt-role-based-workflows](20-metagpt-role-based-workflows.md), [73-multica-managed-agents-platform](73-multica-managed-agents-platform.md), [121-multi-agent-coordination-patterns](121-multi-agent-coordination-patterns.md), [127-loan-processing-multi-agent-case-study](127-loan-processing-multi-agent-case-study.md).
- **Skills layer**: [04-skills](04-skills.md), [19-voyager-skill-libraries](19-voyager-skill-libraries.md), [68-atomic-skills-scaling-coding-agents](68-atomic-skills-scaling-coding-agents.md), [71-karpathy-skills-single-file-guardrails](71-karpathy-skills-single-file-guardrails.md), [79-skill-rag](79-skill-rag.md), [89-voyager-deep](89-voyager-deep.md), [154-ctx2skill-self-evolving-context-skills](154-ctx2skill-self-evolving-context-skills.md), [156-heavyskill-parallel-reasoning-deliberation](156-heavyskill-parallel-reasoning-deliberation.md).
- **Verification, evaluation, integrity**: [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [18-chain-of-verification-self-refine](18-chain-of-verification-self-refine.md), [21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md), [38-claw-eval](38-claw-eval.md), [115-evaluating-llm-systems](115-evaluating-llm-systems.md), [122-explainability-compliance](122-explainability-compliance.md), [135-trustworthy-generation](135-trustworthy-generation.md).
- **Persistent state and continuity**: [09-memory-files](09-memory-files.md), [10-multi-session-continuity](10-multi-session-continuity.md), [12-todo-scratchpad-state](12-todo-scratchpad-state.md), [72-claude-mem-persistent-memory-compression](72-claude-mem-persistent-memory-compression.md), [144-build-your-own-harness](144-build-your-own-harness.md), [150-temporal-durable-execution-substrate](150-temporal-durable-execution-substrate.md), [151-memtier-why-flat-memory-breaks-at-72-hours](151-memtier-why-flat-memory-breaks-at-72-hours.md).
- **Synthesis next**: [157-may-2026-synthesis](157-may-2026-synthesis-memory-and-skills.md).

## References

1. Feynman repository — https://github.com/getcompanion-ai/feynman.
2. Pi runtime — https://github.com/badlogic/pi-mono.
3. Pi skills — https://github.com/badlogic/pi-skills.
4. alphaXiv — https://www.alphaxiv.org/.
5. Feynman docs — https://feynman.is/docs.
6. Feynman release notes — `RELEASES.md` in the repository.
7. AGENTS.md (in-repo) — repo-level agent contract.
8. `.feynman/agents/researcher.md`, `.feynman/agents/reviewer.md`, `.feynman/agents/writer.md`, `.feynman/agents/verifier.md` — subagent definitions.
9. `prompts/deepresearch.md`, `prompts/lit.md`, `prompts/audit.md`, `prompts/replicate.md` — workflow templates.
10. The 19 `skills/` SKILL.md files — capability layer definitions.
11. Adjacent harness chapters: [29](29-dive-into-claude-code.md), [42](42-langchain-deep-agents.md), [52](52-dive-into-open-claw.md), [61](61-archon-harness-builder.md), [62](62-everything-claude-code.md), [63](63-ragflow-agent-patterns.md), [64](64-lobehub-ai-framework.md), [65](65-deer-flow-bytedance.md), [66](66-meta-harness-landscape.md), [144](144-build-your-own-harness.md), [145](145-comparing-coding-harnesses.md).
