# TECH SPEC: TinyURL Service (toy / local)

Implements Approach 1 from [PROJECT.md](./PROJECT.md): single-process FastAPI app
with an in-memory store hidden behind a narrow storage interface.

## 1. Architecture Overview

```
            ┌─────────────────────────────────────────────┐
            │                FastAPI app                   │
            │                                              │
  client ──▶│  Routes  ──▶  Service layer  ──▶  Store      │
            │  (HTTP)       (validation,        (Protocol) │
            │               code-gen, TTL,                 │
            │               analytics)              │      │
            └───────────────────────────────────────┼──────┘
                                                     │
                                       ┌─────────────▼─────────────┐
                                       │  InMemoryStore (v1)        │
                                       │  dict: code -> LinkRecord  │
                                       └────────────────────────────┘
                                       (swap seam: SQLiteStore later)
```

- **Routes layer:** HTTP concerns only — request/response models, status codes.
- **Service layer:** owns business rules — code generation, alias validation,
  collision handling, expiry checks, analytics recording.
- **Store (Protocol):** narrow interface — `get`, `put`, `exists`,
  `record_click`. v1 implementation is an in-memory dict. The interface is the
  documented upgrade seam to SQLite/Redis without touching routes or service.

Single process, single uvicorn worker (see Constraints). No background threads.

## 2. Data Model

### LinkRecord (stored value, keyed by short `code`)

| Field          | Type                | Notes                                              |
|----------------|---------------------|----------------------------------------------------|
| `code`         | string              | Short code (the map key). 1–32 chars.              |
| `long_url`     | string              | Destination URL. Must be http/https.               |
| `created_at`   | datetime (UTC)      | Set at creation.                                   |
| `expires_at`   | datetime (UTC)/null | Null = never expires.                              |
| `is_custom`    | bool                | True if caller supplied the alias.                 |
| `click_count`  | int                 | Incremented on each successful redirect.           |
| `clicks`       | list of ClickEvent  | Append-only; capped (see Constraints) to bound RAM.|

### ClickEvent

| Field        | Type           | Notes                                  |
|--------------|----------------|----------------------------------------|
| `timestamp`  | datetime (UTC) | When the redirect was served.          |
| `referrer`   | string/null    | From `Referer` header, if present.     |
| `user_agent` | string/null    | From `User-Agent` header, if present.  |

All timestamps are timezone-aware UTC. The map and all records live in process
memory and are discarded on shutdown.

### Short-code generation

- Default: base62 (`[0-9A-Za-z]`) token, length 7 (≈ 3.5e12 space — ample for a
  toy). Generated randomly; on the rare collision, regenerate (bounded retries).
- Custom alias: caller-supplied; validated against `^[0-9A-Za-z_-]{1,32}$`;
  rejected with `409` if already taken.
- Reserved codes (cannot be used as aliases): `health`, `docs`, `openapi.json`,
  `stats` and any path segment the API itself owns.

## 3. API Contracts

Base URL assumed `http://localhost:8000`. All request/response bodies are JSON
except the redirect.

### POST /shorten — create a short link

Request body:

```json
{
  "long_url": "https://example.com/some/very/long/path?q=1",
  "alias": "my-link",
  "ttl_seconds": 3600
}
```

- `long_url` (required): http/https URL, max length 2048.
- `alias` (optional): custom code matching `^[0-9A-Za-z_-]{1,32}$`.
- `ttl_seconds` (optional): positive integer; sets `expires_at = now + ttl`.
  Omitted/null = never expires.

Success `201 Created`:

```json
{
  "code": "my-link",
  "short_url": "http://localhost:8000/my-link",
  "long_url": "https://example.com/some/very/long/path?q=1",
  "created_at": "2026-06-30T12:00:00Z",
  "expires_at": "2026-06-30T13:00:00Z"
}
```

Errors:

| Status | Condition                                              |
|--------|--------------------------------------------------------|
| `400`  | Missing/invalid `long_url`, non-http(s) scheme, bad `alias` format, non-positive `ttl_seconds`. |
| `409`  | `alias` already in use, or reserved.                   |
| `422`  | Body fails schema validation (FastAPI default).        |

### GET /{code} — redirect to the long URL

- `302 Found` with `Location: <long_url>`; records a ClickEvent and increments
  `click_count`.
- `404 Not Found` if the code is unknown.
- `410 Gone` if the link exists but `expires_at` has passed (expired links are
  treated as dead; not auto-deleted in v1 — see Constraints).

