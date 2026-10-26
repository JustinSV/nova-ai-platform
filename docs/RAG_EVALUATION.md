# NOVA AI — RAG Evaluation with Ragas

Status: **design.** No datasets, results, or baselines exist. Metric names and Ragas APIs change across releases — **[Assumption: verify metric class names, required dataset columns, and judge-model configuration against the pinned Ragas version]**. Related: [RAG_PIPELINE](RAG_PIPELINE.md) §20, [TESTING_STRATEGY](TESTING_STRATEGY.md) §11, [DATA_FLOW](DATA_FLOW.md) Flow 5.

## 1. Objectives
1. Decide, with evidence, which retrieval configuration (C1–C4) NOVA AI ships.
2. Detect quality regressions from code, prompt, model, or chunking changes.
3. Quantify trade-offs: quality vs latency vs token cost.
4. Supply a feedback loop from production reviews into the dataset.

## 2. Evaluation corpus and dataset

**Corpus:** a frozen, versioned set of documents (`evaluation/corpora/<version>/`) with clear licensing (own/public-domain/open-license documents; **no real user data**). Mix of lengths, formats, tables, and overlapping topics. Record a content hash manifest.

**Dataset file:** `evaluation/datasets/<name>/<version>.jsonl`, one object per question:

```json
{
  "id": "q-0042",
  "question": "What notice period applies if the supplier wants to terminate?",
  "category": "factual_lookup",
  "difficulty": "easy",
  "reference_answer": "Sixty (60) days written notice.",
  "reference_contexts": [
    { "document": "supplier-contract.pdf", "page": 7, "quote": "either party may terminate ... sixty (60) days written notice" }
  ],
  "answerable": true,
  "history": [],
  "tags": ["numeric", "legal"],
  "corpus_version": "corpus-v1",
  "annotator": "human-1",
  "reviewed": true
}
```

- **Categories:** factual lookup, keyword/identifier-heavy, paraphrase/semantic, numeric/table, multi-chunk/multi-hop, follow-up (with `history`), **unanswerable** (`answerable=false`, expected abstention), adversarial (prompt-injection documents), long-tail vocabulary.
- **Gold labels by quote/span, not by chunk id.** Chunk ids change when chunking changes; a reference context counts as retrieved if a retrieved chunk **overlaps the quoted span** (≥ threshold, e.g. 50 % of the quote). This keeps benchmarks valid when chunk size is an experiment variable.
- Size: start small but meaningful (e.g. 100–200 questions across categories; **statistical power must be assessed**, small datasets give wide confidence intervals); grow over time. Hold out a **test split** that is not used for tuning thresholds.
- Synthetic question generation (LLM-generated) may bootstrap the dataset, but every item must be **human-reviewed**; mark `generated=true`.

## 3. Run record schema (per question, per configuration)

```json
{
  "run_id": "…", "question_id": "q-0042", "config_id": "C3",
  "retrieved": [{"chunk_id":"…","document":"…","page":7,"rank":1,"lexical_rank":2,"dense_rank":1,"fused_score":0.0323,"rerank_score":null}],
  "contexts_sent": ["…text…"], "answer": "…", "citations": [1],
  "abstained": false, "degraded": false,
  "timings_ms": {"lexical":12,"embed":85,"dense":20,"fuse":1,"rerank":0,"first_token":640,"total":2100},
  "usage": {"prompt_tokens":2100,"completion_tokens":96}, "errors": []
}
```
Stored as JSONL/Parquet artifacts per run (S3 or CI artifacts); aggregates in `EvaluationRun.metrics`.

## 4. Metrics

### 4.1 Retrieval metrics (deterministic, no LLM judge — **primary gate**)
- **Recall@k** (k = 5, 10, 20): fraction of questions whose reference contexts are (all / any — report both) in the top k.
- **MRR:** mean reciprocal rank of the first relevant chunk.
- **nDCG@k** where graded relevance exists.
- Evaluated on retrieval output *before* generation → cheap, repeatable, comparable across C1–C4.

### 4.2 Ragas metrics (LLM-assisted)
| Metric | Question answered | Needs |
|---|---|---|
| **Faithfulness** | Are answer claims supported by the retrieved contexts? | question, answer, contexts |
| **Answer relevancy** | Does the answer address the question? | question, answer (uses embeddings/LLM) |
| **Context precision** | Are relevant contexts ranked high among those retrieved? | question, contexts, reference (answer or contexts) |
| **Context recall** | Do retrieved contexts cover the reference answer's information? | reference answer, contexts |

### 4.3 NOVA-specific deterministic metrics
- **Citation accuracy:** (a) *validity* — every `[n]` maps to supplied evidence; (b) *precision* — fraction of cited chunks that overlap a reference context; (c) *recall* — fraction of reference contexts that are cited; (d) optional *support* check (human or judge model) that the cited passage supports the sentence.
- **Abstention accuracy:** on `answerable=false` questions → abstained; on answerable → not abstained; report false-abstention and false-answer rates separately.
- **Degraded-rate:** share of turns with fallbacks.

### 4.4 Efficiency
Latency per stage (p50/p95), time to first token, prompt/completion tokens, estimated cost (from provider price table kept in config — **never hard-coded constants presented as facts**), rerank candidate count.

### 4.5 Human review
Sample (stratified by category and by low/high metric disagreement) reviewed against a rubric: correctness, groundedness, completeness, citation support, tone. Two reviewers on a subset to estimate agreement. Negative production feedback ([DATA_FLOW](DATA_FLOW.md) Flow 5) feeds candidate dataset items after de-identification.

