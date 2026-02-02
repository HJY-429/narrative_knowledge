# Pipeline Parameters Documentation

## Overview

This document provides comprehensive information about input/output parameters for each component in the knowledge graph pipeline. The system uses a **daemon-orchestrator-tool** architecture where the daemon monitors for processing tasks, the orchestrator chains tools dynamically, and individual tools handle specific processing stages.

## Architecture Overview

**Main Processing Flow**:
```
Daemon (daemon.py)
  └─> Detects processing tasks (file uploads, memory data)
       └─> PipelineOrchestrator (orchestrator.py)
            └─> Chains tools based on pipeline type:
                 ├─ DocumentETLTool (document_etl_tool.py)
                 ├─ BlueprintGenerationTool (blueprint_generation_tool.py)
                 ├─ GraphBuildTool (graph_build_tool.py)
                 ├─ MemoryGraphBuildTool (memory_graph_build_tool.py)
                 └─ KnowledgeBuilderTool (knowledge_builder_tool.py)
```

### Standard Pipelines

**Knowledge Graph Pipelines:**
- `single_doc_existing_topic`: `["etl", "graph_build"]`
- `batch_doc_existing_topic`: `["etl", "blueprint_gen", "graph_build"]`
- `new_topic_batch`: `["etl", "blueprint_gen", "graph_build"]`

**Knowledge Building Pipeline:**
- `knowledge_build`: `["knowledge_builder"]`

**Memory Pipelines:**
- `memory_direct_graph`: `["memory_graph_build"]`
- `memory_single`: `["memory_graph_build"]`

## Pipeline Orchestrator (PipelineOrchestrator)

**Location**: `tools/orchestrator.py`

The orchestrator is the central execution engine that chains tools based on the pipeline type and context.

### Input Parameters

```python
context: Dict containing processing request data
  {
    # For knowledge graph processing:
    "target_type": "knowledge_graph",
    "process_strategy": {
        "pipeline": ["etl", "blueprint_gen", "graph_build"]  # Optional
    },
    "metadata": {
        "topic_name": str,              # Required
        "is_new_topic": bool,           # Optional
        "force_regenerate": bool,       # Optional
        "database_uri": str             # Optional
    },
    "files": [
        {
            "filename": str,
            "content_type": str,
            "title": str,               # Optional
            "author": str,              # Optional
            "metadata": dict            # Optional: file-specific metadata
        }
    ],
    "links": [str],                    # Document URLs matching files array
    
    # For personal memory processing:
    "target_type": "personal_memory",
    "metadata": {
        "user_id": str,                # Required
        "topic_name": str              # Optional
    },
    "input": [                         # Chat messages array
        {
            "role": "user" | "assistant",
            "content": str,
            "date": str                # ISO format timestamp
        }
    ]
  }
```

### Output

```python
{
    "success": bool,
    "data": {
        "results": {
            "etl": {...},              # ETL stage results
            "blueprint_gen": {...},    # Blueprint generation results
            "graph_build": {...}       # Graph building results
        },
        "pipeline": [...],             # List of executed tools
        "duration_seconds": float
    },
    "error_message": str,
    "execution_id": str
}
```

## Background Daemon (PipelineDaemon)

**Location**: `tools/daemon.py`

The daemon monitors for pending processing tasks and triggers orchestrator execution.

### Operation Modes

- **files mode**: Monitors `RawDataSource` entries with `status="uploaded"` and orchestrates document processing
- **memory mode**: Monitors `BackgroundTask` entries for memory data and orchestrates memory graph building

### Execution Flow

1. Scans database for pending tasks
2. Groups tasks by topic (for efficiency)
3. Constructs context from task data
4. Invokes `PipelineOrchestrator.execute_pipeline()`
5. Updates task status based on results

## 1. DocumentETLTool (ETL)

**Purpose**: Processes raw document files into structured SourceData

**Location**: `tools/document_etl_tool.py`

### Input Parameters

**Single File Mode:**
```python
{
    "file_path": str,           # Required: Path to document file
    "topic_name": str,          # Required: Topic name for grouping
    "metadata": dict,           # Optional: Custom metadata
    "force_regenerate": bool,   # Optional: Force reprocessing
    "link": str,                # Optional: Document URL
    "original_filename": str    # Optional: Original filename
}
```

