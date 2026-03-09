# PLAN: smart-grep

## Vision

Build **smart-grep**, a semantic code search tool for local codebases that finds code by **meaning**, not exact keywords.

Unlike `grep` or `ripgrep`, which only match literal text or regex patterns, smart-grep helps developers answer questions like:

- "Where is payment error handled?"
- "Where do we retry failed billing operations?"
- "What code is responsible for access control checks?"

Even when the actual code uses completely different names such as:

- `Stripe::CardError`
- `BillingException`
- `handleChargeFailure()`
- `PaymentDeclinedError`

The core value proposition: return the **most semantically relevant code chunks** with minimal noise and minimal token usage — for both human developers and coding agents like Claude Code.

Beyond search, smart-grep is a **semantic understanding layer** for codebases: it can explain what a codebase is about, detect conceptual staleness after refactors, find meaning-level changes across git history, surface naming inconsistencies and duplicate logic, and serve as a first-class context assembly tool for LLM agents.

---

## Problem Statement

Traditional search tools such as `grep` and `ripgrep` are optimized for **exact string** and **regex** matching. That breaks down when:

- symbols have been renamed or abbreviated,
- different languages express the same concept differently,
- intent is described in natural language rather than code syntax,
- coding agents must scan too many files to find the right answer.

This leads to: missed results, high token consumption, repeated full-repo scans, and weak cross-language discovery.

smart-grep solves this by building a **local semantic index** over code structure and using **vector similarity search** to retrieve ranked matches instantly — with zero API calls at search time.

---

## Product Goals

### Primary
1. Semantic code search over a local repository — no internet required.
2. Support multiple programming languages via tree-sitter.
3. Return precise ranked results with file path, line range, and score.
4. Fast enough for terminal use and coding agents.

### Secondary
1. Minimize token cost when used with LLM tools.
2. Incremental indexing — re-index only changed files.
3. Single Rust binary, `cargo install`-able.
4. Transparent and debuggable index.
5. Native MCP server mode for direct agent integration.
6. Context assembly for LLM prompt budgets.
7. Passive codebase quality intelligence (naming consistency, duplicate logic, test coverage gaps).
8. Confidence-aware results — knows when to say "I don't know."

### Non-goals for v1
- Full IDE replacement or LSP server.
- Full type resolution across languages.
- Remote hosted search service.
- Perfect understanding of every framework-specific pattern.

---

## Why Rust

- **Fast** for indexing and search — parallel file walking with `rayon`
- **Memory-efficient** for developer laptops
- **Safe** for concurrent parsing and indexing
- **Portable** as a single statically-linked CLI binary
- **Ecosystem fit**: tree-sitter, SQLite, and ANN libraries all have solid Rust bindings

---

## Why Workspace + `crates/` (not `src/`)

Popular Rust CLIs like ripgrep, fd, and bat all use this pattern. Ripgrep specifically uses `crates/core/main.rs` as its entry point, not `src/main.rs`.

- Workspace root holds config, docs, poc, and tests — **no source code**
- Each concern (indexing, storage, search) is an independently testable crate
- `src/` still exists but only **inside each sub-crate**, never at the root
- Makes it easy to later publish `crates/indexer` as a standalone library

---

## High-Level Architecture

```text
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

---

## File Structure

```
smart-grep/
|-- Cargo.toml              # workspace root -- no source code here
|-- Cargo.lock
|-- README.md
|-- POC.md
|-- PLAN.md
|-- .gitignore              # includes .smart-grep/
|
|-- crates/
|   |-- core/               # binary entry point
|   |   |-- Cargo.toml
|   |   `-- main.rs         # CLI: index / search / diff / map / lint / session / serve
|   |
|   |-- indexer/            # tree-sitter chunking + local embedding
|   |   |-- Cargo.toml
|   |   |-- lib.rs
|   |   |-- chunker.rs          # AST-based code chunking (not line windows)
|   |   |-- embedder.rs         # Embedder trait + fastembed-rs impl
|   |   |-- embedding_text.rs   # normalized text formatter (Language/Symbol/Kind/Path/Code)
|   |   |-- graph.rs            # call/reference graph extraction from AST
|   |   |-- structural.rs       # AST shape fingerprinting for structural pattern search
|   |   |-- synonym_map.rs      # codebase-derived query expansion map
|   |   |-- project_detect.rs   # framework detection for auto-config
|   |   |-- languages.rs        # language dispatch
|   |   `-- languages/
|   |       |-- javascript.rs
|   |       |-- typescript.rs
|   |       |-- rust.rs
|   |       |-- python.rs
|   |       `-- ruby.rs
|   |
|   |-- store/              # sqlite + usearch HNSW index
|   |   |-- Cargo.toml
|   |   |-- schema.sql          # canonical DDL -- source of truth for migrations
|   |   |-- lib.rs
|   |   |-- db.rs               # rusqlite: chunk metadata + mtime + graph edges
|   |   |-- snapshot.rs         # index snapshot serialization/restore for time-travel
|   |   `-- index.rs            # usearch HNSW: vector storage + ANN search
|   |
|   `-- search/             # ranking and result formatting
|       |-- Cargo.toml
|       |-- lib.rs
|       |-- rank.rs             # hybrid scoring: HNSW cosine + BM25
|       |-- bm25.rs             # pure-Rust BM25 over SQLite chunk content
|       |-- expand.rs           # contextual expansion via call graph
|       |-- reachability.rs     # call-graph BFS for --from <entry> scoping
|       |-- budget.rs           # token-budget-aware result selection + truncation
|       |-- cluster.rs          # k-means clustering for codebase map
|       |-- session.rs          # session tracking, dedup, cumulative context
|       |-- lint.rs             # naming inconsistency, duplicate logic, orphan detection
|       |-- coverage.rs         # semantic test coverage mapping
|       `-- confidence.rs       # score calibration, low-confidence detection
|
|-- docs/
|   |-- architecture.md
|   `-- benchmark.md            # query benchmark results vs ripgrep
|
|-- poc/
|   |-- package.json
|   `-- index.js            # Node.js POC -- validate concept before Rust
|
|-- test-fixtures/
|   |-- billing.js          # payment handling code without the word "payment"
|   |-- auth.js             # authentication code without "authentication"
|   `-- retry.js            # retry/backoff logic without the word "retry"
|
|-- tests/
|   |-- search_test.rs          # integration: index -> search round-trip
|   |-- incremental_test.rs     # index -> mutate file -> re-index -> verify
|   |-- hybrid_rank_test.rs     # BM25 + semantic scoring correctness
|   |-- expansion_test.rs       # call graph contextual expansion
|   |-- reachability_test.rs    # --from scoping correctness
|   |-- session_test.rs         # session accumulation and dedup
|   `-- confidence_test.rs      # low-confidence detection and calibration
|
`-- benches/
    `-- index_bench.rs
