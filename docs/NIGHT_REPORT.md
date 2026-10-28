# Rapport de nuit (2026-10-27 -> 2026-10-28)

Travail autonome pendant la nuit, tâches 1 à 7 dans l'ordre. Le détail de chaque tâche
est aussi dans le « Journal de progression » du README.

## Environnement constaté
- Dépôt dans `Proj/` (commit initial `c1cb26d`), branche `main`, remote `origin` GitHub.
- Windows 11, 16 cœurs logiques, 296 Go libres sur C:.
- GPU : NVIDIA GeForce RTX 2050, 4 Go de VRAM (pilote 610.78, CUDA 13.3).
- Docker 29.7.2, serveur Docker Desktop actif.
- Node 24.19.0, npm 12.0.2. Python 3.14.7 (le Python 3.12 « Astral » listé par `py` est introuvable).
- Le `.venv` racine vient d'une autre machine et ne démarre pas : laissé intact, nouveau `.venv-dev`.

## Résumé (à lire en premier)

Les 7 tâches sont faites. Tout est commité et poussé sur `main` (aucun force push).

| Chiffre | Valeur |
|---|---|
| Classes de comprimés (C3PI) | 150 (24 à 30 photos consumer chacune) |
| Images téléchargées | 4 206 fichiers, 4,8 Go, 0 échec, suivies par DVC |
| Photos étiquetées automatiquement (boîtes) | 3 306 / 4 056 (81,5 %), précision ~90 % (audit de 48) |
| Identification (YOLO11s-cls) | test top-1 **93,5 %**, top-5 **98,1 %** (589 crops) |
| Détection (YOLO11n) | test mAP50 **0,972**, mAP50-95 **0,861** |
| Bout en bout, ONNX (600 photos de test) | détecté 98,5 %, top-1 **89,2 %**, top-5 **95,8 %** |
| Classes avec principe(s) actif(s) (RxNav) | 149 / 150 (82 principes actifs) |
| Paires d'interactions DDInter entre ces principes actifs | 2 350 (125 Major) ; 40 508 avec n'importe quel médicament |
| Tests | 279 pipeline + 37 backend + 14 vision, tous verts ; CI GitHub verte |
| Téléchargement total | ~9 Go sur 20 Go autorisés |

Attention en lisant les scores : train et test contiennent des photos différentes des
mêmes comprimés physiques, et les boîtes de référence sont automatiques (docs/VISION.md).

Cases cochées dans « Exigences des modules » : détection YOLO, identification par
transfer learning, MLflow tracking, MLflow Model Registry, ONNX + ONNX Runtime. Non
cochées (partielles) : DVC (pas de remote), Docker Compose (pas de backend/frontend),
CI/CD (CI seulement), dépôt GitHub documenté (à juger par vous).

