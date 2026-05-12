# Rclone Configs Backup Guide

A simple Linux backup setup that compresses `/opt/configs` into a `.tar.gz` archive, uploads it to an rclone remote, and keeps only the newest 5 backups.

## What This Does

This guide creates scheduled backups for:

```text
/opt/configs
```

The backup is saved temporarily as:

```text
/tmp/configs-backup-YYYY-MM-DD.tar.gz
```

Then uploaded to:

```text
remote:configs-backup
```

After upload, older backups are removed so only the newest 5 backups remain.

## Requirements

You need:

- Linux system or container
- `rclone` installed and configured
- A working rclone remote named `remote`
- Access to `/opt/configs`
- `tar` installed

Check that your rclone remote works:

```bash
rclone lsd remote:
```

## Recommended Cron Job

Edit your crontab:

```bash
crontab -e
```

Add this line:

```cron
0 3 * * 1,3,5 BACKUP="/tmp/configs-backup-$(date +\%Y-\%m-\%d).tar.gz" && tar -czf "$BACKUP" -C /opt configs && rclone copy "$BACKUP" remote:configs-backup && rclone lsf remote:configs-backup --files-only --include "configs-backup-*.tar.gz" | sort -r | tail -n +6 | while read -r file; do rclone deletefile "remote:configs-backup/$file"; done && rm -f "$BACKUP"
```

This runs at 3:00 AM every Monday, Wednesday, and Friday.

## Manual Test Command With Progress

Run this manually first to make sure everything works:

```bash
BACKUP="/tmp/configs-backup-$(date +%Y-%m-%d).tar.gz" && tar -czf "$BACKUP" -C /opt configs && rclone copy "$BACKUP" remote:configs-backup --progress && rclone lsf remote:configs-backup --files-only --include "configs-backup-*.tar.gz" | sort -r | tail -n +6 | while read -r file; do rclone deletefile "remote:configs-backup/$file"; done && rm -f "$BACKUP"
```

The manual version includes:

```bash
--progress
```

so you can see the upload progress.

## How the Backup Naming Works

Backups are named by date:

```text
configs-backup-2026-05-12.tar.gz
configs-backup-2026-05-13.tar.gz
configs-backup-2026-05-15.tar.gz
```

The date format keeps filenames easy to sort.

## Keeping Only the Newest 5 Backups

This part lists matching backup files:

```bash
rclone lsf remote:configs-backup --files-only --include "configs-backup-*.tar.gz"
```

This sorts newest first:

```bash
sort -r
```

This selects everything after the newest 5:

```bash
tail -n +6
```

Then each older file is deleted:

```bash
while read -r file; do rclone deletefile "remote:configs-backup/$file"; done
```

With the Monday, Wednesday, and Friday schedule, 5 backups keeps about 1.5 to 2 weeks of backups.

## Restore a Backup

To download a backup:

```bash
rclone copy remote:configs-backup/configs-backup-YYYY-MM-DD.tar.gz /tmp --progress
```

Example:

```bash
rclone copy remote:configs-backup/configs-backup-2026-05-12.tar.gz /tmp --progress
```

Extract it:

```bash
tar -xzf /tmp/configs-backup-2026-05-12.tar.gz -C /tmp
```

This extracts a `configs` folder into `/tmp`:

```text
/tmp/configs
```

## Notes

- `tar.gz` is recommended for Linux config backups because it preserves permissions and symlinks better than zip.
- Cron requires percent signs in `date` commands to be escaped as `\%`.
- The manual command does not need escaped percent signs.
- The temporary local backup is removed after a successful upload and cleanup.
- This guide assumes your rclone remote is named `remote`. Change it if your remote has a different name.

## Verify Uploaded Backups

List uploaded backups:

```bash
rclone lsf remote:configs-backup --files-only --include "configs-backup-*.tar.gz"
```

Show more detail:

```bash
rclone ls remote:configs-backup --include "configs-backup-*.tar.gz"
```
