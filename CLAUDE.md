# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Retrieval-Augmented Generation (RAG) web application** for querying course materials. It uses:
- **FastAPI** backend with **ChromaDB** vector database
- **Anthropic Claude** (Sonnet 4) for AI generation with tool-based search
- **Sentence Transformers** (all-MiniLM-L6-v2) for embeddings
- Vanilla JavaScript frontend with markdown rendering
- **Python 3.13+** managed via **uv** package manager

## Running the Application

### Start Server
```bash
./run.sh
# OR manually:
cd backend && uv run uvicorn app:app --reload --port 8000
```

Access:
- Web UI: http://localhost:8000
- API Docs: http://localhost:8000/docs

### Install/Update Dependencies
```bash
uv sync
```

### Environment Setup
Requires `.env` file in root:
```
ANTHROPIC_API_KEY=sk-ant-...
```

## Architecture Overview

### Core Data Flow Pattern

The application uses a **two-stage AI agentic pattern** with tool calling:

1. **User Query** → FastAPI endpoint (`app.py:56`)
2. **RAG System** orchestrates all components (`rag_system.py:102`)
3. **First Claude API call** with search tool available
4. **Claude autonomously decides** whether to use `search_course_content` tool
5. If tool used: **Vector search** executes (fuzzy course name resolution → filtered semantic search)
6. **Second Claude API call** with search results to synthesize final answer
7. **Session manager** saves conversation history (max 2 exchanges)
8. **Response** returned with answer + source citations

### Two-Collection Vector Storage Strategy

**Why two collections?**
- `course_catalog`: Course metadata for fuzzy course name matching (e.g., "MCP" → "MCP: Build Rich-Context AI Apps with Anthropic")
- `course_content`: Actual chunked course content for semantic search

**Search Process** (`vector_store.py:61-100`):
1. Resolve partial course name via semantic search in `course_catalog`
2. Build filter dictionary for exact course title + optional lesson number
3. Semantic search in `course_content` with filters applied
4. Return top 5 chunks with metadata

### Document Processing Pipeline

**Format Expected** (`/docs/*.txt`):
```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: [title]
Lesson Link: [url]
[content...]

Lesson 1: [title]
[content...]
```

**Processing** (`document_processor.py:97-259`):
1. Parse structured format (regex-based lesson markers)
2. Chunk each lesson into 800-char segments with 100-char overlap (sentence-aware splitting)
3. Add context prefix to chunks: `"Course {title} Lesson {N} content: {chunk}"`
4. Generate embeddings via `all-MiniLM-L6-v2`
5. Store in both ChromaDB collections with metadata

**Why sentence-aware chunking?**
- Regex avoids splitting on abbreviations like "Dr." or "U.S."
- Overlap preserves context across boundaries
- Maintains semantic coherence for better retrieval

### Tool-Based Search Architecture

**Abstract Tool Pattern** (`search_tools.py:6-17`):
```python
class Tool(ABC):
    def get_tool_definition(self) -> Dict  # Anthropic tool schema
    def execute(self, **kwargs) -> str      # Tool implementation
```

**CourseSearchTool** (`search_tools.py:20-114`):
- Provides Claude with structured search capability
- Optional parameters: `query` (required), `course_name`, `lesson_number`
- Tracks sources in `last_sources` for UI display
- Formats results with `[Course - Lesson N]` headers

**ToolManager** (`search_tools.py:116-154`):
- Registers tools and provides definitions to Claude
- Executes tools based on Claude's decisions
- Aggregates sources from all tool executions

### Session & Conversation Management

**Session Manager** (`session_manager.py`):
- Creates UUID-based sessions on first query
- Stores message history as `List[Message]` with role + content
- **Auto-truncates** to last 2 exchanges (4 messages total) to control costs
- Formats history as string for Claude's system prompt

**Why limit history?**
- Reduces token usage and API costs
- Maintains focus on recent context
- Sufficient for most Q&A interactions

### Configuration System

