*Projet réalisé dans le cadre du cursus de l'École 42 Paris par gustgonz.*

# Born2beRoot — serveur Debian sécurisé et script de monitoring

## Description

Born2beRoot est le premier projet d'administration système du cursus 42 : installer **à la main** un serveur Debian sans interface graphique dans une machine virtuelle, puis le sécuriser selon un cahier des charges strict (partitionnement chiffré, pare-feu, SSH, politique de mots de passe, sudo) et écrire un script de supervision.

La machine virtuelle elle-même n'est pas publiable (image disque de plusieurs Go). Ce dépôt contient :

- **`monitoring.sh`** — le script de supervision que j'ai écrit, analysé commande par commande ci-dessous ;
- le **rappel des exigences du sujet**, présentées comme telles : elles décrivent ce que la VM devait respecter à l'évaluation, pas une configuration vérifiable dans ce dépôt.

---

## Architecture

```
  Hôte ── VirtualBox ──────────────────────────────────────────────────────┐
  │   ssh -p 4242 user@vm                                                  │
  │        │                                                               │
  │   ┌────▼───────────────────────── VM Debian (sans GUI) ─────────────┐  │
  │   │  UFW : seul 4242/tcp ouvert      sshd : port 4242, root interdit│  │
  │   │  AppArmor actif                  sudo : politique renforcée     │  │
  │   │                                                                 │  │
  │   │  cron (toutes les 10 min) ──▶ monitoring.sh ──▶ wall (tous les  │  │
  │   │                                                terminaux)       │  │
  │   │  Disque (schéma type) :                                         │  │
  │   │   /boot (non chiffrée)                                          │  │
  │   │   LUKS ──▶ LVM (VG) ──▶ LV root, swap, home, …                  │  │
  │   └─────────────────────────────────────────────────────────────────┘  │
  └────────────────────────────────────────────────────────────────────────┘
```

---

## Ce que j'ai mis en place

### Exigences du sujet (partie obligatoire)

| Domaine | Exigence |
|---|---|
| Système | dernière Debian stable (ou Rocky), **sans interface graphique** ; AppArmor actif au démarrage |
| Disque | au moins **2 partitions chiffrées avec LVM** |
| Hostname | `login42` |
| Utilisateur | un utilisateur au nom du login, membre des groupes `user42` et `sudo` |
| SSH | service sur le **port 4242** uniquement, **connexion root interdite** |
| Pare-feu | **UFW** actif, seul le port 4242 ouvert |
| Mots de passe — durée | expiration tous les **30 jours**, **2 jours** minimum entre deux changements, avertissement **7 jours** avant expiration |
| Mots de passe — complexité | **10 caractères** minimum, au moins une majuscule, une minuscule et un chiffre ; pas plus de **3 caractères identiques consécutifs** ; ne doit pas contenir le nom de l'utilisateur ; au moins **7 caractères différents** de l'ancien mot de passe (sauf root) ; la politique s'applique aussi à root |
| sudo | **3 essais** maximum ; message d'erreur personnalisé ; journal des entrées et sorties dans **`/var/log/sudo/`** ; mode **TTY** obligatoire ; **`secure_path`** restreint |
| Monitoring | un script **bash** `monitoring.sh` qui affiche des informations système sur **tous les terminaux** au démarrage puis **toutes les 10 minutes** |

Outils du sujet associés à ces exigences : partitionnement LVM/LUKS de l'installateur Debian, `ufw`, `/etc/ssh/sshd_config`, `/etc/login.defs` + `chage` (durées), `libpam-pwquality` (complexité), `visudo` / `/etc/sudoers.d/`, `crontab`.

### `monitoring.sh` — analyse commande par commande

Le script calcule chaque valeur dans une variable, puis envoie tout en un seul message avec `wall`.

