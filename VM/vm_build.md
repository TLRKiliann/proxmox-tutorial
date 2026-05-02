# VM installation & CMD

## Download ISO image (Debian)

Click on => local (pve) => ISO images

Downnload from URL

`https://debian.ethz.ch/debian-cd/current/amd64/iso-cd/debian-13.4.0-amd64-netinst.iso`

File name

`debian-13.4.0-amd64-netinst.iso`

## Create VM

- Click on pve (node)

- Click on Create VM

---

1) General
   
   Give a name to your VM

2) OS
    
    Choose the ISO image

3) System
    
    Choose Qemu agent
    TPM (yes but only for Windows)

4) Disks

    Select 32 GB
    Select discard (fstrim auto-cleaner for cache every week)

5) CPU
   
    1 socket
    2 cores (or 4)

6) Memory

    4096 MiB

7) Network

    Nothing

8)  Confirm


[⬆-up](#vm-installation-&-cmd)


## Start VM

Click on pve => Shell

`qm start <VM_ID>`

choose install (no graphical install)

**Partition**

- Entire disk

- All files in one partition

Scan extra installation media => No

HTTP proxy => let it empty

If you need one later you can access to file

`/etc/apt/apt.conf`

You don't need Debien desktop environment

NO GNOME, NO Xfce4, just standard system utilities & SSH

- After installation

- Qemu Agent

Into your VM

`sudo apt update`

`sudo apt install qemu-guest-agent -y`

`sudo systemctl enable qemu-guest-agent --now`


[⬆-up](#vm-installation-&-cmd)


## From pve node Shell


List all VM

`qm list`

Display status of VM

`qm status <VM_ID>`

Display config of a VM

`qm config <VM_ID>`

Start VM 

`qm start <VM_ID>`

Terminal

`qm terminal <VM_ID>`

Shutdown VM

`qm shutdown <VM_ID>`

Clone a VM

`qm clone <VM_ID_source> <new_VM_ID> --name <nom> --full`

Reboot VM

`qm reboot <VM_ID>`

Destroy VM

`qm destroy <VM_ID> --purge`

Snapshot

`qm snapshot <VM_ID> <snapshot_name>`

List

`qm listsnapshot <VM_ID>`

Restore

`qm rollback <VM_ID> <snapshot_name>`

Delete

`qm delsnapshot <VM_ID> <snapshot_name>`

Migration

`qm migrate <VM_ID> <node_aim>`

Change config

`qm set <VM_ID> --<option> <value>`

ex: qm set 100 --memory 2048

Unlock

`qm unlock <VM_ID>`

Stop (not recommanded, as a last resort) 

`qm stop <VM_ID>`


[⬆-up](#vm-installation-&-cmd)