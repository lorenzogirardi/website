# AI Agentic Development Changes Who Builds Software — and That's an Infrastructure Problem

### Table of Contents

  * The Shift Is Already Happening
  * The Problem Nobody Prepared For
  * The Design Principle: Safe by Default
  * The Platform Contract — What Every App Must Be
    * Supported Languages and Base Images
    * Supported Components
    * T-shirt Sizing
    * Port Contract
    * Health Endpoints
    * Secrets: Sealed, Always
    * Images: Commit SHA, Never Latest
    * Network: Default Deny, Every Time
  * The Three-Tier Monitoring Contract
    * System Tier — Automatic
    * Framework Tier — App Metrics
    * Business Tier — What the App Actually Does
  * The Review Gate — 35+ Checks Before Deploy
  * The Helm Chart — Five Questions, Full Platform
  * The CI/CD Pipeline — AI App Meets GitOps
  * The Skill Pipeline — AI Onboarding an AI App
  * The RACI Collapse
  * Conclusion

---

Here we are. Somewhere in the last twelve months, something quietly changed.

## The Shift Is Already Happening

People who have never written a line of code in their professional lives are now producing working software. Not prototypes in the "I dragged boxes around in a no-code tool" sense. Real applications: Python backends with data models and business logic, React frontends that talk to APIs, database schemas that reflect years of domain knowledge no engineer could have extracted through interviews alone.

They built these with Claude, Cursor, Copilot. They described what they wanted in plain language. The agent coded it. They reviewed the output the way they review a Word document — checking the content, not the syntax.

This is not hype. I've watched it happen.

The question the industry keeps asking is "will AI replace developers?" That's the wrong question, and it's wasting time. The right question is: **when non-technical people can build software, what are the operational consequences?**

Because there are operational consequences. Real ones.

---

## The Problem Nobody Prepared For

Run an AI coding bootcamp. Give a group of non-technical employees — analysts, brand managers, merchandisers, finance people — access to Claude or Cursor for a week. Ask them to build something that solves a real problem in their job.

They will build things. Things that actually run.

Then ask: where do you put them?

The options are obvious, and they are all wrong.

**Their laptop.** The app lives. Nobody else can use it. It dies when the laptop sleeps.

**A shared development server.** Everyone is root. One person's experiment kills everyone else's. Data from different projects bleeds together. There is no deployment process — it's `scp` and `nohup`. Eventually someone pastes real data into a seed script and you have a security incident.

**Production.** Naaaa. A vibe-coded app — built by someone who doesn't know what a liveness probe is, whose "error handling" is whatever Claude decided to generate — in production, serving real users, touching real data. The support cost alone would be catastrophic.

None of these work. And if you give people the ability to build software without giving them a safe place to run it, you have created a problem, not solved one.

**Vibe-Env** is the answer: a purpose-built Kubernetes environment where vibe-coded apps can be deployed, tested, and shown to stakeholders — with every guardrail enforced by infrastructure, not documentation.

---

## The Design Principle: Safe by Default

"Safe" gets applied to a lot of things in tech that are not safe. So let me be specific.

Every protection in this platform is **enforced by infrastructure**, not left to the app creator to implement correctly. A non-technical user cannot be expected to configure NetworkPolicies, know that `latest` image tags are mutable, understand why plain Kubernetes Secrets in Git are readable by anyone with repo access, or choose non-root containers.

The platform does all of this automatically. Where it cannot be automated, it blocks deployment if the condition is not met. The Helm chart validates required fields and fails loudly. The CI pipeline blocks on secrets and critical CVEs. The review step runs 35+ checks. The cluster blocks internet egress entirely.

Safe by default. No exceptions.

```mermaid
graph TB
    vpn(["🔒 VPN<br/>corporate only<br/>no public internet"])

    subgraph cluster["k3s Cluster — private VIP via MetalLB"]
        traefik["Traefik Ingress<br/>MetalLB VIP"]

        subgraph obs["vibe-observability namespace"]
            grafana["Grafana<br/>dashboards"]
            vm["VictoriaMetrics<br/>metrics pull"]
            loki["Loki<br/>+ Minio<br/>7d retention"]
            otel["OTEL<br/>Collector"]
            promtail["Promtail<br/>DaemonSet"]
        end

        subgraph nsA["&lt;prefix&gt;-vibe-&lt;app-A&gt;"]
            appA["app"]
            dbA[("DB<br/>PVC")]
            appA --- dbA
        end

        subgraph nsB["&lt;prefix&gt;-vibe-&lt;app-B&gt;"]
            appB["app"]
            dbB[("DB<br/>PVC")]
            appB --- dbB
        end

        subgraph nsN["&lt;prefix&gt;-vibe-&lt;app-N&gt;"]
            appN["app"]
            dbN[("DB<br/>PVC")]
            appN --- dbN
        end
    end

    egress["🚫 Egress blocked<br/>cluster-wide<br/>Platform/Arch owns"]

    vpn --> traefik
    traefik --> appA
    traefik --> appB
    traefik --> appN
    appA -.->|"/metrics + logs"| obs
    appB -.->|"/metrics + logs"| obs
    appN -.->|"/metrics + logs"| obs
    promtail -.->|"ship stdout"| loki
    cluster -.- egress

    style obs fill:#fef3c7,stroke:#d97706
    style nsA fill:#dbeafe,stroke:#3b82f6
    style nsB fill:#dbeafe,stroke:#3b82f6
    style nsN fill:#dbeafe,stroke:#3b82f6
    style egress fill:#fee2e2,stroke:#dc2626
```

---

## The Platform Contract — What Every App Must Be

The contract is the center of the platform. It is not a style guide or a recommendation. It is a binary gate: the app meets it, or it does not deploy.

### Supported Languages and Base Images

