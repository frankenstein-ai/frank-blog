+++
date = '2026-02-15T14:40:04-03:00'
draft = false
title = 'Building a production-grade Pix escrow integration: lessons from implementing Woovi (OpenPix) in Find Workers'
+++

Problem — how do you add Pix escrow payments (OpenPix / Woovi) to a marketplace backend and make it reliable in production? Creating a charge and waiting for webhooks is just the start. Real payment systems must handle durability boundaries, idempotency under concurrent requests, webhook security and flood control, PII sanitization, dispute linkage, provider-side edge cases (insufficient balance, delayed approvals), and a code structure that’s easy to fix later.

Over the last sprint we shipped a Woovi (OpenPix) integration into Find Workers and hardened it iteratively. We released the client, service, and webhook, then fixed several P0/P1 bugs around durability and reconciliation, tightened webhook controls, split a 1.5k-line module into focused pieces, and added tests that exercise edge cases. Below I walk through what we built, the real bugs we hit, and the concrete changes that made the integration production-ready.

What we shipped first
- An async WooviClient for HTTP calls: charge creation, refunds, Pix Out (payout), and small subscription helpers.
- RSA SHA-256 verification for Woovi webhook signatures and parsing.
- PixPaymentService implementing escrow split logic (platform fee + provider sub-account) with APIs: create_charge, refund, release, reconcile.
- A Woovi webhook endpoint that turns lifecycle events into local state transitions.
- Tests: 35 tests covering client behavior, webhook parsing, and service happy paths.

Why the first pass wasn’t enough
Payments are rare and sensitive; small mistakes become outages or money loss. During internal review and simulated races we saw several recurring problems:

- Durability boundary mistakes: we used session.flush() before remote calls. If the process crashed after flush but before commit, local state could disagree with provider-side operations.
- Race conditions on charge creation: check-then-create without locks allowed concurrent requests to both call Woovi and then race on DB insert — duplicates and integrity errors followed.
- Re-issuing charges in terminal states produced uniqueness violations or nondeterministic behavior.
- Webhook parsing and flood exposure: RSA verification is CPU-heavy. Without body-size caps and ingress throttling, an attacker can force repeated expensive work.
- PII leakage in quarantined webhook records and error paths that leaked correlation IDs or DB internals.
- Reconciliation gaps: provider events sometimes carried only endToEndId (provider key) instead of our correlation_id. Disputes and reversals could be left unlinked.
- Idempotency and retry paths: persisting pending states before calling provider APIs avoided duplicate calls but required deterministic retry/reconciliation when provider calls failed.

Key fixes and concrete changes
Below are the most important fixes, with minimal example code and why we made each change.

1) True durability boundary: commit() before remote calls
Problem: flush() is not durable. If we crashed after flush() but before the provider call, local state didn’t reflect whether the provider operation happened.

Fix: commit local state before making external calls. That leaves a deterministic reconciliation point: if the provider operation may have been created and we crash, reconciliation jobs and webhook handlers can continue from durable local state.

Example:
```python
# before: flush() -> remote call (risky)
session.add(payment)
session.flush()
# call provider API ...

# after: commit() -> remote call (durable)
session.add(payment)
session.commit()
# call provider API ...
```

2) Serialize payment mutation paths with SELECT ... FOR UPDATE
Problem: check-then-create allowed duplicate external calls and IntegrityError races.

Fix: lock the payment row during mutation flows (create_service_charge, release_to_provider, refund_charge, webhook handlers that mutate payment). This prevents two requests from observing the same stale status and both initiating the same external operation.

Example:
```python
with db.begin():
    payment = (
        db.query(Payment)
          .filter_by(id=payment_id)
          .with_for_update()
          .one()
    )
    payment.status = PaymentStatus.release_pending
    db.commit()
```

3) Idempotency and deterministic re-issuance
Problem: create_service_charge short-circuited only for active/completed payments and allowed nondeterministic behavior for other states.

Fix: make create_service_charge deterministic for all states:
- If a payment is pending, return 409 (explicit conflict).
- If a payment is terminal (expired/refunded/released), only reissue a charge when the state allows it. For example, generate a new correlation_id and create a new charge for expired payments, but forbid reissuance for refunded or released states. This avoids unique-constraint surprises and tells callers whether to retry or handle an error.

4) Webhook security: body-size caps, pre-verify throttling, and a global error handler
Problem: webhooks could be used to force expensive cryptographic verification or to send very large payloads; error paths exposed internals.

