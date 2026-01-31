+++
date = '2026-01-28T17:43:51-03:00'
draft = false
title = 'Frank Lab AI'
+++

## Welcome to Frank Lab AI

Frank Lab AI is an AI R&D initiative focused on exploring the frontiers of artificial intelligence through hands-on experimentation and practical implementation. We believe in learning by doing, documenting everything, and sharing knowledge openly.

## What We Do

We investigate emerging AI technologies and their practical applications in areas like:

- **Browser-Based AI** - Running ML workloads entirely client-side using WebGPU, WebAssembly, and modern web APIs for privacy-first applications
- **AI Agents & Automation** - Building intelligent workflows for browser automation using tools like Browser Use, Playwright, and remote browser control with Steel
- **Computer Vision & Document Processing** - Experimenting with vision language models for receipt analysis, document extraction, and OCR alternatives
- **Search & Information Retrieval** - Implementing hybrid search systems combining BM25 lexical ranking with semantic vector embeddings
- **Privacy-First Solutions** - Creating AI applications that process data locally, keeping sensitive information on-device

## Our Approach

Every experiment is documented with:

- **Clear research questions** - What are we trying to learn?
- **Hypotheses** - What do we believe will work?
- **Results** - What actually happened, both successes and failures
- **Insights** - Durable knowledge that informs future work

We embrace a research-first methodology, prioritizing learning over production, and documenting failures alongside successes.

## Recent Projects

### Frank Bookmark

An AI-powered Chrome extension for intelligent bookmark management. Through 9 documented experiments, we evolved from a simple prototype to a production-grade system with state-of-the-art search.

**Current capabilities:**
- **BM25 + FTS5** for relevance-ranked lexical search
- **Vector embeddings** using Transformers.js for semantic understanding
- **Hybrid search** combining both approaches for best results
- **Complete privacy** - zero data leaves your device, fully offline-capable
- **Sub-300ms search** with 1,000+ bookmarks

**The evolution:**
- Experiments 1-4: DuckDB → SQL.js → sqlite-vec
- Experiments 5-7: Added AI-powered semantic search
- Experiment 8: CI/CD automation
- Experiment 9: BM25 relevance ranking

**Read the story:**
- [Initial journey (Experiments 1-8)](/posts/frank-bookmark-journey)
- [Evolution to production (Experiment 9)](/posts/frank-bookmark-evolution)
- [Search evolution explained](/posts/search-evolution-simple-to-sophisticated)
- [BM25 implementation deep-dive](/posts/bm25-fts5-search-relevance)

## Get Involved

All our research is documented in the [lab-work repository](https://github.com/frankenstein-ai), where we maintain:

- Daily notebooks of experiments
- Insight memos with actionable findings
- Weekly summaries of research progress

Follow along, learn from our experiments, and build upon our findings.
