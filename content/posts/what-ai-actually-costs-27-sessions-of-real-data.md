---
title: "What AI Actually Costs: 27 Sessions of Real Data"
date: 2026-07-26
draft: false
description: "Everyone talks about AI ROI, nobody shows numbers. Here are 27 real coding sessions parsed from the billing logs: where the money goes, why context is the cost, and a framework to measure it."
tags:
  - ai
  - cost saving
  - monitoring
  - automation
  - dora
  - platform engineering
featuredImage: /images/what-ai-actually-costs-27-sessions-of-real-data/featured.png
images:
  - "/images/what-ai-actually-costs-27-sessions-of-real-data/featured.png"
---
### Table of Contents

  * Introduction
  * The Question Nobody Answers
  * The Dataset: 27 Sessions, 61 Days
  * Where the Money Actually Goes
    * Cache Read Is the Bill
    * The Whale Session
    * Head and Tail
  * The Model Mix Problem
  * Output Compression: The Lever Everyone Reaches For First
  * The Velocity Half of the Question
    * km/l Is Not the Metric. km/€ Is.
    * The Efficiency Gap Between People
  * A Framework for Getting the Data
    * Step 1: Parse Your Own Logs
    * Step 2: The Ticket Protocol
    * Step 3: The Metrics That Matter
  * From Measurement to a Development Framework
    * Rule 1: One Ticket, One Session
    * Rule 2: Route the Model to the Task
    * Rule 3: Compress the Output
    * Rule 4: Build Skills, Not Prompts
  * The Maturity Path
  * The Honest Caveats
  * Conclusion
  * Reflections



Here we are. Every conversation about AI in engineering right now runs on vibes. It's faster. It's cheaper. It's transformative. Ask for a number and the room goes quiet.

I got tired of that. So I parsed my own billing logs: 27 real coding sessions, 61 days, 15 projects, every assistant turn with its actual token accounting. Not an estimate, not a vendor benchmark, the numbers the API itself reports.

What came out was uncomfortable, in a useful way. The cost of AI-assisted development has almost nothing to do with what you thought. It isn't the code it writes. It's the context it re-reads to write it.

In this article I'll walk through the raw data, explain where the money actually goes, and then hand over the framework I built to collect this properly across a team: a parser, a ticket protocol, and a set of metrics that turn "AI is helping, I think" into a number you can defend in a budget meeting.

## The Question Nobody Answers

There are three questions that everybody skips straight past.

**What does it cost?** Not the subscription price. The real per-unit-of-work cost. If a ticket takes an hour of AI-assisted work, what did that hour consume?

**How much faster is it?** Faster than what baseline? Measured how? "It feels faster" is the single most common answer, and it's worthless the moment finance asks.

**How is it actually used?** This is the one that matters most and gets asked least. Where does the spend land: on the tokens you send, the tokens it generates, or the enormous invisible middle where the whole conversation gets re-read on every single turn?

That third question is where I started, because it's the only one you can answer from data you already have sitting on your laptop.

## The Dataset: 27 Sessions, 61 Days

Claude Code writes a JSONL log for every session under `~/.claude/projects`. Each assistant turn carries a `usage` object with the exact token counts the API billed. No sampling, no estimation.

Here's the shape of one turn:

```json
{
  "type": "assistant",
  "timestamp": "2026-07-19T07:47:21.320Z",
  "sessionId": "1c1d73cf-6c69-44ac-91e0-96eede47c658",
  "message": {
    "model": "claude-opus-5",
    "usage": {
      "input_tokens": 2,
      "cache_creation_input_tokens": 15570,
      "cache_read_input_tokens": 12725,
      "output_tokens": 222
    }
  }
}
```

Look at that ratio for a second. Two uncached input tokens. Two hundred and twenty-two output tokens. And **12,725 tokens read back out of cache**, just to produce them. That single turn is the whole article in miniature.

Aggregated across everything:

| Metric | Value |
|---|---|
| Sessions | 27, over 61 days, across 15 projects |
| Assistant turns | 6,887 (median 127 per session) |
| Output tokens | 5.52M (median 53k per session) |
| Cache read tokens | 954M |
| Uncached input tokens | 532k |
| API-equivalent cost | $1,036.28 |

954 million tokens read. 5.5 million written. That's roughly **173 tokens re-read for every single token generated**.

## Where the Money Actually Goes

