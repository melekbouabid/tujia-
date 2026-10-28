# Rapport de la session 2 (2026-10-28, journée)

Travail autonome, tâches 1 à 6 dans l'ordre. Le détail de chaque tâche est aussi dans le
« Journal de progression » du README ; les décisions sont D44 à D56 dans docs/DECISIONS.md.

## État au départ
- `main` à jour avec GitHub (`b173619`, historique réécrit et poussé par vous).
- 37 tests backend, 279 tests pipeline, 14 tests vision ; CI verte.

## Résumé : ce qui marche

Les 6 tâches sont faites, commitées et poussées (aucun force push). La CI est verte sur
chaque commit testé et la CD a publié les images.

| | Résultat |
|---|---|
| Swagger | `/docs` et `/openapi.json` (l'ancienne adresse `/api/docs` redirige) |
| `POST /api/scan` | détecteur + classifieur ONNX (ONNX Runtime, CPU) : boîtes, top-5, `needs_confirmation`, temps ; ~50 à 220 ms par photo |
| Qualité du scan servi (600 photos de test) | comprimé trouvé 98,5 %, top-1 **89,3 %**, top-5 **95,8 %** (identique à Ultralytics) |
| Seuil de confirmation | **0,85**, choisi sur la validation ; en test, 70,5 % des photos acceptées sans confirmation, 99,3 % justes |
| `POST /api/interactions`, `GET /api/pills` | toutes les paires, sévérité DDInter, source et citation ; jamais « sûr » |
| Page Scanner | upload, webcam (prédictions en direct), galerie de démo, boîtes en SVG, confirmation, interactions par sévérité |
| Docker Compose | toute l'application avec `docker compose up -d --build` ; test curl et parcours Playwright sur les conteneurs OK |
| CD | images backend et frontend sur ghcr.io après une CI verte |
| Tests | backend 37 -> **74** ; pipeline 279 et vision 14 inchangés et verts ; `npm run build` OK |

Cases cochées cette session dans « Exigences des modules » : backend FastAPI asynchrone +
Pydantic + Swagger `/docs` ; frontend React (upload, webcam, prédictions, boîtes) ; Docker
Compose ; GitHub Actions CI/CD. Restent non cochées pour le module 1 : DVC (pas de remote) et
la soutenance.

## Démo pas à pas

1. Démarrer Docker Desktop. À la racine du dépôt, si `models/` est vide :
   `docker compose up -d mlflow`, puis `cd vision`, `.venv/Scripts/activate`,
   `python -m tujya_vision.publish` (copie le classifieur v2 et le détecteur v1 du Model
   Registry). Sur cette machine, `models/` est déjà rempli.
2. `docker compose up -d --build` (tout démarre : postgres, mlflow, chargement des tables,
   API, site). Vérifier avec `docker compose ps` : backend et frontend « healthy ».
3. Ouvrir <http://localhost:8080>, se connecter avec `user@gmail.com` / `user`, cliquer sur
   **Scanner**.
4. **Upload** : glisser une photo C3PI du split de test (ou une photo de comprimé) : la boîte
   et le nom apparaissent sur la photo, le top-5 à droite. Cliquer **Confirm this pill**.
5. **Demo photos** : cliquer la photo « Citalopram », confirmer ; puis « traMADOL »,
   confirmer.
6. **Check interactions** : fluconazole + citalopram et citalopram + tramadol en **Major**,
   fluconazole + tramadol en Moderate, avec la source DDInter, la citation et l'avertissement.
7. Le cas incertain : la photo « Trazodone » (65 %) affiche **Please confirm** ; choisir dans
   le top-5 ou chercher un autre comprimé (par nom, principe actif ou empreinte).
8. **Webcam** : « Start the camera », les boîtes se mettent à jour en direct ; « Capture and
   identify » fige l'image.
9. Montrer Swagger (<http://localhost:8080/docs>), MLflow (<http://localhost:5000>, modèles
   `tujya-pill-classifier` v2 et `tujya-pill-detector` v1, versions renvoyées par chaque
   scan), et l'onglet Actions de GitHub (CI puis CD).

Refaire les captures : `cd frontend`, `npx playwright install chromium` (une fois), puis
`node scripts/screenshots.mjs` (site de développement) ou
`BASE=http://localhost:8080 node scripts/screenshots.mjs` (conteneurs).

## Captures d'écran (docs/screenshots/)

| Fichier | Contenu |
|---|---|
| [01-scanner.jpg](screenshots/01-scanner.jpg) | page Scanner avant tout envoi, avertissement en haut |
| [02-upload-result.jpg](screenshots/02-upload-result.jpg) | upload d'une photo de test : boîte, « FLUCONAZOLE 100MG TABLETS 100% », top-5 |
| [03-demo-result.jpg](screenshots/03-demo-result.jpg) | photo de démo (tramadol, 99 %), sources, liste « My pills » de 3 comprimés |
| [04-interactions.jpg](screenshots/04-interactions.jpg) | 3 paires : 2 Major, 1 Moderate, sources, citations, avertissement |
| [05-needs-confirmation.jpg](screenshots/05-needs-confirmation.jpg) | trazodone à 65 % : boîte orange, « Please confirm », recherche |
| [06-webcam.jpg](screenshots/06-webcam.jpg) | webcam (caméra factice de Chromium, mire de test : 0 comprimé, 164 ms) |
| [07-swagger.jpg](screenshots/07-swagger.jpg) | Swagger UI sur `/docs` |

## Détail par tâche

### Tâche 1 : Swagger sur `/docs` (fait)
- Swagger UI sur `/docs`, schéma OpenAPI sur `/openapi.json` ; `/api/docs` redirige (307)
  vers `/docs`, `/api/openapi.json` sert le même schéma (D44).
- Proxy Vite : `/api`, `/docs`, `/openapi.json` vers le backend. `frontend/nginx.conf`
  (utilisé à la tâche 5) fait la même chose en production.
- 4 tests (`backend/tests/test_docs.py`) ; 41 tests backend passent.
- Remarque : le tableau « Journal de progression » du README ne s'affichait pas comme un
  tableau (la ligne `|---|` était au milieu, défaut de mon script de la nuit) : remis en
  ordre chronologique.

### Tâche 2 : API de scan (fait)
- `POST /api/scan` (asynchrone, connexion requise, D46) : détecteur ONNX -> crop -> classifieur
  ONNX avec ONNX Runtime, sans PyTorch ni Ultralytics dans le backend (D45). Réponse : boîte en
  pixels et normalisée, confiance de détection, top-5 (classe, nom, nom générique, principes
  actifs, probabilité), `needs_confirmation`, taille de l'image, temps d'inférence, versions
  des modèles, sources et avertissement. Limites : JPEG/PNG/WebP, 10 Mo, 50 Mpx.
- Modèles : `python -m tujya_vision.publish` copie les versions du Model Registry MLflow
  (classifieur v2, détecteur v1) dans `models/` (ignoré par git) ; le backend les lit dans
  `SYNERGYRX_MODELS_DIR` et répond 503 avec la commande à lancer s'ils manquent (D51).
- Vérifications : même résultat qu'Ultralytics sur 40 photos de test (même nombre de boîtes
  40/40, IoU 0,9999, même top-1 39/39) ; sur les 600 photos de test, top-1 **89,3 %**,
  top-5 **95,8 %** (comme l'évaluation de la nuit).
- **Seuil de confirmation 0,85** choisi sur la validation : le plus bas tel qu'au plus 1 % des
  photos soient acceptées avec un mauvais comprimé (D47, tableau dans docs/VISION.md). En test :
  70,5 % des photos acceptées sans confirmation, 99,3 % justes.
- Tests : 12 tests avec de petits modèles ONNX factices construits dans le test (aucun poids
  réel en CI). Nouvelle colonne `generic_name` : le chargeur recrée ses tables (D48) ;
  PostgreSQL rechargé (mêmes chiffres). Doc : `docs/API.md`.

### Tâche 3 : API d'interactions (fait)
- `POST /api/interactions` : 2 à 10 éléments (class_id de comprimés et/ou noms de principes
  actifs). Chaque paire : interactions DDInter (sévérité, description, source avec URL et
  citation), sévérité la plus haute, principes actifs présents dans les deux. Sans résultat :
  « Aucune interaction connue dans cette base de données. » ; élément inconnu ou comprimé sans
  principe actif connu : « aucune vérification n'est possible ». Jamais « sûr ».
- `GET /api/pills?q=&limit=` : recherche dans les 150 classes pour la confirmation manuelle.
- Tables PostgreSQL du chargeur ; 16 tests sur SQLite ; 69 tests backend au total.
- Case « Backend FastAPI asynchrone, schémas Pydantic, Swagger sur `/docs` » cochée.

### Tâche 4 : page Scanner (fait)
- Page « Scanner » dans l'application React existante (même style, comptes connectés) :
  upload (glisser-déposer), webcam (capture d'une image ; prédictions en direct sur la vidéo
  environ chaque seconde), galerie de démo. Boîtes et étiquettes dessinées en SVG sur la photo ;
  pour chaque comprimé : top-5 avec probabilités, badge « Please confirm » sous 0,85, recherche
  d'un autre comprimé (`GET /api/pills`), bouton de confirmation. Liste « My pills » sur
  plusieurs photos ; dès deux comprimés différents, vérification des interactions, affichée
  par sévérité avec sources, citations et l'avertissement.
- Galerie : 8 photos du split de test (D52), servies par `GET /api/demo/images` (lecture seule,
  uniquement des photos du split de test listées dans `config/demo_gallery.csv`), jamais
  commitées. 5 tests backend.
- Interface en anglais comme le reste du site ; textes de sécurité en français mot pour mot (D53).
- Preuve : `npm run build` passe ; `frontend/scripts/screenshots.mjs` (Playwright, D54) a fait
  tout le parcours et enregistré 7 captures dans `docs/screenshots/` (voir la fin du rapport).

### Tâche 5 : Docker Compose complet (fait)
- `docker-compose.yml` : postgres (+ pgvector), mlflow, `pharma-init` (recharge les tables de
  médicaments puis s'arrête), backend (FastAPI + ONNX Runtime CPU) et frontend (nginx : site
  React, et transmission de `/api`, `/docs`, `/openapi.json` au backend). Une seule commande :
  `docker compose up -d --build`. Images sans données ni modèles : `config/`, `data/` et
  `models/` montés en lecture seule (D55). Comptes dans PostgreSQL (jamais `backend/app.db`).
- Test de fumée par curl via nginx (port 8080), résultat brut :

```
== GET /api/health
{"status":"ok"}  -> 200
== GET /docs
  -> 200 text/html; charset=utf-8
<title>Tujya API - Swagger UI</title>
== GET /openapi.json
   Tujya API 0.2.0 18 paths; ['/api/scan', '/api/pills', '/api/interactions']
== GET /api/docs (old address)
  -> 307 location=http://localhost:8080/docs
== GET /api/public/summary
{"drugs":4081,"cell_lines":112,"synergy_measurements":689081,"abstracts":174}  -> 200
== POST /api/scan without sign-in
  -> 401
== POST /api/auth/login (demo user)
  -> 200
== POST /api/scan (C3PI test photo, class 9 fluconazole, 1225299 bytes)
   image 2560 x 1920 inference_ms 59.4 models {'detect': 1, 'classify': 2}
   box {'x1': 1109, 'y1': 846, 'x2': 1454, 'y2': 1117} top1 9 FLUCONAZOLE 100MG TABLETS ['fluconazole'] 0.996 needs_confirmation False
   disclaimer: Information uniquement, consultez un pharmacien ou un médecin.
== POST /api/interactions (pills 9 + 120)
   1 paire(s) sur 1 avec une interaction connue dans cette base de données. Major ['c3pi', 'ddinter', 'rxnav']
== GET /api/demo/images
   8 photos
== GET / (web app)
  -> 200
```

- Le script Playwright a aussi fait tout le parcours sur les conteneurs
  (`BASE=http://localhost:8080 node scripts/screenshots.mjs`).

### Tâche 6 : CD (fait)
- `.github/workflows/cd.yml` : déclenché quand le workflow `ci` réussit sur un push vers `main`
  (`workflow_run`), ou à la main. Pour le backend et le frontend : construction de l'image du
  commit testé par la CI, test de fumée du conteneur (backend : `/api/health`, `/docs`,
  `/api/scan` dans `/openapi.json` ; frontend : le site et la route `/scan`), puis envoi sur
  GitHub Container Registry avec `GITHUB_TOKEN` (`permissions: packages: write`), tags
  `latest` et `sha-<commit>` (D56). Doc : `docs/CD.md`.
- Vérifié avec `gh` : run CI 40655719906 vert, puis run CD 40655933124 vert (2 jobs). Images
  poussées (extrait du journal du run) :
  `ghcr.io/melekbouabid/tujya-backend:latest` et `:sha-44b89c7` (digest `sha256:beb99a9b…`),
  `ghcr.io/melekbouabid/tujya-frontend:latest` et `:sha-44b89c7` (digest `sha256:d69a3852…`).
- Mon `gh` local n'a pas le droit `read:packages` : je n'ai pas pu lister les paquets par
  l'API (je n'ai pas modifié votre connexion `gh`). Pour vérifier : onglet Packages de votre
  profil GitHub, ou `gh auth refresh -s read:packages` puis la commande de `docs/CD.md`.
