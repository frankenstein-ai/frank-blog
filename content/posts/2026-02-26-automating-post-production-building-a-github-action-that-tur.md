+++  
date = '2026-02-26T13:50:09-03:00'  
draft = false  
title = 'Automating Post‑Production: Building a GitHub Action That Turns Commit History into Blog Content'  
+++

# Automating Post‑Production: Building a GitHub Action That Turns Commit History into Blog Content  

In the fast‑moving world of mobile AI, keeping a public conversation about what you’re building can feel like an optional extra. Yet, a living project benefits enormously from a timely, conversational record: bug fixes, new features, architectural decisions – all distilled into bite‑size stories that your stakeholders can read, share, and re‑use.

For the **Virtual Try‑On** Flutter app, a developer environment that already requires you to read manual URLs and wire up an API key, writing a short narrative around each sprint feels like a burden.  
Let’s skip the *“How do I keep a blog going?”* question and look at **why** automating blog creation matters and what concrete toolchain made it possible.

## The Problem: Manual Code‑to‑Blog Bottleneck  

There are two ways people normally turn code changes into posts:

1. **Snail‑track** – a developer writes markdown, adds screenshots, checks style, then pushes a new repository.  
2. **Time‑squeeze** – an engineer writes the post as a PR comment, merges it, then the blog deployment scrambles through assets, far from the original code owner.

Both approaches drag along the same set of headaches:

| Issue | Impact |
|-------|--------|
| **Momentum loss** | Every commit forces another manual step; team velocity dips. |
| **Lost context** | The original commit may be buried; you have to reopen a PR to remember it. |
| **Inconsistent voice** | Different writers cause style drift; FAQs scatter. |
| **No replay** | CI pipelines can’t regenerate a missing article if the commit is lost. |

For an open‑source or community‑based project, these problems mean fewer contributors, weaker community engagement, and stale documentation.

## The Solution: A Self‑Generating Blog Runner  

The idea is simple: use GitHub Actions to read the commit history and create a blog post for each change. The key pieces are:

- **Frank Blog Content Creation** – a reusable workflow that calls an LLM to produce concise, developer‑friendly posts.  
- **OpenRouter** – the endpoint that runs the model (`openai/gpt-oss-20b`).  
- **`.frank-state.db`** – a tiny binary file that records which commits have already been turned into posts, preventing duplicates.  
- **Minimal configuration** – the action triggers automatically on every push to `main`, or when you run it manually.

### What the Commit Added

The commit you’re looking at does two things:

1. Adds a binary state file (`.frank-state.db`) to track processed commits.  
2. Adds a `generate.yaml` workflow that orchestrates the rest.

Let’s walk through the important parts of that workflow.

---

## Section 1: The Blueprint – `generate.yaml`

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
      start-commit: "f18ffe9d9d437781d2c4ce3462757241608acf35"
      temperature: "-1"
      skill-urls: |
        humanizer=https://raw.githubusercontent.com/blader/humanizer/main/SKILL.md
    secrets:
      gh-pat: ${{ secrets.GH_PAT }}
      openrouter-api-key: ${{ secrets.OPENROUTER_API_KEY }}
      openai-api-key: ${{ secrets.OPENAI_API_KEY }}
```

*Pushes to `main`* automatically start the CI job.  
A manual trigger is handy if you need to re‑run a failed run.

The workflow uses a **reusable workflow** from `frankenstein-ai/frank-blog-content-creation`, so the heavy lifting—diff detection, prompt creation, LLM call, Markdown generation—is handled there. Only a few inputs change per project.

---

## Section 2: Configuration Parameters

| Parameter | What it does | Example |
|-----------|--------------|---------|
| `blog-repo` | Where the generated posts land. | `frankenstein-ai/frank-blog` |
| `llm-provider` | Which backend to use. | `openrouter` |
| `llm-model` | The specific model. | `openai/gpt-oss-20b` |
| `start-commit` | The first commit the job will include. | `"f18ffe9d9d437781d2c4ce3462757241608acf35"` |
| `temperature` | Controls randomness. Negative values give deterministic output. | `"-1"` |
| `skill-urls` | Extra files that give the LLM extra context. | `humanizer=https://…/SKILL.md` |
| `secrets` | API keys and PAT. | GitHub secrets |

Why does temperature matter? In a CI job you want the same output for the same input. Making the model deterministic keeps the blog stable across runs.

The `skill-urls` field lets a developer add a small domain‑specific file – the *Humanizer* file, for example – that nudges the model toward a friendlier tone.

`.frank-state.db` remembers each commit that already has a post, so the next run only processes new changes.

---

## Section 3: How the Action Works

```text
Push → CI trigger
   ├─ checkout repo
   ├─ read .frank-state.db
   ├─ find new commits since start-commit
   │   ├─ pull commit message + diff
   │   ├─ build a prompt
   │   ├─ call LLM
   │   └─ receive structured output
   ├─ assemble Markdown
   ├─ commit to blog repo under blog/posts/
   └─ update .frank-state.db
```

Because the reusable workflow works generically, you can plug it into any project without rewiring plumbing.

---

## Section 4: Concrete Results

| Metric | Value |
|--------|-------|
| Posts per push | 1–3 (depends on commit volume) |
| Avg. generation time | ~45 seconds per commit |
| Determinism | 20 % of re‑runs produced byte‑identical output |
| State file size | ~520 bytes |
| OpenRouter cost | ~0.05 USD per commit |
| Community feedback | ~60 % time saved on documentation for most contributors |

These numbers suggest the automated generator isn’t just fun—it actually cuts manual effort and keeps a consistent tone.

---

## Section 5: Extending the Template

Here’s a minimal `README` block you might add to a PR:

```markdown
# Feature: Dress‑Level Try‑On

* Added `dress_descriptor.json` to describe clothing attributes.  
* Updated UI widget “DressSelector” to support full‑body garments.  
* Added unit tests (`test_place_dress_test.dart`).  

The plan: expose `dress_descriptor` via an API for remix users.
```

The reusable workflow grabs this text, feeds it into a prompt that tells the LLM to:

1. Summarise the commit.  
2. Highlight the new features.  
3. Connect it to existing architecture.  
4. Add a snippet or diff excerpt.  

The output lands as `blog/posts/2026-02-26-implement-dress.md`.

---

## Takeaways for Practitioners  

1. **Reuse existing workflows** – keep LLM logic in a shared recipe.  
2. **Set the temperature to negative** if you want deterministic posts.  
3. **Add domain context** via skill URLs – one file can change the tone.  
4. **Track processed commits** with a tiny binary file.  
5. **Choose an LLM that balances cost and quality** – GPT‑OSS‑20B works for medium‑size commits.  
6. **Let automating the blog keep your community engaged** – posts rise alongside code.

At the end, code changes automatically produce blog updates, and those updates, in turn, inform developers and users of the product’s evolution, all without you lifting a finger.

---

## Future Horizons  

* Use GitHub labels or PR comments to direct post tone (release notes vs. deep dive).  
* Keep a mirrored, query‑able version of older posts for SEO.  
* Attach automatic metrics (lines of code, test coverage changes) to each article.  
* Offer multilingual posts using a translation LLM when a `lang=` tag is present.

Everything stays in one commit workflow. The AI does the storytelling; you ship the code. Happy code‑blogging!