All base images come from an internal Harbor mirror. Docker Hub is blocked at cluster egress. The approved base images are Debian-based — never Alpine. musl libc compatibility issues with native extensions and certain runtimes cause subtle runtime failures that non-technical creators cannot debug.

| Language | Runtime | Base Image (Harbor mirror) | Notes |
|----------|---------|---------------------------|-------|
| TypeScript / JavaScript | Node.js 22 | `mirror/node:22-slim` | Default for new apps |
| TypeScript / JavaScript | Node.js 20 | `mirror/node:20-slim` | LTS — use if locked |
| Python | Python 3.12 | `mirror/python:3.12-slim` | Debian-slim |
| Python | Python 3.11 | `mirror/python:3.11-slim` | |
| Go | Go 1.23 | `mirror/golang:1.23-bookworm` (build) + `mirror/gcr.io/distroless/static-debian12` (run) | Multi-stage mandatory |
| Java / Kotlin / Scala | JVM 21 | `mirror/eclipse-temurin:21-jre-jammy` | Ubuntu-jammy |
| Java / Kotlin / Scala | JVM 17 | `mirror/eclipse-temurin:17-jre-jammy` | LTS |
| Rust | Rust 1.78 | `mirror/rust:1.78-slim-bookworm` (build) + `mirror/gcr.io/distroless/static-debian12` (run) | Multi-stage |
| PHP | PHP 8.3 | `mirror/php:8.3-fpm-bookworm` | |

Multi-stage Dockerfile examples per runtime:

**Node.js (TypeScript)**
```dockerfile
FROM mirror/node:22-slim AS builder
WORKDIR /app
COPY package*.json tsconfig.json ./
RUN npm install
COPY src ./src
RUN npm run build

FROM mirror/node:22-slim
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd -r appuser && useradd -r -g appuser appuser
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY --from=builder /app/dist ./dist
USER appuser
EXPOSE 8081 8082
CMD ["npm", "start"]
```

**Python**
```dockerfile
FROM mirror/python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt \
    && groupadd -r appuser && useradd -r -g appuser appuser
COPY . .
USER appuser
EXPOSE 8081 8082
CMD ["sh", "-c", "python mgmt_server.py & uvicorn app.main:app --host 0.0.0.0 --port 8081"]
```

**Go (multi-stage, distroless runtime)**
```dockerfile
FROM mirror/golang:1.23-bookworm AS builder
WORKDIR /app
COPY go.* ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server .

FROM mirror/gcr.io/distroless/static-debian12
COPY --from=builder /app/server /server
EXPOSE 8081 8082
CMD ["/server"]
```

If a runtime is not in the table, it must be mirrored to Harbor before use. The creator opens a request to Platform; Platform mirrors it; Platform provides the Harbor path. The creator never pulls from the internet.

### Supported Components

Every component a vibe app can use is pre-defined, pre-sized, and pre-wired:

| Component | Image (Harbor mirror) | Purpose |
|-----------|----------------------|---------|
| **app** | creator-built, pushed to Harbor | main application |
| **nginx** | `mirror/nginx:1.27` | reverse proxy, static files, SPA fallback |
| **postgres** | `mirror/postgres:16` | relational database |
| **mongo** | `mirror/mongo:7.0` | document database |
| **redis** | `mirror/redis:7.4` | cache, sessions |
| **rabbitmq** | `mirror/rabbitmq:3.13-management` | message broker |
| **postgres-exporter** | `mirror/prometheuscommunity/postgres-exporter:v0.15.0` | Prometheus metrics for PostgreSQL |
| **mongodb-exporter** | `mirror/percona/mongodb_exporter:0.40` | Prometheus metrics for MongoDB |
| **redis-exporter** | `mirror/oliver006/redis_exporter:v1.63.0` | Prometheus metrics for Redis |

All in the same namespace. All wired together by the Helm chart. The creator chooses which ones they need; the platform wires the rest.

### T-shirt Sizing

Resource limits are not optional. Every container must declare requests and limits. The platform provides five standard sizes:

| Size | CPU request | CPU limit | Memory request | Memory limit | Use case |
|------|-------------|-----------|----------------|--------------|---------|
| `xs` | 50m | 200m | 64Mi | 128Mi | static sites, minimal APIs |
| `sm` | 100m | 300m | 128Mi | 256Mi | light web apps |
| `md` | 200m | 500m | 256Mi | 512Mi | standard apps (default) |
| `lg` | 500m | 1000m | 512Mi | 1Gi | data-heavy apps |
| `xl` | 1000m | 2000m | 1Gi | 2Gi | intensive workloads |

DB components always get `md` minimum, regardless of app size.

The non-technical creator does not set CPU and memory values in YAML. They pick a t-shirt size. The Helm chart translates it:

```yaml
# From values.yaml — the creator sees this
app:
  size: md      # xs / sm / md / lg / xl

# What the Helm template renders into the Deployment
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### Port Contract

Two ports, hard requirements:

| Port | Name | Purpose | Exposed externally? |
|------|------|---------|---------------------|
| **8081** | `http` | Application traffic — API, frontend, everything the user interacts with | Yes — via Ingress |
| **8082** | `mgmt` | Health probes, metrics — cluster-internal only | **Never** |

Port 8082 is never reachable from outside the cluster. The review gate (check B6) fails deployment if the Ingress routes traffic to port 8082.

### Health Endpoints

Every long-running app must expose four endpoints on port 8082. The port is management-only and never routed externally.

| Endpoint | Response 200 | Response 503 |
|----------|-------------|-------------|
| `GET /readiness` | `{"status":"UP"}` | `{"status":"OUT_OF_SERVICE"}` |
| `GET /liveness` | `{"status":"UP"}` | `{"status":"DOWN"}` |
| `GET /info` | `{"app":"slug","version":"sha"}` | — |
| `GET /shutdown` | `null` | — |

The platform adds `readinessProbe` and `livenessProbe` to every Deployment automatically, pointing at port 8082. If the app can't implement these endpoints, the platform provides an nginx sidecar that responds to them without touching the app code:

```yaml
# values.yaml — enable the mgmt sidecar if app can't implement health endpoints
app:
  mgmtSidecar:
    enabled: true
