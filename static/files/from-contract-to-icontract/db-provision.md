# Skill: db-provision

Trigger: user asks to "add a database", "I need to store data", "add persistence", or asks about postgres/mongo.

## What this skill does

Adds a database to an existing app, or helps choose between postgres and mongo.
Generates: PVC, Deployment, Service, Secrets skeleton, metrics exporter, NetworkPolicy updates.

---

## Step 1 - Choose database

Ask the user ONE question only:

```
What kind of data does your app store?
a) Tables, rows, relationships (e.g. users, orders, products) → PostgreSQL
b) Documents, JSON objects, flexible structure (e.g. events, configs, logs) → MongoDB
c) I'm not sure → I'll recommend based on your app
```

### Auto-recommendation logic

If user says "not sure", infer from their app description:
- E-commerce, CRM, HR, finance → **PostgreSQL**
- CMS, catalog, events, logs, real-time feeds → **MongoDB**
- Caching only → suggest **Redis** (redirect to redis section)
- Unknown → default **PostgreSQL** (simpler ops, mature tooling)

---

## Step 2 - Generate storage manifest (`04-storage.yaml`)

### PostgreSQL PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <APP_SLUG>-postgres-pvc
  namespace: <NAMESPACE>
  labels:
    app: <APP_SLUG>
    component: database
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: <STORAGECLASS_NAME>   # from PLATFORM.md
  resources:
    requests:
      storage: 5Gi   # adjust for expected data volume
```

### MongoDB PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <APP_SLUG>-mongo-pvc
  namespace: <NAMESPACE>
  labels:
    app: <APP_SLUG>
    component: database
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: <STORAGECLASS_NAME>
  resources:
    requests:
      storage: 10Gi  # WiredTiger needs more headroom
```

---

## Step 3 - Generate DB manifest (`05-db.yaml`)

### PostgreSQL full manifest

```yaml
# ─── Deployment ────────────────────────────────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <APP_SLUG>-postgres
  namespace: <NAMESPACE>
  labels:
    app: <APP_SLUG>-postgres
    component: database
spec:
  replicas: 1
  strategy:
    type: Recreate    # MANDATORY - PostgreSQL uses file locks
  selector:
    matchLabels:
      app: <APP_SLUG>-postgres
  template:
    metadata:
      labels:
        app: <APP_SLUG>-postgres
        component: database
    spec:
      securityContext:
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
      containers:
        - name: postgres
          image: <MIRROR_PREFIX>/postgres:16
          imagePullPolicy: IfNotPresent
          ports:
            - name: postgres
              containerPort: 5432
          envFrom:
            - secretRef:
                name: <APP_SLUG>-db-secret
          resources:
            requests: { cpu: 200m, memory: 256Mi }
            limits:   { cpu: 500m, memory: 512Mi }
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
              subPath: postgres      # prevents "data directory not empty" errors
          readinessProbe:
            exec:
              command: [pg_isready, -U, $(POSTGRES_USER), -d, $(POSTGRES_DB)]
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 5
            failureThreshold: 6
          livenessProbe:
            exec:
              command: [pg_isready, -U, $(POSTGRES_USER)]
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: <APP_SLUG>-postgres-pvc
---
# ─── Service ───────────────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: <APP_SLUG>-postgres
  namespace: <NAMESPACE>
spec:
  selector:
    app: <APP_SLUG>-postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
  clusterIP: None  # headless - resolves directly to pod IP
---
# ─── Exporter (metrics) ────────────────────────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <APP_SLUG>-postgres-exporter
  namespace: <NAMESPACE>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: <APP_SLUG>-postgres-exporter
  template:
    metadata:
      labels:
        app: <APP_SLUG>-postgres-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9187"
        prometheus.io/path: "/metrics"
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
        - name: postgres-exporter
          image: prometheuscommunity/postgres-exporter:v0.15.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 9187
          env:
            - name: DATA_SOURCE_NAME
              valueFrom:
                secretKeyRef:
                  name: <APP_SLUG>-db-secret
                  key: EXPORTER_DSN   # postgres://user:pass@<APP_SLUG>-postgres:5432/db?sslmode=disable
          resources:
            requests: { cpu: 50m, memory: 64Mi }
            limits:   { cpu: 200m, memory: 128Mi }
---
apiVersion: v1
kind: Service
metadata:
  name: <APP_SLUG>-postgres-exporter
  namespace: <NAMESPACE>
spec:
  selector:
    app: <APP_SLUG>-postgres-exporter
  ports:
    - port: 9187
      targetPort: 9187
```

