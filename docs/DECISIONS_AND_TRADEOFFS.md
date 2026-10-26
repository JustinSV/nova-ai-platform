# NOVA AI — Decisions and Trade-offs (ADR Index)

Status: all decisions are **proposed**; none has been validated by implementation or measurement. Each ADR states when it should be revisited. Related: [SYSTEM_ARCHITECTURE](SYSTEM_ARCHITECTURE.md), [RAG_PIPELINE](RAG_PIPELINE.md), [AWS_DEPLOYMENT](AWS_DEPLOYMENT.md).

| ADR | Decision | Status |
|---|---|---|
| 001 | Modular monolith with separate worker processes | Proposed |
| 002 | PostgreSQL + pgvector for vector retrieval | Proposed |
| 003 | PostgreSQL FTS first; true BM25 behind an interface | Proposed — needs benchmark |
| 004 | Plain-Python pipeline first; LangChain utilities optional; LangGraph gated | Proposed |
| 005 | Server-Sent Events (via fetch) for streaming | Proposed — verify infra |
| 006 | Cookie sessions + CSRF, same-origin deployment | Proposed |
| 007 | Celery + Redis for background work | Proposed |
| 008 | S3 for original documents; upload through the API | Proposed |
| 009 | ECS Fargate for compute | Proposed |
| 010 | LLM/embedding provider abstraction | Proposed |
| 011 | Chunking strategy and vector index selection | Proposed — needs benchmark |
| 012 | Infrastructure as code tool | **Open** |

---

## ADR-001 — Modular monolith vs. microservices
- **Context:** One developer/small team, a portfolio-grade but production-oriented system, strong consistency needs (permissions + retrieval + deletion).
- **Options:** (a) modular monolith + workers; (b) microservices (auth, ingestion, retrieval, chat); (c) serverless functions.
- **Decision:** (a).
- **Reasons:** transactional consistency in one DB, one deployable image, simpler testing and debugging, lower operational cost; module boundaries enforced via import rules.
- **Disadvantages:** boundaries depend on discipline; one codebase scales as a unit (workers mitigate); a heavy dependency (reranker) affects the API image.
- **Revisit when:** independent scaling or team ownership boundaries become a real constraint, or the reranker/ingestion needs a different runtime — extraction of `ingestion` or `retrieval` into a service is feasible because of the interface-based design.

## ADR-002 — PostgreSQL + pgvector vs. a dedicated vector database
- **Context:** Need dense retrieval with hard permission filters, transactional deletes, and few moving parts.
- **Options:** pgvector; managed/dedicated vector DB (e.g., OpenSearch, Qdrant, Pinecone, Weaviate); embedded libraries (FAISS).
- **Decision:** pgvector in the primary RDS instance.
- **Reasons:** single source of truth; filter + ranking + joins in one SQL statement; atomic version swap and deletion; no sync pipeline; lower cost/ops.
- **Disadvantages:** limited ANN features/tuning compared with specialists; vector search competes with OLTP load; filtered ANN recall caveats; scaling mostly vertical/read-replica.
- **Revisit when:** per-corpus size or QPS exceeds what measured tuning/replicas handle, or advanced features (native hybrid scoring, multi-vector, quantization) become required. Keep the `Retriever` interface so a swap is local. Do not run multiple vector stores without a documented need.

## ADR-003 — PostgreSQL FTS vs. a dedicated BM25 engine
- **Context:** The brief requires BM25-style lexical retrieval. PG FTS (`ts_rank_cd`) is not BM25 and lacks corpus-level IDF.
- **Options:** (a) PG FTS; (b) in-process BM25 (`rank_bm25`-like) over the user's active chunks, cached per retrieval epoch; (c) PG extension implementing BM25 (availability on RDS uncertain); (d) OpenSearch/Elasticsearch.
- **Decision:** `LexicalRetriever` interface; ship (a) as MVP default; implement (b) for the benchmark and small corpora; decide using C1/C3 results. (c)/(d) only if evidence demands.
- **Reasons:** keeps one datastore and exact filters; honest about what "BM25" means; measurable.
- **Disadvantages:** two lexical implementations to maintain during evaluation; (b) uses memory/CPU and rebuild time per user; (a) ranking quality may be lower.
- **Revisit when:** benchmark shows lexical quality gap, or per-user corpora exceed in-memory limits → consider (c)/(d).

