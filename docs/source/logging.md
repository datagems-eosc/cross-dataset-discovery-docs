# Logging & Accounting

The service uses `structlog` to write structured, JSON-formatted logs to standard output, one JSON object per line, following the DataGEMS `json-cf-2` log format.

## Log Structure

Every log entry written by the service contains:

- `Timestamp`: ISO 8601 timestamp (UTC).
- `Message`: The log message (event), e.g. `http_request_finished`.
- `LogLevel`: The log level.
- `DGCorrelationId`: The correlation ID of the HTTP request being processed (see below).
- `SourceContext`: The name of the logger that produced the entry, e.g. `darelabdb.api_datagems_cross_dataset_discovery.app.main`, or `accounting` for accounting entries.
- `Application`: Always `cross-dataset-discovery-api`.

Additional key-value pairs give context for specific events, for example `UserId`, `dataset_id`, `job_id`, or `error`.

Example:

```json
{"query": "renewable energy", "k": 5, "mode": "hybrid", "UserId": "3f2c…", "Application": "cross-dataset-discovery-api", "SourceContext": "darelabdb.api_datagems_cross_dataset_discovery.app.main", "Timestamp": "2026-10-29T10:15:02.123456Z", "Message": "Search completed and filtered.", "LogLevel": "Info", "DGCorrelationId": "b060c227-7002-4be8-8b3c-f7f5ecae85a0", "authorized_results_count": 5}
```

## Correlation ID

Every HTTP request is assigned a correlation ID:

- If the incoming request includes an `x-tracking-correlation` header, its value is used.
- Otherwise, a new UUID is generated.

The ID is added as `DGCorrelationId` to the log entries written while the request is processed. It is also returned to the caller in the `x-tracking-correlation` response header. This makes it possible to trace a request through the service and across DataGEMS services.

## Request Logging

A middleware logs the end of every HTTP request, except the health checks (`GET /` and `GET /health`). The entry has the message `http_request_finished` and contains:

- `RequestMethod`: e.g. `GET`, `POST`.
- `RequestPath`: The full request URL, including the query string.
- `StatusCode`: The response status code.
- `ProcessTimeMS`: Processing time in milliseconds.
- `ResponseSize`: The response size in bytes.

The level is `error` for `5xx` responses, `warning` for `4xx`, and `info` otherwise. If the request fails with an unhandled exception, an `http_request_unhandled_exception` entry with the exception is logged instead.

## Accounting

Billable user actions are logged as accounting entries, with `SourceContext` set to `accounting`. Every accounting entry includes `UserId` (the token's `sub` claim), `Action`, `Resource` (always `CrossDatasetDiscovery`) and `Timestamp`.

| `Action` | Logged when | Additional fields |
| :--- | :--- | :--- |
| `Search` | A `POST /search/` or `POST /search-ap/` request passes the role check. | `Type` (`+`), `Value` (`1`), `Measure` (`Unit`) |
| `Corpus Search` | A `POST /corpus-analysis-search/` request passes the role check. | `Type` (`+`), `Value` (`1`), `Measure` (`Unit`) |
| `IngestDataset` | `POST /ingest/` schedules an ingestion job (not when it is skipped). | `DatasetId`, `Replace`, and `ClientId` when the token has a `clientid` claim |
| `DeleteDataset` | `DELETE /dataset/{dataset_id}` deletes a dataset. | `DatasetId`, `DeletedCount`, and `ClientId` when the token has a `clientid` claim |

Example:

```json
{"UserId": "3f2c…", "Action": "Search", "Resource": "CrossDatasetDiscovery", "Type": "+", "Value": 1, "Measure": "Unit", "Timestamp": "2026-10-29T10:15:02.120001Z", "Application": "cross-dataset-discovery-api", "SourceContext": "accounting", "Message": "Search action performed", "LogLevel": "Info", "DGCorrelationId": "b060c227-7002-4be8-8b3c-f7f5ecae85a0"}
```
