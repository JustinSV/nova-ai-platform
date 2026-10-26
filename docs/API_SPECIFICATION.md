# NOVA AI — REST API Specification (v1)

Status: **proposed contract; no endpoints implemented.** Consistent with [DATABASE_DESIGN](DATABASE_DESIGN.md) and [DATA_FLOW](DATA_FLOW.md).

## 0. Conventions

- **Base path:** `/api/v1`. Breaking changes → `/api/v2`; additive changes stay in v1.
- **Format:** JSON (`application/json`), UTF-8, ISO-8601 UTC timestamps, UUID ids. Exception: the chat endpoint streams `text/event-stream` (§7).
- **Auth:** session cookie. Unsafe methods (`POST/PUT/PATCH/DELETE`) require header `X-CSRFToken` (value from the `csrftoken` cookie). Endpoints marked **Public** need no session.
- **Authorization:** all resources are owner-scoped. Accessing another user's resource returns **404 `not_found`**.
- **Pagination (list endpoints):** cursor-based. `?limit=20` (default 20, max 100) and `?cursor=<opaque>`. Response: `{"results": [...], "next": "<cursor>|null"}`.
- **Filtering/sorting:** documented per endpoint; unknown params are ignored; invalid values → 422.
- **Request id:** every response carries `X-Request-ID`; errors repeat it in the body.
- **Rate limiting:** DRF throttles backed by Redis; exceeded → `429` with `Retry-After`. Proposed defaults (tune with data): login 5/min per IP+email, register 10/hour per IP, uploads 20/hour per user, chat 20/min per user, general 120/min per user. Headers: `Retry-After`.
- **Internal details** (storage keys, embedding vectors, scores beyond what the UI needs, task ids) are never exposed.

### Error format (all non-2xx JSON responses)
```json
{
  "error": {
    "code": "validation_error",
    "message": "One or more fields are invalid.",
    "details": { "email": ["Enter a valid email address."] },
    "request_id": "9f4c2e1a-7b0d-4c53-a1e6-0d2f5b8a1c33"
  }
}
```

### Status codes used
`200 OK`, `201 Created`, `202 Accepted` (async work started), `204 No Content`, `400 Bad Request` (malformed), `401 Unauthorized`, `403 Forbidden` (CSRF/permission), `404 Not Found`, `409 Conflict` (duplicates, idempotency, state), `413 Payload Too Large`, `415 Unsupported Media Type`, `422 Unprocessable Entity` (validation), `429 Too Many Requests`, `500`, `503 Service Unavailable`.

### Common error codes
`validation_error`, `invalid_credentials`, `not_authenticated`, `csrf_failed`, `not_found`, `throttled`, `duplicate_document`, `quota_exceeded`, `file_too_large`, `unsupported_media_type`, `invalid_state`, `provider_unavailable`, `internal_error`.

---

## 1. Authentication

### 1.1 `GET /auth/csrf` — Public
Purpose: set the CSRF cookie. **Response `204`** + `Set-Cookie: csrftoken=…`.

### 1.2 `POST /auth/register` — Public
Purpose: create account and log in.
Request:
```json
{ "email": "ada@example.com", "password": "correct horse battery staple", "display_name": "Ada" }
```
Validation: email valid, ≤254, unique (case-insensitive); password passes Django validators (min 12 recommended); display_name 1–80 chars.
**201:**
```json
{ "id": "5c1d0f6e-2c43-4b7e-9d8e-3a0c7f3c9a11", "email": "ada@example.com", "display_name": "Ada", "date_joined": "2026-01-15T09:30:00Z" }
```
Errors: `422 validation_error`, `409 email_unavailable`, `429 throttled`, `403 csrf_failed`.

### 1.3 `POST /auth/login` — Public
Request: `{ "email": "...", "password": "..." }`
**200:** same profile body as 1.2 + session cookie.
Errors: `401 invalid_credentials` (generic), `429 throttled`, `403 csrf_failed`.

### 1.4 `POST /auth/logout` — Authenticated
Flushes the session. **204.** Errors: `401`, `403 csrf_failed`.

---

## 2. User profile

### 2.1 `GET /users/me` — Authenticated
**200:** profile (as 1.2). Errors: `401`.

### 2.2 `PATCH /users/me` — Authenticated
Request: `{ "display_name": "Ada L." }` (email/password change are separate, [Future]). **200:** updated profile. Errors: `422`.

### 2.3 `DELETE /users/me` — Authenticated
Purpose: delete account and all data. Request: `{ "password": "..." }` (re-authentication). **202** `{ "status": "deletion_scheduled" }`; DB rows removed in-request, S3 purge asynchronous; session ended. Errors: `401 invalid_credentials`, `422`.

---

## 3. Conversations

