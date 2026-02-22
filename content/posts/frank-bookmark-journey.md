+++
date = '2026-01-26T10:00:00-03:00'
draft = false
title = 'Building Frank Bookmark: A Browser-Based AI Journey'
+++

## From DuckDB dreams to what actually shipped

Over two weeks in January 2026, I built Frank Bookmark, a Chrome extension that uses AI for semantic bookmark search and runs entirely in the browser. This is the story of how I got there, what worked, what didn't, and what I learned along the way.

## The problem

Traditional bookmarks are limited. You save hundreds or thousands of links, but finding them later is frustrating. You need to remember exact keywords, URLs, or folder locations. What if your browser could understand what your bookmarks are about, not just what they're called?

I wanted to answer one question: **Can we build a privacy-first, AI-powered bookmark system that runs entirely in the browser?**

## The journey: 8 experiments

### Experiment 1: DuckDB + Transformers.js

I started with DuckDB-Wasm, a SQL database that runs in browsers. The hypothesis: combine it with Transformers.js for vector embeddings, and we'd have semantic search right in the browser.

The web app worked. DuckDB handled data, Transformers.js generated embeddings, keyword search was fast. But vector search wasn't working yet. Browser-based AI is feasible, though there are quirks to figure out.

### Experiment 2: Extension reality check

I tried turning the web app into a Chrome extension. That's when I hit the first major roadblock.

DuckDB-Wasm doesn't work reliably in Chrome Service Workers. The WASM files wouldn't load correctly, and the extension context had compatibility issues. Not all browser technologies behave the same in extensions vs. web apps. Service Workers have real limitations.

### Experiment 3: The SQL.js pivot

I switched to SQL.js, a more mature SQLite implementation for browsers.

SQL.js worked perfectly in extensions, but it lacked native vector similarity functions. I had to calculate cosine similarity manually in JavaScript, which was slow. The lesson here: compatibility matters more than features if the features don't work.

### Experiment 4: sqlite-vec discovery

I found sqlite-vec, an SQLite extension specifically for vector similarity search.

Native vector functions at last. The `vec_distance_cosine()` SQL function was dramatically faster than manual JavaScript calculations. When possible, push computation into the database layer. SQL is fast.

### Experiment 5: Three search modes

With fast vector search working, I implemented three search modes:

1. Keyword - Fast SQL LIKE queries (< 100ms)
2. Semantic - AI-powered conceptual search (< 200ms after model load)
3. Hybrid - Combining both for best results (< 300ms)

Hybrid search provided the best user experience. It finds exact matches and related content. Don't force users to choose; combine approaches.

### Experiment 6: Making it persistent

In-memory databases are fast but lose everything on restart. I implemented persistence using OPFS (Origin Private File System).

The database now survives extension restarts. Save operations complete in under 500ms including content extraction and embedding generation. Modern browsers provide solid storage APIs, and OPFS works well for this.

### Experiment 7: Performance at scale

I tested with 1,000+ bookmarks to make sure the system would hold up.

sqlite-vec's native functions scaled well. Manual cosine similarity would have been unusable at this scale. The 10x speed improvement from native implementations made the difference.

### Experiment 8: CI/CD pipeline

I set up GitHub Actions for automated builds and releases.

Every tagged commit creates a release. Users download a ZIP, extract it, and load the extension. No compilation needed. Automation reduces friction and helps me iterate faster.

## The technology stack

The stack evolved quite a bit:

| Component | Initial | Final | Why Changed |
|-----------|---------|-------|-------------|
| Storage | DuckDB-Wasm | sqlite-vec | Service Worker compatibility |
| Build | None | Vite | Extensions require bundling |
| Search | Manual cosine | Native SQL | 10x performance improvement |
| Persistence | In-memory | OPFS + IndexedDB | Data survival required |
| Model Loading | On demand | Background preload | Better UX |

## What I learned

Modern browsers can run complete ML workflows. Content extraction with Mozilla Readability, text embedding with all-MiniLM-L6-v2 (384 dimensions), vector similarity search with sqlite-vec, all without sending data to servers. That surprised me.

Not all browser tech works everywhere, though. DuckDB-Wasm is great for web apps but broken in Service Workers. SQL.js works everywhere but lacks vector functions. sqlite-vec turned out to be the sweet spot.

Architecture matters too:

```
Extension Structure:
├── Popup (UI only)
│   └── Quick save/search interface
├── Background Service Worker
│   ├── AI Model (persistent)
│   ├── Database (sqlite-vec + OPFS)
│   └── Processing (extract, embed, search)
└── No localhost dependency
```

The separation is key: popup for UI, background for heavy lifting.

Because nothing leaves the browser, privacy is built in. Zero network requests after initial load. All data stays on device. No analytics, tracking, or telemetry. It works offline too.

Hybrid search turned out to be the right default. Users don't know whether they remember exact words or concepts. "React hooks" finds articles with those exact words, plus articles about component lifecycle and state management.

## What shipped after two weeks

- Self-contained Chrome extension
- Three search modes (keyword, semantic, hybrid)
- AI-powered content extraction
- 384-dimension vector embeddings
- Persistent storage
- Automated CI/CD
- Comprehensive documentation

## What's next

Optional enhancements I'm considering:
- Cross-device sync (optional cloud backup)
- Tags and folders
- Bulk operations
- Query suggestions
- Cross-browser support (Firefox, Edge, Safari)

## Try it

Frank Bookmark is open source. The code, documentation, and all the research notes are available. You can see exactly how I built it, what worked, what didn't, and why.

## Research documentation

All eight experiments are documented in detail:
- Notebooks: Daily experiment logs with questions, hypotheses, and results
- Insight Memos: Durable knowledge and recommendations
- SUMMARY.md: Complete project timeline and learnings

The full research process is transparent, including all the dead ends and pivots. Real R&D isn't linear, and I documented it that way.