```

The sidecar nginx config is auto-generated:
```nginx
# nginx-mgmt-sidecar ConfigMap (auto-generated by Helm)
server {
  listen 8082;
  location /readiness { return 200 '{"status":"UP"}'; add_header Content-Type application/json; }
  location /liveness  { return 200 '{"status":"UP"}'; add_header Content-Type application/json; }
  location /shutdown  { return 200 'null'; add_header Content-Type application/json; }
  location /info      { return 200 '{"git":{"branch":"unknown"}}'; add_header Content-Type application/json; }
}
```

### Secrets: Sealed, Always

Plain Kubernetes Secrets in Git are not secret. Anyone with repository read access can `base64 -d` them. This is a well-known problem and it still shows up in incident reports every month.

Vibe-Env uses **Bitnami Sealed Secrets**. The full flow — from plaintext to running pod — never exposes the secret in Git or in the pod spec:

```mermaid
flowchart LR
    subgraph platform["Platform Team (onboarding)"]
        plain["plain Secret manifest<br/>(never committed)"]
        ks["kubeseal CLI<br/>+ cluster public key"]
        plain -->|seal| ks
    end

    subgraph git["Git Repo (safe to commit)"]
        sealed["SealedSecret YAML<br/>encryptedData: AgB3x..."]
    end

    subgraph cluster["k3s Cluster"]
        direction TB
        argocd2["ArgoCD<br/>sync"]
        controller["Sealed Secrets<br/>controller<br/>(vibe-observability)"]
        k8ssecret["Kubernetes Secret<br/>(in-memory, namespace-scoped)"]
        app2["App Pod<br/>env var via secretKeyRef"]

        argocd2 -->|"apply SealedSecret"| controller
        controller -->|"decrypt with cluster private key"| k8ssecret
        k8ssecret -->|"secretKeyRef"| app2
    end

    keybackup[("🔑 Cluster private key<br/>BACKUP REQUIRED<br/>kube-system/sealed-secrets-key")]

    ks -->|output| sealed
    sealed -->|"git push → ArgoCD detects"| argocd2
    controller -.->|"key lives here"| keybackup

    never["🚫 NEVER<br/>plain Secret in Git<br/>credentials in ConfigMap<br/>credentials in Dockerfile ENV<br/>credentials in Deployment spec"]

    style platform fill:#dbeafe,stroke:#3b82f6
    style git fill:#dcfce7,stroke:#16a34a
    style cluster fill:#fef3c7,stroke:#d97706
    style keybackup fill:#f3e8ff,stroke:#9333ea
    style never fill:#fee2e2,stroke:#dc2626
```

The sealing workflow:

```bash
# 1. Create a plain Secret — locally, NEVER commit this file
kubectl create secret generic my-app-secret \
  --from-literal=DATABASE_URL="postgres://user:pass@my-app-postgres:5432/appdb" \
  --from-literal=API_KEY="sk-..." \
  --dry-run=client -o yaml > /tmp/secret-plain.yaml

# 2. Seal with cluster public key — output is safe to commit
kubeseal \
  --cert cluster.pub \
  --namespace platform-vibe-my-app \
  --format yaml \
  < /tmp/secret-plain.yaml > k8s/02-sealed-secrets.yaml

# 3. Delete the plaintext immediately
rm /tmp/secret-plain.yaml

# 4. Commit the sealed version — ArgoCD syncs it
git add k8s/02-sealed-secrets.yaml && git commit -m "chore: add app secret"
```

The encrypted SealedSecret is namespace-scoped and cluster-scoped — it cannot be decrypted in a different namespace or on a different cluster. Safe to commit to GitHub.

The Deployment template injects secrets only via `secretKeyRef`. No plaintext appears anywhere in the pod spec:

```yaml
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: my-app-secret
        key: DATABASE_URL
  - name: API_KEY
    valueFrom:
      secretKeyRef:
        name: my-app-secret
        key: API_KEY
