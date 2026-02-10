# C# Analyzer Provider — Technical Architecture

## Table of Contents

1. [Overview](#1-overview)
2. [Konveyor Ecosystem Context](#2-konveyor-ecosystem-context)
3. [Technology Stack](#3-technology-stack)
4. [Concepts Primer](#4-concepts-primer)
5. [High-Level Architecture](#5-high-level-architecture)
6. [Project Structure](#6-project-structure)
7. [gRPC Service Interface](#7-grpc-service-interface)
8. [Initialization Flow](#8-initialization-flow)
9. [Evaluate (Query) Flow](#9-evaluate-query-flow)
10. [The Stack Graph — Core Data Structure](#10-the-stack-graph--core-data-structure)
11. [Tree-Sitter + Stack Graphs Integration](#11-tree-sitter--stack-graphs-integration)
12. [TSG Language Guide](#12-tsg-language-guide)
13. [Query Type System](#13-query-type-system)
14. [Search Pattern Matching Engine](#14-search-pattern-matching-engine)
15. [Worked Example (Detailed)](#15-worked-example-detailed)
16. [Dependency Resolution Pipeline](#16-dependency-resolution-pipeline)
17. [SDK Detection & Target Framework](#17-sdk-detection--target-framework)
18. [SQLite Caching Layer](#18-sqlite-caching-layer)
19. [File Change Notification Flow](#19-file-change-notification-flow)
20. [Thread Safety & Concurrency Model](#20-thread-safety--concurrency-model)
21. [Transport Layer Options](#21-transport-layer-options)
22. [Result Deduplication](#22-result-deduplication)
23. [End-to-End Example Trace](#23-end-to-end-example-trace)
24. [Build & Deployment](#24-build--deployment)
25. [Testing Infrastructure](#25-testing-infrastructure)
26. [Contributor/Development Workflow](#26-contributordevelopment-workflow)
27. [Design Rationale](#27-design-rationale)
28. [Known Limitations & Edge Cases](#28-known-limitations--edge-cases)
29. [File-by-File Reference](#29-file-by-file-reference)

---

## 1. Overview

This repository implements a **gRPC microservice written in Rust** that performs semantic code analysis of C# projects. It acts as a "search engine" for C# source code — given a query pattern like `System.Web.Mvc.*`, it returns exact file paths, line numbers, and code locations where those symbols are referenced.

The service is a plugin in the **Konveyor** ecosystem, a toolkit for migrating legacy applications (e.g., .NET Framework to modern .NET 8). It enables migration planning tools to automatically identify code that needs to change during a platform upgrade.

**Core capabilities:**
- Parse C# source files into a semantic graph of symbols
- Search the graph for references to specific namespaces, classes, methods, and fields
- Resolve and optionally decompile NuGet dependencies
- Parse SDK XML documentation for symbol discovery
- Cache analysis results in SQLite for incremental updates
- Serve results over gRPC with multi-transport support (TCP, Unix sockets, Windows named pipes)

---

## 2. Konveyor Ecosystem Context

```
+-------------------------------------------------------------+
|                     Konveyor Platform                        |
|                                                              |
|  +---------------+     +--------------------------------+   |
|  |  UI / Kantra  |---->|         analyzer-lsp            |   |
|  |  (CLI tool)   |     |  (orchestrator -- written in Go)|   |
|  +---------------+     +----------------+---------------+   |
|                                          |                   |
|                    gRPC calls to language providers           |
|                                          |                   |
|         +--------------------------------+----------+        |
|         |                                |          |        |
|         v                                v          v        |
|  +---------------+  +---------------------+  +----------+   |
|  | Java Provider |  | C# Analyzer Provider|  | Go, etc. |   |
|  | (Go-based)    |  | (THIS REPO - Rust)  |  |          |   |
|  +---------------+  +---------------------+  +----------+   |
|                                                              |
+--------------------------------------------------------------+
```

The **analyzer-lsp** is the orchestrator. It reads **rulesets** (YAML files describing what to look for during migration), and delegates actual code searching to **language-specific providers** over gRPC. This repository is the C# provider.

**Communication protocol:** All communication between `analyzer-lsp` and this provider happens over gRPC, defined in `src/build/proto/provider.proto`.

---

## 3. Technology Stack

| Component | Technology | Purpose |
|---|---|---|
| Language | **Rust** (2021 edition) | Primary implementation language |
| Async Runtime | **Tokio** (32 worker threads) | Async I/O and task scheduling |
| RPC Framework | **Tonic** (gRPC) + **Prost** (protobuf) | Service interface and serialization |
| C# Parsing | **tree-sitter** + **tree-sitter-c-sharp** | Concrete syntax tree generation |
| Semantic Analysis | **stack-graphs** + **tree-sitter-stack-graphs** | Symbol relationship graph |
| Database | **SQLite** (via stack-graphs crate) | Persistent graph caching |
| XML Parsing | **quick-xml** | .csproj and SDK XML documentation |
| CLI | **clap** | Command-line argument parsing |
| Logging | **tracing** + **tracing-subscriber** | Structured logging |
| Serialization | **serde** / **serde_json** / **serde_yml** | JSON and YAML handling |
| Pattern Matching | **regex** | Wildcard query support |
| File Discovery | **walkdir** | Recursive directory traversal |

**External runtime tools (not Rust crates):**

| Tool | Purpose |
|---|---|
| **.NET SDK 9.x** | Reference assembly access for modern .NET |
| **Paket** | NuGet dependency resolution |
| **ILSpy (ilspycmd)** | Decompilation of .dll to .cs source |
| **dotnet-install script** | Automated SDK installation |

---

## 4. Concepts Primer

This section explains the foundational technologies used by the analysis engine. If you already know what tree-sitter, stack-graphs, and TSG are, skip to [Section 5](#5-high-level-architecture).

### 4.1 What is Tree-Sitter?

**Tree-sitter** is a parser generator tool and an incremental parsing library. Given a grammar for a programming language, it produces a parser that can turn source code into a syntax tree.

Key properties:
- **Fast** — parses an entire file in milliseconds, and can re-parse after an edit in microseconds by only re-processing the changed region
- **Error-tolerant** — produces a valid (partial) tree even if the source code has syntax errors
- **Language-agnostic** — grammars exist for 100+ languages; this project uses the `tree-sitter-c-sharp` grammar (v0.23)

Tree-sitter is **not** a compiler. It does not type-check, resolve references, or understand semantics. It only understands syntax.

### 4.2 CST vs AST

Tree-sitter produces a **Concrete Syntax Tree (CST)**, not an **Abstract Syntax Tree (AST)**. The difference matters:

```
C# Source:    using System.Web.Mvc;

CST (what tree-sitter produces):          AST (what a compiler might produce):
  using_directive                           ImportDeclaration
    qualified_name                            name: "System.Web.Mvc"
      identifier "System"
      identifier "Web"
      identifier "Mvc"
```

A **CST** preserves every token from the source — braces, semicolons, individual identifiers within qualified names. An **AST** abstracts away syntactic details and keeps only the semantic meaning.

This matters because TSG rules (Section 12) match on CST node type names like `using_directive`, `class_declaration`, `member_access_expression`. These names come from the tree-sitter C# grammar, not from this project.

### 4.3 What are Stack Graphs?

**Stack graphs** are a data structure for **name binding** — determining what a name in source code refers to. They were developed by GitHub for code navigation features (go-to-definition, find-references).

The core idea:

- **Definition nodes** ("pop_symbol"): declare "I define a symbol named X"
- **Reference nodes** ("push_symbol"): declare "I use something named X"
- **Scope nodes**: represent lexical scopes that contain definitions
- **Edges**: connect definitions, references, and scopes

Name resolution works by finding a path through the graph from a reference node to a matching definition node, using a stack-based algorithm that pushes and pops symbol names.

**How this project uses them differently:** This project does not use stack graphs for code navigation (go-to-definition). Instead, it uses the graph structure to build a **searchable index of fully qualified symbol names**. The containment edges (namespace contains class, class contains method) allow the query engine to reconstruct FQDNs like `System.Web.Mvc.Controller` by walking upward from any node.

### 4.4 What is TSG?

**TSG** (Tree-Sitter Graph) is a domain-specific language for declaratively transforming tree-sitter CST nodes into stack graph nodes and edges. It is interpreted by the `tree-sitter-stack-graphs` library.

TSG rules are pattern-match rules: "when tree-sitter produces a node of type X, create graph nodes Y and Z with these properties and connect them with edges." The file `src/c_sharp_graph/stack-graphs.tsg` (1061 lines) contains all the C# analysis logic in this DSL.

Without TSG, you would need to write imperative Rust code for every C# construct (class declarations, method declarations, using directives, etc.). TSG lets you express the same logic declaratively.

For a complete syntax reference, see [Section 12 (TSG Language Guide)](#12-tsg-language-guide).

### 4.5 Why This Combination?

Why tree-sitter + stack-graphs instead of using the official C# compiler (Roslyn)?

- **No .NET runtime dependency for parsing** — Roslyn requires a .NET runtime; tree-sitter runs natively in Rust
- **Works on incomplete/invalid code** — tree-sitter is error-tolerant; Roslyn may reject code with errors
- **Lightweight** — the parser is a few hundred KB, not the full compiler toolchain
- **Cross-platform** — no platform-specific dependencies for the core analysis
- **Good enough for migration analysis** — the project does not need full semantic analysis (type inference, overload resolution). It needs to find where specific APIs are referenced, which FQDN-based pattern matching handles well

The tradeoff: no type inference, no overload resolution, no flow analysis. See [Section 28 (Known Limitations)](#28-known-limitations--edge-cases) for details.

### 4.6 Key Terminology Quick Reference

| Term | Meaning in this project |
|---|---|
| **CST node** | A syntax element produced by tree-sitter (e.g., `class_declaration`, `using_directive`) |
| **Graph node** | A node in the stack graph, tagged with a symbol name and syntax type |
| **FQDN** | Fully Qualified Domain Name of a symbol (e.g., `System.Web.Mvc.Controller`) |
| **TSG rule** | A pattern-action block in `stack-graphs.tsg` that transforms CST nodes into graph nodes |
| **Pop symbol** | A definition node — "I define a name" (the name is popped from the resolution stack) |
| **Push symbol** | A reference node — "I use a name" (the name is pushed onto the resolution stack) |
| **Precedence-10 edge** | A back-reference edge from child to parent, used by the query engine to walk upward and reconstruct FQDNs |
| **Source type** | Whether a file is user code (`source`) or a third-party library (`dependency`) |
| **Containment edge** | A precedence-0 edge from parent to child (namespace -> class -> method) |

---

## 5. High-Level Architecture

```
+------------------------------------------------------------------+
|                        main.rs (Entry Point)                      |
|  - Parses CLI args (--port, --socket, --db-path, etc.)            |
|  - Sets up logging (file or stdout via tracing)                   |
|  - Creates Tokio runtime (32 worker threads)                      |
|  - Starts gRPC server on TCP or Unix Domain Socket                |
+------------------------------+-----------------------------------+
                               |
                               | registers 3 gRPC services
                               v
+------------------------------------------------------------------+
|                     gRPC Service Layer                             |
|                                                                   |
|  +---------------------------+  +-------------------------------+ |
|  |   ProviderService         |  | ProviderCodeLocationService   | |
|  |  (Init, Evaluate,         |  | (GetCodeSnip -- returns code  | |
|  |   GetDependencies,        |  |  snippets with context lines) | |
|  |   NotifyFileChanges,      |  +-------------------------------+ |
|  |   Capabilities, etc.)     |                                    |
|  +-------------+-------------+                                    |
|                |                                                  |
|                | Proto defined in: src/build/proto/provider.proto  |
|                | Generated Rust:   src/analyzer_service/provider.rs|
+----------------+--------------------------------------------------+
                 |
                 | implemented by
                 v
+------------------------------------------------------------------+
|                  CSharpProvider (provider/csharp.rs)               |
|                                                                   |
|  State:                                                           |
|    config:  Arc<Mutex<Option<Config>>>                             |
|    project: Arc<Mutex<Option<Arc<Project>>>>                      |
|    db_path: PathBuf                                               |
|    context_lines: u32                                             |
|                                                                   |
|  Key Methods:                                                     |
|    init()     -- Sets up project, graph, dependencies             |
|    evaluate() -- Runs a search query against the graph            |
|    notify_file_changes() -- Invalidates + rebuilds changed files  |
+----------------+--------------------------------------------------+
                 |
                 | delegates to
                 v
+------------------------------------------------------------------+
|                     Project (provider/project.rs)                  |
|                                                                   |
|  State:                                                           |
|    location:        PathBuf (project root)                        |
|    graph:           Mutex<Option<StackGraph>>                     |
|    dependencies:    tokio::Mutex<Option<Vec<Dependencies>>>       |
|    language_config: Arc<RwLock<Option<LanguageConfig>>>            |
|    tools:           Tools { ilspy, paket, dotnet-install, sdk }    |
|    target_framework: Mutex<Option<String>>                        |
|    analysis_mode:   Full | SourceOnly                             |
|                                                                   |
|  Key Methods:                                                     |
|    get_project_graph()              -- Build/load stack graph     |
|    validate_language_configuration() -- Init tree-sitter          |
|    resolve()                         -- Resolve NuGet deps        |
|    load_to_database()                -- Persist graph to SQLite   |
+----------------+--------------------------------------------------+
                 |
       +---------+--------------------------+
       |                                    |
       v                                    v
+--------------------+    +---------------------------------+
|  c_sharp_graph/    |    |  dependency_resolution.rs        |
|  (Query Engine)    |    |  + sdk_detection.rs              |
|                    |    |  + target_framework.rs           |
+--------------------+    +---------------------------------+
```

---

## 6. Project Structure

```
c-sharp-analyzer-provider/
|
|-- src/
|   |-- main.rs                          Server bootstrap (187 lines)
|   |-- lib.rs                           Re-exports modules
|   |
|   |-- analyzer_service/
|   |   |-- mod.rs                       Include generated proto code
|   |   |-- provider.rs                  AUTO-GENERATED from .proto (77KB)
|   |   +-- provider_service_descriptor.bin  Proto reflection descriptor
|   |
|   |-- provider/
|   |   |-- mod.rs                       Module declarations + re-exports
|   |   |-- csharp.rs                    gRPC trait impl (974 lines)
|   |   |-- project.rs                   Project state management (348 lines)
|   |   |-- dependency_resolution.rs     Paket + ILSpy pipeline (895 lines)
|   |   |-- sdk_detection.rs             .NET SDK finder (318 lines)
|   |   |-- target_framework.rs          .csproj TFM parser (991 lines)
|   |   +-- code_snip.rs                 Code snippet extraction (102 lines)
|   |
|   |-- c_sharp_graph/
|   |   |-- mod.rs                       Module exports
|   |   |-- query.rs                     Core query engine (1667 lines)
|   |   |-- loader.rs                    Graph construction (370 lines)
|   |   |-- language_config.rs           Tree-sitter setup (138 lines)
|   |   |-- namespace_query.rs           Namespace search impl (458 lines)
|   |   |-- class_query.rs              Class search impl (307 lines)
|   |   |-- method_query.rs             Method search impl (292 lines)
|   |   |-- field_query.rs              Field search impl (296 lines)
|   |   |-- results.rs                  Result types (158 lines)
|   |   |-- dependency_xml_analyzer.rs  SDK XML parser (792 lines)
|   |   |-- stack-graphs.tsg            Graph construction rules (600+ lines)
|   |   |-- builtins.cs                 Built-in C# type definitions
|   |   +-- builtins.cfg                Builtins configuration
|   |
|   +-- pipe_stream/
|       |-- mod.rs                       Platform-conditional exports
|       +-- server.rs                    Windows named pipe implementation
|
|-- src/build/
|   +-- proto/
|       +-- provider.proto               gRPC service definitions
|
|-- build.rs                             Proto compilation script
|-- Cargo.toml, Cargo.lock              Rust dependencies
|-- Dockerfile                           Multi-stage container build
|-- Makefile                             Build & test automation (249 lines)
|
|-- tests/
|   |-- integration_test.rs             Integration tests
|   +-- demos/                           Test cases organized by type
|       |-- namespace_search/
|       |-- class_search/
|       |-- method_search/
|       +-- field_search/
|
|-- testdata/
|   |-- nerd-dinner/                     ASP.NET MVC4 test project
|   +-- net8-sample/                     .NET 8 sample project
|
|-- docs/                                Documentation
|-- rulesets/                            Example analysis rules
|   +-- dotnet-core-migration/
|-- scripts/                             Utility scripts
|-- e2e-tests/                           End-to-end test configuration
+-- .github/workflows/                  CI/CD pipelines
```

---

## 7. gRPC Service Interface

Defined in `src/build/proto/provider.proto`, compiled to Rust by `build.rs` using `tonic-build`.

### Services

```
service ProviderService {
    rpc Capabilities()                              -> CapabilitiesResponse
    rpc Init(Config)                                -> InitResponse
    rpc Evaluate(EvaluateRequest)                   -> EvaluateResponse
    rpc NotifyFileChanges(NotifyFileChangesRequest) -> NotifyResponse
    rpc GetDependencies(ServiceRequest)             -> DependencyResponse
    rpc GetDependenciesDAG(ServiceRequest)          -> DAGResponse
    rpc Prepare(PrepareRequest)                     -> PrepareResponse
    rpc StreamPrepareProgress(PrepareRequest)       -> stream ProgressEvent
    rpc Stop(ServiceRequest)                        -> StopResponse
}

service ProviderCodeLocationService {
    rpc GetCodeSnip(GetCodeSnipRequest) -> GetCodeSnipResponse
}

service ProviderDependencyLocationService {
    rpc GetDependencyLocation(GetDependencyLocationRequest)
        -> GetDependencyLocationResponse
}
```

### Key Messages

| Message | Fields | Purpose |
|---|---|---|
| `Config` | location, dependencyPath, analysisMode, providerSpecificConfig | Project configuration sent during Init |
| `EvaluateRequest` | cap, conditionInfo, codeSnipRequest | Query request with pattern |
| `EvaluateResponse` | matched, incidentContexts | Query results |
| `IncidentContext` | fileURI, lineNumber, codeLocation, variables | Single match result |
| `Location` | startPosition, endPosition | Code span |
| `Position` | line, character | Point in source |
| `Dependency` | name, version, classifier, type, labels | Package info |
| `FileChange` | uri, content, isSaved | File modification notification |
| `ProgressEvent` | message, percentage | Streaming progress updates |

### Interaction Pattern

The core interaction is always **Init then Evaluate**:

```
Client (analyzer-lsp)                    Server (this provider)
        |                                          |
        |-------- Capabilities() ---------------->|
        |<------- ["referenced"] -----------------|
        |                                          |
        |-------- Init(Config) ------------------>|
        |         location: "/path/to/project"     |
        |         analysisMode: "source-only"       |
        |<------- InitResponse { success: true } --|
        |                                          |
        |-------- Evaluate(Request) ------------->|
        |         pattern: "System.Web.Mvc.*"       |
        |<------- EvaluateResponse { matched,  ----|
        |           incident_contexts: [...] }     |
        |                                          |
        |-------- Evaluate(Request) ------------->|
        |         pattern: "System.Configuration.*" |
        |<------- EvaluateResponse { ... } --------|
        |                                          |
        |-------- NotifyFileChanges([...]) ------->|
        |<------- NotifyResponse ------------------|
        |                                          |
```

---

## 8. Initialization Flow

This is the most complex operation. Here is exactly what happens when a client calls `Init()`:

```
Client calls Init(Config)
  |
  |  Config contains:
  |    - location: "/path/to/csharp/project"
  |    - analysisMode: "source-only" or "full"
  |    - providerSpecificConfig: { ilspy_cmd, paket_cmd, ... }
  |
  v
+-----------------------------------------------------------+
| Step 1: Extract tools from config                          |
|                                                            |
|   Project::get_tools(config)                               |
|     - Looks for "ilspy_cmd" in config, or runs `which`     |
|     - Looks for "paket_cmd" in config, or runs `which`     |
|     - Looks for "dotnet_install_cmd" (default path)        |
|     - Looks for "dotnet_sdk_path" (optional override)      |
|     - Returns Tools struct or error if critical missing    |
+---------------------------+--------------------------------+
                            v
+-----------------------------------------------------------+
| Step 2: Detect target framework                            |
|                                                            |
|   TargetFrameworkHelper::get_earliest_from_directory()      |
|     - Walks project directory for .csproj files            |
|     - Parses XML: <TargetFramework>net8.0</...>            |
|     - If multiple projects, picks the earliest/lowest      |
|     - Returns TargetFramework enum                         |
|                                                            |
|   If modern .NET (net5.0+):                                |
|     SdkDetector::find_sdk(target_framework)                |
|       - Checks configured sdk_path                         |
|       - Checks DOTNET_ROOT env var                         |
|       - Checks platform paths (/usr/share/dotnet, etc.)    |
|       - If found: spawns task -> load_sdk_from_path()      |
|       - If not found: spawns task -> run dotnet-install     |
|                                                            |
|   If .NET Framework (net45, net472):                       |
|     Skips SDK detection entirely                           |
+---------------------------+--------------------------------+
                            v
+-----------------------------------------------------------+
| Step 3: Initialize tree-sitter language config             |
|                                                            |
|   project.validate_language_configuration()                |
|     - Creates SourceNodeLanguageConfiguration              |
|     - Loads tree-sitter C# grammar                         |
|     - Loads stack-graphs.tsg rules (600+ lines of          |
|       graph construction rules for C# syntax)              |
|     - Loads builtins.cs (built-in C# type definitions)     |
|     - Stores in project.source_language_config             |
+---------------------------+--------------------------------+
                            v
+-----------------------------------------------------------+
| Step 4: Build or load the stack graph                      |
|                                                            |
|   project.get_project_graph()                              |
|     - First tries: Load from SQLite database               |
|       SQLiteReader::open(db_path)                          |
|       db_reader.load_graphs_for_file_or_directory()        |
|       If DB exists and has data, deserialize graph          |
|                                                            |
|     - If no DB: Build from scratch                         |
|       init_stack_graph(location, language_config)           |
|       Walks all .cs files in project directory              |
|       For each file:                                       |
|          1. tree-sitter parses into CST                    |
|          2. stack-graphs.tsg rules transform CST nodes     |
|             into graph nodes with edges                    |
|          3. Nodes tagged with SyntaxType:                  |
|             Import, CompUnit, NamespaceDeclaration,        |
|             ClassDef, MethodName, FieldName, etc.          |
|       Result: StackGraph in memory                         |
+---------------------------+--------------------------------+
                            v
+-----------------------------------------------------------+
| Step 5: Resolve dependencies (if analysisMode = Full)      |
|                                                            |
|   project.resolve()                                        |
|     - Spawns async task                                    |
|     - Runs: `paket install` in project directory           |
|     - Reads paket.lock to get dependency list              |
|     - For each dependency:                                 |
|       - Finds .dll in packages/ directory                  |
|       - Runs: `ilspycmd <dll>` to decompile to .cs         |
|       - Records decompiled file locations                  |
|     - Stores Vec<Dependencies> in project.dependencies     |
+---------------------------+--------------------------------+
                            v
+-----------------------------------------------------------+
| Step 6: Wait for SDK loading task (if spawned)             |
|                                                            |
|   Awaits the JoinHandle from Step 2                        |
|   SDK XML files get parsed and added to graph via          |
|   load_sdk_xml_files_to_database()                         |
+---------------------------+--------------------------------+
                            v
+-----------------------------------------------------------+
| Step 7: Persist everything to SQLite                       |
|                                                            |
|   project.load_to_database()                               |
|     - For source-only mode:                                |
|       Load dependency XML files into graph                 |
|     - For full mode:                                       |
|       Load decompiled .cs files into graph                 |
|     - For each file in graph:                              |
|       1. Compute partial paths (stack-graphs stitching)    |
|       2. SQLiteWriter::store_result_for_file()             |
|     - Reload full graph from DB into memory                |
|     - Store in project.graph                               |
|                                                            |
|   Returns InitResponse { success: true }                   |
+-----------------------------------------------------------+
```

---

## 9. Evaluate (Query) Flow

```
Client calls Evaluate(EvaluateRequest)
  |
  |  EvaluateRequest contains:
  |    cap: "referenced"
  |    conditionInfo: '{"referenced":{"pattern":"System.Web.Mvc.*",
  |                     "location":"METHOD"}}'
  |
  v
+-----------------------------------------------------------+
| Step 1: Parse the condition                                |
|                                                            |
|   YAML/JSON -> CSharpCondition                             |
|     referenced:                                            |
|       pattern: "System.Web.Mvc.*"                          |
|       location: METHOD  (or CLASS, FIELD, All)             |
|       file_paths: [optional filter]                        |
+---------------------------+--------------------------------+
                            v
+-----------------------------------------------------------+
| Step 2: Create a Search object from the pattern            |
|                                                            |
|   Search::create_search("System.Web.Mvc.*")                |
|     - Splits on "." -> ["System", "Web", "Mvc", "*"]       |
|     - Each part becomes a SearchPart:                      |
|       SearchPart { part: "System", regex: None }           |
|       SearchPart { part: "Web",    regex: None }           |
|       SearchPart { part: "Mvc",    regex: None }           |
|       SearchPart { part: "*",      regex: Some(/.*/) }     |
|     - "*" gets compiled to regex ".*" (match anything)     |
|     - Regular text: exact string match (no regex)          |
+---------------------------+--------------------------------+
                            v
+-----------------------------------------------------------+
| Step 3: Choose query type based on location filter         |
|                                                            |
|   location=METHOD -> MethodSymbolsGetter                   |
|   location=CLASS  -> ClassSymbolsGetter                    |
|   location=FIELD  -> FieldSymbolsGetter                    |
|   location=All    -> NamespaceSymbolsGetter                |
|                                                            |
|   Each implements the SymbolsGetter trait:                 |
|     fn get_searchable_nodes(graph, search) -> Vec<Node>    |
|     fn get_fqdn(graph, node)              -> Fqdn         |
|     fn get_result_node(node, fqdn)        -> ResultNode   |
+---------------------------+--------------------------------+
                            v
+-----------------------------------------------------------+
| Step 4: The Querier traverses the StackGraph               |
|         (query.rs -- the core 1667-line search engine)     |
|                                                            |
|   Querier::find(graph, search, source_type, file_paths)    |
|                                                            |
|   Phase A -- Find candidate nodes:                         |
|     For each node in graph:                                |
|       - Check node has a symbol (name)                     |
|       - Check node belongs to a "source" file              |
|       - Check symbol matches last part of search pattern   |
|         e.g., search="System.Web.Mvc.*",                   |
|         node symbol="Controller"                           |
|         -> "*" matches "Controller"                        |
|       - If file_paths filter: check node's file is in list |
|       - Collect matching nodes                             |
|                                                            |
|   Phase B -- Resolve FQDN:                                 |
|     For each candidate node:                               |
|       - Walk graph edges upward to find:                   |
|         method -> class -> namespace                       |
|       - Build Fqdn struct:                                 |
|         Fqdn {                                             |
|           namespace: Some("System.Web.Mvc"),               |
|           class: Some("Controller"),                       |
|           method: Some("OnActionExecuting"),               |
|           field: None                                      |
|         }                                                  |
|       - Uses syntax_type tags set during graph building    |
|                                                            |
|   Phase C -- Match FQDN against search pattern:            |
|     For each FQDN:                                         |
|       - search.match_namespace(fqdn.namespace) must match  |
|       - search.match_symbol(fqdn.class/method/field) match |
|       - If match -> create ResultNode with file + line     |
|                                                            |
|   Phase D -- FQDN disambiguation via imports:              |
|     When multiple FQDNs resolve for same node:             |
|       - Collect `using` statements from the file           |
|       - select_best_fqdn() picks the one whose namespace  |
|         matches an import statement exactly                |
|       - e.g., file has `using System.Web.Mvc;`             |
|         -> prefer FQDN with namespace "System.Web.Mvc"     |
|         over "MyApp.Custom.Mvc"                            |
+---------------------------+--------------------------------+
                            v
+-----------------------------------------------------------+
| Step 5: Deduplicate results                                |
|                                                            |
|   deduplicate_results(results)                             |
|     - Groups by (file_uri, line_number)                    |
|     - For same file+line, keeps the TIGHTEST span          |
|       (smallest end_line - start_line)                     |
|     - Tiebreaker: earliest start character                 |
|     - Handles tree-sitter creating multiple nodes          |
|       for the same source location                         |
|                                                            |
|   Sort by (file_uri, line_number) for determinism          |
+---------------------------+--------------------------------+
                            v
+-----------------------------------------------------------+
| Step 6: Return results                                     |
|                                                            |
|   EvaluateResponse {                                       |
|     matched: true,                                         |
|     incident_contexts: [                                   |
|       IncidentContext {                                     |
|         file_uri: "file:///path/to/Controllers/Home.cs",   |
|         line_number: 42,                                   |
|         code_location: {                                   |
|           start: { line: 42, character: 16 },              |
|           end:   { line: 42, character: 38 }               |
|         },                                                 |
|         variables: { "location": "METHOD" }                |
|       },                                                   |
|       ...                                                  |
|     ]                                                      |
|   }                                                        |
+-----------------------------------------------------------+
```

---

## 10. The Stack Graph -- Core Data Structure

A **stack graph** is a directed graph where nodes represent C# symbols (classes, methods, fields, namespaces) and edges represent containment and reference relationships.

```
                         +---------------+
                         |   ROOT NODE   |
                         +-------+-------+
                                 |
                +----------------+------------------+
                |                                   |
     +----------v-----------+          +------------v-----------+
     | source_type_node     |          | dependency_type_node   |
     | (user's code)        |          | (third-party libs)     |
     +----------+-----------+          +------------+-----------+
                |                                   |
       +--------+--------+               +---------+---------+
       |                 |                |                   |
    +--v----+        +---v---+        +---v---+          +----v--+
    |File A |        |File B |        |Dep 1  |          |Dep 2  |
    |.cs    |        |.cs    |        |.cs    |          |.xml   |
    +--+----+        +--+----+        +-------+          +-------+
       |                |
    +--v----------------+
    | namespace:         |
    | "MyApp.Web"        |
    | [NamespaceDecl]    |
    +--+-----------------+
       |
    +--v-----------------+
    | class:              |
    | "HomeController"    |
    | [ClassDef]          |
    +--+-----------------+
       |
    +--v-----------------+
    | method:             |
    | "Index"             |
    | [MethodName]        |
    | line: 42            |
    +--------------------+
```

### Node Properties

Each node in the stack graph carries:

- **Symbol** — the identifier name (e.g., `"HomeController"`)
- **Syntax type** — what kind of C# construct it represents:

| SyntaxType | C# Construct | Example |
|---|---|---|
| `Import` | using directive | `using System.Web.Mvc;` |
| `CompUnit` | the file itself | `HomeController.cs` |
| `NamespaceDeclaration` | namespace block | `namespace MyApp.Web { }` |
| `ClassDef` | class definition | `class HomeController { }` |
| `MethodName` | method definition | `public ActionResult Index()` |
| `FieldName` | field/property | `private string _name;` |
| `LocalVar` | local variable | `var x = ...;` |
| `Argument` | method parameter | `(string name)` |

- **Source file** and **line/character position**
- **Source type** — whether this node comes from user code (`source`) or a dependency (`dependency`)

### Edge Types

| Edge Type | Precedence | Meaning |
|---|---|---|
| Containment | 0 | Parent-to-child (namespace contains class) |
| FQDN back-reference | 10 | Child-to-parent (used to resolve fully qualified names) |
| Reference | varies | Symbol usage pointing to definition |

---

## 11. Tree-Sitter + Stack Graphs Integration

This is the core of the analysis engine. The system uses a **three-phase pipeline** to build and query the stack graph from C# source code.

### 11.1 Phase 1: Tree-Sitter Parsing (C# Source -> Concrete Syntax Tree)

**Tree-sitter** is a parser generator. The `tree-sitter-c-sharp` crate provides a grammar that parses C# source into a Concrete Syntax Tree (CST) — a tree that represents every syntactic element in the source.

```
C# Source:                          Tree-Sitter CST:

using System.Web.Mvc;               compilation_unit
                                      +-- using_directive
namespace NerdDinner.Controllers {    |     +-- qualified_name: "System.Web.Mvc"
    public class DinnersController    +-- namespace_declaration
        : Controller {                      +-- name: "NerdDinner.Controllers"
        public ActionResult Index() {       +-- declaration_list
            return View();                        +-- class_declaration
        }                                              +-- name: "DinnersController"
    }                                                  +-- base_list: "Controller"
}                                                      +-- declaration_list
                                                             +-- method_declaration
                                                                  +-- name: "Index"
                                                                  +-- body: block
                                                                        +-- return_statement
                                                                              +-- invocation_expression
                                                                                    +-- "View"
```

Tree-sitter gives us the **structure**, but not the **semantics**. It doesn't know that `Controller` in the base list refers to `System.Web.Mvc.Controller`. That's what stack-graphs provide.

### 11.2 Phase 2: TSG Rules (CST -> Stack Graph)

The `stack-graphs.tsg` file (1061 lines) contains **declarative rules** that transform tree-sitter CST nodes into stack graph nodes and edges. The file is written in the TSG (Tree-Sitter Graph) domain-specific language. For a complete syntax reference, see [Section 12 (TSG Language Guide)](#12-tsg-language-guide).

#### Global Variables

```
global FILE_PATH              -- relative path of the file being processed
global PROJECT_NAME = ""      -- project name for isolation
global ROOT_PATH = ""         -- project root directory
global SOURCE_TYPE_NODE       -- injected node handle for source/dependency tagging
global JUMP_TO_SCOPE_NODE     -- scope navigation
global ROOT_NODE              -- graph root
```

#### Attribute Shorthands

The TSG file defines shorthand attributes that simplify rule writing:

```
attribute node_definition = node  => type = "pop_symbol", is_definition
attribute node_reference = node   => type = "push_symbol", is_reference
attribute fqdn_edge = edge        => precedence = 10
```

The `fqdn_edge` attribute is critical — it marks edges with **precedence 10** which the query engine later uses to walk upward from a symbol to reconstruct its fully qualified domain name.

#### Key TSG Rules (How Each C# Construct Becomes Graph Nodes)

**Rule 1: `using` directive -> Import node**

```
(using_directive
  [(identifier) @name (qualified_name) @name]
) @using {
  node @using.def
  attr (@using.def) symbol = (source-text @name), syntax_type = "import"
}
```

When tree-sitter finds a `using_directive`, this rule creates a graph node with:
- Symbol = the text of the name (e.g., `"System.Web.Mvc"`)
- Syntax type = `"import"`

**Rule 2: `compilation_unit` -> CompUnit node (the file itself)**

```
(compilation_unit) @comp_unit {
  node @comp_unit.def
  attr (@comp_unit.def) symbol = FILE_PATH, syntax_type = "comp_unit"
  edge SOURCE_TYPE_NODE -> @comp_unit.def    -- links file to source/dependency
  edge ROOT_NODE -> SOURCE_TYPE_NODE         -- links to graph root
}
```

This rule creates the file-level node and connects it to the `SOURCE_TYPE_NODE` (which marks whether this file is user source or a dependency). The edge chain is: `ROOT -> SOURCE_TYPE -> comp_unit`.

**Rule 3: `namespace_declaration` -> NamespaceDeclaration node**

```
(namespace_declaration
  name: [(identifier) @namespace (qualified_name) @namespace]
) @decl {
  node @decl.def
  attr (@decl.def) symbol = (source-text @namespace),
                   syntax_type = "namespace_declaration"
}
```

**Rule 4: `class_declaration` -> ClassDef node**

```
(class_declaration
  name: (identifier) @classname
) @class_declaration {
  node @class_declaration.def
  attr (@class_declaration.def) symbol = (source-text @classname),
                                syntax_type = "class_def"
}
```

**Rule 5: Connecting classes to namespaces (containment + FQDN edges)**

```
(namespace_declaration
  body: (declaration_list
    (class_declaration) @class_declaration
  )
) @namespace {
    edge @namespace.def -> @class_declaration.def          -- containment (down)
    edge @class_declaration.def -> @namespace.def          -- FQDN back-ref (up)
    attr (@class_declaration.def -> @namespace.def) fqdn_edge  -- precedence = 10
}
```

This is the crucial rule. It creates **two edges**:
1. **Containment edge** (precedence 0): namespace -> class (for traversal downward)
2. **FQDN edge** (precedence 10): class -> namespace (for resolving the fully qualified name upward)

The same pattern repeats for methods, fields, constructors, and properties inside classes:

```
(class_declaration
  body: (declaration_list
    (method_declaration)? @method_declaration
    (field_declaration)? @field_declaration
    (property_declaration)? @property_declaration
    (constructor_declaration)? @constructor_declaration
  )
) @class_declaration {
  if some @method_declaration {
    edge @class_declaration.def -> @method_declaration.def       -- down
    edge @method_declaration.def -> @class_declaration.def       -- up (FQDN)
    attr (@method_declaration.def -> @class_declaration.def) fqdn_edge
  }
  -- (same pattern for fields, properties, constructors)
}
```

**Rule 6: `method_declaration` -> MethodName node**

```
(method_declaration
  name: (identifier) @method_name
) @decl {
  node @decl.def
  attr (@decl.def) symbol = (source-text @method_name),
                   syntax_type = "method_name"
}
```

**Rule 7: `member_access_expression` -> Reference node (e.g., `ConfigurationManager.AppSettings`)**

```
(member_access_expression
  expression: [(identifier) (predefined_type)] @expr
  name: (_) @name
) @mem_expr {
  attr (@mem_expr.def) symbol = (format "{}.{}" (source-text @expr) (source-text @name)),
                       is_reference
}
```

This rule is interesting — for `ConfigurationManager.AppSettings`, it creates a **single node** with symbol `"ConfigurationManager.AppSettings"`. The query engine then splits this on `.` to resolve the accessor and the accessed member.

**Rule 8: `variable_declaration` -> LocalVar node with type reference**

```
(variable_declaration
    type: (_) @type
    (variable_declarator
        name: (identifier) @name
    )
) @var_decl {
    node @var_decl.def
    node var_type
    attr (@var_decl.def) symbol = (source-text @name), syntax_type = "local_var"
    attr (var_type) symbol = (source-text @type), is_reference
    edge @var_decl.def -> var_type
}
```

Local variables get both a definition node and a **type reference edge**, allowing the query engine to resolve what type a variable is and follow member accesses through it.

#### Statement and Expression Handling (Lines 247-1061)

The rest of the TSG file handles every C# statement and expression type to ensure all code paths are represented in the graph:

- **Statements** (lines 249-287): `block`, `if_statement`, `for_statement`, `foreach_statement`, `while_statement`, `switch_statement`, `try_statement`, `catch_clause`, `return_statement`, `using_statement`, `lock_statement`, etc.
- **Expressions** (lines 749-1061): `invocation_expression`, `object_creation_expression`, `member_access_expression`, `binary_expression`, `lambda_expression`, `conditional_expression`, `cast_expression`, `query_expression`, etc.

Each creates a node and connects it to its children, building a full graph of every code path in the file.

### 11.3 Phase 3: Loading Pipeline (How Files Become a Persisted Graph)

The loading pipeline is implemented in `loader.rs` and `language_config.rs`:

```
+-----------------------------------------------------------------+
|  SourceNodeLanguageConfiguration::new()  (language_config.rs)    |
|                                                                  |
|  1. Create StackGraphLanguage from TSG rules + C# grammar       |
|     sgl = StackGraphLanguage::from_source(                      |
|       tree_sitter_c_sharp::LANGUAGE,                             |
|       STACK_GRAPHS_TSG_SOURCE                                    |
|     )                                                            |
|                                                                  |
|  2. Create builtins graph                                        |
|     builtins = StackGraph::new()                                 |
|     Load source_type and dependency_type symbols                 |
|     Add "<builtins>" virtual file                                |
|     Build builtins.cs through the TSG rules                      |
|                                                                  |
|  3. Create LanguageConfiguration                                 |
|     language: tree_sitter_c_sharp::LANGUAGE                      |
|     file_types: ["cs"]                                           |
|     sgl: the compiled TSG rules                                  |
|     builtins: the builtins graph                                 |
+-----------------------------------------------------------------+
                            |
                            v
+-----------------------------------------------------------------+
|  init_stack_graph()  (loader.rs)                                 |
|                                                                  |
|  For each .cs file in the project directory:                     |
|                                                                  |
|    load_graph_for_file(path, graph, config, source_type)         |
|    |                                                              |
|    | 1. Check if file matches ".cs" extension                    |
|    |    or is a registered special file (XML analyzer)            |
|    |                                                              |
|    | 2. Read file content, compute SHA1 hash (for change detect) |
|    |                                                              |
|    | 3. Add file to StackGraph: graph.add_file(path)             |
|    |                                                              |
|    | 4. Load source_type node into graph                         |
|    |    (marks this file as "source" or "dependency")            |
|    |                                                              |
|    | 5a. If special file (XML): use DepXMLFileAnalyzer           |
|    |     analyzer.build_stack_graph_into(graph, file, source)    |
|    |                                                              |
|    | 5b. If normal .cs file: use TSG rules                       |
|    |     builder = sgl.builder_into_stack_graph(graph, file, src)|
|    |     builder.inject_node(source_type_node_id)                |
|    |       -- injects SOURCE_TYPE_NODE global variable            |
|    |     builder.build(&globals, &NoCancellation)                |
|    |       -- tree-sitter parses the source                      |
|    |       -- TSG rules fire on each CST node                    |
|    |       -- graph nodes and edges are created                  |
|    |                                                              |
|    | 6. Store result to SQLite                                   |
|    |    db.store_result_for_file(graph, file_handle, tag, ...)   |
|                                                                  |
+-----------------------------------------------------------------+
```

### 11.4 Source Type Tagging

Every file loaded into the graph is tagged as either **source** (user code) or **dependency** (third-party library). This is done through the `SourceType` enum in `loader.rs`:

```
enum SourceType {
    Source     { symbol_handle }   -- "konveyor.io/source_type=source"
    Dependency { symbol_handle }   -- "konveyor.io/source_type=dependency"
}
```

The source type is injected into the TSG rules as the `SOURCE_TYPE_NODE` global variable. The compilation_unit rule then creates edges:

```
ROOT_NODE -> SOURCE_TYPE_NODE -> comp_unit_node
```

This lets the query engine filter results to only user code (source) or include dependencies.

### 11.5 FQDN Resolution (How the Query Engine Walks the Graph)

When the query engine finds a candidate node (e.g., a `MethodName` node with symbol `"Index"`), it needs to resolve its **Fully Qualified Domain Name** to check if it matches the search pattern.

The `get_fqdn()` function in `query.rs` does this by walking **precedence-10 edges** upward:

```
get_fqdn(method_node "Index")
  |
  | Follow precedence-10 edge upward
  v
get_fqdn(class_node "DinnersController")
  |
  | Follow precedence-10 edge upward
  v
get_fqdn(namespace_node "NerdDinner.Controllers")
  |
  | No more precedence-10 edges -- base case
  v
Return Fqdn { namespace: "NerdDinner.Controllers" }

-- Back at class level:
Return Fqdn { namespace: "NerdDinner.Controllers", class: "DinnersController" }

-- Back at method level:
Return Fqdn {
  namespace: "NerdDinner.Controllers",
  class: "DinnersController",
  method: "Index"
}
```

The recursion works by:
1. Getting the node's symbol and syntax_type
2. Collecting outgoing edges with `precedence == 10` (FQDN edges)
3. Sorting edges by sink handle for determinism
4. Recursing on the first FQDN edge's sink node
5. Filling in the appropriate FQDN field based on syntax_type:
   - `NamespaceDeclaration` -> sets `fqdn.namespace`
   - `ClassDef` -> sets `fqdn.class`
   - `MethodName` -> sets `fqdn.method`
   - `FieldName` -> sets `fqdn.field`

### 11.6 Member Access Resolution

For expressions like `ConfigurationManager.AppSettings`, the TSG creates a single node with symbol `"ConfigurationManager.AppSettings"`. The query engine's `get_type_with_symbol()` function handles this:

```
Symbol: "ConfigurationManager.AppSettings"
  |
  | Split on "." -> accessor="ConfigurationManager", accessed="AppSettings"
  |
  | 1. Find "ConfigurationManager" in the file's known definitions
  | 2. Check its syntax_type:
  |    - ClassDef? Follow outgoing edges to find child "AppSettings"
  |    - LocalVar? Resolve the variable's type, then find "AppSettings" on it
  | 3. Get FQDN of the resolved child node
  | 4. Use file imports to disambiguate if multiple candidates exist
  |    select_best_fqdn() picks the one matching a `using` statement
```

### 11.7 Builtins

The `builtins.cfg` file sets the `SOURCE_TYPE_NODE` variable to `"<builtin>"` for the virtual builtins file. The `builtins.cs` file (currently empty/minimal) defines built-in C# types that are always available. These get processed through the same TSG rules and become part of every graph.

### 11.8 Dependency XML Analyzer (Alternative to TSG for XML files)

For SDK XML documentation files (which aren't C# source), the `DepXMLFileAnalyzer` in `dependency_xml_analyzer.rs` implements the `FileAnalyzer` trait and builds graph nodes directly — bypassing tree-sitter entirely:

```
XML Input:                              Graph Nodes Created:
<member name="T:System.String">         ClassDef("String") + Namespace("System")
<member name="M:System.String.Format">  MethodName("Format") + ClassDef("String") + Namespace("System")
<member name="F:System.Int32.MaxValue"> FieldName("MaxValue") + ClassDef("Int32") + Namespace("System")
<member name="P:System.String.Length">  FieldName("Length") + ClassDef("String") + Namespace("System")
<member name="N:System.Configuration">  Namespace("System.Configuration")
```

The member type prefix (`T`, `M`, `F`, `P`, `N`) determines how the dotted name is split into namespace, class, and member components. Edges are created with the same precedence scheme (0 for down, 10 for FQDN up) so FQDN resolution works identically.

---

## 12. TSG Language Guide

This section is a complete reference for the TSG (Tree-Sitter Graph) DSL as used in `src/c_sharp_graph/stack-graphs.tsg`. For how these rules fit into the overall pipeline, see [Section 11 (Tree-Sitter + Stack Graphs Integration)](#11-tree-sitter--stack-graphs-integration).

### 12.1 File Structure

TSG files use semicolon comments (`;; comment`) and consist of three parts in order:

1. **Global variable declarations** — values injected by the Rust runtime
2. **Attribute shorthands** — macros that expand into multiple `attr` properties
3. **Match rules** — pattern-action blocks that fire when CST nodes match

**Execution model:** When a `.cs` file is parsed by tree-sitter, the resulting CST is walked. For each CST node, all matching TSG rules fire (order within the file does not matter). A single CST node can trigger multiple rules.

### 12.2 Global Variables

**Syntax:** `global NAME` or `global NAME = "default_value"`

Globals are injected by the Rust runtime before rules execute. They are read-only within TSG rules.

| Global | Injected By | Value |
|---|---|---|
| `FILE_PATH` | `loader.rs` | Relative path of the file being processed |
| `PROJECT_NAME` | `loader.rs` | Project name (default: `""`) |
| `ROOT_PATH` | `loader.rs` | Project root directory (default: `""`) |
| `SOURCE_TYPE_NODE` | `loader.rs` | Handle to the source/dependency type node |
| `JUMP_TO_SCOPE_NODE` | stack-graphs library | Internal scope navigation node |
| `ROOT_NODE` | stack-graphs library | Root node of the graph |

### 12.3 Attribute Shorthands

**Syntax:** `attribute shorthand_name = parameter => key1 = value1, key2 = value2, ...`

Shorthands are macros that expand into multiple `attr` key-value pairs when used.

| Shorthand | Parameter | Expands To |
|---|---|---|
| `node_definition` | node | `type = "pop_symbol"`, `node_symbol = node`, `is_definition` |
| `node_reference` | node | `type = "push_symbol"`, `node_symbol = node`, `is_reference` |
| `pop_symbol` | symbol | `type = "pop_symbol"`, `symbol = symbol` |
| `push_symbol` | symbol | `type = "push_symbol"`, `symbol = symbol` |
| `scoped_node_definition` | node | `type = "pop_scoped_symbol"`, `node_symbol = node`, `is_definition` |
| `symbol_reference` | symbol | `type = "push_symbol"`, `symbol = symbol`, `is_reference` |
| `node_symbol` | node | `symbol = (source-text node)`, `source_node = node` |
| `fqdn_edge` | edge | `precedence = 10` |

**Example:** `attr (@using.def) node_definition = @using` expands to `attr (@using.def) type = "pop_symbol", symbol = (source-text @using), source_node = @using, is_definition`.

### 12.4 Match Blocks (The Core Construct)

Match blocks are the fundamental unit of a TSG file. They pair a tree-sitter query pattern with actions.

**Basic syntax:**

```
(tree_sitter_node_type
  field_name: (child_type) @capture
) @outer_capture {
  ;; actions go here: node, attr, edge statements
}
```

The parenthesized pattern is a tree-sitter query. `@name` tokens are **captures** that bind to matched CST nodes. The braces contain actions that create graph nodes and edges.

**Alternation (match any of several types):**

```
(using_directive
  [(identifier) @name (qualified_name) @name]
) @using { ... }
```

Square brackets mean "match any of these alternatives." Here, the `name` can be either an `identifier` or a `qualified_name`.

**Optional children:**

```
(method_declaration
  body: (_)? @body
) @decl {
  if some @body {
    edge @decl.def -> @body.def
  }
}
```

The `?` makes the child optional. Use `if some @capture` to check whether it matched.

**Wildcard type:**

```
(variable_declaration
  type: (_) @type
) @var_decl { ... }
```

`(_)` matches any node type.

**Multiple node types in one rule:**

```
[
  (block)
  (if_statement)
  (for_statement)
] @stmt {
  node @stmt.def
}
```

A single rule that applies to multiple CST node types.

### 12.5 Node Declarations

**Syntax:** `node @capture.property` or `node local_name`

Creates a new stack graph node. Two forms:

- `node @capture.def` — creates a node associated with a capture. The name `@capture.def` is accessible in this rule and in other rules that match the same capture for the same CST node.
- `node var_type` — creates a local anonymous node. Only usable within this rule block.

### 12.6 Attribute Statements

**Syntax:** `attr (node) key = value, key = value, ...`

Sets properties on a graph node. Core properties:

| Property | Values | Meaning |
|---|---|---|
| `type` | `"pop_symbol"`, `"push_symbol"`, `"pop_scoped_symbol"`, `"push_scoped_symbol"` | Node type in the stack graph |
| `symbol` | string or `(source-text @capture)` | The symbol name for this node |
| `source_node` | `@capture` | Which CST node this graph node corresponds to (for source location) |
| `is_definition` | (flag, no value) | Marks this node as defining a symbol |
| `is_reference` | (flag, no value) | Marks this node as referencing a symbol |
| `is_exported` | (flag, no value) | Marks a scope as exported (visible outside) |
| `syntax_type` | `"import"`, `"comp_unit"`, `"class_def"`, `"method_name"`, `"field_name"`, `"local_var"`, `"argument"`, `"name"`, `"namespace_declaration"` | Custom tag used by the query engine |

Attributes on **edges** set edge properties:

```
attr (@method.def -> @class.def) fqdn_edge   ;; sets precedence = 10
attr (@method.def -> @class.def) precedence = 5  ;; direct precedence
```

### 12.7 Edge Statements

**Syntax:** `edge SOURCE -> SINK`

Creates a directed edge in the stack graph. The dual-edge pattern used throughout the file:

```
;; Containment (parent -> child), precedence 0 (default)
edge @namespace.def -> @class.def

;; FQDN back-reference (child -> parent), precedence 10
edge @class.def -> @namespace.def
attr (@class.def -> @namespace.def) fqdn_edge
```

Every containment relationship creates **two edges**: one down (for traversal) and one up (for FQDN resolution).

### 12.8 Conditional Blocks

**Syntax:** `if some @capture { ... }`

Executes the block only if the optional capture matched a node:

```
(method_declaration
  body: (_)? @body
  parameters: (parameter_list)? @list
) @decl {
  if some @body {
    edge @decl.def -> @body.def
  }
  if some @list {
    edge @decl.def -> @list.def
  }
}
```

Also supports negation:

```
if (not (eq "comment" (node-type @expr))) {
  edge @stmt.def -> @expr.def
}
```

### 12.9 Built-in Functions

| Function | Syntax | Returns |
|---|---|---|
| `source-text` | `(source-text @capture)` | The source code text of the captured CST node |
| `format` | `(format "{}.{}" (source-text @a) (source-text @b))` | A formatted string with `{}` placeholders |
| `node-type` | `(node-type @capture)` | The tree-sitter node type name as a string |
| `eq` | `(eq "value" (node-type @x))` | Boolean equality test |
| `not` | `(not (eq ...))` | Boolean negation |

### 12.10 Scoped Variables

Each capture has two implicitly available scoped properties:

- `@capture.def` — the definition node for this capture
- `@capture.lexical_scope` — the lexical scope node for this capture

Both must be explicitly created with `node @capture.def` / `node @capture.lexical_scope` before use. They are **shared across all rules** that match the same capture name for the same CST node — this is how multiple rules collaborate on the same node.

### 12.11 Symbol Types

| Symbol Type | TSG `type` value | Meaning | Example |
|---|---|---|---|
| Definition | `pop_symbol` | "I define a name" | class, method, field declarations |
| Reference | `push_symbol` | "I use a name" | member access, identifier usage |
| Scoped definition | `pop_scoped_symbol` | Definition with scope attachment | (used in shorthands) |
| Scoped reference | `push_scoped_symbol` | Reference with scope | using directive references |

### 12.12 Complete List of syntax_type Values

| `syntax_type` value | C# construct | TSG rule location (approximate lines) |
|---|---|---|
| `"import"` | `using` directive | 49-62 |
| `"comp_unit"` | `compilation_unit` (the file) | 64-69 |
| `"namespace_declaration"` | `namespace` block | 84-94 |
| `"class_def"` | `class` declaration | 96-103 |
| `"method_name"` | method declaration | 154-160 |
| `"field_name"` | field or property declaration | 221-226 |
| `"local_var"` | variable declaration | 228-245 |
| `"argument"` | named argument | 497-504 |
| `"name"` | generic name / identifier | 527-535 |

---

## 13. Query Type System

Each query type implements the `GetMatcher` trait, which produces a `SymbolMatcher`:

```
trait GetMatcher {
    type Matcher: SymbolMatcher;
    fn get_matcher(graph, root_nodes, search) -> Result<Matcher>;
}

trait SymbolMatcher {
    fn match_symbol(symbol: &str) -> bool;
    fn match_fqdn(fqdn: &Fqdn) -> bool;
}
```

### Query Types

```
+--------------------------+-------------------------------------------------+
| Query Type               | What it searches for                            |
+--------------------------+-------------------------------------------------+
| NamespaceSymbolsGetter   | All nodes in matching namespaces                |
| (location=All)           | Checks: imports, namespace decls, usages        |
|                          | Returns: any reference in that namespace         |
+--------------------------+-------------------------------------------------+
| ClassSymbolsGetter       | Class definition nodes                          |
| (location=CLASS)         | Checks: ClassDef syntax type                    |
|                          | Returns: where class is defined/used            |
+--------------------------+-------------------------------------------------+
| MethodSymbolsGetter      | Method name nodes                               |
| (location=METHOD)        | Checks: MethodName syntax type                  |
|                          | Returns: where method is called/defined         |
+--------------------------+-------------------------------------------------+
| FieldSymbolsGetter       | Field/property nodes                            |
| (location=FIELD)         | Checks: FieldName syntax type                   |
|                          | Returns: where field is accessed                |
+--------------------------+-------------------------------------------------+
```

### How Each Query Type Traverses the Graph

All query types follow a consistent pattern:

1. Start from root node(s) in the stack graph
2. Traverse outgoing edges (skip FQDN edges at precedence 10)
3. Check each node's syntax type for the target type
4. Apply the search pattern filter to the node's symbol
5. Resolve the FQDN by following precedence-10 edges upward
6. Validate the full FQDN against the search pattern
7. Store matching nodes in a `BTreeMap<Fqdn, Handle<Node>>` for deterministic ordering

The `NamespaceSymbolsGetter` is the most general — it also creates nested `ClassSymbols`, `MethodSymbols`, and `FieldSymbols` matchers and delegates to them.

---

## 14. Search Pattern Matching Engine

The `Search` struct (in `query.rs`) handles pattern matching with dot-separated parts and wildcard support.

### Pattern Compilation

```
Pattern: "System.Web.Mvc.*"

Search {
  parts: [
    SearchPart { part: "System", regex: None },       // exact match
    SearchPart { part: "Web",    regex: None },       // exact match
    SearchPart { part: "Mvc",    regex: None },       // exact match
    SearchPart { part: "*",      regex: Some(/.*/) }  // wildcard
  ]
}
```

### Three Matching Modes

| Method | Example | Result | Purpose |
|---|---|---|---|
| `partial_namespace("System.Web")` | `true` | Prefix check — does the search start with these parts? |
| `match_namespace("System.Web.Mvc")` | `true` | Full namespace match with wildcard support |
| `match_symbol("Controller")` | `true` | Matches last part of pattern against a symbol name |

### Wildcard Handling

The `*` wildcard in the middle of a pattern uses a lookahead algorithm to skip intermediate segments:

```
Pattern: "System.*.Mvc"
  matches: "System.Web.Mvc"          YES
  matches: "System.Anything.Mvc"     YES
  matches: "System.Web.Http"         NO

Pattern: "System.Web.*"
  matches: "System.Web.Mvc"          YES
  matches: "System.Web.Http"         YES
  matches: "System.IO.File"          NO

Pattern: "System.*.*"
  matches: "System.Web.Mvc"          YES
  matches: "System.IO.File"          YES
  matches: "Other.Web.Mvc"           NO
```

---

## 15. Worked Example (Detailed)

This section traces the complete processing of a single C# file through every stage: parsing, graph construction, and querying. For the concepts behind each phase, see [Section 4 (Concepts Primer)](#4-concepts-primer) and [Section 11 (Tree-Sitter + Stack Graphs Integration)](#11-tree-sitter--stack-graphs-integration).

### 15.1 The Source File

```csharp
// File: testdata/nerd-dinner/mvc4/NerdDinner/App_Start/FilterConfig.cs
using System.Web;
using System.Web.Mvc;

namespace NerdDinner
{
    public class FilterConfig
    {
        public static void RegisterGlobalFilters(GlobalFilterCollection filters)
        {
            filters.Add(new HandleErrorAttribute());
        }
    }
}
```

### 15.2 Phase 1: Tree-Sitter Parsing (Source -> CST)

Tree-sitter-c-sharp parses this into the following CST (node types and fields shown):

```
compilation_unit                                    (lines 1-13)
  using_directive                                   (line 1)
    qualified_name "System.Web"
  using_directive                                   (line 2)
    qualified_name "System.Web.Mvc"
  namespace_declaration                             (lines 4-13)
    name: identifier "NerdDinner"
    body: declaration_list
      class_declaration                             (lines 6-12)
        name: identifier "FilterConfig"
        body: declaration_list
          method_declaration                        (lines 8-11)
            type: predefined_type "void"
            name: identifier "RegisterGlobalFilters"
            parameters: parameter_list
              parameter
                type: identifier "GlobalFilterCollection"
                name: identifier "filters"
            body: block
              expression_statement
                invocation_expression
                  function: member_access_expression
                    expression: identifier "filters"
                    name: identifier "Add"
                  arguments: argument_list
                    object_creation_expression
                      type: identifier "HandleErrorAttribute"
```

### 15.3 Phase 2: TSG Rules Fire (CST -> Stack Graph)

For each CST node, matching TSG rules fire. Here is every significant rule activation:

| # | CST Node | TSG Rule | Graph Node Created | Key Properties |
|---|---|---|---|---|
| 1 | `using_directive` "System.Web" | line 49 | `N1: @using.def` | symbol=`"System.Web"`, syntax_type=`"import"` |
| 2 | `using_directive` "System.Web.Mvc" | line 49 | `N2: @using.def` | symbol=`"System.Web.Mvc"`, syntax_type=`"import"` |
| 3 | `compilation_unit` | line 64 | `N3: @comp_unit.def` | symbol=`"App_Start/FilterConfig.cs"`, syntax_type=`"comp_unit"` |
| 4 | `compilation_unit` (2nd rule) | line 71 | (edges only) | Edges: N3->N1, N3->N2, N3->N4 |
| 5 | `namespace_declaration` | line 84 | `N4: @decl.def` | symbol=`"NerdDinner"`, syntax_type=`"namespace_declaration"` |
| 6 | `class_declaration` | line 96 | `N5: @class_declaration.def` | symbol=`"FilterConfig"`, syntax_type=`"class_def"` |
| 7 | namespace->class containment | line 106 | (edges only) | N4->N5 (prec 0), N5->N4 (prec 10 FQDN) |
| 8 | `method_declaration` | line 154 | `N6: @decl.def` | symbol=`"RegisterGlobalFilters"`, syntax_type=`"method_name"` |
| 9 | class->method containment | line 117 | (edges only) | N5->N6 (prec 0), N6->N5 (prec 10 FQDN) |
| 10 | `member_access_expression` | line 796 | `N7: @mem_expr.def` | symbol=`"filters.Add"`, type=`"push_symbol"` |
| 11 | `object_creation_expression` | line 814 | `N8: @obj.def` | symbol=`"HandleErrorAttribute"`, type=`"push_symbol"` |

### 15.4 Phase 3: The Resulting Graph

```
ROOT_NODE
  |
  v (prec 0)
SOURCE_TYPE_NODE ("konveyor.io/source_type=source")
  |
  v (prec 0)
N3: comp_unit "App_Start/FilterConfig.cs"
  |
  +---> N1: import "System.Web"
  +---> N2: import "System.Web.Mvc"
  +---> N4: namespace_declaration "NerdDinner"
              |                          ^
              | (prec 0)                 | (prec 10 = FQDN)
              v                          |
              N5: class_def "FilterConfig"
                    |                          ^
                    | (prec 0)                 | (prec 10 = FQDN)
                    v                          |
                    N6: method_name "RegisterGlobalFilters"
                          |
                          +---> N7: ref "filters.Add"
                          +---> N8: ref "HandleErrorAttribute"
```

### 15.5 Phase 4: Query Against This Graph

**Query:** `{"referenced": {"pattern": "System.Web.Mvc.*"}}`

**Step 1 -- Pattern compilation:**

```
Search { parts: ["System", "Web", "Mvc", "*"] }
```

The `*` wildcard matches any symbol name.

**Step 2 -- Scan all "source" nodes, check if last search part matches symbol:**

| Node | Symbol | Last part `*` matches? | Proceed? |
|---|---|---|---|
| N1 | `"System.Web"` | Yes (import node) | Yes |
| N2 | `"System.Web.Mvc"` | Yes (import node) | Yes |
| N3 | `"App_Start/FilterConfig.cs"` | comp_unit, skipped | No |
| N4 | `"NerdDinner"` | Yes | Yes |
| N5 | `"FilterConfig"` | Yes | Yes |
| N6 | `"RegisterGlobalFilters"` | Yes | Yes |
| N7 | `"filters.Add"` | Yes | Yes |
| N8 | `"HandleErrorAttribute"` | Yes | Yes |

**Step 3 -- Resolve FQDN for each candidate and match against full pattern:**

| Node | FQDN Resolution | Full FQDN | Matches `System.Web.Mvc.*`? |
|---|---|---|---|
| N1 | import, symbol is `"System.Web"` | namespace=`"System.Web"` | No (`"System.Web"` != `"System.Web.Mvc"`) |
| N2 | import, symbol is `"System.Web.Mvc"` | namespace=`"System.Web.Mvc"` | Yes (exact namespace match) |
| N4 | no prec-10 edges up | namespace=`"NerdDinner"` | No |
| N5 | prec-10 -> N4 | namespace=`"NerdDinner"`, class=`"FilterConfig"` | No |
| N6 | prec-10 -> N5 -> N4 | namespace=`"NerdDinner"`, class=`"FilterConfig"`, method=`"RegisterGlobalFilters"` | No |
| N7 | member access, resolves via imports | depends on import disambiguation | Possibly (see below) |
| N8 | object creation, resolves via imports | depends on import disambiguation | Possibly |

For N7 (`filters.Add`): The query engine splits on `.` and tries to resolve `filters` as a type, then find `Add` on it. Since `filters` is a parameter (not a class), this resolution may not find a match in `System.Web.Mvc`.

For N8 (`HandleErrorAttribute`): The query engine checks if any import matches. File has `using System.Web.Mvc;`, and `HandleErrorAttribute` exists in that namespace in the dependency graph, so this would match.

**Step 4 -- Result:**

```
EvaluateResponse {
  matched: true,
  incident_contexts: [
    { file: "FilterConfig.cs", line: 2, symbol: "System.Web.Mvc" },
    ...
  ]
}
```

---

## 16. Dependency Resolution Pipeline

### Full Mode (decompile dependencies)

```
+------------------+     +-------------+     +----------------+
|  .csproj files   |---->|   Paket     |---->| NuGet Packages |
|  (list deps)     |     |  (resolver) |     | (.nupkg)       |
+------------------+     +-------------+     +--------+-------+
                                                      |
                                                extract .dll
                                                      |
                                                      v
                                               +------+-------+
                                               |    ILSpy      |
                                               |  (decompiler) |
                                               +------+--------+
                                                      |
                                            decompile to .cs
                                                      |
                                                      v
                                               +------+--------+
                                               | Decompiled     |
                                               |  .cs files     |
                                               +------+---------+
                                                      |
                                            tree-sitter parse
                                            + add to graph as
                                            "dependency" nodes
                                                      |
                                                      v
                                               +------+---------+
                                               |  Stack Graph   |
                                               |  (combined)    |
                                               +----------------+
```

### Source-Only Mode (SDK XML docs)

Source-only mode skips decompilation entirely. Instead, it parses **SDK XML documentation files** (the `.xml` files that ship with the .NET SDK alongside reference assemblies):

```xml
<!-- SDK XML documentation format -->
<doc>
  <members>
    <member name="T:System.String">...</member>           Type
    <member name="M:System.String.Format(...)">...</member>  Method
    <member name="F:System.Int32.MaxValue">...</member>      Field
    <member name="P:System.String.Length">...</member>       Property
    <member name="N:System.Configuration">...</member>       Namespace
  </members>
</doc>
```

The prefix letter (`T`, `M`, `F`, `P`, `N`) tells the `DepXMLFileAnalyzer` what kind of symbol it is. The analyzer creates graph nodes accordingly — without needing actual source code.

### Dependencies Struct

```
Dependencies {
    location:            PathBuf              // Package directory
    name:                String               // Package name
    version:             String               // Package version
    highest_restriction: String               // Target framework restriction
    decompiled_location: Arc<Mutex<HashSet<PathBuf>>>  // Decompiled file paths
    decompiled_size:     Mutex<Option<u64>>    // Size tracking
}
```

### Resolution Steps (detail)

1. `paket convert-from-nuget` — converts `.csproj` NuGet references to Paket format
2. `paket install` — resolves and downloads packages
3. Parse `paket.lock` — extract dependency list with versions and framework restrictions
4. For each dependency:
   - Read `paket-installmodel.cache` to find `.dll` paths matching the framework
   - In **full mode**: run `ilspycmd` to decompile each `.dll` to `.cs` files
   - In **source-only mode**: locate `.xml` documentation files instead
5. Load results into the stack graph and persist to SQLite

---

## 17. SDK Detection & Target Framework

### Target Framework Detection

The `TargetFrameworkHelper` scans `.csproj` files to detect which .NET version the project targets:

```
.csproj XML                          Normalized TFM
--------------------------------------------------
<TargetFramework>net8.0</...>     -> "net8.0"
<TargetFrameworkVersion>v4.5</..> -> "net45"
<TargetFramework>net6.0-android</>-> "net6.0"  (strip platform suffix)
<TargetFramework>netcoreapp3.1</> -> "netcoreapp3.1"
<TargetFramework>netstandard2.0</>-> "netstandard2.0"
```

When multiple `.csproj` files exist, the **earliest/lowest** TFM is selected (lexicographic comparison).

### SDK Detection

The `SdkDetector` searches for an installed .NET SDK in this order:

```
1. User-configured path (providerSpecificConfig.dotnet_sdk_path)
      |
      v (not found or invalid?)
2. DOTNET_ROOT environment variable
      |
      v (not set?)
3. Platform-specific default paths:
      Linux:  /usr/share/dotnet, /usr/lib/dotnet, ~/.dotnet
      macOS:  /usr/local/share/dotnet, ~/.dotnet
      Windows: C:\Program Files\dotnet, C:\Program Files (x86)\dotnet
      |
      v (none found?)
4. Fall back to dotnet-install script
      Runs: dotnet-install.sh -Channel 8.0 --install-dir /tmp/dotnet-sdks/net8.0
```

### SDK Validation

A found SDK path is validated by checking for the presence of reference packs:

```
{sdk_path}/packs/Microsoft.NETCore.App.Ref/{version}/ref/{tfm}/
```

If this directory exists and contains `.xml` files, the SDK is considered valid for the target framework.

---

## 18. SQLite Caching Layer

```
+--------------------------------------------------+
|                   SQLite DB                       |
|                                                   |
|  Stores:                                          |
|    - Serialized StackGraph nodes + edges          |
|    - PartialPaths (precomputed path segments)     |
|    - File-to-tag mappings (source vs dependency)  |
|                                                   |
|  Write Operations:                                |
|    SQLiteWriter::open(path)                       |
|    writer.store_result_for_file(graph, file, ..)  |
|    writer.clean_file(file)  <-- on file change    |
|                                                   |
|  Read Operations:                                 |
|    SQLiteReader::open(path)                       |
|    reader.load_graphs_for_file_or_directory(..)   |
|    reader.get() -> (graph, partials, database)    |
|                                                   |
|  Purpose:                                         |
|    - Avoid re-parsing all files on restart         |
|    - Incremental updates (only reparse changed)   |
|    - Shared state across dependency loading tasks  |
+--------------------------------------------------+
```

### File Tagging

Each file stored in the database is tagged with a SHA1 hash of its content. This enables change detection — if the hash changes, the file is re-indexed.

### Partial Path Stitching

The stack-graphs library computes **partial paths** — precomputed path segments through the graph that can be stitched together at query time for efficient symbol resolution. These partial paths are stored in SQLite alongside the graph data.

---

## 19. File Change Notification Flow

```
Client calls NotifyFileChanges([file1.cs, file2.csproj])
  |
  v
+-----------------------------------------------+
| Filter: Only process .cs and .csproj files     |
| Convert file:// URIs -> filesystem paths       |
+----------------------+------------------------+
                       v
+-----------------------------------------------+
| For each changed .cs file:                     |
|   1. SQLiteWriter::open(db_path)               |
|   2. db.clean_file(file)  <-- remove old data  |
|   3. load_and_store_file(file, graph, ..)      |
|      -> re-parse with tree-sitter              |
|      -> rebuild stack graph nodes              |
|      -> store new results in SQLite            |
+----------------------+------------------------+
                       v
+-----------------------------------------------+
| Reload entire graph from SQLite                |
|   SQLiteReader -> deserialize -> new graph     |
|   Replace project.graph with new graph         |
|                                                |
| (Invalidates old in-memory graph and replaces  |
|  it with fresh data from the updated DB)       |
+-----------------------------------------------+
```

---

## 20. Thread Safety & Concurrency Model

```
+-------------------------------------------------------------+
|                    Concurrency Architecture                   |
|                                                               |
|  Tokio Runtime (32 worker threads)                            |
|    |                                                          |
|    +-- gRPC request handlers (async)                          |
|    |                                                          |
|    +-- Dependency resolution tasks (JoinSet -- parallel)      |
|    |     Each dependency gets its own spawn() task            |
|    |                                                          |
|    +-- SDK loading task (spawned during init)                 |
|                                                               |
|  Shared State Protection:                                     |
|                                                               |
|    project.graph:                                             |
|      std::sync::Mutex<Option<StackGraph>>                     |
|      Synchronous lock (graph operations are CPU-bound)        |
|      Handles poisoned mutex (clears poison on recovery)       |
|                                                               |
|    project.dependencies:                                      |
|      tokio::sync::Mutex<Option<Vec<Dependencies>>>            |
|      Async lock (dependency resolution is I/O-bound)          |
|                                                               |
|    project.source_language_config:                            |
|      Arc<RwLock<Option<LanguageConfig>>>                      |
|      Read-heavy, write-once (many readers, few writers)       |
|                                                               |
|    provider.config, provider.project:                         |
|      Arc<Mutex<Option<...>>>                                  |
|      Set once during init(), read during evaluate()           |
|                                                               |
|    dependency.decompiled_location:                            |
|      Arc<Mutex<HashSet<PathBuf>>>                             |
|      Multiple decompile tasks write to same set               |
+---------------------------------------------------------------+
```

### Why Two Different Mutex Types?

- **`std::sync::Mutex`** — Used for CPU-bound operations (graph traversal, tree-sitter parsing). These operations don't yield to the async runtime, so an async mutex would be wasteful.
- **`tokio::sync::Mutex`** — Used for I/O-bound operations (dependency resolution, file reading). These operations `await` and need to yield the thread back to Tokio.

---

## 21. Transport Layer Options

```
+--------------------------------------------------+
|              Transport Configuration              |
|                                                   |
|  Option 1: TCP/HTTP2 (--port 9000)                |
|    Server::builder()                              |
|      .add_service(ProviderServiceServer)          |
|      .add_service(CodeLocationServiceServer)      |
|      .add_service(ReflectionServer)               |
|      .serve(addr)                                 |
|                                                   |
|  Option 2: Unix Domain Socket (--socket /tmp/x)   |
|    UnixListener::bind(socket_path)                |
|    Server::builder()                              |
|      .http2_keepalive_interval(7200s)             |
|      .http2_keepalive_timeout(20s)                |
|      .tcp_keepalive(7200s)                        |
|      .serve_with_incoming(uds_stream)             |
|                                                   |
|  Option 3: Windows Named Pipe (--socket \\.\pipe) |
|    get_named_pipe_connection_stream()             |
|    Server::builder()                              |
|      .serve_with_incoming(pipe_stream)            |
|                                                   |
|  Default port in Docker: 14651                    |
+--------------------------------------------------+
```

### CLI Arguments

| Argument | Default | Purpose |
|---|---|---|
| `--port <PORT>` | none | TCP port for HTTP/2 gRPC |
| `--socket <SOCKET>` | none | Unix socket or named pipe path |
| `--name <NAME>` | none | Service name |
| `--db-path <DB_PATH>` | `/tmp/c_sharp_provider.db` | SQLite database path |
| `--log-file <LOG_FILE>` | stdout | Log file path |
| `-v, --verbosity` | info | Log verbosity level |
| `--context-lines <N>` | 10 | Lines of context around code snippets |

---

## 22. Result Deduplication

Tree-sitter can create multiple syntax nodes for the same source location due to parsing ambiguities. The deduplication logic in `csharp.rs` handles this:

### Algorithm

1. Group results by `(file_uri, line_number)` using a `BTreeMap`
2. For each group, keep the result with the **smallest span** (`end_line - start_line`)
3. Tiebreaker 1: earliest `start_position.character`
4. Tiebreaker 2: earliest `end_position.character`

### Example

```
Input (3 results for AccountController.cs:240):
  - span: line 240-240, char 0-94    (single line)     <-- WINNER
  - span: line 240-241, char 0-20    (crosses to 241)
  - span: line 239-240, char 0-94    (starts earlier)

Output: keeps only the single-line span (240-240)
```

Results on **different** line numbers are never merged — they are separate matches.

### Determinism

- Results are sorted by `(file_uri, line_number)` after deduplication
- `BTreeMap` is used instead of `HashMap` for deterministic iteration order
- Edge sorting in graph traversal uses `sort_by` on sink handles for consistent ordering

---

## 23. End-to-End Example Trace

```
================================================================
SCENARIO: Find all uses of System.Web.Mvc.* in NerdDinner app
================================================================

1. INIT:
   grpcurl Init {
     location: "/testdata/nerd-dinner/mvc4",
     analysisMode: "source-only"
   }

   -> Finds .csproj -> detects net45 (old .NET Framework)
   -> Skips SDK installation (not modern .NET)
   -> tree-sitter parses 15 .cs files
   -> Builds StackGraph with ~500 nodes
   -> Persists to /tmp/c_sharp_provider.db
   -> Returns { success: true }

2. EVALUATE:
   grpcurl Evaluate {
     cap: "referenced",
     conditionInfo: '{"referenced":{"pattern":"System.Web.Mvc.*"}}'
   }

   -> Parses pattern -> Search with 4 parts
   -> location=All -> NamespaceSymbolsGetter
   -> Scans all graph nodes:

     Node "Controller" at AccountController.cs:12
       -> resolve FQDN -> System.Web.Mvc.Controller
       -> matches pattern!

     Node "ActionResult" at DinnersController.cs:25
       -> resolve FQDN -> System.Web.Mvc.ActionResult
       -> matches pattern!

     Node "HtmlHelper" at Views/Shared/_Layout.cshtml:3
       -> resolve FQDN -> System.Web.Mvc.HtmlHelper
       -> matches pattern!

     Node "String" at Models/Dinner.cs:8
       -> resolve FQDN -> System.String
       -> does NOT match pattern

   -> Deduplicate: remove duplicate nodes for same file+line
   -> Sort by file URI then line number

   -> Returns:
     {
       matched: true,
       incident_contexts: [
         { file: "AccountController.cs", line: 12, ... },
         { file: "DinnersController.cs", line: 25, ... },
         { file: "_Layout.cshtml", line: 3, ... },
         ...
       ]
     }
```

---

## 24. Build & Deployment

### Build System

The `build.rs` script handles protobuf compilation:

- **With `generate-proto` feature**: Downloads `protoc` if needed, compiles `provider.proto` to Rust
- **Without feature**: Verifies pre-generated `provider.rs` and descriptor binary exist; panics if missing

### Makefile Targets

| Target | Purpose |
|---|---|
| `make build` | Compile Rust binary |
| `make run-tests` | Build + run integration tests + cleanup |
| `make build-image` | Docker image creation |
| `make run` | Run server locally on port 9000 |
| `make download_proto` | Fetch latest proto from konveyor/analyzer-lsp |
| `make run-analyzer-integration-local` | Full e2e testing with analyzer-lsp |

### Docker Image (Multi-Stage)

```
+-----------------------------------------------------+
| Stage 1: Builder                                     |
|   Base: Red Hat UBI9                                 |
|   Installs: rust-toolset, unzip                      |
|   Builds: Release binary                             |
|   Output: c-sharp-analyzer-provider-cli               |
+-----------------------------------------------------+
                        |
                        v
+-----------------------------------------------------+
| Stage 2: Runtime                                     |
|   Base: Red Hat UBI9-minimal                         |
|   Installs:                                          |
|     - dotnet-sdk-9.0 + dotnet-runtime-9.0            |
|     - Paket (.NET tool for dependency resolution)    |
|     - ilspycmd (.NET tool for decompilation)          |
|   User: non-root (UID 1001)                          |
|   Env: RUST_LOG=info,c_sharp_analyzer_provider=debug |
|   Entrypoint: /usr/local/bin/c-sharp-provider         |
|   Default: --name c-sharp --port 14651                |
+-----------------------------------------------------+
```

---

## 25. Testing Infrastructure

### Integration Tests (`tests/integration_test.rs`)

The integration test framework:

1. **Starts the server** as a child process with `ServerGuard` (RAII pattern — auto-kills on drop)
2. **Waits for readiness** by polling gRPC connection (up to 60 attempts, 500ms apart)
3. **Sends Init** with test project path and source-only mode
4. **Walks demo directories** (`tests/demos/`) containing:
   - `request.yaml` — the query to send
   - `demo-output.yaml` — expected results
5. **Compares** actual vs expected results using symmetric difference
6. **Cleans up** automatically via `ServerGuard` drop

### Test Data

| Project | Path | Purpose |
|---|---|---|
| NerdDinner | `testdata/nerd-dinner/mvc4/` | Real-world ASP.NET MVC4 app |
| Net8 Sample | `testdata/net8-sample/` | Modern .NET 8 project |

### Demo Test Format

```yaml
# tests/demos/namespace_search/request.yaml
cap: "referenced"
id: 5
condition_info: |
  {"referenced": {"pattern": "System.Web.*"}}
```

Supports location filtering: `CLASS`, `METHOD`, `FIELD`, or omitted for all.

### CI/CD Pipelines

| Workflow | Purpose |
|---|---|
| `demo-testing.yml` | Integration test automation |
| `release-binaries.yml` | Binary artifact generation |
| `image-build.yaml` | Docker image building |
| `verifyPR.yml` | PR validation |
| `stale.yml` | Automated stale issue handling |

---

## 26. Contributor/Development Workflow

### 26.1 Prerequisites

| Tool | Required? | Installation |
|---|---|---|
| Rust 1.70+ | Yes | [rustup.rs](https://rustup.rs) |
| .NET SDK 9.x | Yes (for tests) | [dotnet.microsoft.com/download](https://dotnet.microsoft.com/download) |
| ilspycmd | Optional (full mode) | `dotnet tool install -g ilspycmd` |
| paket | Optional (dep resolution) | `dotnet tool install -g paket` |
| grpcurl | Optional (manual testing) | `brew install grpcurl` or from GitHub releases |
| protoc | Only if modifying .proto | Auto-downloaded by `build.rs` |

### 26.2 Initial Setup

```bash
git clone <repository-url>
cd c-sharp-analyzer-provider
cargo build                    # first build downloads all crates
make run-tests                 # verify everything works
```

### 26.3 Build Commands

| Command | What it does |
|---|---|
| `cargo build` | Debug build |
| `cargo build --release` | Optimized release build |
| `make build` | Same as `cargo build` |
| `make build-image` | Build Docker image via podman/docker |
| `cargo build --features generate-proto` | Rebuild Rust bindings if `provider.proto` changed |

### 26.4 Running the Server Locally

```bash
cargo run -- --port 9000 --name c-sharp --db-path testing.db
```

With debug logging:

```bash
RUST_LOG=debug cargo run -- --port 9000 --name c-sharp --db-path testing.db
```

### 26.5 Manual Testing with grpcurl

```bash
# Initialize with test project
grpcurl -plaintext -d '{
  "analysisMode": "source-only",
  "location": "'"$(pwd)"'/testdata/nerd-dinner"
}' localhost:9000 provider.ProviderService.Init

# Run a query
grpcurl -plaintext -d '{
  "cap": "referenced",
  "conditionInfo": "{\"referenced\": {\"pattern\": \"System.Web.Mvc.*\"}}"
}' localhost:9000 provider.ProviderService.Evaluate
```

### 26.6 Running Tests

```bash
make run-tests              # recommended: build + test + cleanup
cargo test -- --nocapture   # just the tests, no cleanup
```

The test framework automatically:
1. Builds the server binary
2. Starts a server instance on an available port
3. Sends Init + Evaluate requests from `tests/demos/*/request.yaml`
4. Compares results against `tests/demos/*/demo-output.yaml`
5. Kills the server and cleans up DB files

### 26.7 Adding a New Test Case

1. Choose the search type directory: `namespace_search/`, `class_search/`, `method_search/`, or `field_search/`
2. Create a new subdirectory: `tests/demos/<type>/<descriptive_name>/`
3. Create `request.yaml`:
   ```yaml
   cap: "referenced"
   id: 99
   condition_info: |
     {"referenced": {"pattern": "Your.Pattern.Here", "location": "METHOD"}}
   ```
4. Run the server and query manually (or run the test framework) to get actual output
5. Create `demo-output.yaml` with the expected results (use `<REPLACE_ME>` for the absolute path prefix — the test framework substitutes it)
6. Run `make run-tests` to verify

### 26.8 Modifying TSG Rules

File: `src/c_sharp_graph/stack-graphs.tsg`

When adding support for a new C# construct:

1. **Identify the CST node type** — use [tree-sitter playground](https://tree-sitter.github.io/tree-sitter/playground) or parse a sample file to see what node types tree-sitter produces
2. **Write a match rule** using the CST node type name as the pattern
3. **Create graph nodes** with appropriate `syntax_type` tags
4. **Create edges** following the dual-edge pattern: containment (prec 0) down + FQDN (prec 10) up
5. **Add a test case** that exercises the new rule
6. **Run `make run-tests`** and verify no regressions

Common pitfalls:
- Node declarations (`node @x.def`) must appear before any `attr` or `edge` referencing them
- The `@capture.def` property is shared across rules — if another rule already creates `@capture.def` for the same CST node type, your rule adds to it rather than replacing
- Rule order within the file does not matter; all matching rules fire for each CST node

### 26.9 Debugging

**Log levels:**

| Level | Command | Use for |
|---|---|---|
| `info` | `RUST_LOG=info cargo run -- ...` | Normal operation (default) |
| `debug` | `RUST_LOG=debug cargo run -- ...` | Graph construction and query execution details |
| `trace` | `RUST_LOG=trace cargo run -- ...` | Individual node/edge creation (very verbose) |

**Log to file:** `--log-file /tmp/provider.log`

**Test server logs:** Written to `test-server-<port>.log` during test runs.

---

## 27. Design Rationale

This section explains the "why" behind non-obvious technical decisions. Each subsection references the section where the decision is implemented.

### 27.1 Precedence 10 for FQDN Edges

FQDN back-reference edges use precedence 10 rather than 1. This leaves the range 0-9 available for other edge types. Precedence 0 is used for containment edges. The query engine's `get_fqdn()` function in `query.rs` filters specifically for edges with `precedence == 10` to walk upward through the hierarchy, so these values must not overlap.

*(See [Section 11.5](#115-fqdn-resolution-how-the-query-engine-walks-the-graph))*

### 27.2 BTreeMap over HashMap

All result collections use `BTreeMap` and `BTreeSet` rather than `HashMap`/`HashSet`. `BTreeMap` iterates in sorted key order, which makes:
- Test outputs reproducible across runs (no hash-dependent ordering)
- Debugging predictable (same input always produces same output order)
- Diff-based test verification reliable

*(See [Section 13](#13-query-type-system) and [Section 22](#22-result-deduplication))*

### 27.3 Smallest Span Deduplication

When tree-sitter produces multiple CST nodes for the same source location (due to how the grammar handles nested expressions), the deduplication logic keeps the node with the **smallest span** (fewest lines). A tighter span corresponds to the most precise, most specific token for that location. For example, a `member_access_expression` spanning one line is preferred over a parent `invocation_expression` spanning two lines.

*(See [Section 22](#22-result-deduplication))*

### 27.4 Source-Only vs Full Analysis Mode

- **Source-only mode** uses SDK XML documentation files. These are static metadata files that ship with the .NET SDK. They are fast to parse, safe (no code execution), and sufficient for detecting namespace and class references during migration analysis.
- **Full mode** uses ILSpy to decompile `.dll` files to `.cs` source, then parses the decompiled source with tree-sitter. This gives deeper analysis (method bodies, implementation details) but is slower and requires ILSpy.

The default in production is source-only. Full mode exists for cases where decompiled source analysis is needed.

*(See [Section 16](#16-dependency-resolution-pipeline))*

### 27.5 Two-Part Member Access

The TSG rule for `member_access_expression` creates a single symbol `"expression.name"` (e.g., `"filters.Add"`). Tree-sitter represents `a.b.c` as nested binary nodes: `member_access(member_access(a, b), c)`. Each level creates its own two-part symbol. The query engine splits on `.` to resolve the first two parts but does not recursively chain through multiple levels.

Chained access like `a.b.c.d` would require recursive type resolution (resolve `a`, find `b` on it, resolve the type of `a.b`, find `c` on that type, etc.), which is not implemented.

*(See [Section 11.6](#116-member-access-resolution) and [Section 28.1](#281-c-language-constructs-not-fully-handled))*

### 27.6 Constructor Handling as Method

Constructors are treated as methods (`syntax_type = "method_name"`) in the TSG rules. This simplification means they appear in method searches, which is useful for migration analysis ("find all places that construct type X"). The tradeoff is potential ambiguity: a class named `Foo` with a constructor will have both a `ClassDef` node and a `MethodName` node with symbol `"Foo"`.

### 27.7 std::sync::Mutex vs tokio::sync::Mutex

The project uses two different mutex types intentionally:
- **`std::sync::Mutex`** for CPU-bound work (graph traversal, tree-sitter parsing) — these operations do not `await` and would not benefit from yielding
- **`tokio::sync::Mutex`** for I/O-bound work (dependency resolution, file reading) — these operations `await` and must yield the thread back to the Tokio runtime

*(See [Section 20](#20-thread-safety--concurrency-model))*

### 27.8 SHA1 for File Change Detection

File changes are detected by comparing SHA1 hashes of file content. SHA1 is used because the stack-graphs library uses it internally for content addressing (not for cryptographic security). Using the same hash algorithm avoids mismatches between the project's change detection and the library's internal state.

*(See [Section 18](#18-sqlite-caching-layer))*

---

## 28. Known Limitations & Edge Cases

### 28.1 C# Language Constructs Not Fully Handled

| Construct | C# Version | Status | Impact |
|---|---|---|---|
| Chained member access (`a.b.c.d`) | All | Only 2-part access resolved per node | Misses deeply chained references |
| Generic type parameters (`List<T>`) | All | Stored as raw string, not resolved | No type parameter substitution |
| `async`/`await` semantics | C# 5+ | Parsed but not semantically handled | Treated as regular method calls |
| Pattern matching (`is`, switch expressions) | C# 7+ | Minimal node creation | Patterns not fully traversed |
| Record types | C# 9+ | No differentiation from classes | Records treated as classes |
| Extension methods | C# 3+ | Not resolved to target type | Extension calls not linked to the extended type |
| Method overloads | All | Not resolved | All overloads treated as same method |
| Nullable reference types (`string?`) | C# 8+ | Syntax handled | No flow analysis |
| Global using directives | C# 10+ | Not handled | Won't be discovered as imports |
| File-scoped namespaces (`namespace X;`) | C# 10+ | Not handled | Namespace won't be detected |
| Primary constructors | C# 12+ | Not handled | -- |

### 28.2 Graph Construction Limitations

| Limitation | Description | Consequence |
|---|---|---|
| No incremental graph update | Entire graph reloaded from SQLite on any file change | Potentially slow for large projects with frequent changes |
| No cycle detection in FQDN resolution | `get_fqdn()` follows prec-10 edges recursively with no visited-set | Could loop infinitely on malformed graphs (unlikely in practice) |
| Constructor FQDN ambiguity | Constructor and class share the same symbol name | May produce duplicate FQDN entries |
| Single graph per project | All files share one StackGraph instance | No isolation between compilation units |
| `builtins.cs` is empty | No built-in type definitions pre-loaded | Built-in types (`int`, `string`, etc.) not in graph unless found in SDK XML |

### 28.3 Query Engine Limitations

| Limitation | Description | Workaround |
|---|---|---|
| Import disambiguation false negatives | If no `using` statement matches a candidate, returns None even for single candidates | Unimported types are missed |
| No transitive reference resolution | Only direct references found | Indirect references through type aliases or inheritance not followed |
| Case-sensitive matching | All symbol matching is case-sensitive | Must use exact casing in query patterns |
| Wildcard performance | `System.*.Mvc` works but may scan many nodes | Use specific namespace parts when possible |

### 28.4 Dependency Resolution Limitations

| Limitation | Description |
|---|---|
| Paket required | Cannot use NuGet CLI directly for dependency resolution |
| ILSpy required for full mode | No support for alternative decompilers |
| One target framework | If multiple TFMs exist, uses earliest/lowest only |
| No transitive dependency analysis | Only direct dependencies resolved by default |

### 28.5 TSG Rule Gaps

The following C# expression types have TODO comments in `stack-graphs.tsg` (lines 1047-1059) and are not yet handled:

- `implicit_array_creation_expression`
- `implicit_object_creation_expression` (target-typed `new()`)
- `implicit_stackalloc_expression`
- `makeref_expression`
- `range_expression` (`x..y`)
- `reftype_expression` / `refvalue_expression`
- `sizeof_expression`
- `stackalloc_expression`
- `throw_expression` (expression form, not statement)

Symbol references inside these unhandled constructs will not appear in the stack graph and will not be found by queries.

### 28.6 Known Edge Cases

1. **Commented-out code** — tree-sitter correctly ignores comments, but files entirely wrapped in `/* ... */` produce an empty `compilation_unit` with no graph nodes.

2. **Partial/invalid source** — tree-sitter is error-tolerant and produces a partial CST. TSG rules fire on whatever nodes exist, so partial results are possible. This is generally desirable for migration analysis.

3. **Very large files** — No explicit size limit, but graph size grows linearly with file size. Files with thousands of expressions may slow queries.

4. **Duplicate file paths** — If the same relative path is added twice (e.g., from both source and decompiled output), the second addition fails with a "file already exists" error.

5. **Unicode identifiers** — C# supports Unicode identifiers. Tree-sitter handles them correctly, but `source-text` in TSG returns raw Unicode characters, which may cause matching issues if query patterns use ASCII-only.

---

## 29. File-by-File Reference

### Entry Points

| File | Lines | Purpose |
|---|---|---|
| `src/main.rs` | 187 | Server bootstrap, CLI parsing, transport setup |
| `src/lib.rs` | 6 | Re-exports `analyzer_service`, `c_sharp_graph`, `provider` modules |

### Service Layer

| File | Lines | Purpose |
|---|---|---|
| `src/analyzer_service/mod.rs` | 6 | Includes generated proto code, exposes file descriptor set |
| `src/analyzer_service/provider.rs` | ~77KB | Auto-generated from `provider.proto` (gRPC client/server stubs, message types) |

### Provider Implementation

| File | Lines | Purpose |
|---|---|---|
| `src/provider/mod.rs` | 12 | Module declarations, re-exports `CSharpProvider`, `Project`, `AnalysisMode` |
| `src/provider/csharp.rs` | 974 | Main gRPC trait implementation: `init()`, `evaluate()`, `notify_file_changes()`, deduplication logic |
| `src/provider/project.rs` | 348 | Project state management: tool discovery, graph loading, language config |
| `src/provider/dependency_resolution.rs` | 895 | Paket + ILSpy pipeline, database loading for dependencies |
| `src/provider/sdk_detection.rs` | 318 | .NET SDK discovery across configured, env, and platform paths |
| `src/provider/target_framework.rs` | 991 | .csproj XML parsing, TFM normalization, SDK installation, XML file discovery |
| `src/provider/code_snip.rs` | 102 | Code snippet extraction with configurable context lines |

### Analysis Engine

| File | Lines | Purpose |
|---|---|---|
| `src/c_sharp_graph/mod.rs` | 11 | Module exports, re-exports `NotFoundError` |
| `src/c_sharp_graph/query.rs` | 1667 | Core query engine: `Querier`, `Search`, FQDN resolution, import disambiguation, `SymbolMatcher` trait |
| `src/c_sharp_graph/loader.rs` | 370 | Graph construction: `init_stack_graph()`, `add_dir_to_graph()`, `load_and_store_file()`, `SourceType` enum |
| `src/c_sharp_graph/language_config.rs` | 138 | Tree-sitter + stack-graphs configuration, builtins loading |
| `src/c_sharp_graph/namespace_query.rs` | 458 | Namespace search — traverses graph, delegates to class/method/field matchers |
| `src/c_sharp_graph/class_query.rs` | 307 | Class search — finds `ClassDef` nodes, matches by FQDN |
| `src/c_sharp_graph/method_query.rs` | 292 | Method search — finds `MethodName` nodes, expects `Class.Method` format |
| `src/c_sharp_graph/field_query.rs` | 296 | Field search — finds `FieldName` nodes, expects `Class.Field` format |
| `src/c_sharp_graph/results.rs` | 158 | `ResultNode`, `Location`, `Position` structs, protobuf conversion |
| `src/c_sharp_graph/dependency_xml_analyzer.rs` | 792 | SDK XML documentation parser (`DepXMLFileAnalyzer`), handles N/T/M/F/P member types |
| `src/c_sharp_graph/stack-graphs.tsg` | 600+ | Declarative rules mapping C# CST nodes to stack graph nodes/edges |
| `src/c_sharp_graph/builtins.cs` | - | Built-in C# type definitions |
| `src/c_sharp_graph/builtins.cfg` | - | Builtins configuration |

### Platform Support

| File | Lines | Purpose |
|---|---|---|
| `src/pipe_stream/mod.rs` | - | Platform-conditional module exports |
| `src/pipe_stream/server.rs` | - | Windows named pipe implementation |

### Build & Config

| File | Purpose |
|---|---|
| `build.rs` | Proto compilation script (conditional on `generate-proto` feature) |
| `Cargo.toml` | Rust package manifest and dependencies |
| `Cargo.lock` | Locked dependency versions |
| `Dockerfile` | Multi-stage container build (UBI9) |
| `Makefile` | Build, test, and deployment automation (249 lines) |

### Tests

| File | Purpose |
|---|---|
| `tests/integration_test.rs` | gRPC integration tests with server lifecycle management |
| `tests/demos/namespace_search/` | Namespace query test cases |
| `tests/demos/class_search/` | Class query test cases |
| `tests/demos/method_search/` | Method query test cases |
| `tests/demos/field_search/` | Field query test cases |
| `testdata/nerd-dinner/` | ASP.NET MVC4 test project |
| `testdata/net8-sample/` | .NET 8 test project |
