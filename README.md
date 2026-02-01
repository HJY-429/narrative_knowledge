# Narrative Knowledge System

A comprehensive, AI-powered knowledge extraction and graph building system that processes documents to construct semantic knowledge graphs. This system uses large language models (LLMs) to extract entities, relationships, and narrative insights from various document types, enabling advanced knowledge representation and querying.

## Overview

The Narrative Knowledge System provides an intelligent pipeline for transforming unstructured documents (PDFs, Markdown, text, SQL) into rich, interconnected knowledge graphs. It combines document extraction, semantic analysis, blueprint generation, and knowledge graph construction into a flexible, tool-based architecture.

### Key Capabilities

- **Multi-format Document Processing**: Support for PDF, Markdown, TXT, and SQL files
- **Semantic Analysis**: Extract entities and relationships using multiple LLM providers
- **Knowledge Graph Construction**: Build interconnected knowledge graphs with quality guarantees
- **Topic-Based Organization**: Organize documents and their extracted knowledge by topic
- **REST API**: Full REST API with batch processing support
- **Flexible LLM Integration**: Support for OpenAI, Gemini, Ollama, Bedrock, and custom OpenAI-compatible providers
- **Batch Processing**: Process multiple documents in parallel with configurable worker counts
- **Quality Standards**: Built-in entity and relationship quality validation

## Architecture

The system is built on a **Tool-Based Pipeline Architecture** that decomposes knowledge extraction into independent, composable tools:

### Core Components

#### 1. **Tools** (Smallest Unit of Work)
Tools are stateless, independent functions that perform single, well-defined tasks:

- **DocumentETLTool** (`tools/document_etl_tool.py`)
  - Processes raw document files into structured SourceData
  - Supports single and batch file processing
  - Extracts text, creates initial entity mappings
  
- **BlueprintGenerationTool** (`tools/blueprint_generation_tool.py`)
  - Generates analysis blueprints from documents
  - Identifies shared themes and cross-document insights
  - Provides context for consistent knowledge extraction
  
- **GraphBuildTool** (`tools/graph_build_tool.py`)
  - Extracts knowledge from documents using blueprints
  - Creates entities and relationships in the knowledge graph
  - Applies quality standards during construction

- **KnowledgeBuilderTool** (`tools/knowledge_builder_tool.py`)
  - Standalone knowledge extraction and storage
  - Compatible with other tools
  
- **MemoryGraphBuildTool** (`tools/memory_graph_build_tool.py`)
  - Builds knowledge graphs from memory/narrative chat messages

#### 2. **Pipeline Orchestrator** (`tools/orchestrator.py`)
Dynamically chains tools into pipelines based on processing scenarios:

**Available Pipelines:**
- `single_doc_existing_topic`: ETL → Graph Build
- `batch_doc_existing_topic`: ETL → Blueprint Gen → Graph Build
- `new_topic_batch`: ETL → Blueprint Gen → Graph Build
- `text_to_graph`: Direct graph building from text
- `knowledge_build`: Create knowledge blocks and extraction
- `memory_direct_graph`: Direct memory graph building
- `memory_single`: Single memory processing

#### 3. **Knowledge Graph Components** (`knowledge_graph/`)

- **GraphBuilder** (`knowledge_graph/graph_builder.py`)
  - Core logic for constructing knowledge graphs
  - Manages cognitive maps and analysis blueprints
  - Parallel processing of documents
  
- **NarrativeKnowledgeGraphBuilder** (`knowledge_graph/graph.py`)
  - Low-level graph construction and entity management
  - Triplet extraction and graph enrichment
  
- **KnowledgeBuilder** (`knowledge_graph/knowledge.py`)
  - High-level document processing workflow
  - Content parsing and knowledge block creation
  
- **DocumentCognitiveMapGenerator** (`knowledge_graph/congnitive_map.py`)
  - Generates cognitive maps for documents
  - Captures document structure and key concepts

- **Data Models** (`knowledge_graph/models.py`)
  - SQLAlchemy ORM models for all core entities
  - ContentStore, RawDataSource, SourceData, KnowledgeBlock
  - Entity, Relationship, AnalysisBlueprint models
  - GraphBuild tracking and BackgroundTask management
  
