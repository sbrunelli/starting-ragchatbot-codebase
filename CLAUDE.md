# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A RAG chatbot that answers questions about course transcripts in `docs/`. FastAPI backend + ChromaDB vector store + Anthropic Claude with tool calling; vanilla HTML/JS frontend served by the same FastAPI app.

## Commands

Uses `uv` (Python 3.13+). Requires `ANTHROPIC_API_KEY` in `.env` at the repo root (see `.env.example`).

**Always use `uv` to run the server and manage dependencies. Never use `pip` directly.** Use `uv sync` / `uv add <pkg>` instead of `pip install`, and `uv run ...` to run Python commands.

```bash
uv sync                      # install dependencies
./run.sh                     # start server on :8000 (UI at /, OpenAPI docs at /docs)
cd backend && uv run uvicorn app:app --reload --port 8000   # manual start
```

The server **must be started from `backend/`**: `app.py` uses relative paths `../docs`, `../frontend`, and `./chroma_db`.

There is no test suite, linter, or formatter configured. `main.py` at the root is an unused stub.

## Architecture

All backend modules live flat in `backend/` and import each other by bare module name (`from config import config`), so they only resolve with `backend/` as the working directory.

**Request flow** (`POST /api/query`):
`app.py` → `RAGSystem.query()` (`rag_system.py`, the orchestrator that owns every component) → `AIGenerator.generate_response()` → Claude decides whether to call the `search_course_content` tool → `ToolManager` runs `CourseSearchTool.execute()` → `VectorStore.search()` → the tool results go back to Claude in a second API call **without tools**, so only one round of tool use is possible → answer + sources returned.

**Sources side-channel:** `CourseSearchTool` stores the sources of its last search in `self.last_sources`. After generation, `RAGSystem` reads them via `ToolManager.get_last_sources()` and then calls `reset_sources()`. Sources are not part of Claude's output. Any new tool that should contribute UI sources must expose a `last_sources` attribute.

**Adding a tool:** subclass `Tool` in `search_tools.py` (implement `get_tool_definition()` returning an Anthropic tool schema and `execute(**kwargs)`), then register it with `self.tool_manager.register_tool(...)` in `RAGSystem.__init__`.

**Vector store (two ChromaDB collections):**
- `course_catalog`: one document per course; the ID is the course title. Lesson lists are stored as a JSON string in the `lessons_json` metadata field, because Chroma metadata must be scalar values.
- `course_content`: text chunks with `course_title`, `lesson_number`, `chunk_index` metadata; IDs are `{title_with_underscores}_{chunk_index}`.
- `search()` first resolves a fuzzy `course_name` to an exact title by running a vector query on `course_catalog`, then queries `course_content` with a `where` filter.

**Ingestion:** on startup, `app.py` calls `add_course_folder("../docs")`, which skips any course whose title already exists in Chroma. If you change chunking or parsing logic, delete `backend/chroma_db/` (or pass `clear_existing=True`) to rebuild the database; otherwise the old chunks remain.

**Document format** expected by `document_processor.py`, which reads every file as UTF-8 text even though `.pdf` and `.docx` files pass the extension filter:
```
Course Title: ...
Course Link: ...
Course Instructor: ...

Lesson 0: Introduction
Lesson Link: ...
<transcript text>
Lesson 1: ...
```
Chunks are sentence-based (`CHUNK_SIZE`=800 and `CHUNK_OVERLAP`=100 characters, set in `config.py`). The context prefix differs between lessons: for most lessons, only the first chunk gets `Lesson N content:`, but for the **last** lesson every chunk gets `Course <title> Lesson N content:` (the two code paths in `process_course_document`).

**Sessions:** `SessionManager` keeps history in memory only, with `MAX_HISTORY`=2 exchanges. History is injected into the **system prompt** as plain text, not as message turns.

**Frontend:** `frontend/script.js` calls `/api/query` and `/api/courses` using relative URLs, renders assistant answers with `marked` (loaded from a CDN), and shows sources in a collapsible block.

All tunables (model, embedding model, chunk sizes, `MAX_RESULTS`, Chroma path) are in `backend/config.py`.
