# Story 3.3: Valkey Pub/Sub for Ingestor Output and Feed Heartbeat

**Status:** ready-for-dev
**Epic:** 3 — Live Feed Ingestion & Feed Health Visibility
**Sprint sequence:** Third story of Epic 3; unblocks 3.4 (DB writes), 4.1 (Processor service — subscribes to entity events)

---

## Story

As the Panoptes ingestor service,
I need to publish each normalized ADS-B entity state to Valkey after every successful normalization cycle, and maintain a liveness heartbeat key that refreshes every 15 seconds,
so that the processor service has a real-time pub/sub stream of entity events to subscribe to, and downstream services can detect feed staleness passively through key expiry — without any polling of the ingestor process and without silent failure.

---

## Acceptance Criteria

**AC 1 — `health.py`: `FeedHeartbeatPublisher` class**

- `services/ingestor/ingestor/health.py` is fully implemented (replaces the empty stub).
- Defines module-level constants:
  ```python
  HEARTBEAT_INTERVAL_SECONDS: int = 15
  HEARTBEAT_TTL_SECONDS: int = 30
  ```
- Defines `FeedHeartbeatPublisher` class:
  - `__init__(self, redis_client, provider_name: str) -> None`
    - Stores `self._redis = redis_client`.
    - Stores `self._provider_name = provider_name`.
    - Creates `self._log = logging.getLogger("ingestor.health")`.
    - Does NOT attempt a connection check or `ping()` at init time.
  - `async def run(self) -> None` — infinite heartbeat loop:
    1. Calls `await self._redis.setex(f"feed:health:{self._provider_name}", HEARTBEAT_TTL_SECONDS, "alive")`.
    2. Logs at DEBUG level: `"Heartbeat published: feed:health:{provider} (TTL={TTL}s)"`.
    3. Sleeps `HEARTBEAT_INTERVAL_SECONDS`.
    4. On any exception from `setex` or `sleep`: logs at WARNING level with the error, does NOT re-raise — loop continues. This is critical for the no-silent-failure posture: a Valkey hiccup must not kill the heartbeat loop.
- The heartbeat `run()` loop never exits on transient errors.
- The key `feed:health:{provider}` expires naturally (TTL=30s) when the ingestor is down — this passive expiry is the staleness detection mechanism for Story 3.5/3.6. No explicit "stale" write is ever made by the ingestor.

**AC 2 — `opensky.py`: entity-state Valkey publish wired into `run()`**

- `services/ingestor/ingestor/adapters/opensky.py` is updated.
- Adds module-level constant:
  ```python
  ENTITY_STATE_TTL_SECONDS: int = 7200  # 2 hours — per project-context.md key namespace spec
  ```
- `__init__` signature extended to accept an optional Redis client:
  ```python
  def __init__(self, config: ProviderConfig, settings=None, redis_client=None) -> None:
  ```
  - Stores `self._redis = redis_client`.
  - All other `__init__` logic (token acquisition, httpx client, guard, logger) is unchanged.
- Adds `async def _publish_state(self, state: NormalizedAdsbState) -> None`:
  - Serializes the state for Valkey: `payload: str = state.model_dump_json(exclude={"raw_payload"})`.
  - `raw_payload` is explicitly excluded — it belongs to DB writes (Story 3.4), not the internal event bus.
  - Calls `await self._redis.setex(f"entity:adsb:{state.icao24}", ENTITY_STATE_TTL_SECONDS, payload)` — last-known state, 2h TTL.
  - Calls `await self._redis.publish(f"entity:adsb:{state.icao24}", payload)` — per-entity pub/sub channel; processor subscribes via `PSUBSCRIBE entity:adsb:*`.
  - If `self._redis` is None: logs at DEBUG level and returns immediately. No AttributeError, no crash.
  - If either Valkey operation raises: logs at WARNING level with the icao24 and error. Does NOT re-raise — individual publish failures are non-fatal.
- `run()` loop is updated:
  - After each successful `normalize_adsb()` call, calls `await self._publish_state(state)`.
  - `published_count` tracks how many publishes succeeded (publish that returned without exception).
  - Cycle log updated to include the published count:
    ```
    Cycle complete: fetched={N}, normalized={N}, skipped={N}, published={N}
    ```
  - If `self._redis` is None for the entire cycle, logs a one-time DEBUG note and `published_count=0`.
  - No structural change to the existing fetch/normalize/backoff loop from Story 3.2.

