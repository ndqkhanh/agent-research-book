# 135 — Trustworthy Generation: Citation, Grounding, Anti-Hallucination, Refusal-on-Unknown

**Sources.** Lakshmanan & Hapke, *Generative AI Design Patterns*, Patterns 11–12 (Trustworthy Generation, Deep Search); Devlin, *Building LLM Agents with RAG Knowledge Graphs*, Chapter 3 (RAG: The Backbone of Truthful Agents); the academic literature on hallucination (Ji et al. 2023, Maynez et al. 2020 on faithfulness); plus production patterns from Anthropic citations, OpenAI's structured-outputs, Perplexity's citation system.

**One-line definition.** Trustworthy generation is the discipline of producing LLM outputs that are *grounded* (every claim traceable to a source), *cited* (sources surfaced inline to the user), *faithful* (no claims unsupported by retrieved context), and *refusal-aware* (the system says "I don't know" when grounding fails) — implemented as a post-retrieval / pre-output stack of patterns that catches the hallucinations RAG by itself does not.

## Why this matters

RAG by itself does not produce trustworthy generation. It retrieves relevant context and conditions the LLM on it; the LLM then composes a response that *usually* sticks to the retrieved facts but *sometimes* invents details, attributes wrong sources, or produces correct-but-unsupported claims. For internal-use agents, the occasional hallucination is annoying. For customer-facing, regulated, or auditable agents, it is disqualifying.

Trustworthy generation is the layer of patterns that turns "the model is grounded most of the time" into "every claim is traceable, citations are correct, and unsupported claims are refused." It is the pattern stack that lets a stage-3 production agent ship to regulated customers without the legal team blocking it.

For agent builders, trustworthy generation is also the *user-trust* discipline: users who can see citations and ask "where did this come from?" trust the agent more, even when the underlying answer is identical. The patterns are simultaneously safety, compliance, and product features.

## Problem it solves

Six concrete generation failures trustworthy patterns address:

1. **Plausible hallucination.** Model invents a citation that doesn't exist; user reads it as fact.
2. **Wrong attribution.** Model cites the wrong source for a claim; auditor finds discrepancy.
3. **Composite hallucination.** Model combines two real facts into a false claim; each source individually correct, the composite is not.
4. **Unsupported confident answer.** Model has no retrieved evidence but answers confidently anyway.
5. **No "don't know."** Model never refuses; on out-of-knowledge questions it confabulates.
6. **Citation surface absent.** User has no way to verify the agent's answer; trust does not develop.

Each is a category of bug that trustworthy generation eliminates structurally.

## Core idea in one paragraph

Trustworthy generation is a *post-retrieval / pre-output* stack: after retrieval ([134-semantic-indexing](134-semantic-indexing.md)) and reranking ([133-reranking-for-agentic-retrieval](133-reranking-for-agentic-retrieval.md)), but before showing the user the answer. It has four core patterns: **grounded generation** (the LLM is prompted to use only the retrieved context, with explicit refusal language when the context is insufficient); **inline citation** (the LLM emits citation markers tied to specific retrieved chunks; the system surfaces these to the user); **faithfulness verification** (a separate verifier model — typically smaller/cheaper — checks that every claim in the output is supported by the cited context); **refusal-on-unknown** (when verification fails or context is insufficient, the system explicitly says "I don't know" or "I cannot answer that question with the available information"). Layered, these patterns bring hallucination from "occasionally embarrassing" to "structurally rare," with the residual hallucinations caught by verification before the user sees them. The cost is verifier compute and slightly longer prompts; the benefit is shippable trustworthiness.

## Mechanism (step by step)

### 1. Grounded generation prompt pattern

The base prompt for the synthesiser:

```text
You are answering a question using ONLY the provided context.

Rules:
- Every claim in your answer must be supported by the context.
- Cite the source for each claim using [source_id:position] notation.
- If the context does not contain enough information to answer, respond:
  "I cannot answer this question with the available information."
- Do NOT use external knowledge.
- Do NOT invent citations.

Context:
[1] <retrieved chunk 1, source: doc-A:para-3>
[2] <retrieved chunk 2, source: doc-B:para-1>
...

Question: {question}

Answer:
```

The prompt is deliberately constraining. With a strong base model and clear context, it produces grounded outputs most of the time. The remaining failures are caught by the verifier.

### 2. Inline citation format

Citations appear inline:

```text
The policy was updated in March 2024 [1:para-3] to include new pricing tiers [2:para-1]. The effective date is April 1, 2024 [1:para-5].
```

