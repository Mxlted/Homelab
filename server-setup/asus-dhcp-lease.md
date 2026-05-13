# ASUS Router DHCP Lease Persistence Guide

This guide explains how to make DHCP leases survive a reboot or short power loss on an ASUS router running ASUSWRT-Merlin.

By default, ASUS stores the active DHCP lease file in a temporary location. After a reboot or power cut, that temporary file can be lost. This can cause the router to rebuild its DHCP lease table from scratch, which may increase the chance of devices receiving IP addresses that were recently used by other devices.

This guide moves the DHCP lease file to `/jffs`, which survives reboots, and then links the normal lease file path back to that saved file.

## What This Does

This setup makes the DHCP lease file persistent across router reboots.

After setup:

```sh
/var/lib/misc/dnsmasq.leases
```

points to:

```sh
/jffs/dnsmasq.leases
```

That means when the router DHCP service writes lease updates, it writes them directly to the persistent file in `/jffs`.

## What This Does Not Do

This guide does not extend DHCP lease time.

It does not change leases to 30 days.

It does not replace proper DHCP reservations.

For devices that must always keep the same IP address, such as servers, NAS devices, printers, cameras, Proxmox, Home Assistant, or gaming PCs, you should still use DHCP reservations in the ASUS router.

## Requirements

- ASUS router running ASUSWRT-Merlin
- JFFS custom scripts enabled
- SSH access to the router

## Step 1: Enable JFFS Custom Scripts

Log into the ASUS router web interface.

Go to:

```text
Administration > System
```

Enable:

```text
Enable JFFS custom scripts and configs: Yes
```

Do not enable JFFS formatting unless you intentionally want to wipe the JFFS partition.

Leave this disabled unless needed:

```text
Format JFFS partition at next boot: No
```

Click **Apply**.

## Step 2: Enable SSH

Still under:

```text
Administration > System
```

Enable SSH:

```text
Enable SSH: LAN only
```

The default SSH port is usually:

```text
22
```

Click **Apply**.

## Step 3: SSH Into the Router

From your PC, open Terminal, PowerShell, or Windows Terminal.

Use your ASUS router IP address.

Example:

```sh
ssh admin@192.168.1.1
```

If your ASUS router uses a different LAN IP, use that instead.

Example:

```sh
ssh admin@192.168.1.254
```

Log in with your ASUS router username and password.

## Step 4: Create the Scripts Folder

Run:

```sh
mkdir -p /jffs/scripts
```

## Step 5: Copy the Current DHCP Lease File to JFFS

Run:

```sh
cp -fp /var/lib/misc/dnsmasq.leases /jffs/dnsmasq.leases
```

This saves the current DHCP lease table to persistent storage.

## Step 6: Create the dnsmasq.postconf Script

Run:

```sh
cat > /jffs/scripts/dnsmasq.postconf << 'EOF'
#!/bin/sh

ln -sf /jffs/dnsmasq.leases /var/lib/misc/dnsmasq.leases
EOF
```

This script runs when dnsmasq starts or reloads. It makes the normal lease file path point to the persistent lease file in `/jffs`.

## Step 7: Make the Script Executable

Run:

```sh
chmod a+rx /jffs/scripts/*
```

## Step 8: Restart dnsmasq

Run:

```sh
service restart_dnsmasq
```

## Step 9: Verify the Symlink

Run:

```sh
ls -l /var/lib/misc/dnsmasq.leases
```

You should see something similar to:

```text
/var/lib/misc/dnsmasq.leases -> /jffs/dnsmasq.leases
```

If you see that, the setup is working.

## Step 10: Verify the Persistent Lease File Exists

Run:

```sh
ls -l /jffs/dnsmasq.leases
```

You can also view the current leases:

```sh
cat /jffs/dnsmasq.leases
```

## Full Command Block

Use this if you want to run the full setup from SSH:

```sh
mkdir -p /jffs/scripts

cp -fp /var/lib/misc/dnsmasq.leases /jffs/dnsmasq.leases

cat > /jffs/scripts/dnsmasq.postconf << 'EOF'
#!/bin/sh

ln -sf /jffs/dnsmasq.leases /var/lib/misc/dnsmasq.leases
EOF

chmod a+rx /jffs/scripts/*

service restart_dnsmasq

ls -l /var/lib/misc/dnsmasq.leases
```

## How It Works After a Reboot or Power Loss

When the router reboots:

1. The temporary runtime directories are recreated.
2. dnsmasq starts.
3. ASUSWRT-Merlin runs `/jffs/scripts/dnsmasq.postconf`.
4. The script recreates the symlink.
5. dnsmasq writes lease updates to `/jffs/dnsmasq.leases`.

Because `/jffs/dnsmasq.leases` survives reboots, the DHCP lease table is preserved instead of starting empty.

## Do You Need a Cron Job?

No.

A cron job is not needed for this setup.

Once the symlink is active, `/jffs/dnsmasq.leases` becomes the live lease file. dnsmasq updates it automatically whenever DHCP leases are added, renewed, changed, or expired.

## How Often Are Leases Updated?

The lease file updates whenever dnsmasq changes the DHCP lease table.

Common reasons include:

- A device gets a new DHCP lease
- A device renews its current lease
- A lease expires
- A device changes hostname or client ID
- A device receives a different IP address

Most DHCP clients try to renew their lease at about 50% of the lease time.

For example, if the lease time is 24 hours, many devices try to renew around 12 hours.

## How to Check If the File Is Updating

Check the timestamp:

```sh
ls -l /jffs/dnsmasq.leases
```

View current leases:

```sh
cat /jffs/dnsmasq.leases
```

Watch the file live:

```sh
watch -n 5 'ls -l /jffs/dnsmasq.leases && cat /jffs/dnsmasq.leases'
```

You can force a client to renew by disconnecting and reconnecting Wi-Fi, or by running this on Windows:

```bat
ipconfig /release
ipconfig /renew
```

## Important Notes

This helps preserve DHCP leases after a short outage or reboot.

It does not guarantee that every device will always keep the same IP forever.

For important devices, use DHCP reservations.

Recommended approach:

- Use this guide to preserve the live DHCP lease table across reboots.
- Use DHCP reservations for devices that must never change IP addresses.
- Leave DHCP lease duration alone unless you specifically need longer leases.

## Undo Instructions

To remove this setup, SSH into the router and run:

```sh
rm -f /jffs/scripts/dnsmasq.postconf
rm -f /jffs/dnsmasq.leases
service restart_dnsmasq
```

Then verify:

```sh
ls -l /var/lib/misc/dnsmasq.leases
```

It should no longer point to `/jffs/dnsmasq.leases`.

## Reference

This guide is based on ASUSWRT-Merlin DHCP lease persistence behavior using `dnsmasq.postconf` and a persistent lease file stored in `/jffs`.
