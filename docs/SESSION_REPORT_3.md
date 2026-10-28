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

## Tâche 1 : corpus documentaire (fait)
- **Notices openFDA** : connecteur `openfda_label` (`src/synergyrx_data/connectors/openfda.py`)
  dans le style du pipeline (PoliteClient, 3 requêtes/s, cache HTTP, manifeste avec SHA-256,
  colonnes de provenance, schéma Pydantic). Une notice par principe actif, choisie par une
  règle fixe (D60) : notice qui concerne bien le principe actif, mono-principe actif avant
  association, sur ordonnance avant OTC, le plus de sections utiles, la plus récente. Les 25
  candidates sont gardées avec la notice pour pouvoir vérifier le choix.
  Résultat : **82/82** (81 sur ordonnance, 1 OTC : fexofénadine), 975 sections, dates
  2021-09-30 à 2026-09-11.
- **Jamais de posologie** : `dosage_and_administration`, `dosage_forms_and_strengths`,
  `overdosage`, `how_supplied`, emballage, pharmacocinétique, études cliniques sont retirés
  avant l'écriture du fichier brut, refusés par le schéma et par le découpage.
- **PubMed** : le connecteur existant a un nouveau mode `ingredients` (une requête par
  principe actif : interactions ou effets indésirables, résumés en anglais depuis 2000, tri
  par pertinence, **20 au plus**, D61). 1 570 articles distincts (17 à 20 par principe actif),
  dont 4 rétractés (signalés, exclus à la recherche par défaut).
- **DDInter** : les 2 350 paires entre principes actifs des classes, une par chunk (D62).
- **Découpage** : notices par section (1 500 caractères au plus, une phrase de
  chevauchement), résumés par paragraphe (BACKGROUND, RESULTS...), DDInter une ligne par
  paire. Métadonnées : type, principe(s) actif(s), principes actifs cités, section, titre,
  source, URL (DailyMed, PubMed, DDInter), date, version, licence, type d'article, rétraction,
  langue, date de récupération.
- **Total : 8 661 chunks pour 4 002 documents** (notices 2 895, résumés 3 416, DDInter 2 350).
- **DVC** : `data/rag/raw`, `data/rag/interim`, `data/rag/manifest.json` par `dvc add` ;
  chunks par l'étape `rag_corpus` de `dvc.yaml` (`dvc repro rag_corpus`). Toujours pas de
  remote DVC (cache local).
- Documentation : `docs/RAG.md`. Tests : 8 (`tests/unit/test_tujya_rag.py`, respx) ; 287
  tests du pipeline passent, ruff et mypy propres.
- Limite : d'autres sections des notices citent des quantités (1 010 chunks contiennent
  « <nombre> mg », par exemple les doses des études) ; le nœud de sécurité de l'agent retire
  toute phrase avec une dose.

## Tâche 2 : récupération (fait)
- Machine : 31,7 Go de RAM, i7-13620H (16 threads), RTX 2050 4 Go. Modèle d'embeddings :
  **bge-m3** (multilingue, 1 024 dimensions) servi par le même conteneur Ollama (D59) :
  0,7 Go de VRAM, tient avec qwen2.5:3b (les deux à 100 % GPU), aucun PyTorch dans le backend.
  Une phrase française et sa traduction anglaise : cosinus 0,87 ; phrase sans rapport : 0,53.
