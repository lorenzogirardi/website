# AI Makes Developers Faster. Does It Make the Team Faster?

### Table of Contents

  * AI-First and AI-Native
  * Aurora: a Real AI-First Showback
  * The Security Review as a Usage Strategy
  * The Team's Own Account
  * Where the Speed Goes
  * What AI-Native Looks Like
  * The Sameness Risk
  * How You Would Know
  * Conclusion
  * Reflections



Here we are. Everybody wants to measure how much faster AI makes developers, and it's fair: an agent writes code faster, that's indisputable. The number lands on a slide, next to a dollar figure and a session count, and everyone nods. Nobody asks what it's divided by, or what happens to the code after it's written.

## AI-First and AI-Native

Two words the market uses loosely, so let me pin them down.

By **AI-first** I mean what most companies do today: give the tools to individual developers, the assistants and the agents, and leave everything around them as it was. The AI is bolted onto a process designed for a world without it.

By **AI-native** I mean the operating model redesigned around the intelligence: work decomposed so agents and automation take the repetitive parts, compliance and security checks living inside the pipeline instead of waiting in a queue, approvals driven by risk criteria instead of by whoever is available.

The first is a purchase. The second is a redesign. And most "AI-augmented" labels on slides describe the first, backed by a spend figure.

In the run-up to 2008 a lot of smart money bought collateralized debt obligations on the strength of a rating. AAA, diversified: the label did the work that reading the loans should have done. "Team X is AI-augmented" is the same move. Two teams can show the identical spend for opposite reasons, and the dashboard can't tell them apart. So here is one attempt to open the tranche: a real AI-first showback, and what it could and couldn't say.

## Aurora: a Real AI-First Showback

A team I work with ran an AI usage showback on a loyalty identity service I'll call **Aurora**, built for a large retail organization over three months. The company, the team and every downstream system are anonymized placeholders. The numbers, the findings and the reasoning are real. It had three parts: development cost tracked by the engineers, a security review run by the platform engineering lead, and a debrief with the team.

The cost came from **ccusage**, a CLI that reads Claude Code's local session logs. At the end of each session the engineer ran one command and pasted the JSON as a comment on the ticket.

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

Eleven thousand tokens typed by the engineer, 54 million served from cache: the whole codebase, tests and config, loaded once and reused. That ratio is the economic story of an agentic coding session. Ten sessions, six tickets, $139.40 in total:

| Ticket | Model | Description | Output tokens | Cache read | Cost |
|---|---|---|---|---|---|
| AUR-9001 | Opus + Sonnet | Data layer for account mappings | 293,430 | 79.3M | $63.10 |
| AUR-9002 | Sonnet | IdP token verification | 361,354 | 54.1M | $32.95 |
| AUR-9003 | Opus | Deploy pipeline setup | 70,791 | 17.8M | $23.40 |
| AUR-9004 | Opus | Repo bootstrap + IaC | 37,599 | 12.2M | $8.15 |
| AUR-9005 | Sonnet | IdP credential-rotation hook | 109,907 | 12.6M | $7.60 |
| AUR-9006 | Opus | Telemetry + dashboards | 33,395 | 5.2M | $4.20 |
| **Total** | | | **906,476** | **181.3M** | **$139.40** |

Don't read the table as difficulty. AUR-9001 tops the cache volume because the database integration needed five iterative sessions, not because it was five times harder than the JWT story. Routing choices and iteration count explain most of the variance.

Now look at how it was done. The cost was collected by hand, one pasted blob per session. The security review was run by hand. The agent was given to the engineers, and everything around it was people doing what people did before. That's AI-first, and it's a perfectly reasonable place to start.

## The Security Review as a Usage Strategy

The security review is the cleanest example of how the AI was actually used on Aurora.

