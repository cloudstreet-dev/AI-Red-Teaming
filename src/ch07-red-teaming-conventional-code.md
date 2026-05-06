# Red-Teaming Conventional Code with Today's Models

This chapter switches tracks. The first half of the book was about attacking AI features in your products. This chapter is about using AI to attack the conventional code in your products. Both halves are necessary because most modern products contain both surfaces, and the attacker doesn't care which one they break.

The Mythos disclosures from April 2026 made the upper bound of this capability concrete: a frontier model with the right scaffolding can chain vulnerabilities into working exploits in real codebases. You do not have Mythos. You have Claude Opus 4.7, GPT-5, and the publicly available agentic frameworks. The question this chapter answers is what those tools can and cannot do for the engineer who wants to red-team their own conventional code on a budget that fits in a Tuesday afternoon.

The honest summary up front: these tools will miss things Mythos finds. They will also find things you would not have caught manually, in your own code, in less time than a security audit costs. The gap between "what current public models can do" and "what Mythos can do" is real and material; the gap between "what current public models can do" and "what most teams actually have time to do today" is also real and is the gap this chapter aims to close.

## The basic loop

The minimum viable AI-assisted code audit, scaled down from the Mythos-style harness:

1. **Containerize the target.** Mount the code into an ephemeral environment with no network access (or carefully scoped network access for tests that need it). The model is going to write code, run it, see what happens, and try again. You want this to happen in a sandbox.

2. **Give the model an agentic scaffold.** Tools for reading files, listing directories, running shell commands, executing test suites. The Anthropic *Claude Code* CLI does this directly. The OpenAI *Codex* CLI is a similar shape. Open-source frameworks like *SWE-agent*, *OpenHands*, *Aider*, and *Goose* compose models with code-execution tools.

3. **Give it a clear initial prompt.** "Find a vulnerability in this program. Report it with a proof of concept and reproduction steps. Triage your own findings: rate severity, note false-positive risk, link to the code path that produces the bug."

4. **Let it run.** Tens of minutes to several hours per target, depending on size and budget. Tens of cents to a few dollars in API costs per run.

5. **Triage the output.** A non-trivial fraction of findings will be false positives or restatements of known limitations. The triage step is where human judgment buys most of its value.

This is the loop. Most of this chapter is about the variations and tactics that make it work better.

## File ranking

The single highest-leverage trick is to not give the model the entire codebase at once. A frontier model with a 200K to 1M token context window can technically read a moderate codebase wholesale, but the cost goes up linearly, the recall goes down nonlinearly, and the model's attention disperses across files that aren't where the bugs are.

The alternative is *file ranking*: have the model triage which files are most likely to contain bugs, then start with the high-value ones.

A file-ranking pass looks like this. Hand the model the directory tree, plus a one-paragraph summary of each file (extracted automatically by a prior pass, or manually if the codebase is small enough). Ask it to rank the files by likely-attack-surface. Score factors the model handles well:

- Files that parse untrusted input (HTTP request handlers, command-line argument parsers, file-format readers, deserialization)
- Files that handle authentication, session management, or access control
- Files that construct dynamic queries (SQL, shell, system calls)
- Files that interact with the filesystem with paths that could include user input
- Files that handle uploads, downloads, or any I/O against attacker-controllable URLs
- Cryptography-adjacent code (token validation, signature verification, padding handling)
- Network protocol implementations
- Any "TODO," "FIXME," or "XXX" comments mentioning security or known issues

The model is good at this triage because it is fundamentally a reading-comprehension task with strong priors about what insecure-looking code looks like. Spend the first 10–20% of your budget on the ranking pass, then spend the rest in priority order.

You will be surprised, on real codebases, how many of the actual bugs live in the top quarter of the ranked list. You will also be surprised by the bugs that come from files the ranking missed entirely. Both are useful information.

## The "find a vulnerability" prompt and its variants

