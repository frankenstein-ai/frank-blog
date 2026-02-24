+++
date = '2026-02-22T22:14:05-03:00'
draft = false
title = 'Hardening Find Workers: webhooks, PII sanitization, MCP key rotation, and simplifying WhatsApp'
+++

## Problem first

Find Workers is a backend that connects Brazilian homeowners with local professionals. It uses MCP tools, Pix escrow via Woovi, and WhatsApp for delivery of notifications. That surface combines several risk factors:

- Incoming webhooks from payment providers (Woovi/OpenPix) and WhatsApp that must be processed reliably and safely.
- Financial flows that must avoid double charges, unreconciled releases, or misrouted payouts.
- High-privilege admin features and machine-to-machine (MCP) API keys used by AI assistants.
- Personal data (phone, CPF, Pix keys) appearing in free-text fields and webhook payloads.
- A tight developer feedback loop: changes must be covered by tests and runnable in CI.

In 2026‑W08 we ran a focused hardening pass that addressed webhook hygiene, PII sanitization, MCP key lifecycle, concurrency and race conditions around bookings/payments, and a decision to simplify the WhatsApp surface to notifications-only. Below I list what we changed, why, and concrete code patterns we used.

## What we changed (high level)

- Added MCP API key expiration and rotation: per-key expires_at (90-day default) and a POST rotate endpoint; verify_key rejects expired keys.
- Optional CIDR-based IP allowlisting for incoming webhooks, enforced before rate limiting.
- sanitize_free_text() helper to mask Brazilian phone numbers and CPF patterns in free-text API responses and stored webhook artifacts.
- Removed WhatsApp conversational machinery — WhatsApp is now notifications-only (client, templates, delivery tracking retained).
- Strengthened concurrency controls: SELECT FOR UPDATE on worker rows, a partial unique index to prevent double bookings, and IntegrityError fallbacks.
- Moved webhook dedup and replay protection to Redis with an in-memory fallback; improved quarantine and redaction of sensitive data in fraud logs and event payloads.
- Expanded rate limiting to protect auth and payment endpoints before expensive signature checks.

The test suite grew substantially during the week. The repo maintains a large regression suite (the work in this period exercises roughly 1.2k–3.6k tests across commits) and we kept CI green while iterating.

## Theme 1 — Webhook hygiene: trust nothing, check everything

Webhooks are both a reliability and a security problem. You still must validate authenticity (HMAC, RSA, etc.), but cheap checks up front stop obvious junk before you do expensive crypto or JSON parsing.

Two concrete changes

1) IP allowlisting before crypto and rate limiting

We added an optional CIDR allowlist that runs at webhook ingress. If configured, the upstream IP is checked before signature verification and rate limiting:

```python
# pseudo-code (api/webhooks.py)
if settings.webhook_ip_allowlist:
    if not ip_in_cidrs(request.client.host, settings.webhook_ip_allowlist):
        raise HTTPException(403, "webhook source not allowed")
# then run request-rate limiter and signature verification
```

This keeps an attacker from forcing repeated signature checks and JSON parsing.

2) Durable dedup and quarantine

In-process OrderedDicts used for dedup were replaced with a Redis-backed cache (with an in-memory fallback). Replay detection now survives restarts and multi-instance deployments. When a webhook can't be correlated to a local resource (charge/payout), we create a quarantine record and emit a structured alert rather than silently acknowledging and dropping it.

Before persisting we recursively sanitize webhook payload fragments, removing or redacting Pix keys, tax IDs and other sensitive identifiers.

## Theme 2 — PII sanitization and privacy constraints

Free-text fields and webhook blobs often include phone numbers and Brazilian tax identifiers (CPF/CNPJ). Those appeared in admin UIs and stored artifacts. We added a small sanitizer and enforce masking at network boundaries and for stored payloads.

sanitize_free_text masks phone and CPF patterns:

```python
# src/find_workers/utils.py (illustrative)
import re

_PHONE_RE = re.compile(r'(\+55\s?)?(?:\(?\d{2}\)?\s?)?\d{4,5}-?\d{4}')
_CPF_RE = re.compile(r'\b\d{3}\.\d{3}\.\d{3}-\d{2}\b|\b\d{11}\b')

def sanitize_free_text(text: str) -> str:
    text = _PHONE_RE.sub('+55***-****', text)
    text = _CPF_RE.sub('***.***.***-**', text)
    return text
```

We also:

