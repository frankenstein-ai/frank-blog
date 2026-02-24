+++
date = '2026-02-17T23:19:44-03:00'
draft = false
title = 'When an MCP sub-app returns 500: lifespan, Docker layers and a smoother dev experience'
+++

## The problem

We saw intermittent 500 responses from /mcp when the MCP sub-application was mounted under FastAPI. At the same time our production Docker build and compose workflow felt brittle during local iteration.

During the Week 8 sprint I tracked down a reproducible 500 from the Find Workers backend's /mcp/* endpoint. That endpoint hosts a FastMCP (Starlette-based) sub-application mounted at /mcp. When the sub-app ran directly everything behaved; when mounted with FastAPI’s app.mount() the MCP route's lifespan hooks never ran and the session manager (which manages MCP conversations and session lifecycle) never started. That left requests hitting an uninitialized runtime and caused 500s and failing integration tests.

While fixing the lifecycle bug I also tightened the Dockerfile, added a small production-like compose configuration with a Caddy reverse proxy and health checks, and clarified developer docs so we don't repeat this.

Below I explain the root cause, show the code and container changes, note the test implications, and summarize developer experience improvements.

## Diagnosis: Starlette/FastAPI lifespan semantics

Starlette (and FastAPI) support lifespan hooks for startup and shutdown. Mounting a Starlette sub-app with app.mount("/mcp", sub_app) attaches its routes but does not run the sub-app's lifespan handlers. That behavior is intentional: route mounting does not wire sub-app lifecycles into the parent.

Our MCP stack exposes a streamable ASGI app (streamable_http_app()) that expects its session manager to start in its lifespan. When mounted, those startup handlers never executed and the session manager stayed uninitialized. Requests to /mcp failed with 500 errors and tests that exercised the mounted app failed.

Two simple lessons:
- If a component needs its own startup, mounting it as a sub-app is not enough.
- Start background services from the parent app’s lifespan when you mount a sub-app.

## The fix: start the MCP session manager in the parent FastAPI lifespan

I moved session manager startup into the FastAPI lifespan so the parent app starts and stops the MCP runtime.

Illustrative change (actual commit updates src/find_workers/main.py):

```python
# src/find_workers/main.py  (illustrative)
from find_workers.mcp_server import session_manager
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # start the MCP session manager so mounted /mcp routes have the runtime they need
    await session_manager.run()
    try:
        yield
    finally:
        await session_manager.shutdown()

app = FastAPI(lifespan=lifespan)

# mount the MCP sub-app (it still provides routes; its lifecycle is handled above)
app.mount("/mcp", streamable_http_app(), name="mcp")
```

Concretely:
- session_manager.run() is invoked inside the parent app’s lifespan rather than relying on the sub-app to run it.
- That ensures the MCP runtime is available when /mcp is hit, removing the 500s.

After this change, tests that previously failed on /mcp (see tests/test_mcp_server.py) passed and the mounted MCP routes behaved reliably under the full FastAPI app.

## Shipping production-ready images and a sensible dev compose

While fixing the lifecycle issue I also improved the container build and compose setup to make local production-like runs easier and more reproducible.

Added artifacts:
- Multi-stage Dockerfile that uses uv for dependency management and produces a small runtime image with gunicorn + uvicorn workers.
- Entrypoint script that runs alembic migrations before starting the app.
- Caddyfile for automatic TLS and reverse proxying to the API service.
- docker-compose configuration that adds api and caddy services and includes health checks.

Caddyfile (reverse proxy + auto-TLS; DOMAIN env var configurable):

```text
{$DOMAIN:localhost} {
    handle /webhooks/* {
        reverse_proxy api:8000
    }

    handle /mcp* {
        reverse_proxy api:8000
    }

    handle /health* {
        reverse_proxy api:8000
    }

    handle /metrics {
        reverse_proxy api:8000
    }

    handle /* {
        reverse_proxy api:8000
    }
}
```

Dockerfile (multi-stage build with uv tooling and runtime layer):

