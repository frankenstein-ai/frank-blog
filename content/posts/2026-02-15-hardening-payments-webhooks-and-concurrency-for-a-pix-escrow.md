+++
date = '2026-02-15T23:51:20-03:00'
draft = false
title = 'Hardening payments, webhooks and concurrency for a Pix escrow marketplace'
+++

Building a marketplace that holds money in escrow and coordinates Pix payouts looks simple until it isn't. In 2026-W07 we spent the sprint on the ugly corner cases: webhook races and replay, double-disbursement risks, DB invariants and serialization races, PII leakage from provider payloads, and several authentication/Redis hardening fixes that together prevent catastrophic accounting or privacy bugs.

Below I describe what we found, the concrete fixes we committed, and the patterns other backend and payments engineers can reuse when integrating with external payment providers and webhooks.

## The problem

Find Workers is an MCP-first backend that uses Woovi (OpenPix) for Pix charges, virtual sub-accounts (escrow), and Pix Out payouts. That implies a few hard requirements:

- The system must keep a clear, auditable mapping of which booking or payment a balance belongs to and whether funds can be collected or must be released.
- Provider webhooks arrive out of order, are retried, and can hit while an API call is in flight.
- Webhook payloads carry customer and payer data (PII) that we must not persist verbatim.
- Concurrency between API callers, webhook handlers, and background reconciliation tasks creates race conditions that show up as double credits, duplicate active charges, or stale transitions.
- Infrastructure such as Redis needs secure configuration (certificate validation, idempotency state, rate limits) or attackers can spoof or escalate.

We prioritized P0 and P1 bugs that could cause money to leave twice, payments to get stuck, or sensitive data to leak. The patchset closed several high-severity issues.

## What we changed (summary)

- Webhooks: deduplication, quarantine for poison messages, PII sanitization, and support for rotating signature keys
- Payments: enforce single-active-charge, remove upfront provider splits, explicit create→approve payout flow, re-lock rows after remote calls to avoid double-credit
- DB and concurrency: FOR UPDATE on critical checks, CHECK constraints on state and financial columns, and reject ambiguous fallback lookups
- Auth and infra: defer user creation until OTP verification, require TLS verification for rediss://, and add Redis tests
- Ops and dev ergonomics: reproducible seed script, Pix key change rate limits, HTTPS-only profile_photo_url validation
- Tests: dozens of regression tests across payments, webhooks, bookings, and auth

Below I walk through the most interesting problems and the exact approaches we used.

## Payments and webhooks: preventing double-disbursement and double-credit

Problem 1 — double-disbursement
The original flow mixed two disbursement mechanisms. Charges could include a provider split at creation, and later the system would also call Pix Out to release funds. That created a real risk of paying a provider twice.

Fix
- Remove provider splits from create_service_charge.
- Hold the full amount in the platform escrow (sub-account).
- Disburse to the provider only via an explicit release_to_provider flow.

Commit note: "Removed splits from create_service_charge. Full amount now held in platform escrow; provider disbursement happens only via release_to_provider."

Problem 2 — race between webhook and synchronous API response causing double-credit
A remote refund or release could complete on Woovi while our API call was still in flight. If a webhook for the same event arrived concurrently, we could apply the change twice.

Fix
- After the remote API returns a terminal status (REFUNDED, CONFIRMED, etc.), re-acquire the payment row with FOR UPDATE before applying local changes. If the webhook already processed the event, skip updates and event creation.

Example (conceptual Python/SQLAlchemy):

```python
# re-acquire with FOR UPDATE after remote call
with session.begin():
    payment = (
        session.query(Payment)
        .filter(Payment.id == payment_id)
        .with_for_update()
        .one()
    )
    if payment.last_event_id == remote_event_id:
        # already processed by webhook — skip
        return
    payment.state = "refunded"
    session.add(payment)
```

Problem 3 — single-active-charge invariant
Reissuing expired charges without checking for existing collectible charges could create multiple collectible charges for one booking.

Fix
- Before reissuing an expired charge, check for any payment rows for the same booking that are in a collectible state (active, release_pending, refund_pending, etc.). Reject reissue if one exists.
- Add tests for both sequential and concurrent creation attempts.

Problem 4 — ambiguous endToEndId fallbacks
Some lookup paths used limit(1) and could accidentally match the wrong payment when multiple rows shared an endToEndId.

Fix
- Fetch all matching payments and explicitly reject when multiple match. This prevents cross-payment interference and makes ambiguous cases explicit. Tests cover the fallback paths.

## Webhook hygiene: dedup, quarantine, sanitize, and key rotation

We hardened webhook processing in several ways.