## ADR-004 — LangChain vs. LangGraph (vs. neither)
- **Context:** Fixed, predictable pipeline; security-critical filtering and citation logic; desire to showcase modern tooling without overengineering.
- **Options:** pure Python; LangChain chains; LangGraph state machine.
- **Decision:** Pure Python baseline with interfaces; LangChain only for utilities (splitters/loaders/provider wrappers) where it saves code; LangGraph introduced in Phase 13 **only if** measured benefit ([AI_WORKFLOW_DESIGN](AI_WORKFLOW_DESIGN.md) §8).
- **Reasons:** clarity, testability, minimal dependency surface; framework APIs churn.
- **Disadvantages:** may reinvent small utilities; less "framework showcase" unless the gate passes.
- **Revisit when:** conditional routing/retry logic grows, or tracing/state inspection benefits become compelling.

## ADR-005 — SSE vs. WebSockets for streaming
- **Context:** Server → client token streaming; client → server messages are discrete requests.
- **Options:** SSE (via `fetch` stream on POST); WebSockets; long polling.
- **Decision:** SSE.
- **Reasons:** plain HTTP, works with session cookies/CSRF, simple proxy semantics, auto-framing, easy to test; unidirectional fits.
- **Disadvantages:** need `fetch`+stream (no native `EventSource` for POST); connection limits per browser/HTTP version; buffering/timeouts in proxies need configuration; no resumption in MVP.
- **Revisit when:** bidirectional realtime (collaboration, voice), or proxy constraints can't be resolved. Verify CloudFront/ALB behavior in staging first.

## ADR-006 — Cookie sessions vs. token-based authentication
- **Context:** First-party React SPA; security-sensitive content.
- **Options:** Django session cookie + CSRF; JWT in memory + refresh cookie; JWT in localStorage (rejected).
- **Decision:** Session cookie + CSRF, same-origin via CloudFront.
- **Reasons:** token not readable by JS; simple revocation; fewer moving parts; built-in Django support.
- **Disadvantages:** needs same-site deployment; CSRF handling; not ideal for native/third-party clients; sticky-free but session store required.
- **Revisit when:** native mobile apps or public API clients arrive (add token auth for those clients specifically, keep cookies for web).

## ADR-007 — Celery + Redis vs. alternatives
- **Context:** Ingestion is slow, retryable, and provider-bound; chat is synchronous streaming.
- **Options:** Celery+Redis; Celery+SQS; RQ/Dramatiq; AWS Step Functions/Lambda; DB-backed queue.
- **Decision:** Celery + Redis (required by the brief), with DB state as source of truth, `acks_late`, idempotent tasks, and a reconciler.
- **Reasons:** mature, Django-friendly, supports retries/backoff/periodic tasks.
- **Disadvantages:** Redis durability is weak; visibility-timeout pitfalls; Celery operational complexity; ElastiCache compatibility caveats.
- **Revisit when:** task loss or ops burden is observed → switch broker to SQS or a DB-backed queue; or if chat generation must be offloaded from the API.

## ADR-008 — S3 for documents; upload via API
- **Context:** Originals must not live in PostgreSQL; validation must be centralized.
- **Options:** S3 via API proxy; presigned direct-to-S3 uploads; filesystem/EFS.
- **Decision:** S3, upload through the API in the MVP.
- **Reasons:** simpler security validation (size/type before storing), single code path, easy dev with MinIO.
- **Disadvantages:** API handles upload bandwidth/memory/time (bounded by 20 MB default); less scalable than presigned uploads.
- **Revisit when:** larger files or high upload concurrency → presigned multipart uploads with post-upload validation by the worker (requires re-working the validation flow).

## ADR-009 — ECS Fargate vs. other compute
- **Context:** Containerized API + workers; small team; no Kubernetes allowed without justification.
- **Options:** ECS Fargate; ECS on EC2; EKS; App Runner; Lambda; Elastic Beanstalk; single EC2 + Compose.
- **Decision:** ECS Fargate (required by the brief).
- **Reasons:** no host management, native ALB/IAM/Secrets integration, simple rolling deploys and rollback.
- **Disadvantages:** higher per-vCPU cost than EC2 at steady load; limited host-level control; cold start of new tasks; long-lived SSE needs timeout tuning.
- **Revisit when:** sustained high load makes EC2 capacity cheaper, GPU/special hardware is needed (local reranker at scale), or cost pressure mandates cheaper compute.

## ADR-010 — LLM/embedding provider abstraction
- **Context:** Vendor lock-in, price/quality changes, testability.
- **Options:** direct vendor SDK; own thin interface; LangChain model wrappers; gateway proxy (e.g. LiteLLM-style).
- **Decision:** Thin internal interfaces (`LLMProvider`, `EmbeddingProvider`, `Reranker`) with adapters + fakes; adapters may use vendor SDKs or LangChain wrappers internally.
- **Reasons:** swap providers; deterministic tests; centralize retries, timeouts, token accounting.
- **Disadvantages:** lowest-common-denominator risk (provider-specific features need extension points); maintenance of adapters.
- **Revisit when:** a provider-specific feature (e.g., native citations, structured outputs) offers measurable benefit.
- **Note:** embedding model changes force re-indexing ([DATA_FLOW](DATA_FLOW.md) §2.6).

## ADR-011 — Chunking strategy and vector index selection
- **Context:** Retrieval quality depends on chunking; vector index affects latency/recall.
- **Chunking options:** fixed windows; recursive structure-aware; semantic; parent-child.
- **Vector index options:** exact scan after filter; HNSW; IVFFlat.
- **Decision (provisional):** recursive structure-aware chunking (~500 tokens, small overlap); exact filtered scan first, partial HNSW when `EXPLAIN ANALYZE` + recall checks justify.
- **Reasons:** simplest robust defaults; avoids premature index tuning; per-user corpora are small initially.
- **Disadvantages:** defaults may be suboptimal; HNSW filter-recall caveats; semantic chunking costs more.
- **Revisit when:** benchmark sweeps show a better configuration, corpus sizes grow, or p95 retrieval latency exceeds the target.

## ADR-012 — Infrastructure as code (open)
- **Options:** Terraform; AWS CDK; CloudFormation; console-only (rejected).
- **Leaning:** Terraform for portability and ecosystem; CDK acceptable for a Python-centric team.
- **Decision needed before:** Phase 17 (staging deployment).

---

## Unresolved architectural decisions (prioritized)

| # | Decision | Why it matters | Needed by |
|---|---|---|---|
| 1 | Embedding model + dimension | Fixes DB column, cost, quality, re-index burden | Phase 3 |
| 2 | LLM provider/model and data-retention terms | Privacy, cost, streaming API | Phase 7 |
| 3 | Lexical backend (PG FTS vs. in-memory BM25) and text-search config/languages | Core retrieval quality | Phase 10–11 |
| 4 | PDF/DOCX parsing libraries and OCR stance | Ingestion quality and security | Phase 8 |
| 5 | Reranker: local model vs. hosted API (memory, latency, licensing) | Fargate sizing, cost | Phase 12 |
| 6 | Streaming via CloudFront/ALB verification | Core UX in production | Phase 7 (local) / 17 (AWS) |
| 7 | IaC tool (ADR-012), AWS account strategy | Staging deployment | Phase 17 |
| 8 | Broker durability (Redis vs. SQS) | Reliability | Phase 8 / 17 |
| 9 | Error-tracking/observability stack | Operations | Phase 18 |
| 10 | Malware scanning of uploads | Security posture | Phase 15 |
| 11 | Data retention periods (audit, logs, backups) and user disclosure | Privacy | Phase 15 |
| 12 | Email verification / password reset in MVP or later | Account security | Phase 4 or later |
| 13 | Multi-tenancy/organizations | Schema and retrieval filter | Post-MVP |
| 14 | Dataset size and annotation process for evaluation | Statistical validity | Phase 14 |