Each `[citation_id:position]` references a chunk in the retrieved context. The user-facing UI renders these as clickable footnotes that surface the source.

Citation discipline is part of the prompt; structural enforcement is part of constrained decoding ([112-constrained-decoding](112-constrained-decoding.md)) — the citation regex is a grammar constraint.

### 3. Faithfulness verification

A separate verifier model checks the output against the context:

```text
[output sentences] × [retrieved context]
   ↓
[verifier: for each sentence, does the cited context support it?]
   ↓
{
  "sentence_1": {"supported": true, "citation": "[1]"},
  "sentence_2": {"supported": true, "citation": "[2]"},
  "sentence_3": {"supported": false, "issue": "no matching evidence"}
}
```

Implementations:
- **NLI-based verifier**: natural language inference model (BART-large-MNLI, DeBERTa) checks entailment.
- **LLM-judge verifier**: prompted LLM rates faithfulness per sentence.
- **Hybrid**: NLI for fast majority, LLM for ambiguous cases.

Output: per-sentence verification result.

### 4. Refusal-on-unknown

If verification fails or context insufficiency is detected:

```text
if any unsupported_sentence in verification:
    return "I cannot fully answer your question. The context I have access to lacks information about [topic]. Would you like me to search for more sources or escalate to a human?"

if context_relevance_below_threshold:
    return "I don't have enough information to answer this question reliably."
```

The agent never confabulates; it acknowledges uncertainty.

This is the *right-to-explanation* surface from a user-trust perspective ([122-explainability-compliance](122-explainability-compliance.md)).

### 5. Deep search loop — for multi-step grounding

For complex questions, single-shot RAG is insufficient; need iterative deep search:

```text
[query]
   ↓
[search 1: broad retrieval]
   ↓
[draft answer with citations]
   ↓
[verifier: identify unsupported claims]
   ↓
[search 2: targeted retrieval for unsupported claims]
   ↓
[refine answer]
   ↓
[verifier: check again]
   ↓ (loop until verifier accepts or max-iterations)
[final grounded answer]
```

This is the [25-agentic-rag](25-agentic-rag.md) pattern productionised for trustworthiness. It catches more hallucinations at the cost of latency and compute.

### 6. Citation correctness verification

Citations themselves can be wrong (model cites [3] for a fact actually in [5]). A separate citation-correctness check:

```text
for each citation in output:
    [cited_chunk] = retrieve(citation_id)
    [claim] = sentence containing the citation
    if not entails(cited_chunk, claim):
        flag for re-citation or refusal
```

Lightweight; catches wrong-attribution errors that grounded generation alone misses.

### 7. Provenance preservation

Every retrieved chunk carries metadata:
- Source document ID + version.
- Position in document (paragraph, page).
- Retrieval score.
- Index source (which index returned it).

The synthesiser preserves provenance through to the citation. The user UI can drill from claim → citation → source document → exact paragraph.

This is the audit trail compliance ([122-explainability-compliance](122-explainability-compliance.md)) requires.

### 8. Adversarial robustness

Trustworthy generation must also defend against:
- **Poisoned context**: a malicious chunk inserted into the corpus ([82-poisonedrag](82-poisonedrag.md)).
- **Prompt injection in retrieved content**: a chunk that says "ignore previous instructions". Defense: structural separation of context vs instruction in the prompt.

These are the [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md) extensions for grounded systems.

### 9. The complete trustworthy-generation pipeline

```text
[query]
   ↓
[retrieve + rerank → top-K]
   ↓
[grounded generation prompt with citations + refusal language]
   ↓
[draft output with [citation] markers]
   ↓
[citation-correctness check]
   ↓
[faithfulness verifier — per sentence]
   ↓ (any unsupported → loop or refuse)
[final answer + clickable citations OR refusal]
   ↓
[user sees answer with verifiable citations]
```

Multi-stage. Each stage adds confidence.

### 10. User-facing UX

The user-facing UI for trustworthy generation:
- Inline citation badges in the answer text.
- Click-to-source: badge expands to show the cited paragraph.
- Confidence indicator: visual signal of how grounded the answer is.
- "Not sure?" affordance: explicitly invokes deeper search or HITL.

See [143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md) for the UX patterns.

## Empirical anchors

