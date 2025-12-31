# RAG API - Technical Specification

## Project Context
A learning project to build a production-ready RAG (Retrieval-Augmented Generation) API system that enables users to upload PDF documents and query them using natural language, with all processing happening locally.

## Architecture

### System Components
1. **FastAPI Application**: REST API server handling upload and query endpoints
2. **MongoDB**: Document database for job queue and metadata storage
3. **Qdrant**: Vector database for storing and retrieving document embeddings
4. **Ollama**: Local LLM inference server running Llama models
5. **LlamaIndex**: Orchestration framework for the RAG pipeline
6. **CLI Worker**: Command-line tool to process document jobs from queue

### Data Flow

**Upload Flow (Asynchronous):**
```
1. API Layer:
   PDF Upload → FastAPI → Save File → Create Job in MongoDB → Return job_id

2. Processing Layer (CLI Worker):
   CLI Command → Fetch Pending Jobs from MongoDB →
   LlamaIndex (Load PDF) → LlamaIndex (Text Chunking) →
   LlamaIndex (Embedding via Ollama) → LlamaIndex (Vector Storage to Qdrant) →
   Update Job Status in MongoDB
```

**Query Flow (Synchronous):**
```
User Question → FastAPI → LlamaIndex (Query Embedding via Ollama) →
LlamaIndex (Vector Search in Qdrant) → LlamaIndex (Context Retrieval) →
LlamaIndex (LLM Generation via Ollama) → Streaming Response
```

**Status Check Flow:**
```
GET /api/documents/{job_id}/status → MongoDB → Return job status
```

## Tech Stack Details

### Core Dependencies
- **FastAPI**: Modern, fast async web framework for building APIs
- **MongoDB**: Document database for job queue and document metadata
  - `motor`: Async MongoDB driver for Python
  - `pymongo`: Sync MongoDB driver (for CLI worker)
- **LlamaIndex** (v0.10+): RAG orchestration framework
  - `llama-index-llms-ollama`: Ollama LLM integration
  - `llama-index-embeddings-ollama`: Ollama embeddings integration
  - `llama-index-vector-stores-qdrant`: Qdrant vector store integration
- **Qdrant Client**: Python client for Qdrant vector database
- **Ollama**: Local LLM runtime for running Llama models
- **Typer**: CLI framework for building the worker command

### Supporting Libraries
- `python-multipart`: File upload handling in FastAPI
- `uvicorn`: ASGI server for running FastAPI
- `python-dotenv`: Environment variable management
- `aiofiles`: Async file I/O operations
- `pydantic-settings`: Configuration management
- `rich`: Beautiful terminal output for CLI worker

**Note:** PDF parsing is handled natively by LlamaIndex's `SimpleDirectoryReader`, so no separate PDF library is needed.

## File Organization

### Project Structure (SOLID Principles + Provider Pattern)

```
learn-rag/
├── app/
│   ├── __init__.py
│   ├── main.py                    # FastAPI app initialization
│   ├── config.py                  # Configuration management
│   │
│   # API Layer (Routes)
│   ├── api/
│   │   ├── __init__.py
│   │   ├── upload.py              # Upload endpoint (async - returns job_id)
│   │   ├── query.py               # Query endpoint with streaming
│   │   ├── status.py              # Job status check endpoint
│   │   └── health.py              # Health check endpoint
│   │
│   # Core Interfaces (Abstractions)
│   ├── interfaces/
│   │   ├── __init__.py
│   │   ├── vector_store.py        # Vector DB interface
│   │   ├── llm.py                 # LLM interface
│   │   └── embedding.py           # Embedding interface
│   │
│   # Provider Implementations
│   ├── providers/
│   │   ├── __init__.py
│   │   ├── vector_db/
│   │   │   ├── __init__.py
│   │   │   ├── qdrant.py          # Qdrant implementation
│   │   │   └── factory.py         # Vector DB factory
│   │   ├── llm/
│   │   │   ├── __init__.py
│   │   │   ├── ollama.py          # Ollama LLM implementation
│   │   │   └── factory.py         # LLM factory
│   │   └── embedding/
│   │       ├── __init__.py
│   │       ├── ollama.py          # Ollama embedding implementation
│   │       └── factory.py         # Embedding factory
│   │
│   # Business Logic (Services)
│   ├── services/
│   │   ├── __init__.py
│   │   ├── job_service.py         # Job queue management (MongoDB)
│   │   ├── document_service.py    # Document upload & indexing orchestration
│   │   ├── query_service.py       # Query processing orchestration
│   │   └── storage_service.py     # File storage operations
│   │
│   # Request/Response Models
│   ├── models/
│   │   ├── __init__.py
│   │   ├── job.py                 # Job domain model (MongoDB)
│   │   ├── document.py            # Document domain model
│   │   └── schemas.py             # Pydantic request/response schemas
│   │
│   # CLI Worker
│   ├── cli/
│   │   ├── __init__.py
│   │   └── worker.py              # CLI command to process jobs
│   │
│   # Utilities
│   └── utils/
│       ├── __init__.py
│       └── file_utils.py          # File handling utilities
│
├── data/
│   └── uploads/                   # PDF storage directory
│
├── tests/
│   ├── __init__.py
│   ├── test_api.py                # API endpoint tests
│   ├── test_services.py           # Service tests
│   └── test_providers.py          # Provider tests
│
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
├── .env.example
└── README.md
```

