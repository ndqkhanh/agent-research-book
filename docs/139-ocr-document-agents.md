# 139 — OCR & Text-from-Images for Document Agents: Layout-Aware Extraction, Table Parsing, Form Understanding

**Sources.** Kar, *Building Multimodal Generative AI*, Chapter 16 (GenAI for Extracting Text from Images); the document-AI literature (LayoutLMv3, Donut, Pix2Struct, ColPali, Tesseract); plus modern document-understanding services (Chunkr, Unstructured, Azure Document Intelligence, AWS Textract, Google Document AI).

**One-line definition.** Document-extraction agents convert *unstructured* document inputs (scanned PDFs, photos of documents, complex layouts) into *structured* outputs (text, tables, forms, key-value pairs) using a stack of *layout-aware OCR*, *structure parsing*, and *LLM-driven semantic extraction* — with the production reality being that no single tool is sufficient and the right architecture composes specialised parsers (Tesseract for clean text, LayoutLM for layout, Donut for end-to-end, Vision LLMs for fallback) into a pipeline that each contribute their best.

## Why this matters

Documents are the workhorse of enterprise data. Invoices, contracts, medical records, identity documents, scientific papers, legal filings, insurance claims — all are documents. A document agent that can reliably extract structured data from them unlocks automation across many domains. A document agent that fails on the long tail (handwritten notes, low-quality scans, foreign-language scans, complex tables, multi-page forms) breaks the workflow that depends on it.

For agent builders, document extraction is the substrate under almost every enterprise text-processing agent. It's also the most under-specified — every team writes their own ingest pipeline, gets it wrong on the long tail, and spends quarters fixing edge cases.

This chapter is the production playbook: the specialised tools, when each wins, the orchestration pattern, and the long-tail failure modes that distinguish a stage-1 demo from a stage-3 production system.

## Problem it solves

Six concrete document-extraction failures:

1. **Pure OCR loses structure.** Tesseract returns text in reading order; loses table structure, form fields, layout context.
2. **Layout-aware models miss long tail.** LayoutLM fails on unusual formats; can't extract handwritten content.
3. **Vision LLMs hallucinate.** GPT-4V invents values for missing fields; can't be trusted for critical data.
4. **Tables crushed.** Multi-row, multi-column, merged-cell tables become unreadable text.
5. **Forms fragmented.** Field-value pairs lost as the agent reads top-to-bottom.
6. **Multi-page documents.** Each page processed independently; context across pages lost.

Each is a category of failure that production document agents address by composing specialised tools.

## Core idea in one paragraph

Document extraction is a *pipeline of specialised tools*, each best at one thing. **OCR** (Tesseract, Azure OCR, Cloud Vision) extracts raw text from images; high accuracy on printed, low on handwritten or stylised. **Layout-aware models** (LayoutLMv3, Donut, Pix2Struct) understand spatial structure — tables, forms, sections, columns — by jointly modelling text + position + visual features. **Document parsing services** (Chunkr, Unstructured, Document Intelligence, Textract) wrap these into hosted APIs with format-specific parsers. **Vision LLMs** (GPT-4V, Gemini, Claude with vision) provide a fallback for unusual layouts, with hallucination risk. **ColPali / page-level embedders** simplify by embedding entire pages as images, sidestepping OCR for retrieval purposes. The right architecture is a *router* that picks the appropriate tool per document type, falls back to vision LLMs for the long tail, and composes results — text + tables + forms + structure — into a structured output. Combined with verification (cross-check extracted values against source) and human-in-the-loop on low-confidence cases, this is production-grade.

## Mechanism (step by step)

### 1. The tool landscape

