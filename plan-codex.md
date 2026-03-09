# smart-grep — Technical Plan

## Vision
Build **smart-grep**, a semantic code search tool for local codebases that finds code by **meaning**, not exact keywords.

Unlike `grep` or `ripgrep`, which only match literal text or regex patterns, smart-grep should help developers answer questions like:

- "Where is payment error handled?"
- "Where do we retry failed billing operations?"
- "What code is responsible for access control checks?"

Even when the actual code uses different names such as:

- `Stripe::CardError`
- `BillingException`
- `handleChargeFailure()`
- `PaymentDeclinedError`

The core value is to return the **most semantically relevant code chunks** with minimal noise and minimal token usage for coding agents.

---

## Problem Statement
Traditional search tools such as `grep` and `ripgrep` are optimized for **exact string** and **regex** matching.

That breaks down when:

- symbols have been renamed,
- different languages express the same concept differently,
- intent is described in natural language rather than code syntax,
- code agents must scan too many files to find the right answer.

This leads to:

- missed results,
- high token consumption,
- repeated full-repo scans,
- weak support for cross-language discovery.

smart-grep solves this by building a **local semantic index** over code structure and using **vector similarity search** to retrieve ranked matches instantly.

---

## Product Goals

### Primary goals
1. Provide semantic code search over a local repository.
2. Work fully offline during search.
3. Support multiple programming languages.
4. Return precise ranked results with file and line range.
5. Keep the developer workflow fast enough for terminal usage and coding agents.

### Secondary goals
1. Minimize token cost when used with LLM tools.
2. Support incremental indexing.
3. Keep installation simple with a single Rust binary.
4. Make indexing transparent and debuggable.

### Non-goals for v1
- Full IDE replacement.
- Full code intelligence like type resolution across all languages.
- Remote hosted search service.
- Perfect semantic understanding of every framework-specific pattern.

---

## Why Rust
Rust is a strong fit because smart-grep needs to be:

- **fast** for indexing and search,
- **memory-efficient** for local development machines,
- **safe** for concurrent parsing and indexing,
- **portable** as a single CLI binary,
- **well-suited** for systems integration with SQLite, tree-sitter, and ANN libraries.

Rust also makes it easier to build:

- parallel file walkers,
- AST chunking pipelines,
- efficient embedding/index storage layers,
- a reliable CLI users can trust on large repositories.

---

## High-Level Architecture

```text
                 +----------------------+
                 |   smart-grep CLI     |
                 +----------+-----------+
                            |
               +------------+------------+
               |                         |
               v                         v
      +------------------+      +------------------+
      |   Index Command   |      |  Search Command  |
      +--------+---------+      +--------+---------+
               |                         |
               v                         v
      +------------------+      +------------------+
      | File Discovery    |      | Query Embedding  |
      +--------+---------+      +--------+---------+
               |                         |
               v                         v
      +------------------+      +------------------+
      | tree-sitter AST   |      | Vector Retrieval |
      | chunk extraction  |      | (HNSW cosine)    |
      +--------+---------+      +--------+---------+
               |                         |
               v                         v
      +------------------+      +------------------+
      | Embedding Engine  |      | Ranked Results   |
      | (local)           |      | file + lines     |
      +--------+---------+      +------------------+
               |
               v
      +------------------------------+
      | SQLite + HNSW index storage  |
      | .smart-grep/index.db         |
      +------------------------------+
```

---

## Core Workflow

### 1. Index phase
Run once initially, then incrementally.

Pipeline:
1. Walk repository files.
2. Detect supported language by extension or parser mapping.
3. Parse files using tree-sitter.
4. Extract meaningful AST-based chunks.
5. Normalize chunk text and metadata.
6. Generate embeddings locally.
7. Store chunks, metadata, and vectors in SQLite.
8. Update HNSW index for fast nearest-neighbor search.

### 2. Search phase
Run on every query.

Pipeline:
1. Embed the natural language query.
2. Search nearest vectors by cosine similarity.
3. Re-rank or filter if needed.
4. Return top-N results with:
   - file path,
   - line range,
   - score,
   - chunk preview,
   - symbol/context metadata.

---

## Technical Design

## 1. Repository Layout

Suggested Rust workspace structure:

```text
smart-grep/
├── Cargo.toml
├── crates/
│   ├── cli/
│   ├── core/
│   ├── parser/
│   ├── chunker/
│   ├── embed/
│   ├── index/
│   ├── storage/
│   └── search/
├── tree-sitter-grammars/
├── docs/
└── examples/
```

