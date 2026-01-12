# RAG API - Implementation Tasks

This document tracks the implementation progress for the RAG API project based on the milestones defined in [PRD.md](PRD.md).

**Note:** This is a learning project with iterative development. Tasks are organized by milestone but can be completed flexibly.

---

## Milestone 1: Foundation (Week 1)

### 1.1 Documentation
- [x] Create `Claude.md` - Technical specification
- [x] Create `PRD.md` - Product Requirements Document
- [x] Create `README.md` - Project documentation with setup instructions
- [x] Create `TASKS.md` - This file

### 1.2 Project Structure
- [ ] Create directory structure (`app/`, `data/`, `tests/`)
- [ ] Create `.gitignore` for Python projects
- [ ] Create `requirements.txt` with all dependencies
- [ ] Create `.env.example` with environment variable template

### 1.3 Docker Setup
- [ ] Create `Dockerfile` for FastAPI application
- [ ] Create `docker-compose.yml` with all services:
  - [ ] MongoDB service
  - [ ] Qdrant service
  - [ ] Ollama service
  - [ ] FastAPI application service
- [ ] Configure volumes for data persistence
- [ ] Configure networking between services

### 1.4 Basic FastAPI App
- [ ] Create `app/__init__.py`
- [ ] Create `app/main.py` with FastAPI initialization
- [ ] Create `app/config.py` with Pydantic settings
- [ ] Create health check endpoint (`app/api/health.py`)
- [ ] Test Docker Compose setup (`docker-compose up`)
- [ ] Verify all services start correctly

**Milestone 1 Completion Criteria:**
- [ ] All four services (API, MongoDB, Qdrant, Ollama) start successfully
- [ ] Health check endpoint returns 200 OK
- [ ] Documentation is complete and accurate

---

## Milestone 2: Async Upload Pipeline (Week 2)

### 2.1 MongoDB Integration
- [ ] Create `app/models/job.py` with JobStatus enum and DocumentJob model
- [ ] Create `app/services/job_service.py` with JobService class
  - [ ] Implement `create_job()` method
  - [ ] Implement `get_job()` method
  - [ ] Implement `update_job_status()` method
  - [ ] Implement `get_pending_jobs_sync()` method for CLI worker
- [ ] Add MongoDB connection to `app/main.py` (startup/shutdown events)
- [ ] Create MongoDB indexes (job_id, status, created_at)

### 2.2 File Storage Service
- [ ] Create `app/services/storage_service.py`
- [ ] Implement file saving with unique filenames (UUID-based)
- [ ] Implement file validation (PDF type, size limit)
- [ ] Create `data/uploads/` directory
- [ ] Add file cleanup utilities in `app/utils/file_utils.py`

### 2.3 Upload Endpoint
- [ ] Create `app/models/schemas.py` with UploadResponse schema
- [ ] Create `app/api/upload.py` with upload endpoint
  - [ ] Accept PDF file via multipart/form-data
  - [ ] Validate file type (PDF only)
  - [ ] Validate file size (≤10MB)
  - [ ] Save file to local storage
  - [ ] Create job in MongoDB with status "pending"
  - [ ] Return job_id immediately
- [ ] Add upload router to `app/main.py`
- [ ] Test upload endpoint with curl/Postman

### 2.4 Job Status Endpoint
- [ ] Add JobStatusResponse schema to `app/models/schemas.py`
- [ ] Create `app/api/status.py` with status endpoint
  - [ ] Accept job_id as path parameter
  - [ ] Query MongoDB for job
  - [ ] Return job status, timestamps, error_message, chunks_created
  - [ ] Handle 404 for invalid job_id
- [ ] Add status router to `app/main.py`
- [ ] Test status endpoint with valid/invalid job_ids

**Milestone 2 Completion Criteria:**
- [ ] Upload endpoint returns job_id immediately
- [ ] Jobs are created in MongoDB with "pending" status
- [ ] Status endpoint returns accurate job information
- [ ] File validation works correctly

---

## Milestone 3: CLI Worker (Week 3)

### 3.1 Provider Interfaces
- [ ] Create `app/interfaces/vector_store.py` with VectorStoreInterface
- [ ] Create `app/interfaces/llm.py` with LLMInterface
- [ ] Create `app/interfaces/embedding.py` with EmbeddingInterface

### 3.2 Provider Implementations
- [ ] Create `app/providers/vector_db/qdrant.py` with QdrantProvider
- [ ] Create `app/providers/vector_db/factory.py` with create_vector_store()
- [ ] Create `app/providers/llm/ollama.py` with OllamaProvider
- [ ] Create `app/providers/llm/factory.py` with create_llm()
- [ ] Create `app/providers/embedding/ollama.py` with OllamaEmbeddingProvider
- [ ] Create `app/providers/embedding/factory.py` with create_embedding()

### 3.3 Document Processing Service
- [ ] Create `app/services/document_service.py` with DocumentService class
  - [ ] Implement `process_document()` method
  - [ ] Update job status to "processing"
  - [ ] Load PDF using LlamaIndex SimpleDirectoryReader
  - [ ] Chunk text using SentenceSplitter
  - [ ] Generate embeddings via Ollama
  - [ ] Store vectors in Qdrant
  - [ ] Update job status to "completed" with chunks_created
  - [ ] Handle errors and update status to "failed" with error_message

### 3.4 CLI Worker Implementation
- [ ] Create `app/cli/__init__.py`
- [ ] Create `app/cli/worker.py` with Typer CLI
  - [ ] Implement `process_jobs()` command
  - [ ] Connect to MongoDB
  - [ ] Fetch pending jobs
  - [ ] Process each job sequentially
  - [ ] Display progress with Rich
  - [ ] Handle errors gracefully
