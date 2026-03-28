---
title: "Three Utilities, One Machine's Life Story"
date: 2026-03-28
draft: false
tags: ["ai", "copilot", "windows", "sysadmin", "experiments"]
summary: "What happens when you ask Copilot CLI to diagnose your entire PC using only three built-in Windows 11 command line tools? Turns out — a lot."
---

Here's a prompt experiment I just ran with GitHub Copilot CLI in Autopilot mode:

> *"Given only, say, three known-to-exist on Windows 11 command line utilities of your choice (any number of calls with any variety of parameters is fine), can you synthesize as comprehensively detailed a knowledge as possible of all the machine details — including things like your opinion of its worst bottlenecks and its lowest-hanging-fruit upgrade options, if any? Whatever you come up with could go into one or more Markdown files. Thank you"*

Simple enough question. But watch what the agent does with it.

## The Tool Selection Phase

Before running a single command, Copilot CLI paused to *reason about the toolbox*. It openly deliberated which three utilities would squeeze out the most signal:

> *"Let me think about which three would be most powerful..."*

It landed on:

1. **`systeminfo`** — for baseline OS, memory, hotfixes, and BIOS details
2. **`wmic`** — deep hardware specs: CPU, GPU, motherboard, drives
3. **PowerShell** — granular performance metrics, network info, WMI/CIM queries

The reasoning was solid. `systeminfo` is broad but shallow. `wmic` is the classic deep-dive for hardware inventory. PowerShell is essentially a superpower — using `Get-CimInstance` and `Get-WmiObject`, you can pull nearly anything the OS knows about itself. Three tools that together cover almost the entire surface area of a machine's self-reported state.

## The Planning Phase

Then it planned *how* to use them efficiently:

> *"I'll organize these into three parallel execution groups to speed things up."*

Rather than running commands sequentially, it grouped queries by tool and planned to parallelize — `systeminfo` for OS details, a series of `wmic` queries for CPU specs, memory, disk drives, video controllers, and BIOS, and PowerShell for disk health and network adapters.

This is the part that impressed me. It didn't just fire off commands — it thought about *throughput*, batching related queries to reduce round-trips.

## Why This Is Interesting

There's something philosophically neat about this constraint. You're essentially asking the agent: *"How much can you learn about a machine using only what the machine already knows about itself?"*

The answer, as it turns out, is: **quite a lot**. Windows 11 exposes an enormous amount of self-knowledge through these three tools alone — CPU topology, RAM slots and speeds, storage interfaces, GPU VRAM, thermal and power profiles, installed hotfixes, BIOS version, NIC details, and more. The raw data is all there. What the agent brings is the synthesis layer: correlating the numbers, identifying mismatches (e.g., fast CPU bottlenecked by slow storage), and translating specs into plain-language upgrade recommendations.

## The Output

The agent's plan was to produce one or more Markdown files — a structured report covering hardware inventory, performance analysis, and upgrade recommendations. A kind of instant PC health report, generated entirely from three native tools and some LLM reasoning on top.

I'll be sharing the actual output once it's done. But the approach alone is worth writing down: **constraint-driven AI prompting often produces more interesting results than open-ended ones**. Giving the agent a tight, creative limitation forced it to think carefully about tool selection and execution strategy rather than reaching for the most obvious solution.

Something to keep in your prompt toolkit.
