# 295 — Corpus Reading Guide 2026: paths through 50+ deep-dives by reader goal

**Anchors.** All 250-294 deep-dives. Earlier corpus (216-237 scaling laws + 238-243 Karpathy IICYMI). Companion: README.md.

**One-line definition.** A **reading guide** for the 50+ deep-dive corpus — five reader-goal-aligned **reading paths** ("I want to ship a personal agent" / "I'm picking a runtime" / "I'm operationalizing for production" / "I'm hardening for security + compliance" / "I'm starting a new project") each pointing through a curated 8-15 file sequence — making the ~10,000-page corpus traversable rather than overwhelming, with **start-here entry points** per goal and **synthesis chapters** as the load-bearing reads.

## Why a reading guide matters

The 250-294 corpus is 45 deep-dives totaling ~10,000 pages of dense engineering material; reading it linearly is impractical. The corpus is **goal-organized**, not topic-organized — different readers need different paths. This guide gives five paths.

## Path 1: "I want to ship a personal agent" (Lyra-class)

**Entry:** [276-local-first-privacy-first-agents](276-local-first-privacy-first-agents.md) — start here for the Lyra-class deployment shape.

Sequence (8 files):

1. **[276-local-first-privacy-first-agents](276-local-first-privacy-first-agents.md)** — overview of the deployment shape.
2. **[280-lyra-seven-layer-stack-apply-plan](280-lyra-seven-layer-stack-apply-plan.md)** — concrete per-layer plan.
3. **[Lyra Block 07 — three-tier memory](../projects/lyra/docs/blocks/07-memory-three-tier.md)** — memory architecture (foundational reference).
4. **[233-memory-scaling-for-agents](233-memory-scaling-for-agents.md)** — memory scaling theory.
5. **[252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md)** — fire-from-anywhere triggers.
6. **[253-tailscale-nats-mesh-for-distributed-agents](253-tailscale-nats-mesh-for-distributed-agents.md)** — multi-device deployment.
7. **[277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md)** — UX patterns.
8. **[278-agent-unit-economics-2026](278-agent-unit-economics-2026.md)** — cost model.

**Total reading time:** ~6 hours. **Output:** clear picture of how to deploy a Lyra-class agent on consumer hardware with privacy-first defaults at $0 marginal cost.

## Path 2: "I'm picking a runtime"

**Entry:** [263-production-agent-runtime-synthesis-2026](263-production-agent-runtime-synthesis-2026.md) — the decision matrix.

Sequence (7 files):

1. **[263-production-agent-runtime-synthesis-2026](263-production-agent-runtime-synthesis-2026.md)** — six-runtime decision matrix.
2. Pick the most-relevant runtime deep-dive:
   - **[259-langgraph-deep-dive](259-langgraph-deep-dive.md)** — production state-machine
   - **[260-openai-agents-sdk-deep-dive](260-openai-agents-sdk-deep-dive.md)** — handoff workflows
   - **[261-autogen-v04-deep-dive](261-autogen-v04-deep-dive.md)** — event-driven actors
   - **[262-google-adk-deep-dive](262-google-adk-deep-dive.md)** — multi-language enterprise
   - **[164-crewai-multi-agent-framework](164-crewai-multi-agent-framework.md)** — declarative crews
   - **[250-anthropic-agent-teams](250-anthropic-agent-teams.md)** — Claude Code teams
3. **[251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md)** — multi-agent failure modes (MAST).
4. **[258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md)** — protocol layer that lets runtimes interop.

**Total reading time:** ~5 hours. **Output:** confident decision on runtime + protocol + cross-runtime interop.

## Path 3: "I'm operationalizing for production"

**Entry:** [268-agent-operations-synthesis-2026](268-agent-operations-synthesis-2026.md) — the four legs.

Sequence (10 files):

1. **[268-agent-operations-synthesis-2026](268-agent-operations-synthesis-2026.md)** — overview.
2. **[264-agent-observability-stack-2026](264-agent-observability-stack-2026.md)** — observability.
3. **[265-agent-evaluation-2026](265-agent-evaluation-2026.md)** — evaluation.
4. **[266-agent-durability-and-idempotency](266-agent-durability-and-idempotency.md)** — durability.
5. **[267-agent-sre](267-agent-sre.md)** — SRE.
6. **[225-agent-era-scaling-synthesis](225-agent-era-scaling-synthesis.md)** — capability axes.
7. **[278-agent-unit-economics-2026](278-agent-unit-economics-2026.md)** — cost.
8. **[274-prompt-and-context-engineering-2026](274-prompt-and-context-engineering-2026.md)** — context discipline.
9. **[275-agent-finetuning-2026](275-agent-finetuning-2026.md)** — finetuning lever.
10. **[235-inference-compression-scaling](235-inference-compression-scaling.md)** — deployment economics.

