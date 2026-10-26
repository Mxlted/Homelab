# Move a TrueNAS VM ZFS Pool Back to the Proxmox Host

> Step-by-step guide to returning an existing TrueNAS VM ZFS pool to the Proxmox host **without recreating the pool or copying the data**, then replacing TrueNAS NFS mounts with local ZFS access for LXC and Docker.

![Proxmox](https://img.shields.io/badge/Proxmox-VE-orange?logo=proxmox)
![TrueNAS](https://img.shields.io/badge/TrueNAS-SCALE-blue?logo=truenas)
![ZFS](https://img.shields.io/badge/Storage-OpenZFS-3584e4)
![License](https://img.shields.io/badge/license-MIT-green)

> [!CAUTION]
> This procedure changes which operating system owns the disks. **Back up irreplaceable data first.** Never import the same writable ZFS pool into TrueNAS and Proxmox simultaneously. Do not select **Delete Pool**, **Destroy data**, initialize disks, run `zpool create`, or use `wipefs`.

---

## Table of Contents

- [Overview](#overview)
- [Original Setup (Before Migration)](#original-setup-before-migration)
- [Verified Host Configuration After Migration](#verified-host-configuration-after-migration)
- [Requirements and Preflight](#requirements-and-preflight)
- [Step 1: Record the Current Pool and VM Configuration](#step-1-record-the-current-pool-and-vm-configuration)
- [Step 2: Stop Services Using TrueNAS Storage](#step-2-stop-services-using-truenas-storage)
- [Step 3: Export the Pool from TrueNAS](#step-3-export-the-pool-from-truenas)
- [Step 4: Return the Disks to the Proxmox Host](#step-4-return-the-disks-to-the-proxmox-host)
- [Step 5: Import the Existing Pool on Proxmox](#step-5-import-the-existing-pool-on-proxmox)
- [Step 6: Mount and Verify the ZFS Datasets](#step-6-mount-and-verify-the-zfs-datasets)
- [Step 7: Replace TrueNAS NFS Mounts](#step-7-replace-truenas-nfs-mounts)
- [Step 8: Restart LXC and Docker Workloads](#step-8-restart-lxc-and-docker-workloads)
- [Step 9: Verify Persistence After Reboot](#step-9-verify-persistence-after-reboot)
- [Optional: Add ZFS Storage to Proxmox](#optional-add-zfs-storage-to-proxmox)
- [Troubleshooting](#troubleshooting)
- [Rollback to TrueNAS](#rollback-to-truenas)
- [References](#references)

---

## Overview

Originally, the TrueNAS VM owned the storage drives and exposed datasets to Proxmox over NFS. This guide documents moving **ownership of the existing ZFS pool** back to the Proxmox host. ZFS metadata, datasets, snapshots, and the existing RAIDZ layout stay on the original disks.

The process is:

1. Stop everything using the TrueNAS shares.
2. Cleanly **export** the ZFS pool from TrueNAS.
3. Shut down TrueNAS and remove only the relevant disk/controller passthrough.
4. Import that **same pool** on Proxmox.
5. Mount the datasets and switch LXC/Docker away from NFS.
6. Test access and boot persistence before bringing the services back online.

No `zpool create` or ZFS replication is required when Proxmox can access the original physical drives directly. This is **not** a guide for converting a virtual-disk-backed pool into a physical-disk pool.

---

## Original Setup (Before Migration)

Based on this repository's existing [TrueNAS → Proxmox → LXC → Docker guide](./truenas-nfs-proxmox-lxc-docker.md) and [NFS mount notes](./current-proxmox-nfs-mount.md):

| Component | Current configuration |
|-----------|-----------------------|
| **Host** | Proxmox VE |
| **NAS** | TrueNAS SCALE running as a VM |
| **Main ZFS pool** | `tank` |
| **Main pool disks** | 4 × WD Red Plus 8 TB HDDs passed through to TrueNAS |
| **Main datasets** | `tank/Media` and `tank/Immich` |
| **Existing NFS server** | `192.168.1.245` |
| **Existing Proxmox paths** | `/mnt/media-nas` and `/mnt/immich-nas` |
| **Other pool** | `ssd` with the `ssd/data` dataset and `/mnt/ssd` NFS mount |
| **Applications** | Docker in an unprivileged Proxmox LXC |

**Scope:** The worked example migrates `tank`. The original TrueNAS setup also referenced an `ssd` pool, but it is **not part of the `tank` migration**. Never remove an unrelated pool's NFS mount or passthrough without checking its actual configuration.

**Expected result:** Proxmox directly owns `tank`, while existing LXC/Docker paths can be preserved. TrueNAS-specific NFS/SMB share definitions, user accounts, and service settings are **not** transferred by a ZFS pool import.

---

## Verified Host Configuration After Migration

The Proxmox host shell output from **October 26, 2026** confirmed the following state (the original setup table above is retained as historical context):

| Component | Verified details |
|-----------|------------------|
| **ZFS pool** | `tank`, imported and `ONLINE` directly on Proxmox |
| **Vdev layout** | Four WD Red Plus 8 TB HDDs in a single RAIDZ1 vdev |
| **Pool size** | 29.1 TiB raw, with 9.50 TiB allocated and 19.6 TiB raw free at inspection |
| **Datasets** | `tank/Media` and `tank/Immich` |
| **Actual host mountpoints** | `/tank/Media` and `/tank/Immich` (not `/mnt/tank/...`) |
| **Last scrub** | October 11, 2026 — 0 bytes repaired and 0 errors |
| **Other imported ZFS pools** | None listed; the old `ssd` pool was not present |
| **Proxmox-managed VM disks** | NVMe-backed LVM-thin storage (`local-lvm`); `tank` need not be registered with `pvesm` to be used by LXC bind mounts |

These verified values describe the **completed migration**, not what the historical TrueNAS VM originally mounted. Before reusing the procedural commands on a different host, always recheck live ZFS mountpoints and disk membership.

---

## Requirements and Preflight

- Shell/root access to Proxmox and administrator access to TrueNAS.
- A separate backup of important files and any encryption keys/passphrases.
- Console/IPMI access to Proxmox if reclaiming an entire PCIe HBA requires a reboot.
- The TrueNAS OS/boot disk **must not** be one of the disks being migrated.
- All pool members must be visible to Proxmox when passthrough is removed.
- Proxmox OpenZFS must support the pool's enabled feature flags. **Do not run `zpool upgrade`** as part of the migration.
- No required VM disks, LXC root disks, or Proxmox backup targets should depend on the NFS share during the outage.

> [!IMPORTANT]
> A ZFS snapshot is **not** a substitute for an independent backup. If the pool contains encrypted datasets, export/back up the keys before disconnecting TrueNAS. A Proxmox import alone does not unlock encrypted datasets.

### Back up the Proxmox configuration

On the **Proxmox host**, first identify the TrueNAS VM ID:

```bash
qm list
qm config YOUR_TRUENAS_VMID
```

Replace `YOUR_TRUENAS_VMID` with the actual numeric VM ID before running commands. Record which `hostpciN` or `scsiN` entries correspond **only** to the TrueNAS data disks.

Back up the local configuration files:

```bash
cp -a /etc/fstab /root/fstab.before-truenas-migration
cp -a /etc/pve/storage.cfg /root/storage.cfg.before-truenas-migration
cp -a /etc/pve/qemu-server/YOUR_TRUENAS_VMID.conf /root/truenas-vm.conf.before-migration
```

Save the TrueNAS system configuration from its web UI as well. Record storage shares, users, datasets, and any encryption keys. The VM can be kept powered off afterward for rollback.

---

## Step 1: Record the Current Pool and VM Configuration

In the **TrueNAS shell**, inspect the pool before changing anything:

```bash
zpool status -v tank
zfs list -r tank
zfs get -r mountpoint,canmount,encryption,keylocation tank
```

Confirm that `tank` is healthy, record its disk serials, and note its ZFS mountpoints. If the pool is degraded, missing a member, or actively resilvering, resolve that first.

On the **Proxmox host**, record disk and storage information:

```bash
qm config YOUR_TRUENAS_VMID
lsblk -o NAME,SIZE,MODEL,SERIAL,TYPE,FSTYPE,MOUNTPOINTS
ls -l /dev/disk/by-id/
zpool status
pvesm status
```

Check which containers and host mounts depend on TrueNAS:

```bash
grep -nE '192[.]168[.]1[.]245|truenas|/mnt/media-nas|/mnt/immich-nas|/mnt/ssd' /etc/fstab /etc/pve/storage.cfg
pct config YOUR_DOCKER_CTID
findmnt -M /mnt/media-nas
findmnt -M /mnt/immich-nas
```

The `tank` pool should **not** already be imported on Proxmox while TrueNAS is actively using it.

---

## Step 2: Stop Services Using TrueNAS Storage

Stop Docker applications that read/write the storage (for example qBittorrent, Sonarr, Radarr, and Immich), then stop their LXC containers:

```bash
pct shutdown YOUR_DOCKER_CTID
pct status YOUR_DOCKER_CTID
```

Repeat for other dependent containers/VMs. Confirm that no process is writing to the NFS shares and no Proxmox VM/CT disks depend on the NFS storage.

Disable automatic startup for dependent guests **temporarily** in Proxmox if necessary. If `truenas-nfs` (or another NFS storage ID) exists under **Datacenter → Storage**, disable its use during migration; **do not remove storage entries until you have checked which workloads reference them**.

If the old NFS mounts are idle, unmount them from Proxmox while TrueNAS is still online:

```bash
umount /mnt/media-nas
umount /mnt/immich-nas
```

If a mount is busy, use `findmnt`, `fuser -vm /mnt/media-nas`, and `fuser -vm /mnt/immich-nas` to locate users. Do not use force/lazy unmount as a substitute for stopping applications. The existing `x-systemd.automount` units may need to be stopped or will need to be cleared at reboot.

**Do not unmount `/mnt/ssd` yet** unless you are also migrating the `ssd` pool.

---

## Step 3: Export the Pool from TrueNAS

In the **TrueNAS web UI**, go to **Storage**, locate `tank`, and choose **Export/Disconnect**, **Disconnect**, or **Export Pool** (the exact wording depends on TrueNAS version).

1. Review the services and shares that depend on the pool.
2. Choose **Export Pool** / **Export/Disconnect**, **not Delete Pool**.
3. **Never select "Destroy data on this pool"** or any erase/destroy option.
4. Review whether TrueNAS will remove saved share configurations. Preserve/back up the configuration if you want an easier rollback.
5. Confirm the **non-destructive export**, then wait for it to finish.

A successful export makes the pool unavailable to TrueNAS without erasing its ZFS data. Confirm it is no longer listed as an imported pool.

Now **shut down the TrueNAS VM completely**. Do not let it auto-start with the old passthrough configuration.

> [!WARNING]
> If export fails, **stop here** and resolve the dependency or pool issue. Do not force-import a pool into Proxmox while TrueNAS could still be using the same disks.

---

## Step 4: Return the Disks to the Proxmox Host

In **Proxmox → TrueNAS VM → Hardware**, inspect how the pool's devices were passed through.

### Option A: Individual physical HDD passthrough

If the VM has physical disks mapped as `scsiN` / `sataN` / `virtioN` entries pointing to `/dev/disk/by-id/...`:

1. Verify every device is a **data disk belonging to `tank`**.
2. With TrueNAS **stopped**, remove those physical disk assignments **from the VM configuration only**.
3. Keep the TrueNAS boot/OS virtual disk and unrelated `ssd` devices untouched.
4. Do **not** choose an option that deletes a backing storage volume or destroys a disk.

### Option B: Entire HBA/SATA controller passthrough

If TrueNAS owns an entire controller through a `hostpciN` device:

1. With TrueNAS **stopped**, remove that `hostpciN` device from the VM hardware configuration.
2. Ensure the physical controller is bound to its **normal host storage driver**, not `vfio-pci`.
3. If host VFIO binding or a driver blacklist still claims the controller, review/reverse only the relevant passthrough configuration and reboot the host if required.

Check controller ownership:

```bash
lspci -nnk
lsblk -o NAME,SIZE,MODEL,SERIAL,TYPE,FSTYPE,MOUNTPOINTS
ls -l /dev/disk/by-id/
```

**Before continuing:** Confirm all four `tank` member disks appear on Proxmox, with the expected sizes/serials, and that none is the Proxmox boot disk.

> If the pool was built on virtual qcow2/raw **disk images**, instead of the original physical drives, this direct-import method may not apply. Do not guess the image's underlying block-device offset or import a mounted virtual image.

---

## Step 5: Import the Existing Pool on Proxmox

On the **Proxmox host**, list available ZFS pools without importing:

```bash
zpool import -d /dev/disk/by-id
```

Inspect the reported name, ID, vdev layout, and health. Confirm `tank` is listed and all expected drives are present. If `tank` is missing, **stop** and troubleshoot disk visibility instead of creating a new pool.

Import the existing pool **without mounting datasets yet**:

```bash
zpool import -N -d /dev/disk/by-id tank
```

Check the result:

```bash
zpool status -v tank
zfs list -r tank
zpool get cachefile tank
```

**Do not add `-f` unless you have conclusively verified that no other system has the pool imported and understand why ZFS reports it as active elsewhere.** Do not use `-F`, `-X`, `-m`, or other recovery flags as routine migration steps.

If Proxmox says the pool uses unsupported ZFS features, **stop**. Compare the host's OpenZFS version with TrueNAS rather than trying random import/recovery flags. Also avoid upgrading feature flags after importing; that can prevent a rollback to TrueNAS.

---

## Step 6: Mount and Verify the ZFS Datasets

Check the original mountpoint properties **before** mounting anything:

```bash
zfs get -r -o name,property,value mountpoint,canmount,keystatus tank
zfs list -r -o name,mountpoint,mounted tank
```

On this migrated Proxmox host, the verified ZFS mountpoints are `/tank/Media` and `/tank/Immich`. **Do not assume** these will be correct on another host; use the values shown by `zfs get`.

Check whether any intended target directories are already mounted or contain unrelated files. When mountpoints are correct and there are no conflicts:

```bash
zfs mount -a
zfs list -r -o name,mountpoint,mounted tank
zfs mount
```

`zfs mount -a` attempts to mount eligible datasets (including those from other imported pools). Encrypted datasets must have their keys loaded first; datasets with `canmount=noauto` or `mountpoint=legacy` require separate handling. Never overwrite non-empty host paths to make mounting succeed.

If the properties remain as in the old TrueNAS deployment, verify the datasets:

```bash
findmnt -M /tank/Media
findmnt -M /tank/Immich
ls -la /tank/Media
ls -la /tank/Immich
```

If either mount is absent, do not start applications or write data into that directory. Resolve the mountpoint, key, or ZFS issue first.

**Permissions:** Direct ZFS mounts are not NFS mounts. Existing numeric Unix UID/GID ownership is retained, but the old TrueNAS NFS `mapall` behavior no longer applies. Check Docker/LXC UID mappings and ACL behavior before changing permissions. Avoid broad `chown -R` commands on existing datasets.

---

## Step 7: Replace TrueNAS NFS Mounts

Choose **one** of these approaches. The first is useful when existing applications already reference `/mnt/media-nas` and `/mnt/immich-nas`.

### Option A: Preserve existing host paths using local bind mounts

Back up and edit `/etc/fstab`:

```bash
cp -a /etc/fstab /root/fstab.before-local-zfs-binds
nano /etc/fstab
```

**Remove or comment out only** the old `tank` NFS lines:

```text
# 192.168.1.245:/mnt/tank/Immich /mnt/immich-nas nfs4 ...
# 192.168.1.245:/mnt/tank/Media  /mnt/media-nas  nfs4 ...
```

**If** `zfs get` confirmed the local source paths, replace those lines with:

```fstab
/tank/Immich /mnt/immich-nas none bind,x-systemd.requires-mounts-for=/tank/Immich 0 0
/tank/Media  /mnt/media-nas  none bind,x-systemd.requires-mounts-for=/tank/Media  0 0
```

If an unrelated `ssd/data` NFS line still exists, leave it alone until its role is verified (the inspected Proxmox host does not currently list an imported `ssd` ZFS pool). Ensure the destination directories exist:

```bash
mkdir -p /mnt/immich-nas /mnt/media-nas
systemctl daemon-reload
```

Stop any remaining old NFS automount units for these paths; use `systemctl list-units --type=automount --all` to identify the exact names. Verify the old NFS mounts are gone, then mount the new entries:

```bash
findmnt -M /mnt/immich-nas
findmnt -M /mnt/media-nas
mount /mnt/immich-nas
mount /mnt/media-nas
findmnt -M /mnt/immich-nas -o SOURCE,FSTYPE,TARGET
findmnt -M /mnt/media-nas -o SOURCE,FSTYPE,TARGET
```

Both destinations must resolve to the **local ZFS datasets via bind mounts**, not `192.168.1.245` via NFS. If `mount` says the target is already mounted or systemd retains an old automount, resolve it before proceeding; a clean reboot with dependent guests disabled may be needed.

> [!IMPORTANT]
> A plain directory on the host is **not proof that ZFS is mounted**. Always confirm the ZFS source with `findmnt -M` before allowing Docker to write to it. The `x-systemd.requires-mounts-for` option supplies systemd mount dependencies, but it does not replace verification of ZFS import, dataset mounts, and startup order.

### Option B: Point LXC directly at ZFS dataset paths

Instead of preserving the old host-side `/mnt/*-nas` paths, update the LXC bind mounts themselves after confirming ZFS is mounted.

For example, to keep the container's `/mnt/nas` location unchanged:

```bash
pct config YOUR_DOCKER_CTID
pct set YOUR_DOCKER_CTID --mp0 /tank/Media,mp=/mnt/nas
```

Only replace `mp0` if that is the **existing** mount index for your media data; use the correct index otherwise. Change other mounts individually as needed. This avoids extra host-side bind mounts but may require updating host applications, Docker Compose paths, and additional containers.

---

## Step 8: Restart LXC and Docker Workloads

First confirm the required datasets and host/container mount sources really exist:

```bash
zpool status -v tank
zfs list -r -o name,mountpoint,mounted tank
findmnt -M /mnt/media-nas
findmnt -M /mnt/immich-nas
pct config YOUR_DOCKER_CTID
```

The `findmnt` checks for the old host paths apply when using **Option A**. With Option B, check the direct ZFS dataset paths instead.

Start a dependent LXC only after its source mounts have been verified:

```bash
pct start YOUR_DOCKER_CTID
pct exec YOUR_DOCKER_CTID -- mount
pct exec YOUR_DOCKER_CTID -- id
```

Check an existing file's metadata, read a small known file, and test writing **only in a designated test directory** where you have confirmed correct ownership. Validate Docker Compose binds and application access before re-enabling all services.

If the unprivileged LXC uses the repository's custom UID/GID 1000 mapping, inspect `/etc/pve/lxc/YOUR_DOCKER_CTID.conf` and permissions; **do not assume** the old NFS `mapall` still translates ownership.

---

## Step 9: Verify Persistence After Reboot

Proxmox usually uses its ZFS import services and pool cache for automatic import at boot. Check that the imported pool is recorded persistently:

```bash
zpool get cachefile tank
ls -l /etc/zfs/zpool.cache
systemctl status zfs-import-cache.service zfs-import-scan.service zfs-mount.service
```

Service availability and which import service is enabled can differ by Proxmox installation. If `cachefile=none` or automatic import is not configured, resolve that **before** expecting unattended startup. Do not blindly enable competing import methods.

With dependent LXCs/VMs set **not to auto-start**, perform a planned host reboot. Then verify **before** restarting workloads:

```bash
zpool status -v tank
zfs list -r -o name,mountpoint,mounted tank
findmnt -M /tank/Media
findmnt -M /tank/Immich
findmnt -M /mnt/media-nas
findmnt -M /mnt/immich-nas
pvesm status
```

The final two `findmnt` checks apply to Option A only. Once all required mounts are correct and the data is accessible, restore container auto-start behavior and normal Docker services.

**Success criteria:** Proxmox imports `tank` on boot, `tank/Media` and `tank/Immich` mount from local disks, LXC can access the expected files with the right permissions, and no `tank` NFS dependency remains.

---

## Optional: Add ZFS Storage to Proxmox

**Not required** just to expose existing datasets to LXC/Docker. Proxmox's **ZFS storage backend** is primarily for hosting VM disks and LXC root volumes, not for turning arbitrary media files into VM storage.

If you separately want a dedicated new dataset for Proxmox-managed VM/LXC storage, you can create one **after the migration is fully verified**:

```bash
zfs create tank/pve-vmdata
pvesm add zfspool tank-vmdata --pool tank/pve-vmdata --content images,rootdir --sparse 1
pvesm status
```

Do **not** point multiple Proxmox storage IDs at the same underlying VM/LXC volume set. Do not use the `tank/Media` or `tank/Immich` datasets for Proxmox VM disks unless you intentionally design and manage them that way.

---

## Troubleshooting

| Problem | What to check |
|---------|---------------|
| `tank` not listed by `zpool import` | Verify all disk serials, passed-through devices, controller driver, `/dev/disk/by-id`, and any missing vdev member. |
| Pool reports it is in use elsewhere | Confirm the TrueNAS VM is fully stopped and the pool was exported; do **not** blindly force-import. |
| Unsupported ZFS features | Compare installed Proxmox/OpenZFS and TrueNAS versions. Don't upgrade the pool or use destructive recovery flags. |
| Imported pool, empty `/tank/...` directory | Check `zfs get mountpoint,canmount,keystatus` and `zfs mount`; don't write into an unmounted directory. |
| Encrypted files are unavailable | Restore/load the appropriate ZFS encryption keys; importing the pool does not unlock them. |
| LXC/Docker sees permission denied | Check `stat`, `id`, LXC idmaps, and existing ACL type; NFS `mapall` is gone. |
| Old NFS mount keeps returning | Inspect `/etc/fstab`, active systemd automount units, and Proxmox Datacenter storage entries. |
| Pool works now but not after reboot | Inspect ZFS import/mount services, `zpool get cachefile`, and host startup ordering. |
| Old `ssd/data` NFS reference remains | This is a separate historical configuration; inspect current mounts and storage before changing it. |

**Never troubleshoot import problems by erasing labels, rebuilding RAIDZ, creating a replacement pool with the same name, or applying irreversible recovery flags without an independent backup and a verified recovery plan.**

---

## Rollback to TrueNAS

If host-side use fails and you want TrueNAS to own the pool again:

1. Stop all Proxmox containers/VMs and host processes using `tank`.
2. Unmount local bind mounts that depend on `tank`.
3. Export the host pool cleanly with `zpool export tank` **only after all consumers are stopped**.
4. Remove host-side storage/mount configurations that would try to access the exported pool.
5. Reattach the physical disks or HBA to the **powered-off** TrueNAS VM, restoring its original passthrough mapping.
6. Start TrueNAS, import the **existing** pool in TrueNAS, and verify data/health.
7. Restore TrueNAS shares/configuration as needed, then restore the NFS mounts and LXC dependencies.

Keep the two systems mutually exclusive: when TrueNAS regains ownership, Proxmox must not import that same pool locally.

---

## References

- [TrueNAS: Managing Pools (export versus delete)](https://www.truenas.com/docs/scale/storage/pools/managepools/)
- [OpenZFS: zpool-import](https://openzfs.github.io/openzfs-docs/man/master/8/zpool-import.8.html)
- [Proxmox: PCI(e) Passthrough](https://github.com/proxmox/pve-docs/blob/master/qm-pci-passthrough.adoc)
- [Proxmox: ZFS Pool Storage Backend](https://github.com/proxmox/pve-docs/blob/master/pve-storage-zfspool.adoc)
- [Proxmox: Storage Manager](https://github.com/proxmox/pve-docs/blob/master/pvesm.adoc)
