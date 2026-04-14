# RAG-Based Question Answering API

A production-ready Retrieval-Augmented Generation (RAG) system built with FastAPI, FAISS, and Claude.

---

## Architecture

```
User
 │
 ▼
FastAPI (Rate Limiting Middleware)
 ├── POST /api/v1/documents/upload      ← accepts PDF / TXT
 │       │
 │       └── BackgroundTask (ThreadPoolExecutor)
 │               ├── DocumentProcessor  (extract + chunk)
 │               ├── EmbeddingService   (sentence-transformers)
 │               └── VectorStoreService (FAISS IndexFlatIP)
 │
 └── POST /api/v1/query/
         ├── EmbeddingService   (embed query)
         ├── VectorStoreService (similarity search)
         ├── GenerationService  (Anthropic Claude)
         └── MetricsCollector   (latency + scores)
```

See `architecture.png` or `architecture.drawio` in the repo root for a visual diagram.

---

## Setup

### 1. Clone & install

```bash
git clone https://github.com/your-username/rag-qa-system.git
cd rag-qa-system

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure environment

```bash
cp .env.example .env
# Edit .env and set ANTHROPIC_API_KEY
```

> **Note**: The system works without an API key — it falls back to extractive answers (shows the most relevant chunk directly). Set the key for LLM-generated answers.

### 3. Run

```bash
uvicorn app.main:app --reload --port 8000
```

API docs: http://localhost:8000/docs

---

## Usage

### Upload a document

```bash
# PDF
curl -X POST http://localhost:8000/api/v1/documents/upload \
  -F "file=@my_document.pdf"

# TXT
curl -X POST http://localhost:8000/api/v1/documents/upload \
  -F "file=@notes.txt"
```

Response:
```json
{
  "document_id": "uuid-here",
  "filename": "my_document.pdf",
  "status": "pending",
  "message": "Document queued for processing..."
}
```

### Check ingestion status

```bash
curl http://localhost:8000/api/v1/documents/status/{document_id}
```

### Ask a question

```bash
curl -X POST http://localhost:8000/api/v1/query/ \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are the main findings of the report?",
    "top_k": 5,
    "similarity_threshold": 0.3
  }'
```

Response includes: `answer`, `retrieved_chunks` with scores, `latency_ms`, `model_used`.

### View metrics

```bash
curl http://localhost:8000/api/v1/query/metrics
```

---

## Design Decisions

### Chunking Strategy

**Chunk size: 512 characters (~128 tokens)**

Rationale:
- Matches the input window of `all-MiniLM-L6-v2` (256 token limit, safely within range)
- Semantically focused: one chunk ≈ one topic/paragraph
- Avoids "topic blending" that degrades retrieval precision
- **Overlap: 20% (102 chars)** preserves sentence continuity at boundaries

### Retrieval Failure Case Observed

**Short/keyword queries** (e.g., `"summary"`, `"PDF?"`) cause similarity scores across all chunks to cluster in a narrow band (~0.4–0.55), making it impossible to distinguish relevant from irrelevant content. This happens because the embedding model has too little semantic signal to work with.

**Mitigation**: Enforce `min_length=3` on queries, prompt users for natural-language questions, and set a `similarity_threshold` (default 0.3) to avoid returning low-confidence results.

### Metric Tracked: Query Latency + Similarity Score

Two metrics are tracked per query:

1. **End-to-end latency (ms)**: from request receipt to response. Helps identify bottlenecks (embedding vs. LLM vs. FAISS search). Target: p95 < 2000ms for retrieval-only.

2. **Top-1 similarity score**: highest cosine similarity score of retrieved chunks. Low scores (< 0.35) indicate the query is out-of-domain or the document lacks relevant content. Tracked over time to spot retrieval degradation.

---

## API Reference

| Endpoint | Method | Description |
|---|---|---|
| `/api/v1/health` | GET | System health + stats |
| `/api/v1/documents/upload` | POST | Upload PDF or TXT |
| `/api/v1/documents/status/{id}` | GET | Check ingestion status |
| `/api/v1/documents/list` | GET | List all documents |
| `/api/v1/query/` | POST | Ask a question |
| `/api/v1/query/metrics` | GET | Aggregated metrics |

---

## Rate Limiting

20 requests per 60 seconds per IP. Returns `429` when exceeded.

---

## Tech Stack

| Component | Choice | Reason |
|---|---|---|
| Web framework | FastAPI | Async, auto-docs, Pydantic built-in |
| Embedding model | `all-MiniLM-L6-v2` | Fast CPU inference, 384-dim, no API key |
| Vector store | FAISS `IndexFlatIP` | Local, exact search, zero infra |
| LLM | Anthropic Claude Haiku | Low latency, cost-effective for RAG |
| Background jobs | `asyncio` + `ThreadPoolExecutor` | Non-blocking ingestion in same process |

---

## Project Structure

```
rag-qa-system/
├── app/
│   ├── main.py                    # FastAPI app + middleware
│   ├── models/schemas.py          # Pydantic request/response models
│   ├── routers/
│   │   ├── documents.py           # Upload + status endpoints
│   │   ├── query.py               # Query + metrics endpoints
│   │   └── health.py              # Health check
│   ├── services/
│   │   ├── document_processor.py  # Text extraction + chunking
│   │   ├── embeddings.py          # Sentence-transformer embeddings
│   │   ├── vector_store.py        # FAISS index + metadata store
│   │   ├── ingestion_job.py       # Background ingestion pipeline
│   │   └── llm.py                 # Anthropic Claude generation
│   └── utils/
│       ├── rate_limiter.py        # Sliding-window rate limiter
│       └── metrics.py             # Latency + similarity tracking
├── data/                          # Auto-created: FAISS index + metrics
├── requirements.txt
├── .env.example
└── README.md
```
