+++
date = '2026-02-23T02:33:01-03:00'
draft = false
title = 'Hardening the Admin SPA: OTP Step‑Up, API Shape Alignments, and Playwright E2E'
+++

Problem first: the admin Single Page App was brittle in several small but painful ways. Admins hit 403s on sensitive operations (OTP/step-up), many pages crashed when the backend returned wrapped responses, some pages queried the wrong endpoints or fields (404 / 405), and hosting the SPA under /admin/ made the site harder to reach. We also had no reliable e2e harness, so regressions slipped through.

During the 2026‑W09 work period we focused on three things:
- Fix the step-up (OTP) flow and its UX regressions (Operations page 403).
- Align frontend API calls and response handling with the backend (fix 404/405/422 and DataTable crashes).
- Add Playwright e2e tests and developer tooling so the admin UI stays healthy.

This post explains what we changed, why it mattered, and the pragmatic patterns we used (code examples included) so you can apply them in your own admin SPAs.

## What was failing and why it mattered

1. Step-up (OTP) flow
   - The client requested and verified OTPs without getting the user phone or specifying the OTP purpose. The backend requires a phone and a purpose, and the verify endpoint expects otp_code, not code. Result: requests failed or verifications returned 422.
   - Operations that required step-up returned 403 and the UI didn’t offer a retryable dialog, so admins hit dead ends.

2. API contract drift
   - Some admin pages expected arrays at the top level (e.g., .items) while the backend returned wrapped shapes (.templates, .keys, .phones). DataTables crashed with r.map is not a function.
   - Several endpoints and query params used the wrong paths or filters (for example /v1/admin/workers vs /v1/admin/workers-list), causing 404/405.
   - Coverage categories requested limit=200 while the backend enforces limit <= 50, which produced 422.

3. Developer UX and regressions
   - The SPA was mounted at /admin/ but we preferred the landing page at /. FastAPI mounts, Caddy rules, and vite base needed to match.
   - We had no robust e2e tests; some regressions reached users. Adding Playwright tests with deterministic API mocking fixed that.

## Concrete fixes and examples

### Fixing the OTP (step-up) flow

Problem: the client sent wrong fields and lacked context.

Before (buggy payloads)
- Request OTP sent without phone or purpose.
- Verify used { code: "123456" }.

After
- Fetch the current user (GET /v1/auth/me) to get phone_number.
- POST /v1/auth/request-otp with phone_number and purpose: "stepup".
- Verify with otp_code.

Key code change (TypeScript):

```typescript
// admin/src/components/StepUpDialog.tsx
api
  .get<{ phone_number: string }>("/v1/auth/me")
  .then((me) =>
    api.post("/v1/auth/request-otp", {
      phone_number: me.phone_number,
      purpose: "stepup",
    }),
  )
  .then(() => {
    setOtpSent(true);
    inputRef.current?.focus();
  });

// Later: verify
const response = await api.post<{ token: string }>("/v1/auth/step-up", {
  otp_code: code, // <- was `code` before, now matches backend schema
});
```

Why this matters
- Passing purpose ensures the server uses the OTP only for step-up.
- Fetching the phone avoids sending OTPs to the wrong number and keeps the UX predictable.
- Using otp_code prevents 422 validation errors (the schema forbids unknown fields).

### Handle backend response shapes on the UI

Pages crashed because the frontend assumed raw arrays. We changed each page to extract the correct property:
- WhatsApp templates → response.templates
- McpKeys → response.keys
- BlockedPhones → response.phones
- Categories → response.items (and fixed the limit param described below)

The pattern: make the contract explicit in the client. Fail loudly in dev, but handle unexpected shapes gracefully in production.

### Endpoint and filter corrections

Fixed incorrect paths and filters:
- Workers list: use /v1/admin/workers-list (not /v1/admin/workers)
- Worker detail: /v1/admin/workers/{id}/detail (not /v1/admin/workers/{id})
- Coverage categories: use public /v1/categories (not /v1/admin/categories)
- Filter rename: verification_status → status