## SOLID Principles Applied

**1. Single Responsibility Principle (SRP)**
- Each module has one clear purpose
- `DocumentService`: Handles document upload logic only
- `QueryService`: Handles query processing only
- `QdrantProvider`: Handles Qdrant operations only

**2. Open/Closed Principle (OCP)**
- Open for extension, closed for modification
- Add new vector DB: Create new provider, update factory
- Add new LLM: Create new provider, update factory
- No changes needed in services or API layer

**3. Liskov Substitution Principle (LSP)**
- Providers can be swapped without breaking functionality
- Any `VectorStoreInterface` implementation works with services
- Swap Qdrant for Pinecone via config - zero code changes

**4. Interface Segregation Principle (ISP)**
- Small, focused interfaces
- `VectorStoreInterface`: Only vector DB operations
- `LLMInterface`: Only LLM text generation
- `EmbeddingInterface`: Only embedding generation

**5. Dependency Inversion Principle (DIP)**
- Services depend on interfaces, not concrete implementations
- `DocumentService` depends on `VectorStoreInterface`, not `QdrantProvider`
- Providers are injected via dependency injection (FastAPI's `Depends`)

## Component Breakdown

**`app/interfaces/vector_store.py`**
```python
from abc import ABC, abstractmethod
from typing import List

class VectorStoreInterface(ABC):
    """Abstract interface for vector databases"""

    @abstractmethod
    async def create_collection(self, name: str, dimension: int) -> None:
        """Create a new collection for storing vectors"""
        pass

    @abstractmethod
    async def upsert(self, collection: str, vectors: List, metadata: List) -> None:
        """Insert or update vectors with metadata"""
        pass

    @abstractmethod
    async def search(self, collection: str, query_vector: List[float], top_k: int) -> List:
        """Search for similar vectors"""
        pass
```

**`app/interfaces/llm.py`**
```python
from abc import ABC, abstractmethod
from typing import AsyncGenerator

class LLMInterface(ABC):
    """Abstract interface for LLM operations"""

    @abstractmethod
    async def generate(self, prompt: str) -> str:
        """Generate completion"""
        pass

    @abstractmethod
    async def stream_generate(self, prompt: str) -> AsyncGenerator[str, None]:
        """Stream completion tokens"""
        pass
```

### 2. MongoDB Job Model

**`app/models/job.py`**
```python
from enum import Enum
from datetime import datetime
from typing import Optional
from pydantic import BaseModel, Field

class JobStatus(str, Enum):
    """Job processing status"""
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"

class DocumentJob(BaseModel):
    """Document processing job stored in MongoDB"""
    job_id: str = Field(..., description="Unique job identifier (UUID)")
    filename: str = Field(..., description="Original PDF filename")
    file_path: str = Field(..., description="Path to uploaded file")
    status: JobStatus = Field(default=JobStatus.PENDING, description="Current job status")
    created_at: datetime = Field(default_factory=datetime.utcnow, description="Job creation timestamp")
    processed_at: Optional[datetime] = Field(None, description="Job completion timestamp")
    error_message: Optional[str] = Field(None, description="Error message if job failed")
    chunks_created: Optional[int] = Field(None, description="Number of chunks created")

    class Config:
        use_enum_values = True
```

**MongoDB Collection Schema:**
```javascript
// Collection: document_jobs
{
  "_id": ObjectId("..."),
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "filename": "example.pdf",
  "file_path": "/app/data/uploads/550e8400-e29b-41d4-a716-446655440000.pdf",
  "status": "pending",
  "created_at": ISODate("2024-01-15T10:30:00Z"),
  "processed_at": null,
  "error_message": null,
  "chunks_created": null
}

// Indexes
db.document_jobs.createIndex({ "job_id": 1 }, { unique: true })
db.document_jobs.createIndex({ "status": 1 })
db.document_jobs.createIndex({ "created_at": -1 })
```

### 3. Providers (Concrete Implementations)

**`app/providers/vector_db/qdrant.py`**
```python
from qdrant_client import QdrantClient
from app.interfaces.vector_store import VectorStoreInterface

class QdrantProvider(VectorStoreInterface):
    """Qdrant implementation of vector store"""

    def __init__(self, host: str, port: int):
        self.client = QdrantClient(host=host, port=port)

    async def create_collection(self, name: str, dimension: int) -> None:
        # Qdrant-specific implementation
        self.client.create_collection(name=name, vectors_config=...)

    async def upsert(self, collection: str, vectors: List, metadata: List) -> None:
        self.client.upsert(collection_name=collection, points=...)

    async def search(self, collection: str, query_vector: List[float], top_k: int) -> List:
        return self.client.search(collection_name=collection, query_vector=query_vector, limit=top_k)
```

**`app/providers/llm/ollama.py`**
```python
from llama_index.llms.ollama import Ollama
from app.interfaces.llm import LLMInterface

class OllamaProvider(LLMInterface):
    """Ollama implementation via LlamaIndex"""

    def __init__(self, base_url: str, model: str):
        self.llm = Ollama(base_url=base_url, model=model)

    async def generate(self, prompt: str) -> str:
        response = await self.llm.acomplete(prompt)
        return response.text

    async def stream_generate(self, prompt: str):
        async for chunk in self.llm.astream_complete(prompt):
            yield chunk.text
```

**`app/providers/vector_db/factory.py`**
```python
from app.config import Settings
from app.interfaces.vector_store import VectorStoreInterface
from .qdrant import QdrantProvider

def create_vector_store(settings: Settings) -> VectorStoreInterface:
    """Factory to create vector store based on config"""
    if settings.vector_db_provider == "qdrant":
        return QdrantProvider(
            host=settings.qdrant_host,
            port=settings.qdrant_port
        )
    # Add more providers here
    raise ValueError(f"Unknown provider: {settings.vector_db_provider}")
```

### 4. Services (Business Logic)

**`app/services/job_service.py`**
```python
from motor.motor_asyncio import AsyncIOMotorClient
from pymongo import MongoClient
from typing import Optional
from datetime import datetime
import uuid
from app.models.job import DocumentJob, JobStatus

class JobService:
    """Manages document processing jobs in MongoDB"""

    def __init__(self, mongo_client: AsyncIOMotorClient, db_name: str = "rag_db"):
        self.db = mongo_client[db_name]
        self.jobs = self.db["document_jobs"]

    async def create_job(self, filename: str, file_path: str) -> str:
        """Create a new pending job and return job_id"""
        job_id = str(uuid.uuid4())
        job = DocumentJob(
            job_id=job_id,
            filename=filename,
            file_path=file_path,
            status=JobStatus.PENDING
        )
        await self.jobs.insert_one(job.dict())
        return job_id

    async def get_job(self, job_id: str) -> Optional[DocumentJob]:
        """Retrieve job by job_id"""
        job_dict = await self.jobs.find_one({"job_id": job_id})
        if job_dict:
            return DocumentJob(**job_dict)
        return None

    async def update_job_status(
        self,
        job_id: str,
        status: JobStatus,
        error_message: Optional[str] = None,
        chunks_created: Optional[int] = None
    ):
        """Update job status and metadata"""
        update_data = {"status": status}
        if status in [JobStatus.COMPLETED, JobStatus.FAILED]:
            update_data["processed_at"] = datetime.utcnow()
        if error_message:
            update_data["error_message"] = error_message
        if chunks_created is not None:
            update_data["chunks_created"] = chunks_created

        await self.jobs.update_one(
            {"job_id": job_id},
            {"$set": update_data}
        )

    def get_pending_jobs_sync(self) -> list[DocumentJob]:
        """Get all pending jobs (sync version for CLI worker)"""
        # This method uses sync pymongo for CLI worker
        sync_client = MongoClient(self.db.client.HOST)
        jobs_collection = sync_client[self.db.name]["document_jobs"]
        pending = jobs_collection.find({"status": JobStatus.PENDING})
        return [DocumentJob(**job) for job in pending]
```

**`app/services/document_service.py`**
```python
from llama_index.core import SimpleDirectoryReader, VectorStoreIndex
from llama_index.core.node_parser import SentenceSplitter
from app.interfaces.vector_store import VectorStoreInterface
from app.interfaces.embedding import EmbeddingInterface
from app.services.job_service import JobService
from app.models.job import JobStatus

class DocumentService:
    """Handles document indexing (called by CLI worker)"""

    def __init__(
        self,
        vector_store: VectorStoreInterface,
        embedding: EmbeddingInterface,
        job_service: JobService,
        chunk_size: int = 1024,
        chunk_overlap: int = 200
    ):
        self.vector_store = vector_store
        self.embedding = embedding
        self.job_service = job_service
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap

    async def process_document(self, job_id: str, file_path: str):
        """Process a document and update job status"""
        try:
            # Update job to processing
            await self.job_service.update_job_status(job_id, JobStatus.PROCESSING)

            # 1. Load PDF using LlamaIndex
            documents = SimpleDirectoryReader(input_files=[file_path]).load_data()

            # 2. Chunk text
            parser = SentenceSplitter(
                chunk_size=self.chunk_size,
                chunk_overlap=self.chunk_overlap
            )
            nodes = parser.get_nodes_from_documents(documents)

            # 3. Generate embeddings and store in vector DB
            # LlamaIndex handles this automatically when creating index
            index = VectorStoreIndex.from_documents(
                documents,
                embed_model=self.embedding,
                vector_store=self.vector_store
            )

            # 4. Update job as completed
            await self.job_service.update_job_status(
                job_id,
                JobStatus.COMPLETED,
                chunks_created=len(nodes)
            )

        except Exception as e:
            # Update job as failed
            await self.job_service.update_job_status(
                job_id,
                JobStatus.FAILED,
                error_message=str(e)
            )
            raise
```

**`app/services/query_service.py`**
```python
from app.interfaces.vector_store import VectorStoreInterface
from app.interfaces.llm import LLMInterface
from app.interfaces.embedding import EmbeddingInterface

class QueryService:
    """Handles query processing"""

    def __init__(
        self,
        vector_store: VectorStoreInterface,
        llm: LLMInterface,
        embedding: EmbeddingInterface
    ):
        self.vector_store = vector_store
        self.llm = llm
        self.embedding = embedding

    async def query(self, question: str):
        # 1. Generate query embedding
        # 2. Search vector store
        # 3. Build context
        # 4. Stream LLM response
        async for chunk in self.llm.stream_generate(prompt):
            yield chunk
```

### 4. Configuration

**`app/config.py`**
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    """Application configuration"""

    # Provider selection
    vector_db_provider: str = "qdrant"
    llm_provider: str = "ollama"
    embedding_provider: str = "ollama"

    # MongoDB settings
    mongo_uri: str = "mongodb://localhost:27017"
    mongo_db_name: str = "rag_db"

    # Qdrant settings
    qdrant_host: str = "localhost"
    qdrant_port: int = 6333
    qdrant_collection_name: str = "documents"

    # Ollama settings
    ollama_base_url: str = "http://localhost:11434"
    ollama_model: str = "llama2"
    ollama_embedding_model: str = "llama2"

    # Upload settings
    max_upload_size_mb: int = 10
    upload_directory: str = "/app/data/uploads"

    # RAG settings
    chunk_size: int = 1024
    chunk_overlap: int = 200
    top_k_results: int = 5

    class Config:
        env_file = ".env"
```

### 5. CLI Worker

**`app/cli/worker.py`**
```python
import typer
from rich.console import Console
from rich.progress import Progress
from pymongo import MongoClient
from app.config import Settings
from app.services.job_service import JobService
from app.services.document_service import DocumentService
from app.providers.vector_db.factory import create_vector_store
from app.providers.embedding.factory import create_embedding

app = typer.Typer()
console = Console()

@app.command()
def process_jobs():
    """Process all pending document jobs from MongoDB queue"""
    console.print("[bold blue]Starting job processor...[/bold blue]")

    settings = Settings()

    # Initialize services (sync versions for CLI)
    mongo_client = MongoClient(settings.mongo_uri)
    job_service = JobService(mongo_client, settings.mongo_db_name)

    vector_store = create_vector_store(settings)
    embedding = create_embedding(settings)

    doc_service = DocumentService(
        vector_store=vector_store,
        embedding=embedding,
        job_service=job_service,
        chunk_size=settings.chunk_size,
        chunk_overlap=settings.chunk_overlap
    )

    # Fetch pending jobs
    pending_jobs = job_service.get_pending_jobs_sync()

    if not pending_jobs:
        console.print("[yellow]No pending jobs found[/yellow]")
        return

    console.print(f"[green]Found {len(pending_jobs)} pending jobs[/green]")

    # Process each job
    with Progress() as progress:
        task = progress.add_task("[cyan]Processing documents...", total=len(pending_jobs))

        for job in pending_jobs:
            console.print(f"\nProcessing: {job.filename} (Job ID: {job.job_id})")

            try:
                # Process document synchronously
                doc_service.process_document(job.job_id, job.file_path)
                console.print(f"[green]✓ Completed: {job.filename}[/green]")
            except Exception as e:
                console.print(f"[red]✗ Failed: {job.filename} - {str(e)}[/red]")

            progress.update(task, advance=1)

    console.print("\n[bold green]Job processing complete![/bold green]")

if __name__ == "__main__":
    app()
```

**Usage:**
```bash
# Run worker to process all pending jobs
python -m app.cli.worker process-jobs

# Or with Typer CLI
python app/cli/worker.py process-jobs
```

### 6. API Layer

**`app/api/upload.py`**
```python
from fastapi import APIRouter, UploadFile, Depends, HTTPException
from app.services.job_service import JobService
from app.services.storage_service import StorageService
from app.models.schemas import UploadResponse
import uuid

router = APIRouter()

@router.post("/upload", response_model=UploadResponse)
async def upload_document(
    file: UploadFile,
    job_service: JobService = Depends(),
    storage_service: StorageService = Depends()
):
    """Upload PDF and create processing job (async - returns immediately)"""
    # Validate file type
    if not file.filename.endswith('.pdf'):
        raise HTTPException(status_code=400, detail="Only PDF files allowed")

    # Generate unique job ID
    job_id = str(uuid.uuid4())

    # Save file to storage
    file_path = await storage_service.save_file(file, job_id)

    # Create job in MongoDB
    job_id = await job_service.create_job(
        filename=file.filename,
        file_path=file_path
    )

    return UploadResponse(
        status="success",
        job_id=job_id,
        filename=file.filename,
        message="Document uploaded successfully. Processing in queue."
    )
```

**`app/api/status.py`**
```python
from fastapi import APIRouter, Depends, HTTPException
from app.services.job_service import JobService
from app.models.schemas import JobStatusResponse

router = APIRouter()

@router.get("/documents/{job_id}/status", response_model=JobStatusResponse)
async def get_job_status(
    job_id: str,
    job_service: JobService = Depends()
):
    """Check the status of a document processing job"""
    job = await job_service.get_job(job_id)

    if not job:
        raise HTTPException(status_code=404, detail="Job not found")

    return JobStatusResponse(
        job_id=job.job_id,
        filename=job.filename,
        status=job.status,
        created_at=job.created_at,
        processed_at=job.processed_at,
        error_message=job.error_message,
        chunks_created=job.chunks_created
    )
```

**`app/api/query.py`**
```python
from fastapi import APIRouter, Depends
from fastapi.responses import StreamingResponse
from app.services.query_service import QueryService
from app.models.schemas import QueryRequest

router = APIRouter()

@router.post("/query")
async def query_documents(
    request: QueryRequest,
    query_service: QueryService = Depends()
):
    """Query indexed documents with streaming response"""
    async def generate():
        async for chunk in query_service.query(request.question):
            yield chunk

    return StreamingResponse(generate(), media_type="text/event-stream")
```

**`app/main.py`**
```python
from fastapi import FastAPI
from motor.motor_asyncio import AsyncIOMotorClient
from app.api import upload, query, status, health
from app.config import Settings

def create_app() -> FastAPI:
    app = FastAPI(title="RAG API")

    settings = Settings()

    # Initialize MongoDB connection on startup
    @app.on_event("startup")
    async def startup_db_client():
        app.mongodb_client = AsyncIOMotorClient(settings.mongo_uri)
        app.mongodb = app.mongodb_client[settings.mongo_db_name]

    @app.on_event("shutdown")
    async def shutdown_db_client():
        app.mongodb_client.close()

    # Include routers
    app.include_router(upload.router, prefix="/api", tags=["upload"])
    app.include_router(query.router, prefix="/api", tags=["query"])
    app.include_router(status.router, prefix="/api", tags=["status"])
    app.include_router(health.router, prefix="/api", tags=["health"])

    return app

app = create_app()
```

## Benefits of This Architecture

### 1. Easy Provider Switching
Change `.env` to switch providers:
```bash
# Switch from Qdrant to Pinecone
VECTOR_DB_PROVIDER=pinecone

# Switch from Ollama to OpenAI
LLM_PROVIDER=openai
```

No code changes needed - factory creates the right provider automatically!

### 2. Testable
```python
# Unit test with mocked dependencies
def test_document_service():
    mock_vector_store = Mock(spec=VectorStoreInterface)
    mock_embedding = Mock(spec=EmbeddingInterface)

    service = DocumentService(
        vector_store=mock_vector_store,
        embedding=mock_embedding
    )

    # Test business logic independently
```

### 3. Adding New Providers
To add Pinecone:

1. Create `app/providers/vector_db/pinecone.py`:
```python
class PineconeProvider(VectorStoreInterface):
    # Implement interface methods
    pass
```

2. Update factory:
```python
def create_vector_store(settings):
    if settings.vector_db_provider == "pinecone":
        return PineconeProvider(...)  # Add this
```

That's it! Services unchanged.

### 4. Clean Dependencies
```
API → Services → Interfaces ← Providers
```

- Services depend on **interfaces**, not implementations
- Providers implement interfaces
- Easy to mock, test, and swap

## Key Implementation Details

### Document Upload Flow (Asynchronous)
**API Layer:**
1. Receive PDF upload via FastAPI POST `/api/upload`
2. Validate file (PDF format, size limit)
3. Generate unique job_id (UUID)
4. Save file to local storage (`/app/data/uploads/{job_id}.pdf`)
5. Create job record in MongoDB with status `PENDING`
6. Return job_id immediately to client

**Processing Layer (CLI Worker):**
1. Run CLI command: `python -m app.cli.worker process-jobs`
2. Query MongoDB for all jobs with status `PENDING`
3. For each pending job:
   - Update status to `PROCESSING`
   - Load PDF using LlamaIndex `SimpleDirectoryReader`
   - Chunk text using `SentenceSplitter` (configurable size/overlap)
   - Generate embeddings via Ollama embedding provider
   - Store vectors in Qdrant vector store
   - Update status to `COMPLETED` with chunks_created count
   - On error: Update status to `FAILED` with error_message

### Status Check Flow
1. Client sends GET `/api/documents/{job_id}/status`
2. Query MongoDB for job by job_id
3. Return job status, timestamps, error message (if any), chunks count

### Query Processing Pipeline (Synchronous)
1. Receive question via POST `/api/query`
2. Generate query embedding via Ollama
3. Vector similarity search in Qdrant (retrieve top-k chunks)
4. Build context from retrieved chunks
5. Send context + question to Llama via Ollama
6. Stream LLM response tokens back to client in real-time

### Dependency Injection
Use FastAPI's `Depends()` for dependency injection:

```python
# In main.py or dependencies.py
def get_vector_store() -> VectorStoreInterface:
    settings = Settings()
    return create_vector_store(settings)

def get_document_service(
    vector_store: VectorStoreInterface = Depends(get_vector_store)
) -> DocumentService:
    return DocumentService(vector_store=vector_store, ...)
```

## Development Guidelines

### Code Style
- Follow PEP 8
- Use type hints everywhere
- Use async/await for I/O operations
- Keep functions small and focused

### Error Handling
- Use FastAPI's HTTPException for API errors
- Log errors with appropriate severity
- Return meaningful error messages to users

### Testing
- Unit tests: Test services with mocked interfaces
- Integration tests: Test providers with real dependencies
- E2E tests: Test API endpoints with TestClient

This architecture is simple, maintainable, and follows SOLID principles without unnecessary complexity!
