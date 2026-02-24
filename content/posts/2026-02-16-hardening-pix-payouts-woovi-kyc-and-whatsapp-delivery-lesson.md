+++
date = '2026-02-16T18:15:01-03:00'
draft = false
title = 'Hardening Pix payouts, Woovi KYC and WhatsApp delivery: lessons from a week of fixes'
+++

What happens when your payments provider, webhook surface, and a mobile messaging channel are all slightly off-spec — and those gaps cause money misroutes, silent failures, or incorrect accounting? Over the 2026-W08 sprint we fixed a cluster of production issues in Find Workers: Woovi (OpenPix) integration bugs, missing payout integrity checks, platform-fee rounding errors, KYC onboarding gaps, WhatsApp API changes, delivery-state persistence, and a consumer dispute API.

This post explains the problems we found, the engineering fixes we shipped, and the tests added. I focus on code-level changes and the production risks they reduce. This is practical if you integrate Pix, any two-step payout API, or rely on webhook-driven state machines.

## The problem space (what went wrong)

We operate with three risky surfaces:

- Payments via Woovi (OpenPix): charges, split sub-accounts, Pix Out (payouts), webhooks
- Provider onboarding (KYC/KYB) via Woovi BaaS account-register APIs
- Notifications via WhatsApp Business API

During the sprint we observed several issues that could lead to lost money, annoyed customers, or regulatory exposure:

1. Payout flow skipped Woovi's approve step. We called create_payment but not approve_payment; local state moved to release_pending while the remote payment stayed in a created/requested state. That leaves room for false assumptions about funds in transit and payouts that appear "in progress" but never complete.

2. Release path trusted caller-supplied payout destinations instead of the stored worker payout data. An attacker who can update a worker profile could redirect payouts.

3. Platform fee calculation used int() truncation, producing systematic under-collection on cent-precision currencies (centavos).

4. Woovi client used wrong HTTP verbs and expected payload shapes for several endpoints (subscription cancel used the wrong verb, application creation returned an unexpected shape, rejected reason field read from the wrong path).

5. WhatsApp API moved to v24.0 and webhook delivery statuses changed; we weren’t persisting delivery state, which prevents reconciliation and retries.

6. No consumer-facing dispute API to allow customers to trigger a refund/dispute flow from our backend.

7. Webhook handler did not validate Woovi company/account context in all cases, risking processing webhooks intended for other merchants.

These problems quietly cause issues: misrouted payouts, under-collected fees, uncertainty about whether a QR code was delivered, and workers stuck in KYC limbo.

## What we changed (overview)

Across four commits we implemented targeted fixes:

- Added approve_payment to the Woovi client and made release flow call create_payment + approve_payment before marking local state as release_pending.
- Validated approve_payment response destination against the stored worker pix_key. On mismatch we transition to a failure/suspicious state and persist metadata for forensics.
- Made the release endpoint admin-only and made it ignore caller-provided destinations; the server always uses stored worker payout info.
- Fixed Woovi client bugs (subscription cancel verb, create_application payload handling, reading rejected reason from accountRegister).
- Implemented Woovi account-register onboarding and application creation for KYC/sub-account setup.
- Replaced int() truncation with round() for platform fee calculation to avoid systematic under-collection.
- Upgraded the WhatsApp client to v24.0 and persisted delivery statuses from webhooks.
- Implemented a consumer dispute API and wired it to Woovi refund/dispute flows.
- Added tests covering the above; commit messages include test counts as snapshots (for example: 1,916 / 1,862 / 1,537 / 3,599).

Below are the most impactful code-level changes and why they matter.

## Two-step Woovi payout: create + approve + validate destination

The core mistake was trusting a single create_payment call. Woovi requires a second approve call that confirms the payout instruction and returns the actual destination used. The approve response can contain platform-resolved destination details; we must assert they match the intended worker pix_key.

We added approve_payment to the Woovi client and updated the release flow to:

1. create_payment
2. approve_payment using the same correlation_id
3. check approve response destination matches worker.pix_key
4. persist remote metadata and only then mark local payment state as release_pending

Example validation (simplified):

```python
resp = woovi_client.approve_payment(correlation_id=correlation_id)
remote_dest = resp["destination"]["pixKey"]
if remote_dest != worker.pix_key:
    # persist suspicious metadata and transition to failure state
    audit.log("release_destination_mismatch", payment_id=payment.id, remote=resp)
    raise ReleaseDestinationMismatch("approve response destination mismatch")
# otherwise continue to persist release_pending and schedule Pix Out monitoring
```

Why this matters: if the approve response overrides the requested pix_key (misconfigured provider, bank routing, or malicious change), we detect it before releasing funds or marking them as in-flight.

