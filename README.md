# Homelab

Personal homelab setup, configurations, and automation for self-hosted infrastructure. This repository serves as a centralized documentation hub for my local server environment, containerized services, and network configurations.

## Core Infrastructure

* **Hypervisor**: Proxmox VE (LXC and VM management).
* **Storage**: TrueNAS-backed NFS mounts for persistent container data.
* **Containers**: Docker managed via Portainer for service orchestration.
* **Networking**: Automated Discord notifications for system status and alerts.

---

## Repository Index

### 📂 docker-compose
*Deployment manifests and container orchestration.*

| Document | Description |
| :--- | :--- |
| [arrStack.md](./docker-compose/arrStack.md) | Configuration for the Media Automation stack (Prowlarr, Radarr, Sonarr). |
| [updates.md](./docker-compose/updates.md) | Automation for container image updates using Watchtower. |

### 📂 server-setup
*Infrastructure provisioning, troubleshooting, and hardware maintenance.*

| Document | Description |
| :--- | :--- |
| [current-proxmox-nfs-mount.md](./server-setup/current-proxmox-nfs-mount.md) | Configuring NFS mounts within Proxmox LXC containers. |
| [docker-lxc-fix.md](./server-setup/docker-lxc-fix.md) | Resolving OCI runtime issues for Docker on Proxmox LXC. |
| [portainer-fix.md](./server-setup/portainer-fix.md) | Portainer configuration fixes and logging options. |
| [portfolio-setup.md](./server-setup/portfolio-setup.md) | Documentation for the deployment of the personal portfolio site. |
| [proxmox-discord-notifications.md](./server-setup/proxmox-discord-notifications.md) | Proxmox system alert integration via Discord Webhooks. |
| [truenas-nfs-proxmox-lxc-docker.md](./server-setup/truenas-nfs-proxmox-lxc-docker.md) | Guide for TrueNAS NFS exports to Proxmox and Docker. |
| [weekly-printer-maintenance.md](./server-setup/weekly-printer-maintenance.md) | Procedures for Epson printer setup and automation. |
