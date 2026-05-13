# 210 — Mentat-Learn Apply Plan: The Most Natural Fit for the Collaborative-AI Canon

> **Disambiguation.** This file is the **Mentat-Learn–specific** deep apply plan. Read after [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md) (Polaris), [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md) (cross-project matrix), [208-lyra-multi-hop-collaborative-apply-plan](208-lyra-multi-hop-collaborative-apply-plan.md) (Lyra), [209-argus-multi-hop-collaborative-apply-plan](209-argus-multi-hop-collaborative-apply-plan.md) (argus). The collaborative-AI canon is in [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md); the multi-hop block is [198](198-multi-hop-qa-datasets-canon.md) → [202](202-multi-agent-multi-hop-reckoning-2026.md).

## One-line definition

A staged plan to fold collaborative-AI techniques (typed personal memory, branching, MCP marketplace, voice + screen-share, skill auto-creation, per-user constitution, local-first / IndexedDB) into Mentat-Learn — the **self-improving personal assistant** that fuses OpenClaw's multi-channel gateway, SemaClaw's DAG teams + SOUL.md persona + PermissionBridge, and Hermes Agent's closed skill-learning loop + Honcho dialectic user model — and to do so *without* over-investing in multi-hop reasoning techniques that are off-niche for short-horizon assistant tasks, because Mentat-Learn is **the most natural fit in the in-tree project portfolio for the "teammate that grows with you" framing** and the right plan reflects that.

## §1 — Why Mentat-Learn is the canonical collaborative-AI consumer

Of the 11+ in-tree projects, Mentat-Learn is the one whose **mission statement** is the framing of [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md): an agent that "actually gets more capable over time — not by updating its configuration, but by extracting reusable procedures from completed workflows." That is *literally* the LobeHub README pitch ("agent teammates that grow with you") rephrased into Mentat-Learn's own architecture document.

The implications for the apply plan are sharp:

- **Multi-hop QA techniques are mostly off-niche.** Mentat-Learn handles short-horizon assistant tasks across messaging channels (Slack, WhatsApp, Telegram, email, iMessage, web). The user is asking for help with a calendar conflict, a draft email, a quick summary — not deep multi-hop research. HippoRAG / Plan-on-Graph / Search-R1 RL belong in Polaris and Lyra, not Mentat.
- **Collaborative-AI techniques are *all* on-niche.** ICPEA personal memory, Voyager-line skill auto-creation, MCP marketplace, branching conversations, voice + screen-share, per-user constitution, local-first storage — every one of them maps directly onto a Mentat-Learn design target.
- **Cross-channel persona consistency is the binding constraint.** Mentat-Learn's design target #2 is "user-surveyed *agent felt consistent* score ≥ 0.9 across ≥ 5 interactions." This is a demand that ICPEA + per-user constitution + SOUL.md persona + Honcho dialectic model collectively deliver — and none of them alone is sufficient.

Take this plan seriously and three things change. (1) Mentat-Learn becomes the **showcase deployment** of the collaborative-AI canon — the in-tree project that demonstrates what "teammate that grows with you" actually looks like in production. (2) The four-layer memory (procedural / facts / session / dialectic) generalises into a six-layer memory (the four existing layers + ICPEA Identity + ICPEA Preference) that makes cross-channel persona truly portable. (3) Skill auto-creation feeds back into argus's curator (cf. [209-argus-multi-hop-collaborative-apply-plan](209-argus-multi-hop-collaborative-apply-plan.md) §3.6) — Mentat becomes a major contributor to the cross-project skill commons.

## §2 — The Mentat-Learn × technique mapping table

Each row is one technique, mapped to a Mentat-Learn module + tier slot. Mentat-Learn's existing module structure (from `src/mentat_learn/`): unified gateway, agent loop, four-layer memory (procedural / facts / session / dialectic), tool layer + MCP, skill library, skill extractor loop.

