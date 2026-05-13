# 302 — Multimodal Agents 2026: vision, audio, video, multi-modal multi-agent teams

**Anchors.** Claude 3.5/3.7 Sonnet vision, GPT-4o multi-modal, Gemini 2.5 multi-modal, Llama-4-Vision, Qwen-2.5-VL. Voice: OpenAI Realtime, Anthropic voice mode (2025), ElevenLabs TTS. Video: Sora-2, Veo-3, Runway-Gen-4, Wan-2 (open-weights). Companions: [105-agentic-world-modeling-visual-generation](105-agentic-world-modeling-visual-generation.md), [137-voice-agents](137-voice-agents.md), [288-harmony-voice-seven-layer-stack-apply-plan](288-harmony-voice-seven-layer-stack-apply-plan.md), [192-world-r1-3d-constraints-t2v](192-world-r1-3d-constraints-t2v.md), [136-multimodal-rag](136-multimodal-rag.md).

**One-line definition.** A 2026 picture of **multimodal agents** beyond the text-centric corpus — covering **vision agents** (UI automation, document understanding, visual diagnostics), **voice agents** ([288](288-harmony-voice-seven-layer-stack-apply-plan.md), Realtime API + handoff to text specialists, latency-bound), **video-input agents** (security analysis, content moderation, sports / surveillance), **video-generation agents** (storyboarding, training-data synthesis), and **multi-modal multi-agent teams** (vision specialist + text specialist + voice front-end coordinated via [251](251-multi-agent-teams-2026-synthesis.md) lead-and-spokes) — emphasizing the **modality-specific failure modes** (image-injection attacks, audio-transcription bias, video-temporal-coherence collapse), **cost economics** (vision tokens are expensive — 1024 tokens per image for high-res), and the **convergence with the seven-layer stack** (multimodal agents adopt the same protocols, runtimes, ops, security with modality-specific overlays).

## Why multimodal matters in 2026

Through 2024 the agent corpus was text-centric. By 2026 frontier models are **natively multimodal** (Claude 3.5+, GPT-4o, Gemini 2.5) and production deployments increasingly require multimodal agents: UI automation (read the screen, click buttons), document understanding (PDF + images + tables), security analysis (video surveillance + log correlation), content moderation (image + text + audio), voice agents with screen-share, and video-generation agents for training-data synthesis. The 250–296 stack adopts naturally with **modality-specific overlays** for capability, cost, security, and UX.

## Modality-specific characteristics

| Modality | Token cost | Latency | Failure modes |
|---|---|---|---|
| Vision (image) | ~1024 tok/image (high-res) | +200-500ms first-token | OCR errors, UI element mis-identification, image injection |
| Voice (audio) | ~50 tok/sec speech | <300ms TTS first-token | Transcription bias, accent drift, audio injection |
| Video | ~1024 tok/frame × frames | seconds for short clips | Temporal coherence collapse, frame-skip artifacts |
| Generation (image) | output tokens | 2-30s | Prompt injection in description, deepfake risk |
| Generation (video) | output tokens (massive) | 10s-min | Coherence, copyright, deepfake risk |

## Vision agents

**Use cases:** UI automation (Anthropic Computer Use, OpenAI Operator), document understanding (PDFs with figures + tables), visual diagnostics (medical imaging — covered by [287-helix-bio](287-helix-bio-seven-layer-stack-apply-plan.md)), screen-aware coding agents.

**Architecture:**

- Vision-capable base model (Claude 3.7, GPT-4o, Gemini 2.5, Llama-4-Vision).
- Image-aware MCP servers (screen-capture, OCR, document-parse).
- Spotlight prompting on image-extracted content ([269-prompt-injection-2026](269-prompt-injection-2026.md)) — prompt injection via image text is a documented attack vector.
- Cost router: vision tokens 5-10× cheaper-per-action models when applicable.

**Production case:** Anthropic Computer Use as the canonical UI-automation reference; Operator for browsing.

## Voice agents

**Already covered in [288](288-harmony-voice-seven-layer-stack-apply-plan.md).** Production architecture:

- OpenAI Realtime API + Agents SDK for the voice agent.
- Handoff to text specialists via `transfer_to_*` tool.
- Audio classifier at transcription boundary for prompt-injection defense.
- Latency-bound: P99 first-token <300ms.

**Production case:** Harmony-Voice + Mentat-Learn integration.

## Video-input agents

**Use cases:** security surveillance, sports analysis, content moderation, training-data extraction.

**Architecture:**

- Video-capable model (Gemini 2.5, GPT-4o video, Claude 3.7 video — frame-sampled).
- Frame-extraction + temporal-coordination tools.
- Cost-router decision: video is expensive; sample sparsely + escalate to dense on signal.

**Failure mode:** temporal-coherence collapse on long clips; mitigation = sliding-window analysis with explicit cross-window summarization.

## Video-generation agents

