# 178 — Online Skill Discovery & Curation On-the-Go (May 2026)

**Scope.** The *runtime / live / streaming* surface of skill discovery and curation — discovery that happens *while the agent is operating*, not in an offline batch. Complement to [docs/173](173-offline-sim-skill-discovery.md) (offline simulator) and [docs/176](176-skill-discovery-curator-oss-landscape-may-2026.md) (static catalogs).

**One-paragraph thesis.** Online (on-the-go) skill discovery sits orthogonal to the offline / batch axis: the agent extracts, validates, and promotes skills *during* live operation. Three load-bearing patterns dominate the 2026 wave: (1) **live execution feedback** — Voyager-style iterative-prompting, EXIF-style Alice/Bob exploration, Cradle-style atomic-then-composite skill registration during play; (2) **runtime marketplace hot-load** — Anthropic's progressive disclosure + Skills API, smolagents Hub-pull, MCP `tools/list_changed` notifications; (3) **in-session dialogue extraction** — AutoSkill's `P_ext` pipeline running over each user turn. Online discovery is governed by three constraints offline discovery doesn't face: **latency-bounded extraction** (the loop must complete inside one tick), **drift detection** (a skill that worked yesterday may fail today), and **mid-session governance** (curation gates fire while the agent is still working, not during idle review). The 2026 best-practice stack runs offline batch *and* online streaming as complementary loops — offline for high-stakes promotion, online for the long-tail discoveries the batch loop will never see.

---

## §1 — Why online discovery is its own problem

`docs/171` and `docs/177` plot the design space along *signal source* and *artifact form*. A separate axis — **operational mode** — distinguishes:

```
offline batch ←——————————————————→ online runtime ←——→ hybrid
(EvoSkill Pareto search,             (Voyager,           (most production
 CoEvoSkills surrogate,               Cradle,             stacks: idle-cycle
 Polaris auto_creator on              EXIF,               batch + live
 idle housekeeping)                   AutoSkill,          extraction)
                                      SkillRL training)
```

Three constraints make this axis non-trivial:

1. **Latency budget.** An offline batch can spend hours searching the Pareto frontier; an online extractor must finish before the user's next turn. Polaris's heartbeat scheduler (v2.2 P28) honours this by separating Reflection (cheap, frequent) from Consolidation (expensive, infrequent).
2. **Drift.** A skill that passed validation a week ago may fail today — APIs change, tool surfaces evolve, vendor models update. Online curation must include a *drift detector* that re-validates on a cadence; offline curators can rely on the validation timestamp.
3. **Mid-session governance.** Curation gates that block-and-ask are fine during idle review; in a live loop they would freeze the agent. Online curation defaults to *speculatively promote, audit asynchronously, retract if needed* — a different governance model than offline's *gate-and-promote*.

These three constraints define the design space for the rest of this chapter.

---

## §2 — Pattern 1: Live execution feedback

The largest and most-cited family. The agent acts in a real environment and uses outcomes as the discovery signal.

### Voyager — the canonical lifelong-learning loop

