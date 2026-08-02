# Skill: scaffold

Trigger: user asks to "deploy my app", "create environment for", "scaffold app",
or provides a repo / app description.

## What this skill does

Generates the complete Kubernetes manifest set for an app, deriving as much as
possible from a single per-cluster variables file so the developer answers the
minimum number of questions.

---

## Step 1: Gather minimum inputs (ask in ONE message)

```
To deploy your app I need 6 things:
1. App name (e.g. "inventory-tool") → becomes the slug
2. What does it do? (one sentence)
3. Does it need a database? → postgres / mongo / none
4. Does it need cache (Redis) or messaging (RabbitMQ)? → yes/no each
5. Language and runtime?
   → Node.js 22 / Python 3.12 / Go 1.23 / JVM 21
6. Does the app have a separate frontend (React/Vue/Angular)? → yes/no
```

Do NOT ask about ports, Docker, Kubernetes, images, or networking. Those are derived.

---

## Step 2: Derive configuration

From the answers plus the cluster variables file:

```yaml
APP_SLUG:     <lowercase-hyphens-max20>                    # from app name
NAMESPACE:    <cluster.prefix>-<platform>-<APP_SLUG>       # never typed by hand
IMAGE:        <cluster.registry>/<team>/<APP_SLUG>-app:<commit-sha>
INGRESS_URL:  <derived from cluster.ingressPattern>
BASE_IMAGE:   <cluster.mirrorPrefix>/<image>:<tag>          # never alpine
```

`NAMESPACE` and `INGRESS_URL` are never entered manually. Get a new cluster with a
different prefix, and every skill that reads this file resolves the new value with
no per-app edits.

---

## Step 3: Namespace and network policy

```yaml
# 00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <NAMESPACE>
  labels:
    env: <ENV>
    app: <APP_SLUG>
    managed-by: platform
```

`01-network-policies.yaml` always includes, in order: `default-deny-ingress` first,
then `allow-ingress-controller`, `allow-metrics-scrape-app`, and one
`allow-app-to-<dependency>` rule per enabled dependency (DB, Redis, RabbitMQ).

---

## Step 4: Secrets: reference, never a literal value

> **NEVER generate a plain `Secret` kind or a literal credential value in any
> manifest.** Plain secrets in Git are a CRITICAL violation (`review` check E1).

Secrets are pulled at runtime from the organization's secret manager (HashiCorp
Vault or AWS Secrets Manager) via the External Secrets Operator, provisioned once
per cluster as a `ClusterSecretStore`. The skill only ever writes a reference:

```yaml
# 02-external-secret.yaml: Vault backend
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: <APP_SLUG>-db-secret
  namespace: <NAMESPACE>
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: <APP_SLUG>-db-secret
    creationPolicy: Owner
  data:
    - secretKey: POSTGRES_PASSWORD
      remoteRef:
        key: secret/data/<NAMESPACE>/<APP_SLUG>
        property: db_password
```

Same clause, AWS Secrets Manager backend:

```yaml
spec:
  secretStoreRef:
    name: aws-secretsmanager-backend
    kind: ClusterSecretStore
  data:
    - secretKey: POSTGRES_PASSWORD
      remoteRef:
        key: prod/<NAMESPACE>/<APP_SLUG>/db-credentials
        property: password
```

Tell the user:
```
Your app never holds a credential in Git. Put the actual value in Vault (or AWS
Secrets Manager) at the path above, once. Rotate it there whenever you need to,
the running Secret updates automatically within the refresh interval, no new commit.
```

---

## Step 5: Storage (for every stateful component)

```yaml
# 04-storage.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <APP_SLUG>-<component>-pvc
  namespace: <NAMESPACE>
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: <STORAGECLASS_NAME>
  resources:
    requests:
      storage: <SIZE>   # postgres: 5Gi, mongo: 10Gi, rabbitmq: 2Gi
```

Database Deployments MUST use `strategy.type: Recreate` (file locks require it), and
the app MUST have an `initContainer` that waits for the DB before starting.

---

## Step 6: App deployment

```yaml
# 08-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <APP_SLUG>-app
  namespace: <NAMESPACE>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: <APP_SLUG>-app
  template:
    metadata:
      labels:
        app: <APP_SLUG>-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8081"
        prometheus.io/path: "/metrics"
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
      containers:
        - name: app
          image: <IMAGE>
          ports:
            - { name: http, containerPort: 8081 }
            - { name: mgmt, containerPort: 8082 }
          envFrom:
            - configMapRef:
                name: <APP_SLUG>-app-config
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: <APP_SLUG>-db-secret
                  key: DATABASE_URL
          resources:
            requests: { cpu: 200m, memory: 256Mi }
            limits:   { cpu: 500m, memory: 512Mi }
          livenessProbe:
            httpGet: { path: /liveness, port: 8082 }
            initialDelaySeconds: 15
            timeoutSeconds: 5
          readinessProbe:
            httpGet: { path: /readiness, port: 8082 }
            initialDelaySeconds: 15
            timeoutSeconds: 5
          securityContext:
            allowPrivilegeEscalation: false
            capabilities: { drop: [ALL] }
```

`secretKeyRef` above reads from the native `Secret` the External Secrets Operator
creates from the `ExternalSecret` in Step 4, the app container never knows or cares
that the value came from Vault rather than a plain `Secret`.

---

## Step 7: Output to user

```
✅ Your app "[APP_NAME]" is ready to deploy.

   Components: [list what was created]
   Access URL: [INGRESS_URL]

   Next steps:
   1. Put the real credential values in Vault/AWS Secrets Manager at the paths above
   2. Run: kubectl apply -f k8s/
   3. Run `review` before you do, it will catch anything this skill missed
```
