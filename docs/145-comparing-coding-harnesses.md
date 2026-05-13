# 145 — Comparing Modern AI Coding Harnesses (2026): Claude Code, Cursor, Cline, Devin, Aider, Continue, Windsurf, Sourcegraph Cody, Replit Agent

**Sources.** book2-comparing-en (Claude Code vs Codex side-by-side); the public documentation and community analyses of each system; [62-everything-claude-code](62-everything-claude-code.md), [29-dive-into-claude-code](29-dive-into-claude-code.md), [66-meta-harness-landscape](66-meta-harness-landscape.md); plus the LangChain *State of AI Agents* surveys.

**One-line definition.** This chapter compares the leading AI coding harnesses of 2026 — Claude Code, Cursor, Cline, Devin, Aider, Continue, Windsurf, Sourcegraph Cody, Replit Agent, OpenAI Codex CLI — across nine axes (control plane, query loop, tools/sandboxes, skills/hooks, persistence, multi-agent, target user, deployment, lock-in) so engineers can pick the right harness for their workflow rather than the most-marketed one.

## Why this matters

By 2026, the AI coding harness market has consolidated around ~10 serious products. Each has distinct opinions about how an agent should be structured, what tools it should have, what control the user has, how persistence works, and which model(s) it serves. The differences matter operationally: a team picking Cursor for individual productivity gets different value than a team picking Devin for autonomous tasks.

For agent builders, this comparison serves two purposes: (1) help engineers pick the right harness for their team; (2) provide a comparative reference for builders designing their own harness — what conventions are emerging, what's still differentiated.

This chapter is the cross-product comparison written from an engineering perspective rather than a marketing one.

## Problem it solves

Five concrete coding-harness decision failures:

1. **Wrong tool for the job.** Team picks Cursor (IDE-integrated, individual) for autonomous task workflows that need Devin.
2. **Mis-budgeted autonomy.** Team picks Devin for tasks that need pair-programming, where Cursor wins.
3. **Lock-in by accident.** Team picks Claude Code; later wants OpenAI; switching cost is high.
4. **Missing security review.** Team picks a harness without sandboxing for code that affects production systems.
5. **Frankenstein.** Team uses three harnesses simultaneously; configuration chaos.

## Core idea in one paragraph

AI coding harnesses split along five major axes. **Target user**: solo IDE-integrated (Cursor, Continue, Cline) vs autonomous task (Devin, Replit Agent) vs hybrid CLI-orchestrator (Claude Code, OpenAI Codex CLI, Aider) vs enterprise repo-scale (Sourcegraph Cody). **Control plane**: runtime-first dynamic prompt assembly (Claude Code) vs policy-language explicit (OpenAI Codex CLI) vs IDE-event-driven (Cursor) vs project-config (Aider). **Query loop**: synchronous pair-programming loop (Cursor) vs continuous task loop (Devin) vs CLI command-driven (Claude Code) vs git-aware patch loop (Aider). **Tools/sandboxes**: built-in fixed (Cursor) vs MCP-extensible (Claude Code, Cline) vs custom extensions (Sourcegraph). **Persistence**: project-local (CLAUDE.md, .cursor/rules) vs no persistence (Aider) vs server-side memory (Devin). The right harness depends on the workflow shape; no single harness wins all categories. Most engineers in 2026 use 1–2 harnesses primary, occasionally a third for specific tasks.

## Mechanism (step by step)

### 1. The product taxonomy

| Product | Type | Vendor | Notes |
|---|---|---|---|
| **Claude Code** | CLI orchestrator | Anthropic | Runtime-first; CLAUDE.md persistence; MCP-native |
| **Cursor** | IDE (VS Code fork) | Anysphere | IDE-integrated; pair-programming; .cursor/rules |
| **Cline** | VS Code extension | open source | MCP-native; runs in Cursor or VS Code |
| **Devin** | Autonomous agent | Cognition | Continuous loop; server-side; task-oriented |
| **Aider** | CLI | open source | Git-aware; patch-based; minimal persistence |
| **Continue** | IDE extension | open source | Customisable; multi-LLM |
| **Windsurf** | IDE (Codeium) | Codeium | Cascade-style; full-context awareness |
| **Sourcegraph Cody** | Enterprise IDE+repo | Sourcegraph | Repo-scale context; enterprise features |
| **Replit Agent** | Browser IDE | Replit | Cloud-native; full-stack scaffolding |
| **OpenAI Codex CLI** | CLI | OpenAI | Policy-language; explicit safety controls |

