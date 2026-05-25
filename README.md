# CHiPS — Run Guide (Stages 01–05)

This README explains how to run the CHiPS pipeline stages (01–05) and describes the repository layout.

Last updated: 2026-05-25

---

## Quick overview

- Stage 01: Preprocessing (PDF → images, image cleaning)
- Stage 02: OCR (Docling-based OCR and structure extraction)
- Stage 03: Chunking (Docling HybridChunker)
- Stage 04: Embeddings & RAG (embeddings, vector store)
- Stage 05: Web UI (Flask backend + Node/Express UI server)

This file shows the minimal commands to run each stage and a short explanation of the project structure.

---

## Prerequisites

- Python 3.8+ (3.10 recommended)
- Node.js 16+ (for the web UI)
- pip and optionally virtualenv
- On Windows: ensure Git/WSL or environment supports long paths if needed

Install Python deps (project root):

```bash
python -m venv .venv
# Windows PowerShell
.venv\Scripts\Activate.ps1
# Unix/macOS
source .venv/bin/activate

pip install -r requirements.txt
```

Install Node deps (web UI):

```bash
cd 05_webui/nodejs
npm install
```

Configuration:
- Copy `.env.template` to `.env` and set values (API keys, paths, ports) before production runs.

---

## Project structure (high level)

- 01_preprocessing/ — Stage 1 & Stage 2 tools
  - stage1_image_prep/ — image preprocessing modules (denoise, deskew, pdf_to_image, stamp_detector)
  - stage2_ocr/ — OCR pipeline modules (docling wrappers, postprocess)
  - run_stage1.py — Stage 1 entrypoint
  - run_stage2.py — Stage 2 entrypoint

- 02_optimization/ — Text/word-level optimization utilities (spell correction, dictionaries)
- 03_chunking/ — Document chunking utilities
  - docling_chunker.py — Preferred chunker entrypoint (Docling HybridChunker)

- 04_embeddings_and_kg/ — Embedding & knowledge-graph (RAG) scripts
  - scripts/ — rag pipeline, embeddings scripts, indexers
  - db/ — local databases (qdrant, etc.)

- 05_webui/ — Web UI and API server
  - app.py — Flask backend for RAG API
  - nodejs/ — Express server and SPA (server.js, run_server.js helper)

- utils/ — shared utilities (logging_config, helpers)
- cleanup_unused.py — helper to review/delete unused files
- main_pipeline.py — top-level orchestrator to run Stages 01–04 in sequence
- ORCHESTRATION_GUIDE.md — longer guide (also in repo)

---

## Run Stage 01 — Preprocessing

Converts PDFs to preprocessed images and runs initial image cleaning.

Script: `01_preprocessing/run_stage1.py`

Usage examples:

Single PDF:

```bash
python 01_preprocessing/run_stage1.py path/to/document.pdf
```

Process a folder (default reads `01_preprocessing/input_pdfs` and writes to `01_preprocessing/stage1_output`):

```bash
python 01_preprocessing/run_stage1.py /path/to/pdf_folder/ -o 01_preprocessing/stage1_output
```

Notes:
- The script validates PDF size and existence before processing.
- Logs are written using the centralized logging utilities in `utils/`.

---

## Run Stage 02 — OCR

Runs Docling-based OCR on Stage 1 output.

Script: `01_preprocessing/run_stage2.py`

Basic usage (process default stage1 output → stage2 output):

```bash
python 01_preprocessing/run_stage2.py
```

Process a specific input directory and output directory:

```bash
python 01_preprocessing/run_stage2.py 01_preprocessing/stage1_output -o 01_preprocessing/stage2_output
```

Notes:
- On Windows, the script handles Hugging Face symlink issues and forces safe download behavior.
- Outputs include recognized text, confidence logs, and structured JSON/text outputs.

---

## Run Stage 03 — Chunking

Script: `03_chunking/docling_chunker.py`

Default (chunks all docs in a folder):

```bash
python 03_chunking/docling_chunker.py --input 01_preprocessing/stage2_output --output 03_chunking/output
```

Single file:

```bash
python 03_chunking/docling_chunker.py --input 01_preprocessing/stage2_output/file.md --output 03_chunking/output
```

Options:
- `--model` — change tokenizer/embedding model (default uses `BAAI/bge-m3` in the project)
- `--max-tokens` — max tokens per chunk
- `--mapping` — optional mapping file to rename outputs

Notes:
- This is the recommended chunker (Docling HybridChunker) and replaces legacy scripts.

---

## Run Stage 04 — Embeddings & RAG

Entrypoint(s): `04_embeddings_and_kg/scripts/rag_pipeline.py`

Generate embeddings and index into the vector store (Qdrant) used by the RAG system.

Basic run (example):

```bash
python 04_embeddings_and_kg/scripts/rag_pipeline.py --chunks-dir 03_chunking/output --index-name chips_index
```

Notes:
- Ensure Qdrant is configured and running if using a remote/local service.
- Environment variables control API keys and model endpoints. Use the `.env` file.
- There are helper scripts in `04_embeddings_and_kg/scripts/` for indexing and maintenance.

---

## Run Stage 05 — Web UI (Flask + Node)

Stage 05 provides the user-facing web UI and proxied API to the Flask backend.

1. Start Flask backend (API):

`npm start` in `05_webui/nodejs` now starts the Flask backend automatically if it is not already running.

If you want to run Flask manually:

```bash
# from repo root
cd 05_webui
python app.py
```

Flask listens on port `5000` by default (configurable via `app.py` or environment variables).

2. Start Node/Express server (UI + proxy):

```bash
cd 05_webui/nodejs
node server.js
```

or simply:

```bash
npm start
```

Open the UI in a browser:

```
http://localhost:3000
```

## Tips for production

- Use `.env` for secrets, never commit secrets to the repo.
- Pin Python packages in `requirements.txt` for reproducible builds.
- Run the pipeline on a machine with available CPU/GPU and sufficient RAM/disk for the vector DB.
- Monitor `04_embeddings_and_kg/db` disk usage; Qdrant stores vectors locally by default.
- For the web UI, use a process manager (pm2/systemd) to run the Node server and the Flask app.

---