| Tool | Best at | Weakness |
|---|---|---|
| **Tesseract** | Clean printed text | Layout, handwriting, low-quality |
| **Cloud OCR (Azure/AWS/GCP)** | Mixed quality, multilingual | Cost; vendor lock-in |
| **LayoutLMv3** | Form / table / structure understanding | Open weights; need ML serving |
| **Donut** | End-to-end document → JSON | Custom training per format |
| **Pix2Struct** | Charts and visualisations | Specific use case |
| **ColPali** | Page-level retrieval | No fine-grained extraction |
| **Document services (Chunkr, Unstructured, Document Intelligence, Textract)** | Hosted, format-aware, robust | Cost; less control |
| **Vision LLMs (GPT-4V, Gemini, Claude)** | Long-tail, ambiguous | Hallucination risk; cost |

No single tool is sufficient for production. The pipeline composes.

### 2. The pipeline pattern

```text
[document]
   ↓
[document type classification: invoice / contract / form / scan / ...]
   ↓
[primary extractor — selected by type]
   ↓
[secondary parsers for specific structures: tables, forms, signatures]
   ↓
[validation: schema check, cross-reference, confidence]
   ↓ (low confidence → escalate)
[vision LLM fallback for residuals]
   ↓
[final structured output]
```

Each stage handles its specialty.

### 3. OCR — the foundation

For pure printed text:
- Cloud OCR APIs (Azure Read, AWS Textract, GCP Document AI) are the production default.
- Tesseract for self-hosted; less robust but free.
- For handwritten / stylised text: cloud OCRs have specialised models; vision LLMs as fallback.

OCR output: text with position (bounding box per word/line). Position metadata enables downstream layout reasoning.

### 4. Layout-aware extraction

LayoutLMv3 (and similar) takes (text, position, image patches) and produces:
- Section labels.
- Form field labels and values.
- Table rows and columns.
- Reading order.

Trained on document datasets (FUNSD, RVL-CDIP, DocBank). Production use: fine-tune on your document templates if templates are stable; use pre-trained as fallback.

### 5. Table extraction

Tables are particularly hard:
- **Simple tables** (regular grid): Tesseract + table-aware OCR.
- **Complex tables** (merged cells, nested headers): LayoutLM + table-specific models (TableFormer, TATR).
- **Tables in scanned documents**: cloud document services typically work best.

Output: structured representation (CSV, JSON, Markdown table).

### 6. Form extraction

Forms have field-value pairs in spatial proximity:
- LayoutLM identifies field labels and values by spatial relationship.
- Output: dictionary of `{field_name: value}`.

For known templates: trained extractors hit 95%+ accuracy. For unknown formats: vision LLM fallback.

### 7. Vision LLM fallback

For unusual or low-quality documents:

```text
Prompt: "Extract the following fields from this document image: invoice number, date, total amount. Output JSON. If a field is unclear, output null."

Image: [the document]
```

GPT-4V / Gemini / Claude with vision are competent. Two caveats:
- **Hallucination**: model invents plausible values when fields are missing.
- **Verification**: cross-check extracted values against the source image (re-prompt with the value, ask "is this in the document?").

### 8. Multi-page handling

Multi-page documents:
- Process each page independently for extraction.
- Use document-level metadata (cover sheet, page numbers, headers) to maintain context.
- Stitch results: per-page extracted data + cross-page references (continuation text, table spans).
- Special handling for: continuation pages, indexed/table-of-contents pages, signatures pages.

### 9. Validation and confidence

Per extracted field:
- **Schema check**: type, format (e.g. date, currency).
- **Cross-reference**: does the value appear elsewhere consistently?
- **Confidence score**: from the extractor, or from a separate verifier.
- **Human-in-the-loop**: low confidence escalates ([23-human-in-the-loop](23-human-in-the-loop.md)).

### 10. The complete production pattern

```text
[document]
   ↓
[classify type: invoice / form / contract / receipt / etc.]
   ↓
[route to specialist extractor (or default OCR + LayoutLM)]
   ↓
[handle structures: OCR for text, table extractor for tables, form extractor for forms]
   ↓
[validate fields: schema + cross-reference]
   ↓
[low-confidence fields → vision LLM]
   ↓
[verify vision LLM output: re-prompt for grounding]
   ↓
[final structured output + confidence per field]
   ↓
[low overall confidence → human review queue]
```

