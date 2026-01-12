# RAG API - Local Document Q&A System

## Overview
A self-contained RAG (Retrieval-Augmented Generation) API system that enables users to upload PDF documents and query them using natural language, with all processing happening locally without external API dependencies or cloud services.

**Problem Statement:** Developers and organizations need a way to interact with their documents using natural language questions, but existing solutions either require expensive cloud API subscriptions, send sensitive data to external services, or lack proper architecture for learning and extension.

## Goals
- Create a working RAG system that demonstrates core concepts (document upload, embedding, retrieval, generation)
- Provide a reusable, well-architected foundation following SOLID principles
- Enable private document Q&A without data leaving the local machine
- Deliver a learning-friendly codebase with clear separation of concerns

**Success Metrics:**
- Users can upload PDFs and receive queryable results
- Query response time < 2 seconds for first token (assumes warm model, no cold start)
- System handles 100+ queries per minute on standard hardware (8GB RAM, 4 CPU cores)
- Setup process takes < 10 minutes from clone to running system

## Non-Goals
- Production-grade authentication or authorization
- Multi-user support with document isolation
- Cloud deployment or SaaS offering
- Support for non-PDF document formats (DOCX, HTML, etc.)
- Automatic document deletion or lifecycle management
- Real-time collaboration features
- Advanced RAG features (reranking, hybrid search, metadata filtering)
- Monitoring dashboard or analytics

## User Stories

### US-1: Document Upload (Async)
**As a** developer/user
**I want to** upload PDF documents via a simple API endpoint
**So that** I can build a searchable knowledge base from my documents

**Acceptance Criteria:**
- Given a valid PDF file under 10MB
- When I POST to the upload endpoint
- Then the system returns a job_id immediately without waiting for processing
- And the file is saved to local storage
- And a job is created in the queue with "pending" status
- And I receive a success message indicating processing will happen separately

- Given an invalid file (non-PDF or over size limit)
- When I attempt to upload
- Then I receive a clear error message explaining the rejection reason

- Given a corrupted or encrypted PDF
- When I upload the file
- Then the upload succeeds but processing will fail later (reported via status endpoint)

### US-2: Manual Document Processing
**As a** developer/operator
**I want to** manually trigger document processing via CLI command with control over batch size
**So that** I have control over when and how many resource-intensive jobs are processed

**Acceptance Criteria:**
- Given pending jobs exist in the queue
- When I run the CLI worker command with a batch size argument (e.g., process 5 jobs)
- Then the system fetches up to the specified number of pending jobs from the queue
- And processes each job sequentially (extract text, chunk, embed, index)
- And updates job status to "processing" during execution
- And updates job status to "completed" with chunk count on success
- And updates job status to "failed" with error message on failure
- And displays progress in the terminal with clear formatting
- And stops after processing the specified number of jobs (or fewer if queue is exhausted)

- Given no pending jobs exist
- When I run the CLI worker command
- Then the system displays a message indicating no jobs to process
- And exits cleanly

- Given a job fails during processing
- When the worker encounters the error
- Then the system marks the job as "failed" with error details
- And continues processing remaining jobs in the batch without stopping

- Given the batch size argument is not provided
- When I run the CLI worker command
- Then the system uses a default batch size (e.g., 10 jobs) or processes all pending jobs

### US-3: Job Status Checking
**As a** user
**I want to** check the processing status of my uploaded document
**So that** I know when it's ready to query

**Acceptance Criteria:**
- Given a valid job_id
- When I GET the status endpoint with that job_id
- Then I receive the current status (pending, processing, completed, failed)
- And I receive timestamps showing when the job was created and processed
- And I receive the number of chunks created (if completed)
- And I receive an error message (if failed)

- Given an invalid or non-existent job_id
- When I request the status
- Then I receive a 404 error with a clear message

### US-4: Document Querying with Streaming
**As a** user
**I want to** ask questions about uploaded documents and receive streaming responses
**So that** I can quickly retrieve relevant information with a responsive experience

**Acceptance Criteria:**
- Given at least one document has been successfully indexed
- When I POST a question to the query endpoint
- Then the system searches all indexed documents for relevant context
- And generates a natural language answer using the local LLM
- And streams the response tokens in real-time (no buffering, using SSE or similar streaming protocol)
- And the first token arrives within 2 seconds

- Given no documents have been indexed
- When I attempt to query
- Then I receive an error indicating no documents are available

- Given an empty or invalid question
- When I attempt to query
- Then I receive a validation error with clear guidance

