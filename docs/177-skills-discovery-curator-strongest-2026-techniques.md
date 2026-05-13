# 177 — Skills Discovery & Skill Curation: The Strongest 2026 Techniques

**Scope.** Cross-paper + cross-repo synthesis of the strongest current techniques for **skills discovery** (how new skills enter the library) and **skill curation** (how skills earn the right to be loaded into a live agent). Builds on the four-corners synthesis at [docs/171](171-skill-self-evolution-2026-synthesis.md) and the post-171 wave covered in [docs/173–176](173-offline-sim-skill-discovery.md).

**One-paragraph thesis.** The 2025–2026 wave converges on **five structural commitments** the entire literature now treats as default: (1) *skills are files, not weights*; (2) *evolution beats one-shot generation*; (3) *the curator is non-optional* after the 26.1% vulnerability finding; (4) *metadata is what activates, the body is what executes*; (5) *online iteration with execution feedback beats offline LLM-judge*. Layered on top of these are **five orthogonal design-space axes** (signal source, artifact form, parameter access, curation discipline, acquisition mode). Across these axes the **strongest 2026 stack** is opinionated: folder-shaped skill artefacts (EvoSkill / CoEvoSkills shape with `SKILL.md` frontmatter as the activation surface), hybrid offline-simulator + ground-truth feedback (the 173/168 pair), automated vulnerability scanner gating every non-author contribution (the 175 four-tier model), failure-driven evolution + EXIF-style interactive exploration during idle cycles (the 174 dialogue), and SkillRL co-evolution only when weights are open. Polaris and Lyra both already inhabit most of this stack — what's missing is the *retrieval surface* once catalogs cross the 50-skill threshold, which is where [docs/179](179-skill-retrieval-routing-and-activation.md) takes over.

---

## §1 — Convergence findings (what's now load-bearing across the literature)

| Commitment | Evidence | Stake |
|---|---|---|
| **Skills are files** | AutoSkill / EvoSkill / CoEvoSkills / Ctx2Skill all use markdown-with-frontmatter; Anthropic's `anthropics/skills` (130k★) ships the same shape | Audit chains, version control, transferability between harnesses |
| **Evolution beats one-shot** | EvoSkill ([168](168-evoskill-coding-agent-skill-discovery.md)) Pareto search; CoEvoSkills ([169](169-coevoskills-co-evolutionary-verification.md)) co-evolution; EXIF ([174](174-autonomous-skill-exploration-iterative-feedback.md)) iterative loop; SakanaAI Scientist tree-search | Headline +7-44pp gains over one-shot LLM-as-judge baselines |
| **The curator is non-optional** | [175](175-agent-skills-ecosystem-and-security.md): 26.1% community-skills vulnerable; Anthropic ships scanned-only catalog; CrewAI marketplace gates; ToolBench filters | Attack surface compounds with catalog size; manual review doesn't scale |
| **Metadata activates, body executes** | Anthropic's three-tier disclosure; LlamaIndex `ObjectIndex`; MCP `tools/list`; ToolBench instruction-API index | Allows thousands-of-skills catalogs without context-budget collapse |
| **Online iteration > offline judge** | EXIF same-model self-evolution; Voyager iterative-prompting; Cradle live-game adaptation; Eureka in-context evolutionary RL | LLM-as-judge has 16-17% recall vs. retrieval-grounded validation's 68% (CiteGuard-style; same finding shape) |

These are the five structural commitments any 2026-grade skill stack must respect. Harnesses that violate any of the five — e.g. ship a marketplace without a scanner, or ship a one-shot extractor without an evolution loop — accumulate technical debt that compounds as the catalog grows.

---

## §2 — The full design-space matrix (extending docs/171)

`docs/171` plotted four corners along three axes (signal source, artifact form, parameter access). The post-171 wave extends this to a **five-axis matrix** with two added axes (curation discipline, acquisition mode) and several new points on the existing axes.

### Axis 1 — Feedback signal source

```
ground-truth ← offline-sim ← exploration+feedback ← surrogate ← adversarial ← LLM judge ← no signal
oracle         ([173])      ([174] / EXIF)         verifier    self-play     (P_judge)   only
                                                   ([169])     ([154])       ([167])
```

