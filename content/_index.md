+++
date = '2026-01-28T17:43:51-03:00'
draft = false
title = 'Frank Lab AI'
+++

## Welcome to Frank Lab AI

Frank Lab AI is an AI R&D initiative focused on exploring the frontiers of artificial intelligence through hands‑on experimentation and practical implementation. We believe in learning by doing, documenting everything, and sharing knowledge openly.

## What We Do

We investigate emerging AI technologies and their practical applications in areas like:

- **Browser-Based AI** - Running ML workloads entirely client-side using WebGPU, WebAssembly, and modern web APIs for privacy-first applications
- **Mobile AI** - On-device inference on iOS and Android using ONNX Runtime, exploring embeddings and text generation with quantized models
- **AI Agents & Automation** - Building intelligent workflows for browser automation using tools like Browser Use, Playwright, and remote browser control with Steel
- **Computer Vision & Document Processing** - Experimenting with vision language models for receipt analysis, document extraction, and OCR alternatives
- **Search & Information Retrieval** - Implementing hybrid search systems combining BM25 lexical ranking with semantic vector embeddings and Reciprocal Rank Fusion
- **Privacy-First Solutions** - Creating AI applications that process data locally, keeping sensitive information on-device, whether in browsers or mobile apps
- **AI Agent Skills Marketplace** - Curating and managing a community‑driven marketplace for Claude Code skills, enabling developers to share and install reusable plugins

## Our Approach

Every experiment is documented with:

- **Clear research questions** - What are we trying to learn?
- **Hypotheses** - What do we believe will work?
- **Results** - What actually happened, both successes and failures
- **Insights** - Durable knowledge that informs future work

We embrace a research-first methodology, prioritizing learning over production, and documenting failures alongside successes.

## Recent Projects

### Frank Bookmark

An AI-powered Chrome extension for intelligent bookmark management. Through 11 documented experiments, we evolved from a simple prototype to a production-grade system with state‑of‑the‑art search and bulletproof data safety.

**Current capabilities:**
- **BM25 + FTS5** for relevance-ranked lexical search
- **Vector embeddings** using Transformers.js for semantic understanding
- **Reciprocal Rank Fusion (RRF)** for optimal hybrid search ranking
- **Dual persistence** (OPFS + chrome.storage.local) ensuring zero data loss
- **Complete privacy** - zero data leaves your device, fully offline-capable
- **Sub-300ms search** with 1,000+ bookmarks

**The evolution timeline:**
- Experiments 1-4: Storage architecture (DuckDB → SQL.js → sqlite-vec)
- Experiments 5-7: AI-powered semantic search
- Experiment 8: CI/CD automation
- Experiment 9: BM25 relevance ranking upgrade
- Experiment 10: Reciprocal Rank Fusion for hybrid search
- Experiment 11: Dual persistence for data safety

**Read the complete story:**
- [Initial journey (Experiments 1-8)](/posts/frank-bookmark-journey)
- [Evolution to production (Experiment 9)](/posts/frank-bookmark-evolution)
- [Search evolution explained](/posts/search-evolution-simple-to-sophisticated)
- [BM25 implementation](/posts/bm25-fts5-search-relevance)
- [Reciprocal Rank Fusion](/posts/reciprocal-rank-fusion-hybrid-search)
- [Dual persistence strategy](/posts/dual-persistence-browser-data-safety)
- [Three Search Modes: Keyword, Semantic, and Hybrid](/posts/three-search-modes-bookmark-systems)
- [Chrome Extension Architecture for AI Workflows](/posts/extension-architecture-ai-workflows)
- [Storage Evolution: Finding the Right Database for Vector Search](/posts/storage-evolution-vector-search)
- [Browser-Based AI: A Technical Feasibility Study](/posts/browser-based-ai-feasibility)

### Mobile AI with ONNX

Exploring on-device AI for iOS and Android using ONNX Runtime and Flutter. Can you run embeddings and text generation entirely on a phone?

**What we built:**
- **Embeddings** - all-MiniLM-L6-v2 running in <200ms on mobile
- **Text generation** - SmolLM2-135M generating text in ~3s for 20 tokens
- **INT8 quantization** - 4x size reduction, 4x speed improvement
- **KV cache** - 3x speedup for autoregressive generation
- **Complete privacy** - all processing on-device, no API calls

**Key findings:**
- INT8 quantization is mandatory for iOS compatibility
- KV cache is critical for text generation performance
- Real-time embeddings are practical for mobile apps
- 135M parameter models are the sweet spot for mobile

**Read about it:**
- [Running AI models on mobile](/posts/mobile-ai-onnx-flutter)

### Frank Blog Content Generator CLI

An AI‑driven command line tool for generating blog posts from code repositories. It supports incremental generation, multiple LLM back‑ends, dry‑run previewing, and cross‑platform releases.

**Key findings:**
- Incremental generation reduces unnecessary recomputation
- Multiple LLM back‑ends allow switching providers without changing code
- Dry‑run preview lets developers see output before committing
- Cross‑platform binaries simplify deployment on Linux, macOS, and Windows

**Read about it:**
- [Building a Robust, Extensible CLI for AI‑Driven Blog Generation](/posts/2026-02-14-building-a-robust-extensible-cli-for-ai-driven-blog-generation)

### Automating Hugo Site Updates with Claude 4.6

An AI‑driven workflow that automatically updates Hugo site configuration, commits, and generates content using Claude 4.6.

**Key benefits:**
- Incremental, deterministic updates
- Persistent config generation
- Commit‑based generation

**Read about it:**
- [Automating Hugo Site Updates with Claude 4.6: From Persistent Configs to Commit‑Based Generation](/posts/2026-02-14-automating-hugo-site-updates-with-claude-4-6-from-persistent)

### Claude Code Agent Skills Marketplace

An AI-powered marketplace for Claude Code skills, allowing developers to share and install reusable plugins. This project extends our AI Agents & Automation work by providing a community‑driven plugin ecosystem.

**Read the complete story:**
- [Building a Multi‑Skill Marketplace for Claude Code: From a Single Skill to a Plug‑In Hub](/posts/2026-02-19-building-a-multi-skill-marketplace-for-claude-code-from-a-si)

### Flutter Virtual Try‑On

A mobile‑first solution that lets users upload a photo of themselves and a clothing item, then fuses the two to produce a realistic image of the user wearing the garment. Powered by IDM‑VTON hosted on Replicate, it runs entirely on the device, keeping user data local and results fast.

**What we built:**
- **Mobile‑first design** – iOS and Android only, no web desktop
- **Zero‑backend** – all inference via Replicate, no proprietary servers
- **Fast turnaround** – results in a few seconds
- **Cross‑platform** – built with Flutter, single codebase

**Key findings:**
- The main constraint was keeping the app lightweight while still supporting image‑to‑image inference.
- Using Replicate’s Files API kept backend complexity to a minimum, but required careful handling of API rate limits.
- A mobile‑first UI with camera and gallery pickers made the experience smooth.

**Read about it:**
- [Building a Flutter Virtual Try‑On App with IDM‑VTON and Replicate](/posts/2026-02-20-building-a-flutter-virtual-try-on-app-with-idm-vton-and-repl)

## Get Involved

All our research is documented in the [lab-work repository](https://github.com/frankenstein-ai), where we maintain:

- Daily notebooks of experiments
- Insight memos with actionable findings
- Weekly summaries of research progress

Follow along, learn from our experiments, and build upon our findings.
