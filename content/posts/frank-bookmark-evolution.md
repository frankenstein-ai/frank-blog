+++
date = '2026-01-27T13:00:00-03:00'
draft = false
title = 'Frank Bookmark Evolution: From Prototype to Production'
+++

## The Journey Continues

When we [first built Frank Bookmark](/posts/frank-bookmark-journey), we documented eight experiments that took us from a DuckDB prototype to a production-ready Chrome extension with AI-powered search. That was just the beginning.

After shipping v1.0, we kept iterating, learning, and improving. This is the story of how Frank Bookmark evolved from a working proof-of-concept to a state-of-the-art search system.

## Experiment 9: The Search Quality Problem

A few weeks after launch, we noticed something troubling: users were struggling to find bookmarks they knew they had saved.

The problem wasn't the AI. Semantic search worked beautifully. The problem was **keyword search**.

### What We Discovered

Our keyword search used simple SQL `LIKE` operators:

```sql
SELECT * FROM pages
WHERE title ILIKE '%react hooks%'
   OR content ILIKE '%react hooks%'
ORDER BY created_at DESC
```

This has a fundamental flaw: **all matches are treated equally**.

**Real example:**

Query: "React hooks tutorial"

Results (ordered by save date):
1. "My Blog Post" (saved yesterday, contains "tutorial" once in a 5,000-word article)
2. "React Hooks Guide 2023" (saved last week, "React hooks" in title 3 times)
3. "Complete React Hooks Tutorial" (saved last month, perfect match)

The most relevant results were buried because they were older. Users expected results ranked by **how relevant they are**, not when they were saved.

### The Question

Can we implement relevance-based ranking for keyword search without compromising performance or adding external dependencies?

### The Hypothesis

SQLite's FTS5 (Full-Text Search) extension with BM25 ranking should provide production-grade relevance scoring while:
- Running entirely in the browser
- Maintaining sub-100ms performance
- Requiring minimal storage overhead
- Integrating seamlessly with our existing architecture

### The Implementation

We replaced `LIKE` queries with FTS5 + BM25:

**Step 1: Create Virtual FTS5 Table**

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

The `porter unicode61` tokenizer provides:
- **Stemming** - "running" matches "run", "runner", "ran"
- **Unicode normalization** - "café" matches "Cafe", "CAFÉ"

**Step 2: Sync with Triggers**

```sql
CREATE TRIGGER pages_fts_insert AFTER INSERT ON pages
BEGIN
  INSERT INTO pages_fts (bookmark_id, title, summary, tags_text)
  VALUES (NEW.id, NEW.title, NEW.summary,
    COALESCE(NEW.user_tags, '[]') || COALESCE(NEW.auto_tags, '[]'));
END;
```

Triggers keep the FTS5 index updated automatically.

**Step 3: BM25 Ranking with Field Weights**

```javascript
async function bm25Search(db, query) {
  const stmt = await db.prepare(`
    SELECT
      pages.*,
      bm25(pages_fts, 10.0, 5.0, 2.0, 1.0) AS score
    FROM pages_fts
    JOIN pages ON pages_fts.bookmark_id = pages.id
    WHERE pages_fts MATCH ?
    ORDER BY score DESC
    LIMIT 20
  `);
  return await stmt.all([query]);
}
```

**Field weights:**
- Title: 10.0 (highest priority)
- Summary: 5.0
- Content: 2.0
- Tags: 1.0 (lowest priority)

### The Results

Same query, dramatically different results:

Query: "React hooks tutorial"

**Before (LIKE):**
1. "My Blog Post" (newest, low relevance)
2. "React Hooks Guide 2023" (older, high relevance)
3. "Complete React Hooks Tutorial" (oldest, perfect match)

**After (BM25):**
1. "Complete React Hooks Tutorial" (perfect title match, high term frequency)
2. "React Hooks Guide 2023" (excellent title match, high frequency)
3. "Advanced React Hooks" (good match, multiple occurrences)

BM25 surfaces the most relevant results first, regardless of save date.

**Performance:**
- LIKE search: ~45ms
- BM25 search: ~50ms
- Only 5ms slower for dramatically better results

**Storage:**
- FTS5 index: ~15% database size increase
- Totally worth it for the quality improvement

### Bonus Features Unlocked

FTS5 enabled advanced search syntax we didn't have before:

**Phrase search:**
```
"React hooks" - exact phrase only
```

**Boolean operators:**
```
React AND hooks - both required
React OR Vue - either one
React NOT class - has React but not class
```

**Prefix search:**
```
reac* - matches react, reactive, reacting
```

These features give power users precise control over their searches.

## Integrating with Hybrid Search

Our hybrid search combined keyword and semantic results:

```
Hybrid = (LIKE results) + (Vector results)
```

With BM25, hybrid search became:

```
Hybrid = (BM25 results) + (Vector results)
```

