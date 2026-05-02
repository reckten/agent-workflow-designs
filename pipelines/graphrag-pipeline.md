# Product GraphRAG

> **What this solves:** Product documentation is siloed, unstructured, and slow to query. This pipeline ingests raw markdown docs, extracts domain entities and relationships via LLM, and builds a traversable knowledge graph with hybrid semantic + graph retrieval. The exported context feeds directly into Claude Code as structured domain knowledge — closing the loop between documentation and test automation.

Knowledge Graph + Semantic Search over product documentation. Extracts domain entities and relationships from 49 markdown docs, builds a traversable graph, and provides hybrid retrieval for natural language queries and Claude Code context generation.

## Architecture

```
Raw .md files (mixed UTF-8/UTF-16LE)
  -> [normalize] -> data/normalized/     (all UTF-8, cleaned)
  -> [chunk]     -> data/chunks.json     (~478 chunks, 800 tokens each)
  -> [extract]   -> data/entities.json   (LLM entity extraction)
  -> [build]     -> data/graph.json      (NetworkX node-link format)
  -> [embed]     -> data/faiss.index     (local sentence-transformers)
  -> [query]     <- hybrid retrieval: FAISS similarity + graph neighborhood + LLM synthesis
  -> [export]    -> output/PRODUCT_DOMAIN.md (Claude-consumable context)
  -> [visualize] -> output/graph.html    (interactive browser graph)
```

## Setup

### Prerequisites

- Python 3.12+
- Any OpenAI-compatible API endpoint (OpenRouter recommended — free tier available at [openrouter.ai](https://openrouter.ai))

### Install

```bash
cd "./Product Docs"
pip install -r requirements.txt
```

### Configure

Edit `.env` in the project root. The pipeline is model-agnostic — swap in any OpenAI-compatible model:

```env
API_KEY=your-api-key-here
API_BASE_URL=https://openrouter.ai/api/v1
EXTRACTION_MODEL=google/gemma-4-31b-it:free
QUERY_MODEL=google/gemma-4-31b-it:free
EMBEDDING_MODEL=all-MiniLM-L6-v2
```

## Usage

### Full Pipeline

Run everything end-to-end (normalize, chunk, extract, build graph, embed):

```bash
python -m graphrag pipeline
```

### Individual Steps

```bash
python -m graphrag normalize       # Step 1: Normalize encoding to UTF-8
python -m graphrag chunk           # Step 2: Chunk documents (800 tok, 200 overlap)
python -m graphrag extract         # Step 3: Extract entities via LLM (resumable)
python -m graphrag build-graph     # Step 4: Build NetworkX knowledge graph
python -m graphrag embed           # Step 5: Generate embeddings + FAISS index
```

Extraction is resumable — if interrupted, re-run `python -m graphrag extract` and it picks up where it left off. Use `--no-resume` to start fresh.

### Query

```bash
# Hybrid query (semantic search + graph traversal + LLM synthesis)
python -m graphrag query "How are user roles configured?"

# Semantic search only (embeddings, no graph)
python -m graphrag query "How to process a refund?" --mode semantic

# Graph traversal only (entity relationships, no embeddings)
python -m graphrag query "What entities relate to Payment Methods?" --mode graph
```

### Explore the Graph

```bash
# Show graph statistics
python -m graphrag stats

# List all entities
python -m graphrag entities

# Filter by type
python -m graphrag entities --type PaymentMethod
python -m graphrag entities --type Promotion
python -m graphrag entities --type SecurityRole
python -m graphrag entities --type ProductType
python -m graphrag entities --type Workflow
```

### Visualize

Opens an interactive graph in your browser (drag, zoom, hover for details):

```bash
# Full entity graph (no chunk nodes)
python -m graphrag visualize --no-chunks

# Filter to a specific entity type
python -m graphrag visualize --no-chunks --entity-type PaymentMethod
python -m graphrag visualize --no-chunks --entity-type Promotion

# Filter to a module
python -m graphrag visualize --no-chunks --module ModuleA
python -m graphrag visualize --no-chunks --module ModuleB
```

### Export for Claude Code

```bash
# Full domain knowledge export
python -m graphrag export

# Module-specific export
python -m graphrag export-module ModuleA
python -m graphrag export-module ModuleB

# Single entity context
python -m graphrag export-entity "Payment Method"
```

The exported `output/PRODUCT_DOMAIN.md` can be referenced by Claude Code as domain context when writing tests.

## Entity Types

| Type | Description |
|------|-------------|
| ProductType | Purchasable items and product variants |
| ProductFamily | Groupings of related product types |
| Event / Performance | Events and their scheduled instances |
| PaymentMethod | Supported payment methods and tender types |
| InstallmentPlan / Contract | Recurring payment plans and agreements |
| Promotion | Discount and promotional configurations |
| SecurityRole / SecurityRight | Roles and permission assignments |
| Workstation / Location | Terminal and location topology |
| Organization / Person | Account holders and organizational entities |
| ConfigParam | System configuration parameters |
| Workflow | Multi-step business processes |

## File Structure

```
Product Docs/
  Module A/                     # Source docs
  Module B/                     # Source docs
  Module C/                     # Source docs
  Module D/                     # Source docs
  graphrag/                     # Python package
    __main__.py                 # CLI (click-based)
    config.py                   # .env loader
    normalize.py                # UTF-16LE -> UTF-8 normalization
    chunker.py                  # Markdown-aware chunking
    extractor.py                # LLM entity extraction (resumable)
    graph.py                    # NetworkX graph + query helpers
    embedder.py                 # sentence-transformers + FAISS
    retriever.py                # Hybrid retrieval
    query.py                    # LLM answer synthesis
    export.py                   # Claude context markdown generator
    prompts/                    # LLM prompt templates
  data/                         # Generated artifacts (gitignored)
    normalized/                 # UTF-8 normalized docs
    chunks.json                 # Document chunks
    entities.json               # Deduplicated entities + relationships
    graph.json                  # Serialized knowledge graph
    faiss.index                 # Vector search index
    embeddings.npz              # Chunk embeddings
    extraction_progress.json    # Resume checkpoint
  output/                       # Exports (gitignored)
    PRODUCT_DOMAIN.md           # Claude Code domain context
    graph.html                  # Interactive visualization
  .env                          # API keys (gitignored)
  requirements.txt
```

## Tech Stack

| Component | Technology |
|-----------|-----------|
| LLM (extraction + query) | Configurable via ENV — Gemma 4 31B via OpenRouter used in production (entity extraction is a structured task well-suited to smaller, faster models) |
| Embeddings | sentence-transformers all-MiniLM-L6-v2 (local, 384-dim) |
| Vector search | FAISS (cosine similarity) |
| Knowledge graph | NetworkX (directed graph, JSON serialized) |
| Visualization | pyvis (interactive HTML) |
| CLI | click + rich |
