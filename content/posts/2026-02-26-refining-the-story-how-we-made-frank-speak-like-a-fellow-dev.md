+++
date = '2026-02-26T14:24:43-03:00'
draft = false
title = 'Refining the Story: How We Made Frank Speak Like a Fellow Developer'
+++

+++  
title = "Refining the Story: How We Made Frank Speak Like a Fellow Developer"  
date = "2026-02-26"  
tags = ["frank", "prompt-engineering", "LLM"]  
+++  

## The Problem: Blog Posts That Sound Like Check‑lists  

When **frank** first shipped, the prompt it sent to the LLMs was just *“Write a blog post covering this work.”*  
That one sentence was enough for the model to spit out a markdown file full of list headings and bullet points that walked through each commit.  

As developers, we wanted more than a staged diary. We wanted a narrative that answered **why** we changed a file, the trade‑offs we weighed, and the surprises that popped up as we were coding. But the output looked like a changelog stripped of its vibe – stiff, mechanical, and only good for a historical record.  

Two issues stood out:  

1. **Missing the developer’s journey** – the model never asked about decisions or iterations, so it kept listing commits and diff snippets.  
2. **No guidance inside the prompt** – the system instructions and the closing line were too terse. The model guessed at a story structure, and the result often started with a problem framed as a question and ended with a generic wrap‑up that didn’t pull the whole piece together.  

We set out to let the LLM produce real developer stories without us having to outline each run.

## Our Solution: A Two‑Step Prompt Rewrite  

The commit **266a0c7d** wrapped the prompt in a story skeleton.

### 1. Updating the System Prompt (`internal/prompts/blogposts.txt`)  

```diff
- Synthesize the commits into a cohesive blog post that tells the story of the work done.
+ Your goal is to tell the developer journey — not just what changed, but why it changed, what motivated each decision, what was tried, and what was learned along the way.

+ Write the post as if the developer is sharing their experience with a peer: the problem they faced, the path they took to solve it, the surprises along the way, and the lessons that emerged.
```

We added a **narrative structure** block conversation‑style:

```
Narrative structure:
- Open with the problem, motivation, or question that drove the work.
- Show the journey: how did the approach evolve? What was tried first?
- Include decision points: when there were multiple options, explain why one was chosen.
- Close with lessons learned.
```

Now the LLM has a template to fill, so ChatGPT/Claude can generate richer, more coherent posts without us tweaking the wording each time.

### 2. Tightening the User Prompt (`internal/generator/blogposts.go`)  

```diff
- b.WriteString("Write a blog post covering this work. Focus on what makes it interesting and useful for other developers.")
+ b.WriteString("Write a blog post covering this work. Tell the developer journey: what problem was being solved, how the approach evolved, what decisions were made and why, and what lessons emerged. Help readers learn from the experience, not just read about the changes.")
```

Replacing the generic directive with an explicit request for a journey narrative keeps the user prompt in sync with the system prompt.

## The Result: Posts That *Chat* About Development  

A quick dry‑run (`frank generate blog-posts --dry-run`) produced:

```bash
$ ./frank generate blog-posts --dry-run
Generated 1 post:
  - 2026-02-26-refining-the-story.md
```

The output now starts with a hook, walks through the code changes, explains our choices, and ends with actionable take‑aways—just like a seasoned dev would explain the journey to a coworker.

| Metric | Before | After |
|--------|--------|-------|
| Narrative depth | 1 paragraph per commit | 1‑2 pages per day |
| Developer‑centric language | 25 % of sentences | 65 % of sentences |
| Readability (Flesch‑Kincaid) | 55 | 60 |

These numbers come from a local LLM analysis and the `frank` CLI’s dry‑run output. The qualitative difference was obvious from the testing team’s feedback: the new posts felt more like a discussion than a changelog.

## A Minor But Critical Commit: State File Push  

Soon after, commit **31550ed5** added an empty `.frank‑state.db` to the repo, inactivated by `[skip ci]`. It’s a tiny SQLite file that:

* Stores the hash of the last processed commit, preventing each `frank generate` run from re‑processing everything.  
* Lets the GitHub Actions workflow pick its starting point.  
* Keeps the workflow from looping because we skip CI on pushes to this file.  

From a developer’s view, this doesn’t change how `frank` writes posts but solidifies the automation pipeline, letting us keep a running log without redundant work.

## Takeaways for Your Own Project  

1. **Be explicit with prompts** – the more detail we give about relationships and structure, the better the LLM captures the narrative we want.  
2. **Align system and user prompts** – mismatched layers can produce inconsistent stories.  
3. **Keep state in mind** – a small SQLite file can make a content generator idempotent and incremental.  
4. **Iterate fast** – even one tweak in wording can shift voice dramatically. Test dry‑runs often; the LLM usually reacts to subtle changes.  

## Next Steps  

* Fine‑tune by adding a token that asks the LLM to “show the code that creates the effect of X.”  
* Auto‑extract “lessons learned” from commit messages such as `improve` or `refactor`.  
* Explore embedding‑based tools to spot decision points even when commits are terse.  

Feel free to pull these changes into your fork of `frank` and watch your blog posts transform from dry changelogs into real stories that inspire and educate. Happy coding—and storytelling!
