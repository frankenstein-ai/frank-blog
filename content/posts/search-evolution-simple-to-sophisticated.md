+++
date = '2026-01-27T14:00:00-03:00'
draft = false
title = 'Search Evolution: From Simple Substring Matching to AI-Powered Hybrid Search'
+++

## The Evolution of Search

Search is the heart of a bookmark manager. If users can't find what they saved, the tool is useless. Over nine experiments, Frank Bookmark's search capabilities evolved from basic substring matching to a sophisticated hybrid system combining relevance-ranked lexical search with AI-powered semantic understanding.

This is the story of that evolution.

## Phase 1: Start Simple (Experiments 1-4)

### The First Implementation: SQL LIKE

Our initial search was straightforward:

```sql
SELECT * FROM pages
WHERE title ILIKE '%query%'
   OR content ILIKE '%query%'
ORDER BY created_at DESC
```

**What it did:**
- Matched any bookmark containing the search terms
- Case-insensitive substring matching
- Simple, fast, predictable

**What we learned:**
- Fast enough (< 50ms for 1,000 bookmarks)
- Works reliably
- Easy to implement and understand
- Good enough to ship v1.0

**Limitations we accepted:**
- No semantic understanding
- No relevance ranking
- Results ordered by save date

At this stage, we focused on getting the core functionality working: bookmark saving, storage, and basic retrieval. Simple keyword search was sufficient.

## Phase 2: Add Semantic Understanding (Experiments 5-7)

### The Problem with Keywords Only

Users started reporting: "I know I saved an article about performance optimization, but I can't find it."

Keyword search required remembering exact words from the title or content. If the article was titled "Making Your Website Faster," searching for "performance optimization" returned nothing.

### The Solution: Vector Embeddings

We added semantic search using:
- **Transformers.js** - Run ML models in the browser
- **all-MiniLM-L6-v2** - Generate 384-dimension embeddings
- **sqlite-vec** - Native vector similarity search

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

**What it unlocked:**
- Conceptual matching, not just keywords
- "performance optimization" finds "Making Your Website Faster"
- Discovery of related content

**Performance:**
- Model load: 3-5 seconds (one-time)
- Search: < 200ms after model loaded
- Fast enough for production

**The breakthrough:**
Semantic search found bookmarks users knew they had but couldn't describe with exact keywords.

## Phase 3: Combine Both Approaches (Experiment 5)

### The Problem with Semantic Only

Semantic search introduced a new issue: sometimes it missed obvious exact matches.

Query: "React hooks"

Semantic results:
1. "Component Lifecycle in Modern React" (high semantic similarity)
2. "State Management Patterns" (related concepts)
3. "React Hooks Tutorial" (exact match, ranked lower)

Users expected the exact match first.

### The Solution: Hybrid Search

We combined both approaches with weighted scores:

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

**What it provided:**
- Exact matches ranked highly (keyword component)
- Related content surfaced (semantic component)
- Best of both worlds

**User experience:**
Query: "React hooks"

Hybrid results:
1. "React Hooks Tutorial" (exact match + semantically relevant)
2. "Complete Guide to React Hooks" (exact match)
3. "Component Lifecycle in React" (semantically relevant)

Perfect. Exact matches first, related content after.

## Phase 4: The Relevance Problem (Experiment 9)

### Using the System Revealed the Flaw

After weeks of daily use, we noticed a pattern: keyword search results felt random.

Query: "React performance"

Results:
1. "My Blog Post" (saved yesterday, mentions "performance" once)
2. "JavaScript Tips" (saved last week, mentions "React" in passing)
3. "Optimizing React Performance" (saved last month, perfect match)

The most relevant result was buried at position 3 because it was older.

### The Root Cause

Our keyword search treated all matches equally:

```sql
-- This says: "Does the query exist in this bookmark?"
WHERE title ILIKE '%query%' OR content ILIKE '%query%'

-- But not: "How relevant is this bookmark to the query?"
```

Results were sorted by `created_at`, not relevance.

### The Realization

Professional search engines don't work this way. Google doesn't show results ordered by when pages were created. They rank by **relevance**.

We needed the same for keyword search.

## Phase 5: Relevance Ranking with BM25 (Experiment 9)

### The Question

Can we add relevance ranking without:
- External search engines
- Significant performance cost
- Complex infrastructure

### The Answer: SQLite FTS5 + BM25

SQLite includes a full-text search extension with BM25 ranking built in.

**What BM25 considers:**
1. **Term Frequency** - How often does "React" appear?
2. **Inverse Document Frequency** - How rare is "React" across all bookmarks?
3. **Document Length** - Normalize for short vs. long articles
4. **Field Weighting** - Title matches > content matches

### The Implementation

**Step 1: Create FTS5 table**

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

**Step 2: Search with BM25 ranking**

```sql
SELECT
  pages.*,
  bm25(pages_fts, 10.0, 5.0, 2.0, 1.0) AS score
FROM pages_fts
JOIN pages ON pages_fts.bookmark_id = pages.id
WHERE pages_fts MATCH ?
ORDER BY score DESC
```

**Field weights:**
- Title: 10.0 (if it's in the title, it's probably what you want)
- Summary: 5.0
- Content: 2.0
- Tags: 1.0

### The Results

Same query, dramatically different results:

Query: "React performance"

**Before (LIKE + date sort):**
1. "My Blog Post" (newest)
2. "JavaScript Tips" (middle)
3. "Optimizing React Performance" (oldest)

