# Tutorial Proxmox VE

[Installation](#installation)
[Access to Proxmox via Browser:](#access-to-proxmox-via-browser-)
[To verify from Proxmox Shell:](#to-verify-from-proxmox-shell-)
[Create CT with LXC:](#create-ct-with-lxc-)
[Verify from pve node Shell:](#verify-from-pve-node-shell-)
[Update CT:](#update-ct-)
[Clone CT](#clone-ct)
[UFW firewall installation](#ufw-firewall-installation)
[SSH](#ssh)
[Add user ✅](#add-user-)
[hot standby for website](#hot-standby-for-website)
[Pour aller plus loin : surveiller votre site web](#pour-aller-plus-loin--surveiller-votre-site-web)

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

[UP](#tutorial-proxmox-ve)

---

## Create CT with LXC:

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

[UP](#tutorial-proxmox-ve)

---

## Clone CT

Le conteneur doit être arrêté pour un clone cohérent (sinon le clone peut être dans un état inconsistants)

`pct clone 101 102`

Si le conteneur doit être utilisé en production de manière indépendante : préférez pct clone --full 1 pour éviter toute dépendance avec l'original

`pct clone 101 102 --full 1`

`pct clone 101 102 --hostname mon-nouveau-conteneur`

[UP](#tutorial-proxmox-ve)

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

[UP](#tutorial-proxmox-ve)

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

[UP](#tutorial-proxmox-ve)

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

[UP](#tutorial-proxmox-ve)

---

##  hot standby for website

No preemptive mode (website).

Le remplacement à chaud (hot standby ou hot spare) désigne un scénario où :

- Le serveur de secours est déjà allumé et prêt à prendre le relais immédiatement

- Mais il ne reçoit pas de trafic en temps normal

- Il peut avoir ses propres données synchronisées ou non

Keepalived en mode non-préemptif correspond exactement à cette définition : le CT 101 tourne en arrière-plan, prêt à prendre la main, mais ne fait rien tant que le CT 100 fonctionne.

1) Installer Keepalived sur les deux CT

`apt update`

`apt install keepalived -y`

2) Configurer le CT 100 (MASTER)

Créez le fichier `/etc/keepalived/keepalived.conf` :

```
vrrp_instance VI_1 {
    state MASTER              # Rôle principal
    interface eth0            # Votre interface réseau
    virtual_router_id 51      # Identifiant unique (1-255)
    priority 100              # Priorité plus haute que le BACKUP
    advert_int 1              # Annonce toutes les secondes

    authentication {
        auth_type PASS
        auth_pass MonMotDePasse  # À changer !
    }

    virtual_ipaddress {
        192.168.0.99/24 dev eth0   # Votre VIP
    }
}
```

3) Configurer le CT 101 (BACKUP)

```
vrrp_instance VI_1 {
    state BACKUP             # Rôle de secours
    interface eth0
    virtual_router_id 51     # ID IDENTIQUE au MASTER !
    priority 90              # Plus bas que le MASTER
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass MonMotDePasse   # IDENTIQUE au MASTER !
    }

    virtual_ipaddress {
        192.168.0.99/24 dev eth0
    }
}
```

Points critiques : virtual_router_id et auth_pass doivent être strictement identiques sur les deux CT. La priorité détermine quel CT est actif par défaut.

4) Démarrer et tester

```
# Sur les deux CT
systemctl enable keepalived
systemctl start keepalived

# Vérifier que le MASTER a bien la VIP
ip addr show eth0
```

Vous devriez voir l'IP virtuelle (192.168.0.99) sur le CT 100, mais pas sur le CT 101

5) Tester la bascule

```
# Sur le CT 100 (MASTER)
systemctl stop keepalived   # Simule une panne

# Vérifier sur le CT 101
ip addr show eth0           # La VIP doit apparaître ici maintenant
```

La bascule prend 1 à 3 secondes. Pour réactiver le MASTER, redémarrez Keepalived dessus : il reprendra automatiquement la VIP.

### Pour aller plus loin : surveiller votre site web

Actuellement, Keepalived surveille uniquement que le CT est vivant. Pour qu'il bascule si votre site web tombe (même si le CT tourne), ajoutez un script de vérification:

- Créez `/etc/keepalived/check_website.sh` sur les deux CT :

```
bash

#!/bin/bash
# Teste si le site répond
curl -f http://localhost/ || exit 1
exit 0
```

- Rendez-le exécutable :

```
bash

chmod +x /etc/keepalived/check_website.sh
```

- Ajoutez ces lignes DANS le bloc vrrp_instance de chaque configuration :

```
bash

    track_script {
        check_website
    }
```

- Et avant le bloc vrrp_instance, ajoutez la définition du script :

```
bash

vrrp_script check_website {
    script "/etc/keepalived/check_website.sh"
    interval 2      # Test toutes les 2 secondes
    fall 2          # 2 échecs = KO
    rise 2          # 2 succès = OK
}
```

Ainsi, si votre serveur web plante, Keepalived basculera automatiquement sur l'autre CT.


💎 Résumé : quelle stratégie choisir ?

Stratégie	nopreempt	preempt_delay (ex: 300)
Comportement	Pas de retour automatique. Le BACKUP reste MASTER.	Retour automatique après un délai.
Avantage	Évite toute micro-coupure lors du retour du serveur principal.	Permet de vérifier la stabilité du serveur principal avant de lui redonner la main.
Inconvénient	Le serveur principal reste inactif jusqu'à la prochaine panne du BACKUP.	Il y a tout de même une micro-coupure lors du retour (moins grave qu'une panne).
Cas d'usage	Recommandé pour la plupart des sites web où la stabilité prime.	Utile si vous avez absolument besoin que le trafic retourne sur une machine plus puissante.


Les différents niveaux de "standby"

Niveau	État du serveur de secours	Temps de bascule	Exemple
Cold standby	Éteint. À démarrer manuellement	Minutes à heures	Backup sur disque dur
Warm standby	Allumé, mais services à lancer	Secondes à minutes	Keepalived sans services actifs
Hot standby	Allumé, services prêts, mais sans trafic	Secondes	Keepalived + services tournant
Active-Active	Les deux servent le trafic	Instantané	Load balancer (HAProxy)

[UP](#tutorial-proxmox-ve)