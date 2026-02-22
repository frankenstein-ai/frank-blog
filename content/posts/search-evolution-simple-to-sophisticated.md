+++
date = '2026-01-27T14:00:00-03:00'
draft = false
title = 'Search Evolution: From Simple Substring Matching to AI-Powered Hybrid Search'
+++

## The evolution of search

Search is the heart of a bookmark manager. If users can't find what they saved, the tool is useless. Over nine experiments, Frank Bookmark's search evolved from basic substring matching to a hybrid system combining relevance-ranked lexical search with semantic vector understanding.

This is the story of that evolution.

## Phase 1: Start simple (experiments 1-4)

### The first implementation: SQL LIKE

The initial search was straightforward:

```sql
SELECT * FROM pages
WHERE title ILIKE '%query%'
   OR content ILIKE '%query%'
ORDER BY created_at DESC
```

It matched any bookmark containing the search terms with case-insensitive substring matching. Simple, fast, and predictable. I learned that it was fast enough (under 50ms for 1,000 bookmarks), reliable, and easy to understand. Good enough to ship v1.0.

The limitations I accepted at the time: no semantic understanding, no relevance ranking, and results ordered by save date.

At this stage, I focused on getting the core functionality working: bookmark saving, storage, and basic retrieval. Simple keyword search was sufficient.

## Phase 2: Add semantic understanding (experiments 5-7)

### The problem with keywords only

Users started reporting: "I know I saved an article about performance optimization, but I can't find it."

Keyword search required remembering exact words from the title or content. If the article was titled "Making Your Website Faster," searching for "performance optimization" returned nothing.

### The solution: vector embeddings

I added semantic search using Transformers.js to run ML models in the browser, all-MiniLM-L6-v2 to generate 384-dimension embeddings, and sqlite-vec for native vector similarity search.

```javascript
async function semanticSearch(query) {
  // Generate query embedding
  const queryEmbedding = await model(query);

  // Vector similarity search
  const stmt = await db.prepare(`
    SELECT *, vec_distance_cosine(embedding, ?) AS distance
    FROM pages
    ORDER BY distance ASC
    LIMIT 10
  `);

  return await stmt.all([queryEmbedding]);
}
```

This unlocked conceptual matching instead of just keywords. "Performance optimization" now finds "Making Your Website Faster," and users can discover related content they wouldn't have found otherwise.

The model load takes 3-5 seconds (one-time), and subsequent searches come in under 200ms.

The real win: semantic search found bookmarks users knew they had but couldn't describe with exact keywords.

## Phase 3: Combine both approaches (experiment 5)

### The problem with semantic only

Semantic search introduced a new issue: sometimes it missed obvious exact matches.

Query: "React hooks"

Semantic results:
1. "Component Lifecycle in Modern React" (high semantic similarity)
2. "State Management Patterns" (related concepts)
3. "React Hooks Tutorial" (exact match, ranked lower)

Users expected the exact match first.

### The solution: hybrid search

I combined both approaches with weighted scores:

```javascript
async function hybridSearch(query) {
  // Run both searches
  const keywordResults = await keywordSearch(query);
  const semanticResults = await semanticSearch(query);

  // Combine with 50/50 weighting
  const combined = mergeAndRank(keywordResults, semanticResults, {
    keyword: 0.5,
    semantic: 0.5
  });

  return combined;
}
```

This gives exact matches high rank from the keyword component while still surfacing related content from the semantic component.

Query: "React hooks"

Hybrid results:
1. "React Hooks Tutorial" (exact match + semantically relevant)
2. "Complete Guide to React Hooks" (exact match)
3. "Component Lifecycle in React" (semantically relevant)

Exact matches first, related content after. That's what users expected.

## Phase 4: The relevance problem (experiment 9)

### Using the system revealed the flaw

After weeks of daily use, I noticed a pattern: keyword search results felt random.

Query: "React performance"

Results:
1. "My Blog Post" (saved yesterday, mentions "performance" once)
2. "JavaScript Tips" (saved last week, mentions "React" in passing)
3. "Optimizing React Performance" (saved last month, perfect match)

The most relevant result was buried at position 3 because it was older.

### The root cause

The keyword search treated all matches equally:

```sql
-- This says: "Does the query exist in this bookmark?"
WHERE title ILIKE '%query%' OR content ILIKE '%query%'

-- But not: "How relevant is this bookmark to the query?"
```

Results were sorted by `created_at`, not relevance.

### The realization

Professional search engines don't work this way. Google doesn't show results ordered by when pages were created. They rank by relevance.

I needed the same for keyword search.

## Phase 5: Relevance ranking with BM25 (experiment 9)

### The question

Can I add relevance ranking without external search engines, significant performance cost, or complex infrastructure?

### The answer: SQLite FTS5 + BM25

SQLite includes a full-text search extension with BM25 ranking built in. BM25 considers term frequency (how often does "React" appear?), inverse document frequency (how rare is "React" across all bookmarks?), document length (normalizing for short vs. long articles), and field weighting (title matches count more than content matches).

### The implementation

Step 1: Create FTS5 table

```sql
CREATE VIRTUAL TABLE pages_fts USING fts5(
  bookmark_id,
  title,
  summary,
  content,
  tags_text,
  tokenize = 'porter unicode61'
);
```

Step 2: Search with BM25 ranking

