# Pommel Architecture (C4 Model)

This document describes Pommel's architecture using the [C4 model](https://c4model.com/) notation. Diagrams are in Mermaid format and render in GitHub, Obsidian, and most markdown viewers.

## Overview

Pommel is a **local-first semantic code search system** that:
- Maintains an always-current vector database of code embeddings
- Reduces AI agent context window consumption by 60-80%
- Enables natural language queries against codebases

---

## C1: System Context

Shows Pommel's relationship with external actors and systems.

```mermaid
graph TB
    subgraph External
        User["👤 Developer / AI Agent<br/>(Claude Code, Cursor, etc.)"]
        Ollama["🤖 Ollama<br/>(Local LLM Runtime)<br/>localhost:11434"]
        FileSystem["📁 File System<br/>(Source Code)"]
    end

    subgraph Pommel System
        Pommel["⚙️ Pommel<br/>Semantic Code Search System<br/>Maintains vector DB of code embeddings"]
    end

    User -->|"pm search 'query'"| Pommel
    User -->|"pm init / start / status"| Pommel
    Pommel -->|"Generate embeddings<br/>POST /api/embed"| Ollama
    FileSystem -->|"fsnotify events<br/>(create/modify/delete)"| Pommel
    Pommel -->|"Read source files"| FileSystem
    Pommel -->|"Ranked search results<br/>(file:line + score)"| User
```

### Actors & Systems

| Element | Description |
|---------|-------------|
| **Developer / AI Agent** | Primary users - humans via CLI or AI coding assistants |
| **Ollama** | Local LLM runtime hosting jina-embeddings-v2-base-code model |
| **File System** | Source code being indexed and searched |
| **Pommel** | The semantic search system maintaining embeddings |

---

## C2: Container Diagram

Shows the main deployable units and their interactions.

```mermaid
graph TB
    subgraph External
        User["👤 Developer / AI Agent"]
        Ollama["🤖 Ollama<br/>jina-embeddings-v2-base-code"]
        FS["📁 File System"]
    end

    subgraph "Pommel System"
        CLI["📦 pm (CLI)<br/>─────────────<br/>Go Binary<br/>Cobra + Viper<br/>─────────────<br/>Commands: search, init,<br/>start, stop, status, reindex"]

        Daemon["📦 pommeld (Daemon)<br/>─────────────<br/>Go HTTP Server<br/>─────────────<br/>• File watcher (fsnotify)<br/>• Indexer (Tree-sitter)<br/>• Search service<br/>• API endpoints"]

        DB["📦 SQLite Database<br/>─────────────<br/>.pommel/index.db<br/>─────────────<br/>• chunks table<br/>• chunk_embeddings (vec)<br/>• chunks_fts (FTS5)<br/>• files metadata"]
    end

    User -->|"CLI commands"| CLI
    CLI -->|"HTTP<br/>GET/POST"| Daemon
    Daemon -->|"SQL + sqlite-vec"| DB
    Daemon -->|"POST /api/embed<br/>768-dim vectors"| Ollama
    FS -->|"fsnotify events"| Daemon
    Daemon -->|"Read files"| FS
    CLI -->|"JSON results"| User
```

### Containers

| Container | Technology | Responsibility |
|-----------|------------|----------------|
| **pm (CLI)** | Go + Cobra | User interface, command parsing, daemon communication |
| **pommeld (Daemon)** | Go + http.Server | File watching, indexing, embedding, search serving |
| **SQLite Database** | SQLite + sqlite-vec + FTS5 | Persistent storage of chunks, vectors, and full-text index |

---

## C3: Component Diagram - Daemon

Shows internal components of the pommeld daemon.

