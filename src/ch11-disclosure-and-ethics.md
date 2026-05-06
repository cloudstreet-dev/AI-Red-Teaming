# Disclosure and Ethics

The capability to find vulnerabilities has outrun the social infrastructure for handling them. This chapter is about the gap.

When you find a bug in your own code, the answer is straightforward: fix it, regression-test the fix, ship. When you find a bug in someone else's code — increasingly easy to do, increasingly tempting — the answer is a coordination problem with conventions older than this book and complications newer than they should be. Project Glasswing has formalized one set of conventions for the partner program; the rest of us are figuring it out as we go.

The honest version: the social infrastructure for AI-discovered vulnerabilities is being built in real time, by the same people who are using it. This chapter describes where things stand in May 2026 and what the operative norms are. The norms will change. The principles — minimize harm, get patches in users' hands, don't make the maintainer's life worse than it has to be — will not.

## When you find something in your own code

The easy half. Six steps:

1. **Reproduce it cleanly.** A finding without a reliable reproduction is a candidate, not a bug. The model's report is not a reproduction; the working test case is. If you cannot make the bug fire on demand, do not start fixing it. Fixing a bug you cannot reproduce produces a fix you cannot verify.

2. **Bound the impact.** What can an attacker do with this? Read data, write data, execute code, escalate privilege, deny service? Document the impact in the same place you document the bug. The impact bound determines the urgency, the patch testing rigor, and whether you need to disclose to users.

3. **Fix it.** With the same care you'd fix any other bug, plus a higher than usual bar for "did the fix actually fix it." Many AI-found bugs have subtle preconditions; the obvious fix may close the bug as the model described it while leaving an adjacent variant open. The model can help you check this — "given this fix, what variations of the original bug would still work?" — and is reasonably good at it.

4. **Write the regression test.** Mandatory, not optional. The test fails against the unfixed code, passes against the fix. Add it to your CI suite. Now this specific bug class has a tripwire that costs nothing per run and catches every regression that reintroduces it.

5. **Look for the class, not just the instance.** If the model found one path-traversal in your code, you almost certainly have others. Run a class-targeted pass (chapter 7) before closing the work. The cost of finding the cluster all at once is dramatically lower than finding them one at a time across six months.

6. **Update your threat model.** The bug existed because some assumption in the model was wrong, or absent. Write down what assumption changed. The threat model is a living document; the bug is data.

The first three steps are obvious. Steps 4 through 6 are the ones that compound. A team that runs them disciplined gets meaningfully harder to attack over time. A team that fixes individual bugs without the regression-test, class-search, or threat-model-update steps fixes the same bug in different forms repeatedly.

## When you find something in someone else's code

Harder. The default norm in the security community is *coordinated disclosure*: you report the bug to the maintainer privately, give them a window to fix it, and publish only after either the fix ships or the window expires. The standard windows have been refined over decades:

- **Google Project Zero**: 90 days, with a 14-day grace period after a fix is announced, optional 30-day extension if substantive progress is being made and a fix needs more time.
- **CERT/CC**: 45 days as the default coordinated-disclosure window.
- **Project Glasswing's published convention**: 90 days, 45-day extension allowed if the maintainer demonstrates active patch development. This is the longest of the major windows and was negotiated specifically for the case where the disclosed bug requires architectural rework rather than a one-line fix.

The Glasswing window has, by emergent consensus, become the default for AI-discovered findings against major projects. The rationale was published with the program: AI-discovered bugs tend to come in clusters (the model finds N related issues at once) and patches for them often require coordinated work across multiple repositories or vendors. The 90+45 window gives space for that coordinated work without leaving users exposed indefinitely.

For findings against open-source projects, the operative practice in May 2026:

- **Use the project's published security policy.** Most major projects have a `SECURITY.md` with an email address or a private vulnerability-reporting flow on GitHub. Use it.
- **Do not file a public issue.** This still happens, more often than it should, and it embarrasses everyone involved. The bug is now public the moment it's filed; any patch window is forfeit; the maintainer is annoyed.
- **Provide reproduction steps and impact analysis.** "The model said this is a bug" is not a report. "Here is the file, here is the line, here is the input that triggers it, here is what an attacker can do, here is the test case that demonstrates it" is a report.
- **Disclose your tooling honestly.** If a model wrote the analysis, say so. The maintainer needs to know the model's claims may be wrong; the maintainer also needs to know the model may have missed adjacent variants. Hiding the AI involvement does the maintainer no favors.
- **Be patient with the timeline.** Maintainers are often volunteers. The model's facility at finding bugs does not transfer to the maintainer's facility at fixing them. The 90-day window is a minimum, not a target.

For findings against commercial products, follow the vendor's bug bounty or vulnerability disclosure program if one exists. If none exists, send to a `security@` address; if none exists, escalate to the product team through whatever contact you have. CERT/CC is the fallback for "I cannot get a vendor to engage."

## The awkward case: AI-written analysis

The new wrinkle. When the model wrote the report, the maintainer faces a question they did not have to face when the report came from a human: how much do I trust the severity claim?

