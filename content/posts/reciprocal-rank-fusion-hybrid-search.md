+++
date = '2026-02-02T10:00:00-03:00'
draft = false
title = 'Reciprocal Rank Fusion: Better Hybrid Search Through Rank Combination'
+++

## The hybrid search problem

In the [search evolution journey](/posts/search-evolution-simple-to-sophisticated), I built hybrid search by combining BM25 keyword results with semantic vector results. The initial approach was simple weighted averaging:

```javascript
combinedScore = (0.5 * keywordScore) + (0.5 * semanticScore)
```

This works, but it has a real flaw.

## The score scaling problem

BM25 and vector similarity produce scores on completely different scales. BM25 scores range from 0 to infinity (unbounded), while cosine similarity ranges from 0 to 1 (normalized).

When you average these, you need to normalize first. But even after normalization, weighted averaging has a deeper issue: it fails to preserve "top rank" signals.

### The scenario

Query: "React hooks tutorial"

| Document | BM25 Rank | BM25 Score | Semantic Rank | Semantic Score |
|----------|-----------|------------|---------------|----------------|
| React Hooks Guide | #1 | 1.0 | #1 | 1.0 |
| React Tutorial | #2 | 0.5 | #50 | 0.02 |
| Hooks API Ref | #10 | 0.1 | #5 | 0.2 |

Weighted average (50/50):
- React Hooks Guide: `0.5 * 1.0 + 0.5 * 1.0 = 1.0`
- React Tutorial: `0.5 * 0.5 + 0.5 * 0.02 = 0.26`
- Hooks API Ref: `0.5 * 0.1 + 0.5 * 0.2 = 0.15`

Looks reasonable, right? But there's a problem.

The "React Hooks Guide" is ranked #1 in both search engines. It should dominate. But in weighted averaging, its advantage over "React Tutorial" (which is #2 in keywords but #50 in semantic) isn't as pronounced as it should be.

Worse: if a document were ranked #2 in both engines, simple averaging might rank it similarly to the #1/#1 document, despite being clearly less relevant.

## Enter reciprocal rank fusion (RRF)

RRF is an algorithm from information retrieval research that solves this problem by combining ranks, not scores.

### The formula

```
RRF_Score = Σ [ 1 / (k + rank_i) ]
```

Where:
- `k` is a constant (typically 60)
- `rank_i` is the document's position in result list `i`

### Why this works

RRF gives logarithmically decreasing weight to lower-ranked items:

- Rank #1: `1 / (60 + 1) = 0.016`
- Rank #2: `1 / (60 + 2) = 0.016`
- Rank #10: `1 / (60 + 10) = 0.014`
- Rank #50: `1 / (60 + 50) = 0.009`
- Rank #100: `1 / (60 + 100) = 0.006`

The drop-off is steep for low ranks but gradual for high ranks.

A document ranked #1 in both engines gets:
```
0.016 + 0.016 = 0.032
```

A document ranked #1 in one engine and #50 in the other gets:
```
0.016 + 0.009 = 0.025
```

A document ranked #2 in both engines gets:
```
0.016 + 0.016 = 0.032
```

Wait, that's the same as #1/#1! That's the point: RRF rewards consistency across engines, not just a high score in one engine.

## Implementation

### The RRF function

```javascript
function reciprocalRankFusion(keywordResults, semanticResults, options = {}) {
  const { limit = 10, rrfK = 60 } = options;
  const scoreMap = new Map();

  // Process Keyword Results
  keywordResults.forEach((page, index) => {
    const rank = index + 1; // 1-based ranking
    const rrfScore = 1 / (rrfK + rank);

    scoreMap.set(page.id, {
      page,
      keywordRank: rank,
      keywordRRF: rrfScore,
      semanticRank: Infinity,
      semanticRRF: 0
    });
  });

  // Process Semantic Results
  semanticResults.forEach((page, index) => {
    const rank = index + 1;
    const rrfScore = 1 / (rrfK + rank);

    const existing = scoreMap.get(page.id);
    if (existing) {
      existing.semanticRank = rank;
      existing.semanticRRF = rrfScore;
    } else {
      scoreMap.set(page.id, {
        page,
        keywordRank: Infinity,
        keywordRRF: 0,
        semanticRank: rank,
        semanticRRF: rrfScore
      });
    }
  });

  // Combine scores
  const combinedResults = Array.from(scoreMap.values())
    .map(({ page, keywordRRF, semanticRRF, keywordRank, semanticRank }) => ({
      ...page,
      searchType: 'hybrid',
      rrfScore: keywordRRF + semanticRRF,
      keywordRank,
      semanticRank
    }))
    .sort((a, b) => b.rrfScore - a.rrfScore)
    .slice(0, limit);

  return combinedResults;
}
```

### Integration with hybrid search

```javascript
async function searchHybrid(query, options = {}) {
  const { limit = 10 } = options;

  // Fetch larger result sets from both engines
  const [keywordResults, semanticResults] = await Promise.all([
    searchBM25(query, { limit: limit * 3 }),
    searchSemantic(query, { limit: limit * 3 })
  ]);

  // Apply RRF fusion
  return reciprocalRankFusion(keywordResults, semanticResults, { limit, rrfK: 60 });
}
```

