+++
date = '2026-02-24T11:29:08-03:00'
draft = false
title = 'Serving the Admin SPA: iterating on build, deployment and dev ergonomics for Find Workers'
+++

When you ship a backend API and a single-page admin app from the same repository, practical questions crop up fast: where do you build the frontend, how do you deploy it, and how do you keep local development quick and predictable? Over one sprint we tried three ways to serve the admin SPA in find-workers: share a build via a Docker volume, bake the admin into the Caddy image, and — for developer convenience — let the FastAPI app serve the built SPA at /admin. Along the way we made local development less brittle and added Playwright browser binaries to the dev environment so E2E tests run reliably.

Below I walk through the problem we were solving, the incremental changes (with key code excerpts), tradeoffs we discovered, and practical guidance for teams facing the same issue.

## The problem

Find Workers uses a common split: a Python FastAPI backend and a JS single-page admin UI. In production the API sits behind Caddy. For development and CI we wanted to:

- Build the admin SPA once and make that build available both to the API (for local previews) and to the reverse proxy in production.
- Avoid fragile shared-volume setups that produce surprising "works on my machine" behavior.
- Keep feedback loops fast so developers don't wait on unnecessary rebuilds.
- Produce reproducible, auditable artifacts for production images.

The repo initially used a Docker named volume that the API image populated on first create. That worked, but it relied on Docker's auto-population semantics and caused confusion for new contributors and CI. We explored two alternatives: bake the admin into the Caddy image, or let the FastAPI app serve the admin build when present. We implemented and tested all three patterns and settled on a workflow that keeps production artifacts deterministic while making local development convenient.

## What changed (overview)

Work touched these files and areas:

- Dockerfile: added or removed an admin-builder stage for building the SPA (we experimented with two approaches).
- caddy/Dockerfile: new file to bake the admin build into the Caddy image.
- docker-compose.yml: introduced and adjusted a named volume admin_dist to share the build between services.
- src/find_workers/main.py: FastAPI now optionally mounts the built admin at /admin using Starlette StaticFiles when /srv/admin exists.
- flake.nix: added playwright-driver.browsers to the development environment so E2E tests can run locally and in CI.
- CI/workflow metadata: pinned a start-commit for blog-generation tooling and updated frank-state bookkeeping.

Six commits covered these changes; below are the key diffs and why we made them.

## Iteration 1 — named volume (quick, but brittle)

First we used a named Docker volume so the API image could populate /srv/admin and Caddy would mount the same volume read-only. Docker copies files from an image into an empty named volume on create, so no extra build step was required for Caddy.

docker-compose snippet:

```yaml
services:
  api:
    # ...
    volumes:
      - admin_dist:/srv/admin
  caddy:
    image: caddy:2-alpine
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
      - admin_dist:/srv/admin:ro
volumes:
  redis_data:
  caddy_data:
  caddy_config:
  admin_dist:
```

Why this was tempting
- Easy to wire up locally: build the API image, create the compose stack, and Docker auto-populates admin_dist from the API image contents.
- No separate Caddy build step.

Why it failed us
- Subtle semantics: contributors may not realize the admin files come from the API image, and updates require rebuilding the image and recreating the compose services.
- CI prefers explicit artifacts rather than implicit population.

Because this approach leaked coupling and produced surprise behavior, we moved on to baking the admin into Caddy.

## Iteration 2 — bake admin into the Caddy image (self-contained reverse proxy)

We removed the shared-volume coupling by building the admin inside a Caddy image. The Caddy image now contains both the reverse-proxy config and the compiled static files.

caddy/Dockerfile:

```dockerfile
FROM node:22-slim AS admin-builder
WORKDIR /admin
COPY admin/package.json admin/bun.lock ./
RUN npm install --frozen-lockfile
COPY admin/ .
RUN npm run build

FROM caddy:2-alpine
COPY Caddyfile /etc/caddy/Caddyfile
COPY --from=admin-builder /admin/dist /srv/admin
```

What this gains
- Clear responsibility: the Caddy image is the canonical place for static assets in production.
- Deterministic production images: the artifact you push contains everything needed.
- No runtime volume surprises.

