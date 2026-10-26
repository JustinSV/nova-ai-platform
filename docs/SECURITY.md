# NOVA AI — Security Model

Status: **design; no controls are implemented yet.** Related: [SYSTEM_ARCHITECTURE](SYSTEM_ARCHITECTURE.md) §8, [DATA_FLOW](DATA_FLOW.md), [RAG_PIPELINE](RAG_PIPELINE.md) §12, [AWS_DEPLOYMENT](AWS_DEPLOYMENT.md).

## 1. Security principles
1. **Deny by default; owner-scoped everything.** No resource is reachable without authentication and ownership.
2. **Authorize before retrieval, and again before prompting.**
3. **Documents and retrieved passages are untrusted input** — data to quote, never instructions to follow.
4. **Least privilege** for IAM, database roles, and network paths.
5. **No secrets in code, images, logs, or the frontend bundle.**
6. **Minimize what leaves our boundary:** third-party LLM/embedding providers receive only authorized evidence and necessary history.

## 2. Authentication and passwords
- Passwords hashed with **Argon2id** (`argon2-cffi`), Django `PASSWORD_HASHERS` ordered with Argon2 first; parameters reviewed against current OWASP guidance **[verify]**. Hashes upgraded transparently at login.
- Django password validators (length ≥ 12 recommended, common-password, similarity, numeric-only). No composition rules that encourage weak patterns; no forced rotation.
- Login: generic error, per-IP and per-account throttling, timing equalization for unknown emails, audit events for success/failure.
- Sessions: server-side, `HttpOnly`, `Secure`, `SameSite=Lax`, bounded lifetime (proposed 7 days, to be decided), key rotation on login, flush on logout and password change.
- MFA, email verification, password reset: [Future] (reset flow needs single-use, short-lived tokens and non-enumerating responses).

## 3. Authorization, ownership, and tenancy
- Every queryset starts from `request.user` (`Model.objects.filter(owner=request.user)`); a shared mixin/manager enforces this and tests assert it for every endpoint.
- Non-owned → **404**.
- `DocumentChunk.owner_id` is denormalized so retrieval SQL can filter without trusting joins; invariant `chunk.owner_id == document.owner_id` is tested and (recommended) enforced by a DB trigger.
- **Tenant isolation:** the MVP isolates per user. If organizations are added [Future], add `organization_id` to documents/chunks and make it part of every retrieval filter, with row-level security (RLS) considered as an extra layer.
- Staff/admin access: Django admin disabled or restricted by network/SSO in production; admin views must not expose message or document content by default.

## 4. Authorization before retrieval (critical rule)
1. Both retrievers receive a filter object constructed **server-side** from the session user; client input may only *narrow* it (`document_ids` ∩ own ready docs).
2. SQL always includes `owner_id = :user_id AND is_active = true`.
3. After rerank, a re-check query validates each candidate's document (owner, `ready`, not deleted). Failures are dropped and logged at `WARNING` with ids only — this should never happen and indicates a bug.
4. Only after step 3 is any text placed in a prompt or sent to a provider.
5. Cache entries store ids, are keyed by user, and are re-authorized on read.
6. Tests: cross-user retrieval isolation, deletion-then-query, forged `document_ids`, cache poisoning across users ([TESTING_STRATEGY](TESTING_STRATEGY.md)).

