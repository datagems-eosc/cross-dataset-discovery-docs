# Deployment

The Cross-Dataset Discovery service is deployed as a container on Kubernetes, alongside an embedding service (TEI) and a Qdrant vector database.

## Components

| Component | What it is | Notes |
| :--- | :--- | :--- |
| **API** | This service: a FastAPI app served by `gunicorn` with one `uvicorn` worker, from the `ghcr.io/datagems-eosc/cross-dataset-discovery-api` image. | Runs as a single replica (see [Scaling](#scaling)). |
| **TEI** | [Hugging Face Text Embeddings Inference](https://github.com/huggingface/text-embeddings-inference) serving `BAAI/bge-m3`. Produces the dense embeddings for search and ingestion. | Can run on CPU (`text-embeddings-inference:cpu-*` image) or on GPU (`cuda-*` image). |
| **Qdrant** | The vector database holding the index and the ingestion job records. | A shared Qdrant instance, not deployed with this service. The collections are created automatically on first ingestion. |

The API also depends on these external services:

- **OIDC provider** (Keycloak), to validate tokens.
- **DataGEMS Gateway**, to look up dataset grants.
- **Dataset database servers** (PostgreSQL), to read `sql-db` datasets during ingestion.
- **LLM providers** (SCAYLE and/or Amazon Bedrock), for the LLM steps of `POST /corpus-analysis-search/` (see [Configuration](./configuration.md#llm-corpus-analysis-search)).

## Docker Image

The image is built from the `Dockerfile` in the root of the code repository.

### Build Stages

1. **Builder stage** (`python:3.12-slim`):
    - Installs a CPU-only build of PyTorch first, then the API's Python dependencies (`requirements.txt`). The API runs on CPU, and the default CUDA build of PyTorch would add several GB to the image.
    - Pre-downloads the FastEmbed `Qdrant/bm25` model, used to compute sparse (BM25) vectors.
    - Pre-downloads the `BAAI/bge-m3` tokenizer, used for token-based chunking during ingestion.

    Because both models are part of the image, the service does not need Hugging Face access at runtime.

2. **Final stage** (`python:3.12-slim`):
    - Copies the installed packages, the model cache and the application code.
    - Sets `FASTEMBED_CACHE_PATH` and `CHUNK_TOKENIZER_PATH` to the bundled models.
    - Runs as a non-root user (`appuser`).
    - Exposes port `8000`, with a Docker `HEALTHCHECK` on `/health` every 30 seconds.
    - Starts `gunicorn` with a single `uvicorn` worker.

### Build and Run Locally

```bash
# 1. Build the image
docker build -t cross-dataset-discovery .

# 2. Run the container (requires network access to Qdrant and TEI)
docker run -p 8000:8000 \
  -e OIDC_ISSUER_URL="https://<identity-provider>/oauth/realms/<realm>" \
  -e GATEWAY_API_URL="https://<gateway-host>/gw" \
  -e QDRANT_URL="http://host.docker.internal:6333" \
  -e QDRANT_COLLECTION="<collection>" \
  -e TEI_URL="http://host.docker.internal:8080" \
  --name cdd-service \
  cross-dataset-discovery
```

See [Configuration](./configuration.md) for all variables. Mount the storage holding the dataset profiles and files if you want to test ingestion.

### Publishing

Pushing a Git tag that matches `v*` (e.g. `v1.17.2`) builds the image and publishes it to `ghcr.io/datagems-eosc/cross-dataset-discovery-api` under that tag. See [Automations](./automations.md#docker-image-publishing).

## Kubernetes Setup

The manifests are kept in a deployment repository, one per environment (development and production). For the API they define:

- **ConfigMap:** the non-secret configuration (OIDC, Gateway, Qdrant, TEI, ingestion settings).
- **Secret:** synced from the secret store. Holds the LLM provider tokens and the database credentials (see [Configuration](./configuration.md#secrets)).
- **Deployment:**
    - The image tag to run.
    - The environment variables, taken from the ConfigMap and the Secret. `ROOT_PATH` is set to the path prefix the API is served under.
    - A volume with the dataset storage. Dataset profiles and the files they reference are read from it, so `path_to_profile` and the profiles' `source_file` paths must point inside this mount.
    - The `gunicorn` command, overridden to use a longer worker `--timeout`.
    - The probes: liveness on `GET /`, readiness on `GET /health`.
    - Resource requests and limits. Ingestion is the most memory-hungry operation; if the pod runs out of memory during ingestion, lower `INGESTION_BATCH_SIZE`.
- **Service:** exposes port `8000` inside the cluster.
- **Routing:** how requests reach the Service depends on the platform setup of each environment. If the API is served under a path prefix, `ROOT_PATH` must match it.

TEI has its own Deployment, Service and persistent volume (for the model cache), with separate CPU and GPU variants.

### Scaling

Run the API with **one replica and one worker**:

- Ingestion jobs run as background tasks inside the worker that received the request.
- When the service starts, it marks every unfinished job as `interrupted`, assuming that no other worker could still be running it.

More replicas or workers would break that assumption.

## Deploying a New Version

### Prerequisites

- Access to the code repository and to the deployment repository of the target environment.
- `kubectl` configured for the target cluster (including any required VPN).

Verify access by listing the pods in the service's namespace:

```bash
kubectl get pods -n <namespace>
```

You should see a `cross-dataset-discovery-api-...` pod.

### 1. Tag a Release

Commit your changes to the code repository, then create and push a version tag:

```bash
git tag vX.Y.Z
git push origin vX.Y.Z
```

### 2. Wait for the Image Build

Follow the build in the code repository's **Actions** tab. When it finishes, the image `ghcr.io/datagems-eosc/cross-dataset-discovery-api:vX.Y.Z` is available.

### 3. Update the Deployment

In the deployment repository, set the new tag in the API Deployment manifest:

```yaml
image: ghcr.io/datagems-eosc/cross-dataset-discovery-api:vX.Y.Z
```

If the release needs new or changed settings, update the ConfigMap (or the secret store) as well. Commit the change.

### 4. Apply

Run the deployment repository's on-demand deploy workflow (**Deploy to Dev** or **Deploy to Prod**) from its **Actions** tab, entering the image tag. Make sure the tag is also committed in the manifest (step 3), so the repository matches what is running.

### 5. Monitor the Rollout

```bash
kubectl get pods -n <namespace> -w
```

The new pod becomes ready once `GET /health` succeeds; the old pod is then terminated.

When it starts, the new pod marks any unfinished ingestion jobs as `interrupted`. Avoid deploying while an ingestion is running, or re-run it afterwards with `replace=true` (see [Maintenance](./maintenance.md)).
