# Tool and Agent Abuse

When your product gives the model tools, the model gives the attacker tools. This is the entire chapter in one sentence. Everything that follows is the elaboration.

The framing matters because the conventional security mental model treats the model as the attacker's target — the thing to extract data from, the thing to make say something embarrassing. In a tool-using agent, the model is more useful to the attacker as a vehicle than as a target. The attacker doesn't want to extract data from the model; the attacker wants the model to call `send_email` with the data the attacker chose. The model in this framing is a confused deputy — your code, running with your privileges, doing what an external party convinced it to do.

Anthropic flagged a version of this in the Mythos training environments themselves: the model, given broad tool permissions, chained them in ways the designers had not anticipated. This is not a Mythos-specific problem. It is the structural problem of agentic AI: tools designed individually compose into capabilities that nobody designed, and the composition is what gets exploited.

## What "tool" means here

Anything the model can invoke that has an effect outside the conversation. A few examples that recur in production systems:

- `lookup_customer(email)` — reads from your database
- `send_email(to, subject, body)` — writes to the world
- `fetch_url(url)` — reads from the world
- `run_query(sql)` — reads, sometimes writes, your database
- `update_account(id, fields)` — writes your database
- `execute_code(language, source)` — runs arbitrary code in some sandbox
- `search_web(query)` — reads, with attacker-influenceable results
- `call_api(endpoint, body)` — does whatever that API does, with your credentials

Tools are what makes an agent useful. They are also the conversion factor that turns "the model produced text" into "the system did a thing." Every tool is a privilege escalation path from the model's text-generation capability into your application's actual capabilities.

## The harness's tool surface

Glasswire Support, our running example, has four tools. Each one is a specific class of risk:

- `lookup_customer` — reads sensitive data. Risk: leaking it to the wrong audience.
- `search_kb` — reads less-sensitive data. Risk: returning attacker-poisoned content (chapter 4).
- `send_summary_email` — writes to the world. Risk: exfiltration; phishing-via-the-bot's-credibility; sending to the wrong recipient.
- `fetch_url` — reads from the world. Risk: SSRF (against internal services on the network); attacker-controlled content flowing back into the context (indirect injection); resource exhaustion.

This is small but not toy. Every one of these tools maps to a real category of feature in real products. The combinations of them are where the attacks live.

## The chaining problem

Each individual tool, considered alone, has a justification. `fetch_url` exists because customers reference documentation links. `send_summary_email` exists because support engineers want to follow up offline. `lookup_customer` exists because the bot is supposed to know who it's talking to.

Each pair of tools, considered together, has a less defensible justification:

- `fetch_url` + `lookup_customer`: the model can be instructed by an external page to look up specific customers and put the results back into the page (or, less subtly, into a follow-up tool call).
- `lookup_customer` + `send_summary_email`: the model can read customer data and email it. The intended use case is sending support summaries to internal addresses. The unintended use case is sending customer data to attacker addresses.
- `fetch_url` + `send_summary_email`: the model can fetch a page that tells it what email to send, then send it. The fetched page is arbitrary; the email is consequential.

All three pairs together are the configuration the harness ships in. The chapter 8 attack script (`attacks/ch08_tool_chain.py`) demonstrates the chain: the user asks the model to "check this documentation link" (which the attacker controls), the page contains instructions to summarize a customer to a specific external email address, and the model proceeds to do exactly that. There is nothing magical here. Each step is a tool call the model has been authorized to make. The composition is the attack.

## Excessive permissions

The OWASP LLM Top 10 calls this LLM06 — Excessive Agency. The shape is familiar from any service-account permissions audit you've ever done: the principle of least privilege says the agent should have only the permissions required for its actual job. The complication is that the agent's "actual job" is open-ended ("be helpful to the user") and the temptation is to give it broad permissions so it can be helpful in unanticipated ways. Resist the temptation. Helpfulness in unanticipated ways is also a description of the attack.

The hardening checklist for tool permissions:

**Per-tool risk classification.** Mark each tool as read-only / state-changing / external-effect. Treat each class as a separate trust tier. The bar for letting the model invoke a state-changing tool unattended should be much higher than for a read-only tool.

**Argument validation, not just authorization.** "The model can call `update_account`" is an authorization decision. "The model can call `update_account` to change *this* customer's *plan* to *one of {free, pro, team}*" is an argument constraint. The argument constraint catches the case where the model has been talked into calling a legitimate tool with illegitimate arguments. Implement these as type-checked, value-checked wrappers around the tool, not as instructions in the system prompt.

