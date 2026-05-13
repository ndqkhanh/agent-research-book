# Recommended Breakthrough Project — Gnomon

**Status: proposal, April 2026.** Synthesised from the 66-file corpus, the three April-2026 SEA deep-dives ([57](./57-sea-arxiv-2604-15034.md), [58](./58-sea-arxiv-2507-21046.md), [59](./59-sea-arxiv-2508-07407.md)), the 2026 SEA landscape and github survey ([56](./56-sea-landscape-2026.md), [60](./60-sea-top-github-repos.md)), and the five open-source harness-builder deep-dives ([61 Archon](./61-archon-harness-builder.md), [62 everything-claude-code](./62-everything-claude-code.md), [63 RAGFlow](./63-ragflow-agent-patterns.md), [64 LobeHub](./64-lobehub-ai-framework.md), [65 DeerFlow](./65-deer-flow-bytedance.md)) plus the meta-harness synthesis ([66](./66-meta-harness-landscape.md)).

## TL;DR — what to build right now

Build **Gnomon: a harness-aware evaluator with a built-in closed evolution loop, exposed as a portable harness IR.** In one sentence: Gnomon ingests traces from *any* harness (Claude Code, LangGraph, Archon, DeerFlow, Cursor, Claude Agent SDK, OpenClaw), attributes every failure to a specific **harness primitive** (not just "the agent failed"), and emits reversible **Autogenesis-shaped resource patches** back to the host harness that fix the attributed failure — with rollback if the patch regresses.

Gnomon is the first open-source project that unifies the three gaps the April-2026 landscape leaves open simultaneously:

- **G1** — A harness IR (no LLVM-equivalent exists in 2026).
- **G2** — Primitive-level failure attribution (trace stores exist; *why* a trace failed at the harness-primitive level does not).
- **G3** — Chaos-engineering substrate for agents (no framework injects tool-latency spikes, memory drops, or compaction-induced fact loss today).

Plus it closes the SEA loop: the attribution output *is* the reward signal for an [Autogenesis](./36-autogenesis-self-evolving-agents.md)-style self-modification protocol.

## Why this wins — the evidence

### Gap 1: the harness IR is missing (from doc [66](./66-meta-harness-landscape.md))

Every framework ships its own surface. LangGraph's `StateGraph`, Archon's YAML DAG, DeerFlow's `Plan` artifact, RAGFlow's canvas, Claude Code's agents/skills/hooks, Cursor's rules — all describe *the same underlying computation* (a loop over tool calls with memory and guards) in mutually incompatible formats. No LLVM-equivalent exists. Nothing transpiles. This is the single highest-leverage missing layer in the 2026 landscape.

A Gnomon IR solves this in the only place it will actually get adopted: *as an export format*, not a replacement. Frameworks don't need to adopt it to be authored — they need to emit it for *evaluation and evolution*. That's a much lower bar.

### Gap 2: every tracing vendor ships "the trace failed" and stops there

From the [66](./66-meta-harness-landscape.md) synthesis and the [Vertex-Eval](../projects/vertex-eval/docs/architecture.md) MVP in this repo: LangSmith, Langfuse, Arize Phoenix, Helicone all store traces and let you ask an LLM-as-judge whether the trace succeeded. None of them attribute failures to **harness primitives** — "the compaction layer dropped the fact on turn 7", "the permission engine denied a legitimate action", "the plan-mode gate was bypassed because the tool wasn't classified destructive", "memory retrieval surfaced a stale fact that the planner trusted".

This is the highest-ROI direction because: **you can't improve a harness without knowing which primitive is failing.** HORIZON ([27](./27-horizon-long-horizon-degradation.md)) and Claw-Eval ([38](./38-claw-eval.md)) both push in this direction but neither adopted the primitive-level taxonomy the 2026 harness-patterns literature (docs [40–46](./40-harness-engineering-principles.md)) established.

### Gap 3: chaos engineering for agents doesn't exist

[Chaos-engineering-next-era](./53-chaos-engineering-next-era.md) argues that population-scale reliability depends on failure *decorrelation*. [Agents of Chaos](./49-agents-of-chaos-red-teaming.md) documents 11 failure modes but no framework today **injects** those failures to test recovery paths. DeerFlow has an HITL interrupt; nobody has a tool-latency-spike or memory-backend-drop injector. This is a natural complement to G1 and G2: once you have an IR, you have somewhere to inject.

### SEA side: the field converged but didn't ship anything production

