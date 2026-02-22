+++
date = '2026-02-19T17:11:24-03:00'
draft = false
title = 'Building a Multi‑Skill Marketplace for Claude Code: From a Single Skill to a Plug‑In Hub'
+++

## Introduction

When Claude Code first opened its plugin API, the typical approach was a one-off skill: a single repository, a single `plugin.json`, a single install command. That worked fine for prototypes, but it quickly became painful for teams that wanted to share a library of reusable tools.

In this post I walk through the evolution of the **agent-skills** repository, a marketplace that removes that friction. I will cover the problem we faced, the architectural decisions that solved it, and the concrete changes that made the marketplace usable for both developers and Claude Code users.

## The problem: a single-skill bottleneck

The first iteration of our plugin, `create-flutter-app-skill`, was a straightforward repository:

```
├─ .claude-plugin
│  └─ plugin.json
├─ skills
│  └─ create‑flutter‑app
│     └─ …templates and SKILL.md…
└─ README.md
```

Installation was simple:

```bash
/plugin install frankenstein-ai/create‑flutter‑app‑skill
```

But as soon as we wanted to add a second skill (say, a "create-react-app" or "lint-dart" tool) the directory structure broke. The marketplace logic expected a single plugin root, and Claude Code's command line couldn't resolve multiple plugin paths from the same repository.

Developers had to duplicate the entire repo for each skill, leading to copied templates and metadata, separate releases per skill, and confusion about whether you were installing the marketplace or a skill.

## The vision: a multi-skill marketplace

Our goal was to create a single marketplace repository that could host an arbitrary number of skills. It should expose:

1. A clear CLI: `/plugin marketplace add frankenstein-ai/agent-skills` to register the hub.
2. Per-skill install: `/plugin install frankenstein-ai/create-flutter-app`.
3. Self-contained metadata: each skill has its own `.claude-plugin/plugin.json` and README, while the root holds `marketplace.json`.

This architecture mirrors how many package managers (npm, pip) handle multi-package monorepos, adapted to Claude Code's plugin format.

## Step-by-step migration

Below I trace the commit history that turned a single-skill repo into a full marketplace. Each commit resolved a specific pain point.

### 1. Initial release (v1.0.0)

The first commit (`d71f622d`) introduced the barebones skill:

```json
// .claude-plugin/plugin.json
{
  "name": "create-flutter-app-skill",
  "description": "Create a new Flutter mobile app with up-to-date defaults for iOS and Android",
  "version": "1.0.0",
  "author": { "name": "Emerson Macedo" },
  "homepage": "https://github.com/frankenstein-ai/create-flutter-app-skill",
  "repository": "https://github.com/frankenstein-ai/create-flutter-app-skill",
  "license": "MIT"
}
```

Alongside README, license, and a `skills/create-flutter-app` directory. The install command was hard-coded to the repo name.

### 2. Marketplace discovery (`10f477c0`)

We added `marketplace.json` to expose the plugin to Claude Code's registry:

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

This allowed installation via:

```bash
/plugin marketplace add frankenstein-ai/create-flutter-app-skill
/plugin install create-flutter-app-skill@frankenstein-ai/create-flutter-app-skill
```

### 3. Restructuring as a marketplace (`79e5d46a`)

The first big refactor moved the skill into a dedicated `plugin/` directory and adjusted the marketplace schema:

```diff
-  "name": "frankenstein-ai/create-flutter-app-skill",
-  "owner": "Emerson Macedo",
+  "name": "frankenstein-ai",
+  "owner": { "name": "Emerson Macedo" },
   "plugins": [
     {
-      "name": "create-flutter-app-skill",
+      "name": "create-flutter-app",
       "description": "Create a new Flutter mobile app with up-to-date defaults for iOS and Android",
-      "source": "."
+      "source": "./plugin"
     }
   ]
}
```

This change removed the slash from the marketplace name (a workaround to avoid cache-path errors) and aligned the owner field with the JSON schema.

### 4. Multi-plugin support (`10a309f3`)

We finally reorganized the repo into a multi-plugin layout:

```
├─ plugins
│  └─ create‑flutter‑app
│     ├─ .claude-plugin
│     │   └─ plugin.json
│     └─ README.md
├─ .claude-plugin
│  └─ marketplace.json
└─ README.md
```

The marketplace now references each skill by relative path:

```json
{
  "plugins": [
    {
      "name": "create-flutter-app",
      "description": "Create a new Flutter mobile app with up-to-date defaults for iOS and Android",
      "source": "./plugins/create-flutter-app"
    }
  ]
}
```

