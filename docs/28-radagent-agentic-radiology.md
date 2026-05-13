# 28 — RadAgent: Tool-Using Agents for High-Stakes Domain Reasoning

**Definition.** RadAgent (arXiv:2604.15231) is a tool-using agent framework for stepwise interpretation of chest CT scans. Rather than end-to-end vision-language prediction, it produces a CT report through iterative, tool-augmented reasoning that emits a fully inspectable trace of intermediate decisions. It is a reference example for a much broader pattern: **agentic decomposition as the path to trust in domains where the black box is unacceptable.**

## Problem it solves

Domain experts — clinicians, lawyers, auditors, security analysts — cannot trust a monolithic LLM decision they cannot audit. Vision-language models producing radiology reports have been technically capable for years, but adoption stalls because every report is "correct or wrong, with no way to see why." The agentic alternative exposes the *workflow*: which tool ran, what it measured, how that informed the next decision, which prior studies were retrieved, what level of confidence the agent had at each step. Faithfulness — the tight coupling between observed evidence and output claims — becomes a property you can check, not a claim you have to take on faith.

RadAgent reports +6.0 F1 clinical-accuracy and +24.7 points robustness under adversarial conditions over baseline vision-language models, *while* introducing faithfulness as a new capability. The paper's real contribution is the pattern, not the numbers: for any domain with inspectability requirements, tool-augmented iterative reasoning beats end-to-end prediction.

## Mechanism

Abstracted from RadAgent, applicable to other high-stakes domains:

1. **Decompose the modality.** A CT is not one image; it is N slices, M anatomical regions, several prior studies. The agent reasons one piece at a time.
2. **Bind each step to a tool.** Measurement, segmentation, prior-study retrieval, standardized-vocabulary lookup — each is a tool call with a schema. The model does not "see" the image and declare; it requests a measurement.
3. **Capture the trace.** Every tool call + observation + decision is preserved as an ordered record associated with the output.
4. **Cite in the output.** Each finding in the report references the trace step(s) that produced it. A reviewer can click a finding and see its provenance.
5. **Adversarial evaluation.** Because the trace exists, you can perturb inputs and directly measure which steps break — a dimension of robustness monolithic models simply cannot expose.

## Concrete pattern

A generalized template for "agentic X in a high-stakes domain":

```
Inputs:  primary artifact (scan, contract, log bundle) + prior context
Tools:   decomposition, measurement, retrieval, classification, standards lookup
Output:  report with {findings: [...], citations: [tool_call_ids], confidence: ...}

Trace record: append-only sequence of (step, tool, input, output, rationale).
Review UI:    side-by-side report ↔ trace, with per-finding drill-down.
Eval suite:   faithfulness (every finding traces to evidence),
              robustness (perturb input, measure step-level deltas),
              accuracy (domain metric against ground truth).
```

The two disciplines that distinguish this from naive RAG are (a) tool calls rather than retrieval chunks as evidence — operations are more auditable than passages — and (b) append-only trace, not free-form chain-of-thought.

## Variants & related techniques

- **Agentic RAG** ([25-agentic-rag.md](25-agentic-rag.md)) — the retrieval-specific cousin; RadAgent generalizes the idea to tool use.
- **Verifier/evaluator loops** ([11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md)) — the faithfulness step is a single-shot verifier applied per finding.
- **Chain-of-Verification** ([18-chain-of-verification-self-refine.md](18-chain-of-verification-self-refine.md)) — complementary at the language level.
- **MCP** ([07-model-context-protocol.md](07-model-context-protocol.md)) — the natural distribution format for specialized domain tools (DICOM readers, vocabulary servers).
- **GPT-Rosalind** ([30-gpt-rosalind-domain-specialized.md](30-gpt-rosalind-domain-specialized.md)) — same pattern in life sciences.

## Failure modes & anti-patterns

- **Trace as theater.** A trace nobody reads is just overhead. Fix: embed per-finding drill-down in the review UI so the trace is a natural click, not a separate tab.
- **Tool proliferation.** Fifty tools, each called once; the agent thrashes. Fix: curate a small core tool set; promote composites to named skills.
- **Faithfulness without correctness.** The report is perfectly traced to tools that were wrong. Fix: pair faithfulness eval with ground-truth domain metrics.
- **Hidden end-to-end fallback.** When tools fail, the agent silently falls back to direct image-to-text generation. Fix: explicit "degraded mode" signal in the output; policy on whether to ship or escalate.
- **Adversarial blind spots.** Adversarial eval only perturbs images; ignores tool-side injection from retrieved priors. Fix: threat-model the tool layer too ([22-guardrails-prompt-injection.md](22-guardrails-prompt-injection.md)).
- **Regulator churn.** Any update to tools invalidates prior audits. Fix: version tools and traces; pin schemas per release.

## When to use (and when not to)

**Use** RadAgent-style patterns when:

- The domain demands inspectable decisions (medicine, legal, finance, safety engineering).
- Specialized tools exist or can be built.
- You can run domain-grounded evals, not just LLM-judges.
- Faithfulness is a product requirement, not a research curiosity.

**Don't** use them when:

- Latency is paramount and users won't wait for multi-tool trajectories.
- No good decomposition exists; the "thought" is genuinely atomic.
- Regulators explicitly require a single validated monolithic model path.

## References

- arXiv:2604.15231 — "RadAgent: A tool-using AI agent for stepwise interpretation of chest computed tomography". <https://arxiv.org/abs/2604.15231>
- Anthropic Engineering, "Effective context engineering for AI agents". <https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents>
- FDA draft guidance on AI/ML in medical devices — inspectability & versioning context. <https://www.fda.gov/>
- Related agentic RAG literature ([25-agentic-rag.md](25-agentic-rag.md)).