[github.com/MineDojo/Voyager](https://github.com/MineDojo/Voyager) ships the reference implementation: an automatic curriculum proposes the next sub-task, the agent writes a candidate skill, executes it in live Minecraft, observes errors, refines, and on success embeds the description and adds the skill to the library. The library grows turn-by-turn during play. **No batch step exists** — the loop runs entirely online. The retrieval primitive (description-embedding + cosine similarity) is also the discovery primitive, since each new skill embedding is computed at promotion time.

### Cradle — General Computer Control

[github.com/BAAI-Agents/Cradle](https://github.com/BAAI-Agents/Cradle) extends the pattern to any GUI application: atomic skills are pre-registered (move character, click button), composite skills auto-compose during play, and the registry grows live. Critical engineering choice: the system distinguishes *atomic* (per-environment primitives, hand-curated, T-Reviewed) from *composite* (auto-discovered, T-Scanned). This is a practical instance of the [docs/175](175-agent-skills-ecosystem-and-security.md) four-tier model applied online.

### EXIF — exploration + iterative feedback

[arXiv:2506.04287](https://arxiv.org/abs/2506.04287) (covered in [docs/174](174-autonomous-skill-exploration-iterative-feedback.md)) runs **Alice/Bob** as an online loop in Webshop / Crafter. The same-model self-evolution finding is operationally significant: the cheapest online discovery system is one model wearing two hats (proposer + executor), since the per-tick latency stays low.

### Eureka — RL-as-online-discovery

[github.com/eureka-research/Eureka](https://github.com/eureka-research/Eureka) treats RL training as the discovery loop: an LLM proposes reward functions, IsaacGym executes thousands of rollouts under each, and surviving rewards become curated training signals. *Discovery is online relative to training*; the discovered skill (the resulting policy) is then offline-stable. This is the **online-discovery → offline-deployment** hybrid pattern.

### SkillRL co-evolution

[docs/170](170-skillrl-recursive-skill-augmented-rl.md) co-evolves a policy and a SkillBank during training. The skill library grows turn-by-turn within the training loop — strict online discovery in the RL sense, even though the resulting model is then deployed offline.

---

## §3 — Pattern 2: Runtime marketplace hot-load

Skills are not authored locally; they're *pulled at runtime* from a remote catalog. Discovery shifts from "find a working skill" to "find an *existing* working skill in a 1,000+-skill catalog and load it."

### Anthropic Skills API + progressive disclosure

`anthropics/skills` (130k★) plus the [Claude Skills API](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) ship the canonical pattern. Three-tier disclosure runs *every turn*: at session start, all installed skill descriptions are pre-loaded into the system prompt; when the user prompts, the model decides which descriptions match the query; the matching skills' bodies are loaded just-in-time. The marketplace surface is therefore *live* — Claude Code watches `~/.claude/skills/` for filesystem changes and re-attaches new skills mid-session without restart. (Per the official docs: *"Adding, editing, or removing a skill under `~/.claude/skills/` … takes effect within the current session without restarting."*)

This is **the** reference implementation of online curation — every change to the catalog is immediately discoverable, and the activation surface re-evaluates without a session bounce.

### smolagents Hub-pull

[github.com/huggingface/smolagents](https://github.com/huggingface/smolagents) lets agents `share/pull tools or agents to/from the Hub` at runtime. A user can install a new community tool mid-session, and the next CodeAgent step's import graph picks it up. Same pattern as Anthropic's filesystem watcher, but Hub-rooted.

### MCP `tools/list_changed`

The [Model Context Protocol tools spec](https://modelcontextprotocol.io/docs/concepts/tools) defines a `notifications/tools/list_changed` message: *"When the list of available tools changes, servers that declared the `listChanged` capability **SHOULD** send a notification."* Clients re-issue `tools/list` and absorb the new catalog. This is the *protocol-level* equivalent of Anthropic's filesystem watcher — runtime catalog evolution without a reconnect.

### Glama gateway proxy

[Glama](https://glama.ai/mcp/servers) (21,586 servers) proxies MCP traffic through its gateway with **managed OAuth + per-tool access control + full call logging**. Skills hot-loaded through Glama don't just discover — they're audited, rate-limited, and curated *at the gateway*. This is the most production-grade online curation surface in the OSS landscape.

---

## §4 — Pattern 3: In-session dialogue extraction

Skills emerge from the conversation itself, not from execution.

### AutoSkill — the dialogue extractor

[docs/167](167-autoskill-experience-driven-lifelong-learning.md) ships the canonical loop: every user turn passes through `P_ext` (proposes candidate skill), `P_judge` (LLM judge scores durability), `P_merge` (merges into existing skill if related). The loop runs *per turn*, online. Cost is bounded by skipping turns that lack durability markers (always / never / prefer / avoid / in <language>).

### Lyra's auto-extractor

`lyra-skills/extractor.py` runs per-turn over the agent's trace; AutoSkill descendant. Latency-bounded by deferring judgement to idle cycles — the extractor proposes; the curator reviews offline.

### Polaris's auto_creator

`polaris-skills/auto_creator/` (P10) runs the same shape, integrated with the v2.2 heartbeat scheduler so Consolidation cycles run the curator while extraction runs in the live loop. This is the cleanest split of the *online extraction + offline curation* hybrid.

---

## §5 — Three constraints that govern online discovery

### Latency-bounded extraction

The loop must complete inside the per-tick budget. Three engineering tactics:

1. **Cheap pre-filter, expensive post-filter.** AutoSkill's durability-marker regex (cheap, runs every turn) gates the LLM judge (expensive, runs only when the regex hits). Polaris's heartbeat Reflection (cheap, every 10 min) gates Consolidation (expensive, every 1 hour).
2. **Deferred validation.** The extractor proposes immediately; the validator runs asynchronously. Voyager and Cradle both adopt this — the proposed skill is added to the library in a *probationary* state; failures during subsequent use demote it.
3. **Approximate retrieval at activation time.** Embedding cosine similarity beats LLM rerank on latency by 10–100×. For fewer than ~50 active skills, full progressive disclosure (Anthropic's pattern) fits inside the system-prompt budget; beyond that, layer in vector retrieval — see [docs/179](179-skill-retrieval-routing-and-activation.md).

### Drift detection

Skills that worked yesterday may fail today. The detector re-validates each promoted skill on a cadence and demotes on regression. Implementation patterns:

- **Hash-pinned skills + retraction-style re-scan.** [docs/175](175-agent-skills-ecosystem-and-security.md) §6 — every active skill carries a SHA-256; a body drift drops trust tier to T-Untrusted automatically.
- **Periodic re-test on a held-out task.** Voyager and Cradle both include the skill's earliest passing test in the library so re-running the test on a cadence detects regression.
- **Telemetry-based drift.** Polaris and Lyra both ship usage telemetry; a skill whose pass rate drops below threshold is auto-flagged for review.

### Mid-session governance

The curator can't block the loop. Three workable patterns:

1. **Speculative promote, async audit, retract if needed.** Default for online discovery. The skill is admitted to the library after a cheap pre-filter passes; full audit runs in a background cycle; if audit fails, the skill is demoted and any subsequent uses retracted.
2. **Probation tier.** A new fourth tier between T-Untrusted and T-Scanned: T-Probationary. The skill is loadable but only by the original author or under an active audit subscription. After K successful uses without trip, promotes automatically.
3. **Hard-gate non-author contributions.** For marketplace pulls, the speculative pattern is too risky (per the 26.1% finding). Treat marketplace skills as offline-curation-required; only locally-extracted skills get the speculative path.

---

## §6 — When to use online vs offline

A decision matrix:

| If… | Choose | Why |
|---|---|---|
| You have a benchmark with ground-truth pass/fail | **Offline batch (EvoSkill / CoEvoSkills)** | Pareto search converges faster than online iteration when the signal is clean |
| You have live execution but no canonical tasks | **Online (EXIF, Voyager, Cradle)** | Iterative feedback over real rollouts is the only viable signal |
| You have a stable API surface and a simulator | **Hybrid: offline-sim pre-filter + online execution** | [docs/173](173-offline-sim-skill-discovery.md) — simulator filters bad candidates before they touch live |
| You're consuming a marketplace catalog | **Online hot-load + offline scanner** | Live load matches user intent; offline scanner enforces [docs/175](175-agent-skills-ecosystem-and-security.md) |
| You're in a long-running daemon (≥hours) | **Online with heartbeat scheduler** | Drift detection is non-optional; idle cycles run the curator |
| You're in a short ephemeral session (<1hr) | **Lean on the marketplace; skip online discovery** | The cost of running the discovery loop exceeds the value over so few turns |
| You have open weights and RL infrastructure | **SkillRL co-evolution** | Train the policy and the bank together; online relative to training |

The dominant 2026 production pattern is **hybrid**: offline batch discovery on a fixed cadence (Polaris's Consolidation cycle, Lyra's nightly curator run) plus online streaming extraction during live operation, with both feeding the same SkillBank.

---

## §7 — Polaris integration slot

Polaris already implements most of the online surface. The gaps:

```text
packages/polaris-skills/src/polaris_skills/research/
  exploration_loop.py         # NEW [174] — Alice/Bob loop, fires from
                              # the v2.2 heartbeat scheduler's
                              # Reflection cycle.
  drift_detector.py           # NEW [§5] — periodic re-validation;
                              # demotes on regression.
  speculative_promote.py      # NEW [§5] — three-tier governance
                              # for online vs offline curation.

packages/polaris-mcp/src/polaris_mcp/
  list_changed_handler.py     # NEW — handle `tools/list_changed`
                              # notifications; refresh catalog without
                              # session bounce.
```

Bright-line additions:

```
BL-ONLINE-PROMOTE-COST    Online speculative promotion respects the
                          program's cost envelope; defers if over.
BL-DRIFT-RE-VALIDATE      A skill whose pass rate falls below
                          threshold drops to T-Untrusted automatically.
BL-MARKETPLACE-OFFLINE    Marketplace-fetched skills cannot use the
                          speculative-promote path; offline scanner
                          mandatory before active-library entry.
```

## §8 — Lyra integration slot

```text
packages/lyra-core/src/lyra_core/skills/
  exploration_loop.py         # NEW [174] — Alice/Bob on top of subagent
                              # orchestrator + arena/elo.
  drift_detector.py           # NEW [§5] — usage-telemetry-driven.
  hot_reload_watcher.py       # NEW — filesystem watcher for
                              # ~/.lyra/skills/ (matches Anthropic's
                              # Claude Code pattern).

packages/lyra-mcp/src/lyra_mcp/
  list_changed_handler.py     # NEW — runtime catalog refresh.
```

Lyra's existing `lyra-skills/extractor.py` already runs online; the gaps are drift detection, hot-reload, and the EXIF exploration loop.

---

## §9 — Where this fits

- [173 — Offline-Sim Skill Discovery](173-offline-sim-skill-discovery.md) — the *offline* counterpart axis.
- [174 — EXIF Autonomous Exploration](174-autonomous-skill-exploration-iterative-feedback.md) — the canonical online-execution paper.
- [175 — Agent Skills Ecosystem & Security](175-agent-skills-ecosystem-and-security.md) — the four-tier framework that applies differently online vs. offline.
- [176 — Open-Source Skills on the Internet](176-skill-discovery-curator-oss-landscape-may-2026.md) — Layer B catalogs are the marketplace surface for Pattern 2 above.
- [177 — Strongest 2026 Techniques Synthesis](177-skills-discovery-curator-strongest-2026-techniques.md) — folds online discovery as a first-class axis.
- [179 — Skill Retrieval, Routing, Activation](179-skill-retrieval-routing-and-activation.md) — online discovery's downstream problem: *given* the catalog, which skill activates?
