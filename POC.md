# POC: smart-grep

> Proof of concept demonstrating that semantic code search works and is meaningfully
> better than ripgrep for conceptual queries.

---

## POC Goal

Prove two things:
1. We can chunk a codebase using tree-sitter into meaningful AST units
2. Semantic search finds relevant code that exact grep completely misses

## POC Scope (No Rust Yet)

The POC is written in **Node.js** to validate the concept fast.  
The production tool will be rewritten in Rust for performance.

---

## Setup

```bash
cd poc
npm install
```

**Dependencies:**
```json
{
  "better-sqlite3": "^9.0.0",
  "@xenova/transformers": "^2.17.0",
  "tree-sitter": "^0.21.0",
  "tree-sitter-javascript": "^0.21.0",
  "glob": "^10.0.0"
}
```

---

## POC Code

### `poc/index.js`

```javascript
import Database from 'better-sqlite3'
import { pipeline } from '@xenova/transformers'
import Parser from 'tree-sitter'
import JavaScript from 'tree-sitter-javascript'
import { globSync } from 'glob'
import fs from 'fs'
import path from 'path'

// ─── 1. Setup ────────────────────────────────────────────────────────────────

const db = new Database('.smart-grep/index.db')
db.exec(`
  CREATE TABLE IF NOT EXISTS chunks (
    id INTEGER PRIMARY KEY,
    file TEXT,
    start_line INTEGER,
    end_line INTEGER,
    code TEXT,
    embedding BLOB
  )
`)

const embedder = await pipeline('feature-extraction', 'Xenova/all-MiniLM-L6-v2')
const parser = new Parser()
parser.setLanguage(JavaScript)

// ─── 2. Chunking via tree-sitter ─────────────────────────────────────────────
// Extract top-level functions and classes — not arbitrary line windows

function extractChunks(sourceCode, filePath) {
  const tree = parser.parse(sourceCode)
  const chunks = []
  const lines = sourceCode.split('\n')

  function visit(node) {
    const isChunkable = [
      'function_declaration',
      'class_declaration',
      'method_definition',
      'arrow_function',
      'export_statement'
    ].includes(node.type)

    if (isChunkable && node.endPosition.row - node.startPosition.row > 2) {
      chunks.push({
        file: filePath,
        start_line: node.startPosition.row + 1,
        end_line: node.endPosition.row + 1,
        code: lines.slice(node.startPosition.row, node.endPosition.row + 1).join('\n')
      })
    }

    for (const child of node.children) visit(child)
  }

  visit(tree.rootNode)
  return chunks
}

// ─── 3. Embed and store ───────────────────────────────────────────────────────

async function embed(text) {
  const output = await embedder(text, { pooling: 'mean', normalize: true })
  return Buffer.from(new Float32Array(output.data).buffer)
}

async function indexDirectory(dir) {
  const files = globSync(`${dir}/**/*.{js,ts,jsx,tsx}`, {
    ignore: ['**/node_modules/**', '**/.git/**']
  })

  console.log(`Indexing ${files.length} files...`)

  const insert = db.prepare(`
    INSERT OR REPLACE INTO chunks (file, start_line, end_line, code, embedding)
    VALUES (?, ?, ?, ?, ?)
  `)

  for (const file of files) {
    const source = fs.readFileSync(file, 'utf8')
    const chunks = extractChunks(source, file)

    for (const chunk of chunks) {
      const embedding = await embed(chunk.code)
      insert.run(chunk.file, chunk.start_line, chunk.end_line, chunk.code, embedding)
    }
  }

  console.log(`Done. Indexed chunks: ${db.prepare('SELECT COUNT(*) as n FROM chunks').get().n}`)
}

// ─── 4. Cosine similarity search ─────────────────────────────────────────────

function cosineSimilarity(bufA, bufB) {
  const a = new Float32Array(bufA.buffer, bufA.byteOffset, bufA.length / 4)
  const b = new Float32Array(bufB.buffer, bufB.byteOffset, bufB.length / 4)
  let dot = 0, normA = 0, normB = 0
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i]
    normA += a[i] ** 2
    normB += b[i] ** 2
  }
  return dot / (Math.sqrt(normA) * Math.sqrt(normB))
}

async function search(query, topK = 5) {
  const queryEmbedding = await embed(query)
  const chunks = db.prepare('SELECT * FROM chunks').all()

  const scored = chunks.map(chunk => ({
    ...chunk,
    score: cosineSimilarity(chunk.embedding, queryEmbedding)
  }))

  return scored
    .sort((a, b) => b.score - a.score)
    .slice(0, topK)
}

// ─── 5. CLI ───────────────────────────────────────────────────────────────────

const [,, command, ...args] = process.argv
fs.mkdirSync('.smart-grep', { recursive: true })

if (command === 'index') {
  await indexDirectory(args[0] || '.')

} else if (command === 'search' || !command.startsWith('-')) {
  const query = command === 'search' ? args[0] : command
  const results = await search(query)

  for (const r of results) {
    console.log(`\n${r.file}:${r.start_line}  (score: ${r.score.toFixed(3)})`)
    console.log(r.code.split('\n').slice(0, 3).join('\n'))
    console.log('...')
  }
}
```

---

## Run the POC

```bash
# Index a codebase (try it on itself or any JS project)
node poc/index.js index ./test-fixtures

# Search by meaning
node poc/index.js search "handle payment error"
node poc/index.js search "user authentication"
node poc/index.js search "retry failed request"
```

---

## POC Test Fixtures

Put these in `test-fixtures/` to demonstrate the gap vs grep:

### `test-fixtures/billing.js`
```javascript
// grep for "payment error" finds NOTHING here
async function handleChargeFailure(customerId, invoiceId) {
  const invoice = await Invoice.find(invoiceId)
  await invoice.markAsFailed()
  await notifyCustomer(customerId, 'billing_issue')
}

class CardDeclinedError extends Error {
  constructor(stripeCode) {
    super('Card was declined')
    this.code = stripeCode
  }
}
```

### `test-fixtures/auth.js`
```javascript
// grep for "authentication" finds NOTHING here
function verifyJWT(token) {
  return jwt.verify(token, process.env.SECRET)
}

async function requireLogin(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1]
  if (!token) return res.status(401).json({ error: 'Unauthorized' })
  req.user = verifyJWT(token)
  next()
}
```

---

## Expected POC Result

```bash
$ node poc/index.js search "payment error handling"

billing.js:2  (score: 0.891)
async function handleChargeFailure(customerId, invoiceId) {
  const invoice = await Invoice.find(invoiceId)
...

billing.js:9  (score: 0.847)
class CardDeclinedError extends Error {
  constructor(stripeCode) {
...

$ grep -r "payment error" test-fixtures/
# (no results)
```

**grep returns nothing. smart-grep returns exactly the right code.**

---

## Benchmark Plan (for README credibility)

Run both tools on a real 50k LOC codebase with 10 conceptual queries:

| Query | grep hits | smart-grep hits | Relevant found by smart-grep only |
|---|---|---|---|
| "payment error" | 2 | 8 | 6 |
| "user auth" | 5 | 12 | 7 |
| "cache miss" | 1 | 9 | 8 |
| "retry logic" | 0 | 6 | 6 |

---

## Next Steps (Production Rust)

1. Replace Node.js with Rust + `tree-sitter` crate
2. Replace in-memory cosine with `usearch` HNSW for million-file codebases
3. Incremental indexing — only re-embed changed files
4. `--watch` mode via `notify` crate
5. JSON output mode for Claude Code integration
