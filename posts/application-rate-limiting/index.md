# Application Rate Limit

I needed to implement rate limiting within an application for reasons I'll get into in a follow-up post. When you start thinking about this, you basically have two paths:

1. Logic embedded directly in the application code
2. A sidecar container that handles the rate limiting role

Both work. Both have trade-offs. Let me go through each one.

## The Code Way

This is the simpler approach on the surface, but it comes with some annoying limitations. It can only be reused for applications in the same programming language. And adding rate limiting logic inside the application creates a secondary role — meaning the request interceptor will consume CPU and may produce *false* metrics if you're not tracking it carefully.

For a Flask application, you add `Flask-Limiter` to `requirements.txt`, then wire it up like this:

```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(
    get_remote_address,
    app=app,
    storage_uri="memory://",
)
```

Then decorate the specific route you want to limit:

```python
@limiter.limit("28 per second")
```

### Hands-on

I changed the limit from 28 per second down to 2 per second to test. Build the image:

```
$ docker build -t lgirardi/pytbakrated .
[+] Building 11.1s (9/9) FINISHED
 => [internal] load build definition from Dockerfile  0.0s
 => [internal] load .dockerignore  0.0s
 => [internal] load metadata for docker.io/library/python:3.9-alpine  0.6s
 => [internal] load build context  0.0s
 => transferring context: 4.47kB  0.0s
 => CACHED [1/4] FROM docker.io/library/python:3.9-alpine@sha256:8bda1e9a98fa4e87ff6e3a7682f496532b06fcbae10326a59c8656126051d4df  0.0s
 => [2/4] COPY . /app  0.0s
 => [3/4] WORKDIR /app  0.0s
 => [4/4] RUN pip install -r requirements.txt  9.9s
 => exporting to image  0.4s
 => exporting layers  0.4s
 => writing image sha256:583ec19905662414ad6bdb3b0da1042ec799ec46f8a676fac6553364cd02bf1e  0.0s
 => naming to docker.io/lgirardi/pytbakrated
```

Run it:

```
$ docker run -p 5000:5000 lgirardi/pytbakrated
 * Serving Flask app 'app'
 * Debug mode: off
2023-02-11 10:36:06,264 INFO werkzeug MainThread : WARNING: This is a development server.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:5000
 * Running on http://172.17.0.2:5000
```

Test with a loop:

```
$ while true; do curl -I localhost:5000/api/fib/1 && sleep 0.4;done
HTTP/1.1 200 OK
Server: Werkzeug/2.2.2 Python/3.9.16
...

HTTP/1.1 429 TOO MANY REQUESTS
Server: Werkzeug/2.2.2 Python/3.9.16
...
```

The container logs confirm what's happening:

```
2023-02-11 10:37:54,359 INFO werkzeug Thread-34 : 172.17.0.1 - - [11/Feb/2023 10:37:54] "HEAD /api/fib/1 HTTP/1.1" 200 -
2023-02-11 10:37:55,196 INFO flask-limiter Thread-38 : ratelimit 2 per 1 second (172.17.0.1) exceeded at endpoint: fib
2023-02-11 10:37:55,197 INFO werkzeug Thread-38 : 172.17.0.1 - - [11/Feb/2023 10:37:55] "HEAD /api/fib/1 HTTP/1.1" 429 -
```

It works. But here's the thing I don't like about it:

- It's using the same pool of connections as the app
- It's impacting the same metrics I'm using for autoscaling
- It's stealing resources from the app itself
- It's a really simple rate limit that can fight with other things

## The Sidecar Way

Service meshes and API gateways embrace this pattern, but you don't need a full infrastructure overhaul for it. You can run a standalone sidecar container just for this purpose.

I chose **Envoy** because it's flexible and I'd already been working with it in Istio environments. Fair warning: it's *really* flexible — sometimes too much, in a way that makes configuration confusing.

### Kubernetes Pod Configuration

Add the sidecar to the pod spec:

```yaml
- name: sidecar
  image: envoyproxy/envoy:v1.22-latest
  resources:
    limits:
      cpu: 100m
      memory: 150Mi
    requests:
      cpu: 30m
      memory: 55Mi
  ports:
    - name: http
      containerPort: 5002
      protocol: TCP
  volumeMounts:
    - name: sidecar-config
      mountPath: "/etc/envoy"
      readOnly: true
volumes:
  - name: sidecar-config
    configMap:
      name: pytbakt-configmap
```