- **Parsers** (`knowledge_graph/parser/`)
  - Factory pattern for content parsing
  - Markdown, PDF, and generic parsers
  - Extracts text blocks and structure

#### 4. **LLM Integration** (`llm/`)

Pluggable LLM provider architecture supporting:

- **OpenAI** (`llm/providers/openai.py`)
- **Google Gemini** (`llm/providers/gemini.py`)
- **Ollama** (`llm/providers/ollama.py`) - Local LLM support
- **AWS Bedrock** (`llm/providers/bedrock.py`)
- **OpenAI-Compatible** (`llm/providers/openai_like.py`)

**LLMInterface** (`llm/factory.py`) provides unified access with:
- Text generation with system prompts
- Streaming response support
- Retry logic with exponential backoff

#### 5. **REST API** (`api/`)

FastAPI-based REST service with three main routers:

- **Knowledge Router** (`api/knowledge.py`)
  - Upload documents
  - Trigger processing
  - Query topics and knowledge graphs
  
- **Memory Router** (`api/memory.py`)
  - Memory-based knowledge operations
  
- **Ingest Router** (`api/ingest.py`)
  - Data ingestion endpoints

#### 6. **Database & Settings** (`setting/`)

- **Database Configuration** (`setting/db.py`)
  - SQLAlchemy ORM setup
  - Support for multiple database backends
  - Connection pooling configuration
  
- **Settings** (`setting/base.py`)
  - LLM provider and model configuration
  - Embedding model settings
  - Database URI and pool size
  - Token limits

#### 7. **Utilities** (`utils/`)

- **File Operations** (`utils/file.py`)
- **JSON Utilities** (`utils/json_utils.py`)
- **Token Management** (`utils/token.py`) - Token counting and encoding
- **UUID Utilities** (`utils/uuid_utils.py`)

## Installation

### Prerequisites

- Python 3.12 or higher
- Poetry package manager
- (Optional) Local LLM server (Ollama) for local inference
- Database system

### Setup Steps

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd narrative_knowledge
   ```

2. **Install dependencies**
   ```bash
   poetry install
   ```

3. **Configure environment variables**
   ```bash
   cp .env.example .env
   # Edit .env with your settings
   ```

4. **Environment Configuration** (`.env`)
   ```bash
   # LLM Settings
   LLM_PROVIDER=ollama              # Options: openai, gemini, ollama, bedrock, openai_like
   LLM_MODEL=qwen3:1.7b             # Model identifier
   
   # Optional: For specific providers
   OPENAI_API_KEY=your-key
   GOOGLE_API_KEY=your-key
   OLLAMA_BASE_URL=http://127.0.0.1:11434
   
   # Embedding
   EMBEDDING_MODEL=hf.co/Qwen/Qwen3-Embedding-8B-GGUF:Q8_0
   EMBEDDING_BASE_URL=http://127.0.0.1:11434/v1
   
   # Database
   DATABASE_URI=mysql+pymysql://user:password@localhost:3306/knowledge_graph
   SESSION_POOL_SIZE=40
   ```

## Quick Start

### Start the API Server

```bash
# Method 1: Using poetry
poetry run python -m api.main

# Method 2: Using uvicorn (recommended for production)
poetry run uvicorn api.main:app --host 0.0.0.0 --port 8000 --workers 2
```

The API will be available at `http://localhost:8000`

### View API Documentation

- **Swagger UI**: `http://localhost:8000/docs`
- **ReDoc**: `http://localhost:8000/redoc`

### Upload Documents

```bash
# Single file upload
curl -X POST "http://localhost:8000/api/v1/knowledge/upload" \
  -F "files=@document.pdf" \
  -F "links=https://example.com/document" \
  -F "topic_name=my-topic"

# Batch upload
curl -X POST "http://localhost:8000/api/v1/knowledge/upload" \
  -F "files=@doc1.pdf" \
  -F "files=@doc2.md" \
  -F "links=https://example.com/doc1" \
  -F "links=https://example.com/doc2" \
  -F "topic_name=my-topic"
```

### Trigger Processing

```bash
curl -X POST "http://localhost:8000/api/v1/knowledge/trigger-processing" \
  -F "topic_name=my-topic"
```

### List Topics

```bash
curl "http://localhost:8000/api/v1/knowledge/topics"
```