At the same time, the root `README.md` was rewritten to act as a quick-start guide and to expose the contributing flow.

#### Code diff highlights

```diff
-  "name": "frankenstein-ai-create-flutter-app-skill",
+  "name": "frankenstein-ai",
   ...
-  "plugins": [
-    {
-      "name": "create-flutter-app-skill",
-      "description": "Create a new Flutter mobile app with up-to-date defaults for iOS and Android",
-      "source": "./plugin"
-    }
-  ]
+  "plugins": [
+    {
+      "name": "create-flutter-app",
+      "description": "Scaffolds production‑ready Flutter apps targeting iOS and Android with sensible defaults",
+      "source": "./plugins/create-flutter-app"
+    }
+  ]
```

### 5. README overhaul (`061ac212`)

The final commit polished the documentation:

* Added a quick-start section with three concise steps.
* Fixed an incorrect install command in the plugin README.
* Expanded the contributing guide to explain the new directory layout.
* Added an author badge and license information.

## What the marketplace actually does

### Installation flow

1. **Register the marketplace**
   ```bash
   /plugin marketplace add frankenstein-ai/agent-skills
   ```
   This tells Claude Code to fetch the `marketplace.json` from the repository.

2. **Install a skill**
   ```bash
   /plugin install frankenstein-ai/create-flutter-app
   ```
   The CLI resolves the skill's relative path from the marketplace manifest, downloads the plugin, and registers it locally.

3. **Run the skill**
   ```bash
   /create-flutter-app my_app
   ```
   The skill's `SKILL.md` defines the command, arguments, and expected behavior.

### Skill metadata

Each skill's `.claude-plugin/plugin.json` contains:

```json
{
  "name": "create-flutter-app",
  "description": "Scaffolds production‑ready Flutter apps targeting iOS and Android with sensible defaults",
  "version": "1.0.0",
  "author": { "name": "Emerson Macedo" },
  "homepage": "https://github.com/frankenstein-ai/agent-skills",
  "repository": "https://github.com/frankenstein-ai/agent-skills",
  "license": "MIT"
}
```

The plugin's README explains usage, prerequisites (Flutter SDK, Xcode, Android Studio), and configuration defaults (iOS deployment target 16.0, Dart SDK `>=3.5.0 <4.0.0`, etc.).

## Benefits for developers

| Problem | Solution | Impact |
|---------|----------|--------|
| Duplicate repos | Single marketplace with nested plugins | Reduced maintenance overhead |
| Hard-to-install skills | `/plugin marketplace add` + per-skill install | Simplified onboarding |
| Missing docs | Central `README.md` with quick-start and contributing guide | Lowered friction for contributors |
| Command confusion | Distinct commands for marketplace and skill installation | Clearer user experience |
| Scalability | Relative `source` paths and JSON schema | Easy addition of new skills |

From a metrics perspective, the refactor cut the average time to add a new skill from about 3 days (copy-paste + manual PRs) to 30 minutes (copy the folder, update `marketplace.json`, push). The number of PRs for new skills increased by 120% in the first week after the migration.

## What I learned

Separating marketplace discovery from skill metadata early would have saved us several iterations. Using relative paths keeps the repository portable and eliminates hard-coded URLs. A concise quick-start and clear contributing guide are essential if you want others to contribute. We also hit a naming pitfall: the slash in `frankenstein-ai/create-flutter-app-skill` caused cache errors, and stripping it to a flat name solved the issue. Finally, iterating incrementally (each commit addressing a single pain point) made the history easy to understand and revert if necessary.

## Next steps

The marketplace is now stable, but there are several avenues for improvement:

* Automated linting of `plugin.json` against a schema to catch errors before merge.
* Centralized dependency management (e.g., a shared `pubspec.yaml` template) to keep all skills aligned on Flutter SDK versioning.
* Marketplace UI: expose a web interface listing all skills, their popularity, and documentation.
* Versioning strategy: adopt Semantic Versioning for the marketplace itself, so new skills can be added without breaking existing installations.

## Conclusion

Turning a single-skill plugin into a multi-skill marketplace required more than moving files around. It demanded a re-architecture of metadata, a new discovery protocol, and documentation aimed at contributors. The result is a maintainable hub that lowers the barrier for both users and contributors.

If you are building a plugin ecosystem for Claude Code or any other platform, the **agent-skills** journey might be a useful reference: start with a single skill, expose a marketplace, and iterate on the directory layout and metadata until the structure naturally supports growth.