```

### Root `Cargo.toml`

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

[[bin]]
name = "smart-grep"
path = "crates/core/main.rs"

[workspace.dependencies]
tree-sitter = "0.22"
tree-sitter-javascript = "0.21"
tree-sitter-typescript = "0.21"
tree-sitter-rust = "0.21"
tree-sitter-python = "0.21"
tree-sitter-ruby = "0.21"
fastembed = "3"
usearch = "2"
rusqlite = { version = "0.31", features = ["bundled"] }
ignore = "0.4"
clap = { version = "4", features = ["derive"] }
serde_json = "1"
serde = { version = "1", features = ["derive"] }
anyhow = "1"
thiserror = "1"
rayon = "1"
blake3 = "1"
notify = "6"
```

---

## Core Design: Chunking Strategy

Search quality is almost entirely determined by chunk quality. This is the most important design decision in the project.

### Unit of indexing

Index code as **semantic chunks** — AST-defined units, not arbitrary fixed-size text windows. Good chunk candidates:
- function and method definitions
- class and struct blocks
- trait/interface declarations
- error handling branches (`catch`, `rescue`, `match` arms on error types)
- route/controller actions
- SQL or query-builder blocks
- config sections when meaningful

### Chunk size bounds

Enforce min/max token bounds to prevent degenerate chunks:
- **min**: skip trivial one-liners (getters, empty stubs) — configurable, default ~5 lines
- **max**: split overly large functions at logical sub-blocks — default ~80 lines

### Chunk metadata fields

Every `CodeChunk` struct carries:

```rust
pub struct CodeChunk {
    pub chunk_id:        String,          // blake3 of file_path + symbol_name + start_line
    pub file_path:       PathBuf,
    pub language:        LanguageId,
    pub start_line:      u32,
    pub end_line:        u32,
    pub symbol_name:     Option<String>,
    pub symbol_kind:     Option<String>,  // "function" | "method" | "class" | "impl" ...
    pub parent_symbol:   Option<String>,  // enclosing class/impl name if applicable
    pub is_test:         bool,            // true if in test/ *_test.* *_spec.* spec/ etc.
    pub content:         String,          // raw source -- display only
    pub embedding_text:  String,          // normalized text -- embedding only
    pub structural_fp:   String,          // AST shape fingerprint -- structural search
    pub content_hash:    String,          // blake3 of content -- drives incremental check
    pub prev_embedding:  Option<Vec<f32>>,// prior embedding snapshot -- drives stale detection
}
```

### Embedding text format

Do not embed raw source directly. Build a normalized text header that gives the model semantic context:

```text
Language: Rust
Symbol: handle_payment_error
Kind: function
Path: src/billing/errors.rs
Code:
fn handle_payment_error(err: &PaymentError) -> Response {
    ...
}
```

This consistently outperforms raw source embedding, especially for short functions where the symbol name carries most of the meaning.

**Key rule:** always preserve both `content` (raw source, for display) and `embedding_text` (normalized, for vector generation) as separate fields. Never conflate them.

---

## Core Design: Parsing Layer

Use `tree-sitter` for syntax-aware AST parsing.

### `ChunkExtractor` trait

```rust
pub trait ChunkExtractor: Send + Sync {
    fn language(&self) -> LanguageId;
    fn extract_chunks(&self, path: &Path, source: &str) -> Vec<CodeChunk>;
    fn extract_calls(&self, path: &Path, source: &str) -> Vec<CallEdge>;
    fn extract_structural_fp(&self, node: &Node) -> String; // for structural pattern search
}
```

Each language implements its own extractor on top of tree-sitter node kinds. During chunk extraction, also extract **call edges** for the relationship graph and **structural fingerprints** for pattern search. Gracefully skip files with parse errors rather than crashing; store a `parse_error` flag in the `files` table.

### v1 supported languages

Start focused, prove cross-language value:
- Rust, TypeScript, JavaScript, Python, Ruby

(Go can be added in Phase 4 without structural changes.)

---

## Core Design: Embedding Layer

### `Embedder` trait

```rust
pub trait Embedder: Send + Sync {
    fn embed_texts(&self, texts: &[String]) -> anyhow::Result<Vec<Vec<f32>>>;
    fn embed_query(&self, text: &str) -> anyhow::Result<Vec<f32>>;
    fn dimension(&self) -> usize;
}
```

Hiding fastembed behind this trait means the model can be swapped later (e.g. a code-specific model like `nomic-embed-code`) without touching any other crate.

### v1 model
`fastembed-rs` with `Xenova/all-MiniLM-L6-v2` (384 dimensions). Runs fully offline via local ONNX. No API key. Batch embedding during indexing, single embed at query time.

---

## Core Design: Storage Schema

Persistent data lives in `.smart-grep/index.db`. All DDL is canonical in `crates/store/schema.sql`.

```sql
CREATE TABLE files (
  id           INTEGER PRIMARY KEY,
  path         TEXT    UNIQUE NOT NULL,
  language     TEXT,
  mtime        INTEGER NOT NULL,
  size_bytes   INTEGER NOT NULL,
  content_hash TEXT    NOT NULL,
  parse_error  INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE chunks (
  id             INTEGER PRIMARY KEY,
  file_id        INTEGER NOT NULL,
  chunk_key      TEXT    UNIQUE NOT NULL,
  symbol_name    TEXT,
  symbol_kind    TEXT,
  parent_symbol  TEXT,
  is_test        INTEGER NOT NULL DEFAULT 0,  -- 1 if in test file/directory
  start_line     INTEGER NOT NULL,
  end_line       INTEGER NOT NULL,
  content        TEXT    NOT NULL,
  embedding_text TEXT    NOT NULL,   -- stored separately for debuggability
  structural_fp  TEXT,               -- AST shape fingerprint for structural search
  content_hash   TEXT    NOT NULL,
  FOREIGN KEY(file_id) REFERENCES files(id) ON DELETE CASCADE
);

CREATE TABLE embeddings (
  chunk_id    INTEGER PRIMARY KEY,
  dim         INTEGER NOT NULL,
  vector      BLOB    NOT NULL,
  prev_vector BLOB,               -- snapshot of prior embedding; drives stale detection
  FOREIGN KEY(chunk_id) REFERENCES chunks(id) ON DELETE CASCADE
);

-- Call/reference graph
CREATE TABLE call_edges (
  id          INTEGER PRIMARY KEY,
  caller_id   INTEGER NOT NULL,
  callee_name TEXT    NOT NULL,
  callee_id   INTEGER,            -- chunk_id if callee is indexed, else NULL
  FOREIGN KEY(caller_id) REFERENCES chunks(id) ON DELETE CASCADE
);

-- Codebase-derived synonym map
CREATE TABLE synonyms (
  concept TEXT NOT NULL,
  symbol  TEXT NOT NULL,
  weight  REAL NOT NULL DEFAULT 1.0,
  PRIMARY KEY(concept, symbol)
);

-- Index snapshots for time-travel search
CREATE TABLE snapshots (
  id         INTEGER PRIMARY KEY,
  ref_name   TEXT    NOT NULL,   -- e.g. "HEAD~1", "v2.1.0", "pre-refactor"
  created_at INTEGER NOT NULL,
  db_path    TEXT    NOT NULL    -- path under .smart-grep/snapshots/
);

-- Session tracking: queries + result chunk IDs per named session
CREATE TABLE sessions (
  id         INTEGER PRIMARY KEY,
  name       TEXT    NOT NULL,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);

CREATE TABLE session_results (
  id         INTEGER PRIMARY KEY,
  session_id INTEGER NOT NULL,
  query      TEXT    NOT NULL,
  chunk_id   INTEGER NOT NULL,
  score      REAL    NOT NULL,
  queried_at INTEGER NOT NULL,
  FOREIGN KEY(session_id) REFERENCES sessions(id) ON DELETE CASCADE
);

-- Confidence calibration: rolling window of historical top-1 scores
CREATE TABLE query_scores (
  id         INTEGER PRIMARY KEY,
  query_hash TEXT    NOT NULL,
  top1_score REAL    NOT NULL,
  queried_at INTEGER NOT NULL
);

-- Stores model name, vector dimension, schema version, last index timestamp
CREATE TABLE meta (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
```