```

E1 in the review gate: a plain `Secret` kind found in Git is a CRITICAL violation. Deployment is blocked and a security incident is raised.

### Images: Commit SHA, Never Latest

```yaml
image: harbor.internal.example.com/my-app:a3f91bc   # correct — immutable
image: harbor.internal.example.com/my-app:latest    # CRITICAL violation — deploy blocked
```

`latest` is a mutable tag. Between deploy and the next pod restart, a new push can silently replace what the tag points to. In an environment maintained by someone without an on-call rotation, a silent image change is the kind of thing that causes a Friday afternoon incident that nobody looks at until Monday.

The Helm template enforces this at render time:
```yaml
image: {{ .Values.cluster.registry }}/{{ .Values.app.image.repository }}:{{
  .Values.app.image.tag | required "app.image.tag is required — use commit SHA, never 'latest'"
}}
```

### Network: Default Deny, Every Time

Every namespace has the same runtime anatomy: nginx reverse proxy in front, app container on two ports (8081 for traffic, 8082 for management only), stateful components only reachable via explicit NetworkPolicy, and the observability stack cross-namespace but allowed only for metrics scrape and log shipping.

```mermaid
flowchart TB
    user(["👤 VPN User"])
    traefik["Traefik Ingress"]

    user -->|"HTTPS internal URL"| traefik

    subgraph ns["&lt;prefix&gt;-vibe-&lt;app-slug&gt; namespace"]
        direction TB

        deny["🛡 default-deny-ingress NetworkPolicy<br/>applied FIRST — mandatory"]

        subgraph web["webserver"]
            nginx["nginx :80<br/>reverse proxy<br/>static files / SPA"]
        end

        subgraph application["application"]
            app["app container<br/>:8081 http<br/>:8082 mgmt — NEVER public<br/><br/>/readiness  /liveness<br/>/metrics    /info"]
        end

        subgraph storage["stateful components (optional)"]
            postgres[("PostgreSQL<br/>:5432<br/>PVC<br/>Recreate")]
            mongo[("MongoDB<br/>:27017<br/>PVC<br/>Recreate")]
            redis[("Redis<br/>:6379<br/>PVC")]
            rmq["RabbitMQ<br/>:5672<br/>PVC"]
        end

        subgraph exporters["metrics exporters"]
            pgexp["postgres-exporter<br/>:9187"]
            mgexp["mongo-exporter<br/>:9216"]
        end
    end

    subgraph observability["vibe-observability (cross-namespace)"]
        vm["VictoriaMetrics"]
        loki["Loki"]
        otel["OTEL Collector"]
    end

    traefik -->|"allow-ingress-controller :80"| nginx
    nginx -->|"proxy :8081"| app
    app -->|"allow-app-to-db"| postgres
    app -->|"OR"| mongo
    app -.->|"allow-app-to-redis"| redis
    app -.->|"allow-app-to-rabbitmq"| rmq
    postgres --- pgexp
    mongo --- mgexp
    app -->|"allow-metrics-scrape :8081"| vm
    pgexp -->|"allow-metrics-scrape :9187"| vm
    mgexp -->|"allow-metrics-scrape :9216"| vm
    app -->|"stdout JSON → Promtail"| loki
    app -->|"allow-otel-egress :4318"| otel

    style deny fill:#fee2e2,stroke:#dc2626
    style web fill:#dbeafe,stroke:#3b82f6
    style application fill:#dcfce7,stroke:#16a34a
    style storage fill:#fef3c7,stroke:#d97706
    style exporters fill:#f3e8ff,stroke:#9333ea
    style observability fill:#fef3c7,stroke:#d97706
```

First manifest applied to every namespace, before anything else:

```yaml
# 01-network-policies.yaml — applied FIRST, mandatory
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: platform-vibe-my-app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

Every allowed path requires an explicit policy. Five standard policies are generated by the Helm chart for a typical app with PostgreSQL:

```yaml
---
# Traefik (kube-system) → app:8081
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller
  namespace: platform-vibe-my-app
spec:
  podSelector:
    matchLabels:
      app: my-app-app
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              app.kubernetes.io/name: traefik
      ports:
        - port: 8081
---
# app → postgres:5432
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
  namespace: platform-vibe-my-app
spec:
  podSelector:
    matchLabels:
      app: my-app-postgres
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: my-app-app
      ports:
        - port: 5432
---
# VictoriaMetrics (observability ns) → app:8081 — metrics scrape
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-metrics-scrape-app
  namespace: platform-vibe-my-app
spec:
  podSelector:
    matchLabels:
      app: my-app-app
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: platform-vibe-observability
      ports:
        - port: 8081
```

If a policy is missing, traffic is dropped silently. Unknown traffic should be denied, not allowed and investigated later. Egress from the cluster to the internet is blocked at cluster level by Platform. No app can exfiltrate data by accident.

---

## The Three-Tier Monitoring Contract

Monitoring is CRITICAL in the contract — not advisory, not recommended. Apps that don't expose `/metrics` or don't log JSON cannot be deployed. The review checks G1–G4 are at the same severity level as image and network violations.

The platform defines a three-tier monitoring model. System metrics are free. Framework and Business metrics are the app's responsibility.

```mermaid
flowchart LR
    subgraph apps["vibe app pods (all namespaces *-vibe-*)"]
        a1["app-A<br/>stdout JSON"]
        a2["app-B<br/>stdout JSON"]
        metrics1["/metrics<br/>Prometheus fmt"]
        metrics2["/metrics<br/>Prometheus fmt"]
    end

    subgraph obs["vibe-observability"]
        direction TB
        promtail["Promtail<br/>DaemonSet<br/>/var/log/pods"]
        vm["VictoriaMetrics<br/>auto-scrape<br/>annotations"]
        loki["Loki<br/>log aggregation"]
        minio[("Minio<br/>S3 backend")]
        grafana["Grafana<br/>dashboards<br/>+ Loki Explore"]
        otel["OTEL Collector<br/>:4317 gRPC<br/>:4318 HTTP"]
    end

    a1 -->|stdout| promtail
    a2 -->|stdout| promtail
    promtail -->|push| loki
    loki --> minio
    metrics1 -->|"pull :8081"| vm
    metrics2 -->|"pull :8081"| vm
    vm --> grafana
    loki --> grafana
    a1 -.->|traces| otel
    a2 -.->|traces| otel

    note["📌 App MUST:<br/>1. log JSON to stdout<br/>2. expose /metrics<br/>3. pod annotation:<br/>prometheus.io/scrape: true<br/><br/>Both CRITICAL —<br/>blocks deploy if missing"]

    style obs fill:#fef3c7,stroke:#d97706
    style note fill:#fee2e2,stroke:#dc2626
```

### System Tier — Automatic

cAdvisor runs as a DaemonSet. VictoriaMetrics scrapes it automatically. Every app gets CPU, memory, network I/O, and disk I/O per container — without writing a single line of code.

These metrics answer "is the infrastructure healthy?" They don't tell you if the app is working.

### Framework Tier — App Metrics

The app must expose `/metrics` on port 8081 in Prometheus format. The pod annotation tells VictoriaMetrics to scrape it:

```yaml
# Auto-generated by Helm chart — creator does not write this
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8081"
    prometheus.io/path: "/metrics"
```

The minimum required metrics — generated by the `app-contract` skill:

**Node.js / TypeScript**
```typescript
import { Registry, Counter, Histogram, Gauge, collectDefaultMetrics } from 'prom-client'

const register = new Registry()
collectDefaultMetrics({ register })

export const httpRequestTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  registers: [register],
})

export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request latency',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5],
  registers: [register],
})

export const activeConnections = new Gauge({
  name: 'http_active_connections',
  help: 'Active HTTP connections',
  registers: [register],
})

// Expose on /metrics
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType)
  res.send(await register.metrics())
})
```