- Mark replay seen only after successful processing. Previously we marked a webhook hash as "seen" before processing; if processing failed, provider retries were dropped. Now the "seen" marker is written after commit.
- Quarantine poison-pill webhooks. Malformed or deterministically failing but validly signed payloads are quarantined so retries do not spin indefinitely and cause amplification.
- Sanitize event_data before persisting. Webhook payloads include nested charge, customer, payer, taxID, brCode, pixKey, and other PII. We added a sanitizer and apply it at all PaymentEvent.event_data write points.

Sanitizer sketch:

```python
def sanitize_event_data(raw):
    allowed = {"correlationID", "value", "status", "endToEndId", "createdAt"}
    sanitized = {k: raw[k] for k in raw if k in allowed}
    return sanitized
```

- Signature rotation. Woovi webhook verification used a single hardcoded key. We added support for a configured set of base64 public keys so an old and a new key can both be accepted during rotation windows, and we validate key formats on startup.

## Concurrency and DB constraints

Two practical fixes reduced the race window and moved checks into the database.

1) Booking deduplication
Prevent duplicate active bookings by serializing the dedup SELECT with FOR UPDATE. Without that lock, two concurrent create_booking calls could both pass and create duplicate active bookings.

Representative code:

```python
existing = (
    session.query(Booking)
    .filter_by(consumer_id=consumer_id)
    .filter(Booking.state.in_(ACTIVE_STATES))
    .with_for_update()
    .first()
)
if existing:
    raise DuplicateBookingError
```

2) CHECK constraints
Add DB-level CHECK constraints for state-machine columns and key financial columns to reduce the chance of invalid combinations being persisted. Tests cover these constraints.

These changes raise the bar: those bugs now require an explicit violation of DB constraints rather than just bad timing.

## Auth and Redis hardening

- Deferred user creation. request_otp no longer creates DB user rows; we create users deterministically in verify_otp only. That prevents account pre-creation abuse from attackers seeding phone numbers.
- OTP race tests. Added coverage for consume_otp returning None and for concurrent verify_otp attempts.
- Redis TLS verification. rediss:// connections now use ssl.create_default_context(), enforcing certificate verification and hostname checking; plain redis:// is unchanged for local development.
- Redis tests. Added direct tests for auth/session primitives (step-up, token revocation, consume_refresh_token GETDEL semantics).

## Operational and developer ergonomics

- Seed script. Added an idempotent seed script (find-workers-seed) that loads 10 Brazilian service categories and 5 sample workers with availability, encrypted Pix keys, and two consumers. This makes local dev and test scenarios reproducible.
- Pix key change rate-limit. Workers can change their Pix key at most 3 times per 7-day rolling window. This prevents a malicious actor from griefing releases by repeatedly resetting cooldown timers.
- profile_photo_url validation. Only allow https to avoid mixed-content and insecure-redirect tricks.
- Tests. Each fix shipped with regression tests. Examples: 7 tests for seed, 10 tests for webhook scrubbing, and multiple tests across payments and booking domains. We iterated until CI stability improved.

## Metrics and outcomes

- 28 commits in the sprint touching security, payments, webhooks, auth, and infra hardening.
- Several P0/P1 bugs closed related to double-disbursement, stale charge retry, idempotency, and webhook processing.
- Dozens of regression tests added (for example: 10 tests for PII scrubbing; 2 tests verifying Pix key plaintext is never populated).
- Operational guardrails added or documented: Redis TLS verification, Woovi public key rotation support, and rate limits for booking/charge creation.

## Lessons and practical recommendations

- Treat webhooks as an untrusted, out-of-order event stream. Persist only an allowlist of fields and sanitize PII aggressively.
- Push invariants into the database for financial systems: CHECK constraints and partial unique indexes catch errors you might miss in application code.
- Serialize critical checks (single-active-charge, single-active-booking) with FOR UPDATE, but keep transactions short to limit contention.
- After calling an external API that can trigger webhooks, re-read and re-lock the affected rows before writing final state: remote → webhook can beat your synchronous path.
- Test for concurrency and replay. Add tests for concurrent API calls, webhook duplicates, and stale webhooks replaying after retries.
- Treat operational safety (key rotation, TLS verification) as part of the feature, not as an afterthought.

## Next work

The issue tracker now contains epics for remaining items: WhatsApp rich messaging, MCP server tools, subscription billing via Woovi Pix Automático, semantic search with pgvector, and a few cleanup tasks. The payments and escrow core is safer and more deterministic; next work will focus on subscription billing, richer WhatsApp interactions (templates, media, QR flows), and embedding-based worker search.

If you’re integrating with Pix or any webhook-based payment provider, adopt these patterns: tighten webhook ingestion, minimize persisted payloads, add DB invariants, and design for duplicate and out-of-order events.
