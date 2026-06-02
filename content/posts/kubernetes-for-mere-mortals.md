---
title: "Kubernetes for Mere Mortals"
date: 2018-04-10
draft: false
description: "Building a homemade Kubernetes cluster on ARM OrangePi hardware — from zero to running pods with Traefik ingress and Grafana monitoring."
tags:
  - kubernetes
  - homelab
  - arm
  - hardware
  - monitoring
featuredImage: /images/kubernetes-for-mere-mortals/k8s.arm-lg.jpeg
---

## Hardware Setup

Here we are — a homemade Kubernetes cluster built on ARM hardware. No cloud credits. No expensive servers. Just OrangePis and a USB hub.

Bill of materials:

- 1x Anker 60W PowerPort 6 USB hub (power for all nodes)
- 4x Orange Pi Plus 2E single-board computers

![The cluster](/images/kubernetes-for-mere-mortals/k8s.arm-lg.jpeg)

![OrangePi nodes](/images/kubernetes-for-mere-mortals/IMG_20170605_150237.jpg)

Total cost: well under what a single month of cloud compute would run.

## Software Stack

- **OS:** Armbian 4.10.1 (Debian-based, ARM-optimized)
- **Kubernetes:** kubeadm + kubelet 1.6.4
- **Ingress controller:** Traefik
- **Monitoring:** Heapster + collectd + InfluxDB + Grafana

## Cluster Status

The `kubectl get pods --all-namespaces` view:

**`home-prd` namespace:**
- `k8s-helloworld` — 3 replicas, all running

**`kube-system` namespace:**
- `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler` — control plane
- `kube-flannel` — networking (CNI)
- `kube-dns` — DNS resolution
- `kube-proxy` — service routing
- `kubernetes-dashboard` — web UI
- `traefik-ingress-controller` — external traffic ingress

All nodes reporting Ready. All pods Running.

## Monitoring

Heapster feeds metrics into InfluxDB. Grafana visualizes everything.

The cluster monitors itself: CPU per node, memory per pod, network throughput. On ARM hardware with limited RAM, knowing which pods are heavy consumers matters.

## Lessons

ARM in 2018 was rough around the edges. Some container images didn't have ARM builds. Compilation from source was sometimes the only option. Heapster was already being deprecated.

But it worked. Traefik served traffic. Pods scaled. The monitoring stack showed real data.

The point wasn't production-readiness. The point was understanding — really understanding — what Kubernetes does, by building it yourself on hardware you own.

That understanding doesn't come from clicking through a managed service dashboard.