Resource:
```json
{ "id": "c0a8…", "title": "Q3 contract summary", "created_at": "2026-01-15T10:00:00Z", "updated_at": "2026-01-15T10:07:41Z" }
```

| Method & URL | Purpose | Notes |
|---|---|---|
| `GET /conversations` | List own conversations | `?limit&cursor&q=` (title contains); sorted `updated_at` desc; excludes deleted |
| `POST /conversations` | Create | Body `{ "title": "optional" }` → **201** resource. Title ≤120, defaults to "New chat" and is auto-set from the first question |
| `GET /conversations/{id}` | Retrieve | **200**; 404 if not owner |
| `PATCH /conversations/{id}` | Rename | `{ "title": "..." }` → **200**; 422 if empty/>120 |
| `DELETE /conversations/{id}` | Delete with messages | **204**; idempotent (404 if already gone) |

All: Authenticated; owner-only; throttled with the general scope.

---

## 4. Messages

### 4.1 `GET /conversations/{id}/messages` — Authenticated, owner-only
Query: `limit` (default 50, max 100), `cursor`; ordered by `sequence` ascending; `?order=desc` for loading latest-first.
**200:**
```json
{
  "results": [
    {
      "id": "7d2b…", "sequence": 1, "role": "user", "status": "complete",
      "content": "What is the notice period in the supplier contract?",
      "created_at": "2026-01-15T10:05:00Z", "sources": [], "feedback": null
    },
    {
      "id": "91ac…", "sequence": 2, "role": "assistant", "status": "complete",
      "content": "The notice period is 60 days [1].",
      "created_at": "2026-01-15T10:05:04Z",
      "sources": [
        { "citation_index": 1, "document_id": "e3f1…", "document_title": "supplier-contract.pdf",
          "page_start": 7, "page_end": 7, "excerpt": "…either party may terminate with sixty (60) days written notice…", "cited": true }
      ],
      "feedback": { "rating": 1, "reason_code": null }
    }
  ],
  "next": null
}
```
Errors: `404`, `401`. Internal scores/metadata are not returned.

### 4.2 Sending a message
See **§7 Chat and streaming** (`POST /conversations/{id}/messages`). A message is created only through the chat endpoint; there is no separate "create message" JSON endpoint. Messages are immutable (no edit/delete in MVP).

---

## 5. Documents

Resource (`Document`):
```json
{
  "id": "e3f1b2c4-…", "original_filename": "supplier-contract.pdf", "content_type": "application/pdf",
  "size_bytes": 482113, "status": "ready", "error_code": null, "error_detail": null,
  "page_count": 14, "chunk_count": 42, "created_at": "2026-01-15T09:50:00Z", "ready_at": "2026-01-15T09:50:41Z"
}
```

### 5.1 `POST /documents` — Authenticated; multipart/form-data
Fields: `file` (required). Throttle: upload scope.
Validation: size ≤ `MAX_UPLOAD_MB`; extension ∈ {pdf, docx, txt, md} and sniffed type agrees; non-empty; quota not exceeded.
**202** (`status: "queued"`) with `Location: /api/v1/documents/{id}`.
Errors: `413 file_too_large`, `415 unsupported_media_type`, `409 duplicate_document`
```json
{ "error": { "code": "duplicate_document", "message": "This file was already uploaded.", "details": { "existing_document_id": "e3f1…" }, "request_id": "…" } }
```
`409 quota_exceeded`, `422 validation_error`, `503 storage_unavailable`, `429`.

### 5.2 `GET /documents` — Authenticated
Query: `limit`, `cursor`, `status` (one of the status values), `q` (filename contains), `ordering` (`-created_at` default, `created_at`, `original_filename`, `size_bytes`). **200** paginated list. Deleted/`deleting` documents are excluded.

### 5.3 `GET /documents/{id}` — Authenticated, owner-only
**200** resource. 404 otherwise.

### 5.4 `GET /documents/{id}/status` — Authenticated, owner-only
Purpose: cheap polling. **200:**
```json
{ "id": "e3f1…", "status": "processing", "error_code": null, "error_detail": null, "updated_at": "2026-01-15T09:50:20Z" }
```
Clients poll every 2–5 s with backoff until `ready`/`failed`. (Push updates are [Future].)

### 5.5 `POST /documents/{id}/reindex` — Authenticated, owner-only
Purpose: retry a failed ingestion or rebuild the index. Allowed when `status ∈ {ready, failed}`. **202** resource (`status: "queued"`). Errors: `409 invalid_state` (e.g. already processing), `404`.

### 5.6 `DELETE /documents/{id}` — Authenticated, owner-only
**204.** Retrieval stops immediately; storage purge is asynchronous. Errors: `404`. Idempotent semantics: second call → 404.