- Les paquets sont **privés** par défaut (à rendre publics dans leurs réglages si besoin).

## Ce qui n'a pas marché ou reste limité
- Aucune tâche abandonnée.
- La webcam n'a été testée qu'avec la caméra factice de Chromium (pas de vraie caméra ni de
  vrai comprimé ici) : à essayer avant la soutenance, avec un fond uni et une bonne lumière.
  Les modèles n'ont vu que des photos C3PI : sur une image de webcam, les scores seront plus
  bas.
- Je n'ai pas pu lister les paquets ghcr.io par l'API (droit `read:packages` absent de `gh`) ;
  l'envoi est prouvé par le journal du run CD.
- Swagger UI charge ses fichiers JS/CSS depuis un CDN : sans Internet, `/docs` s'affiche vide.
- Un nouveau clone ne peut pas lancer la démo sans les données : DVC n'a toujours pas de remote.
- Avertissements GitHub : `actions/checkout@v4` et les actions `docker/*` ciblent Node 20
  (déprécié, forcé sur Node 24) ; à passer aux versions suivantes un jour.

## Problèmes rencontrés
- Tâche 2 : la première évaluation sur la validation était 30 fois trop lente (plus de 25 min) :
  les threads d'ONNX Runtime restent en attente active après chaque inférence et privent OpenCV
  de CPU. Attente active désactivée (D49) : 3 min pour 600 photos.