Applying list prices per model (input, output, cache read at 0.1x input, cache write at 1.25x input) to those counts gives the split that reframes the entire problem:

| Cost driver | Tokens | Cost | Share |
|---|---|---|---|
| **Cache read** | 954M | $575.11 | **55.5%** |
| Cache write | 32.9M | $273.12 | 26.4% |
| Output | 5.52M | $183.84 | 17.7% |
| Uncached input | 532k | $4.21 | 0.4% |

{{< mermaid >}}
pie showData
    title Where the money goes
    "Cache read" : 575.11
    "Cache write" : 273.12
    "Output" : 183.84
    "Uncached input" : 4.21
{{< /mermaid >}}

### Cache Read Is the Bill

**Eighty-two percent of the spend is context handling.** Cache read plus cache write. The tokens the model actually produces, the code, the explanations, the thing you asked for, account for less than a fifth of the cost.

This inverts the intuition almost everyone starts with. People try to save money by asking for shorter answers. That optimises the 17.7% slice while ignoring the 55.5% one.

Why is cache read so dominant? Because a conversation is stateless underneath. Every turn re-sends the entire history: your files, the tool outputs, the previous reasoning, all of it. Prompt caching makes that re-read cheap per token (10% of input price) but it doesn't make it free, and the volume is brutal. A session with 100k tokens of accumulated context that runs 500 turns re-reads 50 million tokens. At Fable 5 pricing that alone is $50, before the model generates a single useful line.

The cost of AI is not what it writes. **It's what it has to remember.**

### The Whale Session

The distribution is where it gets actionable. One session dominates everything:

| | |
|---|---|
| Turns | 1,932 |
| Output | 1,295,770 tokens |
| Cache read | **257M tokens** |
| Span | 8.2 days |

That single session, kept open and reopened over eight calendar days, accounts for **23% of all output and 27% of all cache read across two months of work.**

It wasn't eight days of continuous work. It was a session left open, returned to, resumed, and never closed. Every resume dragged the whole accumulated history back through the cache. The work itself was probably a few days. The cost profile was of something much larger.

Five sessions in the dataset stayed open for more than eight calendar days. That's where cache read explodes.

### Head and Tail

Plot the sessions by size and the shape is textbook power law:

- The **top 3 sessions** produce 46% of all output.
- The **8 sessions above 300k tokens** cover 80% of the output.
- **13 of 27 sessions stay under 30k tokens**: quick fixes, one-off questions, a config lookup.
- The 10 smallest sessions combined don't reach 40k tokens. Less than one heavy turn from the whale.

{{< mermaid >}}
flowchart TD
    A[27 sessions] --> B[8 marathon sessions<br>300k+ output]
    A --> C[6 medium sessions]
    A --> D[13 quick sessions<br>under 30k]
    B --> E[80% of output<br>most of cache read cost]
    C --> F[modest share]
    D --> G[rounding error]
{{< /mermaid >}}

The practical read: **you don't have a cost problem across your AI usage. You have a cost problem in eight sessions.** Optimising the quick ones is theatre. The whole budget lives in the marathons.

## The Model Mix Problem

Same dataset, split by model:

| Model | List price (in/out per Mtok) | Input | Cache read | Cache write | Output | Total |
|---|---|---|---|---|---|---|
| Fable 5 | $10 / $50 | $3.26 | $303.10 | $159.86 | $123.71 | **$589.92** |
| Opus 4.8 | $5 / $25 | $0.85 | $191.84 | $94.93 | $35.97 | $323.58 |
| Sonnet 4.6 | $3 / $15 | $0.10 | $80.16 | $18.23 | $24.16 | $122.66 |
| Haiku 4.5 | $1 / $5 | $0.00 | $0.01 | $0.10 | $0.01 | $0.12 |
| **Total** | | **$4.21** | **$575.11** | **$273.12** | **$183.84** | **$1,036.28** |

Fable 5 alone is 57% of total cost. Not because it ran the most turns, but because premium pricing multiplies through the cache read volume. On a top-tier model, a long context isn't just expensive to generate against, it's expensive to *carry*.

Here's the sting: the model price multiplier applies to that 82% context slice too. Running a documentation lookup on a frontier model doesn't cost you a bit more, it costs you 3x to 10x more on every re-read of a context that's growing anyway.