## API Reference

### Document Upload

**POST** `/api/v1/knowledge/upload`

Upload and process documents for knowledge graph building.

**Parameters:**
- `files` (multipart): One or more document files
- `links` (array): Document URLs (must match number of files)
- `topic_name` (string): Topic for grouping documents
- `database_uri` (string, optional): Database connection string

**Supported File Types:**
- PDF (`.pdf`)
- Markdown (`.md`)
- Text (`.txt`)
- SQL (`.sql`)

**File Limits:**
- Max total batch size: 30MB
- Max individual file: 30MB

### Trigger Processing

**POST** `/api/v1/knowledge/trigger-processing`

Manually trigger knowledge graph building for a topic.

**Parameters:**
- `topic_name` (string): Topic to process
- `database_uri` (string, optional): Database URI filter

### List Topics

**GET** `/api/v1/knowledge/topics`

Retrieve all topics and their processing status.

**Query Parameters:**
- `database_uri` (string, optional): Filter by database

## Enhanced Data Processing Pipeline

### Overview

The `/api/v1/save` endpoint provides a unified interface for document upload and JSON input with configurable processing pipelines and background task execution.

### Main Endpoints

**POST** `/api/v1/save` - Submit files or JSON data for processing

**GET** `/api/v1/tasks/{task_id}` - Check status of a background task

### File Upload Processing

#### Build Knowledge Graphs
```bash
curl -X POST "http://localhost:8000/api/v1/save" \
  -F "files=@document.pdf" \
  -F 'links=["https://example.com/doc"]' \
  -F 'metadata={"topic_name":"my_topic","force_regenerate":true}' \
  -F "target_type=knowledge_graph" \
  -F 'process_strategy={"pipeline":["etl","blueprint_gen","graph_build"]}'
```

**Parameters:**
- `files`: Document file(s)
- `links`: Document URLs (optional)
- `metadata`: JSON with `topic_name` (required), `force_regenerate` (optional)
- `target_type`: `"knowledge_graph"`
- `process_strategy`: JSON with `pipeline` array (optional)

#### Build Knowledge Blocks Only
```bash
curl -X POST "http://localhost:8000/api/v1/save" \
  -F "files=@document.md" \
  -F 'metadata={"topic_name":"my_topic"}' \
  -F "target_type=knowledge_build"
```

**Parameters:**
- `files`: Document file(s)
- `metadata`: JSON with `topic_name` (required)
- `target_type`: `"knowledge_build"`

### JSON Input Processing

#### Memory/Chat History
```bash
curl -X POST "http://localhost:8000/api/v1/save" \
  -H "Content-Type: application/json" \
  -d '{
    "input": [
      {"role":"user","content":"Question here","date":"2025-08-07T16:18:04.282415"},
      {"role":"assistant","content":"Answer here","date":"2025-08-07T16:18:13.282415"}
    ],
    "metadata": {"user_id":"user123"},
    "target_type": "personal_memory"
  }'
```

**Parameters:**
- `input`: Chat history array or JSON data
- `metadata`: JSON with `user_id` (required)
- `target_type`: `"personal_memory"`

### Task Status Checking

```bash
curl "http://localhost:8000/api/v1/tasks/task_id_here"
```

Returns `status` (processing/completed) and results when ready.

### Available Pipeline Types

**Knowledge Graph:**
- `single_doc_existing_topic`: `["etl", "graph_build"]`
- `batch_doc_existing_topic`: `["etl", "blueprint_gen", "graph_build"]`
- `new_topic_batch`: `["etl", "blueprint_gen", "graph_build"]`

**Memory:**
- `memory_direct_graph`: Direct memory processing with graph building

### Background Processing

**Daemon:** `python tools/daemon.py [--mode files|memory] [--interval seconds]`

Automatically processes pending documents or memory data in the background.

## Processing Pipeline

### Single Document to Existing Topic

```
Raw Document
    ↓
[ETL Tool] → Structured SourceData
    ↓
[Graph Build Tool] → Knowledge Graph
```

### Batch Documents to Existing Topic

```
Raw Documents (batch)
    ↓
[ETL Tool] → SourceData (parallel)
    ↓
[Blueprint Gen Tool] → Analysis Blueprint (updated)
    ↓
[Graph Build Tool] → Knowledge Graph (parallel)
```

