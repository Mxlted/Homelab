# Printer Setup and Scheduled Test Page Automation

This guide covers configuring an Epson network printer on a Debian server inside a Proxmox LXC container and scheduling automatic test prints to prevent inkjet clogging.

---

## Install CUPS

Update package lists and install CUPS along with required components.

```bash
sudo apt update
sudo apt install cups cups-client cups-daemon
```

Enable and start the CUPS service.

```bash
sudo systemctl enable --now cups
```

---

## Add the Epson Network Printer

The printer is reachable at `192.168.1.251`. Configure it using IPP Everywhere.

```bash
sudo lpadmin -p EpsonNet \
  -E \
  -v ipp://192.168.1.251/ipp/print \
  -m everywhere
```

Enable the printer and allow it to accept print jobs.

```bash
sudo cupsenable EpsonNet
sudo cupsaccept EpsonNet
```

---

## Updating the Printer IP Address

If the printer is assigned a new IP address, update the existing CUPS queue to point to it. No need to delete and recreate the printer. Just re-run `lpadmin` with the new URI:

```bash
sudo lpadmin -p EpsonNet -v ipp://NEW_IP_HERE/ipp/print
```

For example, if the new address is `192.168.1.100`:

```bash
sudo lpadmin -p EpsonNet -v ipp://192.168.1.100/ipp/print
```

Verify the change took effect:

```bash
lpstat -v EpsonNet
```

The output should show the updated URI.

---

## Remove the Printer

To fully remove the printer from CUPS, delete the queue:

```bash
sudo lpadmin -x EpsonNet
```

Verify it has been removed:

```bash
lpstat -p
```

The printer should no longer appear in the list. Any pending jobs for that queue will also be discarded.

---

## Test Print

Run a manual print to confirm printer functionality.

```bash
/usr/bin/lp -d EpsonNet /root/cronmaster/print/Print_Test.pdf
```

---

## Cron Job for Scheduled Test Pages

To prevent inkjet clogging, schedule automatic test prints using one of the options below.

Open the crontab for editing:

```bash
crontab -e
```

### Option A: Weekly (every Wednesday at noon)

```cron
0 12 * * 3 /usr/bin/lp -d EpsonNet /root/cronmaster/print/Print_Test.pdf
```

### Option B: Bi-weekly (1st and 15th of each month at noon)

```cron
0 12 1,15 * * /usr/bin/lp -d EpsonNet /root/cronmaster/print/Print_Test.pdf
```

Choose one option and add it to the crontab. If switching from one schedule to the other, remove the old line first.
