# 11 — Verifier / Evaluator Loops

**Definition.** Verifier loops pair a *generator* agent that produces work with a separate *evaluator* agent (or deterministic check) that scores it, sending failing work back to the generator with feedback. In the extreme, a third *planner* agent sets the spec. Anthropic's three-agent harness (planner + generator + evaluator), modeled after Generative Adversarial Networks, is the 2026 reference architecture for long-running autonomous coding.

## Problem it solves

A single agent is a bad judge of its own output. It declares "done" when tests fail, misses edge cases it didn't think to try, and gets rewarded by its own reasoning for shipping quickly rather than correctly. You can partly fix this with better prompts, but the deeper issue is that the *same* context that generated the code also decides whether the code is good — cognitive-bias failure modes transfer directly to LLMs.

Splitting generation from evaluation into different agents with different prompts, different context, and sometimes different models breaks the bias. The evaluator has fresh eyes and a narrower mandate: "given this spec and this output, does it meet the bar?"

## Mechanism

The three-agent form (Anthropic's Harness Design for Long-Running Application Development):

1. **Planner.** Given a high-level goal, produces a structured feature list, acceptance criteria, and architectural constraints. Runs once per milestone; may be re-invoked if evaluator repeatedly fails generator output on the same criterion, signaling a bad plan.
2. **Generator.** Given the plan and current state, executes: writes code, runs tools, produces artifacts. Does not decide when the work is done.
3. **Evaluator.** Given the plan's criteria and the generator's artifacts, scores objectively where possible (tests pass? lint clean? types check?) and subjectively where necessary (does this UI match the design intent?). Emits either "accept" or "reject + structured critique".

The loop:

```
planner → plan → [generator → artifacts → evaluator → (accept | critique)] → next
                                   ↑_________________________|
```

On reject, the generator receives the critique and iterates. If the evaluator rejects K times on the same criterion, escalate back to the planner (maybe the criterion itself is wrong).

Separation of concerns gives the architecture its GAN-like property: the evaluator's job is to break the generator's work; the generator's job is to beat the evaluator. This asymmetric pressure produces output quality neither alone reaches.

## Concrete pattern

Objective criteria are cheap and should come first:

```python
def evaluate(artifact, plan):
    checks = []
    checks.append(("tests_pass", run("pytest -q").returncode == 0))
    checks.append(("types_clean", run("mypy src").returncode == 0))
    checks.append(("lint_clean", run("ruff check src").returncode == 0))
    checks.append(("plan_items_touched",
                   all(f in git_diff_files() for f in plan.expected_files)))

    if not all(passed for _, passed in checks):
        return Reject(critique=format_failures(checks))

    # Only then invoke the subjective LLM evaluator
    return llm_evaluator(artifact, plan)
```

LLM-evaluator prompt structure (simplified):

```
You are an evaluator, not a coder. You do not fix problems; you identify them.
Given the plan and the artifact:
- Which acceptance criteria are met? Quote evidence.
- Which are not met? Be specific (file:line).
- Are there edge cases the generator didn't handle?

Output strictly:
{ "verdict": "accept" | "reject",
  "failures": [{"criterion": "...", "evidence": "...", "fix_hint": "..."}] }
```

The evaluator's isolation is doctrinal. No history of the generator's reasoning, no sunk-cost sympathy. It sees plan + artifact.

## Variants & related techniques

- **Self-Refine** ([18-chain-of-verification-self-refine.md](18-chain-of-verification-self-refine.md)) uses the same model as both generator and critic. Cheap, but vulnerable to shared bias.
- **CRITIC** (arXiv:2305.11738) grounds critique in external tools (a calculator, a search engine) rather than another model's opinion.
- **LATS** ([15-tree-of-thoughts-lats.md](15-tree-of-thoughts-lats.md)) uses evaluation as a value function in a tree search.
- **Reflexion** ([14-reflexion.md](14-reflexion.md)) keeps the critic's feedback across episodes instead of just the current one.
- **LLM-as-Judge for benchmarking** ([21-llm-as-judge-trajectory-eval.md](21-llm-as-judge-trajectory-eval.md)) is the same idea lifted to eval suites.
- **Two-model asymmetry.** Use a stronger model for evaluation than for generation. Cheaper to run a fast generator and gate with a slow, smart evaluator than to ask the smart model to do both jobs every step.

## Failure modes & anti-patterns

- **Evaluator is too lenient.** Says "accept" on work that's plausibly wrong. Fix: objective checks first; add specific "probe" criteria the evaluator must check (e.g., "verify this edge case by constructing an input").
- **Evaluator is too strict.** Rejects forever on stylistic quibbles. The generator spirals. Fix: separate blocking criteria (must be met) from advisory (nice-to-have). Only blocking causes rejection.
- **Circular reasoning.** Generator sends its own reasoning along with its artifact; evaluator is convinced because the generator is. Fix: evaluator only sees plan + artifact, not reasoning.
- **Critique without fix hints.** Evaluator says "wrong". Generator doesn't know how. Fix: evaluator must return *actionable* critique — specific file, specific expectation, specific suggestion.
- **Evaluator cost blowup.** Every step triggers a full eval. Fix: gate eval on "generator says done"; don't eval mid-stream.
- **Same model hallucinating in both roles.** If both agents use the same base model and share the same blind spot, the GAN dynamic vanishes. Mix models, prompts, or both.
- **Planner drift.** Over many rejection cycles, the planner amends the plan to match what the generator can do, not what was needed. Fix: the *original* plan is the source of truth; amendments require explicit justification.

## When to use (and when not to)

**Use** verifier/evaluator loops when:

- The task has any non-trivial "done" criterion — tests, acceptance, coverage, style.
- The task runs autonomously and can't rely on a human to catch bad completions.
- You're assembling a long-running harness ([10-multi-session-continuity.md](10-multi-session-continuity.md)) and need per-session gates.
- The cost of bad output (shipping broken code) exceeds the cost of rework.

**Don't** use them when:

- The task is short and the user will inspect results anyway.
- You lack objective criteria and the evaluator would just be second-guessing.
- The evaluator model isn't more reliable than the generator on this domain — pure self-critique is usually ineffective without external grounding.

## References

- Anthropic Engineering, "Harness design for long-running application development" — <https://www.anthropic.com/engineering/harness-design-long-running-apps>
- Anthropic Engineering, "Effective harnesses for long-running agents" — <https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents>
- InfoQ, "Anthropic Designs Three-Agent Harness" — <https://www.infoq.com/news/2026/04/anthropic-three-agent-harness-ai/>
- "CRITIC: LLMs Can Self-Correct with Tool-Interactive Critiquing", arXiv:2305.11738 — <https://arxiv.org/abs/2305.11738>
- Rick Hightower, "Anthropic's Harness Engineering: Two Agents, One Feature List" — <https://medium.com/@richardhightower/anthropics-harness-engineering-two-agents-one-feature-list-zero-context-overflow-7c26eb02c807>
