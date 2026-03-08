# PLAN: smart-grep

## Overview

Semantic code search CLI — finds code by meaning, not by exact keywords.
Built in Rust. Zero API calls during search. Fully local vector index.

---

## Why Workspace + `crates/` (not `src/`)

Popular Rust CLIs like ripgrep, fd, and bat all use this pattern.
Ripgrep specifically uses `crates/core/main.rs` as its entry point, not `src/main.rs`.

- Workspace root holds config, docs, poc, and tests — **no source code**
- Each concern (indexing, storage, search) is an independently testable crate
- `src/` still exists but only **inside each sub-crate**, never at the root
- Makes it easy to later publish `crates/indexer` as a standalone library

---

## File Structure

```
smart-grep/
├── Cargo.toml              # workspace root — no source code here
├── Cargo.lock
├── README.md
├── POC.md
├── PLAN.md
├── .gitignore              # includes .smart-grep/
│
├── crates/
│   ├── core/               # binary entry point
│   │   ├── Cargo.toml
│   │   └── main.rs         # CLI: index / search
│   │
│   ├── indexer/            # tree-sitter chunking + local embedding
│   │   ├── Cargo.toml
│   │   ├── lib.rs
│   │   ├── chunker.rs      # AST-based code chunking (not line windows)
│   │   ├── embedder.rs     # fastembed-rs wrapper (local ONNX model)
│   │   ├── languages.rs    # language dispatch — no mod.rs (Rust 1.30+ style)
│   │   └── languages/
│   │       ├── javascript.rs
│   │       ├── typescript.rs
│   │       ├── rust.rs
│   │       └── python.rs
│   │
│   ├── store/              # sqlite + usearch HNSW index
│   │   ├── Cargo.toml
│   │   ├── lib.rs
│   │   ├── db.rs           # rusqlite: chunk metadata + mtime tracking
│   │   └── index.rs        # usearch HNSW: vector storage + ANN search
│   │
│   └── search/             # ranking and result formatting
│       ├── Cargo.toml
│       ├── lib.rs
│       └── rank.rs         # cosine similarity scoring + top-N selection
│
├── poc/
│   ├── package.json
│   └── index.js            # Node.js POC — validate concept before Rust
│
├── test-fixtures/
│   ├── billing.js          # payment handling code without the word "payment"
│   └── auth.js             # authentication code without "authentication"
│
├── tests/
│   └── search_test.rs      # integration: index → search round-trip
│
└── benches/
    └── index_bench.rs
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
fastembed = "3"
usearch = "2"
rusqlite = { version = "0.31", features = ["bundled"] }
ignore = "0.4"
clap = { version = "4", features = ["derive"] }
serde_json = "1"
serde = { version = "1", features = ["derive"] }
anyhow = "1"
```

---

## Phases

### Phase 1 — POC (Node.js, 1 day)

Validate the concept before writing any Rust.

- [ ] tree-sitter parse JS/TS into AST chunks (functions and classes, not line windows)
- [ ] fastembed local embedding — `Xenova/all-MiniLM-L6-v2` via `@xenova/transformers`
- [ ] Store vectors in sqlite
- [ ] Cosine similarity search in memory
- [ ] CLI: `node poc/index.js index .` and `node poc/index.js search "query"`
- [ ] Benchmark against ripgrep on 10 conceptual queries to prove the gap

**Success criteria:** smart-grep returns relevant results that ripgrep returns zero for.

---

### Phase 2 — Rust Core (3–4 days)

- [ ] Set up workspace with 4 crates
- [ ] `crates/indexer`:
  - tree-sitter per language — extract top-level functions and classes as chunks
  - fastembed-rs — local ONNX embedding, no API key needed
- [ ] `crates/store`:
  - rusqlite for chunk metadata (file, line range, content)
  - usearch HNSW for vector storage and approximate nearest neighbor search
- [ ] `crates/search`:
  - Cosine similarity scoring
  - Top-N ranked results with file path and line number
- [ ] `crates/core`:
  - `smart-grep index .` — build or update the index
  - `smart-grep "<query>"` — semantic search
  - `--top N`, `--lang <language>`, `--json` flags

---

### Phase 3 — Incremental Index (1–2 days)

- [ ] Store file mtime and content hash in sqlite alongside chunk data
- [ ] On re-index: skip files unchanged since last run
- [ ] Remove chunks for deleted files
- [ ] `--watch` mode via `notify` crate for automatic re-indexing

---

### Phase 4 — Polish (1 day)

- [ ] `--lang` filter to restrict results by language
- [ ] `--json` structured output for Claude Code and scripts
- [ ] `CLAUDE.md` snippet in README
- [ ] Graceful handling of unparseable files and missing index
- [ ] `cargo install smart-grep`

---

## Integration with Claude Code

Add to any project's `CLAUDE.md`:

```markdown
## Search
Use `smart-grep "<query>"` INSTEAD of grep or ripgrep for any conceptual search.
Use grep or ripgrep only when searching for an exact symbol name or string.
```

---

## Timeline

| Phase | Duration |
|---|---|
| 1 — POC | 1 day |
| 2 — Rust core | 3–4 days |
| 3 — Incremental index | 1–2 days |
| 4 — Polish | 1 day |
| **Total** | **~1 week** |