**Use cases:** storyboarding, training-data synthesis (per [192-world-r1-3d-constraints-t2v](192-world-r1-3d-constraints-t2v.md)), explainer generation, marketing content.

**Architecture:**

- Sora-2 / Veo-3 / Runway-Gen-4 / Wan-2 as generation backend.
- Verifier on output: 3D-coherence-based verifier per [192](192-world-r1-3d-constraints-t2v.md), copyright/deepfake classifier.
- Bright-line gates: `PUBLISH_GENERATED_VIDEO` always escalates due to deepfake risk.

## Multi-modal multi-agent teams

The natural extension of [251-multi-agent-teams-2026-synthesis](251-multi-agent-teams-2026-synthesis.md):

```
Lead (text)
├── Vision specialist (image / video analysis)
├── Voice specialist (audio in/out)
├── Code specialist (text)
├── Citation verifier (text)
└── Cross-channel verifier (different family)
```

Each specialist publishes A2A capability ([254](254-a2a-protocol-deep-dive.md)) with modality-specific input/output schemas in OASF ([255](255-agntcy-oasf-acp-deep-dive.md)).

**MAST cluster-2 risks** ([251](251-multi-agent-teams-2026-synthesis.md)) compound at multimodal boundaries — context-overwrite when video specialist produces large summaries; mitigation = structured artifacts + memory-tier-isolation.

## Multimodal seven-layer stack overlay

| Layer | Modality-specific overlay |
|---|---|
| L2 Capability | vision / voice / video tokens have different cost-per-task; routing matters more |
| L3 Protocol | OASF schemas declare input/output MIME types; A2A artifacts carry inline + URL data |
| L4 Runtime | hybrid (Realtime for voice, LangGraph for state-machine, Agents SDK for handoff) |
| L5 Security | image-injection + audio-injection + deepfake classifiers added to defense |
| L6 Operations | per-modality eval (visual-Q&A, voice-transcription, video-coherence) |
| L7 Compliance | EU AI Act limited risk for chatbots, high-risk for biometric / medical imagery |

## Production-grade multimodal patterns

- **Modality-specific MCP servers** — one MCP per modality keeps cost-routing clean.
- **Per-modality memory tier** — episodic memory for video; semantic for OCR text.
- **Bright-line gates per modality** — voice `VOICE_TRIGGERED_PURCHASE`, video `PUBLISH_GENERATED_VIDEO`, image `PUBLISH_GENERATED_IMAGE_OF_PERSON`.
- **Cost-router with modality tiers** — vision-heavy task → vision-capable cheap-tier; text → text-tier.
- **Cross-channel verifier across modalities** — text-based verifier checks vision-agent output for hallucination.

## Cost economics 2026 (per-modality, $/M tokens)

| Tier | Text input | Text output | Image input | Audio input | Video input |
|---|---:|---:|---:|---:|---:|
| Frontier | $3-15 | $15-75 | $5-20 | $0.01-0.10/sec | $1-5/min |
| Mid | $0.30-3 | $1.10-15 | $1-5 | $0.005-0.03/sec | $0.50-3/min |
| OSS-distilled local | $0 | $0 | $0 | $0 | $0 (where supported) |

Vision is roughly 5-10× more expensive than text per useful action; route accordingly.

## Failure modes (multimodal-specific)

- **Image prompt injection.** Hidden text in images that the vision model reads as instruction. Mitigation: spotlight + image-text-classifier.
- **Audio prompt injection.** TTS output containing instructions; transcription-stage attack.
- **Video temporal collapse.** Long clips → loss of cross-frame coherence; sliding-window mitigation.
- **Deepfake / copyright.** Generation output infringes; verifier + bright-line gate.
- **Modality drift.** Training distribution lacks production diversity; production-eval feedback loop.
- **Multimodal trust calibration.** Visual evidence over-trusted vs text claims; explicit cross-modal verification.

## When multimodal vs text-only

**Multimodal** when the task intrinsically requires non-text input (UI automation, document understanding, voice interface, video analysis). **Text-only** for text-domain tasks even if input has incidental images (often more cost-effective to OCR + text-pipeline than vision-pipeline). **Hybrid** for most production cases (vision-agent for screen reading + text agent for reasoning).

## One-line takeaway

**Multimodal agents in 2026 adopt the seven-layer stack with modality-specific overlays — vision (1024 tokens/image, image-injection defense, UI-automation use cases), voice (OpenAI Realtime + Agents SDK + handoff per [288](288-harmony-voice-seven-layer-stack-apply-plan.md), latency-bound), video (input for surveillance / sports, generation with deepfake bright-line gates) — composed via multi-modal multi-agent teams (lead + vision + voice + text specialists) with per-modality cost routing, modality-specific MCP servers, and cross-channel verifier across modalities; the architectural lesson: **modality is an axis on every layer of the stack**, not a separate stack.**
