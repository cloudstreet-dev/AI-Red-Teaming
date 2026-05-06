# The Mythos Moment

In April 2026, Anthropic announced *Claude Mythos Preview*. The post on `red.anthropic.com` was the kind of corporate communication that does not, on a first read, feel like a turning point: a few paragraphs about a frontier model with notable computer-security capabilities, a description of an evaluation suite, a list of partners. The kind of post that gets bookmarked on Tuesday and forgotten on Friday.

It was not forgotten on Friday.

The CETaS analysis published a week later went through what the post had actually said. *Mythos*, internally, had been used to chain multiple vulnerabilities into working exploits across CTF scenarios that previous frontier models could not finish. It had found zero-days in mainstream operating-system components. It had read browser source trees and produced reproducible crashes. The smart-contract benchmark on `red.anthropic.com` showed the model finding bugs in deployed contracts that had passed independent audits. The blog post had been deliberately understated; the technical appendix and the partner-program documentation that followed it were not. *IEEE Spectrum* covered it the next week under a headline that did not bother to be measured.

Anthropic chose not to release the model publicly. Instead they launched *Project Glasswing*: a partner program of about forty large organizations — AWS, Apple, Google, Microsoft, Nvidia, several governments, a small number of named security firms — granted monitored access to Mythos for the purpose of finding and disclosing vulnerabilities in their own and others' code. The first batch of disclosures from Glasswing partners landed within a month. The FreeBSD NFS RCE (CVE-2026-4747) was the most-covered, because FreeBSD is comparatively small and the patch arrived quickly enough that the disclosure timeline could be dramatic. The Firefox bugs jointly disclosed by Anthropic and Mozilla were the most consequential, because most of the world runs the rendering engine they touched. Other findings — the smart-contract exploits, the kernel side-channels, the disclosed-but-not-yet-patched bugs in two cloud-provider hypervisors — got less press but more private alarm.

The rest of us were not in the room.

That is the situation this book takes as its starting condition. There is a frontier offensive capability in the world, demonstrably. It is not generally available. It is unevenly distributed. Forty organizations have it. The engineer who is building a SaaS product on a Tuesday afternoon does not. The startup founder shipping an AI-augmented onboarding flow does not. The two-person team running a regional fintech does not. The internal-tools developer at a midsize manufacturer does not. The capability gap between attacker and defender, in the specific domain of computer security, is being compressed at the top end and ignored at the long tail.

## What changed

The thing that changed in April 2026 is not that AI can now find vulnerabilities. AI could find vulnerabilities in 2024. By 2025, GPT-5 and Claude Opus 4.6 were both routinely flagging real bugs in code review, well above the false-positive rate that made earlier models useless for the task. Specialist tools like Google's *Big Sleep*, the academic *AutoCodeRover* family, and the increasingly capable open-source agentic scaffolds were producing genuine CVEs throughout 2025. The literature was full of papers titled some variant of "LLMs find bugs," each one with caveats, each one with a real result underneath the caveats.

What changed in April 2026 is that the same kind of system, scaled to a frontier model with the right scaffolding and a specific training emphasis on offensive security tasks, *chained* findings into working exploits, end-to-end. It went from "this function looks suspicious" to "here is a proof of concept that takes the suspicious function, the unrelated parsing bug three files away, and the side-channel in the IPC layer, and produces remote code execution." The leap was integration, not detection.

Mythos is not magic. Internally, it is a very large model fine-tuned and post-trained heavily on security tasks, embedded in a long-running agentic harness with persistent memory, browser access, code-execution tools, and the patience to spend twelve hours on a single target. It composes capabilities you have seen demonstrated in pieces over the last two years. The composition is the news.

## What didn't change

What didn't change is the position of the engineers who are not in Glasswing. Their products still need to ship. Their attack surface is still the same shape it was in March. The capability they have access to — Claude Opus 4.7, GPT-5, Llama 4 if they are self-hosting, Gemini 3 if they are on Google Cloud, the various specialist code-analysis models — has not changed in the announcement. It has not gotten worse. It has not gotten meaningfully better. The Mythos moment did not redistribute capability to the long tail. It clarified that capability is not redistributed.

This matters for the threat model. Two facts compose:

