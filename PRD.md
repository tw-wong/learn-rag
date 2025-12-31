# RAG API - Product Requirements Document

## Product Vision
Build a self-contained RAG (Retrieval-Augmented Generation) API system that enables users to upload PDF documents and query them using natural language, with all processing happening locally without external API dependencies or cloud services.

## Goals
1. **Learn RAG fundamentals**: Understand the core concepts and implementation patterns of RAG systems
2. **Create reusable template**: Build a well-architected foundation that can be extended for production use cases
3. **Enable private document Q&A**: Provide local, secure document querying without data leaving the user's machine

## Target Users
- Developers learning about RAG and LLM applications
- Teams needing private, local document Q&A systems
- Researchers experimenting with RAG techniques
- Organizations with data privacy requirements

## User Stories

### US-1: Document Upload
**As a** developer/user
**I want to** upload PDF documents via a simple API endpoint
**So that** I can build a searchable knowledge base from my documents

**Acceptance Criteria:**
- Accept PDF files via HTTP POST request with multipart/form-data
- Validate file type is PDF (reject other formats)
- Validate file size is within limits (configurable, default 10MB)
- Store file securely on local filesystem
- Create job in MongoDB queue with status "pending"
- Return job_id immediately (asynchronous - does NOT process automatically)
- Documents are processed manually by running CLI worker command
- Handle errors gracefully with clear error messages

**Technical Notes:**
- Use FastAPI's `UploadFile` for file handling
- Generate unique job_id (UUID) to prevent collisions
- Upload API returns immediately - processing happens separately via CLI worker
- Processing is NOT automatic - requires manual trigger via CLI command

### US-2: Document Processing via CLI Worker
**As a** developer/operator
**I want to** manually trigger document processing via CLI command
**So that** I have control over when resource-intensive processing occurs

**Acceptance Criteria:**
- CLI command to process all pending jobs from MongoDB queue
- Fetch all jobs with status "pending" from MongoDB
- Process each job sequentially (extract text, chunk, embed, index to Qdrant)
- Update job status to "processing" during execution
- Update job status to "completed" with chunks count on success
- Update job status to "failed" with error message on failure
- Display progress and status in terminal with rich formatting

**Technical Notes:**
- Use Typer for CLI framework
- Use Rich for beautiful terminal output
- Command: `python -m app.cli.worker process-jobs`
- Worker can be triggered manually or via cron/scheduler

### US-3: Job Status Checking
**As a** user
**I want to** check the processing status of my uploaded document
**So that** I know when it's ready to query

**Acceptance Criteria:**
- Accept GET request with job_id
- Return current job status (pending, processing, completed, failed)
- Return timestamps (created_at, processed_at)
- Return error message if job failed
- Return chunks_created count if job completed
- Handle invalid job_id gracefully (404 error)

**Technical Notes:**
- Endpoint: GET `/api/documents/{job_id}/status`
- Query MongoDB for job metadata
- No polling implementation in v1 (user must manually check)

### US-4: Question Answering with Streaming
**As a** user
**I want to** ask questions about uploaded documents and receive streaming responses
**So that** I can quickly retrieve relevant information with a responsive experience

**Acceptance Criteria:**
- Accept questions via JSON POST request
- Search indexed documents for relevant context
- Generate natural language answers using local LLM
- Stream response tokens in real-time (not buffered)
- Include source attribution (which documents/chunks were used)
- Handle edge cases (no documents indexed, irrelevant questions)

**Technical Notes:**
- Use Server-Sent Events (SSE) or streaming response
- Retrieve top 3-5 most relevant chunks
- Include metadata in final response

## Functional Requirements

### FR-1: Upload API Endpoint
**Endpoint:** `POST /api/upload`

**Request:**
- **Method**: POST
- **Content-Type**: multipart/form-data
- **Body**:
  - `file`: PDF document (binary)

**Response (Success - 200):**
```json
{
  "status": "success",
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "filename": "example-document.pdf",
  "message": "Document uploaded successfully. Processing in queue."
}
```

