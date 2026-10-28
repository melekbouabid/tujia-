# Tujya: project guide for Claude sessions

Solo student project. Tujya is a medicine-safety app: identify pills from a photo, check
whether their active ingredients interact, and answer follow-up questions with cited sources.
The repo started as "SynergyRx" (cancer drug-synergy data pipeline); that code is kept but is
no longer the main goal. Roadmap and progress: [docs/ROADMAP.md](docs/ROADMAP.md).
Remote: https://github.com/melekbouabid/tujia-.git

## Mandatory rules for every session
- **Before any step**, check which professor requirement it serves (the "Exigences des
  modules" checklist at the top of README.md, mirrored in "Graded in two modules" below).
  Never drop a requirement: if one becomes hard, reduce its scope and write the decision in
  docs/DECISIONS.md, but keep it on the list.
- **After every completed step**: update README.md (add a line to the "Journal de
  progression", in French: date, task, what was done, files, results; tick the matching
  "Exigences des modules" boxes only when the requirement is really met), update
  docs/ROADMAP.md, then commit with a clear message and push. Never force push, never
  rewrite history.
- Never commit anything under `data/` (DVC only), never touch `.env` or `backend/app.db`.
- **The project runs only on this PC.** Never deploy it publicly, never open a tunnel, never
  publish Docker images of the app, never bind a service to 0.0.0.0 on the host. The GitHub
  repository stays public (code and docs only).

## Graded in two modules

**Module 1, Industrialisation Vision & MLOps** (defence = live technical demo)
- Vision: pill detection with YOLOv8/v11 (Ultralytics), then identification of each
  detected pill by a classifier trained with transfer learning.
- Async FastAPI REST API, Pydantic schemas, Swagger at `/docs`.
- React app: image upload, webcam stream, live display of predictions and bounding boxes.
- DVC (data versioning), MLflow (tracking + Model Registry), ONNX (optimised deployment),
  Docker Compose, GitHub Actions CI/CD.

**Module 2, AI Engineering** (GitHub repo + commit history checked, technical report, demo)
- RAG with document management, metadata, citations, documentation.
- Tool calling and MCP; LangChain and/or LangGraph.
- Benchmark of retrieval and of answer generation; a web interface.

## Data direction (since 2026-10-27)
- **No personal photos.** Vision uses the public NLM **C3PI** pill image dataset
  (Computational Photography Project for Pill Identification,
  https://catalog.data.gov/dataset/computational-photography-project-for-pill-identification-c3pi,
  files at https://data.lhncbc.nlm.nih.gov/public/Pills/). Licence ODbL; research and
  development use only. Details: [docs/DATASET_C3PI.md](docs/DATASET_C3PI.md).
- **Two stages**: (1) detect pills in the image (YOLO), (2) identify each detected pill
  (classifier with transfer learning), limited to the 100-200 pills with the most images.
  Class list: `config/pill_classes.csv`.
- **ePillID** (https://arxiv.org/abs/2005.14288) is the reference benchmark to compare
  against (it is built from C3PI images).
- Chain: pill -> active ingredient(s) -> DDInter interactions.
- RAG corpus: openFDA drug labels (https://open.fda.gov/apis/drug/label/), DDInter, PubMed.
- The demo uses C3PI **test-split** images (never images the model trained on).

## What the app does
1. **Scan**: photo or webcam image of one or more pills; YOLO detects each pill, the
   classifier identifies it among the closed class list. Low confidence -> ask the user to
   confirm or pick from the top-5.
2. **Check**: look up the active ingredients in DDInter (connector in
   `src/synergyrx_data`): severity, explanation, timing advice only if the source gives it.
3. **Ask**: LangGraph agent answers follow-ups with RAG over openFDA labels, PubMed
   abstracts and DDInter, citing sources. Tools: `identify_pills(image)`,
   `check_interaction(a, b)`, `get_drug_info(drug)`, `search_literature(query, filters)`,
   optional `get_synergy`. Same tools via an MCP server (lower priority, at the end).
4. **Bonus** (only if time remains): page with the existing cancer synergy data.

## Safety rules (always, in code, prompts and UI)
- Never give doses.
- Never say a pair is "safe": say "no known interaction in this database".
- Always cite the source.
- Never invent timing advice; only repeat what the source says.
- Always show "information only, consult a pharmacist or doctor".
- Pill identification is a suggestion, never a certainty: show the confidence.

## Tech stack
- Vision: YOLOv8/v11 detection + YOLO classification (transfer learning) -> ONNX Runtime.
  Own venv in `vision/.venv` (git-ignored).
- MLOps: MLflow, DVC, Docker Compose (postgres + pgvector, mlflow, backend, frontend; later
  ollama), GitHub Actions CI and CD (images on ghcr.io). Run: `docker compose up -d --build`.
- LLM: LangGraph + LangChain, Ollama local LLM (Mistral, Llama or Qwen), bge-m3 embeddings.
- Retrieval: PostgreSQL + pgvector + BM25 hybrid search. Benchmark: RAGAS + Recall@k / MRR.
- Backend: FastAPI (`backend/`, already has accounts, sessions, admin, catalog).
- Frontend: React + Vite + TypeScript (`frontend/`, branded Tujya; Scanner page on the real API).
- API reference: docs/API.md; models served from `models/` (`python -m tujya_vision.publish`).

## Folder conventions
- `backend/` FastAPI app and its tests; `frontend/` React app; `vision/` training and export.
- `src/synergyrx_data/` existing data pipeline (connectors incl. DDInter, PubMed); `tests/` its tests.
- `config/` hand-maintained or generated small lists: `pill_classes.csv`, split lists.
  (`medicines.csv` is from the abandoned medicine-box plan.)
- `data/` is git-ignored (some sources forbid redistribution). Images go under
  `data/vision/`, tracked by DVC, never by git. RAG corpus also versioned with DVC.
- `docs/` plans and reports: ROADMAP.md (progress), STATUS.md, DECISIONS.md, SOURCES.md,
  DATASET_C3PI.md, NIGHT_REPORT.md.
- Secrets in `.env` only (git-ignored); `.env.example` is the template.

## Solo scope
- 100-200 pill classes from C3PI, no own photography or annotation by hand.
- Simple tool calling first, MCP last. No TrOCR, no reranker, no synergy page unless time remains.

## Known issues inherited from SynergyRx
- Only 289 of 4,081 drugs have SMILES; PubChem/ChEMBL/PubMed ran in sample mode only.
- TDC duplicates DrugComb (r = 1.00); O'Neil disagrees (r = 0.41); ~380k unique experiments.
- NCI-ALMANAC not downloaded; no ML model exists.
- Default admin account admin@gmail.com / admin must be changed.

## Working rules
- **Commit small and often**, with clear messages: the commit history is graded.
- Keep tests passing (`python -m pytest tests/unit`, backend tests in `backend/tests`),
  `ruff` and `mypy` clean. `make` is not installed on the dev machine (Windows 11).
- Update the checkboxes in docs/ROADMAP.md when a step is done.
- The root `.venv` was created on another machine and is broken; `.venv-dev` (Python 3.12)
  is the working pipeline venv.
- Line endings are mixed in this repo (Python and some docs LF; README, DECISIONS,
  config.yaml, pyproject.toml, .gitignore CRLF). Keep each file's existing style: `sed -i`
  and Python `write_text` on Windows silently change it (check with `git ls-files --eol`).
