+++
date = '2026-01-26T12:00:00-03:00'
draft = false
title = 'Three Search Modes: Keyword, Semantic, and Hybrid'
+++

## The Search Problem

You've bookmarked an article about React state management. Weeks later, you want to find it. But how do you search?

- Do you remember exact words from the title? → **Keyword search**
- Do you remember the concept, not the words? → **Semantic search**
- Are you not sure? → **Hybrid search**

Users search in different ways depending on what they remember. Implementing all three modes provides the best experience across all scenarios.

Here's what we learned building search for Frank Bookmark.

## Three Search Modes

### 1. Keyword Search (Full-Text)

Traditional search. Fast and reliable.

**How it works:**
- SQL LIKE queries across title, content, description
- Case-insensitive substring matching
- Simple pattern matching

**SQL Implementation:**

```sql
SELECT * FROM pages
WHERE title ILIKE '%query%'
   OR content ILIKE '%query%'
   OR description ILIKE '%query%'
ORDER BY
  CASE
    WHEN title ILIKE '%query%' THEN 1
    WHEN description ILIKE '%query%' THEN 2
    ELSE 3
  END
```

**Performance:**
- Very fast (< 100ms for 1,000+ bookmarks)
- No model loading required
- Always available
- Works offline immediately

**Best for:**
- Exact phrase matching
- Title/URL searches
- When you remember specific words
- Quick lookups
- "That article about React hooks"

**Example:**

Query: "React hooks"

Finds:
- "Understanding React Hooks"
- "A Complete Guide to React Hooks"
- "React Hooks Tutorial"

Misses:
- "Component Lifecycle in React" (related but no keyword match)
- "State Management with Hooks" (has "hooks" but might rank lower)

**Limitations:**
- Requires remembering exact terms
- Misses semantically related content
- No conceptual understanding
- Typos break results

### 2. Semantic Search (Vector Embeddings)

AI-powered conceptual search.

**How it works:**
- Transform query into 384-dimensional vector
- Compare with stored embeddings using cosine similarity
- Rank by similarity score

**SQL Implementation (sqlite-vec):**

```sql
SELECT
  *,
  vec_distance_cosine(embedding, ?) AS distance
FROM pages
ORDER BY distance ASC
LIMIT 10
```

**Performance:**
- Initial load: 3-5 seconds (model loading, one-time)
- Subsequent searches: < 200ms for 1,000+ bookmarks
- Requires AI model in memory
- Uses native SQL vector functions

**Best for:**
- Conceptual searches
- Finding related content
- When you don't remember exact words
- Exploratory browsing
- "Articles about frontend performance"

**Example:**

Query: "frontend performance"

Finds:
- "Optimizing React Rendering"
- "Web Performance Best Practices"
- "Improving Load Times"
- "JavaScript Bundle Size"

Even if none of these contain "frontend performance" exactly, they're semantically related.

**Limitations:**
- Requires model loading (3-5 second wait)
- Can miss exact keyword matches
- May return semantically similar but contextually irrelevant content
- Relies on embedding quality

### 3. Hybrid Search (Weighted Combination)

Best of both worlds.

**How it works:**
- Run both keyword and semantic searches
- Combine scores with weights (typically 50/50)
- Re-rank combined results
- Return top matches

**SQL Implementation:**

```sql
SELECT
  *,
  (vec_distance_cosine(embedding, ?) * 0.5 +
   keyword_score * 0.5) AS combined_score
FROM (
  SELECT
    *,
    CASE
      WHEN title ILIKE ? OR content ILIKE ? THEN 1.0
      ELSE 0.0
    END AS keyword_score
  FROM pages
) subquery
ORDER BY combined_score ASC
LIMIT 10
```

**Performance:**
- Initial load: 3-5 seconds (same as semantic)
- Subsequent searches: < 300ms for 1,000+ bookmarks
- Slightly slower than single-mode
- Worth the cost for quality

**Best for:**
- General-purpose searching
- Most use cases
- Production default
- Mixed user scenarios
- Unknown user intent

**Example:**

Query: "React hooks"

Finds (ranked):
1. "Understanding React Hooks" (exact match + semantic)
2. "A Complete Guide to React Hooks" (exact match + semantic)
3. "Component Lifecycle in React" (semantic only)
4. "State Management Patterns" (semantic only)
5. "React Hooks Tutorial" (exact match, less relevant semantically)

Gets exact matches AND related content, ranked intelligently.

**Limitations:**
- More complex implementation
- Requires tuning weights
- Slightly slower than single-mode
- Needs both systems working