**Response (Error - 400):**
```json
{
  "status": "error",
  "error": "Invalid file type. Only PDF files are supported.",
  "detail": "Received file type: application/msword"
}
```

**Response (Error - 500):**
```json
{
  "status": "error",
  "error": "Failed to process document",
  "detail": "PDF parsing error: Document is encrypted"
}
```

**Validations:**
- File must have `.pdf` extension
- File MIME type must be `application/pdf`
- File size must be ≤ 10MB (configurable via environment variable)
- File must be readable (not corrupted)

**Business Rules:**
- Each upload creates a new job entry in MongoDB (no deduplication in v1)
- Job ID is generated server-side (UUID v4)
- Original filename is preserved in job metadata
- File is saved immediately, but processing happens separately
- Upload returns immediately - does NOT wait for processing to complete
- Processing must be triggered manually via CLI worker command

### FR-2: Job Status API Endpoint
**Endpoint:** `GET /api/documents/{job_id}/status`

**Request:**
- **Method**: GET
- **Path Parameter**: `job_id` (UUID string)

**Response (Success - 200):**
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "filename": "example-document.pdf",
  "status": "completed",
  "created_at": "2024-01-15T10:30:00Z",
  "processed_at": "2024-01-15T10:30:45Z",
  "error_message": null,
  "chunks_created": 42
}
```

**Response (Processing - 200):**
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "filename": "example-document.pdf",
  "status": "processing",
  "created_at": "2024-01-15T10:30:00Z",
  "processed_at": null,
  "error_message": null,
  "chunks_created": null
}
```

**Response (Failed - 200):**
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "filename": "example-document.pdf",
  "status": "failed",
  "created_at": "2024-01-15T10:30:00Z",
  "processed_at": "2024-01-15T10:30:30Z",
  "error_message": "PDF parsing error: Document is encrypted",
  "chunks_created": null
}
```

**Response (Error - 404):**
```json
{
  "status": "error",
  "error": "Job not found",
  "detail": "No job exists with the provided job_id"
}
```

**Validations:**
- job_id must be valid UUID format
- job_id must exist in database

**Business Rules:**
- Job status values: "pending", "processing", "completed", "failed"
- processed_at is null until job finishes (completed or failed)
- error_message is only present when status is "failed"
- chunks_created is only present when status is "completed"

### FR-3: CLI Worker Command
**Command:** `python -m app.cli.worker process-jobs`

**Behavior:**
1. Connect to MongoDB and query for all jobs with status "pending"
2. If no pending jobs, display message and exit
3. For each pending job:
   - Update status to "processing"
   - Load PDF from file_path
   - Extract text using LlamaIndex SimpleDirectoryReader
   - Chunk text using SentenceSplitter
   - Generate embeddings using Ollama
   - Store vectors in Qdrant
   - Update status to "completed" with chunks_created count
   - On error: Update status to "failed" with error_message
4. Display progress in terminal with rich formatting
5. Display summary when complete

**Output Example:**
```
Starting job processor...
Found 3 pending jobs

Processing: example1.pdf (Job ID: 550e8400-...)
✓ Completed: example1.pdf

Processing: example2.pdf (Job ID: 660f9511-...)
✓ Completed: example2.pdf

Processing: example3.pdf (Job ID: 770g0622-...)
✗ Failed: example3.pdf - PDF parsing error

