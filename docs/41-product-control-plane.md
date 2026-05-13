# 41 — The Product Control Plane for Multi-Agent Systems

**Definition.** Adaline Labs' "Missing Product Layer for Multi-Agent Systems" (April 2026) argues that the reason most agentic AI pilots die before production is not model capability but the absence of a **product control plane** — the governance layer *above* the models that handles authority distribution, task handoffs, user visibility, and failure recovery. The piece proposes four essential primitives — **Permissions, Handoffs, Visibility, Recovery** — and the PM/engineering discipline around them.

## Problem it solves

The article's blunt observation: roughly 1 in 10 agentic AI use cases reach production, not because the models can't do the work, but because teams over-invest in agent capability and under-invest in the governance surface. Without an explicit product control plane, multi-agent systems fail predictably:

- An agent performs an action it shouldn't have authority for (Permissions gap).
- Context is lost when work transfers between agents (Handoffs gap).
- The user can't see what the system is doing or intervene (Visibility gap).
- When things go wrong, there is no recovery path — just an error message (Recovery gap).

Each gap is a PM-shaped problem disguised as engineering, or an engineering-shaped problem disguised as product. The control plane makes the interface between the two explicit.

## Mechanism — the four primitives

### 1. Permissions (semantic, not just access-level)

Go beyond RBAC. Each agent's authority is defined not just by "can access tool X" but "can perform operation O on data D under condition C." Example: a customer-support agent can read order details but can issue refunds only under $50 and only for orders in the last 30 days.

### 2. Handoffs (the highest-risk moments)

A handoff is when agent A's work becomes agent B's responsibility. Handoffs carry three risks: incomplete context, ambiguous authority, and invisible failures. The control plane specifies for each handoff: what context transfers, who holds authority after the transfer, and how failure is detected and attributed.

### 3. Visibility (for users and operators)

Users need to understand what the multi-agent system is doing *while it's doing it*, not just in postmortem. The control plane defines user-visible states ("I'm gathering info", "Agent B is drafting the reply", "Waiting for your approval") and operator-facing telemetry (trajectory traces, handoff logs, denial logs).

### 4. Recovery (explicit fallback paths)

Plan for failure: retries with progressively simpler prompts, route to a simpler workflow, degrade gracefully to a scripted response, escalate to a human. Each path is named, tested, and monitored. "We return an error" is not a recovery path.

## Concrete pattern — PRD discipline for multi-agent systems

The piece argues PRDs should specify:

```
Agents:
  - name, role, scope
Permissions:
  - operations × data × conditions
  - prohibited operations (explicit deny list)
Delegation conditions:
  - when agent A hands off to B (signals, triggers)
Handoff specification:
  - transferred context schema
  - authority transition: who owns the task after
  - failure attribution rules
User-visible states:
  - enum + display text
Escalation triggers:
  - when to involve a human; who is paged
```

Engineering side, before production launch:

```
- agent-step tracing
- handoff logging (with context diffs)
- permission-denial telemetry
- trajectory-level evaluation
- per-path recovery tests
```

## Variants & related techniques

- **Permission modes** ([06-permission-modes.md](06-permission-modes.md)) — the coarse dial; Adaline's semantic permissions are the fine-grained extension.
- **Hooks** ([05-hooks.md](05-hooks.md)) — deterministic enforcement of permissions and handoff invariants.
- **Human-in-the-loop** ([23-human-in-the-loop.md](23-human-in-the-loop.md)) — Recovery's ultimate path.
- **Observability / tracing** ([24-observability-tracing.md](24-observability-tracing.md)) — the Visibility substrate.
- **Verifier/evaluator loops** ([11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md)) — automated handoff-quality checks.

## Failure modes & anti-patterns

- **"We'll figure out handoffs later."** Handoffs are where most agentic systems fail; deferring design of the transfer schema is deferring the hardest problem.
- **Permissions as RBAC.** Access control alone doesn't prevent "allowed but inappropriate" actions. Semantic constraints are load-bearing.
- **Visibility as logs.** Backend logs aren't visibility *to users*. Surface agent state in the product.
- **Recovery as error-handling.** Catching exceptions is not recovery; named fallback paths are.
- **Engineering-only framing.** The product control plane is a PM artifact as much as engineering. Treat it as product scope.
- **Copy-paste from single-agent patterns.** Single-agent permission models transfer poorly to multi-agent; the emergent authority conflicts only appear with multiple agents.

## When to use (and when not to)

**Use** product-control-plane thinking when:

- You're building any multi-agent system (including one-user-one-agent-plus-subagents).
- You want the system to reach production, not just demo.
- You have multiple stakeholders (end users, operators, auditors).

**Scope down** when:

- The "system" is a single agent with tightly-scoped tools; single-agent primitives (permission modes + hooks + HITL) suffice.
- No meaningful handoffs exist; you're building a monolithic tool, not a system.

## References

- Adaline Labs, "The Missing Product Layer for Multi-Agent Systems" (April 2026). <https://labs.adaline.ai/p/multi-agent-systems-product-control-plane>
- Anthropic Engineering, "Building agents with the Claude Agent SDK". <https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk>
- Cognition AI, "Don't Build Multi-Agents". <https://cognition.ai/blog/dont-build-multi-agents>
- LangChain Deep Agents. <https://blog.langchain.com/deep-agents/>
- Claw-Eval (arXiv:2604.06132). <https://arxiv.org/abs/2604.06132>
