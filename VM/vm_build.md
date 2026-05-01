# VM installation & CMD

1. Download ISO image (Debian)

Click on => local (pve) => ISO images

Downnload from URL

`https://debian.ethz.ch/debian-cd/current/amd64/iso-cd/debian-13.4.0-amd64-netinst.iso`

File name

`debian-13.4.0-amd64-netinst.iso`

2. Create VM

Click on pve (node)

Click on Create VM

---

1) General
   give a name to your VM

2) OS
    choose the ISO image

3) System
    choose Qemu agent
    TPM (yes but only for Windows)

4) Disks
    choose 32 GB
    choose discard (fstrim auto-cleaner for cache every week)

5) CPU
    choose 1 socket
    choose 2 cores (or 4)

6) Memory
    4096 MiB

7) Network
    Nothing

8)  Confirm

9) Start VM

Click on pve => Shell

`qm start <VM_ID>`

choose install (no graphical mode)

- Entire disk

All files in one partition

Scan extra installation media => No

HTTP proxy => let it empty

if you need one later you can access to file:

`/etc/apt/apt.conf`

You don't need Debien desktop environment

NO GNOME, NO Xfce4, just standard system utilities & SSH

- After installation

- Qemu Agent

`sudo apt update`

`sudo apt install qemu-guest-agent -y`

`sudo systemctl enable qemu-guest-agent --now`

Start VM

`qm start <VM_ID>`

Shutdown VM

`qm shutdown <VM_ID>`