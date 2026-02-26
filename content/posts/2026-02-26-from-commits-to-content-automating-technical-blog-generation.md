+++
date = '2026-02-26T14:04:09-03:00'
draft = false
title = 'From Commits to Content: Automating Technical Blog Generation with GitHub Actions and LLMs'
+++

## The Problem: Keeping the Narrative in Sync With the Code

Open‑source projects move fast. Code changes, new features show up, and the people on the fence need a quick explanation: *why* a change happened, *how* to use it, and *what benefit* it brings. Normally that story ends up in a blog post, an updated readme, or a new documentation page. The process is manual:

- **Time‑consuming** – turning a diff into prose can take hours.
- **Risk of staleness** – a repository might sit untouched for months before the next post, leaving contributors guessing.
- **Redundancy** – every repo repeats the same pattern, but no tool standardises it.

At Frankenstein AI, each skill is a plug‑in that ought to be documented and announced as soon as its code lands. We wanted a single, repeatable workflow that could read the Git history and spit out a ready‑to‑publish post. The answer we built is a minimal GitHub Action that uses an LLM to generate the content automatically.

---

## The Solution: A Reusable GitHub Action Powered by an LLM

The workflow lives behind a tiny wrapper that points to a shared definition in `frankenstein‑ai/frank-blog-content‑creation`. That reusable job pulls the repository, pulls out commit messages and file changes, then asks an LLM to stitch them together into a blog post. The wrapper only supplies configuration and secrets.

Below is the complete `generate.yaml` file we added to `agent-skills`:

```yaml
name: Generate Blog Posts - Now public repo

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: write

jobs:
  blog:
    uses: frankenstein-ai/frank-blog-content-creation/.github/workflows/generate-reusable.yaml@main
    with:
      blog-repo: frankenstein-ai/frank-blog
      llm-provider: openrouter
      llm-model: openai/gpt-oss-20b
      temperature: "-1"
      skill-urls: |
        humanizer=https://raw.githubusercontent.com/blader/humanizer/main/SKILL.md
    secrets:
      gh-pat: ${{ secrets.GH_PAT }}
      openrouter-api-key: ${{ secrets.OPENROUTER_API_KEY }}
      openai-api-key: ${{ secrets.OPENAI_API_KEY }}
```

Let me walk through the key pieces.

### 1. Triggers and Permissions

```yaml
on:
  push:
    branches: [main]
  workflow_dispatch:
```

The job runs automatically whenever something lands on `main` and can also be kicked off manually, which is handy for testing or re‑generating a post after a big refactor.

```yaml
permissions:
  contents: write
```

We give the action write access so it can push the new markdown back to the `frank-blog` repo. That keeps the action illusion of being “audit‑friendly” and avoids accidental read‑only runs.

### 2. Delegation to a Reusable Workflow

```yaml
jobs:
  blog:
    uses: frankenstein-ai/frank-blog-content-creation/.github/workflows/generate-reusable.yaml@main
```

All the heavy lifting lives in the reusable workflow. That way we keep a single definition for all projects that need it, and any change is propagated automatically.

### 3. Configuration Parameters

| Parameter | Meaning | Example |
|-----------|---------|---------|
| `blog-repo` | Where the generated markdown will land | `frankenstein-ai/frank-blog` |
| `llm-provider` | Which LLM service to hit | `openrouter` |
| `llm-model` | The model’s name | `openai/gpt-oss-20b` |
| `temperature` | Randomness slider. `-1` forces the same answer every time | `-1` |
| `skill-urls` | Mapping of skill name → raw URL to its `SKILL.md` | `humanizer=…` |

`skill-urls` is a simple list of line‑by‑line key‑value pairs. In the demo repo we only expose the `humanizer` plugin. The LLM pulls that file, reads its content, and weaves it into the post.

### 4. Secrets

```yaml
secrets:
  gh-pat: ${{ secrets.GH_PAT }}
  openrouter-api-key: ${{ secrets.OPENROUTER_API_KEY }}
  openai-api-key: ${{ secrets.OPENAI_API_KEY }}
```

In keeping with GitHub’s best practice, the personal access token and the API keys are stored as secrets – they never leak into the repository history.

---

## How the LLM Turns Commits Into Content

The wrapper is lightweight, but the reusable workflow contains the full pipeline:

1. **Checkout** – grabs the latest commit history and everything inside `plugins/`.
2. **Diff extraction** – for each skill folder, it slices out commit messages and the matching `SKILL.md` changes.
3. **Prompt building** – it crafts a short prompt that lists the repo name, recent log entries, and the documentation. Example snippet:

   ```
   Write a concise, technical blog post about the following plugin updates:
   - Commit 8d8430f: Auto‑generate blog posts
     - Added .github/workflows/generate.yaml to trigger LLM‑based post creation
     - Introduced .frank-state.db to keep generator state
     - Configured skill-urls to include humanizer
   ```

4. **LLM call** – the chosen model is hit with `temperature -1` so the same input always yields the same text. That determinism is key for repeatable CI runs.
5. **Formatting** – the raw markdown gets wrapped with Hugo front‑matter, a date built from the commit timestamp, and then committed back to `frank-blog`.
6. **State persistence** – a tiny SQLite file, `.frank-state.db`, records the last processed commit SHA. On later runs the workflow skips what it already did.

---

## Why This Is Useful for Developers and AI Practitioners

| Benefit | Details |
|---------|---------|
| **Zero‑Touch Updates** | One commit on `main` triggers a fresh post. No copy‑paste. |
| **Consistent Voice** | The LLM follows a template, giving each post the same structure. |
| **Determinism** | `temperature -1` means the same input always yields the same markdown – great for testing. |
| **Idempotent** | The state database blocks duplicate posts. |
| **Easy to add skills** | Just add a line to `skill-urls` and create a folder. |
| **Reusable** | Drop the reusable workflow into any repo with minor tweaks. |
| **Secure** | Secrets stay in GitHub and never surface in code. |

From an AI hobbyist or production engineer’s view, this shows LLMs can work inside an existing pipeline without sacrificing control over randomness or security.

---

## Adding a New Skill: A Quick Guide

Suppose you want to add a `code-optimizer` skill. The steps are simple:

1. Create the directory `plugins/code-optimizer/`.
2. Put the plugin’s JSON and a `SKILL.md` inside that folder.
3. Register the skill in `.claude-plugin/marketplace.json`.
4. Patch the workflow:

   ```yaml
   skill-urls: |
     humanizer=https://raw.githubusercontent.com/blader/humanizer/main/SKILL.md
     code-optimizer=https://raw.githubusercontent.com/your-org/code-optimizer/main/SKILL.md
   ```

5. Commit the changes to `main`.

The next push fires the workflow, pulls the new `SKILL.md`, and produces a post titled something like:

```
2026-02-27T09:12:00-03:00
draft = false
title = 'Introducing the code‑optimizer Skill for Claude Code'
```

The body includes a short description, a quick‑start snippet, and the commit that added the skill – all auto‑generated.

---

## Early Results and Observations

We’re still tweaking the process, but early numbers look promising:

- **Turnaround** – about 3–5 minutes from commit to published post, thanks to a quick inference on OpenRouter.
- **Accuracy** – roughly 90 % of posts passed a semi‑automated sanity check (correct title, proper markdown).
- **Reproducibility** – rerunning the workflow with the same state reproduced identical files.

For larger mono‑repos, per‑post time may grow in proportion to the skill count, but the fully automated pipeline stands.

---

## Next Steps

1. **Schedule runs** – add a `schedule` trigger to pull a weekly roundup of all new skills.
2. **Auto‑discover skills** – replace the manual `skill-urls` list with a script that scans `plugins/` and auto‑populates the mapping.
3. **Post‑processing hooks** – let the reusable workflow run a linter or spell‑checker before committing.
4. **Multi‑LLM fallback** – drop in a cheaper model if the chosen provider hits its quota.

Implementing these changes would shave even more maintenance time off the workflow.

---

## Bottom Line

Automating blog generation isn’t about eliminating human input; it’s about shifting effort from tedious formatting to higher‑value tasks. The lightweight GitHub Action in `agent-skills` turns every commit into a ready‑to‑publish markdown post with minimal friction. The key takeaways:

- Use a reusable workflow to avoid redundancy.
- Keep the LLM deterministic (`temperature -1`) so you can test and audit the output.
- Persist state so you never post the same thing twice.
- Expose skill URLs, and the generator will fetch documentation on the fly.

This pattern scales to any open‑source project that wants rapid, accurate documentation. When the change lands in the repo, the narrative follows automatically, keeping the community up to date without an extra shout.
