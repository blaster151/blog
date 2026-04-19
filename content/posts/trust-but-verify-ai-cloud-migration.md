---
title: "Trust but Verify: How a 15-Minute Doc Check Saved Our Cloud Migration"
date: 2026-04-19
draft: false
tags: ["ai", "azure", "cloud", "migration", "observations"]
summary: "We were about to deploy to a service scheduled for retirement. A simple 'are you sure about that?' to the AI caught it — and three other gotchas that would have cost us a full day of debugging."
---

We're migrating a Next.js recruiting platform called Prism from GCP to Azure. The codebase is refreshingly portable — one GCP-specific SDK call (Google Document AI for OCR), everything else talks standard protocols. The migration plan practically wrote itself.

That's exactly when you should get suspicious.

## The Setup

I'd been pairing with an AI coding agent (GitHub Copilot, Claude backbone) on this migration for a couple of sessions. We'd done a full codebase audit, written a comprehensive migration plan, scaffolded the Azure OCR provider, merged prep work to master. All tests passing. Feeling good.

The next step was a **#dryNotDry run** — spin up real Azure resources on a personal subscription, deploy the app with `noop` providers, poke at it for an hour, tear it down. Estimated cost: under $2. The goal wasn't to go live, it was to surface friction points early.

But right before we reached for the credit card, I asked a question that turned out to be worth its weight in gold:

> *"Is there any chance that your ambient knowledge of Azure has been superseded since then by changes to their APIs, etc?"*

## The Honest Answer

The agent's response was refreshingly candid: yes, absolutely, its training data has a cutoff, and cloud services change constantly. It offered to go verify against live Microsoft Learn docs before we provisioned anything.

So that's what we did. The agent fetched the current documentation for every Azure service in our migration plan — Container Apps, PostgreSQL Flexible Server, Redis, Document Intelligence — and cross-referenced what the live docs said against what the plan assumed.

Fifteen minutes of fetching and reading. Four findings. Two of them would have been deployment-breaking.

## Finding 1: We Were About to Deploy to a Retired Service

The migration plan recommended **Azure Cache for Redis (Basic C0)** as the replacement for our GCP Memorystore instance. Seemed reasonable — it's the Azure Redis product everyone knows.

One problem: **Microsoft has announced retirement of all Azure Cache for Redis SKUs.** Basic, Standard, and Premium tiers are all end-of-life September 30, 2028. Enterprise tiers even sooner — March 31, 2027.

The replacement is **Azure Managed Redis** — a fundamentally different product based on Redis Enterprise 7.4. Different tier names, different pricing structure, different port number (10000 instead of 6380), different hostname pattern (`*.redis.azure.net` instead of `*.redis.cache.windows.net`), and crucially, different default behavior.

If we'd deployed with the plan as-written, we'd have been provisioning a service that Microsoft is actively sunsetting. Not immediately catastrophic, but exactly the kind of thing that turns into a mandatory re-migration a year from now.

## Finding 2: The Clustering Default That Would Have Broken Everything

This was the real save.

Azure Managed Redis is **clustered by default**. Our app uses BullMQ for background job processing — OCR, text extraction, candidate indexing, the whole worker pipeline. BullMQ is a fantastic job queue library, but it uses multi-key Redis operations internally: scripted `EVAL` calls that touch multiple keys, `BRPOPLPUSH` across queues, that sort of thing.

Clustered Redis restricts cross-slot operations. If two keys hash to different cluster slots, multi-key commands fail. On a default Azure Managed Redis instance, our entire worker pipeline would have silently broken. Jobs would fail, queues would stall, and the error messages from Redis cluster mode are not exactly self-explanatory if you're not expecting them.

The fix is simple: Azure Managed Redis offers a **non-clustered option** for instances up to 25 GB. Our dev workload uses maybe 50 MB of queue data. But you have to *know* to ask for non-clustered mode during provisioning. It's not the default.

