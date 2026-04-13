#   — ADK Knowledge Graph RAG Agent

> **Planning Compliance AI for    **

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [How It Works](#how-it-works)
- [Search Modes](#search-modes)
- [Two-Call Citation Strategy](#two-call-citation-strategy)
- [A er-Based Provenance Search](#a er-based-provenance-search)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running the Server](#running-the-server)
- [API Usage](#api-usage)
- [Chunk Log Files](#chunk-log-files)
- [Known Limitations](#known-limitations)
- [Bug Fix History](#bug-fix-history)
- [Environment Variables](#environment-variables)

---

## Overview

  is a production planning compliance AI assistant built on Google's Agent Development Kit (ADK). It a ers questions about   planning regulations by querying a Knowledge Graph built from 112+   Shire Council planning documents — Local Environmental Plans (LEPs), Development Control Plans (DCPs), and State Environmental Planning Policies (SEPPs).

### Key Capabilities

- Natural language questions about planning controls, setbacks, zones, development types
- Inline sentence-level citations with chapter, section, clause, and page references
- A ers grounded exclusively in retrieved document text — no hallucination from training knowledge
- Knowledge Graph with 2,761 entities and 5,565 relationships
- Multiple retrieval modes: semantic RAG, graph traversal, chain-of-thought reasoning

### Example

> **Q:** What are the waste management requirements for a dual occupancy in E3 zone in   Shire Council?
>
> **A:** Each dwelling must be provided with a waste storage area capable of accommodating a "120-litre garbage bin", a "240-litre recycling bin", and a "240-litre green waste bin" *(  Shire DCP 2015, Chapter 4 – Dual Occupancy, E3 Environmental Management Zone, Section 8.2 Controls, Clause 1, Page 19)*. The waste storage area must not be located forward of the building line and must not detract from the streetscape *(  Shire DCP 2015, Chapter 4, Section 8.2, Clause 2, Page 19)*.

---

## Architecture

```
User Question
      │
      ▼
┌─────────────────────────────────────────┐
│         Google ADK Agent                │
│   (Gemini 2.5 Pro + agent.py rules)     │
└──────────────┬──────────────────────────┘
               │ Two parallel tool calls
      ┌────────┴────────┐
      ▼                 ▼
  CALL 1             CALL 2
RAG_COMPLETION       A er-based
(or graph mode)      Provenance Search
      │                 │
      ▼                 ▼
 Cognee KG          LanceDB direct
 (internal          (your code,
  chunks → LLM)      visible chunks)
      │                 │
      ▼                 ▼
  A ER_TEXT      SOURCE_CHUNKS
  (what the         (with [SOURCE:]
   rule says)        headers for
                     citation)
      └────────┬────────┘
               ▼
    Final Response:
    Facts from A ER_TEXT
    Citations from SOURCE_CHUNKS
    + Mandatory disclaimer
```

---

## Tech Stack

| Component | Technology |
|---|---|
| Agent Framework | Google Agent Development Kit (ADK) |
| LLM | Gemini 2.5 Pro (`gemini-2.5-pro`) |
| Embedding Model | `gemini-embedding-001` (768 dimensions) |
| Knowledge Graph | Cognee 0.5.2 |
| Vector Store | LanceDB (local) |
| Graph Database | Kuzu (via Cognee) |
| Relational Store | SQLite (via Cognee) |
| Web Server | Uvicorn + FastAPI (via ADK) |
| Python | 3.10.18 |
| Environment | Conda (`cognee_env`) |

---

## Project Structure

```
adk-rag-agent-main/
├── run_remote.py                  # Server entry point with CORS, middleware
├── .env                           # API keys (not committed)
│
├── rag_agent/
│   ├── __init__.py                # Cognee init, LanceDB verification, config
│   ├── agent.py                   # ADK Agent definition + tool rules instruction
│   │
│   ├── tools/
│   │   ├── __init__.py
│   │   └── rag_query.py           # Core RAG tool — all retrieval logic
│   │
│   └── config.py                  # DEFAULT_CHUNK_LIMIT, SUPPORTED_COUNCILS, etc.
│
└── chunk_logs/                    # Auto-created on first query
    ├── chunks_latest.json         # Most recent query chunks (overwritten each run)
    └── chunks_log.jsonl           # Full history, one JSON object per line
```

---

## How It Works

### Step 1 — User Question Arrives

The user's question is received by the ADK server and passed verbatim to Gemini 2.5 Pro. The `agent.py` instruction enforces that the query must be passed **character-for-character identical** to `rag_query` — no rewording, no keyword conversion, no synonym substitution.

### Step 2 — Two Parallel Tool Calls

The model fires two `rag_query` calls simultaneously:

**Call 1 — Retrieve the factual a er**
Uses `RAG_COMPLETION` (default) or a graph mode. Cognee runs an internal vector search, retrieves matching chunks, and sends them to Gemini internally to synthesise a clean a er. Returns `result["a er"]` as a string. **Does not return source chunks to your code.**

**Call 2 — Retrieve source chunks for citations**
Runs an **a er-based provenance search**: uses the a er text from Call 1 (not the original question) as the embedding query. Because the a er is a paraphrase of its source, source chunks score much higher against the a er than against the question. Returns `result["chunks"]` with `[SOURCE: Document | Chapter | Page]` headers prepended.

### Step 3 — Construct Final Response

The model:
- Takes every fact from `A ER_TEXT` only (forbidden from adding training knowledge)
- Finds the citation for each fact in `SOURCE_CHUNKS` by reading the `[SOURCE:]` headers
- Formats with sentence-level inline citations per domain expert rules
- Appends the mandatory disclaimer

---

## Search Modes

| Mode | Use Case | Notes |
|---|---|---|
| `RAG_COMPLETION` | **Default.** Specific controls, setbacks, measurements | Semantic search + LLM synthesis |
| `GRAPH_COMPLETION` | Zone permissibility, entity relationships | Single-pass graph traversal |
| `CONTEXT_EXTENSION` | Exploratory or open-ended questions | Iterative graph expansion |
| `GRAPH_COT` | Complex multi-constraint questions | 3–4 min. Use sparingly. |
| `TRIPLET_COMPLETION` | Single fact lookups | Returns raw graph triplets |
| `CHUNKS` | Reserved for Call 2 (citation retrieval) | Direct LanceDB, no LLM synthesis |

**Do not use:** `CHUNKS_LEXICAL`, `SUMMARIES` — known reliability issues in Cognee 0.5.2.

---

## Two-Call Citation Strategy

### The Problem It Solves

`RAG_COMPLETION` returns a clean a er but **no source metadata** — Cognee consumes the chunks internally and throws them away. If you try to cite from a `RAG_COMPLETION` a er, you have nothing to cite from except your training knowledge, which produces wrong chapter numbers, wrong page numbers, and wrong clause references consistently.

### Why A er-as-Query Works

```
BEFORE (query-based chunk search — zone-blind):
  Query:  "waste management requirements E3 zone?"
  Result: B1 zone chunk scores 1.000 ← WRONG
          R4 zone chunk scores 0.999 ← WRONG
          E3 zone chunk scores 0.797 ← CORRECT but ranked 4th

AFTER (a er-based provenance search):
  A er: "In the E3 zone, each dwelling must have a 120-litre bin..."
  Result: E3 zone chunk scores 0.995 ← CORRECT, ranked 1st
          B1 zone chunk scores 0.821 ← excluded by dynamic re-ranking
```

The a er already contains the zone-specific language that RAG_COMPLETION extracted from the correct source. Searching with the a er text finds that source passage directly.

---

## A er-Based Provenance Search

Implemented in `_find_source_chunks_for_a er()` in `rag_query.py`.

### Two-Stage Process

**Stage 1 — A er-based vector search**
Uses the first 500 characters of the a er as the embedding query against the `DocumentChunk_text` LanceDB table. Retrieves top-K chunks by cosine similarity.

**Stage 2 — Dynamic zone re-ranking**
Extracts the zone code (`E3`, `R2`, `B1`, etc.) and zone name (`Environmental Management`, `Low Density Residential`, etc.) directly from the a er text — no predefined zone lists required. Re-ranks chunks so zone-matching chunks always appear first, regardless of raw similarity score.

### Why Dynamic Extraction (Not a Predefined List)

The a er text already tells us which zone applies because `RAG_COMPLETION` generated it from the correct document. Extracting the zone from the a er is self-contained and works for any zone, any council, any document type without configuration.

---

## Installation

### Prerequisites

- Conda (Miniconda or Anaconda)
- Google Gemini API key
- Cognee 0.5.2 with pre-built LanceDB knowledge graph

### Environment Setup

```bash
# Clone the repository
cd adk-rag-agent-main

# Activate the cognee environment
conda activate cognee_env

# Install dependencies (if setting up fresh)
pip install google-adk cognee lancedb pandas
```

### Pre-built Knowledge Graph

The LanceDB knowledge graph must be built separately using Cognee's ingestion pipeline before running this agent. The graph should be located at:

```
/path/to/cognee/.cognee_system/databases/cognee.lancedb
```

Update `LANCEDB_PATH` in `rag_agent/tools/rag_query.py` to match your path.

The graph should contain the `DocumentChunk_text` table (minimum 300+ rows for full   Shire DCP coverage).

---

## Configuration

### `.env` file (project root)

```env
GOOGLE_API_KEY=your_gemini_api_key_here
GEMINI_API_KEY=your_gemini_api_key_here   # Optional duplicate
```

### `rag_agent/config.py`

```python
DEFAULT_CHUNK_LIMIT = 20       # Chunks retrieved per search (increase for better coverage)
DEFAULT_TOP_K = 10             # Top-K for graph modes
DEFAULT_COUNCIL = " _shire"
SUPPORTED_COUNCILS = [" _shire"]
```

### `rag_agent/tools/rag_query.py`

```python
LANCEDB_PATH = "/path/to/cognee/.cognee_system/databases/cognee.lancedb"
```

---

## Running the Server

### Local Development

```bash
conda activate cognee_env
cd adk-rag-agent-main
python run_remote.py
```

Server starts at: `http://localhost:8020/dev-ui?app=rag_agent`

### Remote / Network Access

```bash
python run_remote.py --host 0.0.0.0 --port 8020 --allow_origins "*"
```

Access from network: `http://<your-ip>:8020/dev-ui?app=rag_agent`

### Custom UI Integration

The server exposes a REST API at `/run` (non-streaming) and `/run_sse` (streaming). Your custom UI injects domain expert rules as the first message before the user's question. See the ADK documentation for session management.

---

## API Usage

### Create Session

```bash
POST /apps/rag_agent/users/{user_id}/sessions
```

### Send Message

```bash
POST /run
Content-Type: application/json

{
  "app_name": "rag_agent",
  "user_id": "user",
  "session_id": "your-session-id",
  "new_message": {
    "role": "user",
    "parts": [{"text": "what are the rear setback requirements for a dual occupancy in R2 zone?"}]
  }
}
```

### Response Format

```json
{
  "content": {
    "parts": [{"text": "The minimum rear setback is \"4.0m\" (  Shire DCP 2015, Chapter 4...)"}],
    "role": "model"
  }
}
```

---

## Chunk Log Files

Every CHUNKS call writes to `chunk_logs/` in the project root:

```
chunk_logs/
├── chunks_latest.json    # Most recent query — overwritten each time
└── chunks_log.jsonl      # Full history — one JSON line per query
```

### chunks_latest.json structure

```json
{
  "timestamp": "2026-03-04T15:59:01",
  "query": "what are the waste management requirements...",
  "chunks_count": 10,
  "chunks": [
    {
      "chunk_id": 1,
      "score": 0.997,
      "source": "  Shire Planning Document",
      "text": "[SOURCE:   Shire DCP 2015 | Chapter 4: Dual Occupancy - E3 Environmental Management Zone | Page 19]\nThe design of waste and recyclables storage areas..."
    }
  ]
}
```

Use these logs to diagnose citation accuracy — check whether the top-scoring chunks are from the correct zone and chapter for each query.

```bash
# View last 5 queries and their top sources
tail -5 chunk_logs/chunks_log.jsonl | python3 -c "
import sys, json
for line in sys.stdin:
    d = json.loads(line)
    print(d['query'][:80])
    if d['chunks']:
        print(' ->', d['chunks'][0]['text'][:120])
    print()
"
```

---

## Known Limitations

### Context Window and Session Length

Each turn adds ~4,500 tokens to the context (domain rules ~8K, agent instruction ~2K, per-turn tool results ~3.75K). With Gemini 2.5 Pro's 1M token limit, sessions can handle approximately 200+ turns before overflow. For practical use, session resets after 40 turns are recommended to maintain instruction-following quality.

### Knowledge Graph Coverage

The current graph covers   Shire Council only (330 document chunks, 2,761 entities, 5,565 relationships). Multi-council support is planned via the `council_name` routing parameter.

### Provenance Search Accuracy

A er-based provenance search works well when the a er contains zone-specific or development-type-specific language. For highly generic a ers (e.g. common definitions that appear identically across many zones), the top-ranked source chunk may still be from a different zone. Always verify citations against the `chunks_latest.json` log.

### Cognee 0.5.2 Payload Bug

`cognee.infrastructure.databases.vector.get_vector_engine().search()` returns `ScoredResult` objects where `payload` is always `None`. Only `id` and `score` are populated. This code works around this by fetching rows directly from LanceDB using the returned IDs, then parsing the payload column (a Python dict serialised as a single-quoted string) using `ast.literal_eval`.

---

## Bug Fix History

### v1.3 — A er-Based Provenance Search (Current)

**Problem:** Vector similarity is zone-blind. For an E3 zone query about waste management, B1 and R4 zone chunks score higher (1.000, 0.999) than the correct E3 chunk (0.797) because the rules are textually identical across zones. The model cited the wrong zone.

**Fix:** `_find_source_chunks_for_a er()` — uses the RAG_COMPLETION a er text (not the user question) as the embedding query for CHUNKS. The a er already contains zone-specific language, so the correct source chunk scores highest. Dynamic zone re-ranking extracted from the a er text eliminates residual mismatches without any predefined zone lists.

### v1.2 — Chapter Context Extraction and Token Overflow Fix

**Problem:** Chunks were raw mid-document text fragments with no chapter or page information visible. The model could not cite correctly because citations require chapter/section/page context that wasn't in the chunk text. Additionally, the LanceDB `vector` column (768 floats per row, ~4,000 characters per row when serialised) was being passed to the LLM, causing 500 errors after a few conversation turns due to the 1M token limit being exceeded.

**Fix:**
- `_extract_chapter_context_from_row()` — scans chunk text with regex for `Chapter X - Name`, `Section X.X`, `Page N` patterns and prepends a `[SOURCE: Document | Chapter | Page]` header to each chunk
- Vector column stripped immediately after `table.to_pandas()` with `DROP_COLS`
- Chunk text truncated to 1,500 characters per chunk (~375 tokens × 10 chunks = ~3,750 tokens total)

### v1.1 — LanceDB Payload Parsing Fix

**Problem:** `_search_chunks_direct` returned 10 chunks with 0 text. Every chunk was empty. Root cause: `ScoredResult.payload` is always `None` in Cognee 0.5.2 — the text lives in the LanceDB table's `payload` column as a Python dict serialised as a **single-quoted string** (not JSON). `json.loads` fails on single quotes.

**Fix:** Two-step retrieval:
1. `vector_engine.search()` → get ranked IDs only
2. `lancedb.connect().open_table().where("id IN (...)").to_pandas()` → fetch actual rows
3. `ast.literal_eval(payload_string)` → parse the single-quoted dict → extract `payload_dict["text"]`

### v1.0 — Query Modification and Hallucination Fix

**Problem:** The model rewrote user queries before calling `rag_query` (removing words, converting to keyword format, adding "DCP 2015"), causing retrieval to return slightly wrong document sections. Additionally, missing source constraint rules in `agent.py` (the instruction was entirely commented out with `#`) allowed the model to augment RAG a ers with training knowledge — adding fabricated driveway grade figures,  n Standard references, and wrong clause numbers.

**Fix:**
- `agent.py` Tool Rule 1: explicit FORBIDDEN actions list with concrete examples
- `agent.py` Tool Rule 2: mandatory two-call procedure with source constraint enforcement
- Section 4 source constraint rules written as plain text (not commented out)
- Concrete fabrication example (1:4, 1:5, 1:8 driveway grades) included to anchor the prohibition

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `GOOGLE_API_KEY` | Yes | Gemini API key for LLM and embeddings |
| `GEMINI_API_KEY` | Optional | Duplicate — `GOOGLE_API_KEY` takes precedence if both set |
| `ENABLE_BACKEND_ACCESS_CONTROL` | Optional | Set `false` to disable Cognee multi-user isolation (required if data was ingested before v0.5.0) |

---

## Startup Checklist

When the server starts successfully you should see:

```
✅ App loaded successfully
✅ StripThoughtSignature middleware added
✅ CORS middleware added
Initializing Cognee Knowledge Graph Agent
  ✅ Table 'DocumentChunk_text' has 330 rows
Cognee Knowledge Graph initialization successful
Cognee.search patched: SQLite enum binding workaround active
Available modes: CHUNKS, CHUNKS_LEXICAL, ..., RAG_COMPLETION, ...
Default search mode: RAG_COMPLETION
Found root_agent in rag_agent.agent
```

If you see `source code string cannot contain null bytes` — the `agent.py` or `rag_query.py` file was saved with null bytes (caused by box-drawing Unicode characters in the instruction string). Fix with:

```bash
python3 -c "
files = ['rag_agent/agent.py', 'rag_agent/tools/rag_query.py']
for f in files:
    data = open(f,'rb').read()
    open(f,'wb').write(data.replace(b'\x00', b''))
    print(f'Cleaned: {f}')
"
```

---

## Contributing

This project is internal to DCIL /  . For issues with citation accuracy, check `chunk_logs/chunks_latest.json` to verify:
1. The correct zone's chunks appear in the top results
2. The `[SOURCE:]` header contains the correct chapter and page
3. The provenance search log line shows the correct top source

For retrieval issues, increase `DEFAULT_CHUNK_LIMIT` in `config.py` to retrieve a wider candidate set before re-ranking.

---

*  — Transforming the approval journey into a fast, accurate, and effortless experience.*
