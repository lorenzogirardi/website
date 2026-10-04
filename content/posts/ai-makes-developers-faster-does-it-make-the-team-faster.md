---
title: "AI Makes Developers Faster. Does It Make the Team Faster?"
date: 2026-09-11
draft: true
description: "A real AI showback says what the agent cost, not whether the team got faster. A 50% faster developer is a 20% faster team, and the number hides what's missing."
tags:
  - ai
  - cost saving
  - platform engineering
  - security
  - devsecops
  - dora
  - skills
  - code review
featuredImage: /images/ai-makes-developers-faster-does-it-make-the-team-faster/featured.jpg
images:
  - "/images/ai-makes-developers-faster-does-it-make-the-team-faster/featured.jpg"
---
### Table of Contents

  * The 2008 Analogy
  * One Project, Three Reports
  * How the Numbers Were Collected
  * What the Development Spend Shows
  * The Security Review: Findings vs Judgment
  * The Team's Own Account
  * What Aurora Can't Tell Us
  * What We Should Have Collected
  * The Visibility Cost of Getting Good at This
  * Conclusion
  * Reflections



Here we are. Everybody wants to measure the speed of AI-assisted development, and it's a fair thing to want: an agent writes code faster, that's indisputable. Then the number lands on a slide, next to a dollar figure and a session count, and everyone nods. Nobody asks what's inside it, or what it's divided by.

## The 2008 Analogy

In the run-up to 2008, a lot of very smart money bought collateralized debt obligations on the strength of a rating. AAA, mortgage-backed, diversified: the label did the work that reading the pool of loans should have done. Nobody at scale priced the individual mortgages inside the tranche.

"Team X is AI-augmented" is the same move. It's a label, backed by a spend figure and a session count, standing in for the diligence of reading what's inside. Two teams can burn the same Opus time on the same kind of ticket for opposite reasons, one because it was genuinely hard, one because nobody could do it without hand-holding, and the dashboard shows the same number. Two teams can dismiss most of a security review's findings, one with documented reasoning, one because nobody wanted the work, and the dashboard shows the same "findings resolved" count.

This isn't an argument against using AI heavily. It's an argument against letting the word substitute for the diligence you'd apply to anything else you're rating. What follows is one attempt to open the tranche: a real project, its costs, its security findings, its team's own account. And to find out what the number was never going to tell us.

## One Project, Three Reports

A team I work with ran an AI usage showback on a loyalty identity service I'll call **Aurora**, built for a large retail organization over three months. The company, the team, the project and every downstream system are anonymized placeholders. The numbers, the findings and the reasoning are real.

The showback had three parts, from three people at three different times:

1. **Development cost**, tracked by the engineers building the service: session logs, token counts, spend per ticket.
2. **A security review**, run by the platform engineering lead using a public library of security-review skills against the finished code.
3. **Team feedback**, collected after the fact: what worked, what didn't, across Aurora and two adjacent projects.

None of them needs the other two to exist, and none is enough alone. A dollar figure doesn't say whether the code is safe, a review doesn't say whether the findings were real, feedback doesn't say where the time went. Read together they're better. Read together they're still missing something, and I'll get to it.

## How the Numbers Were Collected

The development cost came from **ccusage**, a CLI that reads Claude Code's local session logs and reports tokens and cost per session. No instrumentation, no separate billing dashboard: Claude Code already writes the logs. The convention on this project: at the end of each session, the engineer ran one command and pasted the JSON output as a comment on the relevant ticket.

```bash
ccusage session --json -i <session-id>
```

```json
{
  "session_id": "20260814_aurora-idp-jwt-auth",
  "model": "claude-sonnet-4-6",
  "started_at": "2026-08-14T09:12:03Z",
  "ended_at":   "2026-08-14T11:47:22Z",
  "tokens": {
    "input":              11391,
    "output":            361354,
    "cache_write":      2356718,
    "cache_read":      54131444
  },
  "cost_usd": 32.95
}
```