## Budget de téléchargement
~9 Go au total : échantillon 0,52 ; XML 0,13 ; DDInter 0,02 ; images 4,8 (+ ~0,1 de fichiers
d'une première sélection, supprimés) ; venv vision ~0,7 ; PyTorch CUDA 2,0 ;
onnxruntime-gpu (installé tout seul par Ultralytics, D42) ~0,3 ; images Docker ~0,3 ;
poids YOLO ~0,05. Le facteur limitant a été le débit (~0,5 à 1,5 Mo/s), pas le budget.

## Avancement

| Tâche | État |
|---|---|
| 1. README et règles du projet | fait |
| 2. Exploration C3PI | fait |
| 3. Sous-ensemble d'entraînement | fait (DVC sans remote) |
| 4. Principes actifs et interactions | fait |
| 5. Infrastructure | fait (postgres + MLflow ; backend/frontend pas encore dans le compose) |
| 6. Pipeline vision | fait (identification, détection, MLflow, registry, ONNX, bout en bout) |
| 7. CI | fait (4 jobs verts sur GitHub) |

## Détail par tâche

### Tâche 1 : README et règles du projet (fait)
- CLAUDE.md : section « Mandatory rules for every session » ; nouvelle orientation données
  (C3PI, pas de photos personnelles, détection puis identification, ePillID, openFDA,
  démo sur les images de test).
- README.md : présentation en français, « Exigences des modules » (aucune case cochée),
  « Sources de données », « Journal de progression » ; l'ancien contenu est sous
  « Annexe : pipeline de données SynergyRx ».
- docs/ROADMAP.md réécrit pour la nouvelle orientation.
- Tests de référence : 270 tests unitaires du pipeline et 34 tests backend passent.
- Problème : pandas 3.1.0 casse 3 tests de bout en bout ; plafonné à `<3.1` (D26).

### Tâche 2 : exploration de C3PI (fait)
- `sampleData.zip` (517 Mo) téléchargé et décompressé dans `data/vision/c3pi/sampleData/` :
  un seul comprimé (fluoxétine 10 mg) ; son dossier `consumer/` ne contient en fait aucune
  photo consumer, seulement des images de référence.
- Les 110 XML de `ALLXML/` (~130 Mo) donnent pour chaque image : NDC, nom générique, nom
  commercial, taille, SHA-1, type, et pour les photos consumer l'appareil, le fond, l'éclairage.
- 40 587 photos consumer (JPG) de 2 400 comprimés ; 380 comprimés ont au moins 20 photos.
- Pas de bounding box ni de masque pour les photos consumer (un comprimé centré sur fond
  simple) ; les images de référence segmentées ont un masque alpha.
- Pas de RxCUI dans les métadonnées : passage par RxNav pour les principes actifs.
- Documentation : `docs/DATASET_C3PI.md`.

### Tâche 3 : sous-ensemble d'entraînement (fait, DVC local)
- 150 classes, 24 à 30 photos consumer chacune (moyenne 27), + 1 image de référence RxNav
  par NDC : 4 206 fichiers, 4,8 Go, tous vérifiés par SHA-1, 0 échec.
- Les appareils Canon G10 et Nikon S3000 sont exclus (fichiers de ~4,5 Mo) : sans cela,
  200 classes auraient pesé 17,8 Go, impossible avec le débit de la nuit (D32).
- Split par photo 70/15/15 stratifié par classe (3 006 / 600 / 600), liste dans
  `config/splits/selected_images.csv` (versionnée par git, D33).
- `config/pill_classes.csv` : class_id, NDC9, liste des NDC11, nom, nom générique, labeler,
  empreinte, couleur, forme, nombres d'images, et (tâche 4) RxCUI et principes actifs.
- DVC initialisé ; `data/vision/c3pi/images` et `metadata` suivis (cache local). Aucun
  remote configuré : à choisir (Google Drive, S3, disque externe) puis `dvc push`.

### Tâche 4 : principes actifs et interactions (fait)
- RxNav : NDC11 -> RxCUI (`ndcstatus`, y compris NDC obsolètes) -> ingrédients de base
  (`historystatus`). 149/150 classes associées, 82 principes actifs, 11 associations
  (ex. losartan + hydrochlorothiazide). Non associée : la multivitamine « Formula B Plus ».
  Les dosages sont présents dans la réponse RxNav mais ne sont jamais stockés.
- DDInter : le connecteur existant ne prenait que 8 des 14 fichiers ATC (il manquait C
  cardiovasculaire et N système nerveux) ; l'étape Tujya prend les 14 : 234 981 lignes.
- 3 alias RxNorm -> DDInter (valproate, vitamines B6 et B12) : 82/82 trouvés.
- Tables : `data/tujya/interactions_between_classes.csv` (2 350 paires : 125 Major,
  826 Moderate, 72 Minor, 1 327 Unknown) et `interactions_with_any_drug.csv` (40 508).
- La colonne `description` ne fait que reformuler DDInter (les fichiers en masse n'ont pas
  de texte de mécanisme ; pas de scraping) : les explications viendront du RAG openFDA.
- Activation : `python -m synergyrx_data.tujya ddinter` ; le `config.yaml` partagé garde
  DDInter désactivé car deux tests existants l'exigent (je ne les ai pas modifiés, D35).
- Doc : `docs/INTERACTIONS.md`. 5 nouveaux tests ; 279 tests du pipeline passent.

### Tâche 5 : infrastructure (fait)
- Docker tournait : `docker compose up -d --build` fonctionne. PostgreSQL 16 + pgvector 0.8.7
  (base `tujya`) et MLflow 3.17.2 (store dans la base `mlflow`, artefacts dans le volume
  `mlartifacts`) sur http://localhost:5000.
- Schéma SQLAlchemy `backend/app/pharma.py` (sources, pills, ingredients, pill_ingredients,
  interactions ; paire stockée une seule fois, contrainte a < b), base séparée de `app.db`.
- Chargeur `python -m app.pharma_loader` : rechargement complet en une transaction.
  Dans PostgreSQL : 150 comprimés, 1 716 ingrédients, 161 liens, 40 508 interactions.
- 3 tests ajoutés (SQLite) ; 37 tests backend passent. Ajout de `psycopg[binary]`.
- Doc : `docs/INFRA.md`. Backend et frontend ne sont pas encore dans le compose.

### Tâche 6 : pipeline vision (fait)
- Matériel : RTX 2050 4 Go. Le venv a d'abord reçu PyTorch CPU (bande passante réservée aux
  images), puis PyTorch 2.14.1 CUDA 13.0 (2 Go) une fois les images téléchargées (D40).
