+++
date = '2026-02-15T18:50:34-03:00'
draft = false
title = 'MCP tools, safer JWTs, and polish: what we shipped in 2026-W07'
+++

This week the Find Workers backend moved two changes forward: we shipped a small MCP server that exposes search and profile tools for AI assistants, and we removed an unmaintained JWT dependency that carried two CVEs. Both changes included tests and minor documentation updates.

Below I describe the problem we solved, the design choices we made, show concrete code excerpts, and explain the operational reasoning behind the JWT migration. If you maintain an MCP-first service or a production API, the trade-offs may be useful.

## The problems we set out to solve

- Let any MCP-capable assistant (ChatGPT, Claude, Gemini, Copilot) discover and inspect local service providers in a queryable, reliable way.
- Reduce risk from third-party auth libraries when those libraries become unmaintained or acquire CVEs.
- Fit small polish changes (docs, locale) into a sprint so support friction is lower.

Concretely:
- Build a small, fast MCP server exposing two tools: search_workers (filter by category, location, rating, sort by distance) and get_worker_profile (full profile with services, availability, recent reviews).
- Replace python-jose (used for JWT handling) after we found it unmaintained and associated with CVE-2024-33663 and CVE-2024-33664, moving to an actively maintained library (PyJWT).

## What we implemented this sprint

- Added a FastMCP-based MCP server module that exposes search_workers and get_worker_profile to assistants.
  - Added mcp>=1.0 to pyproject and 14 unit tests covering tools and DB query helpers; the tests pass in CI.
- Replaced python-jose with PyJWT across the codebase and tests to mitigate the known CVEs.
  - Updated JWT helpers and tests; pyproject.toml and the lockfile were updated.
- Fixed Portuguese diacritics in README.md to improve readability for Brazilian users.

Test counts and repo hygiene
- The MCP work added 14 tests; they pass in CI.
- The JWT migration changed src/find_workers/auth/jwt.py and its tests.
- pyproject.toml and the lockfile were updated twice (MCP dependency, then JWT migration).

## MCP server: design and key implementation details

Goal: expose a small, dependable set of tools an assistant can call for discovery and context retrieval. Keep assistant logic simple (ask for nearby electricians); do the safe, auditable queries on the backend.

Implementation highlights
- We use the mcp SDK’s FastMCP class and register the tools on a small module-level MCP server:
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("find-workers")
```

- Search is an async DB query (SQLAlchemy + PostGIS via geoalchemy2). Supported features:
  - Filter by service category
  - Optional geographic filter (latitude, longitude, radius_km)
  - Minimum rating threshold
  - Distance-based sorting and max_results cap

A core (truncated) search helper from the change:
```python
async def _search_workers(
    category: str,
    latitude: float | None = None,
    longitude: float | None = None,
    radius_km: float = 15.0,
    min_rating: float = 0.0,
    max_results: int = 10,
) -> list[dict]:
    """Query the database for workers matching the given criteria."""
    async with async_session() as session:
        stmt = (
            select(
                Worker.id,
                Worker.display_name,
                Worker.bio,
                Worker.rating_avg,
                Worker.rating_count,
                Worker.response_time_avg_minutes,
                Worker.is_verified,
                Worker.latitude, Worker.longitude,
            )
            .join(WorkerService)
            .join(ServiceCategory)
            .where(ServiceCategory.slug == category)
            .where(Worker.rating_avg >= min_rating)
            .limit(max_results)
        )

        if latitude is not None and longitude is not None:
            # geoalchemy2 functions are available as geo_func
            # compute distance and sort by it
            distance_expr = geo_func.ST_DistanceSphere(
                Worker.geom, func.ST_MakePoint(longitude, latitude)
            )
            stmt = stmt.add_columns(distance_expr.label("distance_m")).order_by("distance_m")
        results = await session.execute(stmt)
        ...
```

- get_worker_profile returns a normalized dict with services, availability (WorkerAvailability rows), and recent reviews; selectinload is used to eager-load relationships.

Why this shape?
- Two narrow tools (search and profile) make integration straightforward: the assistant requests candidates with search_workers, inspects a short list, then calls get_worker_profile for full context before actions such as requesting quotes or scheduling.
- Using PostGIS distance functions keeps queries efficient and sortable, which matters for low-latency assistant interactions.

Testing
- The sprint added 14 tests that exercise DB query helpers and the two MCP tools. They check ranking and filtering behavior (e.g., radius exclusion, rating thresholds, sorting) so we can change UI/assistant prompts without breaking core behavior.

## JWT migration: why, what changed, and compatibility notes

Problem
- python-jose was unmaintained and associated with CVE-2024-33663 and CVE-2024-33664. Because JWTs are central to authentication, we prioritized moving to an actively maintained library.

Decision
- Replace python-jose with PyJWT (imported as jwt or pyjwt).

What we changed
- Rewrote src/find_workers/auth/jwt.py to use PyJWT for encode/decode, preserving required features: HS256/HMAC secret, exp claim, issuer/subject claims, and explicit error handling for invalid or expired tokens.
- Updated tests in tests/test_auth_jwt.py to match the new API behavior.

Concrete example (before/after)
- Before (python-jose style; simplified):
```python
from jose import jwt
token = jwt.encode(payload, secret, algorithm="HS256")
decoded = jwt.decode(token, secret, algorithms=["HS256"])
```

- After (PyJWT):
```python
import jwt as pyjwt
token = pyjwt.encode(payload, secret, algorithm="HS256")
decoded = pyjwt.decode(token, secret, algorithms=["HS256"], options={"require": ["exp", "iat", "sub"]})
```

Operational notes
- PyJWT receives regular updates and has a stable API for our needs. We updated pyproject.toml and the lockfile.
- Tests for token creation and verification are tight to catch claim/expiry differences.
- We verified tokens remain consumable by existing auth flows (refresh, session validation) and updated call sites where decode semantics differed.

## Documentation polish

- README.md received fixes for Portuguese diacritics. Properly accented copy reduces friction for Brazilian product managers, partners, and contributors.

## Lessons learned and next steps

- Narrow, focused MCP tools are sufficient for assistant-driven discovery: search plus profile lets assistants ask clarifying questions and then call existing endpoints for quotes, bookings, or payments.
- Monitor auth libraries and transitive dependencies. The python-jose → PyJWT migration was straightforward, but dependency maintenance and CVE feeds need regular attention.
- Tests matter: the 14 new unit tests and updated JWT tests gave us confidence to ship quickly.
- Ops/security TODO: open issue noting "No request body size limit on non-webhook API endpoints — memory exhaustion via large POST bodies". Next steps are enforcing request size limits and rate limits at the gateway/ASGI server layer or via middleware.

If you’re building something similar
- For MCP integrations: design narrow, deterministic tools that return small, structured responses. Keep heavy lifting (sorting, geo, access control) server-side.
- For auth libraries: prefer actively maintained libraries, pin versions, and add CI checks that scan for known CVEs.
- For testing: add DB-backed tests for ranking and geo queries — regional search regressions are usually the first behavior product teams notice.

This sprint was short and focused: assistant-driven worker discovery is available, the auth dependency risk has been reduced, and small docs fixes are in place. Next up: extend MCP tools with quotes and booking context, add request-size protections, and iterate on ranking signals (response time, recent activity, verified badges).
