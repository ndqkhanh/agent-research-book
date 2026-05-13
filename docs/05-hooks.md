# 05 — Hooks

**Definition.** Hooks are deterministic shell commands, HTTP calls, LLM invocations, or agent spawns that fire on harness events — before or after a tool call, on user prompt submission, on session stop, on notification. They let the harness (not the model) enforce invariants, inject context, or reject actions that violate policy.

## Problem it solves

You cannot trust the model to enforce its own rules. If you tell the model "always run the linter after editing a file", it will follow the rule about 95% of the time — which means ~1 in 20 turns you ship unlinted code. Hooks are the escape hatch: they are code, not prompts. A `PostToolUse` hook on `Edit` that runs the linter is 100% reliable because the harness executes it, not the model.

Hooks are also how you *block* unsafe actions before they happen. A `PreToolUse` hook on `Bash` that inspects the command for `rm -rf ~` can refuse the call deterministically, regardless of whether the model was prompt-injected into trying.

## Mechanism

A hook is registered in `settings.json` (project, user, or system scope) under an event name, optionally with a matcher:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "~/bin/audit-bash.sh" }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "make lint-changed" }
        ]
      }
    ],
    "UserPromptSubmit": [
      { "hooks": [{ "type": "command", "command": "~/bin/inject-context.sh" }] }
    ]
  }
}
```

Event catalog (Claude Code, representative — 24+ event points across systems):

- `UserPromptSubmit` — fires when the user sends a prompt. Stdout is prepended to the prompt: useful for injecting project state.
- `PreToolUse` — before a tool runs. Exit code 2 blocks the tool; stdout can override or annotate the call.
- `PostToolUse` — after a tool runs. Can add a system message, run linters, etc.
- `Stop` / `SubagentStop` — end of a turn / subagent. Often used to run verification steps.
- `Notification` — surfaces to the user via shell, desktop, or webhook.
- `SessionStart`, `SessionEnd`, `PreCompact`, `PostCompact` — lifecycle.

Hook types:

- **shell** — run a bash command. Fast, cheap, deterministic. Most common.
- **HTTP** — call a webhook. Useful for logging, approvals, external policy engines.
- **LLM** — invoke another model to classify/validate. Heavier but flexible.
- **agent** — spawn a subagent for nontrivial policy decisions.

Matchers use regex against the tool name: `Edit`, `Edit|Write`, `mcp__.*` (all MCP tools), `*` (everything). Multiple hooks on the same event run in order; if any exits with code 2, the tool call is blocked.

## Concrete pattern

Blocking a destructive command:

```bash
#!/bin/bash
# ~/bin/audit-bash.sh — blocks dangerous Bash commands.
input=$(cat)  # JSON from stdin: {"tool_input": {"command": "..."}, ...}
cmd=$(echo "$input" | jq -r '.tool_input.command // ""')

if echo "$cmd" | grep -qE '(rm -rf|:(){:|:&};:|mkfs|dd if=.* of=/dev)'; then
  echo "Blocked: destructive command pattern detected." >&2
  exit 2
fi
exit 0
```

Auto-linting after edits:

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "npx eslint --fix ${CLAUDE_FILE_PATHS} 2>&1 || true"
      }]
    }]
  }
}
```

Auto-injecting repo state:

```json
{
  "hooks": {
    "UserPromptSubmit": [{
      "hooks": [{
        "type": "command",
        "command": "git status --porcelain | head -20 && git log -3 --oneline"
      }]
    }]
  }
}
```

## Variants & related techniques

- **Permissions** ([06-permission-modes.md](06-permission-modes.md)) are the declarative cousin of hooks. A permission rule says *"this tool cannot run without approval"*. A hook says *"here's code that decides"*. Use permissions for static rules; hooks for dynamic policy.
- **Guardrails** ([22-guardrails-prompt-injection.md](22-guardrails-prompt-injection.md)) can be implemented as hooks: run a jailbreak classifier on `UserPromptSubmit`.
- **Observability** ([24-observability-tracing.md](24-observability-tracing.md)) frequently hooks into `PreToolUse`/`PostToolUse` to emit spans.
- **Compaction** ([08-context-compaction.md](08-context-compaction.md)) exposes `PreCompact`/`PostCompact` hooks so you can persist state that would otherwise be lost.

## Failure modes & anti-patterns

- **Slow hooks.** A 10-second lint on `PostToolUse(Edit)` multiplied across a 50-step session is 8 minutes of dead time. Fix: make hooks incremental (lint only the changed file), run async where correctness allows, or move to `Stop` (end of turn).
- **Hooks that talk to the model.** If the hook's stdout is long, it leaks into context every turn. Keep output short and structured.
- **Hook cascades.** Hook A triggers tool B, whose `PostToolUse` hook C triggers tool D. Debugging becomes nightmarish. Keep hooks idempotent and avoid chains.
- **Secret leakage.** A hook that echoes env vars to stdout writes them into the transcript. Audit hook output channels.
- **Silent blocking.** A `PreToolUse` hook returns exit 2 without a clear message. The model doesn't know why its action failed and retries. Always include a message on stderr; the harness relays it to the model.
- **Over-reliance.** If the model needs 12 hooks to behave, the system prompt probably needs rewriting. Hooks are safety nets, not the first line of correctness.

## When to use (and when not to)

**Use** hooks when:

- A rule must be enforced deterministically (safety, compliance, "always run X after Y").
- You need observability you can't get from the model (timing, cost, exit codes).
- You need to inject live external state on every prompt.
- A policy decision benefits from code (lookups, regex, auth checks) rather than prompting.

**Don't** use hooks when:

- The rule is a preference the model generally follows — just put it in the system prompt.
- The hook is going to run a 30-second process on every edit — reach for a different lifecycle point.
- You'd be better off with a tighter permission rule or a narrower tool allowlist.

## References

- Claude Code docs, Hooks — <https://docs.claude.com/en/docs/claude-code/hooks>
- Anthropic Engineering, "Building agents with the Claude Agent SDK" — <https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk>
- alexop.dev, "Understanding Claude Code's Full Stack" — <https://alexop.dev/posts/understanding-claude-code-full-stack/>
- awesome-claude-code (community curation of hooks/skills/commands) — <https://github.com/hesreallyhim/awesome-claude-code>
