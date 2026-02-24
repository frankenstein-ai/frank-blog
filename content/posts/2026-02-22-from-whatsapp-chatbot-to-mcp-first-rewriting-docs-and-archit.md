+++
date = '2026-02-22T14:57:57-03:00'
draft = false
title = 'From WhatsApp Chatbot to MCP-First: Rewriting docs and architecture for Find Workers'
+++

Problem
We needed to reconcile an implemented backend with our product vision after parts of the codebase drifted toward the wrong UX assumption. For Find Workers the product must be MCP-first (AI assistants — Claude/ChatGPT/Gemini — act for users), but parts of the repo still assumed consumers would discover and book via WhatsApp. In 2026-W08 we removed that mismatch, documented a concrete admin UI plan, and brought the technical architecture and business-model docs in line with the implementation so engineers, ops, and product can move forward in sync.

Below I describe what we changed, why it matters, and the developer work that follows: removing consumer-facing WhatsApp flows, formalizing a conversation FSM used for support and notifications, mapping admin API surface and implementation gaps, and reconciling finance/DB/infra details with working code.

## What we changed (high level)

We made five coordinated documentation and architecture changes:

- Rewrote five major docs to assert an MCP-first architecture and treat WhatsApp as a notifications channel only: business-model.md, technical-architecture.md, mcp-research.md, client-acquisition-research.md, user-research.md.
- Added an Admin Interface Specification (18+ pages) and an Implementation Prerequisites appendix that inventories existing endpoints, model gaps, and migration needs.
- Reconciled technical-architecture.md with the code: conversation state machine, MCP tools and endpoints, DB schema choices (centavos, pix_key fields), infra layout (MCP embedded at /mcp), and environment variables.
- Updated README and business-model docs to match the real revenue model (subscription + worker commission), adjusted unit economics, and removed a per-use consumer fee.
- Updated project issues (.beads) to record the decision to remove or repurpose the consumer-facing WhatsApp chatbot flow.

These are documentation and architecture changes, but they map directly to code paths and migration steps engineers will implement next.

## Why this matters

Find Workers is MCP-native: assistants do discovery, quoting, and booking for consumers. That shifts backend responsibilities:

- The backend must expose precise MCP tools and deterministic schemas, not a WhatsApp discovery UX.
- WhatsApp should be a one-way channel for notifications (payment QR codes, confirmations, reminders) and occasional simple confirmations, not the primary discovery or booking surface.
- Admin tooling and migrations must reflect how money is stored (centavos), how Pix escrow works (split sub-accounts), and the exact API contract used by AI assistants.

Previously some docs and code implied consumers used WhatsApp for discovery/booking. That caused duplicated UX logic, inconsistent tests, unclear ownership, and extra maintenance. Aligning docs and code removes that ambiguity.

## Concrete changes and examples

Below are representative excerpts and facts pulled from diffs and the new docs.

### 1) Conversation FSM reconciled with implementation

We documented the conversation FSM used for the limited interactions that still go through WhatsApp (confirmations, worker onboarding prompts, quick replies). ConversationContext is immutable and transitions are explicit. Redis session TTLs are configurable (default 1,800 seconds).

Example (simplified):

```python
# src/docs/technical-architecture.md (excerpt)
class ConversationState(Enum):
    IDLE = "idle"
    SEARCHING = "searching"
    VIEWING_QUOTES = "viewing_quotes"
    CONFIRMING_BOOKING = "confirming_booking"
    AWAITING_PAYMENT = "awaiting_payment"
    RATING_SERVICE = "rating_service"
    AWAITING_SERVICE_TYPE = "awaiting_service_type"
    # ... onboarding and other states

@dataclass(frozen=True)
class ConversationContext:
    phone: str
    state: ConversationState = ConversationState.IDLE
    worker_id: str | None = None
    booking_id: str | None = None
    quote_ids: list[str] = field(default_factory=list)
    search_query: str | None = None
    onboarding_data: dict = field(default_factory=dict)
    booking_data: dict = field(default_factory=dict)
    updated_at: float = 0.0

VALID_TRANSITIONS: dict[ConversationState, set[ConversationState]] = {
    ConversationState.IDLE: {ConversationState.SEARCHING, ConversationState.AWAITING_SERVICE_TYPE},
    ConversationState.SEARCHING: {ConversationState.IDLE, ConversationState.VIEWING_QUOTES},
    # ...
}
```

Why it matters: freezing the context and enforcing VALID_TRANSITIONS reduces accidental state corruption and makes unit tests for allowed transitions straightforward.

### 2) MCP tools and endpoints — what exists vs. what’s planned

The technical architecture now lists implemented MCP tools (six implemented tools) and marks the "check_availability" tool as planned. The MCP server is mounted at /mcp in the dev compose, and we documented a token verifier (McpTokenVerifier).

