# 50 — METCL: Neuro-Symbolic Metaphor Reasoning (Delta of Thought)

**Definition.** "The Delta of Thought: Channeling Rivers of Commonsense Knowledge in the Sea of Metaphorical Interpretations" (Lieto, Pozzato, Zoia; CIIT Lab, University of Salerno & University of Turin; IJCAI 2025) introduces **METCL** — Metaphor Elaboration in Typicality-based Compositional Logic — a neuro-symbolic system that generates and identifies metaphors by composing commonsense concepts via a formal typicality-based logic. METCL improves both state-of-the-art LLMs (DeepSeek-R1, GPT-4o, Qwen2.5-Max) and symbolic systems on metaphor identification, and its generated metaphors are well-received by humans.

## Problem it solves

LLMs handle figurative language surprisingly well for common metaphors but break on compositional cases: *Steve Jobs is a Michelangelo of business* requires blending the salient (typical) attributes of Michelangelo with the business domain while suppressing irrelevant ones. Purely statistical models can memorize common metaphors and imitate style, but **reasoning about which attributes transfer and which don't** is a symbolic problem — ontologies, typicality scores, constraint satisfaction. Purely symbolic systems have the reasoning but lack fluency.

METCL is the neuro-symbolic marriage ([37-neuro-symbolic-ai.md](37-neuro-symbolic-ai.md)): symbolic reasoning about concept combination inside an LLM-friendly wrapper.

## Mechanism — TCL and METCL

### TCL (Typicality-based Compositional Logic)

A prior logic framework for commonsense concept combination. Each concept has **prototypical attributes** with graded typicality. Combining concepts (HEAD + MODIFIER like "apartment dog") is a reasoning problem:

- Which HEAD attributes are preserved?
- Which MODIFIER attributes override?
- Which emergent attributes arise?

Formal rules govern the combination, producing a defensible answer.

### METCL

Extends TCL specifically for metaphor:

- **Metaphor identification.** Given a candidate metaphor sentence, decide whether interpreting X as Y is licensed by typicality-based attribute transfer.
- **Metaphor generation.** Given a target concept and a desired rhetorical effect, propose candidate source concepts and the attribute pathways that would make the metaphor work.

LLMs struggle with these because they don't access explicit typicality structure; symbolic systems struggle because they lack world knowledge. METCL pipelines both: LLM proposes candidates; typicality-based logic filters and scores them; LLM renders the final output fluently.

## Concrete pattern — structure of a METCL-style agent

```
Input: target concept C and (optional) rhetorical aim.

Step 1 (neural): propose candidate source concepts S_i with high
        associative similarity to C on the aim dimension.

Step 2 (symbolic, TCL-based): for each (C, S_i),
        compute attribute-transfer feasibility score:
          - prototypical attributes of S that transfer to C domain
          - attribute conflicts that should be suppressed
          - emergent attributes that arise from the combination

Step 3 (neural): render the highest-scoring combination as natural-language
        metaphor; optionally iterate with the user.

Output: candidate metaphors with explanations of attribute transfer.
```

The pattern generalizes beyond metaphor: any task where the right answer requires *composing* structured meaning under constraints benefits from this neuro-symbolic loop.

## Variants & related techniques

- **Neuro-Symbolic AI** ([37-neuro-symbolic-ai.md](37-neuro-symbolic-ai.md)) — the general design space.
- **ConceptNet / ATOMIC / commonsense knowledge bases** — possible sources of the typicality signals.
- **Constrained decoding** — a lightweight symbolic filter over neural outputs.
- **Tool use via CAS / theorem prover / typicality checker** — the operational shape in an agent harness.
- **Tree of Thoughts** ([15-tree-of-thoughts-lats.md](15-tree-of-thoughts-lats.md)) — orthogonal search mechanism; can combine with METCL-style evaluators.

## Failure modes & anti-patterns

- **Typicality without grounding.** If typicality scores come from a single static KB, they become stale or culture-specific. Fix: multi-source typicality, re-estimate from corpora.
- **Candidate starvation.** LLM-proposed candidates are narrow; symbolic layer has nothing good to filter. Fix: diverse sampling, broaden prompts.
- **Symbolic over-rejection.** Overly strict attribute-transfer rules veto creative but valid metaphors. Fix: softer constraints; human feedback calibration.
- **Losing fluency on the way back.** The symbolic layer returns attribute lists; the neural layer produces stilted natural-language output. Fix: render-and-refine with the LLM.
- **Benchmark-specific tuning.** Metaphor corpora are small and skewed. Generalization to new domains requires explicit re-tuning.

## When to use (and when not to)

**Use** this pattern (neuro-symbolic concept combination) when:

- The task has clear compositional structure and attribute-level reasoning.
- Pure LLM approaches demonstrably fail on composition (creative writing quality, legal reasoning, specialized classifications).
- A typicality or ontology source is available or learnable.

**Don't** when:

- The task is fuzzy; constraints don't help.
- You lack the KB substrate and can't build it.

## References

- Lieto, Pozzato, Zoia, "The Delta of Thought: Channeling Rivers of Commonsense Knowledge in the Sea of Metaphorical Interpretations", IJCAI 2025. <https://www.ijcai.org/proceedings/2025/1146>
- Preprint PDF. <https://www.ijcai.org/proceedings/2025/1146.pdf>
- CIIT Lab publications. <https://www.ciitlab.org/en/pubblicazioni/>
- "Neuro-Symbolic AI" (Belle & Marcus, AAAI 2026). <https://ojs.aaai.org/index.php/AAAI/article/view/42130>
- Prior TCL work by the same authors. <https://www.antoniolieto.net/listpub.html>
