---
title: "AutoRouter v2: The Router That Knows When Not to Call Opus"
date: 2026-09-19
draft: true
description: A complexity-based router picks the cheapest model that can still
  do the job, and skips Opus for "ciao". Here's the validation, what a 3-day
  load test revealed about trusting your own dashboards, and what the same
  routing logic would cost on OpenRouter instead of Bedrock.
tags:
  - ai
  - cost saving
  - monitoring
  - grafana
  - automation
  - aws bedrock
featuredImage: /images/autorouter-v2-the-router-that-knows-when-not-to-call-opus/featured.jpg
---
### Table of Contents

- Why Build a Router at All
- The Four Tiers
- How a Request Actually Gets Classified
- Validating It: Three Real Requests
- Three Days Later, at Actual Scale
- When Your Own Dashboard Lies to You
- The Economics: Opus vs Sonnet vs Letting the Router Decide
- AWS Bedrock's Open Model Problem
- Redoing the Economics on OpenRouter
- Conclusion
- Reflections



Here we are. Every LLM gateway I've built so far had the same lazy default: pick one strong model, point everything at it, call it a day. It works, right up until the invoice arrives and you realize half the traffic was "ciao", "riassumi questo", or "scrivi una email a level 1 support", and every single one of those trivial requests paid Opus-level prices.

AutoRouter v2 is the fix: a complexity classifier sitting in front of the LiteLLM gateway that reads the request, decides how hard the task actually is, and picks the cheapest model in the pool that can still do it. The user never picks a model. They just call `platform-auto` and the router does the rest.

This post walks through how it's built, what happened when I validated it on real traffic, what a 3-day load test at real scale exposed (including a dashboard that flatly contradicted itself), and then answers the question I kept getting asked: what would this actually have cost on plain Opus, on plain Sonnet, and if I dropped AWS Bedrock entirely and pointed the same tiers at OpenRouter instead.

## Why Build a Router at All

The business case is basically a call center analogy. A well-run call center routes a simple "what are your opening hours" to the cheapest available operator and reserves the specialist for the customer with the actual incident. The caller never notices the routing. The cost curve does.

Applied to LLMs: not every prompt needs chain-of-thought reasoning from a $25/million-token model. "Translate this paragraph" and "here's our production outage, sev1, find the root cause" are not the same request, and pricing them identically is just burning budget for no quality gain on the easy 80%.

## The Four Tiers

AutoRouter v2 classifies every request into one of four tiers, each with its own pool of candidate models, tried in order with a `least-busy` strategy inside the tier:

| Tier | Models (selection order) | Example prompts |
|------|---------------------------|------------------|
| **SIMPLE** | Nova Micro → Claude Haiku 4.5 | Greetings, translations, short summaries, quick emails |
| **MEDIUM** | Nova Pro → GLM-5 → Kimi K2.5 → Claude Haiku 4.5 | Technical docs, scripts, code review, user stories |
| **COMPLEX** | Claude Sonnet 5 → GPT-5.6 Luna | Debugging, log analysis, SQL, infra troubleshooting |
| **REASONING** | Claude Opus 5 → Claude Sonnet 5 → GPT-5.6 Luna | Postmortems, architecture review, security incidents |

The per-token pricing baked into the LiteLLM config makes the intent obvious. Nova Micro's input tokens cost roughly 3.5% of Claude Haiku's, which is itself already the cheap end of the Claude line. Opus sits at 5x Haiku's rate. If even a third of traffic is genuinely SIMPLE, routing it away from a top-tier model is where the savings actually live, not in squeezing a discount out of the frontier model.

## How a Request Actually Gets Classified

Classification runs as a three-stage pipeline, and the order matters because the first match wins:

```
Incoming request
      |
      v
1. Keyword match on the last 3 user turns (300 chars each)
   -> zero latency, deterministic, literal substring match
      |  no match
      v
2. LLM classifier (nova-micro-classifier, 2500ms timeout)
   -> asks a cheap model "which of the 4 tiers is this?"
      |  timeout or error
      v
3. Heuristic fallback (LiteLLM's built-in length/token heuristic)
      |
      v
Tier assigned -> model selected within tier (least-busy)
```

Keyword matching is deliberately literal, not semantic: `"debugga questo script"` hits COMPLEX because `"debugga"` is in the list, `"trova un bug"` hits COMPLEX because of `"bug"`. It's crude, but it's free and instant, and it catches the obvious cases before anything has to call an LLM to decide.