The bare prompt — "find a vulnerability in this program" — works, weakly. The variants work better.

**Specific class targeting.** "Find any place in this code where user input flows into a SQL query without parameterization." "Find any place where a path is constructed from user input and then passed to a file operation." Models are dramatically better at finding bugs they were told to look for than at open-ended exploration. The class-targeted prompt is closer to what a static analyzer does, but with the model's broader contextual reasoning available.

**Adversarial role framing.** "You are a security researcher conducting a paid audit of this codebase. Your reputation depends on finding real bugs. Begin." This works better than the bare prompt for reasons that are partly mechanical (the model treats "audit" as a known task with known shape) and partly stylistic (the model generates more thorough reports under the framing).

**Trace-back from impact.** "If an attacker could trigger a remote code execution in this codebase, what would be the most likely path? Walk backward from RCE to the entry point that would produce it." This produces a different distribution of findings than forward analysis — better at finding chains, worse at finding isolated single-bug-single-impact issues.

**Compare-to-known-good.** "Here is the implementation. Here is the documented security-best-practice for this kind of feature. Where does the implementation diverge?" Excellent for crypto code, auth flows, anything with a published canonical version.

**Diff-mode.** "Here is the diff for this PR. Find any new bugs introduced or new attack surface created. Pay attention to changes in input handling, permission checks, and error paths." The most efficient mode if you have it integrated into PR review. Diffs are small, the model has the prior code as context, and the bugs that get introduced are usually changes-in-handling rather than new structures.

**Proof-of-concept generation.** Once a candidate bug is identified, "write a reproducible test case that demonstrates this bug" forces the model to commit. If the test passes against the unfixed code and fails against the fix, you have a real bug. If the model cannot write a reliable repro, you have a candidate that is much more likely to be a false positive, and you can deprioritize.

A useful pattern is to chain these: file-ranking, then class-targeting on the top-ranked files, then PoC generation on each candidate, then triage. The whole loop on a 50KLOC Python codebase costs in the low single-digit dollars and takes a few hours of wall-clock time.

## What current public models will and won't find

A blunt assessment, current as of May 2026, against typical web-application code in mainstream languages:

**Will reliably find:** Classic injection bugs (SQL, command, NoSQL) when the unsafe pattern is in a single file. Hardcoded secrets that haven't been moved to environment variables. Use of known-broken cryptographic primitives (MD5 for password hashing, ECB mode, no MAC). Missing CSRF tokens on state-changing endpoints. SSRF vulnerabilities in URL-fetch features. Path traversal where the path manipulation is straightforward. Dependency vulnerabilities when given a lockfile to compare against advisory databases. Race conditions in code where the racy pattern is locally visible.

**Will sometimes find:** Bugs that span two or three files where the call chain is not obvious. Authentication bypasses where the bypass requires understanding the auth flow as a whole. TOCTOU bugs where the time-of-check and time-of-use are in different modules. Subtle authorization issues — "this endpoint checks the user is logged in but not that the resource belongs to them." Type-confusion bugs in dynamically typed code. Off-by-one errors in C/C++ buffer handling.

**Will mostly miss:** Bugs requiring deep understanding of the application's business logic ("this endpoint should only be callable during business hours, but the check is in the wrong layer"). Bugs in custom protocols or undocumented in-house formats. Bugs that require chaining four or more steps through unrelated code paths — this is the gap to Mythos. Side-channel attacks (timing, cache, power) unless given explicit prompting and instrumentation. Anything that requires running the binary and observing it under fuzzing — this is what *Big Sleep* and similar specialist tools do, and current general-purpose agentic loops are weaker at it.

**Will produce false positives at:** A rate that depends heavily on the prompt. Open-ended "find bugs" prompts can produce 30–60% false-positive rates by my read of various 2025 evaluations. Class-targeted prompts ("find SQL injection") often run under 10%. PoC-required workflows (the model must produce a working repro) approach 0% false positive — at the cost of false-negative rate going up, since some real bugs are hard to repro from outside.

