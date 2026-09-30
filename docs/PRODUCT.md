# Lexora — Product Documentation

> User-serving product guide (technical deep docs already in `docs/`).
> Codebase: `/home/sajad/Projects/portfolio/lexora-ai` | Stack: FastAPI + PostgreSQL + Redis/Celery + FAISS + OpenAI | Tests: 35 unit, ~49% cov

## 1. What this product is

Grounded Q&A over your own PDFs/TXT/MD/DOCX: validated upload → extract/chunk/embed → per-user FAISS → cached retrieval with citations → LLM chat + SSE streaming + history.

**Who it's for:** small teams needing private document Q&A without building retrieval, auth, and background jobs from scratch.

## 2. How users use it

1. `POST /api/v1/auth/register|login` → JWT (refresh/logout via jti blacklist in Redis)
2. Upload: `POST /api/v1/documents` (type/size validated, `inline` or `background` via Celery — same `process_document` path)
3. Ask: `POST /api/v1/chat` (+ SSE stream) → answer + source metadata (1h user-scoped retrieval cache)
4. Manage: list/status/delete documents (delete rebuilds per-user FAISS), conversation history
5. Operate: `/health`, `/ready` (DB+Redis), `/metrics` (Prometheus), structlog JSON

## 3. Serve it

```bash
cp .env.example .env  # DATABASE_URL, REDIS_URL, SECRET_KEY, OPENAI_API_KEY
docker compose -f docker/docker-compose.yml up -d postgres redis
uvicorn app.main:app --reload  # /docs
celery -A app.tasks.celery_app worker --loglevel=info
python -m pytest  # SQLite overrides, no real OpenAI
python scripts/eval_retrieval.py --top-k 3  # 3/3 keyword probe
```

## 4. Gaps → roadmap (from repo)

Alembic from day one (currently `create_all`); Redis sliding-window rate limits (now in-memory single-replica); idempotency on uploads/chat; retryable streaming (only non-stream retries 3×); wire Sentry; `APIKey` routes; `nginx.conf`/SSL (referenced but missing); pgvector to replace per-user FAISS; integration + load tests; embedding-recall/faithfulness evals beyond keyword probe; no latency benchmarks yet.
