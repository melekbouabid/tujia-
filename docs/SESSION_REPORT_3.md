# Rapport de la session 3 (2026-10-28, après-midi et soirée) : module 2

Travail autonome sur le module 2 (AI Engineering), tâches 0 à 6 dans l'ordre. Le détail de
chaque tâche est aussi dans le « Journal de progression » du README ; les décisions sont D57
et suivantes dans docs/DECISIONS.md.

## État au départ
- `main` à jour avec GitHub (`1b7b42e`) ; CI et CD vertes.
- 74 tests backend, 279 tests pipeline, 14 tests vision ; aucune brique du module 2.
- Ollama absent de la machine ; PostgreSQL + pgvector déjà dans le compose.

## Petite correction préalable
- docs/VISION.md : le schéma du pipeline indiquait YOLO11n-cls pour l'identification ; le
  modèle final est YOLO11s-cls (corrigé, commit `94304d9`).

## Tâche 0 : Ollama (fait)
- Service `ollama` dans `docker-compose.yml` : image officielle `ollama/ollama:0.40.12`,
  volume nommé `ollama` pour les modèles, port 11434, GPU NVIDIA réservé par
  `deploy.resources.reservations.devices` (D57). Variante sans GPU :
  `docker compose -f docker-compose.yml -f docker/ollama/cpu.yml up -d`.
- **Le GPU marche dans Docker** : `docker compose exec ollama nvidia-smi` montre la RTX 2050
  (4096 Mio) ; journal d'Ollama : `library=CUDA compute=8.6 ... total="4.0 GiB"
  available="3.2 GiB"`.
- Choix du modèle (D58), même question courte en français, température 0, modèle chargé :

| Modèle | VRAM | Placement | Vitesse | Temps de réponse |
|---|---:|---|---:|---:|
| **qwen2.5:3b** | 2,2 Go | 100 % GPU | 47,4 jetons/s | **1,7 s** |
| llama3.2:3b | 2,9 Go | 80 % GPU | 32,0 jetons/s | 3,2 s |
| gemma3:4b | 2,9 Go | 55 % GPU | 11,8 jetons/s | 12,1 s |
| qwen3:4b | 2,9 Go | 82 % GPU | 14,0 jetons/s | 89,9 s (raisonnement en anglais) |

  qwen2.5:3b est le seul qui tient entièrement dans les 4 Go, le plus rapide, avec un bon
  français et l'appel d'outils dans Ollama.
