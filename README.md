# Proxmox VE Tutorial

- [Proxmox VE Tutorial](#proxmox-ve-tutorial)
  - [Installation](#installation)
  - [Access to Proxmox via Browser](#access-to-proxmox-via-browser)
  - [To verify from Proxmox Shell (PVE)](#to-verify-from-proxmox-shell-pve)
  - [VM or CT ?](#vm-or-ct-)
  - [Add user ✅](#add-user-)
  - [hot standby for website](#hot-standby-for-website)
    - [Pour aller plus loin : surveiller votre site web](#pour-aller-plus-loin--surveiller-votre-site-web)

## Installation

1) Choose Ventoy to flash a USB key with Proxmox ISO (last one version).

2) Start your server and press key : delete or F2 or F10 or F12.

3) Into the BOOT menu

- CPU => Advanced => virtualization enable (VMX or VT-x).
- Choose your USB device on first option to boot.
- Save changes and reset.

4) Into the GRUB of Proxmox VE

- If you're in trouble during installation, you can access to GRUB by pressing key "e" to add `nomodeset` at the end of linux line.

- **options -> target disk**

- ext4  => basically
- RAID0 => ZFS (I've got only one Hard Disk)
- RAID1 => ZFS (2 Hard Disks)

- I've choosen RAID0, because there is the best for performances.

- On your router you can choose xx.xx.xx.xx to xx.xx.xx.200 and choose to fix the ip to xx.xx.xx.201. It will be a static IP.

- After installation, remove the USB key before to click restart !

---

## Access to Proxmox via Browser

`https://192.168.xx.xx:8006`


At this page you can copy script:

https://community-scripts.org/categories?category=proxmox-and-virtualization&preview=post-pve-install

copy paste to your shell of pve

```
curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pve-install.sh -o post-install.sh
```

If the command `ping -c 2 github.com` returns "unknown host," the problem is with your DNS. Your DNS server is likely pointing to 127.0.0.1

Copy paste into the shell of pve

```
echo "nameserver 8.8.8.8" > /etc/resolv.conf
echo "nameserver 1.1.1.1" >> /etc/resolv.conf
```

and retry

```
curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pve-install.sh -o post-install.sh
```

`chmod +x post-install.sh`

`./post-install.sh`

then click always `yes`

reboot you pve

`reboot`

and clear your browser's cache !

You can verify your nameserver

`cat /etc/resolv.conf`

After that, go to GUI => pve => updates => repository 

Disable Entreprise & click on `no-subscription` and `ceph squid no-subscription` to replace disable components and create new one.

Then return, to click on updates => refresh

At the end, click on upgrades and press y for yes

reboot your pve

---

## To verify from Proxmox Shell (PVE)

1. Hard Disk:

`$ lsblk`

To display memory space:

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

7. Voir les problème du PVE

`journalctl | grep -E "ERROR|FATAL|segfault|panic"`

8. Lorsque le CT freeze soit ajouter de la RAM ou:
   
`root@pve:~# nano /etc/pve/lxc/100.conf`

et ajouter cette ligne:

`lxc.apparmor.profile: unconfined`

9.  Démarrer une application qui se trouve dans le CT-100

`pct exec 100 -- bash -c "cd /home/user/mon_app && node app.js"`

[⬆-up!](#proxmox-ve-tutorial)

---

## VM or CT ?

VM et CTX peuvent communiquer directement grâce au bridge: `vmbr0`

Si ils sont différents, il faut faire un port-forward et routage adapté. 


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

---

## Add user ✅

`pct enter 100`

1) Créer un utilisateur (ex: my_usr_name)

`adduser my_usr_name`

2) Ajouter aux sudoers

`usermod -aG sudo my_usr_name  # Pour Debian/Ubuntu`

or

`usermod -aG wheel my_usr_name  # Pour certaines distributions`


3) From pve node shell:

`pct push 100 /root/.ssh/id_rsa.pub /home/my_usr_name/.ssh/authorized_keys`

Ou depuis l'intérieur du CT

`pct enter 100`

`mkdir -p /home/my_usr_name/.ssh`

`cat /root/.ssh/authorized_keys >> /home/my_usr_name/.ssh/authorized_keys`

`chown -R my_usr_name:my_usr_name /home/my_usr_name/.ssh`

`chmod 600 /home/my_usr_name/.ssh/authorized_keys`

`ssh my_usr_name@192.168.XX.XX`

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