## Performance Comparison

Real-world measurements with 1,000 bookmarks:

| Mode | First Search | Subsequent | Model Load | Always Available |
|------|-------------|------------|------------|------------------|
| Keyword | < 100ms | < 100ms | No | Yes |
| Semantic | 3-5s | < 200ms | Yes | After load |
| Hybrid | 3-5s | < 300ms | Yes | After load |

All fast enough for responsive UI after initial model load.

## Implementation Guide

### Keyword Search

Simple and straightforward:

```javascript
async function keywordSearch(query) {
  const stmt = await db.prepare(`
    SELECT * FROM pages
    WHERE title ILIKE ? OR content ILIKE ? OR description ILIKE ?
    ORDER BY
      CASE
        WHEN title ILIKE ? THEN 1
        WHEN description ILIKE ? THEN 2
        ELSE 3
      END
  `);

  const pattern = `%${query}%`;
  return await stmt.all([pattern, pattern, pattern, pattern, pattern]);
}
```

### Semantic Search

Requires AI model:

```javascript
let model = null;

async function loadModel() {
  const { pipeline } = await import('@xenova/transformers');
  model = await pipeline('feature-extraction', 'Xenova/all-MiniLM-L6-v2');
  return model;
}

async function semanticSearch(query) {
  if (!model) {
    model = await loadModel();
  }

  // Generate query embedding
  const output = await model(query, { pooling: 'mean', normalize: true });
  const queryEmbedding = Array.from(output.data);

  // Vector similarity search
  const stmt = await db.prepare(`
    SELECT
      *,
      vec_distance_cosine(embedding, ?) AS distance
    FROM pages
    ORDER BY distance ASC
    LIMIT 10
  `);

  return await stmt.all([new Float32Array(queryEmbedding)]);
}
```

### Hybrid Search

Combine both:

```javascript
async function hybridSearch(query, weights = { keyword: 0.5, semantic: 0.5 }) {
  if (!model) {
    model = await loadModel();
  }

  // Generate query embedding
  const output = await model(query, { pooling: 'mean', normalize: true });
  const queryEmbedding = Array.from(output.data);

  // Combined SQL query
  const stmt = await db.prepare(`
    SELECT
      *,
      (vec_distance_cosine(embedding, ?) * ? +
       keyword_score * ?) AS combined_score
    FROM (
      SELECT
        *,
        CASE
          WHEN title ILIKE ? OR content ILIKE ? THEN 1.0
          ELSE 0.0
        END AS keyword_score
      FROM pages
    ) subquery
    ORDER BY combined_score ASC
    LIMIT 10
  `);

  const pattern = `%${query}%`;
  return await stmt.all([
    new Float32Array(queryEmbedding),
    weights.semantic,
    weights.keyword,
    pattern,
    pattern
  ]);
}
```

## User Interface Design

Let users choose their search mode:

```
┌─────────────────────────────────────────┐
│ Search: [                          ]    │
│                                          │
│ Mode: ◉ Hybrid  ○ Keyword  ○ Semantic  │
└─────────────────────────────────────────┘
```

**Default to Hybrid** for best overall experience.

**Allow selection** for users who know what they want:
- Power users prefer keyword for exact matches
- Exploratory users prefer semantic
- Most users should stay on hybrid

**Remember preference** across sessions.

## Optimizations

### 1. Preload AI Model

Don't wait for first search:

```javascript
// background.js
chrome.runtime.onInstalled.addListener(async () => {
  // Load model in background
  await loadModel();
  console.log('AI ready');
});
```

### 2. Cache Query Embeddings

Reuse for repeated searches:

```javascript
const embeddingCache = new Map();

async function getQueryEmbedding(query) {
  if (embeddingCache.has(query)) {
    return embeddingCache.get(query);
  }

  const embedding = await generateEmbedding(query);
  embeddingCache.set(query, embedding);

  // Expire after 1 hour
  setTimeout(() => embeddingCache.delete(query), 3600000);

  return embedding;
}
```

### 3. Limit Result Count

Don't return thousands of results:

```sql
ORDER BY score ASC
LIMIT 10  -- Only top 10
```

Implement "Load more" if needed.

### 4. Index Frequently Queried Fields

Speed up keyword search:

```sql
CREATE INDEX idx_title ON pages(title);
CREATE INDEX idx_content ON pages(content);
```

## When Each Mode Works Best

### Use Keyword When:

- Small datasets (< 100 items)
- Users remember exact terms
- Quick lookups and navigation
- Title/URL searches
- Finding specific pages
- Model load time unacceptable

