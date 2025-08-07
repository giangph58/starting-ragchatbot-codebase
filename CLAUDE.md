# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Setup and Installation
```bash
# Install dependencies (requires uv package manager)
uv sync

# Create .env file with required API key
echo "ANTHROPIC_API_KEY=your_key_here" > .env
```

### Running the Application
```bash
# Quick start (recommended)
chmod +x run.sh && ./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

**Access Points:**
- Web Interface: `http://localhost:8000`
- API Documentation: `http://localhost:8000/docs`

### Prerequisites
- Python 3.13 or higher
- uv package manager
- Anthropic API key
- **Windows users**: Use Git Bash for shell commands

## Development Memories
- Always use uv to run the server, do not use pip directly
- Make sure to use uv to manage all dependencies
- Use uv run to run Python files

## Architecture Overview

### Core RAG System
This is a Retrieval-Augmented Generation (RAG) system with a **tool-augmented Claude approach**. The system uses Claude's tool calling capability to intelligently decide when to search course materials vs. use general knowledge.

**Key Architectural Pattern**: The `RAGSystem` class orchestrates all components, with Claude making autonomous decisions about when to search through the `CourseSearchTool`.

### Component Responsibilities

**Backend (`backend/`)**
- `app.py`: FastAPI application with CORS, serves frontend and API endpoints
- `rag_system.py`: Central orchestrator managing all RAG components
- `ai_generator.py`: Claude API integration with tool execution flow
- `search_tools.py`: Tool definitions for Claude's autonomous search decisions
- `vector_store.py`: ChromaDB integration with dual collections (catalog + content)
- `document_processor.py`: Text chunking with sentence-aware splitting
- `session_manager.py`: Conversation history with configurable limits
- `config.py`: Environment-based configuration management
- `models.py`: Pydantic models for type safety

**Frontend (`frontend/`)**
- Single-page vanilla JavaScript application
- Real-time chat interface with markdown rendering
- Session-aware query handling with loading states

### Data Flow Architecture

**Tool-Augmented Generation Flow:**
1. User query → FastAPI → RAGSystem → AIGenerator
2. Claude receives query + tool definitions → **autonomous decision**
3. If Claude chooses to search: Tool execution → Vector search → Results
4. Claude synthesizes final response using search context or general knowledge
5. Response flows back with source attribution

**Vector Storage Strategy:**
- `course_catalog`: Course metadata for title/instructor matching  
- `course_content`: Chunked content with lesson-aware filtering
- Sentence-transformers embeddings (`all-MiniLM-L6-v2`)

### Session Management
- Sessions auto-created per conversation
- Configurable history limits (default: 2 exchanges)
- Context preserved across queries within session

### Configuration System
Environment variables via `.env`:
- `ANTHROPIC_API_KEY`: Required for Claude API access
- Model: `claude-sonnet-4-20250514` (hardcoded)
- Chunk size: 800 characters with 100 character overlap
- Max search results: 5 per query

### Document Processing
- Supports `.txt`, `.pdf`, `.docx` files in `docs/` folder
- Automatic course parsing from structured content
- Sentence-aware chunking preserves context boundaries
- Lesson-number extraction for granular filtering

### Tool System Design
The `CourseSearchTool` enables Claude to:
- Search with semantic query matching
- Apply optional course name filters (partial matching supported)
- Filter by specific lesson numbers
- Make autonomous decisions about when search is needed

**Important**: The system is designed around Claude's intelligence - it decides when to search vs. when to use general knowledge, making it more efficient and contextually appropriate than traditional RAG systems.

### Development Notes
- Uses `uv` for fast Python package management
- ChromaDB persists to `./backend/chroma_db`
- Frontend serves from root path with no-cache headers for development
- Course documents auto-loaded on startup from `docs/` folder
- Windows compatibility requires Git Bash for shell scripts