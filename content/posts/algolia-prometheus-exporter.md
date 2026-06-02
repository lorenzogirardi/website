---
title: "Algolia prometheus exporter "
date: 2026-05-26
draft: false
tags:
  - ai
  - deepseek v4
  - opencode
  - algolia
  - "prometheus "
featuredImage: /images/Gemini_Generated_Image_u9oq1ru9oq1ru9oq.png
---
# Algolia Usage Exporter — A Case Study in AI-Assisted Tooling

## What is this?

A lightweight Prometheus exporter that collects usage and infrastructure metrics from the Algolia search API and exposes them at `/metrics` for Prometheus / Datadog scraping.

It runs as a standalone HTTP server (single binary via Docker/Podman), requires minimal configuration (just two API keys), and exports metrics like:

- **Usage statistics**: search operations, records, processing time, QPS, write operations
- **Infrastructure metrics**: CPU, RAM, SSD utilization, build times (Premium plan only)
- **Health signals**: scrape success status, timestamps

The entire project lives in a single Python package with ~50KB of code, a Helm chart for Kubernetes deployment, and full test coverage across unit, integration, and e2e layers.

---

## Why build this?

Algolia already provides a dashboard. But dashboards don't alert you at 3 AM when search capacity degrades. Exporting metrics to Prometheus / Datadog means:

- Unified observability alongside the rest of the infrastructure
- Programmatic alerting (latency spikes, capacity saturation)
- Historical comparison and capacity planning

The API is read-only by design — there is no risk of accidental writes or data mutations.

---

## Built with opencode + DeepSeek V4

This project was built from scratch using **opencode** (the open-source Claude Code alternative) running **DeepSeek V4 Flash Free**.

The entire conversation — from analysis to implementation — was guided through natural language prompts. Every file, every test, every configuration was either written or refactored by an AI agent, reviewed and steered by a human operator.

Total active development time: approximately **2 hours** of back-and-forth prompting and validation.

### Why this matters

There is a growing narrative that AI assistants are useful only for prototyping or throwaway code. This project demonstrates the opposite:

1. **Production-ready output**: Helm charts, health endpoints, graceful shutdown, typed Python, comprehensive tests (76 passing), Docker/podman builds, error handling with fallback logic.

2. **Not just Anthropic models**: This entire session ran on **DeepSeek V4**, not Claude, not GPT-4. The ecosystem is diversifying fast — open-weight models can now drive complex, multi-step engineering workflows.

3. **Safe by design**: The exporter only calls GET endpoints and requires a dedicated read-only API key. No mutations, no deletions. AI-generated tooling for observability and monitoring is a **low-risk, high-leverage** use case — it improves operations without touching live business logic.

### The ratio

A traditional approach would have required:

- Reading the Algolia API docs (30 min)
- Designing the Prometheus collector pattern (30 min)
- Writing the HTTP client with retries (30 min)
- Implementing each collector (1h)
- Writing tests (1h)
- Building the Helm chart (30 min)
- Debugging API compatibility issues (variable)

**Total estimate: 4-6 hours** for a senior engineer.

With AI assistance, the same result was achieved in roughly **2 hours**, with **every artifact** produced (code, tests, docs, Helm charts, Dockerfile, CI configuration).

The multiplier is real — not because the AI writes perfect code on the first try (it doesn't), but because the feedback loop is compressed. Refactoring "add a /health endpoint and graceful shutdown" takes 30 seconds instead of 15 minutes. The human stays in the loop, making architectural decisions, validating correctness, and steering toward production quality.

---

## Key technical decisions

| Decision | Rationale |
|---|---|
| **Private CollectorRegistry** | Avoids infinite recursion when registering custom Prometheus collectors |
| **Fallback stat fetching** | Some Algolia plans reject the wildcard `*` — the exporter tries individual stats gracefully |
| **RFC 3339 dates** | The Algolia Usage API recently switched from `YYYY-MM-DD` to ISO 8601 with timezone |
| **Health endpoint** | Dedicated `/health` and `/ready` endpoints decoupled from the `/metrics` scrape weight |
| **Graceful shutdown** | SIGTERM handler ensures clean container termination in Kubernetes |
| **No dependencies beyond stdlib + requests + prometheus_client** | Minimal attack surface, easy to containerize |

---

## Is this production-ready?

Yes. It runs daily in production, scraping an Algolia application every 5 minutes through a Prometheus ServiceMonitor in EKS, with metrics shipped to Datadog.

The exporter handles:

- API key rotation without restarts
- 403 errors from the Monitoring API (Premium-gated features degrade gracefully)
- Per-index collection failures without aborting the global scrape
- HTTP 429 rate limits with exponential backoff
- Wildcard incompatibility across different Algolia plans

---

## The bigger picture

Projects like this are a sweet spot for AI-assisted development: **self-contained, well-documented API surface, clear success criteria, no user-facing complexity, high leverage**.

The code isn't magical. What's interesting is the process: a non-specialist can build reliable infrastructure tooling in a fraction of the traditional time, using models that run at commodity pricing, without touching production data or business logic.

That's a future worth building toward.