### Module responsibilities
- **cli**: command parsing, user-facing output, config loading.
- **core**: shared types, errors, traits, config, result models.
- **parser**: language detection and tree-sitter integration.
- **chunker**: AST-to-chunk extraction logic.
- **embed**: local embedding provider abstraction.
- **storage**: SQLite schema and persistence.
- **index**: HNSW vector index management.
- **search**: query execution, ranking, filtering.

---

## 2. Chunking Strategy
Semantic search quality depends heavily on chunk quality.

### Recommended unit of indexing
Index code as **semantic chunks**, not arbitrary fixed-size text windows.

Good chunk candidates:
- function definitions,
- method definitions,
- class/struct blocks,
- trait/interface methods,
- error handling branches,
- route/controller actions,
- SQL/query builder blocks,
- config sections when meaningful.

### Chunk metadata
Each chunk should include:
- `chunk_id`
- `file_path`
- `language`
- `start_line`
- `end_line`
- `symbol_name` if available
- `symbol_kind`
- `parent_symbol`
- `content`
- `content_hash`
- `mtime`

### Chunk text format for embeddings
Raw source alone is often not enough. Build a normalized text representation such as:

```text
Language: Rust
Symbol: handle_payment_error
Kind: function
Path: src/billing/errors.rs
Code:
...
```

This gives the embedding model more semantic context.

### Important design choice
Preserve both:
1. **raw source** for display,
2. **normalized embedding text** for vector generation.

---

## 3. Parsing Layer
Use `tree-sitter` for syntax-aware parsing.

### Responsibilities
- map file extensions to grammars,
- parse source into AST,
- recover gracefully from syntax errors,
- expose language-specific node extractors.

### v1 supported languages
Start with a focused set:
- Rust
- TypeScript / JavaScript
- Python
- Ruby
- Go

This is enough to prove cross-language value while keeping parser complexity manageable.

### Parser abstraction
Define a trait like:

```rust
pub trait ChunkExtractor {
    fn language(&self) -> LanguageId;
    fn extract_chunks(&self, path: &Path, source: &str) -> Vec<CodeChunk>;
}
```

Each language can implement its own extraction heuristics on top of tree-sitter node kinds.

---

## 4. Embedding Layer
Use a local embedding backend compatible with offline execution.

### Requirements
- no API calls during search,
- deterministic local inference,
- acceptable latency on developer machines,
- small enough model footprint for local use.

### Design
Hide the actual model behind a provider trait:

```rust
pub trait Embedder {
    fn embed_texts(&self, texts: &[String]) -> anyhow::Result<Vec<Vec<f32>>>;
    fn embed_query(&self, text: &str) -> anyhow::Result<Vec<f32>>;
    fn dimension(&self) -> usize;
}
```

This keeps the project flexible if you change models later.

### Likely v1 choice
Use `fastembed` locally for both indexing and query embedding.

---

## 5. Storage Design
Store persistent data in `.smart-grep/index.db`.

### Why SQLite
- embedded and portable,
- easy to inspect and debug,
- transactional updates,
- works well with local CLI tooling,
- sufficient for v1 repository-scale workloads.

### Suggested schema

#### `files`
```sql
CREATE TABLE files (
  id INTEGER PRIMARY KEY,
  path TEXT UNIQUE NOT NULL,
  language TEXT,
  mtime INTEGER NOT NULL,
  size_bytes INTEGER NOT NULL,
  content_hash TEXT NOT NULL
);
```

#### `chunks`
```sql
CREATE TABLE chunks (
  id INTEGER PRIMARY KEY,
  file_id INTEGER NOT NULL,
  chunk_key TEXT UNIQUE NOT NULL,
  symbol_name TEXT,
  symbol_kind TEXT,
  parent_symbol TEXT,
  start_line INTEGER NOT NULL,
  end_line INTEGER NOT NULL,
  content TEXT NOT NULL,
  embedding_text TEXT NOT NULL,
  content_hash TEXT NOT NULL,
  FOREIGN KEY(file_id) REFERENCES files(id)
);
```

#### `embeddings`
Options:
1. store vectors directly in SQLite as blobs,
2. store vectors externally and reference them.

For v1, storing vectors in SQLite is acceptable if dimensions are moderate.

```sql
CREATE TABLE embeddings (
  chunk_id INTEGER PRIMARY KEY,
  dim INTEGER NOT NULL,
  vector BLOB NOT NULL,
  FOREIGN KEY(chunk_id) REFERENCES chunks(id)
);
```

