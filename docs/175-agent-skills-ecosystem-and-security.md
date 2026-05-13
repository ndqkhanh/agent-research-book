# 175 — Agent Skills: Architecture, Acquisition, Security, and the Curator's Mandate

**Source.** Renjun Xu, Yang Yan. *Agent Skills for Large Language Models: Architecture, Acquisition, Security, and the Path Forward*. arXiv:[2602.12430](https://arxiv.org/abs/2602.12430) (Feb 2026).

**One-paragraph thesis.** Skills have become a *foundational abstraction layer* between language models and tools — modular, portable, dynamically loadable units of procedural expertise. As the abstraction settles, the ecosystem is growing faster than its safety substrate: the paper's central empirical finding is that **26.1% of community-contributed skills contain vulnerabilities** (prompt-injection, tool-misuse, privilege-escalation, supply-chain). The paper's proposed mitigation — a "Skill Trust and Lifecycle Governance Framework" with a four-tier, gate-based permission model that maps skill provenance to graduated deployment capabilities — is the load-bearing argument for an *automated skill curator*. The "publish-and-forget" community-marketplace model is broken by the data; curation must be machine-enforced, not human-promised.

## §1 — Architecture taxonomy

The paper draws a three-tier classification of how skills attach to a language model. Polaris and Lyra both already inhabit Tier 2; the taxonomy clarifies the design space:

| Tier | Skill location | Example | Activation cost |
|---|---|---|---|
| **Tier 1 — Model-attached** | Compiled into model weights or as a fine-tuned adapter | Toolformer / ToolkenGPT (special tokens), SkillRL ([170](170-skillrl-recursive-skill-augmented-rl.md)) | High (training); zero at inference |
| **Tier 2 — Runtime-resident** | Loaded into the agent's context as needed | Polaris `polaris-skills`, Lyra `lyra-skills`, Anthropic `~/.claude/skills` | Low; per-session activation |
| **Tier 3 — Community-contributed** | Pulled from a remote marketplace at runtime | Anthropic Skills Hub, smolagents Hub, MCP servers | Variable; per-pull cost + supply-chain risk |

The cross-tier security profile is sharply different — Tier 3 is where 26.1% of the population fails the paper's vulnerability scan.

## §2 — Acquisition modes

Five acquisition modes cover the full pipeline from authoring to runtime activation:

1. **Hand-authored** — a developer writes `SKILL.md` directly. Highest quality bar; lowest scaling.
2. **Auto-extracted** — pattern detection over agent traces produces candidate skills. Polaris `auto_creator`, Lyra extractor. Medium quality bar; high scaling.
3. **Document-derived** — Ctx2Skill ([154](154-ctx2skill-self-evolving-context-skills.md)) and AutoSkill4Doc convert papers / manuals into skills.
4. **Failure-driven** — EvoSkill ([168](168-evoskill-coding-agent-skill-discovery.md)) extracts skills from rejection events. Pareto-frontier search.
5. **Marketplace-fetched** — pull from a shared catalog (Anthropic Skills, MCP servers). Lowest quality bar; highest scaling — and **the source of the 26.1% finding**.

The paper argues every mode needs a different curator policy. Hand-authored skills can rely on author trust. Marketplace-fetched skills cannot — the trust must be computed, not assumed.

## §3 — The 26.1% finding

The paper's central empirical contribution is a **vulnerability scan of community-contributed skills**. The headline: **26.1% of skills carry at least one exploitable issue**. The vulnerability classes break down (paraphrased from the paper's taxonomy):

- **Prompt-injection inside skill body.** A skill includes an attacker-controlled string that, when loaded into the model's context, hijacks behaviour. Variants: instruction-injection in description, hidden Unicode tags, ANSI escape sequences.
- **Tool-misuse.** A skill exposes a tool with broader scope than its description claims. The user accepts the description, the tool acts beyond it.
- **Privilege-escalation.** A skill chains an `allowed-tools` grant against a follow-on tool whose own approvals would have been required separately. The chain bypasses the per-tool gate.
- **Supply-chain vulnerabilities.** A skill imports an external package or fetches a remote resource at activation time; a compromise of the upstream changes skill behaviour without an updated SKILL.md.

Severity is not uniform: prompt-injection in a research-skill is a different threat than privilege-escalation in a deploy-skill. The four categories above represent **the minimum vulnerability scanner** every curator must implement.

## §4 — Skill Trust and Lifecycle Governance Framework

The paper's proposed mitigation is a **four-tier gate-based permission model**:

| Tier | Provenance | Default permissions | Promotion gate |
|---|---|---|---|
| **T-Untrusted** | Just-fetched from a marketplace | Read-only context; no tool grants | Manual review + scan-pass |
| **T-Scanned** | Passed automated vulnerability scan | Read + safe tools | Human review of `allowed-tools` |
| **T-Reviewed** | Human-reviewed and signed | Author-approved tools | Production-readiness gate |
| **T-Pinned** | Cryptographically pinned to a content hash | Full grant | Re-scan on hash change |

