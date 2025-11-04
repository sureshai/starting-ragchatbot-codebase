# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Course Materials RAG System** - a full-stack web application that combines semantic search with AI-powered responses to answer questions about educational course content. It uses ChromaDB for vector storage, Anthropic Claude for generation, and FastAPI for the backend.

## Development Commands

### Setup
```bash
# Install dependencies
uv sync

# Create .env file with required API key
echo "ANTHROPIC_API_KEY=your_key_here" > .env
```

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start (for debugging)
cd backend && uv run uvicorn app:app --reload --port 8000
```

The application serves:
- Web interface: `http://localhost:8000`
- API docs: `http://localhost:8000/docs`

### Project Configuration
All configuration is centralized in `backend/config.py`:
- `ANTHROPIC_MODEL`: claude-sonnet-4-20250514
- `EMBEDDING_MODEL`: all-MiniLM-L6-v2 (sentence-transformers)
- `CHUNK_SIZE`: 800 characters (for document chunking)
- `CHUNK_OVERLAP`: 100 characters
- `MAX_RESULTS`: 5 (search results limit)
- `MAX_HISTORY`: 2 (conversation exchanges to remember)
- `CHROMA_PATH`: ./chroma_db (vector database location)

## Architecture

### RAG Flow (Tool-Based Search Pattern)

This system uses **tool-calling** rather than direct RAG:

1. **User Query** → FastAPI endpoint (`/api/query`)
2. **RAG System** (`rag_system.py`) orchestrates:
   - Retrieves conversation history from `SessionManager`
   - Passes query to `AIGenerator` with tool definitions
3. **Claude decides** whether to use the `search_course_content` tool
4. **Tool execution** (if Claude calls it):
   - `CourseSearchTool` invokes `VectorStore.search()`
   - ChromaDB performs semantic search with optional filters
   - Results formatted and returned to Claude
5. **Claude synthesizes** final answer using search results
6. **Response returned** with sources extracted from tool usage

**Key insight**: The AI decides when to search, not the application. This enables handling general knowledge questions without unnecessary searches.

### Component Interactions

```
app.py (FastAPI)
  ↓
rag_system.py (Orchestrator)
  ├→ document_processor.py (parse & chunk documents)
  ├→ vector_store.py (ChromaDB interface)
  │   ├→ course_catalog collection (metadata: titles, instructors, lessons)
  │   └→ course_content collection (chunked content with filters)
  ├→ ai_generator.py (Claude API with tool-calling)
  ├→ search_tools.py (tool definitions & execution)
  └→ session_manager.py (conversation history)
```

### Two-Collection Vector Strategy

`VectorStore` maintains **two ChromaDB collections**:

1. **`course_catalog`** (metadata):
   - Stores course titles, instructors, links, lessons JSON
   - Used for semantic course name resolution (e.g., "MCP" → "Introduction to MCP")
   - IDs are course titles

2. **`course_content`** (actual content):
   - Stores document chunks with metadata: `course_title`, `lesson_number`, `chunk_index`
   - Supports filtered search by course and/or lesson
   - IDs format: `{course_title}_{chunk_index}`

This separation enables fuzzy course name matching while maintaining granular content search.

### Document Processing Pipeline

`DocumentProcessor.process_course_document()` extracts structured data from text files:

1. **Metadata extraction**: First 5 lines parsed for:
   - Course title (line 1)
   - Instructor (line 2)
   - Course link (line 3)
   - Lesson metadata (lines 4-5)

2. **Content parsing**: Remaining text split into lessons using patterns like "Lesson 1:", "Lesson 2:", etc.

3. **Chunking**: Each lesson chunked with:
   - Fixed size (800 chars) with overlap (100 chars)
   - Metadata preserved: course_title, lesson_number, chunk_index

Output: `(Course, List[CourseChunk])` ready for vector storage

### Session Management

`SessionManager` maintains conversation context:
- Each session has unique UUID
- Stores up to `MAX_HISTORY` exchanges (default: 2)
- History formatted as string for Claude's system prompt
- Sessions persist in-memory (reset on server restart)

## Key Implementation Details

### Tool-Calling Pattern

The system uses Anthropic's tool-calling feature (see `ai_generator.py`):

```python
# 1. Define tools
tools = [
    {
        "name": "search_course_content",
        "description": "...",
        "input_schema": {...}
    }
]

# 2. First API call with tools
response = client.messages.create(..., tools=tools, tool_choice={"type": "auto"})

# 3. If stop_reason == "tool_use", execute tools
# 4. Second API call with tool results as user message
```

This two-round-trip pattern is handled in `AIGenerator._handle_tool_execution()`.

### Search Resolution

`VectorStore.search()` performs multi-step resolution:

1. If `course_name` provided: semantic search in `course_catalog` to resolve fuzzy name → exact title
2. Build ChromaDB filter from resolved course_title and optional lesson_number
3. Query `course_content` collection with filter
4. Return `SearchResults` object

Filters use ChromaDB's `where` syntax:
- Single filter: `{"course_title": "..."}`
- Multiple filters: `{"$and": [{"course_title": "..."}, {"lesson_number": 1}]}`

### Static File Serving

`app.py` uses custom `DevStaticFiles` class to serve frontend with no-cache headers during development, ensuring changes are immediately visible.

### Document Loading

On startup (`app.py` @startup event):
- Scans `../docs` folder for `.pdf`, `.docx`, `.txt` files
- Processes each course document
- Skips courses already in ChromaDB (based on title)
- Logs course additions

## File Structure Notes

- **backend/models.py**: Pydantic models for `Course`, `Lesson`, `CourseChunk` (data structures)
- **backend/search_tools.py**: Implements `Tool` abstract base class and `ToolManager` for registering/executing tools
- **frontend/**: Vanilla HTML/CSS/JS with Marked.js for markdown rendering
- **docs/**: Course script files (4 courses currently)

## Common Development Patterns

### Adding a New Tool

1. Create class inheriting from `Tool` in `search_tools.py`
2. Implement `get_tool_definition()` and `execute()`
3. Register in `rag_system.py`: `self.tool_manager.register_tool(YourTool())`

### Modifying Search Behavior

All search logic is in `VectorStore.search()`. Key methods:
- `_resolve_course_name()`: course name fuzzy matching
- `_build_filter()`: ChromaDB filter construction
- Adjust `MAX_RESULTS` in `config.py` to change result limits

### Changing AI Behavior

System prompt is in `AIGenerator.SYSTEM_PROMPT` (static). Key directives:
- "One search per query maximum" (prevents tool calling loops)
- "No meta-commentary" (direct answers only)
- Temperature: 0 (deterministic), max_tokens: 800

### Adding New Course Materials

Place `.txt`, `.pdf`, or `.docx` files in `docs/` folder. Server restart will auto-load them. Files must follow format:
```
[Course Title]
[Instructor Name]
[Course Link]
[Lessons metadata]

Lesson 1: [Title]
[Content...]

Lesson 2: [Title]
[Content...]
```
- always use uv to run the server. Do not user pip directly
- use uv to run Python files
- use uv to manage all dependencies