This is a moving target. The 2024 versions of these models could not do half of the "will reliably find" list. The 2027 versions will probably do most of the "will mostly miss" list. The structural advice — file ranking, class targeting, PoC for verification, triage as the load-bearing human contribution — should outlive the specific capability list.

## Triage: the load-bearing human contribution

The model produces findings. The triage step decides which findings are real, which are noise, and which need follow-up.

A practical triage workflow:

1. **Reproduce.** If the model produced a PoC, run it. If the bug doesn't reproduce, deprioritize aggressively. If it does reproduce, you have a confirmed bug.
2. **Bound the impact.** Ask the model — and yourself — what an attacker can actually do with this bug. "Read the password hash" is one impact. "Read the password hash, decrypt it offline because we used MD5, log in as the admin" is a much higher one. Mythos-style chaining can be approximated by asking the model to chain a finding it just produced with adjacent code paths.
3. **Compare to existing tracking.** Search your issue tracker, your security backlog, your `KNOWN_ISSUES.md`. The model often finds things you already know about. That is fine — confirms its calibration — but doesn't generate work.
4. **Fix and regression-test.** When you fix, write the test that proves the fix. The test should fail against the unfixed code and pass against the fixed code. Add it to your suite. Now this specific bug class has a tripwire.
5. **Backlog the rest.** Findings that are real but lower-priority go in the backlog with the model's full report attached. The next iteration of the model, six months later, can re-triage them.

The triage step is currently irreducibly human. Models can produce triage reports — and you should ask them to — but the decision of "is this worth fixing in this sprint" is product judgment, not security analysis. A model that auto-files all its findings as P0 issues will burn out your team in a week.

## What to do with the findings about your own code

Fix them. This sounds glib but it is the entire answer for code you control. The disclosure-and-coordination machinery from chapter 11 is for findings against code you don't control. For your own code:

1. Fix it. (Or, if not fixable today, mitigate and document.)
2. Write the regression test. (Mandatory, not optional.)
3. Commit. (Reference the finding in the commit message; do not include the full PoC in the public commit if the bug is undisclosed in dependencies you ship.)
4. Note the class. (If you found one SQL injection in your codebase, you have likely written others; do a class-wide pass.)
5. Update your threat model. (The bug appeared because some assumption in your model was wrong. Find the assumption, fix it.)

That is the loop. Run it on any codebase you own, with any of the available models, on any Tuesday. The improvement, sustained over a few months, will be more visible than any single audit.

## A note on running this against code you don't own

Wait for chapter 11. The short version: don't, until you've read the disclosure ethics and coordination chapter. Finding bugs in third-party code with AI is now cheap. Disclosing them responsibly, dealing with the awkward case where the model wrote the analysis, and not creating new problems for the maintainers is not cheap. The capability has outrun the social infrastructure. Do not treat this chapter as license to run an agentic auditor against every open-source dependency you have and start filing CVEs the next morning.

## Sources

- Anthropic, *Claude Code* documentation: <https://docs.claude.com/claude-code>
- OpenAI, *Codex CLI* documentation, 2025–2026.
- *SWE-agent*: Yang et al., "SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering," NeurIPS 2024.
- *OpenHands* (formerly OpenDevin): documentation at <https://docs.all-hands.dev>.
- Google Project Zero, "Naptime: Project Zero's Big Sleep Framework," and the follow-up "Big Sleep" series, 2024–2025.
- *Aider* documentation, <https://aider.chat>.
- The CETaS analysis of Mythos cited in chapter 1, for the upper-bound capability comparison this chapter is calibrating against.
- Various 2025 papers on LLM-assisted vulnerability discovery — *AutoCodeRover* (Zhang et al.), *RepoAudit* (multiple groups), and the Patchwork tooling papers — for the false-positive-rate numbers cited.
