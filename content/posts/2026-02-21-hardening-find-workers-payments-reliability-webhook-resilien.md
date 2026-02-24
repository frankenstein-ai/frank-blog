+++
date = '2026-02-21T22:41:13-03:00'
draft = false
title = 'Hardening Find Workers: payments reliability, webhook resilience, and the MCP-first pivot (2026‑W08)'
+++

When your platform touches three hard problems at once—real money through an external Pix provider, brittle asynchronous webhooks and idempotency, and a shift from WhatsApp-first to an MCP/AI-agent-first architecture—small races and edge cases become high-risk problems. In week 2026‑W08 we shipped a focused set of fixes across the Find Workers backend to reduce that risk.

Below I explain what we changed, why each change works, and the engineering patterns that made the fixes safe and testable. If your system calls external payment providers or relies on webhooks and assistants, these changes should map directly to your stack.

## Summary: what we shipped this week

Highlights:

- Payments reliability and safety
  - Validate refund amounts before calling Woovi to prevent over-refunds.
  - Move Woovi outbound cancellation to after session.commit() so DB row locks are not held during slow provider calls.
  - Mark subscriptions failed when Woovi returns provider errors; deduplicate subscription webhooks with a DB unique index.
  - Block auto-release for disputed or payer-mismatch payments and handle payout webhook timing mismatches gracefully.

- Webhooks and dedup
  - Use the correct app.state Redis key for webhook dedup.
  - Add TTL-based eviction for the in-memory dedup fallback so restarts or memory pressure don’t let the structure grow unbounded.
  - Clear dedup tokens for quarantined events so replays after fixes are accepted.

- Resilience improvements
  - Add a circuit breaker and retry wrapper for Woovi and WhatsApp HTTP clients to avoid cascading failures from slow providers.
  - Verify DB and Redis connectivity during FastAPI lifespan startup to fail fast on broken deployments.

