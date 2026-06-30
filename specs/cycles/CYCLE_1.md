# CYCLE 1 — TinyURL Service v1

**Duration:** 1 week (work estimate: S, < 1 day)
**Spec refs:** [PROJECT.md](../PROJECT.md) · [TECH_SPEC.md](../TECH_SPEC.md)
**Linear feature ticket:** SUJ-21
**Goal:** Ship the full v1 feature set — shorten, redirect, expiry, custom
aliases, click analytics — as a single-process in-memory FastAPI app behind a
swappable `Store` protocol.

---

## STORY-1 — App skeleton, storage interface, health check

**As** a developer, **I want** a runnable FastAPI app with the storage seam and
a health endpoint **so that** later stories have a foundation and smoke tests
can confirm the service is up.

**Effort:** S · **Depends on:** none

**Acceptance Criteria**
- **Given** the app is started with a single uvicorn worker, **when** I `GET /health`, **then** I receive `200` with `{ "status": "ok", "links": 0 }`.
- **Given** the codebase, **when** I inspect the storage layer, **then** there is a `Store` protocol exposing `get` / `put` / `exists` / `record_click` and an `InMemoryStore` implementing it.
- **Given** the data model, **when** I inspect it, **then** `LinkRecord` and `ClickEvent` carry the fields defined in TECH_SPEC §2 with timezone-aware UTC timestamps.

---

## STORY-2 — Create a short link

**As** an API caller, **I want** to `POST /shorten` a long URL with an optional
custom alias and TTL **so that** I get back a short code I can share.

**Effort:** S · **Depends on:** STORY-1

**Acceptance Criteria**
- **Given** a valid `long_url`, **when** I `POST /shorten` with no alias, **then** I receive `201` with a 7-char base62 `code` and a `short_url`.
- **Given** a valid `long_url` and an unused `alias` matching `^[0-9A-Za-z_-]{1,32}$`, **when** I `POST /shorten`, **then** the returned `code` equals my alias.
- **Given** an `alias` already in use or reserved, **when** I `POST /shorten`, **then** I receive `409`.
- **Given** a `ttl_seconds` of N, **when** the link is created, **then** `expires_at = created_at + N` (and is `null` when `ttl_seconds` is omitted).
- **Given** an invalid input (empty/oversized URL, non-http(s) scheme, bad alias format, non-positive `ttl_seconds`), **when** I `POST /shorten`, **then** I receive `400`.

---

## STORY-3 — Redirect and record the click

**As** an end user, **I want** to hit a short code and be redirected to the
original URL **so that** the short link works, while the service records the
click.

**Effort:** S · **Depends on:** STORY-2

**Acceptance Criteria**
- **Given** a live code, **when** I `GET /{code}`, **then** I receive `302` with `Location` set to the long URL.
- **Given** a live code, **when** the redirect is served, **then** `click_count` increments by 1 and a `ClickEvent` (timestamp, referrer, user-agent) is appended.
- **Given** an unknown code, **when** I `GET /{code}`, **then** I receive `404`.
- **Given** an expired code, **when** I `GET /{code}`, **then** I receive `410` and no click is recorded.
- **Given** a link that has received more than 100 clicks, **when** I inspect its stored events, **then** at most the 100 most recent are retained while `click_count` reflects the true total.

---

## STORY-4 — Per-link analytics

**As** an API caller, **I want** to `GET /stats/{code}` **so that** I can see
how often and when a link was used.

**Effort:** S · **Depends on:** STORY-3

**Acceptance Criteria**
- **Given** an existing code, **when** I `GET /stats/{code}`, **then** I receive `200` with `code`, `long_url`, `created_at`, `expires_at`, `expired`, `click_count`, and `recent_clicks`.
- **Given** an expired code, **when** I `GET /stats/{code}`, **then** stats are still returned with `expired: true`.
- **Given** an unknown code, **when** I `GET /stats/{code}`, **then** I receive `404`.
- **Given** a link with more than 100 clicks, **when** I read `recent_clicks`, **then** it contains at most 100 entries, most recent first.

---

## Cycle Exit Criteria
- All four stories' acceptance criteria pass.
- Test coverage ≥ 80% overall (TDD per org testing rules).
- `GET /docs` serves the OpenAPI UI.
- README/run note documents the single-worker constraint and ephemerality.
