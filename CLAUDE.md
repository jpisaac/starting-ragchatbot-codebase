# Course Materials RAG System - Technical Documentation

## Overview

A sophisticated **Retrieval-Augmented Generation (RAG) chatbot** that enables intelligent querying of course materials using semantic search and AI-powered responses. The system combines ChromaDB vector storage, Anthropic's Claude AI, and a modern web interface to provide contextual answers about educational content.

## Architecture

### System Design
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Frontend      │◄──►│   FastAPI       │◄──►│   RAG System    │
│   (HTML/JS/CSS) │    │   Backend       │    │   Orchestrator  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                        │
                        ┌─────────────────┐    ┌─────────────────┐
                        │   Anthropic     │◄──►│   Vector Store  │
                        │   Claude API    │    │   (ChromaDB)    │
                        └─────────────────┘    └─────────────────┘
```

### Core Components

#### 1. **RAG System** (`rag_system.py`)
- **Purpose**: Main orchestrator coordinating all components
- **Key Functions**:
  - `query()` - Processes user queries with conversation context
  - `add_course_document()` - Ingests single course files
  - `add_course_folder()` - Batch processes document directories
- **Dependencies**: All other backend components

#### 2. **Document Processor** (`document_processor.py`)
- **Purpose**: Parses and chunks course documents
- **Key Features**:
  - Structured document parsing (course title, instructor, lessons)
  - Intelligent sentence-based chunking (800 chars with 100 char overlap)
  - Context enrichment for each chunk
- **Input Format**:
  ```
  Course Title: [title]
  Course Link: [url]
  Course Instructor: [instructor]
  
  Lesson 0: Introduction
  Lesson Link: [optional]
  [content...]
  ```

#### 3. **Vector Store** (`vector_store.py`)
- **Purpose**: Manages ChromaDB for semantic search
- **Collections**:
  - `course_content` - Text chunks with embeddings
  - `course_metadata` - Course and lesson information
- **Embedding Model**: `all-MiniLM-L6-v2` (384-dimensional vectors)
- **Search Capabilities**: Course filtering, lesson filtering, semantic similarity

#### 4. **AI Generator** (`ai_generator.py`)
- **Purpose**: Interfaces with Anthropic Claude API
- **Model**: `claude-sonnet-4-20250514`
- **Features**:
  - Tool-based search integration
  - Conversation context management
  - Temperature: 0 (deterministic responses)
  - Max tokens: 800

#### 5. **Search Tools** (`search_tools.py`)
- **Purpose**: Provides Claude with search capabilities
- **Tool**: `search_course_content`
- **Parameters**:
  - `query` (required) - Search term
  - `course_name` (optional) - Course filter
  - `lesson_number` (optional) - Lesson filter
- **Source Tracking**: Maintains sources for UI display

#### 6. **Session Manager** (`session_manager.py`)
- **Purpose**: Manages conversation history
- **Features**:
  - Session creation and management
  - Message history (max 5 exchanges)
  - Context formatting for AI

#### 7. **FastAPI Backend** (`app.py`)
- **Endpoints**:
  - `POST /api/query` - Process user queries
  - `GET /api/courses` - Get course statistics
  - `GET /` - Serve frontend
- **Middleware**: CORS, TrustedHost for proxy support

#### 8. **Frontend** (`frontend/`)
- **Files**:
  - `index.html` - Main interface with sidebar and chat
  - `script.js` - API interactions and UI management
  - `style.css` - Modern responsive styling
- **Features**:
  - Real-time chat interface
  - Loading animations
  - Collapsible sources display
  - Suggested questions
  - Course statistics sidebar

## Data Models

### Core Models (`models.py`)

```python
class Lesson(BaseModel):
    lesson_number: int
    title: str
    lesson_link: Optional[str] = None

class Course(BaseModel):
    title: str                    # Unique identifier
    course_link: Optional[str] = None
    instructor: Optional[str] = None
    lessons: List[Lesson] = []

class CourseChunk(BaseModel):
    content: str                  # Enriched text content
    course_title: str            # Course reference
    lesson_number: Optional[int] = None
    chunk_index: int             # Position in document
