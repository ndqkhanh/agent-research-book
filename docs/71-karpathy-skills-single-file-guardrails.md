# 71 — forrestchang/andrej-karpathy-skills: Single-File CLAUDE.md Guardrails Against LLM Coding Pathologies

**Definition.** `forrestchang/andrej-karpathy-skills` is a **single-file `CLAUDE.md` guideline** — ~60 lines of prose — that codifies four behavioral principles to counter the specific pathologies Andrej Karpathy identified in LLM coding behavior in an influential November 2025 X post. The principles are: **Think Before Coding**, **Simplicity First**, **Surgical Changes**, **Goal-Driven Execution**. The repo ([github.com/forrestchang/andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills)) provides the guidelines as (a) a per-project `CLAUDE.md` snippet, (b) a Claude Code plugin installable via `/plugin install andrej-karpathy-skills@karpathy-skills`, and (c) a Cursor project rule `.cursor/rules/karpathy-guidelines.mdc` — multi-surface delivery of the same behavioral contract. The project has **71,398+ stars** (April 2026), making it the most-starred CLAUDE.md artifact on GitHub and one of the top-starred single-document repositories of the year.

The author, Jiayuan "Forrest" Chang ([@jiayuan_jy](https://x.com/jiayuan_jy)), also created Multica ([doc 73](73-multica-coding-agent-platform.md)) — a multi-agent coding-platform for running and managing reusable skills. The Karpathy-skills repo is the *single-file progenitor* of that larger project: one file, four principles, massive adoption.

## Problem it solves — the four LLM coding pathologies Karpathy named

The entire project is built around four specific, recurring LLM failure modes Karpathy articulated. From the original post:

> "The models make wrong assumptions on your behalf and just run along with them without checking. They don't manage their confusion, don't seek clarifications, don't surface inconsistencies, don't present tradeoffs, don't push back when they should."

> "They really like to overcomplicate code and APIs, bloat abstractions, don't clean up dead code... implement a bloated construction over 1000 lines when 100 would do."

> "They still sometimes change/remove comments and code they don't sufficiently understand as side effects, even if orthogonal to the task."

> "LLMs are exceptionally good at looping until they meet specific goals... Don't tell it what to do, give it success criteria and watch it go."

These four quotes map 1:1 onto four failure modes and four countervailing principles:

| Karpathy's observation | Failure mode name | Principle that counters it |
|---|---|---|
| "Make wrong assumptions... run along without checking" | Silent assumption adoption | **Think Before Coding** |
| "Overcomplicate code and APIs, bloat abstractions" | Speculative overengineering | **Simplicity First** |
| "Change/remove comments and code they don't sufficiently understand" | Orthogonal drive-by edits | **Surgical Changes** |
| "Exceptionally good at looping until they meet specific goals" | (positive: exploit for leverage) | **Goal-Driven Execution** |

This is an unusually clean taxonomy — four pathologies, four principles, no overlap. The fit is so tight partly because Karpathy articulated the pathologies with a writer's precision, and partly because Forrest filtered them to exactly those four rather than dozens.

## The four principles — mechanism and enforcement

### Principle 1 — Think Before Coding

**Core rule:** Don't assume. Don't hide confusion. Surface tradeoffs.

**Specific directives:**
- State assumptions explicitly; if uncertain, ask.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

**Mechanism:** This directly targets the "silent assumption adoption" failure mode. The principle instructs the model that when it encounters ambiguity, the *correct* action is not to pick one interpretation and proceed — it is to surface the ambiguity to the user. This inverts the default LLM behavior (which is trained to be helpful, i.e., to produce an answer) in favor of *calibrated uncertainty*.

**Relationship to the corpus:** This is the same principle as [13-clarifying-questions.md](13-clarifying-questions.md), expressed as a harness-level rule instead of a loop-level behavior. Where docs 13 describes the mechanism (`ask_user` tool calls, clarification loops), this principle enforces it as a CLAUDE.md directive.

### Principle 2 — Simplicity First

**Core rule:** Minimum code that solves the problem. Nothing speculative.

**Specific directives:**
- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If 200 lines could be 50, rewrite it.

**Mechanism:** This targets speculative overengineering. The model is trained to produce "good-looking" code — which often means code with abstractions, configuration, error handling, and extensibility hooks. Those patterns are useful in production, but *inappropriate when the task does not require them*. The "senior engineer test" — "Would a senior engineer say this is overcomplicated?" — gives the model a heuristic to self-audit.

**Relationship to the corpus:** This principle sits next to [04-skills.md](04-skills.md) (Claude Skills are about *defining* simplicity boundaries per-task) and [15-claude-md-canonical-memory.md](15-claude-md-canonical-memory.md) (CLAUDE.md as the delivery mechanism). It also connects to [62-everything-claude-code.md](62-everything-claude-code.md) where "reduce overengineering" is one of the recurring themes across community skills.

### Principle 3 — Surgical Changes

**Core rule:** Touch only what you must. Clean up only your own mess.

**Specific directives:**
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, *mention* it — don't delete it.
- When your changes create orphans, remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.
- **The test:** Every changed line should trace directly to the user's request.

**Mechanism:** This targets orthogonal drive-by edits — the behavior where the model, while implementing a requested change, also "improves" adjacent code (removes comments it doesn't understand, changes formatting, renames variables, restructures unrelated code). The failure mode is not that any individual drive-by edit is wrong; it is that the user cannot review dozens of unrelated changes in a PR, so bad edits slip through review, and good edits still expand the review burden.

The **"every changed line traces to the user's request"** test is unusually operational — it gives reviewers and the model itself a clear acceptance criterion.

**Relationship to the corpus:** This principle directly addresses failure modes identified in [16-guardrails-and-failure-modes.md](16-guardrails-and-failure-modes.md) and the research cited in [70-voltagent-awesome-ai-agent-papers.md](70-voltagent-awesome-ai-agent-papers.md)'s Evaluation bucket — papers analyzing the gap between intended and actual changes in agent-authored PRs (*Analyzing Message-Code Inconsistency in AI Coding Agent-Authored Pull Requests*, *Why AI Agent Involved PRs Remain Unmerged*, *Let's Make Every PR Meaningful*). The Karpathy-skills principle is the *behavioral contract* that would reduce the underlying problem those papers measure.

### Principle 4 — Goal-Driven Execution

**Core rule:** Define success criteria. Loop until verified.

**Specific directives — transformation templates:**

| Imperative instruction | Goal-driven transformation |
|---|---|
| "Add validation" | "Write tests for invalid inputs, then make them pass" |
| "Fix the bug" | "Write a test that reproduces it, then make it pass" |
| "Refactor X" | "Ensure tests pass before and after" |

**For multi-step tasks, state a brief plan:**

```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

**Mechanism:** This principle is *positive* — it exploits the model's well-established ability to loop against a verifiable criterion. The key insight from Karpathy's post is that LLMs are *very good* at this iteration when the criterion is clear. The failure mode being addressed is not the model's ability to loop but the user's tendency to give imperative instructions ("make it work", "fix the bug") instead of verifiable goals ("write a failing test, then make it pass").

The `[Step] → verify: [check]` plan template is a lightweight **TDD-at-the-agent-level** pattern that aligns with [12-tdd-loop-test-driven-agents.md](12-tdd-loop-test-driven-agents.md) and [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md).

**Relationship to the corpus:** This is the same operational pattern as the verifier-evaluator loop ([11](11-verifier-evaluator-loops.md)) and the TDD-at-the-agent-level thread ([12](12-tdd-loop-test-driven-agents.md)), expressed as a CLAUDE.md directive rather than a harness capability. The difference: the harness *supports* looping, but the CLAUDE.md *instructs the agent to set up the loop properly* by transforming imperative tasks into goal-criteria form.

## Multi-surface delivery — the CLAUDE.md / plugin / Cursor rule triple

One of the most-copied patterns in the Karpathy-skills repo is its **multi-surface delivery model**. The same four principles are delivered through three different channels:

### Surface 1 — `CLAUDE.md` (per-project or global)

```bash
# New project
curl -o CLAUDE.md https://raw.githubusercontent.com/forrestchang/andrej-karpathy-skills/main/CLAUDE.md

# Append to existing CLAUDE.md
echo "" >> CLAUDE.md
curl https://raw.githubusercontent.com/forrestchang/andrej-karpathy-skills/main/CLAUDE.md >> CLAUDE.md
```

This is the canonical CLAUDE.md ([15-claude-md-canonical-memory.md](15-claude-md-canonical-memory.md)) delivery model — a flat file at the project root that Claude Code reads into context. The tradeoff: each project needs the file, and updates require re-copying.

### Surface 2 — Claude Code Plugin

From within Claude Code:
```
/plugin marketplace add forrestchang/andrej-karpathy-skills
/plugin install andrej-karpathy-skills@karpathy-skills
```

This uses the Claude Code plugin system to install the guideline *across all projects* without per-project CLAUDE.md copying. The plugin system (as of Claude Code 2.x) supports:
- Marketplace discovery
- Versioned install
- Per-user global scope
- Plugin metadata (commands, hooks, slash commands)

This is the cleanest delivery surface for "a single guideline applied everywhere" — it is the CLAUDE.md pattern scaled from per-project to per-user.

### Surface 3 — Cursor Project Rule

`.cursor/rules/karpathy-guidelines.mdc` — committed to the repo, automatically loaded by Cursor when the project is opened. Cursor's rule system supports:
- Project rules (committed, scoped to the project)
- User rules (per-user, across projects)
- `.mdc` format with YAML frontmatter for metadata

**Why deliver to multiple surfaces:** the same behavioral contract works regardless of which coding-agent harness the developer uses. Forrest's choice to ship all three is a model for anyone wanting community adoption — developers using different tools can adopt the guidelines without switching tools.

**Lesson for harness engineers:** if your behavioral guideline is valuable, ship it to *every* surface the target audience uses. The marginal cost is low (the same markdown, different file headers / install flows) and the adoption benefit is multiplicative.

## What the 71,398 stars tell us

71,398 stars (as of April 2026, growing by thousands per month) is an unusual signal. For context:
- The median top-10,000 GitHub repo has <1,000 stars.
- Single-file documentation repositories rarely exceed 10,000 stars.
- This repo has **no code**, just a ~60-line markdown file and the associated install scripts.

This is empirical evidence that the problem is acute and widely felt. Several factors amplify the signal:

1. **Karpathy's authority.** Andrej Karpathy is one of the most-followed AI voices. His post articulating the pathologies provided social proof and a recognizable name.
2. **Zero-friction adoption.** `curl -o CLAUDE.md ...` is as low-friction as a software install gets. The plugin install is one command. The Cursor rule is automatic. Users can try the guidelines in 30 seconds.
3. **Immediately verifiable improvement.** Users report the four "it's working" signals — fewer unnecessary diff lines, fewer rewrites, clarifying questions before implementation, clean minimal PRs. These are all observable in the next PR after install.
4. **Single-file simplicity.** No framework to learn, no architecture to understand, no migration. Just add 60 lines to CLAUDE.md.
5. **Taxonomic clarity.** Four principles, clearly named, no overlap, memorable. Easy to cite, easy to reason about.
6. **Viral propagation through Karpathy himself.** The original X post reached 2M+ views, and the repo link was the most-shared response.

The lesson: **articulating a problem clearly and shipping a 60-line solution can outperform articulating it clearly and shipping a 10,000-line framework**. The Forrestchang repo is a clinic in how to productize a behavioral observation.

## Relationship to the Claude Code / CLAUDE.md ecosystem

### Contrast with full Claude Skills

- **Claude Skills (`/skills/` directories with SKILL.md per skill)**: complex, per-task, conditional (triggered by description matching), support scripts and subagents. Suited to complex workflows.
- **Karpathy-skills (`CLAUDE.md` with four principles)**: simple, cross-task, unconditional (always applies), no scripts. Suited to broad behavioral contracts.

The comparison is important: not every valuable guideline needs to be a Skill. Some behavior — "don't make silent assumptions", "don't drive-by edit", "transform imperative tasks into verifiable goals" — applies to *every* task and is better delivered as a flat CLAUDE.md principle than as a Skill that has to be triggered.

**Rule of thumb from this project:**
- If the rule applies **across all tasks**, put it in CLAUDE.md.
- If the rule applies **to a specific class of tasks** (testing, security, PR review, etc.), put it in a Skill.
- If the rule applies **to a specific type of file or action** (e.g., *.tsx files, git commits), put it in a Cursor rule with a specific scope.

### Contrast with `everything-claude-code` ([doc 62](62-everything-claude-code.md))

Where everything-claude-code is a *broad curation* of community Claude Code skills (dozens of skills, agents, commands, settings patterns), Karpathy-skills is a *focused artifact* — one guideline, four principles, deeply refined. Both are valuable, but they serve different needs:
- Everything-claude-code: "show me what the community is building for Claude Code"
- Karpathy-skills: "give me one proven behavioral upgrade with no setup cost"

Users typically adopt Karpathy-skills *first* (zero friction, immediate improvement) and then explore broader curations once they understand the Claude Code config surface.

### Contrast with `gstack` ([doc 75](75-gstack-garry-tan-claude-code-setup.md))

Where gstack is Garry Tan's full personal Claude Code setup (MCP servers, personal skills, context priming, detailed CLAUDE.md structure), Karpathy-skills is the minimum-viable CLAUDE.md. A typical adoption path:
1. Start with Karpathy-skills (60 lines, universal).
2. Add project-specific sections as needed.
3. Graduate to gstack-style full configuration for heavy Claude Code use.

## Failure modes — where the principles break down

The README explicitly notes the tradeoff:

> "These guidelines bias toward caution over speed. For trivial tasks (simple typo fixes, obvious one-liners), use judgment — not every change needs the full rigor."

This is an important concession. The four principles *do* slow the agent down when applied rigorously:

1. **Think Before Coding adds clarification overhead.** For a task with genuinely clear intent, the "state assumptions explicitly" step is noise. The cost is minor but real.
2. **Simplicity First can undershoot.** The principle biases toward minimum viable code. For tasks where the user genuinely wants extensibility or configurability but did not explicitly request it, the agent may produce code that needs to be extended later.
3. **Surgical Changes can leave known bad code in place.** If the agent notices dead code, duplicated logic, or obvious bugs adjacent to the requested change, the principle says *mention* but *don't fix*. The user has to explicitly approve the cleanup. For engineering teams with high review throughput this is fine; for solo developers who want the agent to be more autonomous, it can feel overly cautious.
4. **Goal-Driven Execution requires writable tests / verifiable criteria.** If the task is genuinely ambiguous ("improve UX"), transforming it to goal-criteria form requires the user to articulate success. The principle pushes this work back to the user, which is correct but adds friction.

**When to override the principles:**
- Trivial changes (typo fixes, obvious one-liners).
- Exploratory/brainstorming work where overcomplication is the point.
- Prototypes meant to be thrown away.
- Tasks where the user has explicitly requested extensibility / configurability.

## Patterns harness engineers should steal

1. **Ground your behavioral guidelines in a named authority's observations.** Karpathy's name and post reframe the guidelines from "opinions" to "a response to Karpathy's observations." This is citation leverage — you are not asserting authority, you are deploying someone else's. When you write guidelines, anchor them to a specific named observation / paper / post.

2. **Four principles or fewer.** Five or more principles exceed working memory; developers will not remember them. Four is the upper bound on what most humans hold simultaneously. If you have more than four, consolidate or drop some.

3. **Each principle has a "the test is..." clause.** "Every changed line should trace directly to the user's request" and "Would a senior engineer say this is overcomplicated?" are *operational tests*, not aspirational phrasings. Principles without tests are ignored; principles with tests are audit-able.

4. **Transformation templates for common patterns.** The "Instead of... Transform to..." table for goal-driven execution is the most cite-able part of the doc. Developers can literally paste the transformation into prompts. When you write guidelines, include *templates for the exact language change you want*.

5. **Multi-surface delivery.** Ship the same rule to `CLAUDE.md`, plugin, and Cursor rules. Marginal cost is low; adoption multiplier is high.

6. **Explicit tradeoff statement.** "Caution over speed — use judgment for trivial tasks" prevents the guideline from being applied rigidly. Acknowledging the tradeoff preserves the guideline's credibility for the cases where it matters.

7. **"How to know it's working" section.** Four observable signals (fewer unnecessary diff lines, fewer rewrites, clarifying questions first, clean minimal PRs). This lets users *self-audit* whether adoption succeeded. Guidelines without feedback signals never stick.

8. **Customization hook.** The README tells users where to add project-specific guidelines ("Add sections like: ## Project-Specific Guidelines"). This prevents the tension between "keep the principles" and "add your stuff" from becoming adoption-blocking.

## The sister project — pointer to Multica

The Karpathy-skills README opens with a note pointing to Multica ([github.com/multica-ai/multica](https://github.com/multica-ai/multica)):

> "Check out my new project Multica — an open-source platform for running and managing coding agents with reusable skills."

This is covered in [73-multica-coding-agent-platform.md](73-multica-coding-agent-platform.md). The evolution from Karpathy-skills to Multica is worth tracking: **from a single-file CLAUDE.md to a full platform for running agents with reusable skills**. The trajectory — "start with the smallest-possible artifact, scale to a platform as the scope clarifies" — is itself a pattern worth copying.

## References — primary artifacts

- **Main repo.** [github.com/forrestchang/andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills) — 71,398 stars (April 2026), MIT licensed.
- **Canonical CLAUDE.md.** [raw.githubusercontent.com/forrestchang/andrej-karpathy-skills/main/CLAUDE.md](https://raw.githubusercontent.com/forrestchang/andrej-karpathy-skills/main/CLAUDE.md) — the 60-line file.
- **Cursor project rule.** [.cursor/rules/karpathy-guidelines.mdc](https://github.com/forrestchang/andrej-karpathy-skills/blob/main/.cursor/rules/karpathy-guidelines.mdc) — Cursor-specific port.
- **Karpathy's original X post.** [x.com/karpathy/status/2015883857489522876](https://x.com/karpathy/status/2015883857489522876).
- **Author.** Jiayuan "Forrest" Chang — [x.com/jiayuan_jy](https://x.com/jiayuan_jy).
- **Sister project.** [github.com/multica-ai/multica](https://github.com/multica-ai/multica) — deep-dive in [doc 73](73-multica-coding-agent-platform.md).

## Cross-references in this corpus

- [04-skills.md](04-skills.md) — Claude Skills system (task-scoped siblings).
- [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md) — the looping mechanism principle 4 exploits.
- [12-tdd-loop-test-driven-agents.md](12-tdd-loop-test-driven-agents.md) — TDD pattern underneath principle 4.
- [13-clarifying-questions.md](13-clarifying-questions.md) — the loop-level mechanism for principle 1.
- [15-claude-md-canonical-memory.md](15-claude-md-canonical-memory.md) — the delivery mechanism (CLAUDE.md).
- [16-guardrails-and-failure-modes.md](16-guardrails-and-failure-modes.md) — the failure-mode taxonomy this project addresses.
- [54-skill-md-authoring-guide.md](54-skill-md-authoring-guide.md) — the sibling authoring guide for Skills.
- [62-everything-claude-code.md](62-everything-claude-code.md) — broader Claude Code community curation.
- [70-voltagent-awesome-ai-agent-papers.md](70-voltagent-awesome-ai-agent-papers.md) — papers measuring the underlying failure modes.
- [73-multica-coding-agent-platform.md](73-multica-coding-agent-platform.md) — the sister project platform.
- [75-gstack-garry-tan-claude-code-setup.md](75-gstack-garry-tan-claude-code-setup.md) — a contrasting heavy-weight personal Claude Code setup.
