# PDF RAG (Retrieval-Augmented Generation)

A FastAPI-based application for processing PDF documents with AI-powered analysis. The system converts PDF files to images and uses Google's Gemini AI to extract insights and information from the documents.

## Features

- **PDF Upload & Processing**: Upload PDF files through a REST API
- **Async Processing**: Background job queue using Redis/Valkey and RQ (Redis Queue)
- **AI Analysis**: Integration with Google Gemini 2.0 Flash for document analysis
- **Image Conversion**: Automatic conversion of PDF pages to images using `pdf2image`
- **Status Tracking**: Real-time status updates stored in MongoDB
- **Docker Support**: Full containerized development environment

## Architecture

### Components

- **FastAPI Server**: REST API for file uploads and status queries
- **RQ Worker**: Background workers for processing PDF files
- **MongoDB**: Document storage for file metadata and processing results
- **Redis/Valkey**: Task queue backend
- **Qdrant**: Vector database (configured for future RAG features)
- **Neo4j**: Graph database (configured for future knowledge graph features)

### Tech Stack

- **Python 3.12**
- **FastAPI**: Web framework
- **MongoDB**: Database
- **Redis/Valkey**: Queue backend
- **RQ**: Python job queue library
- **Google Gemini API**: AI model for document analysis
- **pdf2image**: PDF to image conversion
- **Poppler**: PDF rendering utility

## Prerequisites

- Docker and Docker Compose
- Python 3.12+ (for local development)
- Google API Key for Gemini API

## Setup

### Using Docker Compose (Recommended)

1. Clone the repository:
   ```bash
   git clone <repository-url>
   cd PDF_RAG
   ```

2. Create a `.env` file in the root directory:
   ```env
   GOOGLE_API_KEY=your_google_api_key_here
   ```

3. Start the services:
   ```bash
   docker-compose -f .devcontainer/docker-compose.yaml up -d
   ```

4. Install dependencies in the app container:
   ```bash
   docker-compose -f .devcontainer/docker-compose.yaml exec app bash
   pip install -r requirements.txt
   ```

5. Start the FastAPI server:
   ```bash
   uvicorn app.server:app --host 0.0.0.0 --port 8000 --reload
   ```

6. In a separate terminal, start the RQ worker:
   ```bash
   docker-compose -f .devcontainer/docker-compose.yaml exec app bash
   rq worker --url redis://valkey
   ```

### Local Development

1. Install system dependencies:
   ```bash
   sudo apt-get update
   sudo apt-get install -y poppler-utils
   ```

2. Create and activate a virtual environment:
   ```bash
   python3 -m venv venv
   source venv/bin/activate
   ```

3. Install Python dependencies:
   ```bash
   pip install -r requirements.txt
   ```

4. Set up environment variables:
   ```bash
   export GOOGLE_API_KEY=your_google_api_key_here
   ```
   Or create a `.env` file with the key.

5. Ensure MongoDB, Redis/Valkey, and other services are running (use Docker Compose or install locally).

6. Start the FastAPI server:
   ```bash
   uvicorn app.server:app --host 0.0.0.0 --port 8000 --reload
   ```

7. In a separate terminal, start the RQ worker:
   ```bash
   rq worker --url redis://localhost:6379
   ```

## API Endpoints

### Health Check

```http
GET /
```

Returns server status.

**Response:**
```json
{
  "status": "Server is up and running"
}
```

### Upload PDF File

```http
POST /upload
Content-Type: multipart/form-data
```

Upload a PDF file for processing.

**Request:**
- Form data with `file` field containing the PDF file

**Response:**
```json
{
  "file_id": "507f1f77bcf86cd799439011"
}
```

### Get File Status

```http
GET /{id}
```

Retrieve the processing status and result of an uploaded file.

**Parameters:**
- `id` (path): The file ID returned from the upload endpoint