Without this verification step, here's the likely timeline:
1. Provision Azure Managed Redis (default = clustered)
2. Deploy app and worker
3. App starts, health checks pass, sign-in works — everything looks fine
4. Upload a resume → job enters queue → worker picks it up → **mysterious Redis error**
5. Spend 2-4 hours debugging, eventually discover the clustering issue
6. Re-provision Redis as non-clustered, redeploy

That's a full day of debugging on what was supposed to be a quick throwaway deployment. All avoidable.

## Finding 3: The Extension Allowlist Nobody Told You About

Our app uses pgvector for semantic search — vector similarity over candidate embeddings. On GCP Cloud SQL, you just run `CREATE EXTENSION IF NOT EXISTS vector;` and it works.

On Azure PostgreSQL Flexible Server, there's an extra step: you must first **allowlist** the extension via the `azure.extensions` server parameter. Without this, `CREATE EXTENSION vector` fails, which means Prisma migrations fail, which means the app doesn't start.

```bash
az postgres flexible-server parameter set \
  --resource-group <rg> --server-name <server> \
  --name azure.extensions --value vector
```

Not hard to fix. But you won't find this requirement by reading PostgreSQL documentation — it's an Azure-specific administrative gate. It has to go in the provisioning script, before database migrations run.

## Finding 4: The Name Changed (Again)

This one's cosmetic but worth noting: "Azure AI Document Intelligence" has been rebranded to **"Azure Document Intelligence in Foundry Tools."** The SDK package name and API are unchanged (`@azure-rest/ai-document-intelligence@1.1.0`, v4.0 GA), so no code impact. But when you're navigating the Azure portal looking for the service to provision it, the name matters.

For those keeping score, this service has been: Form Recognizer → Azure AI Document Intelligence → Azure Document Intelligence in Foundry Tools. Three names in three years. Classic Azure.

## The Meta-Observation

Here's what I find interesting about this interaction pattern. The AI generated the migration plan. The AI also caught the errors in the migration plan — but **only when asked to verify**. Left to its own devices, it would have confidently recommended a service that's being retired, with a default configuration that would have broken our job queue.

This isn't a knock on AI-assisted development. The migration plan was genuinely excellent — comprehensive codebase audit, correct identification of the single GCP-specific code dependency, proper understanding of the provider pattern, sensible phasing strategy. It saved days of manual analysis. The ambient knowledge was 95% right, which is both impressive and dangerous — because 95% right feels like 100% right until you hit the 5%.

The takeaway isn't "don't trust AI." It's **"trust but verify, specifically at the boundaries."** The boundaries in this case are:
- **Service names and availability** (services get renamed, retired, replaced)
- **Default configurations** (especially anything involving clustering, authentication, or network policies)
- **Cloud-specific administrative requirements** (extension allowlisting, parameter gates, quota limits)

These are exactly the things that change between training data cutoffs, and exactly the things that cause silent deployment failures.

## The Recipe

For anyone doing a similar AI-assisted cloud migration:

1. **Let the AI do the audit.** It's genuinely good at scanning a codebase and identifying cloud-specific coupling points. Ours found that only one file in the entire codebase had a GCP-specific import — a finding that would have taken hours manually.

2. **Let the AI write the plan.** It knows the Azure/GCP/AWS service mappings, configuration equivalences, and typical migration patterns. The structural plan was solid.

3. **Before provisioning anything, ask it to verify.** Literally: "Check the live docs for every service you recommended." Fifteen minutes of fetching web pages. It will find its own mistakes — and they'll be the kind of mistakes that would have cost you a day of debugging.

4. **Record what changed.** We added an appendix to the migration plan documenting every stale-knowledge correction. Next time an AI agent reads that plan in a fresh conversation, it won't repeat the same mistakes.

The whole verification exercise took about fifteen minutes. It caught a retired service, a clustering default that would have broken our job queue, a hidden extension allowlisting requirement, and a service rebrand. Call it $2 of API costs and 15 minutes of wall-clock time, versus at least a full day of deployment debugging.

I'll take that trade every time.
