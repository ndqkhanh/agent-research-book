# 36 — Autogenesis: A Self-Evolving Agent Protocol

**Definition.** Autogenesis (NTU, arXiv:2604.15034) is a protocol-level framework for agent systems that enables agents to *evolve their own resources* — prompts, agents, tools, environments, memory — with explicit state, versioned interfaces, lifecycle, and auditable lineage. It operates through two layers: **RSPL** (Resource Substrate Protocol Layer) models five entity types as protocol-registered resources, and **SEPL** (Self Evolution Protocol Layer) implements a closed-loop operator interface for proposing, assessing, and committing improvements with rollback.

## Problem it solves

Most "self-improving" agent research so far — Voyager, Reflexion, Hyperagents — improves *behavior* inside an otherwise fixed substrate. Deeper self-improvement (editing the system prompt, rewriting tools, reshaping memory schemas) routinely breaks things in opaque ways: monolithic compositions and brittle glue code drop the very capabilities that were working. The practical question is not "can agents self-modify?" but "can agents self-modify *with the engineering discipline of production software* — versioning, rollback, audit"?

Autogenesis is the protocol-level answer: treat every agent resource as a first-class entity with its own lifecycle. Evolution is then a change to a typed, versioned resource — diffable, committable, revertable — not a diffuse retraining event.

## Mechanism

### RSPL — Resource Substrate Protocol Layer

Five resource types, each with explicit state, lifecycle, and versioned interfaces:

1. **Prompt** — system prompt, role prompts, few-shot exemplars.
2. **Agent** — subagent definitions, tool allowlists, system posture.
3. **Tool** — schemas, executors, permission metadata.
4. **Environment** — sandboxes, worktrees, credential scopes.
5. **Memory** — episodic, semantic, scratchpad stores.

Each resource has a URI, a version, a schema, and a lifecycle state (draft → active → deprecated → retired). Changes are patches with lineage.

### SEPL — Self Evolution Protocol Layer

A closed-loop operator interface:

1. **Propose.** The agent (or a meta-agent) proposes a change to a resource: "amend `prompt/researcher@v3` to add these edge-case instructions."
2. **Assess.** Automated evaluation (benchmark runs, LLM judges, golden traces) scores the proposed version against baseline.
3. **Commit.** If the assessment passes gating criteria, the new version becomes active. The prior version remains retrievable for rollback.
4. **Rollback.** On observed regression or external signal, revert to a prior version atomically.

### Autogenesis System (AGS)

The runtime dynamically instantiates, retrieves, and refines protocol-registered resources during execution. What evolves (resource) is decoupled from how it evolves (protocol), which the paper positions as its primary advance over prior work on A2A-style agent-to-agent protocols and MCP-style tool protocols.

## Concrete pattern

```
Resource URI:    prompt://researcher@v5
Patch proposal:  + "When retrieving papers, always check publication year;
                  if newer than knowledge cutoff, warn in the output."
Eval gate:       run on golden set of 50 research tasks; must not regress
                  on existing scores by more than 1 point.
Commit:          atomic swap from v4 → v5 for future runs.
Rollback:        observed 7-day regression on metric M ⇒ auto-revert.
```

This is git + CI + feature flags applied to *every part of the agent system*, not just code.

## Variants & related techniques

- **Hyperagents / DGM-H** ([45-hyperagents-self-modification.md](45-hyperagents-self-modification.md)) — Meta's approach; more focused on the *meta-level editing procedure itself being editable*. Autogenesis is more protocol-operational.
- **Reflexion** ([14-reflexion.md](14-reflexion.md)) — narrow self-improvement via verbal reflections; a subset of SEPL's scope.
- **Voyager** ([19-voyager-skill-libraries.md](19-voyager-skill-libraries.md)) — skill-library accumulation; could be framed as Memory + Tool resources under RSPL.
- **Memory files** ([09-memory-files.md](09-memory-files.md)) — a simple static analogue: versioned memory as files.
- **Agentic Context Engineering (ACE, arXiv:2510.04618)** — evolving-playbook view of context with similar incremental-update discipline.

## Failure modes & anti-patterns

- **Evaluation gaps.** If the "assess" step isn't strong enough, bad versions commit. Fix: invest in the eval suite as much as the evolution loop — see [21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md).
- **Reward hacking at the resource level.** Agent proposes prompt changes that boost the metric but hurt real users. Fix: hold-out evals; human spot-checks; metric diversity.
- **Rollback hell.** Too many versions, partial rollbacks inconsistent across resources. Fix: coordinated rollback of related resources; atomic versions per release.
- **Version explosion.** Thousands of prompt patches; the operator interface becomes unreadable. Fix: periodic squash, lineage summaries.
- **Operator latency.** Every proposal is evaluated, serializing change. Fix: parallel evaluation; risk-tiered gating (low-risk resources get lighter evals).
- **Opaque lineage.** Hard to answer "why is the researcher prompt the way it is today?" Fix: lineage messages are part of the protocol, not optional.

## When to use (and when not to)

**Use** an Autogenesis-like protocol when:

- Your agent system has many parts that should evolve (prompts, tools, skills).
- You want self-improvement without sacrificing production discipline.
- You have the eval infrastructure to gate changes.

**Don't** when:

- The system is small — a git repo and manual releases suffice.
- You cannot afford the evaluation budget — evolution without eval is worse than stasis.
- You need strict change control with human sign-off on every resource — Autogenesis is compatible but adds overhead.

## References

- arXiv:2604.15034 — "Autogenesis: A Self-Evolving Agent Protocol" (NTU, April 2026). <https://arxiv.org/abs/2604.15034>
- "Agentic Context Engineering" (arXiv:2510.04618). <https://arxiv.org/abs/2510.04618>
- Model Context Protocol specification. <https://modelcontextprotocol.io/>
- DGM-H / Hyperagents (arXiv:2603.19461). <https://arxiv.org/abs/2603.19461>
- Reflexion (arXiv:2303.11366). <https://arxiv.org/abs/2303.11366>
