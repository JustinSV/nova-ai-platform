# NOVA AI — Documentation Index

Everything here is **design documentation for an unimplemented project.** Status tags used throughout: **[Requirement]**, **[Recommendation]**, **[Assumption]** (verify), **[Future]**.

## Reading order

| # | Document | Read it to… |
|---|---|---|
| 1 | [../README.md](../README.md) | Get the pitch, structure, and intended setup |
| 2 | [PROJECT_OVERVIEW.md](PROJECT_OVERVIEW.md) | Understand scope, requirements, MVP vs. advanced |
| 3 | [SYSTEM_ARCHITECTURE.md](SYSTEM_ARCHITECTURE.md) | See components, boundaries, trade-offs |
| 4 | [DATA_FLOW.md](DATA_FLOW.md) | Follow auth, ingestion, chat, streaming, feedback, CI/CD flows |
| 5 | [DATABASE_DESIGN.md](DATABASE_DESIGN.md) | Implement models, indexes, migrations |
| 6 | [API_SPECIFICATION.md](API_SPECIFICATION.md) | Implement/consume the REST + SSE API |
| 7 | [RAG_PIPELINE.md](RAG_PIPELINE.md) | Implement retrieval and generation |
| 8 | [AI_WORKFLOW_DESIGN.md](AI_WORKFLOW_DESIGN.md) | Decide where LangChain/LangGraph belong |
| 9 | [SECURITY.md](SECURITY.md) | Apply security controls |
| 10 | [TESTING_STRATEGY.md](TESTING_STRATEGY.md) | Plan tests, including non-deterministic AI |
| 11 | [RAG_EVALUATION.md](RAG_EVALUATION.md) | Run Ragas and the C1–C4 benchmark |
| 12 | [DEVOPS_AND_CICD.md](DEVOPS_AND_CICD.md) | Build pipelines, images, releases |
| 13 | [AWS_DEPLOYMENT.md](AWS_DEPLOYMENT.md) | Provision and operate AWS |
| 14 | [OBSERVABILITY_AND_RELIABILITY.md](OBSERVABILITY_AND_RELIABILITY.md) | Instrument and alert |
| 15 | [ENVIRONMENT_CONFIGURATION.md](ENVIRONMENT_CONFIGURATION.md) | Configure environments safely |
| 16 | [DECISIONS_AND_TRADEOFFS.md](DECISIONS_AND_TRADEOFFS.md) | Review ADRs and alternatives |
| 17 | [IMPLEMENTATION_ROADMAP.md](IMPLEMENTATION_ROADMAP.md) | Build in dependency order |

## Canonical vocabulary (used consistently)

| Term | Meaning |
|---|---|
| Document | Uploaded file metadata row; original lives in S3 |
| Chunk (`DocumentChunk`) | Retrieval unit with text, `tsvector`, embedding, `owner_id` |
| `index_version` / `is_active` | Versioning that lets re-indexing swap atomically |
| Lexical retrieval | BM25-style keyword retrieval (PG FTS by default; true BM25 optional) |
| Dense retrieval | pgvector similarity |
| RRF | Reciprocal Rank Fusion, `Σ 1/(k + rank)` |
| Evidence | Authorized chunks selected for the prompt |
| Retrieval epoch | Per-user counter that invalidates retrieval caches |
| C1–C4 | Benchmark configs: lexical, dense, hybrid+RRF, hybrid+RRF+rerank |
| MVP | Phases 1–9 + hardening/staging subset |

## Quality-checklist self-review (as written)

| Check | Result |
|---|---|
| All requested Markdown files created | ✔ (A–Q + `INDEX.md`) |
| Internal links relative and pointing at existing files | ✔ intended; run the link check in CI (`docs` job) to confirm |
| Mermaid diagrams | ✔ written with quoted labels; **not machine-validated here — run `mermaid-cli` (CI `docs` job)** |
| ER model ↔ API ↔ data flow consistency | Reviewed manually (ids, statuses, `index_version`, `client_message_id`, sources) |
| Security controls reflected in retrieval/upload/chat | ✔ (owner filter, re-check, sanitization, no tools) |
| CI/CD matches AWS design | ✔ (migrate task, ECS, S3/CloudFront, OIDC) |
| Roadmap respects dependencies | ✔ (graph in roadmap) |
| No invented benchmarks, prices, or successful deployments | ✔ — all numbers are labeled placeholders/hypotheses |
| No real credentials | ✔ |

Known imperfections: library/service behaviors flagged **[verify]** are unconfirmed; Mermaid syntax not tool-validated; parameter defaults are hypotheses.

## Prioritized unresolved architectural decisions

1. **Embedding model and dimension** (fixes DB schema, cost, re-index burden) — before Phase 3.
2. **LLM provider/model and data-retention terms** — before Phase 7.
3. **Lexical backend** (PG FTS vs. in-memory BM25) and text-search language config — Phases 10–11, decided by benchmark.
4. **Parsing libraries and OCR stance** — Phase 8.
5. **Reranker hosting** (local vs. hosted; Fargate sizing, licensing) — Phase 12.
6. **SSE through CloudFront/ALB** (timeouts, buffering, disconnect propagation) — verify locally in Phase 7, on AWS in Phase 17.
7. **IaC tool and AWS account strategy** — before Phase 17.
8. **Broker durability** (Redis vs. SQS) — Phases 8/17.
9. **Error tracking/observability stack** — Phase 18.
10. **Upload malware scanning** — Phase 15.
11. **Retention periods** (audit, logs, backups) and user disclosure — Phase 15.
12. **Email verification / password reset timing** — Phase 4 or later.
13. **Multi-tenancy** — post-MVP; schema is prepared via `owner_id` on chunks.
14. **Evaluation dataset size/annotation process** — Phase 14.

Full ADR context: [DECISIONS_AND_TRADEOFFS.md](DECISIONS_AND_TRADEOFFS.md).
