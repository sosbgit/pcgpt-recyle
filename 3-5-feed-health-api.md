# Story 3.5: Feed Health API

**Status:** ready-for-dev
**Epic:** 3 — Live Feed Ingestion & Feed Health Visibility
**Sprint sequence:** Fifth story of Epic 3; unblocks 3.6 (Signal Desk feed health UI)

---

## Story

As the Panoptes API service,
I need to expose a `GET /api/v1/feeds/health` endpoint that derives per-provider feed status by checking the presence and TTL of the `feed:health:{provider}` key in Valkey,
so that consumers (and future Signal Desk UI in Story 3.6) can query live feed status — with a stale feed surfacing as explicit `"stale"`, never silently absent or assumed healthy.

---

## Acceptance Criteria

### AC 1 — `shared/schema/feeds.py`: update `FeedHealth.ttl_seconds` to `Optional[int]`

- `shared/schema/feeds.py` is updated with a targeted change to `FeedHealth`:
  - Change `ttl_seconds: int = 30` to `ttl_seconds: Optional[int] = None`.
  - This accurately represents remaining TTL from Valkey (a positive integer when the key is live, `None` when the key is absent/stale or has no TTL).
  - All other fields and classes in `feeds.py` are unchanged.

**After change, `FeedHealth` is:**
```python
class FeedHealth(BaseModel):
    provider: str                        # e.g. "opensky"
    status: Literal["live", "stale"]    # derived from Valkey TTL key presence
    last_heartbeat_at: Optional[datetime] = None  # None — key stores "alive" only, no timestamp
    ttl_seconds: Optional[int] = None   # remaining Valkey TTL when live; None when stale or unknown
```

`FeedHealthResponse` is unchanged:
```python
class FeedHealthResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    feeds: list[FeedHealth]
    checked_at: datetime
```

### AC 2 — `services/api/api/routers/feeds.py`: new router

- `services/api/api/routers/feeds.py` is created.
- Defines module-level constant:
  ```python
  KNOWN_PROVIDERS: list[str] = ["opensky"]
  ```
  This is the registry of providers whose heartbeat keys are checked. Epic 9 will extend it when ADS-B Exchange and AIS NOAA adapters are added.
- Defines `router = APIRouter(prefix="/feeds", tags=["feeds"])`.
- Implements `GET /feeds/health` (full path: `GET /api/v1/feeds/health`):
  ```python
  @router.get("/health")
  async def get_feed_health(
      _: str = Depends(verify_token),
      redis: Redis = Depends(get_redis),
  ) -> JSONResponse:
  ```
  - Requires authentication via `Depends(verify_token)` — consistent with all non-auth API routes.
  - Receives a Valkey client via `Depends(get_redis)` — the existing connection pool from `api/redis_client.py` is reused. No new connection setup.
  - For each provider in `KNOWN_PROVIDERS`, calls `ttl = await redis.ttl(f"feed:health:{provider}")`.
  - TTL derivation logic:
    - `ttl > 0` → `status = "live"`, `ttl_seconds = ttl` (remaining seconds on the key)
    - `ttl == -2` → key absent (ingestor not running or has been silent > 30s) → `status = "stale"`, `ttl_seconds = None`
    - `ttl == -1` → key exists but has no expiry (anomalous — ingestor wrote the key without TTL) → `status = "live"`, `ttl_seconds = None`; log a WARNING once per occurrence: `"feed:health:{provider} key exists with no TTL — expected SETEX, got SET"`
  - `last_heartbeat_at` is always `None`: the Valkey key stores `"alive"` only (no timestamp). No computation of approximate heartbeat time — leave this for a future enhancement if needed.
  - Constructs `FeedHealthResponse(feeds=[...], checked_at=datetime.now(UTC))`.
  - Returns `JSONResponse(content=success_envelope(response.model_dump(mode="json")))`.
  - No DB session — this endpoint is Valkey-only. Do not inject `get_db`.
  - No exception handler needed: Valkey connection errors from `get_redis()` pool are surfaced as 500 via the existing `PanoptesError` handler. Individual TTL calls silently return `"stale"` if Valkey is unreachable (see AC 2 note below).

