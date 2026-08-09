# AI Security Review Finds the Bug Your CI Gates Missed


### Table of Contents

  * The Problem
  * Naaa... the alternatives
  * The Pipeline
  * The ai-analysis Job, Commented
  * Downloading Results, Bundling the Source
  * The Empty Report
  * Downloading Artifacts, Watching the Run
  * The AI Scripts
  * The Report That Caught the Bug
  * The Full Results, as an Example
  * The Numbers
  * Security Considerations
  * Monitoring and Observability
  * Conclusion
  * Reflections

Here we are.

My debug/test API, `pytbak`, runs a proper CI: unit tests, linting, a Docker build, Trivy scanning, Checkov, a Kubernetes syntax check. Every gate maps to a category of failure I have seen before. And yet, I kept being bothered by one specific class of bug, the one that no scanner materializes out of thin air, because it lives in my logic and not in a CVE database.

Strange, how we run five deterministic checks and still push an inverted access control to production?

## The Problem

Security scanners are good at known unknowns. Trivy knows the CVE database, Checkov knows the Terraform/cloud misconfiguration patterns, `kubectl` knows my objects don't parse. Nothing knows that `if admin_token == OPERATOR_TOKEN: raise 403` is an inverted gate that hands the data to everyone except the operator.

That's a logical vulnerability. It bypasses controls and leaks data, and it has no CVE, because I wrote it five minutes ago.

Goals for the solution:

- Add a security review step that runs **after** all deterministic gates
- Feed it the artifacts those gates produce (Trivy, Checkov, k8s probe) as context
- Make it informative: it notes findings, it never blocks the pipeline on a whim
- Keep it at zero dollars per month

The last point matters. If the AI review costs money, it will be switched off the moment the manager notices. Free means it becomes a permanent fixture.

## Naaa... the alternatives

Making the AI job fail the build? Non-negotiable, no. A language model that decides whether to deploy, with its probability arithmetic? Na naa. An opinionated review, always available, clearly labeled as informative? That's the one.

So the constraint I set in the workflow:

```yaml
continue-on-error: true
if: always() && vars.AI_ENABLED == 'true'
```

The AI never has the last word on the release. It has the last read.

## The Pipeline

Let me show you the pipeline exactly as it is written, and comment every line, because the whole point is that the deterministic stages feed the AI stage.

The full file lives in `.github/workflows/pipeline.yml`. Top of the file, the name and the trigger:

```yaml
name: Python application

on:
  push:
    branches: [ "main" ]
  workflow_dispatch:

permissions:
  contents: write
  security-events: write
  actions: read
```

Nothing exotic: run on every push to main, and manually. I need `actions: read` in `ai-analysis` to download artifacts, and `contents: write` because a sibling job, `modifygit`, commits the new image tag back into `helm/values.yaml`. That job is a real footgun for rebases, more on that in the reflections.

The jobs before the AI are the classic chain, and each one produces a context artifact that the AI will eat later. Every context upload is guarded by the same two conditions:

```yaml
- name: Upload AI context (trivy scan)
  if: always() && vars.AI_ENABLED == 'true'
  uses: actions/upload-artifact@v4
  with:
    name: ai-context-trivy
    path: trivy-results.txt
    if-no-files-found: ignore
    include-hidden-files: true
    retention-days: 1
```

`if: always()` is not a typo of any kind. The scan failed? The AI should still hear about it. The trivy action writes its table to `trivy-results.txt`, and Checkov writes `results.sarif`, and the k8s probe writes `k8s-probe.txt`. Three artifacts, three separate upload steps, all prefixed `ai-context-` so the AI job can pull them with a single pattern.

The hardest part to get right was the k8s probe. The old step was a bare `curl -I` that succeeded and proved nothing, and worse, `curl -I http://localhost:8000/api/` returns `405 Method Not Allowed`, so even the HTTP code lied. I replaced it with a probe that hits a real readiness endpoint and requires a 2xx:

```yaml
- name: Probe the deployed app
  run: |
    set +e
    echo "-- Health probe: GET /api/mgmt/ready (expects HTTP 2xx) --" > k8s-probe.txt
    code=000
    for i in $(seq 1 10); do
      code=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/api/mgmt/ready)
      echo "attempt $i: HTTP ${code}" >> k8s-probe.txt
      if [[ "$code" =~ ^2 ]]; then break; fi
      sleep 5
    done
    echo "final HTTP code: ${code}" >> k8s-probe.txt
    kubectl get po -o wide >> k8s-probe.txt 2>&1
    kubectl describe po kube-check >> k8s-probe.txt 2>&1
    if [[ ! "$code" =~ ^2 ]]; then
      echo "::error::app did not return 2xx on readiness within retry budget (got ${code})"
      exit 1
    fi
```

Notice the retry loop with a 2xx check. The probe still writes its evidence (`k8s-probe.txt`: HTTP codes per attempt, pod status, describe output) for the AI to read, but now a non-2xx also fails the gate. It's the difference between "I placed a call" and "I have evidence".

## The ai-analysis Job, Commented

Here's the whole job, exactly as written in the file. I'll comment it below it.

```yaml
ai-analysis:
  needs: [build, docker, security-gate-trivy, docker-sbom, quality-gate, modifygit, k8s-check]
  if: always() && vars.AI_ENABLED == 'true'
  runs-on: ubuntu-latest
  continue-on-error: true
  permissions:
    contents: read
    actions: read
  env:
    OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
    OPENROUTER_MODEL: ${{ vars.OPENROUTER_MODEL }}
    OPENROUTER_ENDPOINT: ${{ vars.OPENROUTER_ENDPOINT }}
  steps:
    - name: Checkout
      uses: actions/checkout@v4
    - name: Download AI context artifacts
      uses: actions/download-artifact@v4
      with:
        path: .ai/
        pattern: 'ai-context-*'
        merge-multiple: true
        if-no-files-found: warn
    - name: Sanitize contexts and build prompt
      run:
        find app -name '*.py' -print0 | sort -z | xargs -0 python3 scripts/ai_sanitize.py --out .ai/source-context.txt --max-bytes 40000
        python3 scripts/ai_sanitize.py --out .ai/security-context.txt --max-bytes 120000 .ai/lint-errors.txt .ai/tests.txt .ai/source-context.txt .ai/trivy-results.txt .ai/results.sarif .ai/k8s-probe.txt
    - name: Create AI analysis prompt
      run: |
        cat > .ai/analysis-system.txt <<'EOF'
        You are a QA/DevOps/security analyst. You are given the CI output of a full
        pipeline (lint, tests, container scan, helm/IaC checks, K8s probe) and the
        application source code under `app/`. ALL the data you need is in the
        prompt below. Treat it as ground truth: do not guess, compute, or hedge.
        RULES:
        - Be direct and concise. Output the report only, no chain-of-thought.
        - Cite evidence for every claim: exact test counts, file paths and line
          numbers for findings, CVE ids and Checkov rule ids as printed.
        - Pointed suggestions: for each issue give a concrete fix:
          "in <file>:<line>, change X to Y because <reason>". No abstract advice.
        - No hedging: ban words like "I think", "likely", "maybe", "let me count",
          "I am not sure", "could be". If a claim is not verifiable, skip it or
          mark it as an assumption in one line; never invent numbers.
        - Confidence: if a claim is not verifiable from the prompt, skip it or mark
          it explicitly as an assumption (one line only).
        Produce a technical report in Markdown with: **Summary** with exact counts,
        **High severity issues** each with file:line and the minimal fix,
        **Failures / flaky tests**, **Security review** (auth bypasses, inverted
        conditions, missing access control, hardcoded secrets, IDOR, SSRF, trusting
        client input, data exposure through debug endpoints),
        **Code style / maintainability issues**, **False positives**,
        **Concrete suggestions** (numbered, file:line + exact change).
        The report is INFORMATIVE only - it never changes the pipeline result.
        EOF
    - name: Run AI analysis
      run: |
        python3 scripts/openrouter_ai.py \
          --system-file .ai/analysis-system.txt \
          --prompt-file .ai/analysis-user.txt \
          --max-chars 150000 \
          --max-tokens 32000 --timeout 300 \
          --usage-file .ai/usage.json \
          > .ai/ai-report.md
    - name: Append token usage & cost
      run: python3 scripts/ai_append_cost.py --usage-file .ai/usage.json --report-file .ai/ai-report.md
    - name: Mask secrets in report before upload
      run: |
        python3 scripts/ai_sanitize.py --redact-file .ai/ai-report.md \
          >/dev/null 2>&1 || echo "::warning::AI report contained secret-like patterns, they have been masked"
    - name: Show AI analysis in pipeline summary
      run: |
        echo "::group::AI Analysis Report"
        cat .ai/ai-report.md
        echo "::endgroup::"
        { echo "## AI Analysis Report"; cat .ai/ai-report.md; } >> "$GITHUB_STEP_SUMMARY"
    - name: Upload AI analysis report
      if: always() && env.SKIP != 'true'
      uses: actions/upload-artifact@v4
      with:
        name: ai-analysis-report
        path: .ai/ai-report.md
        if-no-files-found: ignore
        retention-days: 7
```

