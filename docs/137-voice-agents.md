# 137 — Voice Agents: ASR, TTS, Real-Time Loops, and the Latency-Quality Trade-Off

**Sources.** Kar, *Building Multimodal Generative AI*, Chapter 11 (Voice Input); [48-voiceagentrag-dual-agent](48-voiceagentrag-dual-agent.md) (existing chapter on voice + RAG); the speech-AI literature (Whisper, ElevenLabs, OpenAI Realtime API, Cartesia Sonic, Deepgram); plus the real-time conversation systems pattern (LiveKit, Pipecat, Vapi).

**One-line definition.** A voice agent is a closed-loop architecture — *automatic speech recognition* → *agent* → *text-to-speech* — running in real-time over a streaming audio channel, where the binding constraint is *end-to-end latency* (target < 500ms from end-of-user-speech to first-audio-out) and the architecture trade-off is between full-pipeline latency (modular ASR + LLM + TTS, ~1–3s) and integrated speech-to-speech models (OpenAI Realtime, Gemini Live, ~200–500ms) at the cost of less control.

## Why this matters

Voice is becoming a major agent modality. Customer support, drive-time assistants, accessibility interfaces, in-car systems, smart speakers, healthcare agents — each has voice as the primary or only interface. The architecture for voice agents is structurally different from text agents in one critical way: *latency is user-perceptible at the millisecond level*. A 1-second pause feels broken. A 3-second pause feels frozen.

For agent builders, voice agents are the place where careful architecture matters most for user experience. The latency budget is tight, the failure modes are unforgiving, and the quality of voice output (naturalness, prosody, emotional tone) directly affects user trust. The patterns are mature; getting them wrong is expensive.

This chapter is the architectural playbook: the modular pipeline, the integrated speech-to-speech alternative, the latency budget breakdown, and the patterns that make voice agents shippable.

## Problem it solves

Five concrete voice-agent failures:

1. **Awkward pauses.** End of user speech to start of agent speech > 1 second; user wonders if the system is broken.
2. **Talking over the user.** Agent doesn't detect user interrupting; talks over them.
3. **Robot voice.** TTS sounds artificial; user disengages.
4. **Mishearing.** ASR transcribes wrong words; agent answers a different question.
5. **Long-context degradation.** Multi-turn conversation; agent loses track of what was said earlier.

Each is solved by patterns specific to voice agents.

## Core idea in one paragraph

Two architectural approaches dominate. **Modular pipeline**: streaming ASR (Whisper, Deepgram, Cartesia) → text agent (LLM with tools and memory) → streaming TTS (ElevenLabs, OpenAI Voice, PlayHT, Cartesia Sonic) → user. Latency is the sum: typically 1–3 seconds end-to-end. **Integrated speech-to-speech**: a single model handles ASR + reasoning + TTS in one pass (OpenAI Realtime API, Gemini Live, Sesame). Latency drops to 200–500ms; control over individual components drops too. Both pipelines need *voice activity detection* (when did the user start/stop speaking?), *interruption handling* (user can talk over agent), *streaming output* (agent starts speaking before its full response is generated), and *conversation memory* (multi-turn context). The right choice depends on application: integrated for natural conversation, modular for tool-rich agents that need fine control. Voice agents in 2026 are increasingly hybrid: integrated for the conversational layer, modular for the agentic backend that handles tool calls.

## Mechanism (step by step)

### 1. The modular pipeline — full control, more latency

```text
[user speaks]
   ↓
[microphone → audio stream]
   ↓
[VAD: voice activity detection — when did speech start/end?]
   ↓
[streaming ASR: Whisper, Deepgram, Cartesia Ink]
   ↓ partial transcripts → final transcript
[agent: text LLM + tools + memory]
   ↓ streaming text output
[streaming TTS: ElevenLabs, OpenAI Voice, PlayHT, Cartesia Sonic]
   ↓ audio stream
[speaker]
```

Latency budget (typical):
- VAD detection: 100–300ms
- ASR finalization: 200–500ms (streaming reduces this)
- Agent response: 500–2000ms
- TTS first-audio-out: 100–300ms
- **Total**: 900ms–3s

