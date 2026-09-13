---
title: "How I Manage a Multi-Site Homelab (So It Doesn't Manage Me)"
date: 2026-09-12
draft: true
description: "Five sites, three virtualizers, four Kubernetes clusters, one hub-and-spoke VPN. Here's the docs-as-code system that keeps it operable instead of just growing."
tags:
  - homelab
  - kubernetes
  - proxmox
  - automation
  - monitoring
  - ai
  - documentation
  - vpn
featuredImage: /images/how-i-manage-a-multi-site-homelab-so-it-doesnt-manage-me/featured.jpg
images:
  - "/images/how-i-manage-a-multi-site-homelab-so-it-doesnt-manage-me/featured.jpg"
---
### Table of Contents

  * Introduction
  * From One NUC to Five Sites
  * The Real Problem Isn't Compute, It's Memory
  * Docs as Code, With Rules
  * The Layout: Where New Things Go, Decided Once
  * Templates: the Only Way New Hosts Look Like Old Hosts
  * Runbooks Instead of "I Remember How I Did This"
  * STATUS.md: the One File That Tells the Truth
  * The Migration Folder: Rigor Where It's Easy to Cut Corners
  * Sysadmin via MCP: Claude Remote Agent, With Guardrails
  * Reflections
  * Three Questions for Your Own Docs
  * Does This System Have a Name?
  * Conclusion



![Infrastructure management domains: know-how repository, runbooks, compliance policies, monitoring, feeding an agent reasoning core against the managed infrastructure](/images/how-i-manage-a-multi-site-homelab-so-it-doesnt-manage-me/featured.jpg)

Here we are, again, with a homelab post. Except this time it's not about a NUC under a desk.

## Introduction

Years ago I [wrote about a silent, cheap NUC running ESXi](/posts/homelab-when-small-is-big/) as "big enough" for a homelab. That NUC is long retired. What replaced it is five physical sites, three Proxmox hypervisors, one Banana Pi, one Raspberry Pi, an ARM instance in the public cloud, four Kubernetes clusters running two different distributions, and a hub-and-spoke VPN tying it all together over IPsec.

I already documented the topology itself in a [separate architecture post](/infra-flow/), with the full diagram of sites, tunnels, clusters and monitoring flow. This post is not that. This is about the part that doesn't show up in a network diagram: how the files and descriptions are organized so I can find the current state of something fast, actually intervene on it, and pick a change back up in six months without starting from zero, or add something new already knowing exactly where it's supposed to live. If you're sitting on your own sprawling homelab wondering whether your notes folder still serves you, that's the question worth asking as you read this.

## From One NUC to Five Sites

Growth happened the way it usually does in a homelab: one decision at a time, each one reasonable on its own. A second site for family reasons. A cloud instance because ARM compute was free. A second Kubernetes distribution because `microk8s` was already there and I wanted to compare it against `k3s`. A monitoring box because Grafana Cloud's free tier wasn't enough anymore.

None of these were planned as "let's build a five-site infrastructure." Each was a small, local, justified addition. The sum of those additions is what actually needs managing now: a hub-and-spoke IPsec overlay, three separate monitoring paths (Collectd, Telegraf, VictoriaMetrics) feeding one Grafana, two Proxmox clusters with independent backup targets, and Kubernetes running on four nodes across three trust boundaries.

Strange, isn't it? Nobody sits down and designs this. It accretes.

## The Real Problem Isn't Compute, It's Memory

Once you cross two or three sites, the bottleneck stops being CPU or RAM and becomes something much less glamorous: do you remember why things are the way they are?

Why does traffic from Milano to Casa go through a specific SNAT hop instead of routing directly? Why is one microk8s cluster still on an old Kubernetes minor version? Which Proxmox host backs up where, and which one doesn't get backed up at all? Six months after making a decision, I don't reliably remember the answer, and "ask the person who set it up" doesn't work when that person is also me, on a different weekend, with a different amount of coffee in his system.

If it isn't written down with the *why*, it doesn't exist. That's the premise this whole system is built on.

## Docs as Code, With Rules

The infrastructure lives in one private git repository, separate from this blog, versioned like any other codebase. But a docs repo without rules degrades into the same mess as an undocumented one, just with extra Markdown files nobody trusts.

So the repo has a `CLAUDE.md` at its root. It's not a style guide someone reads once, it's loaded straight into context every time an AI agent works in the repo, Claude Code included. The agent doesn't consult it, it operates under it: same rules whether I'm the one typing or the agent is. It assigns a role ("technical writer with SRE and network engineering expertise") and a small set of non-negotiable rules:

- Every host doc includes IP, OS, type, services, monitoring, key files, quick commands.
- Network docs explain traffic flow step by step: source and destination IP, NAT transformations, tunnel encapsulation.
- Explain the *why*, not just the *what* (why SNAT is needed here, why a specific `rp_filter` value, why one cluster still runs an older Kubernetes minor).
- Tables for structured data, code blocks for configs, Mermaid for topology and sequence diagrams.
- Never commit secrets, passwords, PSKs, or certificates.

That last rule matters more than it looks. A homelab repo full of real pre-shared keys is a liability the moment it leaks, intentionally or not. Keeping documentation strict about *never* including secrets means the repo can be shared, reviewed, even shown in a blog post like this one, without a redaction pass first.

{{< mermaid >}}
flowchart LR
    CHANGE[Infra change happens] --> DOC[Update host or overlay doc]
    DOC --> RULES[CLAUDE.md rules checked, IP/ports/why]
    RULES --> STATUS[STATUS.md updated]
    STATUS --> RUNBOOK[Runbook added if repeatable]
    RUNBOOK --> CHANGE
{{< /mermaid >}}

## The Layout: Where New Things Go, Decided Once

Two questions decide whether a docs repo earns its keep in an actual crisis: how fast can you find the current state of something, and when you're adding something new, do you already know where it goes, or do you improvise a home for it on the spot?

The repo answers both with one fixed tree, written down once in `CLAUDE.md` and never renegotiated per addition:

```
infrastructure-settings/
├── _templates/          # shape for anything new
├── sites/<name>/        # one directory per physical location
│   ├── README.md        # site overview
│   ├── network.md       # diagram, routing, firewall
│   ├── hosts/           # one file per host
│   └── kubernetes/      # k8s cluster docs, if applicable
├── overlay/             # cross-site networks (VPN, routing)
├── monitoring/          # centralized monitoring stack
├── backup/              # backup strategies
└── runbooks/            # operational procedures
```

The rule underneath it: a fact about the infrastructure has exactly one home. VPN tunnel details live in `overlay/vpn.md`, full stop; every site's network doc references that file instead of re-describing the tunnel inline. `CLAUDE.md` says it outright: "Reference VPN IPs from overlay/vpn.md, don't duplicate the tunnel map." Duplication is how two docs quietly disagree six months later, and neither one looks obviously wrong on its own, so neither gets fixed.

That fixed shape answers both opening questions before they're even asked. Need judge's current state? `sites/milano/hosts/judge.md`, no searching. Adding a sixth site? `sites/<name>/` with the same four things every other site has. Adding a new Kubernetes cluster? `_templates/k8s-cluster.md`, copied, filled in, filed under that site's `kubernetes/`. Where something lives was never a decision made in the moment, it was made once, in advance, for every future case that fits the pattern.

That's also what makes coming back after months tractable. I don't re-read the whole repo to re-orient, I go straight to the one file that should hold the fact I need. If it's not there, that itself is informative: either the doc drifted, or the layout has a gap worth closing, both cheaper problems than starting from a blank page.

## Templates: the Only Way New Hosts Look Like Old Hosts

Without a template, the tenth host doc looks nothing like the first one. Some fields get skipped because "it's obvious," some get renamed, and diffing two host docs for a pattern becomes guesswork.

The repo has five templates under `_templates/`: `vm.md`, `lxc-container.md`, `k8s-cluster.md`, `network-site.md`, `appliance.md`. Adding a new LXC container means copying `_templates/lxc-container.md` and filling it in, not writing from a blank page. It's a small constraint, but it's the difference between documentation that stays structurally comparable across 20+ hosts and documentation that decays into personal style per entry.

## Runbooks Instead of "I Remember How I Did This"

Some tasks I do rarely enough that I forget the exact steps between occurrences: adding a VM, adding an LXC container, troubleshooting a StrongSwan tunnel that won't rekey, writing Cilium network policies without breaking east-west traffic, upgrading a k3s cluster's minor version safely.

Each of these is a runbook: `runbooks/new-vm.md`, `runbooks/new-lxc-container.md`, `runbooks/vpn-troubleshooting.md`, `runbooks/cilium-network-policies.md`, and a dedicated `runbooks/k3s-upgrade/` folder with pre-upgrade tests, the upgrade script, post-upgrade tests, and a rollback script, each as an actual executable, not just prose describing what a script *should* do.

The rule of thumb: if I've done a task twice and had to think both times, it becomes a runbook the second time. If it isn't there, it can't save me the third time either.

## STATUS.md: the One File That Tells the Truth