What it costs
- Slightly more build work: CI or your release pipeline must build and publish the Caddy image.
- If you still want local preview from the API, you may duplicate admin build steps between images.

We updated docker-compose.yml to build the caddy image from caddy/Dockerfile. That fixed production determinism.

## Iteration 3 — serve admin from FastAPI when present (developer convenience)

Baking the admin into Caddy is the right production model, but developers wanted a fast local preview without running Caddy. We added a small, opt-in behavior to the FastAPI app: if /srv/admin exists in the container, mount it with Starlette StaticFiles at /admin.

main.py snippet:

```python
import pathlib
_admin_dir = pathlib.Path("/srv/admin")
if _admin_dir.is_dir():
    from starlette.staticfiles import StaticFiles
    app.mount("/admin", StaticFiles(directory=_admin_dir, html=True), name="admin")
```

Why this helps
- Fast local loop: build the admin into /srv/admin or use an API image that includes /srv/admin, then open http://localhost:8000/admin without running Caddy.
- The route only appears when the directory exists, so production images that lack /srv/admin don't expose the admin UI accidentally.

Things to watch
- This can create an environment mismatch: what you see from the API in development might differ from what Caddy serves in production. Treat the API-mounted admin as a developer convenience, not the canonical production configuration.
- The directory check prevents accidental exposure in CI and production images that don't include the admin build.

## Developer environment and testing: Playwright browsers

End-to-end tests need browser binaries. To avoid OS-level setup steps for contributors and CI, we added Playwright browsers to the flake.nix dev shell:

flake.nix snippet:

```nix
pkgs = [
  pkgs.ruff
  pkgs.ty
  pkgs.uv
  pkgs.playwright-driver.browsers
];
```

Now the dev shell includes the browsers needed to run Playwright tests, which lowers the friction for PRs that validate UI flows.

## CI metadata and repo housekeeping

Two small operational changes:

- The blog-generation workflow now pins a start-commit in .github/workflows/generate.yaml so output is reproducible:

```yaml
llm-model: gpt-5-mini-2025-08-07
start-commit: 'f18ffe9d9d437781d2c4ce3462757241608acf35'
temperature: '-1'
```

- We removed and then regenerated a frank-state.db file as part of that workflow change. This is housekeeping for the blog generator state, not related to runtime behavior.

## Practical recommendations and the final mental model

From these iterations we landed on a simple mental model and a few practical rules:

- Production: build the admin into the Caddy image (caddy/Dockerfile). Caddy is the canonical static server for the SPA in production images.
- CI: build and publish the Caddy image as part of your release pipeline so the production artifact is self-contained and reproducible.
- Local dev: allow the FastAPI app to serve /admin if /srv/admin is present. Use this to speed development without changing production assumptions. Document how to produce /srv/admin (either include it in the API image or run a local admin build and mount the directory).
- Avoid relying on Docker auto-population of named volumes for long-term reliability. Named volumes are fine for quick local experiments but can confuse contributors and CI.
- Add Playwright browser binaries to dev shells so E2E tests run consistently in CI and locally.

## How to try it locally

- Build the Caddy image (this bakes the admin build into the image):

  docker build -f caddy/Dockerfile -t find-workers-caddy .

- Build the API image using your normal image build command (whatever your project uses to produce the API image).

- Start the stack:

  docker-compose up

- Fast local-only preview without Caddy:

  - Run npm run build in the admin directory and mount the build output at /srv/admin in the API container, or use an API image that includes /srv/admin.
  - Open http://localhost:8000/admin/

## Takeaways

- Serving an admin SPA from a multi-service repo forces tradeoffs between convenience and reproducibility.
- Baking the frontend into the reverse proxy gives deterministic production artifacts and removes fragile shared-volume coupling.
- Guarded convenience in the API (serve /admin only when the build is present) makes local development faster without changing production behavior.
- Small infra tweaks — adding Playwright browsers to the dev shell, pinning CI generation commits — reduce friction for contributors and make builds and tests more reliable.

If you want, I can add a short onboarding checklist that spells out the local dev workflow and provides a small Makefile or compose wrapper to unify developer commands.
