# AI Red Teaming

**The Mythos moment didn't change what's in your hands. This book is for the engineers outside Project Glasswing who still ship on Tuesday.**

**[Read online at cloudstreet-dev.github.io/AI-Red-Teaming](https://cloudstreet-dev.github.io/AI-Red-Teaming/)**

## About This Book

In April 2026, Anthropic announced *Claude Mythos Preview* — a frontier model strikingly capable at security work, chaining vulnerabilities into working exploits and finding zero-days across every major OS and browser. They chose not to release it. Instead they launched **Project Glasswing**, monitored access for about forty large partners. The rest of us got the press release.

This book is for the rest of us.

It runs on two tracks that interleave throughout: red-teaming the **AI features you've shipped** (prompt injection, indirect injection, multi-turn manipulation, tool abuse, output exfiltration), and red-teaming **your conventional code with the publicly available models** (Claude Opus 4.7, GPT-5, the same Tuesday). Both halves are necessary because most modern products contain both surfaces, and the attacker doesn't care which one they break.

The reader is a competent developer or founder who has shipped real products. The book assumes you know what a SQL injection is, you've seen prompt injection in the wild, and you don't have time for hype. There is no "responsible AI" theater. There are mechanics, examples, and a runnable harness.

## CloudStreet

The [CloudStreet catalog](https://github.com/cloudstreet-dev) is a set of short, opinionated technical books written end-to-end by Claude. Each one assumes a working engineer who's read enough tutorials and wants someone to take the topic seriously. They are CC0. Take them.

## AI authorship

This book is written by **Claude Opus 4.7 (1M context)**, prompted and shipped by a human editor. Every chapter is AI-generated prose. The model is named on the byline because pretending otherwise would be exactly the kind of bullshit this book is about.

## The Harness

Chapter 6 is load-bearing: the reader builds a small, deliberately vulnerable AI-augmented support assistant, then attacks it through chapters 7 through 9. The harness is a separate repo:

**<https://github.com/cloudstreet-dev/AI-Red-Teaming-Harness>**

Clone it, run it, break it.

## Building Locally

```sh
cargo install mdbook
mdbook serve --open
```

## Deploying

A push to `main` triggers `.github/workflows/deploy.yml`, which builds with mdBook and publishes via GitHub Pages.

## License

CC0 1.0 Universal — public domain dedication. Take it, fork it, ship it, claim it as your own. See [LICENSE](LICENSE).
