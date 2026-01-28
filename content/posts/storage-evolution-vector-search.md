+++
date = '2026-01-26T11:00:00-03:00'
draft = false
title = 'Storage Evolution: Finding the Right Database for Vector Search'
+++

## The Storage Dilemma

Building browser-based AI applications requires solving a critical problem: how do you store and search vector embeddings efficiently? The choice of storage technology can make or break your application.

We tested three solutions while building Frank Bookmark. Each had different trade-offs. Here's what we learned.

## The Contenders

### DuckDB-Wasm: Powerful but Incompatible

**What it promised:**
- Full-featured SQL database in the browser
- Native vector functions
- Excellent performance
- Modern architecture

**What we found:**

The good:
- Works beautifully in web apps (localhost:3000)
- Powerful analytics capabilities
- Clean API design

The problems:
- **Does not work reliably in Chrome Service Workers**
- WASM file loading issues in extension context
- Large WASM files (~34MB)
- Vector functions documented but not working in practice
- Service Worker compatibility is a dealbreaker

**Verdict:** Great for web apps, unusable for extensions.

### SQL.js: Reliable but Limited

**What it promised:**
- Pure SQLite compiled to WASM
- Mature and stable
- Known compatibility

**What we found:**

The good:
- Works reliably in Chrome Extensions
- No Service Worker issues
- Mature and well-tested
- Familiar SQLite API

The problems:
- No native vector similarity functions
- Requires manual cosine similarity calculation
- Manual calculation is CPU-intensive and slow
- In-memory by default (needs persistence setup)

**Verdict:** Reliable foundation, but vector search is too slow at scale.

### sqlite-vec: The Sweet Spot

**What it promised:**
- SQLite extension for vector similarity
- Native vector functions
- Extension compatibility

**What we found:**

The good:
- Works perfectly in Chrome Extensions
- Native `vec_distance_cosine()` function
- **10x faster** than manual calculation
- Scales well to 1,000+ vectors
- SQL-based queries combine keyword and vector search
- Integrates with standard SQLite operations

The challenges:
- Requires loading WASM extension files
- Less documentation than SQL.js
- Newer, less established

**Verdict:** Best choice for browser-based AI with vector search.

## Performance Comparison

Testing with 1,000 bookmarks:

| Storage | Keyword Search | Vector Search | Notes |
|---------|---------------|---------------|-------|
| DuckDB-Wasm | N/A | N/A | Doesn't work in extensions |
| SQL.js | < 100ms | ~2000ms | Manual cosine similarity |
| sqlite-vec | < 100ms | < 200ms | Native functions |

The 10x performance difference matters. At 1,000+ items, manual calculation becomes unusable. Native functions keep the UI responsive.

## Technical Deep Dive

### Why DuckDB-Wasm Fails in Extensions

Service Workers have restrictions:
- Limited WASM loading capabilities
- Different execution context than web pages
- Stricter security boundaries
- Extension manifest limitations

DuckDB-Wasm was designed for web apps, not extension Service Workers. The incompatibilities are fundamental, not easily worked around.

### Why Manual Vector Similarity is Slow

Cosine similarity calculation:

```javascript
function cosineSimilarity(a, b) {
  let dotProduct = 0;
  let normA = 0;
  let normB = 0;

  for (let i = 0; i < a.length; i++) {
    dotProduct += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }

  return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
}
```

With 384-dimensional embeddings and 1,000 bookmarks:
- 384,000 floating-point operations
- Pure JavaScript execution
- No vectorization
- Runs in main thread

Native SQL functions:
- Compiled C code
- SIMD optimizations
- Runs in WASM
- 10x faster

### Why sqlite-vec Wins

Native vector operations in SQL:

```sql
SELECT
  *,
  vec_distance_cosine(embedding, ?) AS distance
FROM pages
ORDER BY distance ASC
LIMIT 10
```

Clean, fast, and integrates with other SQL operations:

```sql
SELECT
  *,
  (vec_distance_cosine(embedding, ?) * 0.5 + keyword_score * 0.5) AS score
FROM (
  SELECT
    *,
    CASE
      WHEN title ILIKE ? OR content ILIKE ?
      THEN 1.0
      ELSE 0.0
    END AS keyword_score
  FROM pages
) subquery
ORDER BY score ASC
LIMIT 10
```