The platform engineering lead took a public library of roughly 800 skill files, each describing one vulnerability pattern, and ran two prompts by hand against the finished code. The first reads the stack and picks the relevant skills (8 of ~800). The second runs once per skill and reports severity, confidence, file and line, exploit scenario and fix. Eight findings, $3.05, one session.

Three were fixed. Five were closed without a code change, each with a written reason: three independent library layers already reject `alg:none`, an identity cache that changes the rate-limit threat model, an accepted architecture decision about forwarding the user's token, intentional business logic, a low-risk item queued as hardening. Read as a scorecard, that looks like a 62% false-positive rate. It isn't: every rationale was tribal knowledge that now sits next to a finding, and most of what didn't land concerned the infrastructure around the code, which a review that reads only source can't see.

> The AI generates the challenge with enough depth to be taken seriously. The team answers with context the AI doesn't have. That exchange is the product.

But look at the shape. A second pair of eyes, on demand, at the end, with a person judging every answer. That's AI-first. Generation cost three dollars. The judgment was human, took however long it took, and nobody recorded it.

The AI-native version of the same idea is not a better prompt. It's the same skill library running as a gate on every pull request, with severity and confidence thresholds deciding what blocks, and a person involved only for the exceptions. The skills exist. The output is already structured. What was missing is the pipeline.

## The Team's Own Account

The debrief covered Aurora and two adjacent projects: nine findings, four wins and five pain points.

The wins all had context given upfront: an architectural plan before coding, reference implementations to scaffold from, a shared team context file, a real JWT for local testing. The pain points all had context missing or verification impossible: a deprecated runtime the model couldn't check against, no template for the team's cloud conventions, no visibility into adjacent systems, a UI migration that broke visual state until screenshots went back in.

Notice what none of them is: slow generation. Every one is about what surrounds the agent. That's the signature of AI-first: the bottleneck leaves the code and moves into the system around it.

## Where the Speed Goes

A team doesn't spend 100% of its time developing. Take away the fixed ceremonies, the interrupts, the support rotation, the meetings with other verticals, the waiting on someone else's review. If a team touches 40% of its week with actual development, it's already doing well. Make that slice 50% more productive and the whole team got **20% faster**, not 50%. That's an illustration, not an Aurora measurement, but the number on the slide and the number in the calendar are different numbers, and only the second one is felt by the business.

And Aurora can't say more, because it measured only the numerator:

- **Ten sessions in three months.** I don't have their durations, so I won't turn that into a percentage, but most of the calendar wasn't spent with an agent running.
- **No baseline.** All six tickets were AI-assisted. No unassisted ticket of the same kind to compare, so no speedup can be computed, mine or anyone's.
- **The human time is unrecorded.** Judging eight findings and writing five rationales was human work, and it appears in no row.

In an AI-first company the code takes minutes and then waits: for a review, for an approval, for the next release window. That wait is exactly what a spend figure can't see.

Strange... then why do so many teams that adopted agents feel they should be flying and aren't? I see two readings, and Aurora fits both.

**The speed is queued behind the old process.** Same approval chain, same release windows, same handoffs between verticals, all designed for a world where writing the code was the slow step. I could build a couple of useful things in an afternoon, but they aren't in the roadmap, they aren't in the backlog, and no other team would use them. Done anyway, they'd be speed with nowhere to go.

**The speed turned into something the process has no column for.** More documentation, more architecture, more tests, a harder problem tackled in the time that would have split it into three tickets. On Aurora: five written rationales that didn't exist before, a shared context file. We now write more documents than any human can read, and I'm not convinced a person needs to read them. Their value is that an agent does.

Both can be true at once, and both are the same condition: an old process absorbing the output of a new capability.

## What AI-Native Looks Like

| | AI-first | AI-native |
|---|---|---|
| Where the AI sits | In the developer's editor | In the operating model |
| The work | Same tickets, written faster | Decomposed: agents take repetitive code, regression tests, documentation |
| Security and compliance | A review at the end, by hand | Automated checks inside the CI/CD pipeline |
| Approvals | Queues, meetings, a change board | Risk-based criteria, people for the exceptions |
| What you measure | Spend, sessions | Flow, quality, value delivered |

