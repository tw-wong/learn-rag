# ARCHITECTURE.md - System Architecture

> **Version:** 1.0
> **Last Updated:** 2026-01-12
> **Status:** Approved
> **Owner:** Technical Lead (TL)

---

## 1. Purpose & Scope

### What This Document Covers

This document defines the technical architecture for the RAG API system, including:

- System component structure and responsibilities
- Data flow between components
- Key architectural decisions with rationale
- Technology choices and defaults
- Non-functional requirements and constraints
- Testing strategy at the architecture level

### What Is Explicitly Out of Scope

- **API endpoint specifications** (see API_SPEC.md)
- **Request/response schemas and payloads** (see API_SPEC.md)
- **Task breakdowns and assignments** (see TASKS.md)
- **Implementation code** (coding standards will be defined in CLAUDE.md once created)
- **Product requirements and acceptance criteria** (see PRD.md)
- **Team processes and workflows** (see TEAM.md)
- **Production-grade security, auth, or multi-tenancy** (non-goal per PRD)
- **Cloud deployment or SaaS considerations** (non-goal per PRD)

---

## 2. System Overview

The RAG API is a self-contained, locally-running system that enables users to:

1. **Upload PDF documents** for indexing (asynchronous from API perspective; processing is manually triggered via CLI)
2. **Query the knowledge base** using natural language (synchronous, streaming)
3. **Check processing status** of uploaded documents

### Design Philosophy

- **Local-first**: All processing occurs on the local machine with no external API dependencies
- **Learning-friendly**: Clean separation of concerns with SOLID principles for educational value
- **Modular and extensible**: Provider pattern allows swapping components without code changes
- **Simple over complex**: Minimal viable architecture that meets requirements without over-engineering

### Major Components

| Component | Responsibility |
|-----------|---------------|
| **FastAPI Application** | REST API server; handles HTTP requests for upload, query, status, and health |
| **CLI Worker** | Command-line tool for manual document processing from job queue |
| **MongoDB** | Persistent storage for job queue and document metadata |
| **Qdrant** | Vector database for document embeddings and similarity search |
| **Ollama** | Local LLM runtime for text generation and embedding creation |
| **LlamaIndex** | RAG orchestration framework; handles chunking, indexing, and retrieval |
| **File Storage** | Local filesystem for uploaded PDF files |

---

## 3. Architecture Diagram (Text-Based)

### High-Level Component Diagram

```
+-----------------------------------------------------------------------------------+
|                                   CLIENT                                          |
|   (curl, Postman, custom app)                                                     |
+-----------------------------------------------------------------------------------+
                |                    |                       |
                | POST /upload       | GET /status           | POST /query
                v                    v                       v
+-----------------------------------------------------------------------------------+
|                              FASTAPI APPLICATION                                   |
|  +---------------+    +---------------+    +---------------+    +---------------+ |
|  | Upload API    |    | Status API    |    | Query API     |    | Health API    | |
|  +---------------+    +---------------+    +---------------+    +---------------+ |
|         |                    |                    |                              |
|         v                    v                    v                              |
|  +---------------+    +---------------+    +---------------+                     |
|  | StorageService|    | JobService    |    | QueryService  |                     |
|  +---------------+    +---------------+    +---------------+                     |
+-----------------------------------------------------------------------------------+
         |                     |                    |
         v                     v                    |
+----------------+    +----------------+            |
| FILE STORAGE   |    |   MONGODB      |            |
| (data/uploads) |    | (job queue)    |            |
+----------------+    +----------------+            |
                             ^                      |
                             |                      v
+-----------------------------------------------------------------------------------+
|                              CLI WORKER                                           |
|  +---------------+    +---------------+    +----------------------------------+   |
|  | JobService    |    |DocumentService|    | LlamaIndex Orchestration         |   |
|  | (fetch jobs)  |    | (process docs)|    | (chunk, embed, index)            |   |
|  +---------------+    +---------------+    +----------------------------------+   |
+-----------------------------------------------------------------------------------+
                                  |                          |
                                  v                          v
                         +----------------+         +----------------+
                         |    QDRANT      |         |    OLLAMA      |
                         | (vector store) |         | (LLM runtime)  |
                         +----------------+         +----------------+
```

