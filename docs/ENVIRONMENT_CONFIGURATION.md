# NOVA AI — Environment Configuration

Status: **proposed configuration contract; no settings module exists yet.** All values are placeholders — **never commit real credentials.** Related: [SECURITY](SECURITY.md) §9, [AWS_DEPLOYMENT](AWS_DEPLOYMENT.md), [DEVOPS_AND_CICD](DEVOPS_AND_CICD.md).

## 1. Principles
- Configuration via environment variables, validated at startup (fail fast on missing/invalid values, e.g. with `django-environ` or `pydantic-settings` **[choose]**).
- One settings package with `base`, `dev`, `test`, `staging`, `prod` modules; production-safe defaults (`DEBUG=False`, secure cookies on).
- **Public vs private:** anything prefixed `VITE_` is compiled into the browser bundle and is **public**. Private values exist only on the server (ECS task env / Secrets Manager).
- Secrets in AWS come from Secrets Manager injected at task start; non-secret config via plain task-definition environment.
- `.env.example` at the repo root lists every variable with placeholders; `.env` is git-ignored.

## 2. Environments

| Env | Where | Providers | Data |
|---|---|---|---|
| `development` | docker-compose on a laptop | fake providers by default; real dev keys optional | local volumes, MinIO for S3 |
| `test` | pytest / CI | fake providers always | ephemeral DB |
| `staging` | AWS (staging stack) | real providers, low spend caps | synthetic |
| `production` | AWS (production stack) | real providers, production keys | real |

## 3. Backend variables (private unless noted)

### 3.1 Django core
| Variable | Purpose | Req. | Example placeholder | Scope |
|---|---|---|---|---|
| `DJANGO_SETTINGS_MODULE` | Select settings module | Req | `config.settings.prod` | all |
| `DJANGO_SECRET_KEY` | Signing key (**secret**) | Req | `<generated-50+-char-random>` | all (distinct per env) |
| `DJANGO_DEBUG` | Debug mode | Opt (default false) | `false` | dev=true only |
| `DJANGO_ALLOWED_HOSTS` | Host header allowlist | Req | `app.example.com,localhost` | all |
| `CSRF_TRUSTED_ORIGINS` | Trusted origins for CSRF | Req (staging/prod) | `https://app.example.com` | all |
| `CORS_ALLOWED_ORIGINS` | Dev-only cross-origin allowlist; **empty in prod** | Opt | `http://localhost:5173` | dev |
| `SESSION_COOKIE_SECURE` / `CSRF_COOKIE_SECURE` | Send cookies over HTTPS only | Opt (true in non-dev) | `true` | staging, prod |
| `SECURE_SSL_REDIRECT`, `SECURE_HSTS_SECONDS` | HTTPS enforcement | Opt | `true`, `3600` (increase gradually) | staging, prod |
| `SESSION_COOKIE_AGE_SECONDS` | Session lifetime | Opt | `604800` | all |
| `LOG_LEVEL` | Log verbosity | Opt | `INFO` | all |
| `APP_VERSION` | Image/release version for logs | Opt | `v0.1.0` | set by CI |

### 3.2 Database
| Variable | Purpose | Req. | Example placeholder | Scope |
|---|---|---|---|---|
| `DATABASE_URL` *(or the discrete vars below)* | Connection string (**secret** because it holds a password) | Req | `postgres://nova_app:<password>@db-host:5432/nova` | all |
| `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER` | Discrete settings (non-secret) | Alt | `db-host`, `5432`, `nova`, `nova_app` | all |
| `DB_PASSWORD` | DB password (**secret**) | Alt | `<from-secrets-manager>` | all |
| `DB_SSLMODE` | TLS to the DB | Opt | `require` (`verify-full` preferred where CA available) | staging, prod |
| `DB_CONN_MAX_AGE` | Connection reuse | Opt | `60` | all |
| `DB_MIGRATOR_USER` / `DB_MIGRATOR_PASSWORD` | Separate DDL role for the migrate task (**secret**) | Opt (prod) | `nova_migrator` / `<secret>` | staging, prod |

