+++
date = '2026-02-15T16:24:12-03:00'
draft = false
title = 'Hardening Find Workers: payment authz, OTP safety, webhook resilience, and abuse controls (2026-W07)'
+++

This week we ran a short, security-first sprint across Find Workers’ core surface: payments, authentication, webhooks, quote/booking flows, search, and reviews. The guiding question was practical: what changes stop easy financial abuse, gaming, or low-cost attacks that take the platform down?

We fixed the high-priority issues uncovered by an internal security audit and our follow-ups: payment IDORs and trusting caller-provided payout destinations, refresh-token bypasses, weak default JWT secrets, OTP abuse, unbounded in-memory structures, N+1 distance lookups in search, webhook memory DoS, and self-dealing in quotes/bookings. The work added explicit authorization tracing, stricter config validation, streaming-safe webhook parsing, bounded caches, Redis-backed atomic rate limits, DB-level constraints, and behavioral guards. We also added regression tests.

## The problems we found

- Financial integrity: payment endpoints allowed unauthorized reads/changes; release endpoints used caller-provided Pix destination fields and could send stolen payouts.
- Authentication abuse: refresh tokens were sometimes accepted via unsafe code paths; OTP verification had no attempt throttling; default JWT secret could be weak outside local dev.
- Availability / DoS: webhook handlers buffered full request bodies before size checks, allowing memory DoS. Search did per-worker distance calculations (N+1), multiplying work per request.
- Business fraud: quotes could be accepted in ways that bypassed booking dedup checks; users could create quotes that effectively self-hired their worker profiles.
- Data correctness: review.rating had no DB CHECK, text fields lacked length limits, and worker responses could be overwritten.

Most of these were P0–P2 items in our tracker. The week's goal was to close them and add tests to prevent regressions.

## What we changed (high level)

- Payment authorization: added an ownership-aware lookup (_get_payment_with_authz) that walks payment → booking → worker → owner and enforces least privilege for reads, refunds, and releases. Release now uses the provider’s stored pix_key; caller-provided destination is ignored. Refunds are limited to the consumer and admins.
- OTP and auth: store OTPs as HMACs, add Redis-backed per-phone atomic attempt counters with expiry and reset on success, normalize phone numbers, and reject inappropriate refresh-token flags.
- JWT secrets: fail-fast validation prevents insecure default JWT secrets in non-local deployments.
- Webhooks: switch to streaming chunk accounting with a 64 KB cap before buffering or signature work; malformed Content-Length values are handled safely; unmatched Woovi events go into a WebhookQuarantine table and return 200 with a quarantined flag to avoid endless retries.
- Search: remove N+1 distance queries by batching distance calculations.
- Quotes & bookings: prevent self-dealing (consumer cannot request quotes against their own worker profile) and apply create_booking dedup logic inside accept_quote to avoid duplicate active bookings.
- Reviews & DB constraints: add CHECK(rating BETWEEN 1 AND 5), length columns for review_text and worker_response, and return 409 on attempts to overwrite a worker response.
- Bounded memory: replace unbounded dedup sets with a bounded OrderedDict (10k max, evict oldest half).
- Tests: add regression tests covering webhook quarantine and streaming limits, OTP throttling, quote/booking dedup and self-deal rejection, review response overwrite, and auth/payment access checks.

## Concrete patterns and examples

1) OTP attempt throttling — atomic Redis counters with expiry

We use Redis INCR + EXPIRE so counters are atomic and auto-reset after a window. On verify attempts we increment, check the threshold, and reset on success.

```python
# src/find_workers/auth/redis.py (conceptual)
def attempt_counter_key(phone_normalized: str) -> str:
    return f"otp:attempts:{phone_normalized}"

def incr_attempt(redis, key, window_seconds=300):
    current = redis.incr(key)
    if current == 1:
        redis.expire(key, window_seconds)
    return current

def reset_attempts(redis, key):
    redis.delete(key)
```

Usage:

```python
count = incr_attempt(redis, attempt_counter_key(phone))
if count > MAX_VERIFY_ATTEMPTS:
    raise HTTPException(status_code=429, detail="Too many attempts")
if verify_hmac(stored_hmac, provided_otp):
    reset_attempts(redis, attempt_counter_key(phone))
    # continue
```

This avoids check-then-set races and accidental resets.

2) OTP storage with HMAC

Store HMAC(otp, server_key) instead of plaintext so a datastore leak doesn't expose valid codes.

```python
import hmac, hashlib

def otp_hmac(secret_key: bytes, otp: str) -> str:
    return hmac.new(secret_key, otp.encode(), hashlib.sha256).hexdigest()
```

Compare with constant-time equality.

3) Webhook streaming size enforcement

Do not read request.body() into memory before checking size. Track chunks and enforce a cap early.

```python
# src/find_workers/api/webhooks.py (conceptual)
MAX_WEBHOOK_BYTES = 64 * 1024  # 64KB

async def read_limited_body(request):
    total = 0
    chunks = []
    async for chunk in request.stream():
        total += len(chunk)
        if total > MAX_WEBHOOK_BYTES:
            raise HTTPException(status_code=413, detail="Payload too large")
        chunks.append(chunk)
    return b"".join(chunks)
```

After reading (within limits) parse JSON with json.loads to avoid double buffering. Treat malformed Content-Length as a 400/413 depending on context. For unmatched/unknown events, persist raw payload and metadata to WebhookQuarantine instead of failing the endpoint.

4) Payment authorization tracing

Replace uuid-based direct access with an ownership-walking helper that enforces roles:

- admin: full access
- consumer: read + refund restriction
- provider: limited view; release uses stored payout info and is platform-controlled

Pseudocode:

```python
def _get_payment_with_authz(db, payment_id, caller_user):
    payment = db.query(Payment).filter_by(id=payment_id).one_or_none()
    if not payment:
        raise NotFound
    booking = payment.booking
    worker = booking.worker
    if caller_user.is_admin:
        return payment
    if caller_user.id == booking.consumer_id:
        return payment
    if caller_user.id == worker.user_id and action_allowed_for_provider:
        return payment
    raise Forbidden
```

Always ignore caller-supplied payout destinations on release endpoints and read provider payout info from the DB.

5) Booking dedup and self-deal guard

Reuse the create_booking dedup check in accept_quote. Reject quote requests where the consumer_id equals the worker.user_id to close a simple self-hire vector.

6) DB-level safety for reviews

Add schema-level guards so application bugs can't create invalid rows:

- ALTER TABLE reviews ADD CHECK (rating BETWEEN 1 AND 5);
- review_text VARCHAR(5000)
- worker_response VARCHAR(2000)
- Return HTTP 409 if worker_response already exists for a review.

## Metrics and tests

- Closed the P0–P2 security issues tracked in .beads/issues.jsonl.
- Added regression tests across auth, payments, quotes, webhooks, reviews, and search. The test suite grew in small, focused increments and runs in CI.
- Webhook bodies are now capped at 64 KB and rejected with 413 before signature verification or JSON parsing.

## Lessons and recommendations

- Threat-model early. Combine DB constraints, API guards, and config validation (e.g., JWT secret checks) rather than relying on just one layer.
- Never trust client-provided payout destinations. Keep canonical payout keys in the DB and use those for transfers.
- For webhook endpoints that do expensive crypto or DB work, put cheap pre-checks first (body size, rate limit).
- Use Redis atomic ops for counters and simple rate-limits to avoid races.
- Enforce business invariants at both the API logic level (dedup checks) and the DB level (unique constraints, CHECKs).

## What’s next

Remaining items from the tracker: actor-aware throttling for quote creation, structured alerts and operator workflows for quarantined Woovi events (we persisted unmatched events; next is alerting and triage UIs), and refactoring shared Pix query helpers into woovi/queries.py for reuse.

If you run a payments + webhook + user-generated marketplace, these patterns will likely be on your backlog. The fixes here are small, focused, and testable — they make the platform harder to abuse and easier to maintain.
