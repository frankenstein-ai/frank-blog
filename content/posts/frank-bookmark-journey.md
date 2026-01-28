+++
date = '2026-01-26T10:00:00-03:00'
draft = false
title = 'Building Frank Bookmark: A Browser-Based AI Journey'
+++

## From DuckDB Dreams to Production Reality

Over two weeks in January 2026, we built Frank Bookmark - a Chrome extension that uses AI for semantic bookmark search, running entirely in your browser. This is the story of how we got there, what worked, what didn't, and what we learned along the way.

## The Vision

Traditional bookmarks are limited. You save hundreds or thousands of links, but finding them later is frustrating. You need to remember exact keywords, URLs, or folder locations. What if your browser could understand what your bookmarks are about, not just what they're called?

We set out to answer one question: **Can we build a privacy-first, AI-powered bookmark system that runs entirely in the browser?**

## The Journey: 8 Experiments

### Experiment 1: DuckDB + Transformers.js

We started with DuckDB-Wasm, a powerful SQL database that runs in browsers. Our hypothesis: combine it with Transformers.js for vector embeddings, and we'd have semantic search right in the browser.

**Result:** The web app worked! DuckDB handled data, Transformers.js generated embeddings, keyword search was fast... but vector search wasn't working yet.

**Learning:** Browser-based AI is feasible, but there are quirks to figure out.

### Experiment 2: Extension Reality Check

We tried turning the web app into a Chrome extension. That's when we hit our first major roadblock.

**Result:** DuckDB-Wasm doesn't work reliably in Chrome Service Workers. The WASM files wouldn't load correctly, and the extension context had compatibility issues.

**Learning:** Not all browser technologies work the same in extensions vs. web apps. Service Workers have limitations.

### Experiment 3: The SQL.js Pivot

We switched to SQL.js, a more mature SQLite implementation for browsers.

**Result:** SQL.js worked perfectly in extensions! But it lacked native vector similarity functions. We had to calculate cosine similarity manually in JavaScript, which was slow.

**Learning:** Compatibility matters more than features if the features don't work.

### Experiment 4: sqlite-vec Discovery

We found sqlite-vec, an SQLite extension specifically for vector similarity search.

**Result:** Native vector functions! The `vec_distance_cosine()` SQL function was dramatically faster than manual JavaScript calculations.

**Learning:** When possible, push computation into the database layer. SQL is fast.

### Experiment 5: Three Search Modes

With fast vector search working, we implemented three search modes:

1. **Keyword** - Fast SQL LIKE queries (< 100ms)
2. **Semantic** - AI-powered conceptual search (< 200ms after model load)
3. **Hybrid** - Combining both for best results (< 300ms)

**Result:** Hybrid search provided the best user experience. It finds exact matches AND related content.

**Learning:** Don't force users to choose. Combine approaches for better results.

### Experiment 6: Making it Persistent

In-memory databases are fast but lose everything on restart. We implemented persistence using OPFS (Origin Private File System).

**Result:** Database survives extension restarts. Save operations complete in < 500ms including content extraction and embedding generation.

**Learning:** Modern browsers provide robust storage APIs. OPFS is production-ready.

### Experiment 7: Performance at Scale

We tested with 1,000+ bookmarks to ensure the system would scale.

**Result:** sqlite-vec's native functions scaled well. Manual cosine similarity would have been unusable at this scale.

**Learning:** Native implementations matter for performance. The 10x speed improvement was critical.

### Experiment 8: CI/CD Pipeline

We set up GitHub Actions for automated builds and releases.

**Result:** Every tagged commit creates a release. Users download a ZIP, extract it, and load the extension. No compilation needed.

**Learning:** Automation reduces friction and enables faster iteration.

## The Technology Stack

We evolved our stack significantly:

| Component | Initial | Final | Why Changed |
|-----------|---------|-------|-------------|
| Storage | DuckDB-Wasm | sqlite-vec | Service Worker compatibility |
| Build | None | Vite | Extensions require bundling |
| Search | Manual cosine | Native SQL | 10x performance improvement |
| Persistence | In-memory | OPFS + IndexedDB | Data survival required |
| Model Loading | On demand | Background preload | Better UX |

## Key Learnings

### 1. Browser-Based AI is Production-Ready

Modern browsers can run complete ML workflows:
- Content extraction (Mozilla Readability)
- Text embedding (all-MiniLM-L6-v2, 384 dimensions)
- Vector similarity search (sqlite-vec)
- All without sending data to servers

### 2. Not All Browser Tech Works Everywhere

- DuckDB-Wasm: Great for web apps, broken in Service Workers
- SQL.js: Works everywhere, lacks vector functions
- sqlite-vec: Best of both worlds

### 3. Architecture Matters

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

### 4. Privacy by Design

No servers = no privacy concerns:
- Zero network requests after initial load
- All data stays on device
- No analytics, tracking, or telemetry
- Offline-capable

### 5. Hybrid Search Wins

Users don't know whether they remember exact words or concepts. Hybrid search handles both:
- "React hooks" finds articles with those exact words
- PLUS articles about component lifecycle and state management
- Best of both worlds

## Production Features

After two weeks, Frank Bookmark shipped with:

- Self-contained Chrome extension
- Three search modes (keyword, semantic, hybrid)
- AI-powered content extraction
- 384-dimension vector embeddings
- Persistent storage
- Automated CI/CD
- Comprehensive documentation

## What's Next?

Optional enhancements we're considering:
- Cross-device sync (optional cloud backup)
- Tags and folders
- Bulk operations
- Query suggestions
- Cross-browser support (Firefox, Edge, Safari)

## Try It Yourself

Frank Bookmark is open source. The code, documentation, and all our research notes are available. You can see exactly how we built it, what worked, what didn't, and why.

This project proves that privacy-first AI applications are not just possible—they're practical. Modern browsers are powerful enough to run sophisticated ML workloads without compromising user privacy.

## Research Documentation

All eight experiments are documented in detail:
- Notebooks: Daily experiment logs with questions, hypotheses, and results
- Insight Memos: Durable knowledge and recommendations
- SUMMARY.md: Complete project timeline and learnings

The full research process is transparent, including all the dead ends and pivots. Real R&D isn't linear, and we documented it that way.