These are tedious, but they remove 404/405 errors and restore functionality.

### Fix coverage categories limit

Bug: client requested limit=200 while backend rejects >50. Fix:

```diff
- limit: "200",
+ limit: "50",
```

Lesson: keep client-side pagination limits aligned with backend validation.

### Integrate StepUpDialog into Operations

Operations returned 403 when step-up was required but the UI didn’t guide the user. We integrated StepUpDialog into OperationCard and added a retry callback:
- On 403, open the step-up dialog.
- After successful step-up, retry the original operation automatically.

This small change removed manual navigation and one-off retries from the admin workflow.

### Serve the SPA from root and align server config

We moved the SPA base path from /admin/ to /. Changes:
- Update Caddyfile catch-all to serve the SPA at /.
- Update FastAPI mount from /admin to /.
- Update vite base path.

Serving the admin at root reduced routing surprises and simplified local dev URLs.

## Playwright e2e: tests and mocking strategy

We added Playwright tests with deterministic API mocking to avoid flaky auth redirects and to assert UI behavior reliably.

Highlights
- ~37 focused specs covering login, dashboard, navigation, DataTable rendering, and auth-guard redirects.
- Fake JWTs: generate minimal JWTs with claims (admin_role, sub, exp) so the SPA parses role and userId.
- Route interception: register a catch-all /v1/** handler first, then specific auth routes last. That LIFO order keeps specific mocks from being overridden by the generic handler.
- Serial CI execution (workers: 1) because tests share the same mocked dev phone number for OTP flows.
- Add Node alongside bun because Playwright’s ESM preflight doesn’t work with bun’s loader.

What the mocks do
- Return stable dashboard KPIs so tests do not redirect to login.
- Stub list endpoints (workers, bookings) with deterministic payloads that match backend shapes.

Why this approach
- Full end-to-end tests are valuable, but auth and backend drift make them flaky. Mocking the network keeps browser behavior realistic while making CI stable and fast.

## Developer ergonomics: Nix + Playwright browsers

We wired Playwright browsers through Nix so CI and dev environments get reproducible browser binaries. Updated flake.nix ensures consistent runtimes for the e2e harness and removes "works on my machine" problems.

## Lessons learned and recommendations

1. Enforce and share API contracts
   - Most issues were contract mismatches: wrong field names, wrong endpoints, wrong limits. Generate TypeScript types from OpenAPI or share a small SDK between frontend and backend to reduce these regressions.

2. Make error flows retryable in the UI
   - Operations that can be satisfied by an extra auth step should surface a dialog and retry automatically. Handle 403 at the request layer with a user dialog + retry callback.

3. Protect e2e from external flakiness
   - Mock external dependencies at the network layer in browser tests. Fake JWTs plus route interception provide high confidence in UI behavior without hitting auth or third-party systems.

4. Be conservative with pagination limits
   - If the backend enforces a hard maximum, reflect that in client defaults and use feature flags to change behavior.

## What changed (at a glance)
- 7 commits focused on admin stability and e2e.
- Key files touched:
  - admin/src/components/StepUpDialog.tsx (step-up fixes)
  - admin/src/pages/*.tsx (Coverage, Operations, Login, settings)
  - admin/playwright.config.ts, admin/e2e/*.spec.ts (Playwright tests & fixtures)
  - Caddyfile and FastAPI mount updated to serve SPA at root
  - Nix and package files to provide Playwright and browser binaries

## Next steps
- Generate or share request/response types (OpenAPI → TypeScript) to prevent future contract drift.
- Add contract tests between backend response shapes and frontend expectations.
- Run an additional e2e gate against staging with real auth for integration checks, while keeping mocked tests fast for CI.

If you maintain an admin surface or any rich SPA backed by a custom API, small contract drifts quickly become visible UX problems. The fixes here were intentionally small and low-risk: align shapes, make step-up flows retryable, and add deterministic browser tests so the next regression gets caught before it hits an admin’s workflow.