**Batch File Mode:**
```python
{
    "files": [                  # Required: List of file objects
        {
            "path": str,        # Required: File path
            "filename": str,    # Optional: Original filename
            "link": str,        # Optional: Document URL
            "metadata": dict    # Optional: File-specific metadata
        }
    ],
    "topic_name": str,          # Required: Topic name for grouping
    "request_metadata": dict,   # Optional: Global metadata for all files
    "force_regenerate": bool    # Optional: Force reprocessing
}
```

### Output

```python
{
    "source_data_ids": [str],   # List of created SourceData IDs
    "results": [
        {
            "source_data_id": str,
            "content_hash": str,
            "content_size": int,
            "source_type": str,
            "reused_existing": bool,
            "status": str,
            "file_path": str
        }
    ],
    "batch_summary": {
        "total_files": int,
        "processed_files": int,
        "reused_files": int,
        "failed_files": int
    }
}
```

## 2. BlueprintGenerationTool

**Purpose**: Creates analysis blueprints by analyzing documents in a topic

**Location**: `tools/blueprint_generation_tool.py`

### Input Parameters

```python
{
    "topic_name": str,          # Required: Topic name to generate blueprint
    "source_data_ids": [str],   # Optional: Specific source data IDs to include
    "force_regenerate": bool,   # Optional: Force regeneration
    "llm_client": object,       # Required: LLM client instance
    "embedding_func": object    # Optional: Embedding function (uses default if not provided)
}
```

### Output

```python
{
    "blueprint_id": str,                # ID of created/updated AnalysisBlueprint
    "source_data_version_hash": str,    # Hash of contributing source data versions
    "contributing_source_data_count": int,
    "blueprint_summary": {
        "canonical_entities_count": int,
        "key_patterns_count": int,
        "global_timeline_events": int,
        "processing_instructions_length": int,
        "cognitive_maps_used": int
    },
    "reused_existing": bool             # Whether existing blueprint was reused
}
```

## 3. GraphBuildTool

**Purpose**: Extracts knowledge from documents and builds the knowledge graph

**Location**: `tools/graph_build_tool.py`

### Input Parameters

**Mode 1: Single Document Processing**
```python
{
    "source_data_id": str,      # Required: Single SourceData ID
    "blueprint_id": str,        # Required: ID of AnalysisBlueprint
    "force_regenerate": bool,   # Optional: Force regeneration
    "llm_client": object,       # Required: LLM client instance
    "embedding_func": object    # Optional: Embedding function
}
```

**Mode 2: Batch Documents Processing**
```python
{
    "source_data_ids": [str],   # Required: Array of SourceData IDs
    "blueprint_id": str,        # Required: ID of AnalysisBlueprint
    "force_regenerate": bool,   # Optional: Force regeneration
    "llm_client": object,       # Required: LLM client instance
    "embedding_func": object    # Optional: Embedding function
}
```

**Mode 3: Topic-Based Processing**
```python
{
    "topic_name": str,          # Required: Topic name
    "source_data_ids": [str],   # Optional: Specific sources (uses all if not provided)
    "force_regenerate": bool,   # Optional: Force regeneration
    "llm_client": object,       # Required: LLM client instance
    "embedding_func": object    # Optional: Embedding function
}
```

### Output

```python
{
    "topic_name": str,
    "blueprint_id": str,
    "processed_count": int,
    "failed_count": int,
    "total_entities_created": int,
    "total_relationships_created": int,
    "total_triplets_extracted": int,
    "results": [
        {
            "source_data_id": str,
            "status": str,
            "entities_created": int,
            "relationships_created": int,
            "triplets_extracted": int,
            "error": str  # Only if status == "failed"
        }
    ]
}
```

## 4. MemoryGraphBuildTool

**Purpose**: Processes chat messages and builds personal knowledge graphs

**Location**: `tools/memory_graph_build_tool.py`

### Input Parameters

```python
{
    "chat_messages": [          # Required: List of chat messages
        {
            "content": str,              # Required: Message content
            "role": "user" | "assistant",  # Required: Message role
            "date": str                  # Optional: ISO format timestamp
        }
    ],
    "user_id": str,             # Required: User identifier
    "source_id": str,           # Optional: Reprocess existing source data
    "topic_name": str,          # Optional: Topic name (auto-generated if not provided)
    "force_regenerate": bool,   # Optional: Force reprocessing
    "llm_client": object,       # Required: LLM client instance
    "embedding_func": object    # Optional: Embedding function
}
```

