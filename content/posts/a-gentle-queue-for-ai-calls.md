---
title: "A Gentle Queue For AI Calls"
date: 2026-07-14
draft: false
tags: ["ai", "rate-limiting", "backend"]
summary: "A token bucket with a FIFO queue that absorbs legitimate creative bursts instead of punishing them."
---

AI features have a funny traffic shape. Most of the time, nothing happens. Then one real user gets curious and clicks through a whole creative workflow: classify a prompt, research ingredients, suggest palettes, generate a project, repair the output, regenerate one section, try again.

That is normal use. It is also exactly the shape that can accidentally look like abuse if every OpenAI call goes straight out the door with no pacing.

So the app now has an instance-wide queued rate limiter around OpenAI calls.

The goal is not to punish a legitimate burst. The goal is to absorb it.

The API is intentionally small:

```
await generationRateLimit.run(() =>
  client.responses.create({
    model,
    input,
    // ...
  })
);
```

That reads the way the system behaves: "run this generation work under the generation rate limit."

The limiter gives the app a burst budget first. With the current defaults:

```
OPENAI_GENERATION_RATE_LIMIT=40
OPENAI_GENERATION_RATE_LIMIT_WINDOW_MS=600000
```

the server can make 40 OpenAI calls immediately within a 10-minute budget. If the burst is spent, the next calls do not fail right away. They wait in a queue and release at the refill rate:

```
600000ms / 40 = 15000ms
```

So after the initial burst, one queued call becomes eligible about every 15 seconds.

That matters for AI-aware UX. A user who is behaving normally may just experience a slower generation rather than a hard "try again later." Meanwhile, a runaway loop or mischievous traffic cannot drain the quota at full speed forever; it gets forced through the same narrow pacing lane.

The implementation is a token bucket with a FIFO queue.

Conceptually:

```
class QueuedTokenBucketRateLimiter {
  async run<T>(operation: () => T | Promise<T>): Promise<T> {
    await this.claimSlot();
    return await operation();
  }
}
```

`claimSlot()` is the interesting part:

- If a token is available, take it immediately.
- If no token is available, put a resolver into a queue.
- A timer wakes up when the next token should exist.
- The queue drains in order as tokens refill.

So the protected operation does not even start until it has a slot. That is important: the limiter paces actual OpenAI calls, not just response handling after the expensive work has already begun.

It is also instance-wide. The shared limiter is created once at module scope and cached on `globalThis`, which helps avoid accidental fresh counters during local hot reloads or duplicate module loads.

This is not a complete abuse-prevention system. It is deliberately simple. It protects the app's OpenAI spend and smooths bursts for a single running server instance. In a larger production setup, I would eventually add per-IP, per-user, or distributed limits. But for a seldom-used prototype that needs a "remote just-in-case" guardrail, this is a nice fit.

The most important design choice is that the queue is humane by default. It does not assume the next burst is hostile. It slows things down, keeps ordering predictable, and lets legitimate creative sessions continue.

That feels right for this kind of app: protective, but not prickly.
