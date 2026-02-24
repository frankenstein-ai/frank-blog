+++
date = '2026-02-21T18:05:24-03:00'
draft = false
title = 'Small fixes, big wins: worker-service CRUD, schema alignment, and correct search counts'
+++

Summary
-------
In 2026-W08 we shipped three focused fixes to Find Workers that reduce production errors and improve provider UX:

- Added PATCH and DELETE endpoints for worker service offerings, with ownership checks and stricter request validation (9 new tests).
- Tightened Pydantic max_length constraints to match DB VARCHAR widths so invalid input fails fast with 422 instead of blowing up at persistence with 500.
- Fixed an inflated search-count bug caused by JOIN + DISTINCT + count(*) OVER(); replaced the pattern with an IN subquery so pagination totals are correct.

These are small, targeted changes that remove real friction: providers can update or remove services, clients get consistent API errors for invalid input, and search totals used for paging and UI counts are accurate. The full test suite remains green: 3,630 tests passing.

Why these issues mattered
-------------------------
1) Missing update/delete for worker services
- Providers could create service offerings but could not update price, description, experience, or deactivate/remove outdated offerings via the API.
- Re-creating the same category was blocked by a unique constraint, so there was no practical workaround.

Result: frustrated providers, stale marketplace data, and more support tickets.

2) Pydantic / DB mismatch on max_length
- Some API schemas allowed longer strings than the corresponding DB columns (for example, profile_photo_url: schema 2048 chars vs DB VARCHAR(500)).
- Requests that passed API validation then failed during DB persistence, returning 500 errors instead of a clear 422.

Result: confusing errors for clients and noisy logs that hid the real cause.

3) Search counts inflated
- The search query joined worker_services and used DISTINCT to pick unique workers, but count(*) OVER() was computed before DISTINCT. A worker that matched multiple services contributed multiple rows to the windowed count.
- That produced inflated totals in search responses, which broke pagination and showed misleading UI counters.

What we implemented
-------------------

## 1) Worker service update & delete endpoints

Problem: create existed; update and delete were missing.

What we added:
- PATCH /v1/workers/{worker_id}/services/{service_id}
- DELETE /v1/workers/{worker_id}/services/{service_id}

Key points:
- Ownership verification: requests must come from the worker owner. Unauthorized calls return 403.
- Strict validation: UpdateServiceRequest uses Pydantic extra="forbid" to reject unexpected fields so typos or deprecated fields don't get ignored.
- Service-layer methods (update_service and delete_service) return clear 404 / 403 semantics.

Example Pydantic schema:

```python
from typing import Optional
from pydantic import BaseModel, Field

class UpdateServiceRequest(BaseModel):
    price_cents: Optional[int] = Field(None, ge=0)
    description: Optional[str] = Field(None, max_length=1000)
    experience_years: Optional[int] = Field(None, ge=0)
    is_active: Optional[bool] = None

    class Config:
        extra = "forbid"
```

Conceptual API handler:

```python
@router.patch("/v1/workers/{worker_id}/services/{service_id}")
def patch_service(worker_id: int, service_id: int, payload: UpdateServiceRequest, current_user=Depends(get_current_user)):
    worker = worker_service_layer.get_worker(worker_id)
    if worker.owner_id != current_user.id:
        raise HTTPException(status_code=403, detail="Not owner")
    updated = worker_service_layer.update_service(worker_id, service_id, payload.dict(exclude_none=True))
    if not updated:
        raise HTTPException(status_code=404, detail="Service not found")
    return updated  # 200 with updated resource
```

Testing:
- Added 9 tests covering success paths and failure modes (200/204, 403 owner checks, 404 not found).
- Test suite includes an explicit test that extra fields are rejected.

Why extra="forbid" matters
- It enforces the API contract: clients get a clear 422 when they send unexpected fields instead of silent acceptance that leads to downstream confusion.

## 2) Align Pydantic max_length with DB columns

Problem: schema max_length exceeded DB column sizes for several fields, allowing requests that later failed in the DB.

Examples of mismatches:
- scheduled_time_slot: schema max_length 50, DB VARCHAR(20)
- profile_photo_url: schema max_length 2048, DB VARCHAR(500)
- time_slot (availability): schema max_length 50, DB VARCHAR(20)
- given_via (LGPD consent): schema max_length 100, DB VARCHAR(20)

Fix:
- Tightened the Pydantic max_length values to match the DB column widths so invalid payloads fail at validation time.