Fixes:
- Added a 64 KB body-size limit; oversized bodies are rejected with 413 before crypto verification.
- Added an in-memory IP-based token-bucket limiter at webhook ingress so heavy bursts get 429 early.
- Robust signature verification (RSA SHA-256) and HMAC where appropriate.
- Global exception handler to return a generic 500 and avoid leaking correlation IDs, DB schema, or stack traces.

Example webhook ingress:
```python
@app.post("/webhooks/woovi")
async def woovi_webhook(request: Request):
    body = await request.body()
    if len(body) > 64 * 1024:
        raise HTTPException(status_code=413, detail="payload too large")
    if not rate_limiter.allow(request.client.host):
        raise HTTPException(status_code=429)
    # verify signature and continue...
```

5) Dispute and reversal linkage via endToEndId
Problem: provider webhooks sometimes referenced the provider transaction key (endToEndId) instead of our correlation_id.

Fix: extract endToEndId from webhooks and store provider keys on local entities. Handlers fall back to provider IDs when correlation_id is absent, so reversals and disputes can be linked even if the provider didn’t include our token.

6) PII sanitization & quarantines
Problem: webhook quarantine records contained raw event payloads with PII.

Fix: centralize sanitize_event_data into woovi/sanitize.py and apply it whenever we save quarantined events or log errors. Quarantined records now contain only fields we intentionally keep for troubleshooting.

7) Decomposition for maintainability
Problem: PixPaymentService became a 1,542-line monolith and was hard to review.

Fix: split the service into focused modules:
- woovi/sanitize.py — PII sanitization
- woovi/transitions.py — payment state machine
- woovi/charges.py — charge operations
- woovi/releases.py — payout operations
- woovi/refunds.py — refund operations

The PixPaymentService is now a ~228-line facade that composes those modules via mixins. Reviews and targeted fixes are much faster.

Facade example:
```python
class PixPaymentService(ChargesMixin, ReleasesMixin, RefundsMixin, Transitions, SanitizeMixin):
    def __init__(self, client, db):
        self.client = client
        self.db = db
```

Testing and metrics
We leaned on tests to exercise race conditions and message flows.

- Initial Woovi integration: ~35 targeted tests.
- After the first fixes: test-suite reported 602 passing tests.
- After the module split: 1,048 tests passing.
- Final cleanup and added authz/availability tests: 1,058 passing tests with 0 warnings.

Regression tests added:
- Woovi-unavailable 503 tests (simulate provider outage; verify service returns 503 and doesn’t create inconsistent local state).
- Webhook flood and oversized body tests (assert 429 / 413).
- Admin charge creation bypass and admin subscription GET tests.
- Concurrency tests that assert SELECT FOR UPDATE prevents duplicate external calls.

Operational considerations and lessons learned
- Commit before the remote call. It provides a stable inspection point for reconciliation.
- Lock mutations with FOR UPDATE. Race-driven duplicate external calls are a common payment bug.
- Make re-creation behavior explicit for each payment state. Either return a deterministic conflict or create a new entity.
- Protect webhook ingress. Reject large or bursty traffic before doing heavy crypto work.
- Store provider identifiers (endToEndId or equivalent) so you can link provider-initiated events that don’t include your correlation token.
- Keep services small and composable. A faceted design (charges, refunds, releases, transitions, sanitize) made correctness fixes easier.
- Tests are the contract. Add regression tests for timeouts, reversed payouts, and dispute events that only include provider IDs.

Where we are now
The Woovi integration supports charge creation with escrow (split sub-accounts), refunds, provider release (Pix Out), webhooks with robust signature validation and flood controls, and deterministic recovery for pending states. The code is organized into clear modules, and a broad test suite guards regressions. Several P0/P1 issues found after the initial implementation were fixed.

If you’re integrating a payment provider
- Start with durable local state and a reconciliation plan.
- Treat webhooks as the ground truth for provider-side events, but make them linkable to your entities.
- Expect races — design idempotency and serializability into mutation paths.
- Make the service easy to reason about: small modules, an explicit state machine, and clear API contracts.

Appendix: pointers to the repo
- woovi client & service: src/find_workers/woovi/
- webhook endpoint: src/find_workers/api/webhooks.py
- sanitize & transitions: src/find_workers/woovi/sanitize.py, src/find_workers/woovi/transitions.py
- tests: tests/test_woovi_*.py

If you want, I can follow up with a concrete walkthrough of the payment state machine, show how we model escrow splits, or extract a short test that demonstrates the FOR UPDATE lock preventing duplicate charge creation.
