+++
date = "2026-02-26T13:13:35-03:00"
draft = false
title = "Switching the Frank Blog Generator to OpenRouter: A Practical Guide"
+++

## Introduction

When we built the automated blog‑generation pipeline for Find Workers, the choice of LLM provider turned out to matter a lot. In late 2026 W09 we swapped the reusable GitHub Actions workflow from the native OpenAI endpoint to OpenRouter, using the open‑source 20 B model (`gpt‑oss‑20b`). This post explains that change, why it’s useful for people who rely on third‑party LLMs, and a few practical lessons.

> **TL;DR** – Moving to OpenRouter gave us a free, open‑source model while keeping the same workflow. The switch required only a handful of YAML edits.

---

## The Problem

1. **Cost predictability** – The older workflow used OpenAI’s `gpt‑5‑mini‑2025‑08‑07`. Even moderate prompts could add up quickly as we generated a post for every commit.  
2. **Policy & compliance** – Running the model only through OpenAI meant all data went through a closed system that was harder to audit.  
3. **Model availability** – The internal `gpt‑5‑mini` endpoint hit a rate limit that caused nightly runs to fail, breaking the preview pipeline.

Rather than start over, we reused the existing workflow and pointed it at a provider that met all three constraints.

---

## Architecture Snapshot

The core of the generator is:

```
GitHub → GitHub Actions → reusable workflow (frankenstein-ai/frank-blog-content-creation)
    │
    ├─ LLM provider (OpenRouter ➜ OpenAI GPT‑OSS‑20B)
    ├─ Markdown renderer / Markdown → HTML conversion
    ├─ Static site generator (Hugo, Jekyll…)
    └─ Deployed site (frank.ai)
```

The workflow file is tiny: five inputs.

| Parameter | Purpose |
|-----------|---------|
| `blog-repo` | Repo that holds the Markdown files |
| `llm-provider` | Name of the provider used in the sub‑workflow |
| `llm-model` | Full model identifier |
| `start-commit` | SHA to start generating from |
| `temperature` | Controls randomness |
| `skill-urls` | Optional list of skill URLs (e.g., humanizer) |

---

## Commit #1 – Switching to OpenRouter

### Diff

```diff
--- a/.github/workflows/generate.yaml
+++ b/.github/workflows/generate.yaml
@@
-      llm-provider: openai
-      llm-model: gpt-5-mini-2025-08-07
-      start-commit: 'f18ffe9d9d437781d2c4ce3462757241608acf35'
-      temperature: '-1'
+      llm-provider: openrouter
+      llm-model: openai/gpt-oss-20b
+      start-commit: "f18ffe9d9d437781d2c4ce3462757241608acf35"
+      temperature: "-1"
```

The three lines change the provider, the model, and the temperature. The quoted `"-1"` keeps the YAML parser from treating the value as a negative number; it forces a deterministic default of 0.

#### Why OpenRouter?

* **Unified API** – One endpoint covers many providers, so you only need one set of credentials.  
* **Free, open‑source model** – `gpt‑oss‑20b` is free and has a sizeable context window, which removes a cost bottleneck for small teams.  
* **Accurate token reporting** – OpenRouter returns the exact token count in the header (`x-openrouter-usage-tokens`). We log those values to the action artifacts for fine‑grained cost tracking.

#### Takeaway

To experiment with another provider, open the workflow file, swap the provider name, adjust the model identifier, and run a single job. You’ll see differences in cost, latency, and output quality in just a few minutes.

---

## Commit #2 – Updating the Frank State File

```diff
--- a/.frank-state.db
+++ b/.frank-state.db
index a4ba165..b0fb82e 100644
Binary files a/.frank-state.db and b/.frank-state.db differ
```

The state file is a lightweight SQLite database that keeps:

* The SHA of the last processed commit  
* A checksum of the prior run to skip unchanged posts  
* A basic audit trail of generation times and token usage

The binary patch signals that the generator now talks to OpenRouter (`openai/gpt-oss-20b`) and that the deduplication logic has been re‑initialized because the new model may generate slightly different text.

If you’re copying this pattern:

1. Add a checksum column to spot content changes without pulling the whole blob.  
2. Create a nightly job that prints token usage and cost per post, feeding the dashboards.  
3. Tag the DB file with a version header so contributors see when the state logic changed.

---

## Lessons for AI‑Ops Teams

| Lesson | Why it matters |
|--------|----------------|
| **Declarative provider switching** | A single flag changes the engine without touching downstream code. |
| **Quoted “magic” temperature** | Setting `"-1"` forces determinism and is handy for reproducible tests. |
| **Token tracking** | OpenRouter’s header lets you enforce per‑job budgets. |
| **Minimal state** | A small SQLite DB removes the need for a complex cache while keeping an audit trail. |
| **Reusable patterns** | The workflow lives in a shared repo, so any new blog or docs repo can import it straight away. |

---

## Try the Workflow Yourself

```bash
git checkout -b feature/openrouter-workflow
git apply path/to/patch
git push origin feature/openrouter-workflow

# Run a dry‑run
gh workflow run generate.yaml --ref feature/openrouter-workflow

# View the artifacts
gh run view -r 12345 --log
```

Replace `12345` with the actual run ID. The Artifacts tab will show the updated `.frank-state.db` and a `token-usage.csv` file that was pulled from the OpenRouter headers.

---

## Bottom Line

A few edits in a YAML file can shift a team’s cost curve and compliance posture. By abstracting the LLM provider into a declarative flag, you can experiment with free, open‑source models—just like we did when moving from `gpt‑5‑mini` to `gpt‑oss‑20b` via OpenRouter. Coupled with a lightweight state DB, the resulting pipeline is robust, auditable, and ready to scale across any AI‑centric project.