**Valkey error handling within the endpoint (non-fatal per-provider):**
If `redis.ttl(key)` raises `redis.exceptions.ConnectionError` or similar for a specific provider:
  - Log at WARNING: `"Valkey TTL check failed for feed:health:{provider}: {error}"`
  - Treat as `status = "stale"`, `ttl_seconds = None` for that provider.
  - Continue checking remaining providers.
  - Do NOT propagate the exception — the endpoint must return a complete response even when Valkey is degraded. A Valkey outage IS itself a feed health event.

### AC 3 — `services/api/api/main.py`: wire feeds router

- `services/api/api/main.py` is updated with a targeted addition only.
- Add `feeds` to the router import:
  ```python
  from api.routers import auth, geography, watches, feeds
  ```
- Add router registration after the existing `geography` and `watches` router registrations:
  ```python
  app.include_router(feeds.router, prefix="/api/v1")
  ```
- No other changes to `main.py`.

### AC 4 — `services/api/tests/test_feeds.py`: unit test suite

- `services/api/tests/test_feeds.py` is created.
- All tests use `app.dependency_overrides` to mock `verify_token` and `get_redis` — no live Valkey required.
- Test pattern for mocking `get_redis`:
  ```python
  from api.redis_client import get_redis

  def override_redis_with_ttl(ttl_values: dict[str, int]):
      """ttl_values maps key → TTL return value from redis.ttl()."""
      async def _get_redis():
          mock = AsyncMock()
          mock.ttl = AsyncMock(side_effect=lambda key: ttl_values.get(key, -2))
          return mock
      return _get_redis
  ```
- `verify_token` override (consistent with existing test pattern):
  ```python
  async def _override_token():
      return "operator"
  ```
- Fixture:
  ```python
  @pytest.fixture()
  def f_client():
      app.dependency_overrides[verify_token] = _override_token
      with TestClient(app) as c:
          yield c
      app.dependency_overrides.clear()
  ```

**Required test coverage:**

**Group F — `GET /api/v1/feeds/health`**

- **F1: live feed → status=live with remaining TTL**
  - Mock `redis.ttl("feed:health:opensky")` returns `18`.
  - `GET /api/v1/feeds/health` returns 200.
  - Body: `data.feeds[0].provider == "opensky"`, `data.feeds[0].status == "live"`, `data.feeds[0].ttl_seconds == 18`.

- **F2: stale feed (key absent) → status=stale**
  - Mock `redis.ttl("feed:health:opensky")` returns `-2` (key missing).
  - `GET /api/v1/feeds/health` returns 200.
  - Body: `data.feeds[0].status == "stale"`, `data.feeds[0].ttl_seconds is None`.

- **F3: anomalous key with no TTL → status=live, ttl_seconds=None**
  - Mock `redis.ttl("feed:health:opensky")` returns `-1`.
  - `GET /api/v1/feeds/health` returns 200.
  - Body: `data.feeds[0].status == "live"`, `data.feeds[0].ttl_seconds is None`.

- **F4: all KNOWN_PROVIDERS present in response**
  - `data.feeds` has exactly `len(KNOWN_PROVIDERS)` entries.
  - Each entry has `provider`, `status`, `ttl_seconds`, `last_heartbeat_at` keys.

- **F5: response envelope shape**
  - Response body: `data` key is a dict with `feeds` list and `checked_at` ISO string.
  - `meta.timestamp` is present and is an ISO-8601 string.
  - `error` is `None`.

- **F6: authentication required**
  - Without `verify_token` override, `GET /api/v1/feeds/health` returns 401.
  - In a fresh `TestClient(app)` without dependency overrides, call the endpoint without a Bearer token.
  - Assert response status == 401.

- **F7: Valkey ConnectionError → stale (non-fatal)**
  - Mock `redis.ttl()` raises `redis.exceptions.ConnectionError`.
  - `GET /api/v1/feeds/health` returns 200 (not 500).
  - Body: `data.feeds[0].status == "stale"`.