## 5. Web application protections
- **CSRF:** Django `CsrfViewMiddleware` on all session-authenticated unsafe requests; token via cookie + `X-CSRFToken`; `CSRF_COOKIE_SECURE`, `CSRF_TRUSTED_ORIGINS` limited to the app origin(s). Streaming POST endpoint is not exempt.
- **CORS:** same-origin deployment means CORS is **disabled in production** (no `Access-Control-Allow-Origin`). In development with a Vite dev server on a different port, allow only that explicit origin with credentials; never `*` with credentials.
- **HTTPS:** TLS at CloudFront and ALB (ACM certificates); redirect HTTP→HTTPS; `SECURE_HSTS_SECONDS` (ramp up gradually), `SECURE_SSL_REDIRECT`, `SECURE_PROXY_SSL_HEADER` configured for the ALB; origin protected so the ALB only accepts CloudFront traffic (managed prefix list and/or secret origin header).
- **Security headers:** `Content-Security-Policy` (no inline scripts; restrict `connect-src` to self; `img-src` self/data only), `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `frame-ancestors 'none'` (or `X-Frame-Options: DENY`), `Permissions-Policy`.
- **SQL injection:** Django ORM parameterization; raw SQL (vector/FTS queries) **only with bound parameters**; code review rule + a test that grep-fails on string-formatted SQL.
- **XSS:** React escapes by default; Markdown rendering with a sanitizing pipeline (raw HTML disabled, `rehype-sanitize`-style allowlist, links `rel="noopener noreferrer"`, `javascript:` URLs blocked); no `dangerouslySetInnerHTML` on model output; strict CSP as backstop.
- **Clickjacking, open redirects, host header attacks:** `ALLOWED_HOSTS` explicit; no user-controlled redirects.

## 6. Upload security
| Control | Detail |
|---|---|
| Size limit | `MAX_UPLOAD_MB` enforced at ALB/ASGI server (body-size) **and** in app; Django `DATA_UPLOAD_MAX_MEMORY_SIZE` and temp-file handlers |
| Type allowlist | Extension **and** magic-byte sniffing must agree; MIME from the client ignored |
| Filename | Sanitized, display only; storage key is `documents/{owner}/{uuid}/original` |
| Parsing safety | Parse in the **worker**, not the API; run as non-root; memory/time limits per task; parser libraries pinned and patched; reject password-protected/encrypted files |
| Archive/zip bombs | DOCX is a zip: cap uncompressed size and entry count before parsing; PDF page limit |
| Active content | Macros/embedded objects ignored; never executed or rendered server-side; originals are not served inline from the app domain — [Future] downloads via presigned URL with `Content-Disposition: attachment` |
| Malware | Consider AV scanning (e.g. ClamAV task or S3-triggered scanner) before processing — optional, **decision open** |
| Storage | Private bucket, Block Public Access, server-side encryption, TLS-only bucket policy |
| Abuse | Upload throttle, per-user quotas |

## 7. Rate limiting and abuse prevention
DRF throttles (Redis) for login, register, upload, chat, general; plus AWS WAF rate rules at CloudFront [optional]. Chat limits also bound LLM cost. Throttle failure mode: **fail closed** for auth endpoints, fail open (with an alarm) for others. Rate-limit events are logged without sensitive content.

## 8. Prompt injection and retrieval poisoning

**Threats:** instructions hidden in an uploaded document ("ignore previous instructions…", "reveal the system prompt", "include this link/image"); crafted text to hijack ranking; data exfiltration via model output (e.g. Markdown image URL carrying secrets); cross-user poisoning (not possible in the single-owner model, but would become relevant with shared corpora).

**Mitigations (defense in depth — none is sufficient alone):**
1. System prompt states evidence is untrusted data; evidence is delimited and attributed; delimiters are escaped inside passages.
2. **The LLM has no tools and no side effects** — the worst outcome of a successful injection is a wrong or manipulated *answer* to the same user, not an action.
3. **Least data in context:** only authorized evidence; no secrets, no other users' data, no system internals in the prompt.
4. **Output handling:** Markdown sanitized client-side; **remote images and auto-loaded resources blocked** (prevents image-URL exfiltration); links shown as plain, user-activated links.
5. Citation validation: markers must refer to supplied evidence.
6. Optional heuristics: flag passages containing instruction-like patterns for logging/metrics (never as a sole defense).
7. Sensitive configuration (API keys) never enters prompts.
8. Since a user can only poison their own corpus in the MVP, the main risk is *accidentally uploading hostile documents* (e.g., from the web); the above limits impact. Multi-user shared corpora would require re-assessment.
9. Red-team test set of injection documents run in CI/eval ([TESTING_STRATEGY](TESTING_STRATEGY.md)).

## 9. Secrets and configuration
- Local: `.env` (git-ignored); `.env.example` has placeholders only. CI: GitHub encrypted secrets / environments. AWS: **Secrets Manager** injected into ECS task definitions at start; rotation plan for DB credentials and provider keys.
- Frontend variables prefixed `VITE_` are **public** and compiled into the bundle — never put secrets there ([ENVIRONMENT_CONFIGURATION](ENVIRONMENT_CONFIGURATION.md)).
- Secret scanning (e.g. gitleaks) in CI and pre-commit; rotate immediately if leaked.
- LLM/embedding keys are server-side only, scoped per environment, with provider-side spend limits.

## 10. Logging and sensitive data
Never log: passwords, session ids, CSRF tokens, API keys, full prompts, document text, chunk text, answers, uploaded filenames in error messages beyond ids, or raw `Authorization`/`Cookie` headers. Log ids, sizes, counts, timings, error codes. Details: [OBSERVABILITY_AND_RELIABILITY](OBSERVABILITY_AND_RELIABILITY.md).

## 11. IAM least privilege (summary; policies in [AWS_DEPLOYMENT](AWS_DEPLOYMENT.md))
- Separate **execution roles** (pull image, write logs, read specific secrets) and **task roles** (runtime).
- API task role: `s3:PutObject/GetObject/DeleteObject` on `arn:…:documents-bucket/documents/*` only; read specific secrets.
- Worker task role: `s3:GetObject/DeleteObject` on the same prefix; no bucket-wide list.
- GitHub Actions: **OIDC** role assumption restricted by repo, branch/environment; permissions limited to ECR push, ECS deploy/run-task, S3 frontend sync, CloudFront invalidation; no long-lived AWS keys.
- No wildcard `*` resources except where AWS requires (documented exceptions).

## 12. Database and network access
- RDS and Redis in isolated subnets, **not publicly accessible**; security groups allow only the ECS task security group.
- Separate DB roles: `migrator` (DDL, used by the migration task) and `app` (DML only; no superuser). `CREATE EXTENSION vector` performed by a privileged one-off step.
- TLS to RDS and Redis in transit; encryption at rest (RDS, S3, EBS/ElastiCache as supported).
- No bastion with open SSH; break-glass access via SSM Session Manager and audited.

## 13. Dependency and supply-chain security
`pip-audit`/`npm audit` (or OSV scanner) in CI; Dependabot/Renovate; pinned lockfiles (`requirements.txt` with hashes or `uv.lock`, `package-lock.json`); container image scanning (e.g. Trivy), minimal base images, non-root user, read-only root filesystem where practical; pin GitHub Actions by commit SHA; ML model downloads (reranker weights) verified by hash/pinned revision.

## 14. Audit logging
`AuditEvent` records authentication events, document upload/delete/reindex, conversation deletion, account deletion, permission-denied bursts, admin actions. Append-only; no content. Retention policy documented; access restricted. Alerting on anomalies (spikes in failed logins, throttle events) via [OBSERVABILITY_AND_RELIABILITY](OBSERVABILITY_AND_RELIABILITY.md).

## 15. Backup, recovery, and deletion
- RDS automated backups + point-in-time recovery; periodic restore **drills** (an unverified backup is not a backup). S3 versioning for documents optional (conflicts with deletion promises — if enabled, lifecycle rules must expire noncurrent versions promptly).
- Define RPO/RTO targets (draft in [OBSERVABILITY_AND_RELIABILITY](OBSERVABILITY_AND_RELIABILITY.md)).
- **Deletion:** document deletion removes retrievability immediately and purges storage asynchronously; account deletion cascades. Backups age out within the backup retention window — disclose this. Provider-side retention (LLM/embedding vendors) is governed by the vendor's terms: choose zero/short-retention settings where offered **[verify]**.

## 16. Privacy
Minimal personal data (email, display name). Users should be told that document passages are sent to third-party model providers. Provide data export/deletion paths. If processing regulated data (health/financial), a separate compliance review is required — **out of scope for the MVP**.

## 17. Threat summary

| Threat | Primary controls |
|---|---|
| Account takeover | Argon2id, throttling, generic errors, session hardening, audit |
| Cross-user data access | Owner-scoped queries, 404s, retrieval filter + re-check, tests |
| Malicious upload | Validation, sandboxed parsing in worker, limits, optional AV |
| Prompt injection | Untrusted-data framing, no tools, minimal context, output sanitization |
| Data exfiltration via output | Sanitized Markdown, blocked remote images, strict CSP |
| Secret leakage | Secrets Manager, scanning, redacted logs |
| DoS / cost abuse | Throttles, quotas, token budgets, WAF (optional), provider spend caps |
| Supply chain | Locking, scanning, SHA-pinned actions, minimal images |