### Upload Flow (Asynchronous)

```
Client                    FastAPI              Storage         MongoDB
  |                          |                    |                |
  |-- POST /upload (PDF) --->|                    |                |
  |                          |-- save file ------>|                |
  |                          |<-- file path ------|                |
  |                          |-- create job --------------------->|
  |                          |<-- job_id -------------------------|
  |<-- 200 OK (job_id) ------|                    |                |
  |                          |                    |                |
```

### Processing Flow (CLI Worker - Manual Trigger)

```
Operator                CLI Worker           MongoDB         Qdrant       Ollama
  |                          |                  |               |            |
  |-- run command ---------->|                  |               |            |
  |                          |-- get pending -->|               |            |
  |                          |<-- jobs list ----|               |            |
  |                          |                  |               |            |
  |                          |-- update status (PROCESSING) --->|            |
  |                          |                  |               |            |
  |                          |-- load PDF, chunk text -----------------------|
  |                          |-- generate embeddings ----------------------->|
  |                          |<-- embeddings --------------------------------|
  |                          |-- store vectors ------------>|               |
  |                          |                  |               |            |
  |                          |-- update status (COMPLETED) ->|              |
  |<-- progress output ------|                  |               |            |
  |                          |                  |               |            |
```

### Query Flow (Synchronous with Streaming)

```
Client              FastAPI            QueryService        Qdrant          Ollama
  |                    |                    |                 |               |
  |-- POST /query ---->|                    |                 |               |
  |                    |-- query() -------->|                 |               |
  |                    |                    |-- embed query --------------->|
  |                    |                    |<-- query vector --------------|
  |                    |                    |-- search ------>|               |
  |                    |                    |<-- top-k chunks-|               |
  |                    |                    |-- generate (stream) --------->|
  |                    |                    |<-- tokens (stream) -----------|
  |<-- SSE stream -----|<-- stream tokens --|                 |               |
  |                    |                    |                 |               |
```

---

## 4. Component Breakdown

### 4.1 FastAPI Application

**Responsibility**: HTTP request handling, input validation, response formatting, dependency injection

**Inputs**:
- HTTP requests (file uploads, queries, status checks)

**Outputs**:
- HTTP responses (JSON, streaming SSE)

**Key Interactions**:
- Delegates business logic to services
- Uses FastAPI dependency injection to wire services
- Manages MongoDB connection lifecycle (startup/shutdown)

---

### 4.2 CLI Worker

**Responsibility**: Manual document processing from job queue

**Inputs**:
- Command-line invocation with optional batch size argument
- Pending jobs from MongoDB queue

**Outputs**:
- Console output showing progress and results
- Updated job statuses in MongoDB
- Indexed vectors in Qdrant

**Key Interactions**:
- Reads pending jobs from MongoDB
- Orchestrates document processing via DocumentService
- Updates job status on completion or failure
- Runs synchronously (one job at a time)

---

### 4.3 Services Layer

#### JobService

**Responsibility**: Job queue management (CRUD operations on jobs)

**Inputs**: Job creation requests, job queries, status updates

**Outputs**: Job records, job IDs, status confirmations

**Key Interactions**: MongoDB for persistence

---

#### DocumentService

**Responsibility**: Document indexing orchestration

**Inputs**: Job ID and file path

**Outputs**: Indexed document chunks in vector store

**Key Interactions**:
- LlamaIndex for PDF loading and chunking
- Embedding provider for vector generation
- Vector store provider for storage
- JobService for status updates

---

#### QueryService

**Responsibility**: Query processing and response generation

**Inputs**: User question

**Outputs**: Streaming LLM response

**Key Interactions**:
- Embedding provider for query vectorization
- Vector store provider for similarity search
- LLM provider for response generation

---

#### StorageService

**Responsibility**: File system operations for uploads

**Inputs**: Uploaded file, job ID

**Outputs**: Saved file path

**Key Interactions**: Local filesystem

---

### 4.4 Providers Layer

Providers implement abstract interfaces, enabling component swapping via configuration.

#### Vector Store Provider (Qdrant)

**Responsibility**: Vector storage and similarity search

**Interface**: `VectorStoreInterface`
- `create_collection(name, dimension)`
- `upsert(collection, vectors, metadata)`
- `search(collection, query_vector, top_k)`

