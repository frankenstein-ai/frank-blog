+++
date = '2026-01-27T13:00:00-03:00'
draft = false
title = 'Frank Bookmark Evolution: From Prototype to Production'
+++

## Iterating after launch

When I [first built Frank Bookmark](/posts/frank-bookmark-journey), I documented eight experiments that took us from a DuckDB prototype to a working Chrome extension with AI-powered search. That was just the beginning.

After shipping v1.0, I kept iterating and improving. This is the story of how Frank Bookmark's search went from "it works" to "it works well."

## Experiment 9: The search quality problem

A few weeks after launch, I noticed something troubling: finding bookmarks I knew I had saved was harder than it should be.

The problem wasn't the AI. Semantic search worked well. The problem was keyword search.

### What I discovered

The keyword search used simple SQL `LIKE` operators:

```sql
SELECT * FROM pages
WHERE title ILIKE '%react hooks%'
   OR content ILIKE '%react hooks%'
ORDER BY created_at DESC
```

This has a fundamental flaw: all matches are treated equally.

Here's a real example.

Query: "React hooks tutorial"

Results (ordered by save date):
1. "My Blog Post" (saved yesterday, contains "tutorial" once in a 5,000-word article)
2. "React Hooks Guide 2023" (saved last week, "React hooks" in title 3 times)
3. "Complete React Hooks Tutorial" (saved last month, perfect match)

The most relevant results were buried because they were older. Users expected results ranked by relevance, not by save date.

### The question

Can I implement relevance-based ranking for keyword search without hurting performance or adding external dependencies?

### The hypothesis

SQLite's FTS5 (Full-Text Search) extension with BM25 ranking should give us good relevance scoring while running entirely in the browser, keeping sub-100ms performance, requiring minimal storage overhead, and fitting into the existing architecture.

### The implementation

I replaced `LIKE` queries with FTS5 + BM25.

First, a virtual FTS5 table:

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

The `porter unicode61` tokenizer handles stemming ("running" matches "run", "runner", "ran") and Unicode normalization ("cafe" matches "Cafe", "CAFE").

Then, triggers to keep the index in sync:

```sql
CREATE TRIGGER pages_fts_insert AFTER INSERT ON pages
BEGIN
  INSERT INTO pages_fts (bookmark_id, title, summary, tags_text)
  VALUES (NEW.id, NEW.title, NEW.summary,
    COALESCE(NEW.user_tags, '[]') || COALESCE(NEW.auto_tags, '[]'));
END;
```

And finally, BM25 ranking with field weights:

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

The field weights: title gets 10.0 (highest), summary 5.0, content 2.0, tags 1.0 (lowest).

### The results

Same query, very different results:

Query: "React hooks tutorial"

Before (LIKE):
1. "My Blog Post" (newest, low relevance)
2. "React Hooks Guide 2023" (older, high relevance)
3. "Complete React Hooks Tutorial" (oldest, perfect match)

After (BM25):
1. "Complete React Hooks Tutorial" (perfect title match, high term frequency)
2. "React Hooks Guide 2023" (excellent title match, high frequency)
3. "Advanced React Hooks" (good match, multiple occurrences)

BM25 surfaces the most relevant results first, regardless of save date.

Performance stayed about the same: LIKE search ran at ~45ms, BM25 at ~50ms. Only 5ms slower for much better results. The FTS5 index adds roughly 15% to the database size, which is a fair trade.

### Bonus features unlocked

FTS5 also enabled search syntax I didn't have before:

Phrase search:
```
"React hooks" - exact phrase only
```

Boolean operators:
```
React AND hooks - both required
React OR Vue - either one
React NOT class - has React but not class
```

Prefix search:
```
reac* - matches react, reactive, reacting
```

These give power users more control over their searches.

## Integrating with hybrid search

The hybrid search previously combined keyword and semantic results:

```
Hybrid = (LIKE results) + (Vector results)
```

