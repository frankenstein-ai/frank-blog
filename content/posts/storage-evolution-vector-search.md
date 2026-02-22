+++
date = '2026-01-26T11:00:00-03:00'
draft = false
title = 'Storage Evolution: Finding the Right Database for Vector Search'
+++

## The storage dilemma

Building browser-based AI applications means solving a real problem: how do you store and search vector embeddings efficiently? The storage technology you pick shapes everything that follows.

I tested three solutions while building Frank Bookmark. Each had different trade-offs.

## The contenders

### DuckDB-Wasm: powerful but incompatible

DuckDB-Wasm promised a full-featured SQL database in the browser with native vector functions and solid performance.

In practice, it works well in web apps served from localhost:3000. The analytics capabilities are strong and the API is clean. But for a Chrome extension, the problems were deal-breakers: it does not work reliably in Chrome Service Workers. WASM file loading breaks in the extension context, the files are large (~34MB), and the vector functions that appear in the docs didn't actually work. Service Worker compatibility alone ruled it out.

Great for web apps. Unusable for extensions.

### SQL.js: reliable but limited

SQL.js is pure SQLite compiled to WASM. It's mature, stable, and has a track record of compatibility.

It works reliably in Chrome Extensions with no Service Worker issues. The SQLite API is familiar and well-tested. The catch is there are no native vector similarity functions. You have to calculate cosine similarity manually in JavaScript, which is CPU-intensive and slow. It's also in-memory by default, so you need to set up persistence yourself.

A reliable foundation, but vector search gets too slow once you have more than a few hundred items.

### sqlite-vec: the right fit

sqlite-vec is a SQLite extension for vector similarity search. It works in Chrome Extensions, provides a native `vec_distance_cosine()` function, and runs about 10x faster than manual calculation. It scales well to 1,000+ vectors, and because queries are still SQL, you can combine keyword and vector search naturally.

The trade-offs are real but manageable: you need to load WASM extension files, documentation is thinner than SQL.js, and it's a newer project. For browser-based AI with vector search, though, it was the clear winner.

## Performance comparison

Testing with 1,000 bookmarks:

| Storage | Keyword Search | Vector Search | Notes |
|---------|---------------|---------------|-------|
| DuckDB-Wasm | N/A | N/A | Doesn't work in extensions |
| SQL.js | < 100ms | ~2000ms | Manual cosine similarity |
| sqlite-vec | < 100ms | < 200ms | Native functions |

That 10x performance gap matters. At 1,000+ items, manual calculation makes the UI feel unresponsive. Native functions keep things snappy.

## Technical deep dive

### Why DuckDB-Wasm fails in extensions

Service Workers have restrictions: limited WASM loading capabilities, a different execution context than web pages, stricter security boundaries, and extension manifest limitations. DuckDB-Wasm was designed for web apps, not extension Service Workers. These incompatibilities are fundamental, not something you can easily work around.

### Why manual vector similarity is slow

Here's the cosine similarity calculation:

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

With 384-dimensional embeddings and 1,000 bookmarks, that's 384,000 floating-point operations in pure JavaScript with no vectorization, running in the main thread.

Native SQL functions, by contrast, run compiled C code with SIMD optimizations inside WASM. The result is roughly 10x faster.

### Why sqlite-vec wins

Native vector operations live right in your SQL:

```sql
SELECT
  *,
  vec_distance_cosine(embedding, ?) AS distance
FROM pages
ORDER BY distance ASC
LIMIT 10
```

And you can combine keyword and vector search in a single query:

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

This hybrid search runs as a single SQL statement. Try doing that with manual JavaScript calculations.

## When to use each

DuckDB-Wasm fits web apps (not extensions) where you need advanced analytics or large-scale data processing in a Web Worker context.

SQL.js works for Chrome Extensions that need SQL but not vector search, with moderate datasets. It's also the right pick if backward compatibility matters or if you're willing to implement manual vector similarity for small collections.

sqlite-vec is the one to reach for when you're building Chrome Extensions with vector search, need native vector similarity performance, have 1,000+ items, want hybrid search (keyword + semantic), or are targeting modern browsers.

## Implementation guide

### Setting up sqlite-vec

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

## What I learned

Compatibility beats features. DuckDB-Wasm had the better feature set, but none of that mattered because it didn't work in the environment I needed. Always verify compatibility in your target environment first.

The 10x performance gap between manual and native vector similarity was the difference between something usable and something that felt broken. When working with vectors at any real scale, native implementations are not optional.

I also underestimated how valuable it is to have everything in SQL. Being able to combine keyword and vector search in a single query simplified the codebase significantly. And testing with realistic data volumes early on (1,000+ items) saved me from a painful late-stage migration.

## Recommendations

For new browser-based AI projects, start with sqlite-vec if you need vector search. Use OPFS for persistence so data survives restarts. Test in your actual target environment (extension vs. web app) and measure with realistic data volumes.

For existing SQL.js projects, measure your vector search performance first. If you have fewer than 100 items, manual calculation may be fine. Above 500, consider migrating to sqlite-vec. The migration path is straightforward since both use SQLite.

Avoid DuckDB-Wasm for extensions until Service Worker compatibility is confirmed.

## What comes next

Storage technology in this space is still moving. WebGPU compute shaders could accelerate vector operations further. Native vector support may eventually land in SQLite itself. New browser database APIs may emerge, and WASM performance keeps improving.

For now, sqlite-vec provides the best balance of compatibility, performance, and features for browser-based AI applications.

## Wrapping up

I learned the hard way that choosing the right storage technology matters. DuckDB-Wasm is powerful but incompatible with extensions. SQL.js is reliable but too slow for vector search at scale. sqlite-vec hit the right balance for browser-based AI.

If you're building something similar, start with sqlite-vec. It'll save you weeks of experimentation.