**AC 3 — `main.py`: create Valkey client and run adapter + heartbeat concurrently**

- `services/ingestor/ingestor/main.py` is updated with targeted changes only.
- Adds imports:
  ```python
  import redis.asyncio as aioredis
  from ingestor.health import FeedHeartbeatPublisher
  ```
- In `async def main()`, after the compliance gate and before adapter instantiation:
  ```python
  redis_client = aioredis.Redis(
      host=settings.valkey_host,
      port=settings.valkey_port,
      decode_responses=True,
  )
  ```
  - `decode_responses=True` ensures all Valkey responses are Python strings (compatible with JSON payloads).
  - No `ping()` call — connection is established lazily on first SETEX/PUBLISH. Valkey downtime at startup is non-fatal.
- Adapter instantiation updated from:
  ```python
  adapter = adapter_class(config, settings)
  ```
  to:
  ```python
  adapter = adapter_class(config, settings, redis_client)
  ```
- Creates heartbeat publisher:
  ```python
  heartbeat = FeedHeartbeatPublisher(redis_client, settings.provider)
  ```
- Runs both tasks concurrently:
  ```python
  await asyncio.gather(adapter.run(), heartbeat.run())
  ```
- All other `main()` logic (settings load, registry lookup, compliance gate, logging setup) is unchanged.

**AC 4 — `pyproject.toml`: add `redis` dependency**

- `services/ingestor/pyproject.toml` updated:
  - Add `redis>=5.0` to `[project] dependencies`.
- `pyproject.toml` after update:
  ```toml
  [project]
  dependencies = [
      "panoptes-shared",
      "pydantic-settings>=2.0",
      "httpx>=0.27",
      "redis>=5.0",
  ]
  ```
- `redis>=5.0` includes `redis.asyncio` as a built-in subpackage — no `[asyncio]` extra required at this version.

**AC 5 — `tests/test_feed_heartbeat.py`: unit test suite**

- `services/ingestor/tests/test_feed_heartbeat.py` created.
- All tests use `AsyncMock` for the redis client — no live Valkey required.
- Required test coverage:

  **Group H — `FeedHeartbeatPublisher.run()`**
  - H1: `run()` calls `setex` with key `"feed:health:opensky"`, TTL=30, value=`"alive"` on first iteration.
  - H2: `run()` calls `asyncio.sleep` with `HEARTBEAT_INTERVAL_SECONDS` between setex calls.
  - H3: `setex` raises `redis.exceptions.ConnectionError` → warning logged, loop continues (second setex called).
  - H4: Constants: `HEARTBEAT_INTERVAL_SECONDS == 15` and `HEARTBEAT_TTL_SECONDS == 30`.

**AC 6 — `tests/test_opensky_adapter.py`: extend with Story 3.3 test groups**

- `services/ingestor/tests/test_opensky_adapter.py` updated with new groups only. All prior Story 3.2 tests (Groups A–D) continue to pass unmodified.
- Required new test coverage:

  **Group V — Valkey publish behavior in `_publish_state()` and `run()`**
  - V1: `_publish_state()` calls `setex` with key `f"entity:adsb:{icao24}"`, TTL=7200, and JSON payload that does NOT contain `"raw_payload"` key.
  - V2: `_publish_state()` calls `publish` with channel `f"entity:adsb:{icao24}"` and the same JSON payload as `setex`.
  - V3: `_publish_state()` with `redis_client=None` (adapter instantiated without client) → returns immediately, no exception raised.
  - V4: `setex` raises `redis.exceptions.ConnectionError` → warning logged; `run()` continues to next record in batch (other states still published).
  - V5: Successful cycle with 2 states and a live mock redis → cycle log includes `published=2`.
  - V6: Valkey payload parsed as JSON → has `icao24`, `latitude`, `longitude`, `altitude_ft`, `speed_knots`, `heading`, `observed_at`, `provider` fields present; `raw_payload` key absent.