**Python / FastAPI**
```python
from prometheus_fastapi_instrumentator import Instrumentator
from prometheus_client import Counter, Histogram

app = FastAPI()

# Default metrics from instrumentator (request count, latency, status codes)
Instrumentator().instrument(app).expose(app)

# Add custom metrics as needed
db_query_total = Counter(
    'db_query_total', 'Total database queries', ['operation', 'table']
)
db_query_duration = Histogram(
    'db_query_duration_seconds', 'Database query duration', ['operation']
)
```

**Go**
```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequests = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "http_requests_total", Help: "Total HTTP requests"},
        []string{"method", "path", "status"},
    )
    httpDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request latency",
            Buckets: []float64{0.01, 0.05, 0.1, 0.5, 1, 2, 5},
        },
        []string{"method", "path"},
    )
)

func init() {
    prometheus.MustRegister(httpRequests, httpDuration)
}

// Expose on /metrics at port 8081
mux.Handle("/metrics", promhttp.Handler())
```

These metrics answer "is the application behaving correctly?" — request rate, error rate, latency by endpoint, database query count and duration.

### Business Tier — What the App Actually Does

This is the layer that makes monitoring actually useful for the creator. The platform cannot define these metrics — they depend on what the app does. But the contract requires them.

During the skill pipeline, the assistant asks the creator: *"What are the 3 most important numbers that tell you if your app is working?"* Their answer becomes the metric definitions.

Examples:
- "How many orders were processed today?" → `orders_processed_total` Counter with `status` label
- "How many items are in low stock?" → `inventory_low_stock_items` Gauge with `category` label
- "How many search queries per minute?" → `searches_total` Counter

```python
# Python — business metrics for an inventory app
from prometheus_client import Counter, Gauge

orders_processed = Counter(
    'orders_processed_total',
    'Total orders processed',
    ['status']  # fulfilled, cancelled, pending
)

low_stock_items = Gauge(
    'inventory_low_stock_items',
    'Products with stock below threshold',
    ['category']
)

# Call in business logic
orders_processed.labels(status='fulfilled').inc()
low_stock_items.labels(category='electronics').set(current_count)
```

```typescript
// Node.js — business metrics for a reporting app
import { Counter, Gauge } from 'prom-client'

const reportsGenerated = new Counter({
  name: 'reports_generated_total',
  help: 'Total reports generated',
  labelNames: ['report_type', 'format'],
})

const activeUsers = new Gauge({
  name: 'active_users_current',
  help: 'Currently active users in the last 15 minutes',
})
```

These metrics answer "is the app doing what it is supposed to do?" — not just "is it up and fast?" but "is business actually flowing through it?"

The three tiers together give the creator a Grafana dashboard that answers four questions in sequence: Is everything running? (status row — green/red per pod). Is the infrastructure healthy? (system row — CPU, memory). Is the app behaving correctly? (framework row — request rate, error rate, DB connections). Is the business flowing? (business row — orders, searches, inventory).

### Logging Contract

All logs must go to stdout in JSON format. Required fields: `time` (ISO8601), `level`, `msg`.

**Node.js** — pino (JSON by default):
```typescript
import pino from 'pino'
const logger = pino({ level: process.env.LOG_LEVEL || 'info' })

// Every log call produces structured JSON
logger.info({ userId: user.id, action: 'login' }, 'User authenticated')
// → {"time":"2026-01-01T10:00:00Z","level":"info","msg":"User authenticated","userId":"abc123","action":"login"}
```

**Python** — python-json-logger:
```python
import logging, sys
from pythonjsonlogger import jsonlogger

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(jsonlogger.JsonFormatter(
    '%(asctime)s %(levelname)s %(message)s',
    rename_fields={"asctime": "time", "levelname": "level", "message": "msg"}
))
logging.root.addHandler(handler)
logging.root.setLevel(logging.INFO)

# Usage
log = logging.getLogger(__name__)
log.info("Order created", extra={"order_id": order.id, "amount": order.total})
# → {"time":"2026-01-01T10:00:00Z","level":"INFO","msg":"Order created","order_id":"ord-123","amount":142.50}
```

**Go** — zap (JSON to stdout by default):
```go
import "go.uber.org/zap"
logger, _ := zap.NewProduction()
logger.Info("Order created",
    zap.String("order_id", order.ID),
    zap.Float64("amount", order.Total),
)
```

File-based logging is explicitly prohibited. No `FileHandler`, no `RotatingFileHandler`, no Winston `transports.File`. File logs are invisible to Loki and lost when the pod restarts.

Loki receives every JSON line from every pod stdout via Promtail DaemonSet. Seven-day retention. The creator queries them in Grafana:

```logql
# All logs for this app
{namespace="platform-vibe-my-app"}

# Filter errors
{namespace="platform-vibe-my-app"} | json | level="error"

# Correlate with a trace ID
{namespace="platform-vibe-my-app"} | json | trace_id="abc-123"
```

---

## The Review Gate — 35+ Checks Before Deploy

The `review` skill is the deployment gate. CRITICAL findings block deployment — there is no override. WARNING findings are documented and raised to the app owner.

**A — Image** (all CRITICAL)

| Check | What it verifies |
|-------|-----------------|
| A1 | No `:latest` image tag anywhere |
| A2 | Dockerfile declares `--platform linux/amd64` |
| A3 | `runAsNonRoot: true` in pod securityContext |
| A4 | `capabilities: { drop: [ALL] }` in container securityContext |
| A5 | No bare Docker Hub image references in any manifest |
| A6 | App image references internal Harbor registry |

**B — Health & Ports** (all CRITICAL)