Before -> after example:

```python
# Before
profile_photo_url: Optional[str] = Field(None, max_length=2048)

# After
profile_photo_url: Optional[str] = Field(None, max_length=500)
```

Why it helps
- Clients receive faster, clearer errors.
- Application logs are less noisy and fewer 500s need investigation.
- Front-end and API consumers can rely on validation to produce actionable messages.

## 3) Fix search-count inflation using IN subquery

Problem: count(*) OVER() was calculated before DISTINCT, so joins that produced multiple rows per worker inflated the total.

Root cause:
- The query joined worker_services and then applied DISTINCT to worker rows, but the windowed count was already computed on the multiplied rows.

Fix implemented:
- Replace the JOIN + DISTINCT approach with a WHERE worker.id IN (subquery) pattern.
- The subquery selects DISTINCT worker_ids that match the service filters; the outer query selects unique workers and the windowed count matches unique rows.

Conceptual before/after:

Before (problematic):

```sql
SELECT w.*, count(*) OVER() AS total
FROM workers w
JOIN worker_services ws ON ws.worker_id = w.id
WHERE ws.service_category_id IN (1,2,3)
DISTINCT
LIMIT 20 OFFSET 0;
```

After (fixed):

```sql
SELECT w.*, count(*) OVER() AS total
FROM workers w
WHERE w.id IN (
  SELECT DISTINCT ws.worker_id
  FROM worker_services ws
  WHERE ws.service_category_id IN (1,2,3)
)
LIMIT 20 OFFSET 0;
```

SQLAlchemy-like pseudocode:

```python
subq = (
    db.query(WorkerService.worker_id)
    .filter(WorkerService.service_category_id.in_(service_ids))
    .distinct()
    .subquery()
)

q = db.query(Worker, func.count().over().label("total")).filter(Worker.id.in_(subq))
```

Result:
- The total returned to clients is the count of unique workers that match the filters. Pagination and UI totals are now correct.

Testing:
- Updated tests/test_workers_search.py to assert correct totals when workers have multiple matching services.
- Full test suite remained green: 3,630 tests passing.

Engineering and process notes
-----------------------------
- Each fix included small, focused tests covering both success and failure cases.
- Issues and acceptance criteria were tracked in .beads/issues.jsonl so the repo documents why each change was made.
- We preferred conservative, well-scoped changes (schema tightening, ownership checks, safer query shape) over sweeping refactors to keep reviews and testing fast.

Lessons learned
---------------
1. Align API validation with persistence constraints
- Let validation reject invalid payloads instead of pushing errors into the DB layer and producing opaque 500s.

2. Watch JOINs with window functions
- count(*) OVER() can surprise you when joins multiply rows. If you need unique entities, de-duplicate before applying window functions.

3. Verify ownership on mutating endpoints
- Mutating endpoints must enforce server-side ownership checks. Tests should cover 403 and 404 cases to prevent privilege escalation.

4. Forbid unknown fields when the contract matters
- extra="forbid" helps catch client bugs early and prevents accidental acceptance of deprecated or misspelled fields.

Next steps
----------
- Add a CI check that flags mismatches between Pydantic schemas and DB column widths.
- Consider an SQL lint or rule that warns when window functions are used with joins that can multiply rows.
- Monitor 5xx rates after schema tightening to catch legacy clients and roll out fixes gradually.

Appendix — useful snippets
--------------------------

UpdateServiceRequest:

```python
class UpdateServiceRequest(BaseModel):
    price_cents: Optional[int] = Field(None, ge=0)
    description: Optional[str] = Field(None, max_length=1000)
    experience_years: Optional[int] = Field(None, ge=0)
    is_active: Optional[bool] = None

    class Config:
        extra = "forbid"
```

Search fix pseudocode:

```python
# Subquery to find matching worker ids without multiplying rows
subq = (
    db.query(WorkerService.worker_id)
    .filter(WorkerService.service_category_id.in_(service_ids))
    .distinct()
    .subquery()
)

# Outer query returns unique worker rows and an accurate windowed total
q = (
    db.query(Worker, func.count().over().label("total"))
    .filter(Worker.id.in_(subq))
    .order_by(Worker.score.desc())
    .limit(per_page)
    .offset(offset)
)
```

If you maintain APIs that mix domain data and search filters, these fixes pay off quickly: fewer support tickets, clearer errors, and UI metrics you can trust.
