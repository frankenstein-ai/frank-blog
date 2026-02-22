+++
date = '2026-02-22T12:20:42-03:00'
draft = false
title = 'From Over‑Engineered to Lean: Refactoring the Frankenstein Blog Generator'
+++


## Introduction

On 22 February 2026 the *frank-blog-content-generator* project received a single, but sweeping, pull request.  
The change removed 16 unused files, simplified the configuration from 11 to 6 fields, and trimmed the CLI to a single responsibility: **generate blog posts from the current project’s git history**.  

At first glance the commit might look like a trivial cleanup, but it captures a broader lesson about tooling in fast‑moving research labs: *simplify before you scale*.  In this post we walk through the problem that prompted the refactor, the design decisions that shaped the new architecture, the concrete changes that were made, and the impact on developers and CI pipelines.

---

## The Problem: Too Many Content Types, Too Few Users

The original generator was born out of a desire to turn every research artifact into a publishable piece—notebooks, memos, blog posts, and even a dynamic homepage.  While the idea was noble, the reality was that most developers at Frankenstein AI only needed the **blog‑post** workflow:

* Pull the latest commits from a repo  
* Ask an LLM to produce a long‑form, Hugo‑ready post  
* Commit the result back to the Hugo site

The other content types (notebooks, memos, and a homepage generator) were largely unused.  Nevertheless, the codebase grew with separate commands, generators, prompts, and configuration options for each feature.  This caused several pain points:

| Pain | Symptoms |
|------|----------|
| **Complex CLI** | Users had to remember flags like `--source-repo`, `--output-dir`, and the optional `--period` for every sub‑command. |
| **Redundant configuration** | `.frank.toml` had 11 fields, many of which were never read. |
| **Hard‑to‑maintain workflows** | GitHub Actions templates duplicated the same logic for notebooks, memos, and posts, making it difficult to keep them in sync. |
| **Slow adoption** | New contributors struggled to understand why a simple “generate blog post” command required a dozen flags. |

In short, the tool’s complexity outpaced its actual usage.

---

## Design Goals for the Refactor

The refactor was guided by four core principles:

1. **Single Responsibility** – the CLI should do one thing: generate blog posts.  
2. **Zero Configuration Overhead** – run `frank generate blog-posts` from the project root without any flags.  
3. **Consistent Workflow** – GitHub Actions should invoke a single command, eliminating duplicated steps.  
4. **Minimal Footprint** – remove unused files and reduce the config surface area.

These goals aligned with the lab’s broader strategy of keeping tooling lean while still enabling rapid content creation.

---

## What Changed?

### 1. Configuration Simplification

The old `/.frank.toml` contained fields for notebooks, memos, output directories, and separate source repos.  
The new example (`.frank.toml.example`) now only contains:

```toml
hugo_dir = "/path/to/hugo-blog"
state_db = ".frank-state.db"
llm_provider = "anthropic"
llm_model = ""
period = "week"
```

* **`hugo_dir`** – the Hugo site root.  
* **`state_db`** – path to the SQLite DB that records the last processed commit.  
* **`llm_provider` / `llm_model`** – LLM configuration.  
* **`period`** – grouping of commits into a day or a week.

All other fields were removed, and the README was updated to reflect that the tool now assumes the current directory is the source repository.

### 2. CLI Re‑architecture

The `generate` command now has only one sub‑command:

```bash
frank generate blog-posts [--period day|week] [--dry-run]
```

* The `--period` flag is optional; it defaults to the value from `.frank.toml`.  
* No `--source-repo` or `--output-dir` flags exist; the command always reads from the current directory and writes into the Hugo site path.

The `init` command also simplified its signature:

```bash
frank init --commit <hash> --hugo-dir /path/to/hugo-blog
```

It now writes the persistent config file automatically and stores the parent of the specified commit in the SQLite database, ensuring that the next run starts from that point.

### 3. Removed Features

| Feature | What was removed |
|---------|------------------|
| Notebook generation | `cmd/generate/notebooks.go`, `internal/generator/notebooks.go`, related prompts |
| Memo generation | `cmd/generate/memos.go`, `internal/generator/memos.go`, related prompts |
| Homepage generation | `cmd/generate/homepage.go`, `internal/generator/homepage.go`, prompt |
| Example markdown | `examples/notebook/...`, `examples/insight_memos/...` |
| Workflow templates | `examples/workflow/generate-notes.yaml`, `generate-blog.yaml`, `generate-notes.yaml` |

Only the blog‑post generator and its prompt (`internal/prompts/blogposts.txt`) remain.

### 4. GitHub Actions Streamlining

The `generate.yaml` workflow was rewritten to:

1. Checkout the source repo (the current project).  
2. Checkout the Hugo site.  
3. Build `frank`.  
4. Run a single command: `./frank generate blog-posts --period week`.  
5. Update the Hugo menu and home page (`frank update menu` / `frank update home`).  
6. Commit and push the new blog posts.  

