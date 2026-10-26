# NOVA AI — Project Overview

Status: **requirements and design stage. No code, tests, or infrastructure exist.** Related: [SYSTEM_ARCHITECTURE](SYSTEM_ARCHITECTURE.md), [IMPLEMENTATION_ROADMAP](IMPLEMENTATION_ROADMAP.md).

## 1. What NOVA AI is
NOVA AI is a full-stack, GPT-style conversational application that lets a user **upload their own documents, ask questions about them, and receive answers grounded in those documents with source citations**, while managing their conversations and documents. It is built with React/TypeScript, Django/DRF, PostgreSQL (pgvector), Celery/Redis, and AWS, and doubles as a demonstration of professional full-stack Python engineering.

## 2. Why it exists / problems it solves
1. People need an easy way to search and understand their own documents.
2. Keyword search misses meaning; pure semantic search misses exact terms — hybrid retrieval addresses both (to be **verified by benchmark**).
3. LLMs hallucinate; answers must be grounded, cited, and able to say "not found".
4. Many RAG demos lack permission enforcement, evaluation, monitoring, and tests; NOVA AI treats these as first-class.
5. Real applications need secure APIs, reliable background processing, CI/CD, and deployable infrastructure.

## 3. Target users and use cases
| User | Use case |
|---|---|
| Individual knowledge worker / student | Ask questions across personal notes, PDFs, reports |
| Small team member (later) | Search shared policy/handbook documents *(multi-user sharing is [Future])* |
| Recruiter/engineer reviewing the project | Evaluate architecture, code quality, evaluation rigor |
| Developer/maintainer | Extend retrieval, run benchmarks, deploy |
Example flows: summarize a contract clause with page citation; find where a term is defined; compare statements across two uploaded documents; verify that an answer is supported.

## 4. Functional requirements

### 4.1 MVP (initial release) — **[Requirement]**
| ID | Requirement |
|---|---|
| FR-1 | Register, log in, log out; manage basic profile |
| FR-2 | Create, list, rename, delete conversations; view message history |
| FR-3 | Send a question and receive a **streamed** answer in a chat UI with Markdown rendering |
| FR-4 | Upload PDF/DOCX/TXT/MD documents; see processing status; list, delete, retry |
| FR-5 | Background ingestion: extract, chunk, embed, index |
| FR-6 | Answers grounded in the user's documents only, with **citations** to document/page and visible excerpts |
| FR-7 | Abstain when evidence is insufficient |
| FR-8 | Permission enforcement: users only ever retrieve their own documents |
| FR-9 | Thumbs up/down feedback on answers |
| FR-10 | Cancel a running generation |
| FR-11 | Delete account and data |
| FR-12 | Health checks, structured logs, basic metrics |

### 4.2 Advanced release — **[Requirement per brief; sequenced after MVP]**
Hybrid retrieval (BM25/lexical + dense) with RRF; cross-encoder reranking; query rewriting for follow-ups; LangGraph workflow (only if justified); Ragas evaluation and benchmarking with CI regression gates; audit events; AWS staging + production deployment with monitoring.

### 4.3 Future enhancements — **[Future, not committed]**
Multi-tenancy/sharing, OCR for scanned PDFs, more file types, presigned direct uploads, document viewer with highlighted citations, resumable streams, email verification/password reset/MFA, conversation summaries, hosted reranker, admin API, SSO, mobile client, conversation export.

## 5. Non-functional requirements
| Area | Requirement |
|---|---|
| Security | Authorization before retrieval; untrusted-content handling; secure defaults ([SECURITY](SECURITY.md)) |
| Privacy | Minimal data to third-party providers; deletion honored |
| Reliability | Idempotent ingestion; graceful degradation; documented recovery ([OBSERVABILITY_AND_RELIABILITY](OBSERVABILITY_AND_RELIABILITY.md)) |
| Performance | Draft targets only, set after baselines (no guarantees claimed) |
| Maintainability | Modular monolith, typed code, interface-based providers, enforced module rules |
| Testability | Deterministic fakes for AI components; layered tests ([TESTING_STRATEGY](TESTING_STRATEGY.md)) |
| Observability | Request ids, stage timings, token/cost metrics, no sensitive content in logs |
| Portability | Runs locally with Docker Compose; deployable to AWS |
| Accessibility/usability | Keyboard-accessible chat UI, responsive layout |
| Cost awareness | Token budgets, throttles, spend alarms |

## 6. Scope

**In scope:** single-owner document Q&A, citations, streaming chat, ingestion pipeline, evaluation framework, CI/CD, AWS deployment design.

**Explicitly out of scope (initially):** Kubernetes, Kafka, microservices, multiple vector databases, autonomous agents/tools, web browsing, answering from general knowledge outside the documents, document sharing/collaboration, OCR, real-time multi-user features, regulated-data compliance (HIPAA/PCI), native mobile apps, fine-tuning models.

## 7. Assumptions and constraints
- **Assumptions to verify:** an acceptable LLM/embedding provider with suitable data terms; RDS supports the needed pgvector version; CloudFront/ALB pass SSE reliably; parsing libraries handle sample documents; reranker fits the chosen Fargate size.
- **Constraints:** stack fixed by the brief; modular monolith; no extra infrastructure without ADR; cost-conscious (single developer).
- Document types limited to text-extractable files; per-file size limit (default 20 MB, configurable).
- English-first retrieval tuning; multilingual support must be evaluated before claiming.

## 8. Acceptance criteria (MVP)
1. A new user can register, upload a PDF, wait for `ready`, ask a question, and see a streamed, cited answer.
2. A question unanswerable from the documents yields an abstention, not a fabricated answer.
3. A second user cannot retrieve, see, or cite the first user's documents (automated test).
4. Deleting a document makes it immediately unretrievable and removes stored content.
5. Failed ingestion shows a clear status and can be retried.
6. Tests pass in CI; staging deployment reachable over HTTPS; documented rollback.
7. No secrets in the repo; documentation matches implemented behavior.

## 9. Definition of done (feature level)
Code merged via reviewed PR; unit + integration tests added; API docs/OpenAPI updated; security checklist consulted; logging/metrics added without sensitive data; migration backward compatible; documentation updated; deployed to staging and smoke-tested.

## 10. MVP vs. advanced release

| Capability | MVP | Advanced |
|---|---|---|
| Auth (session + CSRF) | ✔ | MFA, reset (future) |
| Chat + streaming | ✔ | resumable streams (future) |
| Upload + ingestion | ✔ | OCR, more formats (future) |
| Retrieval | Dense baseline (hybrid optional if measured early) | BM25 + dense + RRF + rerank |
| Query rewriting | — | ✔ |
| LangGraph | — | only if justified |
| Citations | ✔ | highlighted viewer (future) |
| Evaluation | small dataset, manual checks | Ragas, benchmarks, CI gates |
| Audit log | minimal | ✔ |
| Deployment | Docker Compose + staging | production + monitoring |
