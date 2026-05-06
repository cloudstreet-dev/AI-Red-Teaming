# Red Team Vocabulary

Before any of the technique chapters do useful work, we need shared words. AI security borrows much of its vocabulary from classic application security, then bends some of the borrowed terms in ways that confuse readers who haven't noticed the bending. This chapter gives a working vocabulary for both surfaces — the AI feature and the conventional code — and maps the OWASP LLM Top 10 against the older taxonomy so you can see where the new attack surface overlaps the old one and where it is genuinely new.

## Threat models, plainly

A threat model is a written answer to four questions. *Who is the attacker.* *What can the attacker do.* *What are they trying to achieve.* *What would convince you they can't.* You can write a useful threat model on the back of an envelope. Most threat models in production were not written at all, which is why most products under attack do not survive contact.

For an AI-augmented product, the four questions get instantiated like this:

- *Who.* A logged-in user with a valid account, possibly malicious. An unauthenticated visitor on the public chat surface. A third party whose content reaches the model indirectly: emails, web pages, documents, calendar invites, whatever the model retrieves or fetches. The user's own browser, which is rendering the assistant's output.
- *What can they do.* Type into the chat box. Upload a file the model will process. Email an inbox the model reads. Edit a document the model will retrieve. Get an HTML page indexed that the model will fetch. Phish a higher-privilege user into pasting attacker-controlled text. The list extends every quarter.
- *What are they trying to achieve.* Read data they shouldn't read (other users' data, system instructions, retrieved-but-not-displayed context). Take actions they shouldn't take (send emails, make API calls, modify state). Make the assistant lie to its user (incorrect information given as authoritative). Make the assistant exfiltrate data through a side channel (image fetches, link rendering, citation URLs). Get free compute. Embarrass the brand.
- *What would convince you they can't.* This is the hard question and the one most people skip. "We tested it" is not an answer. "We have an automated red-team suite of N attacks against the deployed model that runs nightly and the success rate is below X" is an answer. The rest of this book is partly about getting to the point where that sentence is true.

## Trust boundaries

A trust boundary is a line in your system where data crossing it should be treated with more suspicion than data on the other side. Classic application security has well-understood boundaries: the network edge, the auth layer, the database driver, the rendering layer. AI systems introduce new ones, and forget some of the old ones.

The most important new boundary, and the one most teams have not internalized, is **the model's context window**. Everything in the context window is treated by the model as input it can reason over. The model does not — cannot — strongly distinguish between the system prompt, the user's message, the retrieved RAG document, the tool output, and the file you uploaded. They are all tokens. The training-time conditioning that says "instructions in the system prompt have priority" is exactly that: a conditioning, with a probability of being followed, not a structural guarantee. The trust boundary is not where you wish it were; it is wherever the attacker can cause text to land in the context window.

The classic boundaries also still apply, and AI features tend to weaken them. The auth layer that decided this user can read their own customer record is bypassed if the model is happy to read someone else's. The rendering-layer escape that prevents stored XSS is bypassed if the model is allowed to emit unsanitized markdown. The database access controls are weakened if the model is given a tool that runs SQL on its behalf with the application's privileges instead of the user's.

Write down where the boundaries are in your product, on paper, today. Then write down which of them depend on the model behaving correctly. The ones that do are where the budget should go.

## Attacker capabilities

Capabilities, not motivations. The threat model cares what the attacker *can* do; what they *want* to do is a separate axis and is mostly less interesting for design.

A working capability list for a typical AI-augmented SaaS product:

| Capability | Held by |
|---|---|
| Submit text to the model via the documented input | Anyone who can use the product |
| Submit text via an undocumented input (URL params, file upload metadata, image alt text) | Anyone who reads the front-end code |
| Submit text via indirect channels (RAG corpus, web fetch, email subject line, calendar invite, PDF metadata) | Anyone who can get content into one of those channels |
| Cause the model to emit specific output | Anyone with sufficient prompting time |
| Cause the front-end to render specific output (markdown, HTML, image fetches) | Anyone who can cause specific output |
| Cause the back-end to take specific actions (tool calls) | Anyone who can cause specific output, modulated by tool-permission discipline |
| Replay or extend an existing session | The session's owner, plus anyone who phished them |
| Observe response timing, token counts, error patterns | Anyone making requests |

This list is shorter than the corresponding list for a conventional web application, which is misleading. The capabilities are broader than they look, because each one chains into multiple others through the model's willingness to translate inputs into outputs and outputs into actions.

## The OWASP LLM Top 10 against the classic taxonomy

The OWASP LLM Top 10 is now a few revisions in (the 2025 edition is the current reference at the time of writing). It is not a perfect list — categories overlap, the numbering changes between versions, and some entries are organizational rather than technical — but it is the closest thing to a shared vocabulary the field has, and learning to map it onto your existing application-security mental model is worth an afternoon.

The mapping below pairs each LLM Top 10 entry with the classic vulnerability class it most resembles, then notes what is genuinely new.

**LLM01 — Prompt Injection.** The closest classical analogue is **injection**, broadly: SQL injection, command injection, log injection, header injection. The shape is the same: untrusted data flows into a context where it is interpreted with privilege. The novelty is that the "interpreter" is a language model with no formal grammar to escape into, so the classical defense — parameterized queries, prepared statements, structural escaping — has no direct analogue. We will spend chapters 3, 4, and 5 on this.

**LLM02 — Sensitive Information Disclosure.** Classical analogue: **information disclosure** in the OWASP Top 10 web sense. Same shape, new mechanism: the model can be coaxed to reveal training data, system prompts, retrieved documents from other users, or fragments of context it has seen earlier in the session. The defense surface is different — there is no `Cache-Control: private` for a model's recall — but the principle is unchanged: don't put it in the model's context if you don't want it in the model's mouth.

