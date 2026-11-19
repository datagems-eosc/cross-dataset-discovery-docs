# Evaluation

The service has built-in endpoints that measure how well it finds the right data. They index benchmark datasets into a separate Qdrant collection, run each benchmark's queries through the normal search code, and score the results against the benchmarks' correct answers. This setup allows developers to test the service in the exact way it operates on all datasets, including the pre-processing and ingestion pipeline.

## Where It Runs

Only on **dev**. The endpoints exist only when `BENCHMARK_QDRANT_COLLECTION` and `BENCHMARK_ROOT` are both set, and the benchmark collection is separate from the main collection used for searching (see [Configuration](./configuration.md#benchmarks)). Elsewhere, `/benchmarks/...` returns `404`.

| Setting | Dev value | Meaning |
| :--- | :--- | :--- |
| `BENCHMARK_QDRANT_COLLECTION` | `cdd_benchmarks` | Qdrant collection for the benchmark data. Never the main one. |
| `BENCHMARK_ROOT` | `/s3/cdd_benchmarks` | Folder (on the dataset storage) with the datasets, profiles, queries and results. |

The main collection is never touched. Benchmark job records are kept in `cdd_benchmarks_jobs`.

## The Benchmarks

| Query set | Data | Judged per | Queries | Main metric |
| :--- | :--- | :--- | ---: | :--- |
| `vidore_*` (8 corpora) | ViDoRe v3: reports, manuals, slides (PDF) | page | 14,334 | nDCG@10 |
| `irpapers` | IRPAPERS: 166 scientific papers (PDF) | page | 180 | Recall@5 |
| `t2_finqa`, `t2_convfinqa`, `t2_tatdqa` | T²-RAGBench: financial report pages (JSON, text + tables) | file | 23,088 | MRR@3 |
| `ottqa` | OTT-QA: 8,891 Wikipedia tables (JSON) | file | 2,214 | Recall@10 |

12 datasets, 13 query sets (ConvFinQA's answers are in the FinQA dataset).

## How It Works

1. **Ingest.** Each dataset is indexed from its profile with the same code as `POST /ingest/`. For PDFs, the page of every chunk is also stored, because the benchmarks say which *page* answers a query.
2. **Evaluate.** Every query goes through the same search code as `POST /search/` (top 100 hits). Each hit is mapped to the page or file it comes from; several chunks of the same page count once. The ranking is compared with the correct pages or files.
3. **Results.** The scores are averaged per query set, search mode and scope, and kept in the job record.

Ingestion and evaluation run in the background: the call returns a `job_id` at once, and the job is followed with `GET /benchmarks/jobs/{job_id}`. Only one benchmark job runs at a time; starting another returns `409`. An evaluation fails if any dataset it needs has not been ingested.

As with `POST /ingest/`, each ingested dataset records `items_submitted`, `items_indexed` and `points_in_qdrant`; if they differ, the dataset is marked `failed`.

### Job Statuses

A benchmark job is `running`, `completed`, `failed` (see `error`), or `interrupted`. A job that was still running when the service restarted is marked `interrupted` at startup. Re-run an interrupted ingestion with `"replace": true`, since datasets may be partially indexed.

The job record (`GET /benchmarks/jobs/{job_id}`) contains `job_id`, `kind` (`ingest` or `evaluate`), `status`, `started_at`, `finished_at`, the original `request`, `progress`, the outcome per dataset (`datasets`, for ingestion), the scores (`summary`, for evaluation) and `error`.

## Endpoints

All require a privileged role (`dg_admin`, `dg_system` or `dg_dataset-curator`). Their request and response schemas are on the [OpenAPI Specification](./openapi.md) page, under the `Benchmarks` tag.

| Endpoint | What it does |
| :--- | :--- |
| `POST /benchmarks/ingest/` | Index datasets. Body: `{"datasets": [...], "replace": false}`. Without `datasets`, all of them. Already indexed datasets are skipped unless `replace` is true. Returns `202` with the `job_id`. |
| `POST /benchmarks/evaluate/` | Run and score queries. Body: `{"name", "query_sets", "search_modes", "scopes"}`, all optional (default: every query set, all 3 modes, `pooled`). Returns `202` with the `job_id`. |
| `GET /benchmarks/jobs/{job_id}` | Status, progress and outcome of a job. |
| `GET /benchmarks/results/` | The latest evaluations side by side. Options: `last` (default `5`, maximum `50`), `metric` (e.g. `recall@10`; default: each query set's main metric). |
| `DELETE /benchmarks/data/` | Drop the benchmark collection. Job records and results are kept. Refused with `409` while a job is running. |

Unknown dataset or query set keys are rejected with `400`, and an unreadable benchmark manifest with `503`.

## Running an Evaluation

The examples use the API's base URL in `$CDD` and an access token of a user with a privileged role in `$TOKEN` (see [Maintenance](./maintenance.md)).

```bash
H="Authorization: Bearer $TOKEN"

# 1. Index everything (about 25 minutes; datasets already indexed are skipped).
curl -s -X POST "$CDD/benchmarks/ingest/" -H "$H" -H 'Content-Type: application/json' -d '{}'

# 2. Evaluate. Start small: IRPAPERS takes about 10 seconds.
curl -s -X POST "$CDD/benchmarks/evaluate/" -H "$H" -H 'Content-Type: application/json' \
  -d '{"name": "my-test", "query_sets": ["irpapers"]}'
#    The full run (all sets, 3 modes, pooled) takes about 1 hour:
curl -s -X POST "$CDD/benchmarks/evaluate/" -H "$H" -H 'Content-Type: application/json' \
  -d '{"name": "my-run"}'

# 3. Follow the job (use the job_id returned above).
curl -s "$CDD/benchmarks/jobs/<job_id>" -H "$H" | jq '.status, .progress'

# 4. See the results next to earlier runs.
curl -s "$CDD/benchmarks/results/" -H "$H" | jq '.columns, .table'
```

Access tokens expire quickly (5 minutes on dev), so get a fresh one before each step of a long session.

## Reading the Results

**Search modes:** `sparse` (BM25 keywords), `dense` (BGE-M3 embeddings), `hybrid` (both, fused with RRF).

**Scopes:** `pooled` searches all 12 benchmark datasets at once (the realistic case). `restricted` searches only the dataset holding the answers (like passing `dataset_ids` to `POST /search/`). The difference shows how much the other datasets get in the way.

**Metrics:**

| Metric | Meaning |
| :--- | :--- |
| `recall@k` | Share of the correct pages/files found in the top k. |
| `mrr@k` | 1 / rank of the first correct page/file (0 if not in the top k). |
| `ndcg@10` | Quality of the top 10; highly relevant pages count more (ViDoRe grades its pages). |
| `dataset_hit@k` | Whether any of the top k hits comes from the right dataset: the core question for cross-dataset discovery. |

Each row also has `num_queries`, `failed_queries` (searches that errored, left out of the averages) and `unmapped_hit_rate` (hits whose page was unknown; should be about 0). Every query's details are in `eval/runs/<job_id>.jsonl` under `BENCHMARK_ROOT`.

`GET /benchmarks/results/` returns one row per query set, plus a `(mean)` row for benchmarks with several query sets (as ViDoRe reports its corpora), and one column per run, search mode and scope.

## Files under `BENCHMARK_ROOT`

| Path | Content |
| :--- | :--- |
| `dataset/<uuid>/` | The benchmark files (PDFs, JSON pages, JSON tables). |
| `cdd_profile/<uuid>.json` | Each dataset's profile, made with the profiler. |
| `eval/manifest.json` | List of datasets and query sets: profile, judging unit, answer dataset, main metric. |
| `eval/queries/<set>.jsonl` | One query per line, with its correct pages/files and their grade. |
| `eval/pages/<uuid>.json` | Text of every PDF page, used at indexing to find each chunk's page. |
| `eval/runs/<job_id>.jsonl` | Written by each evaluation: every query's scores. |

These files were prepared once, outside the service. To add a benchmark, add its files, its profile, its queries and its manifest entries, then ingest it.
