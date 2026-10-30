# Maintenance

This page describes routine maintenance tasks: keeping the search index in sync with the datasets, following ingestion jobs, and recovering from failures.

The examples use `curl`, with the API's base URL in `$CDD` and a valid access token in `$TOKEN`:

```bash
CDD="https://<host>/cross-dataset-discovery"
TOKEN="<access token>"
```

Ingesting and deleting a dataset require the corresponding grant on that dataset (`dg_ds-edit` or `dg_ds-delete`), or a privileged role (`dg_admin`, `dg_system`, `dg_dataset-curator`). Maintenance across many datasets is easiest with a privileged service account. See [Security](./security.md).

## Checking Service Health

```bash
curl "$CDD/health"
```

A `200` response means the service is up and can reach Qdrant and the embedding service (TEI). A `424` response names the failing component in `message.source`. See [Status & Error Codes](./error-codes.md#failed-dependency-errors).

## Adding a Dataset to the Index

Ingest the dataset from its profile:

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
  "$CDD/ingest/?path_to_profile=/s3/path/to/profile.json"
```

The response contains a `job_id`. Ingestion runs in the background and can take from seconds to tens of minutes, depending on the dataset size. See [Following Ingestion Jobs](#following-ingestion-jobs).

## Updating a Dataset

If a dataset is already indexed, `POST /ingest/` does nothing and returns `"status": "skipped"`. To bring the index up to date after a dataset has changed, re-ingest it with `replace=true`:

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
  "$CDD/ingest/?path_to_profile=/s3/path/to/profile.json&replace=true"
```

This deletes all of the dataset's indexed content and rebuilds it from the profile. While the job runs, searches return incomplete results for that dataset.

## Removing a Dataset from the Index

```bash
curl -X DELETE -H "Authorization: Bearer $TOKEN" "$CDD/dataset/<dataset_id>"
```

This removes the dataset's indexed content only; the dataset itself is not touched. A `404` response means the dataset was not in the index.

## Following Ingestion Jobs

Get the status of a job:

```bash
curl -H "Authorization: Bearer $TOKEN" "$CDD/ingest/<job_id>"
```

The `status` moves from `pending` to `running`, and ends as `completed` or `failed` (or as `interrupted`, see below).

When a job is `completed`, compare the counts:

- `items_submitted` and `items_indexed` should be equal. A lower `items_indexed` means some items could not be embedded or stored and were skipped.
- After a new ingestion, or one with `replace=true`, `points_in_qdrant` should equal `items_indexed`. A difference means points were lost, or another job wrote to the same dataset at the same time.

List recent jobs, optionally filtered by status or dataset:

```bash
# Failed jobs (listing across datasets requires a privileged role)
curl -H "Authorization: Bearer $TOKEN" "$CDD/ingest/?status=failed&limit=50"

# All jobs of one dataset
curl -H "Authorization: Bearer $TOKEN" "$CDD/ingest/?dataset_id=<dataset_id>"
```

## Recovering from Failed or Interrupted Jobs

- **`failed`:** The job stopped with an error, given in the job's `error` field. Fix the cause, for example a missing file or an unreachable database, then re-ingest the dataset with `replace=true`.
- **`interrupted`:** The service restarted, for example during a deployment or after a crash, before the job finished. The dataset may be partially indexed. Re-ingest it with `replace=true`.

In both cases, `replace=true` is needed to remove any content the failed job had already indexed.

After a deployment or restart, check for interrupted jobs:

```bash
curl -H "Authorization: Bearer $TOKEN" "$CDD/ingest/?status=interrupted"
```

To avoid interrupted jobs, do not deploy while an ingestion is running (see [Deployment](./deployment.md)).

## Rebuilding the Whole Index

There is no endpoint that re-indexes every dataset at once. To rebuild the index, re-ingest every dataset with `replace=true` (see [Updating a Dataset](#updating-a-dataset)). Each re-ingestion deletes the dataset's existing points and replaces them, so the index is rebuilt in place, in the same collection, without needing space for a second copy. Wait for each job to finish before starting the next, since ingestion is memory-intensive.

A full rebuild is needed when the content already in the index no longer matches what the service would produce now. For example:

- After a change in how files are processed, such as a new loader or a different chunking.
- After a change of the embedding model served by TEI. To change the model:
    1. Deploy TEI with the new model.
    2. Re-ingest every dataset with `replace=true`.

    Until every dataset has been re-ingested, the collection holds vectors from both models. Queries are embedded with the new model, so dense and hybrid search results for datasets that have not been re-ingested yet are unreliable. Sparse (BM25) search is not affected.

The new model must produce 1024-dimensional vectors, like `BAAI/bge-m3`: the vector size is fixed when the collection is created, and the code expects 1024. A model with a different dimension requires a code change, and the collection must be deleted and rebuilt from scratch.

## Ingestion Job Records

Job records are kept in the ingestion jobs collection in Qdrant (see [Datastores](./datastore.md)). They are never deleted automatically. If the collection grows too large, old records can be deleted directly in Qdrant.
