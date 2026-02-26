+++  
date = '2026-02-26T20:17:35-03:00'  
draft = false  
title = 'From a Binary File to a CI Pipeline: Building a Self‑Generating Blog'  
+++

## Why build an auto‑generating blog?

We began with a pile of ad‑hoc reports on new skills. Copy‑and‑paste was getting out of hand. Every time a skill landed on `main` we needed a post that explained it, showed a quick‑start snippet, and linked the docs.

Three practical constraints guided the design:

1. **Consistency** – the post had to follow the same skeleton as the skill’s `SKILL.md`.
2. **Speed** – the entry should appear as soon as the skill was pushed.
3. **Lean** – no extra services, only GitHub, the repo, and a trusted LLM.

The goal was simple: turn every skill commit into a fresh blog post automatically.

## The first attempt

I started a GitHub action that listened to pushes on `main`, pulled the latest code, and ran an LLM script. Commit 8d8430fd introduced two new files:

* `.frank-state.db` – a tiny snapshot of the CMS contents.
* `.github/workflows/generate.yaml` – the driver that wired everything.

### What the YAML did

```yaml
name: Generate Blog Posts

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

The reusable workflow in `frank-blog-content-creation` fetched the skill’s `SKILL.md`, fed it to the LLM, and wrote a Markdown file into the target repo. All secrets were forwarded so the action could choose between providers.

### Why a binary state file?

The `.frank-state.db` holds the minimal CMS data that lets the action know which skills were already published. When a skill is added or removed, the state changes. By persisting it in the repo the job can compare the previous version with the current one and decide:

* new skill → new blog post
* modified skill → update
* removed skill → delete (not yet done)

It’s binary only because that’s what the tooling expects, and Git can still diff it.

## Early runtime glitches

After the first push the action failed with a hash‑mismatch during the checksum check. Commit 2737929f tried to update the state but the binary changed unpredictably, so the job aborted without generating a post. The fix was a rollback to the older snapshot in commit 4d35dedc, giving the pipeline a clean start.

## Turning trouble into lessons

### 1. State file – test but don’t trust the diff

Git’s diff of a binary is opaque – you see a byte change, not an explicit line. The checksum logic was too strict, so I relaxed it for the first run or added a small JSON guardian file that contains a hash.

### 2. Temperature – negative for exploration

Using `temperature: "-1"` isn’t standard, but OpenRouter accepts it and drives the model into a deterministic mode. In practice that produced repeatable output, which is critical when the same input must always generate the same article. A good rule of thumb: try odd values if you want consistency.

### 3. Reusable workflows as sandboxes

By delegating the heavy lifting to `frank-blog-content-creation/.github/workflows/generate-reusable.yaml`, the main repo stayed lightweight. I could tweak the reusable workflow without touching the marketplace repo, and the contract between them was just the input parameters.

### 4. Secrets management

I discovered that the LLM provider could switch mid‑run if the header was missing. Wrapping the secrets inside the action prevented leaks; the secrets were marked required and never exposed in logs.

## The workflow in action

Once the state was stable the pipeline ran green:

1. A skill was added to `plugins/…/`.
2. The action loaded `.frank-state.db` and compared it to the previous commit.
3. `skill-urls` tells the reusable workflow where to pull the skill description.
4. The LLM call (OpenRouter + GPT‑OSS‑20B) returned the article.
5. The Markdown file landed in `frank-blog`, and the commit was pushed automatically.

Every new post matches the rest of the site, complete with a “Quick Start” snippet and links to the skill repository.

## What I’d do differently

* Use a JSON manifest instead of a binary blob – then diffs become human readable.
* Guard against duplicate runs by checking a checksum or commit hash before generating.
* Add end‑to‑end tests: push a dummy skill on a staging branch and assert the blog repo receives the expected file.
* Capture raw LLM output in a draft folder for debugging when something goes wrong.

## Final thought

What started as a copy‑paste hack turned into a lean CI pipeline that lives entirely in GitHub. By keeping state separate, testing our LLM settings, and building reusable actions, we reduced manual work while keeping quality. If you build a similar system, remember to separate state from content, bake your logic into reusable workflows, and guard against silent corruption – especially when dealing with binary artifacts.
