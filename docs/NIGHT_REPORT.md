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

## Avancement

| Tâche | État | Commit |
|---|---|---|
| 1. README et règles du projet | fait | voir journal du README |
| 2. Exploration C3PI | à faire | |
| 3. Sous-ensemble d'entraînement | à faire | |
| 4. Principes actifs et interactions | à faire | |
| 5. Infrastructure | à faire | |
| 6. Pipeline vision | à faire | |
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

## Problèmes rencontrés
- pandas 3.1.0 incompatible avec l'étape `build` du pipeline (D26).
- `.venv` racine cassé (D27).

## Décisions à relire
- D26 (plafond pandas), D27 (`.venv-dev`), D28 (langue des docs) dans docs/DECISIONS.md.

## Prochaines étapes
- Tâche 2 : télécharger `sampleData.zip` de C3PI et documenter la structure.
