# Skill: observability

Trigger: user asks about "monitoring", "metrics", "logs", "why can't I see my app
in the dashboard", "add tracing", or when `review` flags G1/G2.

## What this skill does

Wires observability for an app across the platform's three-tier metrics model:

1. System metrics (automatic, no action needed)
2. Application metrics (`/metrics`, mandatory)
3. Business metrics (domain-specific, asked from the user)

Plus logs (stdout JSON, already required by `app-contract`) and optional tracing.

---

## Tier 1: System metrics (automatic)

cAdvisor scrapes every container's CPU, memory, and network I/O with zero app-side
configuration. Nothing to generate here. If a user asks "why don't I see CPU usage,"
the answer is always a scrape or NetworkPolicy problem, never a missing metric.

---

## Tier 2: Application metrics (mandatory)

### How auto-scrape works

The cluster metrics backend watches all pods. Any pod carrying these three
annotations gets scraped automatically, no ServiceMonitor, no extra config:

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8081"      # port the app exposes /metrics on
  prometheus.io/path: "/metrics"  # omit if already /metrics
```

Add this block to the pod template metadata in `08-app.yaml`.

### NetworkPolicy for the scrape

Without this, the annotation above is scraped by nothing, `default-deny-ingress`
blocks the observability namespace by default like everything else:

```yaml
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

### Minimal /metrics per language

**Node.js / TypeScript**
```typescript
import { Registry, collectDefaultMetrics } from 'prom-client'
const register = new Registry()
collectDefaultMetrics({ register })
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType)
  res.send(await register.metrics())
})
```

**Python**
```python
from prometheus_client import generate_latest, REGISTRY, CONTENT_TYPE_LATEST
@app.route('/metrics')
def metrics():
    return generate_latest(REGISTRY), 200, {'Content-Type': CONTENT_TYPE_LATEST}
```

**Go**
```go
import "github.com/prometheus/client_golang/prometheus/promhttp"
http.Handle("/metrics", promhttp.Handler())
```

### Verify the scrape is working

```bash
curl -s http://<METRICS_BACKEND_IP>:<METRICS_BACKEND_PORT>/api/v1/targets | \
  python3 -c "
import json, sys
for t in json.load(sys.stdin)['data']['activeTargets']:
    if '<APP_SLUG>' in t.get('scrapeUrl', ''):
        print('health:', t['health'], '| samples:', t.get('lastSamplesScraped', '?'))
"
# health: up, samples > 0 = working. Pod in droppedTargets = NetworkPolicy blocking it.
```

---

## Tier 3: Business metrics (ask, don't guess)

This is the one tier the skill cannot derive from the contract or the codebase,
it's a product decision, not a platform one. Ask directly:

> "What are the three most important numbers that tell you if your app is working
> well?"

Turn the answer into counters or gauges, using the same client library Tier 2
already installed:

```typescript
import { Counter, Gauge } from 'prom-client'

const ordersProcessed = new Counter({
  name: 'orders_processed_total',
  help: 'Orders successfully processed',
  labelNames: ['status'],
})

const inventoryItems = new Gauge({
  name: 'inventory_items_total',
  help: 'Current inventory count',
  labelNames: ['category'],
})
```

| Tier | Who defines it | Example | Priority |
|------|-----------------|---------|----------|
| System | cAdvisor (automatic) | CPU%, memory%, network | Automatic |
| Application | This skill | `http_request_duration_seconds`, error rate | Mandatory |
| Business | The app owner, asked directly | `orders_processed_total`, `inventory_items_total` | High value |

---

## Logs (mandatory, already required by app-contract)

Logs ship automatically once the app logs JSON to stdout, a log-forwarding daemon
handles the rest. Query pattern once it's flowing:

```logql
{namespace="<NAMESPACE>"} | json | level="error"
```

No app-side log shipping config needed. If logs aren't appearing: check the app is
writing JSON (`kubectl logs` output should start with `{`), not writing to a file.

---

## Tracing (optional, recommended)

```yaml
data:
  OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-collector.<OBSERVABILITY_NAMESPACE>.svc.cluster.local:4318"
  OTEL_SERVICE_NAME: "<APP_SLUG>"
```

Requires an egress NetworkPolicy from the app namespace to the collector on port
4318, same shape as the metrics-scrape policy above, just the other direction.

---

## Checklist

- [ ] `prometheus.io/scrape`, `port`, `path` annotations present
- [ ] `allow-metrics-scrape-app` NetworkPolicy present
- [ ] `/metrics` endpoint returns Prometheus text format
- [ ] At least one business metric defined, asked from the app owner, not invented
- [ ] Logs are JSON to stdout
- [ ] `review` passes G1-G4
