+++
date = '2026-02-22T16:29:40-03:00'
draft = false
title = 'Wiring the Landing Page to Live Metrics and Polishing the Admin SPA'
+++

## The problem

Landing pages that show static, hard-coded metrics and testimonials are common. For a transaction platform like Find Workers they miss two practical goals:

- Credibility — visitors want live signals (recent jobs, region badges, current numbers) that show the product works today, not just marketing copy.
- Reliability — the UI must degrade gracefully when the backend or network are slow or down.

At the same time, the admin single-page app had inconsistent scaffolding and visual language across dozens of routes, which made maintenance and onboarding slower. On the backend, the public stats endpoint contained small issues (inline imports, inaccurate comments, unused parameters) that made the code noisier and risked subtle mismatches with DB enums.

This week (2026-W08) we tackled three things:

1. Replace hardcoded landing-page data with live data from /v1/public/stats.
2. Make the landing page resilient (skeletons, fallback stats) and visually consistent (initials avatars, simplified CSS).
3. Clean up admin SPA routes and tidy the public API code for maintainability.

Below I walk through what changed, why it matters, and include concrete snippets and patterns you can reuse.

## What we shipped

- Frontend: Landing.tsx now fetches real public stats and wires six data groups to the API: service signals, hero sidebar, proof/testimonials, region badge, match queue, and testimonials. We added a useLandingStats hook, skeleton shimmer states, and a FALLBACK_STATS object for graceful degradation.
- Frontend polish: removed dead CSS variables, replaced remote picsum avatars with initials-based CSS circles, and unified page scaffolding across admin routes with a consistent emerald/zinc palette and shared layout components.
- Backend: cleaned up src/find_workers/api/public.py — moved inline imports to the top, corrected a TopReview.role comment to match the DB enum (provider), removed an unused Request parameter from get_public_stats, and imported literal_column and User where needed.
- Tests: removed unused helpers in tests/test_public_stats.py and adjusted tests to reflect the new wiring.

Why this matters: visitors see live, trustworthy metrics even under partial failure; the public API surface is smaller and clearer; the admin team has a consistent UI to manage the product.

## Frontend: wiring the landing page to live data

Problem: the previous landing page used hardcoded numbers, static testimonials and placeholder avatars. That made the page brittle and an unreliable representation of real activity.

Solution highlights

- A useLandingStats hook that GETs /v1/public/stats and returns { data, loading, error }.
- A FALLBACK_STATS object to show sensible defaults when the API fails.
- Skeleton shimmer states for all metrics so content doesn't jump as data arrives.
- Match queue built from recent_services (the API field) but shown with synthetic, privacy-safe display names.
- Avatars switched from external picsum.photos images to CSS initials circles to remove extra network requests and keep a consistent look.

Example hook (you can copy this and adapt it):

```ts
// admin/src/pages/Landing.tsx
import { useEffect, useState } from "react";

const FALLBACK_STATS = {
  total_services: 1200,
  avg_response_seconds: 240,
  match_queue: [],
  testimonials: [{ first_name: "Ana", role: "consumer", quote: "Muito bom!" }],
  // ...
};

export function useLandingStats() {
  const [data, setData] = useState(FALLBACK_STATS);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let mounted = true;
    fetch("/v1/public/stats")
      .then((r) => r.json())
      .then((json) => {
        if (mounted) setData(json);
      })
      .catch((err) => {
        console.warn("Landing stats fetch failed:", err);
        setError(err);
      })
      .finally(() => {
        if (mounted) setLoading(false);
      });
    return () => {
      mounted = false;
    };
  }, []);

  return { data, loading, error };
}
```

We also added small formatters for compact numbers, seconds → friendly time, percentages and BRL currency so the presentation code stays concise.

UX win: while data loads each metric shows a skeleton shimmer. That reduces perceived load time and prevents layout shifts when values arrive.

## Backend: small cleanups that reduce cognitive load

The public stats endpoint lives in src/find_workers/api/public.py. The changes are small but useful:

- Move literal_column and User imports to the module top so imports are discoverable.
- Fix TopReview.role comment to match the DB enum: "provider" instead of "worker".
- Remove an unused Request parameter from the get_public_stats function signature.

Before:

```py
# previously had inline imports and a mismatched comment
class TopReview(BaseModel):
    first_name: str
    role: str  # "consumer" or "worker"
    quote: str

# inline import used later:
from sqlalchemy import literal_column
```

After:

```py
from sqlalchemy import distinct, func, literal_column, select
from find_workers.models.user import User

class TopReview(BaseModel):
    first_name: str
    role: str  # "consumer" or "provider"
    quote: str
```

Keeping imports at the top and aligning model comments with canonical DB enums prevents confusion for future contributors writing queries, migrations or tests. Removing unused parameters also makes the endpoint contract clearer and simpler to test.

## Admin SPA: consistent scaffolding and polish

We harmonized pages across the admin app: Dashboard, Users, Workers, Bookings, Disputes, Finance (Revenue, Reconciliation, Payouts, Refunds), LGPD workflows, and Settings. After visual QA we:

- unified layout and page scaffolding across 30+ admin routes;
- standardized color tokens, typography and metric cards;
- consolidated repeated components (DataTable, MetricCard, Timeline, RoleGuard, ConfirmDialog) to reduce duplication.

Result: new admin pages are faster to build and less error prone because they share the same component APIs and design tokens.

Small cleanup example in the landing component:

```diff
-import type { CSSProperties, ReactNode } from "react";
+import type { ReactNode } from "react";
// ...
-      style={{ "--reveal-delay": delay } as CSSProperties}
+      // removed dead CSS var --reveal-delay
```

Removing dead CSS variables and unused TypeScript imports reduces bundle size and the chance of accidental style regressions.

## Tests and QA

- Removed an unused test helper (_ScalarValue) from tests/test_public_stats.py.
- Adjusted tests that relied on hardcoded fixtures; the landing now reads the real API and tests reflect that.

The repo already has a large test surface (1100+ unit and integration tests), which made these changes safe to land.

## Lessons and next steps

- Live signals on public pages increase trust. Design for partial failure: FALLBACK_STATS and skeletons are simple, effective strategies.
- Small backend hygiene (move imports, keep model comments accurate, trim unused parameters) compounds: it makes the codebase easier to reason about and refactor.
- Shared design tokens and components in the admin SPA pay off quickly across dozens of routes.

Straightforward next steps

- Cache /v1/public/stats with a short TTL and server-side aggregation to reduce DB load under traffic.
- Add A/B tracking to measure whether live metrics increase conversion and signups.
- Expand the public stats payload to include anonymized time-series for small graphs (we already have recent_services for the match queue).

Exact diffs and the full useLandingStats hook are available on request.
