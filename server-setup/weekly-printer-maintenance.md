Printer Setup and Weekly Test Page Automation

This guide covers configuring an Epson network printer on a Debian
server inside a Proxmox LXC container and scheduling a weekly automatic
test print to prevent inkjet clogging.

Install CUPS

Update package lists and install CUPS along with required components.

    sudo apt update
    sudo apt install cups cups-client cups-daemon

Enable and start the CUPS service.

    sudo systemctl enable --now cups

Add the Epson Network Printer

The printer is reachable at the network address 192.168.1.251. Configure
it using IPP Everywhere.

    sudo lpadmin -p EpsonNet   -E   -v ipp://192.168.1.251/ipp/print   -m everywhere

Enable the printer and allow it to accept print jobs.

    sudo cupsenable EpsonNet
    sudo cupsaccept EpsonNet

Test Print

Run a manual print to confirm printer functionality.

    /usr/bin/lp -d EpsonNet /root/cronmaster/print/Print_Test.pdf

Cron Job for Weekly Test Page

To prevent inkjet clogging, schedule a weekly print every Wednesday at
noon.

Edit the crontab:

    crontab -e

Add the following line:

    0 12 * * 3 /usr/bin/lp -d EpsonNet /root/cronmaster/print/Print_Test.pdf

This will automatically print the specified test page each Wednesday at
12:00.
