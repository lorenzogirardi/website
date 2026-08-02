# Skill: app-contract

Trigger: user asks to "make my app compliant", "add health endpoints", "fix logging", "add metrics",
or when the `review` skill reports failures on checks B1-B4, G1-G4.

## Purpose

Modify the application source code, not the Kubernetes manifests, so the app meets
the mandatory platform contract requirements:

1. Health endpoints on the management port (`/readiness`, `/liveness`, `/info`, `/shutdown`)
2. Prometheus `/metrics` endpoint on the traffic port
3. JSON structured logging to stdout
4. A Dockerfile built from an approved, non-alpine base image

These are CRITICAL contract requirements. Without them the app cannot be monitored
or health-checked by the cluster, and `review` checks B and G will fail.

---

## CRITICAL rule: always generate COMPLETE files

Never generate patches, diffs, or "add this block" instructions. Read the full current
content of any file being modified, then write the complete replacement, every line,
ready to save. For new files, generate the complete file and state exactly where to
save it.

---

## Step 1: Detect stack and entry point

| Signal | Stack |
|--------|-------|
| `app.py` / `main.py` + `fastapi`/`flask`/`django` | Python |
| `index.js` / `server.js` + `express` | Node.js |
| `main.go` | Go |
| `Application.java` + Spring | Java |

Also check for a separate frontend (React/Vue/Angular with its own build). If present,
generate a `frontend/Dockerfile` too (Step 4).

---

## Step 2: Health endpoints on the management port

The app MUST expose on the management port (never routed publicly):

| Endpoint | Method | Response 200 | Response 503 |
|----------|--------|---------------|---------------|
| `/readiness` | GET | `{"status":"UP"}` | `{"status":"OUT_OF_SERVICE"}` |
| `/liveness` | GET | `{"status":"UP"}` | `{"status":"DOWN"}` |
| `/info` | GET | `{"app":"<slug>","version":"<sha>"}` | - |
| `/shutdown` | GET | `null` | - |

### Python / FastAPI

```python
# mgmt_server.py: runs on the management port, separate from the main app
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
    return {"app": os.getenv("APP_SLUG", "unknown"), "version": os.getenv("APP_VERSION", "dev")}

@mgmt.get("/shutdown")
def shutdown():
    return None

if __name__ == "__main__":
    uvicorn.run(mgmt, host="0.0.0.0", port=int(os.getenv("MGMT_PORT", "8082")))
```

### Node.js / Express

```js
// mgmt.js: management port, separate process from the traffic server
const express = require('express')
const mgmt = express()
const PORT = process.env.MGMT_PORT || 8082

mgmt.get('/readiness', (req, res) => res.json({ status: 'UP' }))
mgmt.get('/liveness',  (req, res) => res.json({ status: 'UP' }))
mgmt.get('/info',      (req, res) => res.json({ app: process.env.APP_SLUG, version: process.env.APP_VERSION }))
mgmt.get('/shutdown',  (req, res) => res.json(null))

mgmt.listen(PORT, () => console.log(JSON.stringify({ level: 'info', msg: 'mgmt server started', port: PORT })))
```

### Go

```go
// mgmt.go
package main

import (
    "encoding/json"
    "net/http"
    "os"
)

func startMgmtServer() {
    mux := http.NewServeMux()
    mux.HandleFunc("/readiness", func(w http.ResponseWriter, r *http.Request) {
        json.NewEncoder(w).Encode(map[string]string{"status": "UP"})
    })
    mux.HandleFunc("/liveness", func(w http.ResponseWriter, r *http.Request) {
        json.NewEncoder(w).Encode(map[string]string{"status": "UP"})
    })
    mux.HandleFunc("/info", func(w http.ResponseWriter, r *http.Request) {
        json.NewEncoder(w).Encode(map[string]string{"app": os.Getenv("APP_SLUG"), "version": os.Getenv("APP_VERSION")})
    })
    go http.ListenAndServe(":"+os.Getenv("MGMT_PORT"), mux)
}
```

