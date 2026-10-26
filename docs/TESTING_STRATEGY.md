# NOVA AI — Testing Strategy

Status: **plan; no tests exist yet.** Related: [RAG_EVALUATION](RAG_EVALUATION.md), [SECURITY](SECURITY.md), [DEVOPS_AND_CICD](DEVOPS_AND_CICD.md).

## 1. Goals and pyramid

Many fast, deterministic tests at the bottom; fewer slow, costly ones at the top. **No test that calls a real LLM or embedding provider runs in the default PR pipeline.**

| Layer | Tooling (proposed) | Runs in |
|---|---|---|
| Python unit | pytest, hypothesis (for pure functions like RRF/chunking) | every PR |
| Django API | pytest-django, DRF `APIClient` | every PR |
| DB integration | PostgreSQL **with pgvector** (CI service container / testcontainers) | every PR |
| Celery tasks | `task_always_eager` for logic; real broker in a small integration job | every PR / nightly |
| React unit/component | Vitest + React Testing Library | every PR |
| Frontend integration | Vitest + MSW (mocked API incl. SSE) | every PR |
| End-to-end | Playwright against docker-compose stack with **fake providers** | merge to main / nightly |
| Security | bandit, pip-audit, npm audit, ZAP baseline (staging), custom authz tests | PR + scheduled |
| Load/latency | k6 or Locust against staging with fake LLM and (separately) real providers | scheduled / pre-release |
| RAG evaluation | custom metrics + Ragas | retrieval-only on PR; full on schedule/manual |

Never use SQLite for tests: pgvector, FTS, and constraints must be exercised on PostgreSQL.

## 2. Test doubles for AI components
- `FakeEmbeddingProvider`: deterministic hash-based vectors (same text → same vector; overlapping words → similar vectors) so retrieval tests are reproducible without a network.
- `FakeLLMProvider`: scripted responses and token streams (including slow streams, mid-stream failures, malformed citation markers).
- `FakeReranker`: scores by keyword overlap or a fixed table.
- Provider contract tests (optional, scheduled, real keys): verify adapters still match provider APIs.

## 3. Fixtures and data
- `user_factory`, `other_user_factory` (isolation tests), `conversation_factory`, `message_factory` (factory_boy).
- `sample_documents/`: tiny PDF, DOCX, TXT, MD with known content and page numbers; an encrypted PDF; a corrupt PDF; an empty file; an image-only PDF; a zip-bomb-like DOCX (small compressed, huge uncompressed); a document containing prompt-injection text; a document with unicode/RTL/emoji; very long single paragraph; table-heavy page.
- `ingested_corpus` fixture: ingests the sample docs with fake providers and returns chunk ids.
- `freezegun`/time control for token/session expiry and reconciler tests.
- S3: `moto` or MinIO for storage tests.

## 4. Unit tests (Python)
- **Chunker:** respects size/overlap, preserves headings/pages, never empty chunks, deterministic, handles unicode, idempotent re-chunking.
- **Normalizer:** whitespace, control chars, de-hyphenation edge cases.
- **RRF:** see §9.
- **Token budgeting:** evidence trimmed in order; per-chunk cap; empty input.
- **Prompt builder:** delimiter escaping, evidence numbering, history trimming, version stamp.
- **Citation parser:** `[1]`, `[1][2]`, `[1, 2]`, invalid `[9]`, markers inside code blocks, no markers.
- **Config/validators:** upload validators, filename sanitization.

## 5. Django API tests
For **every endpoint** in [API_SPECIFICATION](API_SPECIFICATION.md): success, validation errors (422 with documented body), unauthenticated (401), missing/invalid CSRF (403), wrong owner (404), throttling (429 with `Retry-After`), pagination (cursor stability, `limit` bounds), consistent error envelope, no leakage of internal fields. Contract test: response JSON validated against an OpenAPI schema generated from DRF (e.g. drf-spectacular) — **[verify tooling]**.

