---
title: "AutoRouter v2: The Router That Knows When Not to Call Opus"
date: 2026-09-19
draft: true
description: A complexity-based router picks the cheapest model that can still
  do the job, and skips Opus for "ciao". Here's the validation, a 3-day
  load test at real scale, and what the same routing logic would cost on
  OpenRouter instead of Bedrock.
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
- The Economics: Sonnet-Only vs Letting the Router Decide
- AWS Bedrock's Open Model Problem
- Redoing the Economics on OpenRouter
- Conclusion
- Reflections



Here we are. Every LLM gateway I've built so far had the same lazy default: pick one strong model, point everything at it, call it a day. It works, right up until the invoice arrives and you realize half the traffic was "ciao", "riassumi questo", or "scrivi una email a level 1 support", and every single one of those trivial requests paid Opus-level prices.

AutoRouter v2 is the fix: a complexity classifier sitting in front of the LiteLLM gateway that reads the request, decides how hard the task actually is, and picks the cheapest model in the pool that can still do it. The user never picks a model. They just call `platform-auto` and the router does the rest.

This post walks through how it's built, what happened when I validated it on real traffic, what a 3-day load test at real scale looked like, and then answers the question I kept getting asked: what would this actually have cost on plain Opus, on plain Sonnet, and if I dropped AWS Bedrock entirely and pointed the same tiers at OpenRouter instead.

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

Three days after that validation run, I checked back on the Grafana dashboard expecting to see roughly what I'd sent it. Instead the volume tiles were in the millions, requests, tokens, dollars.

![LiteLLM gateway dashboard 3 days after the AutoRouter v2 validation](/images/autorouter-v2-the-router-that-knows-when-not-to-call-opus/dashboard-3day-overview.jpg)

Turns out that dashboard aggregates the entire shared Stargate gateway, every team and service that calls it, not just my own traffic. Cross-checking the headline tiles against the underlying per-minute rate graphs in the same dashboard, they don't even reconcile with each other at that aggregate level, so I didn't trust the totals either way. What I actually wanted was my own usage, filtered by my own `hashed_api_key`:

![AutoRouter v2 tier distribution filtered to my own API key: 108 requests, 0 failures, tokens by model](/images/autorouter-v2-the-router-that-knows-when-not-to-call-opus/tier-distribution-by-api-key.jpg)

That's a number I can actually reason about: **108 requests**, **0 failed**, spread across all four tiers, with real per-model input/output token counts LiteLLM logged directly. No classification errors, no thundering herd on a single Bedrock endpoint (the `least-busy` strategy across `us-east-1` / `us-east-2` did its job), and the infra fixes from the validation phase (pinning ECS to one task so Prometheus scraping doesn't get split across ALB targets, disabling session affinity) held up. This is the dataset the rest of the economics in this post is built on.

## The Economics: Sonnet-Only vs Letting the Router Decide

Here's the question worth actually answering: how much did the routing itself save, compared to just pointing everything at one model? Using the real per-model token counts from my own 108 requests over that 3-day window, no assumed input:output ratio needed this time, LiteLLM logs the two separately:

| Model | Input tokens | Output tokens |
|-------|--------------:|----------------:|
| Claude Sonnet 5 | 4.64M | 18.5K |
| GPT-5.6 Luna | 1.53M | 27.2K |
| Claude Opus 5 | 988K | 2.74K |
| Claude Haiku 4.5 | 901K | 15.6K |
| GLM-5 | 285K | 961 |
| Kimi K2.5 | 101K | 417 |

