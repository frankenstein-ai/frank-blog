+++
date = '2026-02-15T23:32:14-03:00'
draft = false
title = 'Designing Find Workers for Brazil: WhatsApp + Pix Escrow, MCP-first, and a pragmatic stack'
+++

What problem did we set out to solve?

We want a backend that lets any MCP-capable assistant (Claude, ChatGPT, Gemini, Copilot) find and book vetted home-service professionals in Brazil, deliver the payment UX over WhatsApp, and guarantee buyers with Pix escrow. The core questions were: which payment provider and integration pattern to use for Pix escrow, how to expose that reliably to MCP agents, and how complex our runtime stack should be for an MVP.

This sprint focused on research and architecture decisions that answer those questions. We rewrote the repo for a Brazil-first product, switched payment infrastructure to Woovi (OpenPix) and documented exact endpoints and flows, removed an unnecessary task-queue layer from the docs (moving to synchronous processing for now), and added vetting and go-to-market research. Those documentation and design changes now drive implementation work and make the repository a clearer blueprint for building Pix + WhatsApp commerce.

Below I walk through the reasoning, the concrete changes we made, and practical guidance you can reuse if you’re building a similar marketplace.

## Quick snapshot — key metrics & business assumptions

- Market: Brazilian online home-services TAM projected at USD 222.4M by 2030 (CAGR ~15.9%).
- Platform signals: WhatsApp ~148M Brazilian users; Pix ~178M users.
- Business model: consumer subscription R$10/month + per-use fee; provider commission ~5%; unit-economics target LTV:CAC ≈ 15:1.
- Status: research and architecture finalized; infra and API design aligned to Woovi (OpenPix); all docs updated to reflect the pivot.

## What changed (high-level)

- Rewrote research documents for a Brazil-first product: market, user research, business model, MCP research, and technical architecture.
- Switched payment provider from Mercado Pago to Woovi (OpenPix); added a Woovi API reference with real endpoints, escrow/split model, webhooks, refunds, and subscription endpoints.
- Removed Celery/task-queue references from docs; webhooks will be processed synchronously for the MVP.
- Added vetting and client-acquisition research and a vetting/verification doc.
- Updated issue tracking to reflect closed research tasks and new implementation work tied to the Woovi integration.

Edits live in README.md and docs/ (technical-architecture.md, mcp-research.md, business-model.md, docs/woovi-api-reference.md, docs/vetting-verification-research.md).

## Why Woovi (OpenPix) and split sub-accounts for escrow

Pix is the dominant payments rail in Brazil. For a marketplace that needs escrow we require:
- create charges (Pix QR codes);
- split/holding accounts (sub-accounts) so provider funds can be retained until service completion;
- reliable webhooks to observe settlement and trigger release/refund actions;
- refund/claim APIs.

Partnering with a licensed PSP like Woovi gives these features without the time and capital needed to obtain a BCB license. We added explicit Woovi endpoints and examples in the API reference:

- Create charge: POST /api/v1/charge
- Release provider (Pix Out): POST /api/v1/payment
- Refund: POST /api/v1/charge/{correlationID}/refund
- Subscriptions: POST /api/v1/subscriptions

Webhooks: Woovi signs webhook payloads with RSA SHA-256. Verify the signature in your webhook handler before acting on the event.

## Architecture choices and trade-offs

Key decisions for the MVP:
1. No Celery/task queue — process webhooks synchronously.
2. Keep Redis for cache/sessions; FastAPI + FastMCP + Postgres (+ pgvector) remain the backbone.

Why synchronous?
- Simpler stack and faster iteration for an early MVP.
- WhatsApp and Pix webhook volume will be low initially; immediate 200 responses simplify retry behavior.
- Avoids premature complexity around distributed task idempotency, broker availability, and scaling.

Trade-offs and mitigations
- Handlers must be fast and robust. Recommendations:
  - Verify signatures first; drop invalid requests.
  - Persist an idempotency key (webhook event id) quickly to prevent replay.
  - Persist the event before launching longer work.
  - Offload heavy jobs to background threads/processes or schedule background tasks after acknowledging the webhook if scale requires it.
  - Add rate limits and circuit breakers for third-party APIs.

