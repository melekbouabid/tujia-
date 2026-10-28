# Rapport de la session 3 (2026-10-28, après-midi et soirée) : module 2

Travail autonome sur le module 2 (AI Engineering), tâches 0 à 6 dans l'ordre. Le détail de
chaque tâche est aussi dans le « Journal de progression » du README ; les décisions sont D57
et suivantes dans docs/DECISIONS.md.

## État au départ
- `main` à jour avec GitHub (`1b7b42e`) ; CI et CD vertes.
- 74 tests backend, 279 tests pipeline, 14 tests vision ; aucune brique du module 2.
- Ollama absent de la machine ; PostgreSQL + pgvector déjà dans le compose.

## Résumé : ce qui marche

Les 7 tâches (0 à 6) sont faites, commitées et poussées (aucun force push) ; CI et CD vertes.

| | Résultat |
|---|---|
| Ollama | service Docker sur le **GPU** (RTX 2050 vue dans le conteneur) ; LLM **qwen2.5:3b** (D58), embeddings **bge-m3** (D59) |
| Corpus RAG | **8 661 chunks**, 4 002 documents : 82 notices openFDA (sections, jamais de posologie), 1 570 résumés PubMed (4 rétractés signalés), 2 350 paires DDInter ; versionné par DVC |
| Recherche | pgvector + BM25 + RRF ; **dense + filtre de principe actif retenu** : Recall@5 **0,928**, MRR **0,852** (hybride 0,904 / 0,796, BM25 0,783 / 0,660) |
| Agent | LangGraph : comprendre -> outils (appel d'outils du LLM + règles) -> récupération -> réponse citée -> contrôle de sécurité ; `POST /api/chat` JSON ou SSE |
| Génération | **aucune dose chiffrée, 0 « sûr »** (4 conseils d'ajustement de dose sans chiffre, règle étendue depuis), avertissement 100 %, questions de dose refusées 2/2 ; **94,0 %** des phrases citées soutenues par leur source (juge gemma3:4b ; contrôle : 10 % de faux oui) ; 69,8 % des phrases citées |
| RAGAS (8 questions) | fidélité 0,979 (6/8 notées), pertinence non calculable (sorties du juge illisibles), précision du contexte 0,809 (8/8 notées) |
| Temps de réponse | **4,7 s** en moyenne sur 88 questions (médiane 4,5 s ; GPU) ; ~7 s dans le site |
| MCP | serveur MCP officiel (4 outils), essayé en stdio avec un vrai client |
| Site | panneau de chat sur la page Scanner : flux, citations cliquables, outils, avertissement ; captures 08 à 11 |
| Tests | backend 74 -> **99** (dont 2 sur PostgreSQL + pgvector, dans la CI) ; pipeline 279 -> **287** ; vision 14 inchangés ; `npm run build` OK |
| MLflow | expérience **`tujya-rag`** : 6 runs de récupération, 2 de génération (sans puis avec RAGAS), 1 de contrôle du juge |

Cases cochées cette session dans « Exigences des modules » (module 2) : RAG avec documents,
métadonnées, citations et documentation ; tool calling / MCP ; LangChain / LangGraph ;
benchmarking ; interface web ; dépôt documenté avec commits réguliers. Restent : rapport
technique et démonstration.

## Démo pas à pas (module 2)

1. Docker Desktop démarré. À la racine : `docker compose up -d --build`. Au premier démarrage
   sur une autre machine, `rag-init` télécharge qwen2.5:3b et bge-m3 (~3 Go) puis indexe le
   corpus (6 min sur le GPU) ; ici, tout est déjà prêt (`rag-init` : 8 661 chunks inchangés).
   `docker compose ps` : backend et frontend « healthy », ollama « healthy ».
2. <http://localhost:8080>, `user@gmail.com` / `user`, **Scanner** -> **Demo photos** :
   cliquer « Fluconazole », confirmer ; « Citalopram », confirmer.
3. En bas, **Ask about your pills** : cliquer la suggestion « Le fluconazole et le citalopram
   interagissent-ils ? » (ou taper une question en français ou en anglais). La réponse arrive
   phrase par phrase, puis la version vérifiée ; [1] mène à la source DDInter (Major) ;
   déplier « tool call(s) » : `check_interaction` choisi par le modèle, `search_documents`
   ajouté par règle. La première question charge les modèles (10-20 s).
4. Taper « Quelle dose de citalopram dois-je prendre ? » : refus immédiat.
5. Autres bonnes questions : « What are the contraindications of metformin? », « Quel est le
   mécanisme d'action du fluconazole ? », « Has rhabdomyolysis been reported with
   olanzapine? ».
6. Montrer MLflow (<http://localhost:5000>, expérience `tujya-rag`), `docs/BENCHMARK.md`, et
   le serveur MCP : `cd backend`, `python scripts/mcp_smoke.py` (docs/MCP.md).
7. En ligne de commande : `python scripts/ask.py "..."` (backend venv) affiche outils,
   sources et réponse.

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
  (5 à ce moment, 9 à la fin de la session) ; 94 tests backend passent.

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

## Tâche 5 : benchmark (fait)
- **Jeu de 88 questions** (`benchmark/rag_questions.jsonl`, 44 FR / 44 EN) : 24 interactions,
  12 contre-indications, 10 avertissements encadrés, 10 mécanismes, 10 effets indésirables,
  8 mises en garde, 9 constats d'articles PubMed, 5 questions pièges (dose, « est-ce sûr »,
  horaire). Construit par règles à partir du corpus (chunks attendus, réponse de référence
  extraite), puis **relu à la main** : grammaire française corrigée, 2 questions remplacées
  (encadré vide, section absente), 14 références d'interaction et 9 références d'articles
  réécrites, aucune référence ne contient de dose (D66, limites dans docs/BENCHMARK.md).
- **Récupération** (83 questions, top 10) :

| Configuration | Recall@1 | Recall@3 | Recall@5 | Recall@10 | MRR |
|---|---:|---:|---:|---:|---:|
| BM25 | 0,554 | 0,711 | 0,735 | 0,783 | 0,637 |
| dense (bge-m3) | 0,783 | 0,867 | 0,904 | 0,976 | 0,841 |
| hybride (RRF) | 0,699 | 0,855 | 0,892 | 0,940 | 0,779 |
| BM25 + filtre principe actif | 0,566 | 0,747 | 0,783 | 0,819 | 0,660 |
| **dense + filtre** | **0,795** | **0,880** | **0,928** | **0,976** | **0,852** |
| hybride + filtre | 0,711 | 0,892 | 0,904 | 0,928 | 0,796 |

  BM25 échoue sur les questions françaises (Recall@5 0,59 contre 0,98 en anglais) et tire
  l'hybride vers le bas ; **dense + filtre devient le réglage par défaut** (D65).
- **Génération** (agent réel sur les 88 questions, juge gemma3:4b, D67) : aucune dose
  chiffrée, mais 4 réponses gardaient un conseil d'ajustement de dose sans chiffre recopié des
  notices (« dose adjustment of warfarin may be necessary ») : la règle de ce run ne cherchait
  que des quantités ; elle est étendue depuis (trouvé sur les conteneurs juste après) ; 0 « sûr », avertissement sur 100 % des réponses, 2/2 questions de dose refusées ; 70 % des
  phrases citées, **94,0 % des phrases citées soutenues
  par leur source** (contrôle du juge : 10 % de « oui » sur une source sans rapport) ;
  RAGAS 0.4.3 sur 8 questions : fidélité 0,979, pertinence n/a, précision du contexte 0,809. Temps de réponse moyen
  **4,7 s** (médiane 4,5 s).
- **Le benchmark a trouvé des défauts, corrigés (D68)** : avec un contexte de 8 192 jetons,
  qwen2.5:3b et bge-m3 ne tenaient plus ensemble sur le GPU (échange de modèles à chaque
  question, 12 s, une requête bloquée) -> contexte 4 096 (4,7 s) ; une réponse a bouclé sur
  plus de 24 000 jetons -> 450 jetons au plus ; le modèle oubliait les citations en français
  -> exemple dans le prompt et attribution par le sens (bge-m3) ; sur les conteneurs, une
  réponse disait « ajustement de la dose » -> la règle de dose couvre aussi les conseils sans
  chiffre, et les extraits affichés masquent les phrases avec dose.
- Juges essayés : qwen2.5:3b trop sévère (fidélité 0 sur une réponse quasi recopiée),
  llama3.2:3b sorties illisibles (NaN), gemma3:4b retenu ; RAGAS demande ~10 appels du juge
  par question (10 à 70 s chacun, gemma3 à moitié sur CPU), d'où le sous-ensemble de 8 (une question par type ; un premier essai sur 30 puis 12 a été arrêté : plus de 2 h prévues, un appel de 4 min, d'où aussi une limite de 700 jetons pour le juge).
- Tout est dans MLflow, expérience **`tujya-rag`** : 6 runs de récupération, 2 runs de
  génération (sans puis avec RAGAS), 1 run de contrôle du juge. Résultats : `docs/BENCHMARK.md`.

## Tâche 6 : chat dans le site (fait)
- Panneau « Ask about your pills » sous « My pills » sur la page Scanner
  (`frontend/src/pages/ChatPanel.tsx`) : suggestions de questions à partir des comprimés
  confirmés, réponse **en flux** (SSE lu avec `fetch`, phrases déjà vérifiées), puis la réponse
  finale vérifiée ; **citations numérotées cliquables** ([1] mène à la source, surlignée) ;
  liste des sources citées (titre, type, section, date, extrait sans phrase de dose) et des
  sources consultées non citées ; **appels d'outils discrets** (repliés : nom, choisi par le
  modèle ou ajouté par règle, résumé) ; badge quand le contrôle de sécurité a modifié ou
  refusé la réponse ; **avertissement toujours visible** en haut du panneau.
