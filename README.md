# RAG API - Local Document Q&A System

A production-ready RAG (Retrieval-Augmented Generation) API that enables you to upload PDF documents and query them using natural language, with all processing happening locally without external API dependencies.

## Features

- **Asynchronous Document Upload**: Upload PDFs and get immediate job_id response
- **Manual Processing Control**: Trigger document processing via CLI worker when ready
- **Job Status Tracking**: Check processing status of uploaded documents
- **Streaming Query Responses**: Ask questions and receive real-time streaming answers
- **Local Processing**: All operations run locally using Ollama (no OpenAI API calls)
- **SOLID Architecture**: Clean, maintainable code following SOLID principles
- **Provider Pattern**: Easy to swap vector databases or LLM providers via configuration

## Architecture

### System Components

- **FastAPI Application**: REST API server
- **MongoDB**: Job queue and metadata storage
- **Qdrant**: Vector database for document embeddings
- **Ollama**: Local LLM runtime (Llama models)
- **LlamaIndex**: RAG orchestration framework
- **CLI Worker**: Command-line tool for processing documents

### Data Flow

```
Upload Flow (Async):
  PDF Upload → API → Save File → MongoDB (pending) → Return job_id

Processing Flow (Manual):
  CLI Worker → Fetch Pending Jobs → Process (chunk, embed, index) → Update Status

Query Flow (Streaming):
  Question → API → Vector Search → LLM Generation → Stream Response
```

## Prerequisites

- **Docker**: Version 20.10 or higher
- **Docker Compose**: Version 2.0 or higher
- **System Requirements**:
  - Minimum 8GB RAM (16GB recommended)
  - 10GB free disk space (for Ollama models)
  - macOS, Linux, or Windows with WSL2

## Quick Start

### 1. Clone the Repository

```bash
git clone <your-repo-url>
cd learn-rag
```

### 2. Configure Environment

```bash
cp .env.example .env
```

Edit `.env` if you want to customize settings (optional).

### 3. Start Services

```bash
docker-compose up --build
```

**First run:** Ollama will download the Llama model (~4GB). This may take several minutes.

### 4. Verify Services

```bash
# Check health endpoint
curl http://localhost:8000/health
```

Expected response:
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

## API Usage

### Upload a PDF Document

Upload a PDF and receive a job_id for tracking:

```bash
curl -X POST http://localhost:8000/api/upload \
  -F "file=@/path/to/document.pdf"
```

Response:
```json
{
  "status": "success",
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "filename": "document.pdf",
  "message": "Document uploaded successfully. Processing in queue."
}
```

### Process Documents (CLI Worker)

Documents are NOT processed automatically. Run the CLI worker to process pending jobs:

```bash
# Inside the Docker container
docker exec -it learn-rag-api python -m app.cli.worker process-jobs

# Or enter the container first
docker exec -it learn-rag-api bash
python -m app.cli.worker process-jobs
```

Expected output:
```
Starting job processor...
Found 1 pending jobs

Processing: document.pdf (Job ID: 550e8400-...)
✓ Completed: document.pdf

Job processing complete!
```

### Check Job Status

```bash
curl http://localhost:8000/api/documents/550e8400-e29b-41d4-a716-446655440000/status
```

Response (completed):
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "filename": "document.pdf",
  "status": "completed",
  "created_at": "2024-01-15T10:30:00Z",
  "processed_at": "2024-01-15T10:30:45Z",
  "error_message": null,
  "chunks_created": 42
}
```

Status values: `pending`, `processing`, `completed`, `failed`

### Query Documents

Ask questions about uploaded documents with streaming responses:

```bash
curl -N -X POST http://localhost:8000/api/query \
  -H "Content-Type: application/json" \
  -d '{"question": "What is the main topic of the document?"}'
```

The response streams in real-time (using Server-Sent Events).

## API Reference

### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/upload` | Upload PDF and create job (async) |
| `GET` | `/api/documents/{job_id}/status` | Check job processing status |
| `POST` | `/api/query` | Query indexed documents (streaming) |
| `GET` | `/health` | Health check for all services |

### Upload Endpoint

**POST** `/api/upload`

**Request:**
- Content-Type: `multipart/form-data`
- Field: `file` (PDF document, max 10MB)

**Response (200):**
```json
{
  "status": "success",
  "job_id": "uuid-string",
  "filename": "example.pdf",
  "message": "Document uploaded successfully. Processing in queue."
}
```

**Errors:**
- `400`: Invalid file type or file too large
- `500`: Server error

### Status Endpoint

**GET** `/api/documents/{job_id}/status`

**Response (200):**
```json
{
  "job_id": "uuid-string",
  "filename": "example.pdf",
  "status": "completed",
  "created_at": "2024-01-15T10:30:00Z",
  "processed_at": "2024-01-15T10:30:45Z",
  "error_message": null,
  "chunks_created": 42
}
```

**Errors:**
- `404`: Job not found

### Query Endpoint