- `pytest services/ingestor/tests/ -v` passes — all tests green, no external services required.

**AC 7 — Smoke test: Valkey wired end-to-end**

- With a running Valkey instance (`docker run -d -p 6379:6379 valkey/valkey:7`), `PROVIDER=opensky`, and valid `OPENSKY_CLIENT_ID`/`OPENSKY_CLIENT_SECRET`:
  ```bash
  python -m ingestor.main
  ```
  - Service starts without error. Both `adapter.run()` and `heartbeat.run()` active.
  - After ≤15 seconds: `feed:health:opensky` key present in Valkey with TTL ≤30s.
  - After first poll cycle (≤120s): `entity:adsb:{icao24}` keys visible in Valkey with TTL ~7200s.
  - Cycle log line includes `published=N` where N ≈ fetch count (minus any publish errors).
- With Valkey **unavailable** at startup:
  - Service starts without crashing.
  - WARNING-level logs for each failed SETEX/PUBLISH.
  - Fetch-normalize loop continues uninterrupted.
- **Do NOT run the live smoke test in CI.** This AC is local manual verification only.

---

## Tasks / Subtasks

- [ ] **Task 1: Implement `services/ingestor/ingestor/health.py`** (AC 1)
  - [ ] 1.1 Add module docstring explaining the heartbeat's role (passive staleness detection via key expiry)
  - [ ] 1.2 Import `asyncio`, `logging`
  - [ ] 1.3 Define `HEARTBEAT_INTERVAL_SECONDS = 15` and `HEARTBEAT_TTL_SECONDS = 30`
  - [ ] 1.4 Define `FeedHeartbeatPublisher.__init__(self, redis_client, provider_name: str)`
  - [ ] 1.5 Implement `async def run(self)`: setex loop with try/except, WARNING log on error, sleep
  - [ ] 1.6 Verify `run()` never exits on transient error (exception handling covers all Redis exceptions + broad fallback)

- [ ] **Task 2: Update `services/ingestor/ingestor/adapters/opensky.py`** (AC 2)
  - [ ] 2.1 Add `ENTITY_STATE_TTL_SECONDS: int = 7200` constant after existing constants block
  - [ ] 2.2 Add `NormalizedAdsbState` to existing imports from `ingestor.normalizer` (needed for type hint in `_publish_state`)
  - [ ] 2.3 Extend `__init__` signature: add `redis_client=None` as 3rd parameter; store `self._redis = redis_client`
  - [ ] 2.4 Implement `async def _publish_state(self, state: NormalizedAdsbState) -> None`: None guard, model_dump_json(exclude={"raw_payload"}), setex + publish, WARNING on error
  - [ ] 2.5 Update `run()` normalize loop: call `await self._publish_state(state)` after each successful `normalize_adsb()`; track `published_count`
  - [ ] 2.6 Update cycle log format string to include `published={published_count}`
  - [ ] 2.7 Update module docstring: remove "Story 3.3" deferral note; replace with "Story 3.3: Valkey publish wired"
  - [ ] 2.8 Verify: no change to `_acquire_token()`, `fetch()`, or the backoff retry logic

- [ ] **Task 3: Update `services/ingestor/ingestor/main.py`** (AC 3)
  - [ ] 3.1 Add `import redis.asyncio as aioredis` import
  - [ ] 3.2 Add `from ingestor.health import FeedHeartbeatPublisher` import
  - [ ] 3.3 In `main()`: after compliance gate, create `redis_client = aioredis.Redis(host=..., port=..., decode_responses=True)`
  - [ ] 3.4 Pass `redis_client` as 3rd arg to `adapter_class(config, settings, redis_client)`
  - [ ] 3.5 Create `heartbeat = FeedHeartbeatPublisher(redis_client, settings.provider)`
  - [ ] 3.6 Replace `await adapter.run()` with `await asyncio.gather(adapter.run(), heartbeat.run())`
  - [ ] 3.7 Verify: all other `main()` logic unchanged (settings, registry, compliance, log setup)

- [ ] **Task 4: Update `services/ingestor/pyproject.toml`** (AC 4)
  - [ ] 4.1 Add `redis>=5.0` to `[project] dependencies`

