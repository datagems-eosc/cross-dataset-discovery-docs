# Status & Error Codes

The API uses standard HTTP status codes to indicate the success or failure of a request.

## Status Codes

| Status Code | Meaning | When it is returned |
| :--- | :--- | :--- |
| `200 OK` | Success | The request was successful. For `POST /ingest/`, the dataset is already indexed and was skipped (`replace=false`). |
| `202 Accepted` | Job scheduled | `POST /ingest/` scheduled an ingestion job. Poll it with `GET /ingest/{job_id}`. |
| `400 Bad Request` | Validation error | The request body or parameters are invalid: for example `k` is out of range, `dataset_ids` is an empty list, `search_mode` is unknown, or a required field is missing. For `POST /ingest/`, the profile could not be read or has no `@id`. |
| `401 Unauthorized` | Authentication error | The request has no token, or the token is expired, malformed or has an invalid signature. Also returned when the OIDC provider cannot be reached to validate the token. |
| `403 Forbidden` | Authorization error | The token lacks the required role (`user`, `dg_user` or a privileged role), or the caller lacks the dataset grant the endpoint needs (`dg_ds-edit` or `dg_ds-delete`). See [Security](./security.md). |
| `404 Not Found` | Not found | `GET /ingest/{job_id}`: no job with this id. `DELETE /dataset/{dataset_id}`: the dataset is not in the index. Also returned for unknown paths. |
| `405 Method Not Allowed` | Wrong method | The path exists, but not with this HTTP method. |
| `424 Failed Dependency` | Dependency error | A component the request depends on is unavailable. See [Failed Dependency Errors](#failed-dependency-errors). |
| `500 Internal Server Error` | Unexpected error | An unexpected error occurred while processing the request. This includes Qdrant or the embedding service (TEI) failing *during* a search, ingestion request or deletion, and a malformed analytical pattern sent to `POST /search-ap/`. |

Search requests never fail because of missing dataset permissions. Results the caller may not see are filtered out, so the response may simply contain fewer (or no) results.

## Error Response Bodies

### General Errors

`401`, `403`, `404`, `500`, and the `400` errors of `POST /ingest/` return:

```json
{
  "code": 403,
  "error": "User does not have permission to ingest this dataset."
}
```

* `code`: The HTTP status code.
* `error`: A human-readable description of the error.

### Validation Errors

`400` errors caused by an invalid request body or invalid parameters return:

```json
{
  "code": 102,
  "error": "Validation Error",
  "message": [
    {
      "Key": "body.k",
      "Value": ["Input should be greater than 0"]
    }
  ]
}
```

* `code`: Always `102`.
* `message`: One entry per invalid field. `Key` is the field's location (e.g. `body.dataset_ids`, `query.path_to_profile`) and `Value` lists the problems found.

### Failed Dependency Errors

`424` errors return:

```json
{
  "code": 104,
  "error": "error communicating with underpinning service",
  "message": {
    "statusCode": 503,
    "source": "Qdrant"
  }
}
```

* `code`: Always `104`.
* `message.source`: The failing component:
    * `SearchComponent`: the search component failed to initialize when the service started.
    * `Qdrant`: the vector database is unreachable (reported by `GET /health`).
    * `TEI`: the embedding service is unreachable (reported by `GET /health`).
    * `IngestionJobStore`: ingestion jobs could not be saved or read.
* `message.statusCode`: The status of the failing component (`503` when it is unavailable).
* `message.correlationId`, `message.payload`: Additional details, when available.

### Other Errors

* Unknown paths (`404`) and wrong methods (`405`) return FastAPI's default body, e.g. `{"detail": "Not Found"}`.
* A malformed analytical pattern sent to `POST /search-ap/`, or a Qdrant failure while `POST /ingest/` checks whether the dataset is already indexed, returns a plain-text `Internal Server Error`.
