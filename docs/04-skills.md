# 04 — Skills (SKILL.md)

**Definition.** Skills are model-invocable capability packages: a folder containing a `SKILL.md` (name, description, instructions) plus optional scripts, templates, and tool allowlists. The agent autonomously decides when to invoke a skill based on the description; the skill's body is only loaded into context when it's actually needed — "progressive disclosure" rather than front-loading every capability.

## Problem it solves

Without skills, capability instructions compete for context window space. If your agent knows how to do 30 things, the system prompt is 30k tokens, and the model under-attends to each. Worse, "how to do X" and "when to do X" get conflated — the model wastes tokens on procedural details for tasks it isn't doing. Skills decouple discovery (what's available) from execution (how to do it): only the one-line description is always in context; the full body loads on demand.

Skills also solve a packaging problem. A capability may need instructions, example files, companion scripts, and a specific tool allowlist. Embedding that in the system prompt is messy; a skill folder is a clean artifact.

## Mechanism

A skill is a directory like:

```
.claude/skills/run-migrations/
├── SKILL.md          # name, description, instructions
├── schema.sql        # template referenced in the body
└── scripts/
    └── validate.sh   # helper the skill can invoke
```

`SKILL.md` has YAML frontmatter that drives invocation:

```markdown
---
name: run-migrations
description: |
  Use when the user asks to apply or roll back database migrations.
  Loads migration files, validates order, runs them with backup.
allowed-tools: [Bash, Read, Edit]
disable-model-invocation: false
---
# Steps
1. Read migration files from `db/migrations/` in filename order.
2. For each, run `./scripts/validate.sh <file>` first.
3. Apply with `psql $DATABASE_URL -f <file>`.
4. Record applied migrations in `db/applied.log`.
```

Invocation flow:

1. **Discovery.** At session start, the harness scans skill directories and exposes each skill's *name + description* to the model. Instructions are **not** loaded yet.
2. **Decision.** The model's normal reasoning identifies that a user request matches a skill description and calls `Skill(name="run-migrations")`.
3. **Injection.** The harness loads the `SKILL.md` body (plus any referenced companion files, if the skill opts in) into the current turn's context. The tool allowlist is temporarily narrowed to `allowed-tools`.
4. **Execution.** The model proceeds with the now-visible instructions. When the skill completes, the additional context is kept for the session but was paid for only once.

Skills can be bundled into plugins, shared across machines, or invoked by user-typed `/skill-name` commands that bypass the model's autonomous-invocation decision.

## Concrete pattern

A research skill invoked in the current folder:

```markdown
---
name: deep-research
description: |
  Research a topic across recent papers, blogs, and docs.
  Produce a markdown brief with at least 2 authoritative sources.
allowed-tools: [WebSearch, WebFetch, Write]
---
# Instructions
1. Clarify target audience and depth with AskUserQuestion.
2. Run 4–8 WebSearch queries covering: academic papers, blog posts,
   benchmarks, practitioner critiques.
3. WebFetch the 3 most authoritative pages.
4. Write a brief (1000–1500 words) with:
   - one-line definition
   - problem
   - mechanism
   - concrete example
   - failure modes
   - references (at least 2)
```

The description is the single most important line in the file. It's the only thing the model sees until invocation; it must be specific enough to be selected for the right requests and no others.

## Variants & related techniques

- **Slash commands.** User-typed `/name` that directly invokes a skill — deterministic, bypasses model routing. Useful when the user knows what they want.
- **Plugins.** Collections of skills + MCP servers + hooks, distributable as a unit. Claude Code's plugin format is an emerging standard.
- **Tool vs. skill.** A tool is a single operation (`Read`, `Bash`). A skill is a *procedure* that composes tools. Don't force a procedure into a tool schema; don't wrap a single operation in a skill.
- **Subagents ([02-subagent-delegation.md](02-subagent-delegation.md))** are related: both are invokable capabilities. Subagents give the operation its own context window; skills run in the parent's context. Prefer a subagent when the operation would drop a big observation into the parent.

## Failure modes & anti-patterns

- **Description too vague.** `"General research skill"` gets invoked for every question. Fix: describe the *trigger* plainly — "Use when the user asks to ..."
- **Description too specific.** `"Research quarterly earnings for Q3 2024 AWS growth"` never gets invoked again. Fix: generalize.
- **Instruction overlap.** Two skills both claim to handle the same requests; the model picks arbitrarily. Fix: make descriptions mutually exclusive or promote the shared logic up.
- **Skill hell.** 50 skills in a repo, none curated, the model routes badly. Fix: keep the installed skills set small and audit descriptions for specificity.
- **Skills that re-implement the base.** If a skill is just "run the agent loop but with this system prompt", it probably belongs as a system prompt tweak or a subagent, not a skill.
- **Leaking companion files.** Loading all referenced scripts/templates into context eagerly defeats progressive disclosure. Skills should reference files by path and let the model `Read` them if needed.

## When to use (and when not to)

**Use** skills when:

- You have a well-scoped procedure the model should autonomously trigger in the right context.
- The procedure has dedicated companion files (templates, scripts, schemas).
- The instructions are long enough to be worth gating behind description-based routing.
- You want to share a capability across projects or teammates.

**Don't** use skills when:

- The capability is a single tool call — expose it as a tool.
- The capability is always needed — put it in the system prompt.
- The "skill" is the entire job description of a subagent — use a subagent.
- You don't have a crisp trigger description — the model won't route to it reliably.

## References

- Claude Code docs, Extend Claude with skills — <https://docs.claude.com/en/docs/claude-code/skills>
- alexop.dev, "Claude Code Customization Guide: CLAUDE.md, Slash Commands, Skills, and Subagents" — <https://alexop.dev/posts/claude-code-customization-guide-claudemd-skills-subagents/>
- youngleaders.tech, "Claude Skills vs Commands vs Subagents vs Plugins" — <https://www.youngleaders.tech/p/claude-skills-commands-subagents-plugins>
- penligent.ai, "Inside Claude Code: Architecture Behind Tools, Memory, Hooks, and MCP" — <https://www.penligent.ai/hackinglabs/inside-claude-code-the-architecture-behind-tools-memory-hooks-and-mcp/>