- **F8: `last_heartbeat_at` is always None**
  - Mock `redis.ttl("feed:health:opensky")` returns `25` (live).
  - Body: `data.feeds[0].last_heartbeat_at is None`.

- `pytest services/api/tests/ -v` — all prior tests (auth, watches, geography, dependencies) continue to pass unmodified alongside new Group F tests.

### AC 5 — Smoke test: end-to-end

- With Valkey running (`docker compose up valkey -d`) and the ingestor running (`python -m ingestor.main` with PROVIDER=opensky):
  ```bash
  # From repo root, with .env loaded
  uvicorn services.api.api.main:app --reload --host 0.0.0.0 --port 8000 --env-file .env

  # Login first
  curl -s -X POST http://localhost:8000/api/v1/auth/login \
    -H "Content-Type: application/json" \
    -d '{"username":"operator","password":"<your-password>"}' | jq .

  # Verify feed health (with token from above)
  curl -s http://localhost:8000/api/v1/feeds/health \
    -H "Authorization: Bearer <token>" | jq .
  ```
  - With ingestor running: `data.feeds[0].status == "live"`, `data.feeds[0].ttl_seconds` is a positive integer ≤ 30.
  - With ingestor stopped (wait >30s for key to expire): `data.feeds[0].status == "stale"`, `data.feeds[0].ttl_seconds == null`.
  - **Do NOT run the live smoke test in CI.** Local manual verification only.

---

## Tasks / Subtasks

- [ ] **Task 1: Update `shared/schema/feeds.py`** (AC 1)
  - [ ] 1.1 Change `ttl_seconds: int = 30` to `ttl_seconds: Optional[int] = None` in `FeedHealth`
  - [ ] 1.2 Verify `Optional` is already imported in `feeds.py` (it is — `from typing import Literal, Optional`)
  - [ ] 1.3 Verify `FeedHealthResponse` is unchanged; all other models in `feeds.py` are unchanged
  - [ ] 1.4 Verify no other service or test currently imports `FeedHealth.ttl_seconds` with expectation of `int` default (this is a new model, no consumers yet)

- [ ] **Task 2: Create `services/api/api/routers/feeds.py`** (AC 2)
  - [ ] 2.1 Import: `logging`, `from datetime import UTC, datetime`, `from typing import Annotated`
  - [ ] 2.2 Import: `from fastapi import APIRouter, Depends`, `from fastapi.responses import JSONResponse`
  - [ ] 2.3 Import: `import redis.asyncio as aioredis`, `import redis.exceptions`
  - [ ] 2.4 Import: `from api.dependencies import verify_token`, `from api.exceptions import success_envelope`, `from api.redis_client import get_redis`
  - [ ] 2.5 Import: `from schema.feeds import FeedHealth, FeedHealthResponse`
  - [ ] 2.6 Define module logger: `log = logging.getLogger("api.feeds")`
  - [ ] 2.7 Define `KNOWN_PROVIDERS: list[str] = ["opensky"]`
  - [ ] 2.8 Define `router = APIRouter(prefix="/feeds", tags=["feeds"])`
  - [ ] 2.9 Implement `async def get_feed_health(...)` with `@router.get("/health")` decorator
  - [ ] 2.10 For each provider in KNOWN_PROVIDERS: call `await redis.ttl(f"feed:health:{provider}")`, apply TTL derivation logic, construct `FeedHealth` object
  - [ ] 2.11 Wrap Valkey TTL call in try/except `redis.exceptions.ConnectionError` (and broad `Exception` fallback with WARNING log) → treat as stale, continue loop
  - [ ] 2.12 Construct `FeedHealthResponse(feeds=feeds, checked_at=datetime.now(UTC))`
  - [ ] 2.13 Return `JSONResponse(content=success_envelope(response.model_dump(mode="json")))`
  - [ ] 2.14 Verify: no `get_db` / DB session anywhere in this file

