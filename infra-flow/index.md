# Infrastructure Logical Flow


SRE + Architect view - sites, overlay VPN, Kubernetes, monitoring, backup, identity.

> IPs below are illustrative (RFC 1918 / RFC 5737 style) and do not reflect real addresses.

## Full Logical Overview

All sites, hub-spoke VPN (StrongSwan IKEv1, hub at Casa site), Kubernetes clusters (izanagi k3s, itachi microk8s, judge k3s, oracle microk8s), central observability (deva), object storage / SSO / API gateway on izanagi, NAS backup on BRAiN.

{{< mermaid >}}
flowchart TB
    classDef ext fill:#FF7043,stroke:#BF360C,color:#fff
    classDef hub fill:#4CAF50,stroke:#1B5E20,color:#fff,stroke-width:3px
    classDef gw fill:#2196F3,stroke:#0D47A1,color:#fff
    classDef k8s fill:#9C27B0,stroke:#4A148C,color:#fff
    classDef mon fill:#FFC107,stroke:#FF6F00,color:#000
    classDef stor fill:#607D8B,stroke:#263238,color:#fff
    classDef host fill:#ECEFF1,stroke:#37474F,color:#000
    classDef inactive fill:#9E9E9E,stroke:#424242,color:#fff,stroke-dasharray: 5 5

    subgraph EXT[Internet - SaaS]
        USER[Users - Admins]:::ext
        CF[Cloudflare Edge, wildcard TLS]:::ext
        DDNS[DDNS provider, milano.example-ddns.net]:::ext
        DDOG[Datadog SaaS]:::ext
        GCLOUD[Grafana Cloud]:::ext
    end

    subgraph CASA[Casa - 192.168.50.0-24]
        ROUTER1[Router 192.168.50.1, PF UDP 500-4500]:::gw
        ASURA[asura .254, Win2025 AD-DNS]:::host
        MINATO[minato .18, StrongSwan 5.9.13 hub]:::hub
        MADARA[madara .21, Proxmox 9.1.5]:::host
        PAIN[pain .9, Proxmox 9.1.5]:::host
        IZANAGI[izanagi k3s v1.36.2, .15 VM on madara]:::k8s
        ITACHI[itachi microk8s v1.32.9, .22 VM on pain]:::k8s
        DEVA[deva .28, Grafana 12.3 - InfluxDB - Loki]:::mon
        BRAIN[BRAiN Synology, NFS -volume1, 19TB RAID5]:::stor
        MJOLNIR[mjolnir Tinkerboard, Graphite stack]:::host
        MF286D[mf286d, Emergency network, LTE failover WAN, standby]:::inactive
    end

    subgraph MILANO[Milano - 192.168.60.0-24]
        OTOHIBE[otohibe .101, StrongSwan client, SNAT to .105, DNAT 8428-8006]:::gw
        AMATERASU[amaterasu .100, Proxmox]:::host
        CARONTE[caronte .102, Telegraf]:::host
        JUDGE[judge .103, k3s + VictoriaMetrics NP 30428]:::k8s
        EINSTEIN[einstein .120, local Collectd]:::host
    end

    subgraph VARESE[Site C - 192.168.70.0-24]
        MATUSA[matusa .54, Banana Pi, StrongSwan client only]:::gw
    end

    subgraph MENDRISIO[Site D - 192.168.80.0-24]
        MENDR[mendrisio .22, RPi4, StrongSwan + WireGuard 10.88.211.0-24]:::gw
    end

    subgraph OCI[Public Cloud - 10.77.254.0-24]
        ORACLE[cloud-instance .135, microk8s v1.27, ARM]:::k8s
    end

    USER -->|HTTPS| CF
    CF -->|tunnel| IZANAGI
    USER -->|WireGuard| MENDR
    USER -->|VPN| ROUTER1

    ROUTER1 -.->|forward 500-4500| MINATO
    DDNS -.->|DDNS resolve| ROUTER1

    MINATO <==>|IKEv1 ESP-in-UDP .105| OTOHIBE
    MINATO <==>|IKEv1 .104| MATUSA
    MINATO <==>|IKEv1 .101| MENDR
    MINATO <==>|IKEv1 .102| ORACLE

    MADARA --- IZANAGI
    PAIN --- ITACHI
    AMATERASU --- JUDGE

    CARONTE -->|via .101 SNAT| OTOHIBE
    JUDGE -->|via .101 SNAT| OTOHIBE

    OTOHIBE -->|Collectd UDP 25826| DEVA
    CARONTE -->|Telegraf HTTP 8086| DEVA
    JUDGE -->|Telegraf HTTP 8086| DEVA
    MINATO -->|Collectd + Promtail| DEVA
    MATUSA -->|Collectd UDP 25826| DEVA
    MENDR -->|Collectd UDP 25826| DEVA
    ORACLE -->|Collectd UDP 25826| DEVA
    IZANAGI -->|VMSingle NP 30090 scrape| DEVA

    ORACLE -.->|Datadog agent| DDOG
    ORACLE -.->|Grafana agent| GCLOUD
    IZANAGI -.->|APM| DDOG

    MADARA -.->|vzdump NFS Sun 03:00| BRAIN
    PAIN -.->|vzdump NFS Sun 03:00| BRAIN
    AMATERASU -.->|vzdump local| AMATERASU

    ASURA -.->|DNS upstream| IZANAGI
    ASURA -.->|DNS upstream| ITACHI

    ROUTER1 -.->|planned mwan3 failover, not live| MF286D
{{< /mermaid >}}