- Boîtes : aucune annotation dans C3PI, donc étiquetage automatique des photos consumer
  (D38) : 3 306/4 056 photos propres (81,5 %), précision ~90 % sur un audit visuel de 48
  photos. Les photos douteuses sont exclues des deux jeux de données.
- Identification : YOLO11s-cls sur des crops de 224 px (D39), 40 époques GPU (14 min) :
  **test top-1 93,5 %, top-5 98,1 %** sur 589 crops. Attention : mêmes comprimés physiques
  photographiés plusieurs fois entre train et test, crops issus des boîtes automatiques.
- MLflow : expérience `tujya-vision`, modèle `tujya-pill-classifier` v1 (essai CPU) et v2.
- ONNX : export + vérification ONNX Runtime vs PyTorch (100 % d'accord top-1).
- Détection : YOLO11n, 30 époques GPU sur les boîtes automatiques : **mAP50 0,972,
  mAP50-95 0,861** (précision 0,957, rappel 0,936) sur 483 photos de test, mesurés par
  rapport aux boîtes automatiques. `tujya-pill-detector` v1 dans MLflow.
- Bout en bout (détecteur -> crop -> classifieur, les deux en ONNX) sur les 600 photos de
  test, y compris celles que l'auto-étiquetage avait rejetées : comprimé trouvé sur
  98,5 %, **top-1 89,2 %, top-5 95,8 %**.
- Doc : `docs/VISION.md` ; étape DVC `prepare` (`dvc.yaml`, `dvc.lock`).

### Tâche 7 : CI (fait)
- `.github/workflows/ci.yml` : 4 jobs (pipeline : ruff, format, mypy, tests unitaires ;
  backend : ruff, format, pytest ; vision : ruff, format, tests sans entraînement ;
  frontend : `npm ci` + build). Aucun test réseau en direct.
- Vérifié en local, puis sur GitHub Actions : run 40616884118, **4 jobs verts** du premier
  coup.

## Problèmes rencontrés
- pandas 3.1.0 casse 3 tests de bout en bout du pipeline : plafonné à `<3.1` (D26).
- `.venv` racine créé sur une autre machine, inutilisable : laissé intact, `.venv-dev` (D27).
- Débit Internet faible : sous-ensemble limité à 150 classes et à 7 des 9 appareils (D32).
- 316 fichiers listés deux fois dans les XML : sélection corrigée puis relancée ; 38 images
  de la première sélection (téléchargées par moi) supprimées.
- Activer DDInter dans `config.yaml` cassait 2 tests existants qui imposent qu'il soit
  désactivé par défaut : config partagée inchangée, activation par l'étape Tujya (D35).
- Fins de ligne mélangées dans le dépôt (CRLF/LF) ; `sed` avait converti `pyproject.toml`
  en LF dans le commit de la tâche 1 (diff du fichier entier), corrigé au commit suivant ;
  règle ajoutée dans CLAUDE.md.
- Auto-étiquetage : premières versions mauvaises (vignettage, pièces de monnaie, ombres) ;
  4 itérations avec contrôle visuel avant le passage complet.
- Ultralytics a installé `onnxruntime-gpu` de lui-même pendant l'évaluation (D42) :
  auto-installation désactivée ensuite. Les deux runtimes sont dans le venv vision.
- Aucune tâche abandonnée.

## Décisions à relire (docs/DECISIONS.md)
- D26 plafond pandas ; D27 `.venv-dev` ; D28 langue des docs.
- D29-D31 métadonnées XML, classe = NDC9 + part, types d'images utilisés.
- **D32** 150 classes et exclusion des appareils Canon G10 / Nikon S3000 (pour 200
  classes : relancer `select` sans `--exclude-camera`, ~18 Go).
- D33 split par photo (mêmes comprimés physiques en train et en test).
- D34 RxNav ; **D35** DDInter activé par `python -m synergyrx_data.tujya ddinter` et non
  dans `config.yaml` ; **D36** description DDInter minimale ; **D37** appels RxNav/DDInter
  sans email de contact (pas de `.env`, je n'ai pas utilisé votre adresse).
- **D38** boîtes de détection automatiques (pas d'annotation manuelle) ; D39 crops ; D40
  PyTorch CUDA ; D41 modèles ONNX enregistrés ; D42 auto-install ; D43 étape DVC.
- L'identité git est « Votre Nom <vous@example.com> » (déjà dans le commit initial) : je ne
  l'ai pas changée. À corriger avec `git config user.name` / `user.email` si souhaité.
- Les conteneurs `tujya-postgres-1` et `tujya-mlflow-1` tournent encore (`docker compose
  down` pour les arrêter ; les volumes sont conservés).
- Le mot de passe admin par défaut (admin@gmail.com / admin) n'a pas été changé.

## Prochaines étapes (dans l'ordre conseillé)
1. **DVC remote** : choisir un stockage (Google Drive, S3, disque externe), puis
   `dvc remote add -d ...` et `dvc push`, et cocher la case DVC.
2. **API** : `POST /api/scan` (ONNX Runtime : détecteur + classifieur, top-5 et confiance,
   demande de confirmation sous un seuil) et `POST /api/interactions` sur les tables
   PostgreSQL. Swagger : l'app sert la doc sur `/api/docs`, l'exigence dit `/docs` (à
   aligner ou à justifier).
3. **Frontend** : page Scan (upload, webcam, boîtes) avec les images de test C3PI.
4. **Docker Compose** : ajouter backend et frontend (puis Ollama) ; **CD** dans GitHub Actions.
5. **Qualité des boîtes** : vérifier à la main les boîtes du split test, ou refaire un
   passage d'étiquetage avec le détecteur entraîné ; mesurer l'identification sur d'autres
   exemplaires physiques ; comparaison honnête avec ePillID sur son propre split.
6. **Module 2** : corpus RAG openFDA (notices des 82 principes actifs), pgvector + BM25,
   agent LangGraph avec outils (`identify_pills`, `check_interaction`...), benchmark.

## Reprendre le travail
```bash
docker compose up -d                       # PostgreSQL + MLflow (http://localhost:5000)
vision/.venv/Scripts/activate              # venv vision (GPU)
dvc status                                 # données et étape prepare à jour
cd vision && python -m tujya_vision.train classify --weights yolo11s-cls.pt --epochs 40
```
Modèles : `vision/runs/classify-20261028-021601/weights/best.onnx` et
`vision/runs/detect-20261028-023214/weights/best.onnx` (aussi dans le Model Registry).
