# Datastores

The Cross-Dataset Discovery service stores all of its data in a **Qdrant** vector database.

## Vector Database (Qdrant)

-   **Technology:** Qdrant.
-   **Location:** A shared Qdrant instance in the Kubernetes cluster, configured with `QDRANT_URL`. Data is persisted by Qdrant on its own volume.
-   **Collections:** The service uses two collections, both created automatically:
    -   the **index collection** (`QDRANT_COLLECTION`), holding the searchable content;
    -   the **ingestion jobs collection** (`INGESTION_JOBS_COLLECTION`, by default `<QDRANT_COLLECTION>_ingestion_jobs`), holding the job records.

### Index Collection

Each indexed item (a table row, a document chunk, a dataset description, …) is stored as one point with two named vectors:

| Vector | Type | Details |
| :--- | :--- | :--- |
| `dense` | Dense, 1024 dimensions | Produced by `BAAI/bge-m3` via TEI. Cosine distance. Stored on disk, with a **binary-quantized** copy kept in RAM for fast search. Results are rescored with the full vectors. |
| `sparse` | Sparse (BM25) | Term frequencies computed by FastEmbed (`Qdrant/bm25`). Qdrant applies **IDF** weighting at query time. The inverted index is stored on disk. |

**Hybrid search** runs a dense and a sparse search and combines the two rankings with **Reciprocal Rank Fusion (RRF)**.

#### Point Payload

Every point's payload contains these fields, set by the service:

| Field | Description |
| :--- | :--- |
| `content` | The indexed text, returned as `content` in search results. |
| `dataset_id` | The dataset the item belongs to. Used to filter searches and to find, count and delete a dataset's points. |
| `file_object_id` | The file the item comes from (absent for dataset-level items such as the description). Returned as `object_id` in search results. |
| `original_filename` | Path of the source file in the dataset storage. |

Depending on the file format, the payload also contains:

| File format | Additional fields |
| :--- | :--- |
| `csv`, `xlsx`, `xls` | `table_id`; for Excel files also `sheet_name`. |
| `sql-db` | `database`, `table_name`, `table_id`; for tables indexed from a sample, also `sampled_rows`. |
| `json` | All fields of the source record except the indexed text field(s), plus `content_field` (the name of the field the text came from). See below. |
| `docx` | `table_index` for items that come from a table in the document. |
| `pptx` | `slide_index`. |
| `ipynb` | `cell_type`. |

**JSON records keep their own fields.** For JSON files, every field of the record other than the indexed text is stored in the payload as it is, including nested objects and lists. This keeps any extra information the dataset provides (titles, dates, languages, keywords, …) available alongside the text. If a record field has the same name as one of the service's fields (`dataset_id`, `file_object_id`, `original_filename`), the service's value is kept.

For example, an entry from a JSON encyclopedia dataset, whose record has the fields `id`, `title`, `text`, `year`, `language`, `metadata`, and so on, is stored as (content shortened):

```json
{
  "content": "CURB , in the manege, a chain of iron made fast to the upper part of the branches of the bridle, …",
  "content_field": "text",
  "dataset_id": "<dataset-uuid>",
  "file_object_id": "<file-object-uuid>",
  "original_filename": "/s3/dataset/<dataset-uuid>/data.json",
  "id": "kp-eb0707-055005-4422-v2.xml",
  "dataset": "19th century knowledge project",
  "file_name": "kp-eb0707-055005-4422-v2.xml",
  "title": "CURB",
  "year": "1830-1842",
  "language": "en",
  "metadata": {
    "edition": "Britannica 7th Edition",
    "original_filename": "language/kp-editions/eb07/XML_v2/c7/kp-eb0707-055005-4422-v2.xml",
    "keywords": ["Curbs", "Chains", "Eye", "Rings"],
    "raw_title_string": "CURB"
  }
}
```

Here `content` comes from the record's `text` field, `content_field`, `dataset_id`, `file_object_id` and the top-level `original_filename` are added by the service, and all other fields are copied from the record. The record's own nested `metadata` object, including its own `original_filename`, is kept unchanged.

Only `content` is embedded; the other fields are metadata. `POST /corpus-analysis-search/` returns them as `metadata` in its results.

During an ingestion, Qdrant's vector index building (HNSW) is paused for the collection, so the upload is fast. It is resumed when the upload finishes, and the quantization is then re-applied.

### Ingestion Jobs Collection

Each ingestion job is stored as one point with no vectors. Its payload is the job record returned by `GET /ingest/{job_id}`. The collection has payload indexes on `status`, `dataset_id` and `started_at`, used to filter and sort `GET /ingest/`.

## Other Data Sources

The service also reads, but never writes, these sources during ingestion:

-   **Dataset storage:** a volume mounted into the API pod that holds the dataset profiles and the files they reference.
-   **Dataset database servers:** PostgreSQL servers holding the `sql-db` datasets, configured with `SQL_DB_BASE_URLS`.