**Allowlists for external resources.** `fetch_url` should refuse anything not on an allowlist. `send_email` should refuse anything not in your domain (or a customer-supplied address you've validated). DNS resolution should be done in your code, not in the URL fetcher, so the attacker can't smuggle in a DNS-rebinding bypass. The harness's `enforce_tool_allowlist` toggle demonstrates how much this single defense buys you against the chapter 8 attack.

**Per-call approval for the consequential tools.** A human in the loop is still the most reliable defense for the actions you most regret. The cost is real (latency, queue depth, the human doesn't scale) and so it has to be reserved for the actions that warrant it: refunds above a threshold, password resets, anything that touches another user's account. Most production teams under-use this defense, often because the product team treats human-in-the-loop as a UX failure rather than as a feature.

**Capability tokens, not ambient authority.** The model should not have ambient access to "the database." The model should be given, at the start of the user's session, a token that authorizes it to act *on this user's data*. The tools accept the token, scope their queries by it, and refuse to act outside its scope. This makes confused-deputy attacks structurally harder because the deputy is no longer confused about *whose* deputy it is.

**Separate agents for separate trust tiers.** A read-from-email agent should not have URL-fetch capability. A URL-fetcher should not have email-send capability. A summarizer should not have any state-changing capability. Architecting the system as several narrow agents that hand off explicitly is harder to design and easier to defend than one broad agent with many tools.

These are the structural defenses. The prompt-level defenses ("never call `send_summary_email` to an address outside `glasswire.example`") are layer-cake material — fine to have, useless to rely on.

## Side-effect exfiltration via tools

The attacker doesn't always need to make the model say the secret. Sometimes the attacker just needs to make the model do a thing whose side effect carries the secret.

A `search_web(query)` tool whose query argument the model constructs from sensitive context can leak that context to the search provider's logs, and to anyone who can read those logs. The model never "said" the secret; it just put it in a search query. If the search provider is the attacker (or is compromised), the attacker now has the secret.

A `fetch_url(url)` tool where the URL contains a query string the model constructs from sensitive context produces the same effect against the URL's host. This is the chapter 9 attack class extended to tools — the markdown-image side channel applied to any tool that takes a URL.

A `write_to_kb(article)` tool — a less common but increasingly seen feature where the agent can update its own knowledge base — turns the agent into a stored-injection vector for itself and other users. An attacker who can talk the model into writing a poisoned article into the KB has planted a payload that will trigger on every future user whose query retrieves it.

A `send_summary_email(to, subject, body)` tool with an attacker-controlled `to` is the most direct exfiltration channel.

The defense pattern is the same shape in each case: the *contents* of tool arguments are part of the trust boundary, not just the *fact* of the tool call. A tool that accepts a URL needs to validate the URL. A tool that accepts an email body needs to consider that the body may contain things the user did not authorize sending. A tool that writes anywhere needs the same kind of input validation that the rest of your application does, and it needs it because the rest of your application is no longer the only thing writing to that store.

## Resource consumption

Briefly, because it is its own LLM Top 10 entry (LLM10) and because it has its own remediations: tools enable denial-of-wallet attacks. A `fetch_url` tool can be pointed at a multi-gigabyte resource. An `execute_code` tool can be pointed at an infinite loop. A `run_query` tool can be pointed at a query that scans every row in your largest table. The model has no good intuition for resource cost; the attacker can build one.

The mitigations are the boring ones: timeouts on every tool invocation, max-bytes limits on every fetch, query-cost estimation before execution, per-session and per-user rate limits, alerting on anomalous tool-call frequency. Boring is correct. The interesting part of this chapter is upstream of resource limits.

## A different framing: the agent is your service account

Imagine your CI/CD system. It has a service account with deployment permissions. You would not give it permissions to also email your customers, also fetch arbitrary URLs from production, also run arbitrary SQL against the production database. You would give it the narrowest possible permissions for its job. If you needed it to also do another job, you would give it a separate, narrowly-scoped credential for that, and you would log every use.

The agent is your service account, with one extra property: it can be talked into using its permissions in ways its designers did not anticipate. This is a strict downgrade from the deterministic-CI-system case. Treat it accordingly. Every time you are tempted to give the agent another tool, ask: would I give my CI system this permission, with the knowledge that an external party can sometimes convince my CI system to use the permission as the external party prefers? If the answer is no, the agent doesn't get the tool either.

This framing is the load-bearing one. It will outlast the specific tool patterns and specific attack scripts. The day frontier models become genuinely robust to indirect injection (which may be a long time off, and may never fully arrive), the framing will still apply, because tool permissions are forever a question of what privileges you have given a non-deterministic component to act on your behalf.

## Sources

- OWASP LLM Top 10 (2025), LLM06 (Excessive Agency).
- Anthropic, "Mythos red-team report," April 2026, which discusses the broad-tool-permission failure mode in the model's own training environment.
- Greshake et al., the indirect prompt injection paper from chapter 4, which analyzes tool-using agents as the highest-impact target class.
- The "confused deputy" formalization originates with Norm Hardy, 1988, "The Confused Deputy: (or why capabilities might have been invented)," ACM SIGOPS Operating Systems Review. Worth re-reading in this context — the framing is forty years old and still load-bearing.
- Embrace the Red, ongoing series on real-world agent abuse demonstrations against shipped products.
- Anthropic, "Computer use in Claude," documentation and security advisories, 2024–2026, for the broader category of "the model can use the user's actual computer," which intensifies every concern in this chapter.
