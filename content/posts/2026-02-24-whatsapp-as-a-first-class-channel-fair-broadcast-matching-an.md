+++
date = '2026-02-24T02:17:43-03:00'
draft = false
title = 'WhatsApp as a first-class channel, fair broadcast matching, and hardening: find-workers — 2026-W09'
+++

What does it take to turn a backend built for AI assistants into a zero-app marketplace on WhatsApp — and do it securely and fairly?

During 2026-W09 we pushed a set of focused changes to Find Workers so consumers can complete search → quote → booking → payment entirely inside WhatsApp, and providers receive broadcasts that are fair and actionable. We also tightened security around the admin SPA and public APIs, fixed LGPD-sensitive leaks, and added developer conveniences and tests to keep regressions at bay.

Below I explain the problems we solved, why they matter for engineers building chat channels and marketplace dispatch, and share concrete code-level highlights you can reuse.

## problems we faced

- WhatsApp webhooks are stateless, but consumers need a stateful conversation flow (search → view results → request quotes → accept quotes → confirm completion).
- Broadcasts (one-to-many quote requests) must be fair: do not always hit the same highly available workers first.
- Interactive button replies and list selections from WhatsApp must map directly to business actions (accept booking, reject quote, report problem) with robust routing and replay/isolation guards.
- The admin SPA and public API surface needed a CSP and SPA fallbacks so assets load correctly without weakening API security; public endpoints also needed rate limiting.
- PII in reviews and onboarding data required sanitization and handling under LGPD rules.
- Developer ergonomics for local testing were necessary without shipping insecure defaults to production.

Across these changes we added stateful routing, notification templates, a matching engine, security fixes, and dozens of tests to validate the flow.

## what we implemented

High-level summary:

- WhatsApp consumer channel
  - Redis-backed conversation state machine for consumer sessions (idle → viewing_results → awaiting_quotes → reviewing_quotes → confirming_booking → awaiting_payment) with LGPD-compliant phone hashing and TTLs.
  - Inbound router that handles button/title routing first (worker accept/reject, consumer confirm/report) and falls back to a consumer orchestrator for text and list replies.
  - New WhatsApp templates and a notify_quote_requested helper so workers receive quote requests via WhatsApp.
  - Routing actions now set two booking fields: consumer_confirmed and flagged_for_review (migration included).

- Matching and dispatch
  - MatchingEngine with a fairness rotation used for broadcast dispatch so each eligible worker gets rotating priority.

- Security, infra, and UX
  - CSP update to allow SPA assets while keeping API paths restrictive.
  - StaticFiles subclass that serves index.html for client-side routes (SPA fallback).
  - Allowed Google Fonts for the admin SPA (fonts.googleapis.com / fonts.gstatic.com).
  - Public IP rate limiting and per-user rate limits for writes; default public rate_limit_public = 30 req/min.
  - PII sanitization on public reviews.
  - Woovi webhook validation tightened (production requires woovi_account_environment).
  - Dev-mode admin authentication bypass for local dev (guarded by DEV_MODE).

- Tests
  - Integration and unit tests across the new flow: 14 conversation state tests, 23 inbound router tests (17 + 6 integration), notifications tests, matching tests, and more.

Below I dive into the most useful parts for engineers: the conversation state machine, the inbound router, the MatchingEngine rotation, and a couple of concrete code highlights (CSP and SPA fallback).

## redis-backed conversation state machine (WhatsApp)

Problem: WhatsApp inbound webhooks are noisy and stateless. For a smooth UX we must remember where a phone number is in the flow, protect phone numbers under LGPD, and let conversations expire.

Design choices we used
- Hash phone numbers for Redis keys (first 16 hex chars of SHA-256) so raw numbers are not used as keys.
- Store a small JSON blob with state, relevant IDs, and timestamps.
- Use a TTL (30 minutes default) so conversations auto-expire.
- Provide a graceful fallback when Redis is unavailable (stateless handling to avoid blocking messages).

Example shape (src/find_workers/whatsapp/conversation.py):

```python
import hashlib
import json
from typing import Mapping

def phone_hash(phone: str) -> str:
    h = hashlib.sha256(phone.encode("utf-8")).hexdigest()
    return h[:16]

KEY_FMT = "wa_conv:{}"  # wa_conv:{phone_hash}

class ConversationManager:
    def __init__(self, redis, ttl_seconds=1800):
        self.redis = redis
        self.ttl = ttl_seconds

    async def get_state(self, phone: str) -> Mapping | None:
        key = KEY_FMT.format(phone_hash(phone))
        raw = await self.redis.get(key)
        if not raw:
            return None
        return json.loads(raw)

    async def set_state(self, phone: str, state: Mapping):
        key = KEY_FMT.format(phone_hash(phone))
        await self.redis.set(key, json.dumps(state), ex=self.ttl)
```

The commit added 14 tests covering transitions and phone-hash behavior.

## inbound router: button-first, orchestrator fallback

Interactive messages (buttons, lists) should map directly to domain actions to avoid latency and misrouting.

Router behavior
1. Button-title routing: immediate actions like worker Accept/Reject, consumer Accept/Reject quote, consumer Confirm completion, Report problem → mapped to booking/quote endpoints.
2. Consumer orchestrator fallback: text messages and list replies use the conversation manager to continue the search → results → broadcast → accept flow.

The webhook handler now passes redis and contact_name into the router so handlers can be stateful. Six integration tests validate text routing, list replies, button IDs, error isolation, and non-routable messages.