### 5.7 `GET /documents/{id}/download-url` — [Future, post-MVP]
Returns a short-lived presigned S3 URL (owner-only, ≤ 5 min) to view the original. Not part of the MVP.

---

## 6. Feedback

### 6.1 `PUT /messages/{message_id}/feedback` — Authenticated, owner-only
Authorization: the message must belong to a conversation owned by the user and be `role=assistant`.
Request:
```json
{ "rating": -1, "reason_code": "not_grounded", "comment": "The cited page does not mention a 60 day period." }
```
Validation: `rating` ∈ {-1, 1}; `reason_code` only allowed/optional for -1; `comment` ≤ 2000.
**200** (upsert): `{ "message_id": "91ac…", "rating": -1, "reason_code": "not_grounded", "comment": "…", "updated_at": "…" }`.
Errors: `404`, `422`.

### 6.2 `DELETE /messages/{message_id}/feedback` — Authenticated, owner-only
**204**; 404 if none.

---

## 7. Chat and streaming

### 7.1 `POST /conversations/{id}/messages` — **Streaming response**
Purpose: submit a question, run retrieval + generation, stream the answer.
Auth: Authenticated, conversation owner. Throttle: chat scope. CSRF required.

Request (JSON):
```json
{
  "content": "What is the notice period in the supplier contract?",
  "client_message_id": "0f8fad5b-d9cb-469f-a165-70867728950e",
  "document_ids": ["e3f1b2c4-…"]
}
```
Validation: `content` 1–`MAX_MESSAGE_CHARS` (proposed 4000) after trim; `client_message_id` UUID (idempotency); `document_ids` optional, ≤ 20, each must be an own `ready` document (else 422); if omitted, all own ready documents are searched.

**Response:** `200 text/event-stream`. Each frame: `id: <n>\nevent: <name>\ndata: <json>\n\n`. Heartbeat comment `: ping` every ~15 s.

| Event | Data |
|---|---|
| `message.created` | `{"user_message_id":"…","assistant_message_id":"…"}` |
| `retrieval.completed` | `{"evidence_count":5,"degraded":false,"abstained":false}` |
| `token` | `{"delta":"The notice"}` |
| `sources` | `{"sources":[{"citation_index":1,"document_id":"…","document_title":"…","page_start":7,"page_end":7,"excerpt":"…","cited":true}]}` |
| `message.completed` | `{"assistant_message_id":"…","status":"complete"}` |
| `error` | `{"code":"provider_unavailable","message":"The assistant is temporarily unavailable."}` |

Example (abbreviated):
```
id: 1
event: message.created
data: {"user_message_id":"7d2b…","assistant_message_id":"91ac…"}

id: 2
event: token
data: {"delta":"The notice period is 60 days "}

id: 9
event: message.completed
data: {"assistant_message_id":"91ac…","status":"complete"}
```
Pre-stream errors (returned as normal JSON before the stream starts): `401`, `403 csrf_failed`, `404` (conversation), `409 duplicate_request` (same `client_message_id` already processed → client should fetch messages), `422`, `429`. Failures **after** the stream starts are delivered as an `error` event.

Notes: no tool-calling or file arguments; the client cannot supply retrieval parameters (server-controlled, preventing cost/abuse).

### 7.2 `POST /conversations/{id}/messages/{message_id}/cancel` — Authenticated, owner-only
Purpose: backstop cancel for a `streaming` assistant message. **202** if cancellation was requested; **200** if already terminal (idempotent). 404 otherwise.

### 7.3 Evaluation
No public evaluation endpoint. Evaluation runs via CLI/CI against the service layer ([RAG_EVALUATION](RAG_EVALUATION.md)).

---

## 8. Health checks (unversioned, no auth)

| Endpoint | Purpose | Success | Failure |
|---|---|---|---|
| `GET /health/live` | Process is up (ALB/ECS liveness); no dependency checks | `200 {"status":"ok"}` | — |
| `GET /health/ready` | Can serve traffic: DB query succeeds, Redis reachable (optional dependency flagged), migrations applied | `200 {"status":"ok","checks":{"database":"ok","redis":"ok"}}` | `503` with failing check names only (no internals) |

Do not include versions, hostnames, or stack traces. Reachable only via ALB/internal paths in production; not routed by CloudFront if undesired.

## 9. Administration

No admin REST API in the MVP. Operations use the Django admin (restricted by network/SSO, disabled or protected in production) and management commands (`reconcile_documents`, `run_benchmark`). A dedicated admin API is [Future].

## 10. Endpoint ↔ model mapping

| Endpoint group | Models |
|---|---|
| Auth, users | User (+ Session, AuditEvent) |
| Conversations/Messages/Chat | Conversation, Message, MessageSource |
| Documents | Document, DocumentChunk (indirect) |
| Feedback | Feedback |