### Envoy ConfigMap

The complete Envoy configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pytbakt-configmap
  labels:
    app: pytbak
  namespace: pytbak
data:
  envoy.yaml: |
    static_resources:
      listeners:
      - name: listener_0
        address:
          socket_address:
            address: 0.0.0.0
            port_value: 5002
        filter_chains:
        - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              stat_prefix: ingress_http
              route_config:
                name: local_route
                virtual_hosts:
                - name: local_service
                  domains: ["*"]
                  routes:
                  - match:
                      prefix: "/"
                    route:
                      cluster: pytbak
              http_filters:
              - name: envoy.filters.http.local_ratelimit
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
                  stat_prefix: http_local_rate_limiter
                  token_bucket:
                    max_tokens: 29
                    tokens_per_fill: 29
                    fill_interval: 1s
                  filter_enabled:
                    runtime_key: local_rate_limit_enabled
                    default_value:
                      numerator: 100
                      denominator: HUNDRED
                  filter_enforced:
                    runtime_key: local_rate_limit_enforced
                    default_value:
                      numerator: 100
                      denominator: HUNDRED
                  response_headers_to_add:
                  - append_action: OVERWRITE_IF_EXISTS_OR_ADD
                    header:
                      key: x-local-rate-limit
                      value: 'true'
                  local_rate_limit_per_downstream_connection: false
              - name: envoy.filters.http.router
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
      clusters:
      - name: pytbak
        connect_timeout: 0.25s
        type: LOGICAL_DNS
        dns_lookup_family: V4_ONLY
        lb_policy: ROUND_ROBIN
        load_assignment:
          cluster_name: service_a
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: localhost
                    port_value: 5000
```

Let me break down what matters here.

**Listener** — Envoy listens on 5002 so external traffic hits the sidecar, not the app directly:

```yaml
- name: listener_0
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 5002
```

**Routing** — everything gets routed to the `pytbak` cluster:

```yaml
- name: local_service
  domains: ["*"]
  routes:
  - match:
      prefix: "/"
    route:
      cluster: pytbak
```

**Cluster** — this is the reverse proxy target, the app running on port 5000 in the same pod:

```yaml
clusters:
- name: pytbak
  connect_timeout: 0.25s
  type: LOGICAL_DNS
  dns_lookup_family: V4_ONLY
  lb_policy: ROUND_ROBIN
  load_assignment:
    cluster_name: service_a
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: localhost
              port_value: 5000
```

**Rate limit** — token bucket algorithm, 29 tokens refilled every second:

```yaml
token_bucket:
  max_tokens: 29
  tokens_per_fill: 29
  fill_interval: 1s
```

### Hands-on

For the test, I dropped it to 2 per second:

```yaml
token_bucket:
  max_tokens: 2
  tokens_per_fill: 2
  fill_interval: 1s
```

The pod comes up with 2/2 containers running:

```
pytbak pytbak-stable-bd648fd46-nj95m 2/2 Running 0 13s
```

Testing against the live endpoint:

```
$ while true;do curl -I http://oracolo.k8s.it/api/fib/1 && sleep 0.4;done

HTTP/1.1 200 OK
Date: Sat, 11 Feb 2023 11:29:48 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1
Connection: keep-alive
vary: Accept-Encoding
x-envoy-upstream-service-time: 2

HTTP/1.1 200 OK
...

HTTP/1.1 429 Too Many Requests
Date: Sat, 11 Feb 2023 11:29:49 GMT
Content-Type: text/plain
Content-Length: 18
Connection: keep-alive
x-local-rate-limit: true
```

Notice the `x-local-rate-limit: true` header on the 429 — that's Envoy telling you it rejected the request, not the app.

## Conclusion

Both solutions work. The difference is *where* the impact lands. The code way is simpler to set up but bleeds into your app's resource budget and can confuse your autoscaling metrics. The sidecar way isolates the function cleanly — Envoy handles the gate, and the app just processes what gets through. If you need rate limiting that doesn't interfere with your application's own performance story, the sidecar approach gives you exactly that.

I'll share the full context for why I needed this in the first place in an upcoming post.
