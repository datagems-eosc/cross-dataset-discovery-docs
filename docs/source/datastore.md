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

Each point's payload holds:

| Field | Description |
| :--- | :--- |
| `content` | The indexed text, returned as `content` in search results. |
| `dataset_id` | The dataset the item belongs to. Used to filter searches and to find, count and delete a dataset's points. |
| `file_object_id` | The file the item comes from (absent for dataset-level items such as the description). Returned as `object_id` in search results. |
| `original_filename` | Path of the source file in the dataset storage. |
| *loader-specific fields* | For example `table_name` and `database` for SQL tables, or the record's other fields for JSON files. |

Only `content` is embedded; the other fields are metadata, returned by `POST /corpus-analysis-search/` as `metadata`.

During an ingestion, Qdrant's vector index building (HNSW) is paused for the collection, so the upload is fast. It is resumed when the upload finishes, and the quantization is then re-applied.

### Ingestion Jobs Collection

Each ingestion job is stored as one point with no vectors. Its payload is the job record returned by `GET /ingest/{job_id}`. The collection has payload indexes on `status`, `dataset_id` and `started_at`, used to filter and sort `GET /ingest/`.

## Other Data Sources

The service also reads, but never writes, these sources during ingestion:

-   **Dataset storage:** a volume mounted into the API pod that holds the dataset profiles and the files they reference.
-   **Dataset database servers:** PostgreSQL servers holding the `sql-db` datasets, configured with `SQL_DB_BASE_URLS`.