### 3.3 Redis / Celery
| Variable | Purpose | Req. | Example placeholder | Scope |
|---|---|---|---|---|
| `REDIS_URL` | Cache/throttle store (**secret** if it includes AUTH) | Req | `rediss://:<token>@redis-host:6379/0` | all |
| `CELERY_BROKER_URL` | Broker | Req | `rediss://:<token>@redis-host:6379/1` | all |
| `CELERY_TASK_ALWAYS_EAGER` | Run tasks inline (tests) | Opt | `false` | test only |
| `CELERY_WORKER_CONCURRENCY` | Worker parallelism | Opt | `4` | all |
| `CELERY_VISIBILITY_TIMEOUT_SECONDS` | Must exceed longest task | Opt | `3600` | all |

### 3.4 Object storage (S3)
| Variable | Purpose | Req. | Example placeholder | Scope |
|---|---|---|---|---|
| `AWS_REGION` | Region | Req | `eu-west-1` | all |
| `S3_DOCUMENTS_BUCKET` | Documents bucket name | Req | `nova-docs-staging` | all |
| `S3_ENDPOINT_URL` | Override for MinIO | Opt | `http://minio:9000` | dev only |
| `S3_ACCESS_KEY_ID` / `S3_SECRET_ACCESS_KEY` | **Dev/MinIO only**; in AWS use the ECS task role (no keys) | Opt | `minioadmin` / `<dev-only>` | dev |
| `UPLOAD_MAX_MB` | Max upload size | Opt | `20` | all |
| `UPLOAD_ALLOWED_EXTENSIONS` | Allowlist | Opt | `pdf,docx,txt,md` | all |
| `USER_MAX_DOCUMENTS`, `USER_MAX_TOTAL_MB` | Quotas | Opt | `200`, `1000` | all |

### 3.5 LLM, embeddings, reranker
| Variable | Purpose | Req. | Example placeholder | Scope |
|---|---|---|---|---|
| `PROVIDER_MODE` | `real` or `fake` | Opt (default real in staging/prod) | `fake` | dev, test |
| `LLM_PROVIDER` | Adapter name | Req (real mode) | `<provider-name>` | all |
| `LLM_MODEL` | Model identifier | Req | `<model-id>` | all |
| `LLM_API_KEY` | **Secret** | Req | `<secret>` | all (distinct per env) |
| `LLM_TIMEOUT_SECONDS` | Per-call timeout | Opt | `60` | all |
| `LLM_TEMPERATURE` | Generation temperature | Opt | `0.1` | all |
| `LLM_MAX_OUTPUT_TOKENS` | Output cap | Opt | `1000` | all |
| `EMBEDDING_PROVIDER` | Adapter name | Req | `<provider-name>` | all |
| `EMBEDDING_MODEL` | Model identifier | Req | `<model-id>` | all |
| `EMBEDDING_DIMENSIONS` | Must match the DB column dimension | Req | `<N>` | all |
| `EMBEDDING_API_KEY` | **Secret** (may equal LLM key if same vendor) | Req | `<secret>` | all |
| `EMBEDDING_BATCH_SIZE` | Batching | Opt | `64` | all |
| `RERANKER_ENABLED` | Toggle reranking | Opt | `true` | all |
| `RERANKER_MODEL` | Model id + pinned revision | Opt | `<model-id>@<revision>` | all |
| `RERANKER_TIMEOUT_MS` | Hard timeout | Opt | `1500` | all |

### 3.6 Retrieval and chat tuning (config, not secrets; versioned in `retrieval_config`)
| Variable | Purpose | Example | Scope |
|---|---|---|---|
| `LEXICAL_BACKEND` | `pg_fts` or `bm25_memory` | `pg_fts` | all |
| `RETRIEVAL_MODE` | `lexical`, `dense`, `hybrid`, `hybrid_rerank` (C1–C4) | `hybrid` | all |
| `RETRIEVAL_K_LEX`, `RETRIEVAL_K_DENSE` | Candidates per retriever | `30` | all |
| `RRF_K` | RRF constant | `60` | all |
| `RERANK_TOP_N` | Candidates to rerank | `20` | all |
| `EVIDENCE_MAX_CHUNKS` | Evidence cap | `6` | all |
| `EVIDENCE_TOKEN_BUDGET`, `HISTORY_TOKEN_BUDGET` | Prompt budgets | `<tokens>` | all |
| `ABSTENTION_MIN_SCORE` | Sufficiency threshold (calibrated) | `<float>` | all |
| `CHUNK_TARGET_TOKENS`, `CHUNK_OVERLAP_TOKENS` | Ingestion chunking | `500`, `60` | all |
| `PROMPT_VERSION` | Prompt template version | `v1` | all |
| `MAX_MESSAGE_CHARS` | Max user message | `4000` | all |
| `THROTTLE_*` | Per-scope rate limits | `chat=20/min` | all |

