+++
date = '2026-02-15T18:17:05-03:00'
draft = false
title = 'Building a production-ready MCP-native marketplace: bookings, Pix escrow, WhatsApp, LGPD and a week of hardening'
+++

What happens when you combine an MCP-style assistant layer, Pix escrow payments, WhatsApp as the primary UX channel, and Brazil's LGPD? Over 2026-W07 we shipped the core pieces for Find Workers: worker profiles and search, a booking lifecycle, Pix payments and webhooks, WhatsApp integration, review flows, LGPD endpoints (DSAR + erasure), and a focused security hardening pass that fixed several critical operational bugs.

This write-up explains the problems we faced, the trade-offs we made, and concrete code-level patterns we landed in the codebase. It’s aimed at engineers building marketplaces (especially where payments and data-protection matter), platform authors focused on compliance, and folks integrating agent/assistant flows.

## Who this is for
Engineers building escrowed marketplaces, compliance-minded platform teams, and integrators using messaging-first UX.

## Problems we set out to solve
We needed to:
- Let MCP assistants operate on behalf of users without exposing payment routing or breaking LGPD erasure/export requirements.
- Connect WhatsApp interactions to booking and payment state reliably, with signed webhooks, deduplication, and quarantine for malformed events.
- Prevent token or OTP leakage from escalating (avoid cross-purpose OTP reuse, safe token revocation).
- Produce deterministic, auditable exports for DSARs and reproducible ordering for legal timelines.

These requirements touch payments, data protection, communications, auth, and search.

## What we shipped (high level)
- Worker domain: profile creation/update, service offerings, availability, reviews, and PostGIS-backed search with deterministic ranking.
- Booking domain: BookingService state machine (pending → accepted → in_progress → completed/cancelled) and REST endpoints for booking actions.
- Payments: Pix integration via Woovi (charges, refunds, releases) plus webhook validation and event-handling fixes.
- WhatsApp: Business Cloud API client (templates, interactive messages) and webhook handling with HMAC-SHA256 verification and deduplication.
- Reviews: create/respond flows that update worker ratings automatically.
- LGPD: consent recording/revocation endpoints, DSAR export endpoints, erasure requests with in-flight payment checks, and expanded export coverage (profiles, WhatsApp logs, quotes, subscriptions, LGPD history).
- Security & hardening: body-size limits, Cache-Control: no-store, token revocation via Redis epoch, CORS configuration, deterministic export ordering, DB constraints and on-delete policies, JSONB indexes, and migrations.

We iterated quickly across the week. Test counts in commits rose as features and fixes landed (sample milestones: 289 → 317 → 331 → 366 → 845 → 1048 → 1066 passing tests).

## Developer highlights and concrete examples

Below are the design decisions and code-level patterns we found most useful.

### 1) LGPD: DSAR export and safe erasure
Problem: A naive erasure routine can remove payment routing fields (Pix keys, release_destination_alias) while funds are still escrowed, which can permanently lock money.

What we changed
- export_user_data returns worker_profile, WhatsApp messages, quotes, subscriptions, and LGPD request history so DSARs are complete.
- Erasure requests are gated: if the user has in-flight payments, the erasure is blocked until those payments reach a terminal state.

Example (conceptual):

```python
# src/find_workers/lgpd/service.py (pseudocode)
in_flight = db.query(Payment).filter(
    Payment.user_id == user_id,
    Payment.state.in_(["pending", "authorized", "processing"])
).exists()

if in_flight:
    raise LGPDErasureBlocked("Active payments are in-flight; erasure denied until settled")
```

We also made exports deterministic by ordering DSAR queries by (created_at, id). That makes the export reproducible for audits and legal timelines.

Why this matters
Blocking erasure while escrowed funds exist prevents an irreversible operational loss (locked funds) without removing the user's ability to request their data.

### 2) Pix + Woovi: webhook fixes and safety checks
Changes we made
- Require essential correlation fields in webhook events; invalid or missing fields are quarantined instead of silently ignored.
- Fix wiring for sourceAccountId and add validation around reversal amounts to avoid applying incorrect reversals.
- Add JSONB indexes on payment.metadata paths used by reporting queries.

The webhook path now combines strict HMAC verification with payload schema validation before any state transition runs.

### 3) Booking lifecycle & payments: reliable state machine
What we implemented
- BookingService with guarded transitions: pending → accepted → in_progress → completed/cancelled.
- REST endpoints for create, accept, start, complete, cancel, and listing for consumers and workers.
- Payment endpoints (create/get/refund/release) integrated with PixService.

This ensures reviews are only allowed on completed bookings and that review responses can update worker ratings consistently.

### 4) WhatsApp integration: signed webhooks, dedupe, templates
What we shipped
- HTTP client for the Business Cloud API and typed Pydantic schemas for messages, statuses, and contacts.
- HMAC-SHA256 signature verification on webhooks.
- Deduplication and DB logging to avoid double-processing interactive replies.