Six points, each with different cost / fidelity trade-offs. **Hybrid stacks dominate the 2026 frontier**: offline-sim as cheap pre-filter → ground-truth as the eval gate → LLM-judge only as a last-resort fallback.

### Axis 2 — Skill artifact form

```
single SKILL.md ← folder + scripts ← multi-file package ← JSON tiered ← model adapter
                  (EvoSkill, Anthropic skills)            (SkillRL)     (Toolformer / ToolkenGPT)
```

Folder-with-scripts is the **dominant production shape**. The `anthropics/skills` 130k★ adoption settles the format question. Single-file SKILL.md is fine for human-edited preferences; multi-file package wins for compound skills with helper code.

### Axis 3 — Parameter access

```
frozen weights ← LoRA adapter ← partial fine-tune ← full RL training
(AutoSkill / EvoSkill /         (Search-R1)       (SkillRL with policy)
 CoEvoSkills / Ctx2Skill /
 Anthropic / smolagents)
```

Most of the 2026 wave is *frozen* — they assume closed-source frontier models. SkillRL ([170](170-skillrl-recursive-skill-augmented-rl.md)) and Search-R1 dissent: with weight access, train.

### Axis 4 — Curation discipline (NEW)

```
no curation ← human review ← automated scanner ← four-tier framework ← cryptographic pinning
                (Polaris BL-PROMOTE-SKILL,    ([175])                  ([175] T-Pinned)
                 Lyra BL-LYRA-SKILL-PROMOTE)
```

After the 26.1% finding, **automated scanner is the floor**, not the ceiling. The four-tier framework is the production shape.

### Axis 5 — Acquisition mode (NEW)

```
hand-authored ← auto-extracted ← failure-driven ← document-derived ← marketplace-fetched
                ([167] AutoSkill, ([168] EvoSkill, ([154] Ctx2Skill,    (Anthropic Skills,
                 Polaris auto_creator) [173], [174])   AutoSkill4Doc)    smolagents Hub,
                                                                        MCP servers)
```

Marketplace-fetched is the *most operationally important* mode for community ecosystems; it's also the riskiest per axis-4. The two axes interact: marketplace skills must default to T-Untrusted at axis-4 until automated scanning passes.

### The matrix as a single table

|  | **No signal** | **LLM judge** | **Surrogate** | **Exploration+FB** | **Offline sim** | **Ground truth** | **RL reward** |
|---|---|---|---|---|---|---|---|
| **Single .md** | AutoSkill / Ctx2Skill | Anthropic Skills | — | — | — | — | — |
| **Folder+scripts** | — | smolagents Hub | — | EXIF (loose) | This paper ([173]) | EvoSkill ([168]) | — |
| **Multi-file pkg** | — | — | CoEvoSkills ([169]) | — | — | — | — |
| **JSON tiered** | — | — | — | — | — | — | SkillRL ([170]) |
| **Model adapter** | — | — | — | — | — | — | Toolformer / ToolkenGPT |

Empty cells are still open. The most interesting open cell — *ground-truth × multi-file package* — is where Polaris's coding-agent skill discovery should land in v2.5.

---

## §3 — The strongest pattern as of May 2026

Opinionated stack recommendation, layered:

### Skill artifact: **folder + scripts**, SKILL.md as the activation surface

Match `anthropics/skills` frontmatter exactly so skills round-trip across harnesses. Required: `name`, `description`. Recommended: `when_to_use`, `argument-hint`, `disable-model-invocation`, `user-invocable`, `allowed-tools`, `paths`. Body is markdown for instructions; supplementary files (scripts, templates, reference docs) referenced by relative path.

### Discovery: **failure-driven evolution as the spine**, exploration as the idle-time complement

