---
title: "Kubernetes sitespeed.io"
date: 2020-10-02
draft: false
description: "Running sitespeed.io as a Kubernetes CronJob with Argo Workflows to get objective website performance metrics from the customer's perspective."
tags:
  - kubernetes
  - sitespeed
  - argo
  - grafana
  - performance
featuredImage: /images/kubernetes-sitespeedio/reaction.png
---

![Performance reaction](/images/kubernetes-sitespeedio/reaction.png)

First, a thought about what this is and what it isn't: this is about website metrics management. Not the only way, but one way for a high-level overview.

I'm focused on sharing concepts about website monitoring and one possible way to manage this in Kubernetes. You can reach the same goal with just Docker and crontab — but I'm using some other tools in Kubernetes because I'm evaluating them for other purposes.

## Motivations

A website is a product. Whether it's WordPress, a Magento e-commerce store, or a complex multi-application architecture — if a business runs on top of it, you need to think about it from the customer's perspective:

- How long does the website take to answer?
- Is the response time consistent across all sections?
- Are new deployments better or worse for the customer?

If your response time exceeds some threshold and you're selling a commodity product that's available elsewhere, you're losing money and customer loyalty.

There's also the "it's slow" problem. When someone in your company says "the website is slow," you get:

- Faster from my home fiber connection
- Faster from my premium mobile SIM
- Slow from wireless
- Fast from wireless (same person, different day)
- No idea
- Faster than... what, exactly?

No. What we need is **data**. Objective, long-term, reproducible data:

- Release by release
- Feature by feature

And that data needs to come from the customer's position — not a 1 Gbit inline connection with 0.00001ms roundtrip.

## The Customer Point of View

You have to know your product and your customer. If you're Apple, customers are affiliated — they'll wait for a slow page because brand loyalty is strong. If you're a commodity retailer, a slow page means the customer immediately navigates to a competitor.

**Speed** — we usually define slowness by perception. In 2020, Google Analytics can tell you exactly when people leave a page based on load time. Track these:

- Bounce rate vs. response time correlation
- Exit/leave rates at specific response time thresholds

If a search takes more than 10 seconds — say goodbye to the customer.

When measuring speed, move to the customer's side. Not everyone has 1 Gbit. Most people are on mobile. Maybe your server is in the US and your customer is in Korea. Speedtest's global index data is quite optimistic compared to real-world conditions.

Sitespeed.io classifies four network profiles:
- 3g
- 3gfast
- 3gslow
- cable

In 2020, `cable` and `3gfast` cover the worst realistic scenario. Even if you disagree with the specific categories, always look at **delta** in the graphs — not absolute numbers.

## Sitespeed.io

Open source, huge extension ecosystem. In this scenario I'm using only metrics collection — no video recording, no HAR files.

You can see a full report here: https://lorenzogirardi.github.io/sitespeedio-results/

Generated with just the Docker image:

```
docker run --rm -v "$(pwd):/sitespeed.io" sitespeedio/sitespeed.io:15.2.0 https://www.k8s.it/
```

With crontab and some additional options you can store metrics and artifacts to S3:

```
/usr/bin/docker run --privileged --shm-size=1g --rm --network=cable sitespeedio/sitespeed.io https://www.example.com -v -b chrome --video --speedIndex -c cable --browsertime.iterations 1 --s3.key S3_KEY --s3.secret S3_SECRET --s3.bucketname S3_BUCKET --s3.removeLocalResult true --s3.path S3_PATH www.example.com --graphite.host GRAPHITE_HOST --graphite.port GRAPHITE_PORT --graphite.namespace GRAPHITE_PREFIX
```

## My Use Case

For this implementation:
- Metrics only — no artifact storage
- Store metrics in InfluxDB
- Run inside Kubernetes (not Docker)
- Orchestrate with Argo Workflows

Sitespeed.io runs in a container, executes, and exits — which maps perfectly to a Kubernetes CronJob. But I wanted to explore a workflow manager with more capability than a simple cron.

## Argo Workflows

I chose Argo because I wanted something similar to Rundeck but with extended capability — potentially replacing Spinnaker for CI/CD use cases. https://argoproj.github.io/

Installation:

```
kubectl create namespace argo
kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo/stable/manifests/install.yaml
```

Expected pods:

```
NAMESPACE    NAME                                       READY   STATUS
argo         argo-server-6c886c5b77-l8jfv               1/1     Running
argo         workflow-controller-65948977d-zc9vt        1/1     Running
```

I hit a bug running the first job because I use containerd instead of Docker in microk8s:

```
MountVolume.SetUp failed for volume "docker-lib" : hostPath type check failed: /var/lib/docker is not a directory
```

The fix — add this to the `workflow-controller-configmap`:

```yaml
data:
  config: |
    containerRuntimeExecutor: pns
```

Access the UI with port forwarding while experimenting:

```
$ kubectl -n argo port-forward deployment/argo-server 2746:2746
Forwarding from 127.0.0.1:2746 -> 2746
Forwarding from [::1]:2746 -> 2746
```

The UI is clean and authentication follows RBAC policy. Worth exploring if you need more than basic CronJob scheduling.

## Results

Once running, the metrics appear in Grafana. Two backend options work with the sitespeed.io configuration in `001-argo-job-sitespeedio.yaml`:

**InfluxDB backend** — use the official Grafana dashboard:
https://github.com/sitespeedio/grafana-bootstrap-docker/blob/main/dashboards/influxdb/pageSummary.json

![Sitespeed results in Grafana](/images/kubernetes-sitespeedio/sitespeedresults.png)

**Graphite backend** also works with the same metrics and even more provided dashboards.

Live view: https://services.k8s.it/grafana/d/000000053/pagesummary-influxdb?orgId=2&refresh=15m

## What's Next

Sitespeed.io is great for a high-level view but also provides details that can save load time — largest contentful paint, total blocking time, cumulative layout shift. The introduction of Argo enables orchestrating multiple checks across all pages and applications without individual crontab entries per job — a producer/consumer workflow that scales as your monitoring needs grow.