```mermaid
graph TB
    subgraph "pommeld (Daemon Container)"
        subgraph "HTTP Layer"
            Router["Router<br/>http.ServeMux"]
            Handlers["API Handlers<br/>health, status,<br/>search, reindex"]
        end

        subgraph "Core Services"
            Watcher["Watcher<br/>───────<br/>fsnotify<br/>debouncing<br/>pattern filtering"]

            Indexer["Indexer<br/>───────<br/>orchestrates<br/>chunking + embedding"]

            SearchSvc["Search Service<br/>───────<br/>vector search<br/>hybrid (RRF)<br/>filtering"]

            State["State Manager<br/>───────<br/>PID tracking<br/>daemon lifecycle"]
        end

        subgraph "Processing"
            ChunkerReg["Chunker Registry<br/>───────<br/>routes by language"]

            GenericChunker["Generic Chunker<br/>───────<br/>Tree-sitter AST<br/>Go, Python, JS/TS,<br/>Java, C#"]

            FallbackChunker["Fallback Chunker<br/>───────<br/>line-based<br/>unknown languages"]
        end

        subgraph "Embedding"
            CachedEmbed["Cached Embedder<br/>───────<br/>LRU cache<br/>hit/miss metrics"]

            OllamaClient["Ollama Client<br/>───────<br/>HTTP client<br/>batch embedding"]
        end

        subgraph "Persistence"
            DBLayer["DB Layer<br/>───────<br/>chunks, vectors,<br/>FTS, files"]
        end
    end

    subgraph External
        CLI["pm CLI"]
        Ollama["Ollama"]
        FS["File System"]
        SQLite["SQLite + vec"]
    end

    CLI --> Router
    Router --> Handlers
    Handlers --> SearchSvc
    Handlers --> Indexer

    FS --> Watcher
    Watcher --> Indexer

    Indexer --> ChunkerReg
    ChunkerReg --> GenericChunker
    ChunkerReg --> FallbackChunker

    Indexer --> CachedEmbed
    CachedEmbed --> OllamaClient
    OllamaClient --> Ollama

    SearchSvc --> CachedEmbed
    SearchSvc --> DBLayer
    Indexer --> DBLayer
    DBLayer --> SQLite
```

### Daemon Components

| Component | Package | Responsibility |
|-----------|---------|----------------|
| **Router** | `internal/daemon` | HTTP request routing |
| **API Handlers** | `internal/api` | Request/response handling for health, status, search, reindex |
| **Watcher** | `internal/daemon/watcher.go` | fsnotify file watching with debouncing and .pommelignore filtering |
| **Indexer** | `internal/daemon/indexer.go` | Orchestrates chunking and embedding for file changes |
| **Search Service** | `internal/search` | Vector similarity search, hybrid RRF ranking, filtering |
| **State Manager** | `internal/daemon/state.go` | PID file management, daemon lifecycle |
| **Chunker Registry** | `internal/chunker` | Routes files to appropriate chunker by language |
| **Generic Chunker** | `internal/chunker` | Tree-sitter AST parsing for supported languages |
| **Fallback Chunker** | `internal/chunker` | Line-based chunking for unknown languages |
| **Cached Embedder** | `internal/embedder/cache.go` | LRU cache wrapper for embeddings |
| **Ollama Client** | `internal/embedder/ollama.go` | HTTP client for Ollama embedding API |
| **DB Layer** | `internal/db` | SQLite operations for chunks, vectors, FTS |

---

## C3: Component Diagram - CLI

Shows internal components of the pm CLI.