**Legend**

- `hub` (green) - StrongSwan VPN hub (minato)
- `gw` (blue) - site gateway / VPN endpoint
- `k8s` (purple) - Kubernetes cluster / node
- `mon` (yellow) - monitoring / metrics
- `stor` (grey) - storage / NAS
- `ext` (orange) - internet / SaaS
- `inactive` (grey dashed) - standby / planned, not live
- thick `==` = IPsec tunnel, `-->` = data flow, `-.->` = optional / async

**mf286d** - ZTE MF286D LTE router (OpenWrt), planned secondary/failover WAN for Casa via Vodafone SIM. `mwan3` installed and ready but only one node can be powered at a time today: mf286d ships with LAN `192.168.1.1`, same as `router1`, so it's run standalone for now. Also blocked on the modem's PDP session (`wwan` up but data disconnected). Once reassigned to `192.168.1.2/24` with DHCP disabled, it can join Casa as a real dual-WAN failover path.

## VPN Overlay - StrongSwan IKEv1 Hub-and-Spoke

Subnet `172.20.5.0/24` (illustrative). Hub: minato @ Casa. Encryption AES-256-CBC, HMAC-SHA1-96, ESP-in-UDP (NAT-T 4500). IKE rekey 23h, ESP 60min. No cross-spoke routing, all traffic transits minato.

{{< mermaid >}}
flowchart LR
    classDef hub fill:#4CAF50,stroke:#1B5E20,color:#fff,stroke-width:3px
    classDef spoke fill:#2196F3,stroke:#0D47A1,color:#fff
    classDef inactive fill:#9E9E9E,stroke:#424242,color:#fff,stroke-dasharray: 5 5

    MINATO[minato 192.168.50.18, VPN hub, 7 conn - 4 active]:::hub

    OTOHIBE[otohibe - Milano, 203.0.113.10, VPN .105, fix-milano]:::spoke
    MENDR[mendrisio - Site D, 203.0.113.20, VPN .101, fix-mendrisio]:::spoke
    ORACLE[cloud - Public Cloud, 203.0.113.30, VPN .102, fix-oracle]:::spoke
    MATUSA[matusa - Site C, 203.0.113.40, VPN .104, fix-matusa]:::spoke

    ROADW[roadw pool .0-29]:::inactive
    FIXCH[fix-ch .103]:::inactive
    LINGRP[linux-group .128-29]:::inactive

    OTOHIBE <==>|NAT-T 4500 to 10496| MINATO
    MENDR <==>|NAT-T 4500 to 4500| MINATO
    ORACLE <==>|NAT-T 4500 to 4500| MINATO
    MATUSA <==>|NAT-T 4500 to 4500| MINATO

    ROADW -.-> MINATO
    FIXCH -.-> MINATO
    LINGRP -.-> MINATO

    subgraph CASA_PF[Casa Router]
        PF1[UDP 500 to .18-500]
        PF2[UDP 4500 to .18-4500]
    end
    PF1 --> MINATO
    PF2 --> MINATO

    subgraph DNAT[otohibe DNAT 172.20.5.105]
        D1[":8428 to 192.168.60.103-30428, VictoriaMetrics judge"]
        D2[":8006 to 192.168.60.100-8006, Proxmox amaterasu"]
    end
    OTOHIBE --- DNAT
{{< /mermaid >}}