## 6. Database integration tests
- Constraints: unique `lower(email)`, `(owner, sha256)` duplicates, `(conversation, sequence)`, CHECK enums, FK behaviors (SET_NULL on MessageSource).
- Concurrency: parallel message creation yields unique sequences; two concurrent identical uploads → one wins, other 409.
- Chunk invariant: `chunk.owner_id == document.owner_id`.
- Index usage: `EXPLAIN` assertions (smoke) for the FTS and vector queries on a moderately sized synthetic dataset.
- Migrations: apply on empty DB; `makemigrations --check`; forward/backward compatibility test between consecutive releases.
- Deletion: cascade (account, conversation), document deactivation then purge, snapshot blanking.

## 7. Authentication and permission tests
- Registration: weak password, duplicate email, case-variant duplicate, throttling.
- Login: wrong password vs unknown email return identical responses/shape; session rotation; logout invalidates session; inactive user.
- **Isolation matrix:** user B attempting every read/write on user A's conversation, message, document, feedback, status, reindex, delete, cancel → always 404, never data.
- Retrieval isolation: user B's query must never return A's chunks even with identical text, forged `document_ids`, or cached results.

## 8. Document ingestion tests
- Happy path per format; page/section metadata correct.
- Validation: oversize, type/extension mismatch, empty, encrypted, corrupt, no-text → expected `error_code`, no retries for permanent errors.
- Idempotency: running `ingest_document` twice yields one active version; stale task (old `pending_index_version`) is a no-op; task for deleted document exits.
- **Retries:** fake embedding provider raises transient errors N times → eventually succeeds; exhausts → `failed(provider_unavailable)`; no duplicate active chunks.
- **Crash simulation:** kill after partial chunk writes → rerun cleans inactive chunks and completes.
- Atomic swap: concurrent queries during re-index always see either the old or the new complete version, never a mix.
- Reconciler: stuck `uploaded/queued/processing` documents re-enqueued/failed after timeout.
- Deletion: after `DELETE`, query returns nothing from that document immediately (before purge runs).

## 9. Retrieval and RRF tests
- **RRF unit tests (deterministic):** hand-computed examples (e.g. the worked example in [RAG_PIPELINE](RAG_PIPELINE.md) §10 → 0.0323 vs 0.0161); items in only one list; identical lists; empty list; ties broken by chunk id; `k` variations; property tests: monotonic in rank, permutation of retriever order does not change result.
- **Lexical retrieval:** exact identifiers, quoted phrases, stemming behavior, special characters (`'`, `&`, `:`), SQL-injection strings return safely.
- **Dense retrieval (fake embeddings):** nearest neighbor correctness; ordering stable; filter respected; approximate index vs exact recall check on synthetic data.
- **Hybrid:** a query solvable only lexically, one only semantically, and one both; assert the fused list contains the expected chunk.
- **Rerank:** fake reranker reorders; timeout path falls back to RRF order with `degraded`.
- **Evidence selection:** budget enforcement, de-duplication, authorization re-check drops forbidden chunk.

## 10. Citation correctness tests
- Every `[n]` in a persisted answer maps to a `MessageSource` with `citation_index = n`.
- Markers referencing non-existent evidence are removed/flagged; `cited` flags correct.
- Snapshot fields populated; after document deletion, snapshots blanked but message intact.
- *Support* checks (does the cited passage actually support the claim?) are evaluation-level tests ([RAG_EVALUATION](RAG_EVALUATION.md)) — a lexical-overlap heuristic can run in CI; model-based judging only on schedule.