- Recursively sanitize list elements in webhook payloads (transaction lists, history entries).
- Removed raw consumer_id and worker_id from fraud logs and event metadata where not required.
- Kept phone_number plaintext at rest by design: phone numbers are needed for OTP delivery, WhatsApp routing, and lookups. We documented and closed the thread about encrypting phone_number at rest as a deliberate tradeoff.

Tests for the sanitizer were added (tests/test_sanitize_free_text.py) and run in CI.

## Theme 3 — MCP API keys: lifecycle and bounded privileges

MCP API keys are powerful. We implemented lifecycle and binding rules to reduce blast radius:

- expires_at column on MCP ApiKey model (90-day default) via Alembic migration.
- verify_key rejects expired keys.
- POST /v1/admin/mcp-keys/{id}/rotate endpoint to rotate and revoke.
- Require bound_user_id for write-capable keys: creating a write MCP key must include a bound user ID, and enforcement rejects write keys without that binding.

These changes make rotation routine and limit what a leaked key can do.

## Theme 4 — Concurrency and integrity in bookings & payments

To avoid double bookings, duplicate charges, or simultaneous releases, we added database and transactional defenses:

- Partial unique index ix_bookings_one_active_per_consumer_worker to prevent two active bookings for the same consumer-worker pair.
- Lock critical rows with SELECT FOR UPDATE when accepting, starting, or completing bookings and when creating charges. Example pattern (SQLAlchemy):

```python
# pseudo-code from bookings/service.py
stmt = select(Worker).where(Worker.id == worker_id).with_for_update()
worker = await session.scalar(stmt)
# compute capacity and insert booking inside the same transaction
```

- Added an IntegrityError fallback on booking flush to handle rare races.
- Created TOCTOU tests that run concurrent accept/refund sequences to validate the locking logic.

These steps move correctness from best-effort checks to transactional guarantees.

## Theme 5 — Simplify WhatsApp: notifications only

We removed the WhatsApp chatbot/conversation finite-state machine and onboarding flows that assumed interactive WhatsApp sessions. The MCP server (AI assistants) is the primary consumer interface; WhatsApp is now a delivery channel for confirmations, reminders, and QR images.

Deleted modules
- whatsapp/booking_flow.py
- whatsapp/conversation.py
- whatsapp/dispatcher.py
- whatsapp/intent.py
- whatsapp/replies.py
- whatsapp/onboarding.py

Kept and improved
- whatsapp/client.py (send_text, send_template, send_image, send_document)
- whatsapp/notifications.py (templated notifications + retry with exponential backoff)
- whatsapp/templates.py and whatsapp/tracking.py

Removing the stateful chatbot code reduced surface area for bugs and focused development on the MCP-assisted flows.

## Other practical hardenings

- Rate limiting: per-user limits and token-bucket enforcement for auth (OTP/step-up) and payment endpoints (create/refund/release). Webhook routes now have pre-crypto rate limits so CPU can't be exhausted by bogus traffic.
- SSRF prevention: resolve profile_photo_url and reject private/reserved IP addresses before fetching images or documents.
- Metrics and ops: /metrics is protected by a bearer token and validated with hmac.compare_digest to avoid timing leaks. We added counters for permanent notification failures so ops can alert on lost messages.
- Tests and CI: Dozens of unit and regression tests across webhooks, payments, subscriptions, and admin APIs. The repository's test suite ran several thousand tests during the week; CI stayed green while we iterated on migrations and behavior.

## Concrete artifacts and numbers

- New DB migration: added expires_at to MCP ApiKey (90-day default).
- New tests: sanitize_free_text, webhook IP allowlist tests, booking concurrency and TOCTOU tests, among others.
- Deleted modules: six WhatsApp chatbot modules plus seven corresponding tests.
- Test count during the week varied as we expanded coverage, roughly 1.2k–3.6k tests; CI remained green after each major change.

## Lessons and recommendations

- Treat webhooks as untrusted input. Do cheap checks first (IP allowlist, body-size limits, rate limiting) before expensive crypto and JSON work.
- Use durable dedup (Redis or DB) for webhook replay detection. In-memory caches do not survive restarts or multi-instance deployments.
- Enforce business invariants with DB constraints and row locks (no double-bookings, idempotent charge creation).
- Keep conversational logic separate from delivery channels when your primary UX is an AI assistant: notification channels should be stateless and focused on delivery.
- Treat free-text fields as a PII source: sanitize before storing and mask when displayed in admin interfaces.

If it would help, I can extract the sanitizer and webhook allowlist into a small library you can drop into other projects, or sketch an operational runbook for rotating MCP keys and Woovi webhook keys.
