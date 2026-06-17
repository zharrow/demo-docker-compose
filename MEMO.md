# 📘 MÉMO TP — Docker, Compose & CI/CD

> Fiche de révision simple. But : comprendre **ce qu'on a fait** et **pourquoi**, avec
> toutes les commandes de base. Lis-la de haut en bas, c'est le déroulé du TP.

---

## 1. L'idée en une phrase

On a une appli en **2 morceaux** (backend + frontend). On emballe chaque morceau dans une
**image Docker**, on lance le tout ensemble avec **Docker Compose**, et une **pipeline**
(GitHub Actions) construit et envoie automatiquement les images sur **Docker Hub** à chaque
modification du code.

```
   CODE (GitHub)  ──push──►  PIPELINE (GitHub Actions)  ──build+push──►  Docker Hub
        │                                                                    │
        │                                                              (images prêtes
   docker compose up                                                   à déployer partout)
        │
   ┌────┴─────────────────────┐
   │  db  +  backend  +  front │   ← tourne sur ta machine
   └──────────────────────────┘
```

---

## 2. Le vocabulaire (à connaître par cœur)

| Mot | Définition simple |
|-----|-------------------|
| **Image** | Un "gabarit" figé qui contient ton appli + tout ce qu'il lui faut pour tourner. Comme une **photo** d'un système prêt à l'emploi. |
| **Conteneur** | Une **image qu'on lance**. L'image est la recette, le conteneur est le plat servi. On peut lancer plusieurs conteneurs depuis une même image. |
| **Dockerfile** | La **recette** pour fabriquer une image (liste d'instructions : `FROM`, `COPY`, `RUN`…). |
| **Registry** | Un **entrepôt d'images** en ligne. Docker Hub est le plus connu. |
| **Tag** | Une **étiquette de version** sur une image (`latest`, `v1.0`, `sha-18486b4`). |
| **Docker Compose** | Un outil pour lancer **plusieurs conteneurs ensemble** avec un seul fichier (`docker-compose.yml`). |
| **CI/CD** | **CI** = Intégration Continue (on build/teste à chaque commit). **CD** = Déploiement Continu (on publie automatiquement). |
| **Pipeline** | La **suite d'étapes automatiques** déclenchées par un `git push` (ici : build → tag → push). |
| **Monorepo** | **Un seul dépôt Git** qui contient plusieurs projets (ici back + front). |

---

## 3. Le multi-stage (point souvent demandé)

Un Dockerfile **multi-stage** = plusieurs étapes `FROM` dans le même fichier.
On **construit** dans une 1ʳᵉ étape (avec tous les outils lourds), puis on **copie juste
le résultat** dans une 2ᵉ étape minimale.

**Pourquoi ?** → Image finale **bien plus légère** et **plus sûre** (pas de compilateur,
pas de code source inutile en production).

Exemple (frontend) :
```dockerfile
# Étape 1 : on compile avec Node (lourd)
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build          # produit /app/dist

# Étape 2 : image finale = juste Nginx + les fichiers compilés (léger)
FROM nginx:1.27-alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```
👉 Résultat : le front fait **21 Mo** au lieu de plusieurs centaines avec Node embarqué.

---

## 4. Architecture du projet

```
TP_CI_CD/
├── backend/                  # API Python FastAPI
│   ├── app/
│   │   ├── main.py           # les routes (le CRUD)
│   │   ├── models.py         # les 2 tables (whales, containers)
│   │   ├── schemas.py        # validation des données entrantes/sortantes
│   │   └── database.py       # connexion à PostgreSQL
│   └── Dockerfile            # multi-stage
├── frontend/                 # interface React (Vite)
│   ├── src/                  # le code de l'app
│   ├── nginx.conf            # sert l'app + redirige /api vers le backend
│   └── Dockerfile            # multi-stage
├── docker-compose.yml        # lance db + backend + frontend ensemble
├── .github/workflows/ci.yml  # la pipeline
└── README.md
```

**Comment les 3 services se parlent (via Compose) :**
- `frontend` (Nginx, port 8080) reçoit le navigateur, et **renvoie les appels `/api`** au `backend`.
- `backend` (FastAPI, port 8000) lit/écrit dans la base.
- `db` (PostgreSQL) stocke les données.

---

## 5. LES COMMANDES DE BASE

### 🐳 Docker (une image / un conteneur)
```bash
docker build -t mon-image .          # construire une image depuis le Dockerfile du dossier
docker images                        # lister les images locales
docker run -p 8000:8000 mon-image    # lancer un conteneur (port machine:port conteneur)
docker ps                            # lister les conteneurs qui tournent
docker ps -a                         # … y compris ceux arrêtés
docker logs <conteneur>              # voir les logs d'un conteneur
docker exec -it <conteneur> sh       # ouvrir un terminal DANS le conteneur
docker stop <conteneur>              # arrêter
docker rm <conteneur>                # supprimer un conteneur
docker rmi <image>                   # supprimer une image
```

### 🧩 Docker Compose (toute la stack)
```bash
docker compose up                    # lancer tous les services (logs à l'écran)
docker compose up -d                 # … en arrière-plan (detached)
docker compose up --build            # reconstruire les images puis lancer
docker compose ps                    # état des services
docker compose logs -f               # suivre les logs en direct
docker compose logs -f backend       # … d'un seul service
docker compose down                  # tout arrêter et supprimer les conteneurs
docker compose down -v               # … + supprimer les volumes (efface la base !)
docker compose build                 # construire les images sans lancer
```

### 🔐 Docker Hub (le registry)
```bash
docker login -u studioflow           # se connecter (demande le token)
docker tag mon-image studioflow/mon-image:latest   # étiqueter avant push
docker push studioflow/mon-image:latest            # envoyer sur Docker Hub
docker pull studioflow/mon-image:latest            # récupérer une image
```

### 🌿 Git & GitHub
```bash
git init                             # créer un dépôt local
git status                           # voir les fichiers modifiés
git add -A                           # préparer tous les changements
git commit -m "message"              # enregistrer un instantané
git branch -M main                   # nommer la branche principale "main"
git remote add origin <url>          # relier au dépôt GitHub
git push -u origin main              # envoyer sur GitHub
git log --oneline                    # historique des commits
```

### ⚙️ GitHub Actions (la pipeline, via le CLI `gh`)
```bash
gh run list                          # lister les exécutions de pipeline
gh run view <id>                     # détail d'une exécution
gh run view <id> --log-failed        # voir uniquement ce qui a échoué
gh run rerun <id>                    # relancer une pipeline
gh secret set NOM --body "valeur"    # ajouter un secret (ex. identifiants Docker Hub)
gh secret list                       # lister les secrets configurés
```

---

## 6. La pipeline expliquée (`.github/workflows/ci.yml`)

**Déclencheur** (`on`) → **Jobs** → **Steps**. C'est la hiérarchie à retenir.

```
on: push sur main   ┐
on: tag v*          ├─► déclenche le workflow
on: pull request    ┘

job "build-and-push" (tourne sur une machine Ubuntu chez GitHub)
  step 1 : checkout            → récupère le code
  step 2 : setup buildx        → prépare l'outil de build Docker
  step 3 : login Docker Hub    → avec les SECRETS (user + token)
  step 4 : metadata            → calcule les tags (latest, sha-xxx, version)
  step 5 : build & push        → construit l'image ET l'envoie sur Docker Hub
```

Points clés pour l'exam :
- On utilise une **matrice** (`strategy: matrix`) pour faire **back ET front** avec la
  même recette, sans dupliquer le code.
- Les **secrets** (`DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`) ne sont **jamais** écrits dans
  le code : ils sont stockés chiffrés dans GitHub (Settings → Secrets).
- Le token Docker Hub doit avoir les droits **Read & Write** (sinon le `push` échoue avec
  `insufficient scopes` — l'erreur qu'on a eue).
- Sur une **pull request**, on **build sans push** (juste pour vérifier que ça compile).

**Les tags générés :**
| Tag | Quand | À quoi ça sert |
|-----|-------|----------------|
| `latest` | push sur `main` | la dernière version stable |
| `sha-18486b4` | chaque push | tracer **exactement** quel commit a produit l'image |
| `v1.2.0` / `1.2` | quand on pousse un tag git `v*` | versions officielles |

---

## 7. Le déroulé complet du TP (le workflow, étape par étape)

1. **Créer le monorepo** : un dossier `backend/` + un dossier `frontend/`.
2. **Écrire l'appli** : API FastAPI (CRUD sur 2 tables Postgres) + interface React.
3. **Écrire un Dockerfile multi-stage** pour chacun (build → image légère).
4. **Écrire `docker-compose.yml`** : 3 services (db + backend + frontend) qui se parlent.
5. **Tester en local** : `docker compose up --build` → vérifier sur http://localhost:8080.
6. **Mettre sur GitHub** : `git init` → `commit` → `push`.
7. **Écrire la pipeline** `.github/workflows/ci.yml` (build + tag + push).
8. **Configurer les secrets** Docker Hub dans GitHub (user + token Read/Write).
9. **Push** → la pipeline se lance toute seule → images publiées sur Docker Hub.
10. **Rendu** : envoyer le lien https://hub.docker.com/u/studioflow.

---

## 8. Tester l'appli en local (rappel)

```bash
docker compose up --build       # démarre tout
# Frontend  : http://localhost:8080
# API       : http://localhost:8000/api/health
# Doc API   : http://localhost:8000/docs   (Swagger auto-généré par FastAPI)
docker compose down             # arrêter
```

Exemples d'appels API (CRUD) :
```bash
curl http://localhost:8000/api/containers                      # READ (liste)
curl -X POST http://localhost:8000/api/containers \            # CREATE
  -H 'Content-Type: application/json' \
  -d '{"name":"worker","image":"redis:7","status":"running","whale_id":1}'
curl -X PUT http://localhost:8000/api/containers/1 \           # UPDATE
  -H 'Content-Type: application/json' -d '{"status":"stopped"}'
curl -X DELETE http://localhost:8000/api/containers/1          # DELETE
```
> **CRUD** = **C**reate (POST), **R**ead (GET), **U**pdate (PUT), **D**elete (DELETE).

---

## 9. Les pièges qu'on a rencontrés (et qui peuvent tomber à l'exam)

| Erreur | Cause | Solution |
|--------|-------|----------|
| `Username and password required` | secrets Docker Hub absents | ajouter `DOCKERHUB_USERNAME` + `DOCKERHUB_TOKEN` |
| `401 insufficient scopes` | token Docker Hub en lecture seule | recréer un token **Read & Write** |
| `npm ci` échoue | pas de `package-lock.json` | générer le lock avec `npm install` |
| la base se vide | `docker compose down -v` supprime le volume | utiliser `down` sans `-v` pour garder les données |

---

## 10. À retenir absolument

- **Image = recette figée**, **Conteneur = image en train de tourner**.
- **Multi-stage** = build dans une étape, image finale minimale → **plus petit, plus sûr**.
- **Compose** = lancer plusieurs services ensemble avec un fichier.
- **CI/CD** = automatiser build + publication à chaque `push`.
- **Secrets** = identifiants stockés chiffrés, jamais dans le code.
- **Tag `sha-xxx`** = tracer quel commit a produit quelle image.
