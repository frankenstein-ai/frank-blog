+++
date = '2026-01-28T17:43:51-03:00'
draft = false
title = 'Frank Lab AI'
+++

## Welcome to Frank Lab AI

This is my AI research lab. I explore artificial intelligence through hands‑on experiments and practical builds, then I write about what I learn. Successes and failures both make it in.

## What I work on

I dig into emerging AI technologies and figure out how they hold up in practice. The areas I keep coming back to:

- Running ML workloads entirely in the browser using WebGPU, WebAssembly, and modern web APIs, so data never leaves the client
- On‑device inference on iOS and Android using ONNX Runtime, with a focus on embeddings and text generation from quantized models
- AI agent workflows for browser automation with tools like Browser Use, Playwright, and Steel for remote browser control
- Vision language models for receipt analysis, document extraction, and alternatives to traditional OCR
- Hybrid search systems that combine BM25 lexical ranking with semantic vector embeddings and Reciprocal Rank Fusion
- Local‑first AI applications that keep sensitive data on‑device, whether that means the browser or a phone
- A community‑driven marketplace for Claude Code skills where developers can share and install reusable plugins
- Self‑hosted secure file sharing with Frank Mega

## How I work

Every experiment gets documented with a research question, a hypothesis, the actual results (both what worked and what didn’t), and whatever insights came out of it that might be useful later.

I prioritize learning over polish. If something fails, I write about why.

## Recent projects

### Frank Bookmark

A Chrome extension for managing bookmarks with AI‑powered search. Over 11 documented experiments, I went from a rough prototype to a solid system with hybrid search and reliable data persistence.

It currently supports BM25 + FTS5 for lexical search, vector embeddings via Transformers.js for semantic search, and Reciprocal Rank Fusion (RRF) to combine the two. Data is stored with dual persistence using OPFS and chrome.storage.local, so nothing gets lost. Everything runs offline on your device. Search comes back in under 300 ms with 1,000+ bookmarks.

The project evolved in stages. Experiments 1 through 4 were about storage architecture, moving from DuckDB to SQL.js to sqlite‑vec. Experiments 5 through 7 added semantic search. Experiment 8 tackled CI/CD automation. Experiment 9 brought BM25 relevance ranking. Experiment 10 introduced Reciprocal Rank Fusion for hybrid search. Experiment 11 added dual persistence for data safety.

Read the full story:

- [Initial journey (Experiments 1‑8)](/posts/frank-bookmark-journey)
- [Evolution to production (Experiment 9)](/posts/frank-bookmark-evolution)
- [Search evolution explained](/posts/search-evolution-simple-to-sophisticated)
- [BM25 implementation](/posts/bm25-fts5-search-relevance)
- [Reciprocal Rank Fusion](/posts/reciprocal-rank-fusion-hybrid-search)
- [Dual persistence strategy](/posts/dual-persistence-browser-data-safety)
- [Three Search Modes: Keyword, Semantic, and Hybrid](/posts/three-search-modes-bookmark-systems)
- [Chrome Extension Architecture for AI Workflows](/posts/extension-architecture-ai-workflows)
- [Storage Evolution: Finding the Right Database for Vector Search](/posts/storage-evolution-vector-search)
- [Browser‑Based AI: A Technical Feasibility Study](/posts/browser-based-ai-feasibility)

### Mobile AI with ONNX

Can you run embeddings and text generation entirely on a phone? I used ONNX Runtime and Flutter to find out.

I got all‑MiniLM‑L6‑v2 running embeddings in under 200 ms on mobile. SmolLM2‑135M generates text in about 3 seconds for 20 tokens. INT8 quantization gave me a 4× size reduction and 4× speed improvement. Adding a KV cache brought another 3× speedup for autoregressive generation. All processing stays on‑device with no API calls.

The main takeaways: INT8 quantization is mandatory for iOS compatibility. KV cache is critical for text generation performance. Real‑time embeddings are practical on mobile. And 135M parameter models hit the sweet spot for what phones can handle.

- [Running AI models on mobile](/posts/mobile-ai-onnx-flutter)

### Frank Blog content generator CLI

A command line tool that generates blog posts from code repositories using AI. It supports incremental generation so it only processes what changed, multiple LLM back‑ends so I can swap providers without rewriting code, dry‑run previews, and cross‑platform releases for Linux, macOS, and Windows.

- [Building a Robust, Extensible CLI for AI‑Driven Blog Generation](/posts/2026-02-14-building-a-robust-extensible-cli-for-ai-driven-blog-generation)
- [From Commit History to Hugo Blog: A Practical Guide to Automating Content with Frank](/posts/2026-02-14-from-commit-history-to-hugo-blog-a-practical-guide-to-automa)
- [From Over‑Engineered to Lean: Refactoring the Frankenstein Blog Generator](/posts/2026-02-22-from-over-engineered-to-lean-refactoring-the-frankenstein-bl)
- [From Hard‑coded Post‑Processing to a Plug‑in Skill System in Frank](/posts/2026-02-22-from-hard-coded-post-processing-to-a-plug-in-skill-system-in)
- [Fine‑Grained State Control and Skill Re‑Downloads in frank‑blog‑content‑generator](/posts/2026-02-22-fine-grained-state-control-and-skill-re-downloads-in-frank-b)

### Automating Hugo site updates with Claude 4.6

A workflow that uses Claude 4.6 to automatically update Hugo site configuration, commit changes, and generate content. Updates are incremental and deterministic, config generation is persistent, and content generation is driven by commits.

- [Automating Hugo Site Updates with Claude 4.6: From Persistent Configs to Commit‑Based Generation](/posts/2026-02-14-automating-hugo-site-updates-with-claude-4-6-from-persistent)

### Claude Code agent skills marketplace

A marketplace for Claude Code skills where developers can share and install reusable plugins. This grew out of my AI agents work as a way to build a community‑driven plugin ecosystem.

- [Building a Multi‑Skill Marketplace for Claude Code: From a Single Skill to a Plug‑In Hub](/posts/2026-02-19-building-a-multi-skill-marketplace-for-claude-code-from-a-si)

### Flutter virtual try‑on

A mobile app that lets you upload a photo of yourself and a clothing item, then composites the two into a realistic image of you wearing the garment. It uses IDM‑VTON hosted on Replicate for inference. The app targets iOS and Android only, built with Flutter from a single codebase, with no backend server of my own.

The main challenge was keeping things lightweight while supporting image‑to‑image inference. Replicate's Files API kept backend complexity low, though I had to handle API rate limits carefully. A mobile‑first UI with camera and gallery pickers made the user flow straightforward.

- [Building a Flutter Virtual Try‑On App with IDM‑VTON and Replicate](/posts/2026-02-20-building-a-flutter-virtual-try-on-app-with-idm-vton-and-repl)

### Frank Mega

A lightweight, self‑hosted file‑sharing service built with Ruby on Rails, SQLite, and Tailwind CSS. It provides time‑limited links, download counters, invite‑only registration, and optional passkey/2FA authentication, all running behind a Cloudflare Tunnel. The service is designed for personal or small‑team use, with zero external services required.

- [Keeping FrankMega Secure and Dependable: Two Small Commit Stories that Make a Big Difference](/posts/2026-02-21-keeping-frankmega-secure-and-dependable-two-small-commit-sto)
- [FrankMega: Hardening, Deployment, and DX Enhancements](/posts/2026-02-21-frankmega-hardening-deployment-and-dx-enhancements)
- [From N+1 to Inline Previews: FrankMega’s Deep‑Dive into Security, UX