A human security researcher who reports a bug has, implicitly, staked their reputation on the report. If they say it's a remote-code-execution bug and it turns out to be a benign null-pointer dereference, their next report carries less weight. The reputational pressure aligns the researcher's incentives with the maintainer's: the researcher overclaims at their cost.

A model has no reputation, in the relevant sense. The same model can produce a thousand reports a day, each one styled like a human researcher's, each one varying in quality, none of them tied to a credible reputational signal. The maintainer cannot distinguish, from the report alone, the careful AI-assisted finding from the noise.

The current practical conventions, as they're settling in:

- **Sign your reports.** Use your name. The reputational chain runs through *you*, not through the model.
- **Verify before submitting.** If you cannot reproduce the bug yourself, on the actual code, do not submit it. The model's report is a starting point; verification is the price of admission.
- **State explicitly what you verified vs. what the model claimed.** "I verified the bug fires on input X. The model claims it can be chained with Y to achieve impact Z; I have not verified the chain." This is the honest version of disclosure that preserves the maintainer's ability to triage.
- **Provide the model's full analysis as an appendix, not as the report.** The report is your synthesis. The model's raw output is reference material the maintainer can choose to read.
- **Accept that some maintainers will refuse AI-assisted reports entirely.** This is their right. They have been triaging low-quality AI submissions for two years now and have, in some cases, decided the signal-to-noise isn't worth their time. Don't argue. Find another channel or another target.

This is the social infrastructure that's still being built. Five years from now there may be a verification standard, a credentialing system, an integrated tool that lets maintainers reproduce AI-discovered bugs against an isolated copy of their codebase before reading the report. None of that exists today. Today, the answer is "be a good citizen by hand."

## What not to do

A short list of things that are happening in 2026 that should not be:

- **Auto-filing CVEs.** Some teams are running agentic auditors against random open-source projects and auto-filing CVE requests for everything the model flags as a candidate. CVE assignment is a finite, human-mediated process; flooding it with low-verification AI reports is degrading the system. Don't.
- **Public disclosure without coordination.** "I found a bug in $project, here's the PoC, blog post tomorrow" — even when the bug is real, you have just helped the attacker more than the defender. The temptation is real because the publishing is fast and the gratification is immediate. Resist.
- **Using AI-discovery against your competitors as a marketing exercise.** "Look how many bugs we found in $rival_product" reports, with the implication that your product is better, are intellectually dishonest (your product has its own bugs) and ethically dishonest (you are weaponizing the disclosure process for marketing). This pattern emerged in late 2025 and is not getting less common.
- **Selling vulnerabilities to the highest bidder.** The "responsible disclosure or sell to a broker" choice has always been ethically fraught; the AI capability shift makes the calculus worse, because the volume of findings is much higher. The broker market for AI-discovered bugs is, in May 2026, smaller than it might be — partly because the major buyers have been wary of provenance, partly because the major sellers (Glasswing partners) are contractually committed to coordinated disclosure. This will change. Don't be the one who changes it.
- **Doxing maintainers.** A bug report is not an excuse to publish personal contact information about the maintainer or their dependents. This should not need saying. It does.

## The asymmetry is also a disclosure problem

A point worth making explicit in this chapter rather than only in the next: the capability gap from chapter 1 affects disclosure too.

A Glasswing partner who finds a bug in your dependency has a defined disclosure pipeline, organizational backing, legal cover, and the credibility that comes with the program's reputation. You, the engineer who found a bug with Claude Opus 4.7 on a Tuesday, have none of these. Your disclosure is more fragile in every respect: more likely to be misread as noise, more vulnerable to legal action from a vendor who reacts badly, more likely to leak to other channels before the patch lands.

The defensive answer to this is to disclose under whatever institutional cover you can muster. Through your employer's security team if you have one. Through a CERT coordinator if the bug is significant enough to warrant the relationship. Through a bug bounty's intermediary if one exists. Through a security researcher you know with established reputation, if they're willing to vouch. The pure individual disclosure is the riskiest path; the institutional channels are imperfect but not new.

## Sources

- Google Project Zero, disclosure policy: <https://googleprojectzero.blogspot.com/p/vulnerability-disclosure-policy.html>
- CERT/CC, Coordinated Vulnerability Disclosure Guide: <https://vuls.cert.org/confluence/display/CVD/>
- Project Glasswing, "Disclosure norms for AI-discovered vulnerabilities," published with the program launch, April 2026.
- "Coordinated Vulnerability Disclosure: A Guide for Industry," CISA, ongoing series, with the 2025 update covering AI-assisted discovery specifically.
- Bruce Schneier, "AI and Vulnerability Disclosure," `schneier.com`, several posts during 2025–2026, on the social-infrastructure question.
- Anthropic, the Mythos disclosure timeline as published in `red.anthropic.com` posts for the FreeBSD and Mozilla advisories — useful as worked examples of what a coordinated AI-discovered disclosure looks like at the high end.
