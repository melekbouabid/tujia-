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

## Tâche 3 : remote DVC (fait)
- Remote par défaut `localstore` = `C:\Users\msi\dvc-storage\tujya` (hors du dépôt, D71) :
  `dvc remote add -d localstore C:\Users\msi\dvc-storage\tujya`. DVC 3.67.1, installé dans
  `vision/.venv`.
- Note honnête : `.dvc/config` est parti dans le commit `0621af9` (celui de la CD), parce
  que j'avais fait `git add -A` juste après `dvc remote add`. Le contenu est le bon et il est
  poussé ; je n'ai pas réécrit l'historique.
- `dvc push` : **15 306 fichiers**, **5,0 Go**, 1 min 40 ; `dvc status -c` : « Cache and
  remote 'localstore' are in sync. » Contenu : images et métadonnées C3PI, `data/tujya`
  (tables DDInter), `data/rag/raw`, `data/rag/interim`, `data/rag/manifest.json`, et les
  sorties des étapes `prepare` (étiquettes, jeux YOLO) et `rag_corpus` (chunks).
- **Preuve** : clone neuf depuis GitHub (`0621af9`) dans un dossier temporaire, `dvc pull` :
  « 15306 files fetched and 15306 files added » en 10 min 54 ; `dvc status` dans le clone :
  « Data and pipelines are up to date. » ; comparaison SHA-256 fichier par fichier avec le
  dépôt d'origine :

| Dossier | Original | Clone | Manquants | Différents |
|---|---:|---:|---:|---:|
| data/vision/c3pi/images | 4 206 | 4 206 | 0 | 0 |
| data/vision/c3pi/metadata | 117 | 117 | 0 | 0 |
| data/vision/c3pi/labels | 1 | 1 | 0 | 0 |
| data/vision/datasets | 10 731 | 10 726 | 5 | 0 |
| data/tujya | 3 | 3 | 0 | 0 |
| data/rag/raw | 246 | 246 | 0 | 0 |
| data/rag/interim | 4 | 4 | 0 | 0 |
| data/rag/manifest.json | 1 | 1 | 0 | 0 |
| data/rag/corpus | 2 | 2 | 0 | 0 |

  Les 5 fichiers manquants sont les caches d'Ultralytics (`classify/train.cache`,
  `classify/val.cache`, `detect/labels/{train,val,test}.cache`), exclus volontairement par
  `.dvcignore` et recréés à la demande. Tous les fichiers suivis (15 306) sont identiques.
  Clone temporaire supprimé ensuite.
- Documentation : README (« Données (DVC) ») et docs/INFRA.md (« DVC remote ») avec les
  commandes pour passer à Google Drive ou S3 (non configurés). Case « DVC » cochée.
- À savoir : le remote est sur le même disque que le dépôt : ce n'est pas une sauvegarde.
  Copiez `C:\Users\msi\dvc-storage\tujya` sur un disque externe.

## Tâche 4 : script de démo et questions du jury (fait)
- **docs/DEMO.md** : script de 15 étapes (≈ 14 min) : problème, architecture, scan des
  photos de démo, cas incertain, webcam, interactions, chat cité, dose refusée, Swagger,
  MLflow (vision, RAG, registre), DVC, GitHub Actions, MCP, benchmark, limites. Pour chaque
  étape : quoi cliquer ou taper, ce que le jury voit, une phrase à dire. Plus une
  préparation (dont le **préchauffage du LLM**) et un plan B.
- **docs/QUESTIONS_JURY.md** : 25 questions avec réponses courtes et chiffrées (données,
  étiquetage automatique, limites du split, ONNX, seuil 0,85, qwen2.5:3b, RAG, BM25 contre
  dense, benchmark, sécurité, licences, local, CI/CD, suite).
- **Répétition complète sur les conteneurs** (Playwright + CLI), résultats :

