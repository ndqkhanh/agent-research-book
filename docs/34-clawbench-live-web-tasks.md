# 34 — ClawBench: Benchmarking Agents on Live Web Tasks

**Definition.** ClawBench (arXiv:2604.08523, also <https://claw-bench.com/>) is a benchmark of 153 everyday online tasks — purchases, appointments, job applications — executed on **144 live production websites** across 15 categories. Unlike WebArena or SimplerEnv, ClawBench uses real sites with a lightweight interception layer that blocks only the final submission to keep evaluation safe without sacrificing realism. Even Claude Sonnet 4.6 achieves only **33.3% success**, a sober snapshot of the state of consumer web agents.

## Problem it solves

Benchmarks that use simulated websites let agents get good at the simulation, not the real web. Real sites have popups, captchas, design changes, A/B tests, auth flows, rate limits, and non-deterministic content. Prior "live web" evaluations were small or unsafe (agents could actually send emails, complete purchases). ClawBench threads the needle: real sites, real complexity, but submission interception prevents the agent from actually placing orders or submitting applications.

The 33.3% headline is as much a product insight as a research finding. Everyday web automation is *much harder than demos suggest*, and the gap between hype and delivery on "do-everything agents" is large.

## Mechanism

Benchmark structure:

1. **153 tasks, 15 categories.** Tasks drawn from real user needs: buy a specific product, book an appointment, apply for a job, sign up for a service, track a shipment, set a reminder.
2. **144 live production platforms.** Amazon, Expedia, LinkedIn, DoorDash, and the long tail of vertical-specific sites.
3. **Submission interception.** A proxy / browser-extension layer captures the final submission request (payment, form POST) without forwarding it. Success is determined by whether the intercepted payload would have completed the task.
4. **Task prompts with user-attached documents.** Some tasks require the agent to extract information from uploaded files (resumes, PDFs) before acting on the site.
5. **Scoring.** Binary success per task; aggregates per category and per model. Cost, latency, and step-count are reported.

Failure modes dominating the 67%: DOM changes breaking selectors, auth/captcha gating, multi-site workflows (the agent can't recover from a page change mid-task), and ambiguity in user intent.

## Concrete pattern

For teams building a web agent, adopt ClawBench's operational discipline:

```
Eval harness:
  - run against live sites, not a snapshot.
  - intercept *only* the final commit; all prior reads/clicks are real.
  - rerun weekly — the web changes under you.
  - categorize failures: DOM, auth, intent, recovery, environment.

Deployment discipline:
  - HITL on every commit ([23-human-in-the-loop.md](23-human-in-the-loop.md))
    until success rate clears an internal bar.
  - Rollback on DOM-change failure spikes.
  - Per-site performance dashboards.
```

## Variants & related techniques

- **WebArena / VisualWebArena** — earlier sandboxed web benchmarks; cleaner but distant from real web reliability.
- **Mind2Web / Online-Mind2Web** — large-scale but often evaluated offline.
- **LinuxArena** ([26-linuxarena-production-agent-safety.md](26-linuxarena-production-agent-safety.md)) — same idea, different surface (production Linux instead of consumer web).
- **Claw-Eval** ([38-claw-eval.md](38-claw-eval.md)) — trajectory-aware evaluation framework from the same lineage; can run ClawBench-style suites with richer attribution.
- **Human-in-the-loop** ([23-human-in-the-loop.md](23-human-in-the-loop.md)) — compensating control until win rates are acceptable.

## Failure modes & anti-patterns

- **Overfitting to benchmark sites.** If the agent is tuned specifically for the 144 ClawBench sites, real production use (a different mix) will underperform. Fix: hold out sites; rotate.
- **Treating 33% as shippable.** It's shippable only with HITL on the intercepted commit. Autonomous commit at 33% success ships two failed attempts per successful one.
- **Ignoring cost.** A successful web task can be dozens of LLM calls plus browser actions. Track $/task.
- **Submission-interception leaks.** A misconfigured interceptor lets a payment through. Test the interceptor itself.
- **Sensitive-data exposure.** The agent sees the user's real data on real sites; logging or trace storage must treat it as PII.
- **Captcha-drift blind spots.** Sites deploy new captchas; week-to-week success rates drop silently. Alert on per-site degradation.

## When to use (and when not to)

**Useful** for teams:

- Building consumer web automation products (assistants, RPA, auto-apply tools).
- Needing a realistic calibration of how far current agents really are.
- Designing HITL boundaries around high-stakes commits.

**Not useful** for:

- Enterprise agents operating on internal tooling where APIs exist — use those instead of HTML automation.
- Pure reasoning benchmarks; web-specific failures dominate.

## References

- arXiv:2604.08523 — "ClawBench". <https://arxiv.org/abs/2604.08523>
- ClawBench project site — <https://claw-bench.com/>
- Claw-Eval (arXiv:2604.06132). <https://arxiv.org/abs/2604.06132>
- WebArena. <https://webarena.dev/>
- LinuxArena (arXiv:2604.15384). <https://arxiv.org/abs/2604.15384>
