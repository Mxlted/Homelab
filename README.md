# Homelab

Personal homelab setup, configurations, and automation for self-hosted infrastructure. This repository serves as a centralized documentation hub for my local server environment, containerized services, and network configurations.

---

## Infrastructure Overview

| Component | Details |
|-----------|---------|
| **Hypervisor** | Proxmox VE - LXC containers and VM management |
| **Storage** | TrueNAS VM with HDD passthrough; NFS shares exported to Proxmox host across three pools (`tank/Media`, `tank/Immich`, `ssd/data`) |
| **Containers** | Docker managed via Portainer, deployed inside an unprivileged LXC |
| **Reverse Proxy** | Pangolin - handles ingress and SSL termination for internal services |
| **Dashboard** | Homepage - self-hosted service dashboard with Docker socket integration |
| **DNS / Filtering** | AdGuard Home with OISD blocklist and custom filter rules |
| **Monitoring** | Watchtower for automated container image updates; changedetection.io for web change alerts |
| **Notifications** | Proxmox system alerts forwarded to Discord via webhook |
| **Hardware** | Intel Core i7-12700K · 4 × WD Red Plus 8 TB HDD · PNY CS3140 4 TB NVMe |

---

## Repository Index

### docker-compose

Docker Compose manifests for all self-hosted services.

| File | Description |
|------|-------------|
| [arrStack.md](./docker-compose/arrStack.md) | Full media automation stack - qBittorrent (with WireGuard VPN), Prowlarr, Radarr, Sonarr, FlareSolverr, and cross-seed |
| [changedetection.yaml](./docker-compose/changedetection.yaml) | changedetection.io with Sockpuppet Chrome for JavaScript-capable web page monitoring |
| [dockhand.yaml](./docker-compose/dockhand.yaml) | Dockhand - Docker stack management UI, mounts compose stacks from `/opt/stacks` |
| [hawser.yaml](./docker-compose/hawser.yaml) | Hawser - token-authenticated Docker socket proxy for remote stack management |
| [homepage.yaml](./docker-compose/homepage.yaml) | Homepage dashboard bound to internal address, config and Docker socket mounted as volumes |
| [openspeedtest.yaml](./docker-compose/openspeedtest.yaml) | OpenSpeedTest - self-hosted LAN/WAN speed test server |
| [updates.yaml](./docker-compose/updates.yaml) | Watchtower - automated container image updates every 6 hours with cleanup of old images |

---

### server-setup

Infrastructure provisioning, service configuration, troubleshooting, and hardware maintenance.

| File | Description |
|------|-------------|
| [adguardhome-filters.md](./server-setup/adguardhome-filters.md) | AdGuard Home configuration - OISD small blocklist plus custom block/allow rules for IPv4, tracking domains, and service-specific allowlist entries |
| [current-proxmox-nfs-mount.md](./server-setup/current-proxmox-nfs-mount.md) | Active fstab entries for mounting TrueNAS NFS shares (`tank/Media`, `tank/Immich`, `ssd/data`) on the Proxmox host, plus LXC bind-mount config |
| [docker-lxc-fix.md](./server-setup/docker-lxc-fix.md) | Fix for OCI runtime error (`net.ipv4.ip_unprivileged_port_start`) in Proxmox LXC - sets `apparmor.profile: unconfined` and masks the AppArmor module |
| [portainer-fix.md](./server-setup/portainer-fix.md) | Docker daemon config (`/etc/docker/daemon.json`) that resolves Portainer API compatibility - adds `min-api-version`, log rotation, disables `containerd-snapshotter`, and preserves the Nvidia runtime and IPv6 settings |
| [portfolio-setup.md](./server-setup/portfolio-setup.md) | Nginx:Alpine container deployment for a personal portfolio site, bound to an internal Docker bridge (`172.19.0.1:7373`) behind Pangolin reverse proxy to prevent direct public exposure |
| [proxmox-discord-notifications.md](./server-setup/proxmox-discord-notifications.md) | Step-by-step setup for forwarding Proxmox alerts to a Discord channel via webhook - includes Proxmox UI config, JSON body template, and secret management |
| [truenas-nfs-proxmox-lxc-docker.md](./server-setup/truenas-nfs-proxmox-lxc-docker.md) | Comprehensive guide for TrueNAS NFS exports through Proxmox into an unprivileged LXC running Docker - covers fstab options by kernel version, `lxc.idmap` UID/GID 1000 mapping, and pre-creating bind mount directories |
| [weekly-printer-maintenance.md](./server-setup/weekly-printer-maintenance.md) | CUPS setup for an Epson network printer inside a Proxmox LXC, with a cron job to auto-print a test page every Wednesday to prevent inkjet clogging |
