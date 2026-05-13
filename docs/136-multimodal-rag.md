# 136 — Multimodal RAG Architectures: Image, Audio, Video, Document as First-Class Indexed Content

**Sources.** Kar, *Building Multimodal Generative AI*, Chapters 7–8 (Bidirectional Multimodal Retrieval, Multimodal RAG Systems); plus the multimodal-embeddings literature (CLIP, CLAP, ImageBind, BLIP-2), multimodal index libraries (LlamaIndex multimodal, Weaviate multimodal), and Gemini's multimodal RAG integration patterns.

**One-line definition.** Multimodal RAG generalises chunked-text retrieval to *non-text content*: index images by visual embeddings, audio by acoustic + transcript embeddings, video by frame + transcript + scene embeddings, structured documents by layout-aware embeddings — and retrieve across modalities so a text query returns matching images, an image query returns matching documents, and the agent's planner consumes a heterogeneous context of text + image + audio + video chunks; the same indexing portfolio ([134-semantic-indexing](134-semantic-indexing.md)) and trustworthy-generation stack ([135-trustworthy-generation](135-trustworthy-generation.md)) extend, with new patterns for cross-modal embedding alignment and citation surfaces.

## Why this matters

In 2024 most RAG was text-only. By 2026 a meaningful fraction of enterprise content is non-text: PDFs with figures, technical drawings, scanned documents, recorded meetings, screencasts, product images. An agent that retrieves only text from these sources misses the load-bearing content. A research agent that can't read a chart, a customer-support agent that can't see a product image, a legal agent that can't reference a contract clause's signature page — each is structurally limited.

For agent builders, multimodal RAG is the extension that turns text-only RAG into a substrate that handles real-world enterprise content. The engineering cost is real (multi-modal embedding pipelines, modality-aware retrieval, layout-aware document parsing) but the capability lift is qualitatively new — there are queries you simply cannot answer without it.

This chapter is the architecture of multimodal RAG as it stands in 2026: the embedding choices, the retrieval patterns, the citation surfaces, and the failure modes specific to non-text content.

## Problem it solves

Six concrete capabilities multimodal RAG enables:

1. **"Show me figures about X"**: image retrieval from a corpus.
2. **"Where in this video did the speaker discuss Y?"**: timestamp-targeted retrieval.
3. **"Find the diagram in this PDF that depicts the data flow"**: layout-aware figure extraction.
4. **"Compare these two product photos for differences"**: image-to-image retrieval.
5. **"Summarise the architecture across the design docs and screenshots"**: heterogeneous synthesis.
6. **"What did the witness say at minute 23 of the deposition video?"**: aligned audio + transcript retrieval.

Each requires non-text indexing and cross-modal retrieval.

## Core idea in one paragraph

Embed every modality into a *common semantic space* (or aligned spaces) so cross-modal retrieval is dot-product computable. **Images** with vision-language models (CLIP family, SigLIP, OpenAI text-embedding-3 with image encoder, Gemini embeddings). **Audio** as both raw acoustic embeddings (CLAP, Whisper-derived) *and* transcript embeddings (text-side of the multimodal pair). **Video** as frame embeddings plus transcript embeddings plus scene embeddings (with timestamps). **Documents** with layout-aware parsing: text + figures + tables + their spatial relationships. Index each modality in a separate vector store *or* in a unified index, with metadata tagging the modality. At query time, embed the query (text or image or audio) and retrieve from the relevant indexes; combine results across modalities through hybrid fusion ([129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md), [134-semantic-indexing](134-semantic-indexing.md)). The synthesiser LLM (multimodal-capable: Gemini, GPT-4V, Claude with vision) consumes the heterogeneous context and produces an answer with cross-modal citations. The cost is multi-modality pipelines; the capability is qualitatively new.

## Mechanism (step by step)

### 1. The multimodal embedding choice

For text-image alignment:
- **CLIP** (Radford et al. 2021) — the original; OpenCLIP for open-source.
- **SigLIP** — better contrastive training; widely used in 2026.
- **Provider embeddings** — OpenAI's `text-embedding-3` with image variant; Gemini's multimodal embeddings.

For audio-text alignment:
- **CLAP** (Wu et al. 2023) — audio-text contrastive.
- **Whisper-derived** — transcribe with Whisper; embed transcript with text encoder; index transcript with timestamp metadata.

For video:
- Frame-level embeddings (image embedder per sampled frame).
- Transcript embeddings (audio→text→embedding).
- Scene embeddings (whole-clip representations).

