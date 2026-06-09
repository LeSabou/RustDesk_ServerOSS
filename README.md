# Déploiement d'un serveur RustDesk auto-hébergé (OSS) sur Proxmox LXC

> Installation **native** (sans Docker) des serveurs RustDesk `hbbs` et `hbbr`
> dans un conteneur LXC Debian 12, géré via **systemd**.
> Objectif : permettre à une équipe de prendre la main à distance sur des postes clients.

---

## Sommaire

- [Prérequis](#prérequis)
- [Architecture](#architecture)
- [1. Créer le conteneur LXC](#1-créer-le-conteneur-lxc)
- [2. Préparer le système](#2-préparer-le-système)
- [3. Installer les binaires RustDesk](#3-installer-les-binaires-rustdesk)
- [4. Créer un utilisateur dédié](#4-créer-un-utilisateur-dédié)
- [5. Créer les services systemd](#5-créer-les-services-systemd)
- [6. Activer et démarrer les services](#6-activer-et-démarrer-les-services)
- [7. Récupérer la clé publique](#7-récupérer-la-clé-publique)
- [8. Ouvrir les ports](#8-ouvrir-les-ports)
- [9. Configurer les clients RustDesk](#9-configurer-les-clients-rustdesk)
- [10. Tester](#10-tester)
- [Maintenance](#maintenance)
- [Dépannage](#dépannage)
- [Notes de sécurité](#notes-de-sécurité)

---

## Prérequis

- Un serveur **Proxmox VE** fonctionnel
- Un template **Debian 12 standard** disponible
- Une plage IP locale connue (exemple utilisé ici : `192.168.1.0/24`)
- Pour l'accès distant : accès à la configuration de votre box/routeur (redirection de ports)

---

## Architecture

| Composant | Rôle |
|-----------|------|
| `hbbs` | Serveur de rendez-vous / signaling (enregistrement des ID, NAT traversal) |
| `hbbr` | Serveur relais (relaie le flux quand la connexion directe échoue) |

Avantages de la méthode native vs Docker :

- LXC **non privilégié** possible → meilleure sécurité
- Très léger (~512 Mo de RAM)
- Pas de couche Docker à maintenir
- Gestion simple via `systemctl` / `journalctl`

---

## 1. Créer le conteneur LXC

### 1.1 Télécharger le template

Dans Proxmox : **Nœud → local (pve) → CT Templates → Templates**, rechercher
`debian-12-standard` puis **Download**.

### 1.2 Créer le CT

**Create CT** avec les paramètres suivants :

| Section | Valeur |
|---------|--------|
| Hostname | `rustdesk` |
| Unprivileged container | ✅ coché (laissé par défaut) |
| Password | *(définir un mot de passe root)* |
| Template | `debian-12-standard` |
| Disk | 8 Go |
| CPU | 1 cœur |
| Memory | 512 Mo (Swap 512 Mo) |
| Bridge | `vmbr0` |
| IPv4 | Static — `192.168.1.50/24`, Gateway `192.168.1.1` |
| DNS | par défaut ou `1.1.1.1` |

> Adaptez l'adresse IP à votre réseau.

### 1.3 Démarrer

Sélectionner le CT → **Start** → ouvrir la **Console** (ou se connecter en SSH).

---

## 2. Préparer le système

Connecté en `root` dans le conteneur :

```bash
apt update && apt upgrade -y
apt install -y curl wget unzip
```

---

## 3. Installer les binaires RustDesk

Vérifiez d'abord la dernière version sur la page des releases :
<https://github.com/rustdesk/rustdesk-server/releases>

```bash
cd /tmp
wget https://github.com/rustdesk/rustdesk-server/releases/latest/download/rustdesk-server-linux-amd64.zip
unzip rustdesk-server-linux-amd64.zip
```

L'archive contient un dossier `amd64` avec les binaires `hbbs` et `hbbr` :

```bash
cp amd64/hbbs /usr/local/bin/
cp amd64/hbbr /usr/local/bin/
chmod +x /usr/local/bin/hbbs /usr/local/bin/hbbr
```

Vérification :

```bash
hbbs --help
```

> Si l'arborescence du zip diffère, lancez `ls -R /tmp` pour localiser les binaires
> et adaptez les chemins des commandes `cp`.

---

## 4. Créer un utilisateur dédié

On évite de faire tourner le service en `root` :

```bash
useradd -r -s /usr/sbin/nologin rustdesk
mkdir -p /opt/rustdesk
chown rustdesk:rustdesk /opt/rustdesk
```

Les clés et la base de données seront stockées dans `/opt/rustdesk`.

---

## 5. Créer les services systemd

### 5.1 Service `hbbs` (signaling)

Fichier `/etc/systemd/system/rustdesk-hbbs.service` :

```ini
[Unit]
Description=RustDesk Signal Server (hbbs)
After=network.target

[Service]
Type=simple
User=rustdesk
Group=rustdesk
WorkingDirectory=/opt/rustdesk
ExecStart=/usr/local/bin/hbbs
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 5.2 Service `hbbr` (relay)

Fichier `/etc/systemd/system/rustdesk-hbbr.service` :

```ini
[Unit]
Description=RustDesk Relay Server (hbbr)
After=network.target

[Service]
Type=simple
User=rustdesk
Group=rustdesk
WorkingDirectory=/opt/rustdesk
ExecStart=/usr/local/bin/hbbr
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## 6. Activer et démarrer les services

```bash
systemctl daemon-reload
systemctl enable --now rustdesk-hbbr
systemctl enable --now rustdesk-hbbs
```

> On démarre `hbbr` (relay) avant `hbbs` (signaling).

Vérifier l'état :

```bash
systemctl status rustdesk-hbbs
systemctl status rustdesk-hbbr
```

Les deux services doivent être **active (running)**.

---

## 7. Récupérer la clé publique

`hbbs` génère la paire de clés au premier démarrage :

```bash
cat /opt/rustdesk/id_ed25519.pub
```

Exemple de sortie :

```
OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=
```

> **Conservez cette clé** : elle est nécessaire pour configurer chaque client.
> Elle authentifie et chiffre les connexions.

---

## 8. Ouvrir les ports

| Port | Protocole | Usage |
|------|-----------|-------|
| 21115 | TCP | hbbs — test du type de NAT |
| 21116 | **TCP et UDP** | hbbs — enregistrement des ID / heartbeat |
| 21117 | TCP | hbbr — relais |
| 21118 | TCP | hbbs — client web (optionnel) |
| 21119 | TCP | hbbr — client web (optionnel) |

> ⚠️ Le port **21116 en UDP** est indispensable. Ne l'oubliez pas.

### 8.1 Dans le LXC

Debian standard n'a pas de firewall actif par défaut. Si vous utilisez `ufw` :

```bash
ufw allow 21115:21119/tcp
ufw allow 21116/udp
```

### 8.2 Sur la box / routeur (accès depuis Internet)

Rediriger les ports ci-dessus vers l'IP du conteneur (`192.168.1.50`).

---

## 9. Configurer les clients RustDesk

Sur chaque poste (techniciens et clients) :

1. Télécharger RustDesk : <https://rustdesk.com/>
2. Ouvrir RustDesk → bouton **⋮** → **Network**
   (ou **Paramètres → Réseau → ID/Relay Server**)
3. Renseigner :

   | Champ | Valeur |
   |-------|--------|
   | ID Server | `192.168.1.50` *(ou IP publique / domaine pour l'accès distant)* |
   | Relay Server | même valeur, ou laisser vide |
   | API Server | vide |
   | Key | la clé publique de l'[étape 7](#7-récupérer-la-clé-publique) |

4. Valider (**OK**) → un voyant vert confirme la connexion.

> **Accès distant (clients externes)** : utilisez votre **IP publique** ou un
> **nom de domaine** (DynDNS si IP dynamique), jamais l'IP locale `192.168.x.x`.

---

## 10. Tester

1. Sur le poste client : RustDesk → **⋮** → définir un **mot de passe permanent**.
2. Depuis votre poste : saisir l'**ID** du client → **Connect** → saisir le mot de passe.
3. La prise en main démarre.

---

## Maintenance

### Mise à jour du serveur

```bash
systemctl stop rustdesk-hbbs rustdesk-hbbr
cd /tmp
wget https://github.com/rustdesk/rustdesk-server/releases/latest/download/rustdesk-server-linux-amd64.zip
unzip -o rustdesk-server-linux-amd64.zip
cp amd64/hbbs amd64/hbbr /usr/local/bin/
chmod +x /usr/local/bin/hbbs /usr/local/bin/hbbr
systemctl start rustdesk-hbbr rustdesk-hbbs
```

### Sauvegarde

Sauvegarder le dossier `/opt/rustdesk` (clés + base de données), ou réaliser un
**backup Proxmox du conteneur** qui capture l'ensemble.

> ⚠️ La perte de `id_ed25519` / `id_ed25519.pub` oblige à reconfigurer **tous** les clients.

---

## Dépannage

| Symptôme | Piste |
|----------|-------|
| Pas de voyant vert sur le client | Vérifier l'ID Server, la clé, et que les ports sont ouverts |
| Connexion impossible à distance | Vérifier la redirection de ports, surtout **21116/UDP** |
| Service qui ne démarre pas | `journalctl -u rustdesk-hbbs -f` et `journalctl -u rustdesk-hbbr -f` |
| Clé publique introuvable | Vérifier que `hbbs` a bien démarré au moins une fois |

Consulter les logs en temps réel :

```bash
journalctl -u rustdesk-hbbs -f
journalctl -u rustdesk-hbbr -f
```

---

## Notes de sécurité

- Définir un **mot de passe permanent fort** sur chaque poste client.
- Limiter l'exposition Internet au strict nécessaire ; envisager un **VPN** plutôt
  qu'une exposition directe des ports lorsque c'est possible.
- Tenir le serveur RustDesk **à jour** (voir [Maintenance](#maintenance)).
- Restreindre les redirections de ports aux seules adresses nécessaires si votre
  routeur le permet.

---

## Références

- Dépôt du serveur RustDesk : <https://github.com/rustdesk/rustdesk-server>
- Documentation officielle : <https://rustdesk.com/docs/>
- Client RustDesk : <https://rustdesk.com/>

---

*Documentation fournie à titre informatif — adaptez les adresses IP et chemins à votre environnement.*
