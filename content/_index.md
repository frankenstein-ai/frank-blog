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

### Smart Wardrobe

A Flutter mobile app for managing your personal closet, building outfits, getting outfit suggestions, and virtual try‑on powered by IDM‑VTON via Replicate.

- [From single‑screen try‑on to a local‑first Smart Wardrobe powered by OpenRouter](/posts/2026-03-05-from-single-screen-try-on-to-a-local-first-smart-wardrobe-po)
- [From a single‑screen try‑on to a full smart wardrobe: building an offline‑first Flutter app](/posts/2026-03-03-from-a-single-screen-try-on-to-a-full-smart-wardrobe-buildin)
- [From CLI to Makefile: Simplifying Android Builds and Test Install for Smart Wardrobe](/posts/2026-03-05-from-cli-to-makefile-simplifying-android-builds-and-test-ins)

### Frank Blog content generator CLI

A command line tool that generates blog posts from code repositories using AI. It supports incremental generation so it only processes what changed, multiple LLM back‑ends so I can swap providers without rewriting code, dry‑run previews, and cross‑platform releases for Linux, macOS, and Windows.

- [Building a Robust, Extensible CLI for AI‑Driven Blog Generation](/posts/2026-02-14-building-a-robust-extensible-cli-for-ai-driven-blog-generation)
- [From Commit History to Hugo Blog: A Practical Guide to Automating Content with Frank](/posts/2026-02-14-from-commit-history-to-hugo-blog-a-practical-guide-to-automa)
- [Automating Hugo Site Updates with Claude 4.6: From Persistent Configs to Commit‑Based Generation](/posts/2026-02-14-automating-hugo-site-updates-with-claude-4-6-from-persistent)
- [From Over‑Engineered to Lean: Refactoring the Frankenstein Blog Generator](/posts/2026-02-22-from-over-engineered-to-lean-refactoring-the-frankenstein-bl)
- [From Hard‑coded Post‑Processing to a Plug‑in Skill System in Frank](/posts/2026-02-22-from-hard-coded-post-processing-to-a-plug-in-skill-system-in)
- [Fine‑Grained State Control and Skill Re‑Downloads in frank‑blog‑content‑generator](/posts/2026-02-22-fine-grained-state-control-and-skill-re-downloads-in-frank-b)
- [Switching the Frank Blog Generator to OpenRouter: A Practical Guide](/posts/2026-02-26-switching-the-frank-blog-generator-to-openrouter-a-practical)
- [From a Binary File to a CI Pipeline: Building a Self‑Generating Blog](/posts/2026-02-26-from-a-binary-file-to-a-ci-pipeline-building-a-self-generati)
- [From Code to Comment: Strengthening Frank’s Prompt Integrity and State Management](/posts/2026-02-26-from-code-to-comment-strengthening-frank-s-prompt-integrity-)
- [Refining the Story: How We Made Frank Speak Like a Fellow Developer](/posts/2026-02-26-refining-the-story-how-we-made-frank-speak-like-a-fellow-dev)
- [From Commits to Content: Automating Technical Blog Generation with GitHub Actions and LLMs](/posts/2026-02-26-from-commits-to-content-automating-technical-blog-generation)
- [Automating Post‑Production: Building a GitHub Action That Turns Commit History into Blog Content](/posts/2026-02-26-automating-post-production-building-a-github-action-that-tur)

### Frank Bookmark (continued)

- [Three Search Modes: Keyword, Semantic, and Hybrid](/posts/three-search-modes-bookmark-systems)

### Frank Bookmark (continued)

- [Chrome Extension Architecture for AI Workflows](/posts/extension-architecture-ai-workflows)
- [Storage Evolution: Finding the Right Database for Vector Search](/posts/storage-evolution-vector-search)

### Frank Bookmark (continued)

- [Browser‑Based AI: A Technical Feasibility Study](/posts/browser-based-ai-feasibility)

### Flutter virtual try‑on

A mobile app that lets you upload a photo of yourself and a clothing item, then composites the two into a realistic image of you wearing the garment. It uses IDM‑VTON hosted on Replicate for inference. The app targets iOS and Android only, built with Flutter from a single codebase, with no backend server of my own.