- nginx : `/api/chat` sans tampon et avec un délai de 300 s (premier chargement des modèles).
- `npm run build` passe. Pile complète : `docker compose up -d --build` (postgres, mlflow,
  ollama, pharma-init, rag-init, backend, frontend) ; `rag-init` : 8 661 chunks inchangés.
- Captures Playwright sur les conteneurs (`BASE=http://localhost:8080 node
  scripts/chat-screenshots.mjs`) : connexion, deux comprimés de démo confirmés (fluconazole,
  citalopram), question en français, réponse en ~7 s avec DDInter « Major » cité [1],
  clic sur une citation, question de dose refusée.

| Fichier | Contenu |
|---|---|
| [08-chat-empty.jpg](screenshots/08-chat-empty.jpg) | panneau vide : avertissement, suggestions sur les comprimés confirmés |
| [09-chat-answer.jpg](screenshots/09-chat-answer.jpg) | réponse réelle avec citations, appels d'outils dépliés, sources |
| [10-chat-citation.jpg](screenshots/10-chat-citation.jpg) | après un clic sur [1] : la source surlignée ; extraits sans dose ([…]) |
| [11-chat-dose-refused.jpg](screenshots/11-chat-dose-refused.jpg) | « Quelle dose de citalopram dois-je prendre ? » : refus |