- [ ] Add Typer and Rich to requirements.txt
- [ ] Test CLI worker: `python -m app.cli.worker process-jobs`

**Milestone 3 Completion Criteria:**
- [ ] CLI worker processes pending jobs successfully
- [ ] Job status transitions: pending → processing → completed/failed
- [ ] Documents are chunked and indexed in Qdrant
- [ ] CLI displays clear progress and status

---

## Milestone 4: Query Pipeline (Week 4)

### 4.1 Query Service
- [ ] Create `app/services/query_service.py` with QueryService class
  - [ ] Implement `query()` method
  - [ ] Generate query embedding via Ollama
  - [ ] Search Qdrant for top-k similar chunks
  - [ ] Build context from retrieved chunks
  - [ ] Stream LLM response via Ollama

### 4.2 Query Endpoint
- [ ] Add QueryRequest and QueryResponse schemas to `app/models/schemas.py`
- [ ] Create `app/api/query.py` with query endpoint
  - [ ] Accept question via JSON POST
  - [ ] Validate question (non-empty, max 500 chars)
  - [ ] Call QueryService
  - [ ] Stream response using StreamingResponse
  - [ ] Handle errors (no documents, query failures)
- [ ] Add query router to `app/main.py`

### 4.3 Streaming Implementation
- [ ] Implement async generator for streaming responses
- [ ] Use FastAPI StreamingResponse with media_type="text/event-stream"
- [ ] Test streaming with curl: `curl -N http://localhost:8000/api/query`
- [ ] Verify tokens stream in real-time (not buffered)

### 4.4 Integration Testing
- [ ] End-to-end test: Upload PDF → Process via CLI → Query document
- [ ] Test with multiple documents
- [ ] Test query relevance with different questions
- [ ] Test error cases (no documents, invalid queries)

**Milestone 4 Completion Criteria:**
- [ ] Query endpoint accepts questions and returns streaming responses
- [ ] Vector search retrieves relevant chunks
- [ ] LLM generates coherent answers based on context
- [ ] Streaming works without buffering

---

## Milestone 5: Testing & Documentation (Week 5)

### 5.1 Error Handling & Validation
- [ ] Review all endpoints for proper error handling
- [ ] Add input validation with Pydantic
- [ ] Add meaningful error messages
- [ ] Test edge cases (corrupted PDFs, empty queries, etc.)
- [ ] Add logging with appropriate severity levels

### 5.2 Unit Tests
- [ ] Create `tests/__init__.py`
- [ ] Create `tests/test_services.py`
  - [ ] Test JobService (create, get, update)
  - [ ] Test DocumentService (process_document)
  - [ ] Test QueryService (query)
  - [ ] Mock interfaces for isolated testing
- [ ] Create `tests/test_providers.py`
  - [ ] Test QdrantProvider
  - [ ] Test OllamaProvider
  - [ ] Test with real dependencies (integration tests)

### 5.3 API Tests
- [ ] Create `tests/test_api.py`
  - [ ] Test upload endpoint (success, validation errors)
  - [ ] Test status endpoint (valid job, invalid job)
  - [ ] Test query endpoint (streaming, errors)
  - [ ] Test health endpoint
- [ ] Use FastAPI TestClient for API tests
- [ ] Add pytest and httpx to requirements.txt

### 5.4 CLI Worker Tests
- [ ] Create `tests/test_cli_worker.py`
- [ ] Test CLI worker with mocked MongoDB
- [ ] Test error handling in CLI worker
- [ ] Test idempotency (safe to run multiple times)

### 5.5 Documentation
- [ ] Complete `README.md` with:
  - [ ] Project description and features
  - [ ] Architecture diagram (optional)
  - [ ] Prerequisites (Docker, Docker Compose)
  - [ ] Setup instructions (clone, .env, docker-compose up)
  - [ ] Usage examples (upload, status, query, CLI worker)
  - [ ] API documentation (endpoints, request/response examples)
  - [ ] Troubleshooting guide
- [ ] Create usage guide with CLI commands
- [ ] Add inline code comments for complex logic
- [ ] Review and update `Claude.md` and `PRD.md` if needed

### 5.6 Final Testing
- [ ] Test complete workflow on fresh environment
- [ ] Verify setup instructions work for new users
- [ ] Test with different PDF types and sizes
- [ ] Performance testing (upload, query latency)
- [ ] Load testing (concurrent requests)

**Milestone 5 Completion Criteria:**
- [ ] All tests pass (unit, integration, E2E)
- [ ] Code coverage > 70%
- [ ] Documentation is complete and accurate
- [ ] Setup works on fresh environment

---

## Post-Completion (Optional)

### Code Quality
- [ ] Run linter (ruff/black) and fix issues
- [ ] Add type hints to all functions
- [ ] Code review and refactoring
- [ ] Security audit (dependency scanning)

### Performance Optimization
- [ ] Profile query performance
- [ ] Optimize chunk retrieval
- [ ] Cache embeddings if needed
- [ ] Benchmark with different models

### Future Enhancements (Out of Scope for v1.0)
- [ ] Document management endpoints (list, delete)
- [ ] Multiple document collections
- [ ] Authentication and authorization
- [ ] Automatic job processing (cron/scheduler)
- [ ] Job retry mechanism for failed jobs
- [ ] Timeout handling for stuck jobs
- [ ] Production deployment configuration
- [ ] Monitoring and observability

---

## Notes

- Mark items as complete by changing `[ ]` to `[x]`
- Update this file as implementation progresses
- Add sub-tasks if needed for complex items
- Reference this file in commits: "feat: implement upload endpoint (Task 2.3)"
