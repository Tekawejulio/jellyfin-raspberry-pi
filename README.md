# Serveur multimédia Jellyfin sur Raspberry Pi 5

Déploiement de Jellyfin avec Docker Compose sur un Raspberry Pi 5.

## Prérequis
- Raspberry Pi 5 sous Raspberry Pi OS
- Docker et Docker Compose installés
- Dossiers médias : `~/media/films`, `~/media/series`, `~/media/musique`

## Installation
```bash
mkdir -p ~/jellyfin/{config,cache} ~/media/{films,series,musique}
cd ~/jellyfin
docker compose up -d
```
Interface web : `http://<IP-DU-PI>:8096`

## Points d'attention
- `user: 1000:1000` : Docker attend l'UID/GID numériques, pas le nom `julio` (qui n'existe pas dans le conteneur). Vérifier avec la commande `id`.
- Le Pi 5 n'a pas d'encodeur matériel H.264 : privilégier les fichiers MP4/MKV en H.264 pour la lecture directe (direct play).
- Les dossiers `config/` et `cache/` ne sont pas versionnés (réglages et comptes).

## Dépannage rencontré
| Problème | Cause | Solution |
|---|---|---|
| `unable to find user julio` | nom d'utilisateur au lieu de l'UID | `user: 1000:1000` |
| Médiathèque vide | dossier non ajouté dans Jellyfin | ajouter `/media/films` (chemin du conteneur) |

## Auteur
Julio Tekawe — étudiant BAC IT (réseaux et systèmes)