| Technique | Source | Mentat-Learn module | Tier | Effort | Payoff |
|---|---|---|---|---|---|
| **Cross-channel personal-memory substrate (Mentat's primary axis)** | | | | | |
| ICPEA Identity layer alongside SOUL.md persona | [205](205-lobehub-collaborative-teammate-platform.md), [206](206-collaborative-ai-canon-2026.md) §1 | `src/mentat_learn/memory/icpea/identity.py` (composes with `persona/soul.py`) | 0 | M | Cross-channel identity stable across Slack / WhatsApp / iMessage |
| ICPEA Preference layer for tone / channel / language | [206](206-collaborative-ai-canon-2026.md) §1 | `src/mentat_learn/memory/icpea/preference.py` | 0 | S | Channel-appropriate tone (formal email vs casual WhatsApp) |
| ICPEA Experience layer fused with existing dialectic | [206](206-collaborative-ai-canon-2026.md) §1 | `src/mentat_learn/memory/icpea/experience.py` (composes with `memory/dialectic.py`) | 1 | M | "What worked last time on this person's calendar?" |
| Async memory extractor on conversation turn | [206](206-collaborative-ai-canon-2026.md) §1 | `src/mentat_learn/memory/icpea/extractor.py` | 0 | S | Turn-by-turn memory write decoupled from response latency |
| Per-channel layer access control | [205](205-lobehub-collaborative-teammate-platform.md) §2.1 | `src/mentat_learn/memory/icpea/access.py` | 1 | S | Slack agent can't read WhatsApp Identity blob |
| Ebbinghaus-style decay on Activity layer | [206](206-collaborative-ai-canon-2026.md) §1 (MemoryBank) | `src/mentat_learn/memory/icpea/decay.py` | 1 | S | Three-month-old "I'm in Berlin" decays appropriately |
| **Voyager-line skill auto-creation (Mentat's secondary axis)** | | | | | |
| Trajectory-driven skill extractor | [167](167-autoskill-experience-driven-lifelong-learning.md), [197](197-argus-omega-vol-3-recursive-skills-curator.md), Hermes ([55](55-hermes-agent-self-improving.md)) | `src/mentat_learn/skill_extractor/extract.py` (extends existing extractor) | 0 | M | Successful workflows become reusable skills |
| Surrogate-verifier gating | [169](169-coevoskills-co-evolutionary-verification.md) | `src/mentat_learn/skill_extractor/surrogate_verifier.py` | 1 | M | +30pp ablation lift on skill quality |
| Submit promoted skills to Argus curator | [197](197-argus-omega-vol-3-recursive-skills-curator.md), [209](209-argus-multi-hop-collaborative-apply-plan.md) | `src/mentat_learn/skill_extractor/argus_submit.py` | 1 | S | Mentat's skills become available to other projects |
| Pull cross-project skills from Argus | [209](209-argus-multi-hop-collaborative-apply-plan.md) | `src/mentat_learn/skill_library/argus_pull.py` | 1 | S | Inherits skills curated by Polaris / Lyra / Atlas |
| **MCP marketplace (Mentat's tool surface)** | | | | | |
| One-click MCP install through Argus trust gate | [205](205-lobehub-collaborative-teammate-platform.md) §2.3, [209](209-argus-multi-hop-collaborative-apply-plan.md) | `src/mentat_learn/tools/mcp/marketplace.py` | 1 | S | Trust-verified tool installation |
| Channel-scoped MCP installs | new | `src/mentat_learn/tools/mcp/channel_scope.py` | 1 | M | Calendar MCP for email channel only |
| **Branching as workflow-alternative UX** | | | | | |
| Branching for "what-if" exploration | [206](206-collaborative-ai-canon-2026.md) §4 | `src/mentat_learn/conversation/branching.py` | 1 | M | "Show me three drafts of this email" |
| Channel-specific branching UX | new | `src/mentat_learn/conversation/branching/{slack,whatsapp,web}.py` | 2 | M | Slack threads = branches; web app = tree view |
| **Voice + multimodal collaboration surfaces** | | | | | |
| `@lobehub/tts` for voice channels | [205](205-lobehub-collaborative-teammate-platform.md) §2.5, [206](206-collaborative-ai-canon-2026.md) §6 | `src/mentat_learn/channels/voice/tts.py` | 1 | M | Voice-channel TTS with EdgeSpeech / Microsoft / OpenAI |
| OpenAI Realtime / Gemini Live for phone | [206](206-collaborative-ai-canon-2026.md) §6 | `src/mentat_learn/channels/voice/realtime.py` | 2 | L | Real-time voice channel |
| Screen-share for desktop-app channel | [206](206-collaborative-ai-canon-2026.md) §6 | `src/mentat_learn/channels/desktop/screen_share.py` | 2 | L | "Show me your calendar" voice-vision interaction |
| **Per-user constitution + local-first** | | | | | |
| Per-user 3-principle constitution | [206](206-collaborative-ai-canon-2026.md) §5, ICAI | `src/mentat_learn/constitution/{model,inject,update}.py` | 1 | M | Stable persona-as-contract |
| Local-first / IndexedDB mode for web channel | [205](205-lobehub-collaborative-teammate-platform.md) §2.4, [206](206-collaborative-ai-canon-2026.md) §7 | `src/mentat_learn/channels/web/local_db.py` | 0 | M | Privacy-by-default for sensitive deployments |
| Confidential-VM deployment for high-stakes users | [206](206-collaborative-ai-canon-2026.md) §7 | deployment-side | 2 | L | Per-tenant SEV-SNP / TDX |
| **Multi-hop substrate (sparingly, only where on-niche)** | | | | | |
| BELLE-style intent router | [202](202-multi-agent-multi-hop-reckoning-2026.md) §1 | `src/mentat_learn/routing/intent_router.py` | 1 | M | Classify "summarise / draft / schedule / answer" per turn |
| Self-Ask externalised chain on multi-step requests | [199](199-multi-hop-reasoning-techniques-arc.md) Phase 1 | `src/mentat_learn/reasoning/self_ask.py` | 1 | S | "Reschedule my Friday meeting and email Bob about it" → 2 explicit hops |
| Decomposition cache | [199](199-multi-hop-reasoning-techniques-arc.md) | `src/mentat_learn/memory/decomposition_cache.py` | 1 | S | Latency cut on recurring multi-step requests |
| **Cross-cutting discipline** | | | | | |
| Pure-function agents | [202](202-multi-agent-multi-hop-reckoning-2026.md) §4 | `src/mentat_learn/agent/pure_function.py` | 0 | S | Replayable trajectories for skill extractor |
| Cross-session productivity benchmark (steps / token) | Mentat design target | `tests/benchmarks/productivity.py` | 0 | S | Verify the ≥30% improvement design target |
| Persona-fidelity benchmark | Mentat design target | `tests/benchmarks/persona_fidelity.py` | 0 | M | Verify the ≥0.9 consistency design target |
| Equal-budget + active-params + TTC curve | [202](202-multi-agent-multi-hop-reckoning-2026.md) | `tests/benchmarks/{equal_budget,active_params,ttc_curve}.py` | 0 | S | Honest benchmark discipline |

## §3 — Tier 0 (days 1-14): the personal-memory + skill-extractor hardening

Mentat-Learn's existing four-layer memory (procedural / facts / session / dialectic) is excellent but lacks a typed *user-identity* layer. Tier 0 adds ICPEA alongside, integrates the skill extractor with argus's curator, and lands the cross-cutting discipline.

### 0.1 — ICPEA Identity + Preference layers alongside SOUL.md

**File.** `src/mentat_learn/memory/icpea/{identity,preference,extractor,injection}.py`

**What.** Adopt the ICPEA schema from [205-lobehub-collaborative-teammate-platform](205-lobehub-collaborative-teammate-platform.md) §2.1, but **compose** rather than replace SemaClaw's SOUL.md persona partition. SOUL.md handles persona shape (the agent's voice, principles); ICPEA Identity handles user shape (who the user is). They sit next to each other in the memory stack.

For Mentat-Learn specifically:
- **I**dentity (user) — "I'm a marketing manager at a Berlin startup; my partner is Sarah; my parents are in Hanoi." Stable; updated rarely.
- **C**ontext — current channel, time-zone, day-of-week, recent message thread. High-volume, FIFO-rotated.
- **P**reference — formal in email, casual in WhatsApp, English with parents but Vietnamese on weekends. Promoted by edit-pattern signal.
- **E**xperience — already partly covered by `memory/dialectic.py`; ICPEA Experience extends with cross-channel "this approach worked" traces.
- **A**ctivity — existing `memory/procedural.py` already covers; ICPEA Activity is a thin wrapper.

**Async extractor.** Triggers on conversation-turn-end (not session-end — Mentat is multi-channel; sessions overlap). Reads the last N turns + outcomes, extracts ICPEA rows, validates, inserts.

**Injection.** Per-channel budget. Email channel injects all five layers (formal, persona-rich); SMS injects only Identity + Preference (terse). Decided by `memory/icpea/injection.py`.

### 0.2 — Per-channel layer access control

**File.** `src/mentat_learn/memory/icpea/access.py`

**What.** Slack channel agent declares which layers it reads (e.g. Identity + Context + Preference, *not* Experience because Slack data shouldn't influence WhatsApp persona). The PermissionBridge from SemaClaw handles enforcement.

### 0.3 — Trajectory-driven skill extractor (extending existing)

**File.** `src/mentat_learn/skill_extractor/extract.py`

**What.** Mentat-Learn already has a skill extractor (Hermes Agent–style). Extend it to:
- Submit promoted skills to argus's curator endpoint (`argus.curator.submit_trajectory`).
- Pull skills from argus that other projects (Polaris, Lyra, Atlas) have promoted.
- Tag promoted skills with ICPEA-derived metadata (who used them; what channel; what tone).

### 0.4 — Pure-function agents

**File.** `src/mentat_learn/agent/pure_function.py`

**What.** Yenugula's pattern ([202-multi-agent-multi-hop-reckoning-2026](202-multi-agent-multi-hop-reckoning-2026.md) §4) — agents are side-effect-free; side-effecting tools (calendar API, email send, payment) go through a gated channel that logs and requires explicit BL-* permission. Critical precondition for the skill extractor (you can't extract a reusable skill from a non-deterministic trajectory).

### 0.5 — Local-first / IndexedDB mode for the web channel

**File.** `src/mentat_learn/channels/web/local_db.py`

**What.** Privacy-by-default for the web channel. Mentat's user is the highest-leverage privacy target in the in-tree project portfolio (cross-channel personal memory). LobeHub's IndexedDB + Dexie pattern ([205](205-lobehub-collaborative-teammate-platform.md) §2.4) is the reference. Optional CRDT sync for multi-device once the local mode is validated.

### 0.6 — Cross-session productivity + persona-fidelity benchmarks

**File.** `tests/benchmarks/{productivity,persona_fidelity}.py`

**What.** Mentat-Learn's two key design targets need explicit benchmarks:
- **Productivity.** Mean steps / token cost for a workflow class drops ≥30% from turn N to turn N+20. Synthesise workflow classes (calendar conflict, email draft, summary, schedule) and replay them at different points in a simulated user's history.
- **Persona fidelity.** User-surveyed "agent felt consistent" score ≥0.9 across ≥5 interactions. Use an LLM-as-judge with a held-out persona-fidelity rubric ([21-llm-as-judge-trajectory-eval](21-llm-as-judge-trajectory-eval.md)).

Run nightly; alarm on regression.

### 0.7 — Equal-budget + active-params + TTC curve

**File.** `tests/benchmarks/{equal_budget,active_params,ttc_curve}.py`

**What.** Cross-cutting discipline (cf. [203](203-polaris-multi-hop-reasoning-apply-plan.md) §3, [208](208-lyra-multi-hop-collaborative-apply-plan.md) §3, [209](209-argus-multi-hop-collaborative-apply-plan.md) §3).

## §4 — Tier 1 (days 15-45): user-aware skills + branching + voice

After Tier 0, Mentat has typed personal memory and a skill extractor that contributes to argus. Tier 1 makes the skills user-aware, exposes branching UX, and adds voice channels.

### 1.1 — ICPEA Experience layer fused with dialectic

**File.** `src/mentat_learn/memory/icpea/experience.py` (composes with `memory/dialectic.py`)

**What.** Mentat-Learn's existing dialectic memory (Honcho-style user model) is essentially the Experience layer. Promote it to a typed ICPEA Experience layer with explicit cross-channel access. The dialectic stays as the "model of the user" representation; Experience is the cross-channel projection.

### 1.2 — Surrogate-verifier gating on skill auto-creation

**File.** `src/mentat_learn/skill_extractor/surrogate_verifier.py`

**What.** Info-isolated surrogate verifier ([169-coevoskills-co-evolutionary-verification](169-coevoskills-co-evolutionary-verification.md)). +30pp ablation lift on skill quality. argus also runs a surrogate verifier (cf. [209](209-argus-multi-hop-collaborative-apply-plan.md) §3.7) — Mentat's runs *first* (filtering before submission), argus's runs *second* (independent verification).

### 1.3 — Per-user constitution

**File.** `src/mentat_learn/constitution/{model,inject,update}.py`

**What.** ICAI / C3AI 3-principle per-user constitution ([206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md) §5). Examples for a Mentat user:
- "Always be terse on SMS; verbose in email."
- "Never schedule on Sundays; ask before scheduling on Saturdays."
- "Default to Vietnamese with parents; English everywhere else."

Constitution is composed with SOUL.md persona; inject in the system prompt at every turn.

### 1.4 — BELLE-style intent router + Self-Ask

**File.** `src/mentat_learn/routing/intent_router.py`, `src/mentat_learn/reasoning/self_ask.py`

**What.** A small classifier that types incoming requests into:
- **Summarise** — short, single-step (no hops needed).
- **Draft** — multi-step (sub-questions about tone, recipient, length).
- **Schedule** — calendar API + multi-step.
- **Answer** — short Q&A.
- **Multi-step compound** — "reschedule my Friday meeting and email Bob about it" — kicks Self-Ask externalised chain.

This is the **only** multi-hop technique that's clearly on-niche for Mentat. Self-Ask materialises the bridge entity (the meeting being rescheduled) so the second hop (emailing Bob) doesn't fall victim to the compositionality gap ([201-compositionality-gap-canon](201-compositionality-gap-canon.md)).

### 1.5 — MCP marketplace through Argus trust gate

**File.** `src/mentat_learn/tools/mcp/marketplace.py`

**What.** Consume `argus.HostAdapter.install_mcp(...)` for tool installation. Channel-scoped: a Calendar MCP installed for the email channel doesn't auto-attach to the WhatsApp channel.

### 1.6 — Voice channel via `@lobehub/tts`

**File.** `src/mentat_learn/channels/voice/tts.py`

**What.** [`@lobehub/tts`](https://www.npmjs.com/package/@lobehub/tts) v5.1.2 React Hooks adapted for Mentat's channel framework. Supports EdgeSpeech / Microsoft / OpenAI TTS-STT; ICPEA Preference layer drives voice choice (formal voice for older users, casual for younger).

### 1.7 — Branching for "what-if" exploration

**File.** `src/mentat_learn/conversation/branching.py`

**What.** "Show me three drafts of this email" → fork at the message, three parallel agent runs, present as branches. Pure-function agents (Tier 0) make this safe.

## §5 — Tier 2 (days 46-90): real-time voice, screen-share, confidential VMs

### 2.1 — OpenAI Realtime / Gemini Live for phone channel

**File.** `src/mentat_learn/channels/voice/realtime.py`

**What.** Phone channel (Twilio + OpenAI Realtime API). Real-time voice with sub-300ms latency. ICPEA Identity + Preference inject into the system prompt; dialectic memory provides per-user persona continuity.

### 2.2 — Screen-share for desktop-app channel

**File.** `src/mentat_learn/channels/desktop/screen_share.py`

**What.** "Show me your calendar" voice-vision interaction. Gemini Live or Qwen3-VL for vision; ICPEA Context layer captures *what the user is currently looking at*.

### 2.3 — Channel-specific branching UX

**File.** `src/mentat_learn/conversation/branching/{slack,whatsapp,web}.py`

**What.** Slack threads = branches (each thread is a fork). WhatsApp = sequential alternatives in the same thread. Web app = tree view (LobeHub-style). Different UX per channel; same underlying branching primitive.

### 2.4 — Confidential-VM deployment for high-stakes users

**File.** Deployment-side (helm chart + cloud-provider manifests).

**What.** SEV-SNP / TDX deployment for users in regulated industries (healthcare, legal, finance). Once Mentat stores cross-channel personal memory + skills + secrets, the threat model is "data in use." Cf. [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md) §7.

## §6 — Five sequencing decisions worth defending

### Decision 1 — ICPEA composes with SOUL.md, not replaces it

SOUL.md handles persona (the agent's voice). ICPEA Identity handles user (who the user is). These are different roles; merging them loses both. SemaClaw's SOUL.md design ([54-semaclaw-general-purpose-agent](54-semaclaw-general-purpose-agent.md)) is preserved verbatim; ICPEA sits next to it.

### Decision 2 — Skill extractor submits to argus, doesn't run an isolated curator

Mentat-Learn's existing skill extractor ([55-hermes-agent-self-improving](55-hermes-agent-self-improving.md)–style) is the trajectory-side. argus owns the curation side ([209-argus-multi-hop-collaborative-apply-plan](209-argus-multi-hop-collaborative-apply-plan.md) §3). Mentat submits trajectories to `argus.curator.submit_trajectory` and pulls promoted skills back via `argus.curator.list_promoted_skills`. This avoids two independent curators that drift apart.

### Decision 3 — Multi-hop techniques sparingly, only on-niche

BELLE intent router and Self-Ask multi-step compound — yes. HippoRAG / Plan-on-Graph / Search-R1 RL — no. Mentat is short-horizon assistant, not deep research. The compositionality-gap fix (Self-Ask externalising bridge entities) is the *only* multi-hop technique that materially helps assistant-class workflows.

### Decision 4 — Local-first is Tier 0, not later

Once Mentat stores cross-channel personal memory, the privacy threat model is severe. IndexedDB mode for the web channel ships in Tier 0 alongside ICPEA, not as a later add-on. Sensitive deployments are blocked otherwise.

### Decision 5 — Voice + screen-share is Tier 2, but only because Tier 0–1 are prerequisite

Voice + screen-share are high-value for Mentat's user-feel, but require ICPEA + per-user constitution + intent router to feel coherent. Without those, voice is just another channel that doesn't know who the user is. Sequence them last so they shine.

## §7 — Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| ICPEA + SOUL.md duplicate / conflict | Med | Med | Explicit composition rules: SOUL.md = persona, ICPEA = user; conflicts logged for review |
| Cross-channel memory leakage (e.g., Slack content in WhatsApp) | Low-Med | High | Per-channel layer access control + PermissionBridge enforcement + ICPEA per-channel-namespace |
| Skill extractor over-promotes noise | Med | Med | Mentat surrogate verifier + argus curator gating + 2-occurrence threshold + held-out eval |
| Voice channel exposes ICPEA Identity in TTS metadata | Low | Med | TTS endpoint sandboxed; metadata stripped; explicit user opt-in for voice channels |
| Local-first IndexedDB exhausts browser storage | Med | Low-Med | Per-layer retention policies; Activity rotated FIFO; user-visible storage dashboard |
| Per-user constitution drifts as user evolves | Med | Low-Med | Versioned constitutions + drift detection at extractor; user prompted to confirm major changes |
| Branching exhausts rate limits on tool APIs | Med | Med | Per-branch budget caps; calendar/email APIs return cached results when possible |
| Confidential VM cost overrun | Low | Med | Optional deployment tier; default users on shared infra; opt-in for SEV-SNP / TDX |
| Realtime voice latency exceeds 1.2s p95 design target | Med | Med | Cache-aware skill loading; ICPEA injection at session-warm; Realtime API tuning |
| Equal-budget benchmark misrepresents multi-channel architecture | High | Low | Report per-channel cost separately; document multi-channel routing in eval reports |
| argus curator outage breaks skill submission | Low | Low | Local skill cache; graceful degradation; retry-with-backoff |

## §8 — Concrete day-by-day Tier 0 checklist

A 14-day Tier 0 plan:

- **Day 1-2** — `memory/icpea/{identity,preference,extractor}.py` skeleton; first cross-channel extraction.
- **Day 3** — `memory/icpea/injection.py` per-channel budget allocation.
- **Day 4** — `memory/icpea/access.py` per-channel layer access control via PermissionBridge.
- **Day 5-6** — `agent/pure_function.py` base class; refactor existing agent loop.
- **Day 7-8** — `skill_extractor/extract.py` extension to submit-to-argus; `skill_extractor/argus_pull.py` reciprocal pull.
- **Day 9** — `channels/web/local_db.py` IndexedDB mode for the web channel.
- **Day 10-11** — `tests/benchmarks/{productivity,persona_fidelity}.py` with synthesised workflow replay.
- **Day 12** — `tests/benchmarks/{equal_budget,active_params,ttc_curve}.py`.
- **Day 13** — First full cross-channel test: Slack → WhatsApp persona consistency check.
- **Day 14** — Tier 0 retro: how many ICPEA rows extracted? how many skills submitted to argus? local-first storage usage? sign-off for Tier 1.

## §9 — How Mentat-Learn becomes the showcase deployment

Once Tier 1 ships, Mentat-Learn is the in-tree project that demonstrates **all six axes of [206-collaborative-ai-canon-2026](206-collaborative-ai-canon-2026.md) at L2+ maturity**:

| Axis | Mentat-Learn after Tier 1 | Maturity |
|---|---|---|
| Personal memory | ICPEA five-layer + SOUL.md + dialectic + procedural + four-layer existing memory | **L3** (frontier) |
| Multi-agent | DAG teams from SemaClaw; cross-channel agent dispatch via gateway | L2 |
| Skill catalog | argus marketplace + auto-creator + cross-project skill commons | **L3** (frontier) |
| Branching UX | Channel-specific branching + Pages-style | L2 |
| Co-evolution | Hermes-style closed loop + argus curator + per-user constitution | **L3** (frontier) |
| Local-first | IndexedDB web channel + per-tenant SEV-SNP at Tier 2 | L2 → L3 at Tier 2 |

After Tier 2, Mentat-Learn clears L3 on every axis. No other in-tree project gets there as cleanly because no other project's *mission* is the collaborative-AI canon.

## §10 — Cross-references

- Mentat-Learn lineage: [52-dive-into-open-claw](52-dive-into-open-claw.md), [54-semaclaw-general-purpose-agent](54-semaclaw-general-purpose-agent.md), [55-hermes-agent-self-improving](55-hermes-agent-self-improving.md).
- Sibling apply plans: [203-polaris-multi-hop-reasoning-apply-plan](203-polaris-multi-hop-reasoning-apply-plan.md), [207-cross-project-collaborative-multi-hop-apply-plan](207-cross-project-collaborative-multi-hop-apply-plan.md), [208-lyra-multi-hop-collaborative-apply-plan](208-lyra-multi-hop-collaborative-apply-plan.md), [209-argus-multi-hop-collaborative-apply-plan](209-argus-multi-hop-collaborative-apply-plan.md).
- Memory canon: [09-memory-files](09-memory-files.md), [184-strongest-memory-techniques-synthesis-may-2026](184-strongest-memory-techniques-synthesis-may-2026.md), [185-memory-integration-playbook](185-memory-integration-playbook.md), [188-witness-provenance-memory-techniques-synthesis](188-witness-provenance-memory-techniques-synthesis.md).
- Skill canon: [04-skills](04-skills.md), [19-voyager-skill-libraries](19-voyager-skill-libraries.md), [167-autoskill-experience-driven-lifelong-learning](167-autoskill-experience-driven-lifelong-learning.md), [171-skill-self-evolution-2026-synthesis](171-skill-self-evolution-2026-synthesis.md), [197-argus-omega-vol-3-recursive-skills-curator](197-argus-omega-vol-3-recursive-skills-curator.md).
- Voice + multimodal: [48-voiceagentrag-dual-agent](48-voiceagentrag-dual-agent.md), [137-voice-agents](137-voice-agents.md), [104-glm-5v-turbo-native-multimodal-agents](104-glm-5v-turbo-native-multimodal-agents.md), [136-multimodal-rag](136-multimodal-rag.md).
- Privacy: [122-explainability-compliance](122-explainability-compliance.md), [125-system-level-production-patterns](125-system-level-production-patterns.md), [147-vendor-lock-in](147-vendor-lock-in.md).

## §11 — One-paragraph summary

Mentat-Learn's collaborative-AI subsystem ships in three tiers: Tier 0 (14 days) hardens the cross-channel personal-memory + skill-extractor surface with ICPEA Identity + Preference + Activity layers (composing with SOUL.md persona) + async extractor + per-channel layer access control + skill extractor submitting to argus + pure-function agents + local-first IndexedDB mode for the web channel + cross-session productivity and persona-fidelity benchmarks; Tier 1 (30 days) makes skills and routing user-aware with ICPEA Experience layer fused with dialectic + surrogate-verifier gating + per-user constitution + BELLE intent router + Self-Ask externalised chain on multi-step compound requests + MCP marketplace through Argus trust gate + `@lobehub/tts` voice channel + branching for "what-if" exploration; Tier 2 (60 days) adds real-time voice (OpenAI Realtime / Gemini Live) + screen-share for desktop + channel-specific branching UX + confidential-VM deployment. The five sequencing decisions defended in §6 turn the collaborative-AI canon into a Mentat-Learn-shaped roadmap that respects SOUL.md persona + PermissionBridge + the existing four-layer memory while levelling Mentat into the showcase L3-maturity deployment of the in-tree project portfolio. Multi-hop QA techniques are sparingly applied — only the BELLE intent router and Self-Ask compositionality-gap fix are clearly on-niche; deeper multi-hop substrate (HippoRAG, Plan-on-Graph, Search-R1) is delegated to Polaris and Lyra.

**The one-line takeaway for harness designers:** Mentat-Learn is the in-tree project where the collaborative-AI canon ships *exactly* as written — make it the showcase deployment by lifting ICPEA + Voyager auto-creation + Argus marketplace + per-user constitution + voice + local-first verbatim, and skip the multi-hop substrate that other projects are better positioned to consume.