`ON DELETE CASCADE` on all FK relationships ensures all dependent rows are automatically cleaned when a file row is deleted — essential for incremental correctness.

---

## Core Design: Hybrid Search (BM25 + HNSW)

Search quality from semantic models alone has a known failure mode: exact symbol names. A user typing `handleChargeFailure` as a query should score that function at rank 1 — but embedding similarity is fuzzy and may not guarantee it. The solution is **hybrid retrieval**, standard in production RAG systems.

### Algorithm

1. HNSW retrieves top-50 semantic candidates (large pool to re-rank from)
2. BM25 scores all 50 candidates using the `content` column (pure TF-IDF, ~150 lines of Rust)
3. Final score combines both:

```
final_score = alpha * semantic_score + (1 - alpha) * bm25_score
```

`alpha` defaults to `0.7` (heavily semantic) but is tunable via `.smart-grep/config.toml`.

### Implementation note
BM25 runs purely over the already-stored `content` column in SQLite. No new dependencies, no new index structure. Pure Rust implementation in `crates/search/bm25.rs`.

---

## Core Design: Query Expansion via Synonym Map

Before embedding a query, run it through a codebase-derived **synonym expander**.

During indexing, `crates/indexer/synonym_map.rs` builds a lightweight concept->symbol correlation map: for each pair of symbols that co-occur frequently in the same files/chunks, record them as related. This builds entries like `"payment" -> ["stripe", "charge", "billing", "invoice", "card"]` — derived entirely from the repo's own naming patterns, with no external data.

At query time, the top-5 most correlated symbols are appended to the query string before embedding:

```
user query:    "where is payment error handled?"
expanded:      "where is payment error handled? stripe charge billing invoice card"
```

This is fully offline, costs one extra embed call, and dramatically improves recall on repos where naming is idiosyncratic. It is essentially a free, zero-shot domain adaptation of the embedding model to each specific codebase.

---

## Core Design: Query-by-Example (`--like`)

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

# Also works with stdin -- composable with shell workflows
cat src/billing/stripe.py | smart-grep --like -
```

The snippet runs through the same `embedding_text` normalizer and embeds directly — no natural-language-to-code translation gap, because the query *is* code. This is the right interface for refactoring workflows: the most common case where you need to find similar code is when you are already looking at an instance of it.

**Use cases:**
- "Find all places we use this same retry pattern"
- "Find the Rust equivalent of this Python function I just wrote"
- "Find similar error handlers to this one I am refactoring"
- Find cross-language analogues of a pattern across a polyglot repo

**Implementation:** the code snippet is normalized through `embedding_text.rs` as if it were a chunk being indexed, then embedded with `Embedder::embed_query`. Zero additional infrastructure beyond what already exists.

---

## Core Design: Relationship Graph + Contextual Expansion

During indexing, `crates/indexer/graph.rs` extracts **call edges** from the AST — function call sites and symbol references — stored in the `call_edges` table. This drives three features:

### `--expand` flag

Automatically include the **callers and callees** of each top result:

```bash
smart-grep "handle charge failure" --expand
# Returns: handle_charge_failure + its callers + its direct callees
```

### `--from <entry>` — Reachability-Scoped Search

Search only the **call-graph-reachable subset** from a given entry point. Performs BFS over `call_edges` from the specified symbol and restricts HNSW search to that subgraph:

```bash
smart-grep "payment error handling" --from src/api/checkout.rs::handle_checkout
# Only searches code reachable from handle_checkout
```

Solves a real problem in large monorepos: a query for "authentication" returns results from 12 different services, but you only care about the auth logic reachable from *this specific request handler*. Implemented in `crates/search/reachability.rs` — BFS over `call_edges`, then a chunk-ID pre-filter on HNSW retrieval. ~50 lines of code on top of existing infrastructure.

### `--why` flag

Shows exactly what drove each match: the `embedding_text` embedded, BM25 score breakdown, and which tokens from the query appear in the chunk:

```bash
smart-grep "payment retry logic" --why
# Result 1: src/billing/retry.rs:45-67 [score=0.91]
#   Embedding text used: "Language: Ruby\nSymbol: retry_charge\nKind: method\n..."
#   BM25 tokens matched: retry, charge
#   Semantic score: 0.87  BM25 boost: 0.04
```

---

## Core Design: Context Budget Mode

Agents and LLM workflows have a fixed token budget. Instead of returning top-K chunks and leaving token management to the caller, smart-grep gets **token-aware**:

```bash
smart-grep "payment error handling" --budget 4000
```

Given a token budget, smart-grep greedily selects highest-scoring chunks that fit, automatically truncating each chunk to its most query-relevant lines if needed (highest keyword-density subrange within the chunk). Output is a single ready-to-paste context block:

```text
# src/billing/stripe.rs:120-148  [rust]  score=0.91
fn handle_charge_failure(err: &StripeError) {
    ...
}

# app/services/payments.rb:44-63  [ruby]  score=0.88
rescue Stripe::CardError => e
    ...
