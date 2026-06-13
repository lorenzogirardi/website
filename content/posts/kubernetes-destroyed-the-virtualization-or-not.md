---
title: "Kubernetes Destroyed the Virtualization... or NOT"
date: 2020-12-07
draft: false
description: "Why Kubernetes and virtualization are not antagonists — the case for running Kubernetes on VMs and why major cloud providers agree."
tags:
  - kubernetes
  - virtualization
  - containers
  - cloud
featuredImage: /images/kubernetes-destroyed-the-virtualization-or-not/vm_vs_pod.png
images:
  - "/images/kubernetes-destroyed-the-virtualization-or-not/vm_vs_pod.png"
---

![VM vs Pod](/images/kubernetes-destroyed-the-virtualization-or-not/vm_vs_pod.png)

**NO.**

That's the short answer. Kubernetes has not destroyed virtualization. Let me explain why.

I've been watching the "Kubernetes rush" for a few years now. Companies highlight success stories and cost savings, but I often wonder if they're sharing the complete picture. I want to focus specifically on Kubernetes and virtualization — not the broader data center ecosystem with databases, legacy systems, and corporate infrastructure.

## Historical Context: From Borg to Kubernetes

Google's internal system, Borg — formally open-sourced as Kubernetes — introduced an initial question that made sense at the time: "why the virtualization overhead if we will use Kubernetes?" The underlying logic: if you're migrating a VM as a Kubernetes pod, why keep the VM layer at all?

The technology evolved through phases:

- **Phase 1:** Early adoption with VM-to-pod migration logic
- **Phase 2:** Kubernetes expansion across organizations
- **Phase 3:** Infrastructure fragmentation through node labeling

## The Infrastructure Fragmentation Problem

This is where I see teams get into trouble. Using node labels to isolate workloads creates problems that compound over time:

> "Isolating some workloads with labels can create an HW lock-in"

The issues cascade:
- Hardware lock-in that requires multi-cluster solutions to escape
- Complex update management across labeled roles
- Storage node maintenance complications
- Legacy systems with unclear ownership
- Single points of failure
- New hardware generations requiring fresh provisioning just because the label doesn't match

![VM fragmentation pattern 1](/images/kubernetes-destroyed-the-virtualization-or-not/vm_1.png)

![VM fragmentation pattern 2](/images/kubernetes-destroyed-the-virtualization-or-not/vm_2.png)

![VM fragmentation pattern 3](/images/kubernetes-destroyed-the-virtualization-or-not/vm_3.png)

## The Resource Density Problem

Here's real data from a cluster showing CPU allocation:

```
CPU Requests: 55 (98%)
CPU Limits: 101300m (180%)
Memory Requests: 86376Mi (44%)
Memory Limits: 121672Mi (62%)
```

On 56-core hardware with hyperthreading, 98% CPU request rate. This reflects pods requiring baseline CPU allocation even when their runtime consumption is low. The cluster looks "full" based on requests, but the actual utilization is much lower. You can't schedule more pods, but the hardware isn't working hard.

## The Case for Kubernetes on Virtual Machines

Running Kubernetes on virtualized infrastructure solves these problems cleanly:

**Provisioning flexibility:** Tune VMs based on role rather than retrofitting bare metal. Need a storage node? Create a VM with that profile. Need more compute? Adjust the VM spec.

**Fault tolerance:** Live migration covers hardware failures transparently. A physical host goes down — vSphere moves the VMs. The Kubernetes cluster doesn't know anything happened.

**Maintenance ease:** Blue-green deployment and rolling updates at the VM layer become possible. Update hypervisor hosts one at a time while workloads migrate.

**Scalability:** Higher node density on the same hardware. Instead of one giant bare-metal Kubernetes node, run five smaller VMs that give you better failure domain isolation.

**Operational flexibility:** Easier datacenter migration. Team scaling becomes more straightforward — different teams can own different parts of the virtualization stack.

![Team and scale considerations](/images/kubernetes-destroyed-the-virtualization-or-not/team_scale.png)

**Consolidated technology:** VMware-based operations with unified management means your ops team has one operational model instead of two.

## The Industry Already Answered This

Look at how the major cloud providers deliver managed Kubernetes:

- **GKE** runs on Google Compute Engine VMs
- **AKS** runs on Azure VMs
- **EKS** runs on EC2 instances

They're not running Kubernetes on bare metal. The biggest Kubernetes deployments in the world sit on virtual machines.

If it's good enough for Google, Amazon, and Microsoft, it should prompt some reflection before declaring virtualization dead.

## Conclusion

Virtualization and Kubernetes are not antagonists — they're complementary. Rather than replacing virtualization, Kubernetes has shifted its role. Instead of VMs hosting applications directly, VMs now provide a homogeneous, manageable platform foundation on which Kubernetes runs.

The result is better than either layer alone: you get Kubernetes's container orchestration and developer experience, with virtualization's operational maturity, live migration, and maintenance tooling underneath.

Keep both. Use both. They're better together.
