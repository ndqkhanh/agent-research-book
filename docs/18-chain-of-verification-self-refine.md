# 18 — Chain-of-Verification & Self-Refine

**Definition.** Two closely-related self-critique patterns. **Self-Refine** (Madaan et al. 2023) has the model generate an answer, critique its own answer, and refine it — iterating until convergence or a budget. **Chain-of-Verification (CoVe)** (Dhuliawala et al. 2023) goes further: after drafting, the model generates *verification questions* about its own claims, answers them independently, and revises the draft based on the verification results. Both are cheap, model-only defenses against hallucination.

## Problem it solves

LLMs produce confident, plausible, wrong answers. A naive fix is "add RAG", but many hallucinations are not information-lookup failures — they're reasoning consistency failures. The answer contradicts itself, cites non-existent sources, or violates a constraint that was in the prompt. Self-critique loops catch these without any external tool. CoVe's key insight: the model is often better at *answering a specific factual sub-question* than at *writing a paragraph without errors*. Decomposing the claim into verifiable questions raises the bar.

## Mechanism

### Self-Refine

Three prompts looped:

```
1. Generate: produce an initial answer y₀ to task x.
2. Feedback: given x and y_i, produce structured critique c_i.
3. Refine: given x, y_i, and c_i, produce improved y_{i+1}.
Stop when: critique says "no issues" OR iteration budget reached.
```

The same base model plays all three roles. The paper shows consistent gains across tasks — math word problems, code optimization, sentiment reversal — with 2–3 iterations.

### Chain-of-Verification (CoVe)

Four stages:

1. **Baseline response.** Draft answer (may contain hallucinations).
2. **Plan verifications.** Generate a list of factual questions whose answers would check the draft's claims.
3. **Execute verifications.** Answer each question *independently*, without seeing the original draft (to avoid anchoring bias).
4. **Final response.** Rewrite the draft using the verification answers; drop or correct claims contradicted by verifications.

The "independent" step 3 is the crucial one. If the model sees the draft while answering sub-questions, it confirms whatever the draft said.

## Concrete pattern

Self-Refine on a coding task (abbreviated):

```
System: You are a code refiner. Iterate until the feedback says "OK".

Generate:
  Task: write a Python function that flattens a nested list.
  Output: def flatten(xs): return sum(xs, [])

Feedback:
  Issues:
  - `sum(xs, [])` only handles one level of nesting.
  - Task said "nested", suggesting arbitrary depth.
  - Also O(n²) for long lists.

Refine:
  def flatten(xs):
      out = []
      for x in xs:
          if isinstance(x, list):
              out.extend(flatten(x))
          else:
              out.append(x)
      return out
```

CoVe on a factual paragraph:

```
Baseline: "Tim Berners-Lee invented the World Wide Web at CERN in 1990.
He received the Turing Award in 2017 and was knighted in 2004."

Plan verifications:
  Q1: Where did Tim Berners-Lee invent the World Wide Web?
  Q2: In what year?
  Q3: Did he receive the Turing Award? If so, when?
  Q4: Was he knighted? If so, when?

Execute (independently, without the draft in context):
  A1: CERN.
  A2: 1989 (proposal), first web page 1990–1991.
  A3: Yes, 2016.
  A4: Yes, 2004.

Final response: "Tim Berners-Lee invented the World Wide Web at CERN,
proposing it in 1989 and releasing the first page in 1990. He received
the Turing Award in 2016 and was knighted in 2004."
```

The invented date (1990 Turing year) in the draft was caught by the independent answer (2016).

## Variants & related techniques

- **CRITIC** (arXiv:2305.11738) replaces the self-critic with a tool-augmented critic — web search, a calculator, an interpreter. Much stronger than self-critique alone for factuality.
- **Self-Consistency** samples multiple answers and majority-votes; no critique, but cheap and surprisingly effective.
- **Reflexion** ([14-reflexion.md](14-reflexion.md)) is the *across-episodes* version — the critique is kept as a lesson for next time.
- **Verifier/Evaluator loops** ([11-verifier-evaluator-loops.md](11-verifier-evaluator-loops.md)) extend the idea with a *separate model* for the critic, breaking the shared-blind-spot problem.
- **Constitutional AI (Anthropic).** Model critiques its own outputs against a set of principles; an industrialized Self-Refine at fine-tuning time.

## Failure modes & anti-patterns

- **Sycophantic critic.** The critic says "looks good!" because the generator and critic share context and beliefs. Fix: separate the prompts more aggressively (CoVe's independence discipline), or switch critics to a different model.
- **Critique-induced hallucination.** The critic invents problems that aren't there; the refiner "fixes" them, making things worse. Fix: ground critiques in checkable claims ("quote the exact problematic span").
- **Unbounded iteration.** Refinement loops forever because every draft generates new critiques. Fix: hard iteration cap + termination condition ("no blocking issues").
- **Anchoring in CoVe step 3.** If the verification prompt accidentally includes the draft, independent answers aren't independent. Fix: literal prompt audit.
- **Cost blowup.** CoVe multiplies LLM calls by 3–5×. For simple tasks, not worth it.
- **Rewriting for the critic, not the user.** The final output optimizes to pass the critique rather than to answer the user. Fix: constrain critiques to blockers, not style preferences.

## When to use (and when not to)

**Use** Self-Refine / CoVe when:

- Output correctness matters more than latency (summarization of facts, technical writing, code review).
- No external tool ground truth is available (otherwise, CRITIC or actual tool-grounded verification is better).
- You can afford 2–5× the base call cost.
- Hallucination is the dominant failure mode, not reasoning skill.

**Don't** use them when:

- You have real ground truth — run tests or queries instead of asking the model.
- The task is pattern-following (classification, extraction) where the model either gets it or doesn't.
- Latency budget is tight and a single-pass model is "good enough".

## References

- Madaan et al., "Self-Refine: Iterative Refinement with Self-Feedback", arXiv:2303.17651 — <https://arxiv.org/abs/2303.17651>
- Dhuliawala et al., "Chain-of-Verification Reduces Hallucination in Large Language Models", arXiv:2309.11495 — <https://arxiv.org/abs/2309.11495>
- Gou et al., "CRITIC: LLMs Can Self-Correct with Tool-Interactive Critiquing", arXiv:2305.11738 — <https://arxiv.org/abs/2305.11738>
- Wang et al., "Self-Consistency Improves Chain-of-Thought Reasoning", arXiv:2203.11171 — <https://arxiv.org/abs/2203.11171>
- Bai et al., "Constitutional AI: Harmlessness from AI Feedback", arXiv:2212.08073 — <https://arxiv.org/abs/2212.08073>