### 3.7 Observability
| Variable | Purpose | Req. | Example | Scope |
|---|---|---|---|---|
| `SENTRY_DSN` | Error tracking (**treated as secret-ish**) | Opt | `<dsn>` | staging, prod |
| `SENTRY_ENVIRONMENT`, `SENTRY_TRACES_SAMPLE_RATE` | Tagging/sampling | Opt | `staging`, `0.0` | staging, prod |
| `LOG_FORMAT` | `json` or `console` | Opt | `json` | json in cloud, console in dev |

## 4. Frontend variables (**PUBLIC — embedded in the bundle**)
| Variable | Purpose | Req. | Example placeholder | Scope |
|---|---|---|---|---|
| `VITE_API_BASE_URL` | API base; relative path for same-origin deploys | Opt (default `/api/v1`) | `/api/v1` (dev proxy → `http://localhost:8000`) | all |
| `VITE_APP_ENV` | Display/diagnostic env name | Opt | `staging` | all |
| `VITE_SENTRY_DSN` | Frontend error tracking (public DSN) | Opt | `<public-dsn>` | staging, prod |
| `VITE_MAX_UPLOAD_MB` | Client-side hint only (server enforces) | Opt | `20` | all |
**Rule:** if exposing the value would hurt you, it must not start with `VITE_`.

## 5. Local development (`infra/docker-compose.yml`, proposed services)
`db` (PostgreSQL image with pgvector — **[verify image/tag]**), `redis`, `minio` (+ bucket init), `backend` (ASGI, autoreload), `worker`, `frontend` (Vite dev server with proxy to backend so cookies stay same-origin). Dev defaults: `PROVIDER_MODE=fake`, `DJANGO_DEBUG=true`, cookies non-secure.

## 6. `.env.example` skeleton
```dotenv
# --- Django ---
DJANGO_SETTINGS_MODULE=config.settings.dev
DJANGO_SECRET_KEY=change-me-dev-only
DJANGO_DEBUG=true
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
CSRF_TRUSTED_ORIGINS=http://localhost:5173
CORS_ALLOWED_ORIGINS=http://localhost:5173
# --- Database ---
DATABASE_URL=postgres://nova:nova@db:5432/nova
# --- Redis / Celery ---
REDIS_URL=redis://redis:6379/0
CELERY_BROKER_URL=redis://redis:6379/1
# --- S3 (MinIO in dev) ---
AWS_REGION=us-east-1
S3_DOCUMENTS_BUCKET=nova-docs-dev
S3_ENDPOINT_URL=http://minio:9000
S3_ACCESS_KEY_ID=minioadmin
S3_SECRET_ACCESS_KEY=minioadmin
# --- Providers ---
PROVIDER_MODE=fake
LLM_PROVIDER=
LLM_MODEL=
LLM_API_KEY=
EMBEDDING_PROVIDER=
EMBEDDING_MODEL=
EMBEDDING_DIMENSIONS=
EMBEDDING_API_KEY=
# --- Frontend (PUBLIC) ---
VITE_API_BASE_URL=/api/v1
VITE_APP_ENV=development
```

## 7. Secret handling matrix

| Secret | Local | CI | Staging/Prod |
|---|---|---|---|
| `DJANGO_SECRET_KEY` | `.env` | GitHub secret (test value) | Secrets Manager |
| DB credentials | `.env` | service container defaults | Secrets Manager (rotated) |
| LLM/embedding keys | `.env` (optional) | GitHub environment secret (eval jobs only) | Secrets Manager |
| AWS access | n/a (MinIO) | OIDC role (no keys) | ECS task roles |
| Redis auth token | n/a | n/a | Secrets Manager |

## 8. Validation at startup (proposed)
Reject: `DEBUG=true` in staging/prod; `DJANGO_SECRET_KEY` shorter than a minimum or equal to the example value; wildcard `ALLOWED_HOSTS` in prod; `EMBEDDING_DIMENSIONS` ≠ the DB vector column dimension; `PROVIDER_MODE=fake` in production.
