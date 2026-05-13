# 06 — Permission Modes

**Definition.** Permission modes are named postures that control which tool calls run automatically, which require user approval, and which are forbidden outright. They let an agent and its user move along a gradient of autonomy without reconfiguring from scratch.

## Problem it solves

Agents perform a wide range of tool calls — some trivial (`Read`), some risky (`git push`, `Bash rm`). A binary "ask every time" is unusably noisy; "ask for nothing" is a ticking time bomb. Users also want different postures at different times: tight control during an unfamiliar task, relaxed approvals during a tight iteration loop, full delegation for an overnight run. Permission modes encode those postures as first-class states.

They also solve a trust bootstrapping problem. As users build confidence in an agent's judgement, they can shift to a more permissive mode without editing allowlists per-tool.

## Mechanism

Claude Code ships with four canonical modes:

- **Plan** — read-only. Only Read, Grep, Glob, WebFetch-type tools run. Any mutation requires exiting plan mode. See [03-plan-mode.md](03-plan-mode.md).
- **Default** — balanced. Read/search tools auto-run; Edit, Write, Bash, and other mutations prompt for approval per call; the user may approve "just this once", "always for this tool", or deny.
- **acceptEdits** — fast iteration. Edit and Write run automatically; Bash still prompts. Intended for trusted-repo coding sessions.
- **bypass** (a.k.a. `--dangerously-skip-permissions`) — all tools run without prompting. Intended for sandboxed CI/overnight scenarios only.

The mode is orthogonal to:

- **Permission rules** in `settings.json` — fine-grained allow/deny per tool, per argument pattern. Rules override modes for specific calls.
- **Hooks** ([05-hooks.md](05-hooks.md)) — can still block any call regardless of mode.

Resolution order per call:

1. Rule-level `deny` — block, always.
2. `PreToolUse` hook exit 2 — block.
3. Rule-level `allow` — run without prompt.
4. Mode default for this tool — run, ask, or block.
5. Ask the user.

The model is aware of the current mode (it's announced in the system prompt) and can be instructed to make different trade-offs in different modes ("in plan mode, be thorough; in acceptEdits, be concise").

## Concrete pattern

Minimal `settings.json` with rules layered on modes:

```json
{
  "permissions": {
    "defaultMode": "default",
    "allow": [
      "Bash(git status:*)",
      "Bash(git diff:*)",
      "Bash(npm test:*)",
      "Read(*)",
      "Grep(*)"
    ],
    "deny": [
      "Bash(rm -rf:*)",
      "Bash(curl:*)",
      "mcp__production-db__*"
    ],
    "ask": [
      "Bash(git push:*)",
      "Write(**/.env*)"
    ]
  }
}
```

This sets a `default` posture but carves out a permanent allowlist for safe read-only commands, a deny-list for obviously dangerous ones, and a force-ask list for anything touching production or secrets.

Typical posture ladder during a session:

1. Start in **Plan** for a new unfamiliar task.
2. Review the plan; switch to **Default** to begin execution, approving each edit.
3. After a few approved edits show the agent is on-track, switch to **acceptEdits** for iteration speed.
4. Before committing or pushing, drop back to **Default** so the push call is gated.
5. Never run production-touching MCP tools outside explicit `ask`.

## Variants & related techniques

- **Role-based tool allowlists.** A subagent of type `code-reviewer` gets only read tools. The `Plan` subagent gets only read + WebFetch. This is permission modes at the subagent granularity — see [02-subagent-delegation.md](02-subagent-delegation.md).
- **Time-bounded tokens.** Some harnesses issue short-lived credentials (AWS STS, Vault-style) for tool calls, so even a compromised agent can only do damage within a window.
- **Sandboxed execution.** Combining `bypass` with container/VM isolation is the standard for long-running autonomous runs. Blast radius is physical (the sandbox), not logical (a permission rule). See [23-human-in-the-loop.md](23-human-in-the-loop.md) for hybrid patterns.
- **Capability tokens from guardrails.** Instead of per-call approval, the agent can earn a capability for a batch of similar calls by passing a safety check.

## Failure modes & anti-patterns

- **Prompt fatigue in Default mode.** If every edit requires approval, the user rubber-stamps. Fix: use acceptEdits once you trust the task, or tighten the system prompt so edits are fewer and chunkier.
- **bypass-by-default.** Some teams run CI with `bypass` against production-capable tools. A prompt injection becomes a data-exfiltration event. Fix: sandbox or deny-list production tools even in bypass.
- **Silent overrides.** A user-level rule allows something the project-level mode intended to block. Fix: adopt a least-privilege precedence (project beats user beats system) and audit rules in `settings.local.json`.
- **Mode amnesia.** The model forgets it's in Plan mode and tries to `Edit`. The harness blocks it, the model retries. Fix: reinject the current mode on every turn; treat mode as part of the system prompt, not a one-time message.
- **Coarse modes for fine-grained work.** If your codebase has zones where the agent should never write, modes alone aren't enough — layer denies on specific paths.
- **Approval-in-the-hot-path.** Approving every prompt adds latency; approvals should be async where possible (queue + notification) for long-running runs.

## When to use (and when not to)

**Use** permission modes deliberately:

- **Plan** for understanding-first tasks, new codebases, destructive migrations.
- **Default** for normal pair-programming sessions.
- **acceptEdits** for tight iteration on a well-defined task in a trusted repo.
- **bypass** only inside a sandbox, and only for autonomous runs where approvals would be impossible.

**Don't** rely on modes alone:

- Modes are a coarse dial. Pair them with per-tool rules and hooks for anything safety-critical.
- A mode label doesn't make a harness safe. An `acceptEdits` session inside a repo with `rm -rf /` history is not safer than `default`; the right answer is to clean up history and restrict Bash.

## References

- Claude Code docs, Permissions — <https://docs.claude.com/en/docs/claude-code/iam>
- Anthropic Engineering, "Building agents with the Claude Agent SDK" — <https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk>
- The New Stack, "Anthropic's harness shakeup 'just fragments workflows'" — <https://thenewstack.io/anthropic-claude-harness-restrictions/>
- Introl, "Claude Code CLI: The Definitive Technical Reference" — <https://introl.com/blog/claude-code-cli-comprehensive-guide-2025>
