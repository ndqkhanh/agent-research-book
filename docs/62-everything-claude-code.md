# 62 — Everything Claude Code: The Community Harness Optimization Kit

**Definition.** Everything Claude Code (ECC, [github.com/affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code)) is an MIT-licensed, cross-harness **plugin bundle** — a curated collection of agents, skills, commands, rules, hooks, and MCP configs — that installs into Claude Code, Cursor, Codex, OpenCode, Antigravity, and Gemini from a single source of truth. Built by Affaan Mustafa during the *Built with Opus 4.6* Cerebral Valley × Anthropic hackathon (Feb 10–16, 2026) and iterated publicly ever since, it has become one of the most-starred harness-layer projects of 2026. As of 2026-04-20 the GitHub API reports **161,754 stars, 25,145 forks, 131 open issues, 835 watchers, 30 MB repo size, default branch `main`**, created 2026-01-18, last pushed 2026-04-19. License: **MIT**.

ECC is not a new agent harness. It is a **configuration-layer product**: take the harnesses you already have (Claude Code primary, five others as porting targets) and layer opinionated defaults, safety rails, evaluation hooks, and reusable knowledge on top. The thesis is that most of the *felt* quality difference between two Claude Code users comes from what lives in `~/.claude/` — agents, skills, hooks, rules — not from the binary itself. ECC is a community-maintained baseline for that directory.

## Problem it solves

Out-of-the-box Claude Code is a capable harness but an empty kitchen. A new user gets an agent loop, a tool catalog, and a blank `CLAUDE.md`; what to *put* in it is left as an exercise. Ten thousand users independently reinvent the same `/tdd` command, the same "run prettier on save" hook, the same "don't `rm -rf`" pre-tool guard. ECC's value proposition is **crystallized community defaults**: one `/install` gets you a battle-tested bundle that would otherwise take weeks to assemble. It also solves a cross-tool **drift** problem — a team using Claude Code and Cursor and Codex would otherwise maintain three sets of near-duplicate configs. ECC's adapter pattern keeps them in lockstep from one repo.

A secondary problem: the Claude Code docs describe *mechanisms* (skills, hooks, subagents) but not *worked systems*. ECC is a reference implementation people can read, fork, and cargo-cult from. That makes it as much a **teaching artifact** as a product.

## Scope: what the repo is (and isn't)

ECC is best described as **a cookbook-meta-framework hybrid**. Three framings, each partially true:

- **Collection/cookbook.** The vast majority of the repo is content: ~48 subagent definitions, 116–183 skill folders (depending on release), 79 slash-command shims, 34 rule files, ~20+ hook scripts. Each is a discrete, copyable artifact. A user can cherry-pick.
- **Plugin / marketplace bundle.** The repo ships as a Claude Code plugin installable through the marketplace channel (and manually via clone). A single install command deposits commands, agents, and skills into the right `~/.claude/` subdirectories. Rules require a separate step due to a platform limitation in Claude Code's plugin format.
- **Meta-framework.** The recent `ecc2/` directory contains an alpha **Rust control-plane prototype** and the repo's roadmap talks about a "performance optimization system" with instinct extraction, confidence scoring, and continuous learning. That is harness-level work — not just content — but it is marked alpha and is not the main user surface.

It is **not** an alternative agent loop (contrast OpenClaw, [52-dive-into-open-claw.md](52-dive-into-open-claw.md)), not a research framework (contrast SemaClaw, [54-semaclaw-general-purpose-agent.md](54-semaclaw-general-purpose-agent.md)), and not an SDK. You still run Claude Code; ECC just dresses it up.

## Key patterns shipped — mapped to the corpus

Every ECC component maps to a mechanism this corpus already documents. That is the clearest way to read the repo: it is a **concrete library of the abstract patterns** in docs 04–09 and the harness-engineering principles in docs 40–46.

### Skills (~116–183 skill folders) → [04-skills.md](04-skills.md)