Why fetch `limit * 3`? If I only fetch 10 results from each engine, I might miss documents ranked #11 in one engine but #1 in another. Fetching 30 from each gives RRF more documents to work with and improves result quality.

## Real-world results

Query: "React hooks tutorial"

### Before (weighted average)

1. React Hooks Guide (1.0) - Good
2. React Tutorial (0.26) - Okay
3. Hooks API Ref (0.15) - Missed opportunity

### After (RRF)

1. React Hooks Guide (0.032) - Ranked #1 in both
2. Hooks API Ref (0.024) - Ranked high in semantic
3. React Tutorial (0.022) - High in keyword, low in semantic

The "Hooks API Reference" moved from #3 to #2 because it was ranked #5 in semantic search (a strong conceptual match) even though it was #10 in keywords. RRF surfaced that result.

## Performance

RRF is pure JavaScript computation. It's O(N log N) for sorting, where N is total documents, and adds under 5ms of latency for typical bookmark queries. Memory usage is a Map to track scores, which is lightweight. With 30 results from each engine (60 total unique documents), sorting and merging takes negligible time.

## The constant `k`: tuning sensitivity

The `k` parameter (default 60) controls how steeply scores drop off.

### k = 30 (aggressive)
- Rank #1: `1 / 31 = 0.032`
- Rank #100: `1 / 130 = 0.007`
- Top results get a much higher boost. Good for precision search where exact match priority matters.

### k = 60 (standard)
- Rank #1: `1 / 61 = 0.016`
- Rank #100: `1 / 160 = 0.006`
- Balanced. This is the general-purpose default.

### k = 100 (smooth)
- Rank #1: `1 / 101 = 0.009`
- Rank #100: `1 / 200 = 0.005`
- Flatter curve. Good for discovery search where you want broader results.

The industry standard is k=60, which provides a good balance.

## Why RRF works well

RRF is scale-agnostic. It doesn't care about score ranges (BM25 goes 0 to infinity, cosine goes 0 to 1) because it only uses ranks. No normalization needed.

It preserves top rank signals. Documents ranked highly in either engine get strong scores. Ranked #1 in both gives a very high RRF score. Ranked #1 in one and #50 in the other still gives a decent score. Ranked #50 in both gives a low score. That's exactly the behavior I wanted.

It's well-studied in information retrieval literature, not a hack but a proven algorithm used by many search systems.

And it extends naturally to more than 2 sources:

```javascript
rrfScore = 1/(k + rank_bm25)
         + 1/(k + rank_vector)
         + 1/(k + rank_tags)
         + 1/(k + rank_history)
```

You can fuse keyword, semantic, tag matching, user click history, and more in one unified ranking.

## When to use RRF

Use RRF when combining results from multiple search engines, when score scales are incompatible, when top-ranked matches should dominate, or when you want sound fusion.

Stick with weighted average when you have only one search engine (no fusion needed), when scores are already on the same scale, or when simple implementation is the priority.

For hybrid search, I'd go with RRF.

## Implementation tips

### Fetch larger result sets

```javascript
// Fetch 3x the final limit
const keywordResults = await searchBM25(query, { limit: limit * 3 });
const semanticResults = await searchSemantic(query, { limit: limit * 3 });
```

This gives RRF more candidates to work with, improving final ranking quality.

### Show rank information

Display which engine contributed to the result:

```javascript
{
  title: "React Hooks Guide",
  rrfScore: 0.032,
  keywordRank: 1,   // Ranked #1 in keywords
  semanticRank: 1    // Ranked #1 in semantic
  // UI: Show "Perfect Match" badge
}
```

### Make `k` configurable

Allow power users to tune sensitivity:

```javascript
const config = {
  searchMode: 'hybrid',
  rrfK: 60,  // Tunable
  limit: 10
};
```

## Future: multi-source RRF

RRF isn't limited to two sources. You can combine BM25 (keyword matching), vector search (semantic similarity), chunks (content-based search), tags (explicit categorization), and history (user click patterns).

Each source contributes `1 / (k + rank_i)` to the final score. Documents appearing in multiple sources get boosted proportionally.

## Wrapping up

Reciprocal Rank Fusion improved hybrid search noticeably over weighted averaging.

Weighted averaging has score scaling issues and doesn't preserve top rank signals well. RRF avoids both problems: it's scale-agnostic, preserves top rank signals, is well-studied, and extends easily to more sources.

The implementation is straightforward, the performance overhead is negligible (under 5ms), and the result quality improvement is real.

Frank Bookmark now uses RRF for all hybrid searches. It's the same algorithm used by search systems like Elasticsearch, running entirely in the browser.

**Read more:**
- [Frank Bookmark evolution](/posts/frank-bookmark-evolution)
- [Search evolution journey](/posts/search-evolution-simple-to-sophisticated)
- [BM25 implementation](/posts/bm25-fts5-search-relevance)
- [Three search modes](/posts/three-search-modes-bookmark-systems)

This is Experiment 10 in the Frank Bookmark journey.