---

#### LLM Provider (Ollama)

**Responsibility**: Text generation

**Interface**: `LLMInterface`
- `generate(prompt)` - Complete generation
- `stream_generate(prompt)` - Token-by-token streaming

---

#### Embedding Provider (Ollama)

**Responsibility**: Text to vector conversion

**Interface**: `EmbeddingInterface`
- `embed(text)` - Single text embedding
- `embed_batch(texts)` - Batch embedding

---

### 4.5 External Services

#### MongoDB

**Responsibility**: Job queue persistence, document metadata

**Data Stored**:
- Job records (job_id, filename, file_path, status, timestamps, error_message, chunks_created)

**Indexes**:
- `job_id` (unique)
- `status` (for pending job queries)
- `created_at` (for ordering)

---

#### Qdrant

**Responsibility**: Vector storage and similarity search

**Data Stored**:
- Document chunk embeddings with metadata

**Configuration**:
- Single collection for all documents (unified knowledge base)
- Vector dimension matches embedding model output

---

#### Ollama

**Responsibility**: Local LLM inference

**Capabilities Used**:
- Text generation (Llama models)
- Text embedding generation

---

## 5. Key Architectural Decisions

### AD-1: Monolithic Application with Modular Design

**Decision**: Single deployable application with clear internal module boundaries

**Rationale**:
- Learning project benefits from simplicity
- Docker Compose orchestrates external services
- Provider pattern provides extensibility without microservice complexity
- Easier debugging and deployment for local development

**Trade-offs**:
- Not horizontally scalable (acceptable for learning scope)
- Single point of failure for API (acceptable for local use)

---

### AD-2: Manual Job Processing (CLI Worker)

**Decision**: No background worker in MVP; processing triggered manually via CLI command

**Rationale**:
- Gives operators explicit control over when resource-intensive processing occurs
- Simplifies architecture (no message broker, no daemon worker, no process management)
- Appropriate for learning context where predictability matters
- Reduces system complexity and failure modes

**Trade-offs**:
- Documents not immediately queryable after upload
- Requires manual intervention to process queue

**Note**: Document processing is "asynchronous" from the API perspective (upload returns immediately with job_id), but there is no automatic background processing in MVP. An operator must manually run the CLI worker to process pending jobs.

---

### AD-3: Single Vector Collection (Unified Knowledge Base)

**Decision**: All documents indexed into a single Qdrant collection

**Rationale**:
- Simplifies query logic (no collection routing needed)
- Provides holistic answers across all documents
- Reduces operational complexity

**Trade-offs**:
- Cannot query specific document subsets (acceptable for MVP)
- No document isolation between users (single-user system)

---

### AD-4: Synchronous Query with Streaming Response

**Decision**: Query endpoint blocks until completion but streams tokens

**Rationale**:
- Streaming provides responsive UX (first token < 2 seconds)
- Synchronous request simplifies client implementation
- SSE provides reliable streaming over HTTP

**Trade-offs**:
- Long queries hold connection open
- No query result caching

---

### AD-5: Provider Pattern for Swappable Components

**Decision**: Abstract interfaces with factory-based instantiation

**Rationale**:
- Enables component swapping via configuration
- Facilitates unit testing with mocks
- Demonstrates SOLID principles (learning goal)
- Allows future migration to cloud providers

**Trade-offs**:
- Slight added complexity over direct implementations
- Factories require updates when adding providers

---

### AD-6: Error Handling Strategy

**Decision**: Fail fast at boundaries, atomic job completion

**Rationale**:
- API returns clear HTTP errors for invalid input
- Processing continues for remaining jobs after individual failure
- Job completion is atomic: a job is marked COMPLETED only when ALL chunks are successfully indexed

**Job Completion Semantics**:
- **COMPLETED**: All chunks from the document have been successfully indexed in the vector store
- **FAILED**: Any error during processing (PDF loading, chunking, embedding, or indexing) marks the entire job as FAILED with an error message
- There is no partial success state; if any chunk fails to index, the entire job fails
- This ensures query results only include fully-indexed documents

**Error Categories**:
| Category | Handling |
|----------|----------|
| Invalid input (upload) | 400 Bad Request immediately |
| Job not found | 404 Not Found |
| Processing failure | Mark job FAILED, continue with next job |
| Service unavailable | 503 with retry guidance |

