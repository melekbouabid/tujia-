# Rapport de la session 4 (2026-10-28, nuit) : usage local, CD, DVC, démo, rapports, bonus

Travail autonome, tâches 1 à 7 dans l'ordre. Le détail de chaque tâche est aussi dans le
« Journal de progression » du README ; les décisions sont D69 et suivantes dans
docs/DECISIONS.md.

## État au départ
- `main` à jour avec GitHub (`56ada06`) ; CI et CD vertes.
- 99 tests backend, 287 tests pipeline, 14 tests vision.
- Pile Docker en marche, mais tous les ports publiés sur `0.0.0.0` et `[::]` (accessibles
  depuis le réseau local).

## Tâche 1 : usage strictement local (fait)
- Règle permanente ajoutée dans CLAUDE.md : le projet tourne uniquement sur ce PC (pas de
  déploiement public, pas de tunnel, pas d'image Docker publiée, jamais `0.0.0.0` sur la
  machine) ; le dépôt GitHub reste public (code et docs).
- `docker-compose.yml` : les 5 ports publiés sont liés à `127.0.0.1`
  (`"127.0.0.1:${WEB_PORT:-8080}:80"`, etc.).
- `frontend/vite.config.ts` : `server.host` et `preview.host` = `127.0.0.1`, `strictPort`.
- `backend/app/mcp_server.py` : en HTTP, `--host` n'accepte que `127.0.0.1`, `localhost` ou
  `::1` (nouveau test : `0.0.0.0` et une adresse du réseau local sont refusés).
- README : uvicorn lancé avec `--host 127.0.0.1` ; nouvelle section « Usage strictement
  local ». docs/INFRA.md : section « Strictly local usage » avec la vérification. D69.
- Vérification après `docker compose up -d --build` (tous les services sains) :
  - `docker compose ps` : `127.0.0.1:8000->8000/tcp`, `127.0.0.1:8080->80/tcp`,
    `127.0.0.1:5000->5000/tcp`, `127.0.0.1:11434->11434/tcp`, `127.0.0.1:5432->5432/tcp`
    (avant : `0.0.0.0:8080->80/tcp, [::]:8080->80/tcp`) ;
  - `netstat -ano` : les 5 ports en `LISTENING` uniquement sur `127.0.0.1` ;
  - `http://localhost:8080` 200, `/docs` 200, `/api/health` 200, MLflow `/health` 200, les
    deux modèles du Model Registry listés ;
  - les mêmes ports sur les adresses du PC (192.168.1.6, 172.24.208.1, 172.22.0.1) : connexion
    refusée ;
  - Vite (`npx vite`) : `127.0.0.1:5173` seulement.
- Remarque : dans les conteneurs, uvicorn, MLflow et nginx écoutent toujours sur `0.0.0.0`,
  mais c'est l'interface privée du conteneur, joignable seulement par la redirection de port
  de Docker, elle-même limitée à `127.0.0.1`.

## Tâche 2 : CD sans publication (fait, sauf la suppression des anciens paquets)
- `.github/workflows/cd.yml` : sur une CI verte d'un push vers `main`, le job `images`
  construit les images backend et frontend du commit testé (`load: true`, `push: false`) et
  les teste (backend : `/api/health`, `/docs`, `/api/scan` et `/api/chat` dans
  `/openapi.json` ; frontend : `/` et `/scan`), ports du runner sur `127.0.0.1`. Permissions
  du workflow : `contents: read` seulement. Plus d'artefact « build record » attaché au run.
- L'envoi sur ghcr.io n'existe plus que dans un job séparé `publish`, lancé à la main
  (`gh workflow run cd -f push_images=true`), désactivé par défaut, seul job avec
  `packages: write` (D70). CI inchangée.
- Vérifié avec `gh` : CI 40768873414 verte, puis CD 40769096658 verte : « frontend image
  (build + smoke test) » 22 s, « backend image (build + smoke test) » 41 s, job `publish`
  ignoré ; résumé du run : « Built and smoke-tested at 7d0fbbd. **Not pushed** ».
- **Attention** : le commit de la tâche 1 (`2222e1b`) a encore déclenché l'ancienne CD
  (run 40768854950, avant le changement du workflow) : elle a poussé
  `tujya-backend:sha-2222e1b` et `tujya-frontend:sha-2222e1b` (et `latest`). C'est la
  dernière publication.
- **Suppression des paquets ghcr.io : impossible depuis ici.** Le jeton `gh` local a les
  droits `gist, read:org, repo, user, workflow`, sans `read:packages` ni `delete:packages`
  (`gh api -X DELETE user/packages/container/tujya-backend` -> 403). Je n'ai pas modifié
  votre connexion `gh`. À faire vous-même (2 minutes) :
  1. github.com -> votre profil -> onglet **Packages** -> `tujya-backend` -> **Package
     settings** (en bas à droite) -> **Danger zone** -> **Delete this package** -> taper le
     nom pour confirmer ;
  2. pareil pour `tujya-frontend` ;
  3. ou en ligne de commande : `gh auth refresh -h github.com -s read:packages,delete:packages`
     puis `gh api -X DELETE user/packages/container/tujya-backend` et
     `gh api -X DELETE user/packages/container/tujya-frontend` (détails dans docs/CD.md).
  Les paquets étaient privés et ne contiennent que du code (ni données, ni modèles, ni secret).