- [ ] **Task 5: Create `services/ingestor/tests/test_feed_heartbeat.py`** (AC 5)
  - [ ] 5.1 Import `pytest`, `AsyncMock`, `FeedHeartbeatPublisher`, `HEARTBEAT_INTERVAL_SECONDS`, `HEARTBEAT_TTL_SECONDS`
  - [ ] 5.2 Write H1: mock redis → verify `setex("feed:health:opensky", 30, "alive")` called
  - [ ] 5.3 Write H2: mock sleep → verify called with `HEARTBEAT_INTERVAL_SECONDS`
  - [ ] 5.4 Write H3: setex raises `ConnectionError` → loop continues, second setex called, no exception propagated
  - [ ] 5.5 Write H4: constant value assertions

- [ ] **Task 6: Update `services/ingestor/tests/test_opensky_adapter.py`** (AC 6)
  - [ ] 6.1 Add `import redis.exceptions` (or mock redis errors via `Exception` subclass) for error simulation
  - [ ] 6.2 Write V1: `_publish_state()` → setex key/TTL/payload verification; confirm `raw_payload` absent from payload JSON
  - [ ] 6.3 Write V2: `_publish_state()` → publish channel/payload verification
  - [ ] 6.4 Write V3: adapter with `redis_client=None` → `_publish_state()` returns cleanly, no error
  - [ ] 6.5 Write V4: setex raises → warning logged, other records in batch not blocked
  - [ ] 6.6 Write V5: cycle log with mock redis → `published=2` in log output
  - [ ] 6.7 Write V6: payload JSON parse → field presence/absence assertions
  - [ ] 6.8 Verify all prior Story 3.2 tests (Groups A–D) still pass with updated `__init__` signature (they call without `redis_client` → defaults to None)

- [ ] **Task 7: Local smoke test** (AC 7)
  - [ ] 7.1 `pip install -e shared/ -e services/ingestor[test]` — adds `redis>=5.0`
  - [ ] 7.2 Import smoke: `python -c "import redis.asyncio; from ingestor.health import FeedHeartbeatPublisher; print('ok')"`
  - [ ] 7.3 `pytest services/ingestor/tests/ -v` — all tests green (Groups A–D from 3.2, H, V)
  - [ ] 7.4 With live Valkey: `python -m ingestor.main` → verify heartbeat key and entity keys in Valkey after first cycle

---

## Dev Notes

### File Change Map

```
services/ingestor/
├── pyproject.toml                    ← UPDATE: add redis>=5.0
└── ingestor/
    ├── main.py                       ← UPDATE: redis client creation, FeedHeartbeatPublisher, asyncio.gather
    ├── health.py                     ← IMPLEMENT (was empty stub)
    ├── base.py                       ← DO NOT TOUCH (Story 3.1 seam)
    ├── normalizer.py                 ← DO NOT TOUCH (Story 3.1 seam)
    ├── compliance.py                 ← DO NOT TOUCH (Story 3.1 seam)
    └── adapters/
        ├── __init__.py               ← DO NOT TOUCH
        └── opensky.py                ← UPDATE: redis_client param, _publish_state(), run() publish + count

services/ingestor/tests/
├── test_feed_heartbeat.py            ← CREATE
└── test_opensky_adapter.py           ← UPDATE: Groups V added; Groups A–D preserved
```

### Valkey Key Namespace (from project-context.md)

| Key | Operation | TTL | Value |
|---|---|---|---|
| `feed:health:{provider}` | `SETEX` | 30s | `"alive"` |
| `entity:adsb:{icao24}` | `SETEX` | 7200s (2h) | JSON — `NormalizedAdsbState` minus `raw_payload` |
| `entity:adsb:{icao24}` | `PUBLISH` | — (pub/sub) | same JSON payload as SETEX |

- The `feed:health:opensky` key is the liveness surface. It expires passively at TTL 30s if the ingestor stops publishing. Story 3.5 reads this key's presence/TTL to derive feed status. The ingestor never writes a "down" or "stale" value — absence of the key IS the stale signal.
- The `entity:adsb:{icao24}` SET key is the last-known-state cache for the processor (GET when needed). The PUBLISH on the same channel name is the real-time event stream. The processor will use `PSUBSCRIBE entity:adsb:*` to receive all ADS-B entity events.

