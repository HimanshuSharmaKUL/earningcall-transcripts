This file defines how an agent should understand and navigate this repository.

# Repository Mission
This app is a financial transcript question-and-answer tool. It ingests earnings-call transcripts via DefeatBeta API, stores and searches them in PostgreSQL (FTS + pgvector), and answers questions via RAG (chunking + embeddings + LLM).

---

# Repository Overview
- **Backend**: FastAPI app that exposes ingestion, search, and Q&A endpoints and orchestrates transcript fetch, preprocessing, persistence, and RAG.
- **Database**: PostgreSQL with `pgvector` for embeddings + FTS for raw transcript search.
- **Frontend**: Angular SPA that provides UI for ingestion, FTS search, and RAG Q&A.
- **Infra**: `docker-compose.trendtrackerhimanshu.yml` provisions PostgreSQL + pgAdmin (backend/frontend services are commented but intended).

---

# API Endpoints (FastAPI)
Defined in `backend/main.py` and `backend/routes/*`.

## Core
- `GET /` — Health check, returns a basic message.
- `GET /__debug_cors_t` — Returns resolved CORS origins from config.

## Ingestion
- `POST /ingest/ingest-in` — Ingest a transcript.
  - Request: `IngestRequest` (company query, security type, exchange code, year, quarter).
  - Response: `IngestionResponse` (company + transcript IDs, fiscal period, ticker).
  - Flow: resolve company → fetch transcript → preprocess → persist transcript + org entities → return IDs.

## Search (FTS)
- `POST /search/query` — Full-text search over stored transcripts.
  - Request: `QueryRequest` (query, optional company_id/year/quarter, pagination).
  - Response: `QueryResponse` with ranked `TranscriptHit` (rank + snippet).
  - Uses PostgreSQL `websearch_to_tsquery`, `ts_rank_cd`, and `ts_headline` for snippets.

## Q&A (RAG)
- `POST /qna/ask` — Ask a question against stored transcripts.
  - Request: `RAGRequest` (question + company query + optional fiscal filters).
  - Response: `RAGResponse` (answer + sources).
  - Flow: build chunks → embed new chunks → retrieve top-k → augment prompt → LLM answer → return sources.

---

# Backend Architecture Snapshot
Source of truth: `backend/main.py`, `backend/routes/*`, `backend/services/*`.

## Routing
- `backend/main.py` wires CORS + registers routers: `ingest`, `search`, `quesans`.

## Services
- `backend/services/ticker_from_company.py`
  - Resolves a free-form company name to a ticker via OpenFIGI API.
  - Returns structured resolver response (name, ticker, exchange, security type).
- `backend/services/ingestion.py`
  - Orchestrates ingestion: resolve company → store transcript → return summary payload.
- `backend/services/fetch_transcripts.py`
  - Uses DefeatBeta API (`defeatbeta_api`) to fetch transcript data.
  - Preprocesses content with spaCy to extract ORG entities + metadata.
  - Persists transcripts + org entities to PostgreSQL.
- `backend/services/search.py`
  - Implements FTS search via PostgreSQL `tsvector`/`tsquery` and snippet generation.
- `backend/services/chunking.py`
  - Builds chunks from transcripts using:
    - Paragraph-based chunking (per paragraph, chunked by size).
    - Semantic chunking (sentence similarity + token limit).
- `backend/services/rag.py`
  - Embeds chunks (SentenceTransformers model).
  - Retrieves top-k chunks, optionally with hybrid FTS filter.
  - Builds augmented prompt and generates answer via **OpenAI** or **Ollama** (config-driven).
  - Returns answer + source snippets + scores.
- `backend/services/qna.py`
  - Orchestrates Q&A: build chunks → embed → retrieve → generate → format response.

---

# Frontend Overview
Source: `frontend/src/app/app.component.ts` + `frontend/src/app/app.component.html`.

The UI is a single-page Angular console with three workflows:
1. **Ingest**: collects company query, security type, exchange code, year/quarter; calls `/ingest/ingest-in` and renders summary.
2. **Search**: runs FTS queries with optional company/year/quarter filters; renders ranked snippets split by a delimiter.
3. **Q&A**: sends question + company metadata to `/qna/ask`; renders answer with chunk citations + source list.

Frontend behavior notes:
- Uses a configurable API base URL (`environment.apiBaseUrl`).
- Displays backend error `detail` messages when present.
- Formats citations by extracting `chunk_id` markers and rendering short IDs with tooltips.

---

# Data/Infra Notes
- `docker-compose.trendtrackerhimanshu.yml` brings up:
  - PostgreSQL with `pgvector` on host port **5433**.
  - pgAdmin on host port **5050**.

---

# End-to-End Flow (Mental Model)
1. **Ingest** → resolve ticker → fetch transcripts → preprocess + persist → stored in Postgres.
2. **Search** → FTS against `raw_text` → ranked hits + snippets.
3. **Q&A** → chunking → embedding → retrieve top-k → prompt augmentation → LLM answer + cited sources.