**Total reading time:** ~10 hours. **Output:** complete operations toolkit for production-grade deployment.

## Path 4: "I'm hardening for security + compliance"

**Entry:** [273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md) — the unified security stack.

Sequence (8 files):

1. **[273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md)** — overview + 30-item checklist.
2. **[269-prompt-injection-2026](269-prompt-injection-2026.md)** — prompt injection.
3. **[270-agent-supply-chain-security](270-agent-supply-chain-security.md)** — supply chain.
4. **[271-agent-isolation-patterns](271-agent-isolation-patterns.md)** — isolation.
5. **[272-agent-compliance-and-audit](272-agent-compliance-and-audit.md)** — regulatory.
6. **[267-agent-sre](267-agent-sre.md)** — incident response.
7. **[02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md)** — verifier discipline.
8. **[233-memory-scaling-for-agents](233-memory-scaling-for-agents.md)** — memory poisoning context.

**Total reading time:** ~8 hours. **Output:** complete security model with 30-item production checklist.

## Path 5: "I'm starting a new project"

**Entry:** [285-harness-core-porting-plan](285-harness-core-porting-plan.md) — the shared library skeleton.

Sequence (12 files):

1. **[285-harness-core-porting-plan](285-harness-core-porting-plan.md)** — what to import.
2. **[211-cross-project-power-up-plan-with-tradeoffs](211-cross-project-power-up-plan-with-tradeoffs.md)** — strategic frame.
3. **[225-agent-era-scaling-synthesis](225-agent-era-scaling-synthesis.md)** — capability axes.
4. **[258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md)** — protocol stack.
5. **[263-production-agent-runtime-synthesis-2026](263-production-agent-runtime-synthesis-2026.md)** — runtime selection.
6. **[268-agent-operations-synthesis-2026](268-agent-operations-synthesis-2026.md)** — ops.
7. **[273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md)** — security.
8. **[279-polaris-seven-layer-stack-apply-plan](279-polaris-seven-layer-stack-apply-plan.md)** — example apply plan.
9. **[280-lyra-seven-layer-stack-apply-plan](280-lyra-seven-layer-stack-apply-plan.md)** — local-first example.
10. **[282-argus-seven-layer-stack-apply-plan](282-argus-seven-layer-stack-apply-plan.md)** — provider-role example.
11. **[252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md)** — invocation surface.
12. **[296-corpus-consolidation-review-2026](296-corpus-consolidation-review-2026.md)** — wrap.

**Total reading time:** ~10 hours. **Output:** end-to-end design for a new project leveraging the entire shared infrastructure.

## Reading by topic (alternative organization)

| Topic | Files |
|---|---|
| Capability scaling | 216-217, 222-225, 232-237 |
| Karpathy Software 3.0 | 238-243 |
| Agent Teams + distributed | 250-253 |
| Protocol stack | 254-258 |
| Runtime selection | 259-263 |
| Operations | 264-268 |
| Security | 269-273 |
| Underserved topics | 274-278 |
| Per-project plans | 203, 207-211, 218-221, 279-294 |
| Code skeleton | 285 + projects/harness_core/ |

## Synthesis chapters (read these first if anything)

Synthesis chapters cross-reference 15+ files and give the unified view:

- **[225-agent-era-scaling-synthesis](225-agent-era-scaling-synthesis.md)** — four capability axes
- **[251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md)** — multi-agent failure modes + consensus architecture
- **[258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md)** — six-layer protocol stack
- **[263-production-agent-runtime-synthesis-2026](263-production-agent-runtime-synthesis-2026.md)** — runtime decision matrix
- **[268-agent-operations-synthesis-2026](268-agent-operations-synthesis-2026.md)** — four ops legs
- **[273-agent-security-synthesis-2026](273-agent-security-synthesis-2026.md)** — security checklist
- **[296-corpus-consolidation-review-2026](296-corpus-consolidation-review-2026.md)** — meta-review

Reading any synthesis chapter alone gives you ~80% of the corresponding theme's value.

## Corpus stats (May 2026)

- **Total files:** 296+ in `harness-engineering/docs/`
- **Pages:** ~10,000 (200+ files × ~50 pages each, with synthesis chapters longer)
- **Cross-references:** ~15,000 inter-file links
- **Time to read everything:** ~80 hours (10 working days)
- **Time to read your reader-path:** 5-10 hours (1-2 working days)

## One-line takeaway

**The 50+ deep-dive corpus on production agent engineering is too large to read linearly; pick a reader path (personal agent / runtime selection / operations / security / new project) and follow the curated 8-15 file sequence; read synthesis chapters first if anything (225, 251, 258, 263, 268, 273, 296) — each gives ~80% of its theme's value alone; the corpus is **goal-organized for traversal**, not topic-organized for completion.**
