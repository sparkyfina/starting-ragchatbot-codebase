# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

Requires Python 3.13+ and `uv`. Run from the project root using Git Bash (required on Windows):

```bash
./run.sh
```

Or manually:
```bash
cd backend
uv run uvicorn app:app --reload --port 8000
```

App served at `http://localhost:8000`. API docs at `http://localhost:8000/docs`.

On startup, the server automatically ingests all `.txt/.pdf/.docx` files from `docs/` into ChromaDB. Re-ingestion is skipped if the course title already exists in the vector store.

## Architecture

This is a RAG (Retrieval-Augmented Generation) system. The backend is in `backend/`, the static frontend in `frontend/`, and course document source files in `docs/`.

**Query flow:**

1. `app.py` receives `POST /api/query` and delegates to `RAGSystem.query()`
2. `RAGSystem` fetches session history from `SessionManager`, then calls `AIGenerator`
3. `AIGenerator` makes a first Claude API call with a `search_course_content` tool definition
4. If Claude chooses to search (stop_reason == `"tool_use"`), `ToolManager` dispatches to `CourseSearchTool`, which queries `VectorStore`
5. `VectorStore` first resolves the course name via semantic search on `course_catalog`, then searches `course_content` with optional `course_title`/`lesson_number` metadata filters
6. Results are returned to Claude in a second API call; the final text response is returned up the chain
7. The exchange is stored in `SessionManager` (capped at last 2 exchanges = `MAX_HISTORY`)

**Key design decisions:**
- Claude drives whether to search — it is not forced. General knowledge questions skip the vector DB entirely (one API round-trip vs. two).
- `VectorStore` maintains two separate ChromaDB collections: `course_catalog` (one doc per course, used for fuzzy name resolution) and `course_content` (chunked lesson text, used for semantic search).
- Tool definitions are registered via `ToolManager.register_tool()` — adding a new tool requires only implementing the `Tool` ABC in `search_tools.py` and registering it in `RAGSystem.__init__`.

## Configuration

All tunable values are in `backend/config.py` as a dataclass. Key settings:

| Setting | Default | Purpose |
|---|---|---|
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Claude model used |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | Sentence transformer for ChromaDB |
| `CHUNK_SIZE` | 800 | Characters per text chunk |
| `CHUNK_OVERLAP` | 100 | Character overlap between chunks |
| `MAX_RESULTS` | 5 | Vector search results returned to Claude |
| `MAX_HISTORY` | 2 | Conversation exchanges kept per session |
| `CHROMA_PATH` | `./chroma_db` | ChromaDB persistence path (relative to `backend/`) |

## Course Document Format

Files in `docs/` must follow this structure for `DocumentProcessor` to parse them correctly:

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 1: <title>
Lesson Link: <url>
<lesson content...>

Lesson 2: <title>
...
```

The course title is used as the unique ID in ChromaDB — duplicate titles are skipped on reload.

## Dependencies

Managed by `uv` (`pyproject.toml` + `uv.lock`). Install with:
```bash
uv sync
```

Do not use `pip install` — this project uses `uv` exclusively.
