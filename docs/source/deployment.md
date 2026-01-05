# Deployment

The Cross-Dataset Discovery service is designed for containerized deployment using Docker.

## Docker

The repository includes a `Dockerfile` for the build process.

### Dockerfile Stages

1. **Builder Stage:**
   - Starts from `python:3.11-slim`.
   - Installs Python dependencies.
   - **Pre-downloads** the `Qdrant/bm25` model for FastEmbed to ensure fast startup.

2. **Final Stage:**
   - Copies the application code and the pre-downloaded model cache.
   - Runs as a non-root user (`appuser`).
   - Exposes port `8000`.

### Build and Run

To build and run the container:

```bash
# 1. Build the Docker image
docker build -t cross-dataset-discovery .

# 2. Run the container
# Note: Requires network access to Qdrant and TEI services.
docker run -p 8000:8000 \
  -e DB_CONNECTION_STRING="postgresql://..." \
  -e OIDC_ISSUER_URL="https://..." \
  -e QDRANT_URL="http://host.docker.internal:6333" \
  -e TEI_URL="http://host.docker.internal:8080" \
  --name cdd-service \
  cross-dataset-discovery
```

## Dependencies

The service requires several external systems to function.

### Runtime Dependencies

1. **Qdrant Vector Database:**
   - **Requirement:** A running Qdrant instance containing the datagems collection.
   - **Config:** `QDRANT_URL`, `QDRANT_COLLECTION`.

2. **Text Embeddings Inference (TEI):**
   - **Requirement:** A running TEI service serving the `BAAI/bge-m3` model.
   - **Config:** `TEI_URL`.

3. **OIDC Provider:**
   - **Requirement:** Keycloak (or similar) for JWT validation.

4. **DataGEMS Gateway:**
   - **Requirement:** API access for fetching user permissions.

### Build-time Dependencies

- **Python 3.11**
- **FastEmbed Model:** The `Qdrant/bm25` model is downloaded during the Docker build to enable local sparse vector generation without runtime downloads.