---

### AD-7: Files Retained Indefinitely

**Decision**: Uploaded PDFs are never automatically deleted

**Rationale**:
- Enables reprocessing if needed
- Supports debugging and troubleshooting
- Storage is inexpensive for learning project scope

**Trade-offs**:
- Requires manual cleanup if storage fills
- No lifecycle management in MVP

---

## 6. Data & State Management

### 6.1 Persistent State

| Data | Storage | Lifecycle |
|------|---------|-----------|
| Job records | MongoDB | Created on upload, updated on processing, never deleted |
| PDF files | Local filesystem | Created on upload, never deleted |
| Document vectors | Qdrant | Created during processing, never deleted |

### 6.2 Transient State

| Data | Scope | Notes |
|------|-------|-------|
| MongoDB connections | Application lifecycle | Created on startup, closed on shutdown |
| In-flight requests | Request duration | Standard HTTP request lifecycle |
| Streaming response buffers | Query duration | Cleared after response completes |

### 6.3 Job State Lifecycle

```
                  [Upload received]
                         |
                         v
                    +---------+
                    | PENDING |
                    +---------+
                         |
          [CLI worker picks up job]
                         |
                         v
                   +------------+
                   | PROCESSING |
                   +------------+
                    /          \
      [Success]   /              \   [Failure]
                 v                v
          +-----------+      +--------+
          | COMPLETED |      | FAILED |
          +-----------+      +--------+
```

**State Transitions**:
- `PENDING -> PROCESSING`: Worker starts processing
- `PROCESSING -> COMPLETED`: Processing succeeds (chunks_created populated)
- `PROCESSING -> FAILED`: Processing fails (error_message populated)
- No transitions out of COMPLETED or FAILED (terminal states)

**Stuck Job Handling**: Jobs stuck in PROCESSING require manual database intervention (acceptable for MVP)

### 6.4 Document Ingestion Lifecycle

1. **Upload**: PDF saved to filesystem, job created as PENDING
2. **Processing**: PDF loaded, text chunked, embeddings generated, vectors stored
3. **Indexed**: Document available for queries (no explicit state, implied by COMPLETED job)

---

## 7. Technology Choices (Defaults)

All choices are defaults that can be changed via configuration unless marked as required.

### Core Stack

| Component | Technology | Rationale | Changeability |
|-----------|------------|-----------|---------------|
| Language | Python 3.11+ | Required per PRD; ecosystem fit for ML/AI | Required |
| Web Framework | FastAPI | Async support, OpenAPI generation, dependency injection | Default |
| Job Queue DB | MongoDB | Document flexibility, easy local setup | Default |
| Vector Store | Qdrant | Purpose-built for vectors, local deployment | Swappable via provider |
| LLM Runtime | Ollama | Local-first, open-source models | Swappable via provider |
| RAG Framework | LlamaIndex | Mature RAG tooling, good abstraction | Default |

### Supporting Tools

| Component | Technology | Rationale |
|-----------|------------|-----------|
| CLI Framework | Typer | Developer-friendly, type-safe |
| Config Management | pydantic-settings | Type-safe, environment variable support |
| ASGI Server | uvicorn | FastAPI recommended server |
| Container Runtime | Docker + Compose | Service orchestration for local dev |
| Console Output | rich | Better terminal UX for CLI worker |

### Model Defaults

| Model Type | Default | Notes |
|------------|---------|-------|
| LLM | llama2 (Ollama) | General-purpose, runs on CPU |
| Embedding | nomic-embed-text (Ollama) | Default embedding model; configurable via environment |

**Note on Embedding Model**: The embedding model is configured independently from the LLM model. The default (`nomic-embed-text`) is a suggestion; the actual model and its vector dimension will be validated during implementation. Any embedding model supported by Ollama can be used via configuration.

---

## 8. Non-Functional Considerations

### 8.1 Performance Assumptions

| Metric | Target | Conditions |
|--------|--------|------------|
| First token latency | < 2 seconds | Warm model (Ollama loaded), no cold start |
| Query throughput | 100+ queries/minute | 8GB RAM, 4 CPU cores |
| Upload response time | < 1 second | Just file save + job creation |
| Setup time | < 10 minutes | Clone to running system |