### MongoDB full manifest

```yaml
# ─── Deployment ────────────────────────────────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <APP_SLUG>-mongo
  namespace: <NAMESPACE>
  labels:
    app: <APP_SLUG>-mongo
    component: database
spec:
  replicas: 1
  strategy:
    type: Recreate    # MANDATORY - WiredTiger storage engine file lock
  selector:
    matchLabels:
      app: <APP_SLUG>-mongo
  template:
    metadata:
      labels:
        app: <APP_SLUG>-mongo
        component: database
    spec:
      securityContext:
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
      containers:
        - name: mongo
          image: mongo:7.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: mongo
              containerPort: 27017
          envFrom:
            - secretRef:
                name: <APP_SLUG>-db-secret
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: <APP_SLUG>-db-secret
                  key: MONGO_INITDB_ROOT_USERNAME
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: <APP_SLUG>-db-secret
                  key: MONGO_INITDB_ROOT_PASSWORD
          resources:
            requests: { cpu: 200m, memory: 256Mi }
            limits:   { cpu: 500m, memory: 512Mi }
          volumeMounts:
            - name: data
              mountPath: /data/db
          readinessProbe:
            exec:
              command: [mongosh, --eval, "db.adminCommand('ping')"]
            initialDelaySeconds: 15
            periodSeconds: 5
            timeoutSeconds: 10   # DB I/O probe
            failureThreshold: 6
          livenessProbe:
            exec:
              command: [mongosh, --eval, "db.adminCommand('ping')"]
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 10
            failureThreshold: 3
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: <APP_SLUG>-mongo-pvc
---
# ─── Service ───────────────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: <APP_SLUG>-mongo
  namespace: <NAMESPACE>
spec:
  selector:
    app: <APP_SLUG>-mongo
  ports:
    - name: mongo
      port: 27017
      targetPort: 27017
  clusterIP: None
---
# ─── Exporter ─────────────────────────────────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <APP_SLUG>-mongo-exporter
  namespace: <NAMESPACE>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: <APP_SLUG>-mongo-exporter
  template:
    metadata:
      labels:
        app: <APP_SLUG>-mongo-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9216"
        prometheus.io/path: "/metrics"
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
        - name: mongo-exporter
          image: percona/mongodb_exporter:0.40
          imagePullPolicy: IfNotPresent
          args: ["--collect-all", "--mongodb.uri=$(MONGODB_URI)"]
          ports:
            - containerPort: 9216
          env:
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: <APP_SLUG>-db-secret
                  key: EXPORTER_URI  # mongodb://user:pass@<APP_SLUG>-mongo:27017/admin
          resources:
            requests: { cpu: 50m, memory: 64Mi }
            limits:   { cpu: 200m, memory: 128Mi }
```

---

## Step 4 - Update `02-secrets.yaml`

### PostgreSQL secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <APP_SLUG>-db-secret
  namespace: <NAMESPACE>
type: Opaque
data:
  # Generate: echo -n "value" | base64
  POSTGRES_USER: CHANGE_ME         # base64 of db username
  POSTGRES_PASSWORD: CHANGE_ME     # base64 of db password
  POSTGRES_DB: CHANGE_ME           # base64 of database name
  EXPORTER_DSN: CHANGE_ME          # base64 of postgres://user:pass@<APP_SLUG>-postgres:5432/db?sslmode=disable