- [ ] **Task 3: Update `services/api/api/main.py`** (AC 3)
  - [ ] 3.1 Add `feeds` to the router import: `from api.routers import auth, geography, watches, feeds`
  - [ ] 3.2 Add `app.include_router(feeds.router, prefix="/api/v1")` after geography and watches router registrations
  - [ ] 3.3 Verify: all other `main.py` logic unchanged (lifespan, middleware, exception handlers)

- [ ] **Task 4: Create `services/api/tests/test_feeds.py`** (AC 4)
  - [ ] 4.1 Import: `pytest`, `AsyncMock`, `MagicMock`, `TestClient`
  - [ ] 4.2 Import: `from api.dependencies import verify_token`, `from api.redis_client import get_redis`, `from api.main import app`
  - [ ] 4.3 Import: `import redis.exceptions`
  - [ ] 4.4 Define `_override_token()` and `override_redis_with_ttl()` helpers
  - [ ] 4.5 Define `f_client` fixture using `dependency_overrides` for `verify_token`
  - [ ] 4.6 Write F1: live key (ttl=18) → status=live, ttl_seconds=18
  - [ ] 4.7 Write F2: absent key (ttl=-2) → status=stale, ttl_seconds=None
  - [ ] 4.8 Write F3: no-expiry key (ttl=-1) → status=live, ttl_seconds=None
  - [ ] 4.9 Write F4: all KNOWN_PROVIDERS present in `data.feeds` (len matches)
  - [ ] 4.10 Write F5: envelope shape — `data`, `meta.timestamp`, `error=None`
  - [ ] 4.11 Write F6: no-auth → 401
  - [ ] 4.12 Write F7: ConnectionError → 200, status=stale (non-fatal)
  - [ ] 4.13 Write F8: last_heartbeat_at always None
  - [ ] 4.14 Run `pytest services/api/tests/ -v` — all prior test groups pass alongside Group F

- [ ] **Task 5: Local smoke test** (AC 5)
  - [ ] 5.1 Start infra: `docker compose -f docker-compose.yml -f docker-compose.dev.yml up postgres valkey -d`
  - [ ] 5.2 Start API: `uvicorn services.api.api.main:app --reload --host 0.0.0.0 --port 8000 --env-file .env`
  - [ ] 5.3 Start ingestor: `PROVIDER=opensky python -m ingestor.main` (from repo root, active venv)
  - [ ] 5.4 Login and retrieve token; call `GET /api/v1/feeds/health` with Bearer token
  - [ ] 5.5 Verify `data.feeds[0].status == "live"` while ingestor is running
  - [ ] 5.6 Stop ingestor; wait >30s; call endpoint again; verify `data.feeds[0].status == "stale"`

---

## Dev Notes

### File Change Map

```
shared/
└── schema/
    └── feeds.py                              ← UPDATE: FeedHealth.ttl_seconds Optional[int] = None

services/api/
├── api/
│   ├── main.py                               ← UPDATE: import + include feeds.router
│   └── routers/
│       └── feeds.py                          ← CREATE: GET /feeds/health endpoint
└── tests/
    └── test_feeds.py                         ← CREATE: Groups F1–F8
```

**DO NOT TOUCH:**
- `api/redis_client.py` — `get_redis()` factory is already correct; no changes needed
- `api/config.py` — no new settings required
- `api/database.py` — feeds endpoint has no DB dependency
- `api/dependencies.py` — `verify_token` used as-is
- `api/exceptions.py` — `success_envelope` used as-is
- All ingestor service files — ingestor is unchanged
- `api/routers/auth.py`, `watches.py`, `geography.py` — untouched

### Valkey Key Namespace (from `project-context.md`)

| Key | Set by | TTL | Value |
|---|---|---|---|
| `feed:health:{provider}` | Ingestor `health.py` `FeedHeartbeatPublisher` | 30s (SETEX) | `"alive"` |

The API reads this key with `redis.ttl()`. It never writes to it — the ingestor owns this key exclusively.

### TTL Return Value Semantics (redis-py / Valkey)

| `redis.ttl(key)` return | Meaning |
|---|---|
| `> 0` | Key exists with N seconds remaining |
| `-1` | Key exists with no TTL (anomalous — should not occur given SETEX in ingestor) |
| `-2` | Key does not exist (ingestor stopped or Valkey evicted) |

