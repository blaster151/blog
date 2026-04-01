---
title: "I Used NotebookLM to Build a Systematic Code Review Machine"
date: 2026-03-31
draft: false
tags: ["ai", "code-quality", "tools", "copilot", "notebooklm", "observations"]
summary: "What happens when you feed NotebookLM your entire dependency stack, ask it to generate best-practices review prompts for each technology, and then run those prompts as automated code reviews — including cross-dependency combinations."
---

Here's an experiment I ran recently that felt like it crossed a line from "using AI tools" into "building an AI-powered quality machine" — and it surfaced some genuinely uncomfortable questions about trust, diminishing returns, and the seductive fiction that if all the AIs agree, the code must be good.

## The Setup

I have a reasonably complex TypeScript project. The kind with a `package.json` that tells a story: Next.js, Prisma, Zod, pgvector, a Document AI client wrapper, various auth and ingestion libraries. Each of those technologies has its own universe of best practices, gotchas, and antipatterns that no single developer keeps in their head at all times.

The idea: what if I could **systematically** extract the collective wisdom for each technology in my stack, then convert that wisdom into targeted code review prompts?

## The Process

### Step 1: Build the Corpus

For each major technology in my `package.json`, I used [NotebookLM](https://notebooklm.google.com/) to gather a thorough corpus of:

- **Best practices** — the things the docs recommend but rarely enforce
- **Guidelines** — the patterns that experienced teams converge on
- **Known gotchas and antipatterns** — the mistakes you make once and swear you'll never make again (and then make again)

I didn't just look at the npm package names literally. A dependency like a "Document AI client wrapper" implies I should also research the external service it's wrapping — Google Document AI's own quirks, rate limits, error handling expectations, and data handling requirements.

One technology at a time. Thorough, not fast.

### Step 2: Generate Review Prompts

For each technology's corpus, I asked NotebookLM to **phrase a comprehensive, LLM-targeted architecture/design/code review prompt.** Not "review my code." Something more like:

> "Given a codebase that uses Prisma with PostgreSQL, review the following areas against established best practices: schema design, migration safety, query performance (N+1 detection, select specificity), connection pooling, error handling patterns, type safety at the boundary layer, and seed/test data management. Flag any deviations from the documented guidelines and explain the risk of each."

NotebookLM is good at this because it has the full corpus in its context window — it's essentially writing a review prompt that's informed by everything it just read.

### Step 3: Run the Reviews

This is where it gets interesting. Each generated prompt can be run:

- **Manually and at leisure** — paste it into Copilot Chat in your IDE, point it at relevant files, read the results
- **Delegated to cloud agents** — and here's the neat trick: with GitHub Copilot, you can paste each review prompt into a **GitHub Issue**, click "Assign to Copilot," and you've just created a cloud task that will autonomously run your custom-built review against the codebase, find deficiencies, propose remedies, and submit a PR

That second path turns your stack of review prompts into a parallel, asynchronous code quality pipeline. You go make coffee. The agents go find your sins.

### Step 4: The Combinatorial Layer

This is where it got really fun — and really time-consuming.

Individual technologies have best practices. But *combinations* of technologies have *interaction patterns* that neither technology's docs cover well. What does it look like when Zod and Prisma are used together properly? What are the gotchas when pgvector search results need to be validated through Zod schemas? What about NextAuth session types flowing through Prisma queries?

So I started identifying which **pairs of dependencies** from my stack would benefit from a combined best-practices search. Not every combination — just the ones where the interaction surface is real:

- Zod + Prisma (schema validation at the ORM boundary)
- Next.js App Router + NextAuth (middleware, route protection, session access patterns)
- Prisma + pgvector (hybrid search, index management, query composition)
- Document AI + Zod (parsing uncertain external output into validated types)

Each combination got its own NotebookLM corpus and its own generated review prompt.

## What I Found

The reviews surfaced real issues. Not catastrophic vulnerabilities — but the kind of accumulated drift that makes codebases subtly worse over time:

- Inconsistent error handling across API routes (some threw, some returned error responses, some did both)
- Missing `select` clauses on Prisma queries that fetched entire rows when only two fields were needed
- Zod schemas that validated input shapes but not semantic constraints (accepting negative values for quantities, for instance)
- Auth middleware that protected routes but didn't verify authorization *within* those routes for specific resources

Good catches. Legitimate improvements. Each PR made the code measurably better.

## The Uncomfortable Questions

### Is there a point where there's no more quality to wring out?

Theoretically, yes. Practically, I didn't reach it — but I could *feel* the returns diminishing. The first few reviews found meaningful issues. The later combinatorial reviews increasingly flagged things that were technically imperfect but practically fine. "This could be more type-safe" started to feel like "this could be more type-safe in a way that would never actually prevent a bug."

There's a real risk of over-optimizing: refactoring working code to satisfy a best-practices checklist that was generated by an AI that read some docs. At some point you're not improving the system, you're polishing it for an audience of zero.

The honest answer is that quality is asymptotic. You get 80% of the value from the first pass. The combinatorial layer gets you another 15%. After that, you're in the territory of diminishing returns — and the opportunity cost of *not building new features* starts to dominate.

### The Merge Temptation

This is the one that kept me up at night.

When an AI generates a review prompt, an AI runs the review, an AI writes the fix, and an AI proposes the PR — and the changes look reasonable, and the tests pass — there is a **powerful gravitational pull** toward clicking Merge.

After all, multiple AI systems *agree* that this is the right fix. The code compiles. The tests pass. The review prompt was grounded in real best practices. What could go wrong?

Everything. Everything could go wrong.

The AIs don't disagree with each other because they're drawing from the same well of training data and reasoning patterns. Agreement between AIs is not the same as independent verification. It's more like asking the same person the same question in three different accents.

I caught myself merging a PR that "fixed" an error handling pattern by wrapping every database call in a try-catch that swallowed errors and returned empty arrays. Every AI in the chain thought this was fine because "the function no longer throws." Technically true. Practically, it turned database failures into silent data loss.

The lesson: **AI-generated code reviews are discovery tools, not authority.** They find things worth looking at. They do not replace the judgment of someone who understands what the system is *supposed to do,* not just what it *currently does.*

## The Meta-Observation

What I actually built here is a **quality amplifier with a trust problem.** The process is genuinely powerful — it found real issues I wouldn't have found on my own, across technology boundaries I wouldn't have thought to audit. But the output requires the same critical reading you'd give a junior developer's PR: assume good intentions, verify everything, and never confuse "looks right" with "is right."

The most valuable artifact wasn't any individual PR. It was the collection of review prompts themselves — reusable, refinable, and specific enough to catch the class of issues that "just review my code" never finds. Those prompts are the real product of this experiment.

Would I do it again? Absolutely. Would I let the PRs auto-merge? Absolutely not.
