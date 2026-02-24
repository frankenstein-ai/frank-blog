+++
date = '2026-02-21T22:45:43-03:00'
draft = false
title = 'WhatsApp FSM resilience, Pix QR flow, and message dispatcher — Find Workers (2026-W08)'
+++

## Problem we set out to solve

Find Workers is an MCP-first backend that connects AI assistants, WhatsApp messaging and Pix payments. During this work period we ran into three reliability and UX problems in production:

1. WhatsApp conversation FSMs could get irrecoverably stuck when handlers raised invalid transitions or unexpected exceptions. Stuck conversations left users with dead chats and required manual fixes.
2. The WhatsApp booking flow had to create Pix charges, deliver QR codes to customers, and notify providers when a booking required payment. The full path (create charge → send QR → notify worker → handle webhook on payment complete) needed to be robust and testable.
3. Our message-processing stack lacked a small but important layer: an intent parser and reply formatters to map incoming WhatsApp text to domain actions and consistent replies.

While fixing those, we also removed a tests-only flakiness caused by a retry decorator whose default delay was captured at import time, preventing tests from zeroing waits.

Below I describe the changes we made, why they matter, and code patterns other engineers can reuse.

## What we changed (high level)

- Hardened the WhatsApp conversation FSM so invalid transitions and unhandled errors reset the conversation to a safe idle state and return a friendly fallback message.
- Built a booking→payment path over WhatsApp:
  - Create Pix charge via Woovi (OpenPix) with split sub-account for platform escrow.
  - Deliver QR (text template + PNG) through WhatsApp Business Cloud.
  - Notify the worker when a booking awaits payment.
  - Wire the charge-completed webhook to send payment notifications to customer and worker and to update booking state.
- Added a small WhatsApp dispatcher:
  - Intent parsing to map messages to domain intents (onboarding, booking, help, payment).
  - Reply formatters and templates for consistent payloads.
  - Webhook wiring so inbound WhatsApp events go through the dispatcher.
- Implemented a WhatsApp onboarding FSM and Pix Automático (recurring) subscription endpoints (create and cancel).
- Fixed the retry helper so tests can override the default delay at runtime.

Over the week we expanded and stabilized the WhatsApp/payment integration while keeping the test suite guarded; the repository ran about 3,678 tests during these changes.

## Concrete implementation details

### 1) Auto-resetting the conversation FSM

Problem: ConversationManager.transition used strict FSM transitions and raised on invalid transitions. If a handler raised, the conversation state could remain corrupted.

Before (simplified):

```python
# conversation.py (before)
def transition(conversation, event):
    conversation.fsm.trigger(event)  # raises InvalidTransition
    conversation.save()
```

This bubbled exceptions up and left the FSM in a broken state.

Change:

- Catch InvalidTransition and other exceptions in ConversationManager.transition.
- Log the failure, reset the conversation FSM to an idle state, persist the reset, and return a friendly fallback reply to the user.
- Add a short timeout/cleanup for conversations in unexpected states.

Sketch of the new approach:

```python
try:
    conversation.fsm.trigger(event)
except InvalidTransition:
    logger.warning("Invalid FSM transition, resetting conversation", extra={"conv_id": conversation.id})
    conversation.reset_to_initial()
    conversation.save()
    return reply_text("Desculpe — reiniciando a conversa. Pode dizer novamente o que precisa?")
except Exception as exc:
    logger.exception("Unhandled error in conversation; resetting", exc_info=exc)
    conversation.reset_to_initial()
    conversation.save()
    return reply_text("Ocorreu um erro. Reiniciando a conversa, por favor repita.")
```

Result: users are no longer left in dead chats and support load from manual fixes dropped.

### 2) Booking flow: create Pix charge, send QR, notify worker

We added the booking→payment path inside the WhatsApp booking handler:

- The booking flow calls Woovi to create a Pix charge with a split sub-account so the platform retains escrow.
- When Woovi returns charge data (QR string and PNG), we persist minimal, sanitized event data and send the QR to the customer:
  - A templated text with amount and expiration.
  - An image/media message with the QR via WhatsApp Business Cloud.
- We notify the worker with a short summary and a link to accept or view the job.

Representative flow (simplified):

