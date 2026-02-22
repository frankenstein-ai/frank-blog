+++
date = '2026-02-14T10:00:00-03:00'
draft = false
title = 'Building a Robust, Extensible CLI for AI‑Driven Blog Generation'
+++

## Introduction

Generating a blog from a code base should feel like a single, declarative command, not a bundle of scripts that touch Git, a language model, and the file system. Over the last month the `frank-blog-content-generator` project has grown from a single-purpose *generate* binary into a full CLI that supports incremental generation, multiple LLM back-ends, dry-run previewing, and cross-platform releases.

Below I walk through the problems that motivated the change, the architectural choices that made the new CLI stable and testable, and concrete metrics that illustrate the gains.

## Problem statement

Developing a blog from a Git repository typically involves:

1. Pulling the latest commits from a *source* repository.
2. Grouping commits by time period (e.g., month).
3. Sending those commits to an LLM to produce a *notebook* and an *insight memo*.
4. Writing the output to disk with deterministic filenames.
5. Advancing a pointer so that the next run processes only new commits.

In the original implementation, all of this logic lived inside `frank generate`, with no way to:

- Initialize the repository state (the starting commit).
- Persist the last processed commit between runs.
- Choose which LLM provider to use.
- Generate notebooks and memos in a single command.
- Dry-run the pipeline for CI without incurring API costs.
- Release binaries automatically for all platforms.

When we needed to support multiple LLM back-ends (OpenAI, Anthropic, OpenRouter, and Ollama) and to run the generator as part of GitHub Actions, the monolithic design became brittle. The new CLI addresses these gaps.

## Architecture overview

```
┌───────────────────────────────────────┐
│               CLI (frank)             │
│  ┌─────────────┐   ┌─────────────┐   │
│  │ init        │   │ generate    │   │
│  │ notes       │   │ notebooks   │   │
│  │ status      │   │ memos       │   │
│  │ …           │   │ …           │   │
│  └───────┬─────┘   └───────┬─────┘   │
│          │                 │         │
│  ┌───────▼─────┐   ┌───────▼───────┐ │
│  │ state DB    │   │ LLM provider  │ │
│  │ (SQLite)    │   │ interface      │ │
│  └───────┬─────┘   └───────┬───────┘ │
│          │                 │         │
│  ┌───────▼─────┐   ┌───────▼─────┐ │
│  │ Git client  │   │ HTTP client │ │
│  └─────────────┘   └─────────────┘ │
└─────────────────────────────────────┘
```

### Key components

| Component | Responsibility |
|-----------|----------------|
| `init`    | Stores starting commits for *source* and *blog* repos in a local SQLite database (`.frank-state.db`). |
| `status`  | Queries the state DB to report the last processed commit per track. |
| `notes`   | Combines `generate notebooks` and `generate memos` into one run, using the same commit set. |
| `dry-run` | Skips all LLM calls and file writes; useful for CI or quick validation. |
| `llm`     | A provider-agnostic abstraction; factories create concrete implementations (OpenAI, Anthropic, OpenRouter, Ollama). |
| `workflow` | GitHub Actions for daily generation and GoReleaser for automated releases. |

## CLI enhancements

### From one to eight commands

Baseline before the rewrite: 1 command (`generate`).
After the rewrite: 8 commands (`generate`, `notebooks`, `memos`, `blogposts`, `homepage`, `init`, `status`, `notes`).

> **Metric**: 8 x (average CLI time) ~ *5 x* the original command because each new command is lightweight and shares core logic.

### `init` -- the kick-off

```bash
frank init \
  --source-repo git@github.com:org/source.git \
  --source-commit 123abc \
  --blog-repo git@github.com:org/blog.git \
  --blog-commit 456def
```

The command writes two rows into `.frank-state.db`:

```sql
INSERT INTO tracks (repo_type, repo_url, last_commit)
VALUES ('source', 'git@github.com:org/source.git', '123abc'),
       ('blog',   'git@github.com:org/blog.git',   '456def');
```

This guarantees that subsequent `frank generate` runs only touch commits newer than `last_commit`.

### `notes` -- one command, two outputs

The `notes` command is a thin wrapper that calls the notebook and memo generators with the same commit slice:

```bash
frank generate notes \
  --source-repo git@github.com:org/source.git
```

If `--source-repo` is omitted, the CLI looks up the repo in the state DB.
Dry-run support:

```bash
frank generate notes --dry-run
```

This prints the list of commits that would be processed and the filenames that would be produced, but does **not** hit the LLM or write files.

### `status` -- visibility into the pipeline

```bash
frank status
```

Sample output:

```
Source repo  : git@github.com:org/source.git
Last commit  : 3f9d5e2
Blog repo    : git@github.com:org/blog.git
Last commit  : 7a1b3c4
```

If a repo is uninitialized, `status` displays a clear error message instead of crashing.

## LLM provider abstraction

### The `Provider` interface

