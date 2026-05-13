# 23 — Human-in-the-Loop Approval

**Definition.** Human-in-the-loop (HITL) patterns route high-stakes agent decisions through a human approver before execution. The agent proposes, the human disposes. HITL is not a failure mode of autonomy — it's a design choice that trades speed for reversibility, and in many domains (finance, healthcare, compliance, production ops) it's a regulatory requirement.

## Problem it solves

Even a well-guardrailed agent occasionally proposes the wrong action. For low-stakes operations (editing a local file, opening a branch) that's fine — retry and move on. For high-stakes operations (deploying to prod, transferring money, sending customer email, deleting data) the cost of a wrong action dwarfs the cost of a human checking first. HITL makes that check explicit, auditable, and consistent.

HITL also reduces the trust-bootstrapping problem: users adopt more autonomous agents faster when they know risky actions pause for approval.

## Mechanism

A HITL pattern has four parts:

1. **Classification.** Which actions require approval? Typically expressed as a policy: domain × risk × reversibility × blast radius. Some default axes:
   - **External communication** (emails, Slack, GitHub comments).
   - **Money / account mutations** (payments, refunds, permissions grants).
   - **Infrastructure** (deploys, migrations, force-pushes, role changes).
   - **Destructive** (deletes, drops, overwrites).
2. **Proposal.** The agent, at the moment it would execute, instead emits a structured proposal: what it would do, why, expected effect, alternatives considered.
3. **Decision channel.** How the human gets the proposal — CLI prompt, Slack approval message, email with one-click approve/reject, dashboard, pager.
4. **Resume.** On approval, execute exactly the proposed action and return the result to the agent. On rejection, feed back the rejection reason so the agent can try a different approach.

Two important variants:

- **Synchronous HITL.** Agent blocks on the proposal; pops up in front of the user; waits. Good for short-running interactive sessions.
- **Asynchronous HITL.** Agent's approval request is queued; the run pauses or moves to background. Useful for long-running agents and multi-user workflows.

## Concrete pattern

A permission-mode approval prompt (Claude Code "ask" mode, see [06-permission-modes.md](06-permission-modes.md)):

```
Tool: Bash
Command: gh pr create --title "Feature: dark mode" --body "..."

Reason: submitting the feature PR per user request at step 40.

[1] Approve once
[2] Approve always for this tool/pattern
[3] Reject with reason: ___
```

Asynchronous Slack-based approval, pseudo-flow:

```python
def request_approval(agent, action, context):
    ticket = create_approval_ticket(action, context, agent_run_id=agent.run_id)
    slack.post(channel=context.channel, text=ticket.render(),
               actions=[approve_button(ticket.id), reject_button(ticket.id)])
    agent.pause_until(ticket.id)
    # …later: webhook flips the agent state to "approved" or "rejected";
    # the agent loop resumes with the outcome as a tool result.
```

HumanLayer's approach (among others) packages this pattern as an SDK: you decorate a function with `@require_approval(...)`, and the library handles the proposal/notify/resume flow.

Approval-proposal schema worth adopting:

```json
{
  "action": "db.migration.run",
  "args": { "migration_id": "0042_add_user_pref", "target": "prod" },
  "expected_effect": "Adds `user_pref` JSONB column to `users`. ~50M rows backfilled to default {}.",
  "risk": "non-reversible schema change; affects live table",
  "alternatives": ["Run in staging first", "Wait for Thursday freeze window"],
  "why_now": "user explicitly asked; staging passed",
  "blast_radius": "ops"
}
```

The model is forced to articulate effect, risk, and alternatives — which catches some mistakes before the human ever looks.

## Variants & related techniques

- **Permissions with ask mode** ([06-permission-modes.md](06-permission-modes.md)) — simplest HITL: per-tool approval prompts.
- **Hooks** ([05-hooks.md](05-hooks.md)) — a `PreToolUse` hook that calls an approval service is one implementation.
- **Sandboxed run + diff review.** Agent runs its full plan in a sandbox; outputs a diff the human approves/rejects wholesale. Common in infra / agent-generated PRs.
- **Quorum approval.** Two humans must approve. Standard in SOX, SOC2, financial contexts.
- **Tiered autonomy.** Actions below threshold auto-run; between thresholds need approval; above are blocked outright.
- **Learning-from-approval.** Approved/rejected proposals form a dataset that fine-tunes or few-shots the model to avoid unnecessary requests.

## Failure modes & anti-patterns

- **Approval fatigue.** Every action needs approval; users rubber-stamp. Fix: narrow the class of actions requiring approval; raise the bar above a trust threshold.
- **Context-free approvals.** Human sees "run this bash command" without knowing why. Fix: proposals must include purpose, effect, alternatives.
- **Blocking runs forever.** Agent waits on approval that nobody's watching. Fix: timeouts with defined fallback (pause, reject, escalate), and tight SLAs for high-importance paths.
- **Over-delegation.** Agent proposes a string of trivially-approved actions that together constitute a dangerous one. Fix: approve *plans*, not just individual actions, for coordinated changes.
- **Under-specification.** Approved action is a shell command; the shell command later does more than approved. Fix: parameter-level binding, not free-form text.
- **Slack-fatigue.** Async approvals pile up; nobody notices the important ones. Fix: prioritize, reroute to on-call, or cut volume.
- **Approval without audit.** No record of who approved what and why. Fix: every approval must be logged with actor, timestamp, justification.

## When to use (and when not to)

**Use** HITL when:

- Actions are high-stakes, destructive, or irreversible.
- Regulatory or contractual obligation requires human accountability.
- Trust in the agent is still being built.
- Downstream systems are expensive to undo (prod data, external recipients).

**Don't** use HITL when:

- Actions are genuinely low-stakes — you'll dilute attention for the high-stakes ones.
- No human is available in the agent's time window (autonomous overnight runs). Instead, sandbox aggressively and accept the risk/reversibility trade-off.
- HITL would defeat the agent's purpose (e.g., real-time routing decisions).

## References

- Claude Code docs, Permissions and hooks — <https://docs.claude.com/en/docs/claude-code/iam>
- HumanLayer, "Skill Issue: Harness Engineering for Coding Agents" — <https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents>
- LangChain human-in-the-loop agents — <https://python.langchain.com/docs/how_to/chat_model_human_in_the_loop/>
- AWS, "Human-in-the-loop with Amazon Bedrock Agents" — <https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html>
- Shreya Shankar, essays on AI systems design — <https://www.sh-reya.com/>
- Anthropic Engineering, "Building agents with the Claude Agent SDK" — <https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk>