### Why `raw_payload` Is Excluded from Valkey Payload

`NormalizedAdsbState.raw_payload` is the unmodified OpenSky state-vector dict. It exists solely for the `raw_adsb` DB write in Story 3.4. Including it in the Valkey payload would:
- Roughly double the payload size (raw dict ≈ normalized dict in size)
- Push data the processor does not need onto the bus
- Couple the internal bus format to the provider wire format

The processor receives only the normalized fields. The compliance guard (should_write_raw check) and raw storage path live exclusively in Story 3.4.

### Valkey Payload Shape (example)

```json
{
  "provider": "opensky",
  "entity_type": "adsb",
  "icao24": "abc123",
  "call_sign": "UAL123",
  "latitude": 37.7749,
  "longitude": -122.4194,
  "altitude_ft": 36000.1,
  "speed_knots": 485.3,
  "heading": 270.5,
  "observed_at": "2026-03-11T12:00:00Z",
  "geography_entity_id": null,
  "received_at": "2026-03-11T12:00:01.234567Z"
}
```

`geography_entity_id` is always `null` from the ingestor — the processor assigns it via geospatial watch-matching (Epic 4).

### Heartbeat Timing vs. Poll Cycle Timing

| Mechanism | Interval | Purpose |
|---|---|---|
| Poll cycle (`OPENSKY_POLL_INTERVAL_SECONDS`) | 120s | Fetch ADS-B data from OpenSky |
| Heartbeat (`HEARTBEAT_INTERVAL_SECONDS`) | 15s | Prove ingestor is alive; refresh `feed:health:{provider}` TTL |

These two loops are independent. The heartbeat runs as a concurrent asyncio task. During the 120s sleep between poll cycles, the heartbeat continues publishing every 15s. This means the feed health key never expires due to the poll interval — it only expires if the ingestor process dies or Valkey is unreachable.

The 15s interval vs. 30s TTL gives a 2x safety margin: the key can miss one heartbeat pulse and still be considered alive. The ingestor has to miss two consecutive pulses before the key expires.

### Concurrent Task Architecture in `main.py`

```python
redis_client = aioredis.Redis(
    host=settings.valkey_host,
    port=settings.valkey_port,
    decode_responses=True,
)

adapter = adapter_class(config, settings, redis_client)
heartbeat = FeedHeartbeatPublisher(redis_client, settings.provider)

await asyncio.gather(adapter.run(), heartbeat.run())
```

- Both tasks share the same `redis.asyncio.Redis` client instance. `redis.asyncio.Redis` uses a connection pool internally — sharing the client between coroutines is safe and the recommended pattern.
- `asyncio.gather` runs both coroutines concurrently in the same event loop. If either raises an unhandled exception, `gather` propagates it after the other completes or is cancelled.
- Since both `adapter.run()` and `heartbeat.run()` are infinite loops with internal error handling, they should not raise in practice. The service shuts down only on `KeyboardInterrupt` / `SIGTERM` (handled by `asyncio.run()` at the entry point).

### `redis.asyncio.Redis` — Usage Pattern

```python
import redis.asyncio as aioredis

# Create once; reuse across coroutines (connection-pooled)
redis_client = aioredis.Redis(host="valkey", port=6379, decode_responses=True)

# SETEX (key, ttl_seconds, value)
await redis_client.setex("feed:health:opensky", 30, "alive")
await redis_client.setex("entity:adsb:abc123", 7200, '{"icao24": "abc123", ...}')

# PUBLISH (channel, message)
await redis_client.publish("entity:adsb:abc123", '{"icao24": "abc123", ...}')
```

The `redis` pip package (v5+) includes `redis.asyncio` as a built-in subpackage. No separate install is required — `redis>=5.0` in `pyproject.toml` is sufficient.

### `NormalizedAdsbState.model_dump_json()` — Pydantic v2 Serialization

```python
# Pydantic v2 — serialize to JSON string excluding raw_payload
payload: str = state.model_dump_json(exclude={"raw_payload"})
```