```

Implementation: token counting via `text.split_whitespace().count() * 1.3` approximation; greedy bin-packing in `crates/search/budget.rs`. No new dependencies.

---

## Core Design: Streaming Results (`--stream`)

For large repos or expensive queries (e.g. `--expand` with a deep call graph), stream results as they are found rather than waiting for all results:

```bash
smart-grep "payment handling" --stream
```

```text
[streaming...]
1. src/billing/stripe.rs:120-148  score=0.91
2. app/services/payments.rb:44-63  score=0.88
3. src/billing/legacy.rs:201-234   score=0.82
[complete -- 3 results in 340ms]
```

For human developers, streaming output feels instant even when the full search takes 500ms. For agents, it enables pipelining — start processing the highest-confidence result while lower-ranked ones are still being scored.

In MCP HTTP/SSE mode, `--stream` maps directly to Server-Sent Events, giving agents true incremental delivery.

**Implementation:** Rust channels + streaming output loop. The `--expand` case is naturally streaming because call-graph traversal is depth-first. Near-zero implementation cost.

---

## Core Design: Codebase Map

`smart-grep map` clusters all chunk embeddings using k-means and emits a **machine-readable JSON taxonomy** of what the codebase is about:

```json
{
  "clusters": [
    { "label": "payment processing", "size": 47, "centroid_symbol": "process_payment",     "files": ["src/billing/...", "app/payments/..."] },
    { "label": "authentication",     "size": 23, "centroid_symbol": "verify_token",         "files": ["src/auth/...", "middleware/..."]      },
    { "label": "retry / resilience", "size": 15, "centroid_symbol": "retry_with_backoff",   "files": ["src/jobs/...", "lib/queue/..."]       }
  ]
}
```

Labels are auto-generated from the centroid chunk's symbol name plus top recurring keywords across the cluster.

**Dual use:**
1. **Orientation tool**: new developers and agents get an instant overview of what a codebase does, without reading a single file.
2. **Search acceleration**: before embedding a query, find the nearest cluster and bias HNSW retrieval toward chunks in that cluster, reducing false positives from unrelated areas of large monorepos.

Implementation in `crates/search/cluster.rs` — k-means over existing embedding vectors stored in SQLite.

---

## Core Design: Semantic Diff

```bash
smart-grep diff HEAD~1
smart-grep diff main..feature-branch
```

Emit results not as byte diffs but as **semantic diffs**:

- **Changed meaning**: chunk's embedding moved more than a cosine threshold from its prior version
- **Renamed/moved**: deleted chunk has a high-similarity survivor elsewhere (semantic rename detection)
- **New behavior**: new chunk with no close prior counterpart
- **Stable**: chunk present and semantically unchanged

```text
SEMANTIC DIFF  HEAD~1 -> HEAD

~ CHANGED MEANING
  src/billing/stripe.rs::handle_charge_failure  (similarity to prior: 0.71)

-> RENAMED (likely)
  src/payments/process.rb::charge_card  ->  src/payments/process.rb::attempt_payment  (0.96 similarity)

+ NEW BEHAVIOR
  src/billing/idempotency.rs::check_idempotency_key

= 47 chunks unchanged
```

Implementation: `prev_vector` column in `embeddings` table stores the prior embedding snapshot. On re-index, compare new vs. prior cosine distance. Snapshot updated after each successful index run.

---

## Core Design: Index Snapshots + Time-Travel Search

```bash
# Save a named snapshot
smart-grep snapshot save pre-refactor
smart-grep snapshot save --git HEAD~10

# Query any historical snapshot
smart-grep "payment retry logic" --at pre-refactor
smart-grep "payment retry logic" --at HEAD~10

# Semantic diff between two named snapshots
smart-grep diff pre-refactor..now
```

Store **named index snapshots** tied to git refs or user-defined names. On demand (or via git hook), serialize the current `.smart-grep/index.db` to `.smart-grep/snapshots/<ref>/index.db`. Allow querying any snapshot.

git gives you time-travel for bytes. smart-grep gives you time-travel for *meaning* — enabling questions like "what code handled payments before the refactor?" and "when did retry logic first appear in this codebase?" These are questions no other tool can answer.

**Storage:** only the SQLite DB is snapshotted (not the HNSW index, which rebuilds on demand from stored embeddings). For a 50k-line codebase, a snapshot is typically under 20MB. Managed in `crates/store/snapshot.rs`.

---

## Core Design: Stale Logic Detection

```bash
smart-grep stale src/billing/stripe.rs::handle_charge_failure
```

When a function's meaning changes (its embedding moves significantly on re-index), automatically find other chunks that are **highly similar to the old embedding but have not been updated** — likely stale copies, parallel implementations, or tests that should have changed in sync.

```text
STALE CANDIDATES for handle_charge_failure (changed in last index run)

  src/billing/legacy.rs::process_card_decline      similarity-to-old: 0.89  last-modified: 47 days ago
  test/billing/stripe_test.rb::test_charge_failure  similarity-to-old: 0.84  last-modified: 12 days ago
```

This is a class of analysis impossible without semantic embeddings — `grep` finds textual duplication, smart-grep finds **conceptual staleness**.

Implementation: `WHERE cosine(prev_vector, chunk_embedding) > threshold AND file.mtime < changed_file.mtime`. Pure SQL + vector math, no new infrastructure.

---

## Core Design: Semantic Test Coverage Mapping

```bash
smart-grep coverage src/billing/stripe.rs::charge_card

smart-grep coverage --gaps
# Find production functions with no semantically similar tests
```

Chunks are tagged `is_test = 1` based on file path heuristics (`test/`, `*_test.*`, `*_spec.*`, `spec/`). `smart-grep coverage <symbol>` finds all test chunks with cosine similarity above a threshold to the target production function — without requiring any framework instrumentation, import analysis, or test execution.

```text
SEMANTIC COVERAGE for charge_card

  test/billing/stripe_test.rs::test_charge_success        similarity: 0.91
  test/billing/stripe_test.rs::test_charge_failure        similarity: 0.87
  test/integration/checkout_test.rb::test_full_checkout   similarity: 0.72

COVERAGE GAPS (production functions with no test similarity > 0.65)
  src/billing/idempotency.rs::check_idempotency_key   <- no semantic test coverage found
  src/billing/refunds.rs::process_partial_refund       <- no semantic test coverage found
```

This is **semantic test coverage** — not line coverage (requires running tests) but *conceptual coverage* (which tests are semantically *about* this function). It surfaces integration tests that cover a function even when they do not import it directly. `--gaps` mode is particularly powerful for agents doing autonomous code review.

Implementation: HNSW nearest-neighbor search scoped to `is_test = 1` chunks. ~30 lines in `crates/search/coverage.rs`.

---

## Core Design: Semantic Naming Linter (`smart-grep lint`)

```bash
smart-grep lint
smart-grep lint --check   # exit code 1 if issues found (for CI)
```

Three lint modes derived purely from the embedding index, requiring no new infrastructure:

### 1. Naming inconsistency
Pairs of chunks with cosine similarity > 0.90 but symbol names that are very lexically different — the same concept named two different ways:

```text
NAMING INCONSISTENCY
  src/billing/stripe.rs::handle_charge_failure    similarity: 0.96
  src/payments/legacy.rs::process_payment_error  <- near-identical behavior, different naming
```

### 2. Duplicate logic candidates
Pairs with similarity > 0.93 — likely copy-pasted or accidentally reimplemented:

```text
DUPLICATE LOGIC
  src/auth/middleware.rs::verify_token      similarity: 0.94
  src/api/guards.rs::check_auth_header     <- likely duplicate -- consider extracting
```

### 3. Orphaned code
Chunks with no semantic neighbors within similarity 0.60 — isolated islands that do not connect to any conceptual cluster, often dead code or forgotten utilities:

```text
ORPHANED CODE (no semantic neighbors)
  src/utils/xml_parser.rs::parse_saml_response  <- isolated; may be dead code
```

This is passive codebase quality monitoring at zero additional compute cost — the embeddings are already there. It is a form of static analysis no AST-based linter can perform, because these problems are semantic not syntactic. Add `smart-grep lint --check` to CI to catch naming drift over time.

Implementation: pairwise similarity scan using the existing HNSW index for approximate nearest-neighbor search, filtered by lexical distance check on symbol names. `crates/search/lint.rs`.

---

## Core Design: Multi-Query Fan-Out (`smart-grep multi`)

```bash
smart-grep multi \
  "payment error handling" \
  "stripe webhook processing" \
  "billing retry logic"