Why it matters: agent frameworks need a precise list of tools and JSON schemas. The reconciled docs close the gap between spec and what runs in dev.

### 3) Business model and fees: 10% → 5% platform fee

We aligned the docs with the implementation: platform fee is 5% now, and monetary fields are expressed in centavos across the DB (integers). The business model lists two revenue streams: consumer subscription (R$10/month) and commission on completed jobs. We removed the per-use consumer booking fee.

Why it matters: centavos avoids floating-point accounting errors; the fee change affects unit economics and product choices like promotions and pricing.

### 4) Admin interface spec + implementation prerequisites

We added a 1,161-line Admin Interface Specification for a React SPA (React 19, TypeScript, Vite) and an Implementation Prerequisites appendix mapping backend to UI needs. The docs enumerate implemented admin endpoints and the pages they support. Example inventory excerpt:

- GET /v1/admin/dashboard — api/admin/dashboard.py — Dashboard
- POST /v1/admin/send-reminders — api/admin/dashboard.py — Operations
- GET /v1/admin/workers/pending — api/admin/workers.py — Worker list
- POST /v1/admin/workers/{id}/approve — api/admin/workers.py — Worker detail
- GET /v1/admin/disputes — api/admin/disputes.py — Disputes
- GET /v1/admin/reconciliation — api/admin/finance.py — Finance

The Implementation Prerequisites section lists model/schema changes, service-layer methods to implement, notification functions, RBAC middleware, and an Alembic migration plan.

Why it matters: the admin UI is the operator’s single source of truth. An endpoint inventory and migration plan reduce frontend rework against missing APIs.

### 5) WhatsApp: demoted from primary UX to notifications channel

The docs now record that consumer-facing multi-step WhatsApp flows (full chatbot booking wizard, consumer booking FSM, intent classifiers, dispatcher routes, reply formatters) are incorrect for an MCP-first product. The .beads issue lists modules to remove or repurpose:

- src/find_workers/whatsapp/booking_flow.py
- src/find_workers/whatsapp/conversation.py (consumer booking states)
- src/find_workers/whatsapp/intent.py (search/book intent classification)
- src/find_workers/whatsapp/dispatcher.py (booking/search dispatch paths)
- src/find_workers/whatsapp/replies.py (format_search_results)
- src/find_workers/whatsapp/onboarding.py (needs review — may be used for worker onboarding)

Files kept: client.py (low-level API), notifications.py (payment QR codes, templates), templates.py, tracking.py.

Why it matters: removing duplicated consumer flow logic centralizes booking through MCP tools and the REST API, reduces fragmented tests, and clarifies responsibilities.

## Developer-facing action items (next steps)

Documentation changes make the work explicit. Key engineering tasks:

1. Remove or repurpose consumer WhatsApp modules
   - Delete or scope back booking_flow, consumer conversation/intent/dispatcher/replies.
   - Keep low-level WhatsApp client, templates, and tracking.

2. Implement missing admin endpoints and RBAC middleware
   - Add endpoints flagged in Implementation Prerequisites.
   - Implement step-up auth for sensitive actions (X-Step-Up-Token header flow).

3. Run Alembic migrations
   - Convert monetary columns to integer centavos where needed.
   - Add pix_key fields and three new tables documented in the technical reconciliation.

4. Complete MCP tools
   - Implement the planned "check_availability" tool.
   - Ensure MCP tool schemas in mcp_server.py match the docs.

5. Tests and cleanup
   - Update or remove tests that target the consumer WhatsApp booking flow.
   - Add tests that verify ConversationContext immutability and VALID_TRANSITIONS.

6. Improve docs-to-code traceability
   - Map admin-interface.md pages to test coverage checks.
   - Keep .beads issues updated as refactors progress.

## Lessons learned

- Spec drift is a long-term maintenance risk. Docs must be updated as code changes.
- Explicit state machines with frozen contexts (dataclasses.replace patterns) reduce subtle bugs; small changes here buy a lot of reliability.
- Treat communications channels as channels, not UX platforms. For MCP-first products, assistants are the UX; other channels are delivery and notification mechanisms. That changes where the source of truth lives.
- Documentation should include a precise endpoint inventory and a migration plan. Those reduce rework between frontend and backend teams.

## Closing

This week’s work was mostly documentation and architecture, but it unlocks high-value engineering tasks: removing duplicated consumer chat logic, implementing missing admin APIs, enforcing accounting correctness via centavos fields, and finalizing MCP tool schemas. Be ruthless about defining the source of truth for your UX (in our case, AI agents), then reorganize code and docs so every team member — engineers, QA, product, and ops — can see the next steps clearly.

If you’d like, I can extract the specific file list, tests to update, and an Alembic migration checklist from the new admin prerequisites in a follow-up.
