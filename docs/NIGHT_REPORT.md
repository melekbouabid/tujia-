# Rapport de nuit (2026-10-27 -> 2026-10-28)

Travail autonome pendant la nuit, tâches 1 à 7 dans l'ordre. Ce fichier est mis à jour
après chaque tâche pour pouvoir reprendre le travail si la session s'arrête.

## Environnement constaté
- Dépôt dans `Proj/` (commit initial `c1cb26d`), branche `main`, remote `origin` GitHub.
- Windows 11, 16 cœurs logiques, 296 Go libres sur C:.
- GPU : NVIDIA GeForce RTX 2050, 4 Go de VRAM (pilote 610.78, CUDA 13.3).
- Docker 29.7.2, serveur Docker Desktop actif.
- Node 24.19.0, npm 12.0.2. Python 3.14.7 (le Python 3.12 « Astral » listé par `py` est introuvable).
- Le `.venv` racine vient d'une autre machine et ne démarre pas : laissé intact, nouveau `.venv-dev`.

## Budget de téléchargement
Utilisé à la fin de la tâche 3 : ~6,3 Go (échantillon 0,52 ; XML 0,13 ; DDInter 0,02 ;
images 4,8 + ~0,1 de fichiers abandonnés ; venv vision ~0,7).

## Avancement

| Tâche | État | Commit |
|---|---|---|
| 1. README et règles du projet | fait | voir journal du README |
| 2. Exploration C3PI | fait | voir journal du README |
| 3. Sous-ensemble d'entraînement | fait (DVC sans remote) | voir journal du README |
| 4. Principes actifs et interactions | fait | voir journal du README |
| 5. Infrastructure | fait | voir journal du README |
| 6. Pipeline vision | identification faite ; détection en cours | voir journal du README |
| 7. CI | à faire | |

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

### Tâche 6 : pipeline vision (identification faite, détection en cours)
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
- Doc : `docs/VISION.md` ; étape DVC `prepare` dans `dvc.yaml`.

## Problèmes rencontrés
- pandas 3.1.0 incompatible avec l'étape `build` du pipeline (D26).
- `.venv` racine cassé (D27).
- Débit Internet faible (~0,5 à 0,9 Mo/s) : c'est lui, pas le budget de 20 Go, qui limite
  la taille du sous-ensemble d'images.
- 316 fichiers listés deux fois dans les XML : sélection corrigée (comptés une fois) puis
  relancée.

## Décisions à relire
- D26 (plafond pandas), D27 (`.venv-dev`), D28 (langue des docs) dans docs/DECISIONS.md.
- D29 à D31 : métadonnées XML, classe = NDC9 + part, types d'images utilisés.
- D32 (150 classes, deux appareils exclus), D33 (split et emplacement des listes).
- D34 à D37 : RxNav, activation de DDInter par une étape Tujya, description, email de contact.
- D38 à D41 : boîtes automatiques, crops pour l'identification, PyTorch CPU puis CUDA, modèles ONNX enregistrés.
- L'identité git est « Votre Nom <vous@example.com> » (déjà dans le commit initial) : je
  ne l'ai pas changée ; à corriger avec `git config user.name/user.email` si souhaité.

## Prochaines étapes
- Tâche 6 (suite) : finir la détection, évaluation de bout en bout ; puis tâche 7 (CI).