Documentation drifts. A doc says a task is done; three months later the underlying thing changed and nobody touched the doc. The fix isn't "try harder to remember to update docs," it's a single file whose entire job is to answer one question: what's actually done, and what's missing?

`STATUS.md` at the repo root is a table per site (percentage documented, hosts covered, what's missing) plus a cross-site table for VPN overlay, monitoring, backup, runbooks, and dashboards, each with a completion percentage and a "missing" column that's allowed to say something honest like *"per-site tunnel details"* or *"OCI/BRAiN/deva backup not yet implemented."*

That second part matters. A status file that only ever says 100% is worthless. The value is in it admitting the backup strategy doc itself: the NAS that all other Proxmox hosts back up *to* has no backup of its own. Writing that down in a tracked file, instead of just knowing it uncomfortably, is what eventually gets it fixed instead of quietly re-discovered during an actual failure.

## The Migration Folder: Rigor Where It's Easy to Cut Corners

The clearest test of whether "docs as code" is more than a nice idea is a live migration. Right now that's consolidating one microk8s cluster onto k3s, moving it to different hardware in the process.

Instead of a single "migration notes" file, it's a numbered sequence of folders, each with its own scripts: prerequisites and pre-flight checks, PV creation, stateful workload migration (PostgreSQL, Keycloak, Kong, Konga, Redis, MinIO, Portainer, the Prometheus stack) one component at a time, monitoring stack consolidation, network cutover (DNS, port forwarding, Cloudflare Tunnel config), a testing folder with one script per subsystem (Postgres connectivity, Redis, ingress routes, NodePorts, DNS resolution, the Cloudflare tunnel itself, plus a `smoke-test-all.sh` that runs the lot), and finally cleanup once the old cluster is decommissioned.

This is deliberately more ceremony than a hobby migration strictly needs. But a migration that touches identity (Keycloak), the API gateway (Kong) and object storage (MinIO) at once is exactly the kind of change where skipping the boring numbered checklist is how you end up debugging an outage at 11pm with no idea which of six moving parts broke.

## Sysadmin via MCP: Claude Remote Agent, With Guardrails

The `CLAUDE.md` contract governs writing, but actual sysadmin work, restarting a service, checking disk on a hypervisor, inspecting a stuck pod, is a second agent's job: [Claude Remote Agent](https://github.com/haxorthematrix/claude-remote-agent), wired into the repo as `ssh-mcp/`. Not to be confused with Claude Code itself, this is a separate MCP server that gives Claude Code tools like `remote_execute` and `remote_session_start` over plain SSH, so "check why judge's disk usage spiked" turns into a tool call instead of me tabbing into a terminal.

What makes handing an agent shell access acceptable is the policy layer sitting in front of every host in `hosts.yaml`. Each host declares its own `confirmation_required` level (`never`, `destructive_only`, `write_only`, `always`), a command blocklist, and labels for site/role/os. The two Casa hypervisors, for instance, are pinned to `always`, meaning the agent proposes a command and I approve it before it runs, no exceptions, regardless of how routine the task looks:

```yaml
madara:
  hostname: 192.168.50.21
  policy:
    confirmation_required: always
    allowed_commands: "*"
    blocked_commands:
      - "rm -rf /"
      - "mkfs.*"
      - "dd if=.* of=/dev/.*"
  labels:
    site: casa
    role: hypervisor
    os: proxmox
```

The MCP server keeps its own audit log (`~/.config/claude-remote-agent/audit.log`, command and output, connection pooled per host). But an agent-side log only sees what the agent ran, so `runbooks/command-audit-logging.md` adds a second, independent layer: kernel-level `auditd` (and a lighter `bash` `PROMPT_COMMAND` log as backup) installed directly on the target host, capturing *every* `execve`, agent-initiated or not, root cron jobs included. It's the standard control for anything internet-facing (the Oracle Cloud instances, in particular), rolled out during provisioning rather than bolted on after an incident.

Two audit trails that don't trust each other is deliberate. If I only trusted the MCP server's own log, a bug or a bypass in the agent tooling would be invisible. The host-level log doesn't care what ran the command, agent, human, or a compromised process, it logs the `execve` regardless.

## Reflections

Is this over-engineered for a homelab? Fair question. A single NUC doesn't need a `CLAUDE.md`, five templates and a runbook folder. Five sites, three virtualizers and four Kubernetes clusters do, or at least they do if the goal is being able to touch this infrastructure again in six months without archaeology.

What's still missing? The `STATUS.md` percentages are honest, but they're still a manually maintained file, not something derived automatically from the repo's actual content: nothing currently checks that every host under `sites/*/hosts/` has a matching doc, or flags one that's gone stale relative to a recent change. And the AI-assisted operations side is younger than the documentation side: audit logging exists, but there's no automated review of what got run, only the ability to go look.

## Three Questions for Your Own Docs

None of this requires five sites to be worth adopting. Three questions, applied to whatever you already have scattered across notes, wikis or a README that's stopped being read:

1. **Does every fact have exactly one home?** If the same IP or config value can be found written in two places, they will eventually disagree, and you won't know which one lied until something breaks.
2. **When you add something new, do you know where it goes before you start writing?** If the answer is "I'll figure out a spot," you don't have a layout, you have a pile that happens to be organized today.
3. **Could you resume a six-month-old thread by reading one file, not the whole repo?** If the honest answer is no, the fix usually isn't more documentation, it's one file whose only job is to say what's actually true right now, `STATUS.md` or otherwise.

None of these need a five-site homelab or an AI agent to be worth doing on a single NUC. They just get non-negotiable once forgetting something costs you an evening of archaeology instead of five minutes.

## Does This System Have a Name?

Fair question, and worth answering honestly instead of pretending I invented something new. Five pieces are doing five different jobs here, each fixing a different way documentation normally rots:

`CLAUDE.md` is a prescriptive contract, not a description of what exists. It's read by me and by the agent, same rules for both. Templates are typed content: a `vm.md` and an `lxc-container.md` are different shapes on purpose, so two instances of the same type are structurally comparable, not just similar. Runbooks separate procedural knowledge (how to intervene) from declarative knowledge (what a thing is), because burying "how to fix it" inside "what it is" is how runbooks get lost. `STATUS.md` is a meta-layer that says how much to trust the rest, explicitly, instead of assuming every doc is current. And all of it lives in git, reviewed with the same tooling as code, instead of a wiki that quietly drifts.

The common thread under all five: a fact has exactly one home, the shape of that home is decided before you write into it, and one file exists whose only job is admitting which homes are still empty.

None of this is new in isolation. It sits close to a few named things, none of which cover it exactly:

- **Docs-as-Code** (the Write the Docs community, since around 2014): documentation in plain text, versioned, reviewed via pull request, same tooling as code. This is the git layer here, but on its own it says nothing about structure or content type.
- **Diataxis** (Daniele Procida's framework): four content modes, tutorial, how-to, reference, explanation. Maps almost cleanly: runbooks are how-to, host docs are reference, the `rationale` fields CLAUDE.md demands are explanation. No tutorials here, a private homelab doesn't need onboarding material.
- **DITA**: an OASIS standard for topic-typed technical documentation with reuse. The `_templates/` folder is topic-typing without the XML tooling.
- **ITIL's Configuration Management Database**: configuration items plus their relationships plus periodic accuracy audits. `STATUS.md` is a hand-rolled CMDB completeness audit, done in Markdown instead of a dedicated tool.
- **Google's SRE runbook culture** (the SRE Book, Beyer et al.): runbooks as first-class operational artifacts, distinct from architecture docs. Same split as `runbooks/` versus `sites/*/hosts/`.
- **Architecture Decision Records**: capture the *why* at decision time, in a dedicated file. CLAUDE.md's "explain the why" rule is an informal ADR, spread across every doc instead of living in its own `adr/NNNN-*.md` series.
- **Living Documentation** (Cyrille Martraire's book of the same name): documentation derived from a single source of truth, engineered so it can't quietly lie. A `STATUS.md` that's allowed to say "not done" is exactly this philosophy.

The one piece none of those names cover: `CLAUDE.md` isn't only read by a human, it's loaded into an agent's context and operated under, not just consulted. That's new enough it doesn't have a settled name yet. People are calling the general practice **context engineering**, deliberately curating what enters an LLM's context window, and the emerging `AGENTS.md`/`CLAUDE.md` convention as a repo's "constitution" is one instance of it. Too recent, as of 2026, to have earned a conference-track name of its own.

If forced into one label: docs-as-code, shaped by Diataxis, with an agent-executable constitution layered on top. Accurate, and nobody's going to say that out loud twice.

## Conclusion

The infrastructure grew organically and will keep growing organically, that part isn't going to change. What changed is that growth no longer erases what came before it. A `CLAUDE.md` that sets the rules, templates that keep new docs shaped like old ones, runbooks for anything done twice, a `STATUS.md` that's allowed to say "not done," and a numbered migration folder for the risky stuff: none of it is glamorous, all of it is what actually lets one person run a five-site, multi-virtualizer, multi-Kubernetes homelab without it quietly turning into a black box.
