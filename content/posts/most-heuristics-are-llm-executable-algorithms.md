---
title: "Most Heuristics Are LLM-Executable Algorithms"
date: 2026-04-11
draft: false
tags: ["ai", "tools", "observations", "heuristics"]
summary: "Engineers and domain experts already have the algorithms for a huge class of useful tasks. They just don't recognize them as algorithms because they're written in natural language. The LLM isn't the brain — it's the runtime."
---

I keep a file called HEURISTICS.md. It is a list of reusable patterns for generating product ideas — not ideas themselves, but the shapes that produce families of ideas when you point them at the landscape.

Each heuristic has a "scan prompt" at the bottom. Here is one of them, slightly simplified:

> *What web services or tools limit users by number of entries or items, but don't meaningfully limit the total size or complexity per entry?*

I wrote that as a thinking aid. A prompt for *me* to stare at when I was browsing ProductHunt or the Chrome Web Store, looking for gaps in the market.

But at some point I realized: that is not a thinking aid. That is an algorithm I can run.

## The Algorithm You Already Wrote

A heuristic, stated plainly, is: "Given a space of things, here is how to filter for interesting ones." It is a judgment process — look at X, check for Y, flag when Z.

That is exactly what LLMs are good at.

Take the scan prompt above. Pair it with a data source — say, a dump of the first three pages of ProductHunt from this week. Feed both to an LLM. The model reads the product descriptions, applies the heuristic, and flags candidates where the pattern seems to hold.

Is it perfect? No. Does it surface things I would have missed because I would never have scrolled that far? Yes.

The scan prompt was already code. I just had not noticed because it was in English.

## This Is Not About Intelligence

There is a noisy conversation about whether LLMs can "really reason." This sidesteps that entirely.

Running a heuristic against data is not reasoning in the hard philosophical sense. It is pattern matching against stated criteria. It is closer to grep than to insight. The LLM does not need to be smart. It needs to be literate and tireless.

A human expert applying a rule of thumb to a hundred items will get bored around item thirty, start skimming by item fifty, and quit by item seventy. The LLM does not get bored. It does not skim. It applies the exact same heuristic to item ninety-nine that it applied to item one.

That is the value proposition. Not brilliance. Consistency at scale.

## A Concrete Example

Here is another heuristic from my file:

> *What Chrome extensions or tools solve a useful problem for one specific service that isn't inherently tied to that service? Could the same solution work across multiple platforms?*

The idea: developers build for their own pain point on their own platform. They do not ask "is this problem universal?" The result is a fragmented landscape of narrow tools solving the same underlying problem in isolation.

I ran this as a prompt against a batch of Chrome Web Store extension descriptions. The LLM flagged several extensions that enhance one chat UI (say, ChatGPT's) but could trivially work for others (Claude, Gemini). It flagged search-enhancement tools scoped to one site when the underlying UX gap exists everywhere.

Some of the flags were obvious once pointed out. Some I genuinely had not considered. None required the LLM to do anything harder than "read a description, apply a filter, explain why it matched."

## The General Principle

If you can state a heuristic as a scan prompt + point it at inspectable data = you have an automated scan.

This works for:

- **Product opportunity mining.** The example above.
- **Hiring screening.** "Does this resume suggest someone who has *operated* a system at scale, not just built demos?"
- **Content moderation.** "Does this post pattern-match against engagement bait without being obviously spam?"
- **Deal sourcing.** "Does this startup's description suggest they are solving a real workflow problem or just adding AI to a noun?"
- **Code review.** "Does this PR introduce a dependency that is heavier than the problem it solves?"

In each case, a domain expert already makes this judgment. They apply an unstated rule of thumb. The rule of thumb can be stated. Once stated, it can be prompted. Once prompted, it can run continuously.

## Where It Breaks Down

Not all heuristics are equally automatable.

The ones that work well are **criteria-based**: "check for X, flag when Y." The inputs are text or structured data. The judgment is more "does this match?" than "how does this feel?"

The ones that resist automation are **taste-based**: "Is this image aesthetically powerful?" "Does this song capture emotional tension?" "Is this essay actually good?" These require something closer to a subjective model of the person doing the evaluating, not just a filter. LLMs can assist here — they can articulate what makes something work, they can apply stated aesthetic criteria — but they cannot fully replace the human in the loop. Not yet.

The boundary is roughly: if the heuristic survives being written down as a scan prompt without losing its essence, it is automatable. If writing it down feels like it flattens the thing, the automation will be lossy.

## The Self-Referential Punchline

This observation — "most heuristics are LLM-executable algorithms" — is itself a heuristic.

Its scan prompt would be: *"In some domain, what expert judgment calls follow a pattern like 'look at X, check for Y, flag when Z'? Can the inputs be fed to an LLM? If so, the scan is automatable — and the automated version is a product."*

Which means the meta-heuristic is also LLM-executable.

You could run it. You could feed it descriptions of expert workflows from any industry and have it surface the ones where the expert's judgment process is statable, the data is inspectable, and the automation gap has not been filled.

I wrote the algorithm. I just needed to notice it was one.