```

Execute multiple related queries in parallel, merge and deduplicate results by chunk identity, re-ranking by maximum score across all queries with a coverage boost for chunks appearing in multiple result sets:

```
multi_score = max(scores) + 0.05 * (num_queries_matched - 1)
```

Output annotates each result with which queries matched it:

```text
1. src/billing/stripe.rs:120-148  multi_score=0.96  matched: [payment errors, stripe webhooks]
2. app/services/payments.rb:44-63  multi_score=0.88  matched: [payment errors, billing retry]
```

Real debugging and auditing tasks are rarely single-concept queries. Multi-query with coverage boosting surfaces the *most conceptually central* code for a domain — the functions that sit at the intersection of multiple related concerns. These are almost always the most important functions to understand.

In MCP mode, agents can call `search_code_multi(queries: string[])` to get a single deduplicated ranked list in one tool call, saving round-trips.

**Implementation:** `rayon::join` N parallel HNSW queries, collect `(chunk_id, score, query_index)` tuples, group by chunk_id, apply formula, re-rank. ~80 lines on top of existing search infrastructure.

---

## Core Design: Agent Session Memory

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

A **session layer** that tracks queries across a working session, remembers which results were accessed, and builds a cumulative deduplicated context. Sessions persist in the `sessions` and `session_results` tables — lightweight row references, no re-embedding, no additional compute.

### `session summary`

Shows the conceptual territory covered and gaps:

```text
SESSION: "debugging payment failures"  (4 queries, 14 unique chunks)

Explored:  payment error handling, stripe webhooks, retry logic
Not yet explored (nearby cluster areas): refund flows, subscription billing, idempotency
```

The "not yet explored" suggestions come from comparing session chunk embeddings against the codebase cluster map — if nearby clusters have not been touched, surface them.

### `session context`

Assembles all high-ranking results from the session into a single token-budget-packed context block — deduplicated by chunk_id, sorted by max score across all queries. Do 5 searches, get one consolidated context in a single command.

For agents, this is the canonical deep-exploration workflow: run multiple `search_code` calls, then call `session_context(budget_tokens=8000)` to get the full accumulated picture in one shot.

**Implementation:** session tracking is writes to `session_results`. Deduplication and budget-packing reuse `budget.rs`. Gap-detection in `summary` is a cluster proximity query. Total: ~150 new lines across `session.rs` and minor additions to the search command.

---

## Core Design: Structural Pattern Search (`--pattern`)

```bash
smart-grep --pattern "function that catches error and retries"
smart-grep --pattern "class with init validate execute methods"
smart-grep --pattern "middleware that checks condition then calls next"
```

A complementary search mode operating on **AST structure** rather than semantic embeddings. During indexing, `crates/indexer/structural.rs` extracts normalized shape fingerprints from the AST — a description of a chunk's structural form:

```
try_catch -> condition_check -> recursive_call          # retry pattern
class -> [method:__init__, method:validate, method:execute]  # command pattern
function -> [guard_clause, delegate_call]                # middleware pattern
```

Structural search does fuzzy matching over these fingerprints rather than vector similarity. It finds code that follows the same **design pattern** regardless of domain:

```text
STRUCTURAL PATTERN: "function that catches error and retries"

  src/billing/stripe.rs::handle_charge_failure    shape-match: 0.94
  src/jobs/worker.rs::process_with_retry          shape-match: 0.91
  lib/http/client.rb::request_with_backoff        shape-match: 0.88
```

**Semantic vs. structural are orthogonal dimensions:**
- Semantic search finds "related concept" — `--like` a payment function finds all payment functions
- Structural search finds "same architecture" — `--pattern retry` finds all retry patterns regardless of domain

Combine them: `smart-grep --like <fn> --pattern "catches error and retries"` finds functions that are both semantically similar AND structurally similar — extremely precise for refactoring.

**Implementation:** structural fingerprinting is a tree-sitter AST traversal outputting a normalized shape string, stored in `chunks.structural_fp`. Fuzzy matching is edit distance or substring matching on those strings. No new ML, no new dependencies.

---

## Core Design: Confidence-Aware Results

```bash
# Default: warn if results are low-confidence
smart-grep "kubernetes pod scheduling logic"
# WARNING: LOW CONFIDENCE -- best match score 0.41 is below calibrated threshold 0.60
#    This codebase may not contain Kubernetes scheduling logic.
#    Suggestion: try  smart-grep map  to see what this codebase does contain.

# Strict mode: exit code 1 if no result above threshold (for CI / scripts)
smart-grep "payment handler" --confidence strict