- Given the question is unrelated to indexed documents
- When the system processes the query
- Then the LLM response should indicate uncertainty or lack of relevant context

## MVP vs Future Iterations

### Must Have (v1.0)
- PDF file upload with size validation
- Asynchronous job queue (database choice to be decided by tech lead)
- Manual CLI worker for document processing (no background worker in v1)
- Job status checking endpoint
- Query endpoint with streaming responses
- Single collection for all documents (unified knowledge base)
- Basic error handling and validation
- Docker-based deployment with all services
- Health check endpoint
- Documentation and setup guide
- Automated unit tests and minimal integration tests

### Nice to Have (Future Versions)
- Document management (list, delete, update documents)
- Document isolation (query specific documents vs. all)
- Multiple document collections/namespaces
- Automatic job processing (background worker)
- Retry logic for failed jobs
- Advanced RAG features (reranking, hybrid search)
- Support for additional file formats (DOCX, TXT, Markdown)
- User authentication and multi-tenancy
- Monitoring dashboard and metrics
- Comprehensive end-to-end tests

## Dependencies & Constraints

### External Dependencies
- **MongoDB**: Default choice for job queue and metadata storage (self-hosted)
- **Qdrant**: Default choice for vector storage and similarity search (self-hosted)
- **Ollama**: Default choice for local LLM inference and embeddings (self-hosted)
- **LlamaIndex**: Default choice for RAG orchestration framework
- **Docker**: Required for deployment and orchestration

### Technical Constraints
- Must run entirely on local hardware (no external API calls)
- Must use only open-source tools and libraries
- Must work on CPU-only systems (GPU optional for performance)
- Python 3.11 or higher required
- Minimum 8GB RAM, 16GB recommended
- Minimum 10GB free disk space (for Ollama models)
- Platform support: macOS, Linux, Windows via WSL2

### Resource Constraints
- Processing is synchronous (one job at a time in CLI worker)
- File size limited to 10MB per upload
- Query limited to 500 characters
- This is a learning project with no strict timeline
- Development is iterative with focus on learning over speed

### Integration Requirements
- API must integrate with job queue database for job management
- CLI worker must access same job queue database as API
- RAG framework must integrate with LLM provider for text generation and embeddings
- RAG framework must integrate with vector database for storage and retrieval
- All services must run in Docker containers with proper networking

## Open Questions

### Resolved
✅ **Q: Should document processing happen automatically after upload or require manual trigger?**
A: Manual trigger via CLI worker to give operators control over resource usage (AD-2 in prior version)

✅ **Q: Should each document get its own vector collection or use a single shared collection?**
A: Single shared collection for simpler architecture and unified knowledge base (AD-1 in prior version)

✅ **Q: Should queries search specific documents or all indexed documents?**
A: All indexed documents without filtering to provide comprehensive answers (AD-2 in prior version)

✅ **Q: How should we handle jobs that get stuck in "processing" status?**
A: Worker skips processing jobs; manual database update required for stuck jobs (acceptable for v1)

✅ **Q: Should we delete uploaded PDF files after successful processing?**
A: Keep files indefinitely to allow reprocessing and debugging (storage is cheap)

✅ **Q: What happens if a document has some chunks that fail to process?**
A: Mark job as "completed" with partial results (maximize data availability)

### Open
❓ **Q: What specific error messages should we show for different failure scenarios?**
Priority: Medium - Needs definition for better UX

❓ **Q: Should we implement request timeout limits for long-running queries?**
Priority: Low - Can defer to future iterations

❓ **Q: What level of logging detail do we need for debugging?**
Priority: Medium - Important for learning/troubleshooting

## Change Log

### v1.0 (2026-01-08) - PRD Template Compliance Rewrite
- Restructured entire document to follow strict PRD template from product-manager.md
- Removed technical implementation details (API specs, JSON schemas, code snippets)
- Removed architecture decisions section (belongs in technical docs)
- Removed detailed endpoint specifications (belongs in API spec doc)
- Removed system component descriptions (belongs in technical architecture doc)
- Focused on user problems, acceptance criteria, and product requirements only
- Moved non-functional requirements to constraints and goals
- Simplified language to be product-focused rather than technically prescriptive
- Maintained all core product decisions in user stories and acceptance criteria

### v1.0 (2026-01-08) - Previous Version
- Added "Architecture Decisions" section documenting key design choices
- Clarified single Qdrant collection approach
- Clarified query behavior across all documents
- Clarified worker job processing behavior
- Clarified file lifecycle and partial success handling
