# Cross-Dataset Discovery Service Documentation

## Configuration

The Cross-Dataset Discovery service is configured using a combination of a local configuration file and environment variables.

### Configuration Files

For local development, the service can be configured using a `.env` file placed in the root of the repository.

#### Example `.env` file

```env
# .env

# OIDC Authentication
OIDC_ISSUER_URL="https://datagems-dev.scayle.es/oauth/realms/dev"
OIDC_AUDIENCE="cross-dataset-discovery-api"
GATEWAY_API_URL="https://datagems-dev.scayle.es/gw"

# Database (PostgreSQL)
DB_CONNECTION_STRING="postgresql://user:password@localhost:5432/datagems_db"
TABLE_NAME="your_table_name"

# Search Infrastructure
QDRANT_URL="http://localhost:6333"
QDRANT_COLLECTION="datagems"
TEI_URL="http://localhost:8080"
TEI_TIMEOUT=2.0

# Secrets (for local use only)
IdpClientSecret="your-local-dev-secret"
```

### Environment Overrides

In production (Kubernetes), environment variables override the `.env` file.

| Variable | Description | Example |
|----------|-------------|---------|
| OIDC_ISSUER_URL | The base URL of the OIDC identity provider | https://.../realms/dev |
| OIDC_AUDIENCE | The audience claim required in the JWT | cross-dataset-discovery-api |
| GATEWAY_API_URL | The DataGEMS Gateway API URL for permissions | https://.../gw |
| DB_CONNECTION_STRING | PostgreSQL connection string | postgresql://... |
| QDRANT_URL | URL of the Qdrant Vector Database service | http://qdrant-service:6333 |
| QDRANT_COLLECTION | The name of the Qdrant collection to search | datagems |
| TEI_URL | URL of the Text Embeddings Inference service | http://tei-service:80 |
| TEI_TIMEOUT | Timeout (in seconds) for embedding requests | 2.0 |
| IdpClientSecret | Client secret for the identity provider | your-secret |

### Secrets

* **DB_CONNECTION_STRING**: Contains database credentials
* **IdpClientSecret**: Contains OIDC client secret