# Always return results regardless of confidence
smart-grep "payment handler" --confidence off
```

Rather than always returning top-K results regardless of quality, smart-grep tracks the distribution of top-1 scores across all queries on this index (`query_scores` table, rolling 100-query window) and flags results that fall significantly below the historical mean for strong matches.

### Confidence modes
- `warn` (default): show warning with actionable suggestion, still return results
- `strict`: exit code 1 if no result above threshold — safe for CI scripts and pre-commit hooks
- `off`: always return results with no confidence annotation

### Low-confidence output
When confidence is low, smart-grep actively helps:
- Points to `smart-grep map` to discover what the codebase *does* contain
- Suggests reformulated queries based on nearest cluster labels
- Shows the calibration: "strong matches on this codebase typically score 0.82+"

The most insidious failure mode of any search tool is the silent false positive — returning a 0.41-similarity result that looks like an answer but is not. When used in automated pipelines, this causes agents to act on wrong information. Confidence-awareness makes smart-grep safe for fully automated use.

**Implementation:** percentile tracking of historical query scores. Threshold is the 20th percentile of historical top-1 scores for this index — calibrated per codebase, not a global magic number. Falls back to hardcoded 0.60 until 20+ queries have been recorded. ~80 lines in `crates/search/confidence.rs`.

---

## Core Design: Incremental Indexing

### File state transitions
- **new**: file not in `files` table -> parse + embed + insert
- **changed**: `mtime` or `content_hash` differs -> snapshot current embeddings to `prev_vector` -> delete old chunks (CASCADE handles all dependents) -> re-parse + re-embed + insert
- **deleted**: path no longer on disk -> DELETE from `files` -> CASCADE cleans all dependent rows
- **unchanged**: skip entirely

### HNSW rebuild strategy (v1)
For v1: detect changed files, apply SQLite mutations, then **rebuild the HNSW index from all current embeddings**. Safe, correct, fast enough for repos under ~100k functions. Optimize to online updates in v1.1.

### `--rebuild` flag
Forces full re-index, bypassing all incremental logic. Also triggered automatically when `meta.vector_dim` mismatches the current model.

---

## Core Design: Project Auto-Detection

On first `smart-grep index`, the `project_detect.rs` module scans for known framework signatures and auto-generates `.smart-grep/config.toml` with optimized settings:

| Detected signature | Auto-configured behavior |
|---|---|
| `Gemfile` + `app/` | Rails: index `app/`, skip `vendor/`, `log/`, `tmp/` |
| `package.json` + `src/` | Node/React: index `src/`, skip `node_modules/`, `dist/` |
| `Cargo.toml` workspace | Rust: index `crates/`, skip `target/` |
| `pyproject.toml` | Python: index project src, skip `.venv/`, `__pycache__/` |

Also auto-generates a starter benchmark file with 5 example queries based on detected patterns (e.g. detects Stripe dependency -> seeds query: `"where is the Stripe webhook handler?"`). This teaches query style while demonstrating value on the first run.

---

## Core Design: MCP Server Mode

```bash
smart-grep serve           # starts MCP server on stdio
smart-grep serve --port 8080  # HTTP/SSE mode for Cursor compatibility
```

Exposes smart-grep as a **Model Context Protocol server**, mountable directly in Claude Code, Cursor, or any MCP-aware agent environment.

### MCP tool surface

```json
{
  "tools": [
    {
      "name": "search_code",
      "description": "Semantic search. Supports natural language query or --like code snippet.",
      "parameters": {
        "query": "string", "like": "string?", "top_k": "int",
        "language": "string?", "budget_tokens": "int?",
        "from_symbol": "string?", "stream": "bool?"
      }
    },
    {
      "name": "search_code_multi",
      "description": "Fan-out over multiple related queries with deduplication and coverage boosting",
      "parameters": { "queries": "string[]", "top_k": "int?", "budget_tokens": "int?" }
    },
    {
      "name": "get_chunk",
      "description": "Retrieve a specific chunk by ID",
      "parameters": { "chunk_id": "string" }
    },
    {
      "name": "get_context",
      "description": "Chunk plus its callers and callees from the call graph",
      "parameters": { "chunk_id": "string", "expand_callers": "bool", "expand_callees": "bool" }
    },
    {
      "name": "codebase_map",
      "description": "Semantic cluster map of the entire codebase",
      "parameters": {}
    },
    {
      "name": "find_stale",
      "description": "Find chunks semantically similar to a recently-changed function",
      "parameters": { "chunk_id": "string" }
    },
    {
      "name": "session_context",
      "description": "Deduplicated context block from all queries in the current session",
      "parameters": { "session_name": "string", "budget_tokens": "int" }
    },
    {
      "name": "coverage",
      "description": "Find test chunks semantically covering a production function, or find gaps",
      "parameters": { "chunk_id": "string?", "gaps": "bool?" }
    }
  ]
}
```

The transport layer is `stdio` by default (compatible with Claude Code's MCP config) with optional HTTP/SSE mode for Cursor. Since the transport pattern is already established in the `verse-mcp` codebase, this is largely a structural copy with tool-specific handlers wired to existing functions.

---

## Core Design: Self-Index Mode

```bash
smart-grep "where is embedding batching implemented?" --self
smart-grep "how does the BM25 scorer work?" --self
smart-grep --pattern "catches error and retries" --self
```

smart-grep ships with a **pre-built index of its own source code** embedded in the binary. At release time, the build script runs `smart-grep index` on the repo and serializes the resulting `.smart-grep/` directory; bundled via `include_bytes!()` and extracted to a temp dir when `--self` is invoked.

**Dual purpose:**
1. **Zero-setup demo**: every README example works instantly on any machine with no project to index first.
2. **Embedded documentation**: contributors can explore smart-grep's own source semantically instead of reading rustdoc.

Index size for a codebase of this scale: under 5MB. Negligible binary overhead.

---

## CLI Design

```bash
# Indexing
smart-grep index                        # index current directory
smart-grep index /path/to/repo          # index specific path
smart-grep index --rebuild              # force full re-index

# Natural language search
smart-grep "where is payment error handled?"
smart-grep "retry failed billing" --top 10
smart-grep "auth check before delete" --lang rust
smart-grep "database connection pool" --json

# Query-by-example
smart-grep --like 'fn retry_with_backoff(...) { ... }'
cat src/billing/stripe.py | smart-grep --like -

# Scoped and structured search
smart-grep "payment error handling" --budget 4000
smart-grep "handle charge failure" --expand
smart-grep "payment retry logic" --why
smart-grep "payment error" --from src/api/checkout.rs::handle_checkout
smart-grep "payment handling" --stream

# Multi-query fan-out
smart-grep multi "payment errors" "stripe webhooks" "billing retry"

# Structural pattern search
smart-grep --pattern "function that catches error and retries"
smart-grep --like 'rescue Stripe::CardError' --pattern "catches error and retries"

# Codebase intelligence
smart-grep map
smart-grep diff HEAD~1
smart-grep stale src/billing/stripe.rs::handle_charge_failure
smart-grep coverage src/billing/stripe.rs::charge_card
smart-grep coverage --gaps
smart-grep lint
smart-grep lint --check                 # CI mode: exit 1 if issues found
smart-grep stats
smart-grep doctor

# Sessions
smart-grep session start "debugging payments"
smart-grep session summary
smart-grep session context --budget 8000
smart-grep session list

# Snapshots / time-travel
smart-grep snapshot save pre-refactor
smart-grep snapshot save --git HEAD~10
smart-grep "payment retry logic" --at pre-refactor

# MCP server
smart-grep serve
smart-grep serve --port 8080

# Self-index (pre-built, ships in binary)
smart-grep "how does chunking work?" --self
```

### Flags (search)
- `--top N` — number of results (default: 5)
- `--lang <language>` — filter by language
- `--json` — machine-readable output
- `--budget N` — token-budget context block mode
- `--expand` — include callers and callees from call graph
- `--from <file::symbol>` — restrict to call-graph-reachable subset
- `--why` — show match explanation (embedding text, BM25 tokens, per-component scores)
- `--like <snippet or ->` — query by code example instead of natural language
- `--pattern <description>` — structural AST shape filter
- `--stream` — stream results as found
- `--at <snapshot>` — query a historical snapshot
- `--confidence strict|warn|off` — control low-confidence behavior (default: warn)
- `--verbose` — full chunk metadata in results
- `--path <dir>` — override repo root

### Flags (index)
- `--rebuild` — force full re-index
- `--verbose` — per-file chunk counts and parse errors

### Default terminal output

```text
1. src/billing/stripe.rs:120-148  [rust]  score=0.91
   fn handle_charge_failure(err: &StripeError) { ... }

2. app/services/payments.rb:44-63  [ruby]  score=0.88
   rescue Stripe::CardError => e
```

### JSON output (for agents)

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
    "matched_queries": ["payment errors", "billing retry"],
    "preview": "fn handle_charge_failure(err: &StripeError) {"
  }
]
```

---

## Configuration

Repo-local config at `.smart-grep/config.toml` (auto-created with project-detected defaults on first `index`):