This hybrid search (combining keyword and semantic) runs in a single SQL query. Try that with manual JavaScript calculations.

## When to Use Each

### Use DuckDB-Wasm When:
- Building web apps (not extensions)
- Need advanced analytics
- Large-scale data processing
- Web Worker context (not Service Worker)

### Use SQL.js When:
- Building Chrome Extensions
- Need SQL without vector search
- Simple storage, moderate datasets
- Willing to implement manual vector similarity
- Backward compatibility required

### Use sqlite-vec When:
- Building Chrome Extensions with vector search
- Need native vector similarity performance
- Large-scale vector similarity queries (1,000+)
- Hybrid search (keyword + semantic)
- Modern browser targets

## Implementation Guide

### Setting Up sqlite-vec

```javascript
import { createSQLiteVec } from '@dao-xyz/sqlite3-vec';

// Initialize
const sqlite = await createSQLiteVec();
const db = await sqlite.open(':memory:');

// Create table with vector column
await db.exec(`
  CREATE TABLE pages (
    id INTEGER PRIMARY KEY,
    url TEXT,
    title TEXT,
    content TEXT,
    embedding BLOB
  )
`);

// Insert with embedding
await db.run(
  'INSERT INTO pages (url, title, content, embedding) VALUES (?, ?, ?, ?)',
  [url, title, content, embeddingBlob]
);

// Vector search
const results = await db.all(`
  SELECT *, vec_distance_cosine(embedding, ?) AS distance
  FROM pages
  ORDER BY distance ASC
  LIMIT 10
`, [queryEmbedding]);
```

### Persistence with OPFS

```javascript
// Save database
const data = db.export();
const handle = await navigator.storage.getDirectory();
const fileHandle = await handle.getFileHandle('database.db', { create: true });
const writable = await fileHandle.createWritable();
await writable.write(data);
await writable.close();

// Load database
const handle = await navigator.storage.getDirectory();
const fileHandle = await handle.getFileHandle('database.db');
const file = await fileHandle.getFile();
const data = await file.arrayBuffer();
const db = await sqlite.open(new Uint8Array(data));
```

## Lessons Learned

### 1. Compatibility Trumps Features

DuckDB-Wasm had better features, but they didn't matter because it didn't work where we needed it. Always verify compatibility in your target environment first.

### 2. Native Performance Matters

The 10x performance difference between manual and native vector similarity was the difference between usable and unusable. When working with vectors at scale, native implementations are essential.

### 3. SQL is a Superpower

Being able to combine keyword and vector search in a single SQL query is incredibly powerful. Don't underestimate the value of SQL integration.

### 4. Test at Scale Early

Performance characteristics change dramatically with scale. Test with realistic data volumes (1,000+ items) early to avoid costly pivots later.

## Recommendations

**For new browser-based AI projects:**

1. **Start with sqlite-vec** if you need vector search
2. **Use OPFS for persistence** to survive restarts
3. **Test in your target environment** (extension vs. web app)
4. **Measure at scale** with realistic data volumes

**For existing SQL.js projects:**

1. Measure your vector search performance
2. If < 100 items, manual calculation may be fine
3. If > 500 items, consider migrating to sqlite-vec
4. Migration path is straightforward (both use SQLite)

**Avoid DuckDB-Wasm for extensions** until Service Worker compatibility is confirmed.

## Future Considerations

Storage technology is evolving:

- **WebGPU compute shaders** - Could accelerate vector operations further
- **Native vector support in SQLite** - May eventually be standard
- **Browser database APIs** - New standards may emerge
- **WASM optimization** - Performance keeps improving

For now, sqlite-vec provides the best balance of compatibility, performance, and features for browser-based AI applications.

## Conclusion

Choosing the right storage technology is critical for browser-based AI. We learned this the hard way by testing three different solutions.

**DuckDB-Wasm** - Powerful but incompatible with extensions
**SQL.js** - Reliable but too slow for vector search at scale
**sqlite-vec** - The sweet spot for browser-based AI

If you're building browser-based AI with vector search, start with sqlite-vec. You'll save yourself weeks of experimentation and get native performance from day one.
