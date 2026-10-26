# Homelab

Personal homelab setup, configurations, and automation for self-hosted infrastructure. This repository serves as a centralized documentation hub for my local server environment, containerized services, and network configurations.

---

## Infrastructure Overview

| Component | Details |
|-----------|---------|
| **Hypervisor** | Proxmox VE - LXC containers and VM management |
| **Storage** | Proxmox-managed ZFS `tank` pool: 4 × 8 TB WD Red Plus HDDs in RAIDZ1, with `tank/Media` at `/tank/Media` and `tank/Immich` at `/tank/Immich` (local mounts, no TrueNAS or NFS layer) |
| **Containers** | Proxmox LXCs: 100 (`Media`), 101 (`Immich`), and 102 (`Cron`); Media and Immich each bind-mount their local ZFS dataset to `/mnt/nas` |
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
| [current-proxmox-nfs-mount.md](./server-setup/current-proxmox-nfs-mount.md) | Legacy TrueNAS NFS/fstab configuration (historical reference; no longer in use) |
| [docker-lxc-fix.md](./server-setup/docker-lxc-fix.md) | Fix for OCI runtime error (`net.ipv4.ip_unprivileged_port_start`) in Proxmox LXC - AppArmor unconfined profile and module masking |
| [portainer-fix.md](./server-setup/portainer-fix.md) | Docker daemon config resolving Portainer API compatibility - includes `min-api-version`, log rotation, Nvidia runtime, and IPv6 settings |
| [portfolio-setup.md](./server-setup/portfolio-setup.md) | Nginx:Alpine portfolio deployment bound to internal Docker bridge (`172.19.0.1:7373`) behind Pangolin to prevent direct public exposure |
| [proxmox-discord-notifications.md](./server-setup/proxmox-discord-notifications.md) | Proxmox → Discord webhook integration for system alerts - includes UI config, JSON body template, and secret management |
| [truenas-nfs-proxmox-lxc-docker.md](./server-setup/truenas-nfs-proxmox-lxc-docker.md) | Legacy guide for TrueNAS NFS → Proxmox → unprivileged LXC → Docker (historical reference) |
| [truenas-pool-to-proxmox.md](./server-setup/truenas-pool-to-proxmox.md) | Migrate an existing TrueNAS VM ZFS pool to the Proxmox host without recreating it, including passthrough removal, local mounts, verification, and rollback |
| [weekly-printer-maintenance.md](./server-setup/weekly-printer-maintenance.md) | CUPS setup for Epson network printer with cron-scheduled test prints to prevent inkjet clogging |

---

## Architecture

Proxmox owns and mounts the four-drive RAIDZ1 pool directly; the Media and Immich LXCs use separate host-local ZFS bind mounts, with no TrueNAS VM or NFS layer. The host OS and VM/LXC root volumes are stored on the PNY CS3140 NVMe via LVM/LVM-thin.

| LXC ID | Name | Host ZFS source | Container mount |
|--------|------|-----------------|-----------------|
| **100** | Media | `/tank/Media` | `/mnt/nas` |
| **101** | Immich | `/tank/Immich` | `/mnt/nas` |
| **102** | Cron | No ZFS bind mount configured | — |

The mappings above are confirmed by `/etc/pve/lxc/*.conf`. ZFS import and mount services were enabled and completed successfully at the latest inspected host boot (October 23, 2026). Confirm mount availability **inside** each LXC separately when troubleshooting.

```text
┌─────────────────────────────────────────────────────────────────┐
│                         Proxmox VE Host                         │
│                                                                 │
│  PNY CS3140 4 TB NVMe → Proxmox OS + LVM-thin VM/LXC storage    │
│                                                                 │
│  4 × WD Red Plus 8 TB HDDs → ZFS RAIDZ1 pool (tank)             │
│  ├── /tank/Media ─────── local bind mount ──┐                   │
│  └── /tank/Immich ────── local bind mount ──┼──────────┐        │
│                                            │          │        │
│  ┌─────────────────────────────────────────▼─┐  ┌─────▼──────┐ │
│  │ LXC 100: Media                            │  │ LXC 101:   │ │
│  │ /mnt/nas → /tank/Media                     │  │ Immich     │ │
│  │ Media automation / Docker services        │  │ /mnt/nas   │ │
│  │                                          │  │            │ │
│  └──────────────────────────────────────────┘  └────────────┘ │
│                                                                 │
│  LXC 102: Cron (no ZFS bind mount configured)                   │
│                                                                 │
│  Other infrastructure: Homepage, Dockhand/Hawser, Pangolin,     │
│  Watchtower, changedetection.io, and supporting services        │
└─────────────────────────────────────────────────────────────────┘
```
