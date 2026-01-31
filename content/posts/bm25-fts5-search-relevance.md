+++
date = '2026-01-27T10:00:00-03:00'
draft = false
title = 'Beyond LIKE: Upgrading to BM25 for Better Search Relevance'
+++

## The Keyword Search Problem

In our [initial Frank Bookmark implementation](/posts/three-search-modes-bookmark-systems), keyword search used simple SQL `LIKE` operators:

```sql
SELECT * FROM pages
WHERE title ILIKE '%react hooks%'
   OR content ILIKE '%react hooks%'
```

This works, but it has a critical flaw: **all matches are treated equally**.

A page with "React hooks" in the title three times gets the same relevance score as a page where those words appear once in a 5,000-word article. The results? Ordered by save date, not by relevance to your query.

Users expect search results ranked by **how relevant they are**, not by when you bookmarked them.

## Enter BM25: The Standard for Text Relevance

BM25 (Best Matching 25) is the gold standard algorithm for ranking full-text search results. It's used by Elasticsearch, Apache Lucene, and pretty much every modern search engine.

**What BM25 considers:**

1. **Term Frequency (TF)** - How often does the search term appear in the document?
2. **Inverse Document Frequency (IDF)** - How rare is the term across your entire collection?
3. **Document Length** - Normalizes scores so short documents don't have an unfair advantage

**The result:** Articles where your search terms appear frequently and prominently rank higher than articles where they just happen to exist.

## Implementing BM25 with SQLite FTS5

SQLite's FTS5 (Full-Text Search) extension provides native BM25 ranking. No external search engines, no complex setup—just SQL.

### Step 1: Create a Virtual FTS5 Table

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

**Why `porter unicode61`?**

- **Porter stemming** - "running" matches "run", "runner", "ran"
- **Unicode61** - Case-insensitive, handles accents ("café" = "cafe")

### Step 2: Keep FTS5 in Sync with Triggers

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

### Step 3: Search with BM25 Ranking

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

**Key difference:**
- **Old:** `WHERE title ILIKE '%query%'` (exists or not)
- **New:** `WHERE pages_fts MATCH 'query' ORDER BY bm25(...)` (ranked by relevance)

## Field Weighting: Not All Text is Equal

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

**Weights applied:**
- Title: **10.0** (highest priority)
- Summary: **5.0** (medium priority)
- Content: **2.0** (lower priority)
- Tags: **1.0** (lowest priority)

This aligns with user expectations: "If it's in the title, it's probably what I'm looking for."

## Real-World Improvement

**Query:** "React hooks tutorial"

### Before (LIKE Search)

Results ordered by save date:

1. "React Hooks Guide 2023" ✓
2. "React Tutorial: Introduction to Hooks" ✓
3. "React Performance Tutorial" ✗ (only has "tutorial", low relevance)
4. "My Thoughts on React" ✗ (only has "React", very low relevance)

### After (BM25 Search)

Results ordered by relevance:

1. "React Hooks Guide 2023" ✓ (all terms in title, high frequency)
2. "React Tutorial: Introduction to Hooks" ✓ (all terms in title)
3. "My Thoughts on React Hooks" ✓ (all terms, high frequency in content)
4. "Advanced React Patterns" ✓ (related, has "React")

BM25 surfaces the most relevant results first, regardless of when they were saved.

## Stemming: Smart Matching Out of the Box

The Porter stemmer provides intelligent matching without extra code:

**Query:** "running node"

**Matches:**
- "running node"
- "run node"
- "running Node.js"
- "run Node.js"
- "ran Node"

This broadens search scope intelligently, finding variations without requiring synonyms or manual fuzzy matching.

## Unicode Normalization: International Support

The `unicode61` tokenizer handles case and accents automatically:

**Query:** "Café"

**Matches:**
- "café"
- "Cafe"
- "CAFÉ"
- "cafe"

No need for complex `COLLATE NOCASE` or accent-stripping logic.

## Performance: Still Fast

**With 1,000 bookmarks:**
- **LIKE search:** ~45ms
- **BM25 search:** ~50ms

The 5ms difference is negligible. You get dramatically better relevance ranking with essentially no performance cost.

**Storage overhead:**
- FTS5 index: ~15% database size increase
- Totally worth it for the relevance improvement

