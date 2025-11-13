# Proxmox NFS Mount Setup for TrueNAS

```bash
# Edit the fstab file to configure persistent NFS mounts
nano /etc/fstab
```

```bash
# Add NFS mount entries for TrueNAS shares
192.168.1.245:/mnt/tank/Immich /mnt/immich-nas nfs4 rw,vers=4.1,noatime,_netdev,x-systemd.automount,nofail 0 0
192.168.1.245:/mnt/tank/Media  /mnt/media-nas  nfs4 rw,vers=4.1,noatime,_netdev,x-systemd.automount,nofail 0 0
```

```bash
# Mount all entries in fstab
mount -a
```

```bash
# Reload systemd to apply changes
systemctl daemon-reload
```

Check Mounts
```bash
# Confirm NFS or CIFS mounts are active
mount | grep -E "nfs|cifs"
```

```bash
# Unmount the NFS shares if needed
umount /mnt/media-nas /mnt/immich-nas
```