- Limite visible sur la capture 09 : la réponse sur fluconazole + citalopram donne bien la
  sévérité DDInter mais ajoute des phrases sans rapport (tolvaptan, triazolam) prises dans la
  même section de notice : limite du modèle de 3B (une consigne « ne parle que des
  médicaments de la question » a été essayée, sans effet, puis retirée).

## Ce qui n'a pas marché ou reste limité
- **Qualité des réponses d'un modèle de 3B** : correctes et citées pour les
  contre-indications, mécanismes, encadrés, sévérités DDInter ; mais phrases hors sujet
  prises dans la même section de notice (capture 09 : tolvaptan, triazolam), détail inventé
  une fois (lévothyroxine), réponse « aucune interaction connue » hors sujet une fois, liste
  de sources recopiée une fois. Un modèle de 7-8B ne tient pas dans 4 Go de VRAM.
- **RAGAS** : 3 tentatives d'installation/usage (scikit-network sans roue pour Python 3.14 ->
  installé sans cette dépendance ; import cassé par langchain-community 0.4 -> module
  remplacé dans le script) ; juges qwen2.5:3b (trop sévère) et llama3.2:3b (NaN) écartés ;
  gemma3:4b trop lent pour tout le jeu : RAGAS sur 8 questions seulement (D67). Les
  vérifications de citations et de sécurité, elles, portent sur toutes les réponses.
- **BM25 sur les questions françaises** (corpus en anglais) : d'où le choix du dense ;
  une traduction de la requête pour BM25 n'a pas été faite.
- 30 % des phrases des réponses restent sans citation (surtout en français).
- Les deux premiers lancements complets de l'agent sur le benchmark se sont bloqués
  (échanges de modèles sur le GPU, puis réponse sans fin) : causes trouvées et corrigées
  (D68) ; le run retenu est le troisième.
- Toujours pas de remote DVC : le corpus est versionné mais seulement en cache local.
- La règle de sécurité est faite d'expressions régulières (français, anglais) : une dose
  écrite autrement pourrait passer ; elle retire aussi des phrases sans danger (« bien sûr »,
  « high doses increase the risk »).

## Décisions à relire (docs/DECISIONS.md)
- D57, D58 : Ollama en Docker sur le GPU ; **qwen2.5:3b** (le seul qui tient à 100 % dans
  4 Go).
- D59 : **bge-m3** via Ollama ; recherche dense exacte, sans index HNSW.
- **D60 : une notice openFDA par principe actif** (règle de choix) ; sections de posologie,
  surdosage, pharmacocinétique jamais stockées.
- D61 : 20 résumés PubMed au plus par principe actif ; D62 : DDInter limité aux paires entre
  classes dans l'index texte.
- **D63, D64 : règles de sécurité en code** (pas seulement dans le prompt) ; le LLM choisit les
  outils, le graphe ajoute ceux qui manquent.
- **D65 : recherche dense par défaut** (meilleure que l'hybride sur ce jeu) ; **D66** : jeu
  de benchmark construit et relu par la même personne ; **D67** : juge gemma3:4b, RAGAS sur
  8 questions ; **D68** : contexte 4 096 et 450 jetons max.
- Interface toujours en anglais avec les textes de sécurité en français (D53).

## Prochaines étapes
1. **Rapport technique du module 2** (les éléments sont dans docs/RAG.md, AGENT.md, MCP.md,
   BENCHMARK.md et ce rapport) et répétition de la démo.
2. Qualité des réponses : filtrer les phrases des sources qui ne citent pas les médicaments de
   la question avant de les donner au modèle ; traduire la requête en anglais pour BM25 et
   réessayer l'hybride ; essayer un modèle plus grand si une machine avec plus de VRAM est
   disponible.
3. RAGAS sur tout le jeu avec un juge plus fort (une nuit de calcul ou un GPU plus grand).
4. DVC remote (corpus RAG et images) ; changer le mot de passe admin.
5. Bonus : outils `predict_toxicity` et `predict_synergy` (point d'extension prêt).
