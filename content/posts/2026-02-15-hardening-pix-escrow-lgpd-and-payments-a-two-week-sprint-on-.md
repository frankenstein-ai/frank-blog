+++
date = '2026-02-15T18:23:55-03:00'
draft = false
title = 'Hardening Pix Escrow, LGPD, and Payments: A two-week sprint on Find Workers'
+++

Problem — what can go wrong when your marketplace handles money, Brazilian IDs (CPF), and WhatsApp messages?

Find Workers is an MCP-first backend that connects Brazilian consumers to home-service providers and routes Pix payments via Woovi (OpenPix). Over the sprint we focused on the parts that make that stack useful and safe in production: payment integrity (escrow, releases, refunds), LGPD compliance (consent gating and erasure), webhook resilience, secret handling, and a migration from floats to integer centavos to remove precision bugs.

This post walks through the technical changes, why they mattered, and concrete examples you can reuse in your own marketplace or payments integration.

## Summary of the work

- 48 commits across cross-cutting refactors and security hardening.
- Monetary model migrated from floats to integer centavos across models, schemas, services, seed data, MCP integration, and tests. Alembic migration added.
- LGPD changes: require explicit payment_data_sharing consent before sending CPF to Woovi; implement data erasure and phone hashing; revoke tokens on erasure.
- Payments: Pix state machine, webhook quarantine and dedup, handling MOVEMENT and REFUND events, and removal of premature provider splits to prevent double-disbursement.
- Secrets and keys: Pix keys encrypted at rest (Fernet); require ENCRYPTION_KEY in production and validate Fernet format at startup.
- Webhooks and rate limits: Redis-backed dedup and rate limits with an in-memory fallback; pre-verify body size caps and HMAC/signature required in production.
- Audit and pagination: add AuditLog and switch to COUNT() OVER() window functions for snapshot-consistent pagination.
- Reliability/security fixes: OTP HMAC key derived via HKDF, atomic OTP/refresh operations in Redis, JWT algorithm allowlist, deterministic ordering tie-breakers, and N+1 query fixes in search.
- Test coverage: test suite stabilized at 1,057 passing tests after the changes.

Below I unpack the most impactful changes and show small code snippets from the diffs.

## Why centavos (integers) — concrete problem and fix

Problem: using floats for BRL amounts produced precision errors and fragile conversion boundaries around the payments API. Those bugs can cause accounting drift or invalid requests to Woovi.

What we did
- Converted all monetary fields to integer centavos everywhere: models, Pydantic schemas, services, seed data, MCP outputs, WhatsApp templates, and tests.
- Added an Alembic migration to persist the DB changes.

Example schema change (illustrative):

```python
# before (float)
class PaymentSchema(BaseModel):
    amount_brl: float

# after (integer centavos)
class PaymentSchema(BaseModel):
    amount_centavos: int  # e.g., R$12.34 -> 1234
```

Why this helps: integers remove rounding ambiguity, make comparisons and sums exact, and map directly to provider APIs that expect centavos.

Metric: the migration touched the entire test suite; we ran and fixed tests until 1,057 passed.

## LGPD: require consent before sharing CPF with payment provider

Problem: CPFs are sensitive personal data in Brazil. Sending a CPF to a payment provider without consent violates LGPD and raises legal risk.

What we did
- Only include consumer.taxID (CPF) in Woovi charge payload when the user has an active payment_data_sharing consent.
- Implemented data-erasure flows that anonymize PII across users, workers, WhatsApp messages, bookings, payments, reviews, and consents.
- Store phone numbers in WhatsApp logs as hashes to preserve auditability while minimizing PII exposure.
- Revoke JWT tokens when erasure completes to avoid stale credentials remaining valid.

Illustrative code (consent gate):

```python
customer = {"name": consumer.full_name}
if consumer.has_consent("payment_data_sharing") and consumer.cpf:
    customer["taxID"] = consumer.cpf  # only when consented
```

## Payments: state machine, webhook handling, and escrow safety

Problems we fixed
- Webhook races that could cause double-credit on refunds or releases.
- Missing handlers for movement and refund webhooks, leaving payouts stuck.
- Double-disbursement risk from mixing immediate splits at charge settlement with later Pix Out releases.