| Check | What it verifies |
|-------|-----------------|
| B1 | `containerPort: 8081` declared (application traffic) |
| B2 | `containerPort: 8082` declared (management) |
| B3 | `livenessProbe` on port 8082 path `/liveness` |
| B4 | `readinessProbe` on port 8082 path `/readiness` |
| B5 | Probe `timeoutSeconds >= 5` (10 if probe checks DB) |
| B6 | Port 8082 NOT present in any Ingress rule |

**C — Networking** (all CRITICAL)

| Check | What it verifies |
|-------|-----------------|
| C1 | `default-deny-ingress` NetworkPolicy present |
| C2 | Explicit allow: ingress controller → app:8081 |
| C3 | Explicit allow: observability ns → app:8081 (metrics scrape) |
| C4 | Explicit allow: app → DB port (if DB enabled) |
| C5 | Explicit allow: app → redis:6379 (if Redis enabled) |
| C6 | Explicit allow: app → rabbitmq:5672 (if RabbitMQ enabled) |

**D — Storage** (all CRITICAL)

| Check | What it verifies |
|-------|-----------------|
| D1 | PVC exists for every DB before the DB Deployment |
| D2 | DB Deployment `strategy.type: Recreate` (not RollingUpdate — file locks) |
| D3 | initContainer waiting for DB before app starts |
| D4 | No data in container filesystem (ephemeral — data lost on restart) |

**E — Secrets** (all CRITICAL)

| Check | What it verifies |
|-------|-----------------|
| E1 | No `kind: Secret` in Git — only `kind: SealedSecret` |
| E2 | No passwords or tokens in ConfigMaps |
| E3 | No credentials in Dockerfile `ENV` instructions |
| E4 | No `CHANGE_ME` or `REPLACE_ME` placeholders remaining |
| E5 | Credentials injected via `secretKeyRef`, never as plain `value:` |
| E6 | SealedSecret `metadata.namespace` matches deployment namespace |
| E7 | Sealed with correct controller name and namespace |

**G — Metrics & Observability** (G1–G4, G6 are CRITICAL)

| Check | What it verifies |
|-------|-----------------|
| G1 | Pod has `prometheus.io/scrape`, `prometheus.io/port`, `prometheus.io/path` |
| G2 | App actually exposes `/metrics` in Prometheus format |
| G3 | App logs to stdout in JSON format |
| G4 | No file-based logging (FileHandler, log file mounts) |
| G5 | Log lines contain `time`, `level`, `msg` fields |
| G6 | No PII, credentials, or sensitive data patterns in log output |

G6 is worth calling out separately. The skill scans recent `kubectl logs` output for email addresses, credit card patterns, password fields in JSON, and Bearer tokens. A hit triggers a security incident and immediate namespace shutdown — no warnings.

**F — Resources** and **H — Ingress** produce WARNINGs (deploy allowed, compliance incomplete).

A typical first review on a vibe-coded app looks like:

```
⚠️  Found 6 issues in "Inventory Checker". Fix these before deploying:

CRITICAL (must fix):
  [A1] Image tag is ':latest' — use commit SHA (e.g. my-app:a3f91bc)
  [A3] App runs as root — add runAsNonRoot: true, runAsUser: 1000 to pod securityContext
  [C1] default-deny-ingress NetworkPolicy missing from 01-network-policies.yaml
  [E1] Plain Secret found in k8s/02-secrets.yaml — replace with SealedSecret
  [G1] No prometheus.io/scrape annotation on pod — app won't appear in Grafana
  [G3] App logs are plaintext (not JSON) — Loki can't parse them

WARNING (strongly recommended):
  [F2] nginx container has no memory limit — set to 128Mi
  [H1] Ingress URL doesn't match cluster pattern

Run /vibe-review again after fixing.
```

The creator fixes each finding, re-runs the review, and repeats until it passes. The AI assistant can apply CRITICAL fixes automatically — it modifies the manifest files and the source code without asking.

---

## The Helm Chart — Five Questions, Full Platform

The entire namespace — app container, database, nginx proxy, network policies, RBAC, probes, metrics annotations, HPA — deploys from a single Helm chart. The `values.yaml` is the only thing the creator touches.

```yaml
# values.yaml — the five questions
app:
  slug: "inventory-checker"
  name: "Inventory Checker"
  image:
    repository: "inventory-checker-app"
    tag: "a3f91bc"          # commit SHA from CI
  size: md                   # xs / sm / md / lg / xl

cluster:
  prefix: "platform"
  registry: "harbor.internal.example.com"
  mirrorPrefix: "harbor.internal.example.com/dockerhub-proxy"
  observabilityNamespace: "platform-vibe-observability"
  ingressHostPattern: "{prefix}-{slug}.internal.example.com"

postgres:
  enabled: true
  image: "harbor.internal.example.com/dockerhub-proxy/postgres:16"
  storage: 5Gi
  auth:
    existingSecret: "inventory-checker-postgres-secret"

ingress:
  enabled: true
  tls: true
```

One command:

```bash
helm upgrade --install inventory-checker ./helm/vibe-stack -f values.yaml
```

What Helm renders includes security context baked in — the creator never writes these:

```yaml
# From the real Helm template (app.yaml)
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
      initContainers:
        - name: wait-for-postgres
          image: mirror/debian:12-slim
          command: ['sh', '-c',
            'until (echo >/dev/tcp/inventory-checker-postgres/5432) 2>/dev/null;
             do echo "waiting for postgres..."; sleep 2; done']
      containers:
        - name: app
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
          livenessProbe:
            httpGet:
              path: /liveness
              port: 8082
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readiness
              port: 8082
            timeoutSeconds: 5
            failureThreshold: 3
```

Non-root, no capabilities, management port separated from application port, initContainer waiting for the database. None of this requires the creator to know it exists.

---

## The CI/CD Pipeline — AI App Meets GitOps

