+++
date = '2026-02-24T00:55:30-03:00'
draft = false
title = 'Bringing the marketplace to WhatsApp: consumer orchestrator, interactive messages, and resilient state'
+++

## Problem: consumers live on WhatsApp, not on your web UI

Find Workers is an MCP-first backend that lets AI assistants act for users to find, price, and book home-service professionals. The assistant-to-backend surface was already covered, but most Brazilian consumers don't open an AI assistant — they live in WhatsApp. We needed a way to deliver the full marketplace flow (search, quotes, accept/reject, booking confirmations and Pix payment) entirely inside a WhatsApp conversation.

Key constraints:
- WhatsApp is a webhook-driven, stateless environment. Conversations must be reconstructed from inbound messages and short-lived state.
- Replies must be interactive (lists and buttons) so people can pick a worker or accept a quote with one tap.
- Routing must be deterministic and auditable — core intent routing cannot rely on opaque LLM parsing.
- The system must tolerate missing state (for example, no Redis entry) and avoid crashes on malformed input.
- We must preserve privacy and map WhatsApp phone numbers to consumer accounts automatically so the existing backend (bookings, payments) can be reused.

In 2026-W09 we shipped a set of changes to meet those requirements: a WhatsApp consumer orchestrator, message builders for interactive payloads, defensive error handling, and a test suite.

## What we built (high level)

- A consumer orchestrator that ingests inbound WhatsApp messages, classifies intent, and routes the conversation through a state machine. (src/find_workers/whatsapp/consumer.py)
- Message builders that render:
  - search results as WhatsApp interactive lists,
  - incoming quotes as button messages ("Aceitar" / "Recusar"),
  - booking confirmations and Pix payment CTAs,
  with correct BRL formatting and Brazilian number conventions. (src/find_workers/whatsapp/messages.py)
- Automatic onboarding: unknown phone numbers that message the endpoint are auto-registered as user_type=consumer so downstream services can use that record.
- Defensive handling for missing conversation state (no Redis row) to avoid NoneType crashes.
- Tests: 20 tests for message builders and 22 for the consumer orchestrator covering intents, interactive replies, button workflows, error handling, and push_quote state management.

With this in place a consumer can text "Preciso de um encanador em Pinheiros," receive an interactive list of matched workers, tap one to request quotes, get quote buttons like "João: R$250,00 [Aceitar] [Recusar]," and tap to accept — continuing the lifecycle that triggers booking and Pix payment flows already implemented elsewhere in the platform.

## Design decisions and tradeoffs

### Deterministic intent routing (no LLM in the router)
For the inbound consumer channel we used a keyword-based intent classifier (search, status, help, cancel, unknown). Deterministic parsing keeps routing auditable and avoids the cost and unpredictability of an LLM for a small, high-volume command set.

The classifier sits next to the orchestrator and emits structured intents that the orchestrator consumes.

### Conversation state machine in Redis
WhatsApp sends stateless HTTP webhooks. We store a compact per-phone conversation record in Redis to track where a consumer is (idle → searching → viewing → quoting → paying). That record holds the last list of worker candidates, pending quote IDs, and which message expects an interactive reply.

A recent robustness fix addressed a crash when Redis returned no state. The orchestrator now treats missing conv_state as "idle," continues the flow, and avoids returning 500s for unseen numbers or expired states.

### Interactive message constructions
WhatsApp Business supports two interactive types we rely on:

- Lists: show search results with a small payload per row (title, subtitle, id). Tapping a row returns a structured list reply with the selected id.
- Buttons: show quick actions for a quote (Accept / Reject).

We added helper functions that build these payloads and handle formatting details (currency, localized numbers, safe truncation). That keeps the webhook handler and business logic straightforward and easy to test.

## Concrete examples

Function signatures introduced (representative):

```python
# src/find_workers/whatsapp/consumer.py
async def handle_consumer_message(phone_number: str, text: str, session: AsyncSession):
    """
    Orchestrates incoming consumer messages:
    - parses intent (search/status/help/cancel/unknown)
    - loads/initializes conversation state from Redis
    - routes to the appropriate action (render list, show status, show help, cancel)
    - persists updated conv_state
    """
```

Push quote to consumer (sends an interactive button message):