ECC's `skills/` tree is the largest publicly-available collection of `SKILL.md` definitions as of April 2026. Top-level categories observed include: `agent-harness-construction`, `agent-eval`, `agentic-engineering`, `ai-first-engineering`, `backend-patterns`, `frontend-patterns`, `e2e-testing`, `django-tdd`, `django-security`, `defi-amm-security`, `healthcare-phi-compliance`, `customs-trade-compliance`, `docker-patterns`, `deployment-patterns`, `eval-harness`, `dashboard-builder`, and language-specific folders for C++, C#, Go, Java, Dart/Flutter, .NET. This instantiates the skill pattern from [04-skills.md](04-skills.md) at scale — progressive disclosure across a hundred+ capabilities that would otherwise blow the context window. Many skills also demonstrate the Voyager-style [19-voyager-skill-libraries.md](19-voyager-skill-libraries.md) reuse pattern: new skills can be *generated* from session history via the `/skill-create` and `/learn` commands.

Note: skill counts diverge between ECC's release notes (v1.10.0 cites 116 skills; other README material cites 183). The discrepancy is unverified — the larger number likely includes variants, nested skills, and the `homunculus/instincts/inherited` pattern files (see below).

### Hooks (~20+ scripts across 8+ event types) → [05-hooks.md](05-hooks.md)

ECC ships a near-complete matrix of Claude Code hook events, implemented as cross-platform Node.js scripts (for Windows compatibility):

- **PreToolUse:** dev-server blocker, tmux reminder, git-push reminder, pre-commit quality gate, doc-file warnings, strategic-compact suggestions.
- **PostToolUse:** PR logger, build analysis, quality gate on `Edit`, design-quality check, prettier auto-format, TypeScript type-check, `console.log` warning.
- **Lifecycle (SessionStart, SessionEnd, PreCompact, Stop):** session context loader, pre-compaction state preservation, console.log audit on modified files, session summaries, pattern extraction, cost-tracking telemetry, desktop notifications, cleanup markers.

This covers every use case in [05-hooks.md](05-hooks.md) — invariant enforcement, injection, refusal — plus runtime controls via environment variables (`ECC_HOOK_PROFILE`, `ECC_DISABLED_HOOKS`) so users can toggle behavior without editing config. The `PreCompact` state-preservation hook is a particularly clean implementation of the idea in [08-context-compaction.md](08-context-compaction.md): checkpoint durable state *before* the harness forgets it.

### Permissions — indirect coverage → [06-permission-modes.md](06-permission-modes.md)

