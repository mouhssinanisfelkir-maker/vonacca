# Vona Academy — plateforme de formation interne

Plateforme de formation vidéo façon Udemy, avec **un quiz après chaque vidéo** : l'apprenant ne peut pas accéder à la vidéo suivante tant qu'il n'a pas réussi le quiz de la précédente. Gérée par l'équipe RH, entièrement conteneurisée (Docker Compose).

## Les trois espaces

| Espace | Qui | Ce qu'on y fait |
|---|---|---|
| **Admin** | Administrateurs | Tableau de bord global, gestion de **tous** les comptes (admin, RH, apprenants), accès à tout l'espace RH |
| **RH** | Équipe RH | Création des **formations** (vidéos + quiz), choix des **apprenants concernés**, suivi de progression, export CSV, création/import des comptes apprenants |
| **Apprenant** | Collaborateurs | Voit uniquement les formations qui lui sont affectées, suit les leçons dans l'ordre |

## Démarrage rapide (VM Debian)

```bash
# 1. Installer Docker (une seule fois)
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER    # puis se reconnecter

# 2. Récupérer le projet, puis générer la configuration (secrets aléatoires)
cd vona-academy
./scripts/init-env.sh            # affiche le mot de passe admin initial

# 3. Lancer
docker compose up -d --build

# 4. Ouvrir http://<ip-de-la-vm>  (port défini par HTTP_PORT, 80 par défaut)
```

- Premier compte : `ADMIN_EMAIL` / `ADMIN_PASSWORD` du fichier `.env`. Le mot de passe est à changer dès la première connexion.
- Pour tester rapidement, mettez `SEED_DEMO=true` dans `.env` avant le premier lancement : un compte RH (`rh@vona.local`) et trois apprenants sont créés, mot de passe `Vona2026!`. **À désactiver en production.**
- Logs : `docker compose logs -f backend` · Arrêt : `docker compose down` (les données sont conservées dans les volumes).

## Utilisation

**Admin / RH — créer une formation**
1. *Apprenants* → créer les comptes (un par un, ou **import CSV** `email;prenom;nom;service`). Un mot de passe temporaire est généré et affiché une seule fois ; l'apprenant doit le changer à sa première connexion.
2. *Formations* → **Nouvelle formation** (titre, description, score minimum de réussite, 70 % par défaut).
3. Onglet **Contenu** → **Ajouter une leçon** (titre + vidéo MP4/WebM), puis bouton **Quiz** : questions à choix unique ou multiples, bonnes réponses cochées. Les flèches ↑ ↓ réordonnent les leçons.
4. Onglet **Apprenants concernés** → cocher les personnes (filtre par service, « tout sélectionner »), **Enregistrer**.
5. **Publier la formation** (impossible tant qu'une leçon n'a pas de vidéo ou de quiz). Le bouton **Aperçu apprenant** permet de tester sans rien enregistrer.
6. Onglet **Suivi** : progression, score moyen, dernière activité, **export CSV** (compatible Excel FR).

**Apprenant** : *Mes formations* → la leçon se lit, puis le quiz apparaît à la fin de la vidéo. Score ≥ seuil → leçon suivante débloquée. Sinon, il revoit la vidéo si besoin et réessaie (tentatives illimitées).

## Comment le blocage est garanti

Les règles sont appliquées **côté serveur**, pas seulement dans l'interface :
- Le flux vidéo (`/api/lessons/:id/video`) refuse l'accès tant que les quiz des leçons précédentes ne sont pas réussis, ou si l'apprenant n'est pas affecté à la formation.
- Le quiz n'est envoyé (et accepté) qu'une fois la vidéo vue (≥ 95 %). Les bonnes réponses ne quittent jamais le serveur et la correction est faite côté serveur.
- **Anti-saut** : le lecteur interdit d'avancer au-delà de ce qui a été regardé ; en plus, le serveur plafonne la progression au rythme de lecture réelle (marge x2,5), donc même une requête truquée ne peut pas valider une vidéo instantanément.
- Un compte désactivé perd l'accès immédiatement ; un apprenant désaffecté ne peut plus ouvrir la formation (sa progression est conservée).

