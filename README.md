# Serveur multimédia Jellyfin sur Raspberry Pi 5

Déploiement d'un serveur multimédia **Jellyfin** (films, séries, musique) sur un **Raspberry Pi 5**, avec **Docker Compose**. Le projet explique non seulement *quoi* faire, mais aussi *pourquoi* chaque choix a été fait, ainsi que les problèmes rencontrés et leurs solutions.

---

## Sommaire

1. [Présentation](#1-présentation)
2. [Architecture](#2-architecture)
3. [Prérequis](#3-prérequis)
4. [Structure du projet](#4-structure-du-projet)
5. [Le fichier `compose.yaml` expliqué](#5-le-fichier-composeyaml-expliqué)
6. [Installation pas à pas](#6-installation-pas-à-pas)
7. [Configuration de Jellyfin](#7-configuration-de-jellyfin)
8. [Ajouter des films](#8-ajouter-des-films)
9. [Performances et transcodage](#9-performances-et-transcodage)
10. [Problèmes rencontrés et solutions](#10-problèmes-rencontrés-et-solutions)
11. [Commandes utiles](#11-commandes-utiles)
12. [Sécurité et limites](#12-sécurité-et-limites)
13. [Évolutions possibles](#13-évolutions-possibles)
14. [Compétences mises en pratique](#14-compétences-mises-en-pratique)
15. [Auteur](#15-auteur)

---

## 1. Présentation

**Jellyfin** est un serveur multimédia libre et gratuit : il organise une collection de fichiers vidéo et audio, récupère automatiquement les affiches et les descriptions, puis permet de les lire depuis un navigateur, un téléphone ou une télévision connectée.

**Objectifs du projet :**

- héberger soi-même un serveur multimédia, sans abonnement ni cloud tiers ;
- apprendre à déployer un service avec **Docker Compose** ;
- comprendre les droits d'accès (UID/GID), les volumes et les ports d'un conteneur ;
- documenter la démarche pour la rendre reproductible.

---

## 2. Architecture

```
  PC / téléphone / TV                     Raspberry Pi 5
 ┌───────────────────┐   HTTP :8096   ┌─────────────────────────────┐
 │ Navigateur ou     │ ─────────────► │  Docker                     │
 │ application       │                │  ┌───────────────────────┐  │
 │ Jellyfin          │ ◄───────────── │  │ Conteneur "jellyfin"  │  │
 └───────────────────┘   flux vidéo   │  │  /config  /cache      │  │
                                      │  │  /media  (films...)   │  │
                                      │  └──────────┬────────────┘  │
                                      │             │ volumes       │
                                      │  ~/jellyfin/config, cache   │
                                      │  ~/media/films, series...   │
                                      └─────────────────────────────┘
```

Les clients se connectent à Jellyfin sur le **port 8096**. Les fichiers vidéo restent sur le Pi, dans `~/media`, et sont **montés** dans le conteneur. Les réglages, les comptes et la bibliothèque sont enregistrés dans `~/jellyfin/config`, ce qui permet de recréer le conteneur sans rien perdre.

---

## 3. Prérequis

| Élément | Détail |
|---|---|
| Matériel | Raspberry Pi 5 (4 ou 8 Go recommandés), alimentation officielle, carte microSD ou SSD |
| Système | Raspberry Pi OS (64 bits) |
| Logiciels | Docker et le plugin Docker Compose |
| Réseau | Pi et appareils clients sur le même réseau local |
| Accès | Connexion SSH au Pi (et WinSCP pour transférer des fichiers depuis Windows) |

Vérifier que Docker fonctionne :

```bash
docker --version
docker compose version
```

---

## 4. Structure du projet

Sur le Pi, deux dossiers sont utilisés :

```
~/jellyfin/              <- le projet (versionné sur GitHub)
├── compose.yaml         <- description du service
├── README.md
├── .gitignore
├── config/              <- réglages, comptes (NON versionné)
└── cache/               <- cache d'images, transcodage (NON versionné)

~/media/                 <- les médias (hors du dépôt)
├── films/
├── series/
└── musique/
```

Les dossiers `config/` et `cache/` sont exclus du dépôt par le `.gitignore` : ils contiennent des données propres à l'installation (comptes, base de données) qui n'ont rien à faire sur GitHub.

---

## 5. Le fichier `compose.yaml` expliqué

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    user: 1000:1000
    ports:
      - "8096:8096"
    volumes:
      - ./config:/config
      - ./cache:/cache
      - /home/julio/media:/media
    restart: unless-stopped
```

| Ligne | Rôle |
|---|---|
| `image: jellyfin/jellyfin:latest` | Image officielle de Jellyfin (compatible ARM64, donc Raspberry Pi). |
| `container_name: jellyfin` | Nom fixe du conteneur, pratique pour `docker logs jellyfin` ou `docker exec`. |
| `user: 1000:1000` | Le processus s'exécute avec l'UID/GID de l'utilisateur du Pi, pour que les fichiers de `config/` et `media/` lui appartiennent. **Ce sont des numéros, pas des noms** (voir la section 10). |
| `ports: "8096:8096"` | Port de l'hôte : port du conteneur. L'interface web est accessible sur `http://IP-DU-PI:8096`. |
| `./config:/config` | Les réglages sont stockés sur le Pi, donc conservés si le conteneur est supprimé. |
| `./cache:/cache` | Cache (images, fichiers temporaires). |
| `/home/julio/media:/media` | Les médias du Pi sont visibles dans le conteneur sous `/media`. |
| `restart: unless-stopped` | Le service redémarre automatiquement après un redémarrage du Pi, sauf s'il a été arrêté volontairement. |

> **À adapter :** remplacer `/home/julio/media` par le chemin du dossier média sur votre machine, et `1000:1000` par le résultat de la commande `id`.

---

## 6. Installation pas à pas

**1. Connaître son UID et son GID**

```bash
id
# uid=1000(julio) gid=1000(julio) ...
```

**2. Créer les dossiers**

```bash
mkdir -p ~/jellyfin/{config,cache} ~/media/{films,series,musique}
cd ~/jellyfin
```

**3. Créer le fichier `compose.yaml`** avec le contenu de la section 5 (par exemple avec `nano compose.yaml`).

**4. Démarrer le service**

```bash
docker compose up -d
docker logs --tail 30 jellyfin
```

Le premier démarrage peut prendre 30 à 60 secondes, surtout sur une carte microSD.

**5. Ouvrir l'interface**

Dans un navigateur : `http://<IP-DU-PI>:8096` (en **http**, pas https).

---

## 7. Configuration de Jellyfin

L'assistant de premier lancement demande :

1. **La langue** de l'interface.
2. **Un compte administrateur** : nom d'utilisateur et mot de passe solide.
3. **Les médiathèques** : *Ajouter une médiathèque*, choisir le type (Films, Séries, Musique) et le dossier.
   Les chemins sont ceux **du conteneur** : `/media/films`, `/media/series`, `/media/musique`, et non `/home/julio/media/...`.
4. **La langue des métadonnées** (pour les affiches et descriptions).
5. **L'accès à distance** : laissé **désactivé**, le serveur reste accessible uniquement depuis le réseau local.

Le nom du serveur (ici `Jellyfin-Pi`) est modifiable plus tard dans *Tableau de bord → Général*.

---

## 8. Ajouter des films

**Étape 1 : copier le fichier sur le Pi**

Depuis Windows, avec **WinSCP** : protocole SFTP, hôte = IP du Pi, port 22, utilisateur du Pi. Glisser le fichier du panneau gauche (PC) vers le panneau droit (Pi), dans `/home/<utilisateur>/media/films`.

Depuis le Pi, pour un fichier téléchargeable par lien direct (film libre de droits, par exemple) :

```bash
cd ~/media/films
wget "https://lien-direct-vers-le-fichier.mp4"
```

**Étape 2 : bien nommer le fichier**

Jellyfin identifie un film grâce à son nom. Le format le plus fiable est :

```
films/
└── Nom du film (année)/
    └── Nom du film (année).mp4
```

**Étape 3 : lancer un scan**

*Tableau de bord → Médiathèques → Analyser toutes les médiathèques*, puis recharger la page (`Ctrl+F5`). Jellyfin surveille aussi le dossier et détecte normalement les nouveaux fichiers de lui-même.

> Seuls des contenus que l'on a le droit de posséder doivent être ajoutés : fichiers personnels, films libres de droits (par exemple les films de la fondation Blender) ou copies légales.

---

## 9. Performances et transcodage

Le **transcodage** consiste à convertir une vidéo en temps réel pour qu'un appareil puisse la lire. C'est très gourmand en CPU.

- Le Raspberry Pi 5 **n'a pas d'encodeur matériel H.264** (contrairement au Pi 4) et son décodage HEVC est limité.
- Il vaut donc mieux privilégier la **lecture directe** (*direct play*) : le client lit le fichier tel quel, sans conversion.
- Les fichiers **MP4/MKV en H.264, 1080p** passent sans effort. Les fichiers HEVC ou 4K risquent de forcer un transcodage que le Pi supportera mal.

**Vérifier si le Pi transcode :** pendant une lecture, la charge CPU doit rester basse. Dans le menu de lecture, Jellyfin indique aussi « Lecture directe » ou « Transcodage » (*Informations de lecture*).

**Stockage :** une carte microSD se remplit vite (un film fait de 1 à 5 Go). Pour une vraie collection, un **SSD ou un disque USB 3** est recommandé, puis il suffit de modifier le volume `/home/julio/media:/media` dans le `compose.yaml`.

---

## 10. Problèmes rencontrés et solutions

| Problème | Cause | Solution |
|---|---|---|
| `unable to find user julio: no matching entries in passwd file` | Le `compose.yaml` contenait `user: julio`. Dans le conteneur, cet utilisateur n'existe pas. | Utiliser les numéros : `user: 1000:1000` (résultat de `id`), puis `docker compose down` et `docker compose up -d`. |
| Page inaccessible sur `:8096` | Conteneur arrêté ou encore en démarrage. | `docker compose ps`, `docker logs --tail 30 jellyfin`, patienter une minute. |
| `Permission denied` sur `/config` ou `/media` | Les dossiers n'appartiennent pas à l'UID du conteneur. | `sudo chown -R 1000:1000 ~/jellyfin ~/media` |
| Médiathèque vide | Dossier non ajouté dans Jellyfin, ou mauvais chemin. | Ajouter `/media/films` (chemin **du conteneur**), puis relancer un scan. |
| Vérifier ce que voit le conteneur | : | `docker exec jellyfin ls -lh /media/films` |
| Doublon `fichier.mp4.1` | La commande `wget` a été lancée deux fois. | Supprimer le doublon avec `rm`. |
| Lien de téléchargement en erreur 404 | Lien obsolète. | Rechercher un miroir à jour du fichier. |
| `scp` introuvable sous Windows | Client OpenSSH incomplet. | Utiliser **WinSCP** (interface graphique, protocole SFTP). |

---

## 11. Commandes utiles

```bash
# État et journaux
docker compose ps
docker logs --tail 50 jellyfin
docker logs -f jellyfin                 # suivre en direct

# Cycle de vie
docker compose restart
docker compose down                      # arrêter et supprimer le conteneur (les données restent)
docker compose up -d                     # (re)créer et démarrer

# Mise à jour de Jellyfin
docker compose pull
docker compose up -d

# Vérifications
docker exec jellyfin ls -lh /media/films
curl -I http://localhost:8096
df -h                                    # espace disque
```

**Sauvegarde :** copier le dossier `config/` suffit à conserver les comptes, les réglages et la bibliothèque.

```bash
tar czf jellyfin-config-$(date +%F).tar.gz -C ~/jellyfin config
```

---

## 12. Sécurité et limites

- Le service est exposé **uniquement sur le réseau local** : il n'est pas ouvert sur Internet.
- Le trafic est en **HTTP** (non chiffré), acceptable en réseau domestique mais à éviter hors du réseau local. Pour un accès à distance, passer par un **VPN** (par exemple WireGuard) plutôt que d'ouvrir un port.
- Utiliser un **mot de passe solide** pour le compte administrateur.
- Le conteneur tourne avec un **utilisateur non-root** (`1000:1000`), ce qui limite les dégâts en cas de faille.
- Les dossiers `config/` et `cache/` ne sont pas versionnés sur GitHub.

---

## 13. Évolutions possibles

- Ajouter un **SSD ou un disque USB 3** pour les médias.
- Donner un **nom local** au serveur via un DNS local (par exemple Pi-hole : `jellyfin.lan`).
- Placer Jellyfin derrière un **reverse proxy** (Nginx) avec HTTPS.
- Accès distant sécurisé via **WireGuard**.
- **Sauvegarde automatique** de `config/` avec une tâche `cron`.
- Ajouter les médiathèques **Séries** et **Musique**, ainsi qu'une médiathèque de **vidéos personnelles** séparée.

---

## 14. Compétences mises en pratique

- Déploiement de services avec **Docker et Docker Compose**
- Gestion des **volumes**, des **ports** et des **utilisateurs (UID/GID)** dans un conteneur
- Administration **Linux** en ligne de commande (SSH, permissions, `chown`, `wget`)
- **Diagnostic** de pannes à partir des journaux (`docker logs`) et des messages d'erreur
- Notions de **réseau** (ports, accès local, HTTP/HTTPS, VPN)
- **Documentation** technique et gestion de versions avec **Git/GitHub**

---

## 15. Auteur

**Julio Tekawe** : étudiant en bachelier informatique (orientation réseaux et systèmes), à la recherche d'un stage en administration réseaux et systèmes.

GitHub : [github.com/Tekawejulio](https://github.com/Tekawejulio)