#### `meta`
```sql
CREATE TABLE meta (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
```

Use `meta` for:
- embedding model name,
- vector dimension,
- schema version,
- last index timestamp.

---

## 6. Vector Search Layer
Use an approximate nearest neighbor index with HNSW.

### Why HNSW
- fast top-K retrieval,
- strong recall/latency tradeoff,
- well-known for vector search,
- practical for local repository-sized indexes.

### Search flow
1. load HNSW index metadata,
2. embed query,
3. retrieve top-K candidate chunk IDs,
4. join with SQLite metadata,
5. optionally re-rank by heuristics.

### Re-ranking ideas for v1.1
Combine vector similarity with lightweight boosts for:
- symbol name matches,
- same file proximity,
- error-handling constructs,
- recent edits,
- language preference.

Basic score formula:

```text
final_score = semantic_score + lexical_boost + structural_boost
```

For v1, semantic score alone is enough to ship.

---

## 7. Incremental Indexing
Incremental indexing is a major differentiator.

### Strategy
For each file:
- compare stored mtime and content hash,
- skip unchanged files,
- re-parse changed files,
- delete stale chunks,
- insert updated chunks and embeddings,
- update HNSW entries.

### File states
- new file,
- changed file,
- deleted file,
- unchanged file.

### Practical note
A simple but reliable v1 path is:
1. detect changed files,
2. remove their prior chunk rows,
3. reinsert them,
4. rebuild HNSW if partial updates are hard.

That is less elegant than true online updates, but much easier to make correct.

---

## 8. CLI Design
The CLI should feel familiar to `grep` users while exposing semantic features clearly.

### Commands

```bash
smart-grep index
smart-grep search "where is payment error handled?"
smart-grep search "retry failed billing" --top-k 10
smart-grep search "auth check before deleting user" --json
smart-grep doctor
smart-grep stats
```

### Recommended options
- `--path <repo>`
- `--top-k <n>`
- `--language <lang>`
- `--json`
- `--rebuild`
- `--verbose`

### Output format
Human-friendly default:

```text
1. src/billing/stripe.rs:120-148  score=0.91
   fn handle_charge_failure(...) { ... }

2. app/services/payments.rb:44-63  score=0.88
   rescue Stripe::CardError => e
```

---

## 9. Configuration
Support a repo-local config file later, but keep v1 simple.

Potential config path:

```text
.smart-grep/config.toml
```

Example future fields:

```toml
[embedding]
model = "fastembed-default"

[index]
include = ["src", "app", "lib"]
exclude = ["node_modules", "dist", "vendor", "target"]

[search]
default_top_k = 8
```

---

## 10. Ignoring and Filtering
Respect common ignore patterns.

### Default exclusions
- `.git/`
- `node_modules/`
- `target/`
- `dist/`
- `build/`
- `vendor/`
- generated files
- lockfiles
- minified assets

### Goal
Keep the semantic index focused on source code that developers actually want to search.

---

## 11. Observability and Debuggability
A semantic tool becomes hard to trust if it feels opaque.

### Add visibility features
- `smart-grep stats` for indexed file/chunk counts,
- `smart-grep doctor` for parser/model/index health,
- optional debug output showing why a result matched,
- command to inspect a stored chunk by ID.

This is especially important while tuning chunking quality.

---

## 12. Error Handling
The system should fail clearly and locally.

### Common failure modes
- unsupported language grammar,
- broken parser setup,
- corrupted SQLite DB,
- embedding dimension mismatch,
- stale or missing HNSW index,
- unreadable files.

### Approach
- use typed errors with context,
- return actionable CLI messages,
- preserve partial progress where safe,
- validate index metadata before search.

---

## MVP Scope

### What ships in v1
- local CLI binary,
- indexing for a subset of languages,
- tree-sitter AST chunk extraction,
- local embeddings,
- SQLite-backed metadata storage,
- HNSW top-K semantic search,
- incremental file detection,
- readable terminal output.

### What can wait
- hybrid lexical + semantic ranking,
- watch mode,
- IDE integration,
- query expansion,
- repository-level summaries,
- symbol graph relationships,
- team-shared indexes.

---

## Implementation Phases

## Phase 1 — Foundations
Goal: create a reliable end-to-end vertical slice.

Deliverables:
- Rust workspace setup,
- config and error model,
- file walker,
- tree-sitter integration for one language,
- basic chunk extraction,
- local embedding prototype,
- SQLite persistence.