Limite à connaître : comme pour toute plateforme vidéo, on ne peut pas empêcher techniquement un utilisateur déterminé de télécharger un flux qu'il a le droit de lire (enregistrement d'écran, outils développeur).

## Vidéos : format et dimensionnement

- Format recommandé : **MP4 H.264 + AAC**, 720p. Ré-encodage conseillé avant import (lecture fluide et démarrage rapide) :
  ```bash
  ffmpeg -i source.mov -c:v libx264 -preset medium -crf 23 -vf scale=-2:720 -c:a aac -b:a 128k -movflags +faststart cours.mp4
  ```
- Pour ~70 utilisateurs simultanés : en 720p (~1,5 Mb/s) comptez ≈ 105 Mb/s de débit sortant, en 1080p (~3 Mb/s) ≈ 210 Mb/s. Prévoyez la bande passante de la VM en conséquence et gardez ~1 Go de stockage par heure de vidéo 720p.
- Taille max d'un fichier : `MAX_VIDEO_MB` (2048 Mo par défaut). Si vous l'augmentez, adaptez aussi `client_max_body_size` dans `frontend/nginx.conf`.
- Les vidéos sont stockées dans le volume Docker `video_data` (`/data/videos` dans le conteneur backend), la base dans `db_data`.

## Charte graphique Vona

La charte a été reprise de la présentation Vona « Instant KM » : vert `#0B6104`, vert pâle `#E1ECD0`, fond `#FCFCFA`, polices **Roboto Slab** (titres) et **Ubuntu** (texte), coins arrondis à 8 px. Les polices sont embarquées dans l'application (aucune requête vers Google Fonts).

Tout passe par `frontend/src/theme.css` (variables `--brand-*`) et le composant `Logo` dans `frontend/src/components/ui.tsx`. Le logo actuel est le mot « VONA » en capitales espacées, car la présentation ne contenait pas de fichier logo : pour utiliser le logo officiel, déposez-le dans `frontend/public/` et affichez-le dans `Logo` avec une balise `<img>`. Après modification : `docker compose up -d --build web`.

## Sauvegarde et mise à jour

```bash
./scripts/backup.sh ./backups      # base PostgreSQL + vidéos
docker compose up -d --build       # après mise à jour du code (les volumes sont conservés)
```
Pensez à copier les sauvegardes hors de la VM.

## HTTPS et déploiement Azure

- **Sur une VM Debian (MVP)** : fonctionne en HTTP. Dès qu'un certificat TLS est en place (reverse proxy Caddy/Nginx/Traefik devant le port `HTTP_PORT`, ou Azure Application Gateway / Front Door), mettez `COOKIE_SECURE=true` dans `.env`. **Sans HTTPS, ne pas exposer la plateforme sur Internet** (mots de passe en clair sur le réseau).
- **Sur une VM Azure** : le même `docker-compose.yml` fonctionne tel quel (VM Linux + disque managé pour les volumes).
- **Vers des services managés** (cible possible) : images Docker dans Azure Container Registry → Azure Container Apps ; base → *Azure Database for PostgreSQL* (adapter `DATABASE_URL`) ; vidéos → partage *Azure Files* monté sur `/data/videos`. Un stockage Blob nécessiterait une petite évolution du code (`backend/src/upload.ts` et la route de streaming).

## Architecture

```
navigateur ──► web (Nginx : app React + relais /api) ──► backend (Node.js/Express, TypeScript) ──► db (PostgreSQL 16)
                                                              └──► volume video_data (/data/videos)
```
- `frontend/` : React + Vite + TypeScript (aucune dépendance UI externe)
- `backend/` : Express + TypeScript, auth par cookie de session `httpOnly` (JWT), mots de passe hachés (bcrypt), limitation des tentatives de connexion
- Seul le service `web` est exposé ; la base et l'API ne sont pas accessibles depuis l'extérieur.
- Développement local : `npm run dev` dans `backend/` (variables `DATABASE_URL`, `JWT_SECRET`, `ADMIN_PASSWORD`, `VIDEO_DIR`) et dans `frontend/` (proxy `/api` → port 4000).

## Pistes d'évolution (non incluses)

Certificats PDF de fin de formation, relances e-mail et dates limites, SSO Microsoft Entra ID, groupes d'apprenants, statistiques par question, transcodage vidéo automatique, stockage Blob.
