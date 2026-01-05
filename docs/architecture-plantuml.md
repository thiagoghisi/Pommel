# Pommel Architecture (C4 Model - PlantUML)

This document describes Pommel's architecture using the [C4 model](https://c4model.com/) with PlantUML notation.

## Rendering

Render these diagrams using:
- [PlantUML Online Server](https://www.plantuml.com/plantuml/uml/)
- VS Code with PlantUML extension
- IntelliJ IDEA (built-in)
- CLI: `plantuml docs/architecture-plantuml.md`

---

## C1: System Context

```plantuml
@startuml C1_System_Context
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Context.puml

title System Context - Pommel

Person(user, "Developer / AI Agent", "Claude Code, Cursor, Copilot, or human developer")

System(pommel, "Pommel", "Local-first semantic code search system maintaining vector DB of code embeddings")

System_Ext(ollama, "Ollama", "Local LLM runtime hosting jina-embeddings-v2-base-code model")
System_Ext(filesystem, "File System", "Source code files being indexed")

Rel(user, pommel, "pm search, init, start, status", "CLI")
Rel(pommel, ollama, "Generate embeddings", "HTTP POST /api/embed")
Rel(filesystem, pommel, "File change events", "fsnotify")
Rel(pommel, filesystem, "Read source files", "OS read")
Rel(pommel, user, "Ranked search results", "JSON")

@enduml
```

---

## C2: Container Diagram

```plantuml
@startuml C2_Container
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

title Container Diagram - Pommel

Person(user, "Developer / AI Agent")

System_Boundary(pommel, "Pommel System") {
    Container(cli, "pm", "Go Binary, Cobra + Viper", "CLI interface for search, init, start, stop, status, reindex commands")
    Container(daemon, "pommeld", "Go HTTP Server", "File watcher, indexer, search service, API endpoints")
    ContainerDb(db, "SQLite Database", "SQLite + sqlite-vec + FTS5", "Stores chunks, 768-dim vectors, full-text index, file metadata")
}

System_Ext(ollama, "Ollama", "jina-embeddings-v2-base-code")
System_Ext(fs, "File System", "Source code")

Rel(user, cli, "CLI commands")
Rel(cli, daemon, "HTTP GET/POST", "JSON")
Rel(daemon, db, "SQL queries", "sqlite-vec")
Rel(daemon, ollama, "Generate embeddings", "POST /api/embed")
Rel(fs, daemon, "File events", "fsnotify")
Rel(daemon, fs, "Read files")
Rel(cli, user, "Search results", "JSON/Table")

@enduml
```

---

## C3: Component Diagram - Daemon

```plantuml
@startuml C3_Daemon_Components
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

title Component Diagram - pommeld Daemon

Container_Boundary(daemon, "pommeld (Daemon)") {

    Component(router, "Router", "http.ServeMux", "HTTP request routing")
    Component(handlers, "API Handlers", "internal/api", "health, status, search, reindex endpoints")

    Component(watcher, "Watcher", "internal/daemon/watcher.go", "fsnotify file watching, debouncing, .pommelignore filtering")
    Component(indexer, "Indexer", "internal/daemon/indexer.go", "Orchestrates chunking and embedding")
    Component(search_svc, "Search Service", "internal/search", "Vector similarity, hybrid RRF ranking, filtering")
    Component(state, "State Manager", "internal/daemon/state.go", "PID tracking, daemon lifecycle")

    Component(chunker_reg, "Chunker Registry", "internal/chunker", "Routes files to chunker by language")
    Component(generic_chunker, "Generic Chunker", "internal/chunker", "Tree-sitter AST parsing")
    Component(fallback_chunker, "Fallback Chunker", "internal/chunker", "Line-based for unknown languages")

    Component(cached_embed, "Cached Embedder", "internal/embedder/cache.go", "LRU cache wrapper")
    Component(ollama_client, "Ollama Client", "internal/embedder/ollama.go", "HTTP client for embeddings")

    Component(db_layer, "DB Layer", "internal/db", "Chunks, vectors, FTS, files")
}

Container_Ext(cli, "pm CLI")
Container_Ext(ollama, "Ollama")
Container_Ext(fs, "File System")
ContainerDb_Ext(sqlite, "SQLite + vec")

Rel(cli, router, "HTTP")
Rel(router, handlers, "Routes")
Rel(handlers, search_svc, "Search requests")
Rel(handlers, indexer, "Reindex requests")

Rel(fs, watcher, "fsnotify events")
Rel(watcher, indexer, "File events")

Rel(indexer, chunker_reg, "Chunk file")
Rel(chunker_reg, generic_chunker, "Supported languages")
Rel(chunker_reg, fallback_chunker, "Unknown languages")

Rel(indexer, cached_embed, "Generate embeddings")
Rel(cached_embed, ollama_client, "Cache miss")
Rel(ollama_client, ollama, "POST /api/embed")

Rel(search_svc, cached_embed, "Embed query")
Rel(search_svc, db_layer, "Vector + FTS search")
Rel(indexer, db_layer, "Store chunks")
Rel(db_layer, sqlite, "SQL")

@enduml
```

---

## C3: Component Diagram - CLI

```plantuml
@startuml C3_CLI_Components
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

title Component Diagram - pm CLI

Container_Boundary(cli, "pm (CLI)") {

    Component(root_cmd, "Root Command", "internal/cli/root.go", "Cobra rootCmd, global flags, version")

    Component(init_cmd, "init", "internal/cli/init.go", "--auto --claude --start flags")
    Component(search_cmd, "search", "internal/cli/search.go", "--limit --level --json --path flags")
    Component(start_cmd, "start", "internal/cli/start.go", "Spawn daemon process")
    Component(stop_cmd, "stop", "internal/cli/stop.go", "Kill daemon process")
    Component(status_cmd, "status", "internal/cli/status.go", "Index statistics")
    Component(reindex_cmd, "reindex", "internal/cli/reindex.go", "Force full reindex")
    Component(config_cmd, "config", "internal/cli/config.go", "View/edit configuration")

    Component(client, "HTTP Client", "internal/cli/client.go", "Health(), Status(), Search(), Reindex()")
    Component(output, "Output Formatter", "internal/output", "Table, JSON, verbose formatting")
    Component(config_loader, "Config Loader", "internal/config", ".pommel/config.yaml parsing")
}

Person_Ext(user, "User")
Container_Ext(daemon, "pommeld")

Rel(user, root_cmd, "CLI input")
Rel(root_cmd, init_cmd, "pm init")
Rel(root_cmd, search_cmd, "pm search")
Rel(root_cmd, start_cmd, "pm start")
Rel(root_cmd, stop_cmd, "pm stop")
Rel(root_cmd, status_cmd, "pm status")
Rel(root_cmd, reindex_cmd, "pm reindex")
Rel(root_cmd, config_cmd, "pm config")

Rel(search_cmd, client, "Search request")
Rel(status_cmd, client, "Status request")
Rel(reindex_cmd, client, "Reindex request")

Rel(client, daemon, "HTTP")

Rel(init_cmd, config_loader, "Generate config")
Rel(search_cmd, output, "Format results")
Rel(status_cmd, output, "Format status")

Rel(output, user, "Display")

@enduml
```

---

## Sequence Diagram: Search Request

```plantuml
@startuml Sequence_Search
title Search Request Flow

actor User
participant "pm CLI" as CLI
participant "pommeld" as Daemon
participant "CachedEmbedder" as Embedder
participant "Ollama" as Ollama
participant "Search Service" as Search
database "SQLite+vec" as DB

User -> CLI: pm search "error handling"
CLI -> Daemon: POST /search {query, limit}

Daemon -> Embedder: EmbedSingle(query)
Embedder -> Embedder: Check cache

alt Cache miss
    Embedder -> Ollama: POST /api/embed
    Ollama --> Embedder: 768-dim vector
end

Embedder --> Daemon: queryVector[]

Daemon -> Search: Search(query, vector)
Search -> DB: vec_distance_L2(queryVec)
DB --> Search: vector matches

Search -> DB: FTS5 keyword search
DB --> Search: keyword matches

Search -> Search: RRF combine + rank
Search --> Daemon: Results[]

Daemon --> CLI: SearchResponse JSON
CLI -> CLI: Format output
CLI --> User: Ranked results (file:line, score)

@enduml
```

---

## Sequence Diagram: File Indexing

```plantuml
@startuml Sequence_Indexing
title File Indexing Flow

participant "File System" as FS
participant "Watcher" as Watcher
participant "Indexer" as Indexer
participant "ChunkerRegistry" as Chunker
participant "Tree-sitter" as TreeSitter
participant "CachedEmbedder" as Embedder
participant "Ollama" as Ollama
database "SQLite+vec" as DB

FS -> Watcher: File modified event
Watcher -> Watcher: Debounce (100ms)
Watcher -> Watcher: Check .pommelignore
Watcher -> Indexer: IndexFile(path)

Indexer -> FS: Read file content
Indexer -> Chunker: Chunk(file)
Chunker -> Chunker: Detect language
Chunker -> TreeSitter: Parse AST
TreeSitter --> Chunker: AST nodes
Chunker -> Chunker: Extract chunks\n(file, class, function)
Chunker --> Indexer: ChunkResult[]

loop For each chunk
    Indexer -> DB: InsertChunk(chunk)
    Indexer -> Embedder: EmbedSingle(content)

    alt Cache miss
        Embedder -> Ollama: POST /api/embed
        Ollama --> Embedder: 768-dim vector
    end

    Embedder --> Indexer: vector[]
    Indexer -> DB: InsertEmbedding(id, vec)
    Indexer -> DB: InsertFTSIndex(id, content)
end

Indexer -> DB: Update files metadata

@enduml
```

---

## Database Schema (ERD)

```plantuml
@startuml Database_Schema
title Pommel Database Schema (v3)

entity "files" as files {
    *id : INTEGER <<PK>>
    --
    *path : TEXT <<UNIQUE>>
    language : TEXT
    size : INTEGER
    modified_at : DATETIME
    hash : TEXT
}

entity "chunks" as chunks {
    *id : TEXT <<PK>>
    --
    *file_id : INTEGER <<FK>>
    level : TEXT
    name : TEXT
    content : TEXT
    start_line : INTEGER
    end_line : INTEGER
    parent_id : TEXT <<FK>>
    hash : TEXT
}

entity "chunk_embeddings" as embeddings {
    *chunk_id : TEXT <<PK, FK>>
    --
    embedding : BLOB
    .. 768-dim float32 ..
}

entity "chunks_fts" as fts {
    *rowid : INTEGER <<PK>>
    --
    chunk_id : TEXT <<FK>>
    content : TEXT
    .. FTS5 virtual table ..
}

entity "metadata" as metadata {
    *key : TEXT <<PK>>
    --
    value : TEXT
}

entity "subprojects" as subprojects {
    *id : TEXT <<PK>>
    --
    path : TEXT
    name : TEXT
    marker_file : TEXT
    language : TEXT
}

files ||--o{ chunks : contains
chunks ||--o| embeddings : has
chunks ||--o| fts : indexed_in
chunks ||--o{ chunks : parent_id

@enduml
```

---

## Deployment Diagram

```plantuml
@startuml Deployment
title Deployment - Single Machine

node "Developer Machine" {

    node "User Space" {
        component [pm CLI] as cli
        component [pommeld Daemon] as daemon
        component [Ollama] as ollama
    }

    node "File System" {
        folder "Project Root" {
            folder ".pommel/" {
                file "config.yaml" as config
                database "index.db" as db
            }
            folder "src/" as src
        }
    }
}

cli --> daemon : HTTP\nlocalhost:PORT
daemon --> ollama : HTTP\nlocalhost:11434
daemon --> db : SQLite
daemon ..> src : fsnotify\nwatch
daemon --> config : read

note right of daemon
  PORT is deterministic
  hash of project root
  (or configured)
end note

@enduml
```

---

## Technology Stack

```plantuml
@startuml Tech_Stack
title Technology Stack

package "Application Layer" {
    [pm CLI] as cli
    [pommeld Daemon] as daemon
}

package "Framework Layer" {
    [Cobra] as cobra
    [Viper] as viper
    [chi Router] as chi
}

package "Processing Layer" {
    [Tree-sitter] as treesitter
    [fsnotify] as fsnotify
}

package "Data Layer" {
    [SQLite] as sqlite
    [sqlite-vec] as sqlitevec
    [FTS5] as fts5
}

package "External Services" {
    [Ollama] as ollama
    [jina-embeddings-v2-base-code] as jina
}

cli --> cobra
cli --> viper
daemon --> chi
daemon --> treesitter
daemon --> fsnotify
daemon --> sqlite
sqlite --> sqlitevec
sqlite --> fts5
daemon --> ollama
ollama --> jina

@enduml
```

---

## Component Responsibilities

```plantuml
@startuml Component_Table
title Component Responsibilities

object "pm CLI" as cli {
    language = "Go"
    framework = "Cobra + Viper"
    --
    - Parse user commands
    - Communicate with daemon
    - Format and display results
}

object "pommeld Daemon" as daemon {
    language = "Go"
    server = "net/http"
    --
    - Watch file system changes
    - Index code chunks
    - Generate embeddings
    - Serve search requests
}

object "SQLite + sqlite-vec" as db {
    type = "Embedded DB"
    extensions = "vec, FTS5"
    --
    - Store code chunks
    - Store 768-dim vectors
    - Full-text search index
    - Vector similarity (L2)
}

object "Ollama" as ollama {
    type = "External Service"
    model = "jina-embeddings-v2"
    --
    - Generate code embeddings
    - 768 dimensions
    - Local execution
}

cli --> daemon : HTTP
daemon --> db : SQL
daemon --> ollama : REST API

@enduml
```

---

## Quick Reference

| Diagram | What it Shows |
|---------|---------------|
| C1 System Context | Pommel + external actors (users, Ollama, file system) |
| C2 Container | pm CLI, pommeld daemon, SQLite database |
| C3 Daemon | Internal components: watcher, indexer, chunker, embedder, search |
| C3 CLI | Commands, HTTP client, output formatter, config loader |
| Search Sequence | Query flow from user to ranked results |
| Indexing Sequence | File change to stored vectors |
| Database ERD | Schema with chunks, embeddings, FTS |
| Deployment | Single-machine local setup |

---

## Generating Images

```bash
# Install PlantUML
brew install plantuml  # macOS
apt install plantuml   # Linux

# Generate PNGs from this file
plantuml -tpng docs/architecture-plantuml.md

# Generate SVGs
plantuml -tsvg docs/architecture-plantuml.md

# Or use the online server
# https://www.plantuml.com/plantuml/uml/
```
