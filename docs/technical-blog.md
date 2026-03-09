# Building a Resource Librarian Agent with LangGraph and Supabase

The Resource Librarian Agent started as a small experiment in LangGraph and quickly evolved into a deliberate exploration of agent design, tool integration, and semantic search. This article walks through the technical architecture that powers the project today—from the conversational loop to the pgvector upgrade that brought contextual retrieval online.

## Problem Statement

Knowledge workers save a constant stream of papers, blog posts, and notes across different channels. The goal was to build an “agent-as-a-service” that could:

1. Persist resources with consistent metadata (title, URL, tags, categories, notes).
2. Retrieve relevant items by intent, not just literal keywords.
3. Remain local-first and transparent, with every decision open for inspection.

## Architectural Overview

```
┌─────────────────────────────────────────────────────┐
│ User Interfaces                                     │
│  - CLI loop (src/main.py)                           │
└─────────────────────────────────────────────────────┘
                 │  HumanMessage/AIMessage
┌────────────────▼────────────────────────────────────┐
│ LangGraph Workflow (src/agent/graph.py)             │
│ 1. classify_input                                   │
│ 2. store_resource / fetch_resources / fallback_chat │
│ 3. Result formatting                                │
└────────────────┬────────────────────────────────────┘
                 │ Tool invocations
┌────────────────▼────────────────────────────────────┐
│ Tools and Services                                  │
│  - SupabaseResourceClient (src/tools/supabase_client.py)│
│  - EmbeddingService (src/core/embeddings.py)         │
│  - Supabase RPC `match_resources` (pgvector)         │
└────────────────┬────────────────────────────────────┘
                 │ Supabase SDK
┌────────────────▼────────────────────────────────────┐
│ Supabase (Postgres)                                 │
│  - `resources` table                                │
│  - `embeddings_vector` (vector(1536))               │
│  - `match_resources` function                       │
└─────────────────────────────────────────────────────┘
```

The guiding principle is separation of concerns: LangGraph is the conversation brain; the Supabase toolkit is a set of deterministic tools; Postgres stores truth.

## Conversation Flow with LangGraph

LangGraph’s `StateGraph` gives the agent a deterministic loop:

1. **classification** converts free text into a JSON payload with intent + structured fields for tags, categories, query terms, etc.
2. Depending on the intent we route to **store_resource**, **fetch_resources**, or **fallback_chat**.
3. Every node returns deterministic `AIMessage` responses so the model stays grounded in saved content.

State is a `TypedDict` (`src/agent/state.py`) with `messages`, `intent`, `parsed_request`, and `results`. This keeps the workflow transparent and testable.

## Supabase as the Source of Truth

`SupabaseResourceClient` encapsulates every database interaction:

- Upserts reuse titles/URLs to avoid duplicates.
- Inserts populate `title`, `url`, `notes`, `tags`, `categories`, and the new `embeddings_vector` column.
- Retrieval calls either the semantic RPC or a keyword fallback (including URL/tags/categories).

This wrapper lets the graph talk about “resources” in domain language while the SDK handles SQL details.

## Bringing pgvector Online

We shifted from basic `ilike` matching to contextual search by:

1. **Enabling pgvector** and adding an `embeddings_vector` column to `resources`.
2. **Creating the `match_resources` function** (`supabase/match_resources.sql`) which:
   - Drops older signatures before creating the latest version.
   - Returns `%TYPE` columns to stay compatible with any schema changes.
   - Accepts optional tag/category filters and a configurable `match_threshold` (default `1.0`, meaning “return top-k regardless of distance”).
3. **Embedding every resource** via `src/core/embeddings.py`, which supports OpenAI or Ollama models. On insert/update we compose a text payload (title + notes + URL) and request a vector. A backfill script (`scripts/backfill_embeddings.py`) populates historical rows.
4. **Updating retrieval logic** to blend semantic results with the keyword fallback. When embeddings fail, we still search by plural/singular variants, URLs, tags, and categories.

## Tooling and Tests

- `uv` manages dependencies and Python version locking (`uv.lock`).
- Pytest covers the helper functions (`tests/`), ensuring keyword expansion, embedding size validation, and state utilities behave.
- Environment variables live in `.env`, typed via `pydantic-settings`. The embedding configuration exposes `EMBEDDING_PROVIDER`, `EMBEDDING_MODEL`, `EMBEDDING_DIMENSIONS`, and `EMBEDDING_MATCH_THRESHOLD` for quick tuning.

## Developer Workflow

1. `uv sync` to install dependencies.
2. Configure Supabase keys and embedding provider in `.env`.
3. Run the CLI with `uv run python -m src.main`.
4. When enabling semantic search: apply the SQL function, set `EMBEDDING_MATCH_THRESHOLD`, and run the backfill script.

Commit history captures milestones in `AI.md`, while `techREADME.md` details architectural choices. The README doubles as a quick start guide.

## Lessons Learned

- **Keep the agent declarative.** LangGraph nodes should describe intent and orchestrate tools; implementation details belong in reusable modules.
- **Vectors belong with structured data.** Storing embeddings in the same Supabase table avoids sync headaches and keeps queries transactional.
- **Fallbacks matter.** Even with semantic search, keyword expansion (singular/plural, tags, URLs) is crucial for reliability.
- **Documentation pays off.** Every significant change updated README, tech notes, and the AI development log, making context switch painless.

## What’s Next

- Persisting conversation memory via LangGraph checkpointers stored in Supabase.
- Structured output validation to harden the classifier.
- Richer retrieval summaries (LLM-generated bullet points or grouped results).
- Optional multichannel ingestion (email, RSS, or a dedicated WhatsApp deployment branch).

The Resource Librarian Agent is now a solid foundation for experimentation: a transparent, scriptable agent stack blending deterministic tooling with LLM flexibility.