| Étape | Commande / script | Résultat |
|---|---|---|
| Scan, upload, démo, interactions, cas incertain, webcam, Swagger | `BASE=http://localhost:8080 node scripts/screenshots.mjs` | OK en 29 s (toutes les vérifications du script passent) |
| Chat, citation, dose refusée | `BASE=http://localhost:8080 node scripts/chat-screenshots.mjs` | OK ; 1re réponse **83,9 s** (modèles pas encore chargés après le redémarrage de la tâche 1), 2e passage **5,6 s** ; DDInter Major cité [1], une phrase retirée par le contrôle de sécurité, dose refusée |
| MLflow | nouveau `node scripts/mlflow-screenshots.mjs` | expériences `tujya-vision` (4 runs) et `tujya-rag`, registre : classifieur v2, détecteur v1 ; captures 12 à 15 |
| Agent en CLI | `scripts/ask.py` | dose refusée en 3,1 s sans LLM ; « contraindications of metformin » : réponse citée [1] (insuffisance rénale sévère, hypersensibilité, acidose métabolique) en 15 s |
| MCP | `scripts/mcp_smoke.py` | 4 outils, warfarine + fluconazole Major, notices de la metformine ; **2 min 34 s, puis 11 s après correction** (ci-dessous) |
| DVC | `dvc status`, `dvc status -c`, `dvc remote list`, `dvc dag` | à jour, synchronisé, `localstore (default)`, étapes `prepare` et `rag_corpus` |
| CI/CD | `gh run list` | CI et CD vertes |
| Swagger, MLflow, site | curl | 200 |

- **Corrigé grâce à la répétition (D72)** : depuis la tâche 1, Docker n'écoute que sur
  l'IPv4 `127.0.0.1`. Sur ce PC, `localhost` est d'abord résolu en `::1` : les navigateurs
  et curl repassent tout de suite en IPv4, mais libpq (psycopg) attendait **65 à 100 s** par
  connexion (mesuré : `127.0.0.1` 0,0 s, `localhost` 100 s). Toutes les adresses côté PC
  (URL par défaut d'Ollama et de MLflow dans le code, chaînes de connexion de la
  documentation) passent à `127.0.0.1`. Le smoke test MCP passe de 2 min 34 à 11 s. Tests
  backend et vision toujours verts.
- Ce qui reste à essayer vous-même : la **vraie webcam** avec de vrais comprimés (seule la
  caméra factice de Chromium a été testée), et le préchauffage du LLM avant la soutenance.
- Les cases « Soutenance pratique » et « Démonstration » ne sont pas cochées : elles le
  seront après la soutenance.

## Tâche 5 : rapports techniques (fait)
- `docs/rapports/rapport_module1_vision_mlops.md` et `.pdf` (11 pages, 9 figures) :
  contexte, données C3PI (sous-ensemble, étiquetage automatique et audit), détecteur,
  classifieur, bout en bout, seuil 0,85, ONNX Runtime et API, DVC avec remote, MLflow
  (tracking et registre), Docker Compose, CI/CD, application web, déploiement strictement
  local, limites, références.
- `docs/rapports/rapport_module2_ai_engineering.md` et `.pdf` (8 pages, 6 figures) :
  corpus et métadonnées, découpage et exclusions de sécurité, récupération (pgvector, BM25,
  hybride, benchmark), agent LangGraph (outils, nœud de sécurité, citations), MCP, choix du
  LLM local, benchmark (récupération, génération, RAGAS, citations, sécurité), chat,
  limites, références.
- Uniquement des chiffres présents dans docs/ et MLflow (aucun chiffre inventé).
- Schémas Mermaid (`docs/rapports/diagrams/*.mmd`) rendus en PNG (`docs/rapports/img/`) :
  architecture, chaîne de vision, graphe de l'agent ; captures de `docs/screenshots/`.
- PDF : pas de pandoc ni de LaTeX sur ce PC, donc HTML + impression Chromium (Playwright) :
  `cd frontend && node scripts/build-reports.mjs` (Mermaid et marked chargés depuis jsDelivr
  à des versions fixées : il faut Internet). Le script échoue si une image ne se charge pas.
- Vérification : les deux PDF ouverts avec PyMuPDF (environnement temporaire, hors du
  dépôt) : 11 et 8 pages, 9 et 6 images intégrées ; pages rendues et relues à l'œil
  (images visibles, légendes, tableaux, pied de page numéroté).
- Au passage : docs/RAG.md (remote DVC, mode par défaut dense) et docs/AGENT.md
  (`LLM_NUM_CTX` 4096, 450 jetons) n'étaient plus à jour, corrigés.
