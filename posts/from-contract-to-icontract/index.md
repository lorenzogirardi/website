# From Contract to iContract: Turning a Platform PDF Into a Skill, and the Gate That Makes It Stick

### Table of Contents

  * Introduction
  * The Problem: A PDF Is Not an Interface
  * The Architecture: From Contract to Skill
  * Part One: What the App Must Be
    * Example 1: A Health-Check Clause Becomes Four Endpoints
    * Example 2: "No Root" Becomes a securityContext Block
    * Example 3: "No Persistent Data in a Container" Becomes Three Manifests
    * Example 4: A Four-Layer Image Hierarchy Becomes a Dockerfile and a Tag Strategy
  * Part Two: What the App Must Say
    * Example 5: "Logs MUST Be JSON on stdout" Becomes a Logger and a Field Contract
    * Example 6: A Three-Tier Metrics Rule Becomes an Annotation and a NetworkPolicy
    * Example 7: "MUST Send Traces to a Collector" Becomes an SDK Init and an Egress Policy
    * Example 8: "Alert on Trends, Not Numbers" Becomes a Recording Rule and an HPA
  * Part Three: Where the App Lives
    * Example 9: A Naming Convention Becomes a Derivation Rule
    * Example 10: "Never Commit a Plain Secret" Becomes Vault or AWS Secrets Manager
    * Example 11: "The Management Port MUST NOT Be Public" Becomes Three Absences
  * Part Four: How the App Ships
    * Example 12: "No Human Tampering With the Artifact" Becomes a CI Workflow With Teeth
    * Example 13: Terraform Guardrails Become a Module Call You Cannot Widen
    * Example 14: "MUST Be in the Service Catalog First" Becomes Labels and a Generated Entry
  * The Real Speedup: Building Something the Contract Never Anticipated
  * How to Make Skills Actually Respected: The Gate
  * From a Working App to a Platform Citizen
  * Does the AI Actually Comply?
  * Security Considerations
  * Download the Skills
  * Conclusion
  * Reflections



Well. Every platform team eventually writes the same document: a PDF (or a Confluence page pretending to be one) that says what an application must do before it's allowed to run on the shared cluster. Containerized. Non-root. Health endpoints on a specific port. No plain secrets in Git. It's correct, it's thorough, and almost nobody who needs to follow it has actually read it end to end.

I had six of these documents sitting across a platform contract and five guideline papers: containerization and health probes, GitOps and Terraform module conventions, DNS and namespace naming, a three-tier monitoring philosophy, a four-layer Docker image hierarchy, and a security/SBOM policy with a RACI matrix nobody outside the platform team could recite. Between them, a genuinely well-designed set of rules. In practice, the thing developers actually consulted was whoever answered fastest in the support channel.

So I ran an experiment: turn every MUST, SHOULD, and MAY in those six PDFs into an **iContract**, a set of AI skills that generate compliant code on request and gate deploys when the code doesn't comply. Not a summary of the contract. Not a chatbot that can quote the contract back to you. An artifact the model produces that structurally mirrors what the contract demands, in the language the contract was actually written for: Dockerfiles, Kubernetes manifests, health endpoints, log formatters, Terraform module calls.

This post walks through the transformation pipeline, fourteen concrete before/after examples pulled straight from the source contracts (each shown as the rule as written next to the code the skill actually generates from it), why this approach pays off most for the implementation nobody in the organization has done before, how the enforcement side actually blocks bad deploys instead of just describing good ones, and an honest look at where AI compliance breaks down even when the pipeline works exactly as designed.

## The Problem: A PDF Is Not an Interface

A platform contract has two audiences, and it serves neither one well. For a human developer, it's too long to read before the first deploy and too easy to forget after. For a machine, in the classic sense, it's not machine-readable at all: no schema, no linter, no CI check that fails the build because paragraph four, section two, said port 8082 is mandatory.

The result is a predictable failure mode. A developer builds an app, it works locally, they copy a Kubernetes manifest from a colleague who copied it from someone else two years ago, and it goes to the platform team for review. Every violation the review catches, missing liveness probe, a plain `Secret` committed to Git, a `:latest` tag, is a violation the contract already described. The contract wasn't wrong. It was inert.

![Why PDF contracts fail: the execution gap between read, recall, translate, validate and audit](/images/from-contract-to-icontract/why-pdf-contracts-fail.jpg)

The gap has five distinct stages and only the first one is about reading. A developer has to read the PDF, remember port 8082 weeks later, translate an abstract requirement into their own framework, validate it with no automated check available, and then survive an audit months after the tech debt already shipped. Six separate documents with no single source of truth, and no trigger that tells anyone which one applies right now.

That's the gap I wanted to close, and it maps onto a familiar quadrant: the contract itself is high-importance, low-urgency work. Nobody's pager goes off because a PDF exists. It just quietly costs the platform team review cycles, one Slack thread at a time, for as long as the gap between "documented" and "enforced" stays open.

## The Architecture: From Contract to Skill

The transformation is an eight-step pipeline, and every step earns its place because skipping it produces a worse artifact.

{{< mermaid >}}
flowchart TD
    A[Contract PDF] -->|1 PARSE| B[RFC 2119 verbs: MUST, SHOULD, MAY]
    B -->|2 CLUSTER| C[Group by domain, not by source doc]
    C -->|3 EXPAND| D[Per-stack templates: Python, Node, Go, Java]
    D -->|4 CONSTRAIN| E[MUST-NOT as model invariants, stated before code]
    E -->|5 SEPARATE| F[Generate skill: proactive]
    E -->|5 SEPARATE| G[Gate skill: reactive]
    F -->|6 LINK| H[Cross-referenced by check ID]
    G -->|6 LINK| H
    H -->|7 INJECT| I[Platform variables resolved per cluster]
    I -->|8 GATE| J[Checklist mapped to review check IDs]
{{< /mermaid >}}

**Parse.** Every MUST becomes a CRITICAL check that blocks a deploy. Every SHOULD becomes a WARNING that's reported but doesn't block. Every MAY gets documented and left alone. This single distinction, lifted straight from RFC 2119, is what turns prose into severity levels a gate can actually act on.

**Cluster by domain, not by source document.** The six PDFs didn't align cleanly. Health endpoints and probe timeouts lived in the platform contract. Image layering lived in the Docker images paper. Namespace naming lived in a DNS conventions paper nobody thought to cross-reference against the health endpoint requirements. The skill layer re-clusters everything by what a developer is actually trying to do: "add health endpoints" pulls from three different source PDFs into one coherent step.

**Expand per stack.** A contract clause like "the app must expose a Prometheus-format `/metrics` endpoint" is stack-agnostic on paper and stack-specific in practice. The skill carries a decision table (file presence signals) mapped to concrete implementations for Python/FastAPI, Node.js/Express, Go, and Java/Spring, so the model never has to improvise which client library to reach for.

**Constrain.** This is the step that's easy to skip and expensive to skip. MUST-NOT clauses get written as explicit model invariants, positioned *before* any code in the skill file, not buried in a caveat at the end. "Never use alpine base images" isn't a footnote, it's a rule the model reads before it writes the first `FROM` line. Position matters more than phrasing here: a constraint stated after the example gets treated as a suggestion.

**Separate generate from gate.** This is the structural decision that makes enforcement possible at all, and a later section is built entirely around it.

![The doc-to-skill pipeline: parse, cluster, expand, constrain, separate, cross-link, inject platform variables](/images/from-contract-to-icontract/doc-to-skill-pipeline.jpg)

The same seven steps written out with the reasoning attached to each one. Step four is the one I'd defend hardest: "NEVER generate patches" as an invariant prevents partial compliance that looks correct in a diff and isn't deployable as a file.

The output of the pipeline is a Markdown file with a fixed anatomy, and the anatomy matters as much as the content:

![Anatomy of an iContract skill: trigger, purpose, invariants, decision tree, executable templates](/images/from-contract-to-icontract/anatomy-of-a-skill.jpg)