Two numbers matter. `input` (11K) is what the engineer actually typed: prompts, corrections, follow-ups. `cache_read` (54M) is the project context served from cache: the whole codebase, tests and config, loaded once and reused across every exchange. That ratio is the whole economic story of an agentic coding session.

## What the Development Spend Shows

Ten sessions, six tickets, $139.40 total.

| Ticket | Model | Description | Output tokens | Cache read | Cost |
|---|---|---|---|---|---|
| AUR-9001 | Opus + Sonnet | Data layer for account mappings | 293,430 | 79.3M | $63.10 |
| AUR-9002 | Sonnet | IdP token verification | 361,354 | 54.1M | $32.95 |
| AUR-9003 | Opus | Deploy pipeline setup | 70,791 | 17.8M | $23.40 |
| AUR-9004 | Opus | Repo bootstrap + IaC | 37,599 | 12.2M | $8.15 |
| AUR-9005 | Sonnet | IdP credential-rotation hook | 109,907 | 12.6M | $7.60 |
| AUR-9006 | Opus | Telemetry + dashboards | 33,395 | 5.2M | $4.20 |
| **Total** | | | **906,476** | **181.3M** | **$139.40** |

Cache hit rate across the project: 99.98%. Reuse ratio: 19.5x. Each fresh engineer prompt effectively cost about 2% of what it would have without caching.

Don't read the table as difficulty. AUR-9001 tops the cache volume (79.3M tokens across five sessions) because the database integration needed iterative sessions with the full codebase reloaded each time, not because it was five times harder than the JWT story. Sonnet did the high-output coding (AUR-9002: 361K output tokens), Opus did the infrastructure, where deliberation matters more than throughput. Routing choices and iteration count explain most of the variance.

One row deserves the question you should ask of *any* row. AUR-9003, the deploy pipeline, cost $23.40 on Opus, more than the credential-rotation hook. Is it a skills gap, plain complexity, or a one-time cost that bought future speed? It was greenfield and a single session, and the team feedback below points to the third reading. But one project can't tell "normal" from "needed more hand-holding than most". The cost figure alone never settles it.

## The Security Review: Findings vs Judgment

The second report came from the platform engineering lead, not a dedicated security team, using a public library of roughly 800 security-review skill files, each describing one vulnerability pattern. Two prompts, run manually against Claude Code: the first reads the stack and selects the relevant skills (8 of ~800 for Aurora), the second runs once per skill and reports severity, confidence, file and line, exploit scenario and fix. Eight findings, $3.05, one session, 44K output tokens. The baseline was already solid (parameterized SQL, timing-safe webhook verification, JWKS with issuer and audience checks, secrets via a CSI driver, a non-root container), so these were gaps in depth, not failures.

| ID | Severity | Finding | Confidence |
|---|---|---|---|
| H1 | High | No explicit algorithm restriction on JWT verification | 7/10 |
| H2 | High | DB SSL connection defaults to `disable` when the env var is unset | 9/10 |
| M1 | Medium | Caller's JWT forwarded verbatim to the profile service, no re-scoping | 6/10 |
| M2 | Medium | No rate limiting; each request fans out to three upstream systems | 6/10 |
| M3 | Medium | Commerce-platform accounts created silently on first identity resolution | 5/10 |
| L1 | Low | Full API spec served without authentication | 7/10 |
| L2 | Low | Database credentials embedded in the connection URL string | 5/10 |
| I1 | Info | A capability retained in Helm despite an explicit "drop all" | 6/10 |

The instruction back to the dev team was simple: for each finding, decide doing or not doing, and say why either way.

**Doing (3):** the DB SSL default (a one-line change), the API docs put behind an env flag, and an unused capability removed from the Helm chart (boilerplate copied from a template).

**Not doing (5), each with a written reason:**

- **H1, JWT algorithm:** three independent library layers already reject `alg:none` and confusion attacks. An explicit allowlist adds operational risk without closing a hole.
- **M1, token relay:** an accepted architecture decision, the profile service needs the user's own context.
- **M2, rate limit:** an identity cache short-circuits after the first request per user, so the fan-out needs an attacker who already holds many valid tokens.
- **M3, silent account creation:** intentional business logic.
- **L2, credentials in the URL:** real but low-risk, queued as hardening.