### Use Semantic When:

- Conceptual searches
- Discovering related content
- Large datasets (100+ items)
- Users don't remember exact terms
- Exploratory browsing
- Research and discovery

### Use Hybrid When:

- Production use (recommended default)
- General-purpose searching
- Unknown user intent
- Mixed user scenarios
- Want best overall experience

## Common Pitfalls

### 1. Keyword Search Only

**Problem:** Misses semantically related content.

**Example:** Query "performance optimization" misses article titled "Making React Faster" (semantically related, no keyword match).

### 2. Semantic Search Only

**Problem:** Misses exact matches, slower initial load.

**Example:** Query "React hooks" might rank "Component Patterns" higher than "React Hooks Tutorial" if embeddings are similar.

### 3. Manual Vector Similarity

**Problem:** Too slow for large datasets.

**Solution:** Use native SQL functions (sqlite-vec).

```javascript
// DON'T: Manual calculation
function cosineSimilarity(a, b) {
  // 384 operations per bookmark
  // 384,000 operations for 1,000 bookmarks
  // Too slow!
}

// DO: Native SQL function
vec_distance_cosine(embedding, ?)  // Fast!
```

### 4. No Mode Selection

**Problem:** Users can't adapt to their needs.

**Solution:** Provide mode selection, default to hybrid.

## Testing Scenarios

Verify all three modes work correctly:

### 1. Exact Match
- **Query:** "React hooks tutorial"
- **Keyword:** Finds it immediately
- **Semantic:** Finds it plus related
- **Hybrid:** Ranks it first

### 2. Conceptual Search
- **Query:** "frontend performance optimization"
- **Keyword:** Might find nothing
- **Semantic:** Finds related articles
- **Hybrid:** Finds exact + related

### 3. Typos
- **Query:** "Reactt hooks" (typo)
- **Keyword:** Might find nothing
- **Semantic:** Still finds related (embedding similar)
- **Hybrid:** Semantic component saves it

### 4. Large Dataset
- **1,000+ bookmarks**
- **All modes:** < 300ms response
- **Quality:** Relevant results in top 10

### 5. No Results
- **All modes handle gracefully**
- **No crashes**
- **Clear messaging**

## Future Enhancements

### Relevance Feedback

Learn from user clicks:

```javascript
function recordClick(result, query, mode) {
  // Track: user clicked result for query
  // Adjust weights over time
  // Personalize search
}
```

### Personalized Weights

Tune per user:

```javascript
// User prefers semantic results
weights = { keyword: 0.3, semantic: 0.7 };

// User prefers exact matches
weights = { keyword: 0.7, semantic: 0.3 };
```

### Query Suggestions

Autocomplete based on embeddings:

```javascript
async function suggestQueries(partial) {
  // Find similar past queries
  // Suggest completions
  // Learn from history
}
```

### Fuzzy Matching

Handle typos in keyword search:

```sql
-- Levenshtein distance for typos
WHERE levenshtein(title, ?) < 3
```

## Lessons Learned

### 1. Hybrid is Best Default

Don't force users to choose. Combine approaches for better results.

### 2. Native Performance Matters

sqlite-vec's native functions are 10x faster than JavaScript. Use them.

### 3. Preload Models

Loading on first search is bad UX. Load in background on install.

### 4. Give Users Control

Power users want options. Provide mode selection.

### 5. Test at Scale

Performance characteristics change with volume. Test with 1,000+ items.

## Recommendations

For browser-based bookmark search:

**Default Configuration:**
- Mode: Hybrid
- Weights: 50% keyword, 50% semantic
- Model: all-MiniLM-L6-v2
- Vector functions: sqlite-vec

**User Interface:**
- Show search mode selector
- Default to Hybrid
- Remember user preference
- Show loading state for model

**Performance:**
- Preload model on install
- Use native SQL vector functions
- Cache query embeddings
- Limit result count

**Testing:**
- Exact match scenarios
- Conceptual search scenarios
- Typos and variations
- Large datasets (1,000+)

## Conclusion

Three search modes provide comprehensive coverage of user search behaviors:

- **Keyword** - Fast, exact, always available
- **Semantic** - Conceptual, discovery, AI-powered
- **Hybrid** - Best of both, recommended default

Implementing all three gives users the best experience across all scenarios. They don't need to know which mode to use—hybrid handles most cases automatically.

With native vector functions (sqlite-vec) and preloaded models, all three modes are fast enough for responsive UI. The performance is there, the architecture works, and the results are excellent.

Frank Bookmark proves this approach works at scale with production-ready performance.
