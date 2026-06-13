---
title: "Kubernetes Apache HTTPD — The Front Controller Pattern"
date: 2020-08-11
draft: false
description: "Why Apache HTTPD still has a place in Kubernetes as a front controller — managing complex routing, A/B testing, and multi-country path resolution for large e-commerce sites."
tags:
  - apache httpd
  - front controller
  - httpd
  - kubernetes
  - rewrite rules
featuredImage: /images/kubernetes-apacherr/front-controller.png
images:
  - "/images/kubernetes-apacherr/front-controller.png"
---

## The Semi-Unuseful Apache Implementation in Kubernetes

When you have Ingress resources, Ambassador, Nginx, Traefik, and service meshes — why would you put Apache HTTPD in a Kubernetes pod?

Because sometimes the routing logic isn't ops' problem. It's a product problem.

## Digression

Complex e-commerce sites serve multiple microservices under one domain. `www.example.com` might route like this:

- `/it/` and `/it/offerte` → CMS
- `/uk/` and `/uk/offers` → CMS
- `/it/clienti/` → customer-app
- `/uk/customers/` → customers-app
- `/secure/` → payment-app

And then there are third-party domain acquisitions that need redirects. And A/B test variants. And country-specific promotional paths that change weekly.

Managing this in Nginx ConfigMaps or Ingress annotations becomes a nightmare. The product team can't touch it. Every change needs an SRE.

### The Front Controller Pattern

![Front controller architecture](/images/kubernetes-apacherr/front-controller.png)

A front controller is a business logic layer responsible for:

- Handling all requests and routing to backend applications
- Dynamic path resolution based on business rules
- A/B testing logic
- Request validation and loop prevention
- Exposing admin interfaces for website layout management

This architecture moves Apache from layer 1 (DMZ/edge) to layer 2 (Application). Ingress or HAProxy handles the edge. Apache handles the routing intelligence — and the product team owns it.

## Project Concepts

The key shift: Apache as a **product component**, not an infrastructure component.

- Owned by product engineers, not SREs
- Configuration lives in ConfigMaps (or compiled containers for large sites)
- Routing rules expressed in Apache `RewriteRule` — familiar to anyone who's done SEO work
- Scales independently of the application pods behind it

## Configuration

Route-heavy config via ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: apache-config
  namespace: frontend
data:
  rewrite.conf: |
    RewriteEngine On

    # Country routing
    RewriteRule ^/it/(.*)$  http://cms-it.svc.cluster.local/$1 [P,L]
    RewriteRule ^/uk/(.*)$  http://cms-uk.svc.cluster.local/$1 [P,L]

    # App routing
    RewriteRule ^/it/clienti/(.*)$ http://customer-app.svc.cluster.local/it/$1 [P,L]
    RewriteRule ^/uk/customers/(.*)$ http://customer-app.svc.cluster.local/uk/$1 [P,L]

    # Payment (secure)
    RewriteRule ^/secure/(.*)$ http://payment-app.svc.cluster.local/$1 [P,L]
```

## Deployment

```bash
kubectl apply -f apacherr/deployment/
```

For sites where ConfigMap size hits etcd limits (1MB per object), pre-compile the configuration into the container image via CI/CD pipeline. The pod becomes immutable — routing changes trigger a new build.

## When This Makes Sense

- Large e-commerce with 50+ routing rules owned by product/SEO teams
- Multi-country, multi-brand setups where routing is a business concern
- Legacy Apache rewrite rule sets that would take months to rewrite in Ingress annotations
- Sites where A/B testing logic lives in rewrite conditions

When routing is simple: use Ingress. When routing is a product — use a front controller.