> **Alternative**: if modifying the app is too invasive, enable an nginx sidecar on the
> management port instead. It answers the same four endpoints without touching app code.

---

## Step 3: /metrics endpoint (Prometheus format)

### Python / FastAPI
```bash
pip install prometheus-fastapi-instrumentator
```
```python
from prometheus_fastapi_instrumentator import Instrumentator
app = FastAPI()
Instrumentator().instrument(app).expose(app)   # adds /metrics on the traffic port
```

### Node.js / Express
```bash
npm install prom-client
```
```js
const client = require('prom-client')
client.collectDefaultMetrics()

app.get('/metrics', async (req, res) => {
    res.set('Content-Type', client.register.contentType)
    res.end(await client.register.metrics())
})
```

### Go
```bash
go get github.com/prometheus/client_golang/prometheus/promhttp
```
```go
import "github.com/prometheus/client_golang/prometheus/promhttp"
mux.Handle("/metrics", promhttp.Handler())
```

After adding `/metrics`, set the pod scrape annotations in the deployment manifest
(see the `scaffold` skill).

---

## Step 4: JSON structured logging

All logs MUST go to **stdout** in **JSON** format. Required fields: `time` (ISO8601),
`level`, `msg`.

### Python
```bash
pip install python-json-logger
```
```python
import logging, sys
from pythonjsonlogger import jsonlogger

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(jsonlogger.JsonFormatter(
    '%(asctime)s %(levelname)s %(message)s',
    rename_fields={"asctime": "time", "levelname": "level", "message": "msg"}))
logging.root.addHandler(handler)
logging.root.setLevel(logging.INFO)
```

### Node.js
```bash
npm install pino pino-http
```
```js
const pino = require('pino')
const logger = pino({ level: 'info' })   // JSON by default
```

### Go
```bash
go get go.uber.org/zap
```
```go
logger, _ := zap.NewProduction()   // JSON to stdout by default
```

Remove any file-based logging: `FileHandler`/`RotatingFileHandler` in Python, Winston
`transports.File` in Node.js. Never mount `/var/log` for application logs.

---

## Step 5: Dockerfile (approved base images, no alpine)

`FROM <mirror-prefix>/<base-image>:<tag>`, approved, Debian-based, mirrored internally.
**Never `-alpine`**: musl libc causes compatibility issues with native dependencies.

### Node.js
```dockerfile
FROM <mirror-prefix>/node:22-slim AS builder
WORKDIR /app
COPY package*.json tsconfig.json ./
RUN npm install
COPY src ./src
RUN npm run build

FROM <mirror-prefix>/node:22-slim
RUN groupadd -r appuser && useradd -r -g appuser appuser
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY --from=builder /app/dist ./dist
USER appuser
ENV NODE_ENV=production
EXPOSE 8081 8082
CMD ["npm", "start"]
```

### Python
```dockerfile
FROM <mirror-prefix>/python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8081 8082
CMD ["sh", "-c", "python mgmt_server.py & uvicorn app.main:app --host 0.0.0.0 --port 8081"]
```

### Go (multi-stage, distroless)
```dockerfile
FROM <mirror-prefix>/golang:1.23-bookworm AS builder
WORKDIR /app
COPY go.* ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server .

FROM <mirror-prefix>/gcr.io/distroless/static-debian12
COPY --from=builder /app/server /server
EXPOSE 8081 8082
CMD ["/server"]
```

---

## Checklist

- [ ] Management port serves `/readiness`, `/liveness`, `/info`, `/shutdown`
- [ ] Traffic port has `/metrics` in Prometheus format
- [ ] All log output is JSON to stdout (`time`, `level`, `msg` fields)
- [ ] No `FileHandler` or log file mounts
- [ ] Dockerfile uses an approved, mirrored, Debian-based base image (no alpine)
- [ ] Dockerfile exposes both the traffic and management ports
- [ ] `review` passes B1-B6 and G1-G4