- Stockage : table `rag_chunks` (PostgreSQL + pgvector, `vector(1024)`), métadonnées en
  colonnes, listes de principes actifs en `text[]` ; table `rag_index_meta` (modèle, dimension,
  date d'indexation).
- Indexation idempotente : `python -m app.rag.index` ; 8 661 chunks en **364 s** ; deuxième
  passage : 0 recalculé, 0,1 s. Un chunk modifié est recalculé, un chunk disparu supprimé, un
  autre modèle reconstruit la table.
- Recherche : BM25 (en mémoire, ~6 ms), dense (cosinus exact en SQL, 130-190 ms, surtout
  l'embedding de la question), hybride RRF k = 60 (~170 ms). Filtres : principe actif (sujet
  ou cité), type de document, section, articles rétractés exclus par défaut.
- Tests : `test_rag.py` (SQLite, embeddings factices) et `test_rag_pgvector.py` (PostgreSQL ;
  la CI a maintenant un service `pgvector/pgvector:pg16` dans le job backend).

## Tâche 3 : agent LangGraph et `/api/chat` (fait)
- **Outils** (`backend/app/rag/tools.py`, outils LangChain `StructuredTool`, mêmes fonctions
  que le serveur MCP) : `identify_pills` (la logique de `/api/scan`), `check_interaction`
  (celle de `/api/interactions`, extraite dans `interactions_report` sans changer la route),
  `get_drug_info` (classes de comprimés + extraits de la notice), `search_documents`
  (recherche hybride avec filtres). Point d'extension documenté pour `predict_toxicity`
  (Tox21 / ClinTox) et `predict_synergy` (DrugComb), non implémentés, ajoutés en Bonus dans
  la feuille de route.
- **Graphe LangGraph** : `understand` (noms de médicaments en français ou en anglais,
  comprimés confirmés, langue, question de dose) -> `tools` (le LLM choisit par appel
  d'outils Ollama ; le graphe ajoute `check_interaction` pour deux médicaments ou plus et
  `search_documents` toujours, D64) -> `retrieve` (sources numérotées) -> `answer` (réponse
  uniquement à partir des sources, citations [n], en streaming) -> `safety` (D63).
- **Contrôle de sécurité** : phrases avec dose retirées (sauf valeurs biologiques comme
  « mL/min »), phrases « sûr / sans danger / safe » retirées et remplacées par « aucune
  interaction connue dans cette base de données » si rien n'est trouvé, conseils d'horaire
  non sourcés retirés, citations hors limites retirées, phrase non citée rattachée à la source
  qui contient au moins 60 % de ses mots, aucune source = pas de réponse, avertissement
  toujours à la fin ; question de dose refusée sans appeler le LLM. En streaming, les phrases
  interdites ne sont jamais affichées.
- **`POST /api/chat`** (asynchrone, connexion requise) : `answer`, `citations` (titre, URL,
  section, extrait, `cited`), `tool_calls` (nom, arguments, `chosen_by` llm/rule, résumé),
  `safety`, `timings` ; `stream: true` -> SSE `tool`, `token`, `final`. URL d'Ollama par
  `SYNERGYRX_OLLAMA_URL` (`http://ollama:11434` dans le compose) ; nouveau service ponctuel
  `rag-init` (télécharge les modèles si besoin puis indexe).
- **Essais réels** (qwen2.5:3b + bge-m3, GPU) : « La warfarine interagit-elle avec le
  fluconazole ? » -> le LLM a lui-même appelé `check_interaction`, réponse avec DDInter
  (Major) cité [1] ; « What are the contraindications of metformin? » -> réponse correcte
  (insuffisance rénale sévère, hypersensibilité, acidose métabolique) citée [1]. Environ
  13 à 15 s par question (planificateur ~7 s, réponse ~6 s). Deux corrections après ces
  essais : « eGFR 30 mL/min » n'est plus pris pour une dose ; une phrase recopiée d'une notice
  sans citation est rattachée à cette notice.
- Tests : `test_chat.py` (8, LLM factice scripté, embeddings factices) et `test_safety.py`
  (9) ; 94 tests backend passent.

## Tâche 4 : serveur MCP (fait)
- `backend/app/mcp_server.py` : SDK Python officiel (`mcp` 2.3 ; dans cette version FastMCP
  s'appelle `MCPServer`), mêmes 4 outils que l'agent (mêmes fonctions de `Toolbox`), consignes
  de sécurité dans les `instructions` du serveur. Lancement : `python -m app.mcp_server`
  (stdio) ou `--transport streamable-http --port 8765`.
- Essai réel : `scripts/mcp_smoke.py` démarre le serveur en sous-processus (stdio), liste les
  outils et en appelle deux sur PostgreSQL : warfarine + fluconazole -> Major ; recherche
  « metformin lactic acidosis » -> notices de la metformine.
- Documentation de connexion (Claude Desktop, Claude Code, client HTTP) : `docs/MCP.md`.
- Tests : `test_mcp.py` (client MCP en mémoire) : liste des outils et schémas, deux appels, une
  erreur propre pour une image absente.