```toml
[embedding]
model = "fastembed-default"

[index]
include = ["src", "app", "lib", "crates"]
exclude = ["node_modules", "dist", "vendor", "target", ".git", "build"]
min_lines = 5
max_lines = 80

[search]
default_top_k = 5
hybrid_alpha = 0.7            # weight between semantic (1.0) and BM25 (0.0)
query_expansion = true        # enable synonym map expansion
confidence_mode = "warn"      # warn | strict | off
confidence_threshold = 0.60   # override auto-calibration if set

[map]
num_clusters = 10

[lint]
naming_similarity_threshold = 0.90
duplicate_similarity_threshold = 0.93
orphan_similarity_threshold = 0.60

[coverage]
test_similarity_threshold = 0.65

[snapshots]
auto_snapshot_on_index = false  # set true to auto-snapshot before each re-index
max_snapshots = 10

[sessions]
ttl_days = 7                  # sessions expire after this many days
```

---

## Observability

A semantic tool becomes hard to trust if it feels like a black box.

- `smart-grep stats` — files indexed, chunk count, model name, vector dimension, last index time, graph edge count, session count, snapshot count
- `smart-grep doctor` — verify tree-sitter grammars loaded, fastembed model present, SQLite readable, HNSW index version matches DB, session store intact
- `--why` on search — `embedding_text` used, BM25 token matches, per-component score breakdown, confidence calibration data
- `--verbose` on search — full chunk metadata including `symbol_kind`, `parent_symbol`, cluster assignment, structural fingerprint
- `smart-grep index --verbose` — per-file chunk counts, parse errors, graph edges extracted

---

## Error Handling

Use `thiserror` for typed errors per crate, `anyhow` for propagation in `crates/core`.

| Failure | Mitigation |
|---|---|
| Unsupported language | Warn and skip, never crash |
| Parse error | Mark `files.parse_error = 1`, continue indexing |
| Corrupted SQLite | Surface clear message, suggest `--rebuild` |
| Embedding dimension mismatch | Detected via `meta.vector_dim`; auto-trigger `--rebuild` |
| Stale/missing HNSW index | Rebuild automatically from existing SQLite embeddings |
| Unreadable files | Warn and skip |
| Model swapped between runs | `meta.model_name` mismatch -> require `--rebuild` with clear message |
| Low-confidence results | Flag with warning + actionable suggestions (see Confidence Design) |
| Snapshot not found | Clear error: list available snapshots with `smart-grep snapshot list` |
| Session not found | Clear error: list available sessions with `smart-grep session list` |
| Reachability entry not indexed | Warn: symbol not in index, fall back to full search |
| `--from` BFS too deep | Configurable `--from-depth` limit (default: 3 hops) |

---

## Evaluation Plan

Quality must be measured, not assumed.

### Benchmark query set

Build 25-50 natural-language queries paired with expected code locations, plus 10 query-by-example cases and 5 structural pattern cases.

| Query type | Example | Expected matches |
|---|---|---|
| Natural language | "where is payment error handled?" | Stripe exceptions, decline handlers, billing retry |
| Natural language | "where do we verify admin permissions?" | Guard clauses, middleware, role checks |
| Natural language | "where are failed jobs retried?" | Retry loops, backoff logic, queue retry config |
| Query-by-example | paste a retry function | All retry pattern implementations across languages |
| Structural | "catches error and retries" | All try/catch+retry patterns regardless of domain |
| Scoped | "auth logic" --from api/checkout | Only auth reachable from checkout flow |
| Multi-query | "payment errors" + "billing retry" | Central billing resilience functions |

### Metrics
- **Top-1 accuracy** — correct result in position 1
- **Top-5 recall** — correct result in top 5
- **Hybrid vs. semantic-only** — BM25 improvement on exact symbol queries
- **Query-by-example precision** — cross-language pattern matching accuracy
- **Structural pattern recall** — finds all instances of a given pattern
- **Reachability precision** — `--from` scope contains only reachable results
- **Confidence calibration** — false-positive rate at default threshold
- **Coverage recall** — `smart-grep coverage` finds known test relationships
- **Lint precision** — naming inconsistencies flagged are genuine (not false alarms)
- **Index time** — wall time for initial full index on ~50k-line repo
- **Search latency** — p50/p99 for single query
- **Incremental re-index latency** — after changing 1 file in 10k-file repo

### Baseline comparison
Compare against `ripgrep` on all benchmark queries. MVP success target: meaningfully higher recall on intent-based queries, sub-100ms search latency for local repos.

---

## Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Bad chunking hurts relevance | AST-aware chunks with size bounds; iterate against benchmark |
| Local embeddings too slow | Batch + `rayon` parallelism; smaller default model |
| Incremental HNSW updates are complex | Rebuild in v1; online updates in v1.1 |
| Cross-language support becomes fragile | `ChunkExtractor` trait + per-language tests |
| Users do not trust semantic matches | `--why` explains every match; confidence modes |
| Model vectors change after upgrade | `meta.model_name` mismatch -> auto-trigger `--rebuild` |
| Call graph extraction is noisy | Graph used only for `--expand`/`--from`/`--stale`; does not affect core search |
| k-means cluster count wrong | Default 10 is reasonable; user-tunable |
| Self-index binary bloat | Index <5MB; gzip-compressed; acceptable tradeoff |
| BM25 degrades semantic precision | `hybrid_alpha = 0.7` keeps semantic dominant; tunable |
| Structural fingerprints are imprecise | Used as a filter/boost, not primary ranking; graceful degradation |
| Snapshot storage grows unbounded | `max_snapshots` config with LRU eviction |
| Confidence calibration wrong for new repos | Falls back to hardcoded threshold until 20+ queries recorded |
| Session memory grows stale | Sessions expire after configurable TTL (default: 7 days) |
| Lint false positives erode trust | All lint output shows similarity scores; `--check` off by default |
| `--from` BFS too deep on large graphs | Configurable `--from-depth` limit (default: 3 hops) |

---

## Phases

### Phase 1 — POC (Node.js, 1 day)

Validate the concept before writing any Rust.

- [ ] tree-sitter parse JS/TS into AST chunks (functions and classes, not line windows)
- [ ] fastembed local embedding — `Xenova/all-MiniLM-L6-v2` via `@xenova/transformers`
- [ ] Store vectors in sqlite
- [ ] Cosine similarity search in memory
- [ ] CLI: `node poc/index.js index .` and `node poc/index.js search "<query>"`
- [ ] Benchmark against ripgrep on the `test-fixtures/` files for 10 conceptual queries
- [ ] Validate that the normalized `embedding_text` format improves recall vs. raw source
- [ ] Validate query-by-example: paste a code snippet, find similar chunks across test fixtures

**Success criteria:** smart-grep returns relevant results that ripgrep returns zero for; query-by-example finds cross-language analogues in test fixtures.

---

### Phase 2 — Rust Foundations + Searchable MVP (3–4 days)

Goal: reliable end-to-end vertical slice — index one language, search it.

- [ ] Set up workspace with 4 crates + shared types in `crates/core`
- [ ] Define `ChunkExtractor` and `Embedder` traits
- [ ] `crates/indexer`:
  - tree-sitter for Rust + TypeScript/JavaScript
  - AST chunk extraction — functions, classes, impls; size-bounded; `is_test` tagging
  - `embedding_text` normalizer
  - `structural.rs` — AST shape fingerprinting
  - fastembed-rs — local ONNX embedding, batched with `rayon`
  - `synonym_map.rs` — concept->symbol co-occurrence map
