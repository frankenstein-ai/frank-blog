+++
date = '2026-02-22T14:31:40-03:00'
draft = false
title = 'Bringing parity, reliability and observability to an MCP-native backend: WhatsApp dispatch, scheduler tasks, and infra hardening'
+++

Find Workers is an MCP-first platform that lets assistants (Claude, ChatGPT, Gemini, Copilot) take real-world actions: find a professional, request a quote, schedule a visit, and create a Pix escrow charge. During week 2026-W08 we focused on three operational questions that matter when assistants, third‑party gateways and users interact:

- How do we make MCP-driven actions match the REST API exactly (including side effects like WhatsApp notifications) without slowing assistant responses?
- How do we run critical background work — payouts, disbursements and privacy deletions — reliably inside the same process?
- What operational visibility and hardening do we need so these flows are safe at scale?

This post explains the changes we made: adding WhatsApp dispatch to MCP tools, wiring an in-process scheduler for payout/disbursement/LGPD tasks, unifying notification code and metrics, adding OpenTelemetry and extracting payment logic, and several infra/security fixes (proxy trust, Redis timeouts, DB timeouts). We kept a test-first mindset throughout — the repo’s tests (3k+ during this period) stayed green at every step.

## 1) Problem: MCP tools missed REST side effects

MCP tools could change bookings in the database, but they didn’t trigger the WhatsApp templates the REST API sends (booking confirmations, worker notifications, start/complete updates). If an assistant performs an action, users must still get the same messages as when they use the website or app.

The challenge: call WhatsApp from MCP tooling without blocking assistant responses when the channel is flaky, and keep message tracking consistent with the REST API.

What we did
- Exposed a module-level _whatsapp_client in the MCP server module and set it from app.state during FastAPI startup.
- MCP tool implementations (book_service, manage_booking) now call the same notification code the REST endpoints use.
- Notification sends are best-effort: failures are logged and never allowed to block MCP responses.
- MCP notification sends reuse the same _outbound_session used by the REST side to keep message tracking consistent.

Example (pattern used to set the client)
```python
# src/find_workers/mcp_server.py (excerpt)
_whatsapp_client: Optional[WhatsAppClient] = None

# src/find_workers/main.py (lifespan/startup)
app.state.whatsapp_client = WhatsAppClient(config.whatsapp)
# expose to MCP tools
mcp_server._whatsapp_client = app.state.whatsapp_client
```

That connection ensures assistant-driven bookings trigger the same WhatsApp templates consumers expect.

## 2) Running scheduled, high-trust tasks in-process: payouts, disbursements, LGPD deletions

We needed deterministic runs for tasks that cannot be fire-and-forget:

- Release funds from escrow to providers (payout release).
- Run disbursement work (Pix Out/settlement checks).
- Execute LGPD deletion pipelines (destruction or anonymization of data after legal workflows).

Instead of a separate worker service, we added a simple in-process asyncio scheduler and registered the periodic tasks with it. The scheduler runs during the app lifespan; tasks are idempotent and safe to run concurrently. We added tests to ensure the scheduler starts, schedules tasks, and tasks behave correctly — the tests stayed green as we landed the feature.

High‑level scheduler registration (concept)
```python
# src/find_workers/tasks/scheduler.py (pattern)
async def _periodic(loop, interval, coro, *args):
    while True:
        try:
            await coro(*args)
        except Exception:
            logger.exception("scheduled_task_failed", task=coro.__name__)
        await asyncio.sleep(interval)
```

Why in-process? Our deployment model is one container per service with autoscaling at the platform level. Running the scheduler in-process reduced operational complexity while keeping tasks resilient (retries + idempotency), and unit tests ensured deterministic behavior. At larger scale we would move to a distributed job runner.

## 3) Unifying notifications & adding observability for failures

We refactored notifications to centralize template sends and reduce PII leakage and payload mistakes.

- send_template_message is now the single call-site for template sends.
- Callers pass a template_key and an ordered parameter list instead of building component payloads in multiple places.
- Logs mask PII (mask_phone), and missing button payloads were fixed.

