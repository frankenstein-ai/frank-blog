+++
date = '2026-02-18T11:19:38-03:00'
draft = false
title = 'Shipping an admin UI for Find Workers: Vite + Tailwind + native TypeScript (tsgo)'
+++

Problem — how do we add a modern admin interface to a Python-first backend while keeping developer DX fast, type-safe, and production-ready?

Find Workers is a Python/FastAPI backend that handles Pix escrow, WhatsApp notifications, booking lifecycle, vetting, and more. We needed a small admin UI for operational tasks: worker verification, dispute resolution, payment troubleshooting, and support flows. The constraints were simple: iterate quickly, keep strong types, keep build times short, and avoid changing backend deployment or CI.

In sprint 2026-W08 we added an admin subproject using React + Vite + TypeScript, wired Tailwind v4 for UI, and switched the type build from tsc to tsgo (TypeScript native-preview) to speed local builds and CI iteration.

what we shipped (summary)
--------------------------
Three small commits (all by Cainã Costa on 2026-02-18) added the admin subproject and iterated quickly:

- Initial scaffold: admin/ with Vite, React, TypeScript files, dev scripts, and bun.lock. Files added include vite config, tsconfigs, src/App.tsx, CSS files, index.html, and .gitignore.
- Tailwind: Tailwind v4 and @tailwindcss/vite plugin added; base/components/utilities directives wired into index.css and App.css.
- Build switch: replaced "tsc -b" with "tsgo -b" in the build script and added @typescript/native-preview to package.json/bun.lock so the repo uses the native-preview tsgo binary.

Concrete package versions from the diffs:
- tailwindcss: ^4.1.18
- @tailwindcss/vite: ^4.1.18
- @typescript/native-preview: ^7.0.0-dev.20260218.1

There are about 18 new files under admin/, and bun.lock contains platform-specific entries for the native preview and Tailwind tooling.

why these choices
-----------------
- Vite + React + TypeScript: fast dev server, HMR, and a familiar component model. Keeping the admin as a separate frontend avoids touching backend CI or deployments.
- Tailwind v4: utility classes let us prototype layouts quickly. The new native helpers in Tailwind v4 work well with Vite via @tailwindcss/vite.
- tsgo (@typescript/native-preview): a native TypeScript distribution with a tsgo CLI. Using tsgo -b reduces the time spent on repeated type builds, which matters in a small admin project where build latency blocks iteration. We add it as an optional native dependency so different OSes pick the correct binary.

what changed in code
--------------------
1) package.json scripts — build step

Before:
```json
"scripts": {
  "dev": "vite",
  "build": "tsc -b && vite build",
  "lint": "eslint .",
  "preview": "vite preview"
}
```

After:
```json
"scripts": {
  "dev": "vite",
  "build": "tsgo -b && vite build",
  "lint": "eslint .",
  "preview": "vite preview"
}
```

The change is small but meaningful: type checking and project references use the tsgo binary from @typescript/native-preview, while Vite handles bundling.

2) Tailwind integration — deps and CSS

We added tailwindcss v4 and @tailwindcss/vite. index.css now includes the standard directives:

```css
/* admin/src/index.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* optional global resets or theme variables */
```

App.css and components use Tailwind utilities for layout and spacing. Example App.tsx skeleton created in the scaffold:

```tsx
/* admin/src/App.tsx */
import React from "react";

export default function App() {
  return (
    <div className="min-h-screen bg-slate-50 text-slate-900">
      <header className="p-4 bg-white shadow">
        <h1 className="text-xl font-semibold">Find Workers — Admin</h1>
      </header>
      <main className="p-6">
        <div className="grid grid-cols-3 gap-4">
          <div className="col-span-2 bg-white p-4 rounded shadow">Bookings</div>
          <aside className="bg-white p-4 rounded shadow">Actions</aside>
        </div>
      </main>
    </div>
  );
}
```

3) bun.lock changes

bun.lock was added and then updated with Tailwind and @typescript/native-preview entries. The lockfile includes optional native packages for platform-specific binaries and a bin mapping so tsgo is available. Example (sanitized):

```json
"@typescript/native-preview": ["@typescript/native-preview@7.0.0-dev.20260218.1", "", {
  "optionalDependencies": {
    "@typescript/native-preview-darwin-arm64": "7.0.0-dev.20260218.1",
    "...": "..."
  },
  "bin": { "tsgo": "bin/tsgo.js" }
}, "sha512-..."]
```

Tailwind v4 adds oxide/node helper entries the plugin depends on.

developer workflow (how to run the admin locally)
-------------------------------------------------
The frontend uses bun and the repo includes bun.lock, but the package.json works with other Node runtimes.

Typical flow:
- cd admin && bun install
- bun run dev
- bun run build   (runs tsgo -b && vite build)
- bun run preview

If you prefer not to use bun, install deps with npm or pnpm. The build script can be changed back to "tsc -b && vite build" if tsgo is not available on your CI.

integration with the Find Workers backend
-----------------------------------------
The admin is a separate frontend that talks to existing FastAPI endpoints in find-workers/src/find_workers/api/*. Initial pages will cover:

- viewing and searching bookings (bookings endpoint)
- viewing payment states (woovi/payments webhooks)
- reviewing worker verification artifacts
- issuing manual releases/refunds via payment APIs
- sending WhatsApp templates via the whatsapp client

Keeping the admin separate keeps the backend codebase Python-only and lets frontend CI run independently.

trade-offs, caveats, next steps
-------------------------------
- tsgo is experimental. The native-preview listed in bun.lock is a dev build. It speeds local type builds, but it is new. We keep it optional and verify CI compatibility. If a runner cannot use the native binary, fall back to "tsc -b && vite build".
- Tailwind v4 brings native helper artifacts. Confirm your CI picks the correct optional native packages for Linux runners.
- Security and privacy: admin will have access to PII. Before enabling staging/production we must:
  - add strong auth and role-based authorization,
  - wire audit logging to existing find-workers audit utilities,
  - restrict WhatsApp/payment actions by role.

Planned follow-ups:
- Connect pages to bookings, payments, and workers REST endpoints and add end-to-end tests.
- Add RBAC and tests that assert audit logs and webhook events.
- Add CI steps that either install tsgo or run a fallback tsc build.

conclusion
----------
This sprint added a small, usable admin scaffold to Find Workers: Vite + React + TypeScript, Tailwind v4, and an experimental switch from tsc to tsgo to speed iteration. The frontend remains decoupled from the Python backend and is ready for pages that manage bookings, payments, and worker verification.

To run locally:
- cd admin
- bun install
- bun run dev

If you want to be conservative, revert the build script to "tsc -b" until tsgo is validated in your CI. If you work on Find Workers, the admin/ folder is in the repo — start with a bookings list page that queries the backend and we'll add RBAC and actions in the next sprint.