The main challenge was keeping things lightweight while supporting image‑to‑image inference. Replicate's Files API kept backend complexity low, though I had to handle API rate limits carefully. A mobile‑first UI with camera and gallery pickers made the user flow straightforward.

- [Building a Flutter Virtual Try‑On App with IDM‑VTON and Replicate](/posts/2026-02-20-building-a-flutter-virtual-try-on-app-with-idm-vton-and-repl)
- [Building a Flutter Virtual Try‑On App Powered by Replicate’s IDM‑VTON](/posts/2026-02-20-building-a-flutter-virtual-try-on-app-powered-by-replicate-s)
- [Keeping CI in Sync: Updating Frank’s State in a Flutter Try‑On App](/posts/2026-02-26-keeping-ci-in-sync-updating-frank-s-state-in-a-flutter-try-o)
- [From a single‑screen try‑on to a full smart wardrobe: building an offline‑first Flutter app](/posts/2026-03-03-from-a-single-screen-try-on-to-a-full-smart-wardrobe-buildin)

### Frank Mega

A lightweight, self‑hosted file‑sharing service built with Ruby on Rails, SQLite, and Tailwind CSS. It provides time‑limited links, download counters, invite‑only registration, and optional passkey/2FA authentication, all running behind a Cloudflare Tunnel. The service is designed for personal or small‑team use, with zero external services required.

- [Keeping FrankMega Secure and Dependable: Two Small Commit Stories that Make a Big Difference](/posts/2026-02-21-keeping-frankmega-secure-and-dependable-two-small-commit-sto)
- [FrankMega: Hardening, Deployment, and DX Enhancements](/posts/2026-02-21-frankmega-hardening-deployment-and-dx-enhancements)
- [From N+1 to Inline Previews: FrankMega’s Deep‑Dive into Security, UX, and Performance](/posts/2026-02-21-from-n-1-to-inline-previews-frankmega-s-deep-dive-into-secur)
- [From Zero to Hero: Hardening and Feature‑Rich Self‑Hosted File Sharing with FrankMega](/posts/2026-02-21-from-zero-to-hero-hardening-and-feature-rich-self-hosted-fil)

### Find Workers

A production‑focused, MCP‑native backend that connects AI assistants to a Brazilian marketplace: worker search, bookings, Pix escrow via Woovi, and WhatsApp delivery. The posts below document architecture decisions, admin UI work, payments and webhook hardening, and operational fixes and recaps from sprint work.

Recent write‑ups:

- [Auditing 3.7k Tests: removing tautologies, adding behavioral asserts, and fixing a Redis leak](/posts/2026-02-24-auditing-3-7k-tests-removing-tautologies-adding-behavioral-a)
- [Serving the Admin SPA: iterating on build, deployment and dev ergonomics for Find Workers](/posts/2026-02-24-serving-the-admin-spa-iterating-on-build-deployment-and-dev-)
- [Automating Blog Post Generation with a Reusable LLM Workflow: lessons from 11 quick fixes](/posts/2026-02-24-automating-blog-post-generation-with-a-reusable-llm-workflow)
- [WhatsApp as a first‑class channel, fair broadcast matching, and hardening: find‑workers — 2026‑W09](/posts/2026-02-24-whatsapp-as-a-first-class-channel-fair-broadcast-matching-an)
- [Bringing the marketplace to WhatsApp: consumer orchestrator, interactive messages, and resilient state](/posts/2026-02-24-bringing-the-marketplace-to-whatsapp-consumer-orchestrator-i)
- [Hardening the Admin SPA: OTP Step‑Up, API Shape Alignments, and Playwright E2E](/posts/2026-02-23-hardening-the-admin-spa-otp-step-up-api-shape-alignments-and)
- [W09 recap: WhatsApp intent + price parsing, Alembic migration fixes, docs humanization and infra tweaks](/posts/2026-02-23-w09-recap-whatsapp-intent-price-parsing-alembic-migration-fi)
- [Hardening Find Workers: webhooks, PII sanitization, MCP key rotation, and simplifying WhatsApp](/posts/2026-02-22-hardening-find-workers-webhooks-pii-sanitization-mcp-key-rot)
- [Hardening the Admin Surface: PII, Audit, Webhook Quarantine, and Safe Payments](/posts/2026-02-22-hardening-the-admin-surface-pii-audit-webhook-quarantine-and)
- [Fine‑Grained State Control and Skill Re‑Downloads in frank‑blog‑content‑generator](/posts/2026-02-22-fine-grained-state-control-and-skill-re-downloads-in-frank-b)
- [From Hard‑coded Post‑Processing to a Plug‑in Skill System in Frank](/posts/2026-02-22-from-hard-coded-post-processing-to-a-plug-in-skill-system-in)
- [Wiring the Landing Page to Live Metrics and Polishing the Admin SPA](/posts/2026-02-22-wiring-the-landing-page-to-live-metrics-and-polishing-the-ad)
- [From Over‑Engineered to Lean: Refactoring the Frankenstein Blog Generator](/posts/2026-02-22-from-over-engineered-to-lean-refactoring-the-frankenstein-bl)
- [Hardening payments, batch jobs and privacy in Find Workers — W08 recap](/posts/2026-02-21-hardening-payments-batch-jobs-and-privacy-in-find-workers-w0)
- [Safe migrations and faster local search: adding backfills and a GIST spatial index to Find Workers](/posts/2026-02-21-safe-migrations-and-faster-local-search-adding-backfills-and)
- [WhatsApp FSM resilience, Pix QR flow, and message dispatcher — Find Workers (2026‑W08)](/posts/2026-02-21-whatsapp-fsm-resilience-pix-qr-flow-and-message-dispatcher-f)
- [Hardening Find Workers: payments reliability, webhook resilience, and the MCP‑first pivot (2026‑W08)](/posts/2026-02-21-hardening-find-workers-payment-authz-otp-safety-webhook-resi)
- [Shipping an admin UI for Find Workers: Vite + Tailwind + native TypeScript (tsgo)](/posts/2026-02-18-shipping-an-admin-ui-for-find-workers-vite-tailwind-native-t)
- [When an MCP sub‑app returns 500: lifespan, Docker layers and a smoother dev experience](/posts/2026-02-17-when-an-mcp-sub-app-returns-500-lifespan-docker-layers-and-a)
- [Hardening Pix payouts, Woovi KYC and WhatsApp delivery: lessons from a week of fixes](/posts/2026-02-16-hardening-pix-payouts-woovi-kyc-and-whatsapp-delivery-lesson)
- [Hardening payments, webhooks and concurrency for a Pix escrow marketplace](/posts/2026-02-15-hardening-payments-webhooks-and-concurrency-for-a-pix-escrow)
- [Designing Find Workers for Brazil: WhatsApp + Pix Escrow, MCP‑First, and a pragmatic stack](/posts/2026-02-15-designing-find-workers-for-brazil-whatsapp-pix-escrow-mcp-fi)
- [MCP tools, safer JWTs, and polish: what we shipped in 2026‑W07](/posts/2026-02-15-mcp-tools-safer-jwts-and-polish-what-we-shipped-in-2026-w07)
- [Hardening Pix Escrow, LGPD, and Payments: A two‑week sprint on Find Workers](/posts/2026-02-15-hardening-pix-escrow-lgpd-and-payments-a-two-week-sprint-on-)
- [Building a production‑ready MCP‑native marketplace: bookings, Pix escrow, WhatsApp, LGPD and a week of hardening](/posts/2026-02-15-building-a-production-ready-mcp-native-marketplace-bookings-)
- [Hardening Find Workers: payment authz, OTP safety, webhook resilience, and abuse controls (2026‑W07)](/posts/2026-02-15-hardening-find-workers-payment-authz-otp-safety-webhook-resi)
- [Building a production-grade Pix escrow integration: lessons from implementing Woovi (OpenPix) in Find Workers](/posts/2026-02-15-building-a-production-grade-pix-escrow-integration-lessons-f)

### Agent Skills Marketplace

A developer‑centric marketplace for Claude Code skills, enabling plugins to be shared and installed across projects.

- [Building a Multi‑Skill Marketplace for Claude Code: From a Single Skill to a Plug‑In Hub](/posts/2026-02-19-building-a-multi-skill-marketplace-for-claude-code-from-a-si)
- [Optimizing Developer Experience in the Frankenstein AI Agent‑Skills Marketplace](/posts/2026-02-19-optimizing-developer-experience-in-the-frankenstein-ai-agent‑skills-marketplace)
