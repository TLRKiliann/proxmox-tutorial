# Create CT with LXC & CMD

| **Utilisez un CT si...** | **Utilisez une VM si...** |
|:--------------------------|:---------------------------|
| Vous voulez un service **léger** qui démarre en quelques secondes. | Vous avez besoin d'**isolation** et de sécurité maximale (ex: serveur exposé sur internet). |
| Vous voulez héberger une **application unique** (Pi-hole, Jellyfin, un serveur web). | Vous voulez utiliser un **autre noyau** (ex: un noyau custom ou un OS comme Windows ou FreeBSD). |
| Vous avez des **ressources limitées** (mémoire, CPU) et voulez en utiliser le moins possible. | Vous voulez **passer du matériel** (GPU, clé USB) simplement et de manière fiable. |
| Vous voulez **tester** une application rapidement et pouvoir la supprimer ou la cloner facilement. | Vous voulez déployer des environnements **Docker complexes** (préférez une VM Debian/Ubuntu dédiée). |

- Pas de live-migration : Contrairement aux VMs, un CT ne peut pas être migré "à chaud" (sans l'éteindre) vers un autre nœud d'un cluster Proxmox.

- Périphériques matériels : Passer du matériel spécifique (comme une clé USB ou une carte GPU) à un CT est plus complexe que pour une VM et nécessite souvent une configuration manuelle avancée (passerelle de périphériques)

**CT = CT + LXC = CTX**

- install template => debian 12 (bookworm)

- Click pve -> Create CT -> configuration

- CT 100 - pwd (& confirmed) - 1 CPU - 8 - 1024 RAM - 512 SWAP - DNS empty

## From pve node Shell

Help

`pct help <CMD>`

`pct help set`

List all CTX

`pct list`

Start CTX

`pct start <CT-ID>`

Display status

`pct status <CT-ID>`

Stop (not recommanded => choose "shutdown")

`pct stop <CT-ID>`

Config

`pct config <CT-ID>`

Enter

`pct enter <CT-ID>`

Reboot

`pct reboot <CT-ID>`

Destroy

`pct destroy <CT-ID>`

Display ip

`pct exec <CT-ID> -- ip a`

Show port

`pct exec <CT-ID> -- ss -tlnp`

Increase RAM or Cores

`pct set <CT-ID> --<option> <value>`

`pct set 100 --memory 4096`

`pct set 100 --cores 4`

Update

`pct exec <CT-ID> -- apt update`

`pct exec <CT-ID> -- bash -c "apt update && apt upgrade -y"`

`pct exec <CT-ID> -- bash -c "apt update && apt upgrade -y && apt autoremove -y"`

⚠️ Don't use dist-upgrade ⚠️

`pct reboot <CT-ID>`

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


[⬆-up!](#create-ct-with-lxc-&-cmd)

---

## Clone CTX

Le conteneur doit être arrêté pour un clone cohérent (sinon le clone peut être dans un état inconsistants)

`pct clone 101 102`

Si le conteneur doit être utilisé en production de manière indépendante : préférez pct clone --full 1 pour éviter toute dépendance avec l'original

`pct clone 101 102 --full 1`

`pct clone 101 102 --hostname my-new-conteneur`


[⬆-up!](#create-ct-with-lxc-&-cmd)

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

`cat /etc/pve/lxc/<CT-ID>.config`

```
...firewall=1...
```

OR

`cat /etc/pve/firewall/<CT-ID>.fw`

```
enable: 1
```

Verify

`systemctl status proxmox-firewall`

Display rule set

`iptables-save`

***To change rule for a CT***

1) Enter

`cat >> /etc/pve/firewall/<CT-ID>.fw << EOF`

1) Write line per line

```
> [RULES]
> IN ACCEPT -p tcp -dport 3000 -log nolog
> EOF
```

3) Restart firewall

`pve-firewall restart`

4) Verify:

`cat /etc/pve/firewall/<CT-ID>.fw`


[⬆-up!](#create-ct-with-lxc-&-cmd) 