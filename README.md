# smart-grep

> Semantic code search for your codebase. Find code by **meaning**, not by exact keywords.

---

## The Problem

Claude Code uses `grep` and `ripgrep` under the hood — both are regex/keyword tools.

When you ask *"where is payment error handled?"*, Claude Code searches for the literal string `payment error`.  
It misses:
- `rescue Stripe::CardError`
- `catch (BillingException e)`
- `handleChargeFailure()`
- `PaymentDeclinedError`

All of these are semantically the same thing. None of them match.

## Why smart-grep Is Better

| | grep / ripgrep | smart-grep |
|---|---|---|
| Search method | Exact string / regex | Semantic meaning |
| Understands intent | ❌ | ✅ |
| Finds renamed symbols | ❌ | ✅ |
| Works across languages | Partially | ✅ |
| Token cost for Claude | High (reads many files) | Low (ranked results only) |
| Persistent index | ❌ (re-scans every time) | ✅ (incremental updates) |

## How It Works

```
1. Index phase  (once)
   └── tree-sitter parses code into AST chunks
   └── each chunk embedded into vector (fastembed, local)
   └── stored in sqlite + HNSW index

2. Search phase  (instant)
   └── query embedded into same vector space
   └── cosine similarity search
   └── returns top-N ranked results with file + line
```

**Zero API calls during search.** Embedding runs locally via `fastembed`. Index is stored in `.smart-grep/index.db` at project root.

## Tech Stack

| Crate | Purpose |
|---|---|
| `tree-sitter` | AST-based code chunking (not line-by-line) |
| `fastembed` | Local embedding model — no API needed |
| `sqlite` + `usearch` | Vector storage + HNSW similarity search |
| `ignore` | Respects `.gitignore` automatically |
| `clap` | CLI argument parsing |
| `serde_json` | Structured JSON output for Claude Code |

## Installation

```bash
cargo install smart-grep
```

## Usage

```bash
# Build index (run once per project)
smart-grep index .

# Search by meaning
smart-grep "handle payment failure"
smart-grep "user authentication middleware"
smart-grep "database connection retry logic"

# Options
smart-grep "error handling" --top 10          # more results
smart-grep "auth" --lang rust                 # filter by language
smart-grep "cache invalidation" --json        # structured output for scripts
smart-grep index . --watch                    # auto-reindex on file change
```

## Output Example

```
smart-grep "handle payment failure"

src/billing/stripe.rs:142
  fn handle_charge_error(err: StripeError) -> PaymentResult
  Score: 0.94

src/models/payment.rs:87
  impl PaymentDeclinedError { ... }
  Score: 0.91

src/jobs/retry_payment.rs:34
  async fn retry_failed_charge(invoice_id: Uuid)
  Score: 0.88
```

## Integration with Claude Code

Add to your project's `CLAUDE.md`:

```markdown
## Custom Tools

- `smart-grep "<query>"` — semantic code search. Use this INSTEAD of grep/ripgrep
  when searching for logic or behavior (not exact variable names).
  Example: `smart-grep "payment error handling"`
```

Claude Code will automatically prefer `smart-grep` for conceptual searches.

## Index Location

```
your-project/
└── .smart-grep/
    ├── index.db       # vector index
    └── metadata.json  # indexed files + timestamps
```

Add `.smart-grep/` to `.gitignore` — each developer builds their own local index.

---

## Roadmap

- [ ] `--watch` mode for auto-reindex
- [ ] Multi-repo search
- [ ] Custom embedding model support
- [ ] VS Code extension