That's roughly 8.51M tokens across 108 requests, all four tiers actually exercised, not just the three prompts from the validation run. Pricing each model at its documented per-token rate, LiteLLM's own spend tracker puts the real, routed cost at **$20.42** (GPT-5.6 Luna has no cost configured in this setup, so its 1.53M input tokens don't add to that figure, meaning $20.42 is a slight undercount, not an overstatement).

The counterfactual: what would that exact same 8.51M tokens have cost pinned to Claude Sonnet 5 the whole way, at its documented $3/$15 per million input/output tokens?

| Scenario | Cost for the same ~8.51M tokens |
|----------|-----------------------------------|
| **All requests → Claude Sonnet 5** | **$26.32** |
| **AutoRouter v2 (actual routed mix)** | **$20.42** (real, logged; likely undercounted) |

About **22% cheaper**, on real numbers, even though this particular sample skews toward COMPLEX and REASONING work (Sonnet alone is 4.64M of the 8.51M input tokens, this is genuinely demanding dev work, not a SIMPLE-heavy inbox). On a mix with more SIMPLE/MEDIUM traffic, like the three-prompt validation run, the gap would be wider.

## AWS Bedrock's Open Model Problem

Here's the part that made me reconsider the backend, not just the routing logic.

The MEDIUM and COMPLEX tiers lean on open-weight models served through Bedrock: GLM-5, Kimi K2.5, Qwen3 Coder Next. That looks great on paper, Bedrock as one managed endpoint for both Anthropic and third-party open models, no separate vendor integration needed. AWS actually expanded this catalog meaningfully: in February 2026 Bedrock added six fully-managed open-weight models in one release, DeepSeek V3.2, MiniMax M2.1, GLM 4.7, GLM 4.7 Flash, Kimi K2.5, and Qwen3 Coder Next.

The problem is the clock didn't stop in February. By the time this router is actually running in September 2026, the open-weight frontier has moved two generations past what Bedrock is serving. Kimi K2.5 is not the current Moonshot flagship anymore, Kimi K3 is out, with a much larger context window and materially better agentic benchmarks. GLM has moved from the 4.7 line Bedrock onboarded to GLM-5.3, which currently sits at the top of the [Artificial Analysis open-weight intelligence index](https://openrouter.ai/z-ai/glm-5.3), ahead of Kimi K3, GLM-5.3-Flash, and DeepSeek V4 Pro. None of that newer generation is in the Bedrock-hosted pool this router draws from.

This is the actual, structural limit worth naming: **Bedrock's open-model catalog is a curated snapshot, not a live mirror of the open-weight frontier.** AWS has to onboard, validate, and manage-host each model before it shows up as a `bedrock/` model ID, and that process runs on AWS's release cadence, not the open-weight labs' cadence, which right now is shipping a meaningfully better model every 4-6 weeks. A router built to always reach for the cheapest capable open model is, on Bedrock alone, always reaching into a pool that's already a step behind.

## Redoing the Economics on OpenRouter

Same 8.51M real tokens, same per-model split, but this time: what if each of those six models had been a newer OpenRouter open-weight model instead of its Bedrock counterpart, still skipping the expensive American frontier entirely?

| Real model (real tokens, in/out) | Newer OpenRouter equivalent | Price ($/M in / out) | Cost |
|--------------------------------------|--------------------------------|-------------------------|------:|
| Sonnet 5 (4.64M / 18.5K) | [DeepSeek V4 Pro](https://openrouter.ai/deepseek/deepseek-v4-pro) | $0.435 / $0.87 | $2.03 |
| GPT-5.6 Luna (1.53M / 27.2K) | [DeepSeek V4 Pro](https://openrouter.ai/deepseek/deepseek-v4-pro) | $0.435 / $0.87 | $0.69 |
| Opus 5 (988K / 2.74K) | [Qwen3.8 Max](https://openrouter.ai/qwen/qwen3.8-max-0902) | $2.00 / $6.00 | $1.99 |
| Haiku 4.5 (901K / 15.6K) | [DeepSeek V4 Flash](https://openrouter.ai/deepseek/deepseek-v4-flash) | $0.05 / $0.10 | $0.05 |
| GLM-5 (285K / 961) | [GLM-5.3](https://openrouter.ai/z-ai/glm-5.3) | $0.90 / $3.00 | $0.26 |
| Kimi K2.5 (101K / 417) | [Kimi K3](https://openrouter.ai/moonshotai/kimi-k3) | $1.95 / $10.92 | $0.20 |

DeepSeek V4 Pro is the relevant pick for the Sonnet and Luna slots: [80.6% on SWE-bench Verified](https://openrouter.ai/deepseek/deepseek-v4-pro), the strongest published open-weight result on that benchmark, at a fraction of Sonnet 5's rate. Qwen3.8 Max takes the Opus slot for the same reason it came up earlier: highest raw open-weight benchmark score around right now, at a fifth of Opus's rate. Kimi K3 replaces K2.5 simply because it's the current model, K2.5 already isn't Moonshot's frontier anymore.

Add it up, and three references side by side on the exact same 8.51M real tokens:

| Scenario | Cost |
|----------|------:|
| **All requests → Claude Sonnet 5** | **$26.32** |
| **AutoRouter v2 on Bedrock (real mix)** | **$20.42** |
| **Same real mix, on newer OpenRouter models** | **~$5.22** |

Same tokens, same real per-model split, just a newer backend for the exact same routing decisions: roughly **4x cheaper than the Bedrock mix that actually ran**, and **5x cheaper than routing everything to Sonnet**. None of it touches a US frontier model. The catch is the one the Bedrock section above already named: OpenRouter's roster moves fast, so "current best" here has a shelf life measured in weeks, not the quarters Bedrock updates on. You trade a stale-but-stable catalog for a fresh-but-moving one.

## Conclusion

AutoRouter v2 does what it was built to do: read the request, guess the right tier, and stop paying reasoning-model prices for greetings. The keyword-first, LLM-classifier-second, heuristic-fallback-third pipeline held up cleanly through both the 3-prompt validation and 108 real requests spread across all four tiers over the following 3 days, with zero classification errors and zero failed requests.

The economics case is real too, on real per-model token counts logged by LiteLLM for those 108 requests, not an estimate: 8.51M tokens, actually routed cost $20.42, versus $26.32 had every one of those tokens gone to Claude Sonnet 5 instead, 22% cheaper even on a sample this skewed toward COMPLEX and REASONING work. And if you're willing to also swap the backend, repricing that exact same real mix on newer OpenRouter open-weight models instead of Bedrock drops it to roughly $5.22, another 4x, and 5x cheaper than Sonnet-only overall.

## Reflections

The honest caveat: 108 requests is still a personal sample, not fleet-wide production traffic, and it happens to skew toward the kind of demanding dev work I was actually doing that week, more COMPLEX/REASONING than a typical mixed workload would be. The $20.42 figure is also a slight undercount, since GPT-5.6 Luna's tokens aren't priced in this LiteLLM config at all, so the real gap between the router and Sonnet-only is probably a bit smaller than 22%, not bigger. Both caveats work against the router's case, not for it, which is the direction I'd rather be wrong in. The actual next step, per the report's own conclusions, is two weeks of real production traffic across the whole team, logged properly against the AWS bill, before staking a budget conversation on any of this.

The OpenRouter comparison is the part I'd flag as illustrative rather than final on top of that. It assumes a 3:1 input:output ratio, a reasonable approximation rather than a measured one, and "best open-weight model" is a moving target that will look different again in another six weeks. The direction of the number (several times cheaper, comfortably) is more solid than the exact multiple.
