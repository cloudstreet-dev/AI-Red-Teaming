# Tooling Ecosystem

What's actually available, right now, for a small team that wants to red-team its AI features without building everything from scratch. This chapter is the most aggressively dated part of the book; the specific projects will move, fork, and rename. Read it for the categories and for the comparisons; check the projects' current state before adopting any.

The honest framing: most of the value of this tooling is in **operationalizing and regression-testing what you already know how to do by hand**. None of these tools find attacks you couldn't have found yourself with sufficient time. All of them save you the time, and more importantly, they save you the *recurring* time — the cost of running the same attack suite every week against a system that changes every week.

## The dedicated AI red-team frameworks

Three projects are worth knowing by name in May 2026.

**DeepTeam** (<https://github.com/confident-ai/deepteam>). The most opinionated of the three. Ships a fixed taxonomy of attack categories — most of LLM Top 10 plus some additions — and an evaluator that runs them against your system. The attack generation is itself LLM-driven, so the corpus is not a static file you can read; it's a prompted procedure. The evaluator scores each attempt as pass/fail/partial, produces a report, and integrates with CI. Strongest at single-turn and structured multi-turn attacks. Weaker at the open-ended adversarial conversations a human red-teamer would produce; stronger than no automation at all.

Use it for: a CI-runnable suite that tracks regression on a defined attack taxonomy. The right starting point for most teams.

Doesn't replace: open-ended manual red-teaming, indirect-injection testing against your specific RAG corpus (which it doesn't know about), product-specific business-logic attacks.