With BM25, it became:

```
Hybrid = (BM25 results) + (Vector results)
```

Now we're combining proper lexical ranking (BM25) with AI-powered semantic understanding (vector embeddings). This is how most search systems work, and it runs entirely in the browser.

## The evolution so far

Looking back at the technology evolution:

| Experiment | Component | Technology | Reason for Change |
|------------|-----------|------------|-------------------|
| 1-2 | Storage | DuckDB-Wasm | Initial attempt, Service Worker issues |
| 3 | Storage | SQL.js | Extension compatibility |
| 4-7 | Vector Search | sqlite-vec | Native vector functions, 10x faster |
| 8 | Build/Deploy | Vite + GitHub Actions | Automated releases |
| **9** | **Keyword Search** | **FTS5 + BM25** | **Relevance ranking** |

Each change addressed a real limitation discovered through use.

## Performance at scale

With 1,000+ bookmarks, all three search modes remain fast:

| Mode | Performance | Notes |
|------|-------------|-------|
| Keyword (BM25) | < 50ms | Relevance-ranked |
| Semantic (AI) | < 200ms | After model load |
| Hybrid | < 300ms | Best of both |

All sub-second, all running entirely client-side.

## What I learned

I shipped v1.0 with LIKE search because it worked. Using the extension with real bookmarks revealed the relevance problem. If I had tried to build the "perfect" search upfront, I'd still be planning. Testing with sample data didn't expose the issue; real use did.

Each experiment also built on the previous ones. Experiment 7 gave us sqlite-vec for vector search, experiment 9 added FTS5 for keyword search, and both feed into hybrid search. I didn't rebuild from scratch. I improved one component at a time.

Both sqlite-vec and FTS5 taught me the same thing: push computation into native code (SQL/WASM) rather than JavaScript. The performance difference is 10x or more. And having detailed notes from experiments 1-8 made experiment 9 straightforward. I knew the architecture, understood the trade-offs, and could evaluate FTS5 quickly.

## Current state

Frank Bookmark now has:

- BM25-ranked keyword search with stemming
- AI-powered semantic search with vector embeddings
- Hybrid mode combining both approaches
- Advanced query syntax (phrases, boolean, prefix)
- Self-contained Chrome extension with background Service Worker for AI
- sqlite-vec + FTS5 for all search modes
- OPFS persistence across sessions
- Sub-50ms keyword, sub-200ms semantic, sub-300ms hybrid (with 1,000+ bookmarks)
- Zero data sent to servers, fully offline-capable, no analytics or tracking

## What's next

Areas I'm exploring:

- Auto-detecting query type (keyword vs semantic vs boolean)
- Query suggestions based on search history
- Spell correction
- Relevance feedback from clicks
- Tags, folders, and bulk operations
- Export/import
- Optional cross-device sync

Each will be its own experiment, documented and shared.

## Advice for similar projects

If you're building browser-based applications: start simple. LIKE search worked well enough for v1.0. I shipped quickly, learned from real use, and improved iteratively. Use native performance where you can. SQLite FTS5 for text search, WASM for vector operations. These are 10x faster than pure JavaScript equivalents. And document your journey, because future you will appreciate it.

## Try it

Frank Bookmark is open source. You can see the complete implementation, read all 9 experiment notebooks, review 6 insight memos, and clone it for your own needs.

Read more:
- [Initial journey (Experiments 1-8)](/posts/frank-bookmark-journey)
- [BM25 implementation deep-dive](/posts/bm25-fts5-search-relevance)
- [Three search modes explained](/posts/three-search-modes-bookmark-systems)

## The research mindset

This is what R&D looks like in practice: ship working code, use it yourself, notice problems, research solutions, experiment, document findings, and share what you learn.

Nine experiments in, we have a Chrome extension that runs ML workflows in the browser, provides solid search quality, keeps data local, and costs nothing to run. Every step is documented so others can build on it.
