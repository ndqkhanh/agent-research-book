# 288 — Harmony-Voice × Seven-Layer Stack Apply Plan 2026

**Anchors.** Harmony-Voice — voice-first agent ([projects/harmony-voice](../projects/harmony-voice/)). Companion: [137-voice-agents](137-voice-agents.md), [48-voiceagentrag-dual-agent](48-voiceagentrag-dual-agent.md), [77-meta-tts-agentic-coding](77-meta-tts-agentic-coding.md), [260-openai-agents-sdk-deep-dive](260-openai-agents-sdk-deep-dive.md).

**One-line definition.** A **per-layer apply plan** for Harmony-Voice — the **voice-first agent** with Realtime API integration and handoff to text specialists — emphasizing the **latency-bound architecture** (sub-second response is non-negotiable), **OpenAI Agents SDK + Realtime** as the natural runtime ([260](260-openai-agents-sdk-deep-dive.md)), the **voice + text composition pattern** ([277-agent-ux-patterns-2026](277-agent-ux-patterns-2026.md)) where voice agent triages and hands off to text specialists for complex tasks, and the **transcription-injection threat surface** that requires audio-aware classifiers — staged across four 90-day phases.

## Per-layer plan

### L1 Foundation
Standard. Strict Permission Bridge for voice-triggered consequential actions (must bright-line on send-message, transfer-money). Daemon for cross-session voice context.

### L2 Capability
**Pretraining**: low-latency optimized (Haiku / GPT-4o-mini class for real-time; thinking models for handoff specialists). **TTC**: minimal in voice agent; full in handoff target. **Trajectory**: short in voice; long in text specialists. **Multi-agent**: voice agent + text specialists (handoff pattern).

### L3 Protocol
- **MCP**: voice-tools (calendar, contact lookup, alarms, navigation).
- **A2A**: voice agent invokes text specialists via A2A handoff.
- **AGNTCY**: published OASF.
- **Tailscale + NATS**: cross-device voice continuity.
- **SKILL.md**: voice skills with audio-content awareness.
- **Routines**: scheduled voice notifications / reminders.

### L4 Runtime
**OpenAI Agents SDK + Realtime API** ([260](260-openai-agents-sdk-deep-dive.md)) is the natural fit — voice agent with `transfer_to_text_specialist` as a first-class handoff. Sub-100ms latency requirement constrains other choices.

### L5 Security
- **Prompt injection**: voice-specific attacks (audio-injection via TTS, transcription-bias attacks). Audio classifier at transcription boundary.
- **Supply chain**: vendored voice skills primary.
- **Isolation**: per-session container; egress allowlist (voice-tool APIs only).
- **Bright-line**: `VOICE_TRIGGERED_PURCHASE`, `VOICE_TRIGGERED_MESSAGE_EXTERNAL`, `VOICE_TRIGGERED_FILE_DELETE` — confirmation required.

### L6 Operations
- **Observability**: per-utterance spans + per-handoff trace; latency P99 critical.
- **Eval**: voice transcription accuracy, intent-classification accuracy, handoff success.
- **Durability**: session state in fast K/V; longer trajectories in handoff specialists.
- **SRE**: P99 first-token latency < 300ms; P99 end-to-end < 2s.

### L7 Compliance
- **GDPR**: voice recordings are personal data; retention policy explicit.
- **EU AI Act**: limited risk (voice assistant); transparency obligation (must disclose AI).
- **HIPAA / financial** if used in regulated voice channels.

## Phased rollout

| Phase | Scope | Duration |
|---|---|---|
| **P1** | L1-L2 + Realtime integration + handoff prototype | 90 days |
| **P2** | L3-L4 (Agents SDK) + L5 Security (audio-aware) | 90 days |
| **P3** | L6 Operations (latency-bound SLO) | 90 days |
| **P4** | L7 Compliance + multi-channel polish + Mentat-Learn integration | 90 days |

## One-line takeaway

**Harmony-Voice adopts the seven-layer stack with **latency-bound voice-first architecture** — OpenAI Agents SDK + Realtime as the natural runtime, sub-300ms first-token P99 as load-bearing SLO, voice + text handoff as the canonical pattern (voice triages, text specialists do heavy lifting), audio-aware classifiers at transcription boundary, and short voice trajectories with long handoff specialist trajectories; the cross-channel handoff pattern from [277](277-agent-ux-patterns-2026.md) is the architectural primitive that makes voice agents production-grade.**
