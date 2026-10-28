# Rapport de la session 4 (nuit du 2026-10-28 au 29) : usage local, CD, DVC, démo, rapports, bonus

Travail autonome, tâches 1 à 7 dans l'ordre. Le détail de chaque tâche est aussi dans le
« Journal de progression » du README ; les décisions sont D69 et suivantes dans
docs/DECISIONS.md.

## État au départ
- `main` à jour avec GitHub (`56ada06`) ; CI et CD vertes.
- 99 tests backend, 287 tests pipeline, 14 tests vision.
- Pile Docker en marche, mais tous les ports publiés sur `0.0.0.0` et `[::]` (accessibles
  depuis le réseau local).

## Résumé : ce qui est fait

Les 7 tâches sont traitées, commitées et poussées (aucun force push, historique intact) ;
CI et CD vertes sur le dernier code testé.

| Tâche | État | En bref |
|---|---|---|
| 1. Usage strictement local | fait | tous les ports sur `127.0.0.1` (vérifié avec `netstat`), Vite et MCP HTTP en local seulement, règle permanente dans CLAUDE.md |
| 2. CD sans publication | fait, sauf suppression des paquets | images construites et testées sans envoi ; envoi seulement en option manuelle ; **suppression des 2 paquets ghcr.io à faire par vous** |
| 3. Remote DVC | fait | `C:\Users\msi\dvc-storage\tujya`, 15 306 fichiers (5,0 Go) au départ ; clone neuf + `dvc pull` identique au SHA-256 près ; case DVC cochée |
| 4. Démo et questions du jury | fait | docs/DEMO.md (15 étapes + bonus, préparation, plan B), docs/QUESTIONS_JURY.md (26 questions) ; démo rejouée en entier ; un défaut trouvé et corrigé (D72) |
| 5. Rapports techniques | fait | 2 rapports en Markdown + PDF (11 et 8 pages), schémas Mermaid, captures ; case « Rapport technique » cochée |
| 6. Bonus toxicité | fait | Tox21, LightGBM, split par scaffold, ROC-AUC test moyen 0,725 ; outil agent + MCP + API + page web ; défaut de sécurité du LLM trouvé et corrigé (D74) |
| 7. Bonus synergie | étudié, pas d'outil | règle fixée d'avance non atteinte (−7,3 % de RMSE au lieu de −10 %) : résultat documenté, aucun outil |

Chiffres principaux de la session :
- Tests : backend 99 -> **113** ; pipeline 287 et vision 14 inchangés ; tous verts en local
  et dans la CI (run 40784444929 vert, 4 jobs), CD 40784649327 verte (images construites et
  testées, `publish` ignoré).
- DVC : remote local, 15 306 + 10 fichiers (Tox21, tables `data/parquet`), synchronisé.
- MLflow : 2 nouvelles expériences, `tujya-tox` (modèle `tujya-tox21` v2) et
  `tujya-synergy` (aucun modèle enregistré).

## Où trouver quoi
- Script de démo : [docs/DEMO.md](DEMO.md) ; questions du jury : [docs/QUESTIONS_JURY.md](QUESTIONS_JURY.md).
- Rapports : [docs/rapports/rapport_module1_vision_mlops.pdf](rapports/rapport_module1_vision_mlops.pdf)
  et [docs/rapports/rapport_module2_ai_engineering.pdf](rapports/rapport_module2_ai_engineering.pdf)
  (sources `.md` à côté ; regénérer : `cd frontend && node scripts/build-reports.mjs`).
- Bonus : [docs/TOXICITY.md](TOXICITY.md), [docs/SYNERGY.md](SYNERGY.md).
- Infra locale et DVC : [docs/INFRA.md](INFRA.md) ; CD : [docs/CD.md](CD.md).
- Nouvelles captures : `docs/screenshots/12` à `17` (MLflow, page Toxicity).

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