Trigger first (so the agent knows when to load it), purpose second, invariants third and always before any code, then the stack decision table, then the executable templates. Reorder those five blocks and the same file produces measurably worse output.

Every source document ends up mapped to one or more skills, with an explicit enforcement mode per document:

![Contract to skill coverage matrix: each source PDF mapped to target skills and enforcement mode](/images/from-contract-to-icontract/coverage-matrix.jpg)

Three modes fall out of the mapping: *generate and gate* (the skill writes it and the gate checks it), *generate only* (enforcement happens through scaffolding, nothing to audit afterwards), and *gate only*, which is the interesting one. The SBOM and supply-chain document produces nothing a developer asks for, it only produces reasons to say no.

What follows is fourteen clauses, in four groups, each one shown as the contract sentence and then the artifact the skill produces from it. The groups are the domains the skills re-clustered into: what the app must *be*, what it must *say*, where it must *live*, and how it must *ship*.

## Part One: What the App Must Be

### Example 1: A Health-Check Clause Becomes Four Endpoints

**The contract says**, paraphrased from the platform PDF:

> The application MUST expose a management interface on a dedicated port, separate from application traffic, providing readiness and liveness signals. This port MUST NOT be publicly routable and MUST NOT be under authentication. Every long-living application MUST provide endpoints for lifecycle management, exposed over HTTP 1.1.

**The skill writes.** That one paragraph becomes a table plus four implementations:

| Endpoint | Method | Response 200 | Response 503 |
|----------|--------|---------------|---------------|
| `/readiness` | GET | `{"status":"UP"}` | `{"status":"OUT_OF_SERVICE"}` |
| `/liveness` | GET | `{"status":"UP"}` | `{"status":"DOWN"}` |
| `/info` | GET | `{"git":{"commit":{...}},"app":"<slug>"}` | - |
| `/shutdown` | GET | `null` | - |

And the Python implementation the skill generates when it detects `fastapi` in the repo:

```python
# mgmt_server.py: runs on port 8082, separate from main app
import os, uvicorn
from fastapi import FastAPI

mgmt = FastAPI()

@mgmt.get("/readiness")
def readiness():
    # Add real checks here: DB connection, required services
    return {"status": "UP"}

@mgmt.get("/liveness")
def liveness():
    return {"status": "UP"}

@mgmt.get("/info")
def info():
    return {
        "app": os.getenv("APP_SLUG", "unknown"),
        "git": {"commit": {"id": {"full": os.getenv("GIT_SHA", "dev")}},
                "branch": os.getenv("GIT_BRANCH", "unknown")},
    }

@mgmt.get("/shutdown")
def shutdown():
    return None

if __name__ == "__main__":
    uvicorn.run(mgmt, host="0.0.0.0", port=8082)
```

The `/info` payload is the part most likely to get skipped by a human reading the PDF, because the contract specifies it as a nested `git.commit.id.full` structure buried on page eleven. The skill treats it as non-optional and wires the values in at build time, which means the Dockerfile gets an ARG the developer never had to think about:

```dockerfile
ARG GIT_SHA=dev
ARG GIT_BRANCH=unknown
ENV GIT_SHA=$GIT_SHA GIT_BRANCH=$GIT_BRANCH
```

Node.js, Go and JVM get equivalent implementations from the same clause, keyed off the same detection table (`app.py`/`main.py` plus FastAPI goes to Python, `index.js`/`server.js` plus Express goes to Node, `main.go` goes to Go, `Application.java` plus Spring goes to JVM). One contract paragraph, one detection signal, four language-specific outputs that all satisfy the same four-endpoint contract.

The skill also carries a fallback the PDF never mentions but the platform team wanted anyway. If modifying the app is too invasive, or if the workload isn't an HTTP server at all, an nginx sidecar answers the same four endpoints without touching a line of application code:

```nginx
server {
  listen 8082;
  location /readiness { return 200 '{"status":"UP"}'; add_header Content-Type application/json; }
  location /liveness  { return 200 '{"status":"UP"}'; add_header Content-Type application/json; }
  location /shutdown  { return 200 'null'; }
  location /info      { return 200 '{"git":{"branch":"unknown"}}'; add_header Content-Type application/json; }
}
```

That sidecar is an escape hatch with an honest tradeoff, and it matters more than it looks: it's what makes the contract applicable to workload types the contract's authors never had in mind. More on that later.

![Platform contract PDF clause mapped to the app-contract skill section, with the missing details listed](/images/from-contract-to-icontract/platform-contract-to-skill.jpg)

Put the clause and the skill side by side and the gap is easy to name. The PDF says a readiness endpoint MUST be provided, and then never says which port, what the response schema is, how the Kubernetes probe should be configured, or what changes between Python and Go. Every one of those is a decision the developer has to make anyway. The contract simply declines to make it for them.

### Example 2: "No Root" Becomes a securityContext Block

**The contract says**, paraphrased from the platform PDF's container section:

> Containers MUST NOT run as root. Containers MUST drop all Linux capabilities and add back only what is strictly required for the process to function.

**The skill writes** it as a fixed block in every generated Deployment, not something the developer has to remember to add:

```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
      containers:
        - name: app
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
```

Two MUSTs, four lines, generated identically on every app the scaffold skill touches. There's a matching half in the Dockerfile, because `runAsNonRoot: true` on a pod whose image only has a root user is a `CreateContainerConfigError` at deploy time, not a security win:

```dockerfile
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser
```

The gate skill's `A3`/`A4` checks read back exactly these blocks, so a manifest that skipped this step (hand-written, copied from an old repo, edited after generation) fails the same way regardless of how it was produced.

### Example 3: "No Persistent Data in a Container" Becomes Three Manifests

**The contract says**, paraphrased from the platform PDF:

> Since containers are temporary, an application MUST NOT store any kind of persistent data inside a container. Persistent data includes reports, log files, dumps, or other artifacts produced by the application itself. An application MAY store transient data ONLY IF it is required to make the application work and does not exceed a size that could disrupt infrastructure availability.

On paper this is one sentence with a MUST NOT and a conditional MAY. It's also the clause with the widest gap between "understood" and "implemented", because the honest implementation is not "don't write files", it's "here is where the file goes instead", and that answer is three manifests deep.

**The skill writes** a PVC first, then a Deployment that mounts it, then a strategy change most people get wrong:

```yaml
# 04-storage.yaml: storage exists before anything tries to use it
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <APP_SLUG>-postgres-pvc
  namespace: <NAMESPACE>
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: <STORAGECLASS_NAME>
  resources:
    requests:
      storage: 10Gi
```

```yaml
# 05-db.yaml: Recreate, not RollingUpdate
spec:
  strategy:
    type: Recreate          # RWO volume + two pods = deadlock on rollout
  template:
    spec:
      containers:
        - name: postgres
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
              subPath: pgdata
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: <APP_SLUG>-postgres-pvc
```

And an initContainer on the app itself, because "the database is a separate lifecycle" is the practical consequence of the same clause:

```yaml
initContainers:
  - name: wait-for-db
    image: <MIRROR_PREFIX>/busybox:1.36
    command: ['sh', '-c', 'until nc -z <APP_SLUG>-postgres 5432; do sleep 2; done']
```

Three artifacts from one MUST NOT, and the two that matter most (`Recreate` and the initContainer) appear nowhere in the contract text. They're the operational consequences of taking the clause seriously, and they're exactly the kind of thing a developer discovers by having a rollout hang for ten minutes. The gate covers all three as `D1` through `D4`, and `D4` is the literal reading of the clause: no volumeMounts to the container filesystem for data, `/tmp` excepted as the contract's transient-data carve-out.

### Example 4: A Four-Layer Image Hierarchy Becomes a Dockerfile and a Tag Strategy

**The contract says**, paraphrased from the Docker images paper, which splits image ownership across four layers with a RACI matrix per layer:

> Base Image: MUST be a slim version, MUST have a biweekly refresh job, MUST carry a date tag `YYYYMMDD` incremented every two weeks and a quarterly tag `YYYY-Qx`. Framework Image: MUST use the organization's base image, SHOULD use the quarterly tag, MUST be pinned so a version bump can be issued on a zero-day, a quarterly patch, or an agreed feature need. Application Image: MUST follow the same strategy. Deployment Image: MUST be based on the latest stable application image. Non-production MUST always use latest; production MUST use the tagged images.

This is the clause that reads like governance and lands like a Dockerfile. It also carries the RACI transition that matters operationally: base and framework layers are the platform team's to own, application and deployment layers move accountability to the product team. A developer reading it learns who to ask. A developer reading it does not learn what to type.

**The skill writes** the layer chain as a concrete multi-stage build, with the base and framework layers resolved from the per-cluster variables file rather than chosen by the developer:

```dockerfile
# builder stage uses the framework image (platform-owned, pinned quarterly)
FROM <MIRROR_PREFIX>/node:22-slim AS builder
WORKDIR /app
COPY package*.json tsconfig.json ./
RUN npm install
COPY src ./src
RUN npm run build

# runtime stage is the deployment image (product-owned, SHA-tagged)
FROM <MIRROR_PREFIX>/node:22-slim
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd -r appuser && useradd -r -g appuser appuser
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY --from=builder /app/dist ./dist
USER appuser
ENV NODE_ENV=production
EXPOSE 8081
EXPOSE 8082
CMD ["npm", "start"]
```

And the tag strategy becomes a CI step, because "non-production always latest, production always pinned" is a branching rule, not a philosophy:

```yaml
- name: Set image tag
  id: meta
  run: |
    SHA="${{ github.sha }}"; SHORT_SHA="${SHA:0:7}"
    if [ "${{ github.ref }}" = "refs/heads/main" ]; then
      echo "tags=${REGISTRY}/${IMAGE}:${SHORT_SHA}" >> $GITHUB_OUTPUT   # prod: pinned
    else
      echo "tags=${REGISTRY}/${IMAGE}:latest" >> $GITHUB_OUTPUT          # non-prod: latest
    fi
```

Here's the interesting part, and it's the single most useful thing that came out of the whole exercise. The source paper offers "such as Alpine Linux or Debian slim" as the example of a good minimal base. The skill states, as a hard invariant before any code: **never use alpine images**, because musl libc breaks native modules in three of the four supported stacks and the platform team had spent real hours on it. The contract's own example contradicted the platform team's operational reality, and nobody had noticed for as long as the rule lived in prose.

Encoding a contract forces every ambiguity into the open, because a skill file cannot say "such as". It has to pick. That's not a side effect of the exercise, it's arguably the main deliverable: the pipeline is a linter for the contract itself.

## Part Two: What the App Must Say

### Example 5: "Logs MUST Be JSON on stdout" Becomes a Logger and a Field Contract

**The contract says**, paraphrased from the platform PDF's observability section:

> All logs MUST be printed to standard output. Every log line MUST be written in JSON format, following the definition provided by OpenTelemetry.

Two sentences. Fully understood by everyone who reads them, and violated constantly, because every language's default logger produces exactly the wrong thing and the fix is a library choice plus a formatter configuration plus removing whatever file handler is already in the codebase.

**The skill writes** the library choice per stack, the formatter, and the removal step:

```python
# logging_config.py
import logging, sys
from pythonjsonlogger import jsonlogger

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(jsonlogger.JsonFormatter(
    '%(asctime)s %(levelname)s %(message)s',
    rename_fields={"asctime": "time", "levelname": "level", "message": "msg"}))
logging.root.addHandler(handler)
logging.root.setLevel(logging.INFO)

# usage: log.info("User created", extra={"user_id": user.id})
# -> {"time":"2026-01-01T10:00:00Z","level":"INFO","msg":"User created","user_id":"abc123"}
```

```js
// Node: pino is JSON by default, which is the whole reason it's the pick
const pino = require('pino')
const logger = pino({ level: 'info' })
app.use(require('pino-http')({ logger }))
logger.info({ userId: user.id }, 'User created')
```

```go
// Go: zap production config is JSON to stdout with no further configuration
logger, _ := zap.NewProduction()
logger.Info("User created", zap.String("user_id", userID))
```

And then the part the contract implies but never states, which is the field contract. "JSON" is not a schema. A log aggregator filtering on `level` needs the key to be called `level`, and the gate has to be able to fail a line that calls it `severity`:

| Field | Type | Example |
|-------|------|---------|
| `time` | ISO8601 string | `"2026-01-01T10:00:00.000Z"` |
| `level` | string | `"info"` / `"error"` / `"warn"` |
| `msg` | string | `"Request received"` |

Optional but recommended: `trace_id`, `span_id`, `app`, `env`. The first two are what make Example 7 useful.

The skill also carries an explicit removal instruction, because adding a JSON logger next to an existing file logger satisfies nothing:

> Python: remove `FileHandler`, `RotatingFileHandler`. Node: remove Winston `transports.File` and any log file mount in the Dockerfile. Do not mount `/var/log` as a volume for application logs.

That maps to gate checks `G3` (logs are JSON) and `G4` (no file-based logging), and `G4` exists specifically because `G3` alone passes on an app that writes JSON to stdout *and* plaintext to a file that quietly fills a node's disk.

There's one more check in this family that isn't in the platform contract at all, and it's the one with the sharpest teeth. `G6` scans log samples for personal data: email addresses, card numbers, `"password": "..."` keys, bearer tokens. The contract says logs must be JSON. It doesn't say what must not be *in* the JSON, and that turned out to be a gap worth filling with regexes rather than with a sentence.

### Example 6: A Three-Tier Metrics Rule Becomes an Annotation and a NetworkPolicy

**The contract says**, paraphrased from the monitoring papers:

> Applications MUST expose metrics across three tiers: system (pod-level resources), application (threads, GC, errors per minute, requests per minute), and business (metrics inherent to the application's own scope, for example how many checkout transactions succeeded and how many were rejected). Metrics MUST be collected without requiring manual dashboard configuration per app.

That's a philosophy paper, not code, and it's the clause most likely to be read once and never operationalized. "Three tiers" doesn't tell a developer what file to edit.

![Monitoring paper concepts mapped to the observability skill: what to measure is described, how to measure is absent](/images/from-contract-to-icontract/monitoring-to-observability.jpg)

The source paper is genuinely good writing (it argues for customer-position metrics with an automatic gate as the worked example, and the argument holds up), and it is entirely silent on the translation cost: which Prometheus client library, how auto-scrape annotations work, which NetworkPolicy lets the scrape through, what LogQL to type, what the OTEL endpoint format is. WHAT to measure is described. HOW is absent, and HOW is the part that takes the afternoon.

**The skill writes** three separate things for three separate tiers. System-tier is free: cAdvisor already scrapes every container's CPU, memory, and network, nothing to generate. Application-tier is the pod annotation plus the matching NetworkPolicy that lets the scraper actually reach it:

```yaml
# 08-app.yaml: pod template metadata
metadata:
  labels:
    app: <APP_SLUG>-app
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8081"
    prometheus.io/path: "/metrics"
```

```yaml
# 01-network-policies.yaml: without this, the annotation above is scraped by nothing
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-metrics-scrape-app
  namespace: <NAMESPACE>
spec:
  podSelector:
    matchLabels:
      app: <APP_SLUG>-app
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: <OBSERVABILITY_NAMESPACE>
      ports:
        - port: 8081
```

That pairing is the whole reason this example is worth showing. The annotation is the documented part, the NetworkPolicy is the part that makes it work in a default-deny namespace, and no sentence in the monitoring paper connects them. A developer who reads the contract perfectly still ships an app that's invisible in Grafana.

Business-tier is the one row the skill can't derive on its own, because it's a product decision, not a platform one. So the skill asks instead of generating a placeholder:

> "What are the three most important numbers that tell you if your app is working well?"

Whatever comes back becomes a counter or gauge, in the same file the request-rate metric already lives in:

```typescript
import { Counter } from 'prom-client'

const ordersProcessed = new Counter({
  name: 'orders_processed_total',
  help: 'Orders successfully processed',
  labelNames: ['status'],
})
// ordersProcessed.inc({ status: 'completed' })
```

Three tiers, three different generation strategies: nothing to write, a fixed pair of manifests, and one question the model asks instead of guessing.

### Example 7: "MUST Send Traces to a Collector" Becomes an SDK Init and an Egress Policy

**The contract says**, paraphrased from the platform PDF:

> Telemetry data for tracing MUST follow the OpenTelemetry data format and convention. Each application MUST be able to send tracing data to an OpenTelemetry collector. The application SHOULD rely on the SDK and libraries provided by the OpenTelemetry project for the technology used.

**The skill writes** the SDK bootstrap with the collector endpoint resolved from the cluster variables file, not typed by the developer:

```typescript
// tracing.ts: imported first, before any other module
import { NodeSDK } from '@opentelemetry/sdk-node'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http'
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node'

const sdk = new NodeSDK({
  serviceName: process.env.APP_SLUG,
  traceExporter: new OTLPTraceExporter({
    url: 'http://otel-collector.<OBSERVABILITY_NAMESPACE>.svc.cluster.local:4318/v1/traces',
  }),
  instrumentations: [getNodeAutoInstrumentations()],
})
sdk.start()
```

And then, again, the manifest half nobody writes down. A default-deny namespace blocks egress too, so an app with a perfectly configured OTEL SDK sends traces into a dropped packet:

```yaml
# 01-network-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-otel-egress
  namespace: <NAMESPACE>
spec:
  podSelector:
    matchLabels:
      app: <APP_SLUG>-app
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: <OBSERVABILITY_NAMESPACE>
      ports:
        - port: 4318
          protocol: TCP
```

Note the contract phrases this as a MUST for the format and a SHOULD for the SDK, which parses into different severities: the collector endpoint being wrong is a WARNING (traces are missing, nothing is unsafe), while the format being non-OTEL is a CRITICAL, because a non-conformant payload pollutes a shared backend for everyone else on it.

### Example 8: "Alert on Trends, Not Numbers" Becomes a Recording Rule and an HPA

**The contract says**, paraphrased from the monitoring papers, and this is my favourite clause in the whole set because it's a genuinely good idea written in a form that cannot be enforced:

> It is not one value that shows healthiness, but the combination of multiple metrics. Create metric alerts with trends, not with the pure number. A value of 70 threads alone makes no sense, different applications have different thread values; what matters is the delta. Be smart in the evaluation: use standard deviation and percentiles, since the mean usually drops the relevant values needed to evaluate performance under stress.

Read that as a developer and you agree with all of it and change nothing, because "use percentiles" is advice, not an artifact.

**The skill writes** it as PromQL, which is the only form in which that paragraph can be wrong or right:

```yaml
# alert on relative change, not on an absolute threshold
groups:
  - name: <APP_SLUG>-app
    rules:
      - record: job:app_threads:delta_pct_15m
        expr: |
          100 * (
            app_threads_active{app="<APP_SLUG>"}
            / avg_over_time(app_threads_active{app="<APP_SLUG>"}[15m]) - 1
          )

      - alert: ThreadGrowthAnomalous
        expr: job:app_threads:delta_pct_15m > 50
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "Active threads up more than 50% against the 15m baseline"

      - alert: LatencyP95Degraded
        # p95, not avg: the mean hides exactly the tail this alert exists to catch
        expr: |
          histogram_quantile(0.95,
            sum by (le) (rate(http_request_duration_seconds_bucket{app="<APP_SLUG>"}[5m]))
          ) > 0.8
        for: 10m
        labels: { severity: warning }
```

And the same paper's horizontal-scaling clause ("applications subject to volatile workload SHOULD be designed to scale horizontally", "applications in production SHOULD implement HPA") becomes the manifest, with the contract's own point about combining signals reflected in using two metrics rather than one:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: <APP_SLUG>-app
  namespace: <NAMESPACE>
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: <APP_SLUG>-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
    - type: Pods
      pods:
        metric: { name: http_requests_per_second }
        target: { type: AverageValue, averageValue: "100" }
```

`minReplicas: 2` is not in the contract either. It's the operational reading of a different clause in the same document ("applications MUST NOT assume the infrastructure will always be available"), and putting it in the skill is how a design principle from page five becomes a default on page one of every new app.

## Part Three: Where the App Lives

### Example 9: A Naming Convention Becomes a Derivation Rule

**The contract says**, paraphrased from the naming conventions paper:

> Namespaces MUST follow the pattern `<env>-<app-name>`. Internal ingress MUST follow `<env>-<app-name>.<env>.<second-level>.<tld>`, public endpoints the equivalent public domain, and managed service endpoints (databases, queues, notifications) MUST reuse the same nomenclature against the internal DNS zone.

On paper this is a lookup a developer has to get right by hand every time they scaffold anything: which prefix does this cluster use again, was it `-alpha` or `-a1`, and does the internal RDS endpoint get the environment prefix once or twice.

**The skill writes** a derivation, not a lookup:

```yaml
APP_SLUG:     <lowercase-hyphens-max20>          # from app name
NAMESPACE:    <cluster.prefix>-<platform>-<APP_SLUG>
IMAGE:        <cluster.registry>/<team>/<APP_SLUG>-app:<commit-sha>
INGRESS_URL:  <derived from cluster.ingressPattern>
DB_ENDPOINT:  <env>-rds-<APP_SLUG>.<env>.<cluster.internalZone>
```

The developer answers six plain-language questions (app name, one sentence on what it does, does it need a database, does it need cache or messaging, which runtime, is there a separate frontend), and the skill resolves everything else against a small per-cluster variables file the platform team maintains once. Ports, image paths, namespace strings and DNS names are never asked about, because every one of them is derivable and every one of them is a place a human introduces a typo.

Get a new cluster with a different prefix, and every skill that consumes `NAMESPACE` picks up the new value automatically, no manual find-and-replace across a fleet of manifests. This is the "injection" step from the pipeline: platform-specific values live in exactly one place, and every skill resolves against it rather than hardcoding a value that will eventually drift.

### Example 10: "Never Commit a Plain Secret" Becomes Vault or AWS Secrets Manager

**The contract says**, paraphrased from the platform PDF's security section:

> Credentials MUST NOT be stored in plaintext in version control, in a ConfigMap, or baked into an image layer. Secrets MUST be resolved from the organization's secret manager at runtime.

This is the clause with the sharpest teeth in the whole contract, and it's also the one where a naive reading of "enforce this" produces the worst possible workflow: developer opens a ticket, waits for the platform team to hand-provision a credential, and the one MUST NOT in the entire contract becomes the biggest bottleneck in the pipeline.

**The skill writes** an `ExternalSecret`, not a value. The scaffold skill deploys the [External Secrets Operator](https://external-secrets.io) once per cluster, pointed at whichever backend the platform already runs, HashiCorp Vault or AWS Secrets Manager, as a `ClusterSecretStore`. From that point on, every app skill generates is a *reference* to a path, never a literal:

```yaml
# k8s/02-external-secret.yaml (Vault backend)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: <app>-db-secret
  namespace: <namespace>
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: <app>-db-secret        # native k8s Secret, created by the operator
    creationPolicy: Owner
  data:
    - secretKey: POSTGRES_PASSWORD
      remoteRef:
        key: secret/data/<namespace>/<app>
        property: db_password
```

Same clause, AWS Secrets Manager backend, only the `secretStoreRef` and `remoteRef.key` change:

```yaml
spec:
  secretStoreRef:
    name: aws-secretsmanager-backend
    kind: ClusterSecretStore
  data:
    - secretKey: POSTGRES_PASSWORD
      remoteRef:
        key: prod/<namespace>/<app>/db-credentials
        property: password
```

And the consumption side, which is `E5` in the gate, because injecting the resolved secret with `value:` instead of `secretKeyRef` puts the credential right back into the manifest it was just removed from:

```yaml
env:
  - name: POSTGRES_PASSWORD
    valueFrom:
      secretKeyRef:
        name: <app>-db-secret
        key: POSTGRES_PASSWORD
```

The developer never touches Vault's or AWS's console, and never runs a manual sealing step. The platform team provisions the `ClusterSecretStore` once (a Vault AppRole, or an IAM role via IRSA for AWS Secrets Manager), writes the actual credential into the backend directly, and every app from then on only ever references a path. Rotation is the part a manual encrypt-once flow doesn't give you for free: bump `refreshInterval`, rotate the value in Vault or AWS Secrets Manager, and the synced Kubernetes `Secret` updates on its own, no re-encryption, no new commit, no PR.

### Example 11: "The Management Port MUST NOT Be Public" Becomes Three Absences

**The contract says**, paraphrased from the availability section:

> The management port exposes management endpoints to the platform, private only. Since the management port is not exposed publicly, it MUST NOT be under authentication.

This clause is unusual and worth isolating, because what it demands is not an artifact. It demands the absence of three artifacts, and absence is the single hardest thing to review by eye. Nobody notices the Ingress rule that shouldn't be there by scrolling past it, because it looks like every other Ingress rule.

**The skill writes** the absences explicitly, and the gate checks for them by name. The Ingress routes 8081 and nothing else:

```yaml
# 10-ingress.yaml: 8081 only, never 8082
spec:
  rules:
    - host: <INGRESS_URL>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: <APP_SLUG>-app
                port:
                  number: 8081
```

The namespace denies everything by default, so 8082 is unreachable unless something explicitly allows it:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: <NAMESPACE>
spec:
  podSelector: {}
  policyTypes: [Ingress]
```

And the probes reach 8082 from the kubelet, which is node-local and therefore unaffected by the policy above, which is exactly why the contract can demand an unauthenticated management port without that being reckless:

```yaml
livenessProbe:
  httpGet: { path: /liveness, port: 8082 }
  timeoutSeconds: 5
readinessProbe:
  httpGet: { path: /readiness, port: 8082 }
  timeoutSeconds: 5        # 10 if the probe touches the database
```

The gate's `B6` check is phrased as a negative for the same reason: "Ingress does NOT route to 8082". That's a one-line grep for a machine and a genuinely easy miss for a human reviewing a forty-line manifest at the end of a Friday.

## Part Four: How the App Ships

### Example 12: "No Human Tampering With the Artifact" Becomes a CI Workflow With Teeth

**The contract says**, paraphrased from the automation section of the platform PDF, cross-referenced with the security paper's pipeline chapter:

> Both build and deployment processes MUST be fully automated and reproducible. The process MUST follow a fully-automated pipeline to produce the artifact that runs in production, MUST ensure that no human tampering can be performed on the final artifact, and MUST ensure static code analysis is performed during the pipeline, including a potential gate that blocks the pipeline depending on the result.

The security paper adds the specific gates: vulnerability scanning, secret scanning, static analysis, each with a stakeholder distribution list attached.

**The skill writes** the workflow, and the important detail is a single line in it:

```yaml
      - name: Scan image with Trivy (vulnerabilities)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.meta.outputs.full_image }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1           # this is the gate: without it the scan is decoration
          ignore-unfixed: true

      - name: Scan secrets with Trufflehog
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
          extra_args: --only-verified

      - name: Push image (main branch only)
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ steps.meta.outputs.full_image }}
```

`exit-code: 1` is the whole clause. A Trivy step without it satisfies "static code analysis is performed during the pipeline" and satisfies nothing about the "gate that blocks the pipeline" half of the same sentence, and that's the exact configuration I've seen in the wild more often than any other: a green pipeline with a scan step that reports one hundred and forty findings into a log nobody opens.

The build-then-scan-then-push ordering is the second half of the clause. The image is built without `push`, loaded locally for scanning, and only pushed after the scan passes, on `main` only. "No human tampering with the final artifact" becomes: the artifact that gets pushed is byte-identical to the one that was scanned, and no developer has an intermediate step where they could substitute it.

And the CD side derives the deploy from the same commit SHA rather than from a tag someone types:

```yaml
      - name: Set image tag
        id: meta
        run: |
          SHA="${{ github.event.workflow_run.head_sha }}"
          echo "tag=${SHA:0:7}" >> $GITHUB_OUTPUT
```

### Example 13: Terraform Guardrails Become a Module Call You Cannot Widen

**The contract says**, quoted almost directly from the developer experience paper's guardrails section, which is written as an example rather than as a policy:

> Set of rules to have a clean infrastructure as code made by developers. Example of an RDS database in production. Rules: MUST have a replica available (read-only, to use as double read with a proxy). MUST be between R5 and R7 instance types. MUST have backup (full weekly, retained two weeks, plus incremental daily/hourly). MUST have a tag for the team owner (for updates and support).

The same paper also mandates that infrastructure MUST be provisioned with Terraform and that reusable modules live in a central repository, pinned by ref, called from project-specific code that carries only environment-specific values.

**The skill writes** the module call, and then it writes the thing that makes the guardrail a guardrail: variable validation in the module itself, so a developer copying a module call and changing a value gets an error at `terraform plan` rather than a comment at review time.

```hcl
# terraform/main.tf in the application repository
module "rds" {
  source = "github.com/<org>/reusable-modules//terraform/modules/rds?ref=v2.4.0"

  name           = "${var.env}-rds-${var.app_slug}"
  instance_class = "db.r6g.large"
  environment    = var.env
  owner_team     = "checkout-platform"
}
```

```hcl
# in the reusable module: the contract, as code that refuses to plan
variable "instance_class" {
  type = string
  validation {
    condition = contains([
      "db.r5.large",  "db.r5.xlarge",  "db.r5.2xlarge",  "db.r5.4xlarge",
      "db.r6g.large", "db.r6g.xlarge", "db.r6g.2xlarge", "db.r6g.4xlarge",
      "db.r7g.large", "db.r7g.xlarge", "db.r7g.2xlarge", "db.r7g.4xlarge",
    ], var.instance_class)
    error_message = "Instance class must be an R5-R7 size. See the platform guardrails."
  }
}

variable "owner_team" {
  type = string
  validation {
    condition     = length(trimspace(var.owner_team)) > 0
    error_message = "owner_team is mandatory: it drives patching and on-call routing."
  }
}

resource "aws_db_instance" "main" {
  identifier              = var.name
  instance_class          = var.instance_class
  backup_retention_period = var.environment == "prd" ? 14 : 1     # two weeks in prod
  backup_window           = "02:00-03:00"
  multi_az                = var.environment == "prd"
  tags = { owner_team = var.owner_team, environment = var.environment }
}

resource "aws_db_instance" "replica" {
  count               = var.environment == "prd" ? 1 : 0          # MUST have a replica
  identifier          = "${var.name}-ro"
  replicate_source_db = aws_db_instance.main.identifier
  instance_class      = var.instance_class
}
```

Four MUSTs from a bullet list become one conditional, one retention number, one validation block, and one count expression. Note where each rule ended up: the replica and the backup window are *derived from the environment*, so a developer cannot forget them, while the instance class is *validated*, because it's a legitimate choice within a bounded set. That distinction (derive what has one right answer, validate what has several) is the same split the naming example uses, and it's the most transferable idea in the whole pipeline.

The `?ref=v2.4.0` pin is the paper's own instruction, and it does something the paper didn't intend: it makes the contract version an explicit dependency of every application repo. Bump the guardrails, cut a new module tag, and you can see in one query which repositories are still on the old contract.

### Example 14: "MUST Be in the Service Catalog First" Becomes Labels and a Generated Entry

**The contract says**, paraphrased from the service management section:

> All applications running in production MUST be present in the Service Catalog. An application MUST be present in the Service Catalog BEFORE it enters production. Any change that could affect a service in production MUST be tracked with a Change Record, and automated tracking for CI/CD operations should be in place.

The appendix then lists the fields: service name, status, description, owner team, source repository, outage impact and its customer-facing description, plus recommended attributes for domain, technologies, PCI, GDPR, public accessibility, cluster and deployment model.

This is the clause with the widest gap between how important it is and how reliably it's followed, because it's the only one whose enforcement point is outside the technical pipeline entirely. Nothing in Kubernetes breaks if the catalog entry doesn't exist. Something in an incident at 3am breaks very badly.

**The skill writes** the catalog entry as a file in the repo, generated from answers the scaffold step already collected, plus the same values as labels on the namespace so the cluster itself is queryable:

```yaml
# catalog-info.yaml, generated at scaffold time
service:
  name: <APP_SLUG>
  status: active
  description: <one-sentence answer from scaffold question 2>
  owner_team: checkout-platform
  repository: https://github.com/<org>/<repo>
  outage_impact: high
  outage_impact_description: >
    Customers cannot complete checkout; payment step returns an error.
  domain: commerce
  technologies: [nodejs-22, postgres-16]
  pci: false
  gdpr: true
  publicly_accessible: true
  kubernetes_cluster: <cluster.name>
  deployment: gitops
```

```yaml
# 00-namespace.yaml: the same facts, where kubectl can answer questions about them
apiVersion: v1
kind: Namespace
metadata:
  name: <NAMESPACE>
  labels:
    owner-team: checkout-platform
    outage-impact: high
    gdpr: "true"
    managed-by: platform-skills
```

Two artifacts, and the second one is what makes the clause enforceable at the only moment it can be: `kubectl get ns -l '!owner-team'` is a one-line query that lists every namespace in the cluster nobody owns. The gate's `H2` check reads the same labels per app.

The change-record half of the clause becomes a CI step that opens the ticket from the deploy rather than asking a human to remember:

```yaml
      - name: Create change record
        run: |
          curl -sf -X POST "$CHANGE_API/records" \
            -H "Authorization: Bearer $CHANGE_TOKEN" \
            -d "{\"service\":\"${APP_SLUG}\",
                 \"summary\":\"Deploy ${GITHUB_SHA:0:7} to ${ENV}\",
                 \"description\":$(jq -Rs . <<< "$(git log -1 --pretty=%B)"),
                 \"owner_team\":\"${OWNER_TEAM}\"}"
```

Fourteen examples, fourteen clauses. That's still a hand-picked slice rather than the ceiling: the six source PDFs had dozens more MUSTs left uncovered here (probe timeout floors, ingress TLS, resource sizing tables, SSO and OAuth2 flows, supported-architecture matrices, the biweekly base image refresh job), and nothing about the pipeline caps it at "one platform contract". The same eight steps run identically over a security policy, a data-governance standard, an incident-response runbook, or any other written contract a given role is bound by. Point the same pipeline at a different source document and a different audience and it produces a different set of skills: an SRE gets skills gated against the on-call runbook, a data engineer gets skills gated against retention and PII policy, a frontend team gets skills gated against the accessibility and performance budget contract. An agentic version of the same pipeline, one that reads a role's stack of governing documents and regenerates its own skill set as those documents change, extends to as many roles and domains as there are contracts worth enforcing.

## The Real Speedup: Building Something the Contract Never Anticipated

Everything above compares favourably to a PDF, but the comparison is unfair in a way worth being explicit about: for a standard Spring Boot REST service, a well-run organization doesn't need either artifact. There's a repo from last quarter that already does it right, and copying it is faster than reading anything. The paper contract's real cost isn't in the well-trodden case.

The expensive case is the one nobody has done before. An MCP server. An agent worker that consumes from a queue and never serves HTTP. A websocket gateway. A Go service in a team that has only ever shipped Java. A batch job that has to run inside the cluster because the data can't leave it. There is no repo to copy, the contract never names the thing you're building, and every clause has to be interpreted rather than applied.

Here's what that costs on the paper path, honestly enumerated:

1. **Discovery.** Which of the six documents apply to a workload that isn't a web application? The platform contract's availability section opens with "applications that expose their functionalities across the network", and a queue consumer doesn't. Is it exempt from the management port, or is the clause badly worded? Nobody knows without asking.
2. **Interpretation.** The naming paper gives namespace patterns with worked examples for a customer-facing app. A batch job that runs in three environments and writes to one shared bucket doesn't map cleanly onto `<env>-<app-name>` and the developer has to make a decision the document didn't authorize them to make.
3. **Silent omission.** The clauses the developer never discovered simply don't get implemented. Nobody skips the `/metrics` endpoint on purpose; they skip it because they were reading the availability section and the metrics requirement lives in a different PDF.
4. **Review latency.** The first real feedback arrives when a platform engineer has a free slot. The gap between submitting and hearing back is measured in review-slot availability, not in the amount of work involved, and every round trip pays that cost again.

I've watched that loop take two weeks for an app that took two days to write. None of those two weeks were spent on hard problems.

The skill path collapses this, and the reason is a specific asymmetry rather than general AI enthusiasm. There are two different unknowns in play:

- **What the developer doesn't know**: which port, which path, which JSON field names, which registry prefix, which namespace pattern, which severity a violation carries. This is arbitrary, organization-specific, undiscoverable by reasoning, and it's exactly what the skill file encodes.
- **What the platform doesn't know**: how to structure an MCP server in TypeScript, what a good retry policy looks like for this particular queue, what this app's business metric should count. This is general engineering knowledge plus domain judgment, and it's what the model and the developer are respectively good at.

A paper contract inverts the load: it hands the developer the first category as homework, in prose, indexed by document. A skill set hands the model the first category as invariants and lets the human spend their attention on the second.

{{< mermaid >}}
flowchart TD
    N[New workload type nobody has shipped here before]
    N --> P1[Paper path: work out which of six documents apply]
    P1 --> P2[Interpret clauses written for a workload you are not building]
    P2 --> P3[Build, guessing the undocumented parts]
    P3 --> P4[Submit to platform review]
    P4 --> P5{Reviewer finds violations}
    P5 -->|days per round trip| P3
    P5 -->|clean| DONE[Running in production]
    N --> S1[Skill path: answer six questions about the app]
    S1 --> S2[Invariants apply regardless of workload type]
    S2 --> S3[Gate run, seconds]
    S3 -->|CRITICAL| S2
    S3 -->|pass| DONE
{{< /mermaid >}}

Take the MCP server concretely, because it's the case that convinced me. The string "MCP" appears in exactly zero of the six PDFs, and the contracts predate the protocol entirely. What actually happens:

- The scaffold skill asks its six questions. None of them are "is this a web application", because the skill derives ports rather than asking about them.
- The namespace, image path, ingress hostname and internal DNS name are derived from the naming rule. The rule doesn't care what the workload does.
- The detection table sees `index.js` plus a dependency manifest and resolves to the Node.js branch. Base image, non-root user, dropped capabilities, resource requests: all applied, none discussed.
- The app-contract skill finds no HTTP server to attach a management port to, and takes the documented fallback: the nginx sidecar answers the four lifecycle endpoints on 8082. A contract requirement written for web applications gets satisfied by a workload that isn't one, without anyone having to decide whether the clause applies.
- The review gate flags the missing `/metrics` endpoint as `G1`/`G2` CRITICAL. That's the correct outcome, because it's the one genuinely new decision here: what does an MCP server export as an application metric? The gate stops and asks instead of inventing a number.
- The business-metric question comes back to the developer, in plain language, at the only point in the process where they're the right person to answer it.

The end state is a workload category nobody wrote a section for, compliant by construction on thirty-odd mechanical checks, with a human's attention spent on the one decision that actually needed it. The elapsed time is a conversation, not a review cycle.

Put the two paths side by side:

| | Paper contract | iContract skills |
|---|---|---|
| How knowledge is indexed | By source document | By what the developer is trying to do |
| First-ever implementation of a workload type | Most expensive case: no prior art, clauses must be interpreted | Same cost as any other case: invariants don't care about workload type |
| Unknown unknowns | You must know a rule exists to look it up | Rules apply whether or not anyone thought to ask |
| Novel language or runtime | Developer improvises everything, including the platform parts | Model improvises the runtime idiom only; contract shape is fixed |
| Feedback latency | A human review slot, days | A gate run, seconds |
| Definition of "done" | A reviewer's judgment, varying by reviewer | An enumerated checklist with stable IDs |
| Failure mode | Silent non-compliance, discovered in production or never | Loud CRITICAL, before `kubectl apply` |
| Where a clause conflicts with reality | Stays ambiguous for years | Surfaces the first time someone tries to encode it |

The last row is the alpine story from Example 4, and it generalizes. Prose tolerates "such as" and "where appropriate" indefinitely; a skill file has to pick a value, and the picking is where latent contradictions surface.

One more thing worth saying plainly, because it's the objection I'd raise myself: these are *initial* skills. Four files, one narrow slice of one contract, dozens of clauses uncovered. That partial coverage still beats a complete PDF for the unknown case, and the reason is that completeness of clauses was never the useful metric. The useful metric is coverage of the decisions a newcomer would otherwise make blind, and those decisions are heavily concentrated: the same thirty choices (which port, which base image, which log field, which secret mechanism, which namespace string) recur on essentially every workload, novel or not. Four skill files that pin those thirty decisions do more work than eighty pages that describe two hundred clauses nobody has indexed.

The honest limit: for genuinely novel infrastructure, a stateful engine with no existing PVC pattern, an architecture the platform has never hosted, the skill degrades to its invariants. The model improvises the middle, the gate catches the mechanical part, and a human has to catch the architectural part. That's the residual, it's real, and it's the right residual to leave for a platform engineer.

## How to Make Skills Actually Respected: The Gate

Generation is the easy half. Anyone can get a model to produce a plausible-looking Dockerfile. The half that actually makes a contract *respected* rather than merely *consulted* is the separation the pipeline calls out at step five: every domain gets a **generate** skill and a **gate** skill, and they are not the same artifact.

The generate skill is proactive: "add health endpoints," and it writes the files. The gate skill is reactive: "is this ready to deploy," and it reads existing manifests against the same rules, without generating anything, before `kubectl apply` ever runs. It reports only failures; passes are silent, so the output stays readable even when almost everything is already correct.

![The review gate: check taxonomy A through H, failures-only output format, CRITICAL severities from contract MUST language](/images/from-contract-to-icontract/review-gate.jpg)

The taxonomy on the left is organized by domain (image, health and ports, network, storage, secrets, resources, observability, ingress) and every group cites the source document it came from, which is what makes a failure arguable rather than arbitrary. The output on the right is the whole user experience: failures only, plain language, each one carrying the check ID and the fix.

A trimmed slice of the check matrix:

| # | Check | Pass condition | Severity |
|---|-------|-----------------|----------|
| A1 | No `:latest` tag in production | No image reference ends in `:latest` | CRITICAL |
| A5 | No public-registry images | Every image reference resolves to the internal registry | CRITICAL |
| B3 | Liveness probe on the management port | Correct port and path | CRITICAL |
| B6 | Management port not in Ingress | No Ingress backend targets 8082 | CRITICAL |
| C1 | default-deny-ingress present | NetworkPolicy exists in the namespace | CRITICAL |
| D2 | Database strategy is Recreate | `strategy.type: Recreate` on every DB Deployment | CRITICAL |
| E1 | No plain `Secret` in Git | Only `ExternalSecret` references, no literal value in any manifest | CRITICAL |
| G1 | Metrics scrape annotation present | `prometheus.io/scrape`, `port`, `path` all set | CRITICAL |
| G6 | No personal data in logs | Sample log lines match no PII pattern | CRITICAL |
| F1 | CPU and memory requests set | Every container declares `resources.requests` | WARNING |
| H1 | Ingress hostname matches cluster pattern | Regex match against the derived pattern | WARNING |

Two design choices make this gate actually stick instead of becoming one more document nobody reads:

**Every failure links back to the generate skill that fixes it.** A `B3` failure doesn't just say "probe missing". It says which skill file and which step wrote the code that's supposed to satisfy it. The gate never leaves a developer holding a red X with no next action.

**CRITICAL blocks, WARNING doesn't.** Not everything in a contract deserves the same weight, and treating every clause as equally blocking is how gates get bypassed out of frustration. A missing resource limit gets flagged and deploy proceeds. A plain secret in Git blocks, full stop, no override, because that's the one violation in the entire matrix explicitly called out as dangerous enough to warrant it.

There's also an auto-fix mode: for CRITICAL failures, apply the fix directly and report what changed, no permission prompt required, because a `runAsNonRoot: true` line is not a decision that benefits from a confirmation dialog. WARNING fixes get applied too, just mentioned rather than gated.

## From a Working App to a Platform Citizen

![The iContract skill ecosystem: triggers routing to scaffold, app-contract, gitops and observability, all converging on the review gate](/images/from-contract-to-icontract/skill-ecosystem.jpg)

Skills compose, and the composition is driven by triggers rather than by a developer picking a document. "Scaffold an agent", "add health checks", "set up CI/CD", "I can't see my app in Grafana": four different phrasings, four different skills, one gate that every path converges on before anything reaches the cluster.

Everything so far describes one request producing one compliant app. That's the easy version of the problem. An agent chaining `scaffold` into `app-contract` into `observability` into `review` can produce a pod that runs, passes every check, and is still, in every way that matters operationally, an island: nobody else's monitoring, ownership, or incident process knows it exists. Getting an app *generated* correctly and getting an app *absorbed* into a shared platform are two different problems, and only the first one is solved by chaining skills once.

{{< mermaid >}}
flowchart TD
    subgraph GEN["One request, agent-orchestrated"]
        U[Developer describes the app in one sentence] --> SC[scaffold: namespace, manifests, secret references]
        SC --> AC[app-contract: health endpoints, /metrics, JSON logs, Dockerfile]
        AC --> OB[observability: scrape annotation, NetworkPolicy, business metric]
    end
    OB --> GATE{review gate}
    GATE -->|CRITICAL fail| AC
    GATE -->|pass| APPLY[GitOps sync / kubectl apply]
    subgraph PLAT["Becoming a platform citizen: the real challenge"]
        APPLY --> RUN[Pod running, isolated by default-deny NetworkPolicy]
        RUN --> W1[Secret store: ClusterSecretStore, Vault or AWS Secrets Manager]
        RUN --> W2[Metrics backend: scraped, dashboarded, alertable]
        RUN --> W3[Log pipeline: shipped, queryable, retained]
        W1 --> OWN[Ownership record: team, on-call, SLA]
        W2 --> OWN
        W3 --> OWN
        OWN --> NEXT[Every future change re-enters the gate]
    end
    NEXT -.->|contract revision or app change| SC
{{< /mermaid >}}

The left box is what the examples in this post automate: a handful of skills, one gate, minutes of wall-clock time. The right box is where the actual platform-engineering effort lives, and none of it happens because the manifest was generated correctly. A `ClusterSecretStore` has to exist before an `ExternalSecret` can resolve against it. A metrics backend has to be reachable and retained before a dashboard means anything. An ownership record has to exist somewhere queryable, or an alert with nobody attached to it is just noise, which is precisely why Example 14's namespace labels are worth generating even though nothing in Kubernetes enforces them.

The loop at the bottom is the part easy to leave out of a diagram and expensive to leave out of the system: `NEXT` feeds back into `scaffold`, not into a one-time setup step. The contract isn't a gate the app passes once at birth, it's a gate every future change to that app has to keep clearing, because the platform's own definition of "compliant" keeps moving (a new mandatory header, a revised probe timeout, a stricter secret-rotation window) whether or not the app owner is paying attention that week. An agent that only runs the left box on day one and never revisits the right box is exactly the failure mode this whole pipeline exists to prevent, just with better tooling around the day-one part.

## Does the AI Actually Comply?

Here's the honest part, and it's worth being precise about what "compliance" means when the enforcer is a model rather than a compiler.

A model does not comply with a contract the way a type-checker complies with a schema. Given the same skill and the same repository twice, it can plausibly emit a slightly different Dockerfile both times: one with a comment the other doesn't have, one that orders `RUN` layers differently, occasionally one that forgets a flag the constraint section explicitly called out. That's not a bug in the pipeline, it's the nature of a non-deterministic generator, and pretending otherwise would be dishonest.

![Not perfect, good enough to matter: platform team effort with and without the skill layer](/images/from-contract-to-icontract/good-enough-to-matter.jpg)

What the pipeline actually buys isn't determinism. It's *distance reduction*. Before any of this existed, a developer starting from zero produced an artifact that needed, on average, most of a platform reviewer's attention to bring into compliance: wrong base image, missing probes, secrets in the wrong place, no metrics annotations. After the skill, the model's first output already satisfies the large majority of the same checklist, correctly, because the constraints were stated as invariants before a single line of code was written. The remaining gap is what the gate skill exists to catch, and it catches it automatically instead of in a human review cycle measured in days.

Put a number on it: the goal was never 100% unattended automation, that's not a realistic target for a non-deterministic generator sitting in front of a mandatory security gate. The realistic target, and the one this pipeline was designed around, is a 60-70% reduction in platform-team engagement per app onboarded. The residual 30-40%, genuine architectural exceptions, brand-new service categories the skill's decision tables don't cover yet, contract clauses that turned out to be ambiguous once someone tried to encode them, is exactly the kind of work worth a platform engineer's attention. It's the routine 60-70%, the same probe path typed slightly wrong, the same base image swapped for the wrong one, that the gate now catches without anyone paging a human.

So: does the AI evade the contract sometimes? Yes. Does it still construct an artifact close enough to what a hand-written, contract-compliant manifest would look like that the remaining alignment work drops from a multi-day review cycle to a five-minute gate run? Also yes, and that second fact is the entire value proposition. A contract that took a platform team weeks to write and years to get consistently followed now has a first draft produced in the time it takes to describe an app in six sentences, correct on the majority of checks before a human ever looks at it. The gate exists precisely because "mostly compliant" and "compliant" are different states, and only one of them is safe to deploy on a shared cluster.

## Security Considerations

![iContract design principles: one skill per contract domain, triggers replace navigation, generate and gate separated, polyglot or dead, invariants first](/images/from-contract-to-icontract/design-principles.jpg)

The design rules that came out of the exercise, and the second one is the one I'd keep if I could only keep one: **triggers replace navigation**. The set of trigger phrases is the index, and it's a richer index than any table of contents, because it's written in the words a developer actually uses when they have the problem rather than in the words a policy author used when they wrote the answer.

1. **The gate is the actual enforcement point, not the generate skill.** Treat the generate skill's output as a strong first draft, always, never as pre-approved. If the gate isn't wired into the deploy path, none of this is enforcement, it's autocomplete with better formatting.
2. **CRITICAL checks should never be overridable from inside a skill invocation.** The plain-secret check (`E1`) is the clearest example: it's the single most dangerous misconfiguration in the whole matrix, and the fix has to be "reject and explain", never "warn and proceed".
3. **Constraints must be stated before examples, not after.** A MUST-NOT rule appended as a closing caveat gets treated with less weight by the model than the same rule stated as a precondition. Position it first.
4. **A scan step without a non-zero exit code is not a gate.** Example 12's `exit-code: 1` is the difference between a pipeline that blocks on CRITICAL vulnerabilities and one that reports them into a log nobody opens. The same applies to secret scanning and static analysis.
5. **Platform variables are injected, never hardcoded per app.** A registry prefix or namespace pattern baked into every generated manifest individually becomes a fleet-wide migration the day it changes. Resolve it from one shared source.
6. **Auto-fix is fine for CRITICAL, not for architectural decisions.** Silently rewriting a probe timeout is safe. Silently choosing a database engine is not. Keep the line between "mechanical fix" and "design decision" explicit in the skill itself.
7. **Absence-shaped requirements need explicit checks.** Example 11's "management port must not be in the Ingress" cannot be verified by looking at what's present. Every MUST NOT in the source contract needs a check phrased as a negative, or it silently passes.

## Download the Skills

The skills behind the examples above, sanitized of any org-specific values
(swap the `<cluster.*>` placeholders for your own before using them). These cover
one slice of one contract, treat them as a starting shape to run the same
pipeline against your own documents, not as the complete set:

- [`app-contract.md`](/files/from-contract-to-icontract/app-contract.md): health endpoints, `/metrics`, JSON logging, Dockerfile (Examples 1, 2, 4, 5)
- [`scaffold.md`](/files/from-contract-to-icontract/scaffold.md): namespace derivation, ExternalSecret self-service, app deployment (Examples 9, 10, 11, 14)
- [`observability.md`](/files/from-contract-to-icontract/observability.md): three-tier metrics, scrape NetworkPolicy, OTEL tracing, business-metric prompt (Examples 6, 7, 8)
- [`db-provision.md`](/files/from-contract-to-icontract/db-provision.md): PVC sizing, Recreate strategy, initContainer wait pattern, exporter sidecar (Example 3)
- [`gitops.md`](/files/from-contract-to-icontract/gitops.md): CI build/scan/push with a blocking scan, CD from the scanned SHA (Example 12)
- [`review.md`](/files/from-contract-to-icontract/review.md): the gate skill, pass/fail matrix, auto-fix mode

Drop them into `.claude/skills/` (or your agent's equivalent skill directory) and
fill in the `<cluster.*>` placeholders at the top of `scaffold.md` for your own
registry, namespace prefix, secret backend, and observability namespace.

## Conclusion

A platform contract's real job was never to exist as a PDF, it was to change what gets built. Splitting the pipeline into parse, cluster, expand, constrain, generate, gate, link, inject turned six documents nobody fully internalized into a set of skills that write compliant code on request and refuse to let non-compliant code through the gate. The generate side does the boring 60-70%, correctly and immediately. The gate side catches what the generate side gets wrong, every time, without waiting for a human review slot to open up. Neither half works without the other: generation without a gate is just a faster way to produce plausible-looking violations, and a gate without generation is the same manual compliance burden it always was, just with better error messages.

The payoff concentrates exactly where a written standard is weakest. For the fifth Spring Boot service this quarter, a good contract and a good skill set land in roughly the same place, because someone already solved it and the answer is a `git clone` away. For the first of anything, the paper contract charges the developer for discovery, interpretation and review latency before a single line of the actual problem gets solved, and the skill set charges them for one question about business metrics.

## Reflections

What's still unresolved is mostly about drift, not about the pipeline itself. The contract PDFs will change, someone will revise a probe timeout or add a new mandatory header, and nothing today automatically re-derives the skills when the source document changes. Right now that's a manual re-authoring step, which means the skill can silently fall behind the contract it was built from, the exact failure mode this whole exercise was meant to solve on the human side.

There's also a subtler risk in how convincing a "mostly compliant" artifact looks. A generated manifest that passes 90% of the gate reads, at a glance, exactly like one that passes 100%, and the 10% gap is precisely the part a rushed reviewer is least likely to catch by eye. The gate has to stay mandatory and automated for that reason; the moment it becomes optional "for simple apps", the whole distance-reduction argument collapses back into hoping people read the PDF.

And the honest open question: is version-controlling the skill alongside the contract (skill version pinned to contract version, no drift between a wiki edit and a skill file) actually sufficient to keep them in sync over years, or does it just relocate the review burden from "read the contract" to "review the skill diff"? I don't have a clean answer yet. It's a smaller review than the original problem, which is the whole point, but it's not zero, and treating it as zero is exactly the kind of unstated assumption that turns a safe automation into a load-bearing one nobody's watching.

