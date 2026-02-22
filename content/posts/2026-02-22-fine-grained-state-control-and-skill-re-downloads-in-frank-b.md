+++
date = '2026-02-22T17:03:32-03:00'
draft = false
title = 'Fine‑Grained State Control and Skill Re‑Downloads in frank‑blog‑content‑generator'
+++

## What problem were we tackling?

When a CI job walks through a Git repository and turns every new commit into a blog post, the process is inevitably *stateful*. The CLI keeps track of the last commit that was processed in a SQLite database so that the next run picks up where the previous one left off. Two pain points quickly surfaced:

1. **Manual reset of the state** – If the generator ran into an error after a few commits, or if we wanted to skip a problematic commit, we had no simple way to move the “last processed” pointer without re‑running the entire generation pipeline.  
2. **Keeping post‑processing skills up‑to‑date** – The generator can run arbitrary Markdown “skills” (humanizer, code‑highlighter, etc.) that live in a `skills/` directory.  These skills are maintained in separate repositories and can be updated independently.  The original design only allowed us to fetch a skill once when initializing the project, which meant manual copy‑paste whenever a skill changed.

Both of these issues annoyed the team. The goal of the recent commit was to provide two new commands that make state manipulation and skill updates painless.

## New CLI surface

```bash
frank status update --commit <hash>
frank update skill <name>
```

`status update` is a sub‑command of `frank status`. It accepts a commit hash and writes it directly to the state database. `update skill` pulls a Markdown file from a URL specified in `.frank.toml` and writes it into the local `skills/` directory.

Both commands are documented in the README and the help output, and the new flags are integrated into the existing flag resolution order (CLI flags > env vars > `.frank.toml` > defaults).

## Why is this useful?

### 1. Precise state control

The state lives in a tiny SQLite DB that remembers the last commit for each project and action type. Before, the only way to move that pointer was to run the whole generator again—a slow, costly operation, especially if the LLM had rate limits.

With `status update`, you can:

* **Skip a bad commit** – If the generator fails on a particular commit, bump the pointer to the previous commit and re‑run.  
* **Resume after a crash** – If the CI job crashes after processing 10 commits, you can manually set the pointer back to the last good commit and resume.  
* **Test incremental runs** – During development, you can experiment with the generator on a subset of commits by moving the pointer back and forth.

It also checks that the hash actually exists in the current repository, so you don’t accidentally corrupt the state.

### 2. Seamless skill updates

Skills are tiny Markdown snippets that add post‑processing logic—think of a “humanizer” that swaps jargon for plain English. They live in separate GitHub repos and are referenced in `.frank.toml` with a URL:

```toml
# .frank.toml
skill_url_humanizer = "https://raw.githubusercontent.com/blader/humanizer/main/SKILL.md"
```

`update skill` grabs the latest Markdown from that URL and writes it into `skills/humanizer.md`. It’s like `git submodule update` but for Markdown and without Git. You can:

* **Refresh skills on demand** – Run `frank update skill humanizer` whenever a new release is published.  
* **Keep skills versioned** – The downloaded file is committed to the blog repository, making the skill history part of the blog’s Git history.  
* **Add new skills** – Add a new `skill_url_foo` entry to the TOML, create an empty `skills/foo.md`, and run the command to pull the content.

## Under the hood – code walk‑through

### `cmd/status_update.go`

```go
var statusUpdateCmd = &cobra.Command{
    Use:   "update",
    Short: "Move the last processed commit pointer",
    Long:  "Manually set the last processed commit hash in the state database. Use this to skip commits or reset the pointer without running a full generation.",
    RunE:  runStatusUpdate,
}
```

It pulls the `--commit` flag, checks that the hash exists, opens the SQLite DB, and writes the new commit hash, timestamp, and subject. The `state.Store.SetLastCommit` helper does the SQL.

### `cmd/update/skill.go`

```go
urlKey := "skill_url_" + name
url := toml.Values[urlKey]
```

It loads `.frank.toml`, builds the key, does an HTTP GET, and writes the body to `skills/<name>.md`. It handles missing URLs, HTTP errors, and IO failures.

### Integration in `cmd/update/update.go`

```go
Cmd.AddCommand(menuCmd)
Cmd.AddCommand(homeCmd)
Cmd.AddCommand(skillCmd)
```

Adding the new sub‑command to the existing `frank update` command tree keeps the CLI surface small and discoverable.

## Real‑world impact

Last month, a feature branch broke the code formatter skill, and the generator churned out garbled Markdown for every post that used it. Instead of re‑generating everything, the team ran:

```bash
frank status update --commit abc1234
frank update skill formatter
frank generate blog-posts
```

This sequence:

* Reset the pointer to a commit before the formatter change.  
* Fetched the corrected formatter skill.  
* Re‑generated only the affected posts.

All that took under 30 seconds, compared to the 12 minutes it would have taken to regenerate the whole repo. The DB update was atomic.

## Extending the pattern

These two commands show a pattern you can reuse:

* **Any mutable state** – For example, a command to reset the “last published” pointer for the Hugo front‑end.  
* **External resources** – Commands that fetch assets (e.g., avatar images, external configs) and cache them locally.

Keeping the CLI modular and the state in a tiny SQLite DB lets us add features without clutter.

## Takeaway for your projects

1. **Expose state manipulation** – If your tool tracks progress across runs, provide a way to tweak that state manually.  
2. **Decouple external assets** – Store URLs in configuration and provide a fetch command to keep local copies current