**POST** `/api/query`

**Request:**
```json
{
  "question": "What is the main topic?"
}
```

**Response:**
- Content-Type: `text/event-stream`
- Streams text chunks in real-time

**Errors:**
- `400`: Invalid question (empty or too long)
- `404`: No documents indexed
- `500`: Query processing error

## CLI Worker Commands

### Process Pending Jobs

```bash
python -m app.cli.worker process-jobs
```

Processes all documents with status "pending" in MongoDB queue.

**Features:**
- Sequential processing (one job at a time)
- Progress bar with Rich formatting
- Error handling (failed jobs marked as "failed")
- Idempotent (safe to run multiple times)

## Configuration

### Environment Variables

Edit `.env` to customize:

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

### Switching Providers

The system uses a provider pattern. To switch vector databases or LLMs:

1. Create new provider class implementing the interface
2. Update factory in `app/providers/<type>/factory.py`
3. Change provider selection in `.env`

Example: Switch from Qdrant to Pinecone:
```bash
VECTOR_DB_PROVIDER=pinecone
```

## Project Structure

```
learn-rag/
├── app/
│   ├── api/              # API endpoints
│   ├── cli/              # CLI worker
│   ├── interfaces/       # Abstract interfaces
│   ├── providers/        # Provider implementations
│   ├── services/         # Business logic
│   ├── models/           # Data models
│   └── utils/            # Utilities
├── data/
│   └── uploads/          # PDF storage
├── tests/                # Test suite
├── docker-compose.yml    # Service orchestration
├── Dockerfile            # API container
├── requirements.txt      # Python dependencies
└── README.md             # This file
```

## Development

### Running Tests

```bash
# Run all tests
docker exec -it learn-rag-api pytest

# Run specific test file
docker exec -it learn-rag-api pytest tests/test_api.py

# Run with coverage
docker exec -it learn-rag-api pytest --cov=app tests/
```

### Viewing Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f api
docker-compose logs -f mongodb
docker-compose logs -f qdrant
docker-compose logs -f ollama
```

### Accessing Qdrant UI

Open http://localhost:6333/dashboard in your browser to view the Qdrant dashboard.

### Accessing MongoDB

```bash
# Connect to MongoDB shell
docker exec -it learn-rag-mongodb mongosh

# View jobs
use rag_db
db.document_jobs.find().pretty()
```

## Troubleshooting

### Ollama model not loading

**Issue:** First run takes long time

**Solution:** Ollama downloads the model on first run (~4GB). Be patient.

```bash
# Check Ollama logs
docker-compose logs ollama

# Manually pull model
docker exec -it learn-rag-ollama ollama pull llama2
```

### MongoDB connection failed

**Issue:** API can't connect to MongoDB

**Solution:** Ensure MongoDB is running:
```bash
docker-compose ps
docker-compose restart mongodb
```

### Qdrant collection not found

**Issue:** Query fails with "collection not found"

**Solution:** Process at least one document to create the collection:
```bash
# Upload a PDF first, then process it
docker exec -it learn-rag-api python -m app.cli.worker process-jobs
```

### Worker not processing jobs

**Issue:** CLI worker shows "No pending jobs found" but jobs exist

**Solution:** Check job status in MongoDB:
```bash
docker exec -it learn-rag-mongodb mongosh rag_db --eval "db.document_jobs.find({}, {job_id:1, filename:1, status:1})"
```

### Out of memory errors

**Issue:** Ollama crashes or system runs out of memory

**Solution:**
- Use a smaller model: `OLLAMA_MODEL=llama2:7b` in `.env`
- Increase Docker memory limit in Docker Desktop settings
- Reduce chunk size: `CHUNK_SIZE=512` in `.env`

## Performance Tips

1. **Use smaller models** for faster responses:
   ```bash
   OLLAMA_MODEL=llama2:7b  # Instead of llama2:13b
   ```

2. **Adjust chunk settings** for better retrieval:
   ```bash
   CHUNK_SIZE=512          # Smaller chunks = more precise
   CHUNK_OVERLAP=100       # Less overlap = faster indexing
   TOP_K_RESULTS=3         # Fewer results = faster queries
   ```

3. **Process documents in batches** during off-peak hours

4. **Monitor resource usage**:
   ```bash
   docker stats
   ```

## Contributing

This is a learning project. Contributions are welcome!

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is open source and available under the [MIT License](LICENSE).

## Acknowledgments

- [LlamaIndex](https://www.llamaindex.ai/) - RAG framework
- [Qdrant](https://qdrant.tech/) - Vector database
- [Ollama](https://ollama.ai/) - Local LLM runtime
- [FastAPI](https://fastapi.tiangolo.com/) - Web framework

## Learn More

- **Technical Specification**: See [Claude.md](Claude.md)
- **Product Requirements**: See [PRD.md](PRD.md)
- **Task List**: See [TASKS.md](TASKS.md)

## Support

For issues, questions, or contributions, please open an issue on GitHub.
