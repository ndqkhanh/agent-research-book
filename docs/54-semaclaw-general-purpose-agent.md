# 54 — SemaClaw: Harness-Engineered General-Purpose Personal Agents

**Definition.** SemaClaw (Midea AIRC, [arXiv:2604.11548](https://arxiv.org/abs/2604.11548), April 2026; code at [github.com/midea-ai/SemaClaw](https://github.com/midea-ai/SemaClaw)) is an open-source multi-agent application framework that positions itself as a step toward **general-purpose personal AI agents through harness engineering**. Its four named contributions — (1) a DAG-based two-phase hybrid agent-team orchestration, (2) the **PermissionBridge** behavioral-safety system, (3) a three-tier context management architecture anchored by a **SOUL.md** persona partition, and (4) an **agentic wiki skill** for automated personal knowledge-base construction — together form an opinionated take on what the next-generation personal-agent harness looks like.

## Problem it solves

The rise of OpenClaw in early 2026 ([file 52](52-dive-into-open-claw.md)) pushed personal AI agents from demo to mass deployment — millions of users delegating travel planning, multi-step research, and household coordination. Two bottlenecks surfaced immediately:

1. **Orchestration fragility.** Free-form multi-agent chat produces incoherent outputs (the Cognition "don't build multi-agents" argument); purely static pipelines can't adapt. Neither extreme handles real personal tasks.
2. **Trust & persona drift.** Personal agents that forget preferences across sessions, or silently change tone, erode the long-term relationship that makes the product valuable. OpenClaw ships strong primitives but treats persona and cross-session identity largely as conventions.

SemaClaw argues the harness layer — not the model — is where these are solved, and ships a first concrete architecture designed for it.

## Mechanism — the four contributions

### 1. DAG Teams (two-phase hybrid orchestration)

A **two-stage** scheme that combines dynamic planning with graph-structured execution:

- **Stage 1 — dynamic decomposition.** An LLM planner emits a task DAG whose nodes are sub-tasks and whose edges are explicit dependencies. Nodes may be typed (search, reason, call-tool, hand-off).
- **Stage 2 — deterministic scheduling.** A non-LLM scheduler walks the DAG, dispatching nodes when their predecessors complete, running independent branches in parallel, and collecting partial failures at node granularity.

The win is **fault locality + observability** with dynamic flexibility: a node-level failure does not poison the whole plan, and every step is recoverable with its own retry logic. DAG Teams is the personal-agent analogue of the patterns described in [17 — ReWOO](17-rewoo.md) and [02 — Subagent Delegation](02-subagent-delegation.md), tightened for personal-task scale.

### 2. PermissionBridge (behavioral safety as a runtime primitive)

Instead of treating permission as a layer of prompts or wrapper policies, PermissionBridge **embeds authorization checkpoints as first-class execution primitives** in the runtime. High-risk operations — destructive file ops, external messaging, payment-tier API calls — cannot fire until an explicit authorization token is present on the operation's execution context.

Key design points:

- The check is not performed by the LLM. A deterministic runtime primitive compares the proposed operation's risk tier against the user's standing authorization set.
- Authorization can be long-lived (session-scoped), bounded (approve-once), or bundled (pre-authorize a pattern for the next N minutes).
- PermissionBridge integrates with the DAG scheduler: a node flagged "requires_permission" is parked until the token arrives; the rest of the DAG continues.

This is the same philosophical bet as [06 — Permission Modes](06-permission-modes.md) and [05 — Hooks](05-hooks.md), but elevated from "configure in settings.json" to "schema-declared on every action."

### 3. Three-tier context architecture (with SOUL.md)

Context in SemaClaw spans three layers:

1. **Compressed working memory.** Short-lived, per-task context that survives within a DAG run — the equivalent of Orion-Code's [context engine](../../projects/orion-code/blocks/04-context-engine.md) layers L3–L4.
2. **Retrieval-based external memory.** Durable episodic and semantic content indexed for similarity retrieval — close to classical agentic RAG ([25](25-agentic-rag.md)), but the *agent's own history* rather than a general knowledge base.
3. **Persona partition anchored by `SOUL.md`.** A persistent, human-authored-and-agent-amended file that carries the agent's identity: its role, tone, relational conventions, long-standing constraints, values. Loaded into every session; never compacted away; version-controlled.

SOUL.md is SemaClaw's operational answer to the identity-drift problem. It is cousin to [09 — Memory Files](09-memory-files.md) but narrower: not "everything the agent remembers," just "who the agent *is*." That boundary keeps SOUL.md small, stable, and trustworthy.

### 4. Agentic wiki skill (automated personal knowledge base)

An always-on background skill that watches the session stream and promotes durable facts — decisions, preferences, people, recurring tasks — into a user-private knowledge wiki. The wiki's entries are:

- **Diffable and editable** by the user.
- **Cited** back to the conversation turn that introduced them.
- **Decayed** on staleness unless refreshed.

Future turns retrieve from the wiki before consulting the general memory store, so personal facts get precedence over generic retrieval.

## Concrete pattern

A stylized SemaClaw personal task — "Plan next weekend's trip to Kyoto":

```
PLAN (LLM)               SCHEDULER (deterministic)
===========             =========================
#E1 search flights       → parallel
#E2 search hotels        → parallel
#E3 read SOUL.md           → persona: "budget-conscious, morning flights"
#E4 wiki.lookup(travel_prefs) → prefers short-haul, avoids connections
#E5 filter(#E1, #E3, #E4)  ← depends on E1,E3,E4
#E6 filter(#E2, #E3, #E4)  ← depends on E2,E3,E4
#E7 PERMISSION needed: book.flight   ← parked, pending user token
#E8 draft itinerary        ← depends on E5,E6
#E9 PERMISSION needed: send.calendar_invite ← parked
```

Nodes E7 and E9 do not block E8. The agent produces the itinerary; the user authorizes the booking and calendar send when ready. The wiki is updated with anything new (e.g., "user prefers Kyoto over Osaka for culture trips") at session close.

## Variants & related techniques

- **OpenClaw ([52](52-dive-into-open-claw.md))** — SemaClaw's direct predecessor and contrast. OpenClaw ships Gateway + pluggable harness; SemaClaw specializes the harness with DAG Teams + PermissionBridge + SOUL.md.
- **MetaGPT ([20](20-metagpt-role-based-workflows.md))** — shares the structured-artifact discipline (SemaClaw's DAG is an artifact, its wiki is an artifact), but MetaGPT is SDLC-shaped; SemaClaw is personal-task-shaped.
- **LangChain Deep Agents ([42](42-langchain-deep-agents.md))** — comparable in intent (open harness with planning + subagents); different in emphasis — Deep Agents foregrounds planning + virtual filesystem; SemaClaw foregrounds identity + permission.
- **Externalization survey (arXiv:2604.08224)** and **M⋆ ([arXiv:2604.11811](https://arxiv.org/abs/2604.11811))** are adjacent 2026 papers in the same research thread.
- **Skills ([04](04-skills.md))** — the agentic wiki is a skill; SOUL.md is distributed as a skill-adjacent artifact.

## Failure modes & anti-patterns

- **DAG hallucination.** The planner emits edges that don't actually encode dependencies, or misses a real one. Mitigation: schema validation on the DAG; runtime detection of data-dependency violations; replanner for detected cycles / orphans.
- **PermissionBridge bypass via prompting.** A mis-placed "LLM decides if the op is allowed" check would be fatal. Discipline: the check is runtime-enforced, never a model decision.
- **SOUL.md bloat.** Treating SOUL.md as "everything about the user" turns it into a general memory store and defeats its purpose. Discipline: hard size cap; strict "identity + conventions only" scope.
- **Wiki pollution.** The agentic wiki may promote a misheard fact. Mitigation: diffable entries; user-visible surface; TTL decay on unrefreshed items; citation to source turn.
- **DAG + permission deadlock.** A node parked on permission blocks downstream nodes. Mitigation: scheduler treats permission-pending as a non-terminal state and parallelizes around it; explicit time-out with escalation to user.
- **Two-phase cost.** The LLM-plan step pays an upfront cost that small tasks don't need. Mitigation: confidence-gated "skip DAG" path for trivial tasks.

## When to use (and when not to)

**Use** SemaClaw-style harness when:

- You are building a **personal** agent product (vs. ops, research, coding).
- Cross-session identity and relationship continuity matter.
- Tasks are non-trivial and benefit from parallelism + fault locality.
- You want permission enforcement that's provably not gamable by the model.

**Don't** use it when:

- The problem is single-shot Q&A — DAG Teams overhead isn't justified.
- The agent's role is one-off and has no persona to preserve.
- You need a fully-managed SaaS; SemaClaw is a framework, not a product.

## References

- SemaClaw paper: arXiv:2604.11548 — <https://arxiv.org/abs/2604.11548>; HTML at <https://arxiv.org/html/2604.11548>.
- SemaClaw GitHub (Midea AIRC) — <https://github.com/midea-ai/SemaClaw>.
- OpenClaw ([file 52](52-dive-into-open-claw.md)) — direct predecessor; contrast on orchestration philosophy.
- *Externalization in LLM Agents: A Unified Review of Memory, Skills, Protocols and Harness Engineering* — arXiv:2604.08224.
- *M⋆: Every Task Deserves Its Own Memory Harness* — arXiv:2604.11811.
- Related in-folder files: [02 — Subagent Delegation](02-subagent-delegation.md), [04 — Skills](04-skills.md), [06 — Permission Modes](06-permission-modes.md), [09 — Memory Files](09-memory-files.md), [17 — ReWOO](17-rewoo.md), [20 — MetaGPT Role-Based Workflows](20-metagpt-role-based-workflows.md), [25 — Agentic RAG](25-agentic-rag.md), [42 — LangChain Deep Agents](42-langchain-deep-agents.md), [52 — Dive Into OpenClaw](52-dive-into-open-claw.md).
