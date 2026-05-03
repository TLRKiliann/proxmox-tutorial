# SSH

- CT are only accessible by SSH connection.

```
Your terminal
    │
    ├── $ pct enter <CT-ID> ──► DANS le conteneur
    │       │
    │       ├── nano /etc/ssh/sshd_config
    │       ├── systemctl restart sshd
    │       └── exit
    │
    └── $ pct push <CT-ID> ... ─► From Host Proxmox
```

1) Toujours dans le conteneur (pct enter <CT-ID>)

`apt update`

`apt install openssh-server -y`

- Set SSH config at first and change root after !

`ssh-keygen -t ed25519 -C "votre-email@exemple.com"`

- From pve node shell :

`pct push <CT-ID> ~/.ssh/id_ed25519.pub /root/.ssh/authorized_keys`

If it's failed:

`pct enter <CT-ID>`

`mkdir -p /root/.ssh`

`chmod 700 /root/.ssh`

If it's ok:

`pct enter <CT-ID>`

`nano /etc/ssh/sshd_config`

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

2) Exit of CT (Ctrl+D or "exit")

From host (pve) :

`pct push <CT-ID> /root/.ssh/id_rsa.pub /root/.ssh/authorized_keys`

`ssh root@IP_DU_CONTENEUR_<CT-ID>`

- Depuis votre serveur Proxmox (pas depuis la session SSH)

`ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`


1) En "jump host" depuis votre machine personnelle (la plus élégante)

Configurez votre fichier ~/.ssh/config sur votre machine personnelle :

`ssh-config`

```
Host proxmox-host
    HostName IP_DU_SERVEUR_PROXMOX
    User root

Host lxc-<CT-ID>
    HostName IP_DU_CONTENEUR_<CT-ID>
    User admin
    ProxyJump proxmox-host
```

Ensuite, depuis votre machine personnelle :
bash

`ssh lxc-<CT-ID>`

Vous passez automatiquement par l'hôte Proxmox pour atteindre le conteneur.

Problème fréquent : "Connection refused"

Si SSH refuse la connexion, vérifiez depuis l'intérieur du conteneur :
bash

`pct enter <CT-ID>`

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
                    └─ pct enter <CT-ID> (accès root absolu)

```

[⬆-up!](#ssh)