### 2. Axis 1: Target user

- **Solo developer in IDE**: Cursor, Continue, Cline, Windsurf.
- **Solo developer at terminal**: Claude Code, Aider, OpenAI Codex CLI.
- **Async tasks with handoff**: Devin, Replit Agent.
- **Enterprise repo-scale**: Sourcegraph Cody.

### 3. Axis 2: Control plane (book2's framing)

- **Runtime-first** (Claude Code): prompt assembled dynamically at runtime; minimal upfront declaration.
- **Policy-language** (OpenAI Codex CLI): explicit declarative policies for tools, approvals, sandbox.
- **IDE-event-driven** (Cursor, Windsurf): triggered by editor events.
- **Project-config** (Aider, Continue): files in repo configure behaviour.

This is the deepest architectural division.

### 4. Axis 3: Query loop

- **Synchronous pair-programming** (Cursor, Continue): user types, agent responds, immediate feedback.
- **Continuous task loop** (Devin, Replit Agent): give the agent a task; it works for minutes/hours.
- **CLI command-driven** (Claude Code, Aider): explicit commands; user controls the loop.
- **Repo-aware** (Sourcegraph Cody): agent reasons about whole repo, not just current file.

### 5. Axis 4: Tools & sandboxes

- **Built-in fixed**: Cursor (file edit, terminal, git) — opinionated.
- **MCP-extensible**: Claude Code, Cline — user can add MCP servers.
- **Custom extensions**: Sourcegraph (Cody Tools) — domain-specific.
- **Cloud sandboxes**: Replit, Devin — full environment provided.

MCP is becoming the standard tool protocol; products without it are increasingly the exception.

### 6. Axis 5: Persistence (continuity)

- **Project-local files** (Claude Code's CLAUDE.md, Cursor's .cursor/rules): committed to repo; team-shared.
- **No persistence** (Aider): each session fresh.
- **Server-side memory** (Devin): persists across sessions on the server.
- **Hybrid** (Sourcegraph Cody): repo-aware context + per-user prefs.

### 7. Axis 6: Multi-agent / sub-agent

- **Native sub-agent** (Claude Code): explicit Task tool with sub-agents and isolation.
- **No native multi-agent** (Cursor, Aider): single-agent; user orchestrates.
- **Implicit multi-agent** (Devin, Replit Agent): server-side may use multi-agent under the hood.
- **Multi-cursor** (Sourcegraph Cody): multiple parallel sessions.

### 8. Axis 7: Deployment

- **Local-first**: Claude Code, Aider, Cursor (data stays local).
- **Cloud-hybrid**: Continue, Cline (LLM in cloud, code local).
- **Cloud-only**: Devin, Replit Agent.
- **Enterprise on-prem**: Sourcegraph Cody (offers self-hosted).

### 9. Axis 8: Vendor / model lock-in

| Product | Lock-in |
|---|---|
| Claude Code | Anthropic-tied; some flexibility |
| Cursor | Multi-model; Cursor's own models too |
| Cline | Multi-model |
| Devin | Cognition's own; less flexibility |
| Aider | Multi-model; minimal lock-in |
| Continue | Multi-model; minimal lock-in |
| Windsurf | Codeium; some flexibility |
| Sourcegraph Cody | Multi-model |
| Replit Agent | Replit's; some flexibility |
| OpenAI Codex CLI | OpenAI |

### 10. Axis 9: Cost

- **Subscription**: Cursor ($20/mo), Cody ($9–$19/mo enterprise more), Devin ($500/mo).
- **Per-token**: Claude Code, Aider, Continue (you pay LLM API).
- **Free tier**: Continue, Cline, Aider.

Cost varies 1–2 orders of magnitude across products.

### 11. Decision matrix

| Use case | Primary recommendation |
|---|---|
| Pair-programming, high frequency | Cursor or Windsurf |
| Terminal-heavy CLI workflow | Claude Code |
| Async / autonomous tasks | Devin or Replit Agent |
| Enterprise repo with codebase intelligence | Sourcegraph Cody |
| Minimal-friction patch-based work | Aider |
| Customisable, multi-LLM | Continue or Cline |
| OpenAI ecosystem committed | OpenAI Codex CLI |

