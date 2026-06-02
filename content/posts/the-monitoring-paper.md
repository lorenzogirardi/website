---
title: "The Monitoring Paper"
date: 2021-01-05
draft: false
description: "A framework for monitoring that actually matters: System, Application, and Business metrics — and why buying a tool won't solve your observability problem."
tags:
  - monitoring
  - grafana
  - slo
  - sli
  - sla
  - metrics
  - observability
  - alerting
featuredImage: /website/images/the-monitoring-paper/Screenshot-2021-01-05-at-17.47.24.png
---

Contrary to popular belief, monitoring an infrastructure is the opposite of just having some metrics about applications and network.

There are many excellent resources on this topic. One of the most interesting is just a few pages from Google — [the art of SLOs](https://static.googleusercontent.com/media/sre.google/it//static/pdf/art-of-slos-slides.pdf). I took the book version from a Google on-site deep dive.

To structure this properly, I want to use four simple statements:

- **WHAT**
- **WHY**
- **WHO**
- **HOW**

## WHAT

This is probably the main argument we'll discuss here.

There are no magic formulas or tools. If you expect to buy a product that comes with useful metrics out of the box, you're wrong. Tools are the infrastructure to store and visualize data — but first you need to collect the right data.

**What is the data you need?**

That's what you discover by moving yourself into the customer's position. I use "customer" broadly to mean everyone using your product:

- People buying something on an e-commerce site (Amazon, Bestbuy, Aliexpress)
- People using a website to learn something (CNN, Bloomberg, Wikipedia)
- People working in finance using financial applications (Navision, SAP, JD Edwards)
- People consuming energy from a nuclear plant

Moving to the customer position helps you define the right **customer metrics**.

Stupid example: imagine an automatic gate.

What is the definition of "it's working?"

- It opens.

Obviously yes, but not only:

- What if it's not closing?
- What if it opens in 10 minutes — is that acceptable?
- What if after X operations the automation breaks?

You can define not only objective values (open/close), but also customer subjective frustration. If I'm buying an automatic gate, I won't choose one that takes 10 minutes to open. Not because of a technical spec — because 10 minutes is *too much*, based on my subjective value as a customer.

In the same way, we cannot say our application is working just because it answers.

So what should we consider when monitoring an application in production? Let me start with three macro clusters:

- **System** resources — all metrics related to the pod (CPU, network, memory...)
- **Application** standard metrics — threads, GC, errors per minute, requests per minute...
- **Business** metrics — inherent from the application scope. If the application handles a checkout, we need to know how many transactions succeeded and how many were rejected.

### System

These are the easiest and most-used metrics since you first started monitoring services. They represent the basic resource usage, and as the technology level increases from bare metal to VM to Docker to serverless, these values become progressively less influential in isolation. But you still need to start here to build a baseline.

![System metrics — VMware host](/website/images/the-monitoring-paper/Screenshot-2021-01-05-at-17.47.24.png)

Drilling into a VM inside that host, here's the one I use for my Kubernetes lab:

In both images you can see something moving CPU close to 100% cyclically — that's a [sitespeed.io crawler](/posts/kubernetes-sitespeedio/) running website checks every hour.

### Application

Now move inside the Kubernetes cluster and check how a web application is performing. For a Spring Boot application, you can understand the framework behavior (Java) and behavioral interactions: response time server-to-server, server errors, client errors, SQL interactions, garbage collector behavior, sidecar usage, log creation, metric creation.

![Application metrics dashboard](/website/images/the-monitoring-paper/app_metrics.png)

With these in place you can understand, from an application point of view, how performance evolves release by release. What happens under bugs. What happens under unexpected high traffic. Where you should focus improvement effort.

### Business

OK, system covered, application covered — what about customer perception?

This is the most critical topic. If your application is part of a microservices funnel or a monolithic product, at some point you have business value attached to its performance.

Move yourself to the customer position and monitor your website from that perspective with business metrics. For a simple case — a website:

- How long does it take to get the page?
- Which data matters most to improve?
- Are the trends consistent?

### WHY

With these three clusters of metrics we can define and plan the architecture and forecasting for the company. When something goes wrong with no metrics, you cascade into chaotic situations where everyone is guessing.

**System** metrics share infrastructure usage and density, creating the basis for capacity alerts and platform evolution planning.

**Application** metrics share framework behavior and application evolution day by day. Alerts are defined to monitor scalability, bugs, and application behavior.

**Business metrics share thresholds to use as alerts — but not only that.**

If you're selling something through your application, conversion is always the most relevant buzzword. From 100 visitors, who actually completes a purchase? In e-commerce, it's a small slice.

![Conversion impact score funnel](/website/images/the-monitoring-paper/conversion-impact-score-funnel-NEW.png)

This image comes from a 2015 [Akamai article](https://blogs.akamai.com/2015/07/conversion-impact-score-what-is-it-and-why-do-you-need-to-know-yours.html) that is still very much relevant today.

Conversion is always impacted by page response time. The impact is larger in early navigation phases like searching, compared to checkout where the customer already has clear intent to purchase.

Once we know the data around website usage, we can create **Goals** and **KPIs** to improve those values and better meet customer expectations.

## WHO

The actors vary by company size. Even if you're a full-stack whateverops, a developer, or part of a monitoring team, you should not be working alone here.

**System** and **Application** metrics are usually managed by SRE/DevOps/SysAdmin with help from architecture and developers to define the best framework metrics to monitor.

**Business** metrics are closer to the product side. Who better than the product department to define the KPIs and goals used as metrics — in cooperation with the business department to define the **thresholds?**

This cooperation produces:
- Right alerts to right people
- Support granted by ownership
- No speculation, real data

## HOW

### Infrastructure

First, define the monitoring and alerting platform. The recommendation is to have a scalable platform with a good UI (Grafana) backed by storage (Prometheus + Cortex/Thanos, Graphite + Carbonzipper, InfluxDB, etc.).

If you make a configuration mistake with Graphite (push model), you can saturate storage space since every new metric creates a Whisper file with size based on the retention schema. I prefer Prometheus (pull model), which prevents those issues and fits Kubernetes standards better. It's also easier to create a `/metrics` endpoint in your application than to run sidecar pods streaming metrics with collectd/telegraf/statsd.

### Less is More

How many metrics are enough for a valid alert?

More metrics identifying service behavior means more dedicated alerts — but there's an anti-pattern currently applied in many places: having tons of metrics that generate only **noise** without identifying the actual healthiness of a service.

_Keep it simple._ Define the structure of your System, Application, and Business metrics deliberately.

### Smart Metrics

It's not one value that shows service health — it's the combination of multiple metrics that guarantees status.

This is especially important in a dynamic infrastructure based on Kubernetes HPA. You probably have some values that are linear (like thread count), mitigated by horizontal autoscaling. You need to combine multiple values to define application status.

Create metric alerts with **trends**, not pure numbers. 70 threads alone makes no sense — different applications have different baseline thread counts. What matters is the **delta**. A +50% thread increase makes sense to alert on across all applications, regardless of the absolute number.

Be smart about evaluation with standard deviation and **percentile**. Mean drops the relevant values when evaluating performance under stress. P99 response time tells a completely different story than average response time.