### New Topic with Batch Documents

```
Raw Documents (batch)
    ↓
[ETL Tool] → SourceData (parallel)
    ↓
[Blueprint Gen Tool] → Analysis Blueprint (new)
    ↓
[Graph Build Tool] → Knowledge Graph (parallel)
```

## Core Concepts

### Topic
A logical grouping for related documents. Serves as a context boundary for knowledge extraction and blueprint generation.

### Analysis Blueprint
Shared context generated from all documents in a topic. Contains:
- Key entities and themes
- Cross-document relationships
- Domain-specific context

Created once per topic and updated when new batch documents are added. Used to ensure consistent, high-quality knowledge extraction.

### SourceData
Structured representation of a processed document containing:
- Extracted text blocks
- Content metadata
- Initial entity mappings
- Link to original document

### Knowledge Graph
Network of entities and relationships extracted from documents with:
- Entity deduplication
- Relationship consolidation
- Directional connections
- Quality validation

### Entity
Atomic nodes in the knowledge graph representing real-world concepts. Each entity has:
- Unique identifier and canonical name
- Description and attributes (entity_type, domain, aliases)
- Vector embedding for semantic similarity
- Relationships to other entities

### Relationship
Directional edges connecting entities, representing semantic connections. Contains:
- Source entity reference
- Target entity reference
- Relationship description
- Vector embedding for semantic matching
- Attributes and metadata

### Knowledge Block
Atomic knowledge unit extracted from source documents. Represents smaller, granular pieces of knowledge:
- Multiple types: QA, paragraph, synopsis, code, chat summary, etc.
- Content with optional vector embedding
- Hash-based deduplication to avoid duplicates
- Links to source documents via BlockSourceMapping

### ContentStore
Deduplicated content storage system using SHA-256 hashing:
- Prevents redundant storage of identical content
- Tracks content size and type
- Stores reference to original document links
- Enables efficient content reuse across multiple SourceData entries

### RawDataSource
Represents uploaded files before ETL processing:
- Tracks file path and original filename
- Maintains processing status (uploaded → etl_pending → etl_completed)
- Stores file hash for integrity verification
- Captures custom metadata from upload request

### SourceData
Processed document state after ETL extraction:
- References both RawDataSource (original file) and ContentStore (deduplicated content)
- Tracks graph building status (created → graph_pending → graph_completed)
- Links to original document URL
- Contains document attributes and metadata

### DocumentSummary
Topic-focused summary of individual documents for efficient processing:
- Captures key entities and themes
- Stores business context
- Enables fast blueprint generation by avoiding full document re-analysis
- One summary per document-topic combination

### DocumentCognitiveMap
Internal representation of document structure and key concepts:
- Captures document organization and hierarchy
- Identifies main topics and their relationships
- Used during graph building to enhance extraction quality

## Quality Standards

The system enforces comprehensive quality standards for both entities and relationships:

### Entity Quality
- **Non-redundant**: No duplicate entities
- **Precise**: Clear, unambiguous definitions
- **Accurate**: Factually correct representations
- **Coherent**: Logically consistent knowledge
- **Efficiently Connected**: Optimal relationship pathways

**Entity Attributes:**
- `entity_type`: Classification (Person, Concept, Technology, etc.)
- `domain`: Standardized domain label
- `searchable_keywords`: Array of domain terms and synonyms
- `aliases`: Alternative names or abbreviations

### Relationship Quality
- Clear semantics describing connections
- Verified factual accuracy
- Consistent directionality
- No redundant relationships

See `knowledge_graph_quality_standard.md` for detailed specifications.

## Configuration

### LLM Provider Selection

Set via `LLM_PROVIDER` environment variable:

```python
from llm.factory import LLMInterface

# OpenAI
llm = LLMInterface("openai", model="gpt-4o")

# Local Ollama
llm = LLMInterface("ollama", model="qwen3:1.7b")

# Google Gemini
llm = LLMInterface("gemini", model="gemini-2.0-flash")

# AWS Bedrock
llm = LLMInterface("bedrock", model="anthropic.claude-3-sonnet")
```

### Embedding Model

