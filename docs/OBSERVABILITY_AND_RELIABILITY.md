# NOVA AI — Observability and Reliability

Status: **design; nothing instrumented yet.** All targets below are **draft goals to be validated with measured baselines**, not guarantees. Related: [AWS_DEPLOYMENT](AWS_DEPLOYMENT.md) §10, [SECURITY](SECURITY.md) §10, [DATA_FLOW](DATA_FLOW.md).

## 1. Structured logging
- JSON logs to stdout (one event per line), collected by CloudWatch. Library: `structlog` or stdlib `logging` with a JSON formatter **[choose]**.
- Standard fields: `timestamp`, `level`, `service` (api/worker/beat), `env`, `version` (image tag), `request_id`, `trace_id` (if tracing), `user_id` (UUID only), `event` (stable snake_case name), `duration_ms`, `error_code`, plus event-specific ids (`conversation_id`, `message_id`, `document_id`, `task_id`).
- Levels: `INFO` for business events, `WARNING` for degraded/recoverable (fallback used, authorization re-check drop), `ERROR` for failures needing attention.
- Sampling for high-volume debug events; log retention set per environment.

## 2. Request and trace identifiers
- Middleware generates/accepts `X-Request-ID` (validated format; regenerated if malformed), binds it to the logging context, returns it in the response and in error bodies.
- Celery tasks receive `request_id` as a task header so ingestion logs link to the upload request.
- Chat turns log one **summary event** (`chat_turn_completed`) containing all stage timings and counts, plus per-stage events at `DEBUG`.
- Optional OpenTelemetry tracing [Future]: spans for `retrieval.lexical`, `retrieval.dense`, `rrf`, `rerank`, `llm.generate`.

## 3. Metrics to collect

| Metric | Type | Dimensions (low cardinality) |
|---|---|---|
| `api_request_duration_ms` | histogram | route template, method, status class |
| `api_requests_total` | counter | route, status class |
| `chat_time_to_first_token_ms` | histogram | config_id, degraded |
| `chat_total_duration_ms` | histogram | config_id, outcome (complete/cancelled/failed/abstained) |
| `retrieval_duration_ms` | histogram | stage (lexical, embed, dense, fuse, rerank, authorize) |
| `retrieval_candidates` | histogram | stage |
| `retrieval_evidence_count` | histogram | — |
| `retrieval_empty_total` | counter | — (fed into abstention-rate dashboards) |
| `llm_request_duration_ms`, `llm_errors_total` | histogram/counter | provider, model, error class |
| `llm_tokens_total` | counter | direction (prompt/completion), model |
| `llm_estimated_cost_total` | counter | model (computed from a configurable price table; labeled *estimate*) |
| `embedding_requests_total`, `embedding_tokens_total`, `embedding_errors_total` | counters | provider, model, context (ingest/query) |
| `ingestion_documents_total` | counter | outcome (ready/failed), error_code |
| `ingestion_duration_ms` | histogram | stage (extract, chunk, embed, write) |
| `ingestion_queue_depth`, `ingestion_oldest_task_age_s` | gauge | — |
| `celery_task_retries_total`, `celery_task_failures_total` | counters | task name |
| `reconciler_requeued_total` | counter | reason |
| `throttle_events_total` | counter | scope (login, chat, upload…) |
| `auth_failures_total` | counter | reason class |
| `authorization_recheck_drops_total` | counter | — (**must stay zero**; alert on any) |
| `degraded_responses_total` | counter | reason (embed_fail, rerank_fail, lexical_fail) |
| `feedback_total` | counter | rating, reason_code |
Never use `user_id` or document ids as metric dimensions (cardinality/privacy). Emit via CloudWatch Embedded Metric Format or a Prometheus-style endpoint scraped by an agent **[choose]**.

## 4. Error monitoring
Central error tracking (Sentry or CloudWatch-based) **[decision open]**: capture unhandled exceptions with request id, release version, scrubbed context. Configure PII scrubbing (`send_default_pii=False`, drop request bodies, cookies, headers). Group by error code; release health tracking.

