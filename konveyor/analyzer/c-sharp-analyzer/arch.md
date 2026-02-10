# C# Analyzer Provider — Technical Architecture

## Table of Contents

1. [Overview](#1-overview)
2. [Konveyor Ecosystem Context](#2-konveyor-ecosystem-context)
3. [Technology Stack](#3-technology-stack)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Project Structure](#5-project-structure)
6. [gRPC Service Interface](#6-grpc-service-interface)
7. [Initialization Flow](#7-initialization-flow)
8. [Evaluate (Query) Flow](#8-evaluate-query-flow)
9. [The Stack Graph — Core Data Structure](#9-the-stack-graph--core-data-structure)
10. [Tree-Sitter + Stack Graphs Integration](#10-tree-sitter--stack-graphs-integration)
11. [Query Type System](#11-query-type-system)
12. [Search Pattern Matching Engine](#12-search-pattern-matching-engine)
13. [Dependency Resolution Pipeline](#13-dependency-resolution-pipeline)
14. [SDK Detection & Target Framework](#14-sdk-detection--target-framework)
15. [SQLite Caching Layer](#15-sqlite-caching-layer)
16. [File Change Notification Flow](#16-file-change-notification-flow)
17. [Thread Safety & Concurrency Model](#17-thread-safety--concurrency-model)
18. [Transport Layer Options](#18-transport-layer-options)
19. [Result Deduplication](#19-result-deduplication)
20. [End-to-End Example Trace](#20-end-to-end-example-trace)
21. [Build & Deployment](#21-build--deployment)
22. [Testing Infrastructure](#22-testing-infrastructure)
23. [File-by-File Reference](#23-file-by-file-reference)

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

## 4. High-Level Architecture

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

## 5. Project Structure

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

## 6. gRPC Service Interface

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

## 7. Initialization Flow

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

## 8. Evaluate (Query) Flow

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

## 9. The Stack Graph -- Core Data Structure

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

## 10. Tree-Sitter + Stack Graphs Integration

The system uses a two-phase pipeline to build the stack graph from C# source:

```
+------------------------------------------------------+
|                    C# Source File                      |
|                                                       |
|  using System.Web.Mvc;                                |
|                                                       |
|  namespace NerdDinner.Controllers {                    |
|      public class DinnersController : Controller {    |
|          public ActionResult Index() {                 |
|              return View();                            |
|          }                                             |
|      }                                                 |
|  }                                                     |
+--------------------------+----------------------------+
                           |
                tree-sitter parses
                           |
                           v
+------------------------------------------------------+
|              Concrete Syntax Tree (CST)               |
|                                                       |
|  compilation_unit                                      |
|    +-- using_directive                                 |
|    |     +-- qualified_name: "System.Web.Mvc"          |
|    +-- namespace_declaration                           |
|          +-- name: "NerdDinner.Controllers"            |
|          +-- class_declaration                         |
|                +-- name: "DinnersController"           |
|                +-- base_list: "Controller"             |
|                +-- method_declaration                  |
|                      +-- name: "Index"                 |
|                      +-- body: ...                     |
+--------------------------+----------------------------+
                           |
     stack-graphs.tsg rules transform
     (600+ lines of pattern -> graph rules)
                           |
                           v
+------------------------------------------------------+
|                   Stack Graph Nodes                   |
|                                                       |
|  Node(symbol="System.Web.Mvc", type=Import)           |
|  Node(symbol="NerdDinner.Controllers", type=NS)       |
|  Node(symbol="DinnersController", type=ClassDef)      |
|  Node(symbol="Controller", type=ClassDef) -------+    |
|  Node(symbol="Index", type=MethodName)            |   |
|  Node(symbol="View", type=MethodName)             |   |
|                                                   |   |
|  Edges: NS->Class->Method (containment)          |   |
|  Edges: ClassDef->Import (reference/using)  <-----+   |
+------------------------------------------------------+
```

### The .tsg File

The `stack-graphs.tsg` file (Tree-Sitter Graph) contains declarative rules that map tree-sitter CST nodes to stack graph nodes and edges. Each rule says:

> "When you see a `namespace_declaration` node in the CST, create a stack graph push-node with a `NamespaceDeclaration` syntax type, and connect it to the parent scope."

### Builtins

The `builtins.cs` and `builtins.cfg` files define built-in C# types that are always available (like `object`, `string`, `int`). These are loaded into the graph during language configuration initialization so queries can resolve references to built-in types.

---

## 11. Query Type System

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

## 12. Search Pattern Matching Engine

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

## 13. Dependency Resolution Pipeline

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

## 14. SDK Detection & Target Framework

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

## 15. SQLite Caching Layer

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

## 16. File Change Notification Flow

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

## 17. Thread Safety & Concurrency Model

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

## 18. Transport Layer Options

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

## 19. Result Deduplication

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

## 20. End-to-End Example Trace

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

## 21. Build & Deployment

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

## 22. Testing Infrastructure

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

## 23. File-by-File Reference

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