**PKI** - internal root CA, RSA-4096, server cert on internal wildcard domain.
**SNAT chain** - caronte/judge (192.168.60.x) to otohibe SNAT to 172.20.5.105 to tunnel to minato to Casa LAN.

## izanagi - k3s v1.36.2 Cluster (Casa)

Single-node k3s on VM/madara. Cilium CNI + Hubble, Traefik ingress, cert-manager, VictoriaMetrics stack. Hosts SSO (Keycloak), API gateway (Kong), object storage (MinIO), Argo Workflows, WireGuard, cloudflared tunnel, Cloudflare GraphQL exporter (edge/tunnel metrics on the free plan).

{{< mermaid >}}
flowchart TB
    classDef ns fill:#9C27B0,stroke:#4A148C,color:#fff
    classDef sys fill:#1976D2,stroke:#0D47A1,color:#fff
    classDef data fill:#607D8B,stroke:#263238,color:#fff
    classDef mon fill:#FFC107,stroke:#FF6F00,color:#000
    classDef edge fill:#FF7043,stroke:#BF360C,color:#fff

    CF[Cloudflare Edge, wildcard TLS]:::edge

    subgraph IZANAGI[izanagi 192.168.50.15 - k3s v1.36.2]
        subgraph KSYS[kube-system]
            CILIUM[Cilium + Hubble]:::sys
            COREDNS[CoreDNS, upstream 192.168.50.254]:::sys
            TRAEFIK[Traefik IngressRoute]:::sys
            METRICS[metrics-server]:::sys
        end

        subgraph EDGE_NS[edge]
            CFD[cloudflared 2 pods]:::ns
        end

        subgraph AUTH_NS[auth]
            KEYCLOAK[Keycloak SSO]:::ns
            KCPG[Keycloak PG]:::data
        end

        subgraph GW_NS[gateway]
            KONG[Kong API GW]:::ns
            KONGPG[Kong PG]:::data
            KONGA[Konga UI]:::ns
        end

        subgraph APP_NS[apps]
            ARGO[Argo WF v3.6.18]:::ns
            APACHERR[web services]:::ns
            GUAC[Guacamole + guacd]:::ns
            IMGPROXY[imgproxy]:::ns
            REDIS[Redis + Webdis]:::ns
            POSTFIX[Postfix NP 30025]:::ns
            PYTBAK[pytbak APM]:::ns
        end

        subgraph DATA_NS[data]
            MINIO[MinIO S3]:::data
            PGSHARED[PostgreSQL shared]:::data
        end

        subgraph NET_NS[network]
            WG[WireGuard NP 31820]:::ns
        end

        subgraph MON_NS[monitoring]
            VMOP[VMOperator]:::mon
            VMA[VMAgent, 21 scrape targets]:::mon
            VMS[VMSingle NP 30090, retain 90d - 10Gi]:::mon
            NODE[node-exporter]:::mon
            KSM[kube-state-metrics]:::mon
            CADV[cAdvisor DS]:::mon
        end

        subgraph CFEXP_NS[cloudflare-exp]
            CFEXP[cf-graphql-exporter]:::mon
        end

        CM[cert-manager 3 pods]:::sys
    end

    CF -->|tunnel| CFD
    CFD --> TRAEFIK
    TRAEFIK --> KEYCLOAK
    TRAEFIK --> KONG
    TRAEFIK --> APACHERR
    TRAEFIK --> GUAC
    TRAEFIK --> ARGO

    KEYCLOAK --> KCPG
    KONG --> KONGPG
    KONG --> KEYCLOAK
    KONGA --> KONG

    VMA -->|scrape| NODE
    VMA -->|scrape| KSM
    VMA -->|scrape| CADV
    VMA -->|scrape| CILIUM
    VMA -->|scrape 20s| CFEXP
    VMA --> VMS
    VMOP -.-> VMA
    CFEXP --> VMS

    CF -.->|GraphQL API pull 60s| CFEXP

    DEVA[deva Grafana .28]:::mon
    DEVA -->|datasource :30090| VMS

    PYTBAK -.->|APM| DDOG[Datadog SaaS]:::edge
{{< /mermaid >}}

