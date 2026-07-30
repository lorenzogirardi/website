---
title: "Whoami"
layout: "single"
url: "/whoami/"
summary: "About me"
---

## Lorenzo Girardi: #semi-serious

<p align="center"><img src="/images/whoami/lorenzo.jpg" alt="Lorenzo Girardi" width="50%"></p>

**Contact:** lorenzo.girardi@gmail.com (PGP) · lorenzo@ethzero.it · l@k8s.it
**Web:** [k8s.it](https://www.k8s.it) · [GitHub](https://github.com/lorenzogirardi) · [Grafana](https://services.k8s.it/grafana/d/kxQQuHRZks/proxmox?orgId=2&from=now-24h&to=now&timezone=browser&var-server=madara&var-storage=local-lvm&var-interfaces=vmbr0&var-interfaces=bond0&refresh=1m)
**Nationality:** Italian · **Driving license:** yes
**Resume:** [Download CV (PDF)](/files/CV_Lorenzo_Girardi_ENG_2026.pdf)

Platform & Development Engineering Manager.

Tech enthusiast since before it was a LinkedIn buzzword.

In the late '90s I haunted IRC exploit channels, not to cause damage, but because I quickly realized how much I didn't know.

Early 2000s were spent on packetstormsecurity learning the hard way.

Humility: acquired.

My family wanted me to be an architect (the buildings kind, I trained as a geometra).

Instead I enrolled at Università degli Studi dell'Insubria for IT, which turned out to be *"very far from the needs of the world of work."*

Classic.

Career milestones:

- **Eprice srl**: helpdesk, server room, production servers. Verdict on humans vs. machines: machines won.
- **Body rental / contracting**: exposure to varied environments and scales, zero job security, maximum learning.
- **lastminute.com / BravoNEXT (2012–2023)**: startup to something more. Sysadmin → Team Leader → Team Manager → Platform Architect.
- **Gucci (2023–present)**: build everything from scratch. No internal resources, only an idea.

Landed at BravoNEXT (bravofly at the time) in 2011, a company mid-jump from startup to something bigger.

First job: the Windows infrastructure running the .NET applications, quite different from the "usual" I was used to.

Kept leaning on my Linux side too, never stopped poking at open source.

A few years in, I had enough scope and enough impact on decisions to earn the confidence for team leadership.

Turned out watching a team ship results made me happier than shipping alone, so I took the managerial jump: 12+ people, for about three years.

Summary: *everyone remained alive, no one was mistreated.*

Personal mantras:

> **Keep it simple.**
> **If it isn't there, it can't break.**

---

## The New Adventure

I moved for the chance to build everything from zero, not a small challenge: no internal resources, only an idea.

A very clear idea on paper and blueprints (draw.io addicted) ... then reality.

Not a digital transformation: a transformation of people, towards a different way of looking at the product than the one they were used to.

Not only the layout, but the speed to get there and make it visible.

Not standards for the sake of the word, but the ability to reproduce new things quickly.

Well ... the site is up, now they are microservices, divergency is minimal, every operation runs in CI/CD, and I'm quite satisfied.

Going back? Too easy to say it now, but I would change something, and not because of the technology, but because of the kind of transformation that was *thought* (perception) to be in progress.

Something you only understand after many months of feeling and context building.

And now? Now there is AI ... an immense pile of marketing buzzwords hiding the reality of large language models.

If you know how they work and how to use them, they will be part of a transformation too.

What got built:

- Two teams from zero: one taking over the uncovered as-is estate, one investing in the new AWS foundation
- As-is estate migrated to AWS through the AWS MAP program
- Multi-account AWS landing zone structured by business domain, all accounts behind a transit gateway for clean communication and security segmentation
- EKS clusters plus the service consumption framework on IRSA, assume role and resource-based policies
- A platform contract for development: immutable container standards with full scaffolding up to Helm
- GitOps operations with ArgoCD, security via SCA, SAST, Falco and Kyverno to avoid (and track) any divergency
- Metrics, log and trace framework on open standards, portable to Dynatrace, Datadog or Grafana
- An error budget framework, to discipline the time spent on improvements
- AI pilot: LiteLLM, Grafana, Langfuse, LangGraph, plus the adoption model, the framework for AI components, a Cilium policy model for them, and an on-behalf-of permission model based on RFC 8693

---

## Areas of Experience

SRE · Datacenter management · People management · Business projects · Virtualization · Networking · Multi cloud architecture · OS · High-traffic infrastructure · Monitoring · Observability · Kubernetes · PCI DSS · K-ISMS · GDPR · AI Act · Automation · Architecture · IaC · Digital Transformation · Monolith to microservices · AI Transformation · Enterprise architecture design (infra & code) · Applications contracts (frameworks)

---

## Technical Skills

| Domain | Technologies |
|---|---|
| Java stack | Tomcat, JBoss, Spring Boot, JMX |
| .NET stack | IIS, SQL Server, WSFC, AlwaysOn, NLB |
| Node stack | Fastify, Express, Next.js |
| Web / Proxy | Apache, Nginx, Traefik, Kong, HAProxy |
| Virtualization | VMware ESXi, RHEV/oVirt, Proxmox, Veeam |
| Cloud | AWS (from landing zones to single managed services), Azure, GCP, OracleCloud |
| Automation / IaC | Puppet, Ansible, Terraform, Packer, Jenkins, Spinnaker |
| Monitoring | Grafana, Prometheus, Loki, InfluxDB, Cortex, Graphite |
| Identity | LDAP, AD, ADFS, OAuth, SAML |
| VPN / Network | OpenVPN, StrongSwan, site-to-site, traffic shaping |
| Containers | Kubernetes, EKS, Docker, containerd, Flannel, etcd, HPA |
| AI | LLM, RAG, vectordb, MCP, agents, LangChain, LangGraph, AI gateway |

---

## Work Experience

| Role | Company | Period |
|---|---|---|
| Platform Engineering Manager | Gucci | Mar 2024 – present |
| Principal Infrastructure Engineer | Gucci | Oct 2023 – Mar 2024 |
| SRE Platform Architect | lastminute.com (BravoNEXT sa) | Nov 2022 – Oct 2023 |
| SRE Team Manager | lastminute.com (BravoNEXT sa) | Jan 2019 – Nov 2022 |
| SRE Team Leader | lastminute.com (BravoNEXT sa) | May 2016 – 2018 |
| System Administrator | lastminute.com (BravoNEXT sa) | Jan 2012 – mid 2016 |
| Sysadmin (contractor) | LGI sa via Temera / IT Time | 2010 – 2012 |
| Sysadmin | Eprice srl | 2008 – 2010 |

---

## Education

| | |
|---|---|
| Università degli Studi dell'Insubria | Information Technology, 2003–2008 |
| I.T. Geometri Pier Luigi Nervi | 1997–2002 |

---

*If it isn't there, it can't break.*