## Integration with Hybrid Search

BM25 replaces the keyword component in our hybrid search:

**Before:**
```
Hybrid = LIKE search + Semantic search
```

**After:**
```
Hybrid = BM25 search + Semantic search
```

Now hybrid search combines:
- **State-of-the-art lexical ranking** (BM25)
- **AI-powered semantic understanding** (vector embeddings)

This is production-grade search, not just basic substring matching.

## Advanced Features Unlocked

FTS5 enables advanced query syntax:

### Phrase Search
```sql
WHERE pages_fts MATCH '"react hooks"'
-- Matches exact phrase, not just both words
```

### Boolean Operators
```sql
WHERE pages_fts MATCH 'react AND hooks'
-- Both terms must be present

WHERE pages_fts MATCH 'react OR vue'
-- Either term can be present

WHERE pages_fts MATCH 'react NOT class'
-- Has "react" but not "class"
```

### Prefix Search
```sql
WHERE pages_fts MATCH 'reac*'
-- Matches "react", "reactive", "reacting"
```

These query features give power users precise control over their searches.

## Implementation Checklist

To upgrade your keyword search to BM25:

1. **Verify FTS5 availability** in your SQLite build
2. **Create virtual FTS5 table** with appropriate tokenizer
3. **Set up triggers** to keep FTS5 in sync
4. **Replace LIKE queries** with `MATCH` and `bm25()`
5. **Apply field weights** (title > summary > content)
6. **Test with real queries** to tune weights

## When to Use BM25

**Use BM25 when:**
- Users search with keywords and expect relevant results
- You have text content of varying lengths
- Title/summary should rank higher than body content
- You want stemming and unicode normalization
- Result relevance matters more than exact matching

**Stick with LIKE when:**
- You need exact substring matching (rare)
- Database doesn't support FTS5
- Storage overhead is critical concern
- Queries are mostly ID or URL lookups

## Limitations and Edge Cases

### Typos
BM25 doesn't handle typos well:
- "recat" won't match "react"

**Solutions:**
- Implement fuzzy matching separately
- Use trigram tokenizer (increases index size significantly)
- Provide "Did you mean?" suggestions

### Conceptual Queries
BM25 is still lexical search:
- "How to build modern web apps" relies on those exact words

**Solution:** This is where semantic search (vector embeddings) excels. Use hybrid search for best results.

### Complex Boolean Queries
Very complex `AND/OR/NOT` combinations can be slow:
- `(react OR vue) AND (hooks OR composition) NOT (class OR options)`

**Solution:** Limit query complexity or optimize FTS5 configuration.

## Future Enhancements

### Query Type Detection
Automatically choose between BM25 and semantic based on query:
- Short queries (< 3 words) → BM25
- Quoted phrases → BM25
- Longer descriptive queries → Semantic

### Relevance Feedback
Track which results users click:
- Adjust field weights based on behavior
- Personalize ranking over time
- "Learn" what relevance means for each user

### Multi-Language Support
FTS5 supports different tokenizers:
- `porter` for English
- `unicode61 remove_diacritics 2` for accents
- Custom tokenizers for CJK languages

## Conclusion

Upgrading from `LIKE` to BM25 transformed our keyword search from basic substring matching to production-grade relevance ranking.

**What we gained:**
- **Better results** - Ranked by relevance, not save date
- **Smart matching** - Stemming and unicode normalization
- **Advanced queries** - Phrases, boolean operators, prefix search
- **Field weighting** - Title matches rank higher
- **Negligible cost** - 5ms slower, 15% more storage

**The bottom line:** If you're building search functionality with SQLite, use FTS5 + BM25. It's built-in, fast, and dramatically improves user experience.

Frank Bookmark now combines BM25 lexical search with vector semantic search in hybrid mode. It's the best of both worlds: precise when you need it, intelligent when you want it.

## Implementation Details

**Libraries:**
- SQL.js (with FTS5 extension enabled)
- sqlite-vec (for semantic search integration)

**Tokenizer configuration:**
- `porter unicode61` for English text
- Custom weights per field
- Sync triggers for real-time updates

**Performance:**
- Sub-50ms search with 1,000+ bookmarks
- ~15% storage overhead
- Production-ready

All code is open source in the Frank Bookmark repository.