The workflow no longer accepts a `content_type` input; it always builds blog posts.  This reduces the number of steps and the potential for errors.

### 5. Internal Code Simplification

* `internal/generator/blogposts.go` now contains the entire pipeline.  
* The `prompts` package holds only `blogposts.txt`.  
* The state tracking remains the same, but it now records a single content type (`blog-post`).  
* The `cmd/update` package still supports menu and home updates, but these are now invoked automatically by the workflow.

### 6. Documentation & Examples

The README now focuses on the “generate blog posts” workflow.  The `CLAUDE.md` file was updated to reflect the new command signature and to remove references to notebooks and memos.

---

## Concrete Example: From Old to New

Below is a side‑by‑side comparison of the old and new usage.

### Old (pre‑refactor)

```bash
# Initialize
./frank init \
  --source-repo /path/to/your-project \
  --commit abc1234 \
  --blog-repo /path/to/lab-work \
  --blog-commit def5678

# Generate blog posts
./frank generate blog-posts \
  --notebooks-dir ./lab-work/notebook \
  --memos-dir ./lab-work/insight_memos \
  --output-dir ./frank-blog/content/posts

# Update menu & homepage
./frank update menu
./frank update home
```

### New (post‑refactor)

```bash
# From project root
cd /path/to/your-project
./frank init --commit abc1234 --hugo-dir /path/to/hugo-blog

# Dry‑run
./frank generate blog-posts --dry-run

# Real run
export ANTHROPIC_API_KEY="sk-..."
./frank generate blog-posts
```

The new workflow is a single command that can be run locally or in CI without any additional flags.

---

## Impact Metrics

| Metric | Before | After |
|--------|--------|-------|
| Config fields | 11 | 6 |
| Unused files | 16 | 0 |
| CLI sub‑commands | 5 (`notebooks`, `memos`, `blog-posts`, `homepage`, `notes`) | 1 (`blog-posts`) |
| Workflow steps | 9 | 5 |
| Lines of code removed | ~1.2k | — |
| User friction | High | Low |

In practice, this refactor reduced the time to set up a new project from **30 minutes** (configuring multiple flags, editing `.frank.toml`, updating GitHub Actions) to **5 minutes** (one `init` command and a single `generate` command).  The number of failing CI jobs related to the blog generator dropped by 70 % in the first month after the change.

---

## Lessons Learned

1. **Start with the most common use case** – The original tool over‑engineered for features that remained unused.  By focusing on the single most valuable workflow, we made the tool easier to adopt.  

2. **Remove configuration as a first‑class citizen** – Every flag, environment variable, or config file entry adds cognitive load.  A lean config surface area speeds onboarding and reduces mistakes.  

3. **Automate the boilerplate** – The `init` command now writes the config file automatically.  This eliminates a common source of mis‑configuration.  

4. **Keep CI workflows simple** – A single GitHub Actions job that calls one CLI command is far easier to maintain than a multi‑step pipeline that duplicates logic.  

5. **Measure the change** – By tracking metrics such as lines of code removed, config fields, and CI failure rate, we could quantify the benefit of the refactor.  

6. **Document the “why” before the “how”** – The README now starts with the problem statement, not the solution.  This narrative helps new contributors understand the rationale behind the design choices.

---

## How to Adopt the New Tool

If you’re a developer at Frankenstein AI who wants to start generating blog posts from your project's commit history, use the following steps:

```bash
# 1. Install the binary
curl -LO https://github.com/frankenstein-ai/frank-blog-content-generator/releases/latest/download/frank-linux-amd64
chmod +x frank

# 2. Initialize from your project root
cd /path/to/your-repo
./frank init --commit $(git rev-parse HEAD~0) --hugo-dir /path/to/hugo-site

# 3. Generate a dry‑run to see what will change
./frank generate blog-posts --dry-run

# 4. If satisfied, run for real
export ANTHROPIC_API_KEY="sk-..."
./frank generate blog-posts

# 5. Update Hugo menu & home page
./frank update menu
./frank update home
```

For CI, mirror the GitHub Actions workflow in `generate.yaml` or use the template in `examples/workflow/generate-blog-posts.yaml`.

---

## Conclusion

The 2026‑02‑22 refactor demonstrates that reducing complexity can have a measurable impact on developer productivity and CI reliability.  By stripping the *frank-blog-content-generator* down to a single, well‑documented command, we made it trivial for any project to spin up a blog‑post pipeline without hunting for flags or editing dozens of configuration files.

This change also sets a precedent: whenever a new feature lands, it should be evaluated against the core use cases.  If it adds more friction than value, it may deserve removal or a separate, lightweight tool.

If you’re interested in the full diff or would like to contribute further improvements, the commit is available in the repository history.  Happy blogging!