### GET /stats/{code} — analytics for one link

Success `200 OK`:

```json
{
  "code": "my-link",
  "long_url": "https://example.com/...",
  "created_at": "2026-06-30T12:00:00Z",
  "expires_at": "2026-06-30T13:00:00Z",
  "expired": false,
  "click_count": 12,
  "recent_clicks": [
    { "timestamp": "2026-06-30T12:05:00Z", "referrer": "https://news.example", "user_agent": "Mozilla/5.0 ..." }
  ]
}
```

- `404` if code unknown. `recent_clicks` is capped to the most recent N (see
  Constraints). Stats are readable even for expired links.

### GET /health — liveness

- `200 OK` → `{ "status": "ok", "links": <count> }`. Used for smoke tests.

OpenAPI docs auto-served at `/docs` and `/openapi.json` (FastAPI default).

## 4. Validation & Edge Cases

| Case                                   | Behavior                                  |
|----------------------------------------|-------------------------------------------|
| Empty / whitespace `long_url`          | `400`.                                    |
| Non-http(s) scheme (e.g. `javascript:`)| `400` — only `http`/`https` accepted.     |
| `long_url` > 2048 chars                | `400`.                                    |
| `alias` collides with existing code    | `409`.                                    |
| `alias` matches a reserved word        | `409`.                                    |
| `ttl_seconds` = 0 or negative          | `400`.                                    |
| Redirect to expired link               | `410`.                                    |
| Lookup of unknown code                 | `404`.                                    |
| Duplicate `long_url` (no alias)        | Allowed — creates a second distinct code. |

## 5. Security Baseline

- **Input validation:** every field validated at the boundary (Pydantic models);
  URL scheme allowlist (`http`, `https` only) to block `javascript:`/`data:`
  redirect abuse and open-redirect-to-script vectors.
- **No secrets:** the app holds no credentials; nothing to leak. No `.env`
  required for v1.
- **Open redirect:** inherent to any shortener — documented as accepted for a
  toy. Mitigation deferred (allowlist/denylist is out of scope per PROJECT.md).
- **Injection:** none — no SQL, no shell, no template rendering of user input in
  v1. If upgraded to SQLite, all queries MUST be parameterized.
- **Logging:** do not log full URLs at INFO if they may carry tokens in query
  strings; log the `code` instead. Never log headers verbatim beyond the
  bounded analytics capture.
- **DoS note:** unbounded link/click growth is a memory-exhaustion vector;
  bounded by the caps in Constraints. No rate limiting in v1 (single user).

## 6. Performance Considerations

- All operations are O(1) dict lookups; no indexing needed at this scale.
- Click capture is an in-memory append; per-link `clicks` list is capped to
  bound memory. `click_count` is a separate integer so the lifetime count is
  exact even after the per-link event list is trimmed.
- No caching layer needed — the store *is* memory.

## 7. Constraints & Assumptions

- **Single worker only.** Run uvicorn with one worker; multiple workers each get
  an independent dict and split the state. This is a documented limitation, not
  a bug.
- **Ephemeral.** All data lost on restart, by design.
- **Click history cap:** keep at most the **100** most recent ClickEvents per
  link (`recent_clicks`); `click_count` remains a true running total.
- **Lazy expiry:** expired links are detected on access (redirect/stats) and
  reported as `410`/`expired:true`; no background sweeper in v1. Memory for
  expired links is reclaimed only if a future cleanup endpoint is added (out of
  scope).
- **Base URL** for `short_url` construction is derived from the incoming request
  (host header) or a single config value; defaults to `http://localhost:8000`.

## 8. Testing Notes (for Developer)

Per org testing rules — TDD, ≥80% coverage:
- Unit: code generation (format, collision retry), alias validation, expiry
  logic, reserved-word rejection, click-cap trimming.
- Integration: full `POST /shorten` → `GET /{code}` → `GET /stats/{code}`
  happy path; 400/404/409/410 paths; expired-link redirect.
- Edge: empty/oversized URL, bad scheme, zero/negative TTL, duplicate alias.

## 9. Upgrade Seam (non-goal for v1, documented)

The `Store` Protocol (`get` / `put` / `exists` / `record_click`) is the single
point of change to move to SQLite (durable analytics) or Redis (native TTL +
atomic counters). Routes and service layer remain untouched. This is the
recommended first follow-up if durability is ever required.