It is not a tool you buy. It's a decision to treat code generation as cheap and redesign everything that assumed it was expensive: how work is cut, where a check lives, who approves what, and how often.

Aurora has the seeds, no more than that. The skill library is the seed of a pipeline gate. The shared context file is the seed of work that agents can pick up without a human briefing them. The ccusage blob is the seed of telemetry. Each one is currently done by hand by someone who cared.

## The Sameness Risk

Your competitors can buy the same tools tomorrow. If everybody has the same agents and the processes stay slow, the advantage evaporates: faster writing becomes only more work waiting for approval. The tool is not the edge. The operating model is, and it's far harder to copy.

## How You Would Know

The [SPACE framework](https://queue.acm.org/doi/10.1145/3454122.3454124) says it about developer productivity in general: no single number, several dimensions. Applied to the move from AI-first to AI-native, each hole in Aurora has a fix.

**No baseline? Tag the work.** A few cheap fields on every item: work type, a size class (defined by the team, never used on individuals), AI assistance (none, light, substantial), AI use case, external dependency, and the reason when it was blocked or cancelled. Then you can compare "medium bug, no external dependency" with and without AI.

**Human time unrecorded? Measure transitions.** "In Progress" is too coarse. With timestamps for discovery, implementation, review, validation and release you can test the hypothesis that matters: the agent shortens implementation, but the gain is absorbed by review and product decisions. That's the AI-first signature, and it tells you where to redesign. I'll admit this is the one I don't do, because it takes rigor I don't have on a normal Tuesday.

**Capacity mistaken for value? Split the throughput.** Delivered, cancelled, enablement (refactoring, tests, CI/CD, documentation) and unplanned. A team that invests in enablement stops looking slow, and one that splits stories finer stops looking fast.

**Let the tool report on itself.** Friction used to force visibility: ask a teammate, file a ticket, wait for a review, and someone else knew what you were doing. That was a free byproduct of work being hard. When one engineer plans, builds, tests and ships with agents running unsupervised, the byproduct goes with the friction. The ccusage blob is a sensor bolted back on by hand. Claude Code can export OpenTelemetry metrics for tokens, sessions, tool calls and cost ([monitoring docs](https://code.claude.com/docs/en/monitoring-usage)), aggregated by team, repository and class of work. Never a ranking of people, never tokens as a proxy for productivity. In an AI-native model the measurement is built in, or autonomy and blindness arrive on the same day.

## Conclusion

One project, three reports, and the number that mattered wasn't $139.40 or $3.05.

Aurora is an honest AI-first showback: the tools went to the engineers and the process around them stayed. It shows what that buys: speed in the development slice, which on a team that develops 40% of the time is a fraction of the headline. It also shows what it can't see: the wait after the code is written, the human judgment around every output, the baseline you'd need to claim anything.

AI-native is the other move. Decompose the work, put the checks in the pipeline, let risk criteria approve, keep people for the exceptions. The security review already hints at it: same skills, same output, run as a gate instead of an afternoon. The number to watch is not what the agents cost. It's how long a finished change waits.

## Reflections

I don't know if $139.40 for this project is a good number or a bad one. I don't have ten other teams' showbacks to compare against, and this post doesn't pretend otherwise. It's one team, one project, a template for the next showback, not a verdict.

What I do have is a question I should ask more often on my own AI usage: not "is this cheap", but "what happens to the output once the agent has finished". Mine, too, is mostly AI-first.

If you want the cost side in more detail, I wrote about [what AI actually costs across 27 sessions of real data](/posts/what-ai-actually-costs-27-sessions-of-real-data/), and about [where AI belongs in business processes](/posts/the-safe-zone-where-ai-actually-belongs-in-business-processes/).

