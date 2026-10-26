# Tujya: project guide for Claude sessions

Solo student project. Tujya is a medicine-safety app: scan two medicine boxes, check whether
their active ingredients interact, and answer follow-up questions with cited sources.
The repo started as "SynergyRx" (cancer drug-synergy data pipeline); that code is kept but is
no longer the main goal. Roadmap and progress: [docs/ROADMAP.md](docs/ROADMAP.md).
Remote: https://github.com/melekbouabid/tujia-.git

## Graded in two modules

**Module 1, Industrialisation Vision & MLOps** (defence = live technical demo)
- Vision axis 3: object detection with YOLOv8/v11 (Ultralytics).
- Async FastAPI REST API, Pydantic schemas, Swagger at `/docs`.
- React app: image upload, webcam stream, live display of predictions and bounding boxes.
- DVC (data versioning), MLflow (tracking + Model Registry), ONNX (optimised deployment),
  Docker Compose, GitHub Actions CI/CD.

**Module 2, AI Engineering** (GitHub repo + commit history checked, technical report, demo)
- RAG with document management, metadata, citations, documentation.
- Tool calling and MCP; LangChain and/or LangGraph.
- Benchmark of retrieval and of answer generation; a web interface.

## What the app does
1. **Scan**: photo or webcam of two medicine boxes; YOLO detects each box. Each medicine is
   its own YOLO class (closed list of 10-15 in `config/medicines.csv`), so no OCR in the main
   version (OCR = optional bonus for unknown boxes). Low confidence -> ask the user to confirm.
2. **Check**: look up the two active ingredients in DDInter (connector exists in
   `src/synergyrx_data`, disabled by default): severity, explanation, timing advice only if
   the source gives it.
3. **Ask**: LangGraph agent answers follow-ups with RAG over official leaflets, PubMed
   abstracts and DDInter, citing sources. Tools: `identify_drugs(image)`,
   `check_interaction(a, b)`, `get_drug_info(drug)`, `search_literature(query, filters)`,
   optional `get_synergy`. Same tools via an MCP server (lower priority, at the end).
4. **Bonus** (only if time remains): page with the existing cancer synergy data.

## Safety rules (always, in code, prompts and UI)
- Never give doses.
- Never say a pair is "safe": say "no known interaction in this database".
- Always cite the source.
- Never invent timing advice; only repeat what the source says.
- Always show "information only, consult a pharmacist or doctor".

## Tech stack
- Vision: YOLOv8/v11 -> ONNX Runtime; annotation with Label Studio or Roboflow.
- MLOps: MLflow, DVC, Docker Compose (backend, frontend, postgres, mlflow, ollama), GitHub Actions.
- LLM: LangGraph + LangChain, Ollama local LLM (Mistral, Llama or Qwen), bge-m3 embeddings.
- Retrieval: PostgreSQL + pgvector + BM25 hybrid search. Benchmark: RAGAS + Recall@k / MRR.
- Backend: FastAPI (`backend/`, already has accounts, sessions, admin, catalog).
- Frontend: React + Vite + TypeScript (`frontend/`, branded Tujya; predictions are still mock).

## Folder conventions
- `backend/` FastAPI app and its tests; `frontend/` React app.
- `src/synergyrx_data/` existing data pipeline (connectors incl. DDInter, PubMed); `tests/` its tests.
- `config/` hand-maintained lists, incl. `medicines.csv` (class_id,brand,ingredient,strength,form).
- `data/` is git-ignored (some sources forbid redistribution). Images go under `data/vision/`,
  tracked by DVC, never by git. RAG corpus also versioned with DVC.
- `docs/` plans and reports: ROADMAP.md (progress), STATUS.md, DECISIONS.md, SOURCES.md.
- Secrets in `.env` only (git-ignored); `.env.example` is the template.

## Solo scope
- 10-15 medicines, 30-50 photos each (angles, lights, backgrounds; 2-3 boxes together in a
  third; some from the laptop webcam; ~30 photos with no medicine).
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