## Tâche 6 : bonus, outil de toxicité (fait)
- **Données** : Tox21, copie MoleculeNet (`tox21.csv.gz` de DeepChem, 7 831 molécules, 12
  tests in vitro) dans `data/tox21/` avec `SOURCE.json` (URL, SHA-256), `dvc add` + `dvc push`.
  **Licence : aucune licence explicite trouvée** (ni pour le fichier MoleculeNet, ni sur la
  page du challenge Tox21) ; données publiques produites par des agences fédérales
  américaines ; traitées comme données de recherche publiques, citées, jamais commitées
  (D73). À vérifier par vous si le jury pose la question.
- **Modèle** (`backend/scripts/train_tox21.py`, `backend/app/toxicity.py`) : plus grand
  fragment, empreintes Morgan RDKit rayon 2 / 2 048 bits, **split par scaffold** 80/10/10
  (6 258 / 782 / 783 molécules, 2 285 scaffolds), un LightGBM par test avec arrêt précoce sur
  la validation. 77 s sur CPU.

| Test Tox21 | ROC-AUC test | | Test Tox21 | ROC-AUC test |
|---|---:|---|---|---:|
| NR-AR | 0,694 | | NR-PPAR-gamma | 0,658 |
| NR-AR-LBD | 0,813 | | SR-ARE | 0,647 |
| NR-AhR | 0,822 | | SR-ATAD5 | 0,720 |
| NR-Aromatase | 0,716 | | SR-HSE | 0,709 |
| NR-ER | 0,672 | | SR-MMP | 0,772 |
| NR-ER-LBD | 0,721 | | SR-p53 | 0,759 |

  **Moyenne test 0,725**, validation 0,766 ; ROC-AUC d'entraînement proche de 1 (fort
  surapprentissage) ; peu d'actifs par test (20 à 116), donc chaque chiffre est incertain.
  Aucune comparaison avec la littérature sur ce même split : je n'en revendique aucune.
- **MLflow** : expérience `tujya-tox`, run `6d3e6790…`, modèle **`tujya-tox21` v2** (pyfunc)
  publié dans `models/tox21/` (git-ignoré). La v1 est un premier run identique : le script a
  planté *après* l'enregistrement (emoji d'un message de MLflow sur la console Windows) ; je
  l'ai gardée, run marqué FAILED avec une note, plutôt que de la supprimer.
- **Structures** des 82 principes actifs depuis PubChem (`config/ingredient_smiles.csv`,
  domaine public), 82/82 ; 3 alias (valproate, vitamine B12, vitamine B6 : PubChem renvoyait
  un ion ou une coenzyme). 49 des 82 ont une empreinte identique à une molécule
  d'entraînement ; 4 sont « loin des molécules d'entraînement » (similarité < 0,3) :
  citalopram, mycophénolate mofétil, tiagabine, vitamine B12.
- **Outil** `predict_toxicity(molecules)` (1 ou 2 noms ou SMILES) : agent (choisi par le LLM
  ou ajouté par règle sur une question de toxicité), **serveur MCP (5e outil)**,
  `POST /api/toxicity`, page web **Toxicity** (une carte par molécule, 12 tests, ROC-AUC de
  chaque test, avertissement et formulation toujours visibles). Toujours « prédiction d'un
  modèle sur des tests in vitro, pas une évaluation clinique » ; jamais rien sur le mélange.