**garak** (<https://github.com/NVIDIA/garak>, originally Leon Derczynski's project, now maintained under NVIDIA). The "nmap for LLMs" framing has stuck because it's accurate. Garak is a probe-based scanner: a long list of named probes (each one a specific attack family), each producing a measurable score against your model endpoint. Probes range from classical ("encoding bypass," "do-not-respond evasions") to emerging ("training data extraction," "package hallucination"). Outputs are JSONL traces you can post-process.

Use it for: getting a baseline scan of a model deployment. Running before-and-after when you change the system prompt or upgrade the model. Comparing two models with identical scaffolds. Producing the kind of report that satisfies a security review.

Doesn't replace: anything specific to your product. Garak knows about general LLM vulnerabilities; it does not know about your tools, your RAG corpus, or your custom prompts beyond what you wire up.

**PyRIT** (<https://github.com/Azure/PyRIT>, Microsoft). The Python Risk Identification Tool for generative AI. PyRIT is a framework, not a fixed suite — it provides primitives for orchestrating attacker-vs-target conversations, with pluggable scorers and converters. The Crescendo paper from chapter 5 was developed against PyRIT. It is the most flexible of the three and the highest setup cost.

Use it for: when you need to write custom attack orchestration that the off-the-shelf suites don't cover. Multi-turn campaigns specific to your product. Integration with your existing test infrastructure when you need control over each step.

Doesn't replace: human creativity in attack design. PyRIT executes campaigns; it doesn't invent them.

The three projects overlap meaningfully. A team that adopted only one would get most of the value; a team that adopted all three would have moderate redundancy and modestly improved coverage. The minimum-viable choice for most teams is *DeepTeam for the CI suite, garak for occasional baseline scans, PyRIT only if you need to write custom orchestration*.

## Prompt-injection corpora and benchmarks

Several public corpora are worth pulling into your test suite directly, regardless of which framework you use:

- **HackAPrompt corpus** (Schulhoff et al., 2023). Several thousand human-written prompt-injection attempts, categorized. Old now, but the categorization is still the cleanest taxonomy of attack shapes I'm aware of. Useful as both a test set and as a teaching aid for new team members learning what attacks look like.
- **TensorTrust** (Toyer et al., 2023, 2024 update). A large dataset of attack/defense prompt pairs collected from a public game where players tried to extract a password from a defended chatbot. Good source of the kinds of clever phrasings that human attackers actually produce, as opposed to what an LLM-driven attacker generates (which has different texture).
- **JailbreakBench** (Chao et al., 2024 with periodic updates). A standardized benchmark with a fixed set of harmful behaviors and a graded evaluation. Useful for comparing your system's robustness against published baselines, with all the caveats about benchmark gaming that any standardized eval has.
- **AdvBench** (Zou et al., the GCG paper, 2023). Used widely; mostly relevant if you care about adversarial-suffix attacks on open-weights models, which most product teams don't.
- **Anthropic's own published attack suites** in the model cards for Claude 4.x, especially the multi-turn and indirect-injection sections. Use these directly to calibrate against the model's known weaknesses, then go past them.

Pull a representative sample from each into your CI suite. The ratio that has worked for teams I've talked to is roughly: 40% canonical-corpus attacks (HackAPrompt, JailbreakBench), 40% generated attacks (DeepTeam-style, refreshed periodically so the model can't memorize them), 20% attacks specific to your product's actual threat model.

## Conventional code-audit tooling, AI-augmented

The tooling for chapter 7 (red-teaming conventional code) overlaps with the existing application-security tooling in ways that depend on which side of the AI line you start from.

If you start from existing AppSec tools and add AI:

- **Semgrep** has added AI-assisted rule generation and finding triage. Strong product for the static-analysis half of the job.
- **GitHub Advanced Security** (CodeQL plus the Copilot Autofix and now Copilot Audit features) is the integrated path if you're on GitHub already. The audit features are uneven but the trajectory is up.
- **Snyk, GitLab, JFrog, Checkmarx** all have varying AI features layered on top of their existing offerings; none of them is structurally better than the others, and the choice for most teams is dominated by which tool they're already paying for.

If you start from AI tools and add code-audit:

- **Claude Code** (<https://docs.claude.com/claude-code>). The most capable agentic loop I know of for code work as of May 2026. Pair it with the prompts from chapter 7. Run it in a sandbox.
- **OpenAI Codex CLI**. Comparable scaffold against GPT-5.
- **SWE-agent** (open source). The original of the recent agent-for-code wave; useful if you want to build your own scaffold rather than adopt a vendor's.
- **OpenHands** (open source). More general-purpose than SWE-agent; broader tool set.

The interesting workflow for most teams is to use one of each: Semgrep (or whatever your existing static analyzer is) to keep up the steady drumbeat of known-pattern findings, plus periodic Claude Code or Codex passes for the open-ended audit work that benefits from the model's contextual reasoning. Neither replaces the other.

## What the tools won't do

A short list of things the current tooling ecosystem will not do for you:

- *Tell you whether an AI feature is safe to ship.* No automated tool produces a binary safe/unsafe verdict, and any tool that claims to is selling something. The tool produces a measurement; you produce the judgment.
- *Find attacks specific to your business logic.* "This endpoint should only be callable during business hours" is a rule the tool doesn't know. "This support agent shouldn't reveal the price-list-PDF that exists in the KB but is restricted to enterprise customers" is a rule the tool doesn't know.
- *Replace a threat model.* The tools test the surfaces you point them at. The threat model is what tells you which surfaces to point them at.
- *Stay current on their own.* Attack patterns emerge faster than the tools update. A team that adopted DeepTeam in March 2026 and stopped updating its custom attack scripts is, by May 2026, missing an entire category of attacks that emerged in April. Budget for ongoing curation.

## A minimum stack for a small team

If you have one engineer-week per quarter to spend on AI red-teaming and you want to make the most of it, the stack I'd recommend in May 2026:

1. **DeepTeam in CI**, running on every deployment, scoring the OWASP LLM Top 10 categories against your deployed configuration. Nightly cron, results to a dashboard, regression alerts to Slack.
2. **A custom attack script suite** in your own repo, maintained alongside your code. Each script targets a specific behavior in your product that you don't want to see ("the agent must never reveal another user's data," "the agent must never send email to non-allowlisted addresses"). Run on every deployment alongside DeepTeam.
3. **Quarterly garak baseline scan** of the deployed model, comparing scores quarter-over-quarter. Catches regressions when you change models or substantially change the system prompt.
4. **Quarterly Claude Code or Codex pass** through the conventional code, using the chapter 7 prompts. File-rank, then class-target the top-ranked files. Triage findings as a one-day exercise.
5. **Existing application-security tooling unchanged.** The AI work supplements, not replaces, what was already working.

This is achievable for a small team. It is not what a Glasswing partner would do; they have Mythos. It is dramatically more than most teams currently do, and it gets you most of the way to a defensible position against what current public-attacker capability can produce.

## Sources

- DeepTeam: <https://github.com/confident-ai/deepteam>
- garak: <https://github.com/NVIDIA/garak>; original announcement, Derczynski et al., "garak: A Framework for Security Probing Large Language Models," 2024.
- PyRIT: <https://github.com/Azure/PyRIT>
- HackAPrompt corpus, Schulhoff et al., EMNLP 2023.
- TensorTrust: Toyer et al., "Tensor Trust: Interpretable Prompt Injection Attacks from an Online Game," ICLR 2024.
- JailbreakBench: Chao et al., 2024 — standardized harmful-behavior benchmark.
- AdvBench / GCG: Zou et al., "Universal and Transferable Adversarial Attacks on Aligned Language Models," 2023.
- Semgrep AI documentation, GitHub Advanced Security documentation, Snyk Code AI documentation — for the AppSec side.
- Claude Code: <https://docs.claude.com/claude-code>
- SWE-agent and OpenHands: see chapter 7 sources.