`model_dump_json()` handles `datetime` fields automatically (ISO 8601 format with timezone). The output is a UTF-8 JSON string compatible with `decode_responses=True` Redis client.

With `decode_responses=True`, all Redis GET/SET values are Python strings. The payload does not need base64 or binary encoding.

### Testing `_publish_state()` Without Live Valkey

```python
from unittest.mock import AsyncMock, MagicMock
import pytest

@pytest.fixture
def mock_redis():
    client = MagicMock()
    client.setex = AsyncMock(return_value=True)
    client.publish = AsyncMock(return_value=1)
    return client

async def test_publish_state_calls_setex_and_publish(mock_redis):
    adapter = make_adapter(redis_client=mock_redis)
    state = make_normalized_state(icao24="abc123")

    await adapter._publish_state(state)

    mock_redis.setex.assert_awaited_once()
    call_args = mock_redis.setex.call_args
    assert call_args[0][0] == "entity:adsb:abc123"  # key
    assert call_args[0][1] == ENTITY_STATE_TTL_SECONDS  # TTL

    payload = json.loads(call_args[0][2])
    assert "raw_payload" not in payload
    assert payload["icao24"] == "abc123"

    mock_redis.publish.assert_awaited_once_with("entity:adsb:abc123", call_args[0][2])
```

### Backward Compatibility of `OpenSkyAdapter.__init__` Signature

Story 3.2 tests instantiate `OpenSkyAdapter(config)` or `OpenSkyAdapter(config, settings)`. The new signature `__init__(self, config, settings=None, redis_client=None)` is fully backward-compatible — both patterns continue to work with `redis_client` defaulting to `None`. No Story 3.2 tests need modification.

### Pydantic Class Config Deprecation (Carry-Forward from 3.2)

`IngestorSettings` uses `class Config:` (Pydantic v1 style) and emits a deprecation warning at startup. This is non-blocking pre-existing technical debt from Story 3.2. Do not attempt to fix it in this story.

### Windows / Local Dev Install Sequence

```bash
# From repo root (active venv)
pip install -e shared/ -e "services/ingestor[test]"
```

The `redis>=5.0` dep is added automatically. The bare `panoptes-shared` name pattern (established in Story 3.1 Windows fix) is preserved — no path reference changes needed.

---

## Scope Guardrails

### IN SCOPE — Story 3.3

- `health.py` — `FeedHeartbeatPublisher` full implementation
- `opensky.py` — `redis_client` parameter, `_publish_state()`, run() publish + count
- `main.py` — `redis.asyncio.Redis` creation, `FeedHeartbeatPublisher` creation, `asyncio.gather`
- `pyproject.toml` — `redis>=5.0` dependency
- Unit tests for heartbeat and Valkey publish behavior (no live Valkey required)
- `entity:adsb:{icao24}` SETEX + PUBLISH per normalized state
- `feed:health:{provider}` SETEX heartbeat at 15s interval

### DEFERRED — Do NOT implement in this story

| Item | Deferred to |
|---|---|
| `entity_states` DB writes | Story 3.4 |
| `raw_adsb` DB writes | Story 3.4 |
| `should_write_raw()` enforcement (raw vs. normalized-only gate) | Story 3.4 |
| SQLAlchemy engine / async session setup | Story 3.4 |
| Feed health API (`/api/v1/health`) — derive status from Valkey key | Story 3.5 |
| Signal Desk feed health indicator UI | Story 3.6 |
| Processor service and Valkey subscriber loop | Story 4.1 |
| Heartbeat-based watch staleness (watch transitions on TTL expiry) | Story 4.4 |
| ADS-B Exchange adapter (Story 9.1) and AIS NOAA adapter (Story 9.2) | Epic 9 |
| Dockerfile / Docker Compose changes | Story 1.6 (production deployment) |
| Processor logic, baseline computation, signal detection | Epics 4–7 |
| `watch:status:{watch_id}` Valkey key | Epic 4 (processor writes) |
| `signals:new` Valkey pub/sub channel | Epic 5 (processor emits) |
| `entity:ais:{mmsi}` key namespace | AIS adapter (Story 9.2) |
| Any frontend changes | Not in scope for any ingestor story |

---

## Dependencies / Prerequisites