```python
charge = woovi_service.create_charge(booking_id=..., amount=..., recipient_subaccount=worker.subaccount_id)
whatsapp.send_template(customer_phone, "payment_qr", {"amount": f"R${amount:.2f}"})
whatsapp.send_image(customer_phone, image_bytes=charge.qr_png)
whatsapp.send_text(worker_phone, f"Novo agendamento: {booking.brief}. Cliente já recebeu QR.")
```

We also wired the charge-completed webhook (api/webhooks.py) to:

- Mark the payment as recorded.
- Release or schedule release of funds according to the booking lifecycle.
- Send WhatsApp notifications to customer and provider.

Tests assert that the WhatsApp client receives both template text and binary image messages and that worker notifications occur on charge creation and completion.

### 3) Dispatcher: intent parsing and reply formatters

We added a small dispatcher layer to decouple intent recognition from domain logic:

- intent.py performs lightweight intent parsing from text and maps utterances to intents like "onboarding.start", "booking.create", "payment.confirm".
- replies.py contains reply formatters that generate consistent WhatsApp payloads (templates, quick replies, images).
- A webhook entry point routes incoming events to the dispatcher.

Example mapping:

```python
intent = intent_parser.parse(text)
if intent.name == "booking.create":
    return booking_flow.handle_create_intent(conversation, intent)
if intent.name == "onboarding.start":
    return onboarding.start_flow(...)
```

This keeps booking and onboarding logic focused and makes message-to-action flows easier to test.

### 4) Onboarding FSM and Pix Automático (subscriptions)

- Implemented a WhatsApp onboarding flow module as an FSM similar to the booking FSM.
- Added endpoints and a service layer to create and cancel Pix Automático subscriptions via Woovi.
- Added tests covering happy-path and cancellation flows:
  - tests/test_whatsapp_onboarding.py
  - tests/test_pix_automatic_subscription.py

### 5) Fix: retry decorator default delay capture

Problem: The retry helper used a default parameter like base_delay: float = BASE_DELAY. Python evaluates default parameters at function definition time, so tests that monkeypatch BASE_DELAY could not change the helper's delay.

Old signature:

```python
def _with_retry(..., base_delay: float = BASE_DELAY):
    ...
```

Fixed signature:

```python
def _with_retry(..., base_delay: Optional[float] = None):
    if base_delay is None:
        base_delay = BASE_DELAY
    ...
```

This change lets tests set base_delay to 0 via monkeypatch and removed sleeps from the test suite.

## Tests & validation

- Changed/added tests:
  - tests/test_whatsapp_booking_flow.py (updated)
  - tests/test_whatsapp_conversation.py (updated)
  - tests/test_whatsapp_dispatcher.py (updated/added)
  - tests/test_whatsapp_onboarding.py (added)
  - tests/test_pix_automatic_subscription.py (added)
- Test-suite size observed during commits: 3,678 tests.
- Key fixes that reduced flakiness:
  - The retry default fix removed timing dependencies.
  - Hardened notification tests eliminated race conditions.
- Webhook tests assert sanitized event_data is persisted to avoid PII leakage.

## Lessons learned

- Default arguments evaluated at import time can silently break test fixtures. Use runtime defaults when tests need to override values.
- Conversation FSMs must tolerate handler errors. Resetting on invalid transitions and returning a clear fallback message reduces support work.
- End-to-end payment flows require both domain and operational safeguards: persist only minimal webhook data, implement retries, and notify both consumer and provider.
- A thin dispatcher (intent parsing + reply formatters) simplifies inbound message handling and makes tests easier to write.

## Next steps

- Add metrics and observability for automatic FSM resets (count resets per conversation and correlate with causes).
- Expand the dispatcher to support richer quick replies and structured buttons for “pay”, “cancel”, and “contact provider”.
- Add optimistic locking to booking state transitions to prevent duplicate charges under concurrent requests (flagged in issues).
- Increase test coverage for webhook edge cases, especially partial refunds and dispute flows.

If you work with conversational systems, payment webhooks, or WhatsApp integrations, these patterns are directly reusable: fail-safe FSMs, cautious persistence of webhook payloads, and a small dispatcher layer that separates intent recognition from domain logic.