## Empirical anchors

- **Tesseract** ~95% accuracy on clean printed text; drops sharply on quality issues.
- **Cloud OCR services** ~99% on printed; ~85–90% on handwritten.
- **LayoutLMv3 / Donut** state-of-the-art on form / table understanding benchmarks.
- **Vision LLMs** for long-tail extraction: ~85–90% accuracy, with hallucination risk on missing fields.
- **Production document agents** typically combine 3–5 tools; pure-OCR or pure-LLM rarely sufficient.
- **Adoption** in invoice automation, claims processing, contract analysis, identity verification is widespread.

## Variants and counter-arguments addressed

- **"Vision LLMs replace all of this."** They don't; hallucination risk is unacceptable for many use cases.
- **"Cloud document services are enough."** They're often a great default; advanced cases need composition.
- **"OCR is solved."** OCR is solved for printed text; document understanding (layout, tables, forms) is not.
- **"Just hire annotators."** Annotators are necessary for training data; production needs automated extraction at scale.
- **"Cost makes this unscalable."** Costs have dropped; document services charge fractions of cents per page.

## Failure modes and limitations

1. **Quality variance.** Documents range from pristine to badly photocopied; no extractor handles all.
2. **Format diversity.** Even within "invoices", thousands of templates; per-template fine-tuning is expensive.
3. **Multi-language.** Each language is its own model; mixed-language documents are hard.
4. **Handwriting.** Cursive, doctor's handwriting, signatures — long tail.
5. **Vision LLM hallucination.** Plausible but wrong values when source is unclear.
6. **Cost on volume.** High-volume document processing × cloud services = real money.
7. **Privacy.** Documents often contain PII; processing must be tenant-isolated and compliance-aware.
8. **Compounding errors.** OCR error → wrong field value → downstream agent decision wrong.

## When to use, when not

**Use document-extraction agents for** invoice processing, claims, contracts, identity verification, medical records, scientific paper ingestion, legal discovery.

**Skip for** documents that are already digitally structured (Word, plain HTML, structured XML) — direct parsers are easier.

**Compose tools by document type**; don't try one-size-fits-all.

**Pair with verification + HITL** for high-stakes fields.

## Implications for harness engineering

- **Document-type classifier first.** Saves choosing the wrong extractor.
- **Vendor + open-source mix.** Cloud for the easy 80%, specialised for the long tail, vision LLM for the residual.
- **Validation pipeline.** Schema check + cross-reference + confidence.
- **Confidence-driven HITL.** Low confidence escalates to human review.
- **Eval set per document type.** Different types have different quality bars.
- **Cost dashboards.** Document processing can be expensive; track per type.
- **Privacy: tenant-isolated processing.** PII never leaves tenant boundary.
- **Versioning of extractors.** Models change; output stability matters.
- **Feedback loop.** Human corrections feed back as training / fine-tuning data.

The one-sentence takeaway: **document extraction is a pipeline of specialised tools — OCR, layout-aware models, vision LLMs, validators — composed by document type with confidence-driven HITL escalation, because no single tool handles the full diversity of real-world documents.**

## See also

- [25-agentic-rag](25-agentic-rag.md), [134-semantic-indexing](134-semantic-indexing.md) — RAG over extracted documents.
- [136-multimodal-rag](136-multimodal-rag.md) — multimodal context including documents.
- [108-memento-codebase-mcp](108-memento-codebase-mcp.md) — Memento's `documents_tool` reference.
- [23-human-in-the-loop](23-human-in-the-loop.md) — escalation for low-confidence cases.
- [122-explainability-compliance](122-explainability-compliance.md) — privacy in document processing.
- [104-glm-5v-turbo-native-multimodal-agents](104-glm-5v-turbo-native-multimodal-agents.md) — native multimodal agents.
- [149-sector-use-case-catalog](149-sector-use-case-catalog.md) — invoice / claims / health / legal use cases.