| Dependency | Status | Notes |
|---|---|---|
| Story 3.1 — Ingestor service base | ✅ Done (2026-03-11) | `BaseIngestor`, `NormalizedAdsbState`, `ComplianceGuard`, `IngestorSettings` (incl. `valkey_host`, `valkey_port`) all implemented |
| Story 3.2 — OpenSky adapter | ✅ Done (2026-03-11) | `OpenSkyAdapter` implemented with `run()` loop producing `NormalizedAdsbState`; smoke test passed (10,625 normalized states per cycle) |
| `IngestorSettings.valkey_host` / `valkey_port` | ✅ Already present | Declared in `main.py` from Story 3.1; not yet consumed — 3.3 wires them |
| `.env.example` `VALKEY_HOST` / `VALKEY_PORT` | ✅ Already present | No new env vars needed |
| `redis>=5.0` pip package | Added in this story | Includes `redis.asyncio` built-in; compatible with Valkey 7 |
| Running Valkey instance | For smoke test only | `docker run -d -p 6379:6379 valkey/valkey:7`; not required for `pytest` |
| `health.py` stub | ✅ Present | Empty stub exists at `services/ingestor/ingestor/health.py`; 3.3 implements it |

---

## Config / Env Expectations Before Implementation

All required env vars are **already declared** in `.env.example` and `IngestorSettings`. No new vars needed for Story 3.3.

| Env Var | Current state | Story 3.3 use |
|---|---|---|
| `VALKEY_HOST` | `valkey` (default in IngestorSettings) | `aioredis.Redis(host=settings.valkey_host)` |
| `VALKEY_PORT` | `6379` (default in IngestorSettings) | `aioredis.Redis(port=settings.valkey_port)` |
| `PROVIDER` | `opensky` | Selects adapter AND names the heartbeat key: `feed:health:opensky` |
| `OPENSKY_CLIENT_ID` | Already declared | Unchanged — 3.2 handles auth |
| `OPENSKY_CLIENT_SECRET` | Already declared | Unchanged — 3.2 handles auth |
| `DATABASE_URL` | Present but unused | Not touched in 3.3; Story 3.4 wires it |

**Local dev note:** For smoke testing, if running locally (not in Docker), override `VALKEY_HOST=localhost` in `.env`. The default `valkey` hostname is for Docker Compose service routing.

---

## References

- [Story 3.1 — Ingestor Service Base](3-1-ingestor-service-base.md) — `BaseIngestor`, `NormalizedAdsbState`, `ComplianceGuard`, `IngestorSettings.valkey_host/valkey_port`
- [Story 3.2 — OpenSky Adapter](3-2-opensky-adapter.md) — `OpenSkyAdapter`, `run()` structure, compliance posture
- [Architecture — Valkey key namespaces and pub/sub design](../planning-artifacts/architecture.md)
- [Project Context — Redis Key Namespaces](../project-context.md#redis-key-namespaces-applies-to-valkey)
- [Project Context — Pipeline Stage Separation](../project-context.md#pipeline-stage-separation) — Collect stage publishes to Valkey; Fuse stage subscribes
- [Project Context — Service Boundary Write Rules](../project-context.md#service-boundary-write-rules) — ingestor may write `raw_adsb`, `entity_states`, and Valkey entity keys
- [epics.md — Epic 3: Valkey pub/sub story scope](../planning-artifacts/epics.md)
- [sprint-status.yaml](sprint-status.yaml) — `3-3-valkey-pubsub: backlog → ready-for-dev`
- [services/ingestor/ingestor/health.py](../../services/ingestor/ingestor/health.py) — empty stub to implement
- [services/ingestor/ingestor/adapters/opensky.py](../../services/ingestor/ingestor/adapters/opensky.py) — update: redis_client + _publish_state
- [services/ingestor/ingestor/main.py](../../services/ingestor/ingestor/main.py) — update: redis client + gather
- [services/ingestor/pyproject.toml](../../services/ingestor/pyproject.toml) — update: redis dep
- [.env.example](../../.env.example) — VALKEY_HOST, VALKEY_PORT confirmed present

---

## Dev Agent Record

_(To be completed by the dev agent after implementation)_