**After (BM25 + relevance rank):**
1. "Optimizing React Performance" (perfect title match, high score)
2. "React Performance Guide" (excellent match)
3. "Advanced React Patterns" (related, mentions performance)

**Performance:**
- LIKE: ~45ms
- BM25: ~50ms
- 5ms slower, 100x better results

### Bonus: Stemming and Unicode

The `porter unicode61` tokenizer added features we didn't have:

**Stemming:**
- "running" matches "run", "runner", "ran"
- "optimizing" matches "optimize", "optimization"

**Unicode normalization:**
- "café" matches "cafe", "Cafe", "CAFÉ"
- Handles accents and case automatically

### Advanced Query Syntax

FTS5 unlocked power user features:

```
"React hooks" - exact phrase
React AND hooks - both required
React OR Vue - either
React NOT class - has React but not class
reac* - prefix matching
```

## The Current State: Three-Mode Hybrid System

After nine experiments, Frank Bookmark search combines:

### Mode 1: Keyword (BM25-Ranked)
- Fast (< 50ms)
- Relevance-ranked
- Stemming support
- Advanced syntax
- Best for exact matches

### Mode 2: Semantic (AI-Powered)
- Conceptual matching
- Discovers related content
- Language understanding
- Best for exploration

### Mode 3: Hybrid (Combined)
```
Score = (BM25 score × 0.5) + (Vector similarity × 0.5)
```
- Exact matches ranked highly
- Related content surfaced
- Best overall experience
- **Default mode**

## The Evolution in Numbers

| Phase | Technology | Performance | Capabilities |
|-------|-----------|-------------|--------------|
| 1 | SQL LIKE | 45ms | Substring matching |
| 2 | + Vector embeddings | 200ms | + Semantic understanding |
| 3 | + Hybrid mode | 300ms | + Combined ranking |
| 4 | + BM25 | 50ms (keyword) | + Relevance ranking |
| **Final** | **BM25 + Vectors + Hybrid** | **< 300ms** | **State of the art** |

All running entirely in the browser. No servers, no cloud, no data leaving your device.

## What We Learned About Search

### 1. Start Simple, Iterate

LIKE search was good enough to ship v1.0. We didn't need perfect search on day one. Real use revealed what needed improvement.

### 2. Listen to User Frustration

"I can't find my bookmarks" wasn't a data issue—it was a search quality issue. User pain points guide improvements.

### 3. Combine Approaches

Neither keyword nor semantic search is perfect alone. Hybrid search leverages both strengths:
- Keyword: Precision
- Semantic: Recall
- Hybrid: Balance

### 4. Relevance Matters

Users expect results ranked by relevance, not by arbitrary criteria like save date. BM25 provides this.

### 5. Native Performance Wins

Both vector search (sqlite-vec) and text search (FTS5) taught us: push computation into native code (SQL/WASM), not JavaScript. 10x performance difference.

### 6. Advanced Features for Free

Once you have FTS5 and vector embeddings, you unlock:
- Phrase search
- Boolean operators
- Prefix matching
- Semantic discovery
- Hybrid ranking

All with minimal additional code.

## Lessons for Your Search Implementation

If you're building search:

**Phase 1: Ship with Simple**
- SQL LIKE or equivalent
- Get core functionality working
- Learn from real use

**Phase 2: Add Semantic When Needed**
- Use when users say "I can't find it"
- Transformers.js makes this practical
- Expect 3-5s model load

**Phase 3: Combine with Hybrid**
- Don't make users choose modes
- Combine keyword + semantic
- Tune weights based on use

**Phase 4: Add Relevance Ranking**
- Replace LIKE with FTS5/BM25
- Minimal performance cost
- Dramatically better results

**Monitor at Each Phase:**
- Performance (< 300ms target)
- Result quality (user feedback)
- Ease of use (default to hybrid)

## The Path Forward

Search continues to evolve. Current experiments:

**Query Intelligence:**
- Auto-detect query type
- "react hooks" → keyword mode
- "articles about performance" → semantic mode

**Personalization:**
- Learn from click behavior
- Adjust weights per user
- "This user prefers semantic results"

**Context Awareness:**
- Time of search
- Current browsing context
- Recently viewed bookmarks

Each will be documented, tested, and shared.

## Conclusion

Search evolved from simple substring matching to a production-grade hybrid system over nine experiments:

1. **LIKE** - Simple, works
2. **+ Vectors** - Semantic understanding
3. **+ Hybrid** - Best of both
4. **+ BM25** - Relevance ranking

The result: a bookmark search system that rivals enterprise solutions, running entirely in your browser.

**Key insight:** You don't need to build perfect search upfront. Start simple, ship, learn from use, and improve iteratively. Each phase built on the previous, compounding improvements.

The technology is accessible:
- SQLite FTS5 (built-in)
- Transformers.js (open source)
- sqlite-vec (free)
- Vite (bundler)

The architecture works:
- Chrome Extension
- Service Worker
- Client-side only

The performance is there:
- Sub-300ms hybrid search
- 1,000+ bookmarks
- Zero servers

Privacy-first AI search is not just possible—it's production-ready.

**Read more:**
- [The complete evolution story](/posts/frank-bookmark-evolution)
- [BM25 implementation details](/posts/bm25-fts5-search-relevance)
- [Three search modes explained](/posts/three-search-modes-bookmark-systems)
- [Original journey (Experiments 1-8)](/posts/frank-bookmark-journey)

This is what iterative R&D looks like: ship, learn, improve, document, repeat.