```

### MongoDB secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <APP_SLUG>-db-secret
  namespace: <NAMESPACE>
type: Opaque
data:
  MONGO_INITDB_ROOT_USERNAME: CHANGE_ME
  MONGO_INITDB_ROOT_PASSWORD: CHANGE_ME
  EXPORTER_URI: CHANGE_ME          # base64 of mongodb://user:pass@<APP_SLUG>-mongo:27017/admin
```

---

## Step 5 - Update `01-network-policies.yaml`

Append to existing network policies:

```yaml
---
# Allow app to reach DB
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
  namespace: <NAMESPACE>
spec:
  podSelector:
    matchLabels:
      app: <APP_SLUG>-<postgres|mongo>
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: <APP_SLUG>-app
      ports:
        - port: <5432|27017>
---
# Allow metrics scrape for DB exporter
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-metrics-scrape-db-exporter
  namespace: <NAMESPACE>
spec:
  podSelector:
    matchLabels:
      app: <APP_SLUG>-<postgres|mongo>-exporter
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: <OBSERVABILITY_NAMESPACE>
      ports:
        - port: <9187|9216>
```

---

## Step 6 - Update app `initContainer`

In `08-app.yaml`, add to the app Deployment initContainers:

```yaml
initContainers:
  - name: wait-for-db
    image: <MIRROR_PREFIX>/debian:12-slim
    imagePullPolicy: IfNotPresent
    command:
      - bash
      - -c
      - |
        echo "Waiting for database..."
        # PostgreSQL
        until echo > /dev/tcp/<APP_SLUG>-postgres/5432 2>/dev/null; do
          echo "postgres not ready, retrying in 2s..."
          sleep 2
        done
        # OR MongoDB
        until echo > /dev/tcp/<APP_SLUG>-mongo/27017 2>/dev/null; do
          echo "mongo not ready, retrying in 2s..."
          sleep 2
        done
        echo "Database ready."
```

---

## Step 7 - Tell the user

```
✅ Database ([postgres|mongo]) added to "[APP_NAME]".

What was created:
• Persistent storage (data survives restarts)
• Database server
• Monitoring (you'll see DB metrics in Grafana)

⚠️  Before deploying:
1. Open k8s/02-secrets.yaml and fill in the CHANGE_ME values
2. To generate base64: run → echo -n "mypassword" | base64

Connection string for your app:
  PostgreSQL: postgresql://[user]:[password]@<APP_SLUG>-postgres:5432/[dbname]
  MongoDB:    mongodb://[user]:[password]@<APP_SLUG>-mongo:27017/[dbname]
```

---

## Redis provision (quick pattern)

If user wants Redis instead (cache, sessions):

```yaml
# 06-redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <APP_SLUG>-redis
  namespace: <NAMESPACE>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: <APP_SLUG>-redis
  template:
    metadata:
      labels:
        app: <APP_SLUG>-redis
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9121"
    spec:
      securityContext:
        runAsUser: 999
        runAsGroup: 999
      containers:
        - name: redis
          image: <MIRROR_PREFIX>/redis:7.4
          imagePullPolicy: IfNotPresent
          command: [redis-server, --appendonly, "yes"]
          ports:
            - containerPort: 6379
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { cpu: 300m, memory: 256Mi }
          volumeMounts:
            - name: data
              mountPath: /data
        - name: redis-exporter
          image: <MIRROR_PREFIX>/oliver006/redis_exporter:v1.63.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 9121
          resources:
            requests: { cpu: 50m, memory: 32Mi }
            limits:   { cpu: 100m, memory: 64Mi }
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: <APP_SLUG>-redis-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: <APP_SLUG>-redis
  namespace: <NAMESPACE>
spec:
  selector:
    app: <APP_SLUG>-redis
  ports:
    - name: redis
      port: 6379
      targetPort: 6379
    - name: metrics
      port: 9121
      targetPort: 9121
```

Connection string: `redis://<APP_SLUG>-redis:6379`