**Storage** - k3s `local-path` provisioner.
**TLS** - Cloudflare edge for public, cert-manager internal.
**DNS** - CoreDNS upstream to asura (Win AD DC 192.168.50.254).
**Cloudflare metrics** - `cf-graphql-exporter` (ns `cloudflare-exp`) pulls the Cloudflare GraphQL/REST API every 60s (free-plan zone + tunnel analytics), scraped by VMAgent every 20s into the same local VMSingle as the rest of the cluster metrics.

## Monitoring & Observability Flow

Central stack on deva (192.168.50.28): Collectd RX (UDP 25826), InfluxDB (HTTP 8086, DBs `collectd` + `telegraf`), Loki (TCP 3100), Grafana 12.3. izanagi runs self-contained VictoriaMetrics queried via NP 30090.

{{< mermaid >}}
flowchart LR
    classDef agent fill:#2196F3,stroke:#0D47A1,color:#fff
    classDef store fill:#607D8B,stroke:#263238,color:#fff
    classDef ui fill:#FFC107,stroke:#FF6F00,color:#000

    OTOHIBE[otohibe]:::agent
    MINATO[minato]:::agent
    MATUSA[matusa Site C]:::agent
    MENDR[mendrisio]:::agent
    ORACLE[cloud instance]:::agent

    CARONTE[caronte, Telegraf 10s]:::agent
    JUDGE[Judge k3s, Telegraf 60s]:::agent

    EINSTEIN[einstein, local CSV only]:::agent
    IZANAGI[izanagi VMAgent]:::agent

    subgraph DEVA[deva 192.168.50.28]
        COLLD[Collectd RX, UDP :25826]:::store
        INFLUX[InfluxDB :8086, db collectd, db telegraf]:::store
        LOKI[Loki :3100]:::store
        GRAF[Grafana 12.3]:::ui
        PROM[Prometheus]:::store
    end

    VMS[VMSingle on izanagi, NP :30090, 90d retention]:::store

    OTOHIBE -->|UDP binary| COLLD
    MINATO -->|UDP binary| COLLD
    MATUSA -->|UDP binary| COLLD
    MENDR -->|UDP binary| COLLD
    ORACLE -->|UDP binary| COLLD

    CARONTE -->|HTTP line proto| INFLUX
    JUDGE -->|HTTP line proto| INFLUX

    MINATO -->|Promtail TCP 3100| LOKI

    IZANAGI --> VMS

    COLLD --> INFLUX
    INFLUX --> GRAF
    LOKI --> GRAF
    VMS -->|HTTP datasource| GRAF
    PROM --> GRAF

    EINSTEIN -.->|local CSV only, remote shipping commented| DEVA

    ORACLE -.->|Datadog agent in-cluster| DDOG[Datadog SaaS]
    ORACLE -.->|Grafana agent| GCLOUD[Grafana Cloud]
{{< /mermaid >}}

**Why two protocols?** Collectd legacy hosts use native UDP binary to the `collectd` DB. Newer hosts use Telegraf HTTP InfluxDB line protocol to the `telegraf` DB (collectd serializer unavailable in Telegraf).

**Dashboards**: 19 exported, 14 base + 3 izanagi (cluster, pods, cilium) + 2 Cloudflare (tunnels, edge free plan).

## Backup Topology

Two independent flows. Infra-side: Proxmox vzdump weekly Sunday 03:00, all VMs/LXC on every Proxmox hypervisor - Casa hypervisors (madara, pain) to BRAiN NFS, Milano (amaterasu) to local `/var/lib/vz/dump`. Data-side: an offsite archive of the workstation's own filesystem, pushed via restic to a block volume on the cloud instance - unrelated to vzdump, its own schedule and retention.