```mermaid
graph TB
    subgraph "pm (CLI Container)"
        RootCmd["Root Command<br/>───────<br/>Cobra rootCmd<br/>global flags"]

        subgraph "Commands"
            InitCmd["init<br/>───────<br/>--auto --claude<br/>--start"]
            SearchCmd["search<br/>───────<br/>--limit --level<br/>--json --path"]
            StartCmd["start<br/>───────<br/>spawn daemon"]
            StopCmd["stop<br/>───────<br/>kill daemon"]
            StatusCmd["status<br/>───────<br/>index stats"]
            ReindexCmd["reindex<br/>───────<br/>force full"]
            ConfigCmd["config<br/>───────<br/>view/edit"]
        end

        Client["HTTP Client<br/>───────<br/>Health()<br/>Status()<br/>Search()<br/>Reindex()"]

        Output["Output Formatter<br/>───────<br/>table / JSON<br/>verbose mode"]

        ConfigLoader["Config Loader<br/>───────<br/>.pommel/config.yaml"]
    end

    subgraph External
        User["User"]
        Daemon["pommeld"]
    end

    User --> RootCmd
    RootCmd --> InitCmd
    RootCmd --> SearchCmd
    RootCmd --> StartCmd
    RootCmd --> StopCmd
    RootCmd --> StatusCmd
    RootCmd --> ReindexCmd
    RootCmd --> ConfigCmd

    SearchCmd --> Client
    StatusCmd --> Client
    ReindexCmd --> Client

    Client --> Daemon

    InitCmd --> ConfigLoader
    SearchCmd --> Output
    StatusCmd --> Output

    Output --> User
```

### CLI Components

| Component | File | Responsibility |
|-----------|------|----------------|
| **Root Command** | `internal/cli/root.go` | Cobra root, global flags, version |
| **init** | `internal/cli/init.go` | Project initialization, config generation |
| **search** | `internal/cli/search.go` | Query parsing, result display |
| **start/stop** | `internal/cli/start.go`, `stop.go` | Daemon process management |
| **status** | `internal/cli/status.go` | Index statistics display |
| **reindex** | `internal/cli/reindex.go` | Force full reindex |
| **config** | `internal/cli/config.go` | View/modify configuration |
| **HTTP Client** | `internal/cli/client.go` | Daemon communication |
| **Output Formatter** | `internal/output` | Table, JSON, verbose formatting |
| **Config Loader** | `internal/config` | YAML config parsing and validation |

---

## Sequence Diagrams

### Search Request Flow

```mermaid
sequenceDiagram
    participant User
    participant CLI as pm CLI
    participant Daemon as pommeld
    participant Embedder as CachedEmbedder
    participant Ollama
    participant Search as Search Service
    participant DB as SQLite+vec

    User->>CLI: pm search "error handling"
    CLI->>Daemon: POST /search {query, limit}

    Daemon->>Embedder: EmbedSingle(query)
    Embedder->>Embedder: Check cache
    alt Cache miss
        Embedder->>Ollama: POST /api/embed
        Ollama-->>Embedder: 768-dim vector
    end
    Embedder-->>Daemon: queryVector[]

    Daemon->>Search: Search(query, vector)
    Search->>DB: vec_distance_L2(queryVec)
    DB-->>Search: vector matches
    Search->>DB: FTS5 keyword search
    DB-->>Search: keyword matches
    Search->>Search: RRF combine + rank
    Search-->>Daemon: Results[]

    Daemon-->>CLI: SearchResponse JSON
    CLI->>CLI: Format output
    CLI-->>User: Ranked results (file:line, score)
```

### File Indexing Flow

```mermaid
sequenceDiagram
    participant FS as File System
    participant Watcher
    participant Indexer
    participant Chunker as ChunkerRegistry
    participant TreeSitter as Tree-sitter
    participant Embedder as CachedEmbedder
    participant Ollama
    participant DB as SQLite+vec

    FS->>Watcher: File modified event
    Watcher->>Watcher: Debounce (100ms)
    Watcher->>Watcher: Check .pommelignore
    Watcher->>Indexer: IndexFile(path)

    Indexer->>FS: Read file content
    Indexer->>Chunker: Chunk(file)
    Chunker->>Chunker: Detect language
    Chunker->>TreeSitter: Parse AST
    TreeSitter-->>Chunker: AST nodes
    Chunker->>Chunker: Extract chunks (file, class, function)
    Chunker-->>Indexer: ChunkResult[]

    loop For each chunk
        Indexer->>DB: InsertChunk(chunk)
        Indexer->>Embedder: EmbedSingle(content)
        alt Cache miss
            Embedder->>Ollama: POST /api/embed
            Ollama-->>Embedder: 768-dim vector
        end
        Embedder-->>Indexer: vector[]
        Indexer->>DB: InsertEmbedding(id, vec)
        Indexer->>DB: InsertFTSIndex(id, content)
    end

    Indexer->>DB: Update files metadata
```