- **Hallucination reduction**: well-implemented trustworthy generation cuts hallucination rates 5–10× vs vanilla RAG.
- **Citation correctness** with verifier: ~95%+; without, ~70–85%.
- **User trust** measurably correlates with citation surfaces.
- **Refusal rate**: typically 5–15% of queries should be refused; if 0%, the system is over-confident.
- **Verifier latency**: 100–500ms typical; well within agent budgets.
- **Adoption**: in 2026 high in regulated and customer-facing systems; lower in internal tools (a missed opportunity).

## Variants and counter-arguments addressed

- **"The model already cites well."** Not consistently enough for production; verifier catches the residual cases.
- **"Verification is expensive."** Compute is modest; smaller verifier models are sufficient.
- **"Refusal hurts user experience."** Less than wrong answers do.
- **"Grounded generation is just prompt engineering."** It's prompt engineering plus verifier plus citation infrastructure plus refusal logic. Multi-pattern.
- **"Adversarial robustness is a security concern, not generation."** The boundary is fuzzy; defense-in-depth combines both.

## Failure modes and limitations

1. **Verifier mis-calibration.** Rejects valid claims or accepts invalid ones; periodic human-labelled re-calibration.
2. **Context insufficiency masked.** Verifier accepts a vague claim that "could be inferred"; tighten verifier criteria.
3. **Wrong-citation that entails.** Model cites [5] for a fact in [5] but the user's question is actually different. Verifier needs question-aware checking.
4. **Refusal overuse.** Verifier too strict; legitimate answers refused. Calibrate threshold.
5. **Latency surprise.** Multi-stage verification + deep search adds latency; budget upfront.
6. **Multi-modal grounding gap.** Trustworthy text generation is mature; image, audio, video less so.
7. **Compositional hallucination.** Two facts each grounded but their combination isn't; harder to catch.
8. **Adversarial bypass.** Sophisticated prompt-injection can fool naive verifiers; defense-in-depth.

## When to use, when not

**Mandatory for**: customer-facing, regulated, audited, high-stakes generation.

**Strongly recommended for**: any internal tool where wrong answers degrade trust or productivity.

**Skip for**: pure creative or brainstorming generation where "wrong" doesn't apply.

**Always have refusal**: even in low-stakes systems, the model should be able to say "I don't know."

## Implications for harness engineering

- **Verifier as a sidecar service.** Always-on; small model.
- **Citation infrastructure end-to-end.** Retrieval → generation → UX. Provenance preserved through every stage.
- **Refusal as a first-class output type.** Not an error; a deliberate response with its own UX affordance.
- **Faithfulness eval in CI/CD.** [115-evaluating-llm-systems](115-evaluating-llm-systems.md). Block deploys that regress.
- **Adversarial test suite.** [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md), [82-poisonedrag](82-poisonedrag.md). Periodic red-team.
- **User-facing citations in UI.** Even for internal tools; the trust effect is real.
- **Constrained decoding for citation grammar.** [112-constrained-decoding](112-constrained-decoding.md). Citations parse correctly.
- **Deep search loop for multi-hop questions.** Single-shot RAG insufficient; iterate with verifier feedback.
- **Refusal rate dashboard.** Track over time; sudden changes signal problems.

The one-sentence takeaway: **trustworthy generation is grounded prompt + inline citations + faithfulness verifier + refusal-on-unknown — implemented as a stack on top of retrieval/rerank, it turns "RAG sometimes hallucinates" into "RAG produces auditable, citable, refusal-aware answers."**

## See also

- [22-guardrails-prompt-injection](22-guardrails-prompt-injection.md), [82-poisonedrag](82-poisonedrag.md), [49-agents-of-chaos-red-teaming](49-agents-of-chaos-red-teaming.md) — adversarial robustness side.
- [25-agentic-rag](25-agentic-rag.md) — agentic RAG with self-critique; the pattern this productionises.
- [11-verifier-evaluator-loops](11-verifier-evaluator-loops.md) — the verifier pattern.
- [122-explainability-compliance](122-explainability-compliance.md) — citations as audit trail.
- [120-rag-to-finetuning-spectrum](120-rag-to-finetuning-spectrum.md), [133-reranking-for-agentic-retrieval](133-reranking-for-agentic-retrieval.md), [134-semantic-indexing](134-semantic-indexing.md) — the RAG stack this rests on.
- [112-constrained-decoding](112-constrained-decoding.md) — structural enforcement of citation grammar.
- [143-ux-design-for-agentic-systems](143-ux-design-for-agentic-systems.md) — citation UX surfaces.
- [115-evaluating-llm-systems](115-evaluating-llm-systems.md) — faithfulness eval discipline.