Read as a scorecard, five of eight without a code change looks like a 62% false-positive rate. That reading is wrong. Every rationale holds something that was tribal knowledge before: the chain of libraries, the cache that changes the threat model, the reason for forwarding a JWT instead of re-minting one. Now it's written down, attached to a finding. And most of what didn't land wasn't about the source at all: the managed database enforcing TLS server-side, the Helm production overrides. A review that reads only source can't see that. A human who owns the deployment can.

> The AI generates the challenge with enough technical depth to be taken seriously. The team answers with context the AI doesn't have. That exchange is the product.

## The Team's Own Account

The third report was a plain debrief: nine findings across Aurora and two adjacent projects, four wins and five pain points. Four patterns fell out of it.

**Context is the primary variable.** Every win had context given upfront: an architectural plan before coding, reference implementations to scaffold from, a shared team context file, a real JWT for local testing. Every pain point had context missing: a deprecated runtime the model couldn't verify against, no template for the team's cloud conventions, no visibility into adjacent systems.

**Examples beat instructions.** Greenfield scaffolding worked because existing examples set the standard implicitly. The overengineering failure happened without a reference: the model implemented every high-weight pattern it associates with the request, needed or not.

**Some surfaces need human eyes, structurally.** A legacy service on an end-of-life framework, a UI migration that broke visual state until screenshots went back in as ground truth. Neither is a prompting failure. Both are verification that can't happen inside the model's own context.

**A shared context file is a team asset, not a personal preference.** The fix for "the model needed a lot of correction on our cloud conventions" is not "prompt it better", it's writing the convention down once, in the file every session already loads.

## What Aurora Can't Tell Us

Put the three reports side by side and ask the question everyone asks: did the team get faster? Aurora can't answer. It measured the numerator and nothing else.

- **Ten sessions in three months.** The project ran a quarter, the agent ran in ten sessions. I don't have their durations, so I won't turn that into a percentage, but the shape is clear: most of the calendar wasn't spent with an agent running.
- **No baseline.** All six tickets were AI-assisted. There's no unassisted ticket of the same kind to compare against, so no speedup figure can be computed, mine or anyone's.
- **The human time is unrecorded.** The security review cost $3.05. Judging eight findings and writing five rationales was human work, and nobody wrote down how long it took. Generation got cheap, adjudication didn't, and it appears in no row.

That's where the arithmetic gets uncomfortable. A team doesn't spend 100% of its time developing. Take away the fixed ceremonies, the interrupts, the support rotation, the meetings with other verticals, the waiting on someone else's review. If a team touches 40% of its week with actual development, it's already doing well. Now say agentic development makes that 40% fifty percent more productive. The whole team got **20% faster**, not 50%. That's an illustration, not an Aurora measurement, but the number on the slide and the number in the calendar are different numbers, and only the second one is felt by the business.

Strange... then why do so many teams that adopted agents feel they should be flying and aren't? I see two readings, and Aurora's data fits both.

**The team is held back by everything around it.** The development slice got faster and the rest of the company didn't: same approval chain, same release windows, same handoffs between verticals, all designed for a world where writing the code was the slow step. Look at Aurora's pain points again: missing templates, an unverifiable runtime, unwritten conventions. None of them is about generation speed, all of them are about the organization around the agent. I could build a couple of useful things in an afternoon, but they aren't in the roadmap, they aren't in the backlog, and no other team would use them. Done anyway, they'd be speed with nowhere to go.

**The team is producing something else.** The speed went somewhere the velocity chart doesn't look: more documentation, more architecture, more tests, a harder problem tackled in the time that would have split it into three tickets. Aurora has examples: five written rationales that didn't exist before, a shared context file, the security review itself. None of them is a ticket. Even "more documentation" needs a second look: we now write more documents than any human can read, and I'm not convinced a person needs to read them. Their value is that an agent does. That's a different kind of output, and a ticket count has no column for it.

Both can be true at once, and neither shows up in a spend figure or a session count.

## What We Should Have Collected

