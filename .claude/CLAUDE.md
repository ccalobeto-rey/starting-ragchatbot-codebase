# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

```bash
# Install dependencies (run from project root)
uv sync

# Start server (runs from backend/ internally)
chmod +x run.sh
./run.sh
# or: cd backend && uv run uvicorn app:app --reload --port 8000
```

Requires a `.env` file in the project root:
```
ANTHROPIC_API_KEY=your_key_here
```

- Web UI: http://localhost:8000
- API docs: http://localhost:8000/docs

## Code Quality

```bash
uv sync --group dev   # install dev deps first

./scripts/format.sh   # auto-fix: isort + black + flake8 + mypy
./scripts/lint.sh     # read-only checks (exit 0 = passing)
```

Always use `uv run` to execute Python: `uv run python script.py`. Never use `pip` directly — use `uv add package_name` to add dependencies.

## Architecture

All backend code lives in `backend/`. The entry point is `app.py` (FastAPI). Components are loosely coupled through `RAGSystem`, which owns all subsystems.

**Query flow:**
1. `POST /api/query` → `RAGSystem.query()` → `AIGenerator.generate_response()`
2. Claude (claude-sonnet-4-20250514) makes **two API calls per query**:
   - Call 1: Decides whether to invoke `search_course_content` tool
   - Call 2: Generates final answer using retrieved chunks (if any)
3. `CourseSearchTool` → `VectorStore.search()` → ChromaDB semantic search
4. Sources extracted from tool results are returned alongside the answer

**Dual ChromaDB collections** (`backend/chroma_db/`):
- `course_catalog`: Course-level metadata (title, instructor, link, lessons list) — used for course name resolution
- `course_content`: Text chunks with metadata (`course_title`, `lesson_number`, `chunk_index`) — used for semantic search

**Adding new tools:** Subclass `Tool` (ABC in `search_tools.py`), implement `get_tool_definition()` and `execute()`, then `tool_manager.register_tool(instance)`. Tools auto-appear in Claude's tool list.

**Session history** is stored in-memory by `SessionManager` (last 2 exchange pairs, configurable via `config.MAX_HISTORY`). History is injected into the system prompt, not the messages array.

## Document Format

Docs in `docs/` must follow this structure for full parsing:
```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 1: <title>
Lesson Link: <url>
<lesson content>

Lesson 2: <title>
...
```

Chunks are sentence-aware (800 chars, 100 char overlap). First chunk of each lesson is prefixed with `"Lesson N content: ..."` for retrieval context. Documents without lesson markers are chunked as a single block.

## Key Configuration (`backend/config.py`)

- `CHUNK_SIZE`: 800 / `CHUNK_OVERLAP`: 100
- `EMBEDDING_MODEL`: `all-MiniLM-L6-v2` (SentenceTransformers)
- `MAX_RESULTS`: 5 search results per query
- `MAX_HISTORY`: 2 conversation pairs retained
- `ANTHROPIC_MODEL`: `claude-sonnet-4-20250514`
