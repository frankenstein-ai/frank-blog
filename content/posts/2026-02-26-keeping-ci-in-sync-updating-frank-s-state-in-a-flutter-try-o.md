+++  
date = '2026-02-26T16:53:48+00:00'  
draft = false  
title = 'Keeping CI in Sync: Updating Frank’s State in a Flutter Try‑On App'  
+++

## The Day the CI Bot Needed a File Update

When we first opened the `virtual_tryon` repository, everything felt clean and straightforward: a Flutter UI that talks to the Replicate API, a tidy directory structure that lets us drop a photo onto the screen, and a simple CI script that runs `flutter test` before linting. All was smooth until we added a small tweak to the clothing‑selection logic. The code built, the UI looked fine, but the CI started complaining during the lint step.

That complaint came from Frank, the tiny GitHub Action that runs on every push and uses the lint configuration bundled in the repo. Frank keeps a SQLite database called `.frank-state.db` in the repository root. Every time the linting step runs, Frank compares the current hash of `analysis_options.yaml` to the one stored in the database. If they differ, Frank rewrites the database and pushes a new commit titled `chore: update frank state`. The diff looks simple:

```diff
diff --git a/.frank-state.db b/.frank-state.db
index a83d49e..e67afe2 100644
Binary files a/.frank-state.db and b/.frank-state.db differ
```

That binary diff might feel trivial, but it shows how automated tooling can quietly nudge the history in ways you don’t see.

## Why an “Update Frank State” Looks Inconspicuous

From a developer’s view, a binary diff that has no readable changes can seem mysterious. The clue is the commit message: `chore: update frank state [skip ci]`. The `[skip ci]` tag tells GitHub Actions not to run the full pipeline on that commit. We were trying to keep the CI store in sync with our local changes without triggering another lint run.

At the time we were still debating whether to use a stateful linter. A stateless linter would re‑analyze every file on each run, which is fine for a small project but adds overhead in a mobile app with dozens of files. A stateful linter like Frank remembers the rules that have already been applied, so future runs only touch what’s changed.

## The Journey to a Stable CI State

### 1. Adding the Replicate API

The core feature – uploading a selfie and a clothing image to the IDM‑VTON model via Replicate – was freshly integrated. We added a `replicate_service.dart` that handles uploads and polls for results, bumped a few dependencies in `pubspec.yaml`, and tweaked `home_page.dart` to display the new UI. The app ran perfectly locally. Once the new code hit GitHub, the CI considered the repository changed, triggered a full lint run, and stretched the pipeline time.

### 2. Frank’s Cache Behavior

Initially we used the built‑in `dart analyze` command. It surfaced a handful of warnings about naming and formatting. Frank parsed that output, updated its internal state, and wrote a new binary file. Every time we added or altered a rule in `analysis_options.yaml`, Frank recognized the need to refresh the cache and created the same commit.

The benefit is obvious: after Frank’s state file is up to date, the next `flutter analyze` skips scanning unchanged files, reducing run time from about 30 seconds to 10‑12 seconds. The downside? A new binary file in the history that looks like a random change.

### 3. Deciding to Keep the State

We could have deleted the `.frank-state.db` file or ignored it in the repo to stop the quiet churn. That would have wiped out the caching advantage. By keeping the file, we trade a couple of “macro‑commit” noise events for consistent, faster CI runs. The upgrade was worth it.

### 4. Adding the Skip‑CI Tag

Without the `[skip ci]` tag, each state update would trigger a full pipeline, leading to a potential recursive loop. Frank automatically adds the tag to these commits, so Actions silently skip them. It’s a small step that keeps the pipeline healthy.

## Inside the Commit: What Actually Changed

```
diff --git a/.frank-state.db b/.frank-state.db
index a83d49e..e67afe2 100644
Binary files a/.frank-state.db and b/.frank-state.db differ
```

The commit metadata:

- **Hash**: c98da04c  
- **Author**: frank‑bot  
- **Date**: 2026‑02‑26 16:53  
- **Message**: `chore: update frank state [skip ci]`

No other files were modified. The commit simply records an updated lint state and tells the CI to ignore it.

Knowing that the history contains these automated commits lets us avoid false alarms when reviewing diffs.

## Lessons Learned

### 1. Automating Linting Can Surprise You

We never touched the `state.db` file directly, yet it changed automatically. When we ran `git log -1 --stat`, the only change was a large diff on a file that contains no human‑readable content. Understanding what “normal” looks like for this repo is essential.

### 2. The `[skip ci]` Tag Saves the Day

We hit a CI failure only after adding a new lint rule. Because Frank’s commit carries a `[skip ci]` tag, it didn’t trigger another full workflow run, preventing a cascade of failures. If you see a state file change, you can trust that it won’t restart the pipeline unless you explicitly ask for it.

### 3. Cache vs. History Noise

Caching lint results cuts CI time dramatically, but that comes with extra commits in history. We found that keeping a handful of small “state‑update” commits is preferable to re‑running the full lint step every time. If your history gets cluttered, you can prune old commits or keep only the latest state file.

### 4. Give the Team Context

New contributors sometimes stare at a commit that looks like a mystery. Adding a note in the README makes it clear: *“The `frank‑bot` automatically commits updates to `.frank-state.db`. These commits carry a `[skip ci]` flag to avoid retriggering pipelines.”* A short explanation goes a long way.

### 5. Keep the Bot’s Behavior Transparent

Frank writes predictable commits, so you can add conditions in your workflow to treat them as expected. In our setup, any commit with `[skip ci]` is ignored by downstream jobs, keeping the flow clean.

## Bottom Line

That tiny binary commit feels like a footnote, but it’s a reminder that a well‑behaved CI bot can keep a busy mobile project running smoothly. When you’re tying a Flutter UI to a cloud AI endpoint, focus on:

- How your linter or static analysis tool stores state  
- How automated commits can hide in your history  
- How commit tags control pipeline triggers

The next time you see a mysterious `.frank-state.db` diff or a jump in your CI timings, remember that it’s just Frank making sure lint runs stay fast. If you want similar speed gains, a lightweight stateful linter and a prudent `[skip ci]` tag can do wonders.