### Output

```python
{
    "user_id": str,
    "topic_name": str,
    "source_data_id": str,
    "entities_created": int,
    "relationships_created": int,
    "triplets_extracted": int,
    "status": str
}
```

## 5. KnowledgeBuilderTool

**Purpose**: Standalone knowledge extraction and block creation

**Location**: `tools/knowledge_builder_tool.py`

### Input Parameters
```python
{
    "files": [str],             # Required: List of file paths
    "attributes": dict,         # Optional: Custom attributes for all blocks
    "topic_name": str,          # Optional: Topic name
    "llm_client": object,       # Required: LLM client instance
    "embedding_func": object    # Optional: Embedding function
}
```

### Output

```python
{
    "source_data_ids": [str],   # IDs of created SourceData entries
    "knowledge_blocks_created": int,
    "topic_name": str,
    "results": [
        {
            "source_data_id": str,
            "block_id": str,
            "knowledge_type": str,
            "content_hash": str,
            "status": str
        }
    ]
}
```

## Parameter Flow Between Components

### 1. Knowledge Graph Pipeline (Typical Flow)

```
Daemon detects file uploads
  └─> DocumentETLTool
      ├─ Input: raw files + topic_name
      └─ Output: source_data_ids[]
          └─> BlueprintGenerationTool
              ├─ Input: topic_name + source_data_ids
              └─ Output: blueprint_id
                  └─> GraphBuildTool
                      ├─ Input: source_data_ids + blueprint_id
                      └─ Output: graph building results
```

### 2. Memory Pipeline Flow

```
Daemon detects memory data
  └─> MemoryGraphBuildTool
      ├─ Input: chat_messages + user_id
      └─ Output: memory graph building results
```

### 3. Knowledge Builder Flow (Can combine with 1.)

```
Direct tool execution (via orchestrator)
  └─> KnowledgeBuilderTool
      ├─ Input: files + topic_name
      └─ Output: knowledge blocks
```

## Processing Scenarios

### 1. Single Document to Existing Topic
- **Pipeline**: `["etl", "graph_build"]`
- **ETL**: Processes single file
- **BlueprintGen**: Updates/uses existing blueprint
- **GraphBuild**: Builds graph using blueprint

### 2. Batch Documents to Existing Topic
- **Pipeline**: `["etl", "blueprint_gen", "graph_build"]`
- **ETL**: Processes multiple files in parallel
- **BlueprintGen**: Updates existing blueprint with new documents
- **GraphBuild**: Builds graph for all documents

### 3. New Topic with Batch Documents
- **Pipeline**: `["etl", "blueprint_gen", "graph_build"]`
- **ETL**: Processes multiple files in parallel
- **BlueprintGen**: Creates new blueprint from scratch
- **GraphBuild**: Builds graph for new topic

### 4. Personal Memory Processing
- **Pipeline**: `["memory_graph_build"]`
- **MemoryGraphBuild**: Processes chat messages directly

### 5. Knowledge Blocks
- **Pipeline**: `["knowledge_builder"]`
- **KnowledgeBuilder**: Extracts and stores knowledge blocks

## Common Parameters Across All Tools

| Parameter | Type | Description | Required |
|-----------|------|-------------|----------|
| `llm_client` | object | LLM client instance for processing | Yes (except ETL) |
| `embedding_func` | object | Embedding function for vector operations | Yes (except ETL) |
| `force_regenerate` | bool | Force regeneration even if already processed | No |
| `topic_name` | string | Topic name for grouping documents | Context-dependent |

## Tool Result Format

All tools return standardized `ToolResult` objects:

```python
{
    "success": bool,
    "data": dict,               # Results on success
    "error_message": str,       # Error details on failure
    "execution_id": str,        # Unique execution identifier
    "duration_seconds": float   # Tool execution time
}
```

## Error Handling

The orchestrator and daemon implement robust error handling:
- Individual tool failures don't stop the entire pipeline
- Errors are logged and reported in results
- Task status is updated to reflect success/failure
- Retry logic can be configured per tool