+++
date = '2026-02-23T00:18:04-03:00'
draft = false
title = 'W09 recap: WhatsApp intent + price parsing, Alembic migration fixes, docs humanization and infra tweaks'
+++

This week we tackled three recurring blockers to developer velocity in our early-stage backend: brittle migrations that break local worktrees, noisy WhatsApp replies that need to become structured inputs, and documentation that read like marketing instead of instructions. Below are the concrete changes we made, why they matter, and code you can reuse if you run a similar payments and messaging system.

## 1) Problem: brittle Alembic migrations and local dev ergonomics

Symptoms
- Duplicate Alembic revision IDs caused Alembic to stop applying new migrations in some developer branches.
- A malformed SQL expression (wrong ordering, missing parentheses) in a cleanup migration raised syntax errors during upgrade.
- Local docker-compose sometimes did not expose the API port, and there was no DEV_MODE switch to enable local-only shortcuts (OTP bypass for a test admin).

Impact
- A broken migration blocks multiple features that touch the database (checkout, payments, onboarding).
- Adding a uniqueness index on columns that already have duplicates can fail or leave the database in an inconsistent state.
- Poor local ergonomics slow onboarding and repetitive testing.

What we changed
- Renamed one migration file and fixed its revision id to avoid the duplicate.
- Rewrote the deduplication SQL so it deterministically keeps the most recent booking per group.
- Added DEV_MODE to .env.example and exposed port 8000 in docker-compose.yml so the API is reachable on localhost during development.

Concrete change — safer dedup SQL
The original migration selected MAX(id) per consumer/worker, which does not guarantee the newest row if ids are not monotonic with creation time. We changed the query to pick the most recent id by ordering on created_at and taking the first element of an ordered array:

```sql
SELECT consumer_id, worker_id,
       (ARRAY_AGG(id ORDER BY created_at DESC))[1] AS keep_id
FROM bookings
WHERE status IN ('pending', 'accepted', 'in_progress')
GROUP BY consumer_id, worker_id;
```

Why this helps
- ARRAY_AGG with ORDER BY returns the id of the newest record for each group, regardless of id monotonicity.
- We delete duplicates, commit, then run CREATE INDEX CONCURRENTLY outside a transaction. CREATE INDEX CONCURRENTLY cannot run inside a transaction block, so the cleanup must be its own commit.

Migration hygiene checklist
- Give every Alembic revision a unique revision id and the correct down_revision pointer.
- If you add a uniqueness constraint on potentially duplicated columns:
  - identify duplicates with a stable ordering (created_at or another timestamp),
  - delete duplicates in a reversible way,
  - commit the cleanup before running CREATE INDEX CONCURRENTLY.

Dev-mode and docker-compose
We added an explicit DEV_MODE flag to .env.example to enable local conveniences only in development:

```env
# Dev-mode admin bypass: when true, phone +55 (99) 99999-9999 authenticates
# as super_admin with OTP "000000" without WhatsApp. Do not enable in production.
DEV_MODE=true
```

We also mapped the API port in docker-compose so the app is reachable at http://localhost:8000 during development. These changes are for local iteration and are guarded by the DEV_MODE comment.

## 2) Problem: WhatsApp free-text is noisy — we need structured intents and prices

Context
WhatsApp is our main consumer-facing channel. Messages come in Portuguese with colloquialisms, location snippets, and informal currency expressions. To make the real-time quoting flow work we needed:
- a reliable intent classifier for common WhatsApp actions (search, status, help, cancel),
- a price parser that understands Brazilian Real formats (R$ 250,00; 250 reais; 350).

What we shipped
- An intent parser (src/find_workers/whatsapp/intent.py) that classifies Portuguese free-text into structured intents and extracts category mentions, short description fragments, location hints, and greeting detection.
- A Brazilian Real price parser (src/find_workers/whatsapp/price_parser.py) that normalizes price text into integer centavos suitable for quotes and payments.

Tests
- 26 unit tests for the intent classifier (tests/test_whatsapp_intent.py) covering common Portuguese phrasings.
- Unit tests for the price parser (tests/test_price_parser.py) that cover formatting variants.

Examples

Intent classification examples:

Input:
"Preciso de um encanador, vazamento na cozinha, zona sul, urgente"

Output:
{
  "intent": "search",
  "category": "encanador",
  "description": "vazamento na cozinha",
  "location_hint": "zona sul"
}

Input:
"Qual o status do meu orçamento #1234?"

Output:
{
  "intent": "status",
  "reference": "1234"
}

Input:
"Cancelado, não preciso mais"

Output:
{
  "intent": "cancel"
}

Price parsing examples:

Input: "R$ 250,00"  
Normalized: 25000  # cents (centavos)

Input: "350 reais"  
Normalized: 35000

Input: "250"  
Normalized: 25000 (we interpret bare integers as BRL reais)

Why this matters
- Structured intents let backend services route messages to the correct flow: start a search, show booking status, or process a cancellation.
- Normalized prices let us create Quote records and trigger escrow flows without manual fixes.

Implementation notes
- We used keyword matching tuned for Portuguese colloquialisms plus a small set of deterministic rules. This gives high-precision results and predictable behavior for the first iteration.
- The price parser uses regex patterns that accept optional "R$", optional thousands separators, comma or dot decimals, and currency words like "reais".
- Everything is covered by unit tests so regressions are easy to spot when we add new phrases or locales.

Usage example (pseudocode)

```python
from find_workers.whatsapp.intent import classify_message
from find_workers.whatsapp.price_parser import parse_brl_price

msg = "R$ 250,00 - posso fazer por 250 reais"
intent = classify_message(msg)     # -> {"intent": "quote_response", ...}
price_cents = parse_brl_price(msg) # -> 25000
```

If parse_brl_price returns None, the flow falls back to human review or asks the worker to resend a numeric amount.

Roadmap tie-in
These parsers are prerequisites for the real-time quote broadcast work. MatchingEngine will select top-N workers, broadcast the request via WhatsApp, and ingest replies with the price parser so QuoteService can create quotes automatically.

## 3) Docs: remove marketing language, keep the facts

What we changed
- Rewrote seven documentation files to make the prose clearer and less promotional while preserving all data, code examples, and tables.
- Files edited:
  - docs/business-model.md
  - docs/market-research.md
  - docs/user-research.md
  - docs/client-acquisition-research.md
  - docs/mcp-research.md
  - docs/technical-architecture.md
  - docs/vetting-verification-research.md

What we did to the text
- Switched headings to sentence case.
- Removed filler phrases and excessive bolding.
- Replaced some em-dashes with commas or parentheses where it reads better.

Outcome
- The docs read more like guidance from a teammate and less like marketing copy.
- We kept all factual content and made the prose easier to scan for contributors and partners.

## operational notes

- We added an epic for a "trust engine" (beads) to handle payer-mismatch and admin review flows in payments.
- The repository has 1,100+ tests; we added unit tests for intent and price parsing.
- Small infra tweaks (docker-compose and .env.example) make local onboarding faster.

## lessons and recommendations

Migration safety
- Identify duplicates deterministically, for example by ordering on created_at.
- Delete duplicates in a reversible way and commit the cleanup.
- Run CREATE INDEX CONCURRENTLY only after the cleanup commit.

Messaging channels (WhatsApp)
- Start with deterministic, rule-based parsing for high precision and test coverage.
- Define uncertainty thresholds: when to ask for clarification or escalate to manual review.
- Log raw messages (with PII safeguards) so you can retrain or expand coverage later.

Developer ergonomics
- Small dev-mode conveniences and explicit port mappings significantly reduce friction.
- Gate these behind a DEV_MODE flag and never enable them in production.

## next steps

- Wire the intent and price parsers into the Quote broadcast workflow and add the WhatsApp outbound template for workers.
- Implement an admin review flow for flagged payments (payer-mismatch), high on the fraud mitigation list.
- Expand tests with integration scenarios that simulate MatchingEngine → broadcast → worker reply → parsed quote → create charge.

See the new files under src/find_workers/whatsapp/ (intent.py and price_parser.py) and the updated Alembic revisions in alembic/versions for the exact changes. These fixes are small and focused, but they remove recurring friction and make the WhatsApp channel usable as a structured input for the system.