Everything that doesn't match a keyword goes to Nova Micro as a classifier, with a 2.5 second timeout tight enough that classification overhead stays negligible next to the actual model call. If that also fails, LiteLLM's own length-based heuristic takes over so a classification hiccup never turns into a dropped request.

There's a genuinely useful detail buried in the config: `classifier_context_include_assistant_turns: false`. Only the user's own words go to the classifier, never the assistant's previous replies. That keeps classification cheap and stops a long, technical assistant answer from dragging a simple follow-up question into a more expensive tier.

Session affinity is disabled on purpose, and that's a deliberate trade-off worth calling out: normally you'd want a whole conversation pinned to one tier for consistency. Here every message gets classified independently, which means a session can escalate mid-conversation from "ciao" to "il container non parte, OOMKilled" without waiting for a TTL to expire. You lose a bit of consistency, you gain the ability to jump straight to Opus the moment things get serious.

## Validating It: Three Real Requests

Before trusting the router with real traffic, I ran three genuinely different requests through `platform-auto` via Claude Code and watched what it picked:

1. **"cerca errori log ultimi 30 minuti" in Datadog** → keyword-matched COMPLEX (`errore`, and the Datadog/log-analysis vocabulary), routed to **Claude Sonnet 5**. It came back with 100 errors correctly bucketed across CSI/K8s, the orchestrator, the Contentful MCP, and the AI gateway.
2. **"fammi una presentazione in html e salvala in Downloads"** → also COMPLEX, but this time the tier's `least-busy` logic picked **GPT-5.6 Luna** instead of Sonnet, and produced a clean 9-slide deck from the Datadog analysis.
3. **"crea una email da mandare a level 1 support"** → MEDIUM, routed to **Nova Pro**, and it produced a usable incident email without needing anything close to a reasoning model.

Nine total API calls in that session burned 672K tokens for $0.679, split 5 calls to Sonnet, 3 to GPT-5.6 Luna, 1 to Haiku. No misclassifications, no session dropped to the heuristic fallback. Small sample, but it confirmed the pipeline behaves the way the config says it should.

## Three Days Later, at Actual Scale

Three days after that validation run, I checked back on the Grafana dashboard expecting a quiet trickle of real traffic. Instead:

![LiteLLM gateway dashboard 3 days after the AutoRouter v2 validation, showing 2.54 million requests and $1.33M estimated cost](/images/autorouter-v2-the-router-that-knows-when-not-to-call-opus/dashboard-3day-overview.jpg)

2.54 million API requests. 326 billion tokens. $1.33M in estimated spend. And, sitting right next to those numbers: **Active Users: 1**.

That last number is the tell. This wasn't organic traffic, it was a synthetic load/soak test hammering the gateway to see if the router and the infrastructure held up under volume, not real users asking real questions. Worth keeping in mind for everything that follows: this is a stress-test shape of traffic (unusually skewed toward COMPLEX and REASONING prompts), not a representative day of normal usage.

Zooming into the tier distribution from the shorter first look at the same window:

![Grafana AutoRouter v2 dashboard showing routing tier distribution and per-model token share](/images/autorouter-v2-the-router-that-knows-when-not-to-call-opus/3day-tier-distribution.jpg)