Example synchronous webhook handler (docs updated to match):

```python
async def whatsapp_incoming(request: Request):
    payload = await request.json()

    # Process synchronously
    for entry in payload.get("entry", []):
        for change in entry.get("changes", []):
            value = change.get("value", {})
            # verify signature, persist event id, handle message
```

## Charge → escrow → release lifecycle (concept)

Flow implemented against Woovi:

1. Consumer confirms service; platform creates a Woovi charge (POST /api/v1/charge) including split sub-account info so the provider’s share is captured.
2. Platform delivers the Pix QR on WhatsApp.
3. Woovi webhooks notify the platform when the Pix is paid; verify the RSA-SHA256 signature.
4. Platform marks payment settled and holds provider funds in the platform sub-account (escrow) until the consumer marks service complete (or an auto-release policy applies).
5. When service completes, platform calls Woovi POST /api/v1/payment (Pix Out) to release funds to the provider.
6. If needed, refunds are invoked via POST /api/v1/charge/{correlationID}/refund.

Minimal illustrative example for creating a charge:

```python
import requests

resp = requests.post(
    "https://api.woovi.com.br/api/v1/charge",
    headers={"Authorization": "Bearer <API_KEY>", "Content-Type": "application/json"},
    json={
        "amount": 150.00,
        "currency": "BRL",
        "correlation_id": "booking:12345",
        "split": [
            {"account_id": "subacct_provider_1", "amount": 142.50},
            {"account_id": "platform_fee", "amount": 7.50}
        ],
        "metadata": {"booking_id": 12345, "worker_id": 777}
    }
)
```

Exact fields and schemas are in docs/woovi-api-reference.md; treat the snippet above as a conceptual mapping.

## Documentation work that matters

- Market & user research: refocused on Brazilian personas (classes B/C, MEI providers), WhatsApp-first journeys, and Pix-native UX.
- Business model: clarified Woovi as the PSP and updated revenue assumptions (R$10/month + per-use fee, ~5% commission).
- MCP research: updated tools and JSON schemas for MCP agents to orchestrate WhatsApp and Pix flows.
- Vetting & verification: added steps for identity, background checks, and credential verification; these drive schema choices for workers and bookings.
- Compliance: documented LGPD implications for data export and erasure.

These docs directly affect what metadata we store, the sequence for release/refund logic, and compliance requirements.

## Next implementation tasks

Follow the .beads log and README. Priorities:
- Implement src/woovi/{client,service,charges,releases,refunds,webhook}. Use docs/woovi-api-reference.md as the contract.
- Implement webhook signature verification (RSA-SHA256) and idempotency via a payments events table.
- Implement WhatsApp client integration to send QR codes and template messages (src/whatsapp/client.py).
- Implement booking lifecycle in bookings/service.py to couple booking state → create charge → hold funds → release/refund.
- Add LGPD data export/erasure endpoints as required by the legal research doc.
- Add unit tests for payment transitions and webhook processing.

## Patterns & recommendations for implementers

- Idempotency: record webhook event IDs and correlation IDs immediately. Return 200 quickly if the event repeats.
- Signature verification: verify RSA-SHA256 on every webhook; drop and log invalid events.
- Keep background work minimal: synchronous handling is acceptable if handlers are bounded and events are persisted before longer tasks start.
- Instrumentation: log webhook timings, Woovi API latency, and failures. Those metrics will tell you when to add async queues.
- Tests: cover payment transitions, idempotency, and webhook processing.

## TL;DR

We pivoted the project to a Brazil-first design: documented market and personas, chose Woovi/OpenPix for Pix escrow and split sub-accounts, codified charge/release/refund endpoints and webhook verification, and simplified the stack by removing task-queue references — favor synchronous webhook processing for the MVP. The repo now contains the docs and tasks needed to implement payments, WhatsApp notifications, MCP tools, vetting, and LGPD compliance.

Next step: implement the Woovi client and the end-to-end booking + payment flow with idempotency and observability so the synchronous approach remains reliable as volume grows.