```sql
SELECT
  pages.*,
  bm25(pages_fts, 10.0, 5.0, 2.0, 1.0) AS score
FROM pages_fts
JOIN pages ON pages_fts.bookmark_id = pages.id
WHERE pages_fts MATCH ?
ORDER BY score DESC
```

The field weights are title at 10.0 (if it's in the title, it's probably what you want), summary at 5.0, content at 2.0, and tags at 1.0.

### The results

Same query, dramatically different results:

Query: "React performance"

Before (LIKE + date sort):
1. "My Blog Post" (newest)
2. "JavaScript Tips" (middle)
3. "Optimizing React Performance" (oldest)

After (BM25 + relevance rank):
1. "Optimizing React Performance" (perfect title match, high score)
2. "React Performance Guide" (excellent match)
3. "Advanced React Patterns" (related, mentions performance)

BM25 added about 5ms of latency compared to LIKE, but the result quality jumped dramatically.

### Bonus: Stemming and unicode

The `porter unicode61` tokenizer added features I didn't have before. Stemming means "running" matches "run," "runner," and "ran." "Optimizing" matches "optimize" and "optimization." Unicode normalization means "cafe" matches "cafe," "Cafe," and "CAFE," handling accents and case automatically.

### Advanced query syntax

FTS5 unlocked power user features:

```
"React hooks" - exact phrase
React AND hooks - both required
React OR Vue - either
React NOT class - has React but not class
reac* - prefix matching
```

## The current state: three-mode hybrid system

After nine experiments, Frank Bookmark search combines three modes.

Mode 1: Keyword (BM25-ranked) is fast (under 50ms), relevance-ranked, supports stemming, and offers advanced syntax. Best for exact matches.

Mode 2: Semantic (AI-powered) does conceptual matching, discovers related content, and understands language. Best for exploration.

Mode 3: Hybrid (combined) merges both:

```
Score = (BM25 score * 0.5) + (Vector similarity * 0.5)
```

Exact matches rank highly, related content gets surfaced, and it's the default mode.

## The evolution in numbers

| Phase | Technology | Performance | Capabilities |
|-------|-----------|-------------|--------------|
| 1 | SQL LIKE | 45ms | Substring matching |
| 2 | + Vector embeddings | 200ms | + Semantic understanding |
| 3 | + Hybrid mode | 300ms | + Combined ranking |
| 4 | + BM25 | 50ms (keyword) | + Relevance ranking |
| **Final** | **BM25 + Vectors + Hybrid** | **< 300ms** | **Full hybrid search** |

All running entirely in the browser. No servers, no cloud, no data leaving your device.

## What I learned about search

Starting simple and iterating worked well. LIKE search was good enough to ship v1.0. I didn't need perfect search on day one, and real use revealed what needed improvement.

Listening to user frustration was key. "I can't find my bookmarks" wasn't a data issue; it was a search quality issue. Those pain points guided every phase.

Combining approaches pays off. Neither keyword nor semantic search is perfect alone. Hybrid search leverages both: keyword for precision, semantic for recall.

Relevance matters more than recency. Users expect results ranked by relevance, not by save date. BM25 provides that.

Pushing computation into native code made a real difference. Both vector search (sqlite-vec) and text search (FTS5) taught me the same lesson: native SQL/WASM beats JavaScript by about 10x.

And once you have FTS5 and vector embeddings in place, you get phrase search, boolean operators, prefix matching, semantic discovery, and hybrid ranking with minimal additional code.

## Lessons for your search implementation

If you're building search, here's a rough phasing.

Phase 1: Ship with SQL LIKE or equivalent. Get core functionality working and learn from real use.

Phase 2: Add semantic when users start saying "I can't find it." Transformers.js makes this practical in the browser, though expect 3-5 seconds for model load.

Phase 3: Combine with hybrid. Don't make users choose modes; combine keyword and semantic, then tune weights based on use.

Phase 4: Add relevance ranking. Replace LIKE with FTS5/BM25. Minimal performance cost, dramatically better results.

At each phase, keep an eye on performance (under 300ms target), result quality (user feedback), and ease of use (default to hybrid).

## The path forward

Search continues to evolve. I'm currently experimenting with query intelligence (auto-detecting query type so "react hooks" routes to keyword mode while "articles about performance" routes to semantic mode), personalization (learning from click behavior and adjusting weights per user), and context awareness (factoring in time of search, current browsing context, and recently viewed bookmarks).

Each will be documented, tested, and shared.

## Wrapping up

Search evolved from simple substring matching to a hybrid system over nine experiments:

1. LIKE - Simple, works
2. + Vectors - Semantic understanding
3. + Hybrid - Best of both
4. + BM25 - Relevance ranking

The result is a bookmark search system that runs entirely in the browser. You don't need to build perfect search upfront. Start simple, ship, learn from use, and improve iteratively. Each phase built on the previous one.

The technology is accessible: SQLite FTS5 (built-in), Transformers.js (open source), sqlite-vec (free), Vite (bundler). The architecture is a Chrome Extension with a Service Worker, all client-side. And performance is sub-300ms hybrid search over 1,000+ bookmarks with zero servers.

Privacy-first AI search works, and it works well.

**Read more:**
- [The complete evolution story](/posts/frank-bookmark-evolution)
- [BM25 implementation details](/posts/bm25-fts5-search-relevance)
- [Three search modes explained](/posts/three-search-modes-bookmark-systems)
- [Original journey (Experiments 1-8)](/posts/frank-bookmark-journey)