- **Problème trouvé en essayant le vrai agent, corrigé (D74)** : qwen2.5:3b lisait mal les
  probabilités : « La warfarine n'est pas identifiée comme toxique… toutes les probabilités
  sont inférieures à 0,5 » alors que SR-MMP = 0,83 ; puis « Fluconazole est classé comme
  toxique [3] » en citant la prédiction de la warfarine. Correction : quand l'outil a tourné,
  toute phrase du LLM sur la prédiction est retenue pendant le flux et remplacée par une
  **phrase fixe écrite par le code** à partir du résultat, par exemple : « Modèle Tox21 pour
  warfarin (prédiction d'un modèle sur des tests in vitro, pas une évaluation clinique) :
  probabilité d'activité d'au moins 0,5 dans 1 des 12 tests : SR-MMP (0,83) [1]. » ; et
  nouvelle règle de sécurité 6 pour toutes les réponses : jamais « non toxique », « aucune
  toxicité notable », « no significant toxicity ».
- **Vérifié sur les conteneurs** : `/api/toxicity` (warfarine + fluconazole, nom français
  résolu) ; page Toxicity par Playwright (2 molécules en 1,2 s, 12 tests chacune, formulation,
  avertissement, note sur le mélange, aucun mot interdit ; citalopram marqué hors domaine) ;
  chat : 3 questions (warfarine, fluconazole, les deux) avec réponses correctes en 3 à 5 s, et
  une question d'interaction qui n'appelle pas l'outil.
- **Tests** : backend 100 -> **113** (`test_toxicity.py` : 12 tests avec des prédicteurs
  factices et un petit vrai modèle LightGBM ; `test_safety.py` : règle « non toxique » et
  retenue en flux ; `test_mcp.py` : 5 outils listés). Image backend : RDKit, LightGBM et
  `libgomp1`. Captures 16 (page) et 17 (MLflow).

## Tâche 7 : bonus, synergie (étudiée ; pas d'outil, comme prévu par la règle)
- **Règle écrite avant tout entraînement** (D75) : ajouter `predict_synergy` seulement si,
  sur un test où des médicaments sont laissés de côté, le RMSE est au moins 10 % sous la
  baseline « moyenne » et la corrélation de Pearson au moins 0,3.
- **Dédoublonnage** : une expérience = (paire non ordonnée, lignée) ; 689 081 lignes,
  379 670 expériences, toutes dans DrugComb ; TDC en est une copie exacte (r = 1,00 sur
  Loewe et ZIP, 286 674 expériences communes) ; O'Neil diverge (r = 0,415 sur Loewe,
  22 737 communes). **Source unique : DrugComb.** Cible : **ZIP**. Médicaments avec une
  structure valide : 323 339 expériences, 287 médicaments, 110 lignées.
- **Modèle** (`backend/scripts/train_synergy.py`, 46 s) : somme des empreintes Morgan des
  deux médicaments, 30 composantes principales de l'expression de la lignée, tissu ;
  15 % des médicaments en test, 10 % en validation ; LightGBM (79 itérations).

| Test (laisser des médicaments de côté) | Moyenne : RMSE | Moyenne par lignée : RMSE (r) | LightGBM : RMSE (r) |
|---|---:|---:|---:|
| Tout le test (93 597) | 5,597 | 5,472 (0,215) | **5,189 (0,377)** |
| Un médicament inconnu (86 135) | 5,582 | 5,448 (0,221) | 5,165 (0,381) |
| Deux médicaments inconnus (7 462) | 5,777 | 5,734 (0,142) | 5,455 (0,328) |

- **Gain de RMSE : 7,3 %** (< 10 %) ; r = 0,377 (≥ 0,3) : la règle n'est pas atteinte,
  donc **aucun outil `predict_synergy`** ; je n'ai pas réglé le modèle davantage sur ce test
  (ce serait ajuster au jeu de test). MLflow `tujya-synergy`, run `77af603c…`, tag
  `clearly_better_than_mean_baseline = False`, aucun modèle enregistré.
- `data/parquet` (tables du pipeline SynergyRx) maintenant suivi par DVC et poussé (38 Mo).
- Documentation : docs/SYNERGY.md.

## Ce qui n'a pas marché ou reste limité
- **Paquets ghcr.io non supprimés** : le jeton `gh` n'a pas les droits `delete:packages`
  (403) ; je n'ai pas modifié votre connexion. Et le commit de la tâche 1 a encore déclenché
  l'ancienne CD : `sha-2222e1b` est la dernière publication. Marche à suivre ci-dessous.
