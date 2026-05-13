# 20 — MetaGPT Role-Based Workflows

**Definition.** MetaGPT (Hong et al. 2023) is a multi-agent framework that imposes a software-engineering SDLC onto LLM agents: distinct role-agents (Product Manager, Architect, Project Manager, Engineer, QA) exchange *structured artifacts* (PRDs, design docs, API specs, task lists, test reports) via Standard Operating Procedures (SOPs). The idea generalizes: any multi-agent system gains coherence when you constrain inter-agent communication to structured artifacts rather than free-form chat.

## Problem it solves

Naive multi-agent chat — a group of agents talking to each other in natural language — degenerates fast. Agents repeat themselves, miss one another's points, escalate tangents, and lose track of the shared state. The resulting outputs are inconsistent because there is no single source of truth the agents can align on. Human software teams don't work this way: they produce PRDs, design docs, task lists, and code reviews — each a structured artifact that constrains downstream work.

MetaGPT formalizes this: replace chat with document hand-offs; replace ad-hoc roles with SDLC roles; replace hope with SOPs. The resulting system produces end-to-end software generation that is dramatically more coherent than free-form multi-agent setups on the same models.

## Mechanism

Core elements:

1. **Roles.** Product Manager, Architect, Project Manager, Engineer, QA Engineer — each is an LLM agent with its own system prompt emphasizing its responsibilities and output format.
2. **Artifacts.** Each role emits a structured document: PRD (requirements + user stories), system design (architecture diagrams, data structures, API specs), task list (broken-down tasks), code files, test files, test reports. Documents are the exclusive medium of cross-role communication.
3. **Standard Operating Procedures (SOPs).** The workflow explicitly routes artifacts between roles: PRD → design → tasks → code → tests → QA report. Later roles can ask earlier ones for clarification but cannot bypass the SOP.
4. **Message publishing / subscribing.** A shared message queue lets each role subscribe to the artifacts it cares about; roles process messages asynchronously.
5. **Executable feedback.** Engineers produce runnable code; QA runs it; failures feed back as structured test reports that the Engineer must address.

The system prompt for each role is narrow and job-specific, which reduces conflicting behavior versus one-agent-does-everything setups. The artifact templates are the architecture.

## Concrete pattern

A condensed PRD template the PM role fills in:

```markdown
# Product Requirements Document

## Original requirement
<user's raw ask>

## Goals
- Goal 1, measurable
- Goal 2, measurable

## User stories
- As a <role>, I want to <action>, so that <benefit>.

## Competitive analysis
- Similar product A — pros, cons

## Anti-goals
- Explicitly not building X

## UI draft (ASCII or reference)

## Requirement pool
- P0: <must>
- P1: <should>
- P2: <could>
```

The Architect consumes the PRD, emits:

```markdown
# System Design

## Architecture
- Components, responsibilities
- Tech stack + versions

## Data structures / API spec
- TypeScript/OpenAPI/Proto, literal.

## File list
- src/main.py — entry
- src/api/users.py — CRUD
- ...

## Open questions
- (clarifications Engineer may need)
```

Engineer then generates code one file at a time, referencing the file list, with QA running tests between iterations.

## Variants & related techniques

- **AutoGen** (Microsoft) is a sibling framework: agents, roles, group chat. More flexible, less opinionated than MetaGPT about SOPs and artifacts.
- **CAMEL** pairs two agents in role-play (user ↔ assistant); simpler, same spirit.
- **ChatDev** applies a similar SDLC framing to software-building agent crowds.
- **Claude Code subagents** ([02-subagent-delegation.md](02-subagent-delegation.md)) with curated system prompts and tool allowlists are a lightweight version.
- **Plan Mode + verifier loops** ([03-plan-mode.md](03-plan-mode.md), [11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md)) — a two-role version (planner + doer + evaluator) often yields most of MetaGPT's gains with less orchestration.

## Failure modes & anti-patterns

- **Over-orchestration.** Five roles for a task that really wants one. The artifact bureaucracy costs more than it saves. Fix: scale role count to task complexity; start with two.
- **Artifact rot.** The PRD is written once and never updated as the design evolves. Engineer reads an obsolete spec. Fix: downstream roles must cite artifact versions; amendments flow back up.
- **Rigidity.** The SOP doesn't accommodate small, obvious changes; every task runs a full pipeline. Fix: let roles short-circuit trivial work (e.g., a typo fix bypasses PM → Architect → PM).
- **Role tunnel vision.** The Engineer sees only the file list and code; the Architect sees only requirements. Cross-cutting issues (performance, security) fall in the gaps. Fix: a dedicated "review" role or cross-cutting checklist in each artifact template.
- **Cost.** N roles × M iterations × several prompt turns per iteration. MetaGPT-style workflows can be 10× the cost of a single agent. Worth it only for work that reaches production.
- **Hallucinated artifacts.** Architect cites API routes the PM didn't request; downstream roles build them. Fix: schema-validate artifacts against a template; force each section to reference the artifact above.
- **Multi-agent sprawl vs. coherence.** Without Cognition's "share context" principle or MetaGPT's artifact discipline, multi-agent systems drift apart — see [02-subagent-delegation.md](02-subagent-delegation.md).

## When to use (and when not to)

**Use** MetaGPT-style role workflows when:

- The task is production-quality software generation end-to-end.
- Multiple artifacts (requirements, design, code, tests) are genuinely needed.
- The cost of rigor is less than the cost of rework.
- You need traceability — each artifact is a checkpoint.

**Don't** use them when:

- The task is small or exploratory — lightweight plan/execute is enough.
- You cannot constrain artifacts to schemas — free-form roles become free-form chat.
- You cannot tolerate the cost and latency.

## References

- Hong et al., "MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework", arXiv:2308.00352 — <https://arxiv.org/abs/2308.00352>
- MetaGPT GitHub — <https://github.com/geekan/MetaGPT>
- Wu et al., "AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation", arXiv:2308.08155 — <https://arxiv.org/abs/2308.08155>
- Qian et al., "Communicative Agents for Software Development" (ChatDev), arXiv:2307.07924 — <https://arxiv.org/abs/2307.07924>
- Cognition, "Don't Build Multi-Agents" (counter-argument for why context-sharing beats role-sharding) — <https://cognition.ai/blog/dont-build-multi-agents>