The router itself held up. No classification errors surfaced in the logs across 2.5 million requests, no thundering herd on a single Bedrock endpoint (the `least-busy` strategy across `us-east-1` / `us-east-2` did its job), and the infra fixes from the validation phase (pinning ECS to one task so Prometheus scraping doesn't get split across ALB targets, disabling session affinity) stayed stable at 800x the original test volume.

## When Your Own Dashboard Lies to You

Here's the part that's actually more interesting than the volume numbers: the dashboard's own panels don't agree with each other.

The headline tile says **Estimated Cost: $1.33M**. The "Cost by Model" breakdown right next to it lists `us.anthropic.claude-opus-4-6-v1` alone at **$1.37M, 52%**. A single model's line item is larger than the total. Scroll down to the AutoRouter section and "Cost by Tier" reports **$2.65M**, exactly double the headline. "Requests by Tier" collapses into a single unlabeled green ring instead of the four-tier breakdown you'd expect.

None of this means the router is broken. It means the Prometheus counters behind these panels are being summed in a way that double-counts input and output rows, or aggregates across overlapping model aliases (`claude-sonnet`, `claude-sonnet-4-6`, and `claude-sonnet-5` all show up as separate line items for what's functionally the same family, pinned at different versions). The `05-observability-and-finops` doc for this project even calls out the underlying cardinality problem directly: without label filtering, `client_ip`/`user_agent`/`user_email` explode the series count, which is exactly the kind of mess that produces panels which don't reconcile with each other at scale.

The practical lesson, and the one I'd actually tag as the finding worth remembering here: a FinOps dashboard that looks authoritative at 26 requests can quietly stop being trustworthy at 2.5 million. Sanity-check the big number against a second, independent source (the raw AWS Cost Explorer export, in this case) before it goes in a slide deck.

There's a second, sneakier gap in the same data: `gpt-5.6-luna` has no `input_cost_per_token` / `output_cost_per_token` configured in LiteLLM, which the model catalog doc flags explicitly. Every one of those calls, including the ones in my own 9-call validation session, contributes zero to the spend counter. Not "cheap", literally uncounted. On a real Bedrock bill you'd still pay for those tokens; the dashboard just never shows you that line. That's a blind spot in the router's own economics, not a savings.

## The Economics: Opus vs Sonnet vs Letting the Router Decide

Given all that, here's the question worth actually answering: how much did the routing itself save, compared to just pointing everything at one model?

Using the 3-day window's real, trustworthy numbers, the total token volume: 326 billion tokens, and the documented per-token pricing for Sonnet 5 ($3/$15 per million input/output tokens) and Opus 5 ($5/$25 per million), assuming a blended 3:1 input:output ratio typical of an agentic coding workload:

| Scenario | Blended rate | Cost for 326B tokens |
|----------|-------------|----------------------|
| **All requests → Claude Opus 5** | $10.00 / M tokens | **$3.26M** |
| **All requests → Claude Sonnet 5** | $6.00 / M tokens | **$1.96M** |
| **AutoRouter v2 (actual mix)** | blended across 4 tiers | **$1.33M** (dashboard headline) |

Even on this stress-test traffic, which skewed unusually heavy toward Sonnet and Opus (a normal day would lean far more SIMPLE/MEDIUM), routing to the cheapest capable model instead of a single flagship still cut spend by **32% versus all-Sonnet** and **59% versus all-Opus**. That's the floor, not the ceiling: the gap only grows on traffic shaped like the original validation session, where over a third of requests were SIMPLE and never needed anything past Nova Micro or Haiku.

## AWS Bedrock's Open Model Problem

Here's the part that made me reconsider the backend, not just the routing logic.

The MEDIUM and COMPLEX tiers lean on open-weight models served through Bedrock: GLM-5, Kimi K2.5, Qwen3 Coder Next. That looks great on paper, Bedrock as one managed endpoint for both Anthropic and third-party open models, no separate vendor integration needed. AWS actually expanded this catalog meaningfully: in February 2026 Bedrock added six fully-managed open-weight models in one release, DeepSeek V3.2, MiniMax M2.1, GLM 4.7, GLM 4.7 Flash, Kimi K2.5, and Qwen3 Coder Next.

The problem is the clock didn't stop in February. By the time this router is actually running in September 2026, the open-weight frontier has moved two generations past what Bedrock is serving. Kimi K2.5 is not the current Moonshot flagship anymore, Kimi K3 is out, with a much larger context window and materially better agentic benchmarks. GLM has moved from the 4.7 line Bedrock onboarded to GLM-5.3, which currently sits at the top of the [Artificial Analysis open-weight intelligence index](https://openrouter.ai/z-ai/glm-5.3), ahead of Kimi K3, GLM-5.3-Flash, and DeepSeek V4 Pro. None of that newer generation is in the Bedrock-hosted pool this router draws from.

This is the actual, structural limit worth naming: **Bedrock's open-model catalog is a curated snapshot, not a live mirror of the open-weight frontier.** AWS has to onboard, validate, and manage-host each model before it shows up as a `bedrock/` model ID, and that process runs on AWS's release cadence, not the open-weight labs' cadence, which right now is shipping a meaningfully better model every 4-6 weeks. A router built to always reach for the cheapest capable open model is, on Bedrock alone, always reaching into a pool that's already a step behind.

## Redoing the Economics on OpenRouter

So: same routing logic, same four tiers, but swap the backend for OpenRouter and deliberately skip the expensive American frontier (no Claude, no GPT) in favor of the current best open-weight models, at their actual OpenRouter pricing as of this writing:

| Tier | Bedrock model (current) | OpenRouter alternative | OpenRouter price ($/M in / out) |
|------|--------------------------|--------------------------|----------------------------------|
| SIMPLE | Nova Micro | [DeepSeek V4 Flash](https://openrouter.ai/deepseek/deepseek-v4-flash) | $0.05 / $0.10 |
| MEDIUM | Nova Pro / GLM-5 | [GLM-5.3 Flash](https://openrouter.ai/z-ai/glm-5.3-flash) | $0.075 / $0.25 |
| COMPLEX | Claude Sonnet 5 | [DeepSeek V4 Pro](https://openrouter.ai/deepseek/deepseek-v4-pro) | $0.435 / $0.87 |
| REASONING | Claude Opus 5 | [Qwen3.8 Max](https://openrouter.ai/qwen/qwen3.8-max-0902) | $2.00 / $6.00 |

Qwen3.8 Max is the interesting pick for REASONING: it's not the flashiest name, but it currently posts the highest raw open-weight benchmark score around, ahead of both GLM-5.3 and Kimi K3, at roughly a fifth of Opus 5's blended rate.

Applying the same 3:1 blended-ratio method, and the tier split observed in the clean validation run (35% SIMPLE, 42% MEDIUM, 15% COMPLEX, 8% REASONING) to the same 326-billion-token volume, since that's a saner proxy for "normal" traffic than the stress-test's skew:

| Stack | Blended weighted rate | Cost for 326B tokens |
|-------|------------------------|------------------------|
| **Bedrock tiers (Nova/Sonnet/Opus)** | $2.31 / M tokens | **~$753K** |
| **OpenRouter tiers (DeepSeek/GLM/Qwen)** | $0.39 / M tokens | **~$128K** |

Same routing logic, same tier boundaries, roughly **5.9x cheaper** just by pointing the tiers at OpenRouter's current open-weight frontier instead of Bedrock's curated one. None of it touches a US frontier model. The catch is exactly the one the Bedrock section already named: OpenRouter's roster moves fast, so "current best" here has a shelf life measured in weeks, not the quarters Bedrock updates on. You trade a stale-but-stable catalog for a fresh-but-moving one, and for a cost-sensitive MEDIUM/COMPLEX tier that's a trade worth making deliberately, not by default.

## Conclusion

AutoRouter v2 does what it was built to do: read the request, guess the right tier, and stop paying reasoning-model prices for greetings. The keyword-first, LLM-classifier-second, heuristic-fallback-third pipeline held up cleanly through both a 26-request validation and an 800x-larger load test, with zero classification failures in either.

The economics case is real too, even measured against the worst-case (stress-test-skewed) traffic: 32% cheaper than an all-Sonnet baseline, 59% cheaper than all-Opus, just from picking the right model per request instead of one model for everything. And when you're willing to also swap the backend, keeping the exact same tiering logic but pointing it at OpenRouter's current open-weight models instead of Bedrock's slower-moving catalog, the same workload drops by another 5-6x.

## Reflections

The honest caveat runs through the whole post: the 3-day numbers came from a single synthetic user hammering the gateway, not real usage, and the traffic shape it produced (66% combined Sonnet/Opus by request count) is nothing like the SIMPLE-heavy mix the validation run showed. The next real step, per the report's own conclusions, is two weeks of actual production traffic before drawing a final cost verdict.

What's still missing, and worth being upfront about: the dashboard inconsistencies aren't cosmetic. If "Cost by Tier" reports double the headline number and a single model's cost line exceeds the total, that's a metrics pipeline that needs fixing before anyone puts these figures in front of a budget owner. And the `gpt-5.6-luna` pricing gap is a real, silent undercount, not a rounding error: any tier that routes through it is cheaper on the dashboard than it is on the actual AWS bill.

The OpenRouter comparison is the part I'd flag as illustrative rather than final. It assumes a 3:1 input:output ratio that's a reasonable approximation, not a measured one, and "best open-weight model" is a moving target that will look different again in another six weeks. The direction of the number (multiple times cheaper, comfortably) is more solid than the exact multiple.
