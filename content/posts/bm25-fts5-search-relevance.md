+++
date = '2026-01-27T10:00:00-03:00'
draft = false
title = 'Beyond LIKE: Upgrading to BM25 for Better Search Relevance'
+++

## The keyword search problem

In our [initial Frank Bookmark implementation](/posts/three-search-modes-bookmark-systems), keyword search used simple SQL `LIKE` operators:

```sql
SELECT * FROM pages
WHERE title ILIKE '%react hooks%'
   OR content ILIKE '%react hooks%'
```

This works, but it has a real flaw: **all matches are treated equally**.

A page with "React hooks" in the title three times gets the same relevance score as a page where those words appear once in a 5,000-word article. The results? Ordered by save date, not by relevance to your query.

Users expect search results ranked by **how relevant they are**, not by when they were bookmarked.

## Enter BM25: the standard for text relevance

BM25 (Best Matching 25) is the standard algorithm for ranking full-text search results. It's used by Elasticsearch, Apache Lucene, and pretty much every modern search engine.

BM25 considers three things. Term Frequency (TF) measures how often the search term appears in the document. Inverse Document Frequency (IDF) measures how rare the term is across the entire collection. And document length normalization prevents short documents from having an unfair advantage.

The effect is that articles where your search terms appear frequently and prominently rank higher than articles where they just happen to exist.

## Implementing BM25 with SQLite FTS5

SQLite's FTS5 (Full-Text Search) extension provides native BM25 ranking. No external search engines, no complex setup, just SQL.

### Step 1: create a virtual FTS5 table

```sql
CREATE VIRTUAL TABLE pages_fts USING fts5(
  bookmark_id,          -- Foreign key to main table
  title,
  summary,
  content,
  tags_text,
  tokenize = 'porter unicode61'
);
```

Why `porter unicode61`? Porter stemming means "running" matches "run", "runner", and "ran". Unicode61 handles case-insensitivity and accents, so "cafe" matches "Cafe".

### Step 2: keep FTS5 in sync with triggers

```sql
CREATE TRIGGER pages_fts_insert AFTER INSERT ON pages
BEGIN
  INSERT INTO pages_fts (bookmark_id, title, summary, tags_text)
  VALUES (NEW.id, NEW.title, NEW.summary,
    COALESCE(NEW.user_tags, '[]') || COALESCE(NEW.auto_tags, '[]'));
END;

CREATE TRIGGER pages_fts_update AFTER UPDATE ON pages
BEGIN
  UPDATE pages_fts SET
    title = NEW.title,
    summary = NEW.summary,
    tags_text = COALESCE(NEW.user_tags, '[]') || COALESCE(NEW.auto_tags, '[]')
  WHERE bookmark_id = NEW.id;
END;
```

Triggers ensure the FTS5 index stays current automatically.

### Step 3: search with BM25 ranking

Replace `LIKE` queries with FTS5 `MATCH`:

```javascript
async function bm25Search(db, query) {
  const stmt = await db.prepare(`
    SELECT
      pages.*,
      bm25(pages_fts) AS score
    FROM pages_fts
    JOIN pages ON pages_fts.bookmark_id = pages.id
    WHERE pages_fts MATCH ?
    ORDER BY score DESC
    LIMIT 20
  `);
  return await stmt.all([query]);
}
```

The difference: `WHERE title ILIKE '%query%'` tells you whether a match exists or not. `WHERE pages_fts MATCH 'query' ORDER BY bm25(...)` tells you how good the match is.

## Field weighting: not all text is equal

BM25 allows boosting specific fields. If "React hooks" appears in the title, it's probably more relevant than if it appears deep in the content.

