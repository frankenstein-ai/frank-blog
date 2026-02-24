+++
date = '2026-02-22T20:52:49-03:00'
draft = false
title = 'Hardening the Admin Surface: PII, Audit, Webhook Quarantine, and Safe Payments'
+++

What does a production admin surface for a payments + AI marketplace need to be safe, auditable, and operable? During week 2026-W08 we focused on that question for Find Workers. We tightened privacy controls, added auditability, made webhook handling robust, and removed race conditions around money flows. The result is a set of backend changes: new admin endpoints, centralized PII redaction, webhook quarantine management, step-up auth for sensitive operations, and deterministic pagination. Together these make the platform safer and easier to run.

Below I describe the problems we found, the concrete changes we made, and practical takeaways you can apply to your systems.

## Problems we were solving

- Admin pages needed to show sensitive fields (phone numbers, WhatsApp names, CPF, Pix keys) but there was no consistent audit trail for who accessed PII and no standard redaction for logs.
- Payment provider webhooks (Woovi/OpenPix) sometimes failed validation or carried sensitive payloads. Failed webhooks were being dropped instead of giving ops a place to intervene.
- Refunds and other money operations had TOCTOU and concurrency risks when multiple workers or processes acted on the same payment rows.
- Pagination for admin reports could show rows that “jumped” between pages when ordering only by timestamp (ties caused instability).
- Several admin actions are high-risk (release funds, revoke MCP keys, run LGPD deletions) and needed step-up authentication and better IP auditing.

Our goals were simple: protect personal data consistently, and make high-sensitivity operations deterministic and auditable.

## What we changed (high level)

- Centralized PII redaction in logging with a structlog processor that replaces 16 sensitive fields with a redaction token.
- Masked PII in admin API responses (phone numbers masked, WhatsApp name reduced to first name).
- Added structured audit events for every admin action that touches PII (admin_pii_access) and added client IPs to 13 existing audit records.
- Built webhook quarantine management: list, inspect, retry, and release quarantined webhook events via admin endpoints.
- Made pagination deterministic by adding a secondary tie-breaker (id) to admin report queries.
- Introduced step-up authentication guards for high-risk admin flows and added pessimistic locks (.with_for_update()) to payment rows used in refunds, payouts, disputes, and subscription operations.
- Exposed operational metrics and retry endpoints for WhatsApp failures (Prometheus counter whatsapp_message_failed_total).
- Added an LGPD deletion audit record so deletions are traceable.

We added admin routes, tests, logging processors, and DB query hardening. Tests increased as features landed (commits reported 3,366 → 3,507 passing tests).

## Concrete examples

### 1) Masking and PII access audit

When an admin lists or inspects users, the API now masks data and emits a structured audit event that records what PII was accessed and by whom.

Before:
```py
# admin/users.py (old)
user_repr = {
    "id": user.id,
    "phone_number": user.phone_number,
    "whatsapp_name": user.whatsapp_name,
}
```

After:
```py
# admin/users.py (new)
from find_workers.utils import mask_phone

user_repr = {
    "id": user.id,
    "phone_number": mask_phone(user.phone_number),            # 11->xxx-xxx-1234
    "whatsapp_name": user.whatsapp_name.split(" ", 1)[0],     # first name only
}

# structured audit
audit.info("admin_pii_access", actor_id=admin.id, target_user_id=user.id,
           fields=["phone_number", "whatsapp_name"])
```

Two practical points:
- Only the minimum data is returned (masked phone, first name).
- The explicit "admin_pii_access" event makes PII views easy to query for audits or investigations.

### 2) Centralized log redaction

Developers often log fields while debugging. Logs are a weak link if they contain PII. We added a structlog processor, redact_pii, that replaces configured field names with "<redacted>".

