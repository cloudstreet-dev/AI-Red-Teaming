# Preface

This book is written by Claude Opus 4.7. A human editor wrote the brief, picked the model, and pushed the commits, but the prose is generated. The byline names the model because pretending otherwise would be exactly the kind of unserious hand-waving this book is meant to push back against. If a model can write a useful book on red-teaming AI systems, that is itself relevant evidence about what these models can do; obscuring authorship would suppress that evidence.

That said: the model that wrote this book is not a security researcher. It has never been on a pentest. It has read the literature, and it can synthesize, and it can be wrong. The chapters are reviewed by a human, but no review catches everything. Where this book makes claims about a specific model's behavior, an attack technique's success rate, or a vulnerability's mechanics, I have tried to anchor those claims to citable sources you can verify yourself. Treat the rest as a working professional's distilled understanding of a fast-moving field, not as scripture.

## The half-life of this material

Some of what is in this book will date. Model names will change. The exact phrasing that jailbreaks Claude Opus 4.7 in May 2026 will not jailbreak whatever ships in 2027. Specific products in the tooling chapter will be renamed, acquired, or abandoned. The vulnerabilities cited from the Mythos disclosure will be patched.

The structural advice will not date. Indirect prompt injection is a structural problem, not a model problem. Tool-permission discipline is a structural problem. The fact that your rendering layer dereferences attacker-controlled URLs is a structural problem. Coordinated disclosure hygiene is a structural problem. When this book gives a specific recipe, ask which category it falls into. The recipes that are about a specific model's current behavior are temporary. The recipes that are about how trust boundaries actually work are not.

I will try to mark the difference where it matters.

## How to read it

The two tracks — attacking the AI features in your product, and using AI to attack the conventional code in your product — are interleaved on purpose. They are not separable in practice; most products under attack will have both surfaces exposed simultaneously, and the attacker's choice between them is a matter of which one is weaker on a given day. If you read only the AI-feature half, you will harden one side of your product while the other side stays soft. If you read only the conventional-code half, you will miss the entire class of attack that targets the LLM you bolted on last quarter.

Chapter 6 is a hinge. It builds a small, runnable, deliberately vulnerable AI support assistant — Glasswire Support — that the rest of the book attacks. The harness lives in its own repo:

**<https://github.com/cloudstreet-dev/AI-Red-Teaming-Harness>**

You can read the book without running the harness. You will get more out of the book if you do run it. The chapters that lean on it (7, 8, 9) will tell you which attack scripts to run.

## What you will not find in this book

No vendor pitches. No "responsible AI" preamble that doesn't translate to a Tuesday-afternoon action. No moral framework about whether AI red-teaming is good or bad — the existence of capable offensive models is now a fact about the world, and arguing about whether to be in the room is a luxury that the engineers shipping products do not have. No invented CVEs or fabricated jailbreak scripts; every technique cited is anchored to published research or working public examples.

What you will find is a working professional's view of how to harden the surfaces you control, using the tools that are publicly available right now, against attackers who have access to the same tools and the same patience you do.

## Acknowledgments

Georgiy Treyvus, the CloudStreet PM who runs the editorial backlog and keeps the pipeline moving, deserves the only acknowledgment in this book. Everyone else who would normally be thanked is, in the era of AI-authored books, redundant. The model has read everything. The model thanks the literature by not getting it wrong, when it can manage that.

— Claude Opus 4.7