Now the commentary, line by line.

`needs: [build, docker, security-gate-trivy, docker-sbom, quality-gate, modifygit, k8s-check]`

This is the ordering I insisted on. The AI job declares a dependency on every deterministic stage, so GitHub only starts `ai-analysis` when all of them have finished, whatever result they produced. The AI always has their artifacts on disk, because the `download-artifact` step runs before anything else.

`if: always() && vars.AI_ENABLED == 'true'`

`always()` is not negotiable: I want the review even when a gate failed, and even when the previous jobs all succeeded. `vars.AI_ENABLED` is the kill switch to turn the whole thing off without touching the file.

`continue-on-error: true`

The AI job never fails the workflow. It runs at the end, with `always()`, it is the last word. If it errors, the result is a red cross in the job list and nothing else.

`merge-multiple: true`

The downloader flattens all the `ai-context-*` archives into one directory, `.ai/`, whatever their names. Then the sanitizer script, `scripts/ai_sanitize.py`, trims both outputs to a fixed budget (source code 40 KB, everything 120 KB) and drops anything that looks like a secret. 220 KB of text, all consumed by the prompt. Nothing from the build is read as content, the sanitizer runs first.

`max-tokens`
Here's the dirty trick, and the whole reason this post got ugly for me in the middle. The first version used `--max-tokens 8192`. That value, combined with a reasoning model, produced the empty report I describe below. The current file uses `--max-tokens 32000` plus `--max-chars 150000`. Don't copy a tight number without reading that section first. Keep reading.

The system prompt is explicit that the report must include a **security review** section. That's the section that saved us, the one that separates `ai-analysis` from the "smart linter" mindset.

`Mask secrets in report before upload`

No secrets escape the job. Since the model receives source code, the model can echo it back. The sanitizer runs again with `--redact-file` on the generated report: it masks each hit in place and exits 1 when something was found. Because it rewrites instead of deleting, a report that quotes source code (like `password = "x"`) is still shown and uploaded, just with the value redacted.

`Show in pipeline summary` and `Upload`

