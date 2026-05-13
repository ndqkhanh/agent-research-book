# 321 - Spec-Driven Development and BMAD for AI Coding: Deep Synthesis

**Scope.** A synthesis of Spec-Driven Development (SDD), GitHub Spec Kit, BMAD Method, and adjacent TDD/BDD/ATDD practices for AI-driven software development. The supplied anchors are [github/spec-kit](https://github.com/github/spec-kit), GitHub's [Spec Kit docs](https://github.github.com/spec-kit/), Vishal Mysore's SDD-vs-TDD and BMAD introductions, and [bmad-code-org/BMAD-METHOD](https://github.com/bmad-code-org/BMAD-METHOD).

**Thesis.** AI coding changes the cost model of ambiguity. A vague human requirement used to create meetings and rework. A vague AI-agent requirement creates thousands of plausible lines that compile, pass shallow tests, and encode decisions nobody made. SDD and BMAD are two answers to the same problem:

1. **SDD / Spec Kit:** make intent the source of truth, then derive plan, tasks, and implementation from it.
2. **BMAD:** make the development process itself agent-readable through role-specialized personas, phased workflows, handoffs, and project artifacts.
3. **TDD/BDD/ATDD:** keep executable examples and regression checks as the verification layer, not as a replacement for deciding what to build.

The best AI-era workflow is not "specs instead of tests." It is **spec-first, examples-as-checks, tests-as-regression, and agents-as-implementers under review gates**.

---

## 0. Reading Map

If you only have one hour:

1. **Start with the problem:** ambiguity is the main failure mode of AI coding.
2. **Read Spec Kit:** `Specify -> Plan -> Tasks -> Implement`.
3. **Read BMAD:** Analysis -> Planning -> Solutioning -> Implementation, with specialized agents.
4. **Compare with BDD/TDD/ATDD:** examples and tests verify behavior, but do not discover all intent alone.
5. **Read the evidence:** CodeSpecBench, Self-Spec, CodeAct.
6. **Use the research gaps:** spec drift, spec verification, BMAD-vs-SpecKit, executable acceptance specs.

---

## 1. The Core Problem: AI Makes Ambiguity Expensive

Traditional software development already suffered from requirements drift, hidden assumptions, and mismatched interpretations between product, design, engineering, QA, and operations. AI coding amplifies that problem because the model does not leave ambiguity blank. It fills gaps with likely defaults.

Example: "Allow a user to cancel a book loan" hides many requirements:

| Hidden question | If not specified, AI may guess |
|---|---|
| Who can cancel? | member only, staff only, both, admin only |
| What happens to fees? | cancel fees, keep fees, recalculate fees |
| Is history preserved? | delete row, soft-delete, archive event |
| What audit trail exists? | none, basic log, immutable event |
| What state transitions are valid? | any -> cancelled, only active -> cancelled |

TDD can verify chosen behavior, but it cannot test decisions the team never made. A spec-first process forces those decisions into durable artifacts before the coding agent writes implementation.

---

## 2. Definitions

| Term | Core idea | AI-era interpretation | Best use |
|---|---|---|---|
| **SDD** | specifications drive implementation | specs become living, generative artifacts for agents | greenfield, feature work, modernization |
| **GitHub Spec Kit** | toolkit for SDD | CLI/templates/commands for spec, plan, tasks, implement | cross-agent spec-first workflow |
| **BMAD Method** | Breakthrough Method for Agile AI-Driven Development / Build More Architect Dreams | persona-driven agile AI development with artifacts and handoff gates | broader lifecycle and multi-agent process |
| **TDD** | write failing test, implement, refactor | micro-level implementation safety and design pressure | algorithms, units, regression safety |
| **BDD** | examples describe behavior in shared language | collaborative discovery and executable examples | cross-role alignment and acceptance behavior |
| **ATDD** | acceptance criteria before implementation | acceptance tests derived from agreed requirements | feature acceptance and product validation |
| **Design by Contract** | preconditions/postconditions/invariants | executable behavioral specs for agents | spec verification and API semantics |

---

## 3. Master Comparison Table

| Dimension | GitHub Spec Kit / SDD | BMAD Method | TDD | BDD / ATDD |
|---|---|---|---|---|
| **Primary unit** | specification, plan, task | agent workflow, persona, artifact | test case | scenario / acceptance example |
| **Main question** | what should be built and why? | who should produce which artifact next? | does this code behave correctly? | what behavior do stakeholders expect? |
| **Workflow** | specify -> plan -> tasks -> implement | analysis -> planning -> solutioning -> implementation | red -> green -> refactor | discovery -> formulation -> automation |
| **AI role** | generate artifacts and code from structured intent | act as Analyst, PM, Architect, Dev, QA, Writer | generate tests or code in micro loops | convert examples into checks |
| **Human role** | critique and approve each artifact | facilitate and validate handoffs | decide next behavior and review design | align stakeholders and examples |
| **Artifact shape** | Markdown specs, plans, tasks, constitution | PRD, UX spec, architecture, stories, sprint YAML, retros | unit/integration tests | Gherkin or example docs |
| **Strength** | reduces prompt ambiguity | full lifecycle structure | tight implementation feedback | shared language and acceptance clarity |
| **Failure mode** | doc sprawl, stale specs, false completeness | ceremony creep, artifact drift | tests for wrong requirements | automation theater without discovery |
| **Best pairing** | SDD + BDD examples + tests | BMAD + Spec Kit conventions + QA module | TDD under SDD tasks | BDD examples inside SDD specs |

---

## 4. GitHub Spec Kit

GitHub Spec Kit is a toolkit for SDD. It frames specifications as the center of AI-assisted development, not as discarded planning documents. The current public docs describe the core process as:

```text
Spec -> Plan -> Tasks -> Implement
```

| Phase | Purpose | Typical artifact | Human gate |
|---|---|---|---|
| **Specify** | define what to build and why | feature specification | does this capture intent and user journeys? |
| **Plan** | choose technical path and constraints | implementation plan, data model, contracts | does this fit architecture, security, stack, and performance constraints? |
| **Tasks** | split work into small implementable chunks | task list | are tasks reviewable and testable in isolation? |
| **Implement** | agent writes code task-by-task | code, tests, docs | does implementation match spec and pass checks? |

### Spec Kit Surface

| Feature | Notes |
|---|---|
| **CLI** | `specify init` bootstraps the workflow |
| **Agent integrations** | docs list many integrations, including Copilot, Gemini, Codex, Claude, Windsurf, Kiro, and generic |
| **Templates** | spec, plan, tasks, agent context, scripts |
| **Constitution** | project principles and non-negotiable constraints |
| **Community extensions** | presets, alternate processes, architecture guards, CI guards |
| **Use cases** | greenfield, brownfield feature work, legacy modernization, parallel exploration |

### Why It Matters

Spec Kit turns the agent's context from an ad-hoc prompt into a staged artifact chain. Each stage reduces guessing:

```text
human intent -> spec -> technical plan -> task graph -> implementation
```

The important design choice is that each artifact is reviewable before the next one is generated.

---

## 5. BMAD Method

BMAD Method is a broader AI-driven development framework. The public docs describe BMad as an AI-driven development framework module that supports the whole process from ideation and planning to agentic implementation. The repository tagline frames it as "Breakthrough Method for Agile Ai Driven Development," while the docs brand the ecosystem as "Build More Architect Dreams."

### Default BMAD Agents

| Agent | Role | Typical workflows |
|---|---|---|
| **Analyst / Mary** | research, brainstorm, briefs, PRFAQ | market research, domain research, technical research, create brief |
| **PM / John** | requirements and backlog | PRD, epics, stories, implementation readiness |
| **Architect / Winston** | architecture and readiness | architecture, ADRs, solution review |
| **Developer / Amelia** | implementation | dev story, quick dev, QA generation, code review |
| **UX Designer / Sally** | design | UX design |
| **Technical Writer / Paige** | documentation | docs, standards, diagrams, validation |

### BMAD Workflow Map

| Phase | Goal | Outputs |
|---|---|---|
| **Analysis** | explore idea and problem space | brainstorming report, research notes, product brief, PRFAQ |
| **Planning** | define product direction | PRD, UX spec |
| **Solutioning** | design technical path and decompose work | architecture, ADRs, epics, stories, readiness gate |
| **Implementation** | build iteratively | sprint status, story files, code, tests, reviews, retros |
| **Quick flow** | shortcut for small work | `spec-*.md` plus code |

BMAD's key move is not just "write a spec." It turns the process into an agent-readable organization: personas, commands, dependencies, templates, checklists, and handoff prompts.

<details>
<summary>BMAD as context engineering</summary>

BMAD is context engineering expressed as a development methodology:

- **Write:** each phase produces durable artifacts.
- **Select:** the orchestrator or user chooses the right agent/persona.
- **Compress:** each handoff packages prior context into the next artifact.
- **Isolate:** each agent gets a focused role and command surface.

This is why BMAD fits the wider harness-engineering corpus. It is not only a methodology; it is a way of shaping the LLM's operating context.

</details>

---

## 6. Spec Kit vs BMAD

| Dimension | Spec Kit | BMAD |
|---|---|---|
| **Center of gravity** | spec-first artifacts | role-specialized agile workflow |
| **Main abstraction** | spec, plan, tasks | agents, workflows, artifacts |
| **Workflow style** | compact and SDD-native | broader and ceremony-rich |
| **Best fit** | feature specs, brownfield changes, greenfield starts | larger projects needing product/architecture/story handoffs |
| **Agent model** | any coding agent with commands/templates | named agents with personas and menus |
| **Testing posture** | tasks should be implementable and testable | dev and QA generation hooks, TEA module for deeper testing |
| **Governance** | constitution, templates, extension/preset controls | project-context, readiness gates, workflow artifacts |
| **Risk** | spec sprawl and review fatigue | role/process overhead and artifact drift |
| **Can combine?** | yes | use BMAD for lifecycle, Spec Kit conventions for spec/plan/tasks |

### Practical Decision Tree

| Situation | Prefer |
|---|---|
| Small feature in existing repo | Spec Kit or BMAD quick flow |
| New product with fuzzy requirements | BMAD Analysis + Planning, then Spec Kit-style tasks |
| Regulated enterprise app | Spec Kit constitution + BMAD readiness/QA gates |
| Team wants named AI teammates | BMAD |
| Team wants lightweight spec-first CLI | Spec Kit |
| Need multi-agent product workflow | BMAD |
| Need cross-agent templates and broad integrations | Spec Kit |

---

## 7. SDD, TDD, BDD, and ATDD Should Compose

Spec-first does not eliminate TDD or BDD. It changes where they sit.

```text
Spec-first decides intent.
BDD examples clarify behavior.
ATDD turns acceptance into checks.
TDD drives implementation detail.
CI verifies regressions.
```

| Layer | Artifact | Who owns it | What it prevents |
|---|---|---|---|
| **Intent** | SDD spec | product + engineering | building the wrong thing |
| **Behavior** | BDD examples | product + QA + engineering | mismatched interpretation |
| **Acceptance** | ATDD checks | QA + engineering | unaccepted implementation |
| **Implementation** | TDD tests | engineering | low-level defects |
| **Operations** | observability and runbooks | SRE + engineering | silent production drift |

Dan North's BDD framing matters here because it was originally a response to confusion around TDD: what to test, how to name tests, and how to connect tests to business behavior. Fowler's Given-When-Then and Cucumber's Discovery/Formulation/Automation flow are natural companions to SDD because they turn stakeholder examples into executable specifications.

---

## 8. Evidence and Research Signals

| Work | Core idea | What it says for SDD |
|---|---|---|
| [CodeSpecBench](https://arxiv.org/abs/2604.12268) | benchmark executable pre/postcondition generation at function and repository level | spec generation is harder than code generation; repository-level best pass rate is low, so human review is mandatory |
| [Self-Spec](https://openreview.net/forum?id=6pr7BUGkLp) | model writes its own task-specific spec schema before code | structured intermediate artifacts can improve code generation for some strong models |
| [CodeAct](https://proceedings.mlr.press/v235/wang24h.html) | executable Python actions as agent actions | executable artifacts often outperform rigid text/JSON action formats |
| [BDD](https://dannorth.net/blog/introducing-bdd/) | behavior vocabulary and acceptance scenarios | shared language reduces ambiguity before implementation |
| [Given-When-Then](https://martinfowler.com/bliki/GivenWhenThen.html) | scenario structure for behavior examples | SDD specs need example-backed acceptance criteria |
| [Cucumber BDD](https://docs.cucumber.io/bdd/) | Discovery, Formulation, Automation | examples become checked living documentation |

### The Hard Truth from CodeSpecBench

Executable specs are promising, but current LLMs are not reliably good at generating them in realistic repository contexts. CodeSpecBench reports a sharp performance drop at repository level and argues that strong code generation does not imply deep understanding of intended semantics.

This is the strongest warning against naive SDD hype: **the spec must be reviewed and tested too**.

---

## 9. Failure Modes

| Failure mode | How it appears | Mitigation |
|---|---|---|
| **Spec drift** | code changes but spec is not updated | require spec diff in PR, CI checks for spec/task linkage |
| **False completeness** | spec looks detailed but misses key decisions | adversarial review, checklist-driven ambiguity scan |
| **Over-specification** | 60 questions or huge docs slow work | risk-based spec depth, quick flow for small tasks |
| **Review fatigue** | humans approve walls of AI text | concise artifacts, diff views, structured decision tables |
| **Implementation lock-in** | plan choices become premature constraints | multi-plan exploration before tasks |
| **Automation theater** | BDD scenarios exist but were not discovered collaboratively | discovery workshops or explicit stakeholder review |
| **Test/spec mismatch** | tests verify implementation, not intent | derive acceptance tests from spec examples |
| **Agent role confusion** | generic agent mixes PM, architect, dev, QA | BMAD-style role isolation |
| **Artifact sprawl** | specs, PRDs, stories, plans, tasks duplicate each other | artifact ownership and canonical source rules |
| **Spec hallucination** | AI invents requirements while drafting | human verification gates and source tagging |

---

## 10. OSS and Tooling Landscape

| Tool / repo | Role | Popularity signal | Best use | Limitation |
|---|---|---:|---|---|
| [github/spec-kit](https://github.com/github/spec-kit) | SDD toolkit | very high, GitHub-backed | spec/plan/tasks workflow with many agent integrations | still experimental and artifact-heavy |
| [Spec Kit docs](https://github.github.com/spec-kit/) | reference docs | official | process, install, integrations, extensions | vendor framing |
| [BMAD-METHOD](https://github.com/bmad-code-org/BMAD-METHOD) | agile AI-driven development framework | very high community repo | persona-driven lifecycle and workflows | ceremony and version churn |
| [BMAD docs](https://docs.bmad-method.org/) | reference/tutorials | official | agents, workflow map, customization | docs evolve quickly |
| [CodeSpecBench](https://github.com/SparksofAGI/CodeSpecBench) | executable spec benchmark | research repo | measuring spec-generation ability | young benchmark |
| [Cucumber](https://docs.cucumber.io/bdd/) | BDD tooling and docs | established ecosystem | executable examples and living docs | tool use alone is not BDD |
| [JBehave](https://jbehave.org/) | BDD lineage | established | Java BDD scenarios | ecosystem-specific |
| [Gherkin](https://cucumber.io/docs/gherkin/) | scenario language | established | readable acceptance specs | can become brittle if abused |

---

## 11. Adoption Patterns

### Pattern A: Lightweight Spec Kit Feature Flow

```text
1. /specify feature intent
2. review spec
3. /plan architecture constraints
4. review plan
5. /tasks small chunks
6. implement task-by-task
7. derive tests from acceptance examples
```

Best for: single feature, brownfield change, small greenfield app.

### Pattern B: BMAD Full Lifecycle Flow

```text
1. Analyst researches and creates brief
2. PM creates PRD and epics
3. UX creates design when needed
4. Architect creates architecture and readiness check
5. Scrum/story workflow prepares dev-ready stories
6. Developer implements stories
7. QA/test architect validates risk and coverage
```

Best for: complex projects, teams that want role separation, product-to-engineering handoffs.

### Pattern C: Hybrid SDD + BDD + TDD

```text
1. SDD spec captures intent
2. BDD examples capture behavior
3. ATDD checks capture acceptance
4. TDD tests guide implementation internals
5. CI and review keep spec/code/tests synchronized
```

Best for: production systems where correctness and maintainability matter more than fast demo generation.

---

## 12. Research Gaps and Next-Paper Ideas

| Rank | Gap | Paper idea | First experiment |
|---:|---|---|---|
| 1 | spec drift in AI coding | **SpecDriftBench**: measure divergence among spec, tasks, tests, and code over agent edits | run agents on 100 feature changes with required spec updates |
| 2 | agent-generated spec reliability | **SpecCritic**: verifier for AI-written specs before codegen | compare human vs LLM critique on ambiguous specs |
| 3 | BMAD vs Spec Kit | **ProcessBench-AI**: controlled comparison of lifecycle methods | same project built under vibe, Spec Kit, BMAD, hybrid |
| 4 | executable acceptance specs | **AcceptSpecBench**: BDD/ATDD examples as executable contracts for agents | generate Gherkin + tests from feature specs |
| 5 | multi-plan exploration | **PlanDiversity-SDD**: compare parallel plans before implementation | measure cost, bugs, architecture fit |
| 6 | spec compression | **SpecContext-R**: select/compress long specs for coding agents | ablate full spec vs selected relevant clauses |
| 7 | spec security | **SecureSpec**: detect unsafe or missing requirements before implementation | inject ambiguous auth/privacy/compliance clauses |
| 8 | spec-to-test traceability | **TraceSpec**: map every test and code change to a spec clause | evaluate missing and stale links in PRs |

### Recommended Next Paper

**SpecDriftBench** is the strongest next-paper candidate. It connects your existing corpus themes:

- memory/context engineering: specs as durable context,
- agent skills: spec-writing and spec-review as reusable skills,
- evaluation: traceable divergence metrics,
- security: missing requirements and unsafe assumptions,
- harness engineering: gates before code execution.

---

## 13. Practical Checklist for Teams

| Question | If yes | If no |
|---|---|---|
| Is the change user-visible? | require spec | use quick task note |
| Does it alter data, auth, money, privacy, or compliance? | require explicit constraints | continue |
| Will AI generate most code? | require spec/plan/tasks | still write design note |
| Is the system brownfield? | include integration constraints | greenfield plan can explore variants |
| Are there acceptance examples? | convert to BDD/ATDD tests | run discovery step |
| Is the spec longer than the code? | compress and prioritize | proceed |
| Are tests traceable to spec clauses? | good | add traceability table |
| Can a reviewer see what changed in intent? | good | add spec diff |

---

## 14. Sources

### Supplied Anchors

- [github/spec-kit](https://github.com/github/spec-kit)
- [GitHub Spec Kit docs](https://github.github.com/spec-kit/)
- [What is Spec-Driven Development?](https://github.github.com/spec-kit/concepts/sdd.html)
- [Spec-driven development with AI: Get started with a new open source toolkit](https://resources.github.com/increasing-collaborative-development-with-ai/)
- [SDD vs TDD in the Age of AI](https://medium.com/@visrow/sdd-vs-tdd-in-the-age-of-ai-why-spec-first-development-is-taking-over-0c78b90921f1)
- [BMAD-METHOD](https://github.com/bmad-code-org/BMAD-METHOD)
- [BMAD docs](https://docs.bmad-method.org/)
- [BMAD agents reference](https://docs.bmad-method.org/reference/agents/)
- [What is BMAD-METHOD?](https://medium.com/@visrow/what-is-bmad-method-a-simple-guide-to-the-future-of-ai-driven-development-412274f91419)

### Adjacent Practice and Research

- [Diving Into Spec-Driven Development With GitHub Spec Kit](https://developer.microsoft.com/blog/spec-driven-development-spec-kit)
- [CodeSpecBench](https://arxiv.org/abs/2604.12268)
- [CodeSpecBench repo](https://github.com/SparksofAGI/CodeSpecBench)
- [Self-Spec](https://openreview.net/forum?id=6pr7BUGkLp)
- [CodeAct](https://proceedings.mlr.press/v235/wang24h.html)
- [Introducing BDD](https://dannorth.net/blog/introducing-bdd/)
- [Given When Then](https://martinfowler.com/bliki/GivenWhenThen.html)
- [Cucumber BDD](https://docs.cucumber.io/bdd/)
- [Gherkin](https://cucumber.io/docs/gherkin/)

---

## 15. Final Takeaways

1. **AI coding makes ambiguity the main bottleneck.**
2. **Spec Kit is the lightweight SDD reference implementation.**
3. **BMAD is a broader agentic agile operating system.**
4. **TDD/BDD/ATDD remain essential as verification layers.**
5. **Executable specs are promising but current models still struggle to generate them reliably.**
6. **The missing benchmark is not "can AI code from a spec?" but "can AI keep spec, plan, tasks, tests, and code aligned over time?"**
7. **The best next paper is SpecDriftBench.**