Example: worker approved notification (after refactor)
```python
# src/find_workers/notifications.py
await send_template_message(
    client=client,
    to=worker_phone,
    template_key="worker_approved",
    parameters=[worker_name],
)
```

We added a Prometheus counter for permanently failed notifications. A single counter makes it easier to spot channel outages or template issues in production. This metric helped us find flaky WhatsApp sends and evaluate retry effectiveness.

## 4) Observability (OpenTelemetry) and extracting payment logic

To measure latency across assistant calls, external HTTP calls (Woovi, WhatsApp), and DB operations, we added optional OpenTelemetry instrumentation:

- New otel.py enables FastAPI and httpx instrumentation.
- Settings control tracing (enable_tracing, otel_service_name, otel_exporter_endpoint) so tracing can be toggled per environment.
- We wire tracing during the app lifespan so spans cover incoming API/MCP requests and external calls.

We also moved payments logic out of route handlers into src/find_workers/payments/service.py and introduced a PaymentServiceError. This separation makes unit tests simpler and gives a clear span boundary for tracing payment flows.

Snippet (conceptual)
```python
# src/find_workers/payments/service.py
class PaymentServiceError(Exception):
    pass

async def create_service_charge(db, booking_id, amount):
    # business logic moved from handler to service
    ...
```

The team added ~20 OTel tests and kept the suite green.

## 5) Infra & security hardening

Practical fixes improved robustness:

- Added Redis socket timeouts and DB pool/statement timeouts so stuck connections fail fast instead of blocking the process.
- Enforced proxy trust config in production and safely handle invalid CIDRs to prevent host header abuse and incorrect client-IP attribution.
- Fixed a race in the Woovi client that mutated a shared Authorization header by switching to per-request headers.
- Enforced step-up authentication for high-sensitivity LGPD endpoints to reduce risk if tokens are compromised.

These changes reduce blast radius from misconfigurations and make the system more predictable under load.

## 6) Test metrics & engineering discipline

We kept a heavy focus on tests and CI:

- Mounting the MCP server at /mcp added four tests and kept 1,655 tests passing at that commit.
- OpenTelemetry and payments extraction added ~20 OTel tests; the suite reported ~2,929 tests passing after that work.
- The WhatsApp dispatch change landed with a passing suite of 3,489 tests.

Keeping the test suite green while touching auth, payments, notifications and MCP surface was critical for safe deployments.

## 7) Lessons & guidelines for other teams

- When external agents perform actions, ensure parity of user-facing side effects (notifications, receipts). Small mismatches cause confusing UX and bookkeeping gaps.
- Make channel sends best-effort from interactive flows. Don’t block MCP/assistant responses on brittle third-party channels — surface reliable metrics and logs so you can act on failures.
- Centralize notification templates and PII masking. Centralization simplifies audits and compliance.
- For critical scheduled business processes (payouts, legal deletions), design idempotent tasks and run deterministic periodic workers. In-process schedulers work for single-instance deployments; migrate to a distributed job system for scale.
- Add tracing early. When services and gateways (WhatsApp, Woovi/OpenPix, DB) are involved, distributed traces reveal where latency and errors originate.
- Tighten infra defaults (timeouts, proxy trust) before going public — these are common sources of subtle outages or security issues.

## 8) Where to look in the repo

- MCP wiring and _whatsapp_client: src/find_workers/mcp_server.py, src/find_workers/main.py  
- Notifications and templates: src/find_workers/notifications.py, src/find_workers/whatsapp/  
- Scheduler tasks: src/find_workers/tasks/scheduler.py  
- Payments service extraction: src/find_workers/payments/service.py  
- OpenTelemetry: src/find_workers/otel.py  
- Redis/DB hardening: src/find_workers/auth/redis.py, src/find_workers/db.py, src/find_workers/config.py

Conclusion

This week’s work narrowed the gap between assistant-driven actions and the REST API experience while making the system safer and easier to operate. Assistants can now book and manage bookings with the same WhatsApp confirmations users expect. Important background processes (payouts, disbursements, privacy deletions) run on a monitored schedule. Tracing and payment encapsulation make debugging simpler, and infra hardening reduces risk in public deployments. We shipped all changes with a strong emphasis on tests and metrics — the fastest path to confident production rollouts.
