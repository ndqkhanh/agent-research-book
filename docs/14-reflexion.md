# 14 — Reflexion

**Definition.** Reflexion (Shinn et al., 2023) is an agent architecture in which, after each episode, a dedicated *self-reflection* step converts environment feedback (a failure, a test output, a reviewer's comment) into a short verbal lesson, which is appended to an *episodic memory* consulted at the start of future episodes. The model doesn't update its weights; it updates its self-instructions.

## Problem it solves

Agents fail, retry, and fail the same way again. Reinforcement learning could teach them, but RL requires training infrastructure, reward signals, and gradient updates — none of which are available when you're using a hosted frontier model as-is. Reflexion is **verbal reinforcement**: the agent learns across episodes without any parameter change, purely by writing and re-reading its own lessons.

Reflexion also gives the agent a form of *experience*, not just *memory*. A plain transcript log tells you what happened; a reflection tells you what it meant and what to do differently.

## Mechanism

The architecture layers three cooperating models/components:

1. **Actor.** The ReAct-style agent that performs the task (see [13-react.md](13-react.md)).
2. **Evaluator.** Grades the actor's trajectory against environment feedback — usually a binary success/failure from the environment (tests passed? task completed?), sometimes augmented with a model-based critique.
3. **Self-reflection model.** Given the trajectory and the evaluator's verdict, produces a short verbal lesson: "I failed because I assumed X, but the environment required Y. Next time I should do Z."

After each episode:

- Append the reflection to the episodic memory (an in-context list or an external file).
- Start the next episode with the same task, but now the actor's system prompt includes the accumulated reflections.
- Keep the last *k* reflections (not all of them) to prevent context bloat.

The paper reports substantial gains on decision-making, reasoning, and programming benchmarks — for example, 91% pass@1 on HumanEval with GPT-4, beating GPT-4 alone at 80%.

## Concrete pattern

Pseudo-loop:

```python
reflections = []
for episode in range(max_episodes):
    trajectory = actor.run(task, memory=reflections)
    verdict = evaluator.grade(trajectory, environment)
    if verdict.success:
        break
    lesson = reflector.summarize(
        task=task,
        trajectory=trajectory,
        verdict=verdict,
        prior_reflections=reflections,
    )
    reflections.append(lesson)
    reflections = keep_last_k(reflections, k=3)
```

A concrete reflection (for a coding task):

> **Episode 2 lesson.** I failed because the test expected case-insensitive
> matching, but I used `.startswith(s)` which is case-sensitive. I noticed
> the test message mentioned "case" but didn't act on it. Next time, when
> a failure message uses words like "case", "whitespace", or "order",
> inspect the test input/expected pair *before* re-editing the code.

Contrast with an unhelpful reflection: *"I should try harder and be more careful."* That's advice for humans, not a rule the next episode can apply. Reflexion-style lessons should name the **specific signal** and the **specific response**.

## Variants & related techniques

- **Self-Refine** ([18-chain-of-verification-self-refine.md](18-chain-of-verification-self-refine.md)) reflects *within* an episode (regenerate, critique, refine) rather than across episodes.
- **CRITIC** (arXiv:2305.11738) grounds critique in real tools (calculator, search) rather than the model's self-assessment.
- **Generative Agent memory** (Park et al., 2023) extends the idea with reflection + scoring + retrieval for simulated-society agents.
- **Agent memory files** ([09-memory-files.md](09-memory-files.md)) are the persistent-across-sessions analogue: reflections that outlive the episode lifecycle and apply across *tasks*.
- **RL from AI Feedback (RLAIF).** Reflexion's verbal lessons can be converted into preference data for actual RL fine-tuning.

## Failure modes & anti-patterns

- **Generic lessons.** "Be more thorough." "Think step by step." These add noise without changing behavior. Fix: require lessons to name the specific observation and the specific rule.
- **Over-fitting to one failure.** After a single edge case, the agent adopts a rule that hurts the common case. Fix: annotate each reflection with "applies when X"; retire reflections that haven't fired in N episodes.
- **Context bloat.** Reflections accumulate to thousands of tokens; the actor reads the whole list every episode. Fix: cap (keep last 3–5), or retrieve semantically relevant reflections for the current task.
- **Shared-blind-spot recursion.** Actor and reflector are the same model with the same blind spots; the reflection rationalizes the failure instead of correcting it. Fix: use a different model for reflection, or ground the reflection in external verdicts (test runner, not LLM judge).
- **Reflection loops on unlearnable failures.** Some failures are environment bugs or ambiguous specs; no lesson will change that. Fix: detect repeated "same failure, new lesson, same failure" cycles and escalate to a human.
- **Pre-commitment to wrong lesson.** Actor reads a stale reflection that no longer applies to the new task. Fix: scope reflections to task families, not globally.

## When to use (and when not to)

**Use** Reflexion when:

- You have an environment that produces *objective* feedback (tests pass/fail, a reward signal, an eval).
- The agent is doing the *same class* of task repeatedly, so lessons transfer.
- You can afford multiple attempts per task (reflexion assumes retry budget).

**Don't** use Reflexion when:

- Each task is one-shot with no feedback — nothing to reflect on.
- The evaluator is noisy — bad feedback produces worse-than-nothing lessons.
- You're only running one episode — the overhead of the reflector is wasted.

## References

- Shinn et al., "Reflexion: Language Agents with Verbal Reinforcement Learning", arXiv:2303.11366 — <https://arxiv.org/abs/2303.11366>
- Reflexion GitHub (official) — <https://github.com/noahshinn/reflexion>
- Lilian Weng, "LLM Powered Autonomous Agents" — <https://lilianweng.github.io/posts/2023-06-23-agent/>
- Park et al., "Generative Agents: Interactive Simulacra of Human Behavior", arXiv:2304.03442 — <https://arxiv.org/abs/2304.03442>
- "CRITIC: LLMs Can Self-Correct with Tool-Interactive Critiquing", arXiv:2305.11738 — <https://arxiv.org/abs/2305.11738>
