# Architecture Overview

PathReview is a multi-service application with five major subsystems. This document describes how they fit together.

## High-Level Data Flow

```
User Input (GitHub username, resume PDF, repo URLs)
    │
    ▼
┌─────────────────────┐
│  API Layer (FastAPI) │ ← Authentication, validation, rate limiting
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Ingestion Pipeline  │ ← Parse documents, chunk, embed, store
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Agent Orchestrator  │ ← Plan analysis, execute tools, manage state
│  ┌───────────────┐  │
│  │ GitHub Tool    │  │
│  │ Skill Extract  │  │
│  │ README Scorer  │  │
│  │ Market Analyze │  │
│  │ Tech Detector  │  │
│  └───────────────┘  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  RAG System          │ ← Retrieve context, generate feedback, evaluate
│  (Hybrid Retrieval   │
│   + LLM Generation)  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Safety Layer        │ ← Bias check, content filter, PII scrub
└──────────┬──────────┘
           │
           ▼
      Review Output
```

## Subsystem Details

### API Layer (`api/`)
FastAPI application serving REST endpoints. Handles authentication (JWT), request validation (Pydantic), rate limiting, and CORS. Routes delegate to service layer in `core/services/`.

### Ingestion Pipeline (`ingestion/`)
Processes user-submitted documents into vector embeddings. Parsers implement `BaseParser` and extract structured text. Chunkers split text for embedding. The pipeline orchestrates: parse → chunk → embed → store.

### Agent System (`agent/`)
A plan-execute orchestrator that coordinates multiple analysis tools. Each tool implements `BaseTool` with `name`, `description`, and `execute()`. The orchestrator builds a plan based on available profile data, executes tools with retry and timeout policies, and synthesizes results.

### RAG System (`rag/`)
Hybrid retrieval (vector similarity + BM25 keyword) fetches relevant context from the user's ingested documents. The generator uses prompt templates to produce structured, evidence-based feedback. The evaluator scores retrieval relevance and generation faithfulness.

#### Hybrid retrieval scoring

`HybridRetriever` (`rag/retriever/hybrid.py`) blends a vector score and a BM25 keyword score for each candidate chunk:

```
blended_score = vector_weight × vector_score + keyword_weight × keyword_score
```

- **Default weights:** `vector_weight = 0.7` and `keyword_weight = 0.3`. These are constructor defaults, so the blend favors semantic similarity over exact word matches.
- **Normalization:** each score is divided by the highest score in that search's results for the query, so the top result of each search is 1.0. In math form: `score = w_v · (s_vec / max s_vec) + w_k · (s_bm25 / max s_bm25)`. Scores are relative to the batch, not absolute.
- **Single-search matches:** a chunk found by only one search gets 0 for the other score.
- **Filtering:** chunks with a blended score below `min_score` (default 0.3) are dropped, and the top `max_chunks` (default 10) are returned, highest first.

**Example (illustrative numbers, default weights):**

| Chunk | Vector (normalized) | Keyword (normalized) | Blended score | Result |
|---|---|---|---|---|
| A | 1.00 | 0.50 | 0.7 × 1.00 + 0.3 × 0.50 = 0.85 | Kept |
| C | 0.60 | 1.00 | 0.7 × 0.60 + 0.3 × 1.00 = 0.72 | Kept |
| B | 0.40 | 0 (not found by keyword search) | 0.7 × 0.40 + 0.3 × 0 = 0.28 | Dropped (below 0.3) |

The code in `rag/retriever/hybrid.py` is the source of truth if these values change.

### Safety Layer (`safety/`)
Middleware wrapping the generation pipeline. Components run in sequence: prompt injection defense → content filter → bias detector → PII scrubber. All safety events are logged with structured metadata for monitoring.

## Key Design Decisions

See the Architecture Decision Records in `docs/adr/` for context on major decisions:
- [ADR-001: Chunking Strategy](adr/001-chunking-strategy.md)
- [ADR-002: Embedding Model Selection](adr/002-embedding-model.md)
- [ADR-003: Agent Orchestration Approach](adr/003-agent-orchestration.md)