## 5. Limitations of automated evaluation
- **Judge-model bias and variance:** LLM judges can prefer verbose/own-family outputs, be inconsistent between runs, and be fooled by fluent errors; scores depend on the judge model/version and prompts.
- Faithfulness checks *consistency with retrieved context*, not truth; a wrong context yields a "faithful" wrong answer.
- Reference-based metrics inherit annotation errors and single-reference bias (valid alternative answers penalized).
- Small datasets → noisy comparisons; many metrics → multiple-comparison risk.
- Metrics may not reflect user value (tone, trust, UX).
Mitigations: deterministic retrieval metrics as the primary gate; **judge-model pinned and different from the generation model where practical**; repeat runs; report confidence intervals; human review; never ship on Ragas scores alone.

## 6. Experiment versioning
Every run records: git SHA, dataset version, corpus version, config JSON (chunking, embedding model, retrieval params, reranker model + revision, prompt version, LLM model + params), judge model + version, Ragas version, random seeds, timestamp. Configs live in `evaluation/benchmarks/configs/*.yaml`; results in `evaluation/benchmarks/results/<run_id>/` (large artifacts in object storage). A run without a complete record is not comparable and is rejected by the runner.

## 7. Repeatable benchmark procedure (C1–C4)

1. **Freeze inputs:** corpus version, dataset version (dev vs test split), chunking config, embedding model, LLM model/params, prompt version, reranker model.
2. **Create an isolated environment:** fresh database (or dedicated schema/user), ingest the corpus through the **production ingestion code**, record chunk counts and index build times. Verify index health (chunk counts, no failures).
3. **Define the four configs** (only retrieval differs):
   - C1 lexical only · C2 dense only · C3 hybrid + RRF · C4 hybrid + RRF + rerank.
4. **Phase A — retrieval-only run** (cheap): for each config and question, store ranked chunks (top 20+) with stage scores; compute Recall@k, MRR, nDCG, context-level metrics. Include latency from warmed caches *and* cold start, noting hardware.
5. **Phase B — end-to-end generation:** for each config, generate answers with temperature 0; repeat `n ≥ 3` times per question to measure variance; store answers, citations, tokens.
6. **Compute metrics:** deterministic first; then Ragas (faithfulness, answer relevancy, context precision/recall) with the pinned judge; citation and abstention metrics.
7. **Statistics:** per-category and overall means; **paired bootstrap** confidence intervals (or paired tests) for C3−C1, C3−C2, C4−C3 differences; report effect sizes, not only p-values; note multiple comparisons.
8. **Cost/latency analysis:** p50/p95 stage latency and tokens per question; compute "quality gain per added latency/cost" for C4 vs C3.
9. **Human review** of a stratified sample, especially cases where configs disagree.
10. **Decision record:** write the outcome into an ADR (adopt/reject hybrid, reranker, parameters). Possible, valid outcomes include "C2 is sufficient" or "reranking is not worth it for this corpus." **No outcome is presupposed.**
11. **Sensitivity sweeps** (after choosing a config): chunk size/overlap, `K_*`, RRF `k`, `N_RERANK`, `N_EVIDENCE`, abstention threshold — one factor at a time, evaluated on dev, confirmed once on the held-out test split.

Re-run when: embedding model, chunking, reranker, prompt, LLM, or corpus changes.

## 8. Baselines and regression thresholds
- **Baseline** = the last accepted run for the current default config on the test split, stored with its full record.
- **Regression gates (proposed structure; numeric values set after measuring run-to-run variance):**
  - Retrieval Recall@k and MRR must not drop by more than **δ_ret** absolute versus baseline (hard fail).
  - Citation validity must stay at **100 %** (deterministic invariant).
  - Abstention false-answer rate must not exceed **δ_abs**.
  - Ragas metrics: warn (not fail) on drops beyond **δ_ragas**, because of judge noise; fail only on large, repeated drops.
  - Latency p95 regression beyond **δ_lat** → warn/fail depending on stage.
- Thresholds are calibrated from repeated baseline runs (e.g. a multiple of the observed standard deviation) and revisited periodically; they are **not** specified here to avoid inventing numbers.

## 9. CI integration
- **On every PR (fast, no external LLM):** unit tests for metrics code; retrieval-only evaluation on a **small fixed mini-corpus** with fake/pinned local embeddings, deterministic; assert Recall@k/MRR vs stored baseline.
- **Scheduled (nightly/weekly) and on-demand (`workflow_dispatch`, release candidates):** full benchmark (Phases A+B + Ragas) using provider credentials from a protected GitHub environment with spend caps; publishes a Markdown/JSON report and compares with baseline; failing gates block release tagging.
- Artifacts uploaded; summary posted to the PR/issue; results recorded as `EvaluationRun`.
- Cost control: sampled subsets on PRs, caching of embeddings for the frozen corpus, bounded concurrency.

## 10. Directory layout (proposed)

```
evaluation/
├── corpora/<version>/            # frozen documents + manifest.json
├── datasets/<name>/<version>.jsonl
├── benchmarks/
│   ├── configs/                  # C1..C4 and sweep configs (YAML)
│   ├── baselines/                # accepted baseline metrics (JSON)
│   └── results/                  # run outputs (gitignored; artifacts)
└── ragas/                        # judge config, metric adapters
backend/ai/evaluation/            # runner, metrics, span-overlap labeling
```