Configure in `.env`:
```bash
EMBEDDING_MODEL=hf.co/Qwen/Qwen3-Embedding-8B-GGUF:Q8_0
EMBEDDING_BASE_URL=http://127.0.0.1:11434/v1
```

### Database Configuration

Supports multiple database backends via SQLAlchemy:

```bash
# MySQL/TiDB
DATABASE_URI=mysql+pymysql://user:password@localhost:3306/db

# PostgreSQL
DATABASE_URI=postgresql://user:password@localhost:5432/db

# SQLite (development)
DATABASE_URI=sqlite:///knowledge.db
```

## Development

### Project Structure

```
narrative_knowledge/
├── api/                          # REST API
│   ├── main.py                   # FastAPI app
│   ├── knowledge.py              # Knowledge endpoints
│   ├── memory.py                 # Memory endpoints
│   └── ingest.py                 # Ingest endpoints
├── tools/                        # Tool implementations
│   ├── base.py                   # Base tool class
│   ├── orchestrator.py           # Pipeline orchestrator
│   ├── document_etl_tool.py     # ETL tool
│   ├── blueprint_generation_tool.py
│   ├── graph_build_tool.py       # Graph building tool
│   ├── knowledge_builder_tool.py
│   └── memory_graph_build_tool.py
├── knowledge_graph/              # Graph construction
│   ├── graph_builder.py          # High-level builder
│   ├── graph.py                  # Low-level graph ops
│   ├── knowledge.py              # Knowledge extraction
│   ├── models.py                 # Data models
│   ├── congnitive_map.py         # Cognitive mapping
│   ├── query.py                  # Graph querying
│   ├── summarizer.py             # Text summarization
│   └── parser/                   # Document parsers
├── llm/                          # LLM integration
│   ├── base.py                   # Base provider
│   ├── factory.py                # LLM factory
│   ├── embedding.py              # Embedding functions
│   └── providers/                # LLM implementations
├── setting/                      # Configuration
│   ├── base.py                   # Settings
│   └── db.py                     # Database setup
├── utils/                        # Utilities
│   ├── file.py
│   ├── json_utils.py
│   ├── token.py
│   └── uuid_utils.py
└── test/                         # Test suite
```


### Adding New LLM Providers

1. Create a new provider class in `llm/providers/`
2. Inherit from `BaseLLMProvider`
3. Implement `generate()` and `generate_stream()` methods
4. Register in `llm/factory.py`

### Adding Custom Tools

1. Create a tool class inheriting from `BaseTool`
2. Implement required properties and `execute()` method
3. Register in tool registry
4. Add to pipeline configurations in `tools/orchestrator.py`

## Performance Considerations

- **Parallel Processing**: Documents processed in parallel using thread pools
- **Worker Count**: Configurable (default: 3 workers)
- **Batch Processing**: Optimal for batch operations with blueprint generation
- **Token Limits**: Configurable max prompt tokens (default: 40960)
- **Connection Pooling**: Configurable database connection pool (default: 40)

## Troubleshooting

### Common Issues

**LLM Connection Fails**
- Verify LLM provider is running and accessible
- Check `OLLAMA_BASE_URL` or API key settings
- Test with: `curl http://ollama:11434/api/generate`

**Database Connection Error**
- Verify `DATABASE_URI` is correct
- Ensure database server is running
- Check network connectivity

**Out of Memory**
- Reduce `worker_count` for graph building
- Process smaller document batches
- Increase `SESSION_POOL_SIZE` for database

**Document Processing Fails**
- Check document format is supported
- Verify file isn't corrupted
- Review logs for specific errors

## Documentation

- [Pipeline Design](pipeline_design.md) - Architecture and design decisions
- [Pipeline Parameters](tools/PIPELINE_PARAMETERS.md) - API parameter documentation
- [Quality Standards](knowledge_graph_quality_standard.md) - Quality metrics and standards
- [API Documentation](api/README.md) - Detailed API guide

## Dependencies

Key dependencies:
- **FastAPI**: REST framework
- **SQLAlchemy**: ORM for database
- **Pydantic**: Data validation
- **PyMuPDF**: PDF processing
- **OpenAI/Google/Ollama**: LLM providers
- **TiDB Vector**: Vector database support
- **Boto3**: AWS services
- **python-dotenv**: Environment configuration

See `pyproject.toml` for complete dependency list.
