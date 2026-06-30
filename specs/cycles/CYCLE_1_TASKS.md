# CYCLE 1 — Task Breakdown

**Spec refs:** [CYCLE_1.md](./CYCLE_1.md) · [TECH_SPEC.md](../TECH_SPEC.md)
**Convention:** task IDs `TASK-1-NNN`; commits reference the task ID per
git-workflow rules.

---

## TASK-1-001 — Project scaffold + health endpoint
- **Story:** STORY-1
- **Description:** Initialize the Python project (FastAPI + uvicorn deps), app
  entrypoint, and a `GET /health` route returning `{ "status": "ok", "links": <count> }`.
- **Acceptance:** `GET /health` returns `200`; app boots with one worker;
  `/docs` reachable.
- **Depends on:** none
- **Effort:** S
- **Files (likely):** `pyproject.toml`, `app/main.py`, `app/__init__.py`

## TASK-1-002 — Data models + Store protocol
- **Story:** STORY-1
- **Description:** Define `LinkRecord` and `ClickEvent` (TECH_SPEC §2) and the
  `Store` protocol (`get` / `put` / `exists` / `record_click`). UTC, tz-aware.
- **Acceptance:** Models carry all specified fields with correct types; protocol
  defined with no concrete backend yet.
- **Depends on:** TASK-1-001
- **Effort:** S
- **Files (likely):** `app/models.py`, `app/store/base.py`

## TASK-1-003 — InMemoryStore implementation
- **Story:** STORY-1
- **Description:** Dict-backed `Store` impl; `record_click` increments
  `click_count` and appends a `ClickEvent`, trimming `clicks` to the 100 most
  recent.
- **Acceptance:** Unit tests cover put/get/exists, click increment, and the
  100-event cap with `click_count` preserved.
- **Depends on:** TASK-1-002
- **Effort:** S
- **Files (likely):** `app/store/memory.py`, `tests/test_store.py`

## TASK-1-004 — Code generation + alias/URL validation
- **Story:** STORY-2
- **Description:** 7-char base62 random code generator with bounded
  collision-retry; alias validator (`^[0-9A-Za-z_-]{1,32}$`); reserved-word set
  (`health`, `docs`, `openapi.json`, `stats`); http/https scheme + length
  allowlist for `long_url`.
- **Acceptance:** Unit tests for code format, collision retry, alias regex,
  reserved rejection, scheme/length rejection.
- **Depends on:** TASK-1-002
- **Effort:** S
- **Files (likely):** `app/service/codes.py`, `app/service/validation.py`, `tests/test_validation.py`

## TASK-1-005 — POST /shorten route + service logic
- **Story:** STORY-2
- **Description:** Request/response Pydantic models; service wiring code-gen,
  validation, TTL→`expires_at`, alias collision; error mapping to
  `400`/`409`/`422`; returns `201`.
- **Acceptance:** STORY-2 ACs pass (auto + custom alias, TTL, 400/409 paths).
- **Depends on:** TASK-1-003, TASK-1-004
- **Effort:** S
- **Files (likely):** `app/routes.py`, `app/schemas.py`, `app/service/links.py`, `tests/test_shorten.py`

## TASK-1-006 — GET /{code} redirect + expiry + click capture
- **Story:** STORY-3
- **Description:** Resolve code; lazy expiry check (`410` if past
  `expires_at`); `302` with `Location`; record click (timestamp, `Referer`,
  `User-Agent`); `404` unknown.
- **Acceptance:** STORY-3 ACs pass (302 happy path, click recorded, 404, 410,
  no click on expired).
- **Depends on:** TASK-1-005
- **Effort:** S
- **Files (likely):** `app/routes.py`, `tests/test_redirect.py`

## TASK-1-007 — GET /stats/{code}
- **Story:** STORY-4
- **Description:** Return analytics payload (TECH_SPEC §3) incl. `expired` flag
  and `recent_clicks` (≤100, most-recent-first); `404` unknown; stats served
  for expired links.
- **Acceptance:** STORY-4 ACs pass.
- **Depends on:** TASK-1-006
- **Effort:** S
- **Files (likely):** `app/routes.py`, `app/schemas.py`, `tests/test_stats.py`

## TASK-1-008 — Integration tests + run/README note
- **Story:** all
- **Description:** End-to-end flow test (`shorten` → redirect → stats) and
  error-path coverage; document single-worker + ephemeral constraints and run
  command. Verify ≥80% coverage.
- **Acceptance:** Cycle exit criteria met; coverage gate passes.
- **Depends on:** TASK-1-007
- **Effort:** S
- **Files (likely):** `tests/test_integration.py`, `README.md`

---

## Dependency Graph

```mermaid
graph TD
    T1[TASK-1-001 scaffold + health]
    T2[TASK-1-002 models + Store protocol]
    T3[TASK-1-003 InMemoryStore]
    T4[TASK-1-004 code-gen + validation]
    T5[TASK-1-005 POST /shorten]
    T6[TASK-1-006 GET /{code} redirect]
    T7[TASK-1-007 GET /stats/{code}]
    T8[TASK-1-008 integration + README]

    T1 --> T2
    T2 --> T3
    T2 --> T4
    T3 --> T5
    T4 --> T5
    T5 --> T6
    T6 --> T7
    T7 --> T8
```

**Critical path:** 001 → 002 → {003, 004} → 005 → 006 → 007 → 008.
TASK-1-003 and TASK-1-004 can be done in parallel once 002 lands.