## 5. Health checks
- `GET /health/live` — process liveness only.
- `GET /health/ready` — DB connectivity (and migration state), Redis reachability; does not call LLM/embedding providers (avoid cost and cascading failures). Provider health is tracked via error-rate metrics and a **separate** synthetic canary in staging/production (a periodic low-cost query) [optional].
- Worker health: Celery `inspect ping`-based or heartbeat metric; queue depth/age alarms.

## 6. Alerting (proposed; thresholds to be tuned after baselines)

| Alert | Condition (illustrative) | Severity |
|---|---|---|
| API availability | ALB 5xx ratio above threshold for N minutes; unhealthy targets | page |
| Chat latency | p95 time-to-first-token above target for N minutes | ticket → page if sustained |
| LLM errors | provider error rate above threshold | page/ticket |
| Ingestion failures | failure rate above threshold; oldest queued task age above threshold | ticket |
| Queue backlog | queue depth growing for N minutes | ticket |
| Worker down | no task completions with non-empty queue | page |
| DB | CPU, connections, free storage, replica/backup failures | page |
| Redis | memory > threshold, evictions | ticket |
| Security | spike in failed logins/throttles; **any** `authorization_recheck_drops_total > 0` | page |
| Cost | daily LLM/embedding spend above budget; AWS Budget anomaly | ticket |
| Deployment | rollback triggered; migration failed | notify |
Each alert links to a runbook entry (what it means, how to diagnose, how to mitigate).

## 7. What to log — and not to log

**Do log:** ids (user, conversation, message, document, chunk ids), counts, sizes, durations, status/error codes, model names and token counts, config/prompt versions, retry attempts, throttle scope, degraded reasons, deployment version.

**Do not log:** passwords, hashes, session ids, CSRF tokens, API keys, `Authorization`/`Cookie` headers, full request/response bodies, user questions, prompts, retrieved passages, answers, document text, uploaded filenames in free-text error messages, feedback comments, email addresses in routine logs (use user id).
- Debug-only content logging is **off by default**, forbidden in production, and, if ever enabled for an incident, time-boxed with redaction and approval.
- Provide a central log-redaction filter and a test that asserts sample secrets/text never appear in captured logs ([TESTING_STRATEGY](TESTING_STRATEGY.md) §16).
- Evaluation traces for quality analysis use the evaluation corpus (non-sensitive), not production content.

## 8. Reliability design patterns used
Timeouts on every external call (DB, Redis, S3, providers, reranker) with explicit budgets; bounded retries with exponential backoff and jitter; **idempotent** tasks and idempotent client submissions (`client_message_id`); graceful degradation (lexical-only, no rerank); circuit breaker around providers (open after consecutive failures to fail fast with a clear error) [optional]; bulkheads (separate worker pool/queue for ingestion vs maintenance); backpressure (throttles, quotas, queue-depth-aware upload limits); reconciler for lost tasks; graceful shutdown handling for streams and tasks.

## 9. Reliability targets (draft, to be validated)

These are goals for discussion; no measurements exist.

| Area | Draft target | Notes |
|---|---|---|
| API availability (non-chat) | 99.5 % monthly (initial) | Multi-AZ production only |
| Chat success rate (excluding user cancellations and abstentions) | ≥ 99 % | Depends on provider |
| Time to first token | p95 < a few seconds (value set after baseline) | Includes retrieval + provider latency |
| Ingestion latency | p95 for a typical (~10-page) document within minutes | Provider-limited |
| Ingestion success rate (valid files) | ≥ 99 % | Excludes invalid/encrypted files |
| Recovery point objective (RPO) | ≤ 5 minutes DB (PITR) | RDS automated backups **[verify]** |
| Recovery time objective (RTO) | hours (initial), improve later | Restore drill validates |
| Authorization re-check drops | 0 | Any occurrence is an incident |
Review quarterly; convert to formal SLOs with error budgets once traffic exists.

## 10. Runbooks (to be written)
Provider outage; ingestion backlog; DB failover/restore; Redis loss (requeue procedure); failed migration; rollback; suspected data exposure (disable chat, rotate secrets, audit review); cost spike.
