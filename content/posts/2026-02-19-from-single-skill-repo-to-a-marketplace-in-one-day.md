+++
date = '2026-02-19T17:11:24-03:00'
draft = false
title = 'From Single‑Skill Repo to a Marketplace in One Day'
+++

## the problem that brought us here

When we started the **agent‑skills** repository, we only needed a single Claude Code plugin to scaffold Flutter apps. The idea was straightforward: name the repo *create‑flutter‑app‑skill*, publish it, and let users install it with one command:

```
/plugin install frankenstein‑ai/create‑flutter‑app‑skill
```

Soon we realized that what we had was a skill that could fit into a bigger ecosystem. What if we wanted to add more skills without turning the repo into a monolith? What if other developers wanted to contribute their own custom skills? The short answer: we needed a marketplace that could reliably discover multiple skills.

Our commits from 2026‑02‑19 show that leap. We converted a single‑plugin repo into a lightweight marketplace, reorganised the directory structure, adjusted the marketplace manifest to match Claude’s discovery API, and rewrote the README so the quick‑start instructions read more like a cookbook.

Below I walk through the decisions, the pitfalls we hit, and the lessons that emerged.

## 1. adding `marketplace.json` (10f477c0)

**Goal:** Enable discovery via `/plugin marketplace add`.

The first change was creating a `marketplace.json` file at the repo root:

```json
{
  "name": "frankenstein-ai/create-flutter-app-skill",
  "owner": "Emerson Macedo",
  "plugins": [
    {
      "name": "create-flutter-app-skill",
      "description": "Create a new Flutter mobile app with up-to-date defaults for iOS and Android",
      "source": "."
    }
  ]
}
```

It served two purposes:

1. It gave the marketplace a distinct identity – `frankenstein-ai/create-flutter-app-skill`.
2. It pointed to the single plugin that lived in the repo.

At first this felt enough. But Claude’s marketplace API expects the `plugins` array to contain objects that reference a **local path** relative to the repository root.

## 2. first obstacle: wrong owner format & naming (79e5d46a)

When we ran the marketplace, Claude returned an error. The `owner` field had to be an object, not a plain string:

```diff
-  "owner": "Emerson Macedo",
+  "owner": {
+    "name": "Emerson Macedo"
+  },
```

Another issue was the slash in the marketplace name. That caused a cache‑path error on the server – it treated the name as a nested folder and looked for an unresolved cache location. We renamed it to a flat string:

```diff
-  "name": "frankenstein‑ai/create-flutter-app-skill",
+  "name": "frankenstein-ai-create-flutter-app-skill",
```

The commit `d429359f` captured that change.

### what we learned

- The manifest’s schema matters. A tiny mismatch breaks the entire discovery flow.
- Naming conventions can have hidden side effects. Removing the slash fixed a cache lookup bug.

## 3. making it a marketplace (10a309f3)

The repo grew beyond a single skill. We had to:

- Host multiple plugins in one repository.
- Give each plugin its own metadata, documentation, and source folder.
- Keep the `marketplace.json` pointing to each one correctly.

We renamed the nested `plugin/` directory to `plugins/create-flutter-app/`, and updated its line in `marketplace.json`:

```diff
-      "source": "."
+      "source": "./plugins/create-flutter-app"
```

We also dropped the “‑skill” suffix from the plugin name, because the repo name already signals that we’re in the plugins folder. The final `marketplace.json` looks like this:

```json
{
  "name": "frankenstein-ai",
  "owner": {
    "name": "Emerson Macedo"
  },
  "plugins": [
    {
      "name": "create-flutter-app",
      "description": "Create a new Flutter mobile app with up-to-date defaults for iOS and Android",
      "source": "./plugins/create-flutter-app"
    }
  ]
}
```

The repository root’s README also changed. It now reads:

```markdown
# agent-skills

A Claude Code plugin marketplace by Frankenstein AI – curated skills that extend Claude Code with new capabilities.
```

### what we tried

We could have kept the original, flat structure and just added more plugins to the `plugins` array. That would have made the filesystem tangled and hard to navigate. The nested folder approach gives each plugin its own `.claude-plugin` directory and clean boundaries.

### decision rationale

- **Modularity** – developers can add new plugins by creating a new folder under `plugins/` and wiring it in the manifest.
- **Clarity** – the repository’s name on Claude’s marketplace shows as `frankenstein-ai`, while the plugin namespace remains `create-flutter-app`.

