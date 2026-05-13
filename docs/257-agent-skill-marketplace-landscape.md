# 257 — Agent-Skill Marketplace Landscape: SKILL.md, subagents.cc, buildwithclaude, LobeHub, the marketplace economy

**Anchors.** SKILL.md spec — Anthropic, adopted by OpenAI Codex (Dec 2025); the converged portable specialist-definition format. Major marketplaces: [subagents.cc](https://subagents.cc/), [buildwithclaude.com](https://buildwithclaude.com/), [LobeHub Skills](https://lobehub.com/skills), [Smithery](https://smithery.ai/) (MCP servers); awesome-list scale: [VoltAgent/awesome-claude-code-subagents](https://github.com/VoltAgent/awesome-claude-code-subagents) (~100+ specialists), [wshobson/agents](https://github.com/wshobson/agents) (~44), [everything-claude-code](https://github.com/affaan-m/everything-claude-code) (README reports 208 skills), and applied optimizers such as [Self-Improving Agent Skills](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/awesome_agent_skills/self-improving-agent-skills). Companions: [04-skills](04-skills.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md), [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md), [254-a2a-protocol-deep-dive](254-a2a-protocol-deep-dive.md), [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md), [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md), [258-agent-protocol-stack-synthesis-2026](258-agent-protocol-stack-synthesis-2026.md).

> **Disambiguation.** [04-skills](04-skills.md) covers the core SKILL.md primitive. [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md) covers the *self-evolution* dimension. [175-agent-skills-ecosystem-and-security](175-agent-skills-ecosystem-and-security.md) covers the *security* dimension. This file (257) covers the **marketplace economy** dimension — distribution, discovery, packaging, and the production infrastructure that turns one team's skill into another team's `pip install`.

**One-line definition.** A 2025–2026 ecosystem inflection where the **SKILL.md format converged** (Anthropic-original, OpenAI Codex aligned Dec 2025), **multiple competing marketplaces** emerged (subagents.cc, buildwithclaude, LobeHub Skills for skills/subagents; Smithery for MCP servers), **awesome-list scale repos** consolidated 100+ specialists per repo (VoltAgent, wshobson), and the **production architecture** — Signed Agent Cards ([254](254-a2a-protocol-deep-dive.md)) + OASF schemas ([255](255-agntcy-oasf-acp-deep-dive.md)) + OAuth 2.1 install flows ([256](256-mcp-2025-2026-evolution.md)) + trust tiering — gave the field its first credible **agent-marketplace economy** with hundreds of thousands of monthly installs and the early signals of a power-law distribution where a small set of canonical specialists serve most use cases.

## Why this paper matters (the agent ecosystem developed an economy in 2025–2026)

For the first ten years of the modern AI era, the field treated skills, prompts, and agents as **per-team artifacts** — your prompts, your subagents, your tools, locked in your repo. The 2025–2026 inflection — driven by SKILL.md convergence, MCP standardization ([256](256-mcp-2025-2026-evolution.md)), and the marketplace platforms below — turned this into an **ecosystem economy**: a security reviewer, a doc generator, a database-migration specialist, a code-review-for-Python-projects skill — these are now portable artifacts published, signed, discovered, installed, and rated, with cross-vendor compatibility. The same SKILL.md works in Claude Code, OpenAI Codex, Cline, Continue, and custom agent runtimes that vendor the parser.

The economic implications are non-trivial. **Production agent runtimes can now compose** — your Polaris research agent doesn't need to reinvent the lit-search skill; install one from the marketplace. **Verified-vendor specialists become trust-tiered infrastructure** — security-reviewer-by-vetted-vendor is a different artifact than security-reviewer-by-random-fork. **The skill-discovery-curator pattern** ([177](177-skills-discovery-curator-strongest-2026-techniques.md), [176](176-skill-discovery-curator-oss-landscape-may-2026.md)) becomes practical when the candidate pool is thousands of marketplace skills, not your team's 12 hand-rolled ones. **Cost-routing by skill provider** ([15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md)) becomes a real lever — different vendors price differently.

The four-platform landscape that emerged is informative. **Smithery** (https://smithery.ai/) is the production MCP-server marketplace — signed servers, OAuth-mediated install, search-by-capability. **subagents.cc** is the community discovery surface for Claude Code subagents — searchable, ratings, install-with-one-click. **buildwithclaude.com** is the curation layer — vetted skills aligned to common workflows. **LobeHub Skills** (https://lobehub.com/skills) is the cross-runtime catalog — skills usable across LobeHub, Claude Code, Codex, and the LobeHub-bundled agents. Together they cover the four corners of the marketplace economy: marketplace (Smithery), community (subagents.cc), curation (buildwithclaude), cross-runtime catalog (LobeHub).

Two newer community artifacts add a second axis beyond distribution. [everything-claude-code](https://github.com/affaan-m/everything-claude-code) is a cross-harness corpus of skills, hooks, rules, MCP configs, memory optimization, security scanning, and continuous-learning workflows; its README reports 140k+ stars and 208 skills, while GitHub metadata surfaced 180k stars during this update. [Self-Improving Agent Skills](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/awesome_agent_skills/self-improving-agent-skills) is an optimizer rather than a catalog: Executor/Analyst/Mutator ADK agents generate evals, diagnose failures, mutate one skill-prompt change, and keep/revert based on score. The market is therefore moving from **publish skills** to **operate and improve skills**.

Take this seriously and three things change. **First**, your harness's **`harness_core/skills/` package becomes a hybrid of vendored and marketplace-installed skills** — vendored for must-have determinism, marketplace-installed for breadth. **Second**, the **trust-tiering and signature primitives from OASF / Signed Agent Cards** become production-mandatory; you don't install unsigned marketplace skills with shell access. **Third**, your **eval pipeline includes regression tests against marketplace-skill drift** — when an installed skill updates, you re-run your tests; this is a new lifecycle the field didn't have in 2024.

## Problem it solves (turning skills from per-team artifacts into a production ecosystem)

1. **No portable specialist format.** Each runtime had its own way to define a "subagent" or "specialist." SKILL.md convergence (Anthropic + OpenAI Codex Dec 2025) gave the field one format.
2. **No standard install / discovery flow.** Marketplaces (Smithery, subagents.cc, buildwithclaude, LobeHub) provide search, ratings, OAuth-mediated install.
3. **Trust on first use.** Without signed manifests, every installed skill was a supply-chain risk. OASF trust tiers + Signed Agent Cards mitigate.
4. **Skill drift after install.** No regression-test culture for installed skills. Marketplace + harness eval pipelines create one.
5. **Cross-runtime portability gaps.** A skill written for Claude Code didn't natively run in Codex. SKILL.md convergence closes most of this.
6. **No per-provider cost differentiation.** All skills priced uniformly. Marketplace metadata enables cost-aware routing.
7. **Discovery at scale.** With thousands of skills, search and curation become the binding constraints; the four-platform landscape addresses this.

## Core idea in one paragraph

The 2025–2026 agent-skill marketplace landscape rests on three primitives. **SKILL.md** is the converged portable specialist definition: a Markdown file with frontmatter (name, description, tools, model, version, author) and body (system prompt + skill instructions); Anthropic-original, OpenAI Codex aligned Dec 2025, parseable by any runtime that vendors the spec. **Signed manifests** (OASF + Signed Agent Cards from [254](254-a2a-protocol-deep-dive.md), [255](255-agntcy-oasf-acp-deep-dive.md)) provide cryptographic identity and capability declaration — what tools the skill needs, what trust attributes it carries, what its expected duration and cost are. **OAuth 2.1 install flows** ([256](256-mcp-2025-2026-evolution.md)) turn skill installation from "git clone and trust" into "OAuth-mediated install with explicit scope grant." On these primitives, four marketplace platforms emerged: **Smithery** (MCP server marketplace, production-shape, signed servers, OAuth install), **subagents.cc** (Claude Code subagent community discovery, ratings, search), **buildwithclaude.com** (curated skills aligned to workflows), **LobeHub Skills** (cross-runtime catalog). Awesome-list-shape repos consolidate 100+ specialists per repo (VoltAgent's awesome-claude-code-subagents, wshobson/agents) — the open-source community curation layer that complements the platform marketplaces. The production stack composes naturally: install a skill from a marketplace via OAuth-mediated flow, verify its OASF manifest signature, run it under your harness's permission bridge with scoped tool grants, observe its outputs in your HIR log, regression-test it on update. The field finally has the **distribute → discover → install → run → audit → update** cycle that the rest of software has had for two decades.

## Mechanism (step by step)

### (a) The SKILL.md format

```markdown
---
name: security-code-reviewer
description: Identifies common security issues (OWASP Top 10) in Python code
version: 1.2.0
author: example-org
license: MIT
tools:
  - read_file
  - search_dir
  - run_tests
model_recommendation: claude-sonnet-4-6
expected_duration_seconds: 300
cost_estimate_per_call_usd: 0.20
trust_attributes:
  signed_by: example-org
  audit_date: 2026-04-01
  vulnerability_scan_passing: true
---

# Security Code Reviewer

You are a security-focused code reviewer specializing in Python.

## Process

1. Read each file under review
2. Look for OWASP Top 10 issues:
   - Injection (SQLi, command injection)
   - Broken authentication
   - ...

## Output Format

For each issue, emit a finding with:
- Severity (critical / high / medium / low)
- File path + line number
- Vulnerability category
- Fix recommendation
```

The frontmatter is YAML; the body is Markdown system-prompt + instructions. Parser-portable across runtimes that vendor the spec.

### (b) Marketplace platforms

**Smithery (https://smithery.ai/)** — MCP server marketplace:

- Search by capability: `https://smithery.ai/?q=database`
- Each server has signed manifest, OAuth install URL, ratings, install count
- Install via `smithery install @org/server-name` (CLI) or one-click in supported clients
- Verifies signatures at install; refuses unsigned

**subagents.cc** — Claude Code subagent community:

- Search by category: `code-review`, `documentation`, `testing`, `data`
- Each subagent is a SKILL.md-shape file with metadata
- Install: `claude-code agents install <handle>`
- Community ratings, comment threads, fork tracking

**buildwithclaude.com** — Curation layer:

- Editorial curation of best-in-class skills for common workflows
- Each curated workflow is a bundle of N skills + prompt scaffolding
- Lower volume, higher quality bar than community marketplaces

**LobeHub Skills (https://lobehub.com/skills)** — Cross-runtime:

- Skills usable across LobeHub, Claude Code, Codex, custom agents
- Bundles include MCP server dependencies declared in SKILL.md frontmatter
- Identity-portable: install once, use across runtimes

### (c) OAuth 2.1 install flow

```
1. User clicks "Install" on marketplace listing
2. Marketplace redirects to consent screen showing requested scopes
3. User approves; marketplace generates OAuth bearer token scoped to the skill
4. Token + SKILL.md manifest delivered to local agent runtime
5. Runtime verifies manifest signature against published JWKs
6. Runtime registers skill with permission bridge using scoped grants
7. Skill is callable; bridge audits each tool invocation
```

### (d) Trust tiering integration

OASF trust attributes ([255](255-agntcy-oasf-acp-deep-dive.md)) flow into marketplace UX:

| UI affordance | Backend |
|---|---|
| ✅ "Verified vendor" badge | Manifest signed by registered vendor |
| ✅ "Audited" badge | `audit_date` within last 12 months |
| ✅ "Sandboxed" badge | `runtime_isolation: worktree` |
| ⚠ "Unsigned" warning | No signature; install at own risk |

User can filter by minimum trust tier in marketplace search.

### (e) Awesome-list community curation

Two repos lead in May 2026:

**VoltAgent/awesome-claude-code-subagents** (~100+ specialists):

- Categorized: SWE lifecycle, data, security, devops, design, content
- Each entry: SKILL.md + README + example trace
- PR-based contribution; maintainers triage

**wshobson/agents** (~44 specialists):

- Smaller, opinionated, curated production-quality
- "44 production specialists across SWE lifecycle"

**everything-claude-code** (README reports 208 skills):

- Cross-harness performance corpus for Claude Code, Codex, Cursor, OpenCode, Gemini, and related runtimes
- Packages skills with hooks, memory persistence, security scanning, MCP configs, rules, and continuous-learning workflows

**Self-Improving Agent Skills**:

- Google ADK + Gemini app for uploading a skill folder, generating eval scenarios, running/scoring outputs, diagnosing failures, and mutating the skill prompt
- Important because it treats marketplace artifacts as living software that need regression tests and automated improvement loops
- Used as the canonical reference for non-marketplace consumption

These complement platform marketplaces — community-maintained, no install gating, but no signature verification either.

### (f) Regression testing post-install

Skills update; their behavior drifts. The 2026 production pattern:

```python
# .harness_core/skills_eval/test_security_reviewer.py
def test_security_reviewer_on_known_vulns():
    skill = load_skill("security-code-reviewer", version=">=1.2.0")
    findings = run_skill(skill, fixture="known_sqli.py")
    assert any(f.category == "sql_injection" for f in findings)
    assert any(f.severity == "critical" for f in findings)
```

Run on every skill update via cron-driven routine ([252](252-routines-pattern-for-self-hosted-agents.md)); auto-pin to last-known-good on regression.

### (g) Cost-routing by marketplace

Each skill in a marketplace declares cost estimate; the cost-router ([15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md)) selects per-task:

```python
candidates = marketplace.search(capability="code-review", min_trust="audited")
selected = router.pick(candidates, criterion="lowest_cost_within_quality_threshold")
```

### (h) Marketplace-driven self-evolution

[171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md) — when your harness's auto-creator generates a new skill, it can publish to the marketplace; rated by other consumers; updates flow back as a feedback loop.

## Empirical results (table — May 2026 marketplace state)

| Platform | Type | Estimated catalog size | Signing | Install model |
|---|---|---:|---|---|
| Smithery | MCP servers | 5,000+ | Yes (signed manifests) | OAuth + CLI |
| subagents.cc | Claude Code subagents | ~1,000+ | Optional | claude-code CLI |
| buildwithclaude.com | Curated skills + workflows | ~200 (high curation) | Optional | One-click |
| LobeHub Skills | Cross-runtime skills | ~3,000+ | Optional | LobeHub CLI / direct |
| awesome-claude-code-subagents | Community list (no install gate) | ~100+ | No | git clone |
| wshobson/agents | Curated community list | ~44 | No | git clone |
| everything-claude-code | Cross-harness skill/hook/rule corpus | README reports 208 skills | Repo-level | git clone / package install |
| Self-Improving Agent Skills | Skill optimizer app | upload one skill at a time | App-level eval gate | local web app |

| Pattern | Adoption (May 2026) |
|---|---|
| SKILL.md format used in production | Anthropic, OpenAI Codex, LobeHub, Continue, Cline |
| Signed manifest in marketplace install | Smithery (yes), others (optional) |
| Trust-tier filtering UI | Smithery + buildwithclaude (yes) |
| OAuth 2.1 install flow | Smithery (yes); evolving elsewhere |
| Cross-runtime portability | LobeHub Skills (designed for); SKILL.md (parseable) |

## Variants and ablations

- **Vendored vs marketplace mix.** `harness_core/skills/vendored/` for must-have; `~/.config/harness/skills/installed/` for marketplace.
- **Bundled workflows.** buildwithclaude-style — a "documentation workflow" bundles N skills + a prompt scaffold.
- **Per-team private marketplaces.** Self-hosted Smithery / AGNTCY-shape registry for internal-only skills.
- **Skill-with-MCP-deps bundling.** SKILL.md frontmatter declares MCP server dependencies; install pulls both.
- **Marketplace-driven A/B testing.** Two skill versions live; router picks by win-rate.
- **Skill auto-creator publication.** [10-skill-auto-creator](../projects/polaris/docs/blocks/10-skill-auto-creator.md) — auto-publish new skills to internal marketplace.
- **Citation-graph for skills.** [22-citation-graph-substrate](../projects/polaris/docs/research/p22-citation-graph-substrate.md) — track which skills cite/depend on which.
- **Trust retraction propagation.** [p24-trust-tiering-retractions](../projects/polaris/docs/research/p24-trust-tiering-retractions.md) — when a skill is retracted, dependents notified.

## Failure modes and limitations

- **Supply-chain attacks via unsigned skills.** Hot risk; marketplace gating mitigates only if user enforces minimum trust tier.
- **Skill drift after install.** Vendored is deterministic; marketplace-installed isn't. Regression tests + version pinning required.
- **Marketplace platforms compete with each other.** Fragmentation: a skill on subagents.cc may not be on LobeHub. Cross-listing helps but isn't universal.
- **Trust-tier inflation.** Vendors self-attest; without third-party verifiers, badges become marketing.
- **Cost estimates inaccurate.** Marketplace metadata declares cost; real-world costs vary.
- **Permission-scope expansion attack.** Skill v1.0 needs `read_file`; v1.1 silently adds `write_file`. Permission bridge must re-prompt on scope expansion.
- **Dependency hell with MCP server deps.** Skill needs MCP server X v2.x; another skill needs X v1.x. Versioning matters.
- **Compatibility drift between runtimes.** SKILL.md convergence is real but imperfect; some skills work in Claude Code but not Codex.
- **Marketplace lock-in via custom extensions.** Each marketplace adds metadata fields; portability erodes.
- **Attack via marketplace search SEO.** Malicious skill ranks high via fake ratings; trust tiering addresses but not perfectly.
- **No revocation across marketplaces.** A skill banned on subagents.cc may stay live on a community fork.

## When to use, when not

**Adopt marketplace skills** when capability breadth matters more than determinism (research agents, exploratory tooling, content generation); when a community-curated specialist exists and signed-by-audited-vendor; when your eval pipeline can regression-test installed skills; when cross-runtime portability matters (you ship to Claude Code + Codex + custom). The strongest cases are **personal-assistant agents** (mentat-learn, lyra) and **research agents** (polaris, atlas-research, helix-bio).

**Vendor skills instead of marketplace** when determinism is critical (production deploys, compliance-sensitive workflows); when no acceptable signed/audited skill exists; when your security perimeter cannot tolerate OAuth-installed code; when the skill is core IP your team owns. The strongest cases are **regulated agents** (cipher-sec, aegis-ops) and **safety-critical paths** in any agent.

## Implications for harness engineering

- **Hybrid `harness_core/skills/` package.** Vendored + marketplace-installed; vendored takes precedence on conflict.
- **SKILL.md as the converged format.** [04-skills](04-skills.md), [09-skill-engine](../projects/polaris/docs/blocks/09-skill-engine.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md) — skill engine parses SKILL.md uniformly.
- **OASF manifest at marketplace install.** [255-agntcy-oasf-acp-deep-dive](255-agntcy-oasf-acp-deep-dive.md) — declared trust attributes, signed.
- **OAuth 2.1 install flow.** [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md), [07-permission-bridge-research-edition](../projects/polaris/docs/blocks/07-permission-bridge-research-edition.md) — per-skill scope grants.
- **Trust tiering at install + audit.** [p24-trust-tiering-retractions](../projects/polaris/docs/research/p24-trust-tiering-retractions.md), [14-bright-line-gates](../projects/polaris/docs/blocks/14-bright-line-gates.md) — bridge refuses below threshold.
- **Regression-test installed skills via routine.** [252-routines-pattern-for-self-hosted-agents](252-routines-pattern-for-self-hosted-agents.md) — scheduled `eval-skills` routine.
- **Cost-router with marketplace metadata.** [15-cost-router-and-budget](../projects/polaris/docs/blocks/15-cost-router-and-budget.md), [86-frugalgpt](86-frugalgpt.md), [87-routellm](87-routellm.md), [88-confidence-driven-router](88-confidence-driven-router.md).
- **Auto-creator publishes to internal marketplace.** [10-skill-auto-creator](../projects/polaris/docs/blocks/10-skill-auto-creator.md), [p9-ctx2skill-self-play](../projects/polaris/docs/research/p9-ctx2skill-self-play.md).
- **Skill-with-MCP-server bundle install.** [256-mcp-2025-2026-evolution](256-mcp-2025-2026-evolution.md), [17-mcp-adapter](../projects/polaris/docs/blocks/17-mcp-adapter.md) — installer pulls both.
- **HIR observability of skill calls.** [16-observability-hir](../projects/polaris/docs/blocks/16-observability-hir.md) — log skill name, version, source (vendored vs marketplace), cost.
- **Cross-channel verifier as marketplace skill.** [02-cross-model-adversarial-pair](../projects/polaris/docs/blocks/02-cross-model-adversarial-pair.md), [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md), [223-verifier-and-best-of-n-scaling](223-verifier-and-best-of-n-scaling.md) — verifier published as a signed skill.
- **Discovery + curator pattern.** [177-skills-discovery-curator-strongest-2026-techniques](177-skills-discovery-curator-strongest-2026-techniques.md), [176-skill-discovery-curator-oss-landscape-may-2026](176-skill-discovery-curator-oss-landscape-may-2026.md) — orchestrator queries marketplace for relevant skills.
- **Self-evolution feedback loop.** [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md) — auto-created skills publish; ratings feed back.
- **Anthropic Agent Teams + marketplace specialists.** [250-anthropic-agent-teams](250-anthropic-agent-teams.md) — teammate spawn references a marketplace SKILL.md by handle.

**One-line takeaway for harness designers.** **The 2025–2026 marketplace economy turned skills from per-team artifacts into installable signed verified-vendor specialists with OAuth 2.1 install flows, trust tiering, regression-test discipline, and cross-runtime portability via SKILL.md — adopt a hybrid vendored-plus-marketplace `harness_core/skills/` package, gate installs by minimum trust tier, regression-test on update, and treat the `distribute → discover → install → run → audit → update` cycle as production infrastructure the same way npm or PyPI is.**
