# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
uv sync

# Run the application
./run.sh
# or manually:
cd backend && uv run uvicorn app:app --reload --port 8000
```

The app serves both the API and static frontend at `http://localhost:8000`. API docs at `/docs`.

## Architecture

This is a **tool-based RAG chatbot** for querying course materials. Unlike traditional RAG pipelines, it uses Claude's native tool-calling to decide when and how to search — the AI drives retrieval, not a fixed pipeline.

**Stack:** FastAPI + ChromaDB + SentenceTransformers + Anthropic Claude (claude-sonnet-4-20250514)

### Backend (`backend/`)

| File | Role |
|------|------|
| `app.py` | FastAPI app, CORS/proxy middleware, `/api/query` and `/api/courses` endpoints, startup document loading |
| `rag_system.py` | Orchestrator — wires together all components; `query()` is the main entry point |
| `ai_generator.py` | Claude API integration; implements the agentic tool-calling loop |
| `vector_store.py` | ChromaDB wrapper with two collections: `course_catalog` (metadata) and `course_content` (searchable chunks) |
| `document_processor.py` | Parses `.txt`/`.pdf`/`.docx` course files and chunks them (800 chars, 100 overlap) |
| `search_tools.py` | `CourseSearchTool` definition + `ToolManager`; tools are passed to Claude as schemas |
| `session_manager.py` | In-memory session → conversation history map (max 2 messages) |
| `models.py` | Pydantic models: `Course`, `Lesson`, `CourseChunk` |
| `config.py` | Loads `.env`, embedding model (`all-MiniLM-L6-v2`), ChromaDB path, chunk settings |

### Request Flow

1. Frontend POSTs to `/api/query` with `{question, session_id?}`
2. `RAGSystem.query()` passes conversation history + tools to `AIGenerator`
3. Claude decides whether to call `CourseSearchTool` (query, optional course_name/lesson_number)
4. Tool executes vector search on ChromaDB; results fed back to Claude
5. Claude synthesizes final response; sources returned to frontend

### Frontend (`frontend/`)

Vanilla HTML/CSS/JS. Calls `/api/query` and `/api/courses`. Renders markdown via marked.js. No build step.

### Course Documents (`docs/`)

Expected format for `.txt` files:
```
Course Title
Course Link
Instructor Name

Lesson 1: Title
[content...]

Lesson 2: Title
[content...]
```

Course title is used as the unique deduplication key — loading is idempotent.

## Environment

Requires `ANTHROPIC_API_KEY` in a `.env` file at the project root (see `.env.example`).

ChromaDB persists to `backend/chroma_db/`. Delete this directory to reset the vector store.

## Key Implementation Details

- **Tool-calling loop**: `ai_generator.py` recursively calls Claude until it stops requesting tools. Temperature is 0 for deterministic output.
- **Course name resolution**: `vector_store.py` uses semantic search to fuzzy-match partial course names before filtering.
- **Session history cap**: Capped at 2 exchanges to keep token usage low — see `session_manager.py`.
- **Python 3.13** required; use `uv` (not pip) for dependency management.