The API treats `-2` as `"stale"`. The API treats `-1` as `"live"` (defensive, shouldn't happen) with a WARNING log. No `GET` call on the key value is needed — `TTL` alone is sufficient to determine liveness.

### Passive Staleness Model (Core Principle)

The feed health model is **passive** — the ingestor does NOT write a "stale" or "down" flag. Absence of the key after TTL expiry IS the staleness signal. This design means:

- No coordination needed between ingestor and API.
- A dead ingestor is automatically detected within 30s.
- A restarted ingestor is automatically detected within 15s (next heartbeat cycle).
- The API never assumes a feed is healthy — only an active TTL key counts as `"live"`.

**Do NOT add any mechanism that writes a "stale" or "down" value to Valkey.** That would couple the services and break the passive model.

### `get_redis()` Dependency in FastAPI

The API already has `get_redis()` from `api/redis_client.py`:

```python
# api/redis_client.py (existing — do not change)
_pool = redis.ConnectionPool.from_url(settings.valkey_url, decode_responses=True)

def get_redis() -> redis.Redis:
    return redis.Redis(connection_pool=_pool)
```

This returns a synchronous-style `redis.asyncio.Redis` object (via the pool), which supports `await redis.ttl(key)`. The connection pool is initialized at module load — no per-request connection setup needed.

The feeds router uses it exactly like any other Depends:
```python
redis: redis.asyncio.Redis = Depends(get_redis)
ttl = await redis.ttl(f"feed:health:opensky")
```

### Feeds Router Implementation (Full)

```python
from __future__ import annotations

import logging
from datetime import UTC, datetime

import redis.exceptions
from fastapi import APIRouter, Depends
from fastapi.responses import JSONResponse

from api.dependencies import verify_token
from api.exceptions import success_envelope
from api.redis_client import get_redis
from schema.feeds import FeedHealth, FeedHealthResponse

log = logging.getLogger("api.feeds")

KNOWN_PROVIDERS: list[str] = ["opensky"]

router = APIRouter(prefix="/feeds", tags=["feeds"])


@router.get("/health")
async def get_feed_health(
    _: str = Depends(verify_token),
    redis=Depends(get_redis),
) -> JSONResponse:
    feeds: list[FeedHealth] = []

    for provider in KNOWN_PROVIDERS:
        key = f"feed:health:{provider}"
        try:
            ttl = await redis.ttl(key)
        except Exception as exc:  # ConnectionError or any Valkey failure
            log.warning("Valkey TTL check failed for %s: %s", key, exc)
            feeds.append(FeedHealth(provider=provider, status="stale", ttl_seconds=None))
            continue

        if ttl > 0:
            feeds.append(FeedHealth(provider=provider, status="live", ttl_seconds=ttl))
        elif ttl == -1:
            # Key exists but has no TTL — anomalous (ingestor used SET instead of SETEX)
            log.warning("%s key exists with no TTL — expected SETEX, got SET", key)
            feeds.append(FeedHealth(provider=provider, status="live", ttl_seconds=None))
        else:
            # ttl == -2: key absent — feed is stale
            feeds.append(FeedHealth(provider=provider, status="stale", ttl_seconds=None))

    response = FeedHealthResponse(feeds=feeds, checked_at=datetime.now(UTC))
    return JSONResponse(content=success_envelope(response.model_dump(mode="json")))
```

### `model_dump(mode="json")` — Pydantic v2 Serialization

`FeedHealthResponse.model_dump(mode="json")` serializes the model to a dict with JSON-compatible types (datetimes → ISO-8601 strings, None → null). This is the correct approach for producing content passed to `JSONResponse(content=...)`. The `success_envelope()` function then wraps it in the standard API envelope before the final JSON response is returned.

### Test Pattern — Mocking `get_redis`

```python
from unittest.mock import AsyncMock
from api.redis_client import get_redis
from api.main import app

def _make_redis_override(ttl_map: dict):
    """ttl_map: {'feed:health:opensky': 18}"""
    async def _override():
        mock = AsyncMock()
        async def _ttl(key):
            return ttl_map.get(key, -2)
        mock.ttl = _ttl
        return mock
    return _override

@pytest.fixture()
def f_client():
    app.dependency_overrides[verify_token] = _override_token
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()

def test_live_feed(f_client):
    app.dependency_overrides[get_redis] = _make_redis_override({"feed:health:opensky": 18})
    r = f_client.get("/api/v1/feeds/health")
    assert r.status_code == 200
    feed = r.json()["data"]["feeds"][0]
    assert feed["status"] == "live"
    assert feed["ttl_seconds"] == 18
```

### API Response Shape (Example)

**Live feed:**
```json
{
  "data": {
    "feeds": [
      {
        "provider": "opensky",
        "status": "live",
        "last_heartbeat_at": null,
        "ttl_seconds": 18
      }
    ],
    "checked_at": "2026-03-11T14:32:00.000000Z"
  },
  "meta": {
    "timestamp": "2026-03-11T14:32:00.000000Z",
    "request_id": "uuid-here"
  },
  "error": null
}
```

**Stale feed:**
```json
{
  "data": {
    "feeds": [
      {
        "provider": "opensky",
        "status": "stale",
        "last_heartbeat_at": null,
        "ttl_seconds": null
      }
    ],
    "checked_at": "2026-03-11T14:32:45.000000Z"
  },
  "meta": {
    "timestamp": "2026-03-11T14:32:45.000000Z",
    "request_id": "uuid-here"
  },
  "error": null
}
```

### No New Dependencies Required

The API service already has all required packages:
- `redis>=5.0` — already declared in `services/api/pyproject.toml` (from Story 1.7)
- `fastapi`, `pydantic`, `pydantic-settings` — already present
- No new packages needed in `pyproject.toml`.

### No New Env Vars Required

`VALKEY_HOST` and `VALKEY_PORT` are already in `.env.example` and already loaded by `api/config.py` via `settings.valkey_url`. The `redis_client.py` pool is constructed from `settings.valkey_url` at module load. No new configuration needed.

### Windows / Local Dev Notes

API startup from repo root (active venv, `.env` in repo root):
```bash
uvicorn services.api.api.main:app --reload --host 0.0.0.0 --port 8000 --env-file .env
```

Tests (no Valkey required):
```bash
pytest services/api/tests/ -v
```

The test client patches `get_redis` at the dependency level — no Valkey process needed for unit tests.

---

## Scope Guardrails

### IN SCOPE — Story 3.5

- `shared/schema/feeds.py` — `FeedHealth.ttl_seconds` type fix (`Optional[int] = None`)
- `services/api/api/routers/feeds.py` — new router, `GET /api/v1/feeds/health`
- `services/api/api/main.py` — import and wire `feeds.router`
- `services/api/tests/test_feeds.py` — Groups F1–F8
- Valkey TTL check only — no DB reads, no writes
- Passive staleness model via Valkey key presence/expiry
- Auth required on the endpoint
- `KNOWN_PROVIDERS = ["opensky"]` for v1 Epic 3 scope

### DEFERRED — Do NOT implement in this story

| Item | Deferred to |
|---|---|
| Signal Desk feed health indicator UI (Domain Pulse dot / inline warning banner) | Story 3.6 |
| `useFeedHealth()` React hook | Story 3.6 |
| Domain Pulse full component (per-provider color dots) | Story 9.5 |
| Multi-provider feed health checks (ADS-B Exchange, AIS NOAA) | Stories 9.1, 9.2 — extend `KNOWN_PROVIDERS` then |
| 4-tier retention TTL enforcement | Story 9.4 |
| WebSocket push of feed health changes | Epic 5 (5.8 WebSocket endpoint) |
| Processor service, baseline computation, signal detection | Epics 4–7 |
| Watch status state machine | Stories 4.3–4.4 |
| Heartbeat-based watch DEGRADED transition | Story 4.4 |
| `watch:status:{watch_id}` Valkey key reads | Epic 4 (processor writes, Epic 5 API reads) |
| Any ingestor service changes | Ingestor is complete through Story 3.4 |
| Dockerfile / Docker Compose production changes | Story 1.6 |
| OpenSky compliance posture changes | Operator decision, not a story |

---

## Dependencies / Prerequisites

| Dependency | Status | Notes |
|---|---|---|
| Story 3.3 — Valkey pub/sub (heartbeat) | ✅ Done (2026-03-11) | `FeedHeartbeatPublisher` publishes `feed:health:opensky` with 30s TTL every 15s. This is the key being read. |
| Story 3.4 — Entity state DB writes | ✅ Done (2026-03-11) | Unblocks 3.5 per sprint plan; ingestor pipeline complete. |
| Story 1.7 — FastAPI app skeleton | ✅ Done (2026-03-09) | `api/redis_client.py`, `get_redis()`, `api/config.py`, `api/dependencies.py`, `api/exceptions.py`, `success_envelope()` all present and correct. |
| `shared/schema/feeds.py` — `FeedHealth`, `FeedHealthResponse` | ✅ Present (Story 1.3) | Models exist. `ttl_seconds` type updated in this story (AC 1 — targeted change). |
| `redis>=5.0` in `services/api/pyproject.toml` | ✅ Present (Story 1.7) | Includes `redis.asyncio`. No new deps needed. |
| Valkey running | For smoke test only | `docker compose up valkey -d`; not required for `pytest` |

---

## Config / Env Expectations Before Implementation

No new env vars required. All configuration is already declared.

| Env Var | Current value (`.env.example`) | Story 3.5 use |
|---|---|---|
| `VALKEY_HOST` | `valkey` | Used by `api/config.py` → `settings.valkey_url` → `redis_client.py` pool |
| `VALKEY_PORT` | `6379` | Same |
| `POSTGRES_*` | All present | Not used in feeds router (no DB access) |
| `JWT_SECRET_KEY` | Present | Used by `verify_token` to validate Bearer token on request |
| `OPERATOR_USERNAME` / `OPERATOR_PASSWORD_HASH` | Present | Auth login unchanged |

**Local dev override:** If running Valkey locally (not in Docker Compose), set `VALKEY_HOST=localhost` in `.env`. Default `valkey` hostname is for Docker Compose internal routing.

---

## References

- [Story 3.3 — Valkey Pub/Sub](3-3-valkey-pubsub.md) — `FeedHeartbeatPublisher`, heartbeat key namespace, 15s interval / 30s TTL
- [Story 3.4 — Entity State DB Writes](3-4-entity-state-db-writes.md) — final ingestor story; smoke test results confirm heartbeat working
- [Story 1.7 — FastAPI App Skeleton](1-7-fastapi-app-skeleton.md) — `api/redis_client.py`, `get_redis()`, app scaffold
- [Architecture](../planning-artifacts/architecture.md) — Feed health architecture, Valkey key namespaces
- [Project Context — Feed Health Architecture](../project-context.md#feed-health-architecture) — passive staleness rule
- [Project Context — Redis Key Namespaces](../project-context.md#redis-key-namespaces-applies-to-valkey) — `feed:health:{provider}` key specification
- [Project Context — API Response Envelope](../project-context.md#api-response-envelope) — standard envelope format
- [epics.md — Epic 3: Feed health API story](../planning-artifacts/epics.md)
- [sprint-status.yaml](sprint-status.yaml) — `3-5-feed-health-api: backlog → ready-for-dev`
- [shared/schema/feeds.py](../../shared/schema/feeds.py) — `FeedHealth`, `FeedHealthResponse`
- [services/api/api/redis_client.py](../../services/api/api/redis_client.py) — `get_redis()` factory
- [services/api/api/main.py](../../services/api/api/main.py) — app entry point; wire feeds router here
- [services/api/api/exceptions.py](../../services/api/api/exceptions.py) — `success_envelope()`
- [services/api/api/dependencies.py](../../services/api/api/dependencies.py) — `verify_token`
- [.env.example](../../.env.example) — VALKEY_HOST, VALKEY_PORT confirmed present
