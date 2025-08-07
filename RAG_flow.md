# Complete User Query Processing Flow

This document traces the detailed flow of how a user query is handled from frontend to backend in the Course Materials RAG System.

## Overview

The system processes user queries through a 9-step flow that combines modern web technologies, AI-powered search, and retrieval-augmented generation (RAG) to provide intelligent responses about course materials.

## Detailed Flow

### **1. Frontend Query Initiation** (`script.js:45-96`)

**User Action:** User types query and clicks send button or presses Enter
- `sendMessage()` function triggered
- Input validation and UI state management (disable input/button)
- User message added to chat UI with `addMessage(query, 'user')`
- Loading animation displayed via `createLoadingMessage()`

**HTTP Request:** `script.js:63-72`
```javascript
fetch(`${API_URL}/query`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
        query: query,
        session_id: currentSessionId
    })
})
```

### **2. Backend API Endpoint** (`app.py:56-74`)

**FastAPI Route Handler:** `POST /api/query`
- Receives `QueryRequest` with query text and optional session_id
- Creates new session if none provided via `rag_system.session_manager.create_session()`
- Calls core RAG system: `rag_system.query(request.query, session_id)`
- Returns `QueryResponse` with answer, sources, and session_id
- Error handling with HTTP 500 status for exceptions

### **3. RAG System Processing** (`rag_system.py:102-129`)

**Main Query Handler:**
- Builds AI prompt: `f"Answer this question about course materials: {query}"`
- Retrieves conversation history from `session_manager` if session exists
- Delegates to AI Generator with:
  - Query prompt
  - Conversation history for context
  - Available tool definitions
  - Tool manager for execution

### **4. AI Generator with Tools** (`ai_generator.py:43-87`)

**Claude API Integration:**
- Constructs system prompt with conversation history
- Prepares API call parameters:
  - Model: `claude-sonnet-4-20250514`
  - Temperature: 0 (deterministic responses)
  - Max tokens: 800
  - Tools: Available search tools
  - Tool choice: "auto" (Claude decides when to use tools)

**Claude Decision Making:**
- Analyzes query to determine if course-specific search is needed
- For general questions: Uses existing knowledge
- For course-specific questions: Triggers tool usage

### **5. Tool-Based Search** (`search_tools.py:52-80`)

**Course Search Tool Execution:**
- Claude calls `search_course_content` tool with parameters:
  - `query`: What to search for
  - `course_name`: Optional course filter (partial matches supported)
  - `lesson_number`: Optional lesson number filter
- Tool validates parameters and calls vector store search

### **6. Vector Store Search** (`vector_store.py:34-60`)

**ChromaDB Integration:**
- Uses sentence-transformers embeddings (`all-MiniLM-L6-v2`) for semantic search
- Searches two collections:
  - `course_catalog`: Course metadata (titles, instructors)
  - `course_content`: Actual course material chunks
- Applies filters for course name and lesson number if specified
- Returns `SearchResults` with:
  - Documents: Matching text content
  - Metadata: Source information
  - Distances: Similarity scores
  - Error messages if search fails

### **7. Response Generation** (`ai_generator.py:89-135`)

**Tool Results Processing:**
- Collects search results from all tool executions
- Constructs conversation with:
  - Original user query
  - Assistant's tool use request
  - Tool execution results
- Makes follow-up API call to Claude with search context
- Claude generates final synthesized response based on retrieved content

### **8. Response Assembly** (`rag_system.py:129-140`)

**Final Processing:**
- Extracts sources from search tool's last execution
- Updates session conversation history with user query and response
- Returns tuple of `(response_text, sources_list)`
- Session manager maintains context for future queries

### **9. Frontend Response Handling** (`script.js:74-95`)

**UI Updates:**
- Removes loading animation
- Parses JSON response containing:
  - `answer`: AI-generated response text
  - `sources`: List of source references
  - `session_id`: Session identifier for context
- Updates `currentSessionId` for future requests
- Displays response with:
  - Markdown rendering using `marked.js`
  - Collapsible sources section if sources exist
  - Proper HTML escaping for security
- Re-enables input controls and focuses for next query

## Key System Characteristics

### **Session Management**
- Maintains conversation context across multiple requests
- Each session tracks conversation history (configurable limit)
- Session IDs generated automatically if not provided

### **Tool-Augmented Generation**
- Claude intelligently decides when to search vs. use general knowledge
- Single search per query maximum to optimize performance
- No meta-commentary about search process in responses

### **Semantic Search**
- Uses sentence-transformers for embedding generation
- Supports partial course name matching
- Lesson-specific filtering capability
- ChromaDB for persistent vector storage

### **Error Handling**
- Graceful fallbacks at each processing layer
- User-friendly error messages
- Network timeout and retry logic in frontend

### **Real-time UI Experience**
- Loading states during query processing
- Progressive enhancement with markdown rendering
- Collapsible source references
- Cache-busting headers for development

## Data Flow Summary

```
User Input → Frontend (JS) → FastAPI → RAG System → AI Generator → Claude API
                                                        ↓
User Response ← Frontend ← FastAPI ← RAG System ← Tool Execution ← Vector Search
```

## Performance Optimizations

1. **Pre-built API parameters** in AI Generator
2. **Static system prompts** to avoid rebuilding
3. **Efficient collection management** in ChromaDB
4. **Single search per query** limitation
5. **Conversation history limits** for context management

This architecture provides a robust, scalable solution for intelligent course material querying with proper source attribution and conversational context.