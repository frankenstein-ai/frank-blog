+++
date = '2026-02-26T16:53:48+00:00'
draft = false
title = 'Keeping a Shell‑Armed Sprint Alive: Why a Tiny Binary Commit Matters'
+++

## When a “Chore” is really a lifeline

At first glance the recent commit to the Smart Wardrobe repo might look trivial—just a binary blob named `.frank-state.db` that was swapped, marked with a plain commit message:

```text
chore: update frank state [skip ci]
```

The line says exactly what it says: we’ve bumped Frank’s state and told CI to skip the heavy tests for this file. If you’re a Flutter hobbyist focused on the UI, “Frank state” might feel as opaque as a backend’s log file. In reality, it’s the file that keeps our single‑device workflow honest and reproducible.

Below is the story that led to the commit—how we pieced it together in a few hours, the choices we weighed, and what we learned about working with stateful tooling in a zero‑backend codebase.

---

## The problem: “Love it, but it moves”

Earlier releases relied on a small in‑house bot—Frank—that handled tasks like:

* syncing assets between teammates
* recording which images live on the device
* snapshotting local `SharedPreferences` used by providers
* firing quick UI tests that bite the disk

Frank lives in a tiny SQLite database (`*.db`). This fits our repository model—everything’s local and everything can work offline—but it also introduced a few fragile points:

1. **Non‑human edit** – the DB is binary, so a schema tweak or entry patch shows up as a large blob in history; you don’t see what changed or why.
2. **Hard reset** – the app writes directly to the DB through the dev‑time `ServiceImageStorage`. If a teammate loses state (say, after an OS upgrade that clears files), they must untangle the file‑system until the DB is refreshed from scratch.
3. **CI spamming** – any change in metadata triggers all of the nightly tests on a CI runner. Those tests are precise but expensive, adding noise to the pipeline.

The spark was a Wednesday afternoon: a team member dropped a new clothing item into the mock outfit view, and the subsequent `flutter test` cycle ballooned to 40 minutes. Someone else flagged the slowdown, recalling that Frank had just autoupdated its status for the new item.

---

## First try: a manual reset

Our first instinct was to delete the DB whenever something broke, then regenerate it with:

```sh
flutter pub run bin/generate_state.dart
```

The script dumps local `SharedPreferences` and creates a fresh database. It works, but the trade‑offs were obvious:

* We hid the fact that the schema had drifted.
* A `git diff` omitted the schema changes that caused the glitch, making future debugging harder.
* We lost the “history” of what images were staged or what metadata was added.

The manual approach also meant every developer had to run the generator after pulling new commits, so two people pulling at the same time could create divergent DB files that would conflict on the next sync.

---

## Second attempt: a migration routine

We considered a lightweight migration script inside the repo:

1. pull the last committed DB  
2. read its schema and data  
3. hammer an incremental hot‑patch for new columns while preserving existing data

We pushed a patch that added a `lastSynced` column to Frank’s schema; it worked locally, but the cost showed up quickly:

* **Build time** – opening, reading, and rewriting the whole DB on every pull added about 10 seconds to `git pull`.
* **Binary diffs** – `git diff` still hid the internal changes. We’d need a custom script and a schema inspector to see the differences, and nobody had that set up.
* **Legacy data** – the migration had to remain backwards‑compatible with older DB versions used by our test harness. That added complexity that didn’t translate into a user‑visible improvement.

We paused the experiment because a heavier migration scheme would only add friction for developers and eat the limited CI budget.

---

## Final decision: a commit‑first, skip‑ci strategy

After a day of discussion with senior architects, we settled on a minimal pattern:

| element        | policy |
|----------------|--------|
| State location | `.frank-state.db` is regenerated on every commit. |
| Commit policy  | The DB lives in a private branch and merges into `main` when a release view is ready. |
| CI policy      | It is marked as a “chore” and the job is flagged `[skip ci]`. |
| Developer flow | Any developer who pulls the branch runs `./scripts/refresh_state.sh` to rebuild the DB locally from the committed spec. |

The commit that shows up in the feed looks like a skipped pipeline, but it tells the team a few key things:

1. **State vivification** – Frank’s internal list of thumbnails changed after adding a new garment. A fast‑scan of the log signals the need to run the refresh script.
2. **Avoid CI overload** – our heavy `flutter test` suites run on the main branch, not on every small state change.
3. **Version control for binaries** – we can still see when the DB changed (the hash changes) and even run `git blame` on the human‑readable `GoldenState.txt` we keep in sync to aid debugging.

In practice the model has a few quirks:

* Binary diffs stay opaque; when something breaks we rely on custom tools. We already wrote `hashdiff`, which reads the `*.db` as bytes and prints an MD5 digest so we can spot accidental churn.
* The commit becomes a restore point. If a device wipes a user’s images, they can check out the branch and replace the files. `refresh_state.sh` automates that for them.

The `[skip ci]` tag is a small but powerful signal: no phantom tests or flaky pipelines from a tiny UI data change. It also keeps the PR process lean—merges are quicker and more transparent.

---

## Treating state as a first‑class citizen

Why did this commit feel significant? Because once you start committing binary data that the app depends on, you can’t treat it as an afterthought. The journey highlighted a handful of take‑aways:

1. **Explicit state resolution** – By documenting the DB path and keeping it in the repo, we avoid implicit file‑system assumptions. Everyone knows exactly where “Frank lives.”
2. **Signal through commit messages** – The `[skip ci]` marker turns a noisy integration process into a useful artifact. We’ll use the same idea for icon packs that change often but don’t need unit tests.
3. **Tool the team, not just the code** – `scripts/refresh_state.sh` gives developers a reproducible workflow that eliminates ad‑hoc `rm -rf` calls. New contributors see a clear, documented process.
4. **Don’t juggle fragile binaries blindly** – If a binary blob can be replaced with a serialisable text format (JSON, Protobuf), that is worth considering. For Frank, the quick look convenience and the ability to run SQLite queries locally outweighed the trade‑off.
5. **Balance CI cost vs. reliability** – The `[skip ci]` approach saves time but limits test coverage on every commit. We mitigated this by running a short feature‑branch pipeline that covers the most critical tests and a longer release‑branch pipeline that runs everything. That keeps the overhead in check without compromising safety.

---

## What could have gone differently?

1. **Auto‑merge migration “golden file”** – Auto‑generate a textual diff of the DB schema after each commit and store it as a patch. That would surface binary changes to anyone reading the history.
2. **Lock‑file for the database** – A simple `frank.lock` file recording the DB version and last–changed timestamp would let developers spot stale state more easily.
3. **Unit test for `refresh_state.sh`** – The script is now critical to the production workflow. A test that verifies the DB matches the golden snapshot would catch regressions early.

---

## Closing thoughts: when “skipped” is an ally

The `chore: update frank state [skip ci]` commit was a micro‑adjustment tucked under a binary change, but it demonstrates a broader lesson: **whenever the tooling you rely on lives inside your repo, treat it like any other piece of code—document it, version it, and make its role explicit.** Even a tiny file can become a maintenance headache if it slips into ignorance.

For developers building local‑first, offline‑capable apps that still need remote validation, ask yourself:

* Where will the “state machine” live?
* Who needs to read, edit, or regenerate it?
* Are the CI costs justified for the security that binary checks give you?

As the project grows, the strategies we trade in this commit will scale—sometimes revealing the hidden scaffolding that makes a lean, offline‑first app trustworthy. The next sprint might even swap Frank’s SQLite file for a JSON DOM or a flat‑buffer, but the core insight stays: **Treat state as an artifact and keep it honest in your repo.**