The [56 landscape](./56-sea-landscape-2026.md), [60 top-github](./60-sea-top-github-repos.md), and the three April-2026 deep-dives ([57](./57-sea-arxiv-2604-15034.md), [58](./58-sea-arxiv-2507-21046.md), [59](./59-sea-arxiv-2508-07407.md)) all point to the same converged architecture:

- **Closed learning loop** (propose → assess → commit → rollback).
- **Multi-axis evolution** (what evolves: prompts, tools, memory, scaffold, architecture).
- **Reversible resource patches** (Autogenesis's core contribution, doc [36](./36-autogenesis-self-evolving-agents.md) + [57](./57-sea-arxiv-2604-15034.md)).
- **Skill library with verifier** ([Voyager](./19-voyager-skill-libraries.md), [Hermes](./55-hermes-agent-self-improving.md)).

But the field has **no shared reward signal**. Voyager uses env success, Hermes uses skill-reuse-rate, Autogenesis uses a gated evaluator, ACE uses judge feedback, Live-SWE uses scaffold-improvement-on-benchmark. Every SEA project is re-inventing the reward. **Gnomon's harness-primitive attribution is the missing shared reward.**

That's the unlock: the SEA community needs the G2 attribution layer whether they know it or not. Once attribution exists, the evolution loops stop being isolated experiments and start being deployable regressors against a common objective.

## What Gnomon is, concretely

A layered system. Six commitments, seven pillars, one contract.

### The contract — Harness IR (HIR)

A JSON schema + Protobuf shape describing a harness execution as the 12 primitives the research corpus has converged on:

| HIR primitive | Anchored in |
|---|---|
| `agent_loop` | [01](./01-agent-loop-architecture.md) |
| `subagent_delegation` | [02](./02-subagent-delegation.md) |
| `plan_mode` | [03](./03-plan-mode.md) |
| `skill_invocation` | [04](./04-skills.md), [19](./19-voyager-skill-libraries.md) |
| `hook` | [05](./05-hooks.md) |
| `permission_check` | [06](./06-permission-modes.md) |
| `tool_use` | ubiquitous |
| `compaction_event` | [08](./08-context-compaction.md) |
| `memory_read` / `memory_write` | [09](./09-memory-files.md) |
| `verifier_call` | [11](./11-verifier-evaluator-loops.md) |
| `todo_scratchpad_state` | [12](./12-todo-scratchpad-state.md) |
| `evolution_patch` | [36 Autogenesis](./36-autogenesis-self-evolving-agents.md), [57](./57-sea-arxiv-2604-15034.md) |

Each HIR event carries: `ts`, `run_id`, `primitive`, `inputs`, `outputs`, `latency_ms`, `cost_tokens`, `caller_frame`, `hash`. Traces become hash-linked DAGs.

**Adapters ship for:** Claude Code trace exports, Claude Agent SDK spans, LangGraph checkpointer events, DeerFlow LangGraph state transitions, Archon DAG node events, RAGFlow canvas steps, Cursor chat logs, OpenClaw session exports, Langfuse/LangSmith/Helicone ingestion. Adapters are **lossy in one direction only** — they always map framework concepts into HIR primitives; frameworks don't need to adopt HIR, they just need to be readable.

This is G1 solved in the form that will actually adopt: post-hoc conversion, not upfront buy-in.

### The attribution engine — Harness-Aware Failure Classifier (HAFC)

Given a HIR trace and a rubric result from [Vertex-Eval](../projects/vertex-eval/docs/architecture.md)-style evaluators or LLM-as-judge, HAFC outputs:

- **primitive of origin** (which of the 12 primitives the failure traces back to)
- **class** (one of: dropped-context, stale-memory, mis-permissioned, plan-bypass, unverified-claim, tool-misuse, skill-miss, subagent-handoff, compaction-loss, reward-hack, resource-exhaustion, prompt-injection)
- **causal quote** — the specific HIR event with its input/output
- **minimal repro** — the subtrace that would reproduce the failure
- **suggested patch class** — what kind of Autogenesis-shaped resource patch would address it (new skill, rewritten planner prompt, tightened permission rule, added verifier check, memory re-indexing, etc.)

This is G2 solved. Training data comes from the [LaStraj-for-pentest](./26-linuxarena-production-agent-safety.md) corpus + the [Agents-of-Chaos](./49-agents-of-chaos-red-teaming.md) taxonomy + labeled traces from the 10 in-tree MVP projects — we have labeled failure examples across coding, research, ops, voice, multi-agent, bio, security, personal, math, and eval.

HAFC reuses the cross-channel evidence discipline already shipping in Vertex-Eval: a failure is only attributed to a primitive when **two of three** of trace / audit / state-snapshot agree. Single-channel attributions are flagged as low-confidence.

### The chaos substrate — Stochastic Harness Perturbation (SHP)

An in-HIR fault injector. For each HIR primitive, SHP defines injectors:

- `tool_use` → delay spike, partial result, timeout, wrong-result-right-format
- `memory_read` → stale fact, missing fact, wrong-tenant fact
- `compaction_event` → drop-random-span, drop-recent-span, fabricate-plausible-span
- `permission_check` → denied-on-legitimate, allowed-on-destructive
- `plan_mode` → plan stale, plan ambiguous, plan contradicted by constraints
- `subagent_delegation` → handoff message corrupted, sub-result truncated, sub-agent diverged

SHP runs agents through the same tasks with injector probabilities — measuring which primitives the harness is *robust* to and which are the weak points. This is G3 solved and also feeds the attribution engine: labeled failure traces from SHP become HAFC training data.

### The evolution loop — Autogenesis Adapter + Gated Commit

On an attributed failure, Gnomon proposes a **resource patch** in Autogenesis shape ([36](./36-autogenesis-self-evolving-agents.md) + [57](./57-sea-arxiv-2604-15034.md)):

- `resource_uri` — the thing to modify (`skill://mentat/weekly-review@v3`, `prompt://orion/planner@v2`, `permission://aegis/db-write@v1`, `memory_index://helix/uniprot-cache@v1`).
- `patch` — the diff (unified-diff for text resources, JSON-patch for structured).
- `ancestry` — hash chain to the parent resource.
- `gate_policy` — {evaluator: rubric + HAFC on held-out traces, accept_threshold: 0.85 on replay Pass^3, rollback_window: 72h}.

Patches are **committed** if they pass the gate, **rolled back** if subsequent HAFC traces show regression. The ancestry hash chain means you can audit *which patch introduced which regression* — closing the reward-hacking gap that plagues free-running SEA systems.

### The reward — attribution as signal

Unlike Voyager/Hermes/ACE, Gnomon's reward for a patch is: **the attributed-failure-rate on replayed traces goes down AND the primitive-coverage (number of primitives that see evolution patches over time) goes up**. Two signals, not one — which resists the universal-hammer failure mode (one patch that looks like it fixes everything but actually just games the metric). This is the [mesa-optimization guard](../projects/quanta-proof/src/quanta_proof/gate.py) pattern from Quanta-Proof generalised.

### The exported artifacts — Cross-Harness Skill & Patch Bundle

Ship the same resource patches in [everything-claude-code](./62-everything-claude-code.md)-style cross-harness bundles: the same evolved skill, prompt, or hook runs in Claude Code, Cursor, Codex, and LangGraph. This gives Gnomon a distribution story the moment the attribution engine finds something useful.

## Architecture summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                                Gnomon                                │
│                                                                     │
│  [Framework adapters] ──► [HIR normaliser] ──► [HIR trace store]    │
│     (Claude Code / LangGraph / Archon /                             │
│      DeerFlow / RAGFlow / LobeHub / MCP)                            │
│                                   │                                 │
│                                   ▼                                 │
│                     [Harness-Aware Failure Classifier]              │
│                       (primitive · class · quote · repro)           │
│                                   │                                 │
│                    ┌──────────────┼──────────────┐                  │
│                    ▼              ▼              ▼                  │
│            [Evolution Loop]  [SHP Chaos]  [Dashboards]              │
│            (propose → assess     (inject      (primitive·           │
│             → commit → rollback   faults)     coverage·             │
│             in Autogenesis shape)             regression)           │
│                    │                                                │
│                    ▼                                                │
│            [Cross-Harness Patch Bundle]                             │
│            (everything-claude-code-style export)                    │
└─────────────────────────────────────────────────────────────────────┘
```

## Ten architectural commitments (block roster)

1. **HIR schema + adapter contract.** The 12-primitive shape; one lossless adapter per framework.
2. **HIR trace store.** Append-only, hash-chained, tenant-isolated; OpenTelemetry-compatible.
3. **Harness-Aware Failure Classifier.** Primitive attribution + cross-channel evidence + mesa-guard.
4. **Stochastic Harness Perturbation engine.** Injectors per primitive, deterministic seeds, replay-ready.
5. **Autogenesis-shaped resource patch protocol.** URI-versioned resources, ancestry chain, gate policy.
6. **Evolution loop orchestrator.** Propose / assess / commit / rollback; primitive-coverage tracking.
7. **Replay harness.** Re-runs a HIR trace against a candidate patch in a sandboxed executor, compares attribution deltas.
8. **Cross-harness patch bundler.** Emits Claude Code skills, Cursor rules, LangGraph nodes, Archon DAG patches from a single resource patch.
9. **Decorrelation dashboard.** Per [53](./53-chaos-engineering-next-era.md): pairwise failure correlation across agent instances (already shipping in [Vertex-Eval](../projects/vertex-eval/src/vertex_eval/sla.py)).
10. **Privacy & audit.** Per-tenant isolation, PII scrub, hash-chained audit — reuse patterns from [Cipher-Sec](../projects/cipher-sec/src/cipher_sec/audit.py) and [Aegis-Ops](../projects/aegis-ops/src/aegis_ops/audit.py).

## Why Gnomon will compound

Every other harness builder on the landscape ([66](./66-meta-harness-landscape.md)) competes on *how easy it is to author a harness*. Gnomon competes on a different axis: *how fast a harness gets better once it's running.* Framework-agnostic. Adopt-by-import. Leave your LangGraph / Claude Code / Archon / DeerFlow harness; Gnomon just watches and ships patches back.

This is why Gnomon wins the 2026 landscape even though Archon has better authoring, RAGFlow has better ingestion, DeerFlow has better planning, LobeHub has better memory, and everything-claude-code has better distribution. Those products compete; Gnomon consumes them all.

## What to build first — 10-week walking-skeleton plan

1. **Weeks 1–2.** HIR schema v0.1 + adapters for Claude Code trace exports and LangGraph checkpointer events. Ship as a Python package `prism-hir`.
2. **Weeks 3–4.** HIR trace store + one HAFC classifier (the `compaction-loss` class, because it's the most tractable with high impact). Use the 10 in-tree MVP projects' traces as labeled data.
3. **Weeks 5–6.** SHP with two injectors (`tool_use.latency_spike`, `memory_read.stale_fact`). Wire into the Claude Code adapter.
4. **Weeks 7–8.** Evolution loop v0: one resource type (`skill`), one gate (Pass^3 replay ≥ baseline + 2pp). Autogenesis-shaped patches committed by hash ancestry.
5. **Weeks 9–10.** Cross-harness patch bundler v0 (Claude Code SKILL.md + Cursor `.cursorrules` from one resource patch). Dashboard with primitive coverage + regression tracking.

This is 10 weeks to *walking skeleton*, not 10 weeks to *finished*. Finished is 9–12 months; walking skeleton is what unlocks the OSS flywheel.

## Why not something else

I considered five alternatives and rejected each:

- **Another harness builder (Archon-class).** Archon and DeerFlow already do this well; adding a 15th framework is not a breakthrough. The landscape is crowded with authoring tools and starved of evaluation tools.
- **Another SEA algorithm.** The algorithms have converged ([56 landscape](./56-sea-landscape-2026.md)); the bottleneck is the reward signal. An 11th skill-library implementation doesn't move the field. The attribution layer does.
- **Another RAG framework.** RAGFlow already has a canvas runtime and ships ingestion + inference on one engine. Adding RAG primitives to Gnomon is easy once HIR exists; building a competing RAG framework is not a breakthrough.
- **A chaos-only product.** SHP alone is useful but isolated; without HAFC and the evolution loop, it's a testing utility, not a compounding system. Gnomon needs all three legs.
- **A pure Meta-Harness IR.** Without attribution and chaos, an IR is a schema no one adopts. Gnomon bundles the IR with immediately-useful products (attribution, evolution, chaos) so adopting the IR *earns you* capability that the framework you're already on doesn't ship.

## The breakthrough claims (to prove)

These are the falsifiable claims Gnomon commits to:

1. **HIR coverage claim.** The 12-primitive HIR losslessly represents traces from ≥ 6 frameworks (Claude Code, LangGraph, DeerFlow, Archon, RAGFlow, OpenClaw). *Failure mode to prove false*: a framework whose native concept can't be mapped to any HIR primitive.
2. **Attribution recall claim.** HAFC recovers ≥ 80% of the failure-primitive labels that domain experts assign on a 200-trace held-out set from the in-tree MVP projects — matching [HORIZON's κ=0.84](./27-horizon-long-horizon-degradation.md) ceiling within ε.
3. **Evolution compounding claim.** Over 30 days of unattended running on an MVP workload, Gnomon-driven patches drive primitive-attributed failure rate down ≥ 30% without regressing task success. This is the [Hermes](./55-hermes-agent-self-improving.md) ≥ 30% productivity claim, reframed as failure-rate compounding.
4. **Cross-harness portability claim.** A resource patch that fixes a failure in one framework reduces the same primitive's failure rate in ≥ 2 other frameworks when exported via the bundler.
5. **Chaos-coverage claim.** After 10 SHP injectors are live, every committed patch reduces attribution volume under perturbation *measurably differently* from under nominal conditions — i.e., patches are actually making the harness robust, not just lucky.

If any of these falsify within the first 6 months, the corresponding subsystem gets the scope reduction, not the whole project.

## Honesty disclosures

- **Gnomon does not yet exist.** This is a design proposal, not a shipped system. All numbers in the claims above are design targets anchored in prior-art measurements in the corpus; none are observed.
- **HIR is opinionated.** The 12-primitive set is defensible given the corpus but is not proven to be minimal or complete. Version-bump policy is required from day one.
- **HAFC is an LLM-as-judge system.** It inherits the judge-drift risks ([21](./21-llm-as-judge-trajectory-eval.md)) and needs the human calibration loop [Vertex-Eval](../projects/vertex-eval/docs/architecture.md) already proposes.
- **Autogenesis-shaped resource patches introduce a new supply-chain surface.** A malicious patch could poison a skill library. Cipher-Sec's scope-authorization primitive and Vertex-Eval's cross-channel evidence discipline are direct mitigations, but deploying Gnomon in production requires a pinned provenance chain.

## Closing

If you can only ship one thing in 2026 H1: ship Gnomon. The harness-builder landscape doesn't need another framework. It needs the attribution layer and the closed evolution loop that turns shipped harnesses into compounding ones. Every paper, every repo, and every gap in this corpus points to the same conclusion.

Build it.

## References and cross-links

- SEA: [56](./56-sea-landscape-2026.md), [57](./57-sea-arxiv-2604-15034.md), [58](./58-sea-arxiv-2507-21046.md), [59](./59-sea-arxiv-2508-07407.md), [60](./60-sea-top-github-repos.md), [36](./36-autogenesis-self-evolving-agents.md), [19](./19-voyager-skill-libraries.md), [14](./14-reflexion.md), [45](./45-hyperagents-self-modification.md), [55](./55-hermes-agent-self-improving.md).
- Harness builders: [61 Archon](./61-archon-harness-builder.md), [62 everything-claude-code](./62-everything-claude-code.md), [63 RAGFlow](./63-ragflow-agent-patterns.md), [64 LobeHub](./64-lobehub-ai-framework.md), [65 DeerFlow](./65-deer-flow-bytedance.md), [66 meta-harness landscape](./66-meta-harness-landscape.md), [42 Deep Agents](./42-langchain-deep-agents.md), [52 OpenClaw](./52-dive-into-open-claw.md), [54 SemaClaw](./54-semaclaw-general-purpose-agent.md).
- Primitives (the HIR anchors): [01 agent-loop](./01-agent-loop-architecture.md), [02 subagents](./02-subagent-delegation.md), [03 plan-mode](./03-plan-mode.md), [04 skills](./04-skills.md), [05 hooks](./05-hooks.md), [06 permissions](./06-permission-modes.md), [08 compaction](./08-context-compaction.md), [09 memory](./09-memory-files.md), [11 verifier](./11-verifier-evaluator-loops.md), [12 todo-scratchpad](./12-todo-scratchpad-state.md).
- Evaluation / chaos: [21 LLM-judge](./21-llm-as-judge-trajectory-eval.md), [24 observability](./24-observability-tracing.md), [27 HORIZON](./27-horizon-long-horizon-degradation.md), [38 Claw-Eval](./38-claw-eval.md), [49 Agents-of-Chaos](./49-agents-of-chaos-red-teaming.md), [53 chaos-engineering-next-era](./53-chaos-engineering-next-era.md).
- In-tree precedents: [Vertex-Eval](../projects/vertex-eval/docs/architecture.md) (ships the evaluator scaffold), [Cipher-Sec](../projects/cipher-sec/docs/architecture.md) (audit + scope), [Aegis-Ops](../projects/aegis-ops/docs/architecture.md) (policy + audit), [Quanta-Proof](../projects/quanta-proof/docs/architecture.md) (mesa-guard pattern), [Mentat-Learn](../projects/mentat-learn/docs/architecture.md) (skill extractor + 4-layer memory).
