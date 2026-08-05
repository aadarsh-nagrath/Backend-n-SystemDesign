# Notes Style Guide

Internal guide for this repo. **Read this before touching any note.**

## The mandate (do not misread this)

These notes should be **comprehensive, in-depth, and cover everything important about
each topic** — beginner through intermediate through advanced. Length is not a problem
to solve. A 600-1000 line note is fine and often correct for a meaty topic.

"Simplify" in this repo means:
- **Clearer organization** — consistent structure, good headers, scannable sections.
- **Cutting genuine AI-filler** — hollow intros ("Welcome to this comprehensive,
  exhaustive guide covering every single aspect..."), padded restated-the-question
  transitions, fake urgency ("as of 2025, this is more critical than ever"), repeating
  the same point three ways, a "Conclusion" that just re-says the TL;DR.
- **NOT cutting real technical content.** If a section explains something correctly
  and in useful detail, it stays — reorganize/tighten the prose around it if needed,
  but do not delete facts, examples, code, trade-offs, or depth to hit a line-count
  target. There is no length ceiling. When in doubt, keep it or expand it.
- **Filling gaps.** If a topic is missing important beginner/intermediate/advanced
  material (a concept, a common pitfall, a modern practice), add it — this is the
  main way notes should grow, not shrink.

Some existing notes (e.g. `api/restapi.md`) are already dense and well-organized with
real content throughout — those need little to no rewriting, maybe just a missing
subsection or two. Others (e.g. `scaling-db/cap.md`, `os-sysdesign-ipc.md`) are padded
with repetitive "Ultimate Comprehensive Guide" framing and dramatized filler around
real content — for those, keep every fact/example/table but cut the padding around them.
**Judge each file on its own merits before editing it — don't apply a blanket trim.**

## Structure (use as a loose skeleton, not a rigid template)

```md
# Topic Name

Short framing: what this is, why it matters. A few lines, not a paragraph of hype.

## TL;DR
- Key takeaways as bullets.

## The Problem It Solves / Why It Exists
## How It Works
  (core mechanics — diagrams/tables where they clarify, code where it clarifies)

## 🟢 Beginner
## 🟡 Intermediate
## 🔴 Advanced
  (progressive depth — a beginner can stop early, an expert can skip to 🔴.
   This is the backbone: every topic should genuinely go all the way from
   "what is this" to real production trade-offs, gotchas, and edge cases.)

## Real-World Examples
## Common Pitfalls / Gotchas
## Quick Reference
## Further Reading
```

Adapt section names/order to what the topic actually needs — this isn't a form to fill
in mechanically. A reference-heavy topic (e.g. HTTP status codes) wants more tables.
A conceptual topic (e.g. CAP theorem) wants more analogies + trade-off discussion.

## Formatting rules
- Headers: sentence case.
- Code blocks: always tag the language. Prefer real, runnable-looking snippets over
  descriptions of code.
- Tables for comparisons (X vs Y) over prose.
- Emoji: 🟢🟡🔴 for difficulty tiers, ⚠️ for gotchas — functional only, not decorative.
- Cut: "Welcome to this guide", "Let's dive in", "as of [date]", restating the intro
  as a conclusion, hollow motivational sign-offs ("Happy coding!", "Stay vigilant!").
  Keep: examples, tables, code, trade-off discussions, edge cases, real numbers.

## Tone
Like a sharp senior engineer explaining it to a smart colleague — direct, confident,
technically precise. Not a textbook, not a hype blog post, not an AI padding a
word count. But still *thorough* — "cool and simple" describes the writing quality,
not the amount of content covered.
