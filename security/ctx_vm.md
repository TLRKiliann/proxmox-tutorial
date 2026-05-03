# CTX or VM

- CTX is linked to the pve

- VM is not linked to the pve

Warning with update with template !


Exécution de code non fiable : Si vous téléchargez des fichiers depuis Internet (ex: via un client Torrent), une VM isole totalement ce risque du système Proxmox

Tests de sécurité : Pour analyser des malwares ou faire du "reverse engineering", la VM est indispensable. Un CTX partage le noyau de l'hôte, ce qui représente un risque majeur

Hébergement de services exposés : Pour une application critique qui traite des données sensibles, l'isolation d'une VM est la configuration la plus sûre.

Docker "out of the box" : Contrairement à un CTX, Docker s'installe et fonctionne immédiatement dans une VM, sans réglages de nesting, sans mode privilégié, et sans risque de casse après une mise à jour de Proxmox.

Gestion simplifiée : Des scripts automatisent la création de VMs Debian avec Docker, Cloud-Init, et même des stacks comme Filebrowser ou la surveillance déjà préinstallés.

Portabilité : Une VM avec Docker est facile à sauvegarder, cloner ou migrer entre différents serveurs Proxmox.

💡 Exemple concret : De nombreux utilisateurs créent une unique VM Debian "Docker host" qui fera tourner tous leurs conteneurs (Nextcloud, AdGuard, N8N, etc.)

```
Si votre besoin est...	                                                La meilleure option est...
--------------------------------------------------------------------------------------------------------
Isoler du code non fiable, un serveur Torrent ou un test de sécurité    VM (sécurité maximale)
Faire tourner Docker simplement et sans risques                         VM (compatibilité et stabilité)
Avoir besoin d'un driver ou module noyau spécifique                     VM (liberté totale sur le noyau)
Créer des templates déployables en masse                                VM (Cloud-Init)
Faire tourner Next.js avec Docker en environnement perso                VM (plus simple et plus stable)
Maximiser le nombre de services sur un petit serveur (8 Go de RAM)      CTX (économie de ressources)
Exécuter un petit script ou un service très simple et stable            CTX (démarrage instantané)
```