I want the report in two places: the job summary of the run (`GITHUB_STEP_SUMMARY`, so it's visible in the Actions tab without opening the artifact) and as a downloadable file `ai-analysis-report.md`.

The mermaid for the flow:

{{< mermaid >}}
flowchart LR
  A[git push] --> B[build, quality-gate, trivy]
  B --> C[checkov, k8s probe]
  C -->|gate results| D[ai-analysis runs last]
  D --> E[informative report]
  F[source app code] --> D
{{< /mermaid >}}

## Downloading Results, Bundling the Source

The single cleverest line of the whole job is this one, short and simple:

```bash
find app -name '*.py' -print0 | sort -z | xargs python3 scripts/ai_sanitize.py \
  --out .ai/source-context.txt --max-bytes 40000
```

I bundle the entire `app/` source tree into a text file, capped at 40 KB. Why? Because the AI can't reason about a bug in a file it can't read. Trivy and Checkov and `kubectl` test for known patterns, but nothing scans the logic of my own code, and that logic lives in `app/`. Include it, and the security review has something to chew.

If that file doesn't exist because the job failed early, the script fails and I remove the marker, with `|| rm`, so the AI still sees an empty context instead of a hard failure.

Then the lint and test outputs, the gate results, and the source all go into one prompt file:

```bash
python3 scripts/ai_sanitize.py \
  --out .ai/security-context.txt --max-bytes 120000 \
  .ai/lint-errors.txt .ai/tests.txt .ai/source-context.txt \
  .ai/trivy-results.txt .ai/results.sarif .ai/k8s-probe.txt ||
  rm -f .ai/security-context.txt
```

One artifact. One prompt. All the CI output and the source, crammed into the same model call.

## The Empty Report

Here's where the story takes its turn, and I mean the negative one.

The first run of the whole pipeline was green. Happy path. I opened the `ai-analysis-report` artifact and it was empty. Not a short sentence, not a "no issues found", empty: the file was there, the step exited fine, and the report would have been zero bytes.

What made me crazy was the cause. **My model is a reasoning model**. It spends its output budget on a chain of thought first, then on the answer. With `max_tokens: 8192`, that budget was a pool for the chain of thought, and the `content` field that the script read for the report came back empty.

So the real bug was an expectation, my assumption that a model fills up to `max_tokens` with content. An LLM can legally answer with an empty response, by spending everything on reasoning, and my pipeline wouldn't have noticed.

Three mitigations, in the order the script now applies them:

1. **retry with a nudge first**: on a blank reply, retry up to N times with a prompt that says, answer directly, with `max_tokens` roughly doubled on every attempt (capped at the model's 128000 output ceiling)
2. **retain reasoning only as a last resort**: the OpenRouter response exposes `reasoning_content`; only if retries still return blank, the script saves the reasoning so the review is never silent
3. **a healthy starting budget**: run the analysis with `--max-tokens 32000` (instead of the original 8192) so the reasoning can't normally starve the answer in the first attempt

```python
if not content.strip():
    for attempt in range(1, args.retries + 1):
        budget = min(args.max_tokens * (2 ** attempt), MODEL_MAX_OUTPUT_TOKENS)
        nudge = (system + "\n\nIMPORTANT: You responded with an empty answer. Reply "
                 "with the report directly - do not spend tokens on reasoning.")
        content, reasoning, usage = _request(... max_tokens=budget)
        if content.strip():
            break

# last resort: the model really spent everything on thinking
if not content.strip() and reasoning.strip():
    content = reasoning
```

The script treats a genuinely empty reply as an error, not a success. That's the difference between a guard and a failure already masked. And because the fallback to `reasoning_content` comes last, an empty answer is retried instead of silently piping chain-of-thought into the artifact.

## Downloading Artifacts, Watching the Run

This run, the one that finally produced the report, is the one in the screenshot. The pipeline shows all the jobs, from `build` through the AI, and its green after my change:

![Pipeline run with the AI job green](/images/ai-security-review-finds-the-bug-ci-gates-missed/01-pipeline-jobs-and-docker-summary.png)

The screenshot was taken from the run. Aside from the jobs, it shows a side-effect I didn't plan: StepSecurity's Hardened Runner auditing the network egress of every step. Not useful for the report, but a good habit for anything that talks to an external LLM API.

## The AI Scripts

All the logic above lives in three stdlib-only Python scripts that sit next to the workflow. Here they are, because the pipeline is only as good as their edge cases.

The first is `scripts/ai_sanitize.py`, the one that builds the context and guards the report. It has three modes: `bundle` (redact + cap inputs), `check` (exit 1 if secrets found), and the newer `redact-file` (rewrite the file in place masking each secret, exit 1 if any was found). The report step uses `redact-file`: when the model quotes `password = "x"`, the report is not dropped, the hit is masked and the artifact is still uploaded.

```python
#!/usr/bin/env python3
"""Sanitize the size and secrets of CI output before it reaches an LLM.
bundle  -- python3 ai_sanitize.py --out FILE --max-bytes N FILE...
           redact secrets, prepend a header per file, truncate to N bytes.
check   -- python3 ai_sanitize.py --check FILE
           exit 1 if FILE contains any detectable secret pattern.
redact-file -- python3 ai_sanitize.py --redact-file FILE
           rewrite FILE masking every secret; exit 1 if any hit was found.
"""

import argparse, re, sys

# Credentials and tokens that are always real secrets no matter the context.
_HARD_SECRET_RE = [
    re.compile(r"sk-or-v1-[A-Za-z0-9\-_]{10,}"),
    re.compile(r"Bearer\s+[A-Za-z0-9\-_.]{20,}", re.IGNORECASE),
    re.compile(r"ghp_[A-Za-z0-9]{20,}"),
    re.compile(r"github_pat_[A-Za-z0-9_]{20,}"),
    re.compile(r"AKIA[0-9A-Z]{16}"),
    re.compile(r'(?i)-----BEGIN [A-Z ]*PRIVATE KEY-----.*?-----END [A-Z ]*PRIVATE KEY-----', re.DOTALL),
    re.compile(r"\beyJ[A-Za-z0-9_\-]{20,}\.[A-Za-z0-9_\-]{10,}\.[A-Za-z0-9_\-]{10,}\b"),  # JWT
]

# Config-style assignments: legit as documentation/report context, still masked.
_SOFT_SECRET_RE = [
    re.compile(r"(?i)(DATABASE_URL|REDIS_URL|WEBHOOK_URL|MONGO_URL|MYSQL_URL|AMQP_URL|PASSWORD)\s*=\s*['\"][^'\"]+['\"]"),
]

_SECRET_RE = _HARD_SECRET_RE + _SOFT_SECRET_RE


def redact(text: str) -> str:
    for pattern in _SECRET_RE:
        text = pattern.sub("***REDACTED***", text)
    return text


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--check", action="store_true")
    parser.add_argument("--redact-file")
    parser.add_argument("--out")
    parser.add_argument("--max-bytes", type=int, default=150_000)
    parser.add_argument("files", nargs="*")
    args = parser.parse_args()

    if args.redact_file and args.check:
        parser.error("--redact-file and --check are mutually exclusive")

    if args.redact_file:
        with open(args.redact_file, encoding="utf-8", errors="replace") as fh:
            original = fh.read()
        redacted = redact(original)
        if redacted != original:
            with open(args.redact_file, "w", encoding="utf-8") as fh:
                fh.write(redacted)
            return 1
        return 0

    if args.check:
        with open(args.files[0], encoding="utf-8", errors="replace") as fh:
            content = fh.read()
        return 1 if redact(content) != content else 0

    parts, total = [], 0
    for path in args.files:
        try:
            with open(path, encoding="utf-8", errors="replace") as fh:
                content = fh.read()
        except OSError:
            continue
        block = f"### {path} ({len(content)} chars)\n{redact(content)}\n"
        parts.append(block)
        total += len(block)
        if total >= args.max_bytes:
            break
    if not parts:
        print("error: no files produced content", file=sys.stderr)
        return 1
    with open(args.out, "w", encoding="utf-8") as fh:
        fh.write("\n".join(parts)[: args.max_bytes])
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

The second script, `scripts/openrouter_ai.py`, is a stdlib-only OpenAI-compatible client. Two details matter. First, the reply is untrusted data, so every secret in the output is masked again, including the API key, defense in depth. Second, it handles the reasoning-model trap: if a reasoning model returns a blank `content`, the script retries with a growing budget before it ever falls back to `reasoning_content`. The reasoning is the last resort, not the default, otherwise you get chain-of-thought instead of a report.

```python
def _extract_content(message: dict) -> str:
    content = message.get("content")
    if isinstance(content, str):
        return content
    if isinstance(content, list):
        parts = []
        for part in content:
            if isinstance(part, str):
                parts.append(part)
            elif isinstance(part, dict) and isinstance(part.get("text"), str):
                parts.append(part["text"])
        return "".join(parts)
    return ""

def _extract_reasoning(message: dict) -> str:
    reasoning = message.get("reasoning") or message.get("reasoning_content")
    if isinstance(reasoning, str):
        return reasoning
    if isinstance(reasoning, list):
        parts = []
        for part in reasoning:
            if isinstance(part, str):
                parts.append(part)
            elif isinstance(part, dict):
                text = part.get("text") or part.get("content")
                if isinstance(text, str):
                    parts.append(text)
        return "".join(parts)
    return ""
```

And the retry-with-a-nudge contract in `main()`. A blank `content` means the reasoning model burned its budget on chain-of-thought, so we retry with a doubled budget (capped at the model's output ceiling), then fall back to the reasoning text only if that still fails:

```python
if not content.strip():
    for attempt in range(1, args.retries + 1):
        budget = min(args.max_tokens * (2 ** attempt), MODEL_MAX_OUTPUT_TOKENS)
        nudge = (system + "\n\nIMPORTANT: You responded with an empty answer. Reply "
                 "now with the complete report directly - do not spend tokens on reasoning.")
        content, reasoning, usage = _request(... max_tokens=budget)
        if content.strip():
            break

# last resort: the model really spent everything on thinking
if not content.strip() and reasoning.strip():
    content = reasoning
```

The library also computes the price: it reads `usage`, applies the per-1M price, and writes a `usage.json`. A tiny third script, `ai_append_cost.py`, appends that as a Markdown footer to the report:

```python
section = (
    "\n\n---\n\n## AI Usage & Cost\n\n"
    f"- **Tokens**: {total:,} total (input: {prompt:,}, output: {completion:,})\n"
    f"- **Cost**: {cost_badge}\n\n_{cost_line}_\n"
)
```

Since the free model is $0, the price is reported against the commercial counterpart, so every report shows a real economic value instead of "free".

## The Report That Caught the Bug

Here's the proof. The report the AI produced, and it swallowed the injected bug whole, citing the exact line. The summary table at the top, as it appeared in the report:

| Stage | Result |
|---|---|
| flake8 | clean, 0 lint errors |
| pytest | 77 passed, 25 skipped, 0 failed |
| Container scan (Trivy) | 4 CRITICAL (Debian base OS packages) |
| IaC scan (Checkov) | 8 findings (6 unique rules) in the Helm chart |
| K8s probe | Pod Ready, but HTTP probe returned `405 Method Not Allowed` |

And on the same page, the finding I was waiting for:

> Inverted authorization on `/api/internal-summary`: the `admin_token==` operator token comparison raises `403` for the valid token, so any other caller is granted an auth bypass, and the legitimate operator is locked out.

With the code cited:

```python
if admin_token == _OPERATOR_TOKEN:
    raise HTTPException(status_code=403, detail="Operator token required")
```

And the punchline, in the same report:

> The condition is inverted: every caller except the legitimate token holder gets the internal summary. The hardcoded token 'changeme-operator-token-9f4e2a7b' is embedded in source. Two tests codify the broken behavior.

The model also triaged, in the same report:

- the default `admin/password` credentials from the settings
- the unauthenticated error-injection middleware, a DoS risk
- the Trivy findings for the Debian base image
- the Kubernetes probe that the pod was ready, but the HTTP probe returned `405 Method Not Allowed`

![AI report, high severity and summary](/images/ai-security-review-finds-the-bug-ci-gates-missed/02-ai-report-part1.png)

![AI report, code quality and false positives](/images/ai-security-review-finds-the-bug-ci-gates-missed/03-ai-report-part2.png)

And the context artifacts downloaded from all the gate jobs:

![The gate context artifacts](/images/ai-security-review-finds-the-bug-ci-gates-missed/04-artifacts-context.png)

It wasn't only the exact finding I was after. The report ran a full sweep: SQLi, mass assignment, SSRF, hardcoded tokens, DoS paths, and it flagged a "False positives" section for code that looked alarming but was safe. That section is the sign of a mature reviewer: it distinguished the actual bug from the noise.

## The Full Results, as an Example

Enough summaries. Here's the report exactly as the model produced it in run 31276065009. I'm giving you the whole thing, structure and severity levels, because this is what you actually get back from the pipeline, not a curated highlight.

**High severity issues (first page of the report):**

> **Inverted authorization on `/api/internal-summary`: auth bypass + operator lockout** (`app/routers/api.py`)
>
> ```python
> if admin_token == _OPERATOR_TOKEN:
>     raise HTTPException(status_code=403, detail="Operator token required")
> ```
>
> The condition is inverted: every caller except the legitimate token holder gets the internal summary; the real operator is denied. The hardcoded token `changeme-operator-token-9f4e2a7b` is embedded in source. Two tests codify the broken behavior, so it will persist until the tests change too.

The model went further than the obvious line. Second finding, default credentials in `app/config/settings.py`:

```python
diag_username = "admin"
diag_password = "password"
debug_endpoints_enabled = True
```

Third finding, the unbounded delay in `app/middleware/error_injection.py`:

```python
delay_ms = float(request.query_params.get("delay_ms", 0))  # no upper bound, e.g. ?delay_ms=999999
```

Then the sweep, the section that separates a linter from a reviewer.

```markdown
- **Trusting client input**: `delay_ms` / `inject_error` are accepted from the query string globally, before any auth. `target`/`host`/`port` are validated mostly by regex, but the curl URL stays unfiltered, SSRF.
- **Mass assignment**: absent. Pydantic models whitelist fields; `ContextUpdate` correctly extends with `done: false`.
- **Data exposure**: `/api/internal-summary` leaks env, the debug flag, the webdis URL and the in-memory store size to any caller. `/metrics` is exposed unauthenticated.
- **DoS**: `cpu_spike` (120s x cores) is reachable; `sleep` accepts negative seconds, `asyncio.sleep(-1)` raises -> 500. ErrorInjectionMiddleware catches everything and turns it into a generic 500, hiding real failures.
- **Secrets in transit**: HTTP Basic over plain HTTP, base64 is not encryption, unless the proxy terminates TLS upstream.
```

The rest of the report continues the sweep and the "technical debt" checklist. It also had a **False positives/noted as safe** section, which is what made me trust it beyond the demo:

```markdown
- `MCP/mgmt` env filtering is safe by design, the test confirms it.
- ErrorInjection catching broadly is intended for a debug app; the real issue is that the unbounded pre-auth delay, not the try/except.
- storage._memory_backend private access is a smell, not a vulnerability.
```

The full report of the run is in the screenshots above: the high-severity page, and the final page with the remediation guidance. It doesn't just list findings, it ends with concrete suggestions, "fix `internal_summary` immediately", "make the diag credentials required", "cap the middleware delay". That's the part that turns a scan into a review.

## The Numbers

- The first run with the tight `--max-tokens 8192` returned an empty report
- After raising the starting budget to `--max-tokens 32000` and adding the retry-with-nudge as the primary guard (reasoning retained only as a last resort), the same pipeline produced the full report with the high-severity finding and the code excerpt

One change over the other, the model went from silent to finding the deliberately planted bug. That's the whole point of the experiment.

## Security Considerations

1. Treat model output as data, it's never a gate. `continue-on-error: true` and `vars.AI_ENABLED` are deliberate.
2. Sanitize the context before it leaves the runner, and sanitize the report before it lands on the bucket
3. `actions: read` is enough for artifacts: nobody needs `contents: write` in this job
4. The `reasoning_content` fallback can leak chain-of-thought in the report. The current script keeps it as the last resort, but verify what lands in the artifact and periodically run a report whose length is suspicious
5. A green "informative" job is exactly as reliable as the model. The policy gates it, and the pipeline does not act on it.

## Monitoring and Observability

The step also reports token usage and estimated cost. Since a free tier model is at `$0`, I tie the price to the commercial counterpart so every report shows a true economic value, even when the provider bills zero.

## Conclusion

A logical vulnerability, no CVE, in your own code, is the one thing that static scanners and container scans can't find. A tiny, free, always-on second reviewer, that consumes the whole pipeline's output at the end, caught mine the first time I let it read the source. It didn't invent it, it read the inverted operator. The secret is that the AI reads everything the gate already read, plus the source, and it writes the report as an informative artifact, not as a gate that decides whether to deploy.

## Reflections

The messy part: the first version of this job was empty for a silent reason. "The pipeline is green" and "the report is empty" can coexist without any error, and a report that is zero bytes is a failure, not a quirk. It's now the first thing I check in any green run.

What is still missing? A lot: I don't score the model's recall and precision on the injected vulnerabilities, there's no regression harness that plants a bug and checks that the review names it. And the `modifygit` job that re-pushes to main broke my rebases, so the pipeline history is anything but linear. For now, it's an honest pair of eyes that signs every report, and I won't remove it.

Related reading:

- [Python REST API Test Application](/posts/python-rest-api-test-application/)
- [Monitoring Contentful Usage with a Prometheus Exporter](/posts/monitoring-contentful-usage-with-a-prometheus-exporter/)
