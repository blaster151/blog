---
title: "argparse --help Is a Free Tool Definition"
date: 2026-03-28
draft: false
tags: ["python", "ai", "tools", "til"]
summary: "If a Python script uses argparse, you already have everything you need to auto-generate a tool definition for it. It's been right there the whole time."
---

One of those *"everyone else has known this forever"* moments hit me today.

If a Python script uses `argparse`, you can interrogate it for free:

```bash
python myscript.py --help
```

And the output looks something like this:

```
usage: myscript.py [-h] --input FILE [--output FILE] [--verbose] [--max-tokens INT]

Process documents and summarize them.

options:
  -h, --help        show this help message and exit
  --input FILE      Path to input file
  --output FILE     Path to output file (default: stdout)
  --verbose         Enable verbose logging
  --max-tokens INT  Maximum tokens in summary (default: 512)
```

That output is structured. It has the script's purpose, every argument, each argument's type, and default values. Everything you need to describe a tool to an LLM.

## The Implication

If you're building an AI agent that needs to call arbitrary Python scripts as tools, you don't have to hand-write tool definitions. You just:

1. Run `python {scriptname}.py --help`
2. Parse the output
3. Auto-generate the tool definition

The argument names become parameter names. The description text becomes the parameter descriptions. The type hints (`FILE`, `INT`, etc.) tell you what to expect. Defaults are right there too.

For an OpenAI-style function definition, you'd end up with something like:

```json
{
  "name": "myscript",
  "description": "Process documents and summarize them.",
  "parameters": {
    "type": "object",
    "properties": {
      "input":      { "type": "string", "description": "Path to input file" },
      "output":     { "type": "string", "description": "Path to output file (default: stdout)" },
      "verbose":    { "type": "boolean", "description": "Enable verbose logging" },
      "max_tokens": { "type": "integer", "description": "Maximum tokens in summary (default: 512)" }
    },
    "required": ["input"]
  }
}
```

All of that came from three lines of parsing.

## Why This Is Actually Kind of Elegant

`argparse` was written for humans — so scripts could document themselves for the people running them at the terminal. But that same self-documentation turns out to be exactly what you need for machine consumption too.

The human-readable contract and the machine-readable contract are the same contract. You wrote the tool definition the first time you added `add_argument()` calls to your script. You just didn't know it yet.

## Caveats

- The parsing is a little brittle if someone's `--help` output is unconventionally formatted or uses a custom formatter class.
- You lose some precision — argparse metavars like `FILE` and `INT` are hints, not strict types.
- `argparse` itself actually exposes a proper API (`parser._actions`) you could introspect programmatically instead of screen-scraping `--help` output. Worth knowing for more robust use cases.

But for a quick-and-dirty tool auto-discovery pass across a folder of scripts? `--help` parsing is fast, zero-dependency, and works on *any* argparse-based script without touching the source.

Sometimes the obvious thing really is right there.
