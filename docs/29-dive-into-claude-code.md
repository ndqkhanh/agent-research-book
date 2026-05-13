# 29 — Dive Into Claude Code: The Architecture Under the Hood

**Definition.** The "Dive into Claude Code" paper (arXiv:2604.14228, UCL and independent academic authors, April 2026) reverse-engineers publicly available Claude Code TypeScript source to document how a modern coding harness is actually built. It names thirteen design principles grounded in five values (human decision authority, safety, reliability, capability amplification, contextual adaptability) and compares Claude Code with OpenClaw to show which design questions have one answer and which have many.

## Problem it solves

Harness engineering has been practiced in industry for ~2 years but was under-documented as an academic field. Engineers cargo-culted from Twitter threads and blog posts. A properly sourced, code-grounded audit lets the field move past folklore: when a paper says "Claude Code has a 5-layer compaction pipeline", it's a citable fact rather than a rumor. That shifts the conversation from "what should harnesses look like?" to "given a value, which design principle follows, and which implementations match?"

## Mechanism

The paper's main contributions as reported:

1. **Core loop.** A deceptively simple `while` loop calls the model, runs tools, and repeats. Most complexity is in the surrounding systems, not the loop itself — a finding that mirrors [01-agent-loop-architecture.md](01-agent-loop-architecture.md).
2. **Permission system.** Seven modes plus an ML-based command-risk classifier. More granular than the four public modes most users interact with.
3. **Five-layer compaction pipeline.** Context management is not a single summarization pass but a layered system that kicks in at different thresholds and for different data classes (plan files, tool outputs, old transcript).
4. **Four-part extensibility.** MCP + plugins + skills + hooks. Each has a distinct lifecycle and purpose; the paper maps design principles onto which extension mechanism to use when.
5. **Subagent + worktree isolation.** Subagent delegation comes with filesystem-level (worktree) isolation so parallel subagents cannot corrupt each other's state — see [02-subagent-delegation.md](02-subagent-delegation.md).
6. **Five values → thirteen principles → implementation choices.** The paper makes these traceable, so when Claude Code and OpenClaw differ (e.g., action-level safety vs perimeter-level access control), the difference reflects a principle trade-off, not a whim.

## Concrete pattern

The paper-style audit is itself a pattern any team can use on its own harness:

```
1. Articulate the values (what non-negotiables does the system protect?)
2. Derive principles from values (e.g., "model must not have root by default"
   derives from "safety > convenience").
3. For each principle, name the implementation features that uphold it
   (permissions, sandbox, hooks, observability).
4. For each implementation feature, map to the code: file, function, config.
5. Compare to one peer system; disagreements are your research agenda.
```

The artifact is a matrix of value × principle × feature × file, which doubles as documentation for new engineers and as a security case for reviewers.

## Variants & related techniques

- **Harness pieces documented elsewhere in this folder:** agent loop ([01](01-agent-loop-architecture.md)), subagents ([02](02-subagent-delegation.md)), skills ([04](04-skills.md)), hooks ([05](05-hooks.md)), permission modes ([06](06-permission-modes.md)), MCP ([07](07-model-context-protocol.md)), compaction ([08](08-context-compaction.md)), memory files ([09](09-memory-files.md)).
- **BDTechTalks "art of harness engineering"** ([40-harness-engineering-principles.md](40-harness-engineering-principles.md)) — press-side summary of the same leak/audit.
- **12 harness patterns** ([43-twelve-harness-patterns.md](43-twelve-harness-patterns.md)) — industry-facing taxonomy of the same primitives.
- **Components of a Coding Agent** ([46-components-of-coding-agent.md](46-components-of-coding-agent.md)) — Raschka's six-component view.

## Failure modes & anti-patterns

- **Cargo-culting features.** "Claude Code has seven permission modes, so we need seven." Fix: map each feature back to a value your system actually has, or don't build it.
- **Reverse-engineering as moat-erosion.** Publishing internals can cut the moat. Fix: the moat is usually not the architecture but the evaluation pipeline and data — keep those tight.
- **Overweight the ML classifier.** An ML-based command-risk classifier is nice but not a replacement for policy. Fix: layer classifier + deterministic rules + sandboxes.
- **Ignoring the OpenClaw comparison.** Differences between Claude Code and OpenClaw on the same question reveal open design space. Fix: study peer harnesses when choosing between equally-valid approaches.
- **Pretending your harness is a model.** Harnesses ship with the same velocity as web infrastructure, not as models. Plan for weekly updates, not yearly releases.

## When to use (and when not to)

This paper is reference material for:

- Teams building their own coding or agentic harness who want a well-documented starting point.
- Academic researchers finally getting a citable architectural reference.
- Security reviewers auditing an agent deployment.
- Product engineers deciding when to lean on MCP vs skills vs hooks.

Do **not** treat it as a prescriptive checklist — the values it derives from are Claude Code's, not yours. Start from your values.

## References

- arXiv:2604.14228 — "Dive into Claude Code" (accessed April 2026). <https://arxiv.org/abs/2604.14228>
- Anthropic Engineering, "Building agents with the Claude Agent SDK". <https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk>
- BDTechTalks, "The art of AI harness engineering". <https://bdtechtalks.com/2026/04/06/ai-harness-engineering-claude-code-leak/>
- Generative Programmer, "12 Agentic Harness Patterns". <https://generativeprogrammer.com/p/12-agentic-harness-patterns-from>
- Sebastian Raschka, "Components of a Coding Agent". <https://magazine.sebastianraschka.com/p/components-of-a-coding-agent>