### 8.2 Scalability Boundaries

**This system is NOT designed for**:
- Horizontal scaling (single instance only)
- High concurrency (sequential job processing)
- Large file volumes (no partitioning or sharding)
- Multi-user isolation

**Acceptable Limits** (MVP):
- Single API instance
- Sequential CLI worker processing
- Single Qdrant collection
- 10MB file size limit
- 500 character query limit

### 8.3 Reliability & Recovery

**Recovery Scenarios**:

| Failure | Recovery Path |
|---------|---------------|
| API crash | Restart container; no data loss (state in MongoDB) |
| Worker crash mid-job | Job remains PROCESSING; requires manual reset |
| MongoDB unavailable | API returns 503; worker exits |
| Qdrant unavailable | Processing fails; queries fail |
| Ollama unavailable | Processing fails; queries fail |

**Data Durability**:
- MongoDB data persisted to volume
- Qdrant data persisted to volume
- PDF files persisted to filesystem volume

### 8.4 Security Boundaries

**In Scope**:
- Input validation (file type, size limits, query length)
- Error message sanitization (no internal paths leaked)

**Explicitly Out of Scope** (per PRD non-goals):
- Authentication / authorization
- Multi-user data isolation
- Encryption at rest
- Rate limiting
- Audit logging

---

## 9. Testing Strategy (Architecture-Level)

### 9.1 Unit Test Boundaries

**What is unit tested**:
- Services (mocked dependencies)
- Providers (mocked external clients)
- Utility functions
- Request/response validation

**Approach**:
- Mock interfaces, not implementations
- Test business logic in isolation
- Verify error handling paths

### 9.2 Integration Test Boundaries

**What is integration tested**:
- API endpoints with TestClient
- Provider implementations with real services (Docker-based)
- Database operations with test database

**Approach**:
- Use Docker Compose test environment
- Isolated test data per run
- Verify end-to-end data flow

### 9.3 What Is NOT Tested at Architecture Level

- UI/UX (no UI in this project)
- Performance benchmarks (deferred to future)
- Security penetration testing (not in scope)
- Multi-user scenarios (single-user system)

### 9.4 Test Environment Requirements

| Dependency | Test Approach |
|------------|---------------|
| MongoDB | Docker container with ephemeral data |
| Qdrant | Docker container with ephemeral data |
| Ollama | Docker container with test model |
| Filesystem | Temporary directory per test |

---

## 10. Open Questions & Risks

### 10.1 Architectural Unknowns

| Question | Impact | Mitigation |
|----------|--------|------------|
| Optimal chunk size for different document types | Query quality | Start with 1024/200 default; can tune later |
| Memory usage under concurrent queries | Stability | Monitor during testing; set limits if needed |
| Qdrant collection size limits | Scalability | Acceptable for learning scope; document limits |

### 10.2 Decisions Deferred Intentionally

| Decision | Reason for Deferral | When to Revisit |
|----------|---------------------|-----------------|
| Background worker implementation | Manual CLI simpler for MVP | Future version if auto-processing needed |
| Query timeout limits | Need real-world usage data | If long-running queries become problematic |
| Document deletion API | Not required for MVP | Future version with document management |
| Retry logic for failed jobs | Manual retry sufficient | If failure rate is high |
| Logging framework selection | Can add later without architecture change | When debugging needs become clear |

### 10.3 Known Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Ollama cold start delays | Medium | First query slow | Document requirement to pre-load models |
| Large PDF memory consumption | Medium | Worker crash | 10MB file limit; document RAM requirements |
| Stuck PROCESSING jobs | Low | Manual cleanup | Document recovery procedure |
| Vector dimension mismatch | Low | Indexing failure | Validate embedding dimensions on startup |

---

## Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-12 | TL | Initial architecture document |
| 1.0.1 | 2026-01-12 | TL | Clarifications: CLAUDE.md reference softened; async vs manual processing clarified; job completion semantics defined as atomic (no partial success); embedding model default updated with independence from LLM |

---

*This document is the technical source of truth for system architecture. Changes require TL approval. Implementation details will belong in CLAUDE.md (once created); API specifications belong in API_SPEC.md.*
