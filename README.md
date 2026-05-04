# Homelab

Personal homelab setup, configurations, and automation for self-hosted infrastructure. This repository serves as a centralized documentation hub for my local server environment, containerized services, and network configurations.

---

## Infrastructure Overview

| Component | Details |
|-----------|---------|
| **Hypervisor** | Proxmox VE - LXC containers and VM management |
| **Storage** | TrueNAS VM with HDD passthrough; NFS shares exported to Proxmox host across three pools (`tank/Media`, `tank/Immich`, `ssd/data`) |
| **Containers** | Docker inside an unprivileged LXC, managed via Dockhand and Hawser for remote stack deployment |
| **Reverse Proxy** | Pangolin - handles ingress and SSL termination for internal services |
| **Dashboard** | Homepage - self-hosted service dashboard with Docker socket integration |
| **DNS / Filtering** | AdGuard Home with OISD blocklist and custom filter rules |
| **Monitoring** | Watchtower for automated container image updates (every 6 hours); changedetection.io for web page change alerts |
| **Notifications** | Proxmox system alerts forwarded to Discord via webhook |
| **Hardware** | Intel Core i7-12700K · 4 × WD Red Plus 8 TB HDD · PNY CS3140 4 TB NVMe |

---

## Services

### Media Automation

| Service | Purpose |
|---------|---------|
| **qBittorrent** | Torrent client with WireGuard VPN integration and automatic port forwarding |
| **Prowlarr** | Indexer manager for Sonarr and Radarr |
| **Radarr** | Movie collection manager and automation |
| **Sonarr** | TV series collection manager and automation |
| **FlareSolverr** | Cloudflare bypass proxy for indexers |
| **cross-seed** | Automated cross-seeding for ratio optimization |

### Infrastructure & Management

| Service | Purpose |
|---------|---------|
| **Homepage** | Service dashboard with Docker socket integration |
| **Dockhand** | Web UI for managing Docker Compose stacks |
| **Hawser** | Token-authenticated Docker socket proxy for remote management |
| **Watchtower** | Automated container image updates with cleanup |
| **changedetection.io** | Web page change monitoring with JavaScript rendering |
| **OpenSpeedTest** | Self-hosted LAN/WAN speed testing |

### Other Services

| Service | Purpose |
|---------|---------|
| **Portfolio** | Personal portfolio site (Nginx:Alpine behind Pangolin reverse proxy) |
| **CUPS** | Network printer server with scheduled maintenance prints |
| **AdGuard Home** | DNS-level ad blocking and filtering |

---

## Repository Index

### docker-compose

Docker Compose manifests for all self-hosted services.

| File | Description |
|------|-------------|
| [arrStack.md](./docker-compose/arrStack.md) | Media automation stack - qBittorrent (WireGuard VPN, auto port forward), Prowlarr, Radarr, Sonarr, FlareSolverr, cross-seed |
| [changedetection.yaml](./docker-compose/changedetection.yaml) | changedetection.io with Sockpuppet Chrome for JavaScript-capable page monitoring |
| [dockhand.yaml](./docker-compose/dockhand.yaml) | Dockhand - Docker stack management UI, mounts stacks from `/opt/stacks` |
| [hawser.yaml](./docker-compose/hawser.yaml) | Hawser - token-authenticated Docker socket proxy for remote stack management |
| [homepage.yaml](./docker-compose/homepage.yaml) | Homepage dashboard with Docker socket integration and env-based secrets |
| [openspeedtest.yaml](./docker-compose/openspeedtest.yaml) | OpenSpeedTest - self-hosted LAN/WAN speed test server |
| [updates.yaml](./docker-compose/updates.yaml) | Watchtower - automated container updates every 6 hours with image cleanup |

---

### server-setup

Infrastructure provisioning, service configuration, troubleshooting, and hardware maintenance.

| File | Description |
|------|-------------|
| [adguardhome-filters.md](./server-setup/adguardhome-filters.md) | AdGuard Home configuration - OISD small blocklist plus custom block/allow rules for tracking domains and service-specific allowlists |
| [current-proxmox-nfs-mount.md](./server-setup/current-proxmox-nfs-mount.md) | Active fstab entries for TrueNAS NFS mounts (`tank/Media`, `tank/Immich`, `ssd/data`) and LXC bind-mount configuration |
| [docker-lxc-fix.md](./server-setup/docker-lxc-fix.md) | Fix for OCI runtime error (`net.ipv4.ip_unprivileged_port_start`) in Proxmox LXC - AppArmor unconfined profile and module masking |
| [portainer-fix.md](./server-setup/portainer-fix.md) | Docker daemon config resolving Portainer API compatibility - includes `min-api-version`, log rotation, Nvidia runtime, and IPv6 settings |
| [portfolio-setup.md](./server-setup/portfolio-setup.md) | Nginx:Alpine portfolio deployment bound to internal Docker bridge (`172.19.0.1:7373`) behind Pangolin to prevent direct public exposure |
| [proxmox-discord-notifications.md](./server-setup/proxmox-discord-notifications.md) | Proxmox → Discord webhook integration for system alerts - includes UI config, JSON body template, and secret management |
| [truenas-nfs-proxmox-lxc-docker.md](./server-setup/truenas-nfs-proxmox-lxc-docker.md) | Complete guide: TrueNAS NFS → Proxmox → unprivileged LXC → Docker with UID/GID 1000 mapping and fstab options |
| [weekly-printer-maintenance.md](./server-setup/weekly-printer-maintenance.md) | CUPS setup for Epson network printer with cron-scheduled test prints to prevent inkjet clogging |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Proxmox VE Host                         │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    TrueNAS VM (HDD Passthrough)           │  │
│  │         4 × WD Red Plus 8 TB → ZFS Pool (tank)            │  │
│  │              NFS Exports: Media, Immich, ssd/data         │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                         NFS Mounts                               │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              Unprivileged LXC (Docker Host)               │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  Docker Services                                    │  │  │
│  │  │  • Media Stack (qBit/Sonarr/Radarr/Prowlarr)       │  │  │
│  │  │  • Homepage Dashboard                               │  │  │
│  │  │  • Dockhand / Hawser (Stack Management)            │  │  │
│  │  │  • Watchtower / changedetection.io                 │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                    Pangolin Reverse Proxy                        │
│                              ▼                                   │
└─────────────────────────────────────────────────────────────────┘
                          Internet
```
