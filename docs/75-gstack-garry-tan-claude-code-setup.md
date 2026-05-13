# 75 — garrytan/gstack: The Maximum-Ambition Claude-Code Stack as Virtual Engineering Team

**Definition.** gstack ([github.com/garrytan/gstack](https://github.com/garrytan/gstack)) is **Garry Tan's personal, open-sourced, production-grade Claude Code configuration** — a set of **23 specialist skills + 8 power tools + 2 standalone CLIs + a custom Chromium browser + team-mode auto-update infrastructure**, all wrapped as slash commands and Markdown, shipped under MIT license. The thesis is compact: *turn Claude Code into a virtual engineering team — a CEO who rethinks the product, an eng manager who locks architecture, a designer who catches AI slop, a reviewer who finds production bugs, a QA lead who opens a real browser, a security officer who runs OWASP + STRIDE audits, and a release engineer who ships the PR*. The author is [Garry Tan](https://x.com/garrytan), President & CEO of Y Combinator, former Palantir engineer, Posterous co-founder (acquired by Twitter), builder of Bookface (YC's internal social network). His claim, backed by public reproduction scripts: his 2026 logical-lines-of-code-per-day run rate is **~810× his 2013 pace** (11,417 vs 14 LLOC/day) — shipping 3 production services and 40+ features in 60 days while running YC full-time. gstack is his explanation of *how*.

This is the **maximum-ambition** personal Claude Code setup in the community, the counterpart to [71-karpathy-skills-single-file-guardrails.md](71-karpathy-skills-single-file-guardrails.md)'s **minimum-ambition** 60-line CLAUDE.md. Both are valid approaches; they bracket the spectrum of how deeply a developer can customize Claude Code. gstack is also one of the strongest existence proofs that **a single builder with the right tooling can out-ship a traditional team** — a thesis Karpathy publicly endorsed in March 2026 ("I don't think I've typed like a line of code probably since December, basically").

## The thesis — "my open source software factory"

Garry Tan's framing is explicit:

> "When I heard Karpathy say this [AI agents changed my coding output], I wanted to find out how. How does one person ship like a team of twenty? Peter Steinberger built OpenClaw — 247K GitHub stars — essentially solo with AI agents. The revolution is here. A single builder with the right tooling can move faster than a traditional team."

> "**gstack is my answer.** I've been building products for twenty years, and right now I'm shipping more products than I ever have."

The productivity metric is measured and reproducible:
- **2013 baseline** (building Bookface at YC): 14 logical-lines-of-code/day.
- **2026 run rate** (with gstack, part-time while running YC full-time): 11,417 LLOC/day.
- **Year-to-date 2026** (through April 18): 240× the entire 2013 year.
- Measured across 40 public + private `garrytan/*` repos including Bookface.

The LLOC methodology excludes raw LOC inflation from AI (gstack-authored code generates substantial boilerplate that the metric filters). Full methodology in [docs/ON_THE_LOC_CONTROVERSY.md](https://github.com/garrytan/gstack/blob/main/docs/ON_THE_LOC_CONTROVERSY.md).

The **central claim**: the 810× acceleration is not from better keystrokes — it's from replacing keystrokes entirely with a structured agent workflow. gstack is the structured workflow.

## Architecture — the seven-phase sprint

gstack organizes its 23 skills into a **sprint process**:

**Think → Plan → Build → Review → Test → Ship → Reflect**

Each skill occupies a phase; each skill writes artifacts the next phase reads. This is a *process* encoded as *composable skills*, not a collection of independent commands. `/office-hours` writes a design doc that `/plan-ceo-review` reads. `/plan-eng-review` writes a test plan that `/qa` picks up. `/review` catches bugs that `/ship` verifies are fixed.

### Phase 1 — Think (reframe before code)

| Skill | Specialist | Purpose |
|---|---|---|
| `/office-hours` | YC Office Hours | Six forcing questions that reframe your product. Challenges framing, extracts hidden capabilities, generates alternatives with effort estimates. Writes design doc. |

**Example from the README** — user says "I want to build a daily briefing app." `/office-hours` pushes back: *"I'm going to push back on the framing. You said 'daily briefing app.' But what you actually described is a personal chief of staff AI."* Extracts 5 hidden capabilities, challenges 4 premises, generates 3 approaches with effort estimates, recommends shipping narrowest wedge.

The pattern is **YC's Office Hours methodology encoded as an agent skill**. This is unusual and powerful — the same reframing discipline Garry applies when advising YC founders, available as `/office-hours` to anyone.

### Phase 2 — Plan (lock scope, architecture, design, DX)

| Skill | Specialist | Purpose |
|---|---|---|
| `/plan-ceo-review` | CEO / Founder | Rethink the problem. 4 modes: Expansion / Selective Expansion / Hold Scope / Reduction. |
| `/plan-eng-review` | Eng Manager | ASCII diagrams, data flow, state machines, error paths, test matrix. Forces hidden assumptions into the open. |
| `/plan-design-review` | Senior Designer | Rates each design dimension 0-10, explains what a 10 looks like, edits plan to get there. AI Slop detection. |
| `/plan-devex-review` | DX Lead | Explores developer personas, benchmarks competitor TTHW, designs your "magical moment." 20-45 forcing questions. |
| `/design-consultation` | Design Partner | Builds design system from scratch — researches landscape, proposes creative risks, generates realistic product mockups. |
| `/autoplan` | Review Pipeline | Runs CEO → design → eng → DX review automatically. Auto-detects which apply. Surfaces only taste decisions. |

**Key insight:** each planning specialist has a **forcing-question catalog** — numbered, ordered questions that the agent must work through before generating output. This is structured interrogation as agent behavior, not free-form chat.

### Phase 3 — Build (autonomous coding after planning gates)

gstack does not have a dedicated "build" skill — it relies on Claude Code's native implementation once the plan is approved. The discipline is that **you don't skip to build**; planning gates (`/plan-eng-review`) close first.

### Phase 4 — Review (find bugs before CI, find bugs CI missed)

| Skill | Specialist | Purpose |
|---|---|---|
| `/review` | Staff Engineer | Find bugs that pass CI but blow up in production. Auto-fixes obvious ones, flags completeness gaps. |
| `/investigate` | Debugger | Systematic root-cause debugging. Iron Law: *no fixes without investigation*. Traces data flow, tests hypotheses, stops after 3 failed fixes. |
| `/design-review` | Designer Who Codes | Same audit as `/plan-design-review`, but on shipped code. Fixes what it finds with atomic commits + before/after screenshots. |
| `/devex-review` | DX Tester | Live DX audit. Actually navigates docs, tries getting-started flow, times TTHW. Boomerangs to `/plan-devex-review` scores to show plan vs reality. |
| `/codex` | Second Opinion | Independent review from OpenAI Codex CLI. 3 modes: review (pass/fail gate), adversarial challenge, open consultation. Cross-model analysis. |
| `/cso` | CSO | OWASP Top 10 + STRIDE threat model. 17 false-positive exclusions, 8/10+ confidence gate, independent verification. |

### Phase 5 — Test (QA with a real browser)

| Skill | Specialist | Purpose |
|---|---|---|
| `/qa` | QA Lead | Tests app, finds bugs, fixes with atomic commits, re-verifies. Auto-generates regression tests for every fix. |
| `/qa-only` | QA Reporter | Same methodology but reports only, no code changes. |
| `/browse` | QA Engineer | Give agent eyes. Real Chromium, real clicks, real screenshots. ~100ms per command. |
| `/open-gstack-browser` | GStack Browser | Launches custom Chromium with anti-bot stealth, sidebar, auto-model routing. |

### Phase 6 — Ship (release engineering as an agent)

| Skill | Specialist | Purpose |
|---|---|---|
| `/ship` | Release Engineer | Sync main, run tests, audit coverage, push, open PR. Bootstraps test frameworks if none exist. |
| `/land-and-deploy` | Release Engineer | Merge PR, wait for CI and deploy, verify production health. One command from "approved" to "verified." |
| `/canary` | SRE | Post-deploy monitoring. Watches for console errors, perf regressions, page failures. |
| `/benchmark` | Performance Engineer | Baseline page load, Core Web Vitals, resource sizes. Compare before/after. |
| `/document-release` | Technical Writer | Update all docs to match what shipped. Auto-invoked by `/ship`. |

### Phase 7 — Reflect (learn from every sprint)

| Skill | Specialist | Purpose |
|---|---|---|
| `/retro` | Eng Manager | Team-aware weekly retro. Per-person breakdowns, shipping streaks, test health. `/retro global` across all projects. |
| `/learn` | Memory | Manage what gstack learned. Review, search, prune, export project-specific patterns, pitfalls, preferences. Compounds over sessions. |

### Power tools — non-sprint-phase utilities

| Skill | Specialist | Purpose |
|---|---|---|
| `/careful` | Safety Guardrails | Warns before destructive commands (rm -rf, DROP TABLE, force-push). |
| `/freeze` | Edit Lock | Restricts edits to one directory. Prevents accidental scope creep. |
| `/guard` | Full Safety | `/careful` + `/freeze`. |
| `/unfreeze` | Unlock | Removes `/freeze` boundary. |
| `/pair-agent` | Multi-Agent | Share browser with any AI agent (OpenClaw/Hermes/Codex/Cursor). Each gets its own tab, scoped tokens, tab isolation, rate limiting. |
| `/setup-deploy` | Deploy Configurator | One-time `/land-and-deploy` setup. |
| `/gstack-upgrade` | Self-Updater | Upgrade gstack. Syncs global + vendored. |

## Unique architectural properties

### A. Process-as-code — sprint phases materialized as skills

Most developer tooling provides discrete utilities. gstack provides a **process** — a seven-phase sprint where each phase has specialist skills and each skill's output feeds the next. You can't cheat steps without breaking the pipeline. This enforces discipline that individual skills cannot.

This is materially different from [71-karpathy-skills-single-file-guardrails.md](71-karpathy-skills-single-file-guardrails.md) — Karpathy-skills gives four general principles; gstack gives a phased workflow. Different philosophies: principles enforce *behavior*, process enforces *sequence*. Both are valid.

### B. Specialist role-play over generalist reasoning

Each gstack skill embodies a specific professional role (CEO, Eng Manager, Staff Engineer, Designer, QA Lead, Security Officer, SRE, etc.). The skill's system prompt contains role-specific:
- Decision frameworks (e.g., CEO-mode scope modes).
- Forcing-question catalogs.
- Output formats (ASCII diagrams, rating tables, before/after screenshots).
- Exit criteria.

This is **role-based prompting at scale** — not one "you are a senior engineer" line, but a full specialist persona with methodology, artifacts, and quality bars. The role-play pattern is present across the corpus ([02-subagent-delegation.md](02-subagent-delegation.md), [04-skills.md](04-skills.md)); gstack's contribution is doing it thoroughly and consistently for 23 roles.

### C. Multi-host distribution across 10 agent CLIs

gstack ships to all major agent CLIs:

```
~/.claude/skills/gstack-*/           # Claude Code (default)
~/.codex/skills/gstack-*/            # OpenAI Codex CLI
~/.config/opencode/skills/gstack-*/  # OpenCode
~/.cursor/skills/gstack-*/           # Cursor
~/.factory/skills/gstack-*/          # Factory Droid
~/.slate/skills/gstack-*/            # Slate
~/.kiro/skills/gstack-*/             # Kiro
~/.hermes/skills/gstack-*/           # Hermes
~/.gbrain/skills/gstack-*/           # GBrain (modded)
~/.openclaw/...                       # OpenClaw via ACP
```

Setup auto-detects installed agents and deploys to all of them with one command. Targeting a specific agent: `./setup --host codex`.

This is the **multi-surface delivery pattern** at its most aggressive — nine agent CLIs plus OpenClaw gateway, one codebase. Adding another agent is *"one TypeScript config file, zero code changes"* per the README.

### D. Team mode — auto-update without vendoring

From inside a repo:

```bash
(cd ~/.claude/skills/gstack && ./setup --team) && \
  ~/.claude/skills/gstack/bin/gstack-team-init required && \
  git add .claude/ CLAUDE.md && \
  git commit -m "require gstack for AI-assisted work"
```

Team mode:
- **No vendored files** in the repo — just a reference to gstack.
- **No version drift** — every Claude Code session auto-update-checks (throttled to once/hour, network-failure-safe, completely silent).
- **Opt-in blocker** — `required` blocks teammates without gstack; `optional` nudges them.

This addresses a real problem: most Claude Code extensions get vendored into the repo and diverge from upstream. Team mode keeps everyone on a live shared version without touching the repo.

### E. GStack Browser — custom Chromium with ML prompt-injection defense

gstack's most technically ambitious component. A custom Chromium build with:
- **Anti-bot stealth.** Google, NYTimes, etc. work without CAPTCHAs.
- **Custom branding.** Menu bar says "GStack Browser" instead of "Chrome for Testing." User's real Chrome stays untouched.
- **Sidebar agent.** Natural-language AI browser assistant in the Chrome side panel. Auto-routes to Sonnet for fast actions (click, navigate, screenshot) and Opus for analysis. Each task gets up to 5 minutes.
- **Auto model routing.** Fast actions → Sonnet. Reading + analysis → Opus. Cookie import one-click from sidebar footer.
- **Isolated session.** Sidebar agent doesn't interfere with main Claude Code window.
- **Browser handoff.** Hit a CAPTCHA? `$B handoff` opens visible Chrome at the same page with cookies. Solve the problem, `$B resume` picks up.

**The prompt-injection defense is novel.** A layered defense with:
1. **22MB ML classifier** bundled locally. Scans every page and tool output.
2. **Claude Haiku transcript check** — votes on full conversation shape.
3. **Canary tokens** in system prompt — caught if exfiltrated via text, tool args, URLs, file writes.
4. **Verdict combiner** — requires 2 classifiers to agree before blocking (prevents false positives on Stack Overflow instruction pages).
5. **Status icon** — shield in sidebar header (green/amber/red).
6. **Opt-in stronger ensemble.** `GSTACK_SECURITY_ENSEMBLE=deberta` loads 721MB DeBERTa-v3 for 2-of-3 agreement.
7. **Kill switch.** `GSTACK_SECURITY_OFF=1` emergency disable.

This is the **most thorough prompt-injection defense shipped in any open-source Claude Code extension** as of April 2026. See [ARCHITECTURE.md](https://github.com/garrytan/gstack/blob/main/ARCHITECTURE.md) for the full stack. Direct successor / operationalization of [22-guardrails-prompt-injection.md](22-guardrails-prompt-injection.md) and [35-malicious-intermediary-attacks.md](35-malicious-intermediary-attacks.md).

### F. Design-shotgun → design-html pipeline

Garry's most distinctive creative-workflow contribution:

1. **`/design-shotgun`** — describe what you want. Generates 4-6 AI mockup variants using GPT Image. Opens comparison board in browser. User picks favorites with feedback ("more whitespace", "bolder headline"). Iterates. **Taste memory** biases toward what user picks after a few rounds.
2. **`/design-html`** — take approved mockup, turn it into production HTML/CSS. Uses **Pretext** for computed text layout (text reflows on resize, heights adjust, layouts dynamic). 30KB, zero deps. Detects React / Svelte / Vue. Smart API routing (landing page vs dashboard vs form).

This is **visual iteration replacing verbal specification**. You stop describing your vision to Claude and start *picking from options*. Taste accumulates across projects via `gstack-taste-update` — a persistent per-project taste profile that decays 5%/week.

### G. Parallel-sprint scaling — 10-15 concurrent

> *"I regularly run 10-15 parallel sprints — that's the practical max right now."*

gstack composes with [Conductor](https://conductor.build) (parallel Claude Code sessions, each in its own workspace). The sprint structure is *what makes parallelism work* — without a process, ten agents are ten sources of chaos; with the think → plan → build → review → test → ship → reflect process, each agent knows exactly what to do and when to stop.

**Management model:** "You manage them the way a CEO manages a team: check in on the decisions that matter, let the rest run."

This is the operational counterpart to [02-subagent-delegation.md](02-subagent-delegation.md) at the single-developer scale. One human + 10-15 Claude Code sessions + Conductor + gstack = a team of 10-15 specialists working on different branches simultaneously.

### H. Cross-agent coordination via `/pair-agent`

> *"You're in Claude Code. You also have OpenClaw running. Or Hermes. Or Codex. You want them both looking at the same website. Type `/pair-agent`, pick your agent, and a GStack Browser window opens so you can watch."*

`/pair-agent` opens a shared browser, generates a one-time setup key, and prints a paste block for the other agent. The other agent exchanges the key for a session token, opens its own tab, and starts browsing. **Scoped tokens, tab isolation, rate limiting, domain restrictions, activity attribution.**

If ngrok is installed, tunnel starts automatically — the other agent can be on a different machine. This is **vendor-neutral multi-agent coordination through a shared browser** with real security properties — arguably the first production instance of cross-vendor agent-to-agent coordination outside academic research.

### I. Continuous checkpoint mode

Set `gstack-config set checkpoint_mode continuous` and skills auto-commit work with `WIP:` prefix plus structured `[gstack-context]` body (decisions, remaining work, failed approaches). Survives crashes and context switches. `/context-restore` reads the commits to reconstruct session state. `/ship` filter-squashes WIP commits before PR (preserving non-WIP commits) so `git bisect` stays clean.

Push is opt-in via `checkpoint_push=true` — default is local-only to avoid triggering CI on every WIP commit.

This is **agent-session-state as a git-native primitive**. The insight: git already has atomic commits; use it as the state backend rather than inventing a new persistence layer. Bonus: session recovery is just `git log`.

### J. `gstack-model-benchmark` — cross-model comparison

A standalone CLI that runs the same prompt through Claude, GPT (via Codex), and Gemini; compares latency, tokens, cost, and optional LLM-judge quality score. Auth detected per provider, unavailable providers skip cleanly. Output as table / JSON / markdown. `--dry-run` validates without spending credits.

**Use case:** when choosing which model to route a skill to, benchmark on your actual workload, not on generic benchmarks. This is the operational counterpart to [70-voltagent-awesome-ai-agent-papers.md](70-voltagent-awesome-ai-agent-papers.md)'s evaluation bucket — evaluation applied to model selection at the skill level.

## Relationship to karpathy-skills (71) — the spectrum of personal customization

gstack explicitly positions itself as the successor layer to karpathy-skills:

> *"Andrej Karpathy's AI coding rules (17K stars) nail four failure modes: wrong assumptions, overcomplexity, orthogonal edits, imperative over declarative. gstack's workflow skills enforce all four. `/office-hours` forces assumptions into the open before code is written. The Confusion Protocol stops Claude from guessing on architectural decisions. `/review` catches unnecessary complexity and drive-by edits. `/ship` transforms tasks into verifiable goals with test-first execution."*

> *"If you already use Karpathy-style CLAUDE.md rules, gstack is the workflow enforcement layer that makes them stick across entire sprints, not just single prompts."*

| Property | karpathy-skills | gstack |
|---|---|---|
| Files | 1 CLAUDE.md | 23 skills + 8 power tools + 2 CLIs + browser |
| Lines | ~60 | Tens of thousands |
| Install time | 30 seconds | ~5 min (30 sec for base install) |
| Target | Universal behavioral contract | Full engineering-team workflow |
| Complexity | Zero | High |
| Customization | Add CLAUDE.md sections | Fork entire stack |
| Adoption curve | 71K stars (mass adoption) | High ambition, smaller but devoted base |
| Author | Forrest Chang | Garry Tan |

**Recommended adoption path:**
1. Start with karpathy-skills. Get the four principles stable.
2. If those aren't enough, adopt specific gstack skills (likely `/review`, `/qa`, `/ship` first).
3. If you end up using 5+ gstack skills, go all-in with the full stack and team mode.

Both are MIT licensed. Both are designed to be customized, forked, and improved. The ecosystem is healthier for having both extremes.

## Patterns harness engineers should steal

1. **Encode your process as composable skills.** Don't ship a flat list of utilities. Encode the actual workflow (think → plan → build → review → test → ship → reflect) as ordered skills where each writes artifacts the next reads. Process-as-code enforces discipline.

2. **Specialist personas over generalist prompts.** "You are a senior engineer" is weak. Full role-play with decision frameworks, forcing-question catalogs, output formats, and exit criteria is strong. Each role should feel like a professional, not a persona.

3. **Gates before generation.** Planning skills gate build skills. Build skills gate ship skills. Ship skills gate canary skills. Reviewing agents catches hallucinations that builders would miss because the reviewer is in a different context.

4. **Multi-host from day one.** Supporting 10 agent CLIs is expensive but captures the full market. gstack's "one TypeScript config file per host" is the right abstraction. Invest in the adapter layer.

5. **Team mode without vendoring.** Auto-update checks on session start, throttled to once/hour, silent on failure. No vendored files. This is the right default for shared-repo tooling.

6. **Visual iteration for design.** `/design-shotgun` → `/design-html` replaces verbal specification with visual picking. For any creative domain, consider whether "generate 4-6 options, pick favorites, iterate" beats "describe what you want."

7. **Taste memory.** Persistent per-project taste profiles with decay (5%/week). Taste is learnable; don't force users to re-specify it every session.

8. **Parallel sprints need a process.** Parallelism without discipline produces chaos. Parallelism with a shared sprint process produces output. If your tool is intended for parallel agent use, specify the process as part of the tool.

9. **Cross-agent coordination via shared environment.** `/pair-agent` uses the browser as the shared substrate. For other domains, consider what environment is naturally shared (filesystem, database, API gateway) and build coordination protocols on top.

10. **Continuous checkpoint mode using git.** Don't invent new state persistence. Use git commits with structured metadata. `git log` becomes your session recovery tool. `/ship` squashes WIP commits to keep bisect clean.

11. **ML + LLM + canary token ensemble for prompt-injection defense.** Single-classifier defense has high false positive rate. Multi-classifier agreement is the cost-effective path to low FP + high true-positive coverage. Publish the defense architecture openly.

12. **Cross-model benchmarking as a per-user utility.** `gstack-model-benchmark` lets users benchmark on *their* workload. Generic benchmarks don't map to individual users' use cases. Shipping a benchmark CLI democratizes model selection.

13. **Skill documentation as deep-dives.** Every skill has a "deep dive" in `docs/skills.md`. New users can explore before committing. This is consumption-friendly for a high-complexity tool.

14. **Opt-in telemetry with public schema.** Telemetry default off. First-run opt-in. Schema in repo. Row-level security. Supabase edge functions enforce schema + length limits. This is the **right** defaults for telemetry in open-source tooling — respect privacy first, provide value to users (local analytics always available), only send data with explicit consent.

15. **Privacy by design for personal automation.** The sidebar agent is marketed for personal automation tasks ("extract other parents' contacts from school portal"). This works because (a) authentication is per-session via cookies imported from real browser, (b) sessions are isolated, (c) prompt-injection defense is layered. Personal automation with privacy is a legitimate, underserved use case.

## Failure modes and open questions

1. **High learning curve.** 23 skills + 8 power tools + custom browser = substantial cognitive load. Users coming from a simpler setup will feel overwhelmed. The README acknowledges this by suggesting `/office-hours → /plan-ceo-review → /review → /qa → stop there`.

2. **Claude Code dependency.** Despite multi-host support, gstack is most polished for Claude Code. Using it with Codex / Cursor / Factory Droid works but feels less first-class.

3. **Process rigidity.** The seven-phase sprint is opinionated. For tasks that don't fit (e.g., exploratory research, one-off scripts, hotfixes), the full process is overkill. `/careful` + direct prompting is the escape hatch but isn't always obvious.

4. **Maintenance cost for the author.** 23 skills across 10 host CLIs is a substantial codebase. Garry is actively maintaining it, but the dependency risk is real — if he stops, the community would need to fork aggressively. MIT license mitigates this but doesn't eliminate it.

5. **Telemetry opt-in UX.** First-run opt-in is good privacy UX but users often reflexively accept or reject without reading. The schema is in the repo, but most users won't check it. The defaults are correct regardless.

6. **Benchmark reproducibility challenges.** The 810× productivity claim is measured across Garry's repos. Reproducing the measurement on different developers (different domains, different prior productivity, different team dynamics) is not straightforward. The claim is credible in context but not universally scalable.

7. **Custom Chromium fork maintenance.** GStack Browser is a Chromium fork with custom branding. Chromium updates are frequent and breaking changes happen. Maintaining the fork is non-trivial.

8. **Security of `/pair-agent` cross-machine coordination.** ngrok tunnels expose local state. Scoped tokens and rate limiting mitigate risk, but the attack surface is real. Production deployment would need additional controls.

9. **Cost at scale.** 10-15 parallel sprints with Claude Code (Sonnet + Opus) can burn through substantial API credits. The productivity gain justifies the cost for YC-scale use cases, but solo developers without YC-scale budget may find it prohibitive.

10. **YC-association coupling.** gstack is strongly associated with YC's brand and philosophy. Users outside the YC ecosystem may find some framings (office hours, CEO review, YC culture references) jarring. The skills work regardless but the positioning is YC-native.

## Relationship to the corpus

gstack is the most practically-realized implementation of several corpus threads:

- **Agent loop and sprint structure** ([01-agent-loop-architecture.md](01-agent-loop-architecture.md)) — the seven-phase sprint is an agent loop extended to software-engineering sprint scale.
- **Subagent delegation** ([02-subagent-delegation.md](02-subagent-delegation.md)) — 23 specialist skills as implicit subagents.
- **Skills system** ([04-skills.md](04-skills.md)) — gstack is the production-scale implementation of Claude Skills.
- **Model Context Protocol** ([07-model-context-protocol.md](07-model-context-protocol.md)) — `/pair-agent` uses shared browser as coordination substrate; the browser agent sidebar uses tool invocation pattern similar to MCP.
- **Clarifying questions** ([13-clarifying-questions.md](13-clarifying-questions.md)) — `/office-hours`, `/plan-*-review` all have forcing-question catalogs.
- **CLAUDE.md canonical memory** ([15-claude-md-canonical-memory.md](15-claude-md-canonical-memory.md)) — gstack adds a gstack section to CLAUDE.md; extends rather than replaces.
- **Progressive disclosure** ([17-progressive-disclosure-context-lifecycle.md](17-progressive-disclosure-context-lifecycle.md)) — `/learn` compounds knowledge across sessions.
- **Prompt injection defense** ([22-guardrails-prompt-injection.md](22-guardrails-prompt-injection.md), [35-malicious-intermediary-attacks.md](35-malicious-intermediary-attacks.md)) — GStack Browser's ML + LLM + canary defense is the most advanced public implementation.
- **Observability and tracing** ([24-observability-tracing.md](24-observability-tracing.md)) — `gstack-analytics` + local JSONL + opt-in telemetry.
- **Verifier-evaluator loops** ([11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md)) — `/review`, `/codex`, `/qa` are verifier layers.
- **TDD** ([12-tdd-loop-test-driven-agents.md](12-tdd-loop-test-driven-agents.md)) — `/ship` auto-generates regression tests for every `/qa` fix.
- **Product control plane** ([41-product-control-plane.md](41-product-control-plane.md)) — gstack is a personal control plane; Multica ([73](73-multica-managed-agents-platform.md)) is the team version.
- **Community Claude Code ecosystem** ([62-everything-claude-code.md](62-everything-claude-code.md)) — gstack is the most-recognized personal stack in the community.
- **Karpathy's new programming** ([65-karpathy-new-programming.md](65-karpathy-new-programming.md)) — gstack is the most thorough individual proof of the "810× productivity" thesis.
- **Meta-harness landscape** ([66-meta-harness-landscape.md](66-meta-harness-landscape.md)) — gstack is a meta-harness at the personal scale.
- **Recommended breakthrough project** ([67-recommended-breakthrough-project.md](67-recommended-breakthrough-project.md)) — gstack is infrastructure the Gnomon proposal would integrate with for primitive-level attribution.
- **Karpathy-skills** ([71-karpathy-skills-single-file-guardrails.md](71-karpathy-skills-single-file-guardrails.md)) — opposite end of the complexity spectrum; gstack explicitly extends karpathy-skills as workflow enforcement.
- **Claude-mem** ([72-claude-mem-persistent-memory-compression.md](72-claude-mem-persistent-memory-compression.md)) — memory-layer counterpart; `/learn` is gstack's lightweight memory, claude-mem is the comprehensive memory.
- **Multica** ([73-multica-managed-agents-platform.md](73-multica-managed-agents-platform.md)) — team-scale managed-agents platform; gstack is personal-scale.

## References — primary artifacts

- **Main repo.** [github.com/garrytan/gstack](https://github.com/garrytan/gstack) — MIT licensed. Full 23-skill stack + browser + CLIs.
- **Author.** Garry Tan — [x.com/garrytan](https://x.com/garrytan), President & CEO of Y Combinator.
- **Karpathy quote source.** ["Andrej Karpathy on AI agents, coding, state of psychosis"](https://fortune.com/2026/03/21/andrej-karpathy-openai-cofounder-ai-agents-coding-state-of-psychosis-openclaw/) — Fortune, March 2026.
- **Skill deep dives.** [docs/skills.md](https://github.com/garrytan/gstack/blob/main/docs/skills.md).
- **Builder ethos.** [ETHOS.md](https://github.com/garrytan/gstack/blob/main/ETHOS.md).
- **Architecture.** [ARCHITECTURE.md](https://github.com/garrytan/gstack/blob/main/ARCHITECTURE.md) — includes prompt-injection defense stack details.
- **Browser reference.** [BROWSER.md](https://github.com/garrytan/gstack/blob/main/BROWSER.md).
- **LOC methodology.** [docs/ON_THE_LOC_CONTROVERSY.md](https://github.com/garrytan/gstack/blob/main/docs/ON_THE_LOC_CONTROVERSY.md).
- **OpenClaw integration.** [docs/OPENCLAW.md](https://github.com/garrytan/gstack/blob/main/docs/OPENCLAW.md).
- **Adding a host.** [docs/ADDING_A_HOST.md](https://github.com/garrytan/gstack/blob/main/docs/ADDING_A_HOST.md).
- **Changelog.** [CHANGELOG.md](https://github.com/garrytan/gstack/blob/main/CHANGELOG.md).
- **Conductor (parallel Claude Code).** [conductor.build](https://conductor.build).
- **OpenClaw (referenced, Peter Steinberger's project).** [github.com/openclaw/openclaw](https://github.com/openclaw/openclaw).

## Cross-references in this corpus

- [01-agent-loop-architecture.md](01-agent-loop-architecture.md) — agent loop underlying each skill.
- [02-subagent-delegation.md](02-subagent-delegation.md) — 23 specialist skills as implicit subagent pool.
- [04-skills.md](04-skills.md) — Claude Skills system gstack fully exploits.
- [07-model-context-protocol.md](07-model-context-protocol.md) — `/pair-agent` cross-agent coordination.
- [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md) — `/review`, `/codex`, `/qa`.
- [12-tdd-loop-test-driven-agents.md](12-tdd-loop-test-driven-agents.md) — regression-test generation.
- [13-clarifying-questions.md](13-clarifying-questions.md) — forcing-question catalogs.
- [15-claude-md-canonical-memory.md](15-claude-md-canonical-memory.md) — CLAUDE.md extended by gstack section.
- [17-progressive-disclosure-context-lifecycle.md](17-progressive-disclosure-context-lifecycle.md) — `/learn` compounds knowledge.
- [22-guardrails-prompt-injection.md](22-guardrails-prompt-injection.md) — GStack Browser prompt-injection defense.
- [24-observability-tracing.md](24-observability-tracing.md) — `gstack-analytics`.
- [35-malicious-intermediary-attacks.md](35-malicious-intermediary-attacks.md) — defended against by browser security stack.
- [41-product-control-plane.md](41-product-control-plane.md) — gstack as personal control plane.
- [46-components-of-coding-agent.md](46-components-of-coding-agent.md) — six components all operationalized in gstack.
- [47-adaptation-of-agentic-ai-survey.md](47-adaptation-of-agentic-ai-survey.md) — four adaptation paradigms implicit in gstack.
- [54-skill-md-authoring-guide.md](54-skill-md-authoring-guide.md) — sibling authoring guide.
- [62-everything-claude-code.md](62-everything-claude-code.md) — broader community ecosystem.
- [65-karpathy-new-programming.md](65-karpathy-new-programming.md) — "new programming" thesis gstack embodies.
- [66-meta-harness-landscape.md](66-meta-harness-landscape.md) — meta-harness at personal scale.
- [67-recommended-breakthrough-project.md](67-recommended-breakthrough-project.md) — Gnomon integration target.
- [70-voltagent-awesome-ai-agent-papers.md](70-voltagent-awesome-ai-agent-papers.md) — research ledger informing gstack design.
- [71-karpathy-skills-single-file-guardrails.md](71-karpathy-skills-single-file-guardrails.md) — minimum-ambition counterpart.
- [72-claude-mem-persistent-memory-compression.md](72-claude-mem-persistent-memory-compression.md) — memory-layer counterpart.
- [73-multica-managed-agents-platform.md](73-multica-managed-agents-platform.md) — team-scale counterpart.