```

### API Models

```python
class QueryRequest(BaseModel):
    query: str
    session_id: Optional[str] = None

class QueryResponse(BaseModel):
    answer: str
    sources: List[str]
    session_id: str

class CourseStats(BaseModel):
    total_courses: int
    course_titles: List[str]
```

## Configuration

### Environment Variables (`.env`)
```bash
ANTHROPIC_API_KEY=your_api_key_here
```

### System Configuration (`config.py`)
```python
ANTHROPIC_MODEL = "claude-sonnet-4-20250514"
EMBEDDING_MODEL = "all-MiniLM-L6-v2"
CHUNK_SIZE = 800              # Characters per chunk
CHUNK_OVERLAP = 100           # Overlap between chunks
MAX_RESULTS = 5               # Search results returned
MAX_HISTORY = 2               # Conversation exchanges remembered
CHROMA_PATH = "./chroma_db"   # Vector database location
```

## Dependencies

### Backend (`pyproject.toml`)
```toml
[project]
requires-python = ">=3.13"
dependencies = [
    "chromadb==1.0.15",           # Vector database
    "anthropic==0.58.2",          # Claude API client
    "sentence-transformers==5.0.0", # Embeddings
    "fastapi==0.116.1",           # Web framework
    "uvicorn==0.35.0",            # ASGI server
    "python-multipart==0.0.20",  # Form handling
    "python-dotenv==1.1.1",      # Environment variables
]
```

### Frontend
- **Vanilla JavaScript** - No framework dependencies
- **Marked.js** - Markdown parsing for responses
- **Modern CSS** - Flexbox/Grid layouts, CSS variables

## Query Processing Flow

### 1. User Input → Frontend
```javascript
// User types query, clicks send
sendMessage() → fetch('/api/query', {
    method: 'POST',
    body: JSON.stringify({query, session_id})
})
```

### 2. FastAPI → RAG System
```python
@app.post("/api/query")
async def query_documents(request: QueryRequest):
    answer, sources = rag_system.query(request.query, session_id)
    return QueryResponse(answer=answer, sources=sources, session_id=session_id)
```

### 3. RAG System → AI Generator
```python
def query(self, query: str, session_id: str):
    prompt = f"Answer this question about course materials: {query}"
    history = self.session_manager.get_conversation_history(session_id)
    response = self.ai_generator.generate_response(
        query=prompt,
        conversation_history=history,
        tools=self.tool_manager.get_tool_definitions(),
        tool_manager=self.tool_manager
    )
```

### 4. Claude Decision Point
- **General Knowledge**: Direct response using Claude's training
- **Course-Specific**: Uses `search_course_content` tool

### 5. Tool Execution (if needed)
```python
def execute(self, query: str, course_name: str = None, lesson_number: int = None):
    results = self.store.search(query=query, course_name=course_name, lesson_number=lesson_number)
    return self._format_results(results)  # "[Course - Lesson X]\ncontent..."
```

### 6. Vector Search
```python
def search(self, query: str, course_name: str = None, lesson_number: int = None):
    # Embed query using all-MiniLM-L6-v2
    # Search ChromaDB with filters
    # Return top 5 results with metadata
```

### 7. Response Assembly
- Claude generates final answer using search results
- Sources extracted and formatted for UI
- Conversation history updated
- JSON response returned to frontend

## Performance Characteristics

### Timing Breakdown (Typical Query)
- **Frontend Processing**: ~50ms
- **FastAPI Routing**: ~10ms
- **RAG System Setup**: ~20ms
- **Initial Claude Call**: ~800ms
- **Tool Execution**: ~200ms (if search needed)
- **Final Claude Call**: ~600ms (if search needed)
- **Response Assembly**: ~30ms
- **Frontend Display**: ~50ms

**Total**: 1.8-2.5s (with search) | 1.0-1.5s (general knowledge)

### Resource Usage
- **Memory**: ~500MB (includes sentence-transformers model)
- **Storage**: ChromaDB grows with document corpus
- **Network**: ~2-4 API calls to Anthropic per query

## Setup and Deployment

### Prerequisites
- Python 3.13+
- uv package manager
- Anthropic API key

### Installation
```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install dependencies
uv sync

