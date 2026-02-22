+++
date = '2026-01-26T10:30:00-03:00'
draft = false
title = 'Browser-Based AI: A Technical Feasibility Study'
+++

## Why browser-based AI matters

Traditional bookmark systems rely on keyword searches and manual organization. They fail when you can't remember exact terms or URLs. An AI-powered system could understand meaning, not just match keywords. But can it run entirely in the browser?

This question matters beyond bookmarks. If we can run complete ML workloads client-side, that opens the door to privacy-first AI applications, offline-capable intelligent tools, zero server costs, fast local processing, and user data that never leaves the device.

## What I found

### Technical feasibility: confirmed

Modern browsers can handle complete ML workflows. Here's what I tested:

Transformers.js runs the all-MiniLM-L6-v2 model (384-dimension embeddings) reliably. It uses WebGPU acceleration when available and falls back to WASM for CPU. Model loading takes 3-5 seconds the first time.

sqlite-vec provides native vector similarity search through SQL-based vector operations, including a `vec_distance_cosine()` function. It scales to 1,000+ vectors and integrates with standard SQL queries.

OPFS (Origin Private File System) handles persistent storage. It survives browser restarts, reads and writes quickly, and works well for database files.

Chrome Extensions tie it all together. Service Workers run persistently, have access to all Chrome APIs, can fetch cross-origin content, and maintain database connections.

### Performance: good enough for real use

Real-world measurements with 1,000+ bookmarks:

| Operation | Time | Notes |
|-----------|------|-------|
| Model loading | 3-5s | One-time, can preload |
| Keyword search | < 100ms | Always available |
| Semantic search | < 200ms | After model loaded |
| Hybrid search | < 300ms | Best results |
| Save + embed | < 500ms | Background operation |

These numbers are fast enough for interactive use. There's no perceptible lag in the search experience.

### Storage: sqlite-vec wins

I tested three storage solutions:

DuckDB-Wasm is a capable SQL database with documented native vector functions, but it doesn't work in Service Workers. Great for web apps, unusable for extensions.

SQL.js is a mature SQLite implementation that works everywhere, but it has no native vector functions. Reliable, though it forces you into manual similarity calculation.

sqlite-vec is SQLite with a vector extension. It works in extensions and has native `vec_distance_cosine()`. This turned out to be the right choice.

### Architecture: self-contained extensions

The architecture that worked:

```
Chrome Extension
├── Popup (UI)
│   ├── Quick save interface
│   ├── Search input
│   └── Results display
│
└── Background Service Worker
    ├── AI Model (Transformers.js)
    ├── Database (sqlite-vec + OPFS)
    ├── Content Extractor (Readability)
    └── Search Engine (keyword + semantic + hybrid)
```

A few things I learned from this structure: the extension is self-contained with no localhost dependency. The popup handles UI only, while long operations run in the background. Data stays local through OPFS storage. And the model loads once and gets reused indefinitely.

## When it works

This approach fits well when you have modern browsers (Chrome 90+, Firefox 88+, Edge 90+, Safari 15+), WebAssembly support, and OPFS availability.

Good use cases include privacy-sensitive applications, offline-first workflows, large personal datasets (1,000+ items), and situations where users value privacy over cloud sync. Think bookmark management, personal note search, document organization, research paper libraries, or code snippet collections.

## When it doesn't work

This approach struggles with very old browsers (no WebAssembly, OPFS, or modern JS), DuckDB-Wasm in Service Workers, cross-origin database sharing, and manual vector similarity at scale beyond 10,000 items.

If you need real-time sync across devices, collaborative features, social sharing, or cloud backup, you'll need a server component. Some of these (like cloud backup) could be added as optional features later.

## Recommendations

### For browser-based AI bookmark systems

The technology stack that worked for me:

1. sqlite-vec for storage + vector search
2. Transformers.js for ML inference
3. OPFS for persistence
4. Vite for bundling

Architecture: a self-contained Chrome extension with a background Service Worker for AI, popup for UI only, and no localhost dependency.

For search, default to hybrid (50% keyword + 50% semantic), allow mode selection, generate embeddings on save, and load the model in the background.

Performance tips: preload the AI model on extension install, use native SQL vector functions, implement OPFS persistence, and cache query embeddings for repeated searches.

### Where this approach fits best

- Privacy-first bookmark management
- Offline usage
- Zero server costs
- Fast local processing
- Large bookmark collections (1,000-10,000)

### When to consider alternatives

- Cross-device synchronization (add optional cloud backup)
- Collaborative features (requires server architecture)
- Real-time updates across devices (requires server)
- Very large datasets (>10,000 items may need optimization)

## What else could be built this way

There's a broader set of applications where this approach could work. Privacy-first tools like personal knowledge management, email search with semantic understanding, photo organization with local ML, and document analysis without cloud upload. Offline tools for travel, field research, medical applications with strict privacy requirements, and financial tools handling sensitive data. And zero-cost tools for open-source AI, academic research, personal productivity, and education.

## Wrapping up

Modern browsers are capable AI platforms. They can run complete ML workflows including embedding generation, vector search, and semantic analysis, all client-side with solid performance.

The Frank Bookmark project showed me this is practical. The technology works, the architecture holds up, and the performance is there.

## Technical details

Models used:
- all-MiniLM-L6-v2 (384-dimensional embeddings)
- Mozilla Readability (content extraction)

Libraries:
- @xenova/transformers (Transformers.js)
- @dao-xyz/sqlite3-vec (sqlite-vec)
- Vite (bundling)

Browser APIs:
- WebAssembly
- OPFS (Origin Private File System)
- Chrome Extension API
- Service Workers

All documented and open source.
