# Security

## Authentication

Authentication is handled via the **OAuth 2.0** and **OpenID Connect (OIDC)** protocols. Every request to a protected endpoint must include a valid JSON Web Token (JWT) in the `Authorization` header as a Bearer token.

The service validates incoming JWTs against the public keys (JWKS) of the OIDC provider configured in `OIDC_ISSUER_URL`. It verifies:

- the signature (`RS256`),
- the issuer (`OIDC_ISSUER_URL`),
- the audience (`OIDC_AUDIENCE`, `cross-dataset-discovery-api` by default),
- the expiry.

If the token is missing, invalid or expired, the API responds with `401 Unauthorized`. If the OIDC provider cannot be reached to fetch its keys, the API also responds with `401`.

The OIDC configuration and public keys are fetched on the first authenticated request and then cached for the lifetime of the service process.

### Unauthenticated Endpoints

The following endpoints do not require a token:

- `GET /` (liveness check)
- `GET /health` (readiness check)
- `GET /swagger`, `GET /redoc`, `GET /openapi.json` (API documentation)

## Authorization

Authorization has two layers:

1. A **role check** on the token, which applies to every protected endpoint.
2. A **dataset-level check** against the DataGEMS Gateway, which depends on the endpoint.

### Roles

The service reads roles from the token's `realm_access.roles` claim. To call any protected endpoint, the token must contain at least one of:

- `user`
- `dg_user`
- a privileged role (below)

Otherwise the API responds with `403 Forbidden`.

### Privileged Roles

These realm roles grant access to every dataset and skip all dataset-level checks:

- `dg_admin`
- `dg_system` (for system pipelines, e.g. maintenance tasks run by service accounts)
- `dg_dataset-curator`

A caller with one of these roles can search all datasets without filtering, ingest or delete any dataset, and list all ingestion jobs.

## Dataset-Level Permissions

For callers without a privileged role, the service asks the DataGEMS Gateway which datasets the caller may act on.

It calls the Gateway's `POST {GATEWAY_API_URL}/api/principal/context-grants/query` endpoint, passing the caller's own token, with a body such as `{"roles": ["dg_ds-browse"]}`. The grants returned for datasets give the list of dataset IDs the caller holds that grant on.

The grant required depends on the endpoint:

| Endpoint | Grant required | Effect |
| :--- | :--- | :--- |
| `POST /search/`, `POST /search-ap/`, `POST /corpus-analysis-search/` | `dg_ds-browse` | Results are filtered to the datasets the caller holds this grant on. |
| `POST /ingest/` | `dg_ds-edit` on the profile's dataset | Otherwise `403`. |
| `GET /ingest/{job_id}` | `dg_ds-edit` on the job's dataset | Otherwise `403`. |
| `GET /ingest/` | `dg_ds-edit` on the `dataset_id` filter | Without a privileged role, `dataset_id` is required. Otherwise `403`. |
| `DELETE /dataset/{dataset_id}` | `dg_ds-delete` on the dataset | Otherwise `403`. |

### Search Filtering

- If a search request includes `dataset_ids`, the service intersects this list with the caller's authorized datasets and searches only the datasets in both lists. If none are left, it returns an empty result list.
- All search results, regardless of the input filter, are *always* filtered against the caller's authorized dataset list before being returned. This ensures a caller can never see data from a dataset they are not permitted to access.

### Gateway Failures

If the Gateway call fails (an error response or a network error), the service treats the caller as having **no** dataset grants and logs the error. This fails closed:

- searches return no results;
- ingestion, job status and deletion requests are rejected with `403`.

Callers with a privileged role are not affected, because the Gateway is not called for them.

## Configuration

The authentication and authorization settings are `OIDC_ISSUER_URL`, `OIDC_AUDIENCE` and `GATEWAY_API_URL`. See [Configuration](./configuration.md).

The `DEV_MODE` setting does **not** disable authentication in the API. It only affects the local Streamlit test app.
