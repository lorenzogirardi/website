---
title: "Card to Artifact: Where a Pipeline Should (and Shouldn't) Use AI"
date: 2026-08-13
draft: true
description: A deep dive into an agentic SDLC pipeline that turns a Trello card
  into a reviewed PR, and why AI only touches 2 of its 11 components.
tags:
  - ai
  - automation
  - langgraph
  - github actions
  - kubernetes
  - security
  - python
featuredImage: /images/card-to-artifact-the-agentic-sdlc-pipeline-mechanism/01-full-pipeline-sequence.png
---
### Table of Contents

- Not everything needs to say "AI" on the box
- The mechanism, one diagram
- Where AI actually sits
- Everything else: nine components, zero AI
- Why the split is drawn exactly here
- Deep dive: swapping the LLM provider mid-flight
- The loop I deliberately didn't build
- The near future, and why it isn't today
- Reflections

Here we are, again, staring at a Trello card and wondering how much of what happens next actually needs a model in the loop.

## Not everything needs to say "AI" on the box

I keep running into the same instinct on every project that touches an LLM: reach for "AI" as the label for the whole thing, then work backward to where it fits. That's backward. "AI" isn't a feature, it's a buzzword stapled onto whatever component happens to call an API with a prompt in it. The question that actually matters is narrower and much less exciting to put on a slide: **where does this step need to be deterministic, and where does it need to steer a genuinely non-deterministic interaction?**

Those are different jobs with different failure modes. A deterministic step is a rule: same input, same output, forever, auditable by reading the code once. A non-deterministic step is a judgment call dressed up as a function call: it can be *right*, it can be *plausible but wrong*, and no amount of prompt engineering makes that distinction go away. You pick which one you need by weighing the same four things every time:

- **Speed** - a deterministic rule runs in milliseconds; a model call is measured in seconds to low minutes.
- **Risk** - what happens if this step is wrong and nobody notices?
- **Cost** - tokens aren't free, and neither is the human time spent re-checking a plausible-sounding answer.
- **Exposure** - does this step touch anything that ships, gets deployed, or leaves the building?

Run that checklist against "does this code pass its tests" and the answer is obviously deterministic: pytest doesn't have opinions. Run it against "turn this paragraph a human wrote into a list of things to build" and the answer is obviously not: no fixed rule parses free text into intent reliably. The interesting engineering isn't picking one side, it's drawing the line between them precisely and then never letting either side wander across it.

I built a small pipeline around exactly that question. It's called an agentic SDLC on purpose, not "AI SDLC": the workflow is the point, AI is one ingredient in it, in exactly two places. The repo is public: [github.com/lorenzogirardi/agentic-sdlc](https://github.com/lorenzogirardi/agentic-sdlc).

## The mechanism, one diagram

The pipeline takes a Trello card (title, acceptance criteria, a list of which checks to run) and turns it into an open pull request plus a pushed container image, with a human approval gate in the middle. No step in between is a black box.

![Full sequence: card to Trello, through the webhook, planner, coding agent, verification DAG, reviewer, and back to the card](/images/card-to-artifact-the-agentic-sdlc-pipeline-mechanism/01-full-pipeline-sequence.png)

Read the violet bands first: those are the only two points in the entire flow where a model gets called. Everything else, including the green band (the verdict), is a function evaluating fixed rules over structured data. A webhook receiver verifies an HMAC signature and parses the card into a `TaskSpec`. GitHub Actions checks the repo out and hands off to the orchestrator. The orchestrator asks a planner for an execution order, hands the coding step to an LLM-backed agent, runs a verification DAG over the result, and lets a rule-based reviewer decide the verdict. Only then does anything touch GitHub or a registry.

Zoomed further out, the system context makes the same point from a different angle:

![C4 context diagram: Requester and Reviewer as the two humans, the platform in the middle, Trello/GitHub/LLM Provider/Registry as external systems it talks to](/images/card-to-artifact-the-agentic-sdlc-pipeline-mechanism/02-c4-system-context.png)