ECC does not define new permission modes (Claude Code's Plan/Accept-Edits/Auto/Bypass model is untouched), but **AgentShield** (see below) scans a harness config for permission misconfigurations — an auditor for the permission layer rather than a redesign of it.

### Memory → [09-memory-files.md](09-memory-files.md)

ECC's distinctive contribution here is the `homunculus/instincts/inherited/` tree plus a documented "instinct" concept: session patterns with confidence scores that get auto-extracted and persisted as durable memory artifacts. This is memory-files (docs 09) fused with a Voyager-style skill library (docs 19) and a self-improvement loop (contrast [55-hermes-agent-self-improving.md](55-hermes-agent-self-improving.md) and [45-hyperagents-self-modification.md](45-hyperagents-self-modification.md)). The `v2` confidence-scoring iteration is a direct response to the "junk-memory accumulation" failure mode memory systems suffer when every session writes unfiltered patterns back.

### Subagents (~48 agent markdowns) → [02-subagent-delegation.md](02-subagent-delegation.md)

ECC's agent catalog is wide. Observed agents include:

- **Review:** `code-reviewer`, `security-reviewer`, `a11y-architect`, `architect`, `database-reviewer`, `healthcare-reviewer`.
- **Language reviewers:** `typescript-reviewer`, `python-reviewer`, `go-reviewer`, `java-reviewer`, `kotlin-reviewer`, `rust-reviewer`, `cpp-reviewer`, `csharp-reviewer`, `flutter-reviewer`.
- **Build resolution:** `build-error-resolver`, plus `cpp-build-resolver`, `go-build-resolver`, `rust-build-resolver`, `java-build-resolver`, `kotlin-build-resolver`, `dart-build-resolver`, `pytorch-build-resolver`.
- **Navigation / authoring:** `code-architect`, `code-explorer`, `code-simplifier`, `doc-updater`, `docs-lookup`, `performance-optimizer`, `planner`, `refactor-cleaner`, `tdd-guide`, `type-design-analyzer`.
- **Meta / specialty:** `chief-of-staff`, `harness-optimizer`, `loop-operator`, `conversation-analyzer`, `comment-analyzer`, `e2e-runner`, `gan-evaluator`, `gan-generator`, `gan-planner`, `pr-test-analyzer`, `silent-failure-hunter`, `seo-specialist`, `opensource-forker`, `opensource-packager`, `opensource-sanitizer`.

Language-specific reviewer/resolver pairs show the *bounded subagent* component from Raschka's decomposition [46-components-of-coding-agent.md](46-components-of-coding-agent.md) — each agent has a tight scope and its own prompt, avoiding the "one god-agent does everything" antipattern. The GAN triplet (generator/evaluator/planner) is a direct application of the LLM-as-judge pattern [21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md) to iterative refinement.

### Commands (~79 slash commands)

`/tdd`, `/plan`, `/e2e`, `/code-review`, `/build-fix`, `/learn`, `/skill-create`, and dozens of others. Each is a one-line entry point to a skill + subagent pipeline. This is **harness-level UX** (docs 41, 43) — collapsing a multi-step best practice into a single invocation a user can remember.

### Rules (34 files, `common/` + language dirs)

Always-in-context guidelines for security, coding style, git workflow, testing. These are the lowest-disclosure tier — instructions every turn sees — and ECC's choice to partition them into `common/` plus `typescript/`, `python/`, `go/`, `swift/`, `php/` trees mirrors the context-reduction principle in [08-context-compaction.md](08-context-compaction.md): only load the language rules relevant to the current file.

### AgentShield — security auditor for harness configs

ECC claims 1,282 tests and 102 static analysis rules (unverified independently) that scan a harness configuration for vulnerabilities, misconfigurations, and prompt-injection risks. This is a **harness self-auditor**, conceptually adjacent to [22-guardrails-prompt-injection.md](22-guardrails-prompt-injection.md) and [49-agents-of-chaos-red-teaming.md](49-agents-of-chaos-red-teaming.md) but applied at the config layer: "what dangerous tool paths did you accidentally allowlist?" Well-suited to enterprise rollout where someone has to sign off that a Claude Code install is policy-compliant.

### ECC 2.0 (Rust control plane, alpha)

The `ecc2/` directory is an alpha Rust prototype billed as a control plane. Details are thin in public docs; the direction echoes the product-control-plane thesis in [41-product-control-plane.md](41-product-control-plane.md). Production-readiness: **not there yet**.

## Notable examples / skills / subagents worth studying

- **`agent-harness-construction` skill.** A skill *about* building harnesses. Directly relevant to this corpus' subject matter.
- **`agent-eval` and `eval-harness` skills.** Concrete templates for the evaluator-loop pattern from [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md) and [38-claw-eval.md](38-claw-eval.md).
- **`autonomous-loops` and `continuous-agent-loop` skills.** Packaged long-horizon loops — practical counterparts to [27-horizon-long-horizon-degradation.md](27-horizon-long-horizon-degradation.md).
- **`silent-failure-hunter` agent.** A subagent whose job is to find tests or checks that pass but shouldn't — an applied version of the epistemic-honesty theme in [18-chain-of-verification-self-refine.md](18-chain-of-verification-self-refine.md).
- **`harness-optimizer` agent.** Self-referential — the harness includes an agent for tuning the harness. Aligns with [45-hyperagents-self-modification.md](45-hyperagents-self-modification.md), albeit as a human-in-the-loop recommender rather than an autonomous modifier.
- **Industry-vertical skills.** `healthcare-phi-compliance`, `healthcare-emr-patterns`, `defi-amm-security`, `customs-trade-compliance`. Gestures at a domain-specialized-agent direction (compare [30-gpt-rosalind-domain-specialized.md](30-gpt-rosalind-domain-specialized.md)) but the depth varies — some are substantive checklists, others thin.

## Extension model — how contributors add things

ECC's `CONTRIBUTING.md` / `CLAUDE.md` prescribe a **flat, format-based** contribution model. Every artifact type has a required shape:

- **Agents** — Markdown with YAML frontmatter: `name`, `description`, `tools`, `model`. File naming: lowercase-hyphen.
- **Skills** — Markdown with sections: *When to Use*, *How It Works*, *Examples*. Curated skills in `skills/`; generated/imported skills under `~/.claude/skills/` per `docs/SKILL-PLACEMENT-POLICY.md`.
- **Commands** — Markdown with description frontmatter.
- **Hooks** — JSON with `matcher` + `hooks` array; actual logic in cross-platform Node.js scripts under `scripts/`.
- **Rules** — Markdown, filed under `common/` or a language directory.

There is a test suite (`node tests/run-all.js`) covering hook scripts and utilities. Contributors add a new artifact by dropping a correctly-formatted file in the right directory — no code changes or framework extension required. That flatness is the whole reason community contributions scaled: you do not need to understand the plugin internals to ship a new skill.

Manifest-driven selective installation means a contributor can also add a *profile* — e.g. "only the Python + Django skills" — without touching the content.

## Relationship to Claude Code itself — companion, not alternative

ECC is unambiguously a **companion** to Claude Code, not a replacement. It depends on Claude Code's runtime (agent loop, tool execution, permission mode machine, hook event bus) and only contributes into the extension points Anthropic exposes. The interesting wrinkle is that ECC also targets **five sibling harnesses** — Cursor, Codex, OpenCode, Antigravity, Gemini — from the same source via adapter patterns. The `.opencode/` and Cursor-specific directories are ports, not forks. For the corpus reader, ECC is evidence that the Claude Code extension surface (skills + hooks + agents + rules + MCP) has become a **portable contract** other harnesses can honor to some degree, which is a useful data point for [46-components-of-coding-agent.md](46-components-of-coding-agent.md) and [40-harness-engineering-principles.md](40-harness-engineering-principles.md).

## Comparison

### vs OpenClaw ([52-dive-into-open-claw.md](52-dive-into-open-claw.md))

OpenClaw **replaces** the harness (its own Gateway + pluggable harness plugins); ECC **enriches** an existing one. OpenClaw's contribution is architectural (a new loop, new channels, user-sovereign security); ECC's contribution is content (skills, agents, hooks, rules you drop into the harness you already run). They are complementary — you could theoretically run an ECC-style content bundle *inside* OpenClaw if the harness plugin supports the same extension primitives.

### vs SemaClaw ([54-semaclaw-general-purpose-agent.md](54-semaclaw-general-purpose-agent.md))

SemaClaw is a research artifact — a paper and a reference implementation advancing four named contributions (DAG Teams, PermissionBridge, three-tier context, agentic wiki). ECC is a community product. SemaClaw introduces new mechanisms; ECC packages existing ones. A team that wanted SemaClaw's PermissionBridge semantics on top of ECC would have to build it — ECC's AgentShield is an auditor, not a runtime authorization token system.

### vs this corpus' 10 in-tree MVP projects

The `../projects/` tree (`aegis-ops`, `atlas-research`, `cipher-sec`, `harmony-voice`, `helix-bio`, `mentat-learn`, `orion-code`, `quanta-proof`, `syndicate`, `vertex-eval`) are **vertical** MVPs: each is a scoped end-to-end agent for a domain. ECC is **horizontal** infrastructure: generic skills and agents reusable across any coding domain. They intersect — an MVP project could adopt ECC hooks and subagents wholesale — but do not compete. The closest analogue is if one of the MVPs shipped a skill catalog; that catalog would want to plug into ECC's formats for distribution.

## Production-readiness assessment

**Green (ready):**
- Skill / agent / command / rule content — stable formats, large test matrix, used by tens of thousands of stars' worth of downstream developers.
- Hook scripts — Node.js, cross-platform, Windows CI passing per v1.10.0 notes.
- Selective installation — manifest-driven.

**Amber (use with care):**
- Skill count discrepancies (116 vs 183) and "1,282 tests / 98% coverage / 102 rules" marketing numbers are **self-reported** and not independently verified — treat as directionally correct rather than audited.
- Instinct / continuous-learning system: v2 confidence scoring helps, but auto-writing memory from sessions is a known failure mode in this corpus ([09-memory-files.md](09-memory-files.md)'s "curate, don't hoard" warning). Watch it.
- Cross-harness parity: adapter directories for Cursor, Codex, OpenCode, Antigravity, Gemini exist but depth varies; Claude Code is clearly primary.

**Red (not ready):**
- **ECC 2.0 (`ecc2/` Rust control plane)** — marked alpha. Don't depend on it.
- **Dashboard GUI** (Tkinter desktop app) — useful for exploration, not for production operation.
- Industry-vertical skills (healthcare, DeFi, customs) — uneven depth; audit before using in a regulated context.

On a scale of "cookbook" to "framework": ECC is solidly cookbook-plus. Its *content* is production-ready for augmenting an existing Claude Code install. Its *self-improving loop and control plane* are research-grade.

## Takeaways — copy-worthy ideas

1. **Flat, format-based contribution model.** Every artifact type is a markdown/JSON file with a prescribed shape. No plugin API to learn. This is the single biggest lesson — extension surfaces scale when contributors do not need to understand internals.
2. **Env-var hook controls.** `ECC_HOOK_PROFILE` and `ECC_DISABLED_HOOKS` let a user mute noisy hooks without editing config. Steal this for any hook-heavy harness.
3. **Cross-platform hook scripts from day one.** Node.js is not glamorous, but Windows CI passing is a marker that the project took portability seriously — most community bundles do not.
4. **AgentShield as a pattern.** A static analyzer for your own config. If you ship a harness with a large extension surface, you need one.
5. **Separate curated vs generated skill locations.** `skills/` vs `~/.claude/skills/` — the `SKILL-PLACEMENT-POLICY.md` is a small idea that prevents `/learn`-generated artifacts from polluting the curated catalog. Applies to any system with auto-generated durable artifacts.
6. **Language-specific reviewer + build-resolver pairs.** Bounded subagents with narrow scope consistently outperform one general reviewer. ECC's taxonomy is a working example.
7. **Confidence scoring on extracted instincts (v2).** If you auto-write memory from sessions, attach a confidence signal from the start. You will need it.
8. **Portable extension contract across six harnesses.** ECC demonstrates that skills + hooks + agents + rules + MCP is portable enough to be a de-facto cross-harness standard. For harness designers in 2026, honoring this contract gets you a community of content for free.

## References

Verified 2026-04-20 via GitHub API + WebFetch/WebSearch:

- [github.com/affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code) — primary repo (161,754 stars, 25,145 forks, MIT, default branch `main`, created 2026-01-18, last push 2026-04-19).
- [github.com/affaan-m/everything-claude-code/blob/main/CLAUDE.md](https://github.com/affaan-m/everything-claude-code/blob/main/CLAUDE.md) — contributor-facing architecture summary.
- [github.com/affaan-m/everything-claude-code/releases](https://github.com/affaan-m/everything-claude-code/releases) — release history; v1.10.0 (2026-04-05), v1.9.0 (2026-03-21), v1.8.0 (2026-03-05), v1.7.0 (2026-02-27), v1.6.0 (2026-02-24).
- [cerebralvalley.ai/e/claude-code-hackathon](https://cerebralvalley.ai/e/claude-code-hackathon) — *Built with Opus 4.6* hackathon origin, Feb 2026.
- [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents) — Claude Code subagent docs (extension surface ECC targets).

Unverified (as noted in-text): specific test / coverage / rule counts, agent / skill totals where release notes and README disagree.

Cross-references in this corpus:
- [02-subagent-delegation.md](02-subagent-delegation.md)
- [04-skills.md](04-skills.md)
- [05-hooks.md](05-hooks.md)
- [06-permission-modes.md](06-permission-modes.md)
- [08-context-compaction.md](08-context-compaction.md)
- [09-memory-files.md](09-memory-files.md)
- [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md)
- [19-voyager-skill-libraries.md](19-voyager-skill-libraries.md)
- [21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md)
- [22-guardrails-prompt-injection.md](22-guardrails-prompt-injection.md)
- [27-horizon-long-horizon-degradation.md](27-horizon-long-horizon-degradation.md)
- [29-dive-into-claude-code.md](29-dive-into-claude-code.md)
- [38-claw-eval.md](38-claw-eval.md)
- [40-harness-engineering-principles.md](40-harness-engineering-principles.md)
- [41-product-control-plane.md](41-product-control-plane.md)
- [45-hyperagents-self-modification.md](45-hyperagents-self-modification.md)
- [46-components-of-coding-agent.md](46-components-of-coding-agent.md)
- [49-agents-of-chaos-red-teaming.md](49-agents-of-chaos-red-teaming.md)
- [52-dive-into-open-claw.md](52-dive-into-open-claw.md)
- [54-semaclaw-general-purpose-agent.md](54-semaclaw-general-purpose-agent.md)
- [55-hermes-agent-self-improving.md](55-hermes-agent-self-improving.md)