---

## Database Schema (v3)

```mermaid
erDiagram
    files ||--o{ chunks : contains
    chunks ||--o| chunk_embeddings : has
    chunks ||--o| chunks_fts : indexed_in

    files {
        int id PK
        string path UK
        string language
        int size
        datetime modified_at
        string hash
    }

    chunks {
        string id PK
        int file_id FK
        string level
        string name
        text content
        int start_line
        int end_line
        string parent_id FK
        string hash
    }

    chunk_embeddings {
        string chunk_id PK,FK
        blob embedding "768-dim float32"
    }

    chunks_fts {
        string chunk_id FK
        text content "FTS5 indexed"
    }

    metadata {
        string key PK
        string value
    }

    subprojects {
        string id PK
        string path
        string name
        string marker_file
        string language
    }
```

---

## Key Interfaces

### Embedder Interface

```go
type Embedder interface {
    EmbedSingle(ctx context.Context, text string) ([]float32, error)
    Embed(ctx context.Context, texts []string) ([][]float32, error)
    Health(ctx context.Context) error
    ModelName() string
    Dimensions() int
}
```

Implementations:
- `OllamaClient` - HTTP client to Ollama API
- `CachedEmbedder` - LRU cache wrapper

### Chunker Interface

```go
type Chunker interface {
    Chunk(ctx context.Context, file *models.SourceFile) (*models.ChunkResult, error)
    Language() Language
}
```

Implementations:
- `GenericChunker` - Tree-sitter based, config-driven
- `FallbackChunker` - Line-based for unknown languages

### Searcher Interface

```go
type Searcher interface {
    Search(ctx context.Context, req SearchRequest) (*SearchResponse, error)
}
```

---

## Technology Stack

| Layer | Technology |
|-------|------------|
| Language | Go 1.24+ |
| CLI Framework | Cobra + Viper |
| HTTP Server | net/http + chi (optional) |
| Vector Database | SQLite + sqlite-vec |
| Full-Text Search | SQLite FTS5 |
| Code Parsing | Tree-sitter |
| Embedding Model | jina-embeddings-v2-base-code (768-dim) |
| Embedding Runtime | Ollama |
| File Watching | fsnotify |
| Platforms | macOS, Linux, Windows (x64/ARM64) |

---

## Project Structure

```
pommel/
├── cmd/
│   ├── pm/              # CLI entry point
│   └── pommeld/         # Daemon entry point
├── internal/
│   ├── api/             # HTTP API types and handlers
│   ├── chunker/         # Tree-sitter based code chunking
│   ├── cli/             # Cobra command implementations
│   ├── config/          # YAML configuration loading
│   ├── daemon/          # Daemon server, watcher, indexer
│   ├── db/              # SQLite + sqlite-vec database layer
│   ├── embedder/        # Ollama embedding client + cache
│   ├── models/          # Shared data models
│   ├── output/          # CLI output formatting
│   └── search/          # Vector similarity search + hybrid
├── languages/           # Language config YAML files
├── scripts/             # Installation scripts
├── docs/                # Documentation
└── .pommel/             # Per-project data (gitignored)
    ├── config.yaml      # Project configuration
    └── index.db         # SQLite database with vectors
```

---

## Rendering These Diagrams

These Mermaid diagrams render natively in:
- GitHub (README, issues, PRs)
- GitLab
- Obsidian
- Notion (with Mermaid block)
- VS Code (with Mermaid extension)

For standalone images, use [mermaid.live](https://mermaid.live) or the Mermaid CLI:

```bash
npm install -g @mermaid-js/mermaid-cli
mmdc -i docs/architecture.md -o docs/architecture.png
```