Reduce by:
- **Streaming ASR**: emit partial transcripts; agent starts thinking before user finishes.
- **Streaming agent output**: TTS starts speaking the first sentence while LLM generates the rest.
- **Streaming TTS**: first audio packet within 100–200ms of input.

With aggressive streaming, modular pipelines can hit 500–800ms end-to-end.

### 2. The integrated speech-to-speech approach

Single model handles audio in → audio out:

```text
[user audio] → [speech-to-speech model] → [agent audio]
```

Examples in 2026:
- **OpenAI Realtime API** (gpt-4o-realtime).
- **Gemini Live** (Gemini 2.0 with realtime).
- **Sesame** (Sesame's multi-modal models).

Latency: 200–500ms. Significantly lower than modular.

Trade-offs:
- **Less component control**: cannot swap individual ASR or TTS.
- **Tool integration is bolted on**: the model handles voice well, agent tools awkwardly.
- **Voice variety**: limited to provider's voice library.
- **Cost**: typically higher per minute than modular.

### 3. Voice activity detection (VAD)

VAD decides when the user is speaking:
- **Energy-based**: simple threshold on audio energy.
- **Spectral**: better; handles background noise.
- **ML-based**: Silero VAD, WebRTC VAD; near-perfect.

For interruption handling: VAD also detects user speech *while agent is speaking* — triggers stop-TTS-and-listen.

### 4. Streaming ASR

Modern ASR APIs emit partial transcripts every 100–300ms:

```text
t=0.5s:  "I want"
t=1.0s:  "I want to know"
t=1.5s:  "I want to know about" (partial)
t=2.0s:  "I want to know about the policy" (final)
```

The agent can start preparing on the partial; finalise when the complete transcript is in. Saves 500ms–1s on each turn.

### 5. Streaming TTS

TTS engines that emit audio packets as soon as they're generated, not after the whole text:
- ElevenLabs streaming, OpenAI streaming, PlayHT streaming, Cartesia streaming.
- First-audio-out: 100–200ms.

The agent's text output can be synthesised sentence-by-sentence; user hears the first sentence while later sentences are still being generated.

### 6. Interruption handling

Pattern:

```text
agent speaking
   ↓ (VAD detects user voice mid-utterance)
[stop TTS immediately]
[reset audio buffer]
[ASR begins on new user input]
[agent processes interruption — usually as new turn]
[respond appropriately]
```

Critical: stopping TTS must be < 100ms from user voice detection. Slow stops feel like the agent is talking over the user.

### 7. Conversation memory

Multi-turn voice conversations need:
- **Conversation history**: text transcripts of all turns + agent responses.
- **Audio archive**: optionally, original audio for post-call analysis.
- **Long-term memory**: across calls (user preferences, past issues). See [09-memory-files](09-memory-files.md), [107-memento-cbr-memory](107-memento-cbr-memory.md).

Memory size grows with conversation length; summarise older turns ([08-context-compaction](08-context-compaction.md)) to stay within context.

### 8. Voice quality choices

TTS voices vary in:
- **Naturalness**: human-like vs synthetic-sounding.
- **Prosody**: pitch, rhythm, emphasis.
- **Emotional range**: monotone vs expressive.
- **Accent + language**: per market.
- **Cost**: $0.10–$0.30 per minute of audio.

For brand-sensitive applications, custom voice cloning is available (ElevenLabs Professional Voice Cloning).

### 9. Tool integration in voice agents

Voice agents often need tools (lookup, transactions, scheduling):
- Modular: easy — agent is a text LLM, tools work normally.
- Integrated speech-to-speech: harder — tool calls add latency that breaks the natural conversation flow.

For tool-heavy voice agents: **modular pipeline + aggressive streaming** is the dominant pattern.

### 10. The hybrid architecture

```text
[user voice]
   ↓
[realtime model: handles greeting, simple Q&A, conversation flow]
   ↓ on tool need
[hand off to modular agent: text LLM + tools]
   ↓ tool result
[back to realtime for spoken response]
```

Two models, smooth transition. Latency low for most turns; tools handled cleanly.

## Empirical anchors

- **Sub-1-second latency** is the threshold below which conversation feels natural.
- **Modular pipelines** with aggressive streaming hit 500–800ms.
- **Integrated speech-to-speech** hits 200–500ms.
- **Cost**: typical voice agent at $0.05–$0.30 per minute total (ASR + LLM + TTS).
- **Adoption**: customer support and assistive tech are leading; in-car and smart speakers next.
- **TTS quality** has reached human-indistinguishable for short utterances; longer passages still have tells.

## Variants and counter-arguments addressed

- **"Just use the realtime API."** Easy for simple agents; hard for tool-heavy ones.
- **"Latency under 500ms isn't critical."** It is for natural conversation.
- **"Voice is a niche."** It's a major modality for many use cases (support, accessibility, mobile, in-car).
- **"Whisper is enough."** Whisper is excellent for transcription; for streaming with low latency, streaming-optimised engines are better.
- **"TTS quality is good enough."** It is for many use cases; brand-sensitive needs custom voices.

## Failure modes and limitations

1. **Latency spikes.** Network or API slow; conversation feels broken. Need timeouts and fallbacks.
2. **Mishearing in noisy environments.** ASR fails; agent answers wrong question.
3. **Interrupt-handling lag.** Agent talks over user.
4. **Long-context degradation in multi-turn.** Agent forgets earlier turns.
5. **Cost surprise.** Voice is expensive per minute; long calls add up.
6. **Privacy concerns.** Audio is more sensitive than text; storage and transmission need care.
7. **Accent / dialect bias.** ASR works better for some accents; equity issue.
8. **Hallucinations in voice.** Same problem as text but harder to verify ("did you say X?" requires the user to check).

## When to use, when not

**Use voice agents for** customer support, assistive tech, in-car, mobile, smart speakers, accessibility, hands-free interfaces.

**Modular for** tool-rich agents (transactions, lookups, complex actions).

**Integrated for** conversational-first agents (companions, simple Q&A, brand voices).

**Hybrid for** the best of both.

**Skip voice for** workflows that benefit from visual interfaces (rich data, comparisons, navigation).

## Implications for harness engineering

- **Latency budget is the foundational constraint.** Profile end-to-end; optimise the longest segment.
- **Streaming everywhere.** ASR, agent, TTS — all streaming.
- **VAD as a first-class component.** With interruption handling.
- **Conversation memory in the harness.** Per-call short-term + cross-call long-term.
- **Tool integration carefully.** Voice agents that need many tool calls have to mask tool latency with conversational filler ("Let me check that...").
- **Voice as a UX surface.** [143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md) covers UX more broadly; voice has its own affordances.
- **Audio archive with consent.** Privacy-sensitive; retention policies; compliance ([122-explainability-compliance](122-explainability-compliance.md)).
- **Multi-tenant voice infra.** Different tenants different voices; cost attribution.
- **Eval includes audio quality.** Word error rate, mean opinion score for TTS, end-to-end latency.

The one-sentence takeaway: **voice agents are real-time closed loops where latency is the binding constraint — modular pipelines with streaming hit 500–800ms, integrated speech-to-speech hits 200–500ms, hybrid combines both — and tool-rich agents stay modular, conversational agents go integrated.**

## See also

- [48-voiceagentrag-dual-agent](48-voiceagentrag-dual-agent.md) — voice + RAG architecture.
- [136-multimodal-rag](136-multimodal-rag.md) — multimodal retrieval; voice is one modality.
- [104-glm-5v-turbo-native-multimodal-agents](104-glm-5v-turbo-native-multimodal-agents.md) — native multimodal foundation models.
- [110-transformer-llm-architecture-for-agent-builders](110-transformer-llm-architecture-for-agent-builders.md) — latency considerations.
- [08-context-compaction](08-context-compaction.md) — long conversation memory.
- [143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md) — voice UX patterns.
- [122-explainability-compliance](122-explainability-compliance.md) — audio retention and consent.
- [149-sector-use-case-catalog](149-sector-use-case-catalog.md) — voice in customer support and other sectors.