{{< mermaid >}}
flowchart LR
    classDef ok fill:#4CAF50,stroke:#1B5E20,color:#fff
    classDef store fill:#607D8B,stroke:#263238,color:#fff
    classDef client fill:#ECEFF1,stroke:#37474F,color:#000

    MADARA[madara .21, Proxmox 9.1.5, minato deva asura izanagi]:::ok
    PAIN[pain .9, Proxmox 9.1.5, itachi kabuto thebridge underthebridge]:::ok
    AMATERASU[amaterasu Milano, Proxmox, otohibe caronte judge]:::ok

    BRAIN[BRAiN Synology, NFS -volume1-backup-vm, 19TB RAID5]:::store
    LOCAL[amaterasu local, -var-lib-vz-dump]:::store

    MADARA -->|vzdump zstd suspend, retain 4 - VMID, Sun 03:00| BRAIN
    PAIN -->|vzdump zstd suspend, retain 4 - VMID, Sun 03:00| BRAIN
    AMATERASU -->|vzdump local, retain 2 - VMID, Sun 03:00| LOCAL

    LAPTOP[workstation, macOS, Backrest UI]:::client
    ARCHIVE[cloud instance, dedicated block volume, SFTP chroot target]:::store

    LAPTOP -->|restic via Backrest, SFTP, AES-256 client-side, weekly| ARCHIVE
{{< /mermaid >}}

**Cron** `/etc/cron.d/proxmox-backup` needs `SHELL=/bin/bash`. Script `/root/proxmox-backup.sh`. Log `/var/log/proxmox-backup.log`.

**restic/Backrest** - [Backrest](https://github.com/garethgeorge/backrest) (a local web UI + scheduler wrapping [restic](https://restic.net/)) runs on the workstation itself, not on the cloud side; the cloud instance is only a chrooted SFTP target with its own locked-down user. Client-side encrypted, deduplicated, incremental. Transits directly rather than the VPN overlay, since the workstation isn't a peer on that instance's WireGuard interface - a known gap, on the list to close.

**Note** - deva runs as an LXC on madara, so it's covered by vzdump like every other VM/LXC; the only open gap is that InfluxDB isn't backed up with an app-consistent snapshot (`influxd backup`) on top of the filesystem-level vzdump. Non-Proxmox hosts (BRAiN itself, matusa, mendrisio, the cloud instance) sit outside the vzdump flow entirely since there's no hypervisor to vzdump - and BRAiN, being the vzdump *destination*, has no backup of its own either: single point of failure for all Casa backups. See `backup/strategy.md` for the full per-host gap list and recommendations.

## Ingress, Identity & East-West Service Mesh

North-south: Cloudflare Tunnel to cloudflared to Traefik IngressRoute. Auth: Keycloak SSO. API: Kong gateway. Internal DNS: Win2025 AD (asura .254).

{{< mermaid >}}
sequenceDiagram
    autonumber
    participant U as User Browser
    participant CF as Cloudflare Edge
    participant CFD as cloudflared pod
    participant TR as Traefik
    participant KC as Keycloak
    participant KG as Kong
    participant APP as Upstream svc
    participant PG as PostgreSQL
    participant DNS as asura AD DC

    U->>CF: HTTPS app
    CF->>CFD: tunnel mTLS
    CFD->>TR: HTTP IngressRoute
    TR->>DNS: resolve upstream
    TR->>KG: forward, Kong route match
    KG->>KC: OIDC token validate
    KC->>PG: user-session lookup
    KC-->>KG: JWT valid
    KG->>APP: proxy plus plugins
    APP-->>U: response
{{< /mermaid >}}

## Packet Flow - Milano to Casa (SNAT chain)

caronte traffic to Casa traverses otohibe SNAT (to 172.20.5.105), encapsulated ESP-in-UDP to minato.

{{< mermaid >}}
sequenceDiagram
    autonumber
    participant C as caronte 192.168.60.102
    participant O as otohibe 192.168.60.101
    participant R1 as Milano Router
    participant NET as Internet
    participant R2 as Casa Router
    participant M as minato 192.168.50.18
    participant T as Target 192.168.50.x

    C->>O: src .100.102 dst 192.168.50.x, route via .100.101
    Note over O: SNAT src to 172.20.5.105, XFRM encrypt ESP
    O->>R1: ESP-in-UDP dst Casa_public:4500
    R1->>NET: NAT public IP
    NET->>R2: UDP :4500
    R2->>M: PF UDP 4500
    Note over M: Decrypt ESP, inner src 172.20.5.105 to dst .1.x
    M->>T: deliver LAN
    T->>M: reply dst 172.20.5.105
    Note over M: XFRM match, encrypt
    M->>R2: ESP-in-UDP
    R2->>NET: NAT
    NET->>R1: UDP :4500
    R1->>O: forward
    Note over O: SNAT reverse, dst .100.102
    O->>C: src .1.x dst .100.102
{{< /mermaid >}}

