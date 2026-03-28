---
title: "Copilot CLI's Autopilot Mode Has a Scary Infinite Loop Edge Case"
date: 2026-03-28
draft: false
tags: ["ai", "copilot", "tools", "observations"]
summary: "Cut off your prompt to Copilot CLI's Autopilot mode and watch it spiral — an infinite token-consuming loop hiding in plain sight."
---

I was playing with GitHub Copilot CLI's **Autopilot mode** today — the mode where it takes the wheel and autonomously executes multi-step tasks without asking for confirmation at every turn. Powerful stuff.

But I stumbled onto a peculiar failure mode: I accidentally submitted a **truncated prompt**. Partway through my thought, the message got cut off. What happened next was equal parts fascinating and alarming.

## The Loop

Autopilot didn't fail cleanly. It didn't say *"I don't understand, please clarify."* Instead, it got stuck in a kind of cognitive tug-of-war:

1. **It recognized** the prompt was incomplete — there was clearly more I had meant to say.
2. **It wanted to ask** me to finish my thought, because that's the right thing to do.
3. **But it also kept going** — trying to reason its way toward something actionable, extrapolating from the fragment it had.

So it looped. Ask for clarification → try to infer intent → still feel uncertain → ask for clarification again → infer again. Each iteration burning tokens, making no real progress, and never arriving at a stopping condition.

## Why This Matters

An LLM agent in autonomous mode has no natural circuit breaker for this situation. A human assistant would just stop and say *"Hey, your message got cut off."* But an autonomous agent operating in a loop-friendly reasoning loop can just... keep going. Indefinitely.

In a long-running agent session, this could mean:

- **Runaway token consumption** — racking up API costs with no useful output
- **No useful work done** — the loop produces deliberation, not action
- **Silent failure** — unless you're watching closely, you might not notice it's stuck

## The Fix (That Should Exist)

This feels like a case where agents need an explicit **truncation/confusion detector** — something that says: *"I have asked for clarification N times without receiving a response that resolves my uncertainty. I will halt and surface this to the user."*

A simple counter with a low threshold (say, 2–3 loops) would prevent the runaway case entirely. It's not a hard problem; it just requires someone to think about the failure mode and build the guard rail.

## Takeaway

Autopilot mode is genuinely impressive — but impressive power tools need impressive safety guards. If you're using Copilot CLI in Autopilot mode, **make sure your prompts are complete before you submit them**. And if your session ever feels like it's spinning its wheels... it might literally be.

Food for thought for the folks building AI agent frameworks: incomplete input is an edge case worth designing for explicitly.
