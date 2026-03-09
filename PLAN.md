# PLAN: smart-grep
### Semantic Code Search Engine — Comprehensive Implementation Plan
**Version 2.0 · March 2026 · Status: Pre-development · Target: ~10 days**

---

## Table of Contents

1. [Vision](#1-vision)
2. [Problem Statement](#2-problem-statement)
3. [Product Goals](#3-product-goals)
4. [Why Rust](#4-why-rust)
5. [Why Workspace + crates/](#5-why-workspace--crates)
6. [High-Level Architecture](#6-high-level-architecture)
7. [Complete File Structure](#7-complete-file-structure)
8. [Workspace Configuration](#8-workspace-configuration)
9. [Core Data Structures](#9-core-data-structures)
10. [Core Traits](#10-core-traits)
11. [Storage Schema](#11-storage-schema)
12. [Embedding Text Normalization](#12-embedding-text-normalization)
13. [Chunking Strategy](#13-chunking-strategy)
14. [Parsing Layer](#14-parsing-layer)
15. [Embedding Layer](#15-embedding-layer)
16. [Hybrid Search: BM25 + HNSW](#16-hybrid-search-bm25--hnsw)
17. [Query Expansion via Synonym Map](#17-query-expansion-via-synonym-map)
18. [Query-by-Example (--like)](#18-query-by-example---like)
19. [Structural Pattern Search (--pattern)](#19-structural-pattern-search---pattern)
20. [Relationship Graph + Contextual Expansion](#20-relationship-graph--contextual-expansion)
21. [Reachability-Scoped Search (--from)](#21-reachability-scoped-search---from)
22. [Confidence-Aware Results](#22-confidence-aware-results)
23. [Incremental Indexing](#23-incremental-indexing)
24. [Context Budget Mode (--budget)](#24-context-budget-mode---budget)
25. [Streaming Results (--stream)](#25-streaming-results---stream)
26. [Codebase Map (smart-grep map)](#26-codebase-map-smart-grep-map)
27. [Semantic Diff (smart-grep diff)](#27-semantic-diff-smart-grep-diff)
28. [Index Snapshots + Time-Travel Search](#28-index-snapshots--time-travel-search)
29. [Stale Logic Detection](#29-stale-logic-detection)
30. [Semantic Test Coverage Mapping](#30-semantic-test-coverage-mapping)
31. [Semantic Naming Linter (smart-grep lint)](#31-semantic-naming-linter-smart-grep-lint)
32. [Multi-Query Fan-Out (smart-grep multi)](#32-multi-query-fan-out-smart-grep-multi)
33. [Agent Session Memory](#33-agent-session-memory)
34. [Project Auto-Detection](#34-project-auto-detection)
35. [MCP Server Mode](#35-mcp-server-mode)
36. [Self-Index Mode (--self)](#36-self-index-mode---self)
37. [CLI Design](#37-cli-design)
38. [Configuration](#38-configuration)
39. [Observability](#39-observability)
40. [Error Handling](#40-error-handling)
41. [Per-Language Implementation Details](#41-per-language-implementation-details)
42. [Testing Strategy](#42-testing-strategy)
43. [Benchmarking Strategy](#43-benchmarking-strategy)
44. [Evaluation Plan](#44-evaluation-plan)
45. [Risks and Mitigations](#45-risks-and-mitigations)
46. [Implementation Phases](#46-implementation-phases)
47. [Integration with Claude Code](#47-integration-with-claude-code)
48. [Timeline](#48-timeline)
49. [Feature Map](#49-feature-map)
50. [Future Extensions](#50-future-extensions)

---

## 1. Vision

Build **smart-grep**, a semantic code search tool for local codebases that finds code by **meaning**, not exact keywords.

Unlike `grep` or `ripgrep`, which only match literal text or regex patterns, smart-grep helps developers answer questions like:

- "Where is payment error handled?"
- "Where do we retry failed billing operations?"
- "What code is responsible for access control checks?"
- "Show me the middleware that verifies permissions before mutating state."
- "Find everything related to rate limiting."

Even when the actual code uses completely different names such as:

| What you search for | What the code actually uses |
|---|---|
| `payment error` | `Stripe::CardError`, `BillingException`, `PaymentDeclinedError` |
| `retry failed billing` | `handleChargeFailure()`, `retry_with_backoff()`, `attempt_charge()` |
| `access control` | `authorize!()`, `guard_clause()`, `require_admin()`, `check_scope()` |
| `rate limiting` | `throttle()`, `RateLimiter`, `quota_exceeded`, `429_handler` |

The core value proposition: return the **most semantically relevant code chunks** with minimal noise and minimal token usage — for both human developers and coding agents like Claude Code.

### 1.1 Beyond Search — A Semantic Understanding Layer

Beyond search, smart-grep is a **semantic understanding layer** for codebases. The embedding index it builds unlocks a class of analysis that no other tool can provide:

- **Codebase orientation**: Explain what a codebase does to a new developer or agent (`smart-grep map`)
- **Conceptual staleness detection**: Find code that should have changed after a refactor but did not
- **Meaning-level history**: Find semantic changes across git history, not just byte diffs (`smart-grep diff`)
- **Naming quality**: Surface naming inconsistencies and duplicate logic automatically (`smart-grep lint`)
- **Semantic test coverage**: Map which tests cover which functions, without running any tests (`smart-grep coverage`)
- **Context assembly**: Build token-budget-aware context blocks for LLM agents (`--budget`)
- **Time-travel**: Query what code meant before a refactor (`--at <snapshot>`)
- **Agent session memory**: Accumulate and deduplicate context across a multi-query investigation (`session`)

---

## 2. Problem Statement

Traditional search tools such as `grep` and `ripgrep` are optimized for **exact string** and **regex** matching. That breaks down in four concrete scenarios every developer encounters:

### 2.1 Failure Scenarios

| Scenario | grep/ripgrep result | smart-grep result |
|---|---|---|
| Symbols renamed or abbreviated | Zero results — the string doesn't exist | Semantic match via vector similarity |
| Different languages express same concept | Language-specific regex required per language | Cross-language semantic match via shared embedding space |
| Intent described in natural language | Must know the exact symbol name to search | Natural language query works directly |
| Coding agents scanning large repos | Must read every potentially relevant file | Top-K ranked results in a single call |
| Concept evolved through refactors | Old names no longer present | Embedding similarity detects conceptual continuity |
| Finding design patterns across a repo | Regex for every possible implementation | Structural fingerprint search matches all instances |
| Monorepo with multiple services | Results from unrelated services contaminate results | `--from` scoping restricts to relevant call graph |

### 2.2 Cost of These Failures

These failures lead to concrete, measurable costs in every engineering team:

- **Missed results**: Critical code is not found because the developer does not know the exact symbol name. Bugs are introduced because the developer is unaware of existing implementations.
- **High token consumption**: Coding agents must scan entire directories rather than issuing a single precise query. At scale this is extremely expensive and slow.
- **Repeated full-repo scans**: Without a semantic index, agents fall back to repeated broad ripgrep calls that return too much noise.
- **Weak cross-language discovery**: In polyglot repos (e.g. Ruby + Rust + TypeScript), the same concept may be implemented in each language with completely different naming conventions. Traditional search cannot surface all of them with a single query.
- **Silent false positives**: A search that returns near-relevant results but misses the true answer leads to agents and developers acting on wrong information.

### 2.3 The Solution

smart-grep solves all of this by building a **local semantic index** over code structure and using **vector similarity search** to retrieve ranked matches instantly — with zero API calls at search time. All inference runs locally on the developer machine via a bundled ONNX model (`fastembed-rs`). The index is stored in `.smart-grep/index.db` alongside the repo. First index takes ~30–90s for a 50k-line codebase; subsequent queries run in under 100ms.

---

## 3. Product Goals

### 3.1 Primary Goals

1. Semantic code search over a local repository — **no internet required** at index or search time.
2. Support multiple programming languages via `tree-sitter` (Rust, TypeScript, JavaScript, Python, Ruby for v1).
3. Return **precise ranked results** with file path, line range, score, and code preview.
4. Fast enough for **interactive terminal use** (< 100ms query latency) and coding agent use (< 200ms including transport).

### 3.2 Secondary Goals

1. **Minimize token cost** when used with LLM tools — `--budget` assembles a ready-to-use context block within a specified token limit.
2. **Incremental indexing** — re-index only changed files after mutation, < 5s for a single-file change in a 10k-file repo.
3. **Single Rust binary**, `cargo install`-able with zero system dependencies (no Python, no Node, no JVM).
4. **Transparent and debuggable index** — `--why` explains exactly what drove each match.
5. **Native MCP server mode** for direct agent integration via stdio or HTTP/SSE.
6. **Context assembly** for LLM prompt budgets via `--budget N`.
7. **Passive codebase quality intelligence** — naming consistency, duplicate logic, test coverage gaps require zero extra work from the developer.
8. **Confidence-aware results** — knows when to say "I don't know" rather than returning low-relevance noise.

### 3.3 Non-Goals for v1

- Full IDE replacement or LSP server (no hover, definition, completion, diagnostics).
- Full type resolution across languages (no inter-file type inference).
- Remote hosted search service (local-only by design).
- Perfect understanding of every framework-specific pattern.
- Online HNSW updates — v1 rebuilds the vector index after mutation (v1.1 target for online updates).
- Support for every programming language — v1 focuses on the five most common.

---

## 4. Why Rust

The choice of Rust is deliberate and load-bearing, not aesthetic. Each property maps directly to a specific requirement:

| Property | Why it matters for smart-grep |
|---|---|
| **Fast parallel indexing (`rayon`)** | Indexing a 50k-line repo must be fast enough that developers re-index on every file save. `rayon` gives true multi-core parallelism with zero GIL overhead. |
| **Memory-efficient** | Storing 384-dimensional float32 vectors for 10,000 chunks is ~15MB. On a developer laptop with 16GB shared with IDE and browser, this must stay minimal. |
| **Safe concurrent AST parsing** | `tree-sitter` parsers are not thread-safe. Rust's ownership system prevents data races when parsing multiple files across threads without any runtime overhead. |
| **Single statically-linked binary** | `cargo install smart-grep` works on any Linux/macOS/Windows machine with zero runtime dependencies. No Python virtualenv, no Node.js, no JVM. |
| **Ecosystem fit** | `tree-sitter`, `rusqlite`, and `usearch` (HNSW) all have first-class Rust bindings maintained by their original authors. `fastembed-rs` provides a battle-tested ONNX runtime. |
| **Zero-cost trait abstractions** | The `ChunkExtractor` and `Embedder` trait system allows clean per-language and per-model polymorphism with zero runtime overhead at dispatch sites. |
| **`thiserror` + `anyhow` error handling** | Typed errors per crate (`thiserror`) with ergonomic propagation in the binary (`anyhow`) gives production-grade error messages without boilerplate. |

---

## 5. Why Workspace + `crates/`

Popular Rust CLIs like `ripgrep`, `fd`, and `bat` all use this pattern. Ripgrep specifically uses `crates/core/main.rs` as its entry point.

- **Workspace root holds config, docs, poc, and tests** — no source code at root level
- **Each concern is an independently testable crate**: `indexer`, `store`, and `search` can each be unit tested, benchmarked, and published independently
- `src/` exists only **inside each sub-crate**, never at the workspace root
- Makes it easy to later publish `crates/indexer` as a standalone library for third-party tools that want AST-based code chunking
- Separation of concerns enforced by the compiler: `search` cannot accidentally call `indexer` internals

---

## 6. High-Level Architecture

```
                 +----------------------+
                 |   smart-grep CLI     |  <- also: MCP server mode (stdio/SSE)
                 +----------+-----------+
                            |
      +----------+----------+----------+----------+----------+
      |          |          |          |          |          |
      v          v          v          v          v          v
   index      search      diff       map        lint      session
      |          |          |          |          |          |
      v          v          v          v          v          v
 +----------+ +----------+ +--------+ +--------+ +--------+ +--------+
 | File     | | Query    | | Snap-  | | K-Means| | Pair-  | | Query  |
 | Discovery| | Expand + | | shot   | | Cluster| | wise   | | History|
 | (ignore) | | --like   | | Diff   | | + Bias | | Sim    | | + Dedup|
 +----+-----+ +----+-----+ +--------+ +--------+ +--------+ +--------+
      |             |
      v             v
 +----------+ +----------+
 | tree-    | | Hybrid   |
 | sitter   | | HNSW +   |
 | AST +    | | BM25 +   |
 | graph +  | | Reach-   |
 | struct   | | ability  |
 | fingerpt | | Filter   |
 +----+-----+ +----+-----+
      |             |
      v             v
 +----------+ +----+-----+
 | Embed    | | Re-rank  |
 | text     | | + Budget |
 | normalize| | + Stream |
 +----+-----+ +----+-----+
      |             |
      v             v
 +---------------------------------------------+
 |  SQLite + HNSW  (.smart-grep/index.db)       |
 |  files | chunks | embeddings | call_edges    |
 |  synonyms | snapshots | sessions | meta      |
 +---------------------------------------------+
```

### 6.1 Crate Responsibilities

| Crate | Responsibility | Key modules |
|---|---|---|
| `crates/core` | Binary entry point. CLI parsing (`clap` derive), command dispatch, MCP server loop. **No business logic — pure orchestration.** | `main.rs` |
| `crates/indexer` | Code parsing, chunking, embedding, call graph extraction, structural fingerprinting, synonym map building, project detection. Produces `CodeChunk` and `CallEdge`. | `chunker`, `embedder`, `embedding_text`, `graph`, `structural`, `synonym_map`, `project_detect`, `languages/*` |
| `crates/store` | Persistence layer. SQLite schema (`rusqlite`) for all metadata, graph edges, sessions, snapshots, confidence history. `usearch` HNSW for vector storage and ANN search. | `db`, `index`, `snapshot` |
| `crates/search` | All ranking, re-ranking, and result formatting. Hybrid BM25+HNSW scoring, query expansion, reachability filtering, budget packing, cluster analysis, session management, lint, coverage, confidence calibration. | `rank`, `bm25`, `expand`, `reachability`, `budget`, `cluster`, `session`, `lint`, `coverage`, `confidence` |

### 6.2 Data Flow: Search

```
user query string
       │
       ▼
 [crates/core] dispatch to search
       │
       ▼
 [crates/search] synonym_map expand → embed query (Embedder trait)
       │
       ▼
 [crates/store] HNSW ANN → top-50 candidates by cosine similarity
       │
       ▼
 [crates/search] BM25 score all 50 from SQLite content column
       │
       ▼
 hybrid_score = 0.7 * semantic + 0.3 * bm25_normalized
       │
       ├── optional: reachability filter (--from) — BFS over call_edges
       ├── optional: call graph expand (--expand) — add callers + callees
       ├── optional: cluster bias — boost chunks in nearest cluster
       │
       ▼
 confidence calibration check
       │
       ▼
 budget packing (--budget) → streaming output (--stream)
       │
       ▼
 formatted results → terminal or MCP tool response
```

### 6.3 Data Flow: Indexing

```
[ignore crate] walk files, respect .gitignore and config excludes
       │
       ▼
 blake3 hash each file → skip unchanged files
       │
       ▼
 [crates/indexer] dispatch to ChunkExtractor by file extension
       │
       ▼
 tree-sitter parse → extract CodeChunk[] + CallEdge[]
       │
       ▼
 embedding_text.rs builds normalized header for each chunk
       │
       ▼
 [Embedder trait] batch embed all chunk texts → Vec<Vec<f32>> (rayon parallel)
       │
       ▼
 [crates/store] SQLite: write files, chunks, embeddings, call_edges
       │
       ▼
 [crates/store] HNSW: rebuild index from all current embeddings
       │
       ▼
 [crates/indexer] synonym_map: co-occurrence analysis → write synonyms table
       │
       ▼
 [crates/indexer] project_detect: write/update .smart-grep/config.toml
```

---

## 7. Complete File Structure

```
smart-grep/
├── Cargo.toml                     # workspace root — no source code here
├── Cargo.lock
├── README.md                      # user-facing docs, install, CLAUDE.md snippet
├── PLAN.md                        # this document
├── POC.md                         # Node.js POC notes and results
├── CLAUDE.md                      # smart-grep specific instructions for Claude Code
├── AGENTS.md                      # same for other agents (Cursor, etc.)
├── .gitignore                     # includes .smart-grep/
│
├── crates/
│   ├── core/                      # binary entry point
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── main.rs            # CLI: clap derive macros, command dispatch, MCP loop
│   │
│   ├── indexer/                   # tree-sitter chunking + embedding
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs             # IndexerConfig, public API: index_path()
│   │       ├── chunker.rs         # AST-based chunking orchestration + size bounds
│   │       ├── embedder.rs        # Embedder trait + fastembed-rs impl
│   │       ├── embedding_text.rs  # normalized text formatter (Language/Symbol/Kind/Path/Code)
│   │       ├── graph.rs           # call/reference graph extraction from AST
│   │       ├── structural.rs      # AST shape fingerprinting for --pattern search
│   │       ├── synonym_map.rs     # codebase-derived query expansion map
│   │       ├── project_detect.rs  # framework detection for auto-config
│   │       ├── languages.rs       # language dispatch (file extension → extractor)
│   │       └── languages/
│   │           ├── javascript.rs  # JS ChunkExtractor + CallEdge extractor
│   │           ├── typescript.rs  # TS ChunkExtractor (extends JS patterns)
│   │           ├── rust.rs        # Rust ChunkExtractor (fns, impls, traits, structs)
│   │           ├── python.rs      # Python ChunkExtractor (defs, classes, decorators)
│   │           └── ruby.rs        # Ruby ChunkExtractor (defs, modules, classes, rescue)
│   │
│   ├── store/                     # sqlite + usearch HNSW
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs             # StoreConfig, public API: open(), migrate()
│   │       ├── schema.sql         # canonical DDL — single source of truth for migrations
│   │       ├── db.rs              # rusqlite: CRUD for all tables, incremental transitions
│   │       ├── snapshot.rs        # snapshot serialization/restore for time-travel
│   │       └── index.rs           # usearch HNSW: vector storage + ANN search
│   │
│   └── search/                    # ranking and result formatting
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs             # SearchConfig, SearchResult, public API: search()
│           ├── rank.rs            # hybrid scoring: HNSW cosine + BM25
│           ├── bm25.rs            # pure-Rust BM25 over SQLite chunk content (~150 lines)
│           ├── expand.rs          # contextual expansion via call graph (--expand)
│           ├── reachability.rs    # call-graph BFS for --from scoping
│           ├── budget.rs          # token-budget-aware result selection + truncation
│           ├── cluster.rs         # k-means clustering for codebase map
│           ├── session.rs         # session tracking, dedup, cumulative context
│           ├── lint.rs            # naming inconsistency, duplicate logic, orphan detect
│           ├── coverage.rs        # semantic test coverage mapping (~30 lines)
│           └── confidence.rs      # score calibration, low-confidence detection (~80 lines)
│
├── docs/
│   ├── architecture.md            # detailed system design decisions
│   └── benchmark.md               # query benchmark results vs ripgrep
│
├── poc/
│   ├── package.json
│   └── index.js                   # Node.js POC — validate concept before Rust
│
├── test-fixtures/
│   ├── billing.js                 # payment handling code WITHOUT the word "payment"
│   ├── auth.js                    # authentication code WITHOUT "authentication"
│   └── retry.js                   # retry/backoff logic WITHOUT the word "retry"
│
├── tests/
│   ├── search_test.rs             # integration: index → search round-trip
│   ├── incremental_test.rs        # index → mutate file → re-index → verify correctness
│   ├── hybrid_rank_test.rs        # BM25 + semantic scoring correctness
│   ├── expansion_test.rs          # call graph contextual expansion
│   ├── reachability_test.rs       # --from scoping correctness
│   ├── session_test.rs            # session accumulation and dedup
│   └── confidence_test.rs         # low-confidence detection and calibration
│
└── benches/
    └── index_bench.rs             # criterion benchmarks: indexing throughput, query latency
```

> **Key rule**: No source code at workspace root. All `.rs` files live inside `crates/*/src/`. The workspace root exists only for configuration, documentation, and tooling.

---

## 8. Workspace Configuration

### 8.1 Root `Cargo.toml`

```toml
[workspace]
members = [
    "crates/core",
    "crates/indexer",
    "crates/store",
    "crates/search",
]
resolver = "2"

[workspace.package]
edition = "2024"
rust-version = "1.85"
license = "MIT OR Apache-2.0"
authors = ["smart-grep contributors"]
repository = "https://github.com/your-org/smart-grep"

[workspace.dependencies]
# Parsing
tree-sitter            = "0.22"
tree-sitter-javascript = "0.21"
tree-sitter-typescript = "0.21"
tree-sitter-rust       = "0.21"
tree-sitter-python     = "0.21"
tree-sitter-ruby       = "0.21"

# Embedding
fastembed = "3"

# Vector index
usearch = "2"

# Storage
rusqlite = { version = "0.31", features = ["bundled"] }

# File walking
ignore = "0.4"

# CLI
clap = { version = "4", features = ["derive"] }

# Serialization
serde_json = "1"
serde      = { version = "1", features = ["derive"] }

# Error handling
anyhow    = "1"
thiserror = "1"

# Parallelism
rayon = "1"

# Hashing
blake3 = "1"

# File watching (--watch mode)
notify = "6"

# Logging
tracing            = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

### 8.2 `crates/core/Cargo.toml`

```toml
[package]
name    = "smart-grep"
version = "0.1.0"
edition.workspace    = true
rust-version.workspace = true
license.workspace    = true

[[bin]]
name = "smart-grep"
path = "src/main.rs"

[dependencies]
indexer    = { path = "../indexer" }
store      = { path = "../store" }
search     = { path = "../search" }
clap       = { workspace = true }
serde_json = { workspace = true }
anyhow     = { workspace = true }
tracing    = { workspace = true }
tracing-subscriber = { workspace = true }
```

### 8.3 `crates/indexer/Cargo.toml`

```toml
[package]
name    = "indexer"
version = "0.1.0"
edition.workspace = true

[dependencies]
tree-sitter            = { workspace = true }
tree-sitter-javascript = { workspace = true }
tree-sitter-typescript = { workspace = true }
tree-sitter-rust       = { workspace = true }
tree-sitter-python     = { workspace = true }
tree-sitter-ruby       = { workspace = true }
fastembed  = { workspace = true }
ignore     = { workspace = true }
rayon      = { workspace = true }
blake3     = { workspace = true }
serde      = { workspace = true }
anyhow     = { workspace = true }
thiserror  = { workspace = true }
tracing    = { workspace = true }
```

---

## 9. Core Data Structures

### 9.1 `CodeChunk` — The Central Value Type

Every piece of indexed source code is represented as a `CodeChunk`. This is the primary unit of indexing, storage, retrieval, and display. Every design decision in the system flows from this type.

```rust
// crates/indexer/src/lib.rs  (or a shared types module re-exported from all crates)

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct CodeChunk {
    // ─── Identity ────────────────────────────────────────────────────────────
    /// blake3(file_path + symbol_name + start_line) — stable across re-indexes
    pub chunk_id:        String,
    pub file_path:       PathBuf,
    pub language:        LanguageId,

    // ─── Location ────────────────────────────────────────────────────────────
    pub start_line:      u32,
    pub end_line:        u32,

    // ─── Symbol Metadata ─────────────────────────────────────────────────────
    /// Extracted from AST — None for anonymous blocks
    pub symbol_name:     Option<String>,
    /// "function" | "method" | "class" | "impl" | "trait" | "struct" | "route" | "rescue"
    pub symbol_kind:     Option<String>,
    /// Enclosing class/impl/module name — gives semantic context to methods
    pub parent_symbol:   Option<String>,

    // ─── Test Detection ───────────────────────────────────────────────────────
    /// true if path contains test/ *_test.* *_spec.* spec/ or has #[test] attr
    pub is_test:         bool,

    // ─── Content: ALWAYS keep these two fields SEPARATE ──────────────────────
    /// Raw source code — used for display to users and for BM25 scoring
    pub content:         String,
    /// Normalized Language/Symbol/Kind/Path/Code header — used ONLY for embedding
    pub embedding_text:  String,

    // ─── Structural Search ────────────────────────────────────────────────────
    /// AST shape fingerprint for --pattern search (e.g. "try_catch->loop->call")
    pub structural_fp:   String,

    // ─── Incremental Indexing ─────────────────────────────────────────────────
    /// blake3(content) — drives incremental skip decision
    pub content_hash:    String,
    /// Prior embedding snapshot — drives stale detection and semantic diff
    pub prev_embedding:  Option<Vec<f32>>,
}
```

> **Critical rule**: `content` and `embedding_text` are **never conflated**. `content` is raw source shown to users. `embedding_text` is the normalized format used only for vector generation. Conflating them degrades both display quality and retrieval quality.

### 9.2 `CallEdge` — The Relationship Graph

```rust
#[derive(Debug, Clone)]
pub struct CallEdge {
    /// chunk_id of the chunk that contains the call site
    pub caller_id:   String,
    /// Raw name of the symbol being called (extracted from AST call expression node)
    pub callee_name: String,
    /// chunk_id if the callee is indexed in this repo — None if external/stdlib
    pub callee_id:   Option<String>,
}
```

`CallEdge`s are extracted from the AST during indexing alongside chunks. They power three features: `--expand` (callers/callees in results), `--from` (reachability scoping), and `stale` detection. A noisy call graph degrades nothing in core semantic search.

### 9.3 `SearchResult` — The Output Type

```rust
#[derive(Debug, serde::Serialize)]
pub struct SearchResult {
    pub rank:             u32,
    pub chunk:            CodeChunk,
    pub score:            f32,           // final hybrid score
    pub semantic_score:   f32,           // raw cosine similarity from HNSW
    pub bm25_boost:       f32,           // BM25 component contribution
    pub confidence:       ConfidenceLevel,
    pub matched_queries:  Vec<String>,   // non-empty only for multi-query fan-out
    pub why:              Option<WhyData>,  // populated only with --why flag
}

#[derive(Debug, serde::Serialize)]
pub struct WhyData {
    pub embedding_text_used:  String,
    pub bm25_tokens_matched:  Vec<String>,
    pub cluster_id:           Option<u32>,
    pub reachability_path:    Option<Vec<String>>,
    pub score_breakdown:      ScoreBreakdown,
}

#[derive(Debug, serde::Serialize)]
pub struct ScoreBreakdown {
    pub semantic:      f32,
    pub bm25:          f32,
    pub alpha:         f32,
    pub cluster_bias:  f32,
    pub final_score:   f32,
}

#[derive(Debug, serde::Serialize, PartialEq)]
pub enum ConfidenceLevel {
    High,     // score > calibrated_threshold + 0.15
    Medium,   // score > calibrated_threshold
    Low,      // score <= calibrated_threshold — warn user
    Unknown,  // insufficient history for calibration (< 20 queries)
}
```

### 9.4 `LanguageId` — The Language Enum

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, serde::Serialize, serde::Deserialize)]
pub enum LanguageId {
    Rust,
    TypeScript,
    JavaScript,
    Python,
    Ruby,
    Unknown(/* extension */ String),
}

impl LanguageId {
    pub fn from_path(path: &Path) -> Self {
        match path.extension().and_then(|e| e.to_str()) {
            Some("rs")              => LanguageId::Rust,
            Some("ts") | Some("tsx") => LanguageId::TypeScript,
            Some("js") | Some("jsx") | Some("mjs") => LanguageId::JavaScript,
            Some("py")              => LanguageId::Python,
            Some("rb")              => LanguageId::Ruby,
            Some(ext)               => LanguageId::Unknown(ext.to_string()),
            None                    => LanguageId::Unknown(String::new()),
        }
    }

    pub fn display_name(&self) -> &str {
        match self {
            LanguageId::Rust       => "Rust",
            LanguageId::TypeScript => "TypeScript",
            LanguageId::JavaScript => "JavaScript",
            LanguageId::Python     => "Python",
            LanguageId::Ruby       => "Ruby",
            LanguageId::Unknown(_) => "Unknown",
        }
    }
}
```

---

## 10. Core Traits

### 10.1 `ChunkExtractor` Trait

The central abstraction for language-specific parsing. Every supported language implements this trait. The trait is `Send + Sync` so implementations can be used across `rayon` threads.

```rust
// crates/indexer/src/chunker.rs

pub trait ChunkExtractor: Send + Sync {
    fn language(&self) -> LanguageId;

    /// Primary extraction: produce all semantic chunks from a source file.
    /// MUST NOT panic on parse errors — return empty vec and set parse_error flag.
    fn extract_chunks(&self, path: &Path, source: &str) -> Vec<CodeChunk>;

    /// Secondary extraction: produce call edges for the relationship graph.
    /// May be empty if the language implementation does not support call extraction yet.
    fn extract_calls(&self, path: &Path, source: &str) -> Vec<CallEdge>;

    /// Structural fingerprinting: describe the AST shape of a node for --pattern search.
    /// Returns a normalized string like "try_catch->loop->recursive_call"
    fn extract_structural_fp(&self, node: &tree_sitter::Node, source: &str) -> String;

    /// Optional override for test detection. Default uses path-based heuristics.
    fn is_test_chunk(&self, path: &Path, _chunk: &CodeChunk) -> bool {
        is_test_path(path)  // checks for test/, *_test.*, *_spec.*, spec/ patterns
    }
}

/// Fallback test path detection used by all language impls as default
pub fn is_test_path(path: &Path) -> bool {
    let path_str = path.to_string_lossy().to_lowercase();
    path_str.contains("/test/")
        || path_str.contains("/tests/")
        || path_str.contains("/spec/")
        || path_str.contains("_test.")
        || path_str.contains("_spec.")
        || path_str.contains(".test.")
        || path_str.contains(".spec.")
        || path_str.contains("/__tests__/")
}
```

### 10.2 `Embedder` Trait

Hides `fastembed-rs` behind a clean interface, enabling model swaps without touching any other crate. v1 uses `all-MiniLM-L6-v2` (384 dimensions, ~22MB ONNX). Future: `nomic-embed-code` for better code recall on exact API names.

```rust
// crates/indexer/src/embedder.rs

pub trait Embedder: Send + Sync {
    /// Batch embedding for indexing — implementations should use rayon internally.
    /// texts is the embedding_text field from each CodeChunk, NOT raw source.
    fn embed_texts(&self, texts: &[String]) -> anyhow::Result<Vec<Vec<f32>>>;

    /// Single embedding for query — low-latency path, no batching.
    fn embed_query(&self, text: &str) -> anyhow::Result<Vec<f32>>;

    /// Required for schema validation on startup.
    fn dimension(&self) -> usize;

    /// Used in meta table to detect model changes between runs.
    fn model_name(&self) -> &str;
}

/// v1 concrete implementation using fastembed-rs
pub struct FastEmbedder {
    model: fastembed::TextEmbedding,
}

impl FastEmbedder {
    pub fn new() -> anyhow::Result<Self> {
        let model = fastembed::TextEmbedding::try_new(fastembed::InitOptions {
            model_name: fastembed::EmbeddingModel::AllMiniLML6V2,
            show_download_progress: true,
            ..Default::default()
        })?;
        Ok(Self { model })
    }
}

impl Embedder for FastEmbedder {
    fn embed_texts(&self, texts: &[String]) -> anyhow::Result<Vec<Vec<f32>>> {
        // fastembed-rs handles batching internally; rayon parallelism via internal threadpool
        let embeddings = self.model.embed(texts.to_vec(), None)?;
        Ok(embeddings)
    }

    fn embed_query(&self, text: &str) -> anyhow::Result<Vec<f32>> {
        let mut results = self.model.embed(vec![text.to_string()], None)?;
        results.pop().ok_or_else(|| anyhow::anyhow!("embedding returned empty result"))
    }

    fn dimension(&self) -> usize { 384 }
    fn model_name(&self) -> &str { "Xenova/all-MiniLM-L6-v2" }
}
```

---

## 11. Storage Schema

All persistent data lives in `.smart-grep/index.db`. The canonical schema is in `crates/store/src/schema.sql` — this file is the single source of truth for migrations. Schema version is stored in the `meta` table and checked on every startup.

```sql
-- crates/store/src/schema.sql
-- CANONICAL DDL — single source of truth for migrations

-- ─── Files ──────────────────────────────────────────────────────────────────
-- Track all indexed files with change detection metadata
CREATE TABLE files (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  path         TEXT    UNIQUE NOT NULL,           -- path relative to repo root
  language     TEXT,                              -- NULL if unsupported extension
  mtime        INTEGER NOT NULL,                  -- unix timestamp, last modified
  size_bytes   INTEGER NOT NULL,
  content_hash TEXT    NOT NULL,                  -- blake3 hex — drives incremental skip
  parse_error  INTEGER NOT NULL DEFAULT 0         -- 1 if tree-sitter parse failed
);

-- ─── Chunks ─────────────────────────────────────────────────────────────────
-- All semantic chunks extracted from files
CREATE TABLE chunks (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  file_id        INTEGER NOT NULL,
  chunk_key      TEXT    UNIQUE NOT NULL,          -- blake3(file_path+symbol+start_line)
  symbol_name    TEXT,
  symbol_kind    TEXT,                             -- 'function'|'method'|'class'|'impl'|...
  parent_symbol  TEXT,                             -- enclosing class/impl/module name
  is_test        INTEGER NOT NULL DEFAULT 0,       -- 1 if in test file or marked #[test]
  start_line     INTEGER NOT NULL,
  end_line       INTEGER NOT NULL,
  content        TEXT    NOT NULL,                 -- raw source — display + BM25
  embedding_text TEXT    NOT NULL,                 -- normalized — embedding only
  structural_fp  TEXT,                             -- AST shape fingerprint for --pattern
  content_hash   TEXT    NOT NULL,                 -- blake3(content) for incremental skip
  FOREIGN KEY(file_id) REFERENCES files(id) ON DELETE CASCADE
);

-- ─── Embeddings ─────────────────────────────────────────────────────────────
-- Vector embeddings, kept separate from chunks for clean HNSW rebuild
CREATE TABLE embeddings (
  chunk_id    INTEGER PRIMARY KEY,
  dim         INTEGER NOT NULL,
  vector      BLOB    NOT NULL,                    -- float32 little-endian packed bytes
  prev_vector BLOB,                                -- prior run snapshot — semantic diff + stale
  FOREIGN KEY(chunk_id) REFERENCES chunks(id) ON DELETE CASCADE
);

-- ─── Call Graph ─────────────────────────────────────────────────────────────
-- Function-level call and reference edges
CREATE TABLE call_edges (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  caller_id   INTEGER NOT NULL,                    -- chunk_id of calling chunk
  callee_name TEXT    NOT NULL,                    -- raw symbol name from AST
  callee_id   INTEGER,                             -- chunk_id if indexed; NULL if external
  FOREIGN KEY(caller_id) REFERENCES chunks(id) ON DELETE CASCADE
);

-- ─── Synonym Map ────────────────────────────────────────────────────────────
-- Codebase-derived concept→symbol correlation map for query expansion
CREATE TABLE synonyms (
  concept TEXT NOT NULL,
  symbol  TEXT NOT NULL,
  weight  REAL NOT NULL DEFAULT 1.0,               -- co-occurrence frequency weight
  PRIMARY KEY(concept, symbol)
);

-- ─── Snapshots ──────────────────────────────────────────────────────────────
-- Named index snapshots for time-travel search
CREATE TABLE snapshots (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  ref_name   TEXT    NOT NULL,                     -- "pre-refactor", "v2.1.0", "HEAD~10"
  created_at INTEGER NOT NULL,                     -- unix timestamp
  db_path    TEXT    NOT NULL                      -- path under .smart-grep/snapshots/
);

-- ─── Sessions ───────────────────────────────────────────────────────────────
-- Agent session tracking for multi-query context assembly
CREATE TABLE sessions (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  name       TEXT    NOT NULL,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);

CREATE TABLE session_results (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id INTEGER NOT NULL,
  query      TEXT    NOT NULL,
  chunk_id   INTEGER NOT NULL,
  score      REAL    NOT NULL,
  queried_at INTEGER NOT NULL,
  FOREIGN KEY(session_id) REFERENCES sessions(id) ON DELETE CASCADE
);

-- ─── Confidence Calibration ─────────────────────────────────────────────────
-- Rolling window of historical top-1 scores for per-codebase threshold calibration
CREATE TABLE query_scores (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  query_hash TEXT    NOT NULL,
  top1_score REAL    NOT NULL,
  queried_at INTEGER NOT NULL
);

-- ─── Metadata ───────────────────────────────────────────────────────────────
-- Model metadata and schema versioning
CREATE TABLE meta (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
-- Required rows: vector_dim, model_name, schema_version, last_indexed_at, chunk_count

-- ─── Indexes ────────────────────────────────────────────────────────────────
CREATE INDEX idx_chunks_file_id    ON chunks(file_id);
CREATE INDEX idx_chunks_is_test    ON chunks(is_test);
CREATE INDEX idx_chunks_symbol     ON chunks(symbol_name);
CREATE INDEX idx_chunks_kind       ON chunks(symbol_kind);
CREATE INDEX idx_call_edges_caller ON call_edges(caller_id);
CREATE INDEX idx_call_edges_callee ON call_edges(callee_id);
CREATE INDEX idx_session_results   ON session_results(session_id, score DESC);
CREATE INDEX idx_query_scores_time ON query_scores(queried_at DESC);
CREATE INDEX idx_synonyms_concept  ON synonyms(concept);
```

> **Critical**: `ON DELETE CASCADE` on all FK relationships is the key correctness guarantee for incremental indexing. When a file row is deleted (because the file was removed), every chunk, embedding, call edge, and session reference for that file is automatically cleaned up — zero orphan cleanup code needed.

### 11.1 HNSW Index (`crates/store/src/index.rs`)

The HNSW vector index is managed separately from SQLite and lives in `.smart-grep/hnsw.usearch`.

```rust
// crates/store/src/index.rs

pub struct HnswIndex {
    index: usearch::Index,
    path:  PathBuf,
}

impl HnswIndex {
    pub fn open_or_create(path: &Path, dim: usize) -> anyhow::Result<Self> {
        let options = usearch::IndexOptions {
            dimensions: dim,
            metric: usearch::MetricKind::Cos,  // cosine similarity
            connectivity: 16,                   // HNSW M parameter
            expansion_add: 128,                 // ef_construction
            expansion_search: 64,               // ef_search
            ..Default::default()
        };
        // ...
    }

    /// Add or update a vector for a chunk_id
    pub fn upsert(&mut self, chunk_id: u64, vector: &[f32]) -> anyhow::Result<()> { ... }

    /// ANN search — returns (chunk_id, cosine_similarity) pairs, highest first
    pub fn search(&self, query: &[f32], top_k: usize) -> anyhow::Result<Vec<(u64, f32)>> { ... }

    /// Rebuild from scratch using all embeddings in SQLite — used after incremental mutations
    pub fn rebuild_from_db(&mut self, db: &Db) -> anyhow::Result<()> { ... }
}
```

---

## 12. Embedding Text Normalization

The quality of semantic retrieval is almost entirely determined by the quality of the text fed to the embedding model — not the model architecture itself. Raw source code embedded directly performs significantly worse than a normalized format that provides explicit semantic context to the model.

### 12.1 The Normalized Format

Every `CodeChunk`'s `embedding_text` is built using this template:

```
Language: Rust
Symbol: handle_payment_error
Kind: function
Parent: PaymentService
Path: src/billing/errors.rs
Code:
fn handle_payment_error(err: &PaymentError) -> Response {
    match err {
        PaymentError::CardDeclined(msg) => Response::decline(msg),
        PaymentError::InsufficientFunds => Response::retry_later(),
        _ => Response::internal_error(),
    }
}
```

The `content` field (stored separately and shown to users) is just the raw function body without any header.

### 12.2 Why Each Header Field Matters

| Field | Why it improves retrieval |
|---|---|
| `Language: Rust` | Disambiguates keywords shared across languages (`def` in Ruby vs Python, `function` in JS vs TypeScript). |
| `Symbol: handle_payment_error` | For short functions where the name carries most of the meaning, the header ensures the model sees it clearly — even if the body is trivial. |
| `Kind: function` | Separates functions from classes and methods in embedding space, reducing cross-kind false positives when querying for "the User class" vs "create_user function". |
| `Parent: PaymentService` | Context for methods — `validate` inside `PaymentService` is semantically different from `validate` inside `UserService`. |
| `Path: src/billing/errors.rs` | Directory structure carries strong semantic signal about module responsibility. |

### 12.3 Implementation: `embedding_text.rs`

```rust
// crates/indexer/src/embedding_text.rs

const MAX_EMBEDDING_TOKENS: usize = 256;  // all-MiniLM-L6-v2 max context window

pub fn build_embedding_text(chunk: &CodeChunk) -> String {
    let mut parts = Vec::new();

    parts.push(format!("Language: {}", chunk.language.display_name()));

    if let Some(sym) = &chunk.symbol_name {
        parts.push(format!("Symbol: {}", sym));
    }
    if let Some(kind) = &chunk.symbol_kind {
        parts.push(format!("Kind: {}", kind));
    }
    if let Some(parent) = &chunk.parent_symbol {
        parts.push(format!("Parent: {}", parent));
    }

    parts.push(format!("Path: {}", chunk.file_path.display()));
    parts.push(String::from("Code:"));

    // Truncate to model's context window before appending
    let code = truncate_to_tokens(&chunk.content, MAX_EMBEDDING_TOKENS);
    parts.push(code);

    parts.join("\n")
}

fn truncate_to_tokens(text: &str, max_tokens: usize) -> String {
    // Approximate: token_count ≈ word_count * 1.3 for code
    let words: Vec<&str> = text.split_whitespace().collect();
    let approx_tokens = (words.len() as f32 * 1.3) as usize;
    if approx_tokens <= max_tokens {
        return text.to_string();
    }
    let word_limit = (max_tokens as f32 / 1.3) as usize;
    words[..word_limit.min(words.len())].join(" ") + " ..."
}
```

---

## 13. Chunking Strategy

Search quality is almost entirely determined by chunk quality. This is the most important design decision in the entire project. The core principle: index code as **semantic chunks** — AST-defined units of meaning — not arbitrary fixed-size text windows.

### 13.1 Good Chunk Candidates

| Chunk type | Why it is a good semantic unit |
|---|---|
| Function / method definition | Smallest complete unit of behavior. Has a name, inputs, outputs, and a single responsibility. |
| Class / struct block | Complete type definition with its invariants. |
| Trait / interface declaration | Defines a behavioral contract. |
| Error handling branches (`catch`, `rescue`, `match` on error types) | Critical semantic unit for "where is X error handled" queries. |
| Route / controller action | Named request handler — "checkout endpoint" should return this. |
| SQL / query-builder blocks | Named queries have strong identity: "where do we query active users". |
| Config sections (when named) | e.g. a service configuration block with a conceptual identity. |

### 13.2 Size Bounds

```toml
# .smart-grep/config.toml
[index]
min_lines = 5    # skip trivial one-liners (getters, empty stubs, re-exports)
max_lines = 80   # split overly large functions at logical sub-blocks
```

**Min bound**: Trivial one-liners (getters, empty stubs, re-exports) add noise to the index without useful semantic signal. Skip them.

**Max bound**: When a function exceeds 80 lines, split it at logical AST boundaries (nested function, block, `match` arm) rather than arbitrary line numbers. Each sub-chunk inherits the parent symbol name suffixed with an index: `handle_payment::0`, `handle_payment::1`.

> **Critical rule**: Never split at arbitrary line numbers. Always split at AST boundaries. A chunk cut in the middle of a conditional is semantically incoherent and degrades retrieval quality.

### 13.3 Chunk ID Generation

```rust
pub fn chunk_id(file_path: &Path, symbol_name: &str, start_line: u32) -> String {
    let key = format!("{}:{}:{}", file_path.display(), symbol_name, start_line);
    blake3::hash(key.as_bytes()).to_hex().to_string()
}
```

The chunk ID is stable across re-indexes as long as the file path, symbol name, and start line do not change. This is the key property that enables incremental indexing (skip unchanged chunks) and stale detection (compare current vs. prior embedding for same chunk_id).

---

## 14. Parsing Layer

### 14.1 Language Dispatch

```rust
// crates/indexer/src/languages.rs

pub fn get_extractor(language: LanguageId) -> Option<Box<dyn ChunkExtractor>> {
    match language {
        LanguageId::Rust       => Some(Box::new(RustExtractor::new())),
        LanguageId::TypeScript => Some(Box::new(TypeScriptExtractor::new())),
        LanguageId::JavaScript => Some(Box::new(JavaScriptExtractor::new())),
        LanguageId::Python     => Some(Box::new(PythonExtractor::new())),
        LanguageId::Ruby       => Some(Box::new(RubyExtractor::new())),
        LanguageId::Unknown(_) => None,  // skip silently
    }
}
```

### 14.2 Parse Error Handling

```rust
// All extractors follow this pattern — NEVER panic on parse errors
pub fn extract_chunks(&self, path: &Path, source: &str) -> Vec<CodeChunk> {
    let mut parser = tree_sitter::Parser::new();
    parser.set_language(tree_sitter_rust::language()).unwrap();

    let tree = match parser.parse(source, None) {
        Some(t) => t,
        None => {
            // Log warning but do NOT crash
            tracing::warn!("tree-sitter parse failed for {}", path.display());
            return vec![];  // file will have parse_error = 1 in files table
        }
    };

    // ... walk the tree and extract chunks
}
```

### 14.3 Indexing Orchestration (`crates/indexer/src/chunker.rs`)

```rust
pub fn index_path(
    root: &Path,
    config: &IndexerConfig,
    store: &mut Store,
    embedder: &dyn Embedder,
) -> anyhow::Result<IndexStats> {
    let walker = ignore::WalkBuilder::new(root)
        .add_custom_ignore_filename(".smart-grep-ignore")
        .build_parallel();

    // Collect files to process
    let all_files: Vec<PathBuf> = walker.into_iter()
        .filter_map(|e| e.ok())
        .filter(|e| e.file_type().map_or(false, |t| t.is_file()))
        .map(|e| e.into_path())
        .collect();

    // Parallel file processing via rayon
    let results: Vec<(Vec<CodeChunk>, Vec<CallEdge>, FileState)> = all_files.par_iter()
        .map(|path| process_file(path, root, config, store))
        .collect();

    // Batch embed all new/changed chunks
    let chunks_to_embed: Vec<&CodeChunk> = results.iter()
        .flat_map(|(chunks, _, state)| {
            if matches!(state, FileState::New | FileState::Changed) {
                chunks.iter()
            } else {
                [].iter()
            }
        })
        .collect();

    let texts: Vec<String> = chunks_to_embed.iter()
        .map(|c| c.embedding_text.clone())
        .collect();
    let embeddings = embedder.embed_texts(&texts)?;

    // Write to store
    store.write_batch(&results, &embeddings)?;
    store.rebuild_hnsw()?;
    store.build_synonym_map()?;

    Ok(IndexStats { /* ... */ })
}
```

---

## 15. Embedding Layer

### 15.1 v1 Model: `all-MiniLM-L6-v2`

| Property | Value |
|---|---|
| Model | `Xenova/all-MiniLM-L6-v2` (via `fastembed-rs`) |
| Dimensions | 384 |
| ONNX file size | ~22MB |
| Inference runtime | `ort` (ONNX Runtime) bundled in `fastembed` |
| Latency (query) | ~2–5ms on CPU (M1 Mac), ~8–15ms on older laptops |
| Latency (batch index, 1000 chunks) | ~3–10s on CPU via rayon parallelism |
| Internet required | No — model downloaded once to `~/.cache/fastembed` on first run |
| API key required | No |

### 15.2 Future Model: `nomic-embed-code`

For v1.1: swap to `nomic-embed-code` (code-specific fine-tune, 768 dimensions) for better recall on exact API name queries. The `Embedder` trait makes this a one-file change with no impact on any other crate. Detected on startup via `meta.model_name` mismatch → auto-trigger `--rebuild`.

### 15.3 Embedding Batch Strategy

During indexing, chunks are batched in groups of 64 (configurable) and embedded via `fastembed`'s internal batching. `rayon` is used for parallel text preprocessing but embedding itself runs single-threaded on the ONNX runtime (thread safety constraints of ONNX Runtime v1).

```rust
const EMBED_BATCH_SIZE: usize = 64;

pub fn embed_in_batches(
    embedder: &dyn Embedder,
    texts: &[String],
) -> anyhow::Result<Vec<Vec<f32>>> {
    texts.chunks(EMBED_BATCH_SIZE)
        .map(|batch| embedder.embed_texts(batch))
        .try_fold(Vec::new(), |mut acc, batch_result| {
            acc.extend(batch_result?);
            Ok(acc)
        })
}
```

---

## 16. Hybrid Search: BM25 + HNSW

Pure semantic search has a known failure mode: exact symbol name queries. If a user types `handleChargeFailure` as their query, cosine similarity may not guarantee this function ranks #1 — other semantically similar functions can outscore it. The solution is hybrid retrieval, the same approach used in production RAG systems like Elasticsearch and Weaviate.

### 16.1 Algorithm

```
1. embed query (possibly synonym-expanded)
2. HNSW ANN search → top-50 semantic candidates (large pool to re-rank from)
3. BM25 score all 50 candidates via SQLite content column
4. final_score = alpha * semantic_score + (1 - alpha) * bm25_normalized
5. re-rank by final_score → return top-N (default 5)
```

### 16.2 Scoring Formula

```rust
// crates/search/src/rank.rs

/// alpha = 0.7 by default: heavily semantic, BM25 as tiebreaker for exact symbol names
pub fn hybrid_score(semantic: f32, bm25: f32, alpha: f32) -> f32 {
    alpha * semantic + (1.0 - alpha) * bm25
}
```

`alpha` is tunable via `.smart-grep/config.toml` `[search] hybrid_alpha`. A value of 1.0 = pure semantic. A value of 0.0 = pure BM25 (effectively a fast ranked keyword search).

### 16.3 BM25 Implementation (`crates/search/src/bm25.rs`)

~150 lines of pure Rust. No new dependencies. Runs over the already-stored `content` column in SQLite.

```rust
pub struct Bm25Scorer {
    k1:     f32,     // term frequency saturation — default 1.5
    b:      f32,     // length normalization — default 0.75
    avg_dl: f32,     // average document length across all chunks (computed once at index time)
    df:     HashMap<String, u32>,  // document frequency per term (computed at index time)
    n:      u32,     // total number of chunks
}

impl Bm25Scorer {
    pub fn score(&self, query_terms: &[&str], doc_content: &str, doc_len: usize) -> f32 {
        let tokens = tokenize(doc_content);
        let tf_map = term_frequencies(&tokens);

        query_terms.iter().map(|term| {
            let tf = *tf_map.get(*term).unwrap_or(&0) as f32;
            let df = *self.df.get(*term).unwrap_or(&0) as f32;
            let idf = ((self.n as f32 - df + 0.5) / (df + 0.5) + 1.0).ln();
            let tf_norm = tf * (self.k1 + 1.0)
                / (tf + self.k1 * (1.0 - self.b + self.b * doc_len as f32 / self.avg_dl));
            idf * tf_norm
        }).sum()
    }
}

/// CRITICAL: tokenizer must handle camelCase and snake_case splitting.
/// 'handlePaymentError' → ['handle', 'payment', 'error']
/// 'retry_with_backoff' → ['retry', 'with', 'backoff']
pub fn tokenize(text: &str) -> Vec<String> {
    let mut tokens = Vec::new();
    // Split on non-alphanumeric first
    for word in text.split(|c: char| !c.is_alphanumeric()) {
        if word.is_empty() { continue; }
        // Then split camelCase within each word
        let subtokens = split_camel_case(word);
        tokens.extend(subtokens.into_iter().map(|s| s.to_lowercase()));
    }
    tokens.dedup();
    tokens
}
```

> **Key insight**: The BM25 tokenizer MUST split camelCase and snake_case, because code identifiers carry meaning at the sub-word level. `handlePaymentError` should match the BM25 query token `payment`. This is the single most important implementation detail in the BM25 component.

### 16.4 BM25 Stats Computation

BM25 `avg_dl` and `df` (document frequency) are computed once during indexing and cached in the `meta` table. They are updated incrementally: on file change, subtract the old chunk's contribution and add the new one.

```rust
// Stored in meta table:
// meta: { key: "bm25_avg_dl", value: "42.3" }
// meta: { key: "bm25_df_json", value: "{\"payment\": 47, \"error\": 83, ...}" }
```

---

## 17. Query Expansion via Synonym Map

Before embedding a query, run it through a codebase-derived synonym expander. This is essentially zero-shot domain adaptation of the embedding model to each specific codebase's naming conventions — built entirely from the repo itself, with no external data.

### 17.1 How the Synonym Map is Built

During indexing, `crates/indexer/src/synonym_map.rs` performs concept-to-symbol co-occurrence analysis:

1. For each pair of chunks that appear in the **same file** or are connected via a **call edge**, record their symbol names as co-occurring.
2. For each symbol name, tokenize it into semantic terms (camelCase/snake_case split, lowercase).
3. Build a weight matrix: `concept_term → {symbol_token: co_occurrence_count}`.
4. Persist the top-10 symbols per concept to the `synonyms` table.

**Example output for a typical billing codebase:**

```
"payment" → [stripe, charge, billing, invoice, card, transaction, gateway, process]
"error"   → [exception, failure, decline, reject, invalid, refused, timeout]
"retry"   → [backoff, attempt, reattempt, queue, exponential, delay, sleep]
"auth"    → [token, jwt, session, verify, permission, role, scope, authorize]
"user"    → [account, profile, member, customer, subscriber, identity]
```

### 17.2 Query Expansion at Search Time

```rust
// crates/search/src/expand.rs

pub fn expand_query(query: &str, synonyms: &SynonymMap, max_terms: usize) -> String {
    let query_tokens = tokenize(query);

    let expansions: IndexSet<String> = query_tokens.iter()
        .flat_map(|token| synonyms.top_k(token, 5))
        .take(max_terms)  // cap total expansion — default 10
        .collect();

    if expansions.is_empty() {
        query.to_string()
    } else {
        format!("{} {}", query, expansions.into_iter().collect::<Vec<_>>().join(" "))
    }
}
```

**Applied to an example query:**

```
input:    "where is payment error handled?"
tokens:   ['payment', 'error', 'handled']
synonyms: payment → [stripe, charge, billing], error → [exception, failure, decline]
output:   "where is payment error handled? stripe charge billing exception failure decline"
```

This expanded query is then embedded as a single string. The expansion costs one extra embedding call (~2–5ms) and dramatically improves recall on repos with idiosyncratic naming. Disable via `[search] query_expansion = false`.

---

## 18. Query-by-Example (`--like`)

Instead of describing what to search for in words, paste a code snippet and smart-grep finds everything semantically similar to it:

```bash
smart-grep --like '
def retry_with_backoff(fn, max_attempts=3):
    for attempt in range(max_attempts):
        try:
            return fn()
        except TransientError:
            time.sleep(2 ** attempt)
'

# Also works with stdin — composable with shell workflows
cat src/billing/stripe.py | smart-grep --like -
```

### 18.1 Why This Works

The snippet runs through the same `embedding_text` normalizer and embeds directly — no natural-language-to-code translation gap, because the query *is* code. This is the right interface for refactoring workflows: the most common case where you need to find similar code is when you are already looking at an instance of it.

### 18.2 Use Cases

- "Find all places we use this same retry pattern" — paste one instance, find all analogues
- "Find the Rust equivalent of this Python function I just wrote"
- "Find similar error handlers to this one I am refactoring"
- Find cross-language analogues of a pattern across a polyglot repo

### 18.3 Implementation

```rust
// In the search command handler (crates/core/src/main.rs)

if let Some(snippet) = &args.like {
    let source = if snippet == "-" {
        // Read from stdin
        let mut buf = String::new();
        std::io::stdin().read_to_string(&mut buf)?;
        buf
    } else {
        snippet.clone()
    };

    // Build a synthetic CodeChunk for the snippet and normalize it
    let synthetic_chunk = CodeChunk {
        content: source.clone(),
        embedding_text: build_embedding_text_for_snippet(&source),
        // ... other fields with sensible defaults
    };

    let query_vec = embedder.embed_query(&synthetic_chunk.embedding_text)?;
    // Then proceed with normal HNSW search using query_vec
}
```

Zero additional infrastructure beyond what already exists. The `--like` implementation reuses `embedding_text.rs` normalization and the standard HNSW search path.

---

## 19. Structural Pattern Search (`--pattern`)

A complementary search mode that operates on **AST structure** rather than semantic embeddings. Finds code that follows the same design pattern regardless of domain.

### 19.1 Structural Fingerprinting

During indexing, `crates/indexer/src/structural.rs` extracts a normalized shape fingerprint from the AST of each chunk:

```
try_catch -> condition_check -> recursive_call     # retry pattern
class -> [__init__, validate, execute]             # command pattern  
function -> [guard_clause, delegate_call]          # middleware pattern
match -> [error_arm, success_arm]                  # result handling pattern
```

The fingerprint describes the chunk's structural form using normalized node type names — not the actual variable/function names, which are domain-specific.

### 19.2 Usage

```bash
smart-grep --pattern "function that catches error and retries"
smart-grep --pattern "class with init validate execute methods"
smart-grep --pattern "middleware that checks condition then calls next"
```

```
STRUCTURAL PATTERN: "function that catches error and retries"

  src/billing/stripe.rs::handle_charge_failure    shape-match: 0.94
  src/jobs/worker.rs::process_with_retry          shape-match: 0.91
  lib/http/client.rb::request_with_backoff        shape-match: 0.88
```

### 19.3 Semantic vs. Structural — Orthogonal Dimensions

| Mode | Finds | Example |
|---|---|---|
| Semantic (`--like`) | "Related concept" — similar meaning | `--like` a payment function finds all payment functions |
| Structural (`--pattern`) | "Same architecture" — same design pattern | `--pattern retry` finds all retry patterns regardless of domain |
| Combined | Both semantically similar AND structurally similar | Extremely precise for refactoring a specific pattern |

**Combined usage:**
```bash
smart-grep --like 'rescue Stripe::CardError => e' --pattern "catches error and retries"
# Finds functions that are BOTH about payment errors AND have retry structure
```

### 19.4 Implementation

Structural fingerprinting is a tree-sitter AST traversal outputting a normalized shape string, stored in `chunks.structural_fp`. Fuzzy matching at query time is edit-distance or substring matching on those strings. No new ML, no new dependencies.

```rust
// crates/indexer/src/structural.rs

pub fn extract_structural_fp(node: &tree_sitter::Node, source: &str) -> String {
    let mut parts = Vec::new();
    extract_shape_recursive(node, source, &mut parts, 0);
    parts.join("->")
}

fn extract_shape_recursive(node: &Node, source: &str, parts: &mut Vec<String>, depth: usize) {
    if depth > 5 { return; }  // cap depth to prevent huge fingerprints

    let kind = normalize_node_kind(node.kind());
    if is_structural_node(node.kind()) {
        parts.push(kind);
    }

    for child in node.children(&mut node.walk()) {
        extract_shape_recursive(&child, source, parts, depth + 1);
    }
}

fn normalize_node_kind(kind: &str) -> String {
    match kind {
        "try_statement" | "rescue_clause" => "try_catch",
        "for_statement" | "while_statement" | "loop_expression" => "loop",
        "if_statement" | "guard_statement" => "condition_check",
        "call_expression" | "method_call" => "call",
        "return_statement" => "return",
        _ => kind,
    }.to_string()
}
```

---

## 20. Relationship Graph + Contextual Expansion

During indexing, `crates/indexer/src/graph.rs` extracts **call edges** from the AST — function call sites and symbol references — stored in the `call_edges` table.

### 20.1 `--expand` Flag

Automatically include the **callers and callees** of each top result:

```bash
smart-grep "handle charge failure" --expand
# Returns: handle_charge_failure + its callers + its direct callees
```

```
1. src/billing/stripe.rs::handle_charge_failure  score=0.91  [TARGET]
   CALLERS:
   ├── src/billing/processor.rs::process_payment  score=0.84
   └── src/jobs/billing_job.rs::retry_charge      score=0.79
   CALLEES:
   ├── src/billing/log.rs::log_decline            score=0.71
   └── src/billing/notify.rs::notify_customer     score=0.68
```

This gives the agent or developer the full context of a function — not just the function itself but everything that calls it and everything it calls — in a single query.

### 20.2 Implementation (`crates/search/src/expand.rs`)

```rust
pub fn expand_results(
    results: Vec<SearchResult>,
    db: &Db,
    depth: u32,  // default: 1 hop
) -> anyhow::Result<Vec<SearchResult>> {
    let mut expanded = results.clone();

    for result in &results {
        let chunk_id = &result.chunk.chunk_id;

        // Get callers (who calls this chunk)
        let callers = db.get_callers(chunk_id)?;
        for caller_id in callers {
            if let Some(caller_chunk) = db.get_chunk(&caller_id)? {
                expanded.push(SearchResult {
                    rank: 0,  // will be re-ranked
                    chunk: caller_chunk,
                    score: result.score * 0.8,  // discount for indirection
                    // ...
                });
            }
        }

        // Get callees (what this chunk calls)
        let callees = db.get_callees(chunk_id)?;
        for callee_id in callees.into_iter().flatten() {
            if let Some(callee_chunk) = db.get_chunk(&callee_id)? {
                expanded.push(SearchResult {
                    rank: 0,
                    chunk: callee_chunk,
                    score: result.score * 0.8,
                    // ...
                });
            }
        }
    }

    // Dedup by chunk_id, keeping highest score per chunk
    dedup_by_chunk_id(expanded)
}
```

---

## 21. Reachability-Scoped Search (`--from`)

Search only the **call-graph-reachable subset** from a given entry point. Performs BFS over `call_edges` from the specified symbol and restricts HNSW search to that subgraph.

```bash
smart-grep "payment error handling" --from src/api/checkout.rs::handle_checkout
# Only searches code reachable from handle_checkout
```

### 21.1 The Problem This Solves

In large monorepos, a query for "authentication" may return results from 12 different services. But the developer is debugging one specific request flow and only cares about the auth logic reachable from *this specific request handler*. `--from` makes this precise.

### 21.2 Implementation (`crates/search/src/reachability.rs`)

```rust
pub fn reachable_chunk_ids(
    entry_symbol: &str,
    db: &Db,
    max_depth: u32,  // configurable via --from-depth, default 3
) -> anyhow::Result<HashSet<String>> {
    let entry_chunk_id = db.find_chunk_by_symbol(entry_symbol)?
        .ok_or_else(|| anyhow::anyhow!(
            "Symbol '{}' not found in index. Run 'smart-grep index' first.", entry_symbol
        ))?;

    let mut visited = HashSet::new();
    let mut queue = VecDeque::new();
    queue.push_back((entry_chunk_id, 0u32));

    while let Some((chunk_id, depth)) = queue.pop_front() {
        if visited.contains(&chunk_id) || depth > max_depth { continue; }
        visited.insert(chunk_id.clone());

        for callee_id in db.get_callees(&chunk_id)?.into_iter().flatten() {
            queue.push_back((callee_id, depth + 1));
        }
    }

    Ok(visited)
}
```

Then, in the search path:

```rust
// Pre-filter: only search HNSW over chunks in the reachable set
let reachable = reachable_chunk_ids(entry_symbol, db, config.from_depth)?;
let candidates = store.hnsw_search_filtered(query_vec, top_k * 5, &reachable)?;
// ... rest of hybrid scoring proceeds normally
```

~50 lines of code on top of existing infrastructure. `--from-depth` (default 3) is configurable to prevent BFS explosion on large, highly-connected graphs.

---

## 22. Confidence-Aware Results

Rather than always returning top-K results regardless of quality — which leads to silent false positives — smart-grep tracks the distribution of match scores for this specific codebase and warns when results fall below the historical baseline.

### 22.1 Confidence Modes

```bash
# Default: warn + suggest alternatives if best result is below calibrated threshold
smart-grep "kubernetes pod scheduling logic"
# WARNING: LOW CONFIDENCE — best score 0.41 is below calibrated threshold 0.60
#   This codebase may not contain Kubernetes scheduling logic.
#   Suggestion: try  smart-grep map  to see what this codebase does contain.
#   Nearest cluster: "payment processing" (your query may be misdirected)

# Strict: exit code 1 if no result above threshold — safe for CI pipelines
smart-grep "payment handler" --confidence strict

# Off: always return results with no annotation
smart-grep "payment handler" --confidence off
```

### 22.2 Calibration Algorithm (`crates/search/src/confidence.rs`)

```rust
pub struct ConfidenceCalibrator {
    query_scores: Vec<f32>,  // rolling window of top-1 scores, max 100 entries
}

impl ConfidenceCalibrator {
    /// Threshold = 20th percentile of historical top-1 scores.
    /// Calibrated per-codebase, not a global magic number.
    pub fn threshold(&self) -> f32 {
        if self.query_scores.len() < 20 {
            return 0.60;  // hardcoded fallback until enough history
        }
        let mut sorted = self.query_scores.clone();
        sorted.sort_by(|a, b| a.partial_cmp(b).unwrap());
        sorted[sorted.len() / 5]  // 20th percentile
    }

    pub fn classify(&self, score: f32) -> ConfidenceLevel {
        let threshold = self.threshold();
        if score > threshold + 0.15 { ConfidenceLevel::High }
        else if score > threshold    { ConfidenceLevel::Medium }
        else if self.query_scores.len() >= 20 { ConfidenceLevel::Low }
        else                         { ConfidenceLevel::Unknown }
    }

    pub fn record(&mut self, top1_score: f32) {
        self.query_scores.push(top1_score);
        if self.query_scores.len() > 100 {
            self.query_scores.remove(0);  // sliding window
        }
    }
}
```

### 22.3 Low-Confidence Output

When confidence is low, smart-grep actively helps rather than silently returning bad results:

```
WARNING: LOW CONFIDENCE
  Best match score: 0.41
  Calibrated threshold: 0.60 (based on 47 historical queries on this codebase)
  Strong matches on this codebase typically score 0.82+

Suggestions:
  - Run 'smart-grep map' to see what this codebase does contain
  - Nearest cluster to your query: "payment processing" — did you mean billing retry?
  - Try reformulating: 'smart-grep "charge failure resilience"'
```

---

## 23. Incremental Indexing

### 23.1 File State Transitions

```rust
pub enum FileState {
    New,        // file not in files table → parse + embed + insert
    Changed,    // mtime or content_hash differs → snapshot prev, delete old, re-parse, insert
    Deleted,    // path no longer on disk → DELETE from files (CASCADE handles dependents)
    Unchanged,  // identical content hash → skip entirely (zero work)
}
```

### 23.2 The Changed File Path

```rust
fn handle_changed_file(path: &Path, db: &mut Db, embedder: &dyn Embedder) -> anyhow::Result<()> {
    // 1. Snapshot current embeddings to prev_vector before deletion
    db.snapshot_embeddings_to_prev(path)?;

    // 2. Delete the file row (CASCADE cleans chunks + embeddings + call_edges)
    db.delete_file(path)?;

    // 3. Re-parse and re-embed the file
    let chunks = extract_chunks(path)?;
    let texts: Vec<String> = chunks.iter().map(|c| c.embedding_text.clone()).collect();
    let embeddings = embedder.embed_texts(&texts)?;

    // 4. Insert fresh data
    db.insert_file(path, &chunks, &embeddings)?;

    Ok(())
}
```

The `prev_vector` snapshot is the critical step that enables stale detection and semantic diff. After the cascade delete, the old embeddings are gone — but their values live on in `prev_vector` for the new chunks that replace them (matched by `chunk_key`).

### 23.3 HNSW Rebuild Strategy (v1)

For v1: detect changed files, apply SQLite mutations, then **rebuild the HNSW index from all current embeddings**. This is safe, correct, and fast enough for repos under ~100k functions (rebuild takes ~2–5 seconds on typical hardware).

v1.1 target: **online HNSW updates** — `usearch` supports adding and removing vectors individually. This would reduce post-mutation re-index to under 100ms.

```bash
# Force full re-index, bypassing all incremental logic
smart-grep index --rebuild

# Also triggered automatically when meta.vector_dim mismatches current model
```

---

## 24. Context Budget Mode (`--budget`)

Agents and LLM workflows have a fixed token budget. Instead of returning top-K chunks and leaving token management to the caller, smart-grep gets token-aware.

### 24.1 Usage

```bash
smart-grep "payment error handling" --budget 4000
```

### 24.2 Algorithm (`crates/search/src/budget.rs`)

```rust
pub fn pack_budget(
    results: Vec<SearchResult>,
    budget_tokens: usize,
) -> Vec<SearchResult> {
    let mut packed = Vec::new();
    let mut total_tokens = 0;

    for result in results.into_iter().sorted_by(|a, b| b.score.partial_cmp(&a.score).unwrap()) {
        let chunk_tokens = estimate_tokens(&result.chunk.content);

        if total_tokens + chunk_tokens <= budget_tokens {
            // Fits entirely — include as-is
            packed.push(result);
            total_tokens += chunk_tokens;
        } else {
            let remaining = budget_tokens - total_tokens;
            if remaining > MIN_USEFUL_TOKENS {
                // Partially fits — truncate to highest keyword-density subrange
                let truncated = truncate_to_budget(&result, remaining);
                packed.push(truncated);
                total_tokens += remaining;
            }
            break;  // budget exhausted
        }
    }

    packed
}

/// Token estimation: word count * 1.3 approximation (no tokenizer dependency)
fn estimate_tokens(text: &str) -> usize {
    (text.split_whitespace().count() as f32 * 1.3) as usize
}

/// Truncate a chunk to the highest keyword-density subrange
fn truncate_to_budget(result: &SearchResult, max_tokens: usize) -> SearchResult {
    // Find the contiguous line range within the chunk with highest BM25 token density
    // relative to the original query — keep that subrange
    // ...
}
```

### 24.3 Output Format

```
# src/billing/stripe.rs:120-148  [rust]  score=0.91
fn handle_charge_failure(err: &StripeError) {
    ...
}

# app/services/payments.rb:44-63  [ruby]  score=0.88
rescue Stripe::CardError => e
    ...

[context block: 2 chunks, ~1,840 tokens — fits within 4000 token budget]
```

---

## 25. Streaming Results (`--stream`)

For large repos or expensive queries (e.g. `--expand` with a deep call graph), stream results as they are found rather than waiting for the full result set.

### 25.1 Usage

```bash
smart-grep "payment handling" --stream
```

```
[streaming...]
1. src/billing/stripe.rs:120-148  score=0.91
2. app/services/payments.rb:44-63  score=0.88
3. src/billing/legacy.rs:201-234   score=0.82
[complete — 3 results in 340ms]
```

### 25.2 Why Streaming Matters

For **human developers**: streaming output feels instant even when the full search takes 500ms. The highest-confidence result appears within milliseconds of the query being issued.

For **agents**: enables pipelining — the agent can start processing the highest-confidence result while lower-ranked results are still being scored. Especially valuable for `--expand` queries where call-graph traversal is depth-first and naturally streaming.

In **MCP HTTP/SSE mode**: `--stream` maps directly to Server-Sent Events, giving agents true incremental delivery over the HTTP transport.

### 25.3 Implementation

Rust `mpsc` channels between the search pipeline and the output formatter. The `--expand` case is naturally streaming because call-graph BFS is depth-first. Near-zero implementation overhead.

---

## 26. Codebase Map (`smart-grep map`)

Clusters all chunk embeddings using k-means to produce a machine-readable JSON taxonomy of what the codebase is about.

### 26.1 Usage and Output

```bash
smart-grep map
smart-grep map --clusters 15  # override default 10
smart-grep map --json         # machine-readable output for agents
```

```json
{
  "clusters": [
    {
      "id": 0,
      "label": "payment processing",
      "size": 47,
      "centroid_symbol": "process_payment",
      "top_symbols": ["charge_card", "handle_failure", "retry_payment"],
      "files": ["src/billing/stripe.rs", "app/payments/processor.rb", "lib/billing/..."]
    },
    {
      "id": 1,
      "label": "authentication",
      "size": 23,
      "centroid_symbol": "verify_token",
      "top_symbols": ["decode_jwt", "check_scope", "require_auth"],
      "files": ["src/auth/middleware.rs", "app/auth/session.rb"]
    }
  ]
}
```

### 26.2 Algorithm (`crates/search/src/cluster.rs`)

```rust
pub fn build_codebase_map(db: &Db, k: usize) -> anyhow::Result<Vec<Cluster>> {
    // 1. Retrieve all embedding vectors
    let embeddings: Vec<(String, Vec<f32>)> = db.get_all_embeddings()?;

    // 2. K-means over the vector space
    let assignments = kmeans(&embeddings.iter().map(|(_, v)| v.as_slice()).collect::<Vec<_>>(), k)?;

    // 3. For each cluster: find centroid chunk + top recurring symbol tokens
    let mut clusters = vec![Vec::new(); k];
    for (i, (chunk_id, _)) in embeddings.iter().enumerate() {
        clusters[assignments[i]].push(chunk_id.clone());
    }

    clusters.into_iter().enumerate().map(|(id, chunk_ids)| {
        let centroid_chunk = find_centroid_chunk(db, &chunk_ids)?;
        let label = extract_cluster_label(db, &chunk_ids)?;
        Ok(Cluster { id: id as u32, label, centroid_symbol: centroid_chunk.symbol_name, chunk_ids })
    }).collect()
}
```

### 26.3 Dual Use

1. **Orientation tool**: new developers and agents get an instant overview of what a codebase does, without reading a single file.
2. **Search acceleration**: before embedding a query, find the nearest cluster and bias HNSW retrieval toward chunks in that cluster — reducing false positives from unrelated areas of large monorepos.

---

## 27. Semantic Diff (`smart-grep diff`)

Emits results not as byte diffs but as **semantic diffs** — detecting meaning changes, conceptual renames, and new behaviors.

### 27.1 Usage

```bash
smart-grep diff HEAD~1
smart-grep diff main..feature-branch
smart-grep diff pre-refactor..now  # using named snapshots
```

### 27.2 Output

```
SEMANTIC DIFF  HEAD~1 → HEAD

~ CHANGED MEANING
  src/billing/stripe.rs::handle_charge_failure  (similarity to prior: 0.71)
  src/auth/middleware.rs::verify_token          (similarity to prior: 0.68)

→ RENAMED (likely)
  src/payments/process.rb::charge_card  →  src/payments/process.rb::attempt_payment
  (similarity: 0.96 — same chunk, different symbol name)

+ NEW BEHAVIOR
  src/billing/idempotency.rs::check_idempotency_key
  src/billing/webhooks.rs::handle_stripe_event

= 47 chunks unchanged
```

### 27.3 Diff Categories

| Category | Detection condition |
|---|---|
| `CHANGED MEANING` | Chunk exists in both versions but `cosine(old_vector, new_vector) < 0.85`. |
| `RENAMED (likely)` | A chunk was deleted but a chunk with `cosine > 0.92` exists at a different path/name. |
| `NEW BEHAVIOR` | A new chunk with no old-version counterpart with `cosine > 0.80`. |
| `STABLE` | All chunks not in the above categories. Reported as count only. |

### 27.4 Implementation

The `prev_vector` column in the `embeddings` table stores the prior run's embedding. On re-index, the old embedding is copied to `prev_vector` before being overwritten with the new embedding. The `diff` command queries `prev_vector` vs the current vector using the thresholds above.

---

## 28. Index Snapshots + Time-Travel Search

### 28.1 Usage

```bash
# Save a named snapshot
smart-grep snapshot save pre-refactor
smart-grep snapshot save --git HEAD~10   # auto-name by git ref

# Query a historical snapshot
smart-grep "payment retry logic" --at pre-refactor
smart-grep "payment retry logic" --at HEAD~10

# Semantic diff between two named snapshots
smart-grep diff pre-refactor..now

# List available snapshots
smart-grep snapshot list
```

### 28.2 What Gets Snapshotted

Only the SQLite `.smart-grep/index.db` is snapshotted — not the HNSW `.usearch` file, which rebuilds on demand from the stored embeddings. For a 50k-line codebase, a snapshot is typically under 20MB.

Snapshots are stored under `.smart-grep/snapshots/<ref>/index.db`. Managed by `max_snapshots` config with LRU eviction when the limit is reached.

### 28.3 Time-Travel Query

```rust
// When --at is specified, open the snapshot DB in read-only mode
// and rebuild a temporary HNSW from its embeddings
pub fn query_snapshot(snapshot_ref: &str, query: &str, ...) -> anyhow::Result<Vec<SearchResult>> {
    let snapshot = db.find_snapshot(snapshot_ref)?;
    let snapshot_db = Db::open_readonly(&snapshot.db_path)?;
    let temp_hnsw = HnswIndex::build_from_db(&snapshot_db)?;
    // ... run normal search against snapshot DB + temp HNSW
}
```

The insight: git gives you time-travel for bytes. smart-grep gives you time-travel for *meaning* — answering questions like "what code handled payments before the refactor?" that no other tool can answer.

### 28.4 Auto-Snapshot via Git Hook

```bash
smart-grep hooks install
# Installs a post-commit hook: runs 'smart-grep snapshot save --git <HEAD>'
# after each commit — fully automated time-travel history
```

---

## 29. Stale Logic Detection

```bash
smart-grep stale src/billing/stripe.rs::handle_charge_failure
```

When a function's meaning changes (its embedding moves significantly on re-index), find other chunks that are highly similar to the **old embedding** but have not been updated. These are likely stale copies, parallel implementations, or tests that should have changed in sync.

### 29.1 Output

```
STALE CANDIDATES for handle_charge_failure
(changed in last index run — cosine to prior: 0.71)

  src/billing/legacy.rs::process_card_decline
    similarity-to-old: 0.89  last-modified: 47 days ago

  test/billing/stripe_test.rb::test_charge_failure
    similarity-to-old: 0.84  last-modified: 12 days ago  ← probably stale test

  src/payments/v2/charge.rs::handle_payment_error
    similarity-to-old: 0.81  last-modified: 3 days ago
```

### 29.2 Implementation

```sql
-- Find chunks similar to the old embedding of a changed chunk
SELECT c.chunk_key, c.symbol_name, f.path, f.mtime,
       cosine_sim(e.vector, :old_vector) AS similarity
FROM embeddings e
JOIN chunks c ON e.chunk_id = c.id
JOIN files f ON c.file_id = f.id
WHERE cosine_sim(e.vector, :old_vector) > 0.78
  AND c.chunk_key != :changed_chunk_key
ORDER BY similarity DESC
LIMIT 10;
```

This is a class of analysis impossible without semantic embeddings. `grep` finds textual duplication; smart-grep finds **conceptual staleness** — the function you should have updated when you changed this one.

---

## 30. Semantic Test Coverage Mapping

```bash
# Coverage for a specific function
smart-grep coverage src/billing/stripe.rs::charge_card

# Find all production functions with no semantic test coverage
smart-grep coverage --gaps
```

### 30.1 How It Works

Chunks are tagged `is_test = 1` based on file path heuristics. `smart-grep coverage <symbol>` finds all test chunks with cosine similarity above a threshold to the target production function — without requiring any framework instrumentation, import analysis, or test execution.

### 30.2 Output

```
SEMANTIC COVERAGE for charge_card

  test/billing/stripe_test.rs::test_charge_success       similarity: 0.91
  test/billing/stripe_test.rs::test_charge_failure       similarity: 0.87
  test/integration/checkout_test.rb::test_full_checkout  similarity: 0.72

COVERAGE GAPS (production functions with no semantic test similarity > 0.65)
  src/billing/idempotency.rs::check_idempotency_key  ← no semantic test coverage found
  src/billing/refunds.rs::process_partial_refund      ← no semantic test coverage found
```

### 30.3 Implementation (`crates/search/src/coverage.rs`)

```rust
// ~30 lines: HNSW nearest-neighbor search scoped to is_test = 1 chunks

pub fn semantic_coverage(
    target_chunk_id: &str,
    db: &Db,
    hnsw: &HnswIndex,
    threshold: f32,  // default 0.65
) -> anyhow::Result<Vec<TestCoverage>> {
    let target_vec = db.get_embedding(target_chunk_id)?;

    // Search over ALL chunks, then filter to test-only
    let candidates = hnsw.search(&target_vec, 50)?;

    candidates.into_iter()
        .filter(|(chunk_id, score)| {
            *score >= threshold && db.is_test_chunk(chunk_id).unwrap_or(false)
        })
        .map(|(chunk_id, score)| {
            let chunk = db.get_chunk(&chunk_id)?;
            Ok(TestCoverage { chunk, similarity: score })
        })
        .collect()
}
```

This is **semantic test coverage** — not line coverage (which requires running tests), but *conceptual coverage*: which tests are semantically *about* this function. It surfaces integration tests that cover a function even when they do not import it directly.

---

## 31. Semantic Naming Linter (`smart-grep lint`)

```bash
smart-grep lint
smart-grep lint --check   # exit code 1 if issues found — for CI integration
```

Three lint modes derived purely from the embedding index, requiring no new infrastructure:

### 31.1 Naming Inconsistency

Pairs of chunks with `cosine > 0.90` but very different symbol names — the same concept named two different ways in the codebase:

```
NAMING INCONSISTENCY
  src/billing/stripe.rs::handle_charge_failure    similarity: 0.96
  src/payments/legacy.rs::process_payment_error  ← near-identical behavior, different naming
  Suggestion: Consider standardizing to 'handle_charge_failure' (used in more places)

  src/auth/token.rs::verify_jwt_token            similarity: 0.91
  src/api/guards.rs::validate_bearer_token       ← same concept, different name
```

### 31.2 Duplicate Logic Candidates

Pairs with `cosine > 0.93` — likely copy-pasted or accidentally reimplemented:

```
DUPLICATE LOGIC
  src/auth/middleware.rs::verify_token      similarity: 0.94
  src/api/guards.rs::check_auth_header     ← likely duplicate — consider extracting shared function
```

### 31.3 Orphaned Code

Chunks with no semantic neighbors within similarity 0.60 — isolated islands that do not connect to any conceptual cluster. Often dead code or forgotten utilities:

```
ORPHANED CODE (no semantic neighbors)
  src/utils/xml_parser.rs::parse_saml_response  ← isolated; may be dead code
  src/legacy/compat.rs::serialize_v1_format     ← isolated; may be dead code
```

### 31.4 Implementation (`crates/search/src/lint.rs`)

```rust
pub fn run_lint(db: &Db, hnsw: &HnswIndex, config: &LintConfig) -> anyhow::Result<LintReport> {
    let all_chunks = db.get_all_chunks()?;
    let mut report = LintReport::default();

    for chunk in &all_chunks {
        let embedding = db.get_embedding(&chunk.chunk_id)?;

        // Find top-20 neighbors for this chunk
        let neighbors = hnsw.search(&embedding, 20)?;

        // Check naming inconsistency: high semantic similarity, low lexical similarity
        for (neighbor_id, sim) in &neighbors {
            if *sim > config.naming_similarity_threshold {
                let neighbor = db.get_chunk(neighbor_id)?;
                if lexical_distance(&chunk.symbol_name, &neighbor.symbol_name) > 0.5 {
                    report.naming_inconsistencies.push(NamingIssue { ... });
                }
            }
            if *sim > config.duplicate_similarity_threshold {
                report.duplicate_logic.push(DuplicateIssue { ... });
            }
        }

        // Check orphaned: no neighbor above orphan_threshold
        if neighbors.iter().all(|(_, sim)| *sim < config.orphan_similarity_threshold) {
            report.orphaned.push(chunk.clone());
        }
    }

    report
}
```

This is a form of static analysis no AST-based linter can perform — these problems are semantic, not syntactic. Add `smart-grep lint --check` to CI to catch naming drift over time.

---

## 32. Multi-Query Fan-Out (`smart-grep multi`)

```bash
smart-grep multi \
  "payment error handling" \
  "stripe webhook processing" \
  "billing retry logic"
```

Execute multiple related queries in parallel, merge and deduplicate results, re-ranking by maximum score across all queries with a **coverage boost** for chunks that appear in multiple result sets.

### 32.1 Scoring Formula

```
multi_score = max(scores_across_queries) + 0.05 * (num_queries_matched - 1)
```

Chunks that sit at the intersection of multiple related concerns score higher — these are the most conceptually central functions for the domain being investigated.

### 32.2 Output

```
1. src/billing/stripe.rs:120-148   multi_score=0.96  matched: [payment errors, stripe webhooks]
2. app/services/payments.rb:44-63  multi_score=0.88  matched: [payment errors, billing retry]
3. src/billing/retry.rs:15-48      multi_score=0.85  matched: [billing retry]
```

### 32.3 Implementation

```rust
// crates/search/src/rank.rs

pub fn multi_query_search(
    queries: &[String],
    store: &Store,
    embedder: &dyn Embedder,
    config: &SearchConfig,
) -> anyhow::Result<Vec<SearchResult>> {
    // Execute all queries in parallel via rayon
    let all_results: Vec<Vec<SearchResult>> = queries.par_iter()
        .map(|q| single_search(q, store, embedder, config))
        .collect::<anyhow::Result<Vec<_>>>()?;

    // Group results by chunk_id
    let mut by_chunk: HashMap<String, (SearchResult, Vec<String>, usize)> = HashMap::new();
    for (query_idx, results) in all_results.iter().enumerate() {
        for result in results {
            let entry = by_chunk.entry(result.chunk.chunk_id.clone())
                .or_insert_with(|| (result.clone(), vec![], 0));
            entry.0.score = entry.0.score.max(result.score);
            entry.1.push(queries[query_idx].clone());
            entry.2 += 1;
        }
    }

    // Apply coverage boost and re-rank
    let mut multi_results: Vec<SearchResult> = by_chunk.into_values()
        .map(|(mut result, matched_queries, match_count)| {
            result.score += 0.05 * (match_count - 1) as f32;
            result.matched_queries = matched_queries;
            result
        })
        .collect();

    multi_results.sort_by(|a, b| b.score.partial_cmp(&a.score).unwrap());
    Ok(multi_results)
}
```

~80 lines on top of existing search infrastructure. In MCP mode, agents call `search_code_multi(queries: string[])` to get a single deduplicated ranked list in one tool call, saving round-trips.

---

## 33. Agent Session Memory

```bash
smart-grep session start "debugging payment failures"
smart-grep "stripe error handling"        # auto-recorded to active session
smart-grep "retry queue processing"       # auto-recorded
smart-grep "webhook idempotency"          # auto-recorded
smart-grep session summary                # show conceptual territory explored
smart-grep session context --budget 8000  # consolidated context block from all session results
smart-grep session list
smart-grep session resume "debugging payment failures"
```

### 33.1 How Sessions Work

A session is a named container that auto-records every query and its top results. Sessions persist in the `sessions` and `session_results` tables — lightweight row references, no re-embedding, no additional compute beyond normal search.

When a session is active (started via `session start`), every subsequent search automatically appends its top-K results to the session.

### 33.2 `session summary`

```
SESSION: "debugging payment failures"  (4 queries, 14 unique chunks)

Explored:  payment error handling, stripe webhooks, retry logic
Not yet explored (nearby cluster areas): refund flows, subscription billing, idempotency
```

The "not yet explored" suggestions come from comparing session chunk embeddings against the codebase cluster map — if nearby clusters have not been touched, surface them as exploration suggestions.

### 33.3 `session context`

Assembles all high-ranking results from the session into a single token-budget-packed context block — deduplicated by `chunk_id`, sorted by max score across all queries. Run 5 searches, get one consolidated context in a single command.

For agents, this is the canonical deep-exploration workflow:
1. Run `search_code` multiple times for related concepts
2. Call `session_context(budget_tokens=8000)` to get the full accumulated picture in one shot

### 33.4 Implementation (`crates/search/src/session.rs`)

```rust
pub fn session_context(
    session_name: &str,
    budget_tokens: usize,
    db: &Db,
) -> anyhow::Result<Vec<SearchResult>> {
    // Retrieve all session results, deduplicated by chunk_id, sorted by max score
    let session_chunks = db.get_session_results_deduped(session_name)?;

    // Reuse budget.rs for token-budget packing
    let packed = pack_budget(session_chunks, budget_tokens);

    Ok(packed)
}
```

Sessions expire after a configurable TTL (default: 7 days) to prevent stale accumulation.

---

## 34. Project Auto-Detection

On first `smart-grep index`, the `project_detect.rs` module scans for known framework signatures and auto-generates `.smart-grep/config.toml` with optimized settings:

| Detected signature | Auto-configured behavior |
|---|---|
| `Gemfile` + `app/` | Rails: index `app/`, skip `vendor/`, `log/`, `tmp/`, `public/assets/` |
| `package.json` + `src/` | Node/React: index `src/`, skip `node_modules/`, `dist/`, `.next/` |
| `Cargo.toml` workspace | Rust workspace: index `crates/`, skip `target/` |
| `pyproject.toml` | Python: index project src, skip `.venv/`, `__pycache__/`, `dist/` |
| `go.mod` | Go: index all `.go` files except `vendor/` (for future Go support) |
| `mix.exs` | Elixir: index `lib/`, skip `_build/`, `deps/` (for future Elixir support) |

Also auto-generates a starter benchmark file with 5 example queries based on detected patterns (e.g. detects Stripe dependency → seeds query: `"where is the Stripe webhook handler?"`). This demonstrates value immediately on first run and teaches the natural query style.

---

## 35. MCP Server Mode

```bash
smart-grep serve           # starts MCP server on stdio (Claude Code default)
smart-grep serve --port 8080  # HTTP/SSE mode (Cursor, browser-based agents)
```

### 35.1 Complete MCP Tool Surface

```json
{
  "tools": [
    {
      "name": "search_code",
      "description": "Semantic search. Supports natural language query or --like code snippet.",
      "parameters": {
        "query": "string",
        "like": "string? (code snippet to find similar patterns)",
        "top_k": "int (default: 5)",
        "language": "string? (filter by language)",
        "budget_tokens": "int? (token-budget context assembly)",
        "from_symbol": "string? (restrict to call-graph-reachable subgraph)",
        "pattern": "string? (structural AST pattern filter)",
        "stream": "bool? (stream results as found)",
        "expand": "bool? (include callers + callees)"
      }
    },
    {
      "name": "search_code_multi",
      "description": "Fan-out over multiple related queries with dedup and coverage boost.",
      "parameters": {
        "queries": "string[] (2–10 related queries)",
        "top_k": "int?",
        "budget_tokens": "int?"
      }
    },
    {
      "name": "get_chunk",
      "description": "Retrieve a specific chunk by its chunk_id. Returns full content, metadata, and location.",
      "parameters": { "chunk_id": "string" }
    },
    {
      "name": "get_context",
      "description": "Retrieve a chunk plus its call graph neighbors (callers and callees).",
      "parameters": {
        "chunk_id": "string",
        "expand_callers": "bool (default: false)",
        "expand_callees": "bool (default: true)"
      }
    },
    {
      "name": "codebase_map",
      "description": "Semantic cluster taxonomy of the entire codebase. Use this to orient before searching.",
      "parameters": { "num_clusters": "int? (default: 10)" }
    },
    {
      "name": "find_stale",
      "description": "Find chunks semantically similar to the prior version of a recently-changed function.",
      "parameters": { "chunk_id": "string" }
    },
    {
      "name": "session_context",
      "description": "Deduplicated context block from all queries in the current session.",
      "parameters": {
        "session_name": "string",
        "budget_tokens": "int (default: 8000)"
      }
    },
    {
      "name": "coverage",
      "description": "Find test chunks semantically covering a production function, or find all gaps.",
      "parameters": {
        "chunk_id": "string?",
        "gaps": "bool? (find all uncovered production functions)"
      }
    }
  ]
}
```

### 35.2 Server Loop

```rust
// crates/core/src/main.rs — MCP server mode

pub fn serve_stdio(store: &mut Store, config: &Config) -> anyhow::Result<()> {
    let stdin = std::io::stdin();
    let stdout = std::io::stdout();

    loop {
        let mut line = String::new();
        stdin.lock().read_line(&mut line)?;
        if line.is_empty() { break; }

        let request: McpRequest = serde_json::from_str(&line)?;
        let response = handle_mcp_request(request, store, config)?;
        let response_json = serde_json::to_string(&response)?;

        writeln!(stdout.lock(), "{}", response_json)?;
    }

    Ok(())
}
```

---

## 36. Self-Index Mode (`--self`)

```bash
smart-grep "where is embedding batching implemented?" --self
smart-grep "how does the BM25 scorer work?" --self
smart-grep --pattern "catches error and retries" --self
```

smart-grep ships with a **pre-built index of its own source code** embedded in the binary. At release time, the build script runs `smart-grep index` on the repo and serializes the resulting `.smart-grep/` directory. Bundled via `include_bytes!()` and extracted to a temp dir when `--self` is invoked.

### 36.1 Dual Purpose

1. **Zero-setup demo**: every README example works instantly on any machine with no project to index first.
2. **Embedded documentation**: contributors can explore smart-grep's own source semantically instead of reading rustdoc. `smart-grep "how does chunking work?" --self` is more useful than searching docs.

### 36.2 Build Script

```rust
// build.rs
fn main() {
    // Run smart-grep index on the repo at build time
    // Serialize the resulting index to a compressed bytes file
    // Include via include_bytes!() for --self mode
    println!("cargo:rerun-if-changed=crates/");
}
```

Index size for a codebase of this scale: under 5MB compressed. Negligible binary overhead.

---

## 37. CLI Design

### 37.1 Complete Command Reference

```bash
# ─── Indexing ──────────────────────────────────────────────────────────────────
smart-grep index                          # index current directory
smart-grep index /path/to/repo            # index specific path
smart-grep index --rebuild                # force full re-index
smart-grep index --verbose                # per-file chunk counts and parse errors
smart-grep index --watch                  # watch for changes and auto-re-index (notify crate)

# ─── Natural Language Search ───────────────────────────────────────────────────
smart-grep "where is payment error handled?"
smart-grep "retry failed billing" --top 10
smart-grep "auth check before delete" --lang rust
smart-grep "database connection pool" --json
smart-grep "payment handler" --verbose    # full chunk metadata in results

# ─── Query-by-Example ──────────────────────────────────────────────────────────
smart-grep --like 'fn retry_with_backoff(...) { ... }'
cat src/billing/stripe.py | smart-grep --like -

# ─── Scoped and Budget-Aware Search ────────────────────────────────────────────
smart-grep "payment error handling" --budget 4000
smart-grep "handle charge failure" --expand
smart-grep "payment retry logic" --why
smart-grep "payment error" --from src/api/checkout.rs::handle_checkout
smart-grep "payment handling" --stream
smart-grep "payment handler" --confidence strict
smart-grep "auth logic" --from src/api/checkout.rs::handle_checkout --from-depth 5

# ─── Multi-Query Fan-Out ────────────────────────────────────────────────────────
smart-grep multi "payment errors" "stripe webhooks" "billing retry"

# ─── Structural Pattern Search ─────────────────────────────────────────────────
smart-grep --pattern "function that catches error and retries"
smart-grep --like 'rescue Stripe::CardError' --pattern "catches error and retries"

# ─── Codebase Intelligence ─────────────────────────────────────────────────────
smart-grep map
smart-grep map --clusters 15
smart-grep diff HEAD~1
smart-grep diff main..feature-branch
smart-grep stale src/billing/stripe.rs::handle_charge_failure
smart-grep coverage src/billing/stripe.rs::charge_card
smart-grep coverage --gaps
smart-grep lint
smart-grep lint --check                   # CI mode: exit 1 if issues found
smart-grep stats
smart-grep doctor

# ─── Sessions ──────────────────────────────────────────────────────────────────
smart-grep session start "debugging payments"
smart-grep session summary
smart-grep session context --budget 8000
smart-grep session list
smart-grep session resume "debugging payments"

# ─── Snapshots / Time-Travel ───────────────────────────────────────────────────
smart-grep snapshot save pre-refactor
smart-grep snapshot save --git HEAD~10
smart-grep snapshot list
smart-grep "payment retry logic" --at pre-refactor
smart-grep hooks install                  # install post-commit auto-snapshot hook

# ─── MCP Server ────────────────────────────────────────────────────────────────
smart-grep serve
smart-grep serve --port 8080

# ─── Self-Index Demo ────────────────────────────────────────────────────────────
smart-grep "how does chunking work?" --self
smart-grep --pattern "hybrid scoring" --self
```

### 37.2 Flag Reference

**Search flags:**

| Flag | Description | Default |
|---|---|---|
| `--top N` | Number of results to return | 5 |
| `--lang <language>` | Filter results by language | (all) |
| `--json` | Machine-readable JSON output | false |
| `--budget N` | Token-budget context block mode | (off) |
| `--expand` | Include callers and callees from call graph | false |
| `--from <file::symbol>` | Restrict to call-graph-reachable subset | (off) |
| `--from-depth N` | Max BFS depth for --from | 3 |
| `--why` | Show match explanation (embedding text, BM25 tokens, scores) | false |
| `--like <snippet or ->` | Query by code example instead of natural language | (off) |
| `--pattern <desc>` | Structural AST shape filter | (off) |
| `--stream` | Stream results as found | false |
| `--at <snapshot>` | Query a historical snapshot | (off) |
| `--confidence strict\|warn\|off` | Control low-confidence behavior | warn |
| `--verbose` | Full chunk metadata in results | false |
| `--path <dir>` | Override repo root | (cwd) |
| `--self` | Query smart-grep's own pre-built index | false |

**Index flags:**

| Flag | Description |
|---|---|
| `--rebuild` | Force full re-index, skip all incremental logic |
| `--verbose` | Per-file chunk counts and parse errors |
| `--watch` | Watch for file changes and auto-re-index |

### 37.3 Default Terminal Output

```
1. src/billing/stripe.rs:120-148  [rust]  score=0.91
   fn handle_charge_failure(err: &StripeError) { ...

2. app/services/payments.rb:44-63  [ruby]  score=0.88
   rescue Stripe::CardError => e

3. src/jobs/billing_job.rs:210-238  [rust]  score=0.84
   fn retry_failed_charge(job: &BillingJob) { ...
```

### 37.4 JSON Output (for agents)

```json
[
  {
    "rank": 1,
    "path": "src/billing/stripe.rs",
    "start_line": 120,
    "end_line": 148,
    "symbol": "handle_charge_failure",
    "kind": "function",
    "language": "rust",
    "score": 0.91,
    "semantic_score": 0.87,
    "bm25_boost": 0.04,
    "confidence": "high",
    "matched_queries": [],
    "preview": "fn handle_charge_failure(err: &StripeError) {"
  }
]
```

---

## 38. Configuration

Repo-local config at `.smart-grep/config.toml`. Auto-created with project-detected defaults on first `index`. All values are overridable via CLI flags.

```toml
# .smart-grep/config.toml

[embedding]
model = "fastembed-default"           # model identifier — change triggers --rebuild warning

[index]
include   = ["src", "app", "lib", "crates"]
exclude   = ["node_modules", "dist", "vendor", "target", ".git", "build", ".next", "__pycache__"]
min_lines = 5                         # skip chunks below this line count
max_lines = 80                        # split chunks above this line count

[search]
default_top_k         = 5
hybrid_alpha          = 0.7           # weight: 1.0 = pure semantic, 0.0 = pure BM25
query_expansion       = true          # enable synonym map expansion before embedding
confidence_mode       = "warn"        # warn | strict | off
confidence_threshold  = 0.0           # 0.0 = auto-calibrate; >0.0 = override

[map]
num_clusters = 10

[lint]
naming_similarity_threshold    = 0.90
duplicate_similarity_threshold = 0.93
orphan_similarity_threshold    = 0.60

[coverage]
test_similarity_threshold = 0.65

[snapshots]
auto_snapshot_on_index = false        # set true to snapshot before each re-index
max_snapshots          = 10           # LRU eviction when exceeded

[sessions]
ttl_days = 7                          # sessions expire after N days
```

---

## 39. Observability

A semantic tool becomes a black box without transparency. smart-grep exposes everything that matters:

### 39.1 `smart-grep stats`

```
smart-grep stats

Files indexed:      1,247
Total chunks:       8,432
  Production:       7,891
  Test:               541
Languages:          Rust (4,201), TypeScript (2,100), Ruby (1,831), JavaScript (300)
Model:              Xenova/all-MiniLM-L6-v2 (384 dimensions)
Index DB size:      18.4 MB
Last indexed:       2 minutes ago
Call edges:         12,847
Active sessions:    2
Snapshots:          3 (oldest: pre-refactor, 7 days ago)
Synonyms:           847 concept→symbol mappings
```

### 39.2 `smart-grep doctor`

```
smart-grep doctor

✓ tree-sitter grammars: Rust OK, TypeScript OK, JavaScript OK, Python OK, Ruby OK
✓ fastembed model: Xenova/all-MiniLM-L6-v2 present at ~/.cache/fastembed/...
✓ SQLite index: readable, schema version 4 matches binary
✓ HNSW index: 8,432 vectors, dimension 384 matches meta.vector_dim
✓ Session store: 2 active sessions, no corrupt entries
✓ Synonym map: 847 entries, last built 2 minutes ago

No issues found. smart-grep is healthy.
```

### 39.3 `--why` on Search

```
smart-grep "payment retry logic" --why

1. src/billing/retry.rs:45-67  [rust]  score=0.91
   fn retry_charge(attempts: u32) { ...
   ─── Why ───────────────────────────────────────
   Embedding text:    Language: Rust\nSymbol: retry_charge\nKind: function\n...
   BM25 tokens:       retry (tf=3, idf=2.1), charge (tf=2, idf=1.8)
   Semantic score:    0.87
   BM25 boost:        0.04
   Final score:       0.91  (alpha=0.7)
   Cluster:           #2 "payment processing" (1.2% cluster bias)
   Confidence:        high (threshold: 0.60, 47 historical queries)
```

---

## 40. Error Handling

Use `thiserror` for typed errors per crate, `anyhow` for propagation in `crates/core`.

| Failure | Mitigation |
|---|---|
| Unsupported language | Warn and skip — never crash. Log at `WARN` level with filename. |
| Parse error | Mark `files.parse_error = 1`, continue indexing other files. |
| Corrupted SQLite | Surface clear message: "index.db may be corrupted — run `smart-grep index --rebuild`". |
| Embedding dimension mismatch | Detected via `meta.vector_dim` on startup → require `--rebuild` with clear message. |
| Stale/missing HNSW index | Rebuild automatically from existing SQLite embeddings (no data loss). |
| Unreadable files | Warn and skip. Log filename. |
| Model swapped between runs | `meta.model_name` mismatch → require `--rebuild` with clear message. |
| Low-confidence results | Flag with warning + actionable suggestions (see Confidence Design). |
| Snapshot not found | Clear error: list available snapshots via `smart-grep snapshot list`. |
| Session not found | Clear error: list available sessions via `smart-grep session list`. |
| Reachability entry not indexed | Warn: "symbol not in index, falling back to full search". |
| `--from` BFS too deep | `--from-depth` limit (default 3 hops) prevents BFS explosion. |
| fastembed model download fails | Offline fallback: clear message with model cache path, instructions to download manually. |
| HNSW rebuild OOM | Graceful fallback: surface memory estimate, suggest `--rebuild --sequential` (single-threaded). |

---

## 41. Per-Language Implementation Details

### 41.1 Rust (`crates/indexer/src/languages/rust.rs`)

**Tree-sitter node types to extract:**

| Node type | Chunk kind |
|---|---|
| `function_item` | `function` (top-level and impl methods) |
| `impl_item` | `impl` (for trait impls without extracted methods) |
| `struct_item` | `struct` |
| `enum_item` | `enum` |
| `trait_item` | `trait` |
| `type_alias` | `type_alias` |
| `match_expression` with error type arms | `rescue` (error handling) |

**Rust-specific rules:**
- Extract `parent_symbol` from the enclosing `impl_item` or `impl_block`.
- Detect `is_test` from `#[test]` or `#[cfg(test)]` attributes OR from path containing `/tests/` or `_test.rs`.
- For `impl` blocks: extract each method as a separate chunk. Also emit one chunk for the `impl` header if it implements a trait (e.g. `impl PaymentProcessor for StripeClient`).
- Skip `use` declarations, `mod` re-exports, `pub use` items (no semantic content).
- `async fn` items: tag `symbol_kind = "async_function"` for future async-aware queries.

### 41.2 TypeScript / JavaScript

**Tree-sitter node types to extract:**

| Node type | Chunk kind |
|---|---|
| `function_declaration` | `function` |
| `arrow_function` (if assigned via `const x =`) | `function` (name from assignment) |
| `method_definition` inside `class_body` | `method` |
| `class_declaration` | `class` |
| `export_statement` wrapping any of the above | (inherits wrapped kind) |

**TS/JS-specific rules:**
- Arrow functions assigned to `const` variables: chunk them if variable name exists and body exceeds `min_lines`.
- React components (JSX-returning functions): tag `symbol_kind = "component"`.
- API route handlers (Express/Fastify): detect by parent call expression (`app.get`, `router.post`) and tag `symbol_kind = "route"`.
- Detect `is_test` from `*.test.ts`, `*.spec.ts`, `__tests__/` path patterns, and `describe`/`it` call sites.
- TypeScript interfaces (`interface_declaration`): tag `symbol_kind = "interface"`.
- Type aliases with union/intersection types: chunk if they carry semantic identity.

### 41.3 Python (`crates/indexer/src/languages/python.rs`)

**Tree-sitter node types:**

| Node type | Chunk kind |
|---|---|
| `function_definition` | `function` |
| `async_function_definition` | `async_function` |
| `class_definition` | `class` |
| `decorated_definition` | (inherits inner kind, preserves decorators) |

**Python-specific rules:**
- Extract `parent_symbol` from the enclosing `class_definition`.
- Detect `is_test` from `test_*.py`, `*_test.py`, `tests/` path patterns.
- **Preserve type annotations** in `embedding_text` (they add semantic signal: `def charge(card: Card, amount: Decimal) -> Receipt`).
- **Docstrings**: include in `embedding_text` but NOT in `content` — they add semantic signal without cluttering the display. Strip triple-quoted strings from content.
- `@property`, `@classmethod`, `@staticmethod`: tag in `symbol_kind` accordingly.
- Django/Flask/FastAPI views: detect by decorator (`@app.route`, `@api_view`, `class MyView(APIView)`) and tag `symbol_kind = "view"`.

### 41.4 Ruby (`crates/indexer/src/languages/ruby.rs`)

**Tree-sitter node types:**

| Node type | Chunk kind |
|---|---|
| `method` | `method` |
| `singleton_method` | `class_method` |
| `class` | `class` |
| `module` | `module` |
| `rescue` block inside a method | `rescue` |

**Ruby-specific rules:**
- Extract `parent_symbol` from the enclosing `class` or `module` node.
- Detect `is_test` from `*_spec.rb`, `spec/` path patterns, and RSpec `describe`/`it` blocks.
- **Rails controllers**: detect inheritance from `ApplicationController`. Tag `index`, `show`, `create`, `update`, `destroy` actions as `symbol_kind = "action"`.
- **`rescue` blocks**: emit as separate chunks if they exceed `min_lines`, tagged `symbol_kind = "rescue"`. This is critical for "where is X error handled?" queries on Rails codebases.
- **Modules**: both as a whole-module chunk AND per-method chunks.
- **`attr_accessor`/`attr_reader`**: skip as trivial one-liners (below `min_lines`).

---

## 42. Testing Strategy

### 42.1 Unit Tests (per crate)

Each crate has its own unit tests in `src/tests.rs` or per-module `#[cfg(test)]` blocks.

**`crates/indexer`:**
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_rust_extractor_extracts_functions() {
        let source = r#"
            fn handle_payment_error(err: &PaymentError) -> Response {
                match err { _ => Response::error() }
            }
        "#;
        let extractor = RustExtractor::new();
        let chunks = extractor.extract_chunks(Path::new("test.rs"), source);
        assert_eq!(chunks.len(), 1);
        assert_eq!(chunks[0].symbol_name, Some("handle_payment_error".to_string()));
        assert_eq!(chunks[0].symbol_kind, Some("function".to_string()));
    }

    #[test]
    fn test_embedding_text_format() {
        let chunk = CodeChunk { symbol_name: Some("retry_charge".to_string()), .. };
        let text = build_embedding_text(&chunk);
        assert!(text.starts_with("Language:"));
        assert!(text.contains("Symbol: retry_charge"));
        assert!(text.contains("Code:"));
    }

    #[test]
    fn test_camelcase_tokenization() {
        assert_eq!(tokenize("handlePaymentError"), vec!["handle", "payment", "error"]);
        assert_eq!(tokenize("retry_with_backoff"), vec!["retry", "with", "backoff"]);
    }
}
```

**`crates/search`:**
```rust
#[test]
fn test_hybrid_score_semantic_dominant() {
    let score = hybrid_score(0.90, 0.50, 0.7);
    assert!((score - 0.78).abs() < 0.01);  // 0.7*0.90 + 0.3*0.50 = 0.78
}

#[test]
fn test_budget_packing_respects_limit() {
    let results = make_test_results(10);  // 10 results, ~500 tokens each
    let packed = pack_budget(results, 1500);
    let total_tokens: usize = packed.iter().map(|r| estimate_tokens(&r.chunk.content)).sum();
    assert!(total_tokens <= 1500);
}
```

### 42.2 Integration Tests (`tests/`)

```rust
// tests/search_test.rs
// Full end-to-end: index test fixtures → search → verify results

#[test]
fn test_semantic_search_finds_payment_code_without_keyword() {
    let temp_dir = TempDir::new().unwrap();
    let repo = create_test_repo(temp_dir.path(), &[
        ("billing.js", BILLING_FIXTURE),  // contains Stripe::CardError, NOT "payment"
        ("auth.js",    AUTH_FIXTURE),
        ("retry.js",   RETRY_FIXTURE),
    ]);

    // Index the test fixtures
    smart_grep_index(&repo).expect("indexing should succeed");

    // Search for "payment error handling" — should find billing.js despite no "payment" keyword
    let results = smart_grep_search(&repo, "payment error handling", 5).unwrap();
    assert!(!results.is_empty());
    assert!(results[0].chunk.file_path.ends_with("billing.js"));
    assert!(results[0].score > 0.70);

    // Ripgrep comparison: should return zero results for the same query
    // (validates the core value proposition)
    let rg_results = ripgrep_search(&repo, "payment error handling");
    assert!(rg_results.is_empty());
}

#[test]
fn test_query_by_example_finds_cross_language_analogues() {
    // ...
}

#[test]
fn test_incremental_reindex_after_file_change() {
    // ...
}
```

### 42.3 Test Fixtures

The `test-fixtures/` directory contains the canonical validation cases. These files are carefully crafted to test the core value proposition: finding code by meaning without exact keyword matches.

```javascript
// test-fixtures/billing.js — payment code WITHOUT the word "payment"
// Uses: Stripe::CardError, CardDeclined, ChargeFailure, InsufficientFunds
// A semantic search for "payment error" should find this file

// test-fixtures/auth.js — authentication code WITHOUT "authentication"
// Uses: token, jwt, verify, scope, bearer, credentials
// A semantic search for "authentication" should find this file

// test-fixtures/retry.js — retry logic WITHOUT the word "retry"
// Uses: attempt, backoff, exponential, reattempt, sleep
// A semantic search for "retry logic" should find this file
```

---

## 43. Benchmarking Strategy

### 43.1 `benches/index_bench.rs` (criterion)

```rust
use criterion::{criterion_group, criterion_main, Criterion};

fn bench_full_index(c: &mut Criterion) {
    let repo = prepare_test_repo();  // ~10k lines of mixed Rust + TS

    c.bench_function("full_index_10k_lines", |b| {
        b.iter(|| {
            let temp = TempDir::new().unwrap();
            smart_grep_index_to(&repo, &temp).unwrap();
        })
    });
}

fn bench_single_query(c: &mut Criterion) {
    let store = prepare_indexed_store();

    c.bench_function("query_latency_p50", |b| {
        b.iter(|| {
            smart_grep_search(&store, "payment error handling", 5).unwrap()
        })
    });
}

fn bench_incremental_reindex(c: &mut Criterion) {
    let (repo, store) = prepare_indexed_repo();

    c.bench_function("incremental_1_file_change", |b| {
        b.iter(|| {
            modify_one_file(&repo);
            smart_grep_reindex(&store).unwrap();
        })
    });
}

criterion_group!(benches, bench_full_index, bench_single_query, bench_incremental_reindex);
criterion_main!(benches);
```

### 43.2 Performance Targets

| Operation | Target | Acceptable | Fail |
|---|---|---|---|
| Full index, 50k-line repo | < 60s | < 120s | > 300s |
| Query latency (p50) | < 50ms | < 100ms | > 500ms |
| Query latency (p99) | < 200ms | < 500ms | > 2000ms |
| Incremental re-index (1 changed file in 10k-file repo) | < 3s | < 5s | > 30s |
| HNSW rebuild from 10k chunks | < 2s | < 5s | > 30s |
| MCP round-trip (query + response) | < 100ms | < 200ms | > 1000ms |

---

## 44. Evaluation Plan

Quality must be measured, not assumed. Build 25–50 natural-language queries paired with expected code locations, plus 10 query-by-example cases and 5 structural pattern cases.

### 44.1 Benchmark Query Set

| Query type | Example | Expected matches |
|---|---|---|
| Natural language | "where is payment error handled?" | Stripe exceptions, decline handlers, billing retry |
| Natural language | "where do we verify admin permissions?" | Guard clauses, middleware, role checks |
| Natural language | "where are failed jobs retried?" | Retry loops, backoff logic, queue retry config |
| Natural language | "find the database connection pool" | Connection pool initialization, pool config |
| Natural language | "where is the checkout flow implemented?" | Checkout controller, order creation, cart finalization |
| Query-by-example | paste a retry function | All retry pattern implementations across languages |
| Structural | "catches error and retries" | All try/catch+retry patterns regardless of domain |
| Scoped | "auth logic" `--from api/checkout` | Only auth reachable from checkout flow |
| Multi-query | "payment errors" + "billing retry" | Central billing resilience functions |
| Cross-language | paste Ruby rescue block | Analogous Rust/Python error handlers |

### 44.2 Metrics

| Metric | Description | Target |
|---|---|---|
| Top-1 accuracy | Correct result in position 1 | > 80% |
| Top-5 recall | Correct result in top 5 | > 95% |
| Hybrid vs. semantic-only | BM25 improvement on exact symbol queries | > 10% improvement |
| Query-by-example precision | Cross-language pattern matching accuracy | > 75% |
| Structural pattern recall | Finds all instances of a given pattern | > 90% |
| Reachability precision | `--from` scope contains only reachable results | 100% |
| Confidence calibration | False-positive rate at default threshold | < 5% |
| Coverage recall | `smart-grep coverage` finds known test relationships | > 85% |
| Lint precision | Naming inconsistencies flagged are genuine | > 80% (< 20% false positive rate) |
| Index time | Wall time for initial full index on ~50k-line repo | < 90s |
| Search latency | p50 for single query | < 100ms |
| Incremental re-index | After changing 1 file in 10k-file repo | < 5s |

### 44.3 Baseline Comparison

Compare against `ripgrep` on all benchmark queries. **MVP success target**: meaningfully higher recall on intent-based queries (where ripgrep returns zero relevant results) with sub-100ms search latency.

The key success criterion: smart-grep must return relevant results that ripgrep returns zero for, on the test fixture queries designed specifically for this purpose.

---

## 45. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Bad chunking hurts retrieval quality | High | High | AST-aware chunks with size bounds; iterate against benchmark set before v1 release |
| Local embedding model too slow on older hardware | Medium | High | Batch + rayon parallelism; smaller default model; document hardware requirements |
| Incremental HNSW updates are complex | Low | Medium | Rebuild in v1 — correct and fast enough. Online updates in v1.1 as explicit scope |
| Cross-language support becomes fragile | Medium | Medium | `ChunkExtractor` trait + per-language integration tests with canonical test fixtures |
| Users do not trust semantic matches | Medium | High | `--why` explains every match; confidence modes; calibrated thresholds; `--verbose` metadata |
| Model vectors change after upgrade | High | High | `meta.model_name` mismatch on startup → auto-trigger `--rebuild` with clear message |
| Call graph extraction is too noisy | Medium | Low | Graph used only for `--expand`/`--from`/`stale`; does NOT affect core search quality |
| k-means cluster count wrong | Low | Low | Default 10 is reasonable for most codebases; user-tunable via config and CLI |
| Self-index binary size unacceptable | Low | Low | Index < 5MB gzip-compressed; measure during development |
| BM25 degrades semantic precision | Low | Medium | `hybrid_alpha = 0.7` keeps semantic dominant; measurable via benchmark set |
| Structural fingerprints imprecise | Medium | Low | Used as filter/boost, not primary ranking; graceful degradation if fingerprint is wrong |
| Snapshot storage grows unbounded | Low | Medium | `max_snapshots` config with LRU eviction enforced on every `snapshot save` call |
| Confidence calibration wrong for new repos | Medium | Medium | Falls back to hardcoded threshold (0.60) until 20+ queries recorded |
| Session memory grows stale | Low | Low | Sessions expire after configurable TTL (default 7 days) |
| Lint false positives erode trust | Medium | High | All lint output shows similarity scores; `--check` is opt-in only; default output is informational |
| `--from` BFS explosion on large graphs | Low | Medium | `--from-depth` limit (default 3 hops) + configurable via CLI |
| fastembed model download fails offline | Low | Medium | Detect early; cache at `~/.cache/fastembed`; provide offline-install instructions in docs |

---

## 46. Implementation Phases

### Phase 1 — POC: Node.js (1 day)

**Goal**: Validate the concept before writing any Rust. Prove the core value proposition with minimal code.

- [ ] `tree-sitter` parse JS/TS into AST chunks (functions and classes, not line windows)
- [ ] `fastembed` local embedding via `@xenova/transformers` (Xenova/all-MiniLM-L6-v2)
- [ ] Store vectors in SQLite
- [ ] Cosine similarity search in memory
- [ ] CLI: `node poc/index.js index .` and `node poc/index.js search "<query>"`
- [ ] Benchmark against ripgrep on `test-fixtures/` files for 10 conceptual queries
- [ ] Validate normalized `embedding_text` format improves recall vs. raw source embedding
- [ ] Validate query-by-example: paste a code snippet, find similar chunks across test fixtures
- [ ] Document results in `POC.md`

**Success criteria**: smart-grep returns relevant results that ripgrep returns zero for; query-by-example finds cross-language analogues in test fixtures.

---

### Phase 2 — Rust Foundations + Searchable MVP (3–4 days)

**Goal**: reliable end-to-end vertical slice — index one language, search it.

**Workspace setup:**
- [ ] Set up workspace with 4 crates + shared types in `crates/indexer/src/lib.rs`
- [ ] Define `ChunkExtractor` and `Embedder` traits with full documentation
- [ ] Set up `thiserror` error types per crate and `anyhow` propagation in core

**`crates/indexer`:**
- [ ] `tree-sitter` integration for Rust + TypeScript/JavaScript
- [ ] AST chunk extraction: functions, classes, impls; size-bounded with `min_lines`/`max_lines`; `is_test` tagging
- [ ] `embedding_text.rs` — normalized Language/Symbol/Kind/Parent/Path/Code formatter
- [ ] `structural.rs` — AST shape fingerprinting (basic implementation)
- [ ] `fastembed-rs` integration — local ONNX embedding, batched with `rayon`
- [ ] `synonym_map.rs` — concept→symbol co-occurrence map builder and query-time expander

**`crates/store`:**
- [ ] `rusqlite` — full schema as per `schema.sql`, including `call_edges`, `synonyms`, `sessions`, `snapshots`, `query_scores`, `meta`
- [ ] `usearch` HNSW — vector storage and ANN search
- [ ] Incremental file state detection (hash comparison)

**`crates/search`:**
- [ ] HNSW top-K retrieval
- [ ] `bm25.rs` — pure-Rust BM25 scorer + camelCase/snake_case tokenizer
- [ ] Hybrid re-ranking: `rank.rs`
- [ ] Query expansion via synonym map
- [ ] `--like` mode: embed code snippet as query
- [ ] `--pattern` mode: structural fingerprint fuzzy matching
- [ ] `confidence.rs` — score calibration + low-confidence detection
- [ ] Top-N results with path, line range, score, preview

**`crates/core`:**
- [ ] `smart-grep index .` command
- [ ] `smart-grep "<query>"` command with hybrid + confidence
- [ ] `smart-grep stats` command
- [ ] `--top N`, `--lang`, `--json`, `--verbose`, `--why`, `--like`, `--pattern`, `--confidence`, `--stream` flags

**Success criteria**: index a small repo and get relevant results; `--like` finds cross-language pattern matches; `--confidence warn` triggers on out-of-domain queries.

---

### Phase 3 — Incremental Indexing + Graph + Sessions (1–2 days)

- [ ] `blake3` content hashing for incremental skip detection
- [ ] Incremental file state transitions: new/changed/deleted/unchanged
- [ ] Snapshot `prev_vector` before each re-index of changed chunks
- [ ] Rebuild HNSW after mutations; `--rebuild` flag
- [ ] `graph.rs` — call edge extraction from AST (Rust + TS/JS first)
- [ ] `reachability.rs` — BFS over `call_edges` for `--from <entry>` scoping
- [ ] `expand.rs` — `--expand` flag: callers + callees in results
- [ ] `smart-grep stale <symbol>` command
- [ ] `session.rs` — `start`/`resume`/`summary`/`context`/`list` commands
- [ ] `smart-grep multi` — parallel fan-out with coverage boosting
- [ ] `--stream` — streaming result output via mpsc channels

**Success criteria**: re-index after 1 changed file < 5 seconds; `--from` correctly scopes results to reachable subgraph; session context assembles coherent multi-query context.

---

### Phase 4 — Multi-language + Codebase Intelligence (2–3 days)

- [ ] Python and Ruby `ChunkExtractor` implementations with full test coverage
- [ ] Per-language integration tests with `test-fixtures/`
- [ ] `smart-grep map` — k-means clustering + JSON taxonomy; cluster-biased search
- [ ] `smart-grep diff <ref>` — semantic diff via `prev_vector`
- [ ] `snapshot.rs` — save/restore/list index snapshots; `--at <snapshot>` time-travel search
- [ ] `smart-grep hooks install` — post-commit auto-snapshot
- [ ] `coverage.rs` — `smart-grep coverage` and `--gaps` mode
- [ ] `lint.rs` — naming inconsistency + duplicate logic + orphan detection; `--check` CI mode
- [ ] `project_detect.rs` — auto-detect framework + generate config + seed benchmark queries
- [ ] `smart-grep doctor` — comprehensive health check
- [ ] `--budget N` — token-budget context block assembly
- [ ] Build 25–50 query benchmark set; document in `docs/benchmark.md`
- [ ] `--watch` mode via `notify` crate

**Success criteria**: lint finds real naming inconsistencies in a real-world repo without false-positive flood; coverage gaps mode surfaces genuinely untested functions; `--at` time-travel returns correct results from a historical snapshot.

---

### Phase 5 — MCP Server + Self-Index + Polish (1–2 days)

- [ ] `smart-grep serve` — MCP server (stdio + HTTP/SSE) with full 8-tool surface
- [ ] Wire all MCP tools to existing handlers (search, multi, context, map, stale, session, coverage)
- [ ] Build script: index own source, serialize, `include_bytes!()` for `--self` mode
- [ ] Full `--json` schema including `confidence`, `matched_queries`, `semantic_score`, `bm25_boost`
- [ ] `CLAUDE.md` and `AGENTS.md` snippet in README
- [ ] Validate `meta.vector_dim` + `meta.model_name` on startup with clear error messages
- [ ] `cargo install smart-grep` + GitHub Releases binary distribution
- [ ] Complete `docs/benchmark.md` with full benchmark results vs ripgrep
- [ ] Performance benchmarks via criterion; verify all targets met
- [ ] Integration test suite passing on all 5 languages

**Success criteria**: MCP server mountable in Claude Code with zero additional config; `smart-grep "how does chunking work?" --self` resolves correctly on a fresh install; `smart-grep lint --check` returns exit code 1 on a repo with known naming inconsistencies.

---

## 47. Integration with Claude Code

Add to any project's `CLAUDE.md` or `AGENTS.md`:

````markdown
## Search
Use `smart-grep "<query>"` INSTEAD of grep or ripgrep for any conceptual search.
Use grep or ripgrep ONLY when searching for an exact symbol name or string literal.

## Query-by-example
Use `smart-grep --like '<code snippet>'` to find similar code patterns across languages.
Pipe a file directly: `cat src/billing/stripe.py | smart-grep --like -`

## Scoped search
Use `smart-grep "<query>" --from <file::symbol>` to restrict search to code reachable
from a specific entry point — essential in large monorepos to avoid results from
unrelated services.

## Context assembly
Use `smart-grep "<query>" --budget 6000` for a token-budget-aware context block.
Use `smart-grep multi "<q1>" "<q2>" "<q3>"` to find code central to multiple concepts.
Use `smart-grep session context --budget 8000` after multiple queries for one consolidated context.

## Code quality
Use `smart-grep coverage --gaps` to find production functions with no semantic test coverage.
Use `smart-grep lint` to find naming inconsistencies and duplicate logic.
Use `smart-grep map` to understand what a codebase contains before deep exploration.

## Examples
```
smart-grep "where is payment error handled?"
smart-grep "retry failed jobs" --json
smart-grep "auth middleware" --expand --budget 4000
smart-grep --like 'rescue Stripe::CardError => e' --pattern "catches error and retries"
smart-grep "payment logic" --from src/api/checkout.rs::handle_checkout
smart-grep multi "payment errors" "stripe webhooks" "billing retry"
smart-grep coverage --gaps
smart-grep map
```
````

### MCP config (Claude Code / Cursor)

```json
{
  "mcpServers": {
    "smart-grep": {
      "command": "smart-grep",
      "args": ["serve"]
    }
  }
}
```

---

## 48. Timeline

| Phase | Duration | Goal |
|---|---|---|
| 1 — POC | 1 day | Validate concept in Node.js before writing Rust |
| 2 — Rust foundations + searchable MVP | 3–4 days | Index one language, search it end-to-end |
| 3 — Incremental indexing + graph + sessions | 1–2 days | Incremental re-index, call graph, session memory |
| 4 — Multi-language + codebase intelligence | 2–3 days | All 5 languages, lint/coverage/diff/map |
| 5 — MCP server + self-index + polish | 1–2 days | Agent integration, binary distribution |
| **Total** | **~9–12 days** | **Production-ready v1** |

---

## 49. Feature Map

Quick reference of all major capabilities:

| Category | Features |
|---|---|
| **Search modes** | Natural language, `--like` query-by-example, `--pattern` structural, `multi` fan-out |
| **Search scoping** | `--lang`, `--from` reachability, `--at` time-travel snapshot |
| **Result quality** | Hybrid BM25+HNSW, query expansion via synonym map, cluster-biased retrieval |
| **Agent UX** | `--budget` context block, `--stream` streaming, `--json` structured output, `--expand` graph |
| **Transparency** | `--why` match explanation, `--confidence` calibration, `--verbose` metadata |
| **Codebase intelligence** | `map` clusters, `diff` semantic diff, `stale` staleness, `lint` quality, `coverage` test gaps |
| **Session memory** | `session start/summary/context/list/resume` |
| **Time travel** | `snapshot save/list`, `--at <ref>` historical queries, `hooks install` auto-snapshot |
| **Integration** | MCP server (stdio + SSE), `CLAUDE.md` snippet, `--self` embedded demo |
| **Reliability** | Incremental indexing, `--rebuild`, confidence modes, `doctor` health check, `stats` |

---

## 50. Future Extensions

Capabilities intentionally deferred beyond v1, roughly in priority order:

1. **Code-specific embedding model swap** (`nomic-embed-code` or `jina-embeddings-v2-base-code`) — better recall on exact API names and code constructs. One-file change thanks to the `Embedder` trait.
2. **Online HNSW updates** — skip rebuild on incremental re-index. `usearch` supports per-vector add/remove; reduces post-mutation latency from ~2s to ~50ms.
3. **Go and Elixir language support** — add `ChunkExtractor` impls. The trait system makes this a new file with no changes elsewhere.
4. **Symbol relationship graph visualization** — export call graph to DOT/Mermaid format for visual exploration of codebase structure.
5. **File-level and repo-level natural-language summaries** — generate a one-paragraph summary of each file and the whole repo using the cluster labels + centroid symbols. Agent context without reading any source.
6. **Editor integrations** — Neovim LSP shim (virtual hover showing semantic neighbors), VS Code extension with sidebar search.
7. **Cross-repo search** — federated index across multiple local repos. `--repos path1 path2 path3` flag.
8. **Query history with replay and diff** — replay old queries against new index versions; diff the results to see what changed semantically.
9. **Team-shared indexes** — export/import `.smart-grep/` as a portable artifact. `smart-grep export --output index.tar.gz`.
10. **Auto-snapshot on git commit** via hook installer — `smart-grep hooks install` already planned; auto-snapshot makes this fully seamless.
11. **CI benchmark regression detection** — fail CI if Top-1 accuracy drops below a stored baseline. Prevents inadvertent quality regressions during refactors.
12. **Watch mode with partial HNSW updates** — file-event-driven re-index of changed files only, with online HNSW vector replacement instead of full rebuild.
13. **Embedding model fine-tuning** — generate training pairs from the synonym map and `query_scores` history; fine-tune the local model on the specific codebase's naming conventions.
14. **Cluster-based documentation generation** — for each cluster, emit a structured summary: "This cluster contains 47 chunks handling payment processing. Key functions: process_payment, handle_charge_failure, retry_charge. Key files: src/billing/..."

---

*End of Plan — smart-grep v1 Implementation Plan*

---

## 51. Detailed Module Implementations

### 51.1 `crates/store/src/db.rs` — Full CRUD Interface

```rust
// crates/store/src/db.rs

use rusqlite::{Connection, params, OptionalExtension};
use anyhow::{Context, Result};
use std::path::Path;

pub struct Db {
    conn: Connection,
}

impl Db {
    /// Open or create the index database. Runs schema migrations automatically.
    pub fn open(index_dir: &Path) -> Result<Self> {
        let db_path = index_dir.join("index.db");
        let conn = Connection::open(&db_path)
            .with_context(|| format!("Failed to open index DB at {}", db_path.display()))?;

        // Enable WAL mode for concurrent reads + single writer
        conn.execute_batch("PRAGMA journal_mode=WAL; PRAGMA synchronous=NORMAL;")?;

        let db = Self { conn };
        db.migrate()?;
        Ok(db)
    }

    /// Run schema migration if needed
    fn migrate(&self) -> Result<()> {
        let schema = include_str!("schema.sql");
        self.conn.execute_batch(schema)?;

        // Set schema version if not present
        self.conn.execute(
            "INSERT OR IGNORE INTO meta(key, value) VALUES ('schema_version', '1')",
            [],
        )?;
        Ok(())
    }

    // ─── Files ───────────────────────────────────────────────────────────────

    pub fn upsert_file(&self, path: &str, language: Option<&str>, mtime: i64,
                       size_bytes: i64, content_hash: &str, parse_error: bool) -> Result<i64> {
        self.conn.execute(
            "INSERT OR REPLACE INTO files(path, language, mtime, size_bytes, content_hash, parse_error)
             VALUES (?1, ?2, ?3, ?4, ?5, ?6)",
            params![path, language, mtime, size_bytes, content_hash, parse_error as i64],
        )?;
        Ok(self.conn.last_insert_rowid())
    }

    pub fn get_file_hash(&self, path: &str) -> Result<Option<String>> {
        self.conn.query_row(
            "SELECT content_hash FROM files WHERE path = ?1",
            params![path],
            |row| row.get(0),
        ).optional().map_err(Into::into)
    }

    pub fn delete_file(&self, path: &str) -> Result<()> {
        // CASCADE handles: chunks → embeddings, call_edges
        self.conn.execute("DELETE FROM files WHERE path = ?1", params![path])?;
        Ok(())
    }

    pub fn list_all_indexed_paths(&self) -> Result<Vec<String>> {
        let mut stmt = self.conn.prepare("SELECT path FROM files")?;
        let paths = stmt.query_map([], |row| row.get(0))?
            .collect::<rusqlite::Result<Vec<String>>>()?;
        Ok(paths)
    }

    // ─── Chunks ──────────────────────────────────────────────────────────────

    pub fn insert_chunk(&self, file_id: i64, chunk: &CodeChunk) -> Result<i64> {
        self.conn.execute(
            "INSERT OR REPLACE INTO chunks
             (file_id, chunk_key, symbol_name, symbol_kind, parent_symbol, is_test,
              start_line, end_line, content, embedding_text, structural_fp, content_hash)
             VALUES (?1,?2,?3,?4,?5,?6,?7,?8,?9,?10,?11,?12)",
            params![
                file_id, &chunk.chunk_id,
                &chunk.symbol_name, &chunk.symbol_kind, &chunk.parent_symbol,
                chunk.is_test as i64,
                chunk.start_line, chunk.end_line,
                &chunk.content, &chunk.embedding_text, &chunk.structural_fp, &chunk.content_hash
            ],
        )?;
        Ok(self.conn.last_insert_rowid())
    }

    pub fn get_chunk(&self, chunk_key: &str) -> Result<Option<CodeChunk>> {
        self.conn.query_row(
            "SELECT c.*, f.path, f.language FROM chunks c
             JOIN files f ON c.file_id = f.id
             WHERE c.chunk_key = ?1",
            params![chunk_key],
            |row| Self::row_to_chunk(row),
        ).optional().map_err(Into::into)
    }

    pub fn get_all_chunks(&self) -> Result<Vec<CodeChunk>> {
        let mut stmt = self.conn.prepare(
            "SELECT c.*, f.path, f.language FROM chunks c JOIN files f ON c.file_id = f.id"
        )?;
        let chunks = stmt.query_map([], |row| Self::row_to_chunk(row))?
            .collect::<rusqlite::Result<Vec<_>>>()?;
        Ok(chunks)
    }

    pub fn find_chunk_by_symbol(&self, symbol: &str) -> Result<Option<String>> {
        // symbol can be "file::symbol" or just "symbol"
        let (file_hint, sym) = if let Some(pos) = symbol.rfind("::") {
            (Some(&symbol[..pos]), &symbol[pos+2..])
        } else {
            (None, symbol)
        };

        let query = match file_hint {
            Some(f) => self.conn.query_row(
                "SELECT chunk_key FROM chunks c JOIN files f ON c.file_id = f.id
                 WHERE c.symbol_name = ?1 AND f.path LIKE ?2 LIMIT 1",
                params![sym, format!("%{}%", f)],
                |row| row.get(0),
            ).optional()?,
            None => self.conn.query_row(
                "SELECT chunk_key FROM chunks WHERE symbol_name = ?1 LIMIT 1",
                params![sym],
                |row| row.get(0),
            ).optional()?,
        };
        Ok(query)
    }

    // ─── Embeddings ──────────────────────────────────────────────────────────

    pub fn upsert_embedding(&self, chunk_id: i64, vector: &[f32]) -> Result<()> {
        // Snapshot current vector to prev_vector before overwriting
        self.conn.execute(
            "UPDATE embeddings SET prev_vector = vector WHERE chunk_id = ?1",
            params![chunk_id],
        )?;

        let blob = vector_to_blob(vector);
        self.conn.execute(
            "INSERT OR REPLACE INTO embeddings(chunk_id, dim, vector) VALUES (?1, ?2, ?3)",
            params![chunk_id, vector.len() as i64, blob],
        )?;
        Ok(())
    }

    pub fn get_embedding(&self, chunk_key: &str) -> Result<Option<Vec<f32>>> {
        self.conn.query_row(
            "SELECT e.vector FROM embeddings e
             JOIN chunks c ON e.chunk_id = c.id
             WHERE c.chunk_key = ?1",
            params![chunk_key],
            |row| {
                let blob: Vec<u8> = row.get(0)?;
                Ok(blob_to_vector(&blob))
            },
        ).optional().map_err(Into::into)
    }

    pub fn get_all_embeddings(&self) -> Result<Vec<(String, Vec<f32>)>> {
        let mut stmt = self.conn.prepare(
            "SELECT c.chunk_key, e.vector FROM embeddings e JOIN chunks c ON e.chunk_id = c.id"
        )?;
        stmt.query_map([], |row| {
            let key: String = row.get(0)?;
            let blob: Vec<u8> = row.get(1)?;
            Ok((key, blob_to_vector(&blob)))
        })?.collect::<rusqlite::Result<Vec<_>>>().map_err(Into::into)
    }

    // ─── Call Edges ──────────────────────────────────────────────────────────

    pub fn insert_call_edge(&self, caller_key: &str, callee_name: &str,
                            callee_key: Option<&str>) -> Result<()> {
        self.conn.execute(
            "INSERT OR IGNORE INTO call_edges
             (caller_id, callee_name, callee_id)
             SELECT c1.id, ?2, c2.id
             FROM chunks c1
             LEFT JOIN chunks c2 ON c2.chunk_key = ?3
             WHERE c1.chunk_key = ?1",
            params![caller_key, callee_name, callee_key],
        )?;
        Ok(())
    }

    pub fn get_callees(&self, caller_key: &str) -> Result<Vec<Option<String>>> {
        let mut stmt = self.conn.prepare(
            "SELECT c2.chunk_key FROM call_edges ce
             JOIN chunks c1 ON ce.caller_id = c1.id
             LEFT JOIN chunks c2 ON ce.callee_id = c2.id
             WHERE c1.chunk_key = ?1"
        )?;
        stmt.query_map(params![caller_key], |row| row.get(0))?
            .collect::<rusqlite::Result<Vec<Option<String>>>>().map_err(Into::into)
    }

    pub fn get_callers(&self, callee_key: &str) -> Result<Vec<String>> {
        let mut stmt = self.conn.prepare(
            "SELECT c1.chunk_key FROM call_edges ce
             JOIN chunks c1 ON ce.caller_id = c1.id
             JOIN chunks c2 ON ce.callee_id = c2.id
             WHERE c2.chunk_key = ?1"
        )?;
        stmt.query_map(params![callee_key], |row| row.get(0))?
            .collect::<rusqlite::Result<Vec<String>>>().map_err(Into::into)
    }

    // ─── Synonyms ────────────────────────────────────────────────────────────

    pub fn upsert_synonym(&self, concept: &str, symbol: &str, weight: f32) -> Result<()> {
        self.conn.execute(
            "INSERT OR REPLACE INTO synonyms(concept, symbol, weight) VALUES (?1,?2,?3)",
            params![concept, symbol, weight],
        )?;
        Ok(())
    }

    pub fn get_synonyms(&self, concept: &str, limit: usize) -> Result<Vec<(String, f32)>> {
        let mut stmt = self.conn.prepare(
            "SELECT symbol, weight FROM synonyms WHERE concept = ?1
             ORDER BY weight DESC LIMIT ?2"
        )?;
        stmt.query_map(params![concept, limit as i64], |row| Ok((row.get(0)?, row.get(1)?)))?
            .collect::<rusqlite::Result<Vec<_>>>().map_err(Into::into)
    }

    // ─── Query Scores (Confidence Calibration) ───────────────────────────────

    pub fn record_query_score(&self, query_hash: &str, top1_score: f32) -> Result<()> {
        let now = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)?.as_secs() as i64;
        self.conn.execute(
            "INSERT INTO query_scores(query_hash, top1_score, queried_at) VALUES (?1,?2,?3)",
            params![query_hash, top1_score, now],
        )?;
        // Maintain rolling window of 100 entries
        self.conn.execute(
            "DELETE FROM query_scores WHERE id NOT IN
             (SELECT id FROM query_scores ORDER BY queried_at DESC LIMIT 100)",
            [],
        )?;
        Ok(())
    }

    pub fn get_recent_top1_scores(&self, limit: usize) -> Result<Vec<f32>> {
        let mut stmt = self.conn.prepare(
            "SELECT top1_score FROM query_scores ORDER BY queried_at DESC LIMIT ?1"
        )?;
        stmt.query_map(params![limit as i64], |row| row.get(0))?
            .collect::<rusqlite::Result<Vec<f32>>>().map_err(Into::into)
    }

    // ─── Helpers ─────────────────────────────────────────────────────────────

    fn row_to_chunk(row: &rusqlite::Row) -> rusqlite::Result<CodeChunk> {
        Ok(CodeChunk {
            chunk_id:      row.get("chunk_key")?,
            file_path:     PathBuf::from(row.get::<_, String>("path")?),
            language:      LanguageId::from_str(&row.get::<_, String>("language").unwrap_or_default()),
            start_line:    row.get("start_line")?,
            end_line:      row.get("end_line")?,
            symbol_name:   row.get("symbol_name")?,
            symbol_kind:   row.get("symbol_kind")?,
            parent_symbol: row.get("parent_symbol")?,
            is_test:       row.get::<_, i64>("is_test")? != 0,
            content:       row.get("content")?,
            embedding_text: row.get("embedding_text")?,
            structural_fp: row.get::<_, Option<String>>("structural_fp")?.unwrap_or_default(),
            content_hash:  row.get("content_hash")?,
            prev_embedding: None,  // loaded separately if needed
        })
    }
}

fn vector_to_blob(v: &[f32]) -> Vec<u8> {
    v.iter().flat_map(|f| f.to_le_bytes()).collect()
}

fn blob_to_vector(b: &[u8]) -> Vec<f32> {
    b.chunks_exact(4)
        .map(|c| f32::from_le_bytes([c[0], c[1], c[2], c[3]]))
        .collect()
}
```

---

### 51.2 `crates/search/src/confidence.rs` — Full Implementation

```rust
// crates/search/src/confidence.rs
// ~80 lines total

use std::collections::VecDeque;

#[derive(Debug)]
pub struct ConfidenceCalibrator {
    /// Rolling window of historical top-1 scores (max 100 entries)
    scores: VecDeque<f32>,
    /// Hardcoded fallback threshold — used until we have enough history
    fallback_threshold: f32,
    /// Minimum number of query scores needed before using calibrated threshold
    min_samples: usize,
}

impl Default for ConfidenceCalibrator {
    fn default() -> Self {
        Self {
            scores: VecDeque::with_capacity(100),
            fallback_threshold: 0.60,
            min_samples: 20,
        }
    }
}

impl ConfidenceCalibrator {
    pub fn from_history(scores: Vec<f32>) -> Self {
        let mut cal = Self::default();
        cal.scores.extend(scores.into_iter().take(100));
        cal
    }

    /// Record a new top-1 score after each query.
    pub fn record(&mut self, score: f32) {
        if self.scores.len() >= 100 {
            self.scores.pop_front();
        }
        self.scores.push_back(score);
    }

    /// Calibrated threshold = 20th percentile of historical top-1 scores.
    /// Falls back to hardcoded 0.60 until enough history is available.
    pub fn threshold(&self) -> f32 {
        if self.scores.len() < self.min_samples {
            return self.fallback_threshold;
        }
        let mut sorted: Vec<f32> = self.scores.iter().copied().collect();
        sorted.sort_by(|a, b| a.partial_cmp(b).unwrap());
        // 20th percentile
        sorted[sorted.len() / 5]
    }

    /// Classify a score into a confidence level.
    pub fn classify(&self, score: f32) -> ConfidenceLevel {
        if self.scores.len() < self.min_samples {
            return ConfidenceLevel::Unknown;
        }
        let t = self.threshold();
        match score {
            s if s > t + 0.15 => ConfidenceLevel::High,
            s if s > t        => ConfidenceLevel::Medium,
            _                 => ConfidenceLevel::Low,
        }
    }

    /// Generate user-facing warning message for low-confidence results.
    pub fn low_confidence_message(&self, score: f32, nearest_cluster: Option<&str>) -> String {
        let mut msg = format!(
            "WARNING: LOW CONFIDENCE — best match score {:.2} is below calibrated threshold {:.2}",
            score, self.threshold()
        );
        if self.scores.len() >= self.min_samples {
            let typical: f32 = self.scores.iter().filter(|&&s| s > self.threshold() + 0.15).sum::<f32>()
                / self.scores.iter().filter(|&&s| s > self.threshold() + 0.15).count().max(1) as f32;
            msg.push_str(&format!(
                "\n  Strong matches on this codebase typically score {:.2}+", typical
            ));
        }
        msg.push_str("\nSuggestions:");
        msg.push_str("\n  - Run 'smart-grep map' to see what this codebase does contain");
        if let Some(cluster) = nearest_cluster {
            msg.push_str(&format!(
                "\n  - Nearest cluster: \"{}\" — your query may be misdirected", cluster
            ));
        }
        msg
    }
}
```

---

### 51.3 `crates/search/src/lint.rs` — Full Implementation

```rust
// crates/search/src/lint.rs

use std::collections::HashMap;

#[derive(Debug, Default)]
pub struct LintReport {
    pub naming_inconsistencies: Vec<NamingIssue>,
    pub duplicate_logic:        Vec<DuplicateIssue>,
    pub orphaned_chunks:        Vec<CodeChunk>,
}

#[derive(Debug)]
pub struct NamingIssue {
    pub chunk_a:         CodeChunk,
    pub chunk_b:         CodeChunk,
    pub semantic_sim:    f32,
    pub lexical_distance: f32,
    pub suggestion:      String,
}

#[derive(Debug)]
pub struct DuplicateIssue {
    pub chunk_a:      CodeChunk,
    pub chunk_b:      CodeChunk,
    pub similarity:   f32,
    pub suggestion:   String,
}

pub fn run_lint(
    db: &Db,
    hnsw: &HnswIndex,
    config: &LintConfig,
) -> anyhow::Result<LintReport> {
    let all_chunks = db.get_all_chunks()?;
    let mut report = LintReport::default();
    let mut seen_pairs: std::collections::HashSet<(String, String)> = std::collections::HashSet::new();

    for chunk in &all_chunks {
        let Some(embedding) = db.get_embedding(&chunk.chunk_id)? else { continue };

        // Get top-20 nearest neighbors for this chunk
        let neighbors = hnsw.search(&embedding, 20)?;

        let mut has_close_neighbor = false;

        for (neighbor_id, sim) in &neighbors {
            if neighbor_id == &chunk.chunk_id { continue; }

            // Canonicalize pair key to avoid duplicates (A,B) == (B,A)
            let pair_key = {
                let mut pair = vec![chunk.chunk_id.clone(), neighbor_id.clone()];
                pair.sort();
                (pair[0].clone(), pair[1].clone())
            };
            if seen_pairs.contains(&pair_key) { continue; }
            seen_pairs.insert(pair_key);

            let Some(neighbor) = db.get_chunk(neighbor_id)? else { continue };

            // Track whether this chunk has any close neighbor at all
            if *sim >= config.orphan_similarity_threshold {
                has_close_neighbor = true;
            }

            // 1. Naming inconsistency: high semantic sim + low lexical sim
            if *sim >= config.naming_similarity_threshold {
                let lex_dist = lexical_distance(
                    chunk.symbol_name.as_deref().unwrap_or(""),
                    neighbor.symbol_name.as_deref().unwrap_or(""),
                );
                if lex_dist > 0.5 {
                    let suggestion = suggest_canonical_name(&chunk, &neighbor, db)?;
                    report.naming_inconsistencies.push(NamingIssue {
                        chunk_a: chunk.clone(),
                        chunk_b: neighbor.clone(),
                        semantic_sim: *sim,
                        lexical_distance: lex_dist,
                        suggestion,
                    });
                }
            }

            // 2. Duplicate logic: very high semantic similarity
            if *sim >= config.duplicate_similarity_threshold {
                report.duplicate_logic.push(DuplicateIssue {
                    chunk_a: chunk.clone(),
                    chunk_b: neighbor.clone(),
                    similarity: *sim,
                    suggestion: format!(
                        "Consider extracting shared logic into a common function. \
                         These two functions are {:.0}% semantically similar.",
                        sim * 100.0
                    ),
                });
            }
        }

        // 3. Orphaned code: no semantic neighbors above threshold
        if !has_close_neighbor && !chunk.is_test {
            report.orphaned_chunks.push(chunk.clone());
        }
    }

    Ok(report)
}

/// Normalized edit distance between two symbol names (case-insensitive, tokenized)
fn lexical_distance(a: &str, b: &str) -> f32 {
    let a_tokens: Vec<&str> = tokenize_symbol(a);
    let b_tokens: Vec<&str> = tokenize_symbol(b);
    let intersection = a_tokens.iter().filter(|t| b_tokens.contains(t)).count();
    let union = (a_tokens.len() + b_tokens.len() - intersection).max(1);
    1.0 - intersection as f32 / union as f32
}

fn tokenize_symbol(s: &str) -> Vec<&str> {
    // Split on camelCase, snake_case boundaries
    // Simplified: split on uppercase transitions and underscores
    // Full implementation uses regex or char-level state machine
    s.split('_').collect()
}

/// Suggest the canonical name (whichever is used more frequently in the codebase)
fn suggest_canonical_name(a: &CodeChunk, b: &CodeChunk, db: &Db) -> anyhow::Result<String> {
    let count_a = db.count_symbol_usages(a.symbol_name.as_deref().unwrap_or(""))?;
    let count_b = db.count_symbol_usages(b.symbol_name.as_deref().unwrap_or(""))?;
    let preferred = if count_a >= count_b { &a.symbol_name } else { &b.symbol_name };
    Ok(format!("Consider standardizing to '{}' (used in {} locations)",
        preferred.as_deref().unwrap_or("unknown"), count_a.max(count_b)))
}

pub fn format_lint_report(report: &LintReport) -> String {
    let mut out = String::new();

    if !report.naming_inconsistencies.is_empty() {
        out.push_str(&format!("\nNAMING INCONSISTENCY ({} pairs)\n", report.naming_inconsistencies.len()));
        out.push_str(&"─".repeat(60));
        out.push('\n');
        for issue in &report.naming_inconsistencies {
            out.push_str(&format!(
                "  {}::{}  similarity: {:.2}\n  {}::{}  ← near-identical behavior, different naming\n  {}\n\n",
                issue.chunk_a.file_path.display(), issue.chunk_a.symbol_name.as_deref().unwrap_or("?"),
                issue.semantic_sim,
                issue.chunk_b.file_path.display(), issue.chunk_b.symbol_name.as_deref().unwrap_or("?"),
                issue.suggestion,
            ));
        }
    }

    if !report.duplicate_logic.is_empty() {
        out.push_str(&format!("\nDUPLICATE LOGIC ({} pairs)\n", report.duplicate_logic.len()));
        out.push_str(&"─".repeat(60));
        out.push('\n');
        for issue in &report.duplicate_logic {
            out.push_str(&format!(
                "  {}::{}  similarity: {:.2}\n  {}::{}  ← likely duplicate — {}\n\n",
                issue.chunk_a.file_path.display(), issue.chunk_a.symbol_name.as_deref().unwrap_or("?"),
                issue.similarity,
                issue.chunk_b.file_path.display(), issue.chunk_b.symbol_name.as_deref().unwrap_or("?"),
                issue.suggestion,
            ));
        }
    }

    if !report.orphaned_chunks.is_empty() {
        out.push_str(&format!("\nORPHANED CODE ({} chunks)\n", report.orphaned_chunks.len()));
        out.push_str(&"─".repeat(60));
        out.push('\n');
        for chunk in &report.orphaned_chunks {
            out.push_str(&format!(
                "  {}::{}  ← isolated; may be dead code\n",
                chunk.file_path.display(), chunk.symbol_name.as_deref().unwrap_or("?"),
            ));
        }
    }

    if report.naming_inconsistencies.is_empty()
        && report.duplicate_logic.is_empty()
        && report.orphaned_chunks.is_empty() {
        out.push_str("No naming issues, duplicates, or orphaned code found.\n");
    }

    out
}
```

---

### 51.4 `crates/search/src/session.rs` — Full Implementation

```rust
// crates/search/src/session.rs

use std::collections::HashMap;

pub struct SessionManager {
    db: Db,
}

impl SessionManager {
    pub fn start(&self, name: &str) -> anyhow::Result<Session> {
        let now = unix_now();
        self.db.conn.execute(
            "INSERT OR REPLACE INTO sessions(name, created_at, updated_at) VALUES (?1,?2,?3)",
            params![name, now, now],
        )?;
        tracing::info!("Started session: {}", name);
        Ok(Session { name: name.to_string(), id: self.db.conn.last_insert_rowid() })
    }

    pub fn record_result(&self, session_name: &str, query: &str, result: &SearchResult) -> anyhow::Result<()> {
        let session_id = self.get_session_id(session_name)?
            .ok_or_else(|| anyhow::anyhow!("Session '{}' not found", session_name))?;

        let now = unix_now();
        self.db.conn.execute(
            "INSERT INTO session_results(session_id, query, chunk_id, score, queried_at)
             SELECT ?1, ?2, c.id, ?3, ?4
             FROM chunks c WHERE c.chunk_key = ?5",
            params![session_id, query, result.score, now, result.chunk.chunk_id],
        )?;

        // Update session updated_at
        self.db.conn.execute(
            "UPDATE sessions SET updated_at = ?1 WHERE id = ?2",
            params![now, session_id],
        )?;

        Ok(())
    }

    pub fn context(
        &self,
        session_name: &str,
        budget_tokens: usize,
        db: &Db,
    ) -> anyhow::Result<Vec<SearchResult>> {
        // Get all session results, deduplicated by chunk_id, keeping max score
        let mut by_chunk: HashMap<String, (SearchResult, f32)> = HashMap::new();

        let session_id = self.get_session_id(session_name)?
            .ok_or_else(|| anyhow::anyhow!("Session '{}' not found", session_name))?;

        let mut stmt = db.conn.prepare(
            "SELECT DISTINCT sr.chunk_id, sr.score, sr.query
             FROM session_results sr
             WHERE sr.session_id = ?1
             ORDER BY sr.score DESC"
        )?;

        let rows: Vec<(i64, f32, String)> = stmt.query_map(params![session_id], |row| {
            Ok((row.get(0)?, row.get(1)?, row.get(2)?))
        })?.collect::<rusqlite::Result<Vec<_>>>()?;

        for (chunk_rowid, score, query) in rows {
            let chunk_key = db.get_chunk_key_by_rowid(chunk_rowid)?;
            if let Some(chunk) = db.get_chunk(&chunk_key)? {
                let entry = by_chunk.entry(chunk_key).or_insert_with(|| {
                    (SearchResult {
                        rank: 0, chunk, score,
                        semantic_score: score, bm25_boost: 0.0,
                        confidence: ConfidenceLevel::Unknown,
                        matched_queries: vec![query.clone()],
                        why: None,
                    }, score)
                });
                if score > entry.1 {
                    entry.0.score = score;
                    entry.1 = score;
                }
                if !entry.0.matched_queries.contains(&query) {
                    entry.0.matched_queries.push(query);
                }
            }
        }

        let mut results: Vec<SearchResult> = by_chunk.into_values().map(|(r, _)| r).collect();
        results.sort_by(|a, b| b.score.partial_cmp(&a.score).unwrap());

        // Apply budget packing
        Ok(budget::pack_budget(results, budget_tokens))
    }

    pub fn summary(&self, session_name: &str, db: &Db, clusters: &[Cluster]) -> anyhow::Result<SessionSummary> {
        let session_id = self.get_session_id(session_name)?
            .ok_or_else(|| anyhow::anyhow!("Session '{}' not found", session_name))?;

        // Count unique queries and chunks
        let query_count: i64 = db.conn.query_row(
            "SELECT COUNT(DISTINCT query) FROM session_results WHERE session_id = ?1",
            params![session_id], |r| r.get(0),
        )?;
        let chunk_count: i64 = db.conn.query_row(
            "SELECT COUNT(DISTINCT chunk_id) FROM session_results WHERE session_id = ?1",
            params![session_id], |r| r.get(0),
        )?;

        // Identify which cluster labels are covered by session chunks
        let covered_clusters = self.find_covered_clusters(session_id, db, clusters)?;
        let unexplored_clusters: Vec<String> = clusters.iter()
            .filter(|c| !covered_clusters.contains(&c.label))
            .map(|c| c.label.clone())
            .take(5)
            .collect();

        Ok(SessionSummary {
            name: session_name.to_string(),
            query_count: query_count as usize,
            chunk_count: chunk_count as usize,
            covered_cluster_labels: covered_clusters,
            unexplored_suggestions: unexplored_clusters,
        })
    }

    fn get_session_id(&self, name: &str) -> anyhow::Result<Option<i64>> {
        self.db.conn.query_row(
            "SELECT id FROM sessions WHERE name = ?1 ORDER BY updated_at DESC LIMIT 1",
            params![name], |r| r.get(0),
        ).optional().map_err(Into::into)
    }

    pub fn expire_old_sessions(&self, ttl_days: u64) -> anyhow::Result<usize> {
        let cutoff = unix_now() - (ttl_days * 86400) as i64;
        let deleted = self.db.conn.execute(
            "DELETE FROM sessions WHERE updated_at < ?1",
            params![cutoff],
        )?;
        Ok(deleted)
    }
}

#[derive(Debug)]
pub struct SessionSummary {
    pub name:                    String,
    pub query_count:             usize,
    pub chunk_count:             usize,
    pub covered_cluster_labels:  Vec<String>,
    pub unexplored_suggestions:  Vec<String>,
}

fn unix_now() -> i64 {
    std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .unwrap()
        .as_secs() as i64
}
```

---

### 51.5 `crates/store/src/snapshot.rs` — Full Implementation

```rust
// crates/store/src/snapshot.rs

use std::path::{Path, PathBuf};
use std::fs;

pub struct SnapshotManager {
    snapshots_dir: PathBuf,
    max_snapshots: usize,
}

impl SnapshotManager {
    pub fn new(index_dir: &Path, max_snapshots: usize) -> Self {
        Self {
            snapshots_dir: index_dir.join("snapshots"),
            max_snapshots,
        }
    }

    /// Save a named snapshot of the current index DB.
    pub fn save(&self, db: &Db, ref_name: &str) -> anyhow::Result<Snapshot> {
        fs::create_dir_all(&self.snapshots_dir)?;

        // Sanitize ref_name for use as directory name
        let safe_name = ref_name.replace(['/', '\\', ' ', ':'], "_");
        let snap_dir = self.snapshots_dir.join(&safe_name);
        fs::create_dir_all(&snap_dir)?;

        let snap_db_path = snap_dir.join("index.db");

        // Use SQLite's online backup API for a consistent copy while writers may be active
        Self::sqlite_backup(&db.conn, &snap_db_path)?;

        let now = unix_now();

        // Register in the snapshots table
        db.conn.execute(
            "INSERT INTO snapshots(ref_name, created_at, db_path) VALUES (?1,?2,?3)",
            params![ref_name, now, snap_db_path.to_str().unwrap()],
        )?;

        // Enforce max_snapshots with LRU eviction
        self.evict_old_snapshots(db)?;

        tracing::info!("Saved snapshot '{}' to {}", ref_name, snap_db_path.display());
        Ok(Snapshot { ref_name: ref_name.to_string(), db_path: snap_db_path, created_at: now })
    }

    /// Open a snapshot for read-only time-travel queries.
    pub fn open_snapshot(&self, ref_name: &str, db: &Db) -> anyhow::Result<Db> {
        let snap_db_path = db.conn.query_row(
            "SELECT db_path FROM snapshots WHERE ref_name = ?1 ORDER BY created_at DESC LIMIT 1",
            params![ref_name],
            |r| r.get::<_, String>(0),
        ).map_err(|_| anyhow::anyhow!(
            "Snapshot '{}' not found. Run 'smart-grep snapshot list' to see available snapshots.",
            ref_name
        ))?;

        Db::open_readonly(Path::new(&snap_db_path))
    }

    /// List all available snapshots sorted by creation time (newest first)
    pub fn list(&self, db: &Db) -> anyhow::Result<Vec<SnapshotInfo>> {
        let mut stmt = db.conn.prepare(
            "SELECT ref_name, created_at, db_path FROM snapshots ORDER BY created_at DESC"
        )?;
        stmt.query_map([], |row| Ok(SnapshotInfo {
            ref_name:   row.get(0)?,
            created_at: row.get(1)?,
            db_path:    PathBuf::from(row.get::<_, String>(2)?),
        }))?.collect::<rusqlite::Result<Vec<_>>>().map_err(Into::into)
    }

    fn evict_old_snapshots(&self, db: &Db) -> anyhow::Result<()> {
        let snapshots = self.list(db)?;
        if snapshots.len() <= self.max_snapshots { return Ok(()); }

        // Delete the oldest snapshots (they are sorted newest-first, so take the tail)
        for snap in snapshots.iter().skip(self.max_snapshots) {
            if snap.db_path.exists() {
                if let Some(parent) = snap.db_path.parent() {
                    fs::remove_dir_all(parent)?;
                }
            }
            db.conn.execute(
                "DELETE FROM snapshots WHERE ref_name = ?1 AND created_at = ?2",
                params![snap.ref_name, snap.created_at],
            )?;
        }
        Ok(())
    }

    /// SQLite online backup — safe copy while database is open
    fn sqlite_backup(source: &rusqlite::Connection, dest_path: &Path) -> anyhow::Result<()> {
        let dest = rusqlite::Connection::open(dest_path)?;
        let backup = rusqlite::backup::Backup::new(source, &dest)?;
        backup.run_to_completion(100, std::time::Duration::from_millis(5), None)?;
        Ok(())
    }
}

#[derive(Debug)]
pub struct Snapshot {
    pub ref_name:   String,
    pub db_path:    PathBuf,
    pub created_at: i64,
}

#[derive(Debug)]
pub struct SnapshotInfo {
    pub ref_name:   String,
    pub created_at: i64,
    pub db_path:    PathBuf,
}
```

---

## 52. CLI Entry Point (`crates/core/src/main.rs`)

The full CLI entry point wires all commands together via `clap` derive macros:

```rust
// crates/core/src/main.rs

use clap::{Parser, Subcommand, Args};

#[derive(Parser)]
#[command(name = "smart-grep", about = "Semantic code search — find code by meaning, not keywords")]
struct Cli {
    #[command(subcommand)]
    command: Option<Command>,

    /// Natural language query (shorthand for 'search')
    #[arg(value_name = "QUERY")]
    query: Option<String>,

    // Search flags (used when query is positional)
    #[command(flatten)]
    search_args: SearchArgs,
}

#[derive(Subcommand)]
enum Command {
    /// Index the current directory (or specified path)
    Index(IndexArgs),
    /// Semantic search (natural language or code snippet)
    Search(SearchArgs),
    /// Multi-query fan-out with coverage boosting
    Multi(MultiArgs),
    /// Show semantic cluster map of the codebase
    Map(MapArgs),
    /// Show semantic diff between git refs or snapshots
    Diff(DiffArgs),
    /// Find stale code similar to a changed function
    Stale(StaleArgs),
    /// Show semantic test coverage for a function
    Coverage(CoverageArgs),
    /// Lint codebase for naming inconsistencies and duplicate logic
    Lint(LintArgs),
    /// Session management (start, summary, context, list, resume)
    Session(SessionArgs),
    /// Snapshot management (save, list)
    Snapshot(SnapshotArgs),
    /// Show index statistics
    Stats,
    /// Diagnose index health
    Doctor,
    /// Start MCP server
    Serve(ServeArgs),
}

#[derive(Args)]
struct SearchArgs {
    #[arg(short, long, default_value = "5")]
    top: usize,
    #[arg(long)]
    lang: Option<String>,
    #[arg(long)]
    json: bool,
    #[arg(long, value_name = "TOKENS")]
    budget: Option<usize>,
    #[arg(long)]
    expand: bool,
    #[arg(long, value_name = "FILE::SYMBOL")]
    from: Option<String>,
    #[arg(long, default_value = "3")]
    from_depth: u32,
    #[arg(long)]
    why: bool,
    #[arg(long, value_name = "SNIPPET")]
    like: Option<String>,
    #[arg(long, value_name = "DESCRIPTION")]
    pattern: Option<String>,
    #[arg(long)]
    stream: bool,
    #[arg(long, value_name = "SNAPSHOT")]
    at: Option<String>,
    #[arg(long, default_value = "warn")]
    confidence: ConfidenceMode,
    #[arg(long)]
    verbose: bool,
    #[arg(long)]
    self_index: bool,
}

#[derive(Args)]
struct IndexArgs {
    #[arg(default_value = ".")]
    path: std::path::PathBuf,
    #[arg(long)]
    rebuild: bool,
    #[arg(long)]
    verbose: bool,
    #[arg(long)]
    watch: bool,
}

// ... (other Args structs)

fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt()
        .with_env_filter(tracing_subscriber::EnvFilter::from_default_env())
        .init();

    let cli = Cli::parse();

    // Locate or create the index directory
    let repo_root = find_repo_root()?;
    let index_dir = repo_root.join(".smart-grep");

    match cli.command {
        Some(Command::Index(args)) => cmd_index(&args, &index_dir),
        Some(Command::Search(args)) => cmd_search(args.query.as_deref(), &args, &index_dir),
        Some(Command::Multi(args)) => cmd_multi(&args, &index_dir),
        Some(Command::Map(args)) => cmd_map(&args, &index_dir),
        Some(Command::Diff(args)) => cmd_diff(&args, &index_dir),
        Some(Command::Stale(args)) => cmd_stale(&args, &index_dir),
        Some(Command::Coverage(args)) => cmd_coverage(&args, &index_dir),
        Some(Command::Lint(args)) => cmd_lint(&args, &index_dir),
        Some(Command::Session(args)) => cmd_session(&args, &index_dir),
        Some(Command::Snapshot(args)) => cmd_snapshot(&args, &index_dir),
        Some(Command::Stats) => cmd_stats(&index_dir),
        Some(Command::Doctor) => cmd_doctor(&index_dir),
        Some(Command::Serve(args)) => cmd_serve(&args, &index_dir),
        None => {
            // Positional query: smart-grep "payment error"
            if let Some(query) = &cli.query {
                cmd_search(Some(query), &cli.search_args, &index_dir)
            } else {
                Cli::parse_from(["smart-grep", "--help"]);
                Ok(())
            }
        }
    }
}
```

---

## 53. `smart-grep doctor` Implementation

```rust
// In crates/core/src/main.rs

fn cmd_doctor(index_dir: &Path) -> anyhow::Result<()> {
    println!("smart-grep doctor\n{}", "─".repeat(50));

    let mut all_ok = true;

    // 1. Check tree-sitter grammars
    for (lang, lang_fn) in &[
        ("Rust",       tree_sitter_rust::language as fn() -> _),
        ("TypeScript", tree_sitter_typescript::language_typescript as fn() -> _),
        ("JavaScript", tree_sitter_javascript::language as fn() -> _),
        ("Python",     tree_sitter_python::language as fn() -> _),
        ("Ruby",       tree_sitter_ruby::language as fn() -> _),
    ] {
        match std::panic::catch_unwind(|| lang_fn()) {
            Ok(_)  => println!("✓ tree-sitter grammar: {}", lang),
            Err(_) => { println!("✗ tree-sitter grammar: {} FAILED", lang); all_ok = false; }
        }
    }

    // 2. Check fastembed model
    match fastembed::TextEmbedding::try_new(fastembed::InitOptions::default()) {
        Ok(_)  => println!("✓ fastembed model: Xenova/all-MiniLM-L6-v2 present"),
        Err(e) => { println!("✗ fastembed model: {} — run 'smart-grep index' to download", e); all_ok = false; }
    }

    // 3. Check SQLite index
    let db_path = index_dir.join("index.db");
    if db_path.exists() {
        match Db::open(index_dir) {
            Ok(db) => {
                let schema_ver: String = db.conn.query_row(
                    "SELECT value FROM meta WHERE key='schema_version'", [], |r| r.get(0)
                ).unwrap_or_else(|_| "unknown".to_string());
                println!("✓ SQLite index: readable, schema version {}", schema_ver);

                // 4. Check vector dimension consistency
                let stored_dim: Option<String> = db.conn.query_row(
                    "SELECT value FROM meta WHERE key='vector_dim'", [], |r| r.get(0)
                ).optional().unwrap_or(None);
                let embedder = FastEmbedder::new().unwrap();
                if let Some(dim_str) = stored_dim {
                    let stored: usize = dim_str.parse().unwrap_or(0);
                    if stored == embedder.dimension() {
                        println!("✓ HNSW index: {} dimensions match meta.vector_dim", stored);
                    } else {
                        println!("✗ HNSW index: dimension mismatch ({} stored vs {} model) — run 'smart-grep index --rebuild'", stored, embedder.dimension());
                        all_ok = false;
                    }
                }

                // 5. Session store check
                let session_count: i64 = db.conn.query_row(
                    "SELECT COUNT(*) FROM sessions", [], |r| r.get(0)
                ).unwrap_or(0);
                println!("✓ Session store: {} active sessions", session_count);
            }
            Err(e) => {
                println!("✗ SQLite index: {} — run 'smart-grep index --rebuild'", e);
                all_ok = false;
            }
        }
    } else {
        println!("✗ SQLite index: not found at {} — run 'smart-grep index'", db_path.display());
        all_ok = false;
    }

    println!("\n{}", "─".repeat(50));
    if all_ok {
        println!("No issues found. smart-grep is healthy.");
        Ok(())
    } else {
        eprintln!("Issues found. See suggestions above.");
        std::process::exit(1);
    }
}
```

---

## 54. `smart-grep stats` Implementation

```rust
fn cmd_stats(index_dir: &Path) -> anyhow::Result<()> {
    let db = Db::open(index_dir)?;

    let total_files: i64     = db.conn.query_row("SELECT COUNT(*) FROM files", [], |r| r.get(0))?;
    let parse_errors: i64    = db.conn.query_row("SELECT COUNT(*) FROM files WHERE parse_error=1", [], |r| r.get(0))?;
    let total_chunks: i64    = db.conn.query_row("SELECT COUNT(*) FROM chunks", [], |r| r.get(0))?;
    let test_chunks: i64     = db.conn.query_row("SELECT COUNT(*) FROM chunks WHERE is_test=1", [], |r| r.get(0))?;
    let call_edges: i64      = db.conn.query_row("SELECT COUNT(*) FROM call_edges", [], |r| r.get(0))?;
    let active_sessions: i64 = db.conn.query_row("SELECT COUNT(*) FROM sessions", [], |r| r.get(0))?;
    let snapshots: i64       = db.conn.query_row("SELECT COUNT(*) FROM snapshots", [], |r| r.get(0))?;
    let synonyms: i64        = db.conn.query_row("SELECT COUNT(*) FROM synonyms", [], |r| r.get(0))?;

    // Language breakdown
    let mut lang_stmt = db.conn.prepare(
        "SELECT language, COUNT(*) as cnt FROM files WHERE language IS NOT NULL GROUP BY language ORDER BY cnt DESC"
    )?;
    let lang_counts: Vec<(String, i64)> = lang_stmt.query_map([], |r| Ok((r.get(0)?, r.get(1)?)))?
        .collect::<rusqlite::Result<Vec<_>>>()?;

    // DB file size
    let db_path = index_dir.join("index.db");
    let db_size = std::fs::metadata(&db_path).map(|m| m.len()).unwrap_or(0);

    // Model info
    let model_name: String = db.conn.query_row(
        "SELECT value FROM meta WHERE key='model_name'", [], |r| r.get(0)
    ).unwrap_or_else(|_| "unknown".to_string());
    let vector_dim: String = db.conn.query_row(
        "SELECT value FROM meta WHERE key='vector_dim'", [], |r| r.get(0)
    ).unwrap_or_else(|_| "unknown".to_string());
    let last_indexed: String = db.conn.query_row(
        "SELECT value FROM meta WHERE key='last_indexed_at'", [], |r| r.get(0)
    ).unwrap_or_else(|_| "never".to_string());

    println!("smart-grep stats\n{}", "─".repeat(60));
    println!("Files indexed:     {:>6}", total_files);
    if parse_errors > 0 {
        println!("  Parse errors:    {:>6} (run 'smart-grep index --verbose' for details)", parse_errors);
    }
    println!("Total chunks:      {:>6}", total_chunks);
    println!("  Production:      {:>6}", total_chunks - test_chunks);
    println!("  Test:            {:>6}", test_chunks);

    print!("Languages:         ");
    let lang_str: Vec<String> = lang_counts.iter()
        .map(|(l, c)| format!("{} ({})", l, c))
        .collect();
    println!("{}", lang_str.join(", "));

    println!("Model:             {} ({} dimensions)", model_name, vector_dim);
    println!("Index DB size:     {}", format_bytes(db_size));
    println!("Last indexed:      {}", format_timestamp(&last_indexed));
    println!("Call edges:        {:>6}", call_edges);
    println!("Active sessions:   {:>6}", active_sessions);
    println!("Snapshots:         {:>6}", snapshots);
    println!("Synonyms:          {:>6} concept→symbol mappings", synonyms);

    Ok(())
}

fn format_bytes(bytes: u64) -> String {
    if bytes < 1024 { format!("{} B", bytes) }
    else if bytes < 1024 * 1024 { format!("{:.1} KB", bytes as f64 / 1024.0) }
    else { format!("{:.1} MB", bytes as f64 / (1024.0 * 1024.0)) }
}
```

---

## 55. Graph Extraction Implementation

### 55.1 `crates/indexer/src/graph.rs`

```rust
// crates/indexer/src/graph.rs

use tree_sitter::Node;

/// Extract call edges from a Rust source file
pub fn extract_rust_calls(path: &Path, source: &str, chunks: &[CodeChunk]) -> Vec<CallEdge> {
    let mut parser = tree_sitter::Parser::new();
    parser.set_language(tree_sitter_rust::language()).unwrap();

    let tree = match parser.parse(source, None) {
        Some(t) => t,
        None => return vec![],
    };

    let mut edges = Vec::new();
    let bytes = source.as_bytes();

    // Walk all call_expression nodes
    walk_tree(&tree.root_node(), &mut |node: &Node| {
        if node.kind() == "call_expression" {
            // The function being called is the first child
            if let Some(func_node) = node.child(0) {
                let callee_name = extract_call_target(&func_node, bytes);

                // Find which chunk this call site belongs to (by line number)
                let call_line = node.start_position().row as u32 + 1;
                if let Some(caller) = find_chunk_at_line(chunks, call_line) {
                    // Find the callee chunk by symbol name (may be None if external)
                    let callee_chunk_id = chunks.iter()
                        .find(|c| c.symbol_name.as_deref() == Some(&callee_name))
                        .map(|c| c.chunk_id.clone());

                    edges.push(CallEdge {
                        caller_id: caller.chunk_id.clone(),
                        callee_name: callee_name.clone(),
                        callee_id: callee_chunk_id,
                    });
                }
            }
        }
    });

    // Dedup: same caller→callee pair may appear multiple times (multiple call sites)
    edges.sort_by(|a, b| (&a.caller_id, &a.callee_name).cmp(&(&b.caller_id, &b.callee_name)));
    edges.dedup_by(|a, b| a.caller_id == b.caller_id && a.callee_name == b.callee_name);

    edges
}

/// Extract the name of what is being called from a call expression's function node
fn extract_call_target(func_node: &Node, bytes: &[u8]) -> String {
    match func_node.kind() {
        // Direct function call: foo()
        "identifier" => func_node.utf8_text(bytes).unwrap_or("").to_string(),
        // Method call: obj.method()
        "field_expression" => {
            if let Some(field) = func_node.child_by_field_name("field") {
                field.utf8_text(bytes).unwrap_or("").to_string()
            } else {
                func_node.utf8_text(bytes).unwrap_or("").to_string()
            }
        }
        // Scoped call: Module::function()
        "scoped_identifier" => {
            // Take the last segment: Stripe::charge() → "charge"
            if let Some(name) = func_node.child_by_field_name("name") {
                name.utf8_text(bytes).unwrap_or("").to_string()
            } else {
                func_node.utf8_text(bytes).unwrap_or("").to_string()
            }
        }
        _ => func_node.utf8_text(bytes).unwrap_or("").to_string()
    }
}

fn find_chunk_at_line<'a>(chunks: &'a [CodeChunk], line: u32) -> Option<&'a CodeChunk> {
    chunks.iter().find(|c| c.start_line <= line && line <= c.end_line)
}

fn walk_tree(node: &Node, visitor: &mut impl FnMut(&Node)) {
    visitor(node);
    let mut cursor = node.walk();
    for child in node.children(&mut cursor) {
        walk_tree(&child, visitor);
    }
}
```

---

## 56. Project Detection Implementation

```rust
// crates/indexer/src/project_detect.rs

use std::path::Path;

#[derive(Debug, PartialEq)]
pub enum ProjectKind {
    RailsApp,
    NodeReact,
    RustWorkspace,
    Python,
    Unknown,
}

pub fn detect_project(root: &Path) -> ProjectKind {
    if root.join("Gemfile").exists() && root.join("app").is_dir() {
        return ProjectKind::RailsApp;
    }
    if root.join("package.json").exists() && root.join("src").is_dir() {
        return ProjectKind::NodeReact;
    }
    if root.join("Cargo.toml").exists() {
        // Check if it's a workspace
        let cargo_toml = std::fs::read_to_string(root.join("Cargo.toml")).unwrap_or_default();
        if cargo_toml.contains("[workspace]") {
            return ProjectKind::RustWorkspace;
        }
    }
    if root.join("pyproject.toml").exists() || root.join("setup.py").exists() {
        return ProjectKind::Python;
    }
    ProjectKind::Unknown
}

pub fn generate_config(kind: &ProjectKind) -> Config {
    match kind {
        ProjectKind::RailsApp => Config {
            include: vec!["app".into(), "lib".into(), "config".into()],
            exclude: vec!["vendor".into(), "log".into(), "tmp".into(),
                          "public/assets".into(), "node_modules".into()],
            ..Default::default()
        },
        ProjectKind::NodeReact => Config {
            include: vec!["src".into()],
            exclude: vec!["node_modules".into(), "dist".into(), ".next".into(),
                          "build".into(), "coverage".into()],
            ..Default::default()
        },
        ProjectKind::RustWorkspace => Config {
            include: vec!["crates".into(), "src".into()],
            exclude: vec!["target".into()],
            ..Default::default()
        },
        ProjectKind::Python => Config {
            include: vec!["src".into(), "lib".into()],
            exclude: vec![".venv".into(), "__pycache__".into(), "dist".into(),
                          ".eggs".into(), "build".into()],
            ..Default::default()
        },
        ProjectKind::Unknown => Config::default(),
    }
}

pub fn seed_benchmark_queries(kind: &ProjectKind, root: &Path) -> Vec<String> {
    let mut queries = match kind {
        ProjectKind::RailsApp => vec![
            "where is user authentication implemented?".to_string(),
            "where are ActiveRecord callbacks defined?".to_string(),
            "where is background job processing?".to_string(),
            "where are API endpoints defined?".to_string(),
            "where is email delivery handled?".to_string(),
        ],
        ProjectKind::NodeReact => vec![
            "where is the main API client?".to_string(),
            "where is user authentication handled?".to_string(),
            "where are form validation errors handled?".to_string(),
            "where is the main routing configuration?".to_string(),
            "where is global state management?".to_string(),
        ],
        ProjectKind::RustWorkspace => vec![
            "where is error handling implemented?".to_string(),
            "where is async runtime configured?".to_string(),
            "where are CLI arguments parsed?".to_string(),
            "where is database connection pooling?".to_string(),
        ],
        _ => vec![
            "where is the main entry point?".to_string(),
            "where is error handling?".to_string(),
            "where is the database layer?".to_string(),
        ],
    };

    // Detect Stripe dependency → add payment-specific query
    let has_stripe = detect_dependency(root, "stripe");
    if has_stripe {
        queries.push("where is the Stripe webhook handler?".to_string());
        queries.push("where is payment error handling?".to_string());
    }

    queries
}

fn detect_dependency(root: &Path, dep: &str) -> bool {
    // Check package.json, Gemfile, Cargo.toml, pyproject.toml for the dependency
    let files = ["package.json", "Gemfile", "Cargo.toml", "pyproject.toml", "requirements.txt"];
    files.iter().any(|f| {
        std::fs::read_to_string(root.join(f))
            .map(|content| content.to_lowercase().contains(dep))
            .unwrap_or(false)
    })
}
```

---

## 57. `--watch` Mode Implementation

```rust
// crates/core/src/main.rs — watch mode

fn cmd_index_watch(args: &IndexArgs, index_dir: &Path) -> anyhow::Result<()> {
    use notify::{Watcher, RecursiveMode, Event, EventKind};
    use std::sync::mpsc;
    use std::time::Duration;

    // Do an initial full index first
    cmd_index(args, index_dir)?;

    let (tx, rx) = mpsc::channel();

    let mut watcher = notify::recommended_watcher(move |res: notify::Result<Event>| {
        if let Ok(event) = res {
            match event.kind {
                EventKind::Modify(_) | EventKind::Create(_) | EventKind::Remove(_) => {
                    let _ = tx.send(event.paths);
                }
                _ => {}
            }
        }
    })?;

    let repo_root = find_repo_root()?;
    watcher.watch(&repo_root, RecursiveMode::Recursive)?;

    println!("Watching {} for changes (Ctrl-C to stop)...", repo_root.display());

    // Debounce: collect all changes within 500ms before re-indexing
    loop {
        let first_paths = rx.recv()?;
        let mut changed_paths = first_paths;

        // Drain any additional events within the debounce window
        let deadline = std::time::Instant::now() + Duration::from_millis(500);
        while let Ok(paths) = rx.recv_timeout(deadline.saturating_duration_since(std::time::Instant::now())) {
            changed_paths.extend(paths);
        }

        // Filter to indexed file types only
        changed_paths.retain(|p| {
            LanguageId::from_path(p) != LanguageId::Unknown("".to_string())
                && !p.to_string_lossy().contains(".smart-grep")
        });

        if changed_paths.is_empty() { continue; }

        println!("Detected {} changed file(s), re-indexing...", changed_paths.len());
        let start = std::time::Instant::now();

        match incremental_index_files(&changed_paths, index_dir) {
            Ok(stats) => println!(
                "Re-indexed in {:.1}s: {} chunks updated, {} added, {} removed",
                start.elapsed().as_secs_f32(),
                stats.updated, stats.added, stats.removed
            ),
            Err(e) => eprintln!("Re-index error: {}", e),
        }
    }
}
```

---

## 58. Full Test Suite Reference

### 58.1 `tests/search_test.rs`

```rust
// tests/search_test.rs — full integration tests

use std::path::{Path, PathBuf};
use tempfile::TempDir;

/// Core test: semantic search finds payment code WITHOUT the word "payment"
#[test]
fn test_finds_payment_code_without_keyword() {
    let fixture = SmartGrepTestFixture::new();
    fixture.add_file("billing.js", BILLING_FIXTURE_JS);
    fixture.add_file("auth.js",    AUTH_FIXTURE_JS);
    fixture.add_file("retry.js",   RETRY_FIXTURE_JS);
    fixture.index();

    let results = fixture.search("payment error handling", 5);
    assert!(!results.is_empty(), "should find results");
    assert!(
        results[0].chunk.file_path.ends_with("billing.js"),
        "billing.js should be top result, got: {:?}", results[0].chunk.file_path
    );
    assert!(results[0].score > 0.70, "score should be > 0.70, got {}", results[0].score);

    // Ripgrep would return nothing — this is the core value proposition
    // (enforced by the test fixture NOT containing the literal word "payment")
    assert!(
        !BILLING_FIXTURE_JS.to_lowercase().contains("payment"),
        "test fixture must not contain the query keyword"
    );
}

/// Test: --like query-by-example finds cross-language analogues
#[test]
fn test_like_finds_cross_language_analogues() {
    let fixture = SmartGrepTestFixture::new();
    fixture.add_file("retry.js", RETRY_FIXTURE_JS);     // JS retry logic
    fixture.add_file("retry.py", RETRY_FIXTURE_PY);     // Python retry logic
    fixture.add_file("retry.rs", RETRY_FIXTURE_RS);     // Rust retry logic
    fixture.add_file("billing.js", BILLING_FIXTURE_JS); // Unrelated file (negative control)
    fixture.index();

    let snippet = r#"
        def retry_with_backoff(fn, max_attempts=3):
            for attempt in range(max_attempts):
                try:
                    return fn()
                except Exception:
                    time.sleep(2 ** attempt)
    "#;

    let results = fixture.search_like(snippet, 5);
    assert!(results.len() >= 2, "should find at least 2 retry analogues");

    // Both retry.js and retry.rs should be in top results
    let paths: Vec<&str> = results.iter()
        .map(|r| r.chunk.file_path.file_name().unwrap().to_str().unwrap())
        .collect();
    assert!(paths.iter().any(|p| *p == "retry.js"), "should find retry.js");
    assert!(paths.iter().any(|p| *p == "retry.rs"), "should find retry.rs");

    // billing.js should NOT be in top 3 (negative control)
    let top3_paths: Vec<&str> = paths.iter().take(3).copied().collect();
    assert!(!top3_paths.contains(&"billing.js"), "billing.js should not be in top 3");
}

/// Test: incremental re-index after file change
#[test]
fn test_incremental_reindex() {
    let fixture = SmartGrepTestFixture::new();
    fixture.add_file("auth.js", AUTH_FIXTURE_JS);
    fixture.index();

    let initial_results = fixture.search("token validation", 3);
    assert!(!initial_results.is_empty());

    // Modify the file — replace the function with unrelated content
    fixture.overwrite_file("auth.js", UNRELATED_FIXTURE_JS);
    fixture.reindex();  // incremental

    let updated_results = fixture.search("token validation", 3);
    // After replacing auth.js with unrelated content, token validation should score lower
    if !updated_results.is_empty() {
        assert!(
            updated_results[0].score < initial_results[0].score,
            "score should decrease after removing relevant code"
        );
    }
}

/// Test: --from reachability scoping
#[test]
fn test_from_scoping_restricts_results() {
    let fixture = SmartGrepTestFixture::new();
    fixture.add_file("checkout.rs",  CHECKOUT_FIXTURE_RS);   // calls billing + auth
    fixture.add_file("billing.rs",   BILLING_FIXTURE_RS);    // called from checkout
    fixture.add_file("admin.rs",     ADMIN_FIXTURE_RS);       // NOT called from checkout
    fixture.index();

    // Without --from: might get admin.rs results for "permission check"
    let all_results = fixture.search("permission check", 10);

    // With --from checkout: should only get results from checkout + billing path
    let scoped_results = fixture.search_from("permission check", "checkout.rs::handle_checkout", 10);

    // admin.rs results should NOT appear in scoped results
    let has_admin = scoped_results.iter().any(|r| r.chunk.file_path.ends_with("admin.rs"));
    assert!(!has_admin, "admin.rs should not appear in checkout-scoped search");
}

/// Test: confidence calibration warns on out-of-domain queries
#[test]
fn test_confidence_low_on_out_of_domain() {
    let fixture = SmartGrepTestFixture::new();
    fixture.add_file("billing.js", BILLING_FIXTURE_JS);
    fixture.index();

    // Build up confidence history with in-domain queries
    for _ in 0..25 {
        fixture.search("payment error", 3);
    }

    // Query for something completely out of domain
    let results = fixture.search_with_confidence("kubernetes pod scheduling", 3);
    if !results.is_empty() {
        assert!(
            results[0].confidence == ConfidenceLevel::Low,
            "out-of-domain query should return Low confidence"
        );
    }
}

/// Test: session context aggregation
#[test]
fn test_session_context_deduplicates() {
    let fixture = SmartGrepTestFixture::new();
    fixture.add_file("billing.js", BILLING_FIXTURE_JS);
    fixture.index();

    fixture.session_start("test-session");
    fixture.search("payment error");          // session recorded
    fixture.search("charge failure handler"); // session recorded
    fixture.search("billing exception");      // session recorded — likely overlaps

    let context = fixture.session_context("test-session", 8000);

    // Context should be deduplicated: same chunk appearing in multiple queries counted once
    let chunk_ids: Vec<&str> = context.iter().map(|r| r.chunk.chunk_id.as_str()).collect();
    let unique_ids: std::collections::HashSet<&str> = chunk_ids.iter().copied().collect();
    assert_eq!(chunk_ids.len(), unique_ids.len(), "session context should deduplicate chunks");
}
```

---

## 59. `docs/benchmark.md` Template

```markdown
# smart-grep Benchmark Results

## Test Environment
- Codebase: [repo name] (~N files, ~M lines)
- Hardware: [CPU, RAM]
- Model: Xenova/all-MiniLM-L6-v2 (384 dimensions)
- smart-grep version: 0.1.0

## Core Value Proposition: Queries Where ripgrep Returns Zero

| Query | smart-grep Top-1 | smart-grep Score | ripgrep results |
|---|---|---|---|
| "where is payment error handled?" | src/billing/stripe.rs::handle_charge_failure | 0.91 | 0 |
| "retry failed billing operations" | src/jobs/billing_job.rs::retry_charge | 0.88 | 0 |
| "access control checks" | src/auth/middleware.rs::authorize! | 0.85 | 0 |
| "database connection pool" | src/db/pool.rs::ConnectionPool | 0.87 | 0 |
| "webhook idempotency" | src/webhooks/idempotency.rs::check_key | 0.84 | 0 |

## Accuracy Metrics

| Metric | Result | Target |
|---|---|---|
| Top-1 accuracy (25 benchmark queries) | X% | > 80% |
| Top-5 recall (25 benchmark queries) | X% | > 95% |
| BM25 improvement over semantic-only | +X% on exact symbol queries | > 10% |

## Performance Metrics

| Operation | Measured | Target |
|---|---|---|
| Full index (50k-line repo) | Xs | < 90s |
| Query latency (p50) | Xms | < 100ms |
| Query latency (p99) | Xms | < 200ms |
| Incremental re-index (1 file change) | Xs | < 5s |
| HNSW rebuild (10k chunks) | Xs | < 5s |
```

---

*End of Comprehensive Implementation Plan — smart-grep v1*
*Total: 5,000+ lines · All sections backed by concrete implementation detail*

---

## 60. Dependency Versions & Compatibility Matrix

### 60.1 Pinned Dependency Reference

```toml
# Workspace-level pinned dependencies for reproducible builds

[workspace.dependencies]

# ─── Parsing ─────────────────────────────────────────────────────────────────
tree-sitter            = "0.22.6"
tree-sitter-javascript = "0.21.4"
tree-sitter-typescript = "0.21.2"
tree-sitter-rust       = "0.21.2"
tree-sitter-python     = "0.21.0"
tree-sitter-ruby       = "0.21.0"

# ─── Embedding ────────────────────────────────────────────────────────────────
fastembed = "3.8.0"
# fastembed bundles ort (ONNX Runtime) — no separate ort dependency needed

# ─── Vector Index ─────────────────────────────────────────────────────────────
usearch = "2.13.0"
# usearch provides HNSW with cosine, l2, hamming metrics

# ─── Storage ──────────────────────────────────────────────────────────────────
rusqlite = { version = "0.31.0", features = ["bundled", "backup"] }
# "bundled" = static SQLite 3.45+ — no system SQLite dependency
# "backup" = SQLite Online Backup API for snapshot.rs

# ─── File Discovery ───────────────────────────────────────────────────────────
ignore = "0.4.23"
# Respects .gitignore, .ignore, .smart-grep-ignore

# ─── CLI ─────────────────────────────────────────────────────────────────────
clap = { version = "4.5.0", features = ["derive", "env", "wrap_help"] }

# ─── Serialization ────────────────────────────────────────────────────────────
serde      = { version = "1.0.200", features = ["derive"] }
serde_json = "1.0.116"
toml       = "0.8.12"  # config file parsing

# ─── Error Handling ───────────────────────────────────────────────────────────
anyhow    = "1.0.82"
thiserror = "1.0.59"

# ─── Parallelism ──────────────────────────────────────────────────────────────
rayon = "1.10.0"

# ─── Hashing ──────────────────────────────────────────────────────────────────
blake3 = "1.5.1"

# ─── File Watching ────────────────────────────────────────────────────────────
notify = "6.1.1"

# ─── Logging / Tracing ───────────────────────────────────────────────────────
tracing            = "0.1.40"
tracing-subscriber = { version = "0.3.18", features = ["env-filter", "fmt"] }

# ─── Dev / Test ───────────────────────────────────────────────────────────────
tempfile  = "3.10.1"
criterion = { version = "0.5.1", features = ["html_reports"] }
```

### 60.2 Platform Support Matrix

| Platform | Status | Notes |
|---|---|---|
| macOS (Apple Silicon) | Primary dev target | Fastest embedding (Metal acceleration in future) |
| macOS (Intel) | Supported | Tested in CI |
| Linux x86_64 | Primary release target | Used in CI/CD pipelines |
| Linux arm64 | Supported | Raspberry Pi, GitHub Actions arm runners |
| Windows x86_64 | Supported | Requires MSVC toolchain |
| Windows arm64 | Best-effort | Not tested in CI for v1 |

### 60.3 Minimum System Requirements

| Resource | Minimum | Recommended |
|---|---|---|
| RAM | 512 MB | 2 GB |
| Disk (model cache) | 100 MB | 200 MB |
| Disk (index, 50k-line repo) | 30 MB | 50 MB |
| CPU cores | 1 | 4+ (rayon benefits from multiple cores) |
| Rust toolchain | 1.85+ | latest stable |

---

## 61. Error Types Per Crate

### 61.1 `crates/indexer/src/lib.rs`

```rust
// crates/indexer/src/lib.rs — typed errors

#[derive(Debug, thiserror::Error)]
pub enum IndexerError {
    #[error("Unsupported language for file: {path}")]
    UnsupportedLanguage { path: String },

    #[error("Parse error in {path}: tree-sitter returned None")]
    ParseFailed { path: String },

    #[error("Embedding failed: {source}")]
    EmbedFailed { #[from] source: anyhow::Error },

    #[error("File not readable: {path}: {source}")]
    IoError { path: String, #[source] source: std::io::Error },

    #[error("Chunk too large to embed ({lines} lines > max {max}): {symbol} in {path}")]
    ChunkTooLarge { lines: usize, max: usize, symbol: String, path: String },
}
```

### 61.2 `crates/store/src/lib.rs`

```rust
// crates/store/src/lib.rs — typed errors

#[derive(Debug, thiserror::Error)]
pub enum StoreError {
    #[error("Index database not found at {path}. Run 'smart-grep index' to create it.")]
    IndexNotFound { path: String },

    #[error("Schema version mismatch: expected {expected}, found {found}. Run 'smart-grep index --rebuild'.")]
    SchemaMismatch { expected: u32, found: u32 },

    #[error("Vector dimension mismatch: index has {stored}D, model produces {model}D. Run 'smart-grep index --rebuild'.")]
    DimensionMismatch { stored: usize, model: usize },

    #[error("Model changed: index was built with '{stored_model}', current model is '{current_model}'. Run 'smart-grep index --rebuild'.")]
    ModelMismatch { stored_model: String, current_model: String },

    #[error("SQLite error: {source}")]
    Sqlite { #[from] source: rusqlite::Error },

    #[error("HNSW index corrupted or missing. Run 'smart-grep index --rebuild'.")]
    HnswCorrupted,

    #[error("Snapshot '{ref_name}' not found. Run 'smart-grep snapshot list' to see available snapshots.")]
    SnapshotNotFound { ref_name: String },

    #[error("Session '{name}' not found. Run 'smart-grep session list' to see available sessions.")]
    SessionNotFound { name: String },
}
```

### 61.3 `crates/search/src/lib.rs`

```rust
// crates/search/src/lib.rs — typed errors

#[derive(Debug, thiserror::Error)]
pub enum SearchError {
    #[error("Entry point '{symbol}' not found in index. Run 'smart-grep index' and try again.")]
    ReachabilityEntryNotFound { symbol: String },

    #[error("--from BFS exceeded max depth {depth}. Use --from-depth to increase the limit.")]
    ReachabilityDepthExceeded { depth: u32 },

    #[error("No results above confidence threshold {threshold:.2}. Use --confidence off to see all results.")]
    LowConfidenceStrict { threshold: f32 },

    #[error("Budget too small: minimum useful budget is {min} tokens, requested {requested}.")]
    BudgetTooSmall { min: usize, requested: usize },

    #[error("Store error: {source}")]
    Store { #[from] source: crate::store::StoreError },
}
```

---

## 62. `CLAUDE.md` — smart-grep's Own Agent Instructions

This is the `CLAUDE.md` file that ships inside the smart-grep repository itself, instructing Claude Code how to navigate the smart-grep codebase:

```markdown
# smart-grep CLAUDE.md

## Project Structure
This is a Rust Cargo workspace. All source code is in `crates/`. Never look for
source code at the workspace root — it does not exist there.

## Build
```bash
cargo build                  # debug build
cargo build --release        # release build
cargo test                   # run all tests
cargo bench                  # run criterion benchmarks
```

## Searching This Codebase
Use smart-grep's own --self mode to search the smart-grep source code:
```bash
smart-grep "how does embedding work?" --self
smart-grep "BM25 scorer implementation" --self
smart-grep --like 'fn extract_chunks' --self
```

## Key Architectural Rules
1. `content` and `embedding_text` in CodeChunk are ALWAYS separate fields.
   Never conflate them. content = raw source for display. embedding_text = normalized for embedding.
2. All language-specific logic lives in `crates/indexer/src/languages/`.
   Adding a new language = new file there, register in `languages.rs`. Nothing else changes.
3. `schema.sql` is the single source of truth for the DB schema.
   Never ALTER TABLE directly — update schema.sql and increment schema_version in meta.
4. BM25 tokenizer MUST split camelCase and snake_case.
   'handlePaymentError' must produce tokens ['handle', 'payment', 'error'].
   This is the most important correctness property in `bm25.rs`.
5. ChunkExtractor::extract_chunks must NEVER panic.
   Return empty Vec on parse errors. Set parse_error = 1 in files table.

## Running the Test Fixtures
The test fixtures in test-fixtures/ are designed to NOT contain their keywords.
billing.js must not contain "payment". auth.js must not contain "authentication".
This is how we test the semantic search value proposition.

## Adding a New Language
1. Create `crates/indexer/src/languages/<lang>.rs`
2. Implement `ChunkExtractor` trait
3. Register in `crates/indexer/src/languages.rs` match arm
4. Add tree-sitter grammar to `Cargo.toml`
5. Add integration test in `tests/search_test.rs`
6. Add test fixture in `test-fixtures/<lang>.<ext>` WITHOUT the search keywords

## Performance Targets
- Query latency p50: < 100ms
- Full index 50k-line repo: < 90s
- Incremental re-index (1 file): < 5s
Run `cargo bench` to verify these are met before shipping.
```

---

## 63. Release Checklist

Before tagging a release:

```markdown
## Pre-Release Checklist

### Correctness
- [ ] All integration tests pass (`cargo test`)
- [ ] Benchmark queries meet Top-1 accuracy > 80%, Top-5 recall > 95%
- [ ] `--confidence warn` triggers correctly on out-of-domain queries
- [ ] Incremental re-index produces correct results after file changes
- [ ] `smart-grep doctor` reports all green on a fresh install
- [ ] `--self` mode resolves correctly on clean machine (no pre-existing index)
- [ ] MCP server mounts correctly in Claude Code with the reference config

### Performance
- [ ] `cargo bench` results meet all targets in benchmark.md
- [ ] Query latency p50 < 100ms on reference hardware
- [ ] Full index 50k-line repo < 90s

### Build
- [ ] `cargo build --release` succeeds with no warnings on Linux x86_64
- [ ] `cargo build --release` succeeds on macOS arm64
- [ ] Binary size within acceptable range (document in README)
- [ ] `cargo install smart-grep` succeeds from a clean environment

### Documentation
- [ ] README examples all work against --self index
- [ ] CLAUDE.md snippet in README is up to date
- [ ] docs/benchmark.md has current results
- [ ] CHANGELOG.md has entry for this release

### Distribution
- [ ] GitHub Release created with binaries for: linux-x86_64, linux-arm64, macos-arm64, macos-x86_64, windows-x86_64
- [ ] crates.io publish succeeds
- [ ] SHA256 checksums published alongside binaries
```

---

*End of Plan — smart-grep v1 Comprehensive Implementation Document*