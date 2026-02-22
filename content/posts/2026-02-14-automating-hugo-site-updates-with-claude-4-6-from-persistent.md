+++
date = '2026-02-14T20:44:25-03:00'
draft = false
title = 'Automating Hugo Site Updates with Claude 4.6: From Persistent Configs to Commit‑Based Generation'
+++

## Problem statement

Maintaining a static-site blog that reflects the latest research notebooks, memos, and project updates is a recurring pain point for our lab. Manual editing of `hugo.toml`, the homepage, and individual post frontmatter is error-prone and scales poorly. Two bugs kept showing up:

1. Future-dated posts: the LLM accidentally inserts the next day's date in the Markdown frontmatter, breaking the chronological order in the site's RSS feed.
2. Out-of-sync menus: after publishing a new post, the `hugo.toml` menu stays stale until someone manually adds a "Latest" entry.

The goal was to automate these steps while preserving the flexibility to tweak individual posts by hand.

## Background

*Project:* `frank-blog-content-generator`
*LLM:* Claude 4.6 (Anthropic)
*Metrics:*
- Baseline: 0 new blog posts, 0 menu updates
- Variant: 2 new blog posts, 1 menu entry added, homepage regenerated

The research notebooks and memos live in a separate repo (`frank-blog-content`). Every commit can potentially add a new post or memo that should surface on the Hugo site. The challenge was making the pipeline incremental and deterministic.

## The workflow

1. Detect new changes by scanning the git history of the source repo.
2. Generate Markdown by sending the diff and README context to Claude 4.6.
3. Inject the system date, overriding the LLM's date placeholder with the current ISO 8601 timestamp.
4. Persist configuration by loading `.frank.toml` for project-wide defaults.
5. Update Hugo files: add a "Latest" menu entry to `hugo.toml` and re-render the homepage Markdown sections.
6. Commit state by writing the last processed commit hash to a SQLite DB.

Each step is idempotent and driven by small, declarative files.

## Key challenges

| Challenge | Impact |
|-----------|--------|
| Date injection | LLM hallucinations produce future dates, breaking feeds. |
| Configuration drift | Paths and flags must be re-typed on every run. |
| Homepage hardcoding | Section names in the prompt limit scalability. |
| Redundant processing | Re-generating every post on each run wastes compute. |
| Prompt bloat | Large diffs and README excerpts can exceed token limits. |

The following sections describe how I addressed each of these.

## Solutions

### 1. System date injection for frontmatter

Instead of relying on the LLM to guess the date, we inject the current date directly into the Markdown template. The template contains a placeholder `{{CURRENT_DATE}}`, which the CLI replaces before the LLM processes the prompt.

```toml
# .frank.toml (excerpt)
[post]
template = """
---
title: "{{TITLE}}"
date: "{{CURRENT_DATE}}"
tags: [{{TAGS}}]
---
{{CONTENT}}
"""
```

During generation:

```bash
frank generate post --commit 7f6e2c3
```

The CLI does:

```go
now := time.Now().UTC().Format(time.RFC3339)
rendered := strings.ReplaceAll(template, "{{CURRENT_DATE}}", now)
```

All posts now get stamped with the actual creation time, eliminating future-dated entries.

### 2. Persistent configuration via `.frank.toml`

The `.frank.toml` file lives in the Hugo site root and is ignored by Git. It stores project-wide defaults:

```toml
# .frank.toml
[paths]
repo = "../frank-blog-content"
site = "./hugo-site"

[flags]
period = "week"
```

The CLI automatically merges these defaults with any command-line flags, so you only need to add the file once.

```bash
# First run
frank init .frank.toml
# Subsequent runs
frank generate post
```

### 3. Dynamic homepage generation

Previous iterations hard-coded section names such as "Research" or "Projects" into the prompt. The new approach analyzes the current homepage Markdown, extracts all headings, and lets the LLM decide how to update each section.

```go
sections := parseSections(homepageMarkdown)
// Example output: ["Research", "Projects", "Memos"]
prompt := fmt.Sprintf(
    "Update the following sections based on new posts:\n%s\n%s",
    strings.Join(sections, "\n"),
    diffText,
)
```

The LLM then returns a fully rendered homepage. The CLI writes it back to `content/_index.md`.

### 4. Commit-based blog post generation

A SQLite state file (`frank_state.db`) stores the last processed commit per repository. On each run, the CLI queries commits newer than that hash:

```go
rows, _ := db.Query(`SELECT commit_hash, diff FROM commits
                    WHERE repo = ? AND commit_hash > ?`,
                    repo, lastProcessed)
```

Only new notebooks and memos are sent to the LLM, which cuts runtime significantly. If the state file is missing, the tool falls back to processing all files.

### 5. Enriching LLM prompts with README and code diffs

The prompt sent to Claude 4.6 includes:

1. Commit diffs, limited to the most recent 15 k characters.
2. README excerpt, the first 1 k characters of the repository README.

```text
Context:
- README: "Welcome to the Frankenstein AI Lab..."
- Diff: "+ def new_feature():\n+    pass\n"

Task: Summarize the new feature and produce a blog post.
```

This additional context improves the accuracy of the generated summaries and reduces hallucinations.

## Implementation details

### CLI commands

```bash
# Generate new posts from the latest commit
frank generate post

# Update the Hugo menu with the latest post
frank update menu

# Regenerate the homepage
frank update home
```

### Example of `frank update menu`

The CLI reads `hugo.toml`, finds the `[menu]` section, and inserts a new entry:

```toml
[menu]
  [[menu.main]]
  name = "Latest"
  url  = "/posts/2026-02-13-new-llm-features/"
  weight = 1
```

The tool ensures that the `weight` is incremented automatically to keep the menu order correct.

### Handling state DB conflicts

During concurrent runs, the SQLite file can be locked. The CLI retries with exponential back-off:

```go
for i := 0; i < 5; i++ {
    err := db.Update(...)
    if err == nil { break }
    time.Sleep(time.Duration(1<<i) * time.Second)
}
```

If the lock persists, the user is prompted to resolve the conflict manually.

## Metrics and results

| Metric | Baseline | Variant |
|--------|----------|---------|
| New blog posts generated | 0 | 2 |
| Menu entries updated | 0 | 1 |
| Homepage regenerated | 0 | 1 |
| Generation time | ~4 min | ~1 min |
| LLM hallucinations (future dates) | 1 per run | 0 |

The stateful, commit-based approach cut runtime by 75% and eliminated the date bug entirely. Dynamic homepage logic reduced manual edits by about 90%.

## What I learned

Date injection is non-negotiable. Even a single future date breaks the RSS feed's chronological order. Persistent config via `.frank.toml` removes repetitive flag passing and saves real time on each run.

Dynamic section detection scales better than hard-coded prompts. Commit tracking is essential for large repos; without it the LLM spends hours re-generating content it has already produced. And prompt size limits must be respected: chunking diffs or trimming README excerpts prevents token overflow.

## Next steps

- End-to-end tests for homepage regeneration and menu updates.
- Parallel processing of multiple branches to support multi-project workflows.
- Rollback mechanism for accidental menu overwrites.
- Integration with CI/CD to trigger `frank` automatically on push.

## Conclusion

Automating the Hugo content pipeline with Claude 4.6, persistent configuration, and commit-based generation replaced a manual workflow with something repeatable. By injecting system dates, dynamically updating menus and homepages, and enriching prompts with context, I can focus on writing notebooks instead of tweaking Markdown. The metrics confirm the improvement: faster runs, no date bugs, and far fewer manual edits.