```python
# src/find_workers/whatsapp/consumer.py
async def push_quote_to_consumer(consumer_phone: str, quote_id: uuid.UUID, worker_name: str, amount_cents: int):
    """
    Sends a button message with Accept / Reject actions and records
    the pending quote in redis so button replies are handled idempotently.
    """
```

Example of an interactive button message payload (conceptual JSON sent to WhatsApp client):

```json
{
  "to": "+5511999999999",
  "type": "interactive",
  "interactive": {
    "type": "button",
    "body": { "text": "João ofereceu um orçamento: R$250,00" },
    "action": {
      "buttons": [
        { "type": "reply", "reply": { "id": "quote:accept:1234", "title": "Aceitar" } },
        { "type": "reply", "reply": { "id": "quote:reject:1234", "title": "Recusar" } }
      ]
    }
  }
}
```

Example of a search results list (excerpt):

```json
{
  "to": "+5511999999999",
  "type": "interactive",
  "interactive": {
    "type": "list",
    "body": { "text": "Encontrei esses profissionais perto de Pinheiros:" },
    "action": {
      "sections": [
        {
          "title": "Top matches",
          "rows": [
            { "id": "worker:uuid-1", "title": "João - Encanador", "description": "R$150–R$300 • 4.8 ★" },
            { "id": "worker:uuid-2", "title": "Maria - Encanadora", "description": "R$200 • 4.6 ★" }
          ]
        }
      ]
    }
  }
}
```

Formatting utilities in messages.py ensure amounts display as "R$250,00" and distances/ranges follow Brazilian conventions.

## Testing and resilience

We added 42 tests across the new modules:

- tests/test_whatsapp_messages.py (20 tests): verify list/button construction, BRL formatting, truncation logic, and template edge cases.
- tests/test_whatsapp_consumer.py (22 tests): cover intent routing (search/status/help/cancel/unknown), list-selection handling (worker selection), button replies (accept/reject quote), auto-registration for new phone numbers, and the None-Redis-state crash fix.

Test highlights:
- Interactive replies are idempotent: tapping the same accept button twice does not create duplicate bookings.
- Missing conversation state is treated as idle and the flow recovers into search.
- push_quote_to_consumer records the pending quote in Redis and clears it on accept/reject transitions.

These tests give us a reliable integration surface between the messaging layer and booking/payment domains.

## Why this matters for other developers

- WhatsApp as the primary UI lets you reach users at scale without building a mobile app or web front end.
- A thin orchestrator that uses deterministic intent parsing plus a small Redis state machine keeps the UX simple and debuggable.
- Keep message formatting separate from business logic. Small, testable builders make changes safer.
- Defensive programming matters: tolerate missing state and malformed webhooks or a single 500 will break many users and create support noise.
- The approach fits MCP-first architectures: the same backend (bookings, payments, matching) serves both agent-driven and consumer-driven channels.

## Implementation notes and gotchas

- Persist minimal user metadata (phone, user_type=consumer) when mapping WhatsApp numbers to user records. Add a phone-ownership verification flow if you need stronger authentication later.
- Embed domain identifiers in interactive IDs (worker:uuid, quote:uuid) so the webhook router can handle replies without ambiguous parsing.
- Keep conversations short and idempotent. Consumers may tap stale buttons — return clear error messages when a quote or booking is no longer valid.
- Localize currency and numbers early. Brazilian formatting (R$ with comma decimals) improves trust.
- Watch webhook flood protection and request-size limits before doing expensive verification. For high-volume WhatsApp flows, rate-limit per phone number.

## Next steps

- Add an LLM-based fallback for ambiguous messages, but use it only as an assist and never as the single source of truth for financial actions.
- Add analytics for conversion inside WhatsApp: message→list tap rate, quote accept rate, booking completion, and Pix payment completion.
- Explore richer interactive UIs (multi-step forms, appointment pickers) while keeping lists and buttons for the core flows.
- Continue hardening concurrency around booking transitions (we already use SELECT FOR UPDATE in other parts of the system) to avoid race conditions when multiple providers and the consumer interact.

## Summary

We connected the Find Workers marketplace into consumers' WhatsApp threads: a consumer orchestrator routes intents through a Redis-backed conversation state machine, message builders render search results and quotes using WhatsApp lists and buttons with correct BRL formatting, and the system tolerates missing state and duplicate replies. The change shipped with 42 tests that validate UX paths and edge cases, making the WhatsApp channel a production-ready front door for users who prefer messaging over apps.