Strange, that nobody routes? Because there's no feedback loop. You never see the invoice per task, so the incentive to pick the cheap model for the cheap task simply doesn't exist. You pick the best model available, always, because why wouldn't you.

## Output Compression: The Lever Everyone Reaches For First

I run a compression mode on my sessions (terse responses, dropped filler, technical substance preserved). Benchmarked at roughly 65% fewer output tokens for the same task.

The dataset lets me estimate the counterfactual: what the same 27 sessions would have cost without it.

| | With compression | Without (estimated) | Delta |
|---|---|---|---|
| Output tokens | 5.52M | 15.78M | 10.26M saved |
| Total cost | $1,036.28 | $1,377.70 | **−$341.42** |

$341 saved, 25% of total spend. That's real. It's also, and this is the important part, **the third-biggest lever, not the first.** It only touches the 17.7% output slice (plus a second-order effect: less output means less history feeding subsequent turns, so input and cache would have grown too, which makes this estimate conservative).

The ordering of levers by actual impact:

| Lever | Impact | Effort | Status |
|---|---|---|---|
| **Session discipline** | ~40% | Zero tooling. Pure habit. | Mostly untapped |
| **Model routing** | ~20% | Low. A decision rule. | Mostly untapped |
| Output compression | ~25% | Already automated | Captured |

Session discipline is free, has the highest impact, and requires nothing except closing a session when a ticket is done. One ticket, one fresh session. That single habit addresses the whale problem, the marathon problem, and most of the cache read bill at once.

## The Velocity Half of the Question

Cost is only half. The other half is what you got for it, and this is where most teams have even less data.

### km/l Is Not the Metric. km/€ Is.

Fuel efficiency ignores fuel price. A diesel and a petrol car doing the same km/l are not comparable when the fuel costs differ. You optimise consumption and miss value entirely.

Classical delivery metrics have the same blind spot. Story points per sprint, deploy frequency, cycle time: all of them measure output volume while treating AI spend as a footnote somewhere in a SaaS budget line.

The shift is to make AI cost the **denominator**:

| Old lens | New lens |
|---|---|
| Story points / sprint | PRs merged (or story points) / AI $ |
| Deploy frequency | Deploys per day / AI daily spend |
| Cycle time | (baseline cycle time − AI cycle time) / AI spend |

That normalisation is what lets you compare two teams honestly. Same raw speed, one spending 4x more to get there, is not the same result. Without cost in the denominator, that difference is invisible.

Instrumented on a real platform (GitHub, ArgoCD, GitHub Actions, and LiteLLM cost ingest into Datadog), the baseline signals look like this:

| DORA signal | Value | Trend |
|---|---|---|
| Deploy frequency | 4.2 / day | ↑ 18% |
| PR cycle time | 6.1 hrs | ↓ 22% with AI assist |
| Change failure rate | 3.4% | stable |
| AI-assisted PR share | 38% | growing |

Cycle time down 22% on AI-assisted PRs, with change failure rate holding flat, is the actual speed claim. Not "it feels faster". A measured delta, with a quality guardrail proving you didn't buy speed with defects.

### The Efficiency Gap Between People

Cluster individual users on tokens, cost, cycle time, and PR velocity and three tiers fall out cleanly:

| Tier | Profile | Action |
|---|---|---|
| **A, Efficient** | High token volume, low cost (Haiku/Sonnet mix), high PR velocity | Role model. Go ask them how. |
| **B, Balanced** | Moderate usage, sensible model choice, average outcomes | Optimisable with small nudges |
| **C, Costly** | Low output, always on the frontier model | Needs coaching, model routing first |

**Cluster C spends 3x more per delivered story point than Cluster A**, with cost per PR running 4x to 6x higher. Same tooling, same access, same company. The difference is entirely behavioural: session hygiene and model selection.

That's the finding that justifies the whole exercise. Model governance alone, just routing tasks to appropriate models, could recover roughly 40% of the budget. And it is completely invisible until you normalise cost per unit of delivered work.

Note what this is *not*: a stack rank for performance reviews. Cluster C isn't lazy, they're uninstrumented. Nobody ever showed them the cost of a choice they had no way to see.

## A Framework for Getting the Data

None of the above requires a platform team, a vendor, or a procurement cycle. Here's the sequence, cheapest first.

### Step 1: Parse Your Own Logs

Start here because it costs nothing and you already have the data. This script walks `~/.claude/projects`, aggregates every assistant turn by session, applies list prices, and prints the table:

```python
#!/usr/bin/env python3
"""Parse Claude Code JSONL session logs into a per-session cost table."""
import json, sys, glob, os
from collections import defaultdict

# list price per million tokens: (input, output)
PRICES = {
    "opus":   (5.00, 25.00),
    "fable":  (10.00, 50.00),
    "sonnet": (3.00, 15.00),
    "haiku":  (1.00, 5.00),
}
CACHE_READ_MULT = 0.10   # cache read billed at 10% of input price
CACHE_WRITE_MULT = 1.25  # cache write billed at 125% of input price


def family(model):
    for name in PRICES:
        if name in (model or ""):
            return name
    return "sonnet"


def main(root=os.path.expanduser("~/.claude/projects")):
    sessions = defaultdict(lambda: {
        "turns": 0, "inp": 0, "cr": 0, "cw": 0, "out": 0,
        "cost": 0.0, "models": set(), "project": "",
    })

    for path in glob.glob(os.path.join(root, "*", "*.jsonl")):
        for line in open(path, errors="ignore"):
            try:
                d = json.loads(line)
            except ValueError:
                continue
            if d.get("type") != "assistant":
                continue
            u = d.get("message", {}).get("usage")
            if not u:
                continue
            s = sessions[d.get("sessionId") or path]
            pin, pout = PRICES[family(d["message"].get("model"))]
            s["turns"] += 1
            s["inp"] += u.get("input_tokens", 0)
            s["cr"] += u.get("cache_read_input_tokens", 0)
            s["cw"] += u.get("cache_creation_input_tokens", 0)
            s["out"] += u.get("output_tokens", 0)
            s["cost"] += (
                u.get("input_tokens", 0) * pin
                + u.get("cache_read_input_tokens", 0) * pin * CACHE_READ_MULT
                + u.get("cache_creation_input_tokens", 0) * pin * CACHE_WRITE_MULT
                + u.get("output_tokens", 0) * pout
            ) / 1_000_000
            s["models"].add(family(d["message"].get("model")))
            s["project"] = os.path.basename(os.path.dirname(path))

    rows = sorted(sessions.values(), key=lambda r: -r["cost"])
    tot = defaultdict(float)
    print(f"{'PROJECT':<28}{'TURNS':>7}{'OUT':>10}{'CACHE RD':>12}{'$':>9}  MODELS")
    for r in rows[:15]:
        print(f"{r['project'][-28:]:<28}{r['turns']:>7}{r['out']:>10,}"
              f"{r['cr']:>12,}{r['cost']:>9.2f}  {'/'.join(sorted(r['models']))}")
    for r in rows:
        for k in ("turns", "inp", "cr", "cw", "out", "cost"):
            tot[k] += r[k]
    print(f"\n{len(rows)} sessions | out {tot['out']:,.0f} "
          f"| cache read {tot['cr']:,.0f} | cost ${tot['cost']:,.2f}")
    print(f"ratio: {tot['cr']/max(tot['out'],1):.0f} tokens re-read per token generated")


if __name__ == "__main__":
    main(*sys.argv[1:])
```

Run it and you get your own version of the table above:

```bash
python3 ai_cost.py

# PROJECT                       TURNS       OUT    CACHE RD        $  MODELS
# ...-home-polly                  518   422,318  85,877,492   150.26  fable/sonnet
# ...-on-behalf-of-RFC-8693       434   469,612  82,888,437   132.39  fable
# ...-infrastructure-settings     572   452,359  82,002,417    57.77  opus/sonnet
#
# 21 sessions | out 2,574,944 | cache read 382,286,737 | cost $544.29
# ratio: 148 tokens re-read per token generated
```

Two things jump out immediately on anyone's data. First, your own read/write ratio, which is the single most diagnostic number in the whole exercise. Second, which sessions are your whales.

