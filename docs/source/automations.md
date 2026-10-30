# Automations

This project uses GitHub Actions to automate builds, code analysis, security scanning, testing, and documentation deployment. Apart from image publishing, which runs on every version tag, all workflows are started manually from the repository's **Actions** tab (`workflow_dispatch`).

| Workflow | Repository | Trigger | Purpose |
| :--- | :--- | :--- | :--- |
| Build and Push Search API Docker Image | code | Push of a `v*` tag | Build and publish the Docker image |
| Vulnerability Scan (On-Demand) | code | Manual | Trivy scan of the repository configuration and a published image |
| CodeQL Scan (On-Demand) | code | Manual | Static code analysis |
| Code Metrics (On-Demand) | code | Manual | Radon complexity and maintainability report |
| Test Scenario (On-Demand) | code | Manual | API tests against the deployed service |
| Deploy Docs (On-Demand) | docs | Manual | Publish this documentation site |
| Deploy to Dev (On-Demand) | deployment (dev) | Manual | Apply the Kubernetes manifests to the development cluster (see [Deployment](./deployment.md#4-apply)) |
| Deploy to Prod (On-Demand) | deployment (prod) | Manual | Apply the Kubernetes manifests to the production cluster (see [Deployment](./deployment.md#4-apply)) |

## Dockerfile

The production container image is built by the `Dockerfile` in the root of the code repository, in two stages:

-   **Stage 1 (Builder):** Installs CPU-only PyTorch and the API's Python dependencies, and pre-downloads the models the service needs at runtime: the FastEmbed `Qdrant/bm25` model and the `BAAI/bge-m3` tokenizer.
-   **Stage 2 (Final):** Copies only the installed packages, the model cache and the application code onto a clean `python:3.12-slim` base. It creates a non-root `appuser`, defines a `HEALTHCHECK`, and starts the `gunicorn` server.

See [Deployment](./deployment.md#docker-image) for details.

## Docker Image Publishing

**Workflow:** `.github/workflows/docker-publish.yml`

This workflow builds the service's Docker image and publishes it to the GitHub Container Registry (ghcr.io).

-   **Trigger:** Runs automatically whenever a Git tag matching `v*` (e.g. `v1.17.2`) is pushed.
-   **Process:**
    1.  Logs into the GitHub Container Registry using the workflow's `GITHUB_TOKEN`.
    2.  Uses `docker/metadata-action` to derive the image tag from the Git tag. The image gets exactly that tag; no `latest` tag is published.
    3.  Builds the image with Docker Buildx, reusing layers from the GitHub Actions cache to speed up builds.
    4.  Pushes the image to `ghcr.io/datagems-eosc/cross-dataset-discovery-api`.

## Vulnerability Scanning

**Workflow:** `.github/workflows/vulnerability-scan-on-demand.yml`

This workflow scans the project's configuration and a published Docker image using the **Trivy** security scanner.

-   **Trigger:** Manual. It requires an `image_tag` input naming the published image to scan.
-   **Process:**
    1.  **Configuration Scan:** Scans the `Dockerfile` and other configuration files of the default branch for security misconfigurations.
    2.  **Image Scan:** Pulls `ghcr.io/datagems-eosc/cross-dataset-discovery-api:<image_tag>` and scans its operating system packages and Python libraries for known vulnerabilities (CVEs) of `CRITICAL` or `HIGH` severity.
    3.  **Artifacts:** The JSON reports of both scans are uploaded as build artifacts (`vulnerability-config-scan` and `vulnerability-image-scan`).

## Static Code Analysis

**Workflow:** `.github/workflows/code-scan-on-demand.yml`

This workflow runs **GitHub CodeQL** to find potential bugs, security vulnerabilities, and quality issues.

-   **Trigger:** Manual.
-   **Process:**
    1.  Runs two analysis jobs: one for the Python code and one for the GitHub Actions workflow files.
    2.  Uses the `security-extended` and `security-and-quality` query suites.
    3.  Uploads the results to the repository's **Security** tab, under **Code scanning alerts**.

## Code Metrics

**Workflow:** `.github/workflows/code-metrics-on-demand.yml`

This workflow reports on the complexity and maintainability of the API code using **Radon**.

-   **Trigger:** Manual.
-   **Process:**
    1.  Installs the `radon` Python package.
    2.  Analyzes the `cross_dataset_discovery/darelabdb/api_datagems_cross_dataset_discovery/app` directory:
        -   **Maintainability Index (MI):** A score from 0 to 100 indicating how easy the code is to maintain.
        -   **Cyclomatic Complexity (CC):** The number of independent paths through each function, indicating its complexity.
        -   **Raw Lines of Code (LOC):** Basic code size metrics.
    3.  Uploads the report as a build artifact named `code-metrics-report`.

## API Tests

**Workflow:** `.github/workflows/test-on-demand.yml`

This workflow runs the Postman test collection against the **deployed** service with Newman. It takes a `tag` input (default `main`) that selects which version of the test collection to run. See [Quality Assurance](./qa.md#testing) for details.

## Documentation

**Workflow:** `.github/workflows/deploy-docs-on-demand.yml` (in the documentation repository)

This documentation site is built and published by this workflow.

-   **Trigger:** Manual, with a version number (e.g. `1.0.0`) as input.
-   **Process:**
    1.  Installs `mkdocs`, `mkdocs-material`, `mike`, and the other required plugins.
    2.  Builds the static site from the Markdown files in `docs/`.
    3.  Uses `mike` to commit the built site to the `gh-pages` branch, under the given version.
    4.  Points the `latest` alias at this version and makes `latest` the default, so visitors see the most recent documentation.