| Ligne affichée | Commande | Explication |
|---|---|---|
| Architecture | `uname -a` | nom du noyau, hostname, version et architecture |
| CPU physical | `grep "physical id" /proc/cpuinfo \| sort -u \| wc -l` | chaque processeur physique (socket) a un `physical id` ; `sort -u` garde les valeurs distinctes, `wc -l` les compte |
| vCPU | `grep "processor" /proc/cpuinfo \| wc -l` | une entrée `processor` par cœur logique |
| Memory Usage | `free --mega \| awk '$1 == "Mem:" {…}'` | sur la ligne `Mem:`, colonne 2 = total, colonne 3 = utilisée (en Mo) ; pourcentage calculé avec `printf("%.2f")` |
| Disk Usage | `df -m \| grep "/dev/" \| grep -v "/boot" \| awk …` | somme des tailles (col. 2) et de l'espace utilisé (col. 3) de tous les systèmes de fichiers sur un périphérique `/dev/`, hors `/boot` ; total converti en Go (`/1024`), utilisé affiché en Mo |
| CPU load | `vmstat 1 2 \| tail -1 \| awk '{printf $15}'` puis `expr 100 - …` | `vmstat 1 2` prend deux mesures à 1 s d'intervalle (la première est une moyenne depuis le boot, d'où `tail -1`) ; la colonne 15 est `id` (% d'inactivité) ; charge = 100 − inactivité |
| Last boot | `who -b \| awk '$1 == "system" {print $3 " " $4}'` | ligne `system boot AAAA-MM-JJ HH:MM` : on garde date et heure |
| LVM use | `lsblk \| grep "lvm" \| wc -l` dans un `if … -gt 0` | `lsblk` indique le type `lvm` des volumes logiques ; au moins un → `yes` |
| Connections TCP | `ss -ta \| grep ESTAB \| wc -l` | `ss -t` (TCP) `-a` (tous les états), on compte les connexions établies |
| User log | `users \| wc -w` | `users` liste les sessions ouvertes sur une ligne ; `wc -w` compte les mots (un utilisateur avec 2 sessions compte 2 fois) |
| Network | `hostname -I` et `ip link \| grep "link/ether" \| awk '{print $2}'` | adresses IP de la machine, puis adresse(s) MAC des interfaces Ethernet |
| Sudo | `journalctl _COMM=sudo \| grep COMMAND \| wc -l` | filtre le journal systemd sur les messages émis par `sudo` ; chaque commande exécutée produit une ligne `COMMAND=` |
| (envoi) | `wall "…"` | diffuse le message sur tous les terminaux connectés |

Remarques en relisant le script :
- **Unités de disque** : l'utilisé est en Mo et le total en Go, ce qui donne un affichage du type `1234/7.8Gb` ; le pourcentage, lui, est calculé sur les mêmes unités et reste juste.
- `cpu_fin` applique `printf "%.1f"` à un entier : le résultat est toujours `X.0`.
- `journalctl` et `wall` sans restriction supposent que le script est lancé par **root** (crontab de root), comme prévu dans le sujet.
- Si la VM a plusieurs interfaces Ethernet, plusieurs MAC sont affichées.

---

## Lancer / tester

Sur une Debian (VM ou machine de test), en root :

```bash
chmod +x monitoring.sh
sudo ./monitoring.sh        # le message apparaît sur tous les terminaux ouverts
```

Programmation toutes les 10 minutes, dans la crontab de root (`sudo crontab -e`) :

```cron
*/10 * * * * /usr/local/bin/monitoring.sh
```

Vérifications utiles pendant l'évaluation, sur la VM :

```bash
lsblk                           # partitions chiffrées (crypt) et volumes LVM
sudo ufw status numbered        # seul 4242 autorisé
ss -tunlp | grep 4242           # sshd en écoute sur 4242
sudo chage -l <user>            # politique d'expiration
hostnamectl                     # hostname
groups <user>                   # user42, sudo
ls /var/log/sudo/               # journaux sudo
```

---

## Difficultés rencontrées

- **Calcul de la charge CPU** : la première ligne de `vmstat` est une moyenne depuis le démarrage, pas une mesure instantanée ; il faut deux échantillons et garder le dernier.
- **Choisir la bonne colonne** : `free`, `df` et `vmstat` ont des sorties en colonnes qu'il faut connaître pour les découper avec `awk`.
- **Comptage des commandes sudo** : le journal systemd filtré par `_COMM=sudo` donne une ligne `COMMAND=` par commande exécutée, sans dépendre du fichier de log configuré pour sudo.
- **Mesures cohérentes entre outils** : `free --mega` et `df -m` n'utilisent pas exactement la même unité que l'affichage final ; le mélange Mo / Go de la ligne disque en est une trace.

---

## Compétences mobilisées

- Installation et configuration de Debian sans interface graphique dans une VM (VirtualBox, imposé par le sujet)
- Stockage : partitionnement, chiffrement LUKS, LVM (PV, VG, LV)
- Sécurité : SSH durci, pare-feu UFW, AppArmor, politique de mots de passe (PAM, `login.defs`, `chage`), configuration de sudo
- Bash et outils système : `/proc`, `free`, `df`, `vmstat`, `ss`, `lsblk`, `journalctl`, `awk`, `grep`, `wall`
- Planification avec cron