Success criteria:
- can index a small repo and inspect stored chunks.

## Phase 2 — Searchable MVP
Goal: make search useful.

Deliverables:
- HNSW integration,
- query embedding,
- top-K retrieval,
- CLI `search` command,
- result formatting.

Success criteria:
- natural language query returns relevant file + line results.

## Phase 3 — Incremental Updates
Goal: make it practical for daily use.

Deliverables:
- file hashing / mtime checks,
- changed-file reindexing,
- deleted-file cleanup,
- index consistency checks.

Success criteria:
- reindex runs much faster after small code changes.

## Phase 4 — Multi-language Support
Goal: prove cross-language value.

Deliverables:
- additional tree-sitter grammars,
- per-language chunk extraction rules,
- language filtering.

Success criteria:
- semantic search works across several real-world repos.

## Phase 5 — Relevance Tuning
Goal: improve trust and precision.

Deliverables:
- result evaluation set,
- ranking heuristics,
- chunk formatting improvements,
- debug/explain mode.

Success criteria:
- quality is consistently better than grep for intent-style queries.

---

## Example Search Cases
These should be used as acceptance tests.

### Billing / errors
Query: `where is payment error handled?`
Should surface:
- Stripe card exceptions,
- decline handlers,
- billing retry logic,
- refund failure handling.

### Authentication
Query: `where do we verify admin permissions?`
Should surface:
- guard clauses,
- middleware,
- role checks,
- policy methods.

### Retry behavior
Query: `where are failed jobs retried?`
Should surface:
- retry loops,
- backoff logic,
- queue retry config,
- transient error handlers.

---

## Evaluation Plan
Quality must be measured, not assumed.

### Build a small benchmark set
Create 25–50 natural-language queries paired with expected code locations.

Measure:
- **Top-1 accuracy**
- **Top-5 recall**
- **Index time**
- **Search latency**
- **Incremental reindex latency**

### Compare against baseline
Baseline methods:
- `grep`
- `ripgrep`
- optional hybrid lexical search

Success target for MVP:
- meaningfully higher recall than grep on intent-based queries,
- search latency that feels near-instant for local repo usage.

---

## Risks and Mitigations

### Risk: bad chunking hurts relevance
Mitigation:
- use AST-aware chunks,
- keep chunk sizes bounded,
- iterate with benchmark queries.

### Risk: local embeddings are too slow
Mitigation:
- batch during indexing,
- cache aggressively,
- use a smaller default model.

### Risk: incremental HNSW updates are complex
Mitigation:
- rebuild vector index when needed in v1,
- optimize later once correctness is stable.

### Risk: cross-language support becomes fragile
Mitigation:
- start with a few languages,
- use extractor traits and language-specific tests.

### Risk: users do not trust semantic matches
Mitigation:
- always show file path, lines, preview, and score,
- add explain/debug mode later.

---

## Recommended Crates to Explore
These are candidates, not commitments.

- `clap` for CLI
- `walkdir` or `ignore` for repository traversal
- `tree-sitter` for parsing
- `rusqlite` for SQLite access
- `serde` / `serde_json` for structured output
- `rayon` for parallel indexing
- `anyhow` and `thiserror` for error handling
- `blake3` for hashing
- `fastembed` for local embeddings

For HNSW, choose a Rust library that is actively maintained and can handle local ANN workloads cleanly.

---

## Suggested MVP Milestone Definition
The MVP is successful when a developer can:

1. install one Rust CLI binary,
2. run `smart-grep index` in a repo,
3. run a natural-language search query,
4. get relevant ranked code locations faster and with fewer misses than grep.

---

## Future Extensions
After MVP, strong follow-up ideas include:
- hybrid lexical + semantic ranking,
- editor integrations,
- file-level summaries,
- symbol relationship graphs,
- watch mode for background index refresh,
- query history and saved searches,
- agent-friendly output modes.

---

## Final Recommendation
Build smart-grep as a **Rust-first local CLI** with a narrow, reliable MVP:

- AST-based chunking via tree-sitter,
- local embeddings through a pluggable provider,
- SQLite persistence in `.smart-grep/index.db`,
- HNSW-based top-K search,
- incremental indexing that prioritizes correctness over sophistication.

The key to success is not only vector search. It is the combination of:
1. good code chunking,
2. strong metadata,
3. fast local storage,
4. clear CLI UX,
5. a benchmark suite that proves semantic value over grep.

That combination gives smart-grep a credible path to becoming a practical semantic search layer for developers and coding agents.

