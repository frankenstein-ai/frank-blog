+++
date = '2026-01-26T12:00:00-03:00'
draft = false
title = 'Three Search Modes: Keyword, Semantic, and Hybrid'
+++

## The search problem

You've bookmarked an article about React state management. Weeks later, you want to find it. But how do you search?

- Do you remember exact words from the title? Keyword search.
- Do you remember the concept, not the words? Semantic search.
- Are you not sure? Hybrid search.

People search in different ways depending on what they remember. Building all three modes covers those different habits well. Here's what I learned building search for Frank Bookmark.

## Three search modes

### 1. Keyword search (full-text)

Traditional search. Fast and reliable.

It runs SQL LIKE queries across the title, content, and description fields with case-insensitive substring matching.

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

In practice it's very fast, under 100ms for 1,000+ bookmarks. No model loading required, and it works offline immediately.

This is best for exact phrase matching, title or URL searches, or when you remember specific words. Think: "that article about React hooks."

For example, the query "React hooks" finds titles like "Understanding React Hooks," "A Complete Guide to React Hooks," and "React Hooks Tutorial." But it misses "Component Lifecycle in React" (related but no keyword match) and may underrank "State Management with Hooks."

The downsides: you have to remember the exact terms, it misses semantically related content, and typos break results.

### 2. Semantic search (vector embeddings)

This is the AI-powered conceptual approach. It transforms the query into a 384-dimensional vector, compares it with stored embeddings using cosine similarity, and ranks by similarity score.

```sql
SELECT
  *,
  vec_distance_cosine(embedding, ?) AS distance
FROM pages
ORDER BY distance ASC
LIMIT 10
```

The initial model load takes 3-5 seconds, but subsequent searches come in under 200ms for 1,000+ bookmarks. It uses native SQL vector functions via sqlite-vec.

Semantic search works best for conceptual searches, finding related content, or exploratory browsing when you don't remember exact words.

For example, the query "frontend performance" finds "Optimizing React Rendering," "Web Performance Best Practices," "Improving Load Times," and "JavaScript Bundle Size," even though none of these contain "frontend performance" exactly.

The tradeoffs: there's that initial model loading wait, it can miss exact keyword matches, and it may return semantically similar but contextually irrelevant content. Quality depends on the embeddings.

### 3. Hybrid search (weighted combination)

This combines both approaches. It runs keyword and semantic searches in parallel, combines scores with weights (typically 50/50), re-ranks the combined results, and returns the top matches.

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

Performance is similar to semantic search: 3-5 seconds for the initial model load, then under 300ms for subsequent searches. Slightly slower than single-mode, but worth it for the result quality.

For the query "React hooks," hybrid search returns:

1. "Understanding React Hooks" (exact match + semantic)
2. "A Complete Guide to React Hooks" (exact match + semantic)
3. "Component Lifecycle in React" (semantic only)
4. "State Management Patterns" (semantic only)
5. "React Hooks Tutorial" (exact match, less relevant semantically)

You get exact matches and related content, ranked together.

The downsides are a more complex implementation, the need to tune weights, and a dependency on both systems working.

## Performance comparison

Real-world measurements with 1,000 bookmarks:

| Mode | First Search | Subsequent | Model Load | Always Available |
|------|-------------|------------|------------|------------------|
| Keyword | < 100ms | < 100ms | No | Yes |
| Semantic | 3-5s | < 200ms | Yes | After load |
| Hybrid | 3-5s | < 300ms | Yes | After load |

All fast enough for a responsive UI after the initial model load.

## Implementation guide

### Keyword search

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

### Semantic search

This requires an AI model:

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

### Hybrid search

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

## User interface design

I let users choose their search mode:

```
┌─────────────────────────────────────────┐
│ Search: [                          ]    │
│                                          │
│ Mode: ◉ Hybrid  ○ Keyword  ○ Semantic  │
└─────────────────────────────────────────┘
```

The default is hybrid for the best overall experience. But power users who want exact matches can switch to keyword, and exploratory users can switch to semantic. I also persist the preference across sessions.

## Optimizations

### 1. Preload the AI model

Don't wait for the first search:

```javascript
// background.js
chrome.runtime.onInstalled.addListener(async () => {
  // Load model in background
  await loadModel();
  console.log('AI ready');
});
```

### 2. Cache query embeddings

