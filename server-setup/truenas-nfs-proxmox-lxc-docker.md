# TrueNAS → Proxmox → Unprivileged LXC → Docker: Persistent NFS Storage

> A step-by-step guide to mounting a TrueNAS NFS share on Proxmox, binding it into an unprivileged LXC container, and configuring Docker access while preserving **UID/GID consistency (user 1000)** across all layers.

![Proxmox](https://img.shields.io/badge/Proxmox-7%2B-orange?logo=proxmox)
![TrueNAS](https://img.shields.io/badge/TrueNAS-NFS-blue?logo=truenas)
![Docker](https://img.shields.io/badge/Docker-Compatible-2496ED?logo=docker)
![License](https://img.shields.io/badge/license-MIT-green)

---

## Table of Contents

- [Overview](#overview)
- [Background](#background)
- [System Configuration](#system-configuration)
- [Step 1: Add the NFS Share to Proxmox](#step-1-add-the-nfs-share-to-proxmox)
- [Step 2: Create a Local Mount Point and Configure fstab](#step-2-create-a-local-mount-point-and-configure-fstab)
- [Step 3: Bind-Mount into the Unprivileged LXC](#step-3-bind-mount-into-the-unprivileged-lxc)
- [Step 4: ID Mapping to Preserve UID/GID 1000](#step-4-id-mapping-to-preserve-uidgid-1000)
- [Step 5: Pre-Create Docker Bind Mount Directories](#step-5-pre-create-docker-bind-mount-directories)
- [Verification](#verification)

---

## Overview

This guide covers how to mount a TrueNAS NFS share to a Proxmox host, bind it into an unprivileged LXC container, and allow Docker within that container to use the share directly. The focus is on maintaining consistent file ownership and permissions (UID/GID = 1000) between TrueNAS, Proxmox, and Docker — enabling seamless shared storage access without permission mismatches or root escalation issues.

---

## Background

The goal of this setup is a single, efficient system that serves as both a NAS and an application host without requiring extra dedicated hardware.

TrueNAS is excellent for storage and data protection, but running it bare metal comes with a significant limitation: you cannot install or run apps directly on the boot drive or pool. Adding Docker or other applications requires a separate physical drive or pool.

Rather than that approach, this setup runs TrueNAS as a VM inside Proxmox, which allows:

- Using the Proxmox host for applications and services (Docker, Sonarr, Radarr, etc.)
- Keeping TrueNAS as a dedicated NAS focused purely on storage and file sharing
- HDD passthrough so TrueNAS retains full control over the disks
- Treating TrueNAS as a portable appliance that can be migrated or restored independently

Proxmox handles workloads; TrueNAS provides networked NFS storage.

---

## System Configuration

| Component | Details |
|-----------|---------|
| **CPU** | Intel Core i7-12700K |
| **OS / Apps drive** | PNY CS3140 4 TB NVMe SSD |
| **Storage drives** | 4 × WD Red Plus 8 TB HDD |
| **Host platform** | Proxmox VE |
| **TrueNAS deployment** | VM (not LXC) on the same Proxmox host |
| **Drive configuration** | HDDs passed through directly to the TrueNAS VM |
| **Shared storage** | TrueNAS NFS share exported to Proxmox, bind-mounted into an unprivileged LXC running Docker |
| **NFS export settings** | `mapall` mapped to UID 1000, ensuring consistent ownership across all layers |

> **Note:** Since TrueNAS runs as a VM with direct disk passthrough, NFS performance and permission consistency depend on proper passthrough configuration and NFS export settings — particularly `mapall`.

---

## Step 1: Add the NFS Share to Proxmox

1. In the Proxmox web UI, navigate to **Datacenter → Storage → Add → NFS**.

2. Fill in the required fields:
   - **ID** — a friendly name for the storage entry (e.g., `truenas-nfs`)
   - **Server** — IP address or hostname of your TrueNAS server
   - **Export** — the exported NFS path (e.g., `/mnt/pool1/docker-share`)
   - **Content** — select intended use types (e.g., `Disk image`, `Container`, `ISO image`, `VZDump backup file`)
   - **Nodes** — select which Proxmox nodes should have access
   - Check **Enable**

3. Click **Add**.

To verify, go to **Datacenter → Storage**, select the new entry, and confirm the **Status** tab shows **Active**.

---

## Step 2: Create a Local Mount Point and Configure fstab

Create the directory that will serve as the local mount point on the Proxmox host:

```bash
mkdir -p /mnt/<mount-point>
```

Open `/etc/fstab` for editing:

```bash
nano /etc/fstab
```

### Kernel Version Compatibility

The mount options below apply to **PVE kernel 6.17.13 and later**. If you are on 6.17.9 or earlier, refer to the legacy entry.

**PVE kernel ≤ 6.17.9**

```
<server-ip>:/mnt/<pool>/<dataset>   /mnt/<mount-point>   nfs4   rw,vers=4.1,noatime,_netdev,x-systemd.automount,hard,timeo=600,retrans=5,x-systemd.idle-timeout=600   0   0
```

**PVE kernel ≥ 6.17.13 (current)**

```
<server-ip>:/mnt/<pool>/<dataset>   /mnt/<mount-point>   nfs4   rw,vers=4.1,noatime,_netdev,x-systemd.automount,hard,timeo=600,retrans=5,x-systemd.idle-timeout=600   0   0
```

### Mount Option Reference

| Option | Description |
|--------|-------------|
| `nfs4` | NFS version 4 filesystem type |
| `vers=4.1` | Explicitly requests NFS 4.1 (supports pNFS and session trunking) |
| `rw` | Read and write access |
| `noatime` | Disables access time updates — reduces unnecessary I/O |
| `_netdev` | Defers mount until the network stack is ready |
| `x-systemd.automount` | On-demand mounting via systemd; prevents boot hangs if the share is unavailable |
| `hard` | Retries NFS requests indefinitely until the server responds; safer than `soft` for data integrity |
| `timeo=600` | NFS timeout per attempt in tenths of a second (600 = 60 seconds) |
| `retrans=5` | Number of retries before the client reports an error |
| `x-systemd.idle-timeout=600` | Unmounts the share automatically after 600 seconds of inactivity |
| `0 0` | Disables dump and fsck for this entry (standard for NFS) |

> **`hard` vs `nofail`:** A `hard` mount will block if the NFS server is unreachable at mount time. If you need the system to boot cleanly without TrueNAS available, consider adding `nofail` as well.

After saving, test the mount without rebooting:

```bash
mount -a
```

---

## Step 3: Bind-Mount into the Unprivileged LXC

Use `pct set` to attach the host NFS mount path into your container:

```bash
# <ctid>           — LXC container ID (e.g., 200)
# <host-mount>     — host path of the NFS mount (e.g., /mnt/media-nas)
# <container-path> — path inside the container (e.g., /mnt/nas)

pct set <ctid> --mp0 <host-mount>,mp=<container-path>
```

Restart the container to activate the mount:

```bash
pct reboot <ctid>
# or
pct stop <ctid> && pct start <ctid>
```

---

## Step 4: ID Mapping to Preserve UID/GID 1000

> **Warning:** Modifying user namespace mappings on unprivileged containers can weaken isolation if misconfigured. Proceed only if you understand the implications and have a backup of your container config.

### 4.1 Allow the host to map UID/GID 1000

On the Proxmox host, edit both subordinate ID files:

```bash
nano /etc/subuid
nano /etc/subgid
```

Add or update the following entries. Comment out existing entries so they can be restored if needed:

```
# /etc/subuid
root:100000:65536
root:1000:1
```

```
# /etc/subgid
root:100000:65536
root:1000:1
```

### 4.2 Add custom ID maps to the container config

Edit `/etc/pve/lxc/<ctid>.conf` and add:

```
lxc.idmap: u 0 100000 1000
lxc.idmap: u 1000 1000 1
lxc.idmap: u 1001 101001 64535

lxc.idmap: g 0 100000 1000
lxc.idmap: g 1000 1000 1
lxc.idmap: g 1001 101001 64535
```

Restart the container:

```bash
pct restart <ctid>
```

---

## Step 5: Pre-Create Docker Bind Mount Directories

Before starting any Docker containers that mount paths from the NFS share, create the required directories manually. If Docker creates them at runtime, it will assign root-shifted ownership (`100000:100000`) rather than `1000:1000`, which causes permission failures.

Create the directory structure and set ownership before launching containers:

```bash
mkdir -p /mnt/nas/configs/sonarr
mkdir -p /mnt/nas/media
chown -R 1000:1000 /mnt/nas/configs
chown -R 1000:1000 /mnt/nas/media
```

Example Docker Compose volume configuration:

```yaml
volumes:
  - /mnt/nas/configs/sonarr:/config
  - /mnt/nas/media:/media
```

As long as the directories exist and are owned by UID/GID 1000 before Docker initializes them, permissions will remain consistent across the NFS share.

---

## Verification

Run the following inside the container to confirm ownership and write access:

```bash
id
stat -c "%u:%g %n" <container-path>
touch <container-path>/testfile && ls -l <container-path>/testfile
```

On the Proxmox host, confirm the files appear as `1000:1000`.

If ownership is incorrect, check for:

- Overlapping or missing entries in `/etc/subuid` and `/etc/subgid`
- Typos in the `lxc.idmap` lines in the container config
- NFS export settings applying `all_squash` or `root_squash` on the TrueNAS side