1. **Offensive AI is closer to general availability than defensive Mythos-level scanning is.** Open-weights models keep improving. Agentic scaffolds are open source. The base capability that does the offensive composition — long-context reasoning over code, tool use, planning across many turns — is in the public models already and improving each release. A motivated attacker with a few weeks and modest resources can build a serviceable approximation of a Mythos-style offensive harness today, tuned to a narrower target. The attacker does not need Mythos. The attacker needs *enough*.

2. **Defensive Mythos-level scanning is not in your hands.** Anthropic's stated reason for keeping Mythos behind Glasswing is that releasing it would democratize the offensive capability faster than the world's defenders can absorb the disclosures. Whether you agree with that reasoning is a question for a different book. The operational reality is that you cannot ask Mythos to audit your codebase. You can ask Claude Opus 4.7 to do it. The result will be worse — a non-trivial fraction of what Mythos would have found will be missed — and it will be the best you can get.

This is the asymmetry. It is not new in shape. The offensive side has always had structural advantages: needs to find one bug, gets to choose targets, gets to choose timing, doesn't have to publish methods. The Mythos moment sharpens the advantage by shifting the per-bug cost down — for some attackers, dramatically. The defensive side gets the same shift, eventually, on a delay.

The honest framing is that "wait for Mythos to ship" is not advice. It is, possibly, never going to ship publicly. Anthropic has not committed to a release timeline; the partner program has expanded modestly since launch but remains gated. Even if it ships in 2027, the products you are working on now will have shipped first. The defenses you put in place against the attacks Mythos can already do are defenses you put in place with what you have.

## What this book is for

What you have, on a Tuesday afternoon, is roughly:

- A frontier conversational model (Claude Opus 4.7 or GPT-5) accessible via API at a few dollars per long task.
- One or more agentic scaffolds — Claude Code, the OpenAI Responses API with code-execution and web tools, the open-source SWE-agent and OpenHands frameworks, the various local-only options if you are doing sensitive work — that let you compose the model into longer tool-using loops.
- A growing public corpus of attack techniques against AI features themselves: prompt injection, indirect injection, multi-turn jailbreaks, tool abuse. Most of which work against most deployed systems most of the time.
- The OWASP LLM Top 10 (2025 edition is the current reference), which is a flawed but useful taxonomy of where these systems fail.
- A handful of dedicated red-team tools — DeepTeam, garak, PyRIT — that automate the obvious cases.
- The same conventional-application-security toolchain you already had: Semgrep, fuzzers, dependency scanners, the platform-specific things.

This book is for using all of that to red-team your own products. Both the AI features you have shipped and the conventional code that AI can now audit alongside you. The two halves interleave because the products you ship interleave them; the attacker does not separate them.

The book is short on purpose. The field is moving too fast to write the long version. Several of the specific tools and model capabilities cited will be out of date by the time you read this. The structural claims — that indirect injection is the harder problem than direct injection, that tool permissions are usually the load-bearing failure, that markdown rendering is a side channel, that conventional-code audits with current public models repay the time you spend on them but not in the way the marketing implies — those will not be out of date.

## What this chapter does not do

This chapter does not try to convince you that any of this matters. If you are reading a book on AI red-teaming, you are presumably already convinced or are going to remain unconvinced regardless. The arguments in either direction have been made at length elsewhere and rehearsing them here would waste the chapter.

It also does not try to make a case for what *should* happen — whether Anthropic should release Mythos, whether Glasswing's gating is principled or self-serving, whether the disclosure window the partners have adopted is correct. These are interesting questions. They are not your questions. Your question is what to do on Tuesday.

The rest of the book answers that question.

## Sources

- Anthropic, "Introducing Claude Mythos Preview," `red.anthropic.com`, April 2026.
- Anthropic and Mozilla, joint disclosure post on Firefox vulnerabilities, May 2026.
- FreeBSD Project, security advisory FreeBSD-SA-26:07.nfs (CVE-2026-4747), May 2026.
- CETaS, "Mythos and the Capability Frontier: An Analysis of the Anthropic Disclosure," April 2026.
- *IEEE Spectrum*, "The Vulnerability-Finding Model Anthropic Won't Release," May 2026.
- Google Project Zero, "Big Sleep: Discovering Real-World Vulnerabilities with LLM-Assisted Triage," 2024–2025 series of posts.