- `.dvc/config` est entré dans le commit de la CD (`0621af9`) au lieu d'un commit à part
  (`git add -A` trop large) ; contenu correct, historique non réécrit.
- Régression de la tâche 1 trouvée pendant la répétition de la démo : `localhost` -> IPv6
  `::1` -> 65 à 100 s par connexion PostgreSQL ; corrigé (adresses `127.0.0.1`, D72).
- Tox21 : premier entraînement planté après l'enregistrement (emoji de MLflow sur la console
  Windows) : version 1 gardée et marquée FAILED, version 2 servie.
- Le LLM de 3B lisait mal les probabilités de toxicité et a écrit « pas identifiée comme
  toxique » : corrigé par une phrase générée par le code et une règle « jamais non toxique »
  (D74). Il ne faut donc pas montrer de réponse LLM libre sur la toxicité.
- Synergie : modèle meilleur que la moyenne mais sous la règle (−7,3 % au lieu de −10 %) :
  pas d'outil.
- Toujours limité : vraie webcam jamais testée (caméra factice seulement) ; première réponse
  du chat après un démarrage très lente (84 s mesurées) ; Swagger UI a besoin d'Internet
  (CDN) ; avertissements GitHub sur Node 20 (non bloquants) ; licence Tox21 non explicite.
- Aucune tâche abandonnée après 3 essais.

## Décisions à relire (docs/DECISIONS.md)
- **D69** : usage strictement local (remplace « déploiement sur un serveur »).
- **D70** : CD sans publication ; envoi sur ghcr.io seulement en option manuelle.
- **D71** : remote DVC local ; ce n'est **pas une sauvegarde** (même disque).
- **D72** : `127.0.0.1` au lieu de `localhost` pour les outils côté PC.
- **D73** : Tox21 sans licence explicite, traité comme donnée publique de recherche (à
  vérifier si possible avant la soutenance).
- **D74** : phrases de toxicité écrites par le code ; seuil 0,5 seulement comme aide à la
  lecture ; domaine d'applicabilité 0,3 heuristique.
- **D75** : source unique DrugComb, cible ZIP, split par médicament, règle des 10 %.

## Checklist : à faire vous-même avant la soutenance
1. [ ] **Supprimer les paquets ghcr.io** `tujya-backend` et `tujya-frontend` : github.com ->
   votre profil -> onglet **Packages** -> le paquet -> **Package settings** -> **Delete this
   package** (ou les commandes `gh` de docs/CD.md, puis retirer les droits ajoutés).
2. [ ] **Changer le mot de passe admin** (`admin@gmail.com` / `admin`) dans l'application
   (Admin -> My account) ; je n'y ai pas touché.
3. [ ] **Sauvegarder** `C:\Users\msi\dvc-storage\tujya` sur un disque externe.
4. [ ] Lire les deux rapports PDF ; ajouter ce que demande le professeur (page de garde,
   nom, établissement, date, éventuel format imposé), puis `node scripts/build-reports.mjs`.
5. [ ] Répéter docs/DEMO.md chronomètre en main ; le jour J : `docker compose up -d --build`,
   **préchauffer le LLM** (étape de préparation), vérifier Internet (Swagger).
6. [ ] Essayer la **vraie webcam** avec de vrais comprimés de la liste des 150 classes ;
   sinon le dire et montrer les photos de démo.
7. [ ] Enregistrer une **vidéo de secours** de la démo (case encore ouverte dans la feuille
   de route).
8. [ ] Relire D69 à D75 et la question 26 (bonus) des questions du jury.
9. [ ] Vérifier que `models/` contient le classifieur, le détecteur et `tox21/` (sinon
   `python -m tujya_vision.publish` et `python scripts/train_tox21.py publish`).
10. [ ] Après la soutenance : cocher « Soutenance pratique » et « Démonstration » dans le
    README.