```javascript
async function weightedBM25Search(db, query) {
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

The weights here are: title at 10.0 (highest), summary at 5.0, content at 2.0, and tags at 1.0. This aligns with how people actually think about search: "If it's in the title, it's probably what I'm looking for."

## Real-world improvement

**Query:** "React hooks tutorial"

### Before (LIKE search)

Results ordered by save date:

1. "React Hooks Guide 2023" ✓
2. "React Tutorial: Introduction to Hooks" ✓
3. "React Performance Tutorial" ✗ (only has "tutorial", low relevance)
4. "My Thoughts on React" ✗ (only has "React", very low relevance)

### After (BM25 search)

Results ordered by relevance:

1. "React Hooks Guide 2023" ✓ (all terms in title, high frequency)
2. "React Tutorial: Introduction to Hooks" ✓ (all terms in title)
3. "My Thoughts on React Hooks" ✓ (all terms, high frequency in content)
4. "Advanced React Patterns" ✓ (related, has "React")

BM25 surfaces the most relevant results first, regardless of when they were saved.

## Stemming: smart matching out of the box

The Porter stemmer provides intelligent matching without extra code:

**Query:** "running node"

**Matches:**
- "running node"
- "run node"
- "running Node.js"
- "run Node.js"
- "ran Node"

This broadens search scope intelligently, finding variations without requiring synonyms or manual fuzzy matching.

## Unicode normalization: international support

The `unicode61` tokenizer handles case and accents automatically:

**Query:** "Cafe"

**Matches:**
- "cafe"
- "Cafe"
- "CAFE"
- "cafe"

No need for `COLLATE NOCASE` or accent-stripping logic.

## Performance: still fast

With 1,000 bookmarks:
- LIKE search: ~45ms
- BM25 search: ~50ms

The 5ms difference is negligible. You get much better relevance ranking with essentially no performance cost.

Storage overhead is about a 15% database size increase for the FTS5 index, which is a reasonable trade for the relevance improvement.

## Integration with hybrid search

BM25 replaces the keyword component in our hybrid search:

**Before:**
```
Hybrid = LIKE search + Semantic search
```

**After:**
```
Hybrid = BM25 search + Semantic search
```

Now hybrid search combines proper lexical ranking (BM25) with AI-powered semantic understanding (vector embeddings). It's a real upgrade from basic substring matching.

## Advanced features unlocked

FTS5 enables advanced query syntax:

### Phrase search
```sql
WHERE pages_fts MATCH '"react hooks"'
-- Matches exact phrase, not just both words
```

### Boolean operators
```sql
WHERE pages_fts MATCH 'react AND hooks'
-- Both terms must be present

WHERE pages_fts MATCH 'react OR vue'
-- Either term can be present

WHERE pages_fts MATCH 'react NOT class'
-- Has "react" but not "class"
```

### Prefix search
```sql
WHERE pages_fts MATCH 'reac*'
-- Matches "react", "reactive", "reacting"
```

These query features give power users precise control over their searches.

## Implementation checklist

To upgrade your keyword search to BM25:

1. Verify FTS5 availability in your SQLite build
2. Create a virtual FTS5 table with the right tokenizer
3. Set up triggers to keep FTS5 in sync
4. Replace LIKE queries with `MATCH` and `bm25()`
5. Apply field weights (title > summary > content)
6. Test with real queries to tune weights

## When to use BM25

BM25 makes sense when users search with keywords and expect relevant results, when you have text content of varying lengths, when title/summary should rank higher than body content, when you want stemming and unicode normalization, or when result relevance matters more than exact matching.

Stick with LIKE when you need exact substring matching (rare), when your database doesn't support FTS5, when storage overhead is a serious concern, or when queries are mostly ID or URL lookups.

## Limitations and edge cases

### Typos
BM25 doesn't handle typos well. "recat" won't match "react". You could implement fuzzy matching separately, use a trigram tokenizer (though that increases index size significantly), or provide "Did you mean?" suggestions.

### Conceptual queries
BM25 is still lexical search. "How to build modern web apps" relies on those exact words appearing in the document. This is where semantic search (vector embeddings) excels, and why hybrid search gives the best results.

### Complex boolean queries
Very complex `AND/OR/NOT` combinations can be slow:
- `(react OR vue) AND (hooks OR composition) NOT (class OR options)`

Limit query complexity or optimize FTS5 configuration if this becomes an issue.

## Future enhancements

### Query type detection
I'm considering automatically choosing between BM25 and semantic based on query characteristics: short queries (under 3 words) would use BM25, quoted phrases would use BM25, and longer descriptive queries would go to semantic search.

### Relevance feedback
Tracking which results users click could let us adjust field weights based on behavior, personalize ranking over time, and learn what relevance means for each user.

### Multi-language support
FTS5 supports different tokenizers: `porter` for English, `unicode61 remove_diacritics 2` for accents, and custom tokenizers for CJK languages.

## Wrapping up

Upgrading from `LIKE` to BM25 turned keyword search from basic substring matching into proper relevance ranking.

What we gained: results ranked by relevance instead of save date, stemming and unicode normalization, advanced query syntax like phrases and boolean operators, field weighting so title matches rank higher, and all of it for about 5ms slower and 15% more storage.

If you're building search functionality with SQLite, use FTS5 + BM25. It's built-in, fast, and makes a real difference in user experience.

Frank Bookmark now combines BM25 lexical search with vector semantic search in hybrid mode. Precise when you need exact terms, and intelligent when you need conceptual matching.

## Implementation details

Libraries used:
- SQL.js (with FTS5 extension enabled)
- sqlite-vec (for semantic search integration)

Tokenizer configuration: `porter unicode61` for English text, custom weights per field, and sync triggers for real-time updates.

Performance: sub-50ms search with 1,000+ bookmarks and about 15% storage overhead.

All code is open source in the Frank Bookmark repository.