The [SPACE framework](https://queue.acm.org/doi/10.1145/3454122.3454124) says it about developer productivity in general: no single number, several dimensions. Applied to an AI-assisted team, each hole above has a fix.

**No baseline? Tag the work and compare like with like.** A few cheap fields on every item: work type, a size class (XS to L, defined by the team, never used on individuals), AI assistance (none, light, substantial), AI use case, external dependency, and the reason when it was blocked or cancelled. Then you can compare "medium bug, no external dependency" with and without AI, instead of crediting the agent for the difference between two unrelated tickets.

**Human time unrecorded? Measure transitions, not just "In Progress".** That state is too coarse. With timestamps for discovery, implementation, review, validation and release you can test a concrete hypothesis: the agent shortens implementation, but the gain is absorbed by review and product decisions. If true, total cycle time stays flat and you finally have evidence of the new constraint. I'll admit this is the one I don't do, because it takes rigor I don't have on a normal Tuesday.

**Capacity mistaken for value? Split the throughput.** Four buckets instead of one: delivered (released and accepted), cancelled (closed without release), enablement (refactoring, tests, CI/CD, observability, documentation) and unplanned (incidents, support, urgent requests). A team that invests in enablement stops looking slow. A team that splits stories finer and "closes" more items stops looking fast.

Around that, keep what already exists but change its job: Monte Carlo to forecast, flow metrics to find the bottleneck, quality metrics to check that speed isn't buying debt, product metrics to check that something valuable shipped.

## The Visibility Cost of Getting Good at This

One more thing the debrief hides in plain sight. A reported win was "parallel agents helped with non-supervised operations": QA runs and background tasks, kicked off and left alone. That's real progress, and it's also how visibility quietly erodes.

Friction used to force visibility. Ask a teammate for help and someone else knows what you're doing. File a ticket and it sits in a queue others can see. Wait for a review and another pair of eyes crosses your work before it ships. None of that existed to produce oversight: the work was hard enough that you needed another person, and oversight was a free byproduct. When one engineer can plan, scaffold, test and ship with an agent running unsupervised, the byproduct disappears with the friction. Nobody removed the sensor. It was never designed.

That's why the ccusage-into-a-ticket convention exists, and why it looks like overhead. It's a sensor bolted back on by hand. The better version is the tool reporting on itself: Claude Code can export OpenTelemetry metrics for tokens, sessions, tool calls and cost ([monitoring docs](https://code.claude.com/docs/en/monitoring-usage)), aggregated by team, repository and class of work. Never a ranking of people, never tokens as a proxy for productivity. And the closer you get to agents orchestrating their own loops, the more it matters: if the measurement isn't built in structurally, autonomy and blindness arrive on the same day.

## Conclusion

One project, three reports, and the numbers that mattered weren't $139.40 or $3.05. Three things did.

First, a number needs opening: every AI usage figure meant something different depending on what was inside it. Second, speed gets diluted by everything that isn't development: under generous assumptions a 50% faster developer is a 20% faster team, and whether the rest of the organization moves decides the gap. Third, Aurora couldn't tell any of this apart because it wasn't built to: no baseline, no transitions, no record of human time.

That's one team, one project. It's a template for the next showback, not a verdict. Anything less than a series of them, read for contents and not totals, is a rating without a loan file.

## Reflections

I'll be honest about one thing: I don't know if $139.40 for this project, or $23.40 for that one deploy pipeline ticket, is a good number or a bad one. I don't have ten other teams' showbacks to compare against, and this post doesn't pretend otherwise.

What I do have is the questions the exercise forced me to ask. Not "is this cheap", but "what actually happened in this session, and why". Writing this made something else obvious: I'm not asking that question often enough on my own AI usage. If the argument is that the label isn't the diligence, that cuts both ways. It's not just a management problem, and the next showback should collect the fields above.

If you want the cost side in more detail, I wrote about [what AI actually costs across 27 sessions of real data](/posts/what-ai-actually-costs-27-sessions-of-real-data/), and about [where AI belongs in business processes](/posts/the-safe-zone-where-ai-actually-belongs-in-business-processes/).