**Centralized Config** (`config.py`):
- All tunable parameters in one dataclass
- Key settings:
  - `CHUNK_SIZE: 800` - Balance between context and granularity
  - `CHUNK_OVERLAP: 100` - Prevents information loss at boundaries
  - `MAX_RESULTS: 5` - Search results returned per query
  - `MAX_HISTORY: 2` - Conversation exchanges retained
  - `ANTHROPIC_MODEL: "claude-sonnet-4-20250514"`
  - `EMBEDDING_MODEL: "all-MiniLM-L6-v2"`

## Key Implementation Details

### AI Generator Performance Optimizations

**Prompt Caching** (`ai_generator.py:31-41`):
- Static `SYSTEM_PROMPT` constant to enable Anthropic's prompt caching
- Pre-built `base_params` dictionary reused across calls
- Avoids rebuilding identical prompts on each request

**Two-Call Pattern** (`ai_generator.py:43-135`):
1. First call with tools → Claude decides to search
2. Tool execution → Formatted results
3. Second call without tools → Final synthesis
4. Total overhead: ~1.5-2 seconds typical

### Vector Store Implementation Notes

**Course Name Resolution** (`vector_store.py:102-116`):
- Uses semantic search on course titles (not exact string matching)
- Enables fuzzy matching: "MCP" finds "MCP: Build Rich-Context AI Apps..."
- Returns top 1 match from `course_catalog` collection

**Filter Building** (`vector_store.py:118-133`):
- ChromaDB requires exact metadata matches after resolution
- Supports `$and` operator for course + lesson filtering
- Returns `None` if no filters (searches all content)

### Frontend-Backend Contract

**API Request** (`script.js:63-72`):
```javascript
POST /api/query
{
  "query": "user question",
  "session_id": "session_1" | null
}
```

**API Response** (`app.py:68-72`):
```json
{
  "answer": "AI generated response",
  "sources": ["Course - Lesson 1", "Course - Lesson 2"],
  "session_id": "session_1"
}
```

**Source Tracking**:
- Populated during tool execution in `CourseSearchTool._format_results()`
- Retrieved via `tool_manager.get_last_sources()`
- Reset after each query to avoid stale data

## Data Models

**Core Models** (`models.py`):
- `Course`: Metadata + list of lessons (title is unique ID)
- `Lesson`: Number, title, optional link
- `CourseChunk`: Content + metadata (course_title, lesson_number, chunk_index)

**Why course.title as ID?**
- Natural unique identifier
- Used as ChromaDB collection ID
- Enables direct lookup for metadata retrieval

## Document Ingestion

**Startup Behavior** (`app.py:88-98`):
- On server start, scans `../docs` for `.txt`, `.pdf`, `.docx` files
- Checks existing course titles to avoid re-processing
- Only adds new courses (incremental updates)
- Set `clear_existing=True` to rebuild from scratch

**Adding New Courses**:
1. Place formatted text file in `/docs` folder
2. Restart server (auto-detects and processes)
3. OR call `rag_system.add_course_document(file_path)` programmatically

## Common Modification Patterns

### Changing Search Parameters
Edit `config.py`:
- `MAX_RESULTS`: Number of chunks returned per search
- `CHUNK_SIZE`/`CHUNK_OVERLAP`: Affects retrieval granularity

### Adding New Tools
1. Create class extending `Tool` abstract base class
2. Implement `get_tool_definition()` and `execute()`
3. Register in `rag_system.py:24`: `self.tool_manager.register_tool(new_tool)`

### Modifying AI Behavior
Edit system prompt in `ai_generator.py:8-30`:
- Controls search usage patterns
- Response formatting guidelines
- Tool selection criteria

### Custom Document Formats
Modify `document_processor.py:97-259`:
- Update regex patterns for lesson markers
- Adjust metadata parsing logic
- Change chunking context prefix format
- always use uv to run the server do not use pip directly
- make sure to use uv to manage all dependencies
- use uv to run Python files