# Set up environment
cp .env.example .env
# Edit .env with your ANTHROPIC_API_KEY
```

### Running
```bash
# Quick start
chmod +x run.sh
./run.sh

# Manual start
cd backend
uv run uvicorn app:app --reload --port 8000
```

### Access Points
- **Web Interface**: http://localhost:8000
- **API Documentation**: http://localhost:8000/docs

## Document Processing Pipeline

### 1. Document Ingestion
```python
# Single document
course, chunks = rag_system.add_course_document("path/to/course.txt")

# Batch processing
courses, total_chunks = rag_system.add_course_folder("docs/", clear_existing=False)
```

### 2. Text Processing
- **Metadata Extraction**: Course title, instructor, lesson structure
- **Content Chunking**: Sentence-aware splitting with overlap
- **Context Enrichment**: Each chunk tagged with course/lesson info

### 3. Vector Storage
- **Embeddings**: Generated using sentence-transformers
- **Collections**: Separate storage for content and metadata
- **Indexing**: Automatic similarity indexing by ChromaDB

## API Reference

### POST /api/query
**Request:**
```json
{
    "query": "What is covered in lesson 1?",
    "session_id": "session_123"  // optional
}
```

**Response:**
```json
{
    "answer": "Lesson 1 covers...",
    "sources": ["Course Title - Lesson 1", "Course Title - Lesson 2"],
    "session_id": "session_123"
}
```

### GET /api/courses
**Response:**
```json
{
    "total_courses": 4,
    "course_titles": [
        "MCP: Build Rich-Context AI Apps with Anthropic",
        "Introduction to RAG Systems",
        "Advanced Chatbot Development",
        "AI Application Architecture"
    ]
}
```

## Error Handling

### Backend Errors
- **Document Processing**: Graceful handling of malformed files
- **API Failures**: Retry logic and fallback responses
- **Vector Search**: Empty result handling
- **Session Management**: Automatic session creation

### Frontend Errors
- **Network Failures**: User-friendly error messages
- **Loading States**: Visual feedback during processing
- **Input Validation**: Client-side query validation

## Security Considerations

### API Security
- **CORS**: Configured for development (allow all origins)
- **Input Validation**: Pydantic models validate all inputs
- **Rate Limiting**: Not implemented (add for production)

### Data Privacy
- **Session Data**: Stored in memory (not persisted)
- **API Keys**: Environment variable configuration
- **Document Content**: Local ChromaDB storage

## Extensibility

### Adding New Tools
```python
class CustomTool(Tool):
    def get_tool_definition(self) -> Dict[str, Any]:
        return {"name": "custom_tool", "description": "...", "input_schema": {...}}
    
    def execute(self, **kwargs) -> str:
        # Implementation
        return "result"

# Register with tool manager
tool_manager.register_tool(CustomTool())
```

### Custom Document Processors
- Extend `DocumentProcessor` for different file formats
- Implement custom chunking strategies
- Add metadata extraction for specialized content

### Frontend Customization
- Modify `style.css` for branding
- Extend `script.js` for additional features
- Add new UI components in `index.html`

## Monitoring and Debugging

### Logging
- **Console Output**: Document loading progress
- **Error Messages**: Detailed error information
- **Performance**: Built-in timing for major operations

### Development Tools
- **FastAPI Docs**: Automatic API documentation at `/docs`
- **Hot Reload**: Automatic server restart on code changes
- **Browser DevTools**: Frontend debugging and network inspection

## Future Enhancements

### Potential Improvements
- **Authentication**: User management and access control
- **Persistence**: Database storage for sessions and analytics
- **Caching**: Response caching for common queries
- **Analytics**: Query tracking and performance metrics
- **Multi-modal**: Support for PDF, DOCX, and other formats
- **Real-time**: WebSocket support for streaming responses

### Scalability Considerations
- **Horizontal Scaling**: Multiple FastAPI instances
- **Database Scaling**: Distributed ChromaDB setup
- **CDN**: Static asset delivery optimization
- **Load Balancing**: Request distribution across instances

---

*This documentation provides a comprehensive overview of the Course Materials RAG System architecture, implementation details, and operational characteristics.*