Job processing complete!
```

**Business Rules:**
- Worker processes jobs sequentially (not parallel in v1)
- Worker can be run manually or triggered via cron/scheduler
- Worker must be idempotent (safe to run multiple times)
- Failed jobs remain in "failed" status (no automatic retry)

### FR-4: Query API Endpoint
**Endpoint:** `POST /api/query`

**Request:**
- **Method**: POST
- **Content-Type**: application/json
- **Body**:
```json
{
  "question": "What is the main topic of the uploaded documents?"
}
```

**Response (Success - 200):**
- **Content-Type**: text/event-stream
- **Body**: Streaming text chunks

Example stream:
```
The main topic of the uploaded documents is machine learning...
```

**Response (Error - 400):**
```json
{
  "status": "error",
  "error": "Invalid question",
  "detail": "Question cannot be empty"
}
```

**Response (Error - 404):**
```json
{
  "status": "error",
  "error": "No documents indexed",
  "detail": "Please upload documents before querying"
}
```

**Validations:**
- Question must be non-empty string
- Question must be ≤ 500 characters
- Question must contain at least one alphanumeric character

**Business Rules:**
- Query searches across all indexed documents
- Returns generated answer based on top-k retrieved chunks (k=3-5)
- If no relevant context found, LLM should indicate uncertainty
- Streaming response improves perceived performance

### FR-5: Health Check Endpoint
**Endpoint:** `GET /health`

**Response (Success - 200):**
```json
{
  "status": "healthy",
  "services": {
    "mongodb": "connected",
    "qdrant": "connected",
    "ollama": "connected"
  },
  "version": "1.0.0"
}
```

**Response (Degraded - 503):**
```json
{
  "status": "degraded",
  "services": {
    "mongodb": "connected",
    "qdrant": "connected",
    "ollama": "unavailable"
  },
  "version": "1.0.0"
}
```

**Purpose:**
- Allow monitoring systems to check API health
- Verify connectivity to dependencies (MongoDB, Qdrant, Ollama)
- Support container orchestration health checks

## Non-Functional Requirements

### NFR-1: Performance
- **Upload Processing**: Complete within 30 seconds for typical PDF (< 50 pages, < 5MB)
- **Query Latency**: First response token within 2 seconds
- **Concurrent Requests**: Support at least 5 simultaneous requests without degradation
- **Throughput**: Handle 100+ queries per minute on standard hardware

**Target Hardware:**
- 8GB RAM minimum
- 4 CPU cores
- 10GB free disk space

### NFR-2: Reliability
- **Uptime**: 99%+ availability during development/testing
- **Error Handling**: Graceful degradation when services are unavailable
- **Data Persistence**: Vector data persists across container restarts
- **PDF Handling**: Robust parsing of malformed or complex PDFs
- **Service Recovery**: Automatic reconnection to Qdrant/Ollama after failures

### NFR-3: Usability
- **Clear Error Messages**: Human-readable errors with actionable guidance
- **API Documentation**: Comprehensive README with examples
- **Easy Setup**: Single command deployment (`docker-compose up`)
- **Logs**: Structured logging for debugging
- **Examples**: Sample curl commands for all endpoints

### NFR-4: Scalability
- **Modular Architecture**: Services can be scaled independently
- **Configurable Parameters**: Chunk size, overlap, model selection via config
- **Storage**: Support for growing document collections (100s of documents)
- **Extension Points**: Easy to add new endpoints or features

### NFR-5: Security
- **Input Validation**: All inputs validated and sanitized
- **File Security**: Uploaded files stored with safe naming conventions
- **No Secrets in Code**: All sensitive config via environment variables
- **Container Security**: Run containers with minimal privileges
- **CORS Configuration**: Configurable CORS for API security

### NFR-6: Maintainability
- **Code Quality**: Follow PEP 8, use type hints
- **Testing**: Unit and integration tests for core functionality
- **Documentation**: Inline comments for complex logic
- **Dependency Management**: Pinned versions in requirements.txt
- **Version Control**: Git-friendly structure, clear commit messages

## Technical Constraints
- **Local Deployment**: All processing must run locally without external API calls
- **Open Source**: Use only open-source tools and libraries
- **Docker**: Must run in Docker containers for portability
- **Platform Support**: macOS and Linux (Windows via WSL)
- **Python**: Python 3.11 or higher
- **No GPU Required**: Should work on CPU-only systems (GPU optional for performance)

## API Specifications

### Base URL
- **Development**: `http://localhost:8000`
- **Production**: Configurable via environment variable

### API Versioning
- **Version**: v1 (implicit in initial release)
- **Future**: Use `/api/v1/` prefix when v2 is introduced

### Endpoints Summary
| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| POST | `/api/upload` | Upload PDF and create job (async) | No |
| GET | `/api/documents/{job_id}/status` | Check job processing status | No |
| POST | `/api/query` | Query indexed documents with streaming | No |
| GET | `/health` | Health check for monitoring | No |