The framework is **gate-based, not list-based**: a skill at T-Reviewed is *not* universally trusted — it's trusted *for the tools the reviewer signed off on*. Adding a new tool to the skill drops it back to T-Scanned for that tool only. This preserves the promotion bar's meaning across skill mutations.

The Polaris analogue is the existing `BL-PROMOTE-SKILL` bright-line. The Lyra analogue is the existing `BL-LYRA-SKILL-PROMOTE`. Both are conceptually compatible with the four-tier model — the paper formalises what both harnesses already do informally.

## §5 — What this implies for harness designers

Three concrete consequences:

1. **No marketplace skill should reach the active library without scanning.** The 26.1% finding is *prior* — even a marketplace claiming curation is, on this evidence, more likely than not to ship at least one vulnerable skill in any given install. Polaris and Lyra both already gate marketplace pulls behind a curator; the paper validates the design.
2. **The `allowed-tools` field is the load-bearing surface.** Most observed exploits route through `allowed-tools` widening. The curator's most important automated check is *the diff between the skill's description and the tools it grants itself*. A description that says "summarise files" coupled with `allowed-tools: Bash(*)` is a smell, regardless of who authored it.
3. **Content-pinning is non-optional.** Every active skill should carry a content hash, and any drift from the hash should drop the skill's trust tier until re-reviewed. This is the *retraction-aware fetch* discipline from `docs/172` §3 Gap 5 applied to the skill catalog rather than to the citation catalog.

## §6 — Mitigations the harnesses should adopt

A concrete checklist matching the paper's vulnerability classes to actionable scanner primitives:

| Vulnerability class | Detector | Implementation hint |
|---|---|---|
| Prompt-injection in body | Regex + entropy scan | Existing patterns in `BL-CIPHER-SKILL-EXFIL`; extend with hidden-Unicode check |
| Tool-misuse | Description-vs-`allowed-tools` consistency check | LLM-as-judge vs. explicit rule list; both should run |
| Privilege-escalation | Chain-of-grants graph analysis | Build a tool-grant graph; flag skills whose chain crosses a sensitivity boundary |
| Supply-chain | Remote-fetch detection | Block any skill body that imports HTTP / NPM / PyPI at activation time |

This is the minimum viable scanner. A 2026 implementation should also catch *prompt-injected description* (since description is what drives the activation gate; see [179](179-skill-retrieval-routing-and-activation.md)).

## §7 — Polaris integration slot

```text
packages/polaris-skills/src/polaris_skills/curator/
  vulnerability_scanner.py   # NEW — implements the four vulnerability
                             # classes from §3.
  trust_tiers.py             # NEW — the four-tier model from §4.
  content_pinning.py         # NEW — sha256-anchor every active skill;
                             # detect drift on every load.
```

Bright-line additions (proposed):

```
BL-SKILL-MARKET-FETCH      A marketplace-fetched skill cannot enter
                           the active library without passing the
                           vulnerability scanner.
BL-SKILL-DRIFT             A pinned skill whose body drifts from its
                           hash drops to T-Untrusted automatically;
                           re-review required.
BL-SKILL-DESCRIPTION-DRIFT A skill whose description-vs-allowed-tools
                           diff exceeds a configured threshold blocks
                           promotion until reviewer reconciliation.
```

## §8 — Lyra integration slot

```text
packages/lyra-skills/src/lyra_skills/
  security_scanner.py        # NEW — Lyra's vulnerability scanner.
  curator/                   # extended — the four-tier model.
```

Lyra already has a curator package (`lyra-skills/curator.py`); this paper's framework slots in as the `trust_tier` axis on every promoted skill.

## §9 — Where this fits

Pairs with:

- [056 — SEA Landscape 2026](056-sea-landscape-2026.md) — already cites this paper's 26.1% headline; this doc deepens.
- [171 — Skill Self-Evolution Synthesis](171-skill-self-evolution-2026-synthesis.md) — supply-chain becomes a sixth structural commitment alongside *skills are files / evolution beats one-shot / skills are more portable than weights*.
- [176 — OSS Skill Discovery + Curator Landscape](176-skill-discovery-curator-oss-landscape-may-2026.md) — surveys which OSS systems implement which parts of the four-tier model.
- [177 — Strongest 2026 Techniques Synthesis](177-skills-discovery-curator-strongest-2026-techniques.md) — the curator is non-optional; this paper's argument enters the synthesis as a load-bearing premise.
- [179 — Skill Retrieval, Routing, Activation](179-skill-retrieval-routing-and-activation.md) — the *description* field that drives skill activation is itself injection-attackable; the security scanner must run before the router does.