The LLM provider is drawn the same way GitHub and the container registry are: an external system the platform talks to, not something baked into it. That's not decoration, it's the actual architecture, and it gets tested for real later in this post.

## Where AI actually sits

**Workflow engine:** the orchestrator is a [LangGraph](https://langchain-ai.github.io/langgraph/) `StateGraph`. I picked it specifically because the execution model is a DAG: build a graph of nodes with explicit dependencies, let the engine walk it in the right order, and let each node be whatever it needs to be, model call or subprocess, without the engine caring. LangGraph doesn't know or care that two of its nodes happen to call an LLM. It just runs nodes in dependency order and enforces the interrupts (severity blocking, human-approval routing) around them. That's the right tool for "iterate over a set of agents in a defined order with conditional branching," which is a DAG problem, not an AI problem.

Inside that graph, exactly two components can ever call an LLM:

![C4 component diagram of the Orchestrator: PlannerAgent, CodingAgent, and OpenCodeAdapter in violet at the top; LangGraph StateGraph, PolicyEngine, the five verification agents, and ReviewerAgent in green below](/images/card-to-artifact-the-agentic-sdlc-pipeline-mechanism/03-c4-orchestrator-components.png)

Laid out flat, the split is this simple:

**Uses AI** - only where a natural-language request has to become something concrete:

- `PlannerAgent` - DAG generation, only when the request is ambiguous
- `CodingAgent` - the only source of new code

**Never uses AI** - everything that inspects, verifies, gates, or ships the result:

- `repo_inspector` - static file/extension matching
- `test_pyramid` - runs `pytest`
- `security` - runs `gitleaks` + `semgrep`
- `lint` - runs `ruff check`
- `code_quality` - runs `mypy`
- `docker` - static Dockerfile rules + `hadolint`
- `PolicyEngine` - YAML severity/approval rules
- LangGraph `StateGraph` - deterministic graph execution
- `ReviewerAgent` - rule-based verdict aggregation

**PlannerAgent** resolves every agent name the card requests against a fixed alias table. If a card names its agents explicitly (`repo_inspector`, `coding`, `test_pyramid`...), which is the common case, resolution is deterministic and the DAG gets built by a plain linear fallback in well under a millisecond, no model call at all. The LLM path only fires when a name doesn't resolve, e.g. a card that says "make this more secure" instead of naming `security` directly. Ambiguity is the only thing that buys a model call here.

**CodingAgent** always calls the LLM. It's the only agent that writes code, and it's deliberately the smallest possible surface for that: it reads the target repository's existing files first (up to 20KB, skipping anything that looks like a secret by path), so the model extends real code instead of inventing a parallel implementation from scratch. That "read first" behavior isn't incidental. The first version of this agent didn't do it, and on its first live run it quietly rewrote a working FastAPI service into Flask, from scratch, unprompted. Fixed once, it has converged on the first attempt on every run since.

The messages sent for the coding step, verbatim from the source, three parts every single call:

```text
# 1. injected by the adapter, ahead of everything else, whenever a schema is requested
system: Respond with ONLY valid JSON matching this schema, no prose, no markdown code fences:
{JSON schema of CodingResult}

# 2. the actual instructions
system: You are a senior software engineer. Given a task and the CURRENT
content of the relevant repository files, produce the minimal file
changes to satisfy the task. Extend the existing code shown below,
match its framework, style, and response shapes. Do not rewrite
working code to a different framework or library unless the task
explicitly asks for it. Return valid JSON with 'changes' array. Each
change has: path, content (FULL file), rationale. Never modify .env,
secrets, credentials.
[+ "Fix these issues: ..." appended here on retry turns, if tests failed]

# 3. the task itself
user: {
  "title": "...", "description": "...", "acceptance_criteria": [...],
  "existing_files": "--- app.py ---\n<full file content>\n\n--- test_app.py ---\n<full file content>"
}
```

No creative license, no "use your best judgment about architecture." The model's entire job is: extend this, don't reinvent it, return structured JSON that fits a schema decided in code, not by the model. Everything downstream re-checks it anyway.

For comparison, the planner's system prompt (the one that only fires on an unresolved agent name) is the same kind of boring, just aimed at a DAG instead of a diff:

```text
system: You are a software architect planning agent.
Analyze the task and produce a structured execution plan.
Always return valid JSON matching the schema.

Rules:
- Identify which agents are needed.
- Generate a DAG (Directed Acyclic Graph) with agent names and their dependencies.
- repo_inspector must run before other agents.
- reviewer always runs last after all other agents.
- Independent agents can have no dependencies (empty list).
- Identify risks and assumptions.
- Determine if human approval is needed.

user: {"title": ..., "description": ..., "acceptance_criteria": [...], "requested_agents": [...]}
```

## Everything else: nine components, zero AI

The part that decides whether any of this ships is not AI, on purpose. Nine components in the pipeline never call an LLM under any input: `repo_inspector` (static file/extension matching), `test_pyramid` (`pytest -q`), `security` (`gitleaks` + `semgrep`), `lint` (`ruff check`), `code_quality` (`mypy --ignore-missing-imports`), `docker` (static Dockerfile rules + `hadolint`), the `PolicyEngine` (YAML severity and approval rules), the LangGraph engine itself, and `ReviewerAgent`.

The full inventory, component by component, with the actual source file behind each row:


| Component | Kind | What it actually runs | Source |
| ---------------- | ---------------- | ----------------------------------------------------------------------- | ---------------------------------- |
| PlannerAgent | AI (conditional) | LLM only if agent names are unresolved; else deterministic alias lookup | `agents/planner.py` |
| CodingAgent | AI | Always calls the configured LLM provider | `agents/coding.py` |
| OpenCodeAdapter | AI infra | Provider-agnostic OpenAI-compatible client | `integrations/opencode_adapter.py` |
| repo_inspector | deterministic | Static filename/extension indicator matching | `agents/repo_inspector.py` |
| test_pyramid | deterministic | `pytest -q` | `agents/test_pyramid.py` |
| security | deterministic | `gitleaks` + `semgrep` | `agents/security.py` |
| lint | deterministic | `ruff check` | `agents/lint.py` |
| code_quality | deterministic | `mypy --ignore-missing-imports` | `agents/code_quality.py` |
| docker | deterministic | Static Dockerfile rules + `hadolint` | `agents/docker.py` |
| PolicyEngine | deterministic | YAML rule evaluation (severity, approval) | `orchestrator/policy_engine.py` |
| LangGraph engine | deterministic | Graph construction + execution, any node type | `orchestrator/langgraph_engine.py` |
| ReviewerAgent | deterministic | Rule-based verdict aggregation | `agents/reviewer.py` |


Twelve rows, three tagged AI: `PlannerAgent` and `CodingAgent` are the two that ever decide to call a model, `OpenCodeAdapter` is just the HTTP plumbing both of them share to get there. None of the three decide what ships. The `ReviewerAgent` row is worth reading in full, because it's the entire decision function that gates every artifact this pipeline produces:

```python
# agents/reviewer.py: no LLM call anywhere in this file
if blocker_count > 0:              # any critical/high severity finding
    verdict = "BLOCKED"
elif failed_steps:                  # any agent's success == False
    verdict = "REQUIRES_HUMAN_APPROVAL"
elif warnings:                      # any medium finding or non-zero exit code
    verdict = "PASS_WITH_WARNINGS"
else:
    verdict = "PASS"
```

An if/elif chain over counts. Anyone can read it in ten seconds and know exactly what it will do for any input, forever. On every one of the three demo runs so far, it produced `REQUIRES_HUMAN_APPROVAL`, not because a model got nervous, but because `mypy` exited non-zero and the rule says that's enough to stop and ask a person.

Here's the DAG from one of those runs, agent by agent, with real timings:

![Sequence diagram of the verification DAG: repo_inspector, test_pyramid, security, lint, code_quality, docker, each executing in order, with code_quality failing and the reviewer aggregating to REQUIRES_HUMAN_APPROVAL](/images/card-to-artifact-the-agentic-sdlc-pipeline-mechanism/04-verification-dag-run.png)

Worth being honest about the amber bands: `security` and `docker` reported "pass" here because `gitleaks`, `semgrep`, and `hadolint` weren't installed on that runner, not because they actually scanned anything. A missing binary currently gets treated as "no findings" instead of "couldn't verify," which is a real gap and one I'd fix before trusting those two gates for anything that matters. It's on the list. The point stands regardless: the one real failure (`code_quality`, mypy exit 2) is what actually held the release, and it held it the same deterministic way every time.

## Why the split is drawn exactly here

AI gets used exactly where a fixed rule can't substitute for it: turning an ambiguous, natural-language request into a concrete DAG, or into a concrete diff. Everywhere a fixed rule *can* apply, does this pass its tests, does this leak a secret, does this violate a type contract, is this severity high enough to stop, the system uses one, deliberately.

That means a reviewer never has to trust the model's judgment about what's safe to ship, because the model is never asked for that judgment. They only have to trust, or independently re-verify, a short YAML rule file and a handful of well-known CLI tools. That's a much smaller thing to audit than "trust the model," and it's a much smaller thing to explain to whoever signs off on putting this in front of a real codebase.

## Deep dive: swapping the LLM provider mid-flight

The clearest demonstration of "the AI boundary is narrow and swappable, not load-bearing everywhere" was a run where I switched the LLM provider underneath the coding agent while the pipeline kept running, [PR #18](https://github.com/lorenzogirardi/agentic-sdlc/pull/18) on the public repo. `OpenCodeAdapter` is a thin OpenAI-compatible client with no vendor SDK and no hardcoded endpoint assumptions beyond "this speaks OpenAI-shaped chat completions." Swapping providers meant changing three GitHub Actions variables, nothing else:

```bash
OPENCODE_BASE_URL = openrouter.ai/api/v1
OPENCODE_API_KEY  = <OpenRouter key>
OPENCODE_MODEL    = deepseek/deepseek-v4-pro
```

Zero lines of application code touched. The next coding-agent call went straight to OpenRouter instead of OpenCode Zen and got a real 200 back.

The more interesting part is what happened *while* verifying the swap. A GitHub Actions variable, `OPENCODE_MODEL`, had been declared but never actually wired into the step's environment block, so the pipeline was silently ignoring the model selection. That's not a model catching its own misconfiguration: it's a deterministic check (does the environment the process actually sees match the environment I intended to set) surfacing a real gap, the same way a strict human reviewer would have caught it by reading the YAML closely. Fixed on the spot in one commit, confirmed on the next run.

The run itself, not a reconstruction: the actual GitHub Actions log for this card, planner through reviewer, `mypy` failing at exit code 2 and the run still finishing green because that failure routed to human review instead of blocking the workflow.

![GitHub Actions log for trello-card #32: planner, coding turn against deepseek-v4-pro, test_pyramid, security, lint, code_quality failing, docker, reviewer, ending in REQUIRES_HUMAN_APPROVAL](/images/card-to-artifact-the-agentic-sdlc-pipeline-mechanism/05-tricorn-github-actions-log.jpg)

The card itself asked for a Tricorn (Mandelbar) fractal endpoint, Mandelbrot's complex-conjugate twin. The coding agent, running on `deepseek/deepseek-v4-pro` through OpenRouter this time instead of OpenCode Zen, read the three existing fractal endpoints already in the sample service and produced one file change, +54 lines, converged on the first turn, 127.6 seconds. The entire mathematical difference from Mandelbrot is a sign flip:

```python
def _tricorn_set(w, h, xmin, xmax, ymin, ymax, max_iter) -> list[list[int]]:
    ...
    while zx * zx + zy * zy < 4.0 and iteration < max_iter:
        xtemp = zx * zx - zy * zy + cx
        zy = -2.0 * zx * zy + cy   # sign-flipped: the conjugate step
        zx = xtemp
        iteration += 1
```

That single-line correctness detail, `-2.0` instead of `2.0`, is exactly the kind of thing I don't want a retry loop rubber-stamping without a human glancing at it, at least not yet.

The pipeline then ran the same nine-step verification DAG on the resulting code as every other run:


| Agent | Result | Duration | Note |
| ---------------- | --------------------- | -------- | ------------------------------------------------------------------------ |
| `planner` | pass | 0.26ms | Deterministic fallback, no LLM call |
| `coding` | pass | 127.6s | `deepseek/deepseek-v4-pro` via OpenRouter, 5655 tokens, converged turn 1 |
| `repo_inspector` | pass | 0.9ms | - |
| `test_pyramid` | pass | 1.25s | `pytest -q`, exit 0 |
| `security` | pass (**unverified**) | 1.1ms | gitleaks + semgrep not installed on this runner |
| `lint` | pass | 7ms | `ruff check .`, exit 0 |
| `code_quality` | **fail** | 150ms | `mypy --ignore-missing-imports .`, exit 2 |
| `docker` | pass (**unverified**) | 0.8ms | hadolint not installed on this runner |
| `reviewer` | ran | 0.09ms | `REQUIRES_HUMAN_APPROVAL` |


Same `mypy` finding that gated the Newton run and the Burning Ship run before it, and the human-approval gate held exactly as designed, unaffected by which company's model had written the code:

> The system that decided whether the Tricorn fractal endpoint was good enough to ship didn't know or care that the code underneath it had come from a different company's model than the run before it.

Final outcome for the record: verdict `REQUIRES_HUMAN_APPROVAL`, [PR #18](https://github.com/lorenzogirardi/agentic-sdlc/pull/18) opened at +113/-0, provider selected entirely through configuration. Three honest caveats from that run, worth stating plainly rather than glossing over: `code_quality` has now failed identically, same `mypy` exit 2, on all three demo runs without the actual error text ever being captured; `security` and `docker` are unverified rather than passing on all three runs, because the scanning binaries aren't installed on the GitHub Actions runner; and this particular run predates the Trello card-update step, so the result was posted back to the card by hand rather than automatically.

Every one of these branches is a real run, not a cherry-picked one: `agentic/31637516768` behind PR #18 (this run), plus the ones before and after it, each opened by the same workflow, none merged without a human looking first.

![GitHub branches list: main protected with 5/5 checks, agentic/31638459242, agentic/31638297289, and agentic/31637516768 each behind their own open PR (#20, #19, #18)](/images/card-to-artifact-the-agentic-sdlc-pipeline-mechanism/07-tricorn-branches-prs.jpg)

### Proof it's a running artifact, not just an approved diff

A pull request passing review is only half the claim. The other half is that the image GHCR ends up with actually runs and actually serves the endpoint the card asked for. The build step in the same Actions run confirms the image got built and pushed:

![GitHub Actions build summary: Docker Build summary for ghcr.io/lorenzogirardi/agentic-sdlc-fractal, tagged with commit SHA and latest, completed in 6s](/images/card-to-artifact-the-agentic-sdlc-pipeline-mechanism/09-sample-app-docker-build-summary.jpg)

Pulling that exact image and running it locally, the existing endpoints answer first, proving the coding agent's "extend, don't replace" instruction actually held on real output:

![Terminal: docker run of the ghcr.io fractal image, uvicorn serving /health and /fractal with iterations/zoom/coordinate query params, all 200 OK](/images/card-to-artifact-the-agentic-sdlc-pipeline-mechanism/10-sample-app-docker-run-log.jpg)

And the new `/tricorn` route the coding agent added responds the same way, right next to the pre-existing ones in the same log:

![Terminal: docker log showing repeated GET /fractal calls followed by three GET /tricorn calls, all 200 OK, plus a curl to /tricorn returning valid JSON with type, parameters, and dimensions](/images/card-to-artifact-the-agentic-sdlc-pipeline-mechanism/06-tricorn-docker-log.jpg)

The sample service itself is a small interactive fractal explorer, this is the actual UI the container serves, not a mockup of it:

![Julia Set Explorer web UI: a rendered blue-and-orange fractal canvas with Reset/Zoom In/Zoom Out controls, iteration and zoom sliders, and a green Health: ok status line](/images/card-to-artifact-the-agentic-sdlc-pipeline-mechanism/08-sample-app-fractal-explorer.jpg)

Here's a recording of that interaction end to end, card to PR, with the provider swap happening live in the middle:

Your browser doesn't support embedded video. [Watch the recording directly](https://res.cloudinary.com/ethzero/video/upload/v1786626555/ai/agentic-workflow/agentcode-cut.mp4).

Full write-up of that run, including the code diff and the honest caveats, is in the repo: `[docs/demo/tricorn-run-provider-switch.md](https://github.com/lorenzogirardi/agentic-sdlc/blob/main/docs/demo/tricorn-run-provider-switch.md)`.

## The loop I deliberately didn't build

There's an obvious next step I'm not taking yet, and I want to be explicit about why. The coding agent currently converges or it doesn't: it writes code, the deterministic checks run once, and if something fails, a human gets paged. What it does *not* do is retry against its own failures, feed `mypy`'s output back into another prompt, try again, feed the new failure back, try again.

That loop, coding agent → test/lint/type-check results → coding agent again, closed by something resembling spec-driven development, is a real and useful thing to build eventually. It's also exactly the shape of problem that eats a project alive if you build it before you've earned it: a non-deterministic step iterating against another non-deterministic evaluation of its own output, with no hard floor underneath it, is how you end up with an agent confidently thrashing for twenty minutes and burning tokens to converge on nothing. I'd rather ship the version that stops and asks a human the first time it's unsure than the version that tries to out-clever its own mistake and can't tell when to give up.

So: one shot at the coding step, deterministic gate, human in the loop on anything short of a clean pass. The retry loop is a scoped, deliberate next iteration, not a missing feature I forgot about.

## The near future, and why it isn't today

It's not hard to see where this goes: someone writes a card, the pipeline hands back a running artifact, no human touches code in between, ever. That's a genuinely close future, closer than it was even a few runs ago in this project.

It's also not automatically worth building for every team right now, and I want to say that plainly instead of selling past it. The part of this system that makes it trustworthy isn't the model, it's the deterministic floor underneath it: the rule files, the alias tables, the exact severity thresholds, the retry-loop-that-doesn't-exist-yet decision above. Building and tuning that floor for a real, messy, production codebase (not a fractal demo service) takes someone who understands both the code being touched *and* how to correct a non-deterministic system's choices when they're subtly wrong, not obviously wrong. That's a specific and currently scarce kind of expertise, and paying for it has a cost that has to be weighed against what the automation actually saves, per company, per codebase, honestly.

That's not a reason to not start. It's a reason to pick a target further out than looks comfortable, "card in, artifact out, no human in the loop, ever" is a fine one, and build the foundations toward it deliberately: narrow the AI surface first, make the deterministic gate boringly reliable, prove portability (the provider swap above), and only then start closing loops. Aim high, build the boring parts first. The boring parts are why the exciting part will eventually be safe to ship.

## Reflections

What's still missing, honestly: the missing-binary-means-pass gap in `security` and `docker` should be fixed before those gates mean anything on a real codebase. `code_quality`/mypy has failed identically on every demo run without its actual error text being captured anywhere but the exit code, which means a human still has to run it locally to see what it flagged, a small but real gap in the "everything visible" promise. And this has only run against a small FastAPI demo service with a handful of fractal endpoints; scaling it to a real, larger codebase is a matter of pointing `repository_path` somewhere real and picking which deterministic agents apply, but I haven't done that yet, and I expect it to surface things a toy service can't.

None of that changes the core claim. The mechanism holds regardless of which codebase sits behind it: AI writes, deterministic rules decide, and the boundary between the two is narrow enough to audit in an afternoon.

---

If the "where does AI actually belong" framing is useful, I wrote a more general version of it here: [The Safe Zone: Where AI Actually Belongs in Business Processes](/posts/the-safe-zone-where-ai-actually-belongs-in-business-processes/). And if you're building a deterministic gate of your own, [AI Security Review Finds the Bug Your CI Gates Missed](/posts/ai-security-review-finds-the-bug-ci-gates-missed/) covers the same "AI runs after the deterministic gates, never instead of them" pattern applied to security scanning.