- MCP-first and public landing
  - Update README, docs, and landing copy to reflect an MCP-first model where AI assistants are the primary interface and WhatsApp is a notifications channel.
  - Add MCP resource templates (bookings:// and payments://) and enforce subscription checks for worker discovery in MCP flows.
  - Add a public /v1/public/stats endpoint cached in memory for 5 minutes to drive landing page metrics.

- Security and ops
  - Add 30s JWT clock-skew leeway, fix CORS for PUT, implement several LGPD and audit hardenings, and file roughly 30 production-readiness issues after a Codex security audit.

Many commits include unit and integration tests. Example: the new stats endpoint comes with 8 tests; overall test-suite sizes mentioned in commits are in the 3k–3.7k range.

## The problem: payments + webhooks are a distributed-systems minefield

Common failure modes we saw:

- Long external calls holding DB row locks. When an outbound HTTP call blocks while a transaction keeps a FOR UPDATE lock, concurrent webhooks or other requests can deadlock or race. That produces inconsistent state or duplicate side effects.
- Idempotency and dedup gaps. In-memory dedup caches were node-local and lost on restart. Redis wiring was inconsistent. Quarantined events left dedup tokens set so replays were ignored.
- Missing precondition checks before provider calls. Refund APIs could call Woovi with amounts larger than the remaining refundable balance.
- Payer-mismatch and dispute events that should block auto-release were sometimes released automatically.

If money moves through your system, these failures are existential: double refunds, double payouts, or permanently locked funds.

## Concrete fixes (and why they work)

Below are the main fixes and the engineering ideas behind them.

### 1) Validate refund amount before calling provider

We added application-level checks so we never call Woovi to refund more than the remaining refundable amount. Check local state first and return a 400/409 rather than letting an external call proceed.

Example (conceptual):

```python
# pseudocode: src/find_workers/payments/service.py
def request_refund(session, payment_id, amount_centavos):
    payment = session.get(Payment, payment_id, with_for_update=True)
    remaining = payment.amount_centavos - payment.total_refunded_centavos
    if amount_centavos > remaining:
        raise InvalidRequest("refund amount exceeds remaining refundable")
    # persist a pre-allocated refund record (idempotency key) first...
    # then call Woovi using stable correlation id
```

Why this works: it prevents accidental double refunds and enforces a deterministic local invariant before any external call.

### 2) Release DB locks before slow outbound HTTP calls

Subscription cancellation originally called Woovi while still inside the DB transaction, holding locks. We now persist a local intent or tombstone, commit the transaction, and perform the outbound call afterwards (with retries, a circuit breaker, and compensating actions where needed).

Pattern:

```python
# conceptual change in src/find_workers/api/subscriptions.py
with session.begin():
    service.mark_subscription_cancelled_local(subscription_id)
# DB commit done, locks released
# Now call Woovi.cancel_subscription(subscription.woovi_subscription_id)
```

Why this works: it keeps slow provider calls off the critical path for DB writes and reduces the chance that webhook handlers will run while rows are locked.

### 3) Idempotency, DB-level dedup, and retry-safe webhook handling

- Add a DB unique index on SubscriptionPayment.woovi_payment_id so webhook inserts are idempotent.
- For quote dedup, SELECT ... FOR UPDATE avoids TOCTOU inserts and we add a partial unique index on (consumer_id, worker_id) WHERE status IN ('requested', 'quoted') so expired quotes don't block new ones.
- Webhook dedup now uses Redis (app.state.redis_client) with a TTL-backed in-memory fallback so dedup survives process lifetime and does not grow without bounds.
- Clear dedup tokens for quarantined events so replays after fixes are accepted.

Why this works: move invariants to durable storage (DB/Redis) instead of relying on node-local heuristics. Durable state is essential for correctness in retries and multi-instance deployments.

### 4) Circuit breaker and HTTP retry for external clients

We added a circuit breaker for Woovi and WhatsApp clients (src/find_workers/http_retry.py). The breaker opens on repeated failures or high latency, short-circuits further calls, and returns controlled failures to callers.

Example usage (conceptual):

```python
from find_workers.http_retry import CircuitBreaker, http_retry

woovi_client = WooviClient(...)
woovi_client.request = CircuitBreaker(...)(woovi_client.request)
```

Why this works: it prevents one slow or failing provider from dragging down the whole app and reduces tail latency.

### 5) Payer-mismatch and dispute gating

We persist a payer_mismatch flag when Woovi signals mismatched payer data. release_to_provider now skips payments with payer_mismatch or with dispute_status in ('open', 'under_review'), so money is not released automatically in risky cases.

Why this works: it enforces a conservative policy on risk events and forces explicit human review before funds move.

## MCP-first wiring and public metrics

We moved the product to an MCP-first operational model:

- Docs and landing copy now treat AI assistants as the primary interface, with WhatsApp serving mainly for notifications and optional runtime configuration.
- The MCP server gained bookings:// and payments:// resource templates so assistants can construct structured booking and payment actions.
- Worker search in MCP flows enforces active subscription checks so assistant-discovered workers match public availability rules.
- Added /v1/public/stats (cached for 5 minutes) so the landing page and admin UIs show real signals instead of hardcoded numbers.

Sample payload (conceptual):

```json
{
  "completed_bookings": 12480,
  "satisfaction_pct": 94.3,
  "acceptance_rate_pct": 82.5,
  "active_workers": 1430,
  "coverage_region_count": 28,
  "escrow_released_cents": 3450000,
  "return_rate_pct": 6.1,
  "automation_rate_pct": 71.2,
  "recent_service_descriptions": ["Tubo estourou, vazamento", "Quadro elétrico disparando"],
  "top_reviews": [{"worker_id": "...", "rating": 5, "summary": "Resolvido rápido!"}]
}
```

## Operational hardening and developer ergonomics

- FastAPI lifespan hook checks DB and Redis connectivity at startup so bad deployments fail quickly.
- Add 30s JWT clock-skew leeway to reduce token verification flakiness across distributed clocks.
- Allow PUT in CORS so browser clients can update availability endpoints.
- LGPD and audit changes: add DSAR audit trails, improve export behavior, and block erasure when in-flight payments could cause fund lockups.

## Tests, telemetry and the audit backlog

This week produced many small safety wins covered by tests. Commits report test counts and several changes include unit and integration tests. We ran a Codex-driven security review and filed roughly 30 production-readiness issues covering CSP, rate limiting, OTP storage, and other items. The audit produced a concrete backlog rather than vague TODOs.

Key lesson: rigorous tests and a short security audit turn high-level risks into an actionable list.

## Lessons and next steps

What worked:

- Prefer durable preconditions (DB checks, partial unique indexes, Redis dedup) over in-memory heuristics.
- Release DB locks before outbound calls; persist local intent or use an outbox pattern and do remote calls after commit.
- Treat webhooks as part of the contract: design idempotency keys, durable dedup, and quarantine/replay strategies from the start.
- Circuit-breaker plus guarded fallbacks improve availability during partial provider outages.

Next priorities already in motion:

- Add a runtime trigger for worker embedding refresh to keep semantic search quality current.
- Finish remaining P1/P2 items from the Codex audit (rate limiting, CSP, OTP storage hardening).
- Wire landing page telemetry to the signals we created this week.

We chose a set of small, testable changes—indexes, locking discipline, precondition checks, circuit breakers, and a few config/UX updates—that together raise the bar on correctness for real-money flows without a full rewrite.

If you’d like, I can extract a short checklist (DB indexes, webhook dedup recipe, circuit-breaker pattern, outbox pattern) you can apply to your projects.