Reuse embeddings for repeated searches:

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

### 3. Limit result count

Don't return thousands of results:

```sql
ORDER BY score ASC
LIMIT 10  -- Only top 10
```

Implement "Load more" if needed.

### 4. Index frequently queried fields

Speed up keyword search:

```sql
CREATE INDEX idx_title ON pages(title);
CREATE INDEX idx_content ON pages(content);
```

## When each mode works best

Keyword search fits small datasets (under 100 items), cases where users remember exact terms, quick lookups, title or URL searches, and situations where model load time isn't acceptable.

Semantic search is better for conceptual searches, discovering related content, larger datasets (100+ items), when users don't remember exact terms, and general exploratory browsing.

Hybrid search is the recommended default for general-purpose searching, unknown user intent, and mixed user scenarios.

## Common pitfalls

### Keyword search only

The problem is that it misses semantically related content. For example, querying "performance optimization" misses an article titled "Making React Faster" since there's no keyword overlap even though the topics are closely related.

### Semantic search only

The problem is that it can miss exact matches and has a slower initial load. Querying "React hooks" might rank "Component Patterns" higher than "React Hooks Tutorial" if the embeddings happen to be similar.

### Manual vector similarity

Computing cosine similarity in JavaScript is too slow for large datasets. Use native SQL functions (sqlite-vec) instead.

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

### No mode selection

Users can't adapt search to their needs. Provide mode selection and default to hybrid.

## Testing scenarios

I verify all three modes work correctly across a few scenarios.

For an exact match query like "React hooks tutorial," keyword search finds it immediately, semantic search finds it plus related results, and hybrid search ranks it first.

For a conceptual search like "frontend performance optimization," keyword search might find nothing, semantic search finds related articles, and hybrid search finds exact matches plus related content.

For a typo like "Reactt hooks," keyword search might find nothing, but semantic search still finds related results because the embedding is similar enough. Hybrid search benefits from that semantic component.

For large datasets with 1,000+ bookmarks, all modes stay under 300ms response time with relevant results in the top 10. And all modes handle the no-results case gracefully with clear messaging.

## Future enhancements

### Relevance feedback

Learn from user clicks:

```javascript
function recordClick(result, query, mode) {
  // Track: user clicked result for query
  // Adjust weights over time
  // Personalize search
}
```

### Personalized weights

Tune per user:

```javascript
// User prefers semantic results
weights = { keyword: 0.3, semantic: 0.7 };

// User prefers exact matches
weights = { keyword: 0.7, semantic: 0.3 };
```

### Query suggestions

Autocomplete based on embeddings:

```javascript
async function suggestQueries(partial) {
  // Find similar past queries
  // Suggest completions
  // Learn from history
}
```

### Fuzzy matching

Handle typos in keyword search:

```sql
-- Levenshtein distance for typos
WHERE levenshtein(title, ?) < 3
```

## What I learned

Hybrid turned out to be the best default. Rather than forcing users to choose, combining both approaches just gives better results.

I also found that native performance matters a lot here. sqlite-vec's native functions are roughly 10x faster than doing the same work in JavaScript.

Preloading the model is worth doing. Making users wait on the first search is poor UX; loading in the background on install solves that.

And giving users control is still important. Power users want the option to switch modes, so I kept the mode selector available.

Finally, testing at scale caught issues I wouldn't have seen otherwise. Performance characteristics change with volume, so I made sure to test with 1,000+ items.

## Recommendations

For browser-based bookmark search, I'd suggest this configuration:

- Mode: Hybrid
- Weights: 50% keyword, 50% semantic
- Model: all-MiniLM-L6-v2
- Vector functions: sqlite-vec

For the UI, show a search mode selector, default to hybrid, remember the user's preference, and show a loading state while the model initializes.

For performance, preload the model on install, use native SQL vector functions, cache query embeddings, and limit the result count.

For testing, cover exact match scenarios, conceptual search scenarios, typos and variations, and large datasets (1,000+).

## Wrapping up

The three search modes cover different user search behaviors. Keyword search is fast and exact. Semantic search handles conceptual discovery. Hybrid combines both and works as the recommended default.

Building all three gives users a good experience regardless of how they search. Most of the time hybrid handles things automatically, and users don't need to think about which mode to use.

With native vector functions (sqlite-vec) and a preloaded model, all three modes are fast enough for a responsive UI.