Most engineers use **one IDE-integrated** (Cursor or Windsurf) **+ one CLI** (Claude Code or Aider).

### 12. Cross-product patterns (what's converging)

In 2026:
- **MCP** as the tool protocol standard.
- **CLAUDE.md / .cursor/rules / AGENTS.md**: project-local rule files.
- **Streaming planner** as a UX standard.
- **Diff preview** before file edits.
- **Model abstraction** so users can switch.
- **Hooks** for extensibility.

Every product has these or is adding them. The differentiation is in the loop, prompts, target user, and UX.

## Empirical anchors

- **Cursor adoption** is highest among individual developers in 2025–2026 surveys.
- **Devin's autonomous-task adoption** grew rapidly in 2025; mixed reviews on quality.
- **Claude Code** is the dominant choice for terminal-first developers and for engineers who want CLI-orchestrator flexibility.
- **Sourcegraph Cody** is the enterprise default for codebase-intelligence-heavy organisations.
- **MCP** as a tool protocol has near-universal adoption among harnesses launched after 2024.
- **Multi-harness use** is common: 30–50% of engineers use more than one.

## Variants and counter-arguments addressed

- **"Just one will win."** Unlikely; different workflows have different needs.
- **"Open-source will dominate."** Mixed; some open-source (Aider, Cline, Continue) are excellent; commercial leaders persist.
- **"Devin / autonomous is the future."** Maybe; today, pair-programming is still the dominant developer workflow.
- **"Lock-in matters."** It does; abstract where possible.
- **"Choose by features."** Choose by workflow fit, not features.

## Failure modes and limitations

1. **Tool-multiplicity chaos.** Team uses three harnesses; configuration drift.
2. **Subscription bloat.** Multiple monthly subscriptions add up.
3. **Vendor outage.** Primary harness vendor has an outage; team blocked.
4. **Workflow mismatch.** Picking the wrong category (autonomous when pair-programming was needed).
5. **Skill silo.** Engineers know only their preferred tool; harder to collaborate.

## When to use, when not

**Pick by workflow.** Solo IDE → Cursor / Continue / Windsurf. Solo CLI → Claude Code / Aider. Async tasks → Devin / Replit Agent. Enterprise repo → Sourcegraph Cody.

**Use multiple if needed**: most engineers do.

**Re-evaluate annually**: products evolve fast.

## Implications for harness engineering

- **Adopt MCP for tool protocols.** [07-model-context-protocol](07-model-context-protocol.md). Universal compatibility.
- **CLAUDE.md / similar rule files.** [09-memory-files](09-memory-files.md). Project-local persistence.
- **Streaming planner UX.** [143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md).
- **Diff preview for edits.** Trust + safety.
- **Multi-model abstraction.** [119-agent-ready-llm-selection](119-agent-ready-llm-selection.md).
- **Hooks for extensibility.** [05-hooks](05-hooks.md).
- **Document your stack.** Which harness for which workflow, why.
- **Watch the field.** Products evolve; assumptions get stale.

The one-sentence takeaway: **the AI coding harness landscape in 2026 has consolidated to ~10 serious products distinguished by target user, control plane, query loop, and persistence — pick by workflow fit (solo IDE / solo CLI / async tasks / enterprise repo), use multiple if needed, and watch for convergence on MCP, CLAUDE.md, and streaming UX.**

## See also

- [29-dive-into-claude-code](29-dive-into-claude-code.md), [62-everything-claude-code](62-everything-claude-code.md) — Claude Code deep-dives.
- [52-dive-into-open-claw](52-dive-into-open-claw.md), [61-archon-harness-builder](61-archon-harness-builder.md) — alternative harnesses.
- [66-meta-harness-landscape](66-meta-harness-landscape.md), [76-ten-links-synthesis](76-ten-links-synthesis.md) — landscape views.
- [126-frameworks-comparison](126-frameworks-comparison.md) — frameworks (vs harnesses).
- [144-build-your-own-harness](144-build-your-own-harness.md) — when to build vs adopt.
- [147-vendor-lock-in](147-vendor-lock-in.md) — abstraction strategy.
