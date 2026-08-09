# Skill: review

Trigger: user asks to "check my app", "review before deploy", "is this ready",
or before any `kubectl apply`.

## What this skill does

Validates an app's manifests against the platform contract. Reports only failures,
passes are silent. Output is plain language, no Kubernetes jargon.

This skill never generates code. Its counterpart, `app-contract` / `scaffold`, does
that. `review` only reads what already exists and decides pass or fail.

---

## How to invoke

```
review                 # reviews k8s/ in the current repo
review k8s/08-app.yaml # reviews a single file
```

---

## Checklist (trimmed)

### A: Image

| # | Check | Pass condition | Fail message |
|---|-------|-----------------|---------------|
| A1 | No `latest` tag | No `image: *:latest` anywhere | "Use your commit SHA, e.g. `myapp:abc1234`" |
| A3 | Non-root user | `runAsNonRoot: true` in pod securityContext | "Add `runAsNonRoot: true, runAsUser: 1000`" |
| A4 | Drop capabilities | `capabilities.drop: [ALL]` in container securityContext | "Add `capabilities: { drop: [ALL] }`" |
| A5 | No public-registry images | Every image resolves to the internal registry | "Docker Hub is prohibited. Replace with `<mirror-prefix>/<image>:<tag>`" |

### B: Health & Ports

| # | Check | Pass condition | Fail message |
|---|-------|-----------------|---------------|
| B1 | Traffic port declared | `containerPort: 8081` | "Add `containerPort: 8081`" |
| B2 | Management port declared | `containerPort: 8082` (app or sidecar) | "Add the management port, or the nginx sidecar" |
| B3 | Liveness probe correct | `port: 8082`, `path: /liveness` | "Liveness probe missing or wrong port/path" |
| B4 | Readiness probe correct | `port: 8082`, `path: /readiness` | "Readiness probe missing or wrong port/path" |
| B6 | Management port not public | Ingress does NOT route to 8082 | "Remove the management port from the Ingress" |

### E: Secrets

| # | Check | Pass condition | Fail message |
|---|-------|-----------------|---------------|
| E1 | No plain `Secret` in Git | Only `ExternalSecret` references present, no literal value in any manifest | "Found a plain Secret or a literal credential. Replace with an `ExternalSecret` referencing Vault or AWS Secrets Manager" |
| E5 | Secrets injected via `secretKeyRef` | Deployment env uses `secretKeyRef`, never `value:` | "Credentials hardcoded in the Deployment spec" |

### G: Metrics & Logging (CRITICAL)

| # | Check | Pass condition | Fail message |
|---|-------|-----------------|---------------|
| G1 | Scrape annotation present | `prometheus.io/scrape`, `port`, `path` all set | "App won't appear in Grafana. Add the three annotations to the pod template" |
| G2 | `/metrics` endpoint exists | App exposes Prometheus format on the declared port | "Add a Prometheus client library, see `app-contract`" |
| G3 | JSON logging | `kubectl logs` output starts with `{` | "Logs must be JSON, the log backend can't parse plaintext" |
| G6 | No sensitive data in logs | No email, credit card, password, or bearer-token patterns in samples | "Potential PII/credential in logs. Violation = immediate namespace shutdown" |

Severity: **CRITICAL** (A1-A6, B1-B6, E1-E7, G1-G4, G6) blocks deploy.
**WARNING** (F*, H*) deploy is allowed, but flagged.

---

## Output format

### All checks pass
```
✅ App "[APP_NAME]" passed all platform contract checks. Ready to deploy.
```

### Failures found
```
⚠️  Found [N] issues in "[APP_NAME]". Fix these before deploying:

CRITICAL (must fix):
  [A3] App runs as root: add `runAsNonRoot: true` to pod securityContext
  [E1] Plain Secret found in k8s/02-secret.yaml: replace with ExternalSecret

WARNING (strongly recommended):
  [G1] Metrics not configured: app won't appear in monitoring dashboards

Run review again after fixing.
```

Every failure line names the check ID that also appears in `app-contract` or
`scaffold`, so the next step is always "go fix it there," never a dead end.

---

## Auto-fix mode

If the user asks to "fix the issues," apply CRITICAL fixes directly to the manifest
files and report what changed. Do not ask permission for CRITICAL fixes, apply them
silently, they're mechanical (a missing annotation, a missing securityContext block),
not architectural decisions.

For WARNING fixes, apply and mention at the end: "Also applied N optional
improvements."

Never auto-fix E1 by inventing a credential value. Generate the `ExternalSecret`
reference and tell the user to populate the path in Vault or AWS Secrets Manager
themselves, this skill never sees or writes a real secret value.
