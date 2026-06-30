# PROJECT: TinyURL Service (toy / local)

## Problem Statement

We need a minimal URL-shortening service that runs locally and exposes a REST
API. A caller submits a long URL (optionally with a custom short code and an
expiry) and receives a short code. Hitting the short code redirects to the
original URL and records a click event. The service is explicitly a toy: single
user, in-memory storage, no durability or concurrency guarantees. The goal is a
clean, well-shaped API and data model that could later be upgraded to a real
datastore with minimal churn.

## Requirements Summary (from interview)

- **Scale:** Toy / local. Single user, hundreds of URLs, low concurrency.
- **Storage:** In-memory only. Links and analytics are lost on restart.
- **Interface:** REST API only (FastAPI). No web UI, no CLI.
- **Features in scope:** custom aliases, expiration/TTL, click analytics.
- **Redirect:** `302` (temporary) so analytics keep firing and clients don't
  cache the mapping permanently.

## Solution Approaches

### Approach 1 — Single-process FastAPI app, in-memory dict store (RECOMMENDED)

A FastAPI application with a thin storage layer backed by a plain in-memory
mapping (`code -> record`). Short codes auto-generated as base62 from a counter
or random token; custom aliases written into the same map. Analytics held in the
record. A storage interface (protocol) sits between the API and the dict so the
backend can be swapped later.

- **Pros:** Smallest possible footprint; no external dependencies beyond
  FastAPI/uvicorn; trivial to run; storage interface keeps the upgrade path open.
- **Cons:** No persistence; not safe across multiple worker processes (each
  worker has its own dict); no horizontal scaling.
- **Effort:** S (< 1 day).
- **Risk:** Low. The only real risk is someone running it with multiple uvicorn
  workers and being confused by split state — documented as a constraint.

### Approach 2 — FastAPI + SQLite via SQLAlchemy

Same API surface, but persistence through a single SQLite file. Survives
restarts; analytics durable; still single-machine.

- **Pros:** Durable; analytics survive restarts; still near-zero ops; clean
  migration target for the storage interface in Approach 1.
- **Cons:** More than the user asked for; adds SQLAlchemy + migration concerns;
  SQLite write-locking under concurrency.
- **Effort:** M (1–3 days).
- **Risk:** Low–Medium. Schema/migration overhead unjustified for a toy.

### Approach 3 — FastAPI + Redis

Store mappings and counters in Redis, using native TTL for expiry and atomic
`INCR` for click counts and code generation.

- **Pros:** TTL and atomic counters come for free; multi-process safe; fast.
- **Cons:** Requires running Redis; operationally heavier than a toy warrants;
  analytics beyond simple counters need extra structures.
- **Effort:** M (1–3 days).
- **Risk:** Medium. External dependency contradicts the "runs locally, zero
  config" intent.

## Recommendation

**Approach 1.** It matches the stated scope exactly (toy, local, in-memory,
REST-only) and is the smallest thing that fully satisfies the feature set. The
key design move is to hide the dict behind a narrow storage interface so that
Approach 2 (SQLite) becomes a single-file swap if durability is ever wanted —
this is called out explicitly in the tech spec as the upgrade seam.

## Out of Scope (v1)

- Persistence across restarts (links and analytics are intentionally ephemeral).
- Authentication, accounts, and per-user link ownership.
- Web UI and CLI.
- Multi-process / multi-host deployment and horizontal scaling.
- Rate limiting, abuse/malware-URL detection, and CAPTCHA.
- Custom domains and vanity hostnames.
- Bulk import/export and link editing after creation.

## Estimate

Overall: **S** (< 1 day) for Approach 1 including tests.
