---
title: "From Contract to iContract: Turning a Platform PDF Into a Skill an AI Actually Follows"
date: 2026-08-02
draft: false
description: "How I compiled six platform contract PDFs into AI skills that generate compliant Kubernetes manifests and gate deploys, cutting platform-team alignment time by roughly two thirds."
tags:
  - ai
  - kubernetes
  - automation
  - security
  - containers
  - monitoring
  - dynamic infrastructure
featuredImage: /images/from-contract-to-icontract/featured.jpg
images:
  - "/images/from-contract-to-icontract/featured.jpg"
---
### Table of Contents

  * Introduction
  * The Problem: A PDF Is Not an Interface
  * The Architecture: From Contract to Skill
  * Example 1: A Health-Check Clause Becomes Code
  * Example 2: A Naming Convention Becomes a Derivation Rule
  * Example 3: "Never Commit a Plain Secret" Becomes Vault or AWS Secrets Manager
  * Example 4: A Three-Tier Metrics Rule Becomes an Annotation and a NetworkPolicy
  * Example 5: "No Root" Becomes a securityContext Block
  * How to Make Skills Actually Respected: The Gate
  * From a Working App to a Platform Citizen
  * Does the AI Actually Comply?
  * Security Considerations
  * Download the Skills
  * Conclusion
  * Reflections



Well. Every platform team eventually writes the same document: a PDF (or a Confluence page pretending to be one) that says what an application must do before it's allowed to run on the shared cluster. Containerized. Non-root. Health endpoints on a specific port. No plain secrets in Git. It's correct, it's thorough, and almost nobody who needs to follow it has actually read it end to end.

I had six of these documents sitting across a platform contract and five guideline papers: containerization and health probes, GitOps and Terraform module conventions, DNS and namespace naming, a three-tier monitoring philosophy, a four-layer Docker image hierarchy, and a security/SBOM policy with a RACI matrix nobody outside the platform team could recite. Between them, a genuinely well-designed set of rules. In practice, the thing developers actually consulted was whoever answered fastest in the support channel.

So I ran an experiment: turn every MUST, SHOULD, and MAY in those six PDFs into an **iContract**, a set of AI skills that generate compliant code on request and gate deploys when the code doesn't comply. Not a summary of the contract. Not a chatbot that can quote the contract back to you. An artifact the model produces that structurally mirrors what the contract demands, in the language the contract was actually written for: Dockerfiles, Kubernetes manifests, health endpoints, log formatters.

This post walks through the transformation pipeline, five concrete before/after examples pulled straight from the source contracts, each shown as the rule as written next to the code the skill actually generates from it, how the enforcement side actually blocks bad deploys instead of just describing good ones, and an honest look at where AI compliance breaks down even when the pipeline works exactly as designed.

## The Problem: A PDF Is Not an Interface

A platform contract has two audiences, and it serves neither one well. For a human developer, it's too long to read before the first deploy and too easy to forget after. For a machine, in the classic sense, it's not machine-readable at all: no schema, no linter, no CI check that fails the build because paragraph four, section two, said port 8082 is mandatory.

The result is a predictable failure mode. A developer builds an app, it works locally, they copy a Kubernetes manifest from a colleague who copied it from someone else two years ago, and it goes to the platform team for review. Every violation the review catches, missing liveness probe, a plain `Secret` committed to Git, a `:latest` tag, is a violation the contract already described. The contract wasn't wrong. It was inert.

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

**Separate generate from gate.** This is the structural decision that makes enforcement possible at all, and the next two sections are built entirely around it.

## Example 1: A Health-Check Clause Becomes Code

**The contract says**, paraphrased from the platform PDF:

> The application MUST expose a management interface on a dedicated port, separate from application traffic, providing readiness and liveness signals. This port MUST NOT be publicly routable.

**The skill writes.** That one sentence becomes a table plus four implementations:

| Endpoint | Method | Response 200 | Response 503 |
|----------|--------|---------------|---------------|
| `/readiness` | GET | `{"status":"UP"}` | `{"status":"OUT_OF_SERVICE"}` |
| `/liveness` | GET | `{"status":"UP"}` | `{"status":"DOWN"}` |
| `/info` | GET | `{"app":"<slug>","version":"<sha>"}` | - |
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

if __name__ == "__main__":
    uvicorn.run(mgmt, host="0.0.0.0", port=8082)
