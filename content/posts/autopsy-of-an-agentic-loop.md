---
title: "Autopsy of an Agentic Loop: Six Pull Requests, Zero Humans"
date: 2026-10-03
draft: true
description: "A pipeline with no person in it: one LangGraph state machine, the
  rules that let it merge, abandon or revert, and six real pull requests."
tags:
  - ai
  - automation
  - langgraph
  - github actions
  - python
  - agentic
  - ci
  - kubernetes
featuredImage: /images/Gemini_Generated_Image_gd8shigd8shigd8s.jpeg
---
### Table of Contents

- The goal: a pipeline with no person in it
- The rule that makes it safe
- The graph
  - The engine, as a state machine
  - What happens inside the nodes
  - The system around it: events, workflows, merge
- The principles that replace a reviewer
- Six pull requests
  - Case 1: a dependency bump that merges itself (PR #159)
  - Case 2: my own pull request (PR #167)
  - Case 3: a refactor that changes behaviour (PR #168)
  - Case 4: a behaviour change on purpose (PR #169)
  - Case 5: a failure only the cluster can see (PR #171)
  - Case 6: a change nobody may repair (PR #170)
  - After the merge: the guard
- The results, side by side
- What it costs
- Reflections
  - What is still missing
- Conclusion



Well, here we are. I wanted a pipeline where I write a pull request, go away, and come back to find it merged or closed, with a reason. No approval button, no "needs a human" label, no PR sitting open for a week because nobody knows who owns it.

This post is the autopsy of that pipeline, written after it ran on real pull requests in a real repository, with a real model and real money (a few cents each). Everything below is taken from the pull requests themselves: the comments, the commits, the timings and the costs. I will show you the graph first, because the graph is the whole idea. Then six cases, from a dependency bump that merges itself to a change that no agent is allowed to repair.

## The goal: a pipeline with no person in it

The target is simple to state and hard to honour: **no human in the loop, and the quality and the functionality of the application still guaranteed**.

Those two halves pull in opposite directions. Removing the person removes the judgement. So the judgement has to go somewhere, and I put it in three places:

- **Deterministic gates** that no model can overrule: lint, unit tests, an integration suite against real PostgreSQL and Redis, and an image built from the pull request and run in a Kubernetes cluster.
- **A set of rules in code** that decide what a model is allowed to do: what it may edit, what it may never weaken, what it must quote before it may change a test.
- **A safety net after the merge**: if something slips through, the branch goes back to the last green state on its own.

The model (`deepseek/deepseek-v4.1-flash` through OpenRouter, about $0.30 per million input tokens) writes, reviews and argues. It never decides alone.

## The rule that makes it safe

Every change, whoever wrote it, ends in exactly one of three states:

1. **Merged**: its head commit is certified, and the required checks succeeded on that same commit.
2. **Abandoned**: it did not converge, even after one retry with twice the budget. It is labelled, explained in a comment, and that commit is never retried. The base branch is untouched.
3. **Reverted**: it merged, and the pipeline on `main` then failed for a reason a code change can cause. The branch goes back to the last green state and the change is queued to be redone.

There is no fourth state. A test in the repository enumerates the terminal states and fails if one of them ever hands work to a person, or if the words "needs human" come back into a label, a title or a comment. If it isn't there, it can't wait for anybody.

## The graph

This is the part I care about most. There are two levels: the **engine**, which is a LangGraph state machine, and the **system** around it, which is a set of GitHub Actions workflows that decide when the engine runs and when a pull request merges.

### The engine, as a state machine

The engine lives in one file, `agent_pipeline.py`. It is a LangGraph `StateGraph` with nine nodes and a `start` node that decides where to begin, because the same graph serves four different entry points: an issue, a pull request, a push to `main`, and a failed CI run.

{{< mermaid >}}
flowchart TD
    S[start]
    S -->|issue| W[write]
    S -->|pull request| V[verify]
    S -->|push on main| R[review]
    S -->|CI failed| C[ci_failure]
    W --> V
    V -->|checks pass, code touched| T[tests]
    V -->|checks pass| R
    V -->|code is wrong| W
    V -->|test is wrong| ST[steward]
    V -->|flaky, retry| V
    T -->|tests added| V
    T -->|nothing to add| R
    ST --> V
    ST -->|cannot update legitimately| W
    C -->|code is wrong| W
    C -->|test is wrong| ST
    C -->|environment| V
    R -->|blocking findings| W
    R -->|clean, nothing changed| D[docs]
    R -->|clean, agent changed code| F[final]
    F -->|blocking findings| W
    F -->|clean| D
    D --> E([END])
{{< /mermaid >}}

![The engine as a LangGraph state machine: start, write, verify, ci_failure, steward, tests, review, final, docs and END, with every conditional edge labelled](/images/autopsy-of-an-agentic-loop/engine-graph.png)

The same graph as a picture, for slides and for sharing: nine nodes, four ways in, one way out.

Every conditional edge is decided by the node itself: it writes a `route` into the state and the graph follows it. The only extra edge not drawn is the one every node shares: when the budget is spent, the node routes to `END` and the outcome is `abandoned`.


| Node | What it does | Can it change files? |
| ------------ | ----------------------------------------------------------------------- | ------------------------------ |
| `start` | Picks the entry point | No |
| `write` | The writer: explores the repo read-only, then proposes a patch | Yes, through a validated patch |
| `verify` | Runs the deterministic checks in a process with no secrets | No (it can revert) |
| `ci_failure` | Same as `verify`, but starts from the real logs of a failed CI run | No |
| `steward` | The test steward: updates tests that are wrong, only under `tests/` | Tests only |
| `tests` | Proactive: does the new application code have the tests it needs? | Tests only |
| `review` | Reviewers A and B, in parallel, independent, validated and deduplicated | No |
| `final` | A third reviewer that checks the earlier findings are really fixed | No |
| `docs` | Documentation reviewer, then a deterministic changelog entry | Docs and changelog only |


The whole graph is wrapped by one function: it runs once, and if it does not converge it runs **once more with twice the budget**, continuing from whatever the first attempt committed. If that fails too, the result is `abandoned`. Two attempts, never three.

### What happens inside the nodes

The nodes are small. The interesting logic is in what they call.

**The failure path.** When `verify` or `ci_failure` sees failing tests, nothing is sent to the writer yet. First the failing tests are re-run on the current tree (does it pass the second time? then it is flaky) and on the base commit (did it pass before this change?). That gives one of five hints: flaky, unreproducible, new test, preexisting, regression. Only then does a model, the *failure adjudicator*, classify each failing test as `code_defect`, `test_defect`, `environment` or `preexisting`. And then the code overrules the model: evidence wins, and a `test_defect` stands only if the model quoted the intent of the change verbatim (more on this below).

**The writer's patch.** A patch is JSON: edits with unique anchors, or whole new files, at most 8 changes. Before anything touches the tree it passes a validator: no credential-shaped string in the new text, no file the plan declared out of scope, no workflow file, no `.env`, no key or certificate. After it is applied, a second check compares the tests with how they were: fewer tests, fewer assertions, a new `skip` or `xfail`, a deleted test file, and the patch is reverted and refused. That guard runs on every role, not just the steward.

**The reviewers.** A looks at correctness and design, B at security and operations. They run in parallel, they do not see each other, and their findings must carry a severity, a file, a line that falls inside a changed hunk, the evidence and a suggested fix. A finding that points at a line the diff doesn't contain is dropped. Two findings about the same place and topic are merged and remember who raised them.

**The last gate.** Before anything is published, a deterministic check runs over the commits the agents made: protected files, binary files, a credential in an added line, a patch too large to be a reasoned change (60 files or 3000 lines), tests weakened. A violation turns the result into `abandoned`.

### The system around it: events, workflows, merge

The engine knows nothing about GitHub events. A set of workflows feeds it, and a second set decides what happens to the result.

{{< mermaid >}}
flowchart LR
    PR[PR opened or updated] --> AC[agent-change]
    AC --> G1[engine, start verify]
    CIF[PR Checks failed] --> INF{failed in the runner?}
    INF -->|yes| RR[re-run the job once]
    INF -->|no| G2[engine, start ci_failure]
    PUSH[push on main, no PR] --> ACP[agent-change push]
    ACP --> G3[engine, start review]
    ISS[issue labelled agent] --> PL[planner]
    PL --> G4[engine, start write]
    G1 --> CERT[certified at a sha]
    G2 --> CERT
    G3 --> CERT
    G4 --> CERT
    CERT --> MG{agent-merge}
    CI[required checks green on the same sha] --> MG
    MG -->|yes| MERGED[squash merge]
    REN[Renovate PR] --> SW[ai-review-sweep]
    SW -->|clean and green| MERGED
    SW -->|red CI| G2
    SW -->|blocking finding| G4
    MERGED --> PIPE[pipeline on main]
    PIPE -->|fails| GUARD[agent-main-guard]
    GUARD --> RERUN[re-run failed jobs once]
    RERUN -->|fails again| REVERT[revert to last green and redo]
{{< /mermaid >}}

![From event to merge: the workflows that start the engine, the certification, the merge gate, the pipeline on main and the guard](/images/autopsy-of-an-agentic-loop/system-flow.png)

And the same flow as a picture. Read it left to right: an event starts a workflow, the workflow starts the engine, the engine ends with a certification, and only the merge gate can turn a certification into a merge.

Three of these boxes carry the safety:

- **Certification.** When the engine finishes with no blocking finding and the checks pass, it posts a comment with a marker bound to the exact head commit: `agent-certified: <sha>`. A new push changes the sha, so an old certification never applies to new code. A comment from anyone but the agent account is ignored.
- `**agent-merge`.** It runs on every CI completion, on every push to `main` and every 30 minutes, and each run judges every open pull request. It merges a pull request only if its head is certified, the required checks (`checks`, `integration`, `image`, `workflows`) succeeded on that same commit, and a circuit breaker is closed. Whichever finishes first, the certification or the CI, and whichever event gets lost, the next pass picks it up.
- `**agent-main-guard`.** It re-runs the failed jobs once, to rule out a flake. If the failure repeats, was not already repaired by a later green run, and comes from a job a code change can cause (not a scanner), it reverts everything since the last green run and opens a work item for the pipeline to redo the change. Three automatic reverts in 24 hours open the circuit breaker, which also stops automatic merging.

## The principles that replace a reviewer

Removing the person forces you to write down what the person was doing. Four rules ended up in code.

**1. Tests are the specification.** When a test and the code disagree, the code gives way, unless the change itself says, in words, that it is redefining what the test checks. The model must *quote* those words. The quote is checked by a string comparison against the title, description and plan of the change, normalised for case and whitespace. No quote, no `test_defect`: the verdict becomes `code_defect` and the writer fixes the code.

**2. Evidence overrules the model.** If a test passes when re-run, it is flaky, whatever the model says. If it already failed on the base commit, the change is not to blame.

**3. The checks the agent cannot run, it reads.** The integration suite needs PostgreSQL and Redis, which the agent's job doesn't have. So when CI runs them and fails, the agent reads the real logs of the failed checks and judges those. And reviewers are handed the check results of the exact commit, so "I cannot verify the dependencies resolve on the new Python" is answered by a green `image` check, not by a person.

**4. Nobody is both author and judge.** The writer cannot weaken tests. The steward can only touch `tests/`. The reviewers do not see the writer's reasoning. The merge gate does not trust the model's verdict, only the certification and the CI.

## Six pull requests

All numbers below are from the repository `flask-test-api` (a FastAPI application, PostgreSQL, Redis, deployed on Kubernetes). Times are open-to-merge, costs are the model spend reported by the pipeline itself.

### Case 1: a dependency bump that merges itself (PR #159)

Renovate proposed `python:3.14.7-slim` to `3.14.8-slim`, a patch bump of the base image. Strange... the interesting part is not the bump, it is everything around it.

1. A push to `main` started the review sweep by itself. It noticed the pull request was **behind `main`**. GitHub doesn't report that without branch protection, so the sweep counts the commits (ignoring the pipeline's own bookkeeping commits) using the compare API.
2. It asked Renovate to rebase its own pull request, with the `rebase` label. A branch edited by anyone else is a branch Renovate stops managing, so the pipeline never touches it.
3. CI ran again on current code, including the `image` check: the image **built from the pull request**, deployed in a kind cluster next to real PostgreSQL and Redis, with 25 integration tests run against it.
4. The sweep waited for the required checks, then gave reviewers A and B the check results of that commit as evidence.
5. Verdict: reviewer A, reviewer B, 0 findings, clean. The pull request squash-merged.

![The sweep's verdict on PR #159: reviewer A and reviewer B, independent and deduplicated, zero findings, clean, merged](/images/autopsy-of-an-agentic-loop/pr159-sweep-verdict.png)

The verdict the sweep left on the pull request: no findings, clean, merged.

About seven minutes from the push to the merge, and the base image now runs a verified Python. The old rule in my prompts said a runtime bump is always critical, because a reviewer can't see whether the dependencies still resolve. It is no longer a rule, because now something *does* see it.

### Case 2: my own pull request (PR #167)

A documentation change, "what to expect on your own pull request", opened from a branch like any human would.

- The engine reviewed the diff with the two reviewers and the documentation reviewer, and decided no existing doc was made wrong.
- It posted `Certified at 08fce29`.
- The merge pass merged it once the four checks were green on that commit.
- On `main`, the pipeline ran green end to end (build, image, vulnerability scan, SBOM, the kind cluster with the integration suite), and `changelog.yml` appended the entry to `CHANGELOG.md` by itself, from the pull request title.

![The agent's comment on PR #167: no blocking findings, the checks pass, certified at 08fce29, merges automatically once its CI is green](/images/autopsy-of-an-agentic-loop/pr167-own-pr-certified.png)

The comment the engine leaves is the certification: the sha it is bound to is in the sentence.

That last point is why the title matters. It is the changelog line and the only statement of intent the agent has.

### Case 3: a refactor that changes behaviour (PR #168)

I opened "refactor: simplify the fibonacci loop" with the text "no change in behaviour intended". The change moved the loop by one iteration:

```python
# before
for _ in range(n):
    a, b = b, a + b
# in the pull request
for _ in range(1, n):
    a, b = b, a + b
```

`/api/fib/10` now returned 34 instead of 55. Here is what the pipeline did, in order:

- `verify` ran the unit tests and two failed: `test_api.py::test_fibonacci` and, which I had not even thought of, `test_mcp.py::test_fibonacci`.
- The adjudicator classified **both as `code_defect`**, with the reason written in the comment: the tests passed on the base commit, and the description says no behaviour change was intended.
- The writer fixed the code, not the tests, about 90 seconds after the pull request was opened.
- Reviewers A, B and the final reviewer: 0 findings. Certified at the new commit. Merged.

![The agent's comment on PR #168: certified at a6e6204, and the verdict for each failing test, code_defect, with the reason](/images/autopsy-of-an-agentic-loop/pr168-code-defect.png)

The comment on the pull request carries the reasoning: which tests failed, who was wrong, and why.

Total: **about 5 minutes and $0.016**. On `main` the loop is back to `range(n)`, so the net change of the pull request is empty, and not one test file was touched. The test had the right to win, and it did.

### Case 4: a behaviour change on purpose (PR #169)

The opposite case, the one that decides whether the first rule is usable. I raised the maximum of `/api/sleep/{seconds}` from 10 to 30 seconds, wrote it in the title (`feat: allow sleeping up to 30 seconds`) and in the description (`11 to 30 seconds are now accepted instead of rejected`), and left the old test alone. That test asserts that `/api/sleep/11` answers 400.

The adjudicator returned this, taken from the run record:

```json
{"test": "tests/test_api.py::test_sleep_too_long[asyncio]",
 "classification": "test_defect",
 "confidence": "high",
 "intent_evidence": "This is an intended change of behaviour: requests above 30 seconds are still rejected with 400, but 11 to 30 seconds are now accepted instead of rejected.",
 "reason": "The test asserts /api/sleep/11 returns 400, but the stated intent explicitly says 11 to 30 seconds are now accepted; the diff changes the threshold from 10 to 30, so the test encodes the old behaviour."}
```

The quote is a literal substring of my description, so the code accepted the verdict. The test now moves to the new boundary, with the same assertion and nothing removed:

```diff
 @pytest.mark.anyio
 async def test_sleep_too_long(client):
-    resp = await client.get("/api/sleep/11")
+    resp = await client.get("/api/sleep/31")
     assert resp.status_code == 400
```

And the steward, running proactively on the changed application code, added the test for the new behaviour, with the sleep mocked so the suite doesn't wait 30 seconds:

```python
@pytest.mark.anyio
async def test_sleep_up_to_30_seconds_allowed(client, monkeypatch):
    async def _no_sleep(_seconds):
        return None

    monkeypatch.setattr("asyncio.sleep", _no_sleep)
    resp = await client.get("/api/sleep/30")
    assert resp.status_code == 200
    assert resp.json() == {"message": "Delayed by 30 seconds"}
```

![The agent's comment on PR #169: two commits pushed to the branch, certified at eb99465, tests added or updated in tests/test_api.py](/images/autopsy-of-an-agentic-loop/pr169-test-defect.png)

The agent's comment on this one says what it touched: only `tests/test_api.py`, and nothing in the documentation.

**About 8 minutes and $0.027**, merged. Same pipeline, same rule as case 3, opposite verdict. The difference is a sentence in the description, and the pipeline needs that sentence to be there.

### Case 5: a failure only the cluster can see (PR #171)

This is the case I was least sure about. "refactor: warm the Redis connection before counting" added a warm-up call to the counter endpoint, and the warm-up was an increment:

```python
async def count():
    # Touch the key first so the connection is warm before the value that is returned.
    await storage.redis_incr("hits")
    value = await storage.redis_incr("hits")
```

Every request now advanced the counter by 2. The unit tests don't see it, because without Redis the counter is `None`. Only the integration suite, running against real Redis in the cluster, asserts that two calls differ by one. The agent's own job can't start a cluster, so its local checks were green.

- CI went red on the integration suite.
- `agent-ci-failure` started from the workflow run, first asked whether the job had failed in the runner (no, a test had), then read the **real logs of the failed checks** and handed them to the engine.
- The fix **kept the purpose of the pull request**. It didn't delete the warm-up: it replaced the second increment with a read.

```python
async def count():
    # Warm the connection by touching the counter key first. The touch is a read,
    # so the returned counter still advances by exactly one per request.
    await storage.redis_get("hits")
    value = await storage.redis_incr("hits")
```

It added a small `redis_get` to the storage layer, two regression tests (`test_count_increments_by_one_per_request` and `test_count_warms_connection_before_incrementing`) that now catch this class of bug **without needing Redis**, and a line in the C4 components doc. I checked the diff afterwards: **0 lines removed from tests, 34 added**. ![The agent's comment on PR #171: two commits pushed, certified at 175a9fe, the new regression tests named in the notes, docs updated, model cost $0.0795](/images/autopsy-of-an-agentic-loop/pr171-cluster-only.png)

The notes name the regression tests it added and the documentation it updated.

Cost **$0.08**, about 11 minutes, merged.

### Case 6: a change nobody may repair (PR #170)

Not every pull request can be saved, and a pipeline with no person has to know when to stop. I opened "ci: retry the docs architect job on failure", which adds `retries: 3` to a job in a workflow file. GitHub Actions has no such key, and the workflow lint says so. Workflow files are the one thing the agents may never edit, because their token has no `workflow` scope, and I don't want it to have one: an agent that can edit the CI that controls it is not an agent I can leave alone.

Both reviewers raised it as blocking and said, correctly, that the fix is in a file they are not allowed to touch. The writer agreed in so many words ("that file is explicitly outside my allowed scope"). After the second attempt, with twice the budget, the engine did what the rule says:

- labelled the pull request `agent-abandoned`;
- wrote a comment with the findings and the reason;
- did **not** certify it. The `workflows` check stayed red, so nothing could merge it.

![The agent's comment on PR #170: abandoned after a second attempt with twice the budget, the two blocking findings and the notes of the writer](/images/autopsy-of-an-agentic-loop/pr170-abandoned.png)

The comment that ends it: the findings, the reason and the cost, with no certification.

About 4 minutes, **$0.029**. Because the abandonment is bound to the commit, the repair loop is not run again when the next CI event arrives; a new push by the author starts a fresh attempt, and a certification takes the label off. The pull request is mine, so it stayed open and unmerged. A pull request opened by an agent would have been closed.

### After the merge: the guard

The pipeline on `main` builds the multi-architecture image, scans it, produces the SBOM, deploys it in a cluster with PostgreSQL and Redis and runs the integration suite against the published image. If that goes red after a merge, the guard acts:

1. It re-runs only the failed jobs, once. On a real run the cluster failed to start; the second attempt passed, `main` was never touched, and the guard left a comment on the commit saying why.
2. If the failure repeats, it checks that nothing already repaired it forward, and that the failed jobs are ones a code change can cause. A vulnerability scanner turning red tomorrow is not a reason to revert today's commit.
3. Then it reverts everything since the last green run in a single commit (leaving the pipeline's own bookkeeping commits alone) and opens a work item so the pipeline redoes the change, this time with the failure in front of it.

I could have broken `main` on purpose to watch this end to end, but a broken `main` also leaves the image tag in the Helm values pointing at an image that doesn't exist. Instead the guard has a `--dry-run` that decides and prints without changing anything. I ran it on real runs of the repository, including the one whose cluster had failed to start, and it printed the decision the live guard would take, from real GitHub data.

## The results, side by side


| Case | What I opened | Verdict | Outcome | Time | Model cost |
| --------------- | ---------------------------------- | -------------------------- | ------------------------ | ------- | ---------- |
| #159 | Python base image, patch bump | reviewers: clean | merged by the sweep | ~7 min | n/a |
| #167 | Docs change, my own PR | certified | merged | n/a | n/a |
| #168 | Refactor that breaks behaviour | `code_defect` x2 | merged, net change empty | ~5 min | $0.016 |
| #169 | Intended behaviour change | `test_defect`, quoted | merged, test moved | ~8 min | $0.027 |
| #171 | Defect only visible in the cluster | `code_defect` from CI logs | merged, intent kept | ~11 min | $0.080 |
| #170 | Workflow edit, unrepairable | blocking, out of scope | abandoned, labelled | ~4 min | $0.029 |
| main, flaky job | cluster did not start | `environment` | re-run, nothing reverted | n/a | n/a |


Behind these there is a full test pyramid: 66 unit tests, 25 integration tests against real backends in the pull request checks and again against the published image in the cluster, and 364 tests on the engine itself (graph routing against real throw-away git repositories, the adjudication rules, the merge gate, the guard against a local bare remote, the terminal states).

## What it costs

A pull request that needs no repair costs a few cents: two reviewers and a documentation pass. The first review of a push to `main` cost $0.008. The repairs above came to between $0.016 and $0.08 in total, depending on how much the writer had to read before it was sure.

The more useful saving is attention. A pull request needs a person's eyes only when the pipeline abandoned it, and it says why in the comment.

## Reflections

I thought the hard part would be the model. It isn't. Case 3 and case 4 use the same model, the same prompts and the same code, and reach opposite conclusions about a failing test, because one description contains a sentence and the other doesn't. The model's job is small and well fenced, and the rest is a state machine, a handful of string comparisons and a lot of `git`.

The other thing I got wrong at the start was thinking of "needs human" as a safe default. It isn't. It is a state in which nothing happens, indefinitely, and in a system with no person it is the most dangerous state there is. The three terminal states exist so that every path ends with something done.

### What is still missing

I'd rather say it than have you find it:

- **Agents cannot edit `.github/workflows/`.** A pull request that needs a change in the CI itself stays a human job (or a job for Renovate, which has its own permission for action bumps). That is deliberate, and it is the one real dependency on a person that is left.
- **The guarantee is only as strong as the checks.** A defect none of the checks can see will merge. The guard limits the damage, it doesn't prevent it. A diff-coverage gate, so that every changed line must be exercised by a test, would raise the floor, and I haven't built it yet.
- **Two paths are covered by tests but not yet seen live:** a genuine revert for a genuine break on `main`, and the loop where a blocking finding on a Renovate pull request is handed to the writer. Both work against fakes and real git repositories; I simply haven't had a real occasion.
- **A vague description gives the model room to pick a side.** "Tests win" makes it predictable, but it will sometimes fix code that was right.

## Conclusion

A person used to be the thing that turned "the checks are green" into "this can merge", and "the checks are red" into "this should be fixed, and here is how". Both are now a graph: nine nodes, four entry points, three terminal states, and a handful of rules written in code instead of in someone's head.

The measure that matters to me is not the six cases. It is that in none of them did I do anything after pressing "create pull request".

If you want the surrounding ideas, I wrote about where a pipeline should and shouldn't use a model in [Card to Artifact](/posts/card-to-artifact-the-agentic-sdlc-pipeline-mechanism/), and about who builds software when agents write the code in [AI Agentic Development Changes Who Builds Software](/posts/ai-agentic-development-changes-who-builds-software-and-thats-an-infrastructure-problem/).