## 4. fixing the README & documentation (061ac212)

After reorganising the layout, the root README was stale. Users needed a quick‑start flow that told them:

1. Add the marketplace.
2. Install any plugin.
3. Run the plugin.

We rewrote the README with explicit steps:

```markdown
## quick start

1. Add the marketplace

```
/plugin marketplace add frankenstein-ai/agent-skills
```

2. Install a plugin

```
/plugin install frankenstein-ai/create-flutter-app
```

3. Use it

```
/create-flutter-app my_app
```
```

We also updated the plugin’s own README in `plugins/create-flutter-app/README.md` to use the correct install command. Finally, we added a **contributing** section to the root README:

```markdown
## contributing

To add a new skill to this marketplace:

1. Create a directory under `plugins/<skill-name>/`
2. Add a `.claude-plugin/plugin.json` with your plugin metadata
3. Add a `skills/<skill-name>/SKILL.md` defining the skill behavior
4. Register the plugin in `.claude-plugin/marketplace.json`
5. Open a pull request
```

And an author line:

```
[Emerson Macedo](https://github.com/emerleite) — Frankenstein AI
```

### surprises

The markdown diff made me realise how many implicit assumptions users have: the install command, the structure of the plugin folder, the shape of `marketplace.json`. Making those explicit in documentation was the fastest way to make the repo usable by third parties.

## 5. decision points and trade‑offs

| decision | alternatives | why we picked it |
|----------|--------------|------------------|
| marketplace JSON name | `frankenstein-ai/create-flutter-app-skill` | the slash caused cache errors, so we removed it |
| owner field format | string vs. object | the API needed an object; the object lets us add future data |
| directory layout | flat plugin folder vs. nested `plugins/` | nested keeps adding more skills clean and well‑separated |
| plugin name | `create-flutter-app-skill` vs. `create-flutter-app` | consistency and avoids redundancy |
| README structure | detailed tutorial vs. minimal description | quick‑start plus contributing is clearer for users |

Each choice came from a small bug or user‑experience observation. The commits show incremental refinements.

## 6. lessons learned

### 1. stay schema‑driven

If a manifest is parsed by an external API, the tiniest mismatch will surface only at runtime. Validate against the latest spec or run unit tests that mimic the consuming service.

**Takeaway:** run the platform’s own validation tools before pushing.

### 2. naming matters for caching and routing

An accidental slash can break a cache layer that expects a flat key. Keep marketplace identifiers flat; if you need namespaces, use a delimiter that the platform guarantees safe, like a hyphen.

**Takeaway:** avoid path‑like names for identifiers.

### 3. separate repo from marketplace

The repository should hold all plugin code; the marketplace macro‑descriptor should only reference them. This decoupling means you can add new skills without touching the manifest each time if you adopt a convention like `plugins/*` and list them automatically (e.g., via a script).

**Takeaway:** keep discovery metadata minimal and declarative.

### 4. good documentation is a first‑class dependency

A README that matches the install commands and folder structure saves developers hours of confusion. Write it as a living contract between the repo and its users.

**Takeaway:** treat the README as your primary onboarding guide.

### 5. collaborating with an AI‑powered tool

The commits include co‑authorship by Claude Opus. An LLM can generate code snippets and suggest schema changes, but human oversight remains essential to catch subtle policy or platform constraints—just like we did with the slash in the marketplace name.

## 7. what we would do differently next time

* **Automate marketplace discovery** – instead of editing `marketplace.json` by hand, generate it from the folder structure. A small script that scans `plugins/` and populates the JSON would reduce human error.
* **Add unit tests** – assertions that the `marketplace.json` is valid and that each plugin contains required files.
* **Incremental preview** – run `claude plugin inspect` locally (if available) before a big push.
* **Formal linting for manifest files** – use a JSON schema linter to catch missing fields early.

## 8. takeaways

Converting a single‑plugin repository into a full‑featured marketplace was a day‑long sprint of iterations and lessons. By aligning with the marketplace’s expectations, reorganising the file layout, tightening documentation, and paying close attention to naming conventions, we built a developer‑friendly foundation that can host any number of future skills.

If you’re building or maintaining a plugin ecosystem, start with the **manifest** and **schema** you’ll expose. Anchor your folder structure on that, and let the README guide newcomers. The hard work is in the details—and mastering those details pays off in a smoother onboarding experience for everyone.