```

Node.js and Go get equivalent implementations from the same clause, keyed off the same detection table (`app.py`/`main.py` + FastAPI → Python, `index.js`/`server.js` + Express → Node, `main.go` → Go). One contract sentence, one detection signal, four language-specific outputs that all satisfy the same four-endpoint contract. The skill also carries a fallback the PDF never mentions but the platform team wanted anyway: if modifying the app is too invasive, an nginx sidecar can answer the same four endpoints without touching a line of application code.

## Example 2: A Naming Convention Becomes a Derivation Rule

**The contract says**, paraphrased from the naming conventions paper:

> Namespaces MUST follow the pattern `<env>-<platform>-<app-slug>`, resolved against the cluster's registered prefix. DNS entries MUST NOT be created manually; they are derived from the namespace and application slug.

On paper this is a lookup a developer has to get right by hand every time they scaffold a new app: which prefix does this cluster use again, was it `-alpha` or `-a1`.

**The skill writes** a derivation, not a lookup:

```yaml
APP_SLUG:     <lowercase-hyphens-max20>          # from app name
NAMESPACE:    <cluster.prefix>-<platform>-<APP_SLUG>
IMAGE:        <cluster.registry>/<team>/<APP_SLUG>-app:<commit-sha>
INGRESS_URL:  <derived from cluster.ingressPattern>
```

The developer answers six plain-language questions (app name, does it need a database, does it need a frontend), and the skill resolves everything else, including the namespace string, against a small per-cluster variables file the platform team maintains once. Get a new cluster with a different prefix, and every skill that consumes `NAMESPACE` picks up the new value automatically, no manual find-and-replace across a fleet of manifests. This is the "injection" step from the pipeline: platform-specific values live in exactly one place, and every skill resolves against it rather than hardcoding a value that will eventually drift.

## Example 3: "Never Commit a Plain Secret" Becomes Vault or AWS Secrets Manager

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

The developer never touches Vault's or AWS's console, and never runs a manual sealing step. The platform team provisions the `ClusterSecretStore` once (a Vault AppRole, or an IAM role via IRSA for AWS Secrets Manager), writes the actual credential into the backend directly, and every app from then on only ever references a path. Rotation is the part a manual encrypt-once flow doesn't give you for free: bump `refreshInterval`, rotate the value in Vault or AWS Secrets Manager, and the synced Kubernetes `Secret` updates on its own, no re-encryption, no new commit, no PR.

## Example 4: A Three-Tier Metrics Rule Becomes an Annotation and a NetworkPolicy

**The contract says**, paraphrased from the monitoring papers:

> Applications MUST expose metrics across three tiers: system (automatic, infrastructure-level), application (request rate, latency, error rate), and business (domain-specific counters the app owner defines). Metrics MUST be collected without requiring manual dashboard configuration per app.

That's a philosophy paper, not code, and it's the clause most likely to be read once and never operationalized, "three tiers" doesn't tell a developer what file to edit.

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

## Example 5: "No Root" Becomes a securityContext Block

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

Two MUSTs, four lines, generated identically on every app the scaffold skill touches. The gate skill's `A3`/`A4` checks read back exactly these two blocks, so a manifest that skipped this step (hand-written, copied from an old repo, edited after generation) fails the same way regardless of how it was produced.

Five examples, one clause each. That's deliberately a small, hand-picked slice, not the ceiling. The six source PDFs alone had dozens more MUSTs left uncovered here (probe timeouts, ingress TLS, resource sizing, PVC strategy), and nothing about the pipeline caps it at "one platform contract." The same eight steps, parse, cluster, expand, constrain, generate, gate, link, inject, run identically over a security policy, a data-governance standard, an incident-response runbook, or any other written contract a given role is bound by. Point the same pipeline at a different source document and a different audience, and it produces a different set of skills automatically: an SRE gets skills gated against the on-call runbook, a data engineer gets skills gated against retention and PII policy, a frontend team gets skills gated against the accessibility and performance budget contract. The examples above are what one narrow slice of one platform contract looks like once it's been run through the pipeline by hand. An agentic version of the same pipeline, one that reads a role's stack of governing documents and regenerates its own skill set as those documents change, extends to as many roles and domains as there are contracts worth enforcing.

## How to Make Skills Actually Respected: The Gate

Generation is the easy half. Anyone can get a model to produce a plausible-looking Dockerfile. The half that actually makes a contract *respected* rather than merely *consulted* is the separation the pipeline calls out at step five: every domain gets a **generate** skill and a **gate** skill, and they are not the same artifact.

The generate skill is proactive: "add health endpoints," and it writes the files. The gate skill is reactive: "is this ready to deploy," and it reads existing manifests against the same rules, without generating anything, before `kubectl apply` ever runs. It reports only failures; passes are silent, so the output stays readable even when almost everything is already correct.

A trimmed slice of the check matrix:

| # | Check | Pass condition | Severity |
|---|-------|-----------------|----------|
| A5 | No Docker Hub images | Every image reference resolves to the internal registry | CRITICAL |
| B3 | Liveness probe on the management port | Correct port and path | CRITICAL |
| E1 | No plain `Secret` in Git | Only `ExternalSecret` references present, no literal value in any manifest | CRITICAL |
| G1 | Metrics scrape annotation present | `prometheus.io/scrape`, `port`, `path` all set | CRITICAL |
| F1 | CPU and memory requests set | Every container declares `resources.requests` | WARNING |
| H1 | Ingress hostname matches cluster pattern | Regex match against the derived pattern | WARNING |

Two design choices make this gate actually stick instead of becoming one more document nobody reads:

**Every failure links back to the generate skill that fixes it.** A `B3` failure doesn't just say "probe missing." It says which skill file and which step wrote the code that's supposed to satisfy it. The gate never leaves a developer holding a red X with no next action.

**CRITICAL blocks, WARNING doesn't.** Not everything in a contract deserves the same weight, and treating every clause as equally blocking is how gates get bypassed out of frustration. A missing resource limit gets flagged and deploy proceeds. A plain secret in Git blocks, full stop, no override, because that's the one violation in the entire matrix explicitly called out as dangerous enough to warrant it.

There's also an auto-fix mode: for CRITICAL failures, apply the fix directly and report what changed, no permission prompt required, because a `runAsNonRoot: true` line is not a decision that benefits from a confirmation dialog. WARNING fixes get applied too, just mentioned rather than gated.

## From a Working App to a Platform Citizen

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

The left box is what the examples in this post automate: a handful of skills, one gate, minutes of wall-clock time. The right box is where the actual platform-engineering effort lives, and none of it happens because the manifest was generated correctly. A `ClusterSecretStore` has to exist before an `ExternalSecret` can resolve against it. A metrics backend has to be reachable and retained before a dashboard means anything. An ownership record has to exist somewhere queryable, or an alert with nobody attached to it is just noise, the same failure mode called out earlier for tools that quietly become load-bearing.

The loop at the bottom is the part easy to leave out of a diagram and expensive to leave out of the system: `NEXT` feeds back into `scaffold`, not into a one-time setup step. The contract isn't a gate the app passes once at birth, it's a gate every future change to that app has to keep clearing, because the platform's own definition of "compliant" keeps moving (a new mandatory header, a revised probe timeout, a stricter secret-rotation window) whether or not the app owner is paying attention that week. An agent that only runs the left box on day one and never revisits the right box is exactly the failure mode this whole pipeline exists to prevent, just with better tooling around the day-one part.

## Does the AI Actually Comply?

Here's the honest part, and it's worth being precise about what "compliance" means when the enforcer is a model rather than a compiler.

A model does not comply with a contract the way a type-checker complies with a schema. Given the same skill and the same repository twice, it can plausibly emit a slightly different Dockerfile both times: one with a comment the other doesn't have, one that orders `RUN` layers differently, occasionally one that forgets a flag the constraint section explicitly called out. That's not a bug in the pipeline, it's the nature of a non-deterministic generator, and pretending otherwise would be dishonest.

What the pipeline actually buys isn't determinism. It's *distance reduction*. Before any of this existed, a developer starting from zero produced an artifact that needed, on average, most of a platform reviewer's attention to bring into compliance: wrong base image, missing probes, secrets in the wrong place, no metrics annotations. After the skill, the model's first output already satisfies the large majority of the same checklist, correctly, because the constraints were stated as invariants before a single line of code was written. The remaining gap is what the gate skill exists to catch, and it catches it automatically instead of in a human review cycle measured in days.

Put a number on it: the goal was never 100% unattended automation, that's not a realistic target for a non-deterministic generator sitting in front of a mandatory security gate. The realistic target, and the one this pipeline was designed around, is a 60-70% reduction in platform-team engagement per app onboarded. The residual 30-40%, genuine architectural exceptions, brand-new service categories the skill's decision tables don't cover yet, contract clauses that turned out to be ambiguous once someone tried to encode them, is exactly the kind of work worth a platform engineer's attention. It's the routine 60-70%, the same probe path typed slightly wrong, the same base image swapped for the wrong one, that the gate now catches without anyone paging a human.

So: does the AI evade the contract sometimes? Yes. Does it still construct an artifact close enough to what a hand-written, contract-compliant manifest would look like that the remaining alignment work drops from a multi-day review cycle to a five-minute gate run? Also yes, and that second fact is the entire value proposition. A contract that took a platform team weeks to write and years to get consistently followed now has a first draft produced in the time it takes to describe an app in six sentences, correct on the majority of checks before a human ever looks at it. The gate exists precisely because "mostly compliant" and "compliant" are different states, and only one of them is safe to deploy on a shared cluster.

## Security Considerations

1. **The gate is the actual enforcement point, not the generate skill.** Treat the generate skill's output as a strong first draft, always, never as pre-approved. If the gate isn't wired into the deploy path, none of this is enforcement, it's autocomplete with better formatting.
2. **CRITICAL checks should never be overridable from inside a skill invocation.** The plain-secret check (`E1`) is the clearest example: it's the single most dangerous misconfiguration in the whole matrix, and the fix has to be "reject and explain," never "warn and proceed."
3. **Constraints must be stated before examples, not after.** A MUST-NOT rule appended as a closing caveat gets treated with less weight by the model than the same rule stated as a precondition. Position it first.
4. **Platform variables are injected, never hardcoded per app.** A registry prefix or namespace pattern baked into every generated manifest individually becomes a fleet-wide migration the day it changes. Resolve it from one shared source.
5. **Auto-fix is fine for CRITICAL, not for architectural decisions.** Silently rewriting a probe timeout is safe. Silently choosing a database engine is not. Keep the line between "mechanical fix" and "design decision" explicit in the skill itself.

## Download the Skills

The four skills behind the examples above, sanitized of any org-specific values
(swap the `<cluster.*>` placeholders for your own before using them). These cover
one slice of one contract, treat them as a starting shape to run the same
pipeline against your own documents, not as the complete set:

- [`app-contract.md`](/files/from-contract-to-icontract/app-contract.md): health endpoints, `/metrics`, JSON logging, Dockerfile (Example 1)
- [`scaffold.md`](/files/from-contract-to-icontract/scaffold.md): namespace derivation, ExternalSecret self-service, app deployment (Examples 2, 3, 5)
- [`observability.md`](/files/from-contract-to-icontract/observability.md): three-tier metrics, scrape NetworkPolicy, business-metric prompt (Example 4)
- [`review.md`](/files/from-contract-to-icontract/review.md): the gate skill, pass/fail matrix, auto-fix mode

Drop them into `.claude/skills/` (or your agent's equivalent skill directory) and
fill in the `<cluster.*>` placeholders at the top of `scaffold.md` for your own
registry, namespace prefix, secret backend, and observability namespace.

## Conclusion

A platform contract's real job was never to exist as a PDF, it was to change what gets built. Splitting the pipeline into parse, cluster, expand, constrain, generate, gate, link, inject turned six documents nobody fully internalized into a set of skills that write compliant code on request and refuse to let non-compliant code through the gate. The generate side does the boring 60-70%, correctly and immediately. The gate side catches what the generate side gets wrong, every time, without waiting for a human review slot to open up. Neither half works without the other: generation without a gate is just a faster way to produce plausible-looking violations, and a gate without generation is the same manual compliance burden it always was, just with better error messages.

## Reflections

What's still unresolved is mostly about drift, not about the pipeline itself. The contract PDFs will change, someone will revise a probe timeout or add a new mandatory header, and nothing today automatically re-derives the skills when the source document changes. Right now that's a manual re-authoring step, which means the skill can silently fall behind the contract it was built from, the exact failure mode this whole exercise was meant to solve on the human side.

There's also a subtler risk in how convincing a "mostly compliant" artifact looks. A generated manifest that passes 90% of the gate reads, at a glance, exactly like one that passes 100%, and the 10% gap is precisely the part a rushed reviewer is least likely to catch by eye. The gate has to stay mandatory and automated for that reason; the moment it becomes optional "for simple apps," the whole distance-reduction argument collapses back into hoping people read the PDF.

And the honest open question: is version-controlling the skill alongside the contract (skill version pinned to contract version, no drift between a Confluence edit and a skill file) actually sufficient to keep them in sync over years, or does it just relocate the review burden from "read the contract" to "review the skill diff"? I don't have a clean answer yet. It's a smaller review than the original problem, which is the whole point, but it's not zero, and treating it as zero is exactly the kind of unstated assumption that turns a safe automation into a load-bearing one nobody's watching.