**Response:**
```json
{
  "_id": "507f1f77bcf86cd799439011",
  "name": "document.pdf",
  "status": "Processed",
  "result": "AI analysis result here..."
}
```

**Status Values:**
- `saving`: File is being saved to disk
- `queued`: File is in the processing queue
- `processing`: File is being processed
- `Converting to images`: PDF pages are being converted to images
- `Converting to images success`: Image conversion completed
- `Processed`: Processing completed successfully

## Processing Flow

1. **Upload**: PDF file is uploaded via `/upload` endpoint
2. **Save**: File is saved to `/mnt/uploads/{file_id}/{filename}`
3. **Queue**: Processing job is enqueued in Redis/Valkey
4. **Convert**: Worker converts PDF pages to images using `pdf2image`
5. **Analyze**: Images are sent to Google Gemini API for analysis
6. **Store**: Results are stored in MongoDB

## Project Structure

```
PDF_RAG/
├── app/
│   ├── db/
│   │   ├── client.py          # MongoDB connection
│   │   ├── db.py              # Database instance
│   │   └── collections/
│   │       └── files.py       # File collection schema
│   ├── queue/
│   │   ├── q.py               # RQ queue setup
│   │   └── workers.py         # Background worker functions
│   ├── utils/
│   │   └── file.py            # File utilities
│   ├── main.py                # Application entry point
│   └── server.py              # FastAPI application
├── .devcontainer/
│   ├── Dockerfile             # Development container
│   └── docker-compose.yaml    # Service definitions
├── requirements.txt           # Python dependencies
├── run.sh                     # Server startup script
└── README.md                  # This file
```

## Configuration

### Environment Variables

- `GOOGLE_API_KEY`: Your Google API key for Gemini API access

### Docker Services

The `docker-compose.yaml` defines the following services:

- **app**: Python development container
- **valkey**: Redis-compatible in-memory data store
- **vector-db**: Qdrant vector database (port 6333)
- **mongodb**: MongoDB database (port 27017)
  - Username: `admin`
  - Password: `admin`
- **neo4j**: Neo4j graph database (ports 7474, 7687)

## Usage Example

1. Start the server and worker (see Setup section)

2. Upload a PDF file:
   ```bash
   curl -X POST "http://localhost:8000/upload" \
     -H "Content-Type: multipart/form-data" \
     -F "file=@document.pdf"
   ```

   Response:
   ```json
   {
     "file_id": "507f1f77bcf86cd799439011"
   }
   ```

3. Check processing status:
   ```bash
   curl "http://localhost:8000/507f1f77bcf86cd799439011"
   ```

## Development

### Running Tests

(Add test instructions when tests are added)

### Code Style

The project uses standard Python code style. Consider adding:
- `flake8` for linting
- `black` for code formatting
- `mypy` for type checking

### Adding New Features

1. Database changes: Update schemas in `app/db/collections/`
2. New endpoints: Add routes in `app/server.py`
3. Background jobs: Add worker functions in `app/queue/workers.py`

## Troubleshooting

### Worker not processing jobs

- Ensure Redis/Valkey is running and accessible
- Check worker logs for errors
- Verify queue connection string matches Redis/Valkey host

### PDF conversion fails

- Ensure `poppler-utils` is installed
- Check file permissions on `/mnt/uploads/`
- Verify PDF file is not corrupted

### API key errors

- Verify `GOOGLE_API_KEY` is set correctly
- Check API key has Gemini API access enabled
- Ensure API key is not expired

## Future Enhancements

- [ ] Implement full RAG pipeline with vector embeddings
- [ ] Add chat interface for querying processed documents
- [ ] Integrate Neo4j for knowledge graph construction
- [ ] Support for multiple file formats
- [ ] Batch processing capabilities
- [ ] Web UI for file upload and results viewing
- [ ] Authentication and authorization
- [ ] Rate limiting and request throttling

## License

(Add license information)

## Contributing

(Add contributing guidelines)