We also locked down the release endpoint: only admins can trigger a release and the endpoint ignores any destination supplied by the caller — the backend always reads the stored worker payout information.

## Platform fee rounding: round(), not int()

Cent-precision currencies require correct rounding. Previously we used:

```python
platform_fee = int(total_cents * platform_fee_pct)
```

int() truncates and causes systematic under-collection. We changed the calculation to:

```python
platform_fee = round(total_cents * platform_fee_pct)
```

That change prevents revenue loss from repeated fractional-cent truncation.

## Woovi client fixes: verbs, payloads, and fields

Tests revealed several API contract mismatches. We corrected them:

- Use PUT for subscription cancel per Woovi docs.
- Handle create_application returning an "application" object.
- Read rejected reasons from accountRegister.* where Woovi places them.

These are small, boring fixes, but wrong verbs or incorrect parsing cause silent failures or incomplete state transitions.

## KYC onboarding (account-register, sub-account + app creation)

We implemented worker onboarding using Woovi account-register: submit officialName, taxID (CPF/CNPJ), billingAddress, and representative documents, then persist the returned accountId. We also create an application key for the sub-account so we can call sub-account-specific APIs.

This makes Woovi the source of truth for KYC flow and lets us handle ACCOUNT_REGISTER_APPROVED and ACCOUNT_REGISTER_REJECTED webhooks to update worker verification_status. When Woovi emits ACCOUNT_REGISTER_PENDING we persist requestDocuments and requestReason so workers know what to resubmit.

## WhatsApp v24.0 upgrade & delivery status persistence

WhatsApp v24.0 changed message delivery and template handling. We updated the client and webhook handling and started persisting message delivery statuses (delivered, read, failed) into our whatsapp model. Persisting status enables:

- Reconciliation — confirm whether a QR code or notification reached the recipient
- Retries on transient failures
- Simple analytics on delivery and read rates

Example webhook guard (context validation):

```python
if payload.get("company", {}).get("id") != settings.WHATSAPP_COMPANY_ID:
    raise HTTPException(status_code=403, detail="webhook company mismatch")
# persist delivery status
db_whatsapp_message.update_status(message_id, payload["status"])
```

## Consumer dispute API

We added a consumer-facing dispute endpoint so customers can flag issues and trigger refund/dispute orchestration. The endpoint verifies ownership and then either calls Woovi refunds or creates an internal review depending on payment state.

Example route skeleton:

```python
@router.post("/payments/{payment_id}/dispute")
def consumer_dispute(payment_id: UUID, payload: DisputeRequest):
    payment = payments.get(payment_id)
    authorize_user_owns_payment(current_user, payment)
    dispute = payments.create_consumer_dispute(payment, payload.reason)
    # call woovi.refunds or escalate to ops depending on state
    return {"dispute_id": dispute.id, "status": dispute.status}
```

We added tests (tests/test_dispute_api.py) to validate the handler.

## Webhook context validation & signature checks

To prevent cross-merchant webhook replay, the Woovi webhook handler now validates company_id, account_id, and environment fields against configured values and verifies signatures. We added tests that assert a 403 when any of those fields mismatch. Signatures are necessary but not sufficient when the same signing key can be used across merchants.

## Tests and safety nets

Every change included tests. We expanded coverage around:

- approve_payment + release flow behaviors (success, destination mismatch)
- Woovi client verbs and payload parsing
- webhook context validation (company/account/env mismatch)
- WhatsApp delivery webhook behavior
- dispute API

If you integrate a third-party payment provider, add tests that simulate both the happy path and surprising-but-valid provider responses (different destination, missing fields, delayed webhooks). Persist remote metadata — it is invaluable for post-mortem and reconciliation.

## Lessons and recommendations

- Treat two-step flows as authoritative: many payout APIs split create and approve/execute. Treat approval as final and validate it against your intent.
- Never trust caller-supplied payout destinations at release time. Use stored, step-up-protected payout data and require authorization.
- Monetary math must match business intent. Truncation will drain revenue over time.
- Webhooks are semi-trusted: validate merchant context (merchant id, account id, environment) in addition to signatures.
- Persist remote metadata for every external transition; it makes reconciliation and auditing possible.
- Keep tests that simulate normal and edge-case provider behavior — including wrong-but-signature-valid webhooks.

If you work with Pix, OpenPix/Woovi, or any split-pay provider, these are common footguns. The fixes we shipped are pragmatic, small, and covered by tests; they prevent high-impact failures. The code changes are in src/find_workers/woovi/*, src/find_workers/api/payments.py, src/find_workers/whatsapp/* and related tests.

I can extract the relevant pull-request diffs into a checklist you can run against your own integrations (approve-step, destination-validation, rounding check, webhook context validation, WhatsApp delivery persistence).