- [ ] `crates/store`:
  - rusqlite — full schema including `call_edges`, `synonyms`, `sessions`, `snapshots`, `query_scores`, `meta`
  - usearch HNSW — vector storage and ANN search
- [ ] `crates/search`:
  - HNSW top-K retrieval
  - BM25 scorer (`bm25.rs`) + hybrid re-ranking
  - Query expansion via synonym map
  - `--like` mode: embed code snippet as query
  - `--pattern` mode: fuzzy structural fingerprint matching
  - `confidence.rs` — score calibration + low-confidence detection
  - Top-N results with path, line range, score, preview
- [ ] `crates/core`:
  - `smart-grep index .`
  - `smart-grep "<query>"` with hybrid + confidence
  - `smart-grep stats`
  - `--top N`, `--lang`, `--json`, `--verbose`, `--why`, `--like`, `--pattern`, `--confidence`, `--stream` flags

**Success criteria:** can index a small repo and get relevant results; `--like` finds cross-language pattern matches; `--confidence warn` triggers on out-of-domain queries.

---

### Phase 3 — Incremental Indexing + Graph + Sessions (1–2 days)

- [ ] `blake3` content hashing
- [ ] Incremental file state transitions (new/changed/deleted/unchanged)
- [ ] Snapshot `prev_vector` before each re-index of changed chunks
- [ ] Rebuild HNSW after mutations; `--rebuild` flag
- [ ] `graph.rs` — call edge extraction from AST
- [ ] `reachability.rs` — BFS over call_edges for `--from <entry>` scoping
- [ ] `expand.rs` — `--expand` flag: callers + callees in results
- [ ] `smart-grep stale <symbol>`
- [ ] `session.rs` — start/resume/summary/context/list commands
- [ ] `smart-grep multi` — parallel fan-out with coverage boosting
- [ ] `--stream` — streaming result output

**Success criteria:** re-index after 1 changed file < 5 seconds; `--from` correctly scopes results to reachable subgraph; session context assembles coherent multi-query context.

---

### Phase 4 — Multi-language + Codebase Intelligence (2–3 days)

- [ ] Python and Ruby `ChunkExtractor` implementations
- [ ] Per-language tests with `test-fixtures/`
- [ ] `smart-grep map` — k-means clustering + JSON taxonomy; cluster-biased search
- [ ] `smart-grep diff <ref>` — semantic diff via `prev_vector`
- [ ] `snapshot.rs` — save/restore index snapshots; `--at <snapshot>` time-travel search
- [ ] `coverage.rs` — `smart-grep coverage` and `--gaps` mode
- [ ] `lint.rs` — naming inconsistency + duplicate logic + orphan detection; `--check` CI mode
- [ ] `project_detect.rs` — auto-detect framework + generate config + seed benchmark queries
- [ ] `smart-grep doctor`
- [ ] `--budget N` — token-budget context block
- [ ] Build 25-50 query benchmark set; document in `docs/benchmark.md`

**Success criteria:** lint finds real naming inconsistencies in a real-world repo without false-positive flood; coverage gaps mode surfaces genuinely untested functions; `--at` time-travel returns correct results from a historical snapshot.

---

### Phase 5 — MCP Server + Self-Index + Polish (1–2 days)

- [ ] `smart-grep serve` — MCP server (stdio + HTTP/SSE) with full 8-tool surface
- [ ] Wire all MCP tools to existing handlers (search, multi, context, map, stale, session, coverage)
- [ ] Build script: index own source, serialize, `include_bytes!()` for `--self` mode
- [ ] Full `--json` schema including `confidence`, `matched_queries`, `semantic_score`, `bm25_boost`
- [ ] `CLAUDE.md` and `AGENTS.md` snippet in README
- [ ] Validate `meta.vector_dim` + `meta.model_name` on startup
- [ ] `--watch` mode via `notify` crate
- [ ] `cargo install smart-grep` + GitHub Releases binary distribution

**Success criteria:** MCP server mountable in Claude Code with zero additional config; `smart-grep "how does chunking work?" --self` resolves correctly on a fresh install; `smart-grep lint --check` returns exit code 1 on a repo with known naming inconsistencies.

---

## Integration with Claude Code

Add to any project's `CLAUDE.md` or `AGENTS.md`:

```markdown
## Search
Use `smart-grep "<query>"` INSTEAD of grep or ripgrep for any conceptual search.
Use grep or ripgrep only when searching for an exact symbol name or string.

## Query-by-example
Use `smart-grep --like '<code snippet>'` to find similar code patterns across languages.
Pipe a file directly: `cat src/billing/stripe.py | smart-grep --like -`

## Scoped search
Use `smart-grep "<query>" --from <file::symbol>` to restrict search to code reachable
from a specific entry point — useful in large monorepos.

## Context assembly
Use `smart-grep "<query>" --budget 6000` for a token-budget-aware context block.
Use `smart-grep multi "<q1>" "<q2>" "<q3>"` to find code central to multiple concepts.
Use `smart-grep session context --budget 8000` after multiple queries to get one consolidated context.

## Code quality
Use `smart-grep coverage --gaps` to find production functions with no semantic test coverage.
Use `smart-grep lint` to find naming inconsistencies and duplicate logic.

## Examples
  smart-grep "where is payment error handled?"
  smart-grep "retry failed jobs" --json
  smart-grep "auth middleware" --expand --budget 4000
  smart-grep --like 'rescue Stripe::CardError => e' --pattern "catches error and retries"
  smart-grep "payment logic" --from src/api/checkout.rs::handle_checkout
```

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

## Timeline

| Phase | Duration |
|---|---|
| 1 — POC | 1 day |
| 2 — Rust foundations + searchable MVP | 3–4 days |
| 3 — Incremental indexing + graph + sessions | 1–2 days |
| 4 — Multi-language + codebase intelligence | 2–3 days |
| 5 — MCP server + self-index + polish | 1–2 days |
| **Total** | **~9–12 days** |

---

## Feature Map

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
| **Time travel** | `snapshot save/list`, `--at <ref>` historical queries |
| **Integration** | MCP server (stdio + SSE), `CLAUDE.md` snippet, `--self` embedded demo |
| **Reliability** | Incremental indexing, `--rebuild`, confidence modes, `doctor` health check |

---

## Future Extensions

- Code-specific embedding model swap (`nomic-embed-code`) for better recall on exact APIs
- Online HNSW updates (skip rebuild on incremental re-index)
- Symbol relationship graph visualization (export to DOT/Mermaid)
- File-level and repo-level natural-language summaries for agent context
- Editor integrations (Neovim LSP shim, VS Code extension)
- Cross-repo search (federated index across multiple local repos)
- Query history with replay and diff
- Team-shared indexes (export/import `.smart-grep/` as a portable artifact)
- Auto-snapshot on git commit via hook installer (`smart-grep hooks install`)
- CI benchmark regression detection (fail if Top-1 accuracy drops below baseline)
- Watch mode with file-event-driven partial HNSW updates