For documents:
- **Layout-aware embeddings**: LayoutLMv3, Donut, ColPali. Embed (text + position + visual features).
- **Hybrid**: text chunks + figure embeddings + table embeddings.

### 2. Image-RAG architecture

```text
[image corpus]
   ↓ (per image)
[VLM embedder: CLIP / SigLIP]
   ↓
[vector index, metadata: source, caption, alt-text]
   ↓
[query: text or image]
   ↓
[query embedded with same/aligned encoder]
   ↓
[similarity search → top-K images]
   ↓
[returned with metadata + caption]
```

For *captioning gap* (images without text descriptions): generate captions with a VLM at ingest time; index the caption alongside the image embedding. Improves text→image retrieval.

### 3. Audio-RAG architecture

```text
[audio file]
   ↓
[transcribe with Whisper]
   ↓ (transcript with timestamps)
[chunk transcript by sentence]
   ↓
[per chunk: text embedding + timestamp metadata]
   ↓
[vector index]
```

At query time:
- Embed query text.
- Retrieve top-K transcript chunks.
- Each result has timestamp; agent can reference "at 02:34 the speaker said...".

For audio without speech (music, sound effects): use CLAP for acoustic-text retrieval.

### 4. Video-RAG architecture

```text
[video file]
   ↓
   ├──→ [frame sampling: 1 frame/sec or scene-change-driven]
   │       ↓ frame embeddings + timestamps
   ├──→ [audio extraction → Whisper transcript + timestamps]
   │       ↓ transcript chunks + embeddings
   └──→ [scene clustering: group frames into scenes]
           ↓ scene-level embeddings
   ↓
[multi-index storage]
   - frame index
   - transcript index
   - scene index
```

Query routes to the right index(es); fusion combines.

### 5. Document-RAG with layout

PDFs and scanned documents have spatial structure that text-only embedding loses:

```text
[PDF page]
   ↓ [layout parser: extract text blocks, figures, tables, with bounding boxes]
   ↓
[per element]
   ├── text block: text embedding + position + page
   ├── figure: image embedding + caption (auto-generated) + page
   └── table: structured representation (Markdown/HTML) + embedding
   ↓
[vector index with element metadata]
```

ColPali and similar layout-aware embedders simplify this by embedding each PDF page as a high-dimensional representation that captures both text and visual layout.

### 6. Cross-modal retrieval routing

A query can be text, image, or audio:

```text
[query]
   ↓
[detect modality]
   ↓
[embed with the right encoder]
   ↓
[search aligned indexes]
   - text query → text + image (via CLIP) + audio transcript indexes
   - image query → image index + text-via-caption index
   - audio query → audio index + transcript index
   ↓
[fuse, rerank, return]
```

The multi-modal nature is hidden from the agent; it sees a unified retrieval API.

### 7. Heterogeneous context to the LLM

The retrieved context has multiple types:
- Text chunks.
- Image references with captions and source.
- Audio transcript snippets with timestamps.
- Video frame references with timestamps.
- Document elements with page+position.

The synthesiser LLM (multimodal: Gemini, GPT-4V, Claude with vision) consumes this:

```text
Context:
[text-1] {chunk text} [source: doc-A:para-3]
[image-2] <embedded image> Caption: {caption} [source: doc-B:fig-1]
[audio-3] {transcript chunk} [source: video-C @ 02:34]
[doc-elem-4] <embedded image of PDF page> [source: doc-D:p.5]

Question: ...
```

The LLM cites with cross-modal references: "[image-2] shows the architecture, with the data flow described in [text-1]."

### 8. Citation surfaces

User-facing UI for multimodal citations:
- Text citation: clickable footnote → source paragraph.
- Image citation: clickable thumbnail → full image + source.
- Audio citation: clickable timestamp → audio plays at that point.
- Video citation: clickable thumbnail → video plays at timestamp.
- Document citation: clickable page → PDF opens at page+position.

[143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md) covers the broader UX patterns.

### 9. Multimodal eval

Per-modality recall@K, plus cross-modal:
- Text query → image retrieval.
- Image query → text retrieval.
- Multi-hop across modalities.

Eval set requires multi-modal labelled data — expensive to curate; smaller eval sets are typical for multi-modal than for text.

### 10. Operational considerations

- **Storage**: multi-modal indexes are 5–10× the storage of text-only.
- **Compute**: embedding non-text content is more expensive (GPU-bound for image, video).
- **Latency**: image embedding 50–200ms per image; OK at ingest, problematic at query if embedding live image queries.
- **Versioning**: embedding model swaps require re-embedding; even more painful at multi-modal scale.

## Empirical anchors

