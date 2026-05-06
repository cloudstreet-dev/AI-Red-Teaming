# The Asymmetry Problem

This is the closing chapter, returning to the framing the book opened with. Chapter 1 named the Mythos moment and the capability gap it made visible. The eleven chapters in between have been concrete: techniques to use, defenses to build, tools to adopt. This chapter zooms back out and asks what to do about the gap itself, given that closing it from the defensive side is not on the table for most engineers in any near-term timeframe.

The honest version of the asymmetry: offensive AI capability is closer to general availability than defensive Mythos-level scanning is. The frontier offensive capability is gated behind partner programs that you and I are not in. The frontier defensive capability is also gated behind those partner programs. The publicly available models can do useful work on both sides, but they do less of it on the defensive side, partly because the offensive side has structural advantages (one bug suffices; choose your target; choose your timing) and partly because the offensive techniques have been more thoroughly published than the defensive ones.

Waiting for parity is not a strategy. It might never arrive. Even if it does, the products you ship between now and then will have shipped vulnerable. The work is in the gap.

## What defenders should focus on while the gap is open

Five categories of work compound regardless of how the gap evolves. None of them require you to have access to a frontier offensive model. All of them are achievable by a small team with publicly available tools.

**Defense in depth assumed, not aspired to.** Every defense you have should be considered partial. The system prompt is partial. The guard model is partial. The output sanitization is partial. The tool allowlist is partial. The user authentication is partial. *Each one is a probability, not a guarantee.* The architecture should compose them so that defeating any single defense does not produce catastrophic outcomes. This is what defense in depth has always meant; the AI surface makes it more urgent because the per-defense probability of failure is higher than for the conventional surfaces. Architect under the assumption that any layer can fail and the next one needs to catch the failure.

**Audit cadence over audit depth.** A weekly automated red-team run that catches regressions is more valuable than a quarterly deep audit that produces a report. The deep audits have their place — particularly for the conventional code surface, where a frontier-model-augmented pass produces findings that the weekly run does not — but the cadence is what catches the bugs that get introduced between audits. A static-frequency cadence (nightly is best, weekly is fine, anything less than monthly is too slow) is the discipline the work requires.

**The two surfaces, separately enumerated.** Keep separate threat models, separate test plans, separate metrics for the AI surface and the conventional code surface. They fail in different ways and the consolidating-them-into-one effort produces a single document that is unhelpful on both fronts. Two short documents, kept current, beat one long document that nobody reads.

**Logging that lets the SOC see what the model did.** Most attacks against AI features in production are detected, when they are detected, by anomaly detection on tool-call patterns, output signatures, and error-rate spikes. The SOC's tools were not built for this; they need new event sources. Make sure the events exist. Tool calls with full arguments, output URLs that were emitted, sessions where the assistant followed instructions that did not appear in the user's input — these are the events that make detection possible. They are also, conveniently, the events that make incident response possible after the fact.

**The engineering practices that compound.** Regression tests for every fix. Threat-model updates for every novel finding. Class-search for every instance bug. Disciplined dependency management. The practices that make conventional code less vulnerable also make the AI feature integration points less vulnerable, because the AI feature lives in conventional code. The team that has these practices for its non-AI work has, by default, a stronger foundation for its AI work. The team that doesn't has compounding fragility.

These five are the work for as long as the gap is open. None of them depends on Mythos-level capability becoming available; all of them benefit from incremental improvements in the publicly available models, and all of them also benefit from improvements that have nothing to do with AI.

## What changes when the gap closes

It will close, eventually, partially. The mechanisms I'd watch:

- **Releases of comparable models.** Open-weights models — Llama, the Qwen family, DeepSeek's continuing releases — keep narrowing the gap to the closed frontier. A Llama or DeepSeek release in the next 12 to 24 months that is genuinely competitive with current closed-frontier models on offensive security tasks would change the threat model for every defender. The capability would be in everyone's hands, including attackers'; the defender's reaction time would be the bottleneck.
- **Specialist defensive tools that wrap the public models.** The DeepTeam-and-friends ecosystem from chapter 10 is the early version of this. The next generation will be more capable agentic loops specifically tuned for vulnerability discovery in the defender's own code, with the patient, persistent, multi-day analysis style that Mythos exhibits. Most of the components for this exist publicly today; the engineering work to compose them well is in progress.
- **Eventual public release or broader licensing of Mythos-class models.** Anthropic has not committed to a timeline. The CETaS analysis from chapter 1 estimates that some form of broader access is likely within 18 to 36 months of the original announcement, with high uncertainty. The structure of the eventual release matters: a Mythos that ships to enterprises with stringent safeguards is not the same threat model as a Mythos whose weights leak.
- **Regulatory action that changes the equilibrium.** Several jurisdictions, including the EU and California, are at varying stages of legislation that would mandate disclosure of high-impact AI capabilities, restrict deployment of unaudited models in critical sectors, or require coordinated disclosure of certain classes of AI-discovered vulnerabilities. The probability that the equilibrium changes by political mechanism is non-trivial. The direction is unclear.

When the gap closes — partially, unevenly, on whatever timeline — the immediate effect for defenders will be that the asymmetry shifts: the per-bug discovery cost drops on both sides. The teams that have invested in the compounding practices above will absorb the shift; the teams that have not will discover that their products were already exposed, just by attackers who had not yet gotten around to them.

## The defensive case for using AI to attack your own code

It is worth stating explicitly: the case for incorporating AI-assisted vulnerability discovery into your defensive practice is not "AI will find every bug." The case is that the attacker's marginal cost to find your bugs is dropping, which means your marginal cost to find them first must also drop, which means your audit budget should buy more bugs found per dollar than it currently does. AI-assisted auditing is the way to make the budget go further. Not because the model is a good security researcher in absolute terms, but because the model is a security researcher whose hourly rate is dollars rather than hundreds.

This is the asymmetry observation read backwards. The same capability shift that helps the attacker also helps you, on the defense side, to a lesser extent. The lesser extent is large enough to matter. The defenders who use it will be in a meaningfully better position than the defenders who do not.

## A short closing note

I have tried throughout this book to be honest about what is and isn't in the reader's hands. The Mythos moment was a real shift, and pretending otherwise would have been dishonest. Most of what's in the reader's hands is also real, and pretending it is inadequate would have been the other kind of dishonest. The work is to use what you have, well, with the awareness that what you have is not all there is.

That awareness is the discipline this book has been about. The vocabulary in chapter 2 is a tool for the discipline. The harness in chapter 6 is a tool for the discipline. The technique chapters are tools for the discipline. The discipline itself — the habit of writing down what your assumptions are, testing them on a recurring schedule with whatever tools you can muster, treating every defense as partial, watching the surfaces you cannot see directly — is the thing that compounds across the next decade of capability shifts, regardless of which side of the gap the next one falls on.

Build the habit. Ship on Tuesday. Repeat.

— Claude Opus 4.7

## Sources

- The CETaS analysis cited in chapter 1, for the timeline estimates around broader Mythos availability.
- Open-weights model release notes through 2025–2026 — Llama 4, Qwen 3.5, DeepSeek's continued releases — for the trajectory on the open side of the gap.
- The European Union AI Act implementation timelines (Article 52 and the GPAI provisions); the California SB 1047 successor legislation (currently in progress as of May 2026); the U.S. AI Safety Institute publications on coordinated disclosure norms — for the regulatory direction.
- The previous eleven chapters of this book, for everything else.
