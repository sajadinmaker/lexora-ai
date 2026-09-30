# Lexora

Upload documents, ask questions with source attribution.

## Problem

Small teams need grounded Q&A over their own PDFs/TXT/Markdown/DOCX without building retrieval, auth, and background processing from scratch.

## Solution

A FastAPI backend that validates uploads, extracts/chunks/embeds text, stores per-user vectors, and answers via retrieval + LLM with SSE streaming and conversation history.

## Architecture

```text
Client
  ↓
FastAPI (/api/v1: auth, documents, chat)
  ↓
PostgreSQL (users, documents, conversations, messages)
  ↓
Redis / Background Worker (retrieval cache, token blacklist, document jobs)
  ↓
Document Processing (validate → extract → chunk → embed → per-user FAISS)
  ↓
Retrieval (query embed → top-k → rank → context) → LLM (chat / stream)
```

## Key Engineering Decisions

1. PostgreSQL for relational history with async SQLAlchemy pooling; SQLite only for tests.
2. Redis for two jobs: 1h retrieval cache (user-scoped keys) and JWT jti blacklist for logout/rotation.
3. Dual-mode ingestion (`inline` local, `background` via Celery) — same `process_document` path.
4. Per-user FAISS isolation for simplicity; rebuild on delete (documented trade-off, pgvector is the next step).
5. Prometheus `/metrics` + structlog JSON + `/health` + `/ready` (DB + Redis checks).

## Tech Stack

Python · FastAPI · PostgreSQL · Redis · Celery · FAISS · OpenAI · Docker

## Features

1. JWT register/login/refresh/logout + `/me`
2. Validated upload (type/size) + list/status/delete with pagination
3. Chunk + embed pipeline with per-user vector isolation
4. Cached retrieval with source metadata
5. Chat + SSE streaming + conversation history
6. Health/ready/metrics endpoints
7. Docker Compose (Postgres, Redis, app, worker)

## Running Locally

```bash
python3.11 -m venv venv && source venv/bin/activate
pip install -r requirements.txt -r requirements-dev.txt
docker compose -f docker/docker-compose.yml up -d postgres redis
cp .env.example .env  # set DATABASE_URL, REDIS_URL, SECRET_KEY, OPENAI_API_KEY
uvicorn app.main:app --reload  # docs: /docs, health: /health
```

## Testing

```bash
python -m pytest        # 35 unit tests, ~49% coverage, SQLite overrides, no real OpenAI calls
python -m pytest tests/unit -q
python scripts/eval_retrieval.py --top-k 3   # deterministic chunking recall probe (3/3)
```

## Performance

No latency/throughput benchmarks yet (needs PG + Redis + OpenAI key; see `docs/performance.md`).
Measured: `35 passed`, `49%` coverage, retrieval recall probe `3/3` at top-3.

## Limitations

No Alembic dir (uses `create_all`); `nginx.conf`/SSL missing; Celery target fixed to `app.tasks.celery_app`; rate limiting is single-replica in-memory (Redis sliding-window is the next step); no idempotency on uploads/chat; streaming LLM calls not retried (non-streaming `generate` retries 3×); Sentry not wired (SDK removed); APIKey table without routes. See `docs/` + README history.

## Future Improvements

Alembic from day one; Redis-backed rate limits; replace Celery with `BackgroundTasks` or justify queue; replace FAISS with pgvector; drop LangChain wrappers for direct OpenAI calls; idempotency keys on ingest; integration + load tests; embedding-recall + faithfulness evals beyond the keyword probe.

---

## Maintenance

Last maintained: 2026-09-30 – minor docs touch.