Using WhatsApp as the notification and UX surface gives us an auditable channel for booking confirmations, reminders, and Pix QR codes.

### 5) Worker search: PostGIS + deterministic ranking
Search is production-ready now:
- API: GET /v1/workers/search?service=encanador&lat=...&lng=...&radius=...
- Implementation uses PostGIS ST_DWithin for geo-filtering and aggregates price ranges from worker services.
- Deterministic ordering: rating_avg desc, completion_rate desc; distance included when geo params are present.

Example intent:

```python
# src/find_workers/workers/service.py (pseudocode)
q = session.query(Worker).join(WorkerService).filter(
    func.ST_DWithin(Worker.location, point(lng, lat), radius_meters)
)
q = q.order_by(Worker.rating_avg.desc(), Worker.completion_rate.desc())
```

Determinism helps MCP agents produce reproducible recommendations and simplifies audits.

### 6) Auth hardening: OTP namespaces and token revocation epoch
Small changes with large impact
- OTP namespaces in Redis prevent cross-purpose reuse:
  - login OTPs -> otp:login:{phone}
  - step-up OTPs -> otp:stepup:{phone}

This prevents a low-privilege OTP from being reused for a higher-privilege flow.

- Token revocation: revoke_user_tokens writes a per-user revocation epoch into Redis. get_current_user checks the token's iat against that epoch and rejects tokens issued before revocation.

Conceptual snippet:

```python
# src/find_workers/auth/dependencies.py (pseudocode)
revoked_at = redis.get(f"tokens:revoked_at:{user_id}")
if revoked_at and token.iat <= revoked_at:
    raise InvalidToken("revoked")
```

The revocation check is fail-open (skip if Redis unreachable) to avoid availability problems while improving security when Redis is available. We also used atomic GETDEL or GET+DELETE semantics to consume refresh tokens and avoid replay races.

### 7) Security & middleware improvements
The security batch addressed a set of operational issues and tightened defaults:
- Add Cache-Control: no-store on API responses.
- Configurable DB pool parameters.
- Deterministic ordering on DSAR exports.
- Configurable CORS middleware.
- Body-size limit middleware (default 1 MiB, respond 413 on excess).
- DB CHECK constraints (e.g., user_type) and on-delete policies on 17 foreign keys.
- JSONB functional indexes for common payment metadata queries.
- Alembic migrations for schema changes and a cleanup migration to drop dead columns.

These changes reduce accidental data exposure and eliminate several classes of operational surprises.

## Testing, metrics, and developer ergonomics
- Tests: We added service-layer tests, webhook schema tests, and integration points. Test counts in commits rose as features hardened (see milestone numbers above).
- Fast feedback: Typed Pydantic schemas for webhooks and service-layer tests caught many mistakes early.
- Reproducibility: Deterministic export ordering, strict webhook validation, and OTP namespaces make debugging and audits simpler.

## Lessons learned
- Compliance is an engineering problem. LGPD-driven behaviors (export and erase) need cross-cutting checks into payment and escrow logic. The erasure gate is a necessary safety valve.
- Invest in determinism. Ordering by (created_at, id) and deterministic search/ranking pay off for auditability and for consistent agent responses.
- Small changes matter. OTP key namespaces and token revocation epochs are low-friction changes that remove high-risk attack surfaces.
- Webhook processing should be conservative. Signature verification plus schema checks and quarantine is a robust pattern when upstream delivery is messy.

## Where to read the code
Key files touched this week:
- LGPD: src/find_workers/lgpd/service.py, src/find_workers/lgpd/schemas.py, src/find_workers/api/lgpd.py
- Payments & webhooks: src/find_workers/api/webhooks.py, src/find_workers/woovi/*
- Bookings: src/find_workers/bookings/service.py, src/find_workers/api/bookings.py
- Workers & search: src/find_workers/workers/service.py, src/find_workers/api/workers.py
- WhatsApp: src/find_workers/whatsapp/client.py, src/find_workers/whatsapp/webhook.py
- Auth: src/find_workers/auth/service.py, src/find_workers/auth/redis.py, src/find_workers/auth/dependencies.py

If you’re building a similar marketplace with escrowed payments, agent interfaces, or strict regulatory requirements, the practical patterns worth borrowing are:
- data-erasure gates,
- deterministic exports,
- signed and deduplicated webhooks,
- namespace separation for short-lived secrets,
- defensive middleware and DB constraints.

If you want a deeper dive, tell me which of these you prefer:
- the LGPD export schema plus a sample DSAR JSON,
- the PostGIS query and indices we used for performant search,
- a sequence diagram for WhatsApp → booking → charge → webhook → release.
