
## API Overview

The Cross-Dataset Discovery service exposes a RESTful API for performing search operations.

### Base URL

**Base URL:** [https://datagems-dev.scayle.es/cross-dataset-discovery/](https://datagems-dev.scayle.es/cross-dataset-discovery/)

### Authentication

All endpoints are protected and require a valid JWT Bearer token.

---

## Endpoints

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

* `search_mode`: Optional. Can be `"sparse"` (default), `"dense"`, or `"hybrid"`.

* **Response Body (Success):**

```json
{
  "query_time": "retrieval time in milliseconds",
  "results": [
    {
      "content": "The main text content of the search result.",
      "dataset_id": "uuid-of-the-source-dataset",
      "object_id": "uuid-of-the-source-object",
      "similarity": "score (BM25 score, Cosine Similarity, or RRF score)"
    }
  ]
}
```

### Health Check

Verifies the operational status of the API and its dependencies.

* **Endpoint:** `GET /health`
* **Description:** Checks connectivity to the Database, Qdrant, and the TEI service.
* **Response Body (Success):**

```json
{
  "status": "ok",
  "message": "All dependencies are healthy."
}
```