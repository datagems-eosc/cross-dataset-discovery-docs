# System Architecture

The Cross-Dataset Discovery Service is designed as a microservices-based system within the DataGEMS ecosystem. It acts as a specialized search layer that orchestrates retrieval across vector databases and inference services.

## Core Components

1.  **FastAPI Application (API Pod):** The entry point of the service. It handles authentication, authorization, and orchestrates the search logic. It calculates sparse embeddings locally using `FastEmbed` (CPU) to minimize latency.

2.  **Vector Database (Qdrant Pod):** A high-performance vector search engine. It stores both dense (semantic) and sparse (keyword) vectors for the dataset corpus and performs the core retrieval and ranking operations (including Hybrid Search via Reciprocal Rank Fusion).

3.  **Embedding Service (TEI Pod):** A dedicated **Text Embeddings Inference** service (based on Hugging Face TEI). It provides an API for generating dense vectors (using `BAAI/bge-m3`) from input text.

4.  **Security Layer:** Intercepts requests to validate OIDC tokens and communicates with the DataGEMS Gateway to enforce dataset-level access control.

## Request Flow

A typical search request follows this path:

1.  **Auth:** A user sends a `POST /search/` request with a valid JWT. The Security Layer validates the token and fetches authorized dataset IDs from the DataGEMS Gateway.
2.  **Vector Generation:**
    *   **Sparse Mode:** The API Pod generates a sparse vector (BM25) locally.
    *   **Dense Mode:** The API Pod sends the query text to the **TEI Service** to get a dense vector.
    *   **Hybrid Mode:** Both operations are performed.
3.  **Retrieval:** The API Pod sends the generated vector(s) and the list of authorized dataset IDs (as a filter) to **Qdrant**.
4.  **Ranking:** Qdrant performs the search (using HNSW for dense, Inverted Index for sparse, or RRF for hybrid) and returns the top results.
5.  **Response:** The API formats the results and returns them to the user.