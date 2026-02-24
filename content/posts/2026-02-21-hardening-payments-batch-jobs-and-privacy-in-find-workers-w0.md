+++
date = '2026-02-21T23:02:32-03:00'
draft = false
title = 'Hardening payments, batch jobs and privacy in Find Workers — W08 recap'
+++

This sprint focused on three high-risk areas for marketplaces that move money and personal data: payments reliability, long-running batch jobs, and admin/data-privacy tooling. The work touches DB schema, concurrency control, webhook handling, auth invariants and LGPD (Brazil's data protection law) compliance. All changes were validated with the repo's integration and unit tests.

Problems we addressed

- Payments reliability: Pix gateway webhooks (Woovi/OpenPix) can be transient, reordered, or retried. Batch payout processors could double-disburse funds when multiple workers ran concurrently. Missing indexes made queries slow under load.
- Unbounded background jobs: expire charges, reconcilers and stale-booking cancellation could scan or lock large parts of the DB and reprocess rows.
- Data-privacy and admin tooling: admin surfaces needed better auditability, step-up protection and safer PII handling (masking and deletion).
- Operational and developer pain: SQLAlchemy typing/format issues, missing model columns, and webhook rate-limit fallbacks caused flakiness in tests and local runs.