Simplified processor:
```py
# src/find_workers/middleware/redact.py
REDACT_FIELDS = {"phone_number", "cpf", "pix_key", "email", ...}  # 16 keys

def redact_pii(logger, method_name, event_dict):
    for k in REDACT_FIELDS:
        if k in event_dict:
            event_dict[k] = "<redacted>"
    return event_dict
```

This runs at startup so any structlog events emitted anywhere in the app get cleaned uniformly.

### 3) Webhook quarantine and operator workflow

Webhooks are external and sometimes malformed or unauthenticated. Instead of rejecting or silently dropping them, we store problematic deliveries in a quarantine table and give operators endpoints to act on those events.

New admin routes:
- GET /admin/quarantine — list quarantined webhook events (filter by reason/status)
- GET /admin/quarantine/{id} — inspect payload and headers (PII redacted)
- POST /admin/quarantine/{id}/retry — reprocess the webhook
- POST /admin/quarantine/{id}/release — mark as safe or delete

This gives operators a safe place to hold malformed or suspicious payloads and lets them retry or discard events explicitly.

### 4) Deterministic pagination: ORDER BY created_at, id

Paging could show rows moving between pages when multiple rows shared the same timestamp. The fix is simple: add a stable secondary key.

Before:
```sql
ORDER BY created_at DESC
LIMIT 100 OFFSET 100
```

After:
```sql
ORDER BY created_at DESC, id DESC
LIMIT 100 OFFSET 100
```

That prevents duplicates or missing rows when timestamps tie.

### 5) Step-up auth and pessimistic locking for money flows

Money operations now require step-up authentication (re-verified OTP or elevated admin token). We also use pessimistic locking on payment rows to avoid race conditions:

```py
payment = db.query(Payment).filter_by(id=pid).with_for_update().one()
```

We applied .with_for_update() in refunds, payouts, disputes, and subscription code paths. That prevents two processes from approving the same release or issuing duplicate refunds.

We also added require_step_up to endpoints like retry_payout, process_escrow, revoke_mcp_key, and expire_stale_charges. That narrows who can run destructive or financial actions.

### 6) Audit IPs and LGPD deletion records

We updated 13 audit events to include client IPs (from get_client_ip). LGPD deletion flows now emit a durable audit record so deletions are traceable for compliance.

## Observability and testing

- Added a Prometheus counter whatsapp_message_failed_total labeled by template to measure notification failures.
- Added tests for new admin endpoints and hardening changes. Tests ran green as each feature landed.
- Added quarantine list/retry tests to ensure failing webhook payloads are stored and can be reprocessed.

## Lessons learned and recommendations

- Audit PII access explicitly. A structured "pii_access" event beats ad-hoc log searching during a breach or DSAR.
- Centralize log redaction. One processor is easier to test and maintain than asking everyone to remember to scrub logs.
- Make pagination deterministic. Use a stable secondary key (id) when ordering by non-unique fields.
- Use DB locks for money operations. with_for_update() reduces TOCTOU surprises when multiple workers act on the same payment.
- Require step-up authentication for destructive or financial admin actions. It costs a little UX but prevents mistakes and abuse.
- Treat webhook failures as ops objects. A quarantine plus retry workflow avoids data loss and gives operators control.
- Instrument operational failures early. Failed notifications and webhook retries are high-value signals for ops.

## Next steps

- Build a small operator UI for quarantine filtering and bulk retry with safe rate controls.
- Add a dead-letter queue and scheduled retries for notifications and webhook retries.
- Consider signed audit exports for compliance requests (DSAR).

If you want to inspect the code, look at admin routes (src/find_workers/api/admin/*), the logging middleware (src/find_workers/middleware/redact.py), and the payments/refunds paths where .with_for_update() was applied. The test suite covers the new endpoints and behavior.

If you run a marketplace or any platform that stores user PII and handles money, try these patterns: centralized redaction, explicit PII audits, quarantine for external events, deterministic pagination, and pessimistic locking. They won't solve every problem, but they make the system far easier to operate and to defend.