**LLM03 — Supply Chain.** Classical analogue: **supply-chain attacks**, exactly as understood in package-manager land. The new flavor is *model* supply chain: weights from untrusted sources, fine-tunes shipped through Hugging Face by anonymous accounts, embedded backdoors in vision encoders, training-data poisoning. If you self-host any model you did not train from scratch, this applies to you the same way npm dependencies do, with weaker tooling.

**LLM04 — Data and Model Poisoning.** Genuinely new. The training-time and fine-tuning-time analogue of injection: an attacker influences what the model learned. For most product teams this is not in scope because they don't train. For teams that fine-tune on user-generated data — which, increasingly, is most of them — it is in scope and is harder to test for than the runtime attacks.

**LLM05 — Improper Output Handling.** This is the one that is most familiar and most under-appreciated. Classical analogue: **output encoding failures**, the parent class of XSS, SSRF, open-redirect, and similar. When a model emits text that downstream systems interpret with privilege — markdown rendered as HTML, URLs auto-fetched, code blocks executed, citations dereferenced — every single existing rule about untrusted output applies, except that "the model" is now part of the untrusted-input surface. Chapter 9 is mostly about this.

**LLM06 — Excessive Agency.** Classical analogue: **excessive privilege**, specifically the principle of least privilege as it applies to service accounts. The novelty is that the service account is now an LLM with broad tool access, and the principle of least privilege is a great deal harder to apply when the agent is supposed to be flexible. Chapter 8 lives here.

**LLM07 — System Prompt Leakage.** Classical analogue: **information disclosure**, with a side of "stop putting secrets in places that will be read aloud." The novelty is mostly that people kept treating system prompts as confidential when they were never structurally protected from extraction.

**LLM08 — Vector and Embedding Weaknesses.** Genuinely new. Embedding-space attacks: adversarial documents that score highly against arbitrary queries, dense-retrieval poisoning, embedding inversion that recovers original text from supposedly-opaque vectors. Mostly relevant if you operate the retrieval layer; if you outsource embeddings to a vendor, this is partly their problem and partly yours.

**LLM09 — Misinformation.** Classical analogue: **integrity failures** in the broad CIA sense, dragged into a domain (the model says what it says) where the mitigations are weak. Mostly an organizational and product-design problem rather than a technical one. Will not get a dedicated chapter, but informs how to think about user-facing trust UX everywhere else.

**LLM10 — Unbounded Consumption.** Classical analogue: **denial of service** and **rate-limit bypass**, with a side of cost-amplification because LLM tokens are not free. The novelty is the cost-amplification angle: an attacker who can cause your model to generate a million tokens per request has discovered a denial-of-wallet attack that classical DDoS protections do not look for.

The OWASP list is useful as a checklist. Like all checklists, it does not substitute for the threat model. A team that cleans up every Top 10 issue without writing the threat model will harden the surfaces the list happened to enumerate and miss the ones it didn't. A team that writes the threat model first and then uses the Top 10 as one input among several will get more out of both.

## The two-track view

The book's premise is that your product has two attack surfaces — the AI feature and the conventional code — and both need red-teaming. The vocabulary above unifies them where it can.

For the AI surface:

- The trust boundary is the context window. Everything inside it competes for the model's attention.
- The injection class is prompt injection (direct and indirect), with no parameterized-query analogue.
- The output handling class is markdown rendering, link dereferencing, code execution, citation following.
- The privilege class is tool permissions and the agent's ability to chain them.

For the conventional surface, AI changes the *audit* surface, not the *vulnerability* surface. The bugs you have are the bugs you had: injection, deserialization, race conditions, memory corruption in whatever language is exposed to that, the long tail of things that the application security literature has been writing about for thirty years. What changed is that you can now ask a frontier model to look for them, in your code, on a budget that did not exist two years ago. The attacker can also do this. The asymmetry is not in what bugs exist; it is in who finds them first.

A product team that keeps both surfaces in view, separately enumerated and separately tested, will be in a meaningfully better position than one that thinks of "AI security" as a single category. The two surfaces fail in different ways, defend with different tools, and require different vocabulary to discuss precisely. Confusing them — assuming that the prompt-injection eval suite says anything useful about the SQL injection coverage of your auth layer, or vice versa — is a category error that produces false confidence in both directions.

## A note on terminology drift

"Jailbreak," "prompt injection," and "adversarial input" are used in the literature with overlapping meanings, and you will see all three referring to the same technique in different papers. This book uses the convention that has settled in practice: *prompt injection* is the broad class of attacks where an attacker causes the model to follow instructions other than the developer's, *jailbreak* is the specific subclass where the goal is to bypass safety training (rather than, say, exfiltrate data), and *adversarial input* is reserved for inputs crafted to exploit numerical properties of the model (gradient-based attacks against open-weights models, embedding-space attacks against retrieval). When this book talks about defending a chat product, it almost always means the first two. When it talks about embedding attacks, it will say so explicitly.

With the vocabulary in place, we can start breaking things.

## Sources

- OWASP Foundation, "OWASP Top 10 for Large Language Model Applications," 2025 edition. <https://genai.owasp.org/llm-top-10/>
- Adam Shostack, *Threat Modeling: Designing for Security*, Wiley, 2014. The four-question framing is adapted from chapter 2.
- NIST, "AI Risk Management Framework," AI 100-1, 2023, with the 2024 generative-AI profile addendum. Useful for organizational vocabulary; less useful for Tuesday-afternoon work.
- Simon Willison, "Prompt injection and jailbreaking are not the same thing," `simonwillison.net`, 2024. The terminology distinction this chapter adopts is his.