The deploy pipeline is fully automatic from the moment it is set up. The creator pushes code; the app is live in approximately two minutes. They never touch the cluster directly.

```mermaid
flowchart LR
    dev(["👤 App Creator"])

    subgraph git["CODE"]
        repo["Git repo<br/>(main branch)"]
    end

    subgraph ci["CI / Actions"]
        direction TB
        t1["1. Trufflehog<br/>secret scan"]
        t2["2. Trivy<br/>CRITICAL only"]
        t3["3. Build<br/>linux/amd64"]
        t1 --> t2 --> t3
    end

    subgraph reg["IMAGES"]
        harbor["Harbor<br/>internal registry<br/>&lt;slug&gt;:&lt;sha7&gt;<br/>NEVER :latest"]
    end

    subgraph cd["CD / GitOps"]
        tag["commit tag<br/>update to values.yaml"]
        argocd["ArgoCD<br/>auto-sync<br/>prune + selfHeal"]
        tag --> argocd
    end

    dev -->|"git push main"| repo
    repo -->|"webhook trigger"| ci
    t3 -->|"push image"| harbor
    harbor -->|"tag written to values.yaml"| tag
    argocd -->|"sync"| cluster(["k3s Cluster<br/>~2 min total"])

    style git fill:#dbeafe,stroke:#3b82f6
    style ci fill:#fef9c3,stroke:#eab308
    style reg fill:#dcfce7,stroke:#16a34a
    style cd fill:#f3e8ff,stroke:#9333ea
```

**CI — Build, Scan, Push** (generated by the skill pipeline, runs on every push to main):

```yaml
# .github/workflows/ci.yml — complete real pipeline
name: CI — Build, Scan, Push
on:
  push:
    branches: [main]

jobs:
  build-scan-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set image tag (commit SHA)
        id: meta
        run: echo "tag=${GITHUB_SHA:0:7}" >> $GITHUB_OUTPUT

      - name: Build image (no push — scan first)
        uses: docker/build-push-action@v5
        with:
          platforms: linux/amd64
          push: false
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.tag }}
          outputs: type=docker,dest=/tmp/image.tar

      - name: Check for Docker Hub references
        run: |
          VIOLATIONS=$(grep -rn \
            -e 'image: \+[a-zA-Z0-9._-]*:[a-zA-Z0-9._-]*$' \
            -e 'image: \+docker\.io/' \
            --include="*.yaml" --include="*.yml" --include="Dockerfile*" \
            . | grep -v 'harbor\.' | grep -v 'cluster\.mirrorPrefix' || true)
          if [ -n "$VIOLATIONS" ]; then
            echo "::error::Docker Hub images detected. All images must come from Harbor."
            echo "$VIOLATIONS"
            exit 1
          fi

      - name: Scan secrets with Trufflehog
        uses: trufflesecurity/trufflehog@main
        with:
          extra_args: --only-verified

      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          severity: CRITICAL      # HIGH advisory in alpha — intentional
          exit-code: 1
          ignore-unfixed: true

      - name: Push image to Harbor
        if: github.ref == 'refs/heads/main'
        uses: docker/build-push-action@v5
        with:
          platforms: linux/amd64
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.tag }}
```

Trivy scans only CRITICAL CVEs, not HIGH. In an alpha environment with vibe-coded apps using whatever base image the AI chose, blocking on HIGH would produce a wall of unfixable findings that discourages deployment entirely. CRITICAL CVEs are genuinely dangerous. HIGH CVEs are visible in the SARIF upload, addressed at upgrade time.

**CD — GitOps via ArgoCD** (runs after CI succeeds):

```yaml
# .github/workflows/cd-argocd.yml — does NOT run kubectl or helm
name: CD — Update image tag for ArgoCD sync
on:
  workflow_run:
    workflows: [CI — Build, Scan, Push]
    branches: [main]
    types: [completed]

jobs:
  update-values:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - uses: actions/checkout@v4

      - name: Update image tag in values file
        run: |
          NEW_TAG="${{ github.event.workflow_run.head_sha }}"
          NEW_TAG="${NEW_TAG:0:7}"
          sed -i "s|^  tag:.*|  tag: \"${NEW_TAG}\"|" "helm/vibe-stack/values.yaml"

      - name: Commit and push tag update
        run: |
          git config user.name "vibe-cd-bot"
          git commit -m "chore(deploy): update image to ${NEW_TAG} [skip ci]"
          git push
```

This workflow does not run `kubectl` or `helm`. It commits the new image tag to the repository. ArgoCD watches the repository, detects the change, and syncs the cluster.

```yaml
# ArgoCD Application — auto-sync with selfHeal
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: inventory-checker
  namespace: argocd
spec:
  source:
    repoURL: https://git.internal.example.com/vibe/inventory-checker
    targetRevision: main
    path: helm/vibe-stack
    helm:
      valueFiles: [values.yaml]
  destination:
    namespace: platform-vibe-inventory-checker
  syncPolicy:
    automated:
      prune: true
      selfHeal: true    # reverts any manual kubectl changes
```

`selfHeal: true` means if a creator manually applies something with `kubectl apply`, ArgoCD reverts it on the next sync cycle. The Git repository is the source of truth. Always.

---

## The Skill Pipeline — AI Onboarding an AI App

Here is where it gets meta. We use an AI coding assistant to transform a vibe-coded app into a contract-compliant deployable workload. The tool onboarding your app is the same kind of tool you used to build it.

The pipeline is seven sequential steps, each a skill file the assistant reads as context.