- Tâche 4 : une classe CSS `.source-list` existait déjà (page d'accueil) et cassait la mise en
  page des sources : renommée `.cited-sources`. Les premières captures montraient les cartes à
  moitié transparentes (animation d'entrée) : le script attend la fin des animations.
- Tâche 2 : les modèles téléchargés depuis le Model Registry ne faisaient que 60 à 140 Ko :
  MLflow garde les poids dans un fichier séparé ; `publish` les réunit en un seul fichier.
- Tâche 6 : le run CI du commit de la tâche 5 a été annulé par GitHub (un nouveau push a
  démarré un run plus récent, `cancel-in-progress`), donc le run CD qu'il a déclenché a été
  ignoré, comme prévu ; le commit suivant, qui contient le même code, est vert.

## Décisions à relire (docs/DECISIONS.md)
- D44 : Swagger sur `/docs`, ancienne adresse redirigée ; fichiers JS/CSS de Swagger UI sur un
  CDN (démo hors ligne : à servir localement).
- D45 : le backend refait le pré/post-traitement d'Ultralytics en numpy/OpenCV (pas de PyTorch
  dans l'image) ; toute modification du recadrage à l'entraînement doit y être reportée.
- D46 : scan, recherche et interactions réservés aux comptes connectés ; la démo utilise
  `user@gmail.com`. Pas de route publique.
- **D47 : seuil de confirmation 0,85** (réglable par `SYNERGYRX_SCAN_CONFIRM_THRESHOLD`).
- D48 : le chargeur supprime et recrée ses 5 tables à chaque chargement.
- **D50 : les noms de produits C3PI contiennent le dosage du comprimé** (« AMRIX CAP 30MG ») ;
  affichés tels quels pour l'identification, ce n'est pas une posologie. À enlever si le jury
  le lit comme une dose.
- **D52 : la galerie de démo est choisie à la main** parmi les photos de test, à l'aide des
  résultats du modèle (elle le montre à son meilleur, sauf la photo « à confirmer ») ; les
  vrais chiffres sont ceux des 600 photos de test.
- **D53 : interface en anglais, textes de sécurité en français mot pour mot.** Une interface
  entièrement en français serait plus cohérente pour la soutenance.
- D55 : en Docker, les comptes sont dans PostgreSQL ; le compte admin par défaut
  (admin@gmail.com / admin) y est créé au premier démarrage : mot de passe toujours à changer
  (je n'y ai pas touché).
- D56 : CD = images sur ghcr.io, pas de déploiement (aucun serveur) ; paquets privés par défaut.

## Prochaines étapes
1. **DVC remote** (Google Drive, S3 ou disque) puis `dvc push` : nécessaire pour qu'un autre
   poste lance la démo, et pour cocher la case DVC.
2. Essayer la webcam avec de vrais comprimés ; si besoin, servir Swagger UI sans CDN.
3. Au choix : interface en français ; changer le mot de passe admin.
4. **Module 2** (prochaine grande phase) :
   - corpus RAG : notices openFDA des 82 principes actifs (+ résumés PubMed), découpage avec
     métadonnées (source, médicament, section, URL, date), versionné avec DVC ;
   - embeddings bge-m3 dans pgvector (déjà dans le compose) + BM25, recherche hybride ;
   - agent LangGraph : comprendre -> outils -> récupérer -> répondre avec citations ->
     contrôle de sécurité ; outils `identify_pills` (= `/api/scan`), `check_interaction`
     (= `/api/interactions`), `get_drug_info`, `search_literature` ; Ollama dans le compose ;
   - serveur MCP exposant les mêmes outils ;
   - benchmark : 50 à 100 questions, Recall@k / MRR pour la récupération, RAGAS pour la
     génération ; panneau de chat avec citations dans le site.
