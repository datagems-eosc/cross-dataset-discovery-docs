# API Overview

The Cross-Dataset Discovery service exposes a RESTful API for searching the indexed datasets and for managing which datasets are indexed.

### Base URL

**Base URL:** [https://datagems-dev.scayle.es/cross-dataset-discovery/](https://datagems-dev.scayle.es/cross-dataset-discovery/)

The running service also serves its own interactive documentation at `/swagger` and `/redoc`, and the raw specification at `/openapi.json`. The full specification is also available on the [OpenAPI Specification](./openapi.md) page.

### Authentication

All endpoints except `GET /` and `GET /health` require a valid JWT Bearer token in the `Authorization` header. The token must carry the `user` or `dg_user` role, or a privileged role (`dg_admin`, `dg_system`, `dg_dataset-curator`). Each endpoint additionally checks dataset-level grants, as listed below. See [Security](./security.md) for details.

### Endpoints at a Glance

| Method | Path | Purpose | Dataset grant required |
| :--- | :--- | :--- | :--- |
| `POST` | `/search/` | Search the index | `dg_ds-browse` (results are filtered) |
| `POST` | `/search-ap/` | Search, using an analytical pattern as request and response | `dg_ds-browse` (results are filtered) |
| `POST` | `/corpus-analysis-search/` | LLM-assisted search for text passages | `dg_ds-browse` (results are filtered) |
| `POST` | `/ingest/` | Index a dataset from its profile (background job) | `dg_ds-edit` |
| `GET` | `/ingest/` | List ingestion jobs | `dg_ds-edit` on the `dataset_id` filter (privileged roles can list all) |
| `GET` | `/ingest/{job_id}` | Get the status of an ingestion job | `dg_ds-edit` |
| `DELETE` | `/dataset/{dataset_id}` | Remove a dataset from the index | `dg_ds-delete` |
| `GET` | `/` | Liveness check | none (no authentication) |
| `GET` | `/health` | Readiness check of the service and its dependencies | none (no authentication) |

Callers with a privileged role bypass all dataset grant checks.

---

## Search

### Search

Performs a search query against the indexed datasets.

* **Endpoint:** `POST /search/`
* **Description:** Submits a natural language query and returns a ranked list of relevant results. Supports Sparse (BM25), Dense (Semantic), and Hybrid search modes.
* **Request Body:**

```json
{
  "query": "string",
  "k": 5,
  "dataset_ids": [
    "string"
  ],
  "search_mode": "sparse"
}
```

* `query`: Required. The natural language query.
* `k`: Optional. Number of results to return, between 1 and 100 (default `5`).
* `dataset_ids`: Optional. Restricts the search to these datasets. The list is intersected with the datasets the user may browse; if nothing remains, an empty result list is returned. Must not be an empty list.
* `search_mode`: Optional. Can be `"sparse"` (default), `"dense"`, or `"hybrid"`.

* **Response Body (Success):**

```json
{
  "query_time": 42.17,
  "results": [
    {
      "content": "The main text content of the search result.",
      "dataset_id": "uuid-of-the-source-dataset",
      "object_id": "uuid-of-the-source-object",
      "similarity": 12.3
    }
  ]
}
```

* `query_time`: Processing time in milliseconds.
* `object_id`: The file object the result comes from. For dataset-level results (such as the dataset description), this is the same as `dataset_id`.
* `similarity`: BM25 score, cosine similarity, or RRF score, depending on `search_mode`.

Results from datasets the user is not authorized to browse are always removed, whether or not `dataset_ids` was given.

### Search via Analytical Pattern

* **Endpoint:** `POST /search-ap/`
* **Description:** The same search as `POST /search/`, but the request and response are DataGEMS analytical patterns (APs).
* **Request Body:** An analytical pattern that contains a node labelled `CDD_Operator`. The search arguments are read from that node's `properties`. `query` is required; `k`, `dataset_ids`, and `search_mode` are optional and mean the same as in `POST /search/`.

```json
{
  "ap": {
    "nodes": [
      {
        "id": "op-1",
        "labels": ["CDD_Operator"],
        "properties": {
          "query": "string",
          "k": 5,
          "search_mode": "hybrid"
        }
      }
    ],
    "edges": []
  }
}
```

* **Response Body (Success):** The same analytical pattern, updated with the results:
    * Any nodes previously connected as outputs of the `CDD_Operator` node are removed.
    * One `sc:Dataset` node is added per result dataset, and one `cr:FileObject` node per result object.
    * Each `cr:FileObject` node gets an `output` edge from the `CDD_Operator` node and a `distribution` edge from its dataset.
    * The full search response (`query_time`, `results`) is merged into the pattern's top-level `metadata`.

```json
{
  "ap": {
    "nodes": [
      { "id": "op-1", "labels": ["CDD_Operator"], "properties": { "query": "string" } },
      { "id": "dataset-uuid", "labels": ["sc:Dataset"] },
      { "id": "object-uuid", "labels": ["cr:FileObject"] }
    ],
    "edges": [
      { "from": "op-1", "labels": ["output"], "to": "object-uuid" },
      { "from": "dataset-uuid", "labels": ["distribution"], "to": "object-uuid" }
    ]
  },
  "metadata": {
    "query_time": 42.17,
    "results": [ ... ]
  }
}
```

A pattern without a `CDD_Operator` node, without a `query`, or with invalid optional arguments results in a `500 Internal Server Error`.

### Corpus Analysis Search

* **Endpoint:** `POST /corpus-analysis-search/`
* **Description:** An LLM-assisted search for text passages that answer a natural language question. The steps are:
    1. An LLM extracts the text request from the question. It also extracts time-period, location, and language constraints, but these are not applied yet.
    2. The index is searched with the extracted request.
    3. The LLM scores each result for relevance and drops results that score below 3. Fewer than `k` results may therefore be returned.

    If either LLM step fails, the service uses the original question and the unscored results instead of failing the request. Dataset access filtering works the same as in `POST /search/`.
* **Request Body:**

```json
{
  "query": "string",
  "dataset_ids": [
    "string"
  ]
}
```

* `query`: Required. The natural language question.
* `dataset_ids`: Optional. Same meaning as in `POST /search/`.
* `k` (deprecated): Number of results to retrieve before LLM scoring (default `50`, maximum `100`).
* `search_mode` (deprecated): Retrieval strategy (default `"dense"`).

`k` and `search_mode` are still accepted, but will be removed once the API chooses them automatically.

* **Response Body (Success):** The same shape as `POST /search/`, plus each result's indexed `metadata`:

```json
{
  "query_time": 1830.5,
  "results": [
    {
      "content": "The main text content of the search result.",
      "dataset_id": "uuid-of-the-source-dataset",
      "object_id": "uuid-of-the-source-object",
      "similarity": 0.71,
      "metadata": {
        "dataset_id": "uuid-of-the-source-dataset",
        "file_object_id": "uuid-of-the-source-object",
        "original_filename": "/s3/path/to/file.json"
      }
    }
  ]
}
```

---

## Ingestion

### Ingest a Dataset

* **Endpoint:** `POST /ingest/?path_to_profile={path}&replace={true|false}`
* **Description:** Indexes a dataset described by a dataset profile (JSON). The service reads the profile from `path_to_profile`, a path on the service's mounted storage (for example `/s3/...`). The caller needs a `dg_ds-edit` grant on the profile's `@id`, or a privileged role.

    Ingestion can take a long time, so it runs as a background job. The endpoint returns immediately with a `job_id` to poll with `GET /ingest/{job_id}`.
* **Query Parameters:**
    * `path_to_profile`: Required. Path to the dataset profile JSON, as seen by the service.
    * `replace`: Optional (default `false`). If the dataset is already indexed:
        * `false`: nothing is done and the request returns `200` with status `skipped`.
        * `true`: the dataset's existing indexed content is deleted and rebuilt from the profile.
* **Response Body (`202 Accepted`, job scheduled):**

```json
{
  "job_id": "uuid-of-the-job",
  "status": "pending",
  "dataset_id": "uuid-of-the-dataset"
}
```

* **Response Body (`200 OK`, already indexed and `replace=false`):**

```json
{
  "job_id": null,
  "status": "skipped",
  "dataset_id": "uuid-of-the-dataset"
}
```

#### Dataset Profile

The profile must contain the dataset `@id` and `description`, plus the list of `files` to index:

```json
{
  "@id": "uuid-of-the-dataset",
  "name": "Dataset name",
  "description": "Dataset description (indexed as its own search item).",
  "files": [
    {
      "file_object_id": "uuid-of-the-file",
      "source_file": "/s3/path/to/file.csv",
      "original_format": "csv"
    }
  ]
}
```

If the profile cannot be read or has no `@id`, the request fails immediately with `400`. If `description` is missing, the job fails.

Each file is handled according to its `original_format`:

| `original_format` | How it is indexed | Extra file fields used |
| :--- | :--- | :--- |
| `csv`, `xlsx`, `xls` | One item per row, serialized as `column: value` text (numeric values are left out) | – |
| `sql-db` | One item per table row (serialized as above), read from the database named `name` on one of the configured SQL servers | `name` (database), `tables[].name` (optional; defaults to all tables) |
| `text`, `application/pdf` | One item per pre-computed chunk | `chunked_content` (a list of sections, each with a `chunks` list) |
| `json`, `application/json` | Records split into token-based chunks; the title is prepended to each chunk | `content_fields` (default `["text"]`), `metadata_fields` (default: all other fields), `title_field` (default `"title"`) |
| `docx`, `pptx`, `ipynb` | Text split into token-based chunks | – |

Files with any other format are skipped (logged as a warning).

### Get Ingestion Job Status

* **Endpoint:** `GET /ingest/{job_id}`
* **Description:** Returns the state of an ingestion job. Requires the same `dg_ds-edit` grant (or privileged role) as starting the job.
* **Response Body (Success):**

```json
{
  "job_id": "uuid-of-the-job",
  "status": "completed",
  "dataset_id": "uuid-of-the-dataset",
  "profile_path": "/s3/path/to/profile.json",
  "replace": false,
  "requested_by": "sub-claim-of-the-caller",
  "items_submitted": 12345,
  "items_indexed": 12345,
  "points_in_qdrant": 12345,
  "deleted_count": null,
  "error": null,
  "started_at": "2026-10-14T09:00:00.000000Z",
  "finished_at": "2026-10-14T09:12:31.000000Z"
}
```

* `status`: one of:
    * `pending`: scheduled, not started yet.
    * `running`: in progress.
    * `completed`: finished successfully.
    * `failed`: stopped with an error (see `error`).
    * `interrupted`: the service restarted before the job finished. The dataset may be partially indexed; run the ingestion again with `replace=true`.
* `requested_by`: The `sub` claim of the caller that started the job.
* `items_submitted`: Number of items sent for embedding. Set when the job completes.
* `items_indexed`: Number of points this job wrote to Qdrant. Set when the job completes; normally equal to `items_submitted`.
* `points_in_qdrant`: Total number of the dataset's points in Qdrant after the job. After a fresh or `replace=true` ingestion it should equal `items_indexed`. A mismatch means points were lost, or another job wrote to the same dataset at the same time.
* `deleted_count`: Number of previously indexed points removed (set only when `replace=true`).
* `error`: The failure reason (set only when the job fails or is interrupted).

Job records are stored in Qdrant (in their own collection), so they survive service restarts.

### List Ingestion Jobs

* **Endpoint:** `GET /ingest/?status={status}&dataset_id={dataset_id}&limit={n}`
* **Description:** Lists ingestion jobs, newest first. Callers with a privileged role can list all jobs. Other callers must pass `dataset_id` and hold a `dg_ds-edit` grant on that dataset; otherwise the request is rejected with `403`.
* **Query Parameters:**
    * `status`: Optional. Only return jobs with this status (`pending`, `running`, `completed`, `failed`, or `interrupted`).
    * `dataset_id`: Optional for privileged callers, required for everyone else. Only return jobs for this dataset.
    * `limit`: Optional. Maximum number of jobs to return (default `10`).
* **Response Body (Success):** The job records, in the same format as `GET /ingest/{job_id}`:

```json
{
  "jobs": [
    {
      "job_id": "uuid-of-the-job",
      "status": "interrupted",
      "dataset_id": "uuid-of-the-dataset",
      ...
    }
  ]
}
```

### Delete a Dataset

* **Endpoint:** `DELETE /dataset/{dataset_id}`
* **Description:** Removes all of a dataset's indexed content. The source data is not touched. The caller needs a `dg_ds-delete` grant on the dataset, or a privileged role.
* **Response Body (Success):**

```json
{
  "dataset_id": "uuid-of-the-dataset",
  "deleted_count": 12345
}
```

If the dataset is not in the index, the endpoint returns `404`.

---

## Monitoring

### Liveness

* **Endpoint:** `GET /`
* **Description:** Returns a static response without checking dependencies. Used as the Kubernetes liveness probe. No authentication required.
* **Response Body (Success):**

```json
{
  "status": "ok",
  "message": "Cross Dataset Discovery API is running."
}
```

### Health Check

Verifies the operational status of the API and its dependencies.

* **Endpoint:** `GET /health`
* **Description:** Checks that the search component is initialized and that Qdrant and the TEI embedding service are reachable. Used as the Kubernetes readiness probe. No authentication required. If a check fails, it returns `424 Failed Dependency` naming the failing component.
* **Response Body (Success):**

```json
{
  "status": "ok",
  "message": "All dependencies are healthy."
}
```

See [Status & Error Codes](./error-codes.md) for the error responses of all endpoints.
