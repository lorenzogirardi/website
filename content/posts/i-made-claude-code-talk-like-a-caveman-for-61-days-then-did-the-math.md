---
title: I Made Claude Code Talk Like a Caveman for 61 Days, Then Did the Math
date: 2026-07-19
draft: false
description: I mined 27 Claude Code session logs to measure what caveman-mode
  token compression actually saves in API dollars, and found the real cost was
  hiding somewhere caveman mode can't touch.
tags:
  - ai
  - claude code
  - llm
  - cost saving
  - automation
  - observability
  - performance
featuredImage: /images/i-made-claude-code-talk-like-a-caveman-for-61-days-then-did-the-math/caveman-hero.png
---
### Table of Contents

  * Introduction
  * The Problem: Verbose Agents Cost Real Money
  * Enter Caveman Mode
  * The Question I Actually Wanted Answered
  * Method: Mining 27 Session Logs
  * The Results, Per Model
  * Where the Money Actually Goes
  * How Big Were These Sessions, Really?
  * The Fable 5 Shift
  * From a Homelab Toy to an Enterprise Line Item
  * Reflections
  * Conclusion



Strange... an AI assistant that talks less should cost less. Obvious, right? I wanted a number, not a vibe.

## Introduction

For the last two months I've been running Claude Code with a plugin that makes it respond like a caveman. Dropped articles, no filler, no "I'd be happy to help you with that", technical substance intact. Fragments instead of sentences. It sounded ridiculous the first time I saw it in my terminal, and I kept using it anyway because the responses got out of the way faster.

The plugin ships with a claimed benchmark: about 65% fewer output tokens per task, measured against a benchmark suite. That's a nice number on a README. It is not the same as knowing what it did to *my* actual usage, across *my* actual projects, over two months of real work.

So I stopped guessing and pulled the logs. Every Claude Code session writes a JSONL transcript to `~/.claude/projects`, and every assistant turn in that transcript carries a `usage` block: input tokens, output tokens, cache read, cache write, per model. That's not an estimate. That's exactly what got billed.

In this article I'll walk through what 27 sessions and 6,887 assistant turns actually cost, in API-equivalent dollars, with and without the caveman compression, and why the answer surprised me in a direction the marketing never mentions.

## The Problem: Verbose Agents Cost Real Money

Here's the thing nobody tells you when you start driving an agentic coding tool for hours a day: the model doesn't just answer your question. It narrates. "Let me check that file." "Now I'll run the tests." "Great, that worked!" None of that ceremony changes what gets done. All of it gets billed as output tokens.

Multiply that by an agent that reasons in a loop, calls tools, and reports back after each one, and the narration compounds. A session that runs 500 turns isn't paying for 500 answers, it's paying for 500 answers plus 500 rounds of "here's what I'm about to do" and "here's what I just did."

Caveman mode strips exactly that. Not the technical content, the surrounding prose. What made me curious was whether the 65% number held up outside a benchmark suite, on messy real sessions that ranged from a two-minute typo fix to a multi-day refactor.

## Enter Caveman Mode

The mechanism is a system prompt injected at session start plus a per-message reminder: drop articles, drop hedging, drop pleasantries, keep code blocks and error messages verbatim, write commits and security warnings in normal prose. It has levels (`lite`, `full`, `ultra`) and it drops out of character automatically for anything that needs unambiguous phrasing: security confirmations, multi-step sequences, irreversible actions.

I've been running it at `full` the whole time. The style is genuinely jarring to read at first ("Bug in auth middleware. Fix:") and then it isn't, because the technical content is exactly as complete as before. What's gone is the throat-clearing.

## The Question I Actually Wanted Answered

Not "does it feel faster." Three concrete numbers:

1. How many output tokens did caveman mode actually save, across real sessions, not a benchmark?
2. What does that translate to in API-equivalent dollars, per model?
3. Is output token cost even where the money is going, or was I looking at the wrong line item?

## Method: Mining 27 Session Logs

Every session lives as a `.jsonl` file, one JSON object per line, one line per event. I walked `~/.claude/projects` recursively, parsed every `assistant`-type line, and summed the `usage` fields per model:

```javascript
for (const line of raw.split("\n")) {
  const e = JSON.parse(line);
  if (e.type !== "assistant" || !e.message?.usage) continue;
  const u = e.message.usage, m = e.message.model;
  M[m].out += u.output_tokens || 0;
  M[m].cr  += u.cache_read_input_tokens || 0;
  M[m].cw  += u.cache_creation_input_tokens || 0;
  M[m].in  += u.input_tokens || 0;
}
```

Attribution matters here: a session can switch models mid-way (I switched to a new model release partway through this very analysis), so token counts get summed per model *per assistant turn*, not per session. Get that wrong and your per-model cost table quietly lies.

Result: 27 sessions, spanning 20 May to 19 July, across 15 projects (names anonymized below, my clients don't need to be in a blog post). 6,887 assistant turns. 5.52M output tokens. 954M cache read tokens, which is a number I'll come back to.

For the counterfactual, I applied the plugin's own published ratio: `output ÷ (1 - 0.65)` to get what the same output would have cost uncompressed. It's the only benchmark I had, and it's explicitly conservative for reasons I'll get to in Reflections.

## The Results, Per Model

Four models show up across the two months: Fable 5, Opus 4.8, Sonnet 4.6, and one Haiku 4.5 subagent call. Pricing is current API list price per model, input/output per million tokens, cache read at 0.1x input, cache write at 1.25x input.

| Model | Output (real) | Output cost | Output (no caveman) | Output cost | Δ |
|---|---|---|---|---|---|
| Fable 5 ($10/$50) | 2.47M | $123.71 | 7.07M | $353.44 | −$229.73 |
| Opus 4.8 ($5/$25) | 1.44M | $35.97 | 4.11M | $102.76 | −$66.79 |
| Sonnet 4.6 ($3/$15) | 1.61M | $24.16 | 4.60M | $69.03 | −$44.87 |
| Haiku 4.5 ($1/$5) | 1.3k | $0.01 | 3.8k | $0.02 | −$0.01 |
| **Total** | **5.52M** | **$183.84** | **15.79M** | **$525.26** | **−$341.42** |

That last column is the number I was after: roughly $341 in output-token cost avoided across two months, on output alone. Fable 5 accounts for the largest share, which makes sense since it's also where I did the heaviest, longest sessions in July.

But that's only the output column. Look at the full picture including everything else that gets billed:

| | Actual | Without caveman | Δ |
|---|---|---|---|
| **Total cost** | $1,036.28 | $1,377.70 | −$341.42 |

Same delta, obviously, since caveman mode only touches output. Which is exactly the point that took me by surprise.

## Where the Money Actually Goes

I broke the $1,036.28 down by category instead of by model:

- **Cache read**: $575.11 (55.5%), 954M tokens
- **Cache write**: $273.12 (26.4%), 32.9M tokens
- **Output**: $183.84 (17.7%), 5.52M tokens
- **Input (uncached)**: $4.21 (0.4%), 532k tokens

Caveman mode saved me $341 on a $183.84 line item. It's real money, and I'd take it every time, but it's compressing 17.7% of the bill. The other 82% is context: reading back conversation history, tool results, and file contents on every single turn, over and over, as the session grows.

That's not a criticism of caveman mode, which does exactly what it claims. It's a reminder that in a long agentic session, the model rereading its own accumulated context dwarfs the model *talking*. If I wanted to meaningfully cut the total bill, the next lever isn't verbosity, it's session length and how aggressively I compact or restart context.

## How Big Were These Sessions, Really?

The distribution is brutally uneven. The top 3 sessions produced 46% of all output tokens. Half the sessions (13 of 27) never crossed 30k output tokens: quick fixes, one-off questions, a script written and forgotten.

One session in particular, let's call it Project A, was a genuine outlier: 1,932 assistant turns, 1.30M output tokens, and 257M cache read tokens, reopened across roughly 8 calendar days. That one session alone accounts for 23% of all output and 27% of all cache read across the entire two months. It's not that the task was 20x harder than the others, it's that a long-lived session compounds: every turn rereads everything that came before, so cache read grows roughly quadratically with turn count if you never compact.

The 8 sessions above 300k output tokens cover 80% of the total output. The 10 smallest sessions combined don't add up to 40k tokens, less than a single heavy turn from the whale session.

## The Fable 5 Shift

There's a visible model migration in the data. May and June ran almost entirely on Sonnet 4.6 and Opus 4.8. From early July onward, nearly every new session opens on Fable 5, which now accounts for 57% of total cost despite covering a shorter window. Not a deliberate experiment, just what happens when a more capable model becomes available and the harder tasks migrate to it first. Worth remembering when reading any "cost went up" headline number in isolation: it's not always more usage, sometimes it's a different, pricier model doing harder work.

## From a Homelab Toy to an Enterprise Line Item

Everything above is a homelab number. I run Claude Code on a subscription, I never see a per-token invoice, and until I pulled these logs I genuinely hadn't asked myself whether caveman mode was worth anything. It made the terminal less chatty, that was reason enough. The dollar figure was a curiosity I ran for a blog post, not a decision I needed to make.

That framing breaks the moment you put the same numbers in front of someone paying per token at team scale, which is worth spelling out plainly since the two audiences read this data very differently.

**The output saving is real and it's the easy 25%.** Caveman mode, or anything that compresses assistant verbosity the same way, moved roughly a quarter of my total bill. At enterprise scale that's not a curiosity, that's a line item a finance team will ask about once agentic tools go from "a few engineers experimenting" to "every engineer, every day, running long agent loops." A 25% cut on output cost, applied across a fleet of agents instead of one person's laptop, is the kind of number that funds its own adoption.

**The other 75%, dominated by cache read, is where I think the industry doesn't have a good model yet.** In my data, rereading accumulated context ran to over half the total cost, and that number only goes up as sessions get longer and agents get more autonomous. Nobody optimizing an agentic workflow today is asked "how much of your spend is the model rereading its own history for the tenth time this session?" as a first-class question, the way "how verbose is the output?" already is. There's no equivalent of caveman mode for context rereading: no widely adopted technique that treats stale tool results and old conversation turns as a cost surface to actively manage, rather than something you either compact manually or let grow.

That's a genuine open point for agentic development, not a rhetorical one. A responsible enterprise deployment of long-running agents needs a model, formal or informal, that accounts for context-rereading cost the same way it already accounts for output verbosity: something a team can measure, budget against, and optimize, instead of discovering after the fact in a log-mining exercise like this one. Right now most of us, myself included until this week, simply aren't measuring it. And an unmeasured 55% of a bill is exactly the kind of thing that turns into an uncomfortable surprise once agentic development stops being a side project and starts being the default way software gets written.

## Reflections

A few things I'm not glossing over.

**The 65% ratio is an assumption, not a measurement.** The session logs don't record which caveman level was active on a given turn. I ran everything at `full` the whole period, per my own settings, so I applied the benchmark uniformly. If I'd mixed modes mid-session, this whole analysis would need per-turn mode tracking that doesn't currently exist.

**This likely understates the real saving.** Less output per turn means shorter conversation history feeding into every subsequent turn. Without caveman mode, not just output but *input and cache tokens* would have grown too, since the model is rereading its own longer replies on every later turn. My $341 figure only accounts for the output line item held constant everywhere else, which is the conservative assumption, not the generous one.

**These are API-equivalent dollars, not an invoice.** I run Claude Code on a subscription plan, not pay-per-token API access. Nothing about this analysis is a real receipt. What it *is* a proxy for: less token pressure against usage limits on a subscription, which in practice is the thing that actually constrains how much I can throw at the tool in a day.

**Cache read is the real story here, and caveman mode doesn't touch it.** If I were optimizing purely for cost, the lever isn't verbosity, it's compaction and session hygiene: closing sessions instead of reopening them across 14 days, using context editing to prune stale tool results, splitting genuinely unrelated work into fresh sessions instead of one marathon thread.

## Conclusion

Two months of real Claude Code usage, mined straight from the session logs: caveman mode saved roughly $341 in API-equivalent output cost, right in line with its published 65% claim, and that's the honest, measurable part of the story. The bigger part of the bill, cache read at 55.5% of total cost, is a different problem entirely, driven by how long I keep sessions open rather than how the model phrases its replies. Compression saved me real money on the smallest line item on the invoice. If I want to move the largest one, I need to change how I manage sessions, not how the model talks in them.

Related reading: [Who Is Your AI Agent Acting For? RFC 8693 On-Behalf-Of Delegation](/posts/who-is-your-ai-agent-acting-for-rfc-8693-on-behalf-of-delegation/) if you're also instrumenting what your agents are actually doing under the hood.
