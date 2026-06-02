# AutoDev Architecture

## System Overview Diagram
```text
[ User / Dashboard ] <--> [ FastAPI Backend ] <--> [ Google Gemini 1.5 Flash ]
                               |          |
                               |          +-------> [ Vector DB / Chroma ]
                               |          |                ^
                               |          |                |
                               |          +-------> [ GCS / Local Storage ]
                               v                           |
                    [ Subprocess Sandbox ]                 |
                    (pytest / coverage.py) <---------------+
```

## Component Descriptions
- **API Gateway (FastAPI):** Orchestrates all requests, manages the agent state machine, and serves the React frontend.
- **Intelligence Engine:** Uses `tree-sitter` for structural code parsing and Gemini `text-embedding-004` to create a semantic index of the repository.
- **Custom Agent Loop:** A non-linear state machine that uses Gemini Function Calling to select tools (search, read, test, write), executes them, and observes results.
- **Toolbox:** A collection of Python functions that interface with the filesystem, `pytest`, and `tree-sitter`.
- **Sandbox Environment:** A restricted subprocess execution layer for running generated tests without compromising the host system.

## Data Flows
### 1. Ingestion Flow
`GitHub URL` -> `Clone` -> `Tree-Sitter Parsing` -> `Embedding Generation` -> `ChromaDB Persistence`

### 2. Query Flow
`User Question` -> `Semantic Search (Chroma)` -> `Context Extraction` -> `Gemini 1.5 Flash` -> `Response with Citations`

### 3. Agent Loop Flow (Action/Observation)
`Goal` -> `Gemini Tool Selection` -> `Tool Execution (e.g., Run Test)` -> `Error/Output Capture` -> `Gemini Refinement` -> `Code Fix (Unified Diff)`

## Database Schema (SQLite)
- **Repositories:** `id, name, url, last_indexed_at`
- **Files:** `id, repo_id, path, hash, summary_embedding_id`
- **Tasks:** `id, repo_id, status (pending/running/success/failed), logs`
- **Fixes:** `id, task_id, original_code, proposed_diff, test_results`

## API Surface
| Method | Path | Description |
| :--- | :--- | :--- |
| POST | `/api/ingest` | Trigger repository cloning and indexing |
| GET | `/api/search` | Semantic and keyword search across codebase |
| POST | `/api/agent/fix` | Start an autonomous fix/test loop for a bug |
| GET | `/api/tasks/{id}` | Poll status and logs of a running agent task |
| POST | `/api/tests/generate` | Generate unit tests for a specific file/function |

## Configuration Reference
- `GOOGLE_API_KEY`: For Gemini and Embeddings.
- `STORAGE_BUCKET`: GCS bucket name for index persistence.
- `SANDBOX_TIMEOUT`: Max execution time for tests (default: 30s).
- `MAX_AGENT_RETRIES`: Number of iterations for self-healing (default: 3).

## Known Limitations and Mitigations
- **Stateless Cloud Run:** ChromaDB index must be re-downloaded from GCS on container start. *Mitigation: Keep demo repos small (<100 files).*
- **Sandbox Security:** Subprocess is not a true VM/Docker sandbox. *Mitigation: Use strict timeouts and read-only filesystem mounts where possible.*
- **Gemini Context Limits:** While 1M tokens is large, massive repos will still need RAG. *Mitigation: Use hybrid BM25 + Semantic search.*

## Phase Dependency Map
1. **Phase 1 (Intelligence):** Prerequisite for all search and agent actions.
2. **Phase 2 (Agentic):** Depends on Phase 1 for context and file reading.
3. **Phase 3 (Eval):** Depends on Phase 2 logs and results.
4. **Phase 4 (Cloud):** Wraps all previous phases for production.