For per-session totals without writing anything, [`ccusage`](https://ccusage.com/guide/installation) does the same job from the CLI:

```bash
# total cost of one specific session
ccusage session --json -i <session-id>
```

### Step 2: The Ticket Protocol

Log cost is half the equation. You still need the denominator: what work got delivered. That means correlating spend to tickets, and no tool does that for you because only the human knows which session belonged to which ticket.

So do it manually. No Jira plugin, no integration, no config change. On every ticket where AI was involved, drop a structured comment:

```
AI Usage Log
────────────────────────────────────────
Net work time:   ___ min   (exclude calls, waiting, blockers)
AI session cost: ___ $     (ccusage session --json -i <session-id>)
Model used:      Fable / Sonnet / Opus / Haiku
Session span:    ___ min   (how long was it open)
Multi-person:    yes/no    (if yes, each person logs their own cost)
────────────────────────────────────────
```

Five fields, thirty seconds. The `Session span` field is deliberately separate from `Net work time`: the gap between those two numbers *is* the marathon problem, quantified per ticket.

Pick three or four tickets this week. That's the entire ask. Real data in one sprint, no tooling changes.

### Step 3: The Metrics That Matter

**Now (manual, per ticket):**

- `$/ticket`: AI cost per resolved ticket
- `net_min/ticket`: real hands-on time
- `$/net_hour`: AI efficiency, output per euro invested

**After two or three sprints, once there's enough data:**

- `$/ticket` split by type: bug vs feature vs knowledge-base lookup. Are they actually different? (If a KB lookup costs as much as a feature, you've found a routing problem.)
- Same ticket type, wildly different cost across people. That's a coaching opportunity, not a performance issue.
- Is the backlog trend correlated with lower `$/ticket`?

**North star, when the practice is mature:**

```
Deploy Velocity per AI $     = deploys_per_day / ai_spend_daily
Cycle Time Reduction per $   = (baseline_ct − ai_ct) / ai_spend
Story Velocity per 1M Tokens = story_pts / (tokens_in + tokens_out) × 1M
Model Efficiency Score       = output_quality_proxy / cost_per_1k_tokens
AI Adoption Maturity Tier    = kmeans(tokens, cost, cycle_time, pr_velocity)
```

`Model Efficiency Score` needs a quality proxy that isn't gameable: PR merge rate and first-review pass rate work well. They distinguish smart model selection from brute-force spend, which is exactly the Cluster A versus Cluster C distinction made measurable.

## From Measurement to a Development Framework

Measurement is pointless if nothing changes. Here's what the data actually prescribes, ordered by impact.

### Rule 1: One Ticket, One Session

The whale session is the single most expensive artefact in the dataset. The fix is a habit, not a tool.

Ticket done, close the session. New task, new session. Don't leave a session running across days because "I might come back to it". You will come back to it, and you'll drag 250 million tokens of history behind you when you do.

When context genuinely needs to survive across sessions, write it down. A short handover note in the repo (or a memory file) costs a few hundred tokens to write and a few hundred to read. Keeping the session alive to preserve the same information costs millions.

{{< mermaid >}}
flowchart LR
    A[Ticket assigned] --> B[Fresh session]
    B --> C[Work]
    C --> D{Ticket done}
    D -->|yes| E[Log cost to ticket]
    E --> F[Close session]
    F --> G[Next ticket, new session]
    D -->|no, need break| H[Write handover note to repo]
    H --> F
{{< /mermaid >}}

### Rule 2: Route the Model to the Task

Not every task deserves the frontier model. A simple decision rule covers most of it:

| Task | Model tier |
|---|---|
| Knowledge-base lookup, search, reading docs, summarising | Haiku or Sonnet |
| Standard implementation, tests, refactor within a file | Sonnet |
| Architecture design, complex debugging, multi-file refactor | Opus or Fable |

Same token volume on the wrong model is 4x to 6x the cost per delivered story point. And remember the multiplier hits the context slice, not just the output, so a long session on an expensive model compounds badly.

### Rule 3: Compress the Output

Worth 25% and it's fully automatable, so there's no reason not to. Terse mode, dropped filler, technical substance and code blocks preserved verbatim. Set it once and forget it.

Just don't mistake it for the main lever. It's the one that's easy to reach, not the one that's big.

### Rule 4: Build Skills, Not Prompts

This one isn't in the cost data directly, but it shows up in the ratio. A repeated task re-explained from scratch every time is a session that grows context linearly with repetition. A repeated task encoded as a **skill** (a reusable, versioned instruction set the agent loads on demand) starts each run from a small, fixed context.

The economics: a prompt you retype is a cost you pay every time, with the whole exploratory conversation attached. A skill is a cost you pay once at authoring time, then amortise. The read/write ratio is the metric that catches this. If it climbs above 150 on routine work, you're re-establishing context that should have been codified.

## The Maturity Path

Nobody goes from zero to a dashboard. The sequence that works:

```
Now (manual)            Sprint 3-4 (patterns)       Q4 (dashboard)
──────────────────────────────────────────────────────────────────────
ccusage per ticket   →  $/ticket by type         →  DORA AI-native
Net time logging     →  Cluster A/B/C per person →  Coaching at scale
Session discipline   →  Model routing rules      →  LiteLLM ingest → Datadog
```

The left column is a text comment on a ticket. The right column is a Datadog dashboard fed by LiteLLM cost ingest, correlated with GitHub and ArgoCD deploy data. You cannot start at the right column, because you won't know what to put on it.

## The Honest Caveats

I'd rather state the limits than have someone find them.

**These are API-equivalent dollars, not an invoice.** On a subscription plan the benefit translates into less pressure on usage limits, not a smaller bill. The relative comparisons all hold, the absolute number is a shadow price. That's still the right number for comparing tasks, models, and people, because it's the only one that scales with actual consumption.

**The compression counterfactual is an estimate.** Output divided by 0.35, from a benchmarked 65% average reduction. The logs don't record which mode was active turn by turn, so the assumption is that it was always on. And it's conservative: more output also means longer history feeding later turns, so uncompressed input and cache would have grown too. The real saving is probably higher than $341.

**One person's data is not a team's data.** N=1, 27 sessions, one working style, mostly infrastructure and platform work. The *shape* (cache read dominance, power-law session distribution, whale effect) I'd expect to generalise, because it follows from how the protocol works, not from how I work. The specific percentages should not be quoted as anyone else's.

**Correlation is not causation on the velocity side.** A shrinking backlog while AI usage grows is a hypothesis, not a proof. That's precisely why the ticket protocol exists: to get a denominator instead of a vibe.

**Cluster tiers are a coaching tool, not a ranking.** The moment they're used for performance review, people optimise the metric instead of the work, and the data becomes worthless. Guard this hard.

## Conclusion

The three questions had answers all along, sitting in a log directory nobody reads.

**What it costs:** $1,036 for 61 days of heavy usage across 15 projects, of which 82% is context handling and only 17.7% is the model actually producing something.

**How much faster:** 22% cycle time reduction on AI-assisted PRs with change failure rate flat. Measured, not felt.

**How it's used:** badly, mostly. Marathon sessions and unrouted models. Eight sessions carry 80% of the cost, and the single worst one, kept open eight days, carries a quarter of the total on its own.

The good news is that the three biggest levers cost nothing to pull. Close your sessions. Route your models. Compress your output. Together they're worth most of the bill, and not one of them requires buying anything.

Start with the script. Run it on your own logs today and look at one number: tokens re-read per token generated. If it's over 150, you don't have an AI cost problem. You have a session hygiene problem wearing an AI cost problem's clothes.

## Reflections

The uncomfortable part of this exercise wasn't the total. It was realising that I'd spent months optimising the wrong 17.7% while the 55.5% grew unwatched, because output is the part you can see. You read every token the model writes. You never see the 12,725 it read to write them.

That's the general shape of the problem, and it's not new. It's the same reason cloud bills surprise people: the expensive thing is the thing with no UI.

**What's still missing?** A few things, honestly.

Per-ticket attribution is still manual, and manual means it decays. Somebody will forget, then two people forget, and in six weeks the dataset has holes. The fix is probably a hook that stamps session cost into the commit trailer automatically, so attribution is a side effect of work rather than an extra step. I haven't built it yet.

There's also no quality dimension in any of this. `$/ticket` treats a clean, well-tested, well-reviewed fix and a barely-passing hack as identical outcomes. `Model Efficiency Score` gestures at it with merge rate and first-review pass rate, but those are weak proxies. A cheap ticket that generates two follow-up bugs is not cheap, and nothing in this framework catches that today.

And the whole thing measures cost precisely while measuring value approximately. That asymmetry is dangerous: the precise number tends to win arguments against the fuzzy one, which biases every decision towards cheap over good. Worth naming out loud before anyone builds a dashboard on it.

Still, approximate value against real cost beats what most teams have right now, which is approximate value against no cost at all.

---

Related: [The Safe Zone: Where AI Actually Belongs in Business Processes](/posts/the-safe-zone-where-ai-actually-belongs-in-business-processes/) covers the other half of this question, which processes are worth pointing AI at in the first place.
