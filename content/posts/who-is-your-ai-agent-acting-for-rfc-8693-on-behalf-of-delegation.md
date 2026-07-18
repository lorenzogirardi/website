---
title: Who Is Your AI Agent Acting For? RFC 8693 On-Behalf-Of Delegation
date: 2026-07-18
draft: false
description: How I built a local POC where every LLM and MCP call carries
  both   the human's and the agent's identity, using Keycloak and RFC 8693
  token   exchange. Per-user policy, audit and revocation for autonomous agents.
featuredImage: /images/featured.jpg
---
### Table of Contents

  * Introduction
  * The Problem: agents are anonymous proxies
  * Enter RFC 8693: Token Exchange, On-Behalf-Of
  * The Architecture
  * The Identity Flow, Step by Step
  * Token Anatomy
  * Where Authorization Actually Happens
  * Security Properties
  * Conclusion
  * Reflections



Here we are. Everyone is wiring AI agents to real systems — Kubernetes clusters, CI pipelines, internal APIs — and almost nobody is asking the boring question first: **when the agent calls a tool, who is it?**

## Introduction

I've been playing with MCP tool servers and agentic loops for a while, and there was one thing that made me crazy: every downstream system sees the agent's service account. Always. The human who asked for the task disappears at the first hop.

So I built a small, fully local POC to answer one question: can every hop of an agentic workflow — the LLM proxy, every single MCP tool call — carry **both identities**, the human *and* the agent, in a token that's cryptographically real and independently verifiable?

Spoiler: yes. The standard has existed since 2020. It's **RFC 8693 Token Exchange**, and Keycloak speaks it out of the box.

In this article, I'll walk you through the architecture: a broker that exchanges the user's token for a delegated one, an agent that never sees the user's raw credential, and an audit trail that can finally answer *"what did alice actually do through this agent?"*

![Delegated identity flow overview](/images/agent-identity-rfc-8693-on-behalf-of/featured.jpg)

Everything runs in Docker on localhost — no cloud, no VPN, no TLS ceremony. Keycloak, a Python broker, a FastAPI agent, LiteLLM, a mock MCP server, Redis. That's it.

## The Problem: agents are anonymous proxies

The classic setup: alice logs into some portal, submits a task, an agent picks it up and starts calling tools with its own service-account credentials. Every MCP call arrives as:

```json
{ "sub": "agent-service" }
```

The tool server can answer *"is this agent allowed?"* — but not *"is **alice** allowed to do this **via** this agent?"* Those are very different questions:

- **Permissions**: alice can deploy to staging, bob can deploy to prod. With a shared service account, both can do everything the agent can.
- **Audit**: the action is attributed to a generic service account. Good luck with that during an incident review.
- **Revocation**: alice leaves the company, her session dies — but the agent keeps running with its own credentials, hours later.
- **Rate limiting**: per-agent quota instead of per-user quota. One noisy user starves everyone.

Could you just pass alice's access token straight to the agent? Naaaa... Now the agent holds a raw user credential, can impersonate her *fully* (not just for this task), and you've built a credential-leaking machine with a tool-calling loop attached.

## Enter RFC 8693: Token Exchange, On-Behalf-Of

**RFC 8693** defines a standard OAuth2 grant where a trusted party exchanges one token for another. In the **On-Behalf-Of (OBO)** variant, the exchange takes the user's token as *subject* and a service identity as *actor*, and mints a new JWT carrying both:

```json
{
  "sub": "8c8af53c-...",              // the human (alice)
  "act": { "sub": "agent-service" },  // the agent acting for her
  "iss": "http://localhost:8180/realms/poc"
}
```

**`sub`** = who owns the action. **`act.sub`** = who is executing it. Every downstream system that validates this token can enforce rules on both — and it's a real **RS256 JWT signed by Keycloak**, not something the gateway invented.

## The Architecture

Seven containers, all local:

| Port | Container | Role |
|---|---|---|
| 8180 | `poc-keycloak` | Real IdP (Keycloak 24), runs the RFC 8693 exchange |
| 8081 | `poc-obo-exchange` | OBO broker — **sole holder** of the exchange client secret |
| 8082 | `poc-agent` | AI agent: tool-calling loop, grant store, audit endpoints |
| 8083 | `poc-mcp-mock` | MCP Streamable HTTP server with 4 demo tools |
| 4000 | `poc-litellm` | OpenAI-compatible LLM proxy (Ollama / OpenAI / Anthropic) |
| 8080 | `poc-webapp` | Identity-flow visualizer (simulates the gateway) |
| 6379 | `poc-redis` | Grant store, AES-256-GCM encrypted at rest |

Three Keycloak clients define the trust topology:

```
poc-webapp          public PKCE app     ← human logs in here
agent-service       service account     ← the agent's own identity (actor)
exchange-app        confidential        ← holds the skeleton key; runs RFC 8693
```

The design choice I care most about: the `exchange-app` client secret — the **skeleton key** that can mint delegated tokens for any user — lives in exactly one small, auditable service: the `obo-exchange` broker. The agent never touches it. If it isn't there, it can't leak.

## The Identity Flow, Step by Step

```mermaid
sequenceDiagram
    actor Human as Human (alice)
    participant KC as Keycloak
    participant GW as Gateway / Webapp
    participant OBO as obo-exchange
    participant Agent as Agent
    participant LLM as LiteLLM /v1
    participant MCP as MCP tools

    Note over Human,KC: Step 1 — Login
    Human->>KC: POST /token (alice)
    KC-->>Human: user JWT {sub=alice, aud=exchange-app}

    Note over Human,OBO: Step 2 — Task submit, gateway intercepts
    Human->>GW: POST /task + Bearer user JWT
    GW->>OBO: POST /exchange (subject_token=user JWT)
    OBO->>KC: RFC 8693 exchange (subject=alice, actor=agent-service)
    KC-->>OBO: OBO JWT {sub=alice, act={sub=agent-service}}
    GW->>Agent: POST /a2a/run + Bearer OBO JWT
    Note right of GW: user JWT never reaches the agent

    Note over Agent,MCP: Step 3 — Agent executes
    Agent->>Agent: store grant (AES-256-GCM, Redis)
    loop tool-calling loop
        Agent->>LLM: /v1/chat/completions + OBO JWT
        Agent->>MCP: tools/call + OBO JWT
        Note right of MCP: every hop sees sub=alice, act=agent-service
    end
    Agent-->>Human: {status: COMPLETED}
```

Four things worth noticing:

1. The **user JWT stops at the gateway**. What crosses into the agent backend is only the delegated token.
2. The agent stores the grant **encrypted at rest** (AES-256-GCM in Redis), keyed by run id.
3. The OBO token is short-lived (1h) but comes with a **rotating refresh token** — the broker can renew it offline, so a long-running task survives without the human being present. The `act` claim is preserved across refreshes.
4. Operator-only audit endpoints reconstruct everything after the fact:

```bash
curl http://localhost:8082/admin/instances/$RUN_ID/identity | python3 -m json.tool
curl http://localhost:8082/admin/instances/$RUN_ID/trace    | python3 -m json.tool
```

The logs alone are already worth the exercise:

```
[OBO] run=abc123 sub=8c8af53c act=agent-service has_refresh=True
[MCP] run=abc123 tools/call sub=8c8af53c act=agent-service ok=True
```

## Token Anatomy

User JWT, straight from Keycloak login:

```json
{
  "sub":   "8c8af53c-bcfc-4960-8874-bfb859aba5e0",
  "aud":   "exchange-app",
  "iss":   "http://localhost:8180/realms/poc",
  "email": "alice@poc.local"
}
```

OBO token, minted by Keycloak via RFC 8693:

```json
{
  "sub":   "8c8af53c-bcfc-4960-8874-bfb859aba5e0",
  "act":   { "sub": "agent-service" },
  "iss":   "http://localhost:8180/realms/poc",
  "scope": "openid profile email"
}
```

Same `sub`, same issuer, same signature chain — plus the `act` claim. Any service with the realm's public key can verify it independently. No shared secrets between tool servers and the gateway, no "trust me, it's alice" headers.

## Where Authorization Actually Happens

Knowing *who* alice is doesn't mean she can run every tool. The token is the transport; enforcement happens at independent layers, and each one reads the same two claims:

**Layer 1 — Gateway PDP.** CEL rules on the JWT before the request touches any tool server. In production this is **agentgateway** (Envoy-based) with an extAuth filter:

```yaml
- path: /mcp
  policy: jwt.realm_access.roles.exists(r, r == "ai-platform-user")
```

**Layer 2 — MCP server per-tool checks.** The tool server receives the full OBO token and can gate sensitive tools on alice's roles:

```python
SENSITIVE_TOOLS = {"delete_deployment", "apply_terraform", "merge_pr"}

def _exec_tool(name, arguments, claims):
    if name in SENSITIVE_TOOLS:
        roles = claims.get("realm_access", {}).get("roles", [])
        if "platform-admin" not in roles:
            raise PermissionError(
                f"tool '{name}' requires platform-admin — "
                f"sub={claims['sub']} has roles={roles}"
            )
```

**Layer 3 — Scope negotiation at exchange time.** Mint the OBO token with `mcp:read` but not `mcp:write`, and the tool server refuses writes regardless of roles.

**Layer 4 — Human-in-the-Loop.** For tools that are sensitive no matter who asks, the workflow pauses, notifies the human, and resumes only on explicit approval:

```
agent wants to call: delete_namespace
↓ HITL gate: pause workflow
↓ human sees notification → approve / reject
↓ approve: tool runs — reject: LLM is told "call was rejected"
```

The end-to-end picture, when everything is on:

```mermaid
flowchart TD
    A([OBO token sub=alice, act=agent-service]) --> B{Gateway PDP - CEL on JWT}
    B -->|deny| Z1([403 at the edge])
    B -->|allow| C{MCP server per-tool role check}
    C -->|missing role| Z2([Forbidden])
    C -->|sensitive tool| D{HITL gate}
    D -->|reject| Z3([agent told: rejected])
    D -->|approve| E([tool executes])
    C -->|allowed| E
```

## Security Properties

1. **The agent never sees the user's raw credential** — only the delegated OBO token, scoped to this task.
2. **The exchange secret is held by one small broker** — the skeleton-key pattern. Compromising the agent doesn't give you token-minting power.
3. **Grants are encrypted at rest** — AES-256-GCM in Redis; token material never sits in plaintext.
4. **Tokens are real RS256 JWTs signed by Keycloak** — verifiable by anyone with the realm public key, forgeable by no one.
5. **Every MCP call is traced with the identity it presented** — audit is a query, not an archaeology project.
6. **Revocation works** — alice's session ends, her `sub` becomes unauthorized, and in-flight tool calls fail closed.

## Conclusion

Agent identity is not an exotic problem requiring an exotic solution. RFC 8693 has been sitting there since 2020, Keycloak implements it, and the whole delegation chain — login, exchange, agent run, LLM call, MCP call, audit — fits in seven containers on a laptop. The `sub` + `act` pair turns "is this agent allowed?" into "is this *user* allowed to do this *via* this agent?", which is the question your security team actually wants answered. If you're wiring agents to real infrastructure, put the identity plumbing in before the agents get interesting.

## Reflections

Intellectual honesty time: this POC demonstrates identity **transport**, not enforcement.

What is verified: the token reaching MCP really carries `sub=alice act=agent-service`, the agent never holds alice's raw token, every call is traced. What is still missing?

- The gateway is **simulated** by the webapp — no real CEL policy on `/mcp`. Production wants agentgateway with an extAuth filter.
- The MCP server **logs** `sub` and `act` but blocks nothing. The per-tool role check above is a code sketch, not shipped behavior.
- No custom `mcp:read` / `mcp:write` scopes yet — the OBO token carries plain `openid profile email`.
- HITL is disabled (`ENABLE_HITL=0`). The durable-workflow pause/resume exists as stubs.
- Login uses ROPC (password grant) for demo simplicity — production means PKCE in a browser, plus mTLS everywhere.

Each gap is deliberate: transport first, because without `sub=alice` in the token, no enforcement layer has anything to enforce. The enforcement layers are the fun part — and they're all one `if` statement away once the identity is there.

If this topic touches your stack, you may also like [Stargate LLM Gateway]({{< ref "stargate-llm-gateway.md" >}}) — the production-side LLM gateway story this POC plugs into.
