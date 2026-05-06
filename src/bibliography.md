# Bibliography and Sources

A consolidated list of references cited throughout the book, organized by topic. Where a paper or post is freely available, the link is included. The list is current as of May 2026; some URLs may move.

## The Mythos disclosure and Project Glasswing

- Anthropic, "Introducing Claude Mythos Preview," `red.anthropic.com`, April 2026.
- Anthropic, "Project Glasswing: Coordinated AI-Assisted Vulnerability Discovery," April 2026.
- Anthropic and Mozilla, joint Firefox vulnerability disclosure, May 2026.
- FreeBSD Project, security advisory FreeBSD-SA-26:07.nfs (CVE-2026-4747), May 2026.
- CETaS, "Mythos and the Capability Frontier: An Analysis of the Anthropic Disclosure," April 2026.
- *IEEE Spectrum*, "The Vulnerability-Finding Model Anthropic Won't Release," May 2026.

## Foundational AI security literature

- OWASP Foundation, "OWASP Top 10 for Large Language Model Applications," 2025 edition. <https://genai.owasp.org/llm-top-10/>
- Greshake, Abdelnabi, Mishra, Endres, Holz, Fritz, "Not what you've signed up for: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection," *AISec '23*.
- Perez et al., "Ignore Previous Prompt: Attack Techniques For Language Models," 2022.
- Liu et al., "Prompt Injection Attack Against LLM-Integrated Applications," 2024.
- Bai et al., "Constitutional AI: Harmlessness from AI Feedback," Anthropic, 2022.
- Anil et al., "Many-shot Jailbreaking," Anthropic, April 2024.
- Russinovich, Salem, Eldan, "Great, Now Write an Article About That: The Crescendo Multi-Turn LLM Jailbreak Attack," Microsoft, 2024.

## Corpora and benchmarks

- Schulhoff et al., "Ignore This Title and HackAPrompt," EMNLP 2023.
- Toyer et al., "Tensor Trust: Interpretable Prompt Injection Attacks from an Online Game," ICLR 2024.
- Chao et al., "JailbreakBench," 2024 with periodic updates.
- Zou et al., "Universal and Transferable Adversarial Attacks on Aligned Language Models," 2023 (the GCG / AdvBench paper).

## AI red-team tooling

- DeepTeam: <https://github.com/confident-ai/deepteam>
- garak: <https://github.com/NVIDIA/garak>; Derczynski et al., "garak: A Framework for Security Probing Large Language Models," 2024.
- PyRIT: <https://github.com/Azure/PyRIT>

## AI-augmented code audit

- Anthropic, *Claude Code* documentation: <https://docs.claude.com/claude-code>
- OpenAI, *Codex CLI* documentation, 2025–2026.
- Yang et al., "SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering," NeurIPS 2024.
- *OpenHands* (formerly OpenDevin): <https://docs.all-hands.dev>
- Aider: <https://aider.chat>
- Google Project Zero, "Big Sleep" series, 2024–2025: <https://googleprojectzero.blogspot.com>
- Various 2025 papers on LLM-assisted vulnerability discovery: *AutoCodeRover* (Zhang et al.), *RepoAudit*, the Patchwork tooling papers.

## Output handling and exfiltration

- Microsoft, "EchoLeak" disclosure (CVE-2025-32711), Microsoft 365 Copilot, June 2025.
- Johann Rehberger, *Embrace the Red*, ongoing series at <https://embracethered.com>.
- Riley Goodside, threads on Unicode tag-character smuggling and visual prompt injection, 2024–2025.
- Mozilla, Content Security Policy reference: <https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP>

## Confused-deputy framing

- Norm Hardy, "The Confused Deputy: (or why capabilities might have been invented)," *ACM SIGOPS Operating Systems Review*, October 1988. Forty-year-old paper, still load-bearing.

## Threat modeling and disclosure

- Adam Shostack, *Threat Modeling: Designing for Security*, Wiley, 2014.
- Google Project Zero disclosure policy.
- CERT/CC Coordinated Vulnerability Disclosure Guide.
- Project Glasswing, "Disclosure norms for AI-discovered vulnerabilities," April 2026.
- CISA, "Coordinated Vulnerability Disclosure: A Guide for Industry," 2025 update.
- NIST, "AI Risk Management Framework," AI 100-1, 2023, with the 2024 generative-AI profile addendum.

## Ongoing commentary

- Simon Willison, `simonwillison.net/series/prompt-injection/`, the running series since 2022.
- Bruce Schneier, `schneier.com`, the AI-and-security posts during 2025–2026.
- Anthropic model cards and safety reports for the Claude 4.x family, 2025–2026.

## Regulatory background

- European Union AI Act, especially Article 52 and the GPAI provisions, with the 2025–2026 implementation timelines.
- California SB 1047 successor legislation, in progress as of May 2026.
- U.S. AI Safety Institute publications on coordinated disclosure.