- **Multimodal RAG** lifts retrieval recall significantly on documents with figures (charts, technical drawings).
- **CLIP / SigLIP** are the dominant 2026 image-text embedders for production.
- **ColPali** and layout-aware embedders are standard for PDF-heavy corpora.
- **Whisper** is the de-facto audio transcription standard.
- **Adoption** is growing fast; in 2026 enterprise agent platforms commonly include image and document multimodal RAG; audio and video less so but expanding.
- **Storage cost** is 5–10× text-only.

## Variants and counter-arguments addressed

- **"Just OCR images and embed text."** OCR loses figure semantics; visual embedders preserve them.
- **"Multimodal LLMs handle this without RAG."** They do for in-context content; for large corpora, retrieval is still needed.
- **"Provider RAG handles multi-modal."** Increasingly yes; the principles still apply.
- **"Layout-aware parsing is too expensive."** Cost has dropped; ColPali is tractable; benefits outweigh.
- **"Cross-modal retrieval is unreliable."** Quality has improved sharply; 2024 was rough, 2026 production-grade.

## Failure modes and limitations

1. **Embedding-model misalignment.** Different image and text encoders not pretrained jointly; cross-modal retrieval fails.
2. **OCR errors in scanned documents.** Layout-aware parsers help; not perfect.
3. **Caption-generation drift.** Auto-generated captions miss key details; image retrieval suffers.
4. **Temporal ambiguity in video.** Frame-level retrieval returns a moment; question is about a sequence.
5. **Audio with overlapping speakers.** Transcripts collapse into one; speaker diarisation needed.
6. **Multi-modal eval gap.** Hard to curate eval sets; quality measurements lag text-only.
7. **Storage cost.** Image and video indexes can dwarf text storage.
8. **Cross-modal hallucination.** LLM synthesises across modalities incorrectly; verifier needs cross-modal reasoning capability.

## When to use, when not

**Use multimodal RAG when** corpus contains substantial non-text content (PDFs with figures, recordings, images, videos) and queries refer to that content.

**Skip for** text-only corpora.

**Start with documents and images**; add audio and video only when the demand justifies the engineering.

**Use provider multimodal RAG offerings** (Gemini, OpenAI Assistants, Claude) for prototypes; build custom for production at scale.

## Implications for harness engineering

- **Multi-modal embedding pipeline as platform infra.** Multiple embedders; modality-aware ingest.
- **Layout-aware document parsing.** ColPali or equivalent for any PDF-heavy corpus.
- **Whisper transcription pipeline** for any audio/video.
- **Cross-modal index governance.** Versioning, re-embedding plans, blue/green for major embedder swaps.
- **Multimodal LLM at synthesis.** Choose synth model with strong vision support.
- **Citation surfaces in the UI.** Clickable cross-modal links; user trust depends on it.
- **Multi-modal eval set.** Even if smaller than text eval; some signal beats none.
- **Storage budget.** Multi-modal indexes are 5–10× text storage; tier and compress.
- **CDC for multi-modal.** [132-vector-cdc-pipelines](132-vector-cdc-pipelines.md) extended to multi-modal events.

The one-sentence takeaway: **multimodal RAG extends text-RAG to images, audio, video, and structured documents through aligned embeddings, modality-aware indexing, and cross-modal citation — the engineering cost is real but the capability gain is qualitatively new for any corpus that isn't text-only.**

## See also

- [25-agentic-rag](25-agentic-rag.md) — agentic RAG.
- [104-glm-5v-turbo-native-multimodal-agents](104-glm-5v-turbo-native-multimodal-agents.md) — native multimodal agent foundation model.
- [105-agentic-world-modeling-visual-generation](105-agentic-world-modeling-visual-generation.md) — multimodal generation frontier.
- [120-rag-to-finetuning-spectrum](120-rag-to-finetuning-spectrum.md) — RAG positioning.
- [129-kg-rag-hybrid-retrieval](129-kg-rag-hybrid-retrieval.md), [133-reranking-for-agentic-retrieval](133-reranking-for-agentic-retrieval.md), [134-semantic-indexing](134-semantic-indexing.md) — retrieval foundation.
- [135-trustworthy-generation](135-trustworthy-generation.md) — citation across modalities.
- [137-voice-agents](137-voice-agents.md), [138-text-to-sql-agents](138-text-to-sql-agents.md), [139-ocr-document-agents](139-ocr-document-agents.md) — adjacent specialised patterns.
- [48-voiceagentrag-dual-agent](48-voiceagentrag-dual-agent.md) — voice+RAG architecture.
