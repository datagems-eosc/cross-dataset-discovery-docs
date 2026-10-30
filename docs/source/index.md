# Overview

Welcome to the documentation for the **Cross-Dataset Discovery Service**.

This service lets users search across DataGEMS datasets with natural language. It provides an API to:

- **Search** the indexed content of all datasets a user has access to, using keyword (BM25), semantic, or hybrid search, optionally assisted by an LLM.
- **Ingest** datasets into the search index from their dataset profiles, and **remove** them again.

## How to Use This Documentation

- **System:** Learn about the high-level [Architecture](./architecture.md), the [Security](./security.md) model, and the [Datastores](./datastore.md).
- **API:** Find detailed information on the [API endpoints](./api-overview.md), the [status and error codes](./error-codes.md), and the [OpenAPI specification](./openapi.md).
- **Setup & Monitoring:** Get instructions on [Deployment](./deployment.md), [Configuration](./configuration.md), [Logging](./logging.md), and [Maintenance](./maintenance.md), and see the [Automations](./automations.md) that build, scan, test and publish the service.
- **Developer Guides:** Extend the retrieval framework with the [Onboarding guide](./onboarding.md), and see how the service is checked in [Quality Assurance](./qa.md).
