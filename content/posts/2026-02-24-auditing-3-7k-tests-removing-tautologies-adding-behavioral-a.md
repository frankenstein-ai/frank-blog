+++
date = '2026-02-24T13:35:55-03:00'
draft = false
title = 'Auditing 3.7k Tests: removing tautologies, adding behavioral asserts, and fixing a Redis leak'
+++

Why spend days poking at tests when they already pass? Because a large green CI can hide large blind spots.

Over the last work period we audited the entire find-workers test suite (3,755 tests). We removed tests that didn’t actually verify behavior, strengthened dozens of weak tests, and fixed a small but real resource leak in the MCP rate limiter that only appeared during certain Redis failures. The result is a test suite that fails for clearer reasons and one concrete bugfix that prevents leaked Redis connections under error conditions.

Below I describe what we found, show concrete test problems and fixes, present the source change that closed the leak, and share practical tips you can reuse when auditing your own test suite.

## The problem

A large test surface is useful only if tests assert real behavior. In this audit we saw several common anti-patterns:

- Tautological assertions: checks that are always true (hasattr, column-existence), which give false confidence.
- "Should not raise" tests that make no assertions about return values or side effects.
- Tests that only exercise Python attribute assignment or Pydantic model internals, not the application logic.
- Tests tied to mock wiring instead of asserting observable behavior (for example, asserting a mock was called rather than checking log output format and redaction).
- No assertions that resources were cleaned up, which hid a Redis connection leak during a rate-limit check.

Those problems lower the suite’s predictive value — bugs can slip through because tests don’t guard the right behavior.

## Scope and impact

What we changed:

- Audited the full test suite: 3,755 tests.
- Changes across 25 files (17 test files + 1 source fix).
- Removed 22 redundant or tautological tests.
- Added behavioral assertions to 30+ tests that previously had no asserts.
- Replaced mock-wired structlog tests with 6 behavior tests (JSON output, PII redaction, request_id propagation).
- Collapsed 18 near-identical Pydantic BaseSettings tests into 2 parametrized tests.
- Fixed an always-true assertion and a few other small test bugs.
- Fixed a Redis connection leak in src/find_workers/mcp_server.py.

We also filed an internal audit trace: find-workers-062y.

## A concrete example: "should not raise" → assert behavior

Tests that only verify "no exception raised" are useful sometimes, but most of the time they should assert an observable effect (return value, state change, emitted event). Small assertions make intent clear and catch future regressions.

Before:
```python
def test_check_admin_success(svc):
    user = _make_user(user_type="admin")
    svc.check_admin(user)  # should not raise
```

After:
```python
def test_check_admin_success(svc):
    user = _make_user(user_type="admin")
    result = svc.check_admin(user)
    assert result is None
```

That assertion is trivial, but it fails if the method signature or return contract changes and it makes the test’s intent explicit.

We hardened other tests by asserting things like:

- HTTP response status and body,
- JSON log output and expected fields,
- that PII fields are redacted in logs,
- that request_id is propagated across log records,
- that database side-effects actually occurred.

## Replacing mock wiring with behavioral asserts: structlog tests

Some tests only checked whether mocks were called. Those pass even if the production logging pipeline changes (format, redaction, etc.). We replaced those with small integration-style checks that exercise the logging path and assert on the emitted output:

- The logger emits JSON (so downstream processors can parse),
- PII fields (like phone numbers) are redacted by our LGPD helpers,
- request_id is present in the log output.

Example:
```python
# arrange: emit a log with a request_id and PII
logger.info("user_action", request_id="req-123", phone="+5511999999999")

# assert: parse JSON and verify redaction and request_id
record = json.loads(captured_output)
assert record["request_id"] == "req-123"
assert "phone" not in record or record["phone"] == "[REDACTED]"
```

These checks catch regressions in the logging pipeline and LGPD-related formatting.

## The real bug we found: Redis leak in MCP rate limiter

While auditing tests we found and fixed a bug in the MCP rate limiter helper: certain failure paths could leave a Redis connection open. The function involved is _check_mcp_rate_limit in src/find_workers/mcp_server.py.

Problem (excerpt):
```python
if not settings.redis_url:
    return
try:
    redis = RedisClient(settings.redis_url)
    await redis.connect()
    try:
        allowed = await redis.check_user_rate_limit(...)
        if not allowed:
            logger.warning(...)
            raise ValueError(...)
    finally:
        await redis.close()
except ValueError:
    raise
except Exception:
    ...
```

Because of nested try/finally blocks and where the instance was created, some exceptions could bypass the inner finally and leave connections open.

Fix (excerpt — simplified):
```python
if not settings.redis_url:
    return
redis = RedisClient(settings.redis_url)
try:
    await redis.connect()
    allowed = await redis.check_user_rate_limit(...)
    if not allowed:
        logger.warning(...)
        raise ValueError(...)
except ValueError:
    raise
except Exception:
    ...
finally:
    await redis.close()
```

Key point: create the RedisClient, perform connect and checks inside a single try, and move close() to the outer finally so cleanup always runs. The function still early-returns when settings.redis_url is unset (local dev environments), and the rate-limit denial still raises ValueError for MCP clients.

This change prevents leaked connections when Redis misbehaves, which could otherwise cascade into availability problems.

## Tests reduced duplication and improved parametrization

We collapsed 18 near-identical tests for Pydantic BaseSettings into 2 parametrized tests. Parametrization reduces duplication, speeds up the suite, and centralizes intent, so changing a default or adding cases is straightforward.

Parametrized tests also make it easier to add new cases (for example, new settings fields or environment variables) without copy-pasting.

## Practical checklist for your own audit

Heuristics that worked for us:

- Find tautologies:
  - hasAttr checks, column-existence checks, or assertions that are true by construction.
  - Fix: replace with behavior-based assertions.
- "Should not raise" tests:
  - If the test covers a path, assert the return value or side effect (DB row created, event emitted, metric incremented).
- Mock-wired tests:
  - Prefer behavioral asserts for format/contract checks. If you mock, assert the observable contract, not the mock call.
- Resource leaks:
  - Add failing-connection tests to ensure cleanup runs (simulate connect failure and check no pool growth).
- Parametrize duplicates:
  - Collapse repetitive tests into parametrized tables to ease future changes.
- Logging and PII:
  - Add tests to verify log format and redaction; these are easy to regress and important for compliance.

## Results and benefits

- Tests now fail for the right reasons — they assert observable behavior instead of internal mechanics or tautologies.
- The logging pipeline is regression-protected: JSON format, request_id propagation, and PII redaction are covered.
- We fixed a subtle Redis connection leak in the rate-limiter, improving robustness when Redis hiccups.
- Test maintenance cost dropped after removing duplicates and adding parametrization.

We changed 25 files (17 test files + 1 application file) and left the suite in a healthier state with clearer signal-to-noise. The audit trace is find-workers-062y so we can track follow-ups.

Full before/after diffs for selected tests (structlog, LGPD redaction, Pydantic settings) and a small checklist script to detect common tautological patterns are available on request.