Now we're combining:
- **Production-grade lexical ranking** (BM25)
- **AI-powered semantic understanding** (vector embeddings)

This is the same approach used by enterprise search systems, running entirely in your browser.

## The Evolution So Far

Looking back at our technology evolution:

| Experiment | Component | Technology | Reason for Change |
|------------|-----------|------------|-------------------|
| 1-2 | Storage | DuckDB-Wasm | Initial attempt, Service Worker issues |
| 3 | Storage | SQL.js | Extension compatibility |
| 4-7 | Vector Search | sqlite-vec | Native vector functions, 10x faster |
| 8 | Build/Deploy | Vite + GitHub Actions | Automated releases |
| **9** | **Keyword Search** | **FTS5 + BM25** | **Relevance ranking** |

Each change addressed a real limitation discovered through use.

## Performance at Scale

With 1,000+ bookmarks, all three search modes remain fast:

| Mode | Performance | Notes |
|------|-------------|-------|
| Keyword (BM25) | < 50ms | Relevance-ranked |
| Semantic (AI) | < 200ms | After model load |
| Hybrid | < 300ms | Best of both |

All sub-second, all running entirely client-side.

## What We Learned

### 1. Ship First, Then Iterate

We shipped v1.0 with LIKE search because it worked. Using the extension revealed the relevance problem. If we had tried to build the "perfect" search upfront, we'd still be planning.

### 2. Real Use Reveals Real Problems

The relevance issue only became clear when we actually used the extension with real bookmarks. Testing with sample data didn't expose it.

### 3. Incremental Improvements Compound

Each experiment built on the previous:
- Experiment 7: sqlite-vec for vector search
- Experiment 9: FTS5 for keyword search
- Both integrate into hybrid search

We didn't rebuild from scratch. We improved one component at a time.

### 4. Native Performance Matters

Both sqlite-vec (Experiment 7) and FTS5 (Experiment 9) taught us the same lesson: push computation into native code (SQL/WASM) rather than JavaScript. The performance difference is 10x or more.

### 5. Document Everything

Having detailed notes from Experiments 1-8 made Experiment 9 straightforward. We knew our architecture, understood the trade-offs, and could evaluate FTS5 quickly.

## Current State: Production-Ready

Frank Bookmark now features:

**Search:**
- BM25-ranked keyword search with stemming
- AI-powered semantic search with vector embeddings
- Hybrid mode combining both approaches
- Advanced query syntax (phrases, boolean, prefix)

**Architecture:**
- Self-contained Chrome extension
- Background Service Worker for AI
- sqlite-vec + FTS5 for all search modes
- OPFS persistence across sessions

**Performance:**
- Sub-50ms keyword search
- Sub-200ms semantic search
- Sub-300ms hybrid search
- All with 1,000+ bookmarks

**Privacy:**
- Zero data sent to servers
- Fully offline-capable
- All processing local
- No analytics or tracking

## What's Next?

The journey continues. Current areas of exploration:

**Query Intelligence:**
- Auto-detect query type (keyword vs semantic vs boolean)
- Query suggestions based on search history
- Spell correction and "did you mean?"

**User Experience:**
- Relevance feedback (learn from clicks)
- Personalized ranking weights
- Save context (where you were, what you were doing)

**Advanced Features:**
- Tags and folders
- Bulk operations
- Export/import
- Cross-device sync (optional)

Each will be its own experiment, documented and shared.

## Lessons for Your Projects

If you're building browser-based applications:

**Start Simple:**
- LIKE search worked well enough for v1.0
- Shipped quickly, learned from real use
- Improved iteratively

**Use Native Performance:**
- SQLite FTS5 for text search
- WASM for vector operations
- 10x faster than pure JavaScript

**Document Your Journey:**
- Future you will thank past you
- Helps others learn from your work
- Makes debugging easier

**Iterate Based on Use:**
- Don't over-engineer upfront
- Ship, learn, improve
- Real use reveals real problems

## Try It Yourself

Frank Bookmark is open source. You can:
- See the complete implementation
- Read all 9 experiment notebooks
- Review 6 insight memos
- Clone and modify for your needs

The BM25 upgrade proves that browser-based AI applications can evolve to production-grade quality while maintaining privacy and performance.

**Read more:**
- [Initial journey (Experiments 1-8)](/posts/frank-bookmark-journey)
- [BM25 implementation deep-dive](/posts/bm25-fts5-search-relevance)
- [Three search modes explained](/posts/three-search-modes-bookmark-systems)

## The Research Mindset

This is what R&D looks like in practice:
- Ship working code
- Use it yourself
- Notice problems
- Research solutions
- Experiment with improvements
- Document findings
- Share learnings

Nine experiments later, we have a Chrome extension that:
- Runs complete ML workflows in the browser
- Provides enterprise-grade search quality
- Maintains perfect privacy
- Costs zero to run

And we documented every step so you can build on our work.

The evolution continues.
