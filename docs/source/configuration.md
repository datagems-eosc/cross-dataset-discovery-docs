# Configuration

The Cross-Dataset Discovery service is configured through environment variables. In Kubernetes they are provided by a ConfigMap, a Secret, and the Deployment manifest (see [Deployment](./deployment.md)). The values used in each environment are kept in the deployment repository.

## Local Development (`.env`)

For local development, the variables can be placed in a `.env` file instead. Put it in the `cross_dataset_discovery/` directory (the one containing the `darelabdb` package) and start the service from there:

```bash
cd cross_dataset_discovery
uvicorn darelabdb.api_datagems_cross_dataset_discovery.app.main:app --port 8000
```

Environment variables that are set take precedence over values in `.env`.

### Example `.env` file

```env
# Authentication & authorization
OIDC_ISSUER_URL="https://<identity-provider>/oauth/realms/<realm>"
OIDC_AUDIENCE="cross-dataset-discovery-api"
GATEWAY_API_URL="https://<gateway-host>/gw"

# Search infrastructure
QDRANT_URL="http://localhost:6333"
QDRANT_COLLECTION="cross_dataset_unified"
TEI_URL="http://localhost:8080"

# Ingestion from SQL datasets (only needed to ingest sql-db files)
SQL_DB_BASE_URLS="postgresql://user:password@localhost:5432"

# LLM for /corpus-analysis-search/ (optional)
AWS_BEARER_TOKEN_BEDROCK="your-bedrock-api-key"
```

## Environment Variables

The *Default* column is the value used when the variable is not set.

### Authentication & Authorization

| Variable | Description | Default |
| :--- | :--- | :--- |
| `OIDC_ISSUER_URL` | Base URL of the OIDC identity provider. Tokens must be issued by it, and its public keys are used to verify them. | `https://datagems-dev.scayle.es/oauth/realms/dev` |
| `OIDC_AUDIENCE` | The audience (`aud`) tokens must be issued for. | `cross-dataset-discovery-api` |
| `GATEWAY_API_URL` | Base URL of the DataGEMS Gateway, queried for the caller's dataset grants. | `https://datagems-dev.scayle.es/gw` |

### Search Infrastructure

| Variable | Description | Default |
| :--- | :--- | :--- |
| `QDRANT_URL` | URL of the Qdrant vector database. | `http://localhost:6333` |
| `QDRANT_COLLECTION` | Qdrant collection holding the indexed datasets. | `cross_dataset_unified` |
| `TEI_URL` | URL of the Text Embeddings Inference (TEI) service that produces dense embeddings. | `http://localhost:8080` |
| `TEI_TIMEOUT` | Timeout, in seconds, for embedding requests to TEI (during both search and ingestion). | `30` |

### Ingestion

| Variable | Description | Default |
| :--- | :--- | :--- |
| `INGESTION_BATCH_SIZE` | Number of items embedded and written to Qdrant per batch. Lower values reduce memory use. | `64` |
| `INGESTION_JOBS_COLLECTION` | Qdrant collection that stores ingestion job records. Created automatically. | `<QDRANT_COLLECTION>_ingestion_jobs` |
| `SQL_DB_BASE_URLS` | Comma-separated SQLAlchemy URLs of the database servers that host `sql-db` datasets, **without** a database name (e.g. `postgresql://user:pass@host:5432`). For each `sql-db` file, the servers are tried in order until one has a database with the file's name. If empty, `sql-db` files are skipped. | empty |
| `SQL_DB_BASE_URL` | Single-URL form of `SQL_DB_BASE_URLS`, kept for compatibility. Used only if `SQL_DB_BASE_URLS` is empty. | empty |
| `SQL_DB_MAX_TABLE_MB` | Tables larger than this (in MB) are not indexed row by row. Instead, each column is indexed once, with the distinct values found in a sample of rows. `0` disables the limit. | `512` |
| `SQL_DB_SAMPLE_ROWS` | Number of rows sampled from tables over `SQL_DB_MAX_TABLE_MB`. | `10000` |
| `SQL_DB_READ_CHUNK_ROWS` | Number of rows read from the database at a time when indexing a table row by row. | `20000` |

### LLM (Corpus Analysis Search)

| Variable | Description | Default |
| :--- | :--- | :--- |
| `AWS_BEARER_TOKEN_BEDROCK` | Amazon Bedrock API key, used by `POST /corpus-analysis-search/`. Without it the LLM steps fail and the endpoint falls back to a plain search (see [API Overview](./api-overview.md#corpus-analysis-search)). | not set |

### Runtime

| Variable | Description | Default |
| :--- | :--- | :--- |
| `ROOT_PATH` | Path prefix under which the API is served behind the reverse proxy. Used to build correct links in `/swagger` and `/redoc`. | empty |
| `CHUNK_TOKENIZER_PATH` | Path to the `BAAI/bge-m3` `tokenizer.json` used for token-based chunking during ingestion. If not set, the tokenizer is downloaded from Hugging Face. The Docker image includes the file and sets this variable. | not set |
| `FASTEMBED_CACHE_PATH` | Directory with the cached FastEmbed `Qdrant/bm25` model used for sparse (BM25) vectors. The Docker image includes the model and sets this variable. | FastEmbed default |

`DEV_MODE` also exists in the settings, but it has no effect on the API. In particular, it does **not** disable authentication.

## Fixed Settings

These are set in the code and cannot be changed through configuration:

| Setting | Value |
| :--- | :--- |
| Dense embedding model | Whatever model the TEI service serves. It must be `BAAI/bge-m3` (1024 dimensions) to match the indexed collection. |
| Sparse embedding model | FastEmbed `Qdrant/bm25`, computed inside the API. |
| Vector quantization | Binary quantization, kept in RAM. |
| LLM | `anthropic.claude-3-haiku-20240307-v1:0` on Amazon Bedrock, region `eu-central-1`, temperature `0`. |
| LLM relevance threshold | Results scored below `3` are dropped by `POST /corpus-analysis-search/`. |

## Secrets

These values are confidential. In Kubernetes they are kept in a secret store and provided to the service through a Secret, never in the ConfigMap:

| Variable | Used for |
| :--- | :--- |
| `AWS_BEARER_TOKEN_BEDROCK` | Amazon Bedrock API key (LLM). |
| `SQL_DB_BASE_URLS` | Contains the database credentials. In Kubernetes it is assembled in the Deployment from separate secret values for the user, password, host and port. |

Never commit real secrets to a `.env` file.