### CLI Commands Summary
| Command | Description |
|---------|-------------|
| `python -m app.cli.worker process-jobs` | Process all pending document jobs from MongoDB queue |

### Content Types
- **Request**: `application/json`, `multipart/form-data`
- **Response**: `application/json`, `text/event-stream`

### Authentication
- **Version 1**: None (local use only)
- **Future**: API key or JWT tokens for production deployments

### Rate Limiting
- **Version 1**: None
- **Future**: Configurable rate limits per endpoint

### CORS
- **Development**: Allow all origins (`*`)
- **Production**: Configurable allowed origins

## Success Metrics

### Functional Success
- ✅ Users can successfully upload PDF documents via API
- ✅ Upload returns job_id immediately (async)
- ✅ Jobs are created in MongoDB with "pending" status
- ✅ CLI worker successfully processes pending jobs
- ✅ Documents are correctly processed, chunked, and indexed in Qdrant
- ✅ Job status endpoint returns accurate status information
- ✅ Users can query processed documents and receive relevant answers
- ✅ Streaming responses work without buffering entire response
- ✅ Error cases are handled gracefully with clear messages

### Technical Success
- ✅ System runs reliably in Docker environment
- ✅ All four services (API, MongoDB, Qdrant, Ollama) start correctly
- ✅ MongoDB job queue works correctly
- ✅ CLI worker can be triggered manually
- ✅ Vector search returns relevant document chunks
- ✅ LLM generates coherent answers based on context
- ✅ Setup process works for new users following README

### User Experience Success
- ✅ Setup takes < 10 minutes for new users
- ✅ Upload process is intuitive and returns immediately
- ✅ CLI worker provides clear progress feedback
- ✅ Status endpoint allows tracking job progress
- ✅ Query responses feel responsive (streaming)
- ✅ Error messages help users resolve issues
- ✅ Documentation is clear and complete

## Out of Scope (Future Enhancements)

The following features are explicitly **not included** in version 1.0 but may be added in future iterations:

### Document Management
- List all uploaded documents
- Delete specific documents
- Update/replace existing documents
- Document metadata search (filter by filename, date)
- Batch document upload

### Advanced RAG Features
- Multiple document collections/namespaces
- Hierarchical document structures
- Semantic chunking strategies
- Query result reranking
- Hybrid search (vector + keyword)
- Metadata filtering in queries
- Citation/source highlighting

### User Features
- User authentication and authorization
- Multi-user support with document isolation
- Query history and analytics
- Saved queries and bookmarks
- Custom system prompts

### Operational Features
- Monitoring dashboard
- Prometheus metrics export
- Distributed tracing
- Automated backups
- High availability configuration
- Horizontal scaling

### Format Support
- DOCX, TXT, HTML, Markdown files
- Image OCR for scanned PDFs
- Table extraction and understanding
- URL/web page ingestion

### AI Capabilities
- Fine-tuned embedding models
- Custom LLM prompts per collection
- Multi-language support
- Conversation context (chat history)
- Follow-up questions

## Dependencies

### System Requirements
- **Docker**: Version 20.10 or higher
- **Docker Compose**: Version 2.0 or higher
- **Operating System**: macOS, Linux, or Windows with WSL2
- **Memory**: Minimum 8GB RAM (16GB recommended)
- **Storage**: 10GB free disk space (for Ollama models)
- **Network**: Internet connection for initial setup (pulling images and models)

### External Services (Self-Hosted)
- **MongoDB**: Document database for job queue
- **Qdrant**: Vector database for embeddings
- **Ollama**: Local LLM runtime with Llama model

### Python Dependencies
See `requirements.txt` for complete list. Key dependencies:
- fastapi >= 0.104.0
- uvicorn >= 0.24.0
- motor >= 3.3.0 (async MongoDB driver)
- pymongo >= 4.6.0 (sync MongoDB driver for CLI)
- llama-index >= 0.10.0
- llama-index-llms-ollama
- llama-index-embeddings-ollama
- llama-index-vector-stores-qdrant
- qdrant-client >= 1.7.0
- typer >= 0.9.0 (CLI framework)
- rich >= 13.0.0 (terminal formatting)