EvoSkill's failure-driven Pareto search is the production primitive — every rejection event becomes a candidate skill, the gate runs as soon as new candidates accumulate. Add EXIF-style same-model exploration for the long tail that failure traces don't surface. Run exploration in idle cycles (Polaris's heartbeat scheduler is the natural trigger) so it doesn't block real work.

### Feedback: **hybrid offline-sim + ground-truth**, with LLM-as-judge as a last-resort fallback

For domains with stable APIs (most software automation), use offline-simulator validation ([173](173-offline-sim-skill-discovery.md)) as a cheap pre-filter. Promote across the offline simulator's positive verdicts to the ground-truth gate. Skip ground-truth gate only when neither a benchmark nor a simulator exists, in which case fall back to EXIF-style execution feedback ([174](174-autonomous-skill-exploration-iterative-feedback.md)). Reserve LLM-as-judge for skills where execution is impossible (e.g. open-ended creative dialogue preferences).

### Curation: **automated scanner gating any non-author skill**, four-tier trust model

Implement the [175](175-agent-skills-ecosystem-and-security.md) four-tier framework (T-Untrusted → T-Scanned → T-Reviewed → T-Pinned) over the existing `BL-PROMOTE-SKILL` bright-line. The scanner's minimum implementation: prompt-injection detection (regex + entropy + hidden-Unicode), description-vs-`allowed-tools` consistency check, privilege-escalation graph analysis, supply-chain (remote-fetch-at-activation) detection. Content-pin every active skill via SHA-256.

### Activation: **progressive disclosure** (descriptions in system prompt, body on demand)

Match Anthropic's three-tier disclosure: name + description in the system prompt at startup; full SKILL.md when relevant; supplementary files only when explicitly referenced. Use a per-skill character cap (1,536 chars per Anthropic's default; tune per context window). When the catalog grows past ~50 skills, layer in embedding retrieval — see [179](179-skill-retrieval-routing-and-activation.md) for the full router story.

### Training (open-weights only): **SkillRL co-evolution**

If your harness has open weights (Llama / Qwen / Mistral / DeepSeek class), add SkillRL's policy ↔ SkillBank co-evolution. The contract scaffolds in `polaris-research/rl/` are the integration point. Most production harnesses skip this tier and stay frozen-weights — that's fine; the gain from the prior layers is already substantial.

---

## §4 — Lyra adoption recipe

Lyra already implements most of the stack. The gaps are:

```text
packages/lyra-core/src/lyra_core/skills/
  offline_validator.py     # NEW [173] — Sandbox-typed validator;
                           # composes with the existing terminal/ backends.
  exploration_loop.py      # NEW [174] — Alice/Bob loop on top of the
                           # subagent orchestrator + arena/elo.
  api_graph.py             # NEW [173 §2.1] — typed graph for the tool
                           # catalog; drives failure-rich task generation.

packages/lyra-skills/src/lyra_skills/
  curator/
    vulnerability_scanner.py   # NEW [175] — four vulnerability classes.
    trust_tiers.py             # NEW [175] — T-Untrusted → T-Pinned.
    content_pinning.py         # NEW [175] — sha256-anchor every skill.
    description_drift.py       # NEW [175] — flag description ↔
                               # allowed-tools mismatch.
```

Existing primitives Lyra already has and should be reused:

- `lyra-skills/extractor.py` — already auto-extracts skills from traces.
- `lyra-skills/curator.py` — already runs review pipeline; extend with the four-tier framework.
- `lyra-skills/router.py` — already routes by description; extend with embedding retrieval at scale.
- `lyra-core/subagent/orchestrator.py` + `lyra-core/arena/elo.py` — substrate for Alice/Bob.
- `lyra-core/terminal/` — sandbox abstraction for offline simulators.

Minimum viable v3.8 lift: ~6 weeks across the four NEW files + scanner buildout.

---

## §5 — Polaris adoption recipe (building on v2.2 / v2.3 / v2.4)

Polaris's existing v2.2-v2.4 plans cover most of this stack. The gaps relative to *this synthesis*:

### v2.2-v2.4 already delivers

- ✅ Three-gate AND with attribution + triangulation ([P29 in V2.2](../projects/polaris/POLARIS_V2_2_DEEP_RESEARCH_PLAN.md))
- ✅ Tournament + Elo for hypotheses ([P32 in V2.2](../projects/polaris/POLARIS_V2_2_DEEP_RESEARCH_PLAN.md))
- ✅ Tree-search experiment manager ([P33 in V2.3](../projects/polaris/POLARIS_V2_3_DEEP_RESEARCH_PLAN.md))
- ✅ TTD-DR draft-as-state writer ([P35 in V2.3](../projects/polaris/POLARIS_V2_3_DEEP_RESEARCH_PLAN.md))
- ✅ Search-R1 / GRPO scaffold ([P39 in V2.4](../projects/polaris/POLARIS_V2_4_DEEP_RESEARCH_PLAN.md))
- ✅ AgentPRM/HiPRAG process-reward rubrics ([P40 in V2.4](../projects/polaris/POLARIS_V2_4_DEEP_RESEARCH_PLAN.md))

