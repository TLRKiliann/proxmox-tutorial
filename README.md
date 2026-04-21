# Proxmox VE Tutorial

- [Proxmox VE Tutorial](#proxmox-ve-tutorial)
  - [Installation](#installation)
  - [Access to Proxmox via Browser](#access-to-proxmox-via-browser)
  - [To verify from Proxmox Shell](#to-verify-from-proxmox-shell)
  - [VM or CT ?](#vm-or-ct-)
  - [Create CT with LXC](#create-ct-with-lxc)
  - [Verify from pve node Shell](#verify-from-pve-node-shell)
  - [Update CT](#update-ct)
  - [Clone CT](#clone-ct)
  - [Firewall installation](#firewall-installation)
  - [SSH](#ssh)
  - [Add user ✅](#add-user-)
  - [hot standby for website](#hot-standby-for-website)
    - [Pour aller plus loin : surveiller votre site web](#pour-aller-plus-loin--surveiller-votre-site-web)

## Installation

I've choose Ventoy to flash my USB key with Proxmox ISO (last one).

If you get some trouble with installation during installation, you can access to GRUB by pressing key "e" to add `nomodeset` at the end of linux line.

(options -> target disk)

- RAID0 => ZFS (I've got only one Hard Disk)
- RAID1 => ZFS (2 Hard Disks)

---

## Access to Proxmox via Browser

`https://192.168.xx.xx:8006`

---

## To verify from Proxmox Shell

1. Hard Disk:

`$ lsblk`

`$ df -h /`

2. Voir toutes les interfaces réseau

`ip link show`

3. Display network bridges

`brctl show`

4. Voir la configuration réseau de Proxmox

`cat /etc/network/interfaces`

5. Vérifiez la configuration réseau depuis le nœud Proxmox pour CT-101

`cat /etc/pve/lxc/101.conf`

6. Lister tous les stockages configurés

`pvesm status`

[⬆-up!](#proxmox-ve-tutorial)

---

## VM or CT ?

Quand utiliser une VM ?

    Besoin d’exécuter Windows ou un autre OS non Linux.

    Exigence d’isolation très stricte (sécurité, multi-locataire critique).

    Application nécessitant son propre noyau personnalisé ou modules kernel spécifiques.

    Migration depuis un environnement VMware/Hyper-V.

Quand utiliser un CT (conteneur LXC) ?

    Service Linux pur (web, base de données, cache, DNS, etc.).

    Recherche de haute densité (faire tourner des dizaines de conteneurs sur un même hôte).

    Démarrage ultra-rapide ou mise à l’échelle dynamique.

    Accès direct aux périphériques matériels avec moins de latence.

    ⚠️ Attention : Un CT partage le noyau de l’hôte → une faille d’élévation de privilèges dans le conteneur pourrait potentiellement affecter l’hôte ou d’autres CT. Les VM sont plus sûres à ce niveau.

En résumé : VM = isolation et compatibilité OS / CT = légèreté et performance Linux.

[⬆-up!](#proxmox-ve-tutorial)

## Create CT with LXC

| **Utilisez un CT si...** | **Utilisez une VM si...** |
|:--------------------------|:---------------------------|
| Vous voulez un service **léger** qui démarre en quelques secondes. | Vous avez besoin d'**isolation** et de sécurité maximale (ex: serveur exposé sur internet). |
| Vous voulez héberger une **application unique** (Pi-hole, Jellyfin, un serveur web). | Vous voulez utiliser un **autre noyau** (ex: un noyau custom ou un OS comme Windows ou FreeBSD). |
| Vous avez des **ressources limitées** (mémoire, CPU) et voulez en utiliser le moins possible. | Vous voulez **passer du matériel** (GPU, clé USB) simplement et de manière fiable. |
| Vous voulez **tester** une application rapidement et pouvoir la supprimer ou la cloner facilement. | Vous voulez déployer des environnements **Docker complexes** (préférez une VM Debian/Ubuntu dédiée). |

- Pas de live-migration : Contrairement aux VMs, un CT ne peut pas être migré "à chaud" (sans l'éteindre) vers un autre nœud d'un cluster Proxmox.

- Périphériques matériels : Passer du matériel spécifique (comme une clé USB ou une carte GPU) à un CT est plus complexe que pour une VM et nécessite souvent une configuration manuelle avancée (passerelle de périphériques)

*** CT LXC: ***

- install template => debian 12 (bookworm)

- Click pve -> Create CT -> configuration

- CT 100 - pwd (& confirmed) - 1 CPU - 8 - 1024 RAM - 512 SWAP - DNS empty

## Verify from pve node Shell

`pct stop 100`

`pct start 100`

`pct status 100`

`pct list`

`pct enter 100`

`pct reboot 100`

## Update CT

`pct enter 100`

ip

`pct exec 100 -- ip a`

update

`pct exec 100 -- apt update`

`pct exec 100 -- bash -c "apt update && apt upgrade -y"`

`pct exec 100 -- bash -c "apt update && apt upgrade -y && apt autoremove -y"`

⚠️ Don't use dist-upgrade into a CT ⚠️

`pct reboot 100`

⚠️ Don't use reboot into a CT ⚠️

Mises à jour automatisées : Un script (proxmox-ct-updater) peut gérer les mises à jour de tous vos CT selon un planning (par exemple, tous les 1ers du mois), en générant des logs détaillés .

`proxmox-ct-updater`

```
sudo install -m 755 scripts/proxmox-ct-update.sh /usr/local/sbin/proxmox-ct-update.sh
sudo /usr/local/sbin/proxmox-ct-update.sh
```

Vous pouvez relancer l'assistant à tout moment pour modifier les paramètres cron :

`sudo /usr/local/sbin/proxmox-ct-update.sh --configure`

Ou réinstaller la planification sans repasser par l'assistant :

`sudo /usr/local/sbin/proxmox-ct-update.sh --install`

`cat /etc/cron.d/proxmox-ct-update`

return

`0 2 1 * * root /usr/local/sbin/proxmox-ct-update.sh`

with cron

`bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/tools/pve/update-lxcs-cron.sh)"`

`crontab -l`

[⬆-up!](#proxmox-ve-tutorial)

---

## Clone CT

Le conteneur doit être arrêté pour un clone cohérent (sinon le clone peut être dans un état inconsistants)

`pct clone 101 102`

Si le conteneur doit être utilisé en production de manière indépendante : préférez pct clone --full 1 pour éviter toute dépendance avec l'original

`pct clone 101 102 --full 1`

`pct clone 101 102 --hostname mon-nouveau-conteneur`

[⬆-up!](#proxmox-ve-tutorial)

---

## Firewall installation

Database -> firewall -> options -> firewall (edit) -> yes
pve -> firewall -> options -> firewall (edit) -> yes
ct-100 -> firewall -> options -> firewall (edit) -> yes
       -> network -> firewall = yes

*** To verify ***

For datacenter:

`cat /etc/pve/firewall/cluster.fw`

```
enable: 1
```

For pve:

`pve-firewall status`

```
Status: enabled/running
```

For CT-100 LXC:

`cat /etc/pve/lxc/100.config`

```
...firewall=1...
```

OR

`cat /etc/pve/firewall/100.fw`

```
enable: 1
```

Verify:

`systemctl status proxmox-firewall`

Display rule set:

`iptables-save`

***To change rule for a CT***

1) Enter:

`cat >> /etc/pve/firewall/100.fw << EOF`

2) Write line per line:

```
> [RULES]
> IN ACCEPT -p tcp -dport 3000 -log nolog
> EOF
```

3) Restart firewall:

`pve-firewall restart`

4) Verify:

`cat /etc/pve/firewall/100.fw`

[⬆-up!](#proxmox-ve-tutorial)
 
---

## SSH

CT are only accessible by SSH connection.

```
Your terminal
    │
    ├── $ pct enter 100 ──► DANS le conteneur
    │       │
    │       ├── nano /etc/ssh/sshd_config
    │       ├── systemctl restart sshd
    │       └── exit
    │
    └── $ pct push 100 ... ─► From Host Proxmox
```

1) Toujours dans le conteneur (pct enter 100)

`apt update`

`apt install openssh-server -y`

- Set SSH config at first and change root after !

`ssh-keygen -t ed25519 -C "votre-email@exemple.com"`

- From pve node shell :

`pct push 100 ~/.ssh/id_ed25519.pub /root/.ssh/authorized_keys`

If it's failed:

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

`systemctl status ssh.socket`

normaly enable

2) Exit of CT (Ctrl+D ou "exit")

From host (pve) :

`pct push 100 /root/.ssh/id_rsa.pub /root/.ssh/authorized_keys`

`ssh root@IP_DU_CONTENEUR_100`

- Depuis votre serveur Proxmox (pas depuis la session SSH)

`ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`


1) En "jump host" depuis votre machine personnelle (la plus élégante)

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

`systemctl status sshd`

2. SSH écoute-t-il sur le bon port ?

`netstat -tlnp | grep :22`

OR

`ss -tlnp | grep sshd`

3. Le pare-feu bloque-t-il ?

`iptables -L -n | grep :22`

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

[⬆-up!](#proxmox-ve-tutorial)

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

[⬆-up!](#proxmox-ve-tutorial)

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

| Stratégie | nopreempt | preempt_delay (ex: 300) |
|:----------|:----------|:-------------------------|
| **Comportement** | Pas de retour automatique. Le BACKUP reste MASTER. | Retour automatique après un délai. |
| **Avantage** | Évite toute micro-coupure lors du retour du serveur principal. | Permet de vérifier la stabilité du serveur principal avant de lui redonner la main. |
| **Inconvénient** | Le serveur principal reste inactif jusqu'à la prochaine panne du BACKUP. | Il y a tout de même une micro-coupure lors du retour (moins grave qu'une panne). |
| **Cas d'usage** | Recommandé pour la plupart des sites web où la stabilité prime. | Utile si vous avez absolument besoin que le trafic retourne sur une machine plus puissante. |

*** Les différents niveaux de "standby" ***

| Niveau | État du serveur de secours | Temps de bascule | Exemple |
|:-------|:---------------------------|:-----------------|:--------|
| **Cold standby** | Éteint. À démarrer manuellement | Minutes à heures | Backup sur disque dur |
| **Warm standby** | Allumé, mais services à lancer | Secondes à minutes | Keepalived sans services actifs |
| **Hot standby** | Allumé, services prêts, mais sans trafic | Secondes | Keepalived + services tournant |
| **Active-Active** | Les deux servent le trafic | Instantané | Load balancer (HAProxy) |

[⬆-up!](#proxmox-ve-tutorial)