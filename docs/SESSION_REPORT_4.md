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