### Gaps to close in v2.5 (proposed)

```text
packages/polaris-skills/src/polaris_skills/research/
  offline_sim_gate.py       # NEW [173] — pre-flight offline-simulator
                            # before three-gate AND.
  api_graph.py              # NEW [173 §2.1] — typed API-relationship
                            # graph for failure-rich task generation.
  exploration_loop.py       # NEW [174] — Alice/Bob, fires from the
                            # heartbeat scheduler's idle cycles.

packages/polaris-skills/src/polaris_skills/curator/
  vulnerability_scanner.py  # NEW [175] — four vulnerability classes.
  trust_tiers.py            # NEW [175] — four-tier framework.
  content_pinning.py        # NEW [175] — sha256-pin every active skill.
  description_drift.py      # NEW [175] — `allowed-tools` consistency.
```

Bright-line additions:

```
BL-OFFLINE-SIM-DRIFT       [173] simulator-vs-production divergence
                           triggers re-validation.
BL-EXPLORATION-COST        [174] Alice/Bob respects program cost
                           envelope.
BL-SKILL-MARKET-FETCH      [175] marketplace skills must pass scanner
                           before active-library entry.
BL-SKILL-DRIFT             [175] pinned-skill body drift drops to
                           T-Untrusted.
BL-SKILL-DESCRIPTION-DRIFT [175] description-vs-allowed-tools
                           mismatch blocks promotion.
```

Estimated effort: ~5 weeks across the seven NEW files + bright-line wiring + tests.

---

## §6 — What's NOT in this synthesis (and why)

Two patterns that appear in the literature but are *not* recommended for production harnesses today:

1. **Pure LLM-as-judge curation.** AutoSkill / Ctx2Skill ship this for dialogue / document corners; it's appropriate there because no execution feedback exists. Outside those corners, prefer execution-grounded signals — the [171](171-skill-self-evolution-2026-synthesis.md) data and the [174](174-autonomous-skill-exploration-iterative-feedback.md) EXIF results both show LLM-judge alone is too noisy as the *primary* gate.

2. **Reward-model-only RL.** SkillRL ([170](170-skillrl-recursive-skill-augmented-rl.md)) is included in the recommended stack only when weights are open. For frozen-weights harnesses, prefer process-reward rubrics ([Polaris v2.4 P40](../projects/polaris/POLARIS_V2_4_DEEP_RESEARCH_PLAN.md)) used as eval-gate criteria — same scoring surface, no training pipeline.

---

## §7 — Reading list

In recommended reading order:

### Surveys and synthesis chapters (read these first)

1. [171 — Skill Self-Evolution Synthesis](171-skill-self-evolution-2026-synthesis.md) — predecessor synthesis; four corners.
2. [docs/172](172-polaris-2026-deep-research-roadmap.md) — research-agent landscape (different cross-section).
3. [157 — May 2026 Memory + Skills Synthesis](157-may-2026-synthesis-memory-and-skills.md) — memory pairing.
4. [166 — Research-Agent Frameworks Synthesis](166-research-agent-frameworks-synthesis.md) — frameworks landscape.

### The post-171 wave (gap-fill)