## Deployment Requirements

### Development Environment
- Clone repository
- Copy `.env.example` to `.env`
- Run `docker-compose up --build`
- Wait for Ollama to download Llama model (first run only)
- Access API at `http://localhost:8000`

### Environment Variables
```bash
# MongoDB Configuration
MONGO_URI=mongodb://mongodb:27017
MONGO_DB_NAME=rag_db

# Qdrant Configuration
QDRANT_HOST=qdrant
QDRANT_PORT=6333
QDRANT_COLLECTION_NAME=documents

# Ollama Configuration
OLLAMA_BASE_URL=http://ollama:11434
OLLAMA_MODEL=llama2
OLLAMA_EMBEDDING_MODEL=llama2

# Upload Configuration
MAX_UPLOAD_SIZE_MB=10
UPLOAD_DIR=/app/data/uploads

# RAG Configuration
CHUNK_SIZE=1024
CHUNK_OVERLAP=200
TOP_K_RESULTS=5
```

## Timeline & Milestones

This is a **learning project** with iterative development. No strict deadlines, but suggested milestones:

1. **Milestone 1: Foundation** (Week 1)
   - Project structure and documentation
   - Docker setup with all services (MongoDB, Qdrant, Ollama)
   - Basic FastAPI app with health check

2. **Milestone 2: Async Upload Pipeline** (Week 2)
   - Upload endpoint implementation (async - returns job_id)
   - File storage service
   - MongoDB job queue integration
   - Job status endpoint

3. **Milestone 3: CLI Worker** (Week 3)
   - CLI worker command implementation
   - PDF processing and chunking via LlamaIndex
   - Vector indexing in Qdrant
   - Job status updates (pending → processing → completed/failed)

4. **Milestone 4: Query Pipeline** (Week 4)
   - Query endpoint implementation
   - Vector search integration with Qdrant
   - LLM response generation via Ollama
   - Implement streaming responses

5. **Milestone 5: Testing & Documentation** (Week 5)
   - Error handling and validation
   - Write tests (API endpoints, services, CLI worker)
   - Complete README with examples
   - Create usage guide with CLI commands

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Ollama model too large for local machine | High | Provide smaller model options (e.g., llama2:7b vs 13b) |
| MongoDB connection issues | High | Implement retry logic, health checks, clear error messages |
| PDF parsing failures on complex documents | Medium | Robust error handling in CLI worker, update job status to "failed" with clear error message |
| Slow document processing | Medium | Process jobs asynchronously via CLI worker, allow batching |
| Slow query responses | Medium | Optimize chunk retrieval, use smaller models, cache embeddings |
| Qdrant connection issues | Medium | Implement retry logic, health checks, clear error messages |
| Memory exhaustion with large documents | Medium | Implement file size limits, streaming processing in CLI worker |
| Jobs stuck in "processing" state | Low | CLI worker is idempotent, safe to re-run; implement timeout handling in future |

## Appendix

### Glossary
- **RAG**: Retrieval-Augmented Generation - technique to enhance LLM responses with retrieved context
- **Vector Database**: Database optimized for storing and searching high-dimensional vectors (embeddings)
- **Embedding**: Numerical representation of text that captures semantic meaning
- **Chunking**: Splitting documents into smaller pieces for processing
- **Ollama**: Tool for running large language models locally
- **LlamaIndex**: Framework for building LLM applications with data

### References
- [LlamaIndex Documentation](https://docs.llamaindex.ai/)
- [Qdrant Documentation](https://qdrant.tech/documentation/)
- [Ollama Documentation](https://ollama.ai/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)

### Change Log
- **v1.0** (Initial):
  - Asynchronous upload API (returns job_id immediately)
  - MongoDB job queue for document processing
  - CLI worker for manual job processing
  - Job status endpoint
  - Query API with streaming responses
  - Local processing with Qdrant, LlamaIndex, and Ollama