We also added two booking columns:
- consumer_confirmed (bool)
- flagged_for_review (bool)

A migration keeps the DB consistent.

## matching engine with fairness rotation

Problem: Naive broadcast dispatch repeatedly reaches the same available workers first. That biases opportunity and harms marketplace health.

Approach
- Maintain a fairness key (last_served_at or a per-service rotation pointer).
- When dispatching, sort eligible workers by last_served_at ascending and then by score, pick top N, and update last_served_at for chosen workers.
- Break ties by score (rating, proximity, recent activity).

Illustrative logic (src/find_workers/matching.py):

```python
class MatchingEngine:
    def __init__(self, db):
        self.db = db

    async def broadcast_candidates(self, service_id, limit=10):
        query = """
        SELECT id, last_served_at, score
        FROM workers
        WHERE service_id = :sid AND is_available = true
        ORDER BY COALESCE(last_served_at, '1970-01-01') ASC, score DESC
        LIMIT :limit
        """
        rows = await self.db.fetch_all(query, {"sid": service_id, "limit": limit})
        now = datetime.utcnow()
        await self.db.execute_many(
            "UPDATE workers SET last_served_at = :now WHERE id = :id",
            [{"now": now, "id": r["id"]} for r in rows],
        )
        return rows
```

Tests assert that repeated broadcasts rotate across workers rather than hitting the same subset repeatedly.

## WhatsApp notifications and worker quote template

We added a worker-facing template WORKER_QUOTE_REQUEST and a helper notify_quote_requested(client, worker_phone, worker_name, service_description, consumer_name) in notifications.py. The helper wraps the internal send behavior, logs failures, and does not block quote creation.

Unit tests cover:
- Template existence and variable substitution.
- notify_quote_requested calling the internal send wrapper.
- Notification failures do not prevent Quote objects from being created.

These changes make broadcasts actionable for providers.

## security and infra hardening

A few practical fixes to make the platform more secure and admin-friendly.

CSP and SPA support
- API prefixes (/v1/, /health, /webhooks/, /mcp, /metrics) use a strict CSP and no-store cache headers.
- Admin SPA routes get a CSP that allows scripts, styles, images, and connect-src 'self'. Google Fonts are allowed for the admin UI.
- Implemented a StaticFiles subclass that returns index.html for unknown paths so client-side routing works.

Example (src/find_workers/main.py):

```python
_api_prefixes = ("/v1/", "/health", "/webhooks/", "/mcp", "/metrics")
if request.url.path.startswith(_api_prefixes):
    response.headers["Cache-Control"] = "no-store"
    response.headers["Content-Security-Policy"] = "default-src 'none'; frame-ancestors 'none'"
else:
    response.headers["Content-Security-Policy"] = (
        "default-src 'self'; script-src 'self'; "
        "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; "
        "font-src 'self' https://fonts.gstatic.com; "
        "img-src 'self' data:; connect-src 'self'; frame-ancestors 'none'"
    )
```

SPA fallback:

```python
class _SPAStaticFiles(StaticFiles):
    async def get_response(self, path: str, scope):
        try:
            return await super().get_response(path, scope)
        except _StarletteHTTPException as exc:
            if exc.status_code == 404:
                return await super().get_response("index.html", scope)
            raise
```

Rate limiting and degradation
- IP rate limiting for public endpoints and per-user limits for authenticated writes.
- Default public rate_limit_public = 30 req/min.
- get_optional_redis helper so public endpoints degrade gracefully if Redis is absent.

PII & LGPD
- sanitize_free_text applied to the public reviews endpoint so review_text and worker_response do not leak PII.
- Redis conversation keys use phone hashing (first 16 hex chars of SHA-256).

Webhook validation
- Woovi webhooks require woovi_account_environment in production to reject sandbox webhooks.

## developer ergonomics

Local dev convenience that is guarded in production:
- When DEV_MODE=1 a known phone number with OTP 000000 authenticates as super_admin. This path is rejected at startup in production.
- Tests and local helpers make it faster to iterate without shipping insecure defaults.

## tests and outcomes

- Conversation state machine: 14 tests.
- WhatsApp inbound router: 17 unit tests + 6 integration tests.
- Notifications: unit tests ensure non-blocking behavior.
- Matching and broadcast: tests validate rotation fairness.
- Security: tests cover CSP/static fallback issues that caused regressions before.

The test suite baseline is 1100+ tests; these commits added focused coverage for the new flows.

## lessons and next steps

- Keep the phone-level state small and ephemeral. Storing only needed IDs and a tiny state enum reduces PII surface and simplifies transitions.
- Fairness in broadcast dispatch is cheap to implement (last_served_at) and yields much better UX for providers.
- Route interactive payloads (buttons, lists) to domain actions first and fall back to text/NLP. That reduces latency and avoids unnecessary orchestration.
- Tests that simulate webhook payloads and replay cases pay off. Move replay dedup state into Redis or DB so it survives restarts and multi-instance deployments.

Planned next steps include moving webhook replay deduplication to Redis for multi-instance safety and iterating on the matching ranking function to incorporate worker preferences and surge controls.

If you want to try these changes locally, run the test suite (uv run pytest) and inspect the new modules under src/find_workers/whatsapp and src/find_workers/matching. The conversation manager, router, and MatchingEngine are modular and reusable in other messaging-first marketplaces.