5. [173 — Offline-Sim Skill Discovery](173-offline-sim-skill-discovery.md) — Xu et al., arXiv:[2504.20406](https://arxiv.org/abs/2504.20406).
6. [174 — EXIF Autonomous Exploration](174-autonomous-skill-exploration-iterative-feedback.md) — Yang et al., arXiv:[2506.04287](https://arxiv.org/abs/2506.04287).
7. [175 — Agent Skills Ecosystem & Security](175-agent-skills-ecosystem-and-security.md) — Xu & Yan, arXiv:[2602.12430](https://arxiv.org/abs/2602.12430).
8. [176 — OSS Skill Discovery + Curator Landscape](176-skill-discovery-curator-oss-landscape-may-2026.md) — repos.
9. [178 — Online Skill Discovery & Curation On-the-Go](178-online-skill-discovery-and-curation-on-the-go.md) — runtime/live patterns.
10. [179 — Skill Retrieval, Routing, Activation](179-skill-retrieval-routing-and-activation.md) — the activation surface.

### The 2025–2026 single-paper canon

11. [167 — AutoSkill](167-autoskill-experience-driven-lifelong-learning.md) — arXiv:2603.01145.
12. [168 — EvoSkill](168-evoskill-coding-agent-skill-discovery.md) — arXiv:[2603.02766](https://arxiv.org/abs/2603.02766).
13. [169 — CoEvoSkills](169-coevoskills-co-evolutionary-verification.md) — arXiv:[2604.01687](https://arxiv.org/abs/2604.01687).
14. [170 — SkillRL](170-skillrl-recursive-skill-augmented-rl.md) — arXiv:2602.08234.
15. [154 — Ctx2Skill](154-ctx2skill-self-evolving-context-skills.md).
16. [156 — HeavySkill](156-heavyskill-parallel-reasoning-deliberation.md).

### Foundational ancestors

17. **Voyager** — Wang et al., ICLR 2024 — arXiv:2305.16291.
18. **HMASD** (NeurIPS 2023 c276c3303c…) — Mingyu Yang et al., *Hierarchical Multi-Agent Skill Discovery*. RL skill discovery in MARL; the formal-RL ancestor of the LLM-skill wave.
19. **Toolformer / ToolkenGPT** — Hao et al., NeurIPS 2023 — arXiv:[2305.11554](https://arxiv.org/abs/2305.11554). Tool-as-token; embedding-driven tool selection.
20. **Gorilla** — Patil et al., 2023 — arXiv:[2305.15334](https://arxiv.org/abs/2305.15334). LLaMA + retrieval over 1600+ APIs.
21. **ToolBench** — Qin et al. — 16,464 REST APIs / 3,451 tools; the scale baseline.

### Adjacent papers worth knowing

22. **SkillRouter** — Zheng et al., arXiv:[2603.22455](https://arxiv.org/abs/2603.22455). 1.2B retrieve-and-rerank for 80k-skill registries; 74.0% Hit@1. The routing problem at scale; full treatment in [179](179-skill-retrieval-routing-and-activation.md).
23. **Corpus2Skill** — Sun, Wei, Hsieh, arXiv:[2604.14572](https://arxiv.org/abs/2604.14572). "Don't retrieve, navigate"; hierarchical skill-tree as anti-retrieval.
24. **A-Mem** — arXiv:2502.12110. Agentic memory; complements skill-as-procedure with skill-as-experience.

### High-impact OSS

25. [`anthropics/skills`](https://github.com/anthropics/skills) — 130k★, the canonical SKILL.md ABI.
26. [`MineDojo/Voyager`](https://github.com/MineDojo/Voyager) — 6.9k★, MIT.
27. [`huggingface/smolagents`](https://github.com/huggingface/smolagents) — 27.1k★, Apache-2.0.
28. [`stanfordnlp/dspy`](https://github.com/stanfordnlp/dspy) — 34.3k★, MIT.
29. [`run-llama/llama_index`](https://github.com/run-llama/llama_index) — 49.2k★, MIT (`ObjectIndex`, `ToolRetrieverRouterQueryEngine`).
30. [`OpenBMB/ToolBench`](https://github.com/OpenBMB/ToolBench) — 5.6k★.
31. [`eureka-research/Eureka`](https://github.com/eureka-research/Eureka) — 3.2k★.
32. [`BAAI-Agents/Cradle`](https://github.com/BAAI-Agents/Cradle) — 2.5k★.

---

## §8 — One-paragraph summary

Skill discovery and skill curation, in 2026, converge on a single opinionated stack: **folder-shaped artifacts**, **failure-driven evolution as the spine** with **EXIF-style exploration as the long-tail complement**, **hybrid offline-simulator + ground-truth feedback**, **automated four-tier curator gating any non-author contribution**, **progressive-disclosure activation** with embedding rerank past 50 skills, and (open weights only) SkillRL co-evolution. Polaris and Lyra are both substantially aligned with this stack today; the closing gaps are the offline-simulator pre-flight, the EXIF exploration loop in idle cycles, the four-tier vulnerability scanner, and the embedding-rerank router for catalogs past ~50 skills. Five paper-side references and three OSS references between them carry 90% of the technical weight; the rest of the literature is variation on the same five structural commitments.