```dockerfile
FROM python:3.12-slim AS builder
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-editable
COPY src/ src/
COPY alembic/ alembic/
COPY alembic.ini .
COPY entrypoint.sh .
RUN uv sync --frozen --no-dev

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app /app
RUN chmod +x /app/entrypoint.sh
ENV PATH="/app/.venv/bin:$PATH"
EXPOSE 8000
ENTRYPOINT ["/app/entrypoint.sh"]
CMD ["gunicorn", "find_workers.main:app", \
     "-k", "uvicorn.workers.UvicornWorker", \
     "-b", "0.0.0.0:8000", \
     "--workers", "4"]
```

docker-compose excerpt for the api service and a healthcheck (healthcheck runs a local HTTP GET):

```yaml
services:
  api:
    build: .
    env_file:
      - path: .env
        required: false
    environment:
      DATABASE_URL: postgresql+asyncpg://${POSTGRES_USER:-fw}:${POSTGRES_PASSWORD:-fw}@postgres:5432/${POSTGRES_DB:-findworkers}
      REDIS_URL: redis://redis:6379/0
      HOST: "0.0.0.0"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import httpx; httpx.get('http://localhost:8000/health').raise_for_status()"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

Why these changes help
- The multi-stage build keeps the final image small and makes dependency installation reproducible with the same toolchain used in dev.
- Running alembic in the entrypoint reduces migration drift when containers start.
- Caddy provides an easy local HTTPS surface for webhook testing and mirrors a simple routing layer from production.
- The healthcheck ensures orchestration waits until the app is actually ready before sending traffic.

I also reordered Dockerfile layers so changing code won't force a full reinstall of dependencies, and added a dev-only ALLOW_INSECURE_DEFAULTS option in docker-compose so local developers can opt into relaxed settings without affecting CI or production.

## Tests and small infra hygiene

- tests/test_mcp_server.py was updated to reflect the lifespan change and now exercises the MCP mount correctly. That removed a class of brittle failures caused by lifecycle mismatch.
- .gitignore was tightened to include both /.cache/ and /.cache-home/ to avoid leaking local caches into the repo and reduce noise in CI.

## Developer experience and agent guidance

I updated CLAUDE.md with a few targeted improvements:
- Clearer instructions for running developer tooling in the nix dev environment (for example: nix develop -c uv sync, nix develop -c ruff check).
- A short note to prefer AsyncMock over MagicMock for async tests.
- Reminders about procedural conventions: create a bead (bd) task before starting work and tidy up after landing.

These are small changes but they make the environment easier to reproduce and lower the chance of subtle bugs (like lifecycle surprises) slipping in.

## Concrete takeaways

- Mounting a Starlette/FastAPI sub-app with app.mount() does not run the sub-app's lifespan. Start long-running pieces from the parent app’s lifecycle.
- Make background services (session managers, event loops, connectors) start explicitly in the top-level app.
- Multi-stage Dockerfiles plus an entrypoint for migrations make running near-production containers simpler locally.
- Healthchecks that hit a real route prevent premature routing of traffic during startup.
- Small documentation updates (mocking rules, nix commands) save time across the team.

## What changed, at a glance

- Fixed /mcp 500 by moving session_manager.run() into the FastAPI lifespan (src/find_workers/main.py).
- Added production artifacts: Caddyfile, entrypoint.sh, multi-stage Dockerfile, and docker-compose api + caddy with health checks.
- Improved developer docs (CLAUDE.md) with nix commands and mocking recommendations.
- Ignored .cache-home in .gitignore to keep local state cleaner.
- Adjusted tests to reflect lifecycle changes (tests/test_mcp_server.py).

If you run an ASGI app that mounts other ASGI apps (MCP, admin consoles, websocket apps, etc.), mount the sub-app for routes but orchestrate its lifecycle in the parent. That avoids a class of subtle runtime errors and makes integration tests match the real runtime shape of the service.

If you want the exact diffs or help applying this lifespan pattern to your ASGI components, ping me and I’ll share a small helper function we use to wire lifecycles into FastAPI.