## 11. Testing non-deterministic AI behavior reproducibly
1. **Layer separation:** all logic around the model (retrieval, budgets, parsing, persistence) is tested deterministically with fakes.
2. **Scripted LLM:** a fake provider returns canned output to test streaming, errors, cancellation, citation handling.
3. **Property/invariant assertions** on real model outputs instead of exact text: contains valid citations only, abstains on unanswerable set, no instruction leakage, response language, length bounds.
4. **Pinned settings:** temperature 0 (or low), fixed model version, fixed prompt version, seed where supported (not guaranteed identical).
5. **Repeat and aggregate:** run each evaluation question n times; report mean and variance; use thresholds with tolerance, never exact-match.
6. **Record/replay (VCR-style)** for provider calls in regression suites — recorded cassettes scrubbed of keys; refreshed deliberately.
7. **Multiple evidence types:** deterministic metrics (retrieval recall/MRR, citation validity) as gating; LLM-judged metrics (Ragas faithfulness, etc.) as advisory/trend signals plus **human spot review** — do not rely on LLM-as-judge alone.

## 12. Failure and retry tests
Provider timeout before first token (retry once), mid-stream failure (error event + partial persisted), embedding failure at query time (lexical fallback, `degraded`), reranker timeout, Redis down (upload accepted, document stays `uploaded`, reconciler recovers), DB error during final persistence (message `failed`), S3 failure during upload (503, no `queued` state), duplicate `client_message_id`.

## 13. Streaming tests
Event order and format; heartbeat emission; client abort → status `cancelled` and provider stream closed (assert via fake provider flag); cancel endpoint idempotency; very long answers; proxy-buffering check in staging (first token arrives before completion).

## 14. React tests
- **Component:** message list (Markdown, code highlighting, citation chips), upload dropzone (client-side validation messages), document status badges, auth forms (error display), a11y checks (jest-axe).
- **Hooks/services:** API client adds CSRF header; 401 → redirect; TanStack Query retry/refetch policies; SSE parser handles split chunks, multi-line `data`, unknown events, abort.
- **Integration (MSW):** login → create conversation → send message streaming → citations visible → feedback; upload → polling → ready/failed; sanitization test: model output with `<script>`, `<img src=http://…>`, `javascript:` links renders inert.

## 15. End-to-end (Playwright)
Docker-compose stack with fake providers (`PROVIDER_MODE=fake`): register → upload sample PDF → wait ready → ask question → see streamed answer with citation → click citation → give feedback → delete document → ask again → abstention. Cross-browser smoke (Chromium + one other). Run against staging after deploy with a dedicated test account (smoke only, real providers optional).

## 16. Security tests
- Automated: bandit, pip-audit, npm audit, secret scan, container scan, ZAP baseline on staging.
- Custom: CSRF required on all unsafe routes; session cookie flags; security headers present; open-redirect and host-header checks; SQL-injection strings in search and filters; upload attacks (polyglot files, zip bomb, path traversal filenames like `../../x.pdf`, double extensions); SSRF (no feature should fetch user-supplied URLs); rate-limit enforcement; log-redaction test (secrets/document text never appear in captured logs).
- Prompt-injection suite: documents containing hostile instructions; assert no tool/side effect exists, system prompt not leaked (best-effort check), output sanitized, citations valid.

## 17. Load and latency tests
Scenarios: N concurrent chat streams with fake LLM (measures API/DB capacity and streaming connection limits), retrieval-only latency vs corpus size, ingestion throughput with fake embeddings, upload bursts, queue backlog recovery. Measure p50/p95/p99 per stage using the metrics defined in [OBSERVABILITY_AND_RELIABILITY](OBSERVABILITY_AND_RELIABILITY.md). Targets are set **after** a baseline is measured; no numbers are promised in this document.

## 18. Edge cases checklist
Empty/whitespace question; 4000+ char question; non-English and mixed-language queries; emoji; query with only stopwords; question about a deleted document; conversation with zero documents; 1,000-message conversations (history trimming); duplicate uploads; same text different filename; document with identical chunks repeated; PDF with 500 pages; concurrent re-index and chat; embedding model changed mid-flight; clock skew; cancelled stream followed by new message.

## 19. Coverage and quality gates (proposed)
Line coverage tracked, with a **meaningful floor on `ai/` and permission code** (e.g. branch coverage there), rather than a global vanity number; mutation testing on RRF/budget/citation modules [optional]; flaky-test quarantine policy; every bug fix adds a regression test.
