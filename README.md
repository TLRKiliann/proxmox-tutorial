# Tutorial Proxmox VE

## Installation

I've choose Ventoy to flash my USB key with Proxmox ISO (last one).

If you get some trouble with installation during installation, you can access to GRUB by pressing key "e" to add nomodeset at the end of linux line.

(options -> target disk)

- RAID0 => ZFS (I've got only one Hard Disk)
- RAID1 => ZFS (2 Hard Disks)

---

## Access to Proxmox via Browser:

`https://192.168.xx.xx:8006`

---

## To verify from Proxmox Shell:

1. Hard Disk:

`$ lsblk`

`$ df -h /`

2. Voir toutes les interfaces réseau

`ip link show`

3. Display network bridges

`brctl show`

4. Voir la configuration réseau de Proxmox

`cat /etc/network/interfaces`

5. Lister tous les stockages configurés

`pvesm status`

---

## Create CT with LXC :

- install template => debian 12 (bookworm)

- Click pve -> Create CT -> configuration

- CT 100 - pwd (& confirmed) - 1 CPU - 8 - 1024 RAM - 512 SWAP - DNS empty

## Verify from pve node Shell:

`pct stop 100`

`pct start 100`

`pct list`

`pct enter 100`

## Update CT:

`pct enter 100`

`pct exec 100 -- apt update`

`pct exec 100 -- bash -c "apt update && apt upgrade -y"`

`pct exec 100 -- bash -c "apt update && apt upgrade -y && apt autoremove -y"`

⚠️ Don't use dist-upgrade into a CT ⚠️

`pct reboot 100`

---

## Clone

Le conteneur doit être arrêté pour un clone cohérent (sinon le clone peut être dans un état inconsistants)

`pct clone 101 102`

Si le conteneur doit être utilisé en production de manière indépendante : préférez pct clone --full 1 pour éviter toute dépendance avec l'original

`pct clone 101 102 --full 1`

`pct clone 101 102 --hostname mon-nouveau-conteneur`

---

## UFW firewall installation

```
apt install ufw -y 

ufw default deny incoming
ufw default allow outgoing

ufw allow 22/tcp   # SSH
ufw allow 80/tcp   # HTTP
ufw allow 443/tcp  # HTTPS

ufw enable
```

---

## SSH

CT are only accessible by SSH connection.

Your terminal
    │
    ├── $ pct enter 100 ──► DANS le conteneur
    │       │
    │       ├── nano /etc/ssh/sshd_config
    │       ├── systemctl restart sshd
    │       └── exit
    │
    └── $ pct push 100 ... ─► From Host Proxmox

1) Toujours dans le conteneur (pct enter 100)

`apt update`

`apt install openssh-server -y`

- Set SSH config at first and change root after !

`ssh-keygen -t ed25519 -C "votre-email@exemple.com"`

- From pve node shell :

`pct push 100 ~/.ssh/id_ed25519.pub /root/.ssh/authorized_keys`

If failed:

`pct enter 100`

`mkdir -p /root/.ssh`

`chmod 700 /root/.ssh`

If it's ok:

`pct enter 100`

`nano /etc/ssh/sshd_config`

(into pct 100)

```
PermitRootLogin prohibit-password
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
```

`cat /root/.ssh/id_ed25519.pub >> /root/.ssh/authorized_keys`

`systemctl restart sshd`

`systemctl enable sshd.service`

`systemctl start sshd`

`systemctl status sshd`

2) Sortez du conteneur (Ctrl+D ou "exit")

Puis depuis l'hôte Proxmox :

not sur `pct push 100 ~/.ssh/id_ed25519.pub /root/.ssh/authorized_keys`

ok `pct push 100 /root/.ssh/id_rsa.pub /root/.ssh/authorized_keys`

- Depuis votre machine personnelle ou l'hôte Proxmox

`ssh root@IP_DU_CONTENEUR_100`

- Depuis votre serveur Proxmox (pas depuis la session SSH)

`ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`


3) En "jump host" depuis votre machine personnelle (la plus élégante)

Configurez votre fichier ~/.ssh/config sur votre machine personnelle :

`ssh-config`

```
Host proxmox-host
    HostName IP_DU_SERVEUR_PROXMOX
    User root

Host lxc-100
    HostName IP_DU_CONTENEUR_100
    User admin
    ProxyJump proxmox-host
```

Ensuite, depuis votre machine personnelle :
bash

`ssh lxc-100`

Vous passez automatiquement par l'hôte Proxmox pour atteindre le conteneur.

Problème fréquent : "Connection refused"

Si SSH refuse la connexion, vérifiez depuis l'intérieur du conteneur :
bash

`pct enter 100`

1. SSH est-il installé ?

systemctl status sshd

2. SSH écoute-t-il sur le bon port ?

netstat -tlnp | grep :22

OR

ss -tlnp | grep sshd

3. Le pare-feu bloque-t-il ?

iptables -L -n | grep :22

4. Vérifier la configuration SSH

`grep -E "PermitRootLogin|PasswordAuthentication|PubkeyAuthentication" /etc/ssh/sshd_config`

L'ordre logique des accès à un conteneur
text

```
Votre machine personnelle
    │
    ├─ (1) SSH direct → Conteneur (si réseau accessible)
    │
    ├─ (2) SSH → Hôte Proxmox → (ssh) → Conteneur (jump host)
    │
    └─ (3) Accès console direct depuis l'interface Web Proxmox
                    │
                    └─ pct enter 100 (accès root absolu)

```
---

## Add user ✅

`pct enter 100`

1) Créer un utilisateur (ex: monuser)

`adduser monuser`

2) Ajouter aux sudoers

`usermod -aG sudo monuser  # Pour Debian/Ubuntu`

or

`usermod -aG wheel monuser  # Pour certaines distributions`


3) From pve node shell:

`pct push 100 /root/.ssh/id_rsa.pub /home/monuser/.ssh/authorized_keys`

Ou depuis l'intérieur du CT

`pct enter 100`

`mkdir -p /home/monuser/.ssh`

`cat /root/.ssh/authorized_keys >> /home/monuser/.ssh/authorized_keys`

`chown -R monuser:monuser /home/monuser/.ssh`

`chmod 600 /home/monuser/.ssh/authorized_keys`

`ssh monuser@192.168.XX.XX`

Deactivate ssh root:

`nano /etc/ssh/sshd_config`

```
PermitRootLogin no
```

`systemctl restart ssh`