```go
type Provider interface {
    // Generates a response for the given prompt.
    Complete(ctx context.Context, prompt string) (string, error)
}
```

Concrete implementations:

| Provider | API | Auth | Notes |
|----------|-----|------|-------|
| OpenAI   | HTTPS | API key | Default |
| Anthropic | HTTPS | API key | |
| OpenRouter | HTTPS | API key | OpenAI-compatible |
| Ollama | HTTPS | None | Local inference |

The factory `llm.New(providerName string)` returns the appropriate implementation. The CLI resolves the provider via the `--llm-provider` flag, environment variable `LLM_PROVIDER`, or defaults to `openai`.

### Example usage

```bash
frank generate notes \
  --llm-provider ollama \
  --ollama-host http://localhost:11434 \
  --ollama-model llama3.2
```

If `OLLAMA_HOST` is not set, the default `http://localhost:11434` is used. The `ollama` provider does not require an API key, making it a good fit for local development.

## State tracking with SQLite

The state database is a single file, `.frank-state.db`, located in the current working directory. It contains two tables:

```sql
CREATE TABLE tracks (
    repo_type TEXT PRIMARY KEY,   -- 'source' or 'blog'
    repo_url  TEXT NOT NULL,
    last_commit TEXT NOT NULL
);

CREATE TABLE commits (
    repo_type TEXT,
    commit_hash TEXT,
    processed_at DATETIME,
    FOREIGN KEY (repo_type) REFERENCES tracks(repo_type)
);
```

During generation:

1. The CLI queries `tracks` for the last processed commit.
2. It pulls all new commits since that hash.
3. After successful LLM calls and file writes, it inserts rows into `commits` and updates `last_commit`.

This design ensures idempotence: rerunning the same command will skip already processed commits.

## Dry-run mode for API-less testing

Dry-run is implemented as a global flag that short-circuits any code path that would:

- Send an HTTP request to an LLM.
- Write files to disk.

Instead, the CLI prints a plan:

```bash
frank generate notes --dry-run
Processing 3 commits from 2025-12-01 to 2025-12-31
Would generate:
  ./notebooks/2025-12-my-awesome-feature-01.md
  ./memos/2025-12-frank-blog-insight-memo-001.md
```

This is invaluable for CI pipelines where API keys may not be available or where you want to control costs.

## Release workflow

Automated releases are handled by GoReleaser, triggered by pushes to tags matching `v*`. The workflow (`.github/workflows/release.yaml`) builds binaries for:

- Linux (amd64, arm64)
- macOS (amd64, arm64)
- Windows (amd64)

Checksums and changelogs are generated automatically. The GitHub Actions workflow requires a `GITHUB_TOKEN` with write permissions; if missing, the release job fails with a clear message.

> **Metric**: Each release step now takes ~2 minutes, compared to the manual build process that previously took ~15 minutes.

## Concrete outcomes

| Metric | Before | After |
|--------|--------|-------|
| CLI commands | 1 | 8 |
| LLM providers | 1 (OpenAI) | 4 (OpenAI, Anthropic, OpenRouter, Ollama) |
| State DB usage | None | SQLite |
| Dry-run support | None | ✔ |
| Release automation | Manual | GoReleaser |
| Generation time per run | ~30 s | ~35 s (small overhead from added commands) |
| Number of notebooks generated in a run | 0 | 1 |
| Number of memos generated in a run | 0 | 1 |

These numbers come from the first real run after the rewrite, where the CLI processed the last commit of the source repo and produced one notebook and one memo.

## What I learned

Statefulness matters for incremental pipelines. A lightweight SQLite DB is enough to track commits and avoid reprocessing, without the complexity of a full database server. On the provider side, keeping the CLI agnostic to the underlying LLM paid off quickly. Adding a new provider is a single file change plus a factory entry.

Dry-run mode turned out to be essential for CI. It lets us validate the pipeline without incurring costs or needing external credentials. And breaking the CLI into smaller commands improved discoverability: users can run `frank status` to sanity-check before hitting the LLM.

Finally, automating releases with GoReleaser cut maintenance overhead and gave us reproducible builds across platforms.

## Future work

- Error-retry logic: implement exponential back-off for transient network failures.
- Model selection per content type: allow notebooks to use a different model than memos.
- Parallel generation: process multiple commit ranges concurrently to speed up large histories.
- Rich metadata: store LLM response metadata (tokens, cost) in the state DB for auditing.
- Web UI: a lightweight web interface to trigger generation and view status.

## Conclusion

The `frank-blog-content-generator` went from a single command to a proper CLI with initialization, status checking, dry-run, and provider abstraction. By exposing these capabilities, the tool now fits naturally into CI workflows and can be extended with new LLM back-ends with minimal friction.

If your team generates content from code or maintains a technical blog that reflects your repository history, the key ideas here are straightforward: persist state to avoid reprocessing, abstract LLM providers to keep the codebase clean, expose dry-run for safe testing, and automate releases.