```mermaid
flowchart TD
    input(["👤 Vibe App Creator<br/>raw code from Claude/Cursor<br/>no infra knowledge"])

    subgraph context["Platform Context (read at start)"]
        cv["CLAUDE-vibe-app.md<br/>contract requirements"]
        pt["PLATFORM-template.md<br/>cluster topology + values"]
    end

    input -->|"5 questions: name, DB, size, components, runtime"| s1

    subgraph skills["Skills — applied in order"]
        direction TB

        s0["1. app-contract.md<br/>Modifies app SOURCE CODE:<br/>health endpoints :8082<br/>/metrics Prometheus format<br/>JSON structured logging stdout"]

        s1["2. scaffold.md<br/>Generates full k8s manifest set:<br/>namespace · NetworkPolicies · RBAC<br/>Deployment · Service · Ingress<br/>SealedSecret stubs · ConfigMap · HPA"]

        s2["3. db-provision.md<br/>If DB needed:<br/>PVC + Postgres or Mongo pod<br/>exporter sidecar<br/>NetworkPolicy update<br/>initContainer wait-for-db"]

        s3["4. observability.md<br/>Pod annotations → VictoriaMetrics<br/>JSON stdout confirmed → Loki<br/>OTEL tracing endpoint (optional)"]

        s4["5. gitops.md<br/>GitHub Actions CI:<br/>Trufflehog · Trivy · Harbor push<br/>ArgoCD Application + AppProject<br/>CD: tag commit → auto-sync"]

        s_tests["6. tests.md<br/>Test pyramid:<br/>Unit · Integration · E2E smoke<br/>Wired into CI — blocks deploy on fail"]

        s5["7. review.md<br/>35+ checks:<br/>CRITICAL: A1-A6 · B1-B6 · C1-C4<br/>           D1-D4 · E1-E7 · G1-G4,G6<br/>WARNING:   F1-F3 · H1-H2"]

        s0 --> s1 --> s2 --> s3 --> s4 --> s_tests --> s5
    end

    s5 -->|FAIL| fix["🔧 fix issues<br/>→ re-review"]
    fix --> s5

    s5 -->|PASS| output

    subgraph output["Contract-Compliant App"]
        direction LR
        o1["✓ immutable image in Harbor<br/>✓ namespace isolated + NetworkPolicies<br/>✓ Sealed Secrets — no plain creds in Git"]
        o2["✓ health endpoints + probes<br/>✓ /metrics + JSON logs wired<br/>✓ GitOps → ArgoCD auto-deploy<br/>✓ RBAC scoped to namespace<br/>✓ test pyramid in CI"]
    end

    context -.->|informs| skills

    style input fill:#dbeafe,stroke:#3b82f6
    style context fill:#f3e8ff,stroke:#9333ea
    style skills fill:#fef9c3,stroke:#eab308
    style s5 fill:#fef3c7,stroke:#d97706
    style fix fill:#fee2e2,stroke:#dc2626
    style output fill:#dcfce7,stroke:#16a34a
```

The K8s manifest structure is deterministic — every app has the same numbered files in the same order:

```
k8s/
  00-namespace.yaml           ← Namespace + managed-by: vibe-env label
  01-network-policies.yaml    ← default-deny first, then explicit allows
  02-sealed-secrets.yaml      ← SealedSecret stubs with sealing instructions
  03-configmap.yaml           ← Non-sensitive config, LOG_LEVEL, OTEL endpoint
  04-storage.yaml             ← PVC for every stateful component
  05-db.yaml                  ← postgres OR mongo (strategy: Recreate — mandatory)
  06-redis.yaml               ← Redis (if enabled)
  07-rabbitmq.yaml            ← RabbitMQ (if enabled)
  08-app.yaml                 ← Deployment + Service (non-root, resource limits)
  09-nginx.yaml               ← Reverse proxy (Debian-based)
  10-ingress.yaml
  11-metrics-exporters.yaml   ← postgres-exporter, mongodb-exporter, redis-exporter
  12-hpa.yaml
```

The assistant generates COMPLETE files, never patches. The creator replaces files wholesale. No partial edits, no "add this block at line 47." Non-technical users cannot apply partial edits reliably.

---

## The RACI Collapse

Traditional platform RACI has three columns: Platform, Product, Security. Each has clearly bounded accountabilities.

In Vibe-Env, a vibe app creator is all three simultaneously. They built the app, they own data governance, they own incident response for their namespace.

**What Platform commits to:**
- Cluster maintenance and accepting improvement proposals
- Onboarding (namespace + kubeconfig + ArgoCD setup)
- Security violation response (immediate namespace shutdown)
- Maintaining the observability stack

**What Platform does not commit to:**
- Fixing your app when it crashes
- Any incident response SLA
- Monitoring your namespace

If your app breaks at 2 AM on a Saturday, nobody fixes it before Monday. This is intentional. This environment is for apps that don't yet deserve 24/7 attention. If your app needs that, it needs a proper production process.

**The path from vibe app to production** is not this environment. If an app proves its value:
1. Architecture reviews the codebase
2. A professional team takes ownership
3. Full production process: cloud or on-prem EKS equivalent, SonarQube, OAuth2/SSO, Service Catalog, proper RACI

The vibe prototype may be a complete rewrite informed by what the prototype proved. That is the correct outcome.

---

## Conclusion

The infrastructure problem created by AI agentic development is not going away. More people will build software with these tools. The quality will improve. The velocity will increase.

The organizations that figure out the operational model will capture the value. The ones that don't will either block usage — losing the productivity gain — or let it run unsupervised — accumulating technical debt and security risk at a pace that is very hard to recover from.

A purpose-built sandbox with an explicit platform contract is the minimum viable operational model. Not because contracts are bureaucratic overhead, but because they are the thing that makes "yes, build things" a safe answer instead of a deferred problem.

The contract is the boundary between "the AI coded it" and "the platform runs it." Everything else — the Helm chart, the CI pipeline, the skill pipeline, the review gate — is infrastructure in service of that boundary.

What follows is harder: how does a vibe app become a production service? Who maintains the code when the creator moves on? Those questions don't have clean answers yet. But they have a prerequisite: a place where the apps can run safely while the answers are worked out.

That prerequisite is solvable. We solved it.

