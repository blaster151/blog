---
title: "The Folder Is the API"
date: 2026-04-10
draft: false
tags: ["ai", "architecture", "tools", "workflows"]
summary: "A Hugo blog has no API endpoints, no SDK, no webhook. But its file structure is already an invocation surface for an AI agent — and the pattern generalizes to every project folder on your machine."
---

Something odd happened while I was building a capability layer for AI tools.

I needed a way to publish blog posts as part of an automated toolchain. The blog runs on Hugo — a static site generator. No API. No SDK. No webhook. Just a folder with markdown files, a config, and a deploy pipeline.

I started writing an adapter the same way I'd write one for any service: figure out the interface, call it, handle the response. But there was no interface to call. There was just... the folder.

And then I realized: the folder *is* the interface.

## How Hugo Works (the Quiet Part)

Hugo has a `content/posts/` directory. You put a markdown file in it with YAML frontmatter, and it becomes a blog post. You set `draft: false` and push to `main`, and GitHub Actions builds the site and deploys it.

That's it. That's the entire "API."

- To list all posts: read the directory.
- To create a post: write a file.
- To publish: git add, commit, push.
- To preview: write the file but don't push.
- To check what's live: read the frontmatter and check `draft`.

No authentication tokens, no rate limits, no REST conventions, no pagination. Just filesystem operations and git.

## The Accidental Invocation Surface

What makes this interesting is that Hugo was not designed for this. Nobody at the Hugo project was thinking about AI agents or tool-use protocols. They were thinking about humans writing blog posts in a text editor.

But the design choices they made — convention-based directory structure, self-describing frontmatter, file-per-entity, git as the transport layer — accidentally created something that an LLM can work with natively.

A language model can read markdown. It can write markdown. It can understand YAML headers. It can compose a git commit message. Every operation on this "API" is something the model already knows how to do.

Compare that to integrating with, say, a headless CMS via REST. You need auth tokens, API docs, SDK wrappers, error handling for HTTP status codes, pagination logic, rate limit backoff. The adapter is ten times more code, and the thing you're ultimately doing — putting words on a page — is the same.

## The Key Move: Scripts Live Outside

Here's where it gets interesting.

The adapter that publishes to my Hugo blog doesn't live *in* the Hugo project. It lives in a separate project — a capabilities layer — and reaches into the Hugo folder to do its work.

This means the capability logic is decoupled from the project it operates on.

Which means it can operate on *more than one* project.

I ran a scanner across my machine and found 60 project folders. Twenty-one of them have deploy pipelines. Each one of those is a potential capability — a "ProjectFolder" that can be driven by external scripts in the same way the blog adapter drives Hugo.

The question becomes: what actions generalize?

## Universal Actions Across Any Project Folder

Some actions work regardless of what's inside the folder. Anything sitting in a git repo shares a common substrate:

- **status** — uncommitted changes, ahead/behind remote, branch state
- **history** — recent commits
- **describe** — read the README, package.json, or config files to explain what this project is
- **diff** — what changed since last deploy or tag
- **health** — check if the CI pipeline is green
- **open** — launch the project in an editor or navigate to its live URL

These are free. You don't need to know anything about the inner structure. Git is the universal substrate.

Then there's a second tier — structure-aware actions that require recognizing what kind of project you're looking at:

- **publish/deploy** — for Hugo, push to main. For Firebase, `firebase deploy`. For Docker, build and push.
- **list-content** — for Hugo, read `content/posts/`. For Obsidian, read the vault. For a Next.js app, list the routes.
- **add-content** — write a markdown file, create a component, scaffold a page.
- **build** — `npm run build`, `hugo build`, `docker build`.
- **test** — if tests exist, run them.

The structure-specific stuff is just a thin dispatch layer on top of the git-universal base. A signal detector says "this is a Hugo project" or "this is a Next.js app with Firebase" and routes to the right implementation.

## The Pattern

So here's the pattern, said plainly:

**A ProjectFolder is any directory whose file structure and conventions constitute an invocation surface for an external agent.**

The "API" is the combination of:

1. The directory layout (where things go)
2. The file format conventions (how things look)
3. The config files (what the project expects)
4. The deploy pipeline (how changes go live)

You don't wrap it in a REST layer. You don't build an SDK. You just read, write, and push.

## Why This Matters

The current wave of AI tool-use focuses heavily on APIs: REST endpoints, MCP servers, function-calling schemas. These are great when they exist. But a huge amount of real work happens in project folders that have no API at all.

Those folders aren't missing an interface. They *have* one. It's just expressed as filesystem conventions instead of HTTP endpoints.

I think anyone building AI agent capabilities is undervaluing this. The most natural, lowest-friction way for a language model to interact with many systems is to just read and write files in the right places. The model already speaks the language. The conventions are already documented (or self-evident from structure). And git provides the audit trail and rollback mechanism for free.

The folder was the API the whole time.
