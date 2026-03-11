# Story 3.2: OpenSky Adapter

**Status:** done
**Epic:** 3 — Live Feed Ingestion & Feed Health Visibility
**Sprint sequence:** Second story of Epic 3; unblocks 3.3 (Valkey pub/sub), 3.4 (DB writes)

---

## Story

As the Panoptes ingestor service,
I need a concrete OpenSky Network adapter that continuously fetches ADS-B state vectors, converts them to normalized entity-state objects, and recovers automatically from outages,
so that the system collects live ADS-B data from day one without operator babysitting — and the ingestion pipeline from HTTP fetch through normalization is verified end-to-end before Valkey and DB wiring are added in Stories 3.3 and 3.4.

---

## Acceptance Criteria

**AC 1 — `adapters/opensky.py`: module-level constants**
- `services/ingestor/ingestor/adapters/opensky.py` is created.
- Defines the following module-level constants:
  ```python
  OPENSKY_BASE_URL: str = "https://opensky-network.org/api"
  OPENSKY_POLL_INTERVAL_SECONDS: int = 120   # global polling target per operator directive
  OPENSKY_REQUEST_TIMEOUT_SECONDS: float = 30.0
  OPENSKY_INITIAL_BACKOFF_SECONDS: float = 30.0
  OPENSKY_MAX_BACKOFF_SECONDS: float = 300.0

  OPENSKY_STATE_FIELDS: list[str] = [
      "icao24", "callsign", "origin_country", "time_position",
      "last_contact", "longitude", "latitude", "baro_altitude",
      "on_ground", "velocity", "true_track", "vertical_rate",
      "sensors", "geo_altitude", "squawk", "spi", "position_source",
  ]
  ```
- `OPENSKY_POLL_INTERVAL_SECONDS` is the authoritative inter-cycle sleep duration and must not be derived from `ComplianceGuard.rate_limit_interval()` (which governs burst protection, not cycle cadence). The 120s target reflects the homelab/non-commercial posture.

**AC 2 — `OpenSkyAdapter` class declaration and `__init__`**
- Defines `OpenSkyAdapter(BaseIngestor)` with:
  - `provider_name: ClassVar[str] = "opensky"`
  - `feed_type: ClassVar[FeedTypeEnum] = FeedTypeEnum.adsb`
  - `__init__(self, config: ProviderConfig, settings=None) -> None`:
    - Calls `super().__init__(config)` to store `self.config`.
    - Extracts `opensky_client_id` and `opensky_client_secret` from `settings` using `getattr(settings, "opensky_client_id", None)` and `getattr(settings, "opensky_client_secret", None)`.
    - Stores `self._client_id: Optional[str]` and `self._client_secret: Optional[str]`.
    - Stores `self._bearer_token: Optional[str] = None` — token is acquired via OAuth2 client credentials flow before the first authenticated request and refreshed on `401` responses.
    - Creates `self._client: httpx.AsyncClient` with:
      - `timeout=httpx.Timeout(OPENSKY_REQUEST_TIMEOUT_SECONDS)`
      - No `auth` argument — bearer token is injected as `Authorization: Bearer <token>` header per-request in `fetch()`.
    - Creates `self._guard = ComplianceGuard(config)` for use in the run loop.
    - Creates `self._log = logging.getLogger("ingestor.opensky")`.
- `httpx.AsyncClient` is created once in `__init__` and reused across all fetch cycles (not per-request).

**AC 3 — `_array_to_dict()`: OpenSky array → named dict**
- Defines `_array_to_dict(state_vector: list) -> dict` as a module-level function (not a method).
- Converts a raw OpenSky state-vector array to a named dict using `OPENSKY_STATE_FIELDS`:
  ```python
  def _array_to_dict(state_vector: list) -> dict:
      return dict(zip(OPENSKY_STATE_FIELDS, state_vector))
  ```
- This function is the only place where the array index → field name mapping lives. The normalizer (`normalize_adsb()`) from Story 3.1 expects named dicts, not arrays — `_array_to_dict()` is the adapter's seam between wire format and normalizer contract.
- `zip` naturally handles arrays shorter than `OPENSKY_STATE_FIELDS` (missing tail fields are absent in output dict, which `normalize_adsb()` handles via `.get()` with None fallback).

**AC 4 — `fetch()`: single HTTP polling cycle**
- Implements `async def fetch(self) -> list[dict]`:
  - Makes `GET {OPENSKY_BASE_URL}/api/states/all` using `self._client`.
  - Calls `response.raise_for_status()` — `httpx.HTTPStatusError` propagates to caller (retry handled in `run()`).
  - Parses response JSON; extracts `data["states"]`.
  - If `data["states"]` is `None` or absent, returns `[]` with a debug log ("No states in response — sky may be empty").
  - Converts each state vector array to a named dict using `_array_to_dict()`.
  - Returns `list[dict]` — one dict per aircraft state vector.
- `fetch()` does NOT call `normalize_adsb()` — normalization is `run()`'s responsibility.
- `fetch()` does NOT catch `httpx.HTTPStatusError` or `httpx.RequestError` — both propagate to `run()` for retry handling.
- `fetch()` does NOT access Valkey or DB.

**AC 5 — `run()`: continuous fetch-normalize loop with retry**
- Implements `async def run(self) -> None`:
  - Infinite `while True` loop.
  - **Nominal path (one cycle):**
    1. `received_at = datetime.now(UTC)` — captured before fetch to timestamp the batch.
    2. `raw_states: list[dict] = await self.fetch()`
    3. For each dict in `raw_states`:
       - Call `normalize_adsb(raw, provider=self.provider_name, received_at=received_at)`.
       - On `ValueError` (bad icao24): log a warning with the raw dict's `icao24` key (or `"<missing>"`) and `continue` — one bad record does not abort the batch.
    4. Log cycle summary at INFO level: `f"Cycle complete: fetched={len(raw_states)}, normalized={success_count}, skipped={skip_count}"`.
    5. Reset backoff to `OPENSKY_INITIAL_BACKOFF_SECONDS` after a successful cycle.
    6. `await asyncio.sleep(OPENSKY_POLL_INTERVAL_SECONDS)`.
  - **Error path (fetch or HTTP failure):**
    - Catch `httpx.HTTPStatusError | httpx.RequestError | Exception` (last catch is broad to cover unexpected errors, logged as ERROR).
    - Log the error at WARNING level (HTTP/network) or ERROR level (unexpected).
    - `await asyncio.sleep(backoff)` where `backoff` starts at `OPENSKY_INITIAL_BACKOFF_SECONDS`, doubles each error, caps at `OPENSKY_MAX_BACKOFF_SECONDS`.
    - Backoff resets to `OPENSKY_INITIAL_BACKOFF_SECONDS` after the first successful cycle.
    - Loop continues — never exits on transient errors.
  - `run()` does NOT publish to Valkey (Story 3.3).
  - `run()` does NOT write to DB (Story 3.4).
  - `run()` does NOT call `self._guard.validate_startup()` — that gate was already called in `main.py` before `run()` is invoked.
  - `run()` does NOT check `self._guard.should_write_raw()` — that check belongs to the DB writer in Story 3.4 which uses the guard to decide raw vs. normalized-only storage.

**AC 6 — `main.py`: register adapter and pass settings**
- `services/ingestor/ingestor/main.py` is updated with two targeted changes:
  1. Add import and registry entry (immediately after the `PROVIDER_REGISTRY: dict = {}` definition):
     ```python
     from ingestor.adapters.opensky import OpenSkyAdapter
     PROVIDER_REGISTRY["opensky"] = OpenSkyAdapter
     ```
  2. Update the adapter instantiation line from:
     ```python
     adapter = adapter_class(config)
     ```
     to:
     ```python
     adapter = adapter_class(config, settings)
     ```
     This passes `IngestorSettings` to the adapter so it can extract provider-specific credentials (`opensky_client_id`, `opensky_client_secret`) without re-reading env.
- All other `main.py` logic is unchanged — startup sequence, `PROVIDER_DEFAULTS`, compliance gate, logging setup.

**AC 7 — `adapters/__init__.py`: update comment**
- Update the comment in `services/ingestor/ingestor/adapters/__init__.py` to reflect that OpenSky is now implemented (not just planned):
  ```python
  # Story 3.2: OpenSkyAdapter is registered in main.py PROVIDER_REGISTRY.
  #   from ingestor.adapters.opensky import OpenSkyAdapter
  #   PROVIDER_REGISTRY["opensky"] = OpenSkyAdapter
  #
  # Future adapters (Story 9.1: adsbexchange, Story 9.2: ais_noaa) follow the same pattern.
  ```

**AC 8 — `pyproject.toml`: dependencies**
- `services/ingestor/pyproject.toml` updated:
  - Add `httpx>=0.27` to `[project] dependencies`.
  - Add `pytest-asyncio>=0.23` and `respx>=0.20` to `[project.optional-dependencies] test`.
- `pyproject.toml` after update:
  ```toml
  [project]
  dependencies = [
      "panoptes-shared",
      "pydantic-settings>=2.0",
      "httpx>=0.27",
  ]

  [project.optional-dependencies]
  test = [
      "pytest>=8.0",
      "pytest-asyncio>=0.23",
      "respx>=0.20",
  ]
  ```

**AC 9 — `tests/test_opensky_adapter.py`: unit test suite**
- `services/ingestor/tests/test_opensky_adapter.py` created.
- All tests are unit tests — no live network, no DB, no Valkey.
- Uses `pytest-asyncio` for async tests and `respx` for httpx mocking.
- Test coverage required:

  **Group A — `_array_to_dict()`**
  - A1: Full 17-element array → dict with all OPENSKY_STATE_FIELDS keys mapped correctly (verify `icao24`, `callsign`, `geo_altitude`, `velocity` etc.)
  - A2: Short array (fewer than 17 elements) → only present fields in output dict; no IndexError; missing fields are absent from dict (not None)
  - A3: Empty array `[]` → empty dict `{}`

  **Group B — `OpenSkyAdapter` class declaration**
  - B1: `OpenSkyAdapter.provider_name == "opensky"`
  - B2: `OpenSkyAdapter.feed_type == FeedTypeEnum.adsb`
  - B3: Instantiate with `config` + `settings=None` → `_client_id is None`, `_client_secret is None`, `_bearer_token is None`
  - B4: Instantiate with `settings` carrying `opensky_client_id="id"`, `opensky_client_secret="secret"` → `_client_id` and `_client_secret` stored; `_bearer_token` starts as `None` (acquired lazily)

  **Group C — `fetch()` (respx mock)**
  - C1: Successful response with states list → returns correct `list[dict]` with named fields
  - C2: Response with `states: null` → returns `[]`
  - C3: Response with `states` absent from JSON → returns `[]`
  - C4: HTTP 429 response → `httpx.HTTPStatusError` raised (not caught in `fetch()`)
  - C5: Network error (`httpx.ConnectError`) → `httpx.RequestError` raised (not caught in `fetch()`)
  - C6: Adapter with no client credentials (`_client_id=None`) → request has no `Authorization` header
  - C7: Adapter with `_bearer_token` set → request has `Authorization: Bearer <token>` header

  **Group D — `run()` (mock `fetch`)**
  - D1: Single successful cycle → `normalize_adsb` called once per state vector; cycle log emitted; sleep called with `OPENSKY_POLL_INTERVAL_SECONDS`
  - D2: Record with missing icao24 → `ValueError` caught; warning logged; other records in batch normalized; loop continues
  - D3: Fetch raises `httpx.RequestError` → error logged; sleep called with `OPENSKY_INITIAL_BACKOFF_SECONDS`; second iteration attempted
  - D4: Backoff doubles on consecutive errors; caps at `OPENSKY_MAX_BACKOFF_SECONDS` (test sequence: 30 → 60 → 120 → 240 → 300 → 300)
  - D5: Successful cycle after error resets backoff to `OPENSKY_INITIAL_BACKOFF_SECONDS`

- All 43 existing tests from Stories 3.1 (`test_normalizer.py` + `test_compliance.py`) continue to pass unmodified.
- `pytest services/ingestor/tests/ -v` passes with no external services required.

**AC 10 — Service startup smoke test**
- With `PROVIDER=opensky` set (and `DATABASE_URL`, `VALKEY_HOST` may be absent/invalid — that is fine for this story):
  ```bash
  python -m ingestor.main
  ```
  - Service starts without `RuntimeError`.
  - Registry lookup succeeds: `"opensky"` found in `PROVIDER_REGISTRY`.
  - Compliance gate passes: `storage_allowed=True`, `terms_status="under_review"` → `is_limited_mode()=True`, `should_write_raw()=False`.
  - Compliance log line is emitted: `limited_mode=True, should_write_raw=False`.
  - Adapter enters `run()` loop and attempts first `fetch()` against the live OpenSky endpoint (expected to succeed with empty or populated states; success or network timeout both acceptable — loop continues either way).
- **Do NOT run the live smoke test against OpenSky in CI** — this AC is a local manual verification only.

---

## Tasks / Subtasks

- [x] **Task 1: Create `services/ingestor/ingestor/adapters/opensky.py`** (AC 1–5)
  - [x] 1.1 Add module docstring describing the adapter's role and compliance posture
  - [x] 1.2 Import `asyncio`, `logging`, `datetime`, `UTC` from stdlib; `ClassVar`, `Optional` from `typing`
  - [x] 1.3 Import `httpx`
  - [x] 1.4 Import `BaseIngestor` from `ingestor.base`; `ComplianceGuard`, `ProviderConfig` from `ingestor.compliance`; `normalize_adsb` from `ingestor.normalizer`; `FeedTypeEnum` from `schema.enums`
  - [x] 1.5 Define all 5 module-level constants (AC 1)
  - [x] 1.6 Define `_array_to_dict(state_vector: list) -> dict` (AC 3)
  - [x] 1.7 Define `OpenSkyAdapter(BaseIngestor)` with `provider_name` and `feed_type` ClassVars (AC 2)
  - [x] 1.8 Implement `__init__(self, config, settings=None)`: super() call, credential extraction (`opensky_client_id`, `opensky_client_secret`), bearer token state init, AsyncClient creation, guard + logger setup (AC 2)
  - [x] 1.9 Implement `async def fetch(self) -> list[dict]`: GET request, raise_for_status, parse states, _array_to_dict conversion (AC 4)
  - [x] 1.10 Implement `async def run(self) -> None`: while-True loop, received_at capture, fetch call, normalize loop with ValueError guard, cycle logging, sleep, backoff on error (AC 5)
  - [x] 1.11 Verify import chain: `compliance.py ← base.py ← main.py` — no circular imports with new `adapters/opensky.py`

- [x] **Task 2: Update `services/ingestor/ingestor/main.py`** (AC 6)
  - [x] 2.1 Add `from ingestor.adapters.opensky import OpenSkyAdapter` import
  - [x] 2.2 Add `PROVIDER_REGISTRY["opensky"] = OpenSkyAdapter` after the registry dict definition
  - [x] 2.3 Change `adapter = adapter_class(config)` → `adapter = adapter_class(config, settings)`
  - [x] 2.4 Verify no other changes to main.py; existing startup sequence and PROVIDER_DEFAULTS unchanged

- [x] **Task 3: Update `services/ingestor/ingestor/adapters/__init__.py`** (AC 7)
  - [x] 3.1 Update comment block to reflect OpenSky is implemented, pattern documented for future providers

- [x] **Task 4: Update `services/ingestor/pyproject.toml`** (AC 8)
  - [x] 4.1 Add `httpx>=0.27` to `[project] dependencies`
  - [x] 4.2 Add `pytest-asyncio>=0.23` to `[project.optional-dependencies] test`
  - [x] 4.3 Add `respx>=0.20` to `[project.optional-dependencies] test`

- [x] **Task 5: Create `services/ingestor/tests/test_opensky_adapter.py`** (AC 9)
  - [x] 5.1 Configure `pytest-asyncio` mode (add `asyncio_mode = "auto"` to `pyproject.toml` `[tool.pytest.ini_options]` or use `@pytest.mark.asyncio` per test)
  - [x] 5.2 Write Group A tests: `_array_to_dict()` (A1–A3)
  - [x] 5.3 Write Group B tests: class constants + `__init__` auth wiring (B1–B4)
  - [x] 5.4 Write Group C tests: `fetch()` with `respx` mock (C1–C7)
  - [x] 5.5 Write Group D tests: `run()` with patched `fetch` and patched `asyncio.sleep` (D1–D5)
  - [x] 5.6 Verify all 43 prior tests (test_normalizer.py + test_compliance.py) still pass
  - [x] 5.7 `pytest services/ingestor/tests/ -v` — all tests green

- [x] **Task 6: Local smoke test** (AC 10)
  - [x] 6.1 `pip install -e shared/ -e services/ingestor[test]` from repo root (adds httpx, pytest-asyncio, respx)
  - [x] 6.2 Import smoke: `python -c "from ingestor.adapters.opensky import OpenSkyAdapter; print(OpenSkyAdapter.provider_name)"` — prints `opensky`
  - [x] 6.3 With `PROVIDER=opensky` in env/`.env`: `python -m ingestor.main` — service starts, logs compliance status, enters fetch loop

---

## Dev Notes

### File Change Map

```
services/ingestor/
├── pyproject.toml                    ← UPDATE: add httpx>=0.27; pytest-asyncio, respx to [test]
└── ingestor/
    ├── main.py                       ← UPDATE: register OpenSkyAdapter, pass settings to adapter
    ├── base.py                       ← DO NOT TOUCH (Story 3.1 seam; __init__ signature unchanged)
    ├── normalizer.py                 ← DO NOT TOUCH (Story 3.1 seam; consumed by opensky.py)
    ├── compliance.py                 ← DO NOT TOUCH (Story 3.1 seam; consumed by opensky.py)
    ├── health.py                     ← DO NOT TOUCH (Story 3.3 scope)
    └── adapters/
        ├── __init__.py               ← UPDATE: comment only
        └── opensky.py                ← CREATE

services/ingestor/tests/
└── test_opensky_adapter.py           ← CREATE
```

### OpenSky API Wire Format

`GET https://opensky-network.org/api/states/all` response:

```json
{
  "time": 1710000000,
  "states": [
    ["abc123", "UAL123  ", "United States", 1710000000, 1710000001,
     -122.4194, 37.7749, 10972.8, false, 257.7, 270.5, null, null,
     11278.2, null, false, 0],
    ...
  ]
}
```

- `states` may be `null` (no aircraft in coverage) — treat as empty list.
- Each state array maps to `OPENSKY_STATE_FIELDS` by index (0-based, 17 fields).
- `_array_to_dict()` converts to the named dict format that `normalize_adsb()` expects.
- Fields at index 3 (`time_position`), 5 (`longitude`), 6 (`latitude`), 7 (`baro_altitude`), 9 (`velocity`), 10 (`true_track`), 13 (`geo_altitude`) may be `None`.
- `baro_altitude` (index 7) is NOT used — `geo_altitude` (index 13) is the correct altitude source. `_array_to_dict` maps both; `normalize_adsb()` uses only `geo_altitude`.

### Auth Posture

OpenSky's current programmatic API access mechanism is **OAuth2 client credentials** (per current OpenSky Network API documentation). The adapter uses a `client_id` / `client_secret` → bearer token flow.

**Token acquisition:**
- POST to the OpenSky OAuth2 token endpoint with `grant_type=client_credentials`, `client_id`, and `client_secret` (refer to current OpenSky API docs for the exact token endpoint URL).
- Store the resulting `access_token` as `self._bearer_token`.
- Inject as `Authorization: Bearer <token>` header on each `fetch()` request.
- On a `401` response, discard the cached token and re-acquire before retrying.

**Unauthenticated fallback:**
- If `OPENSKY_CLIENT_ID` is empty/absent, the adapter makes requests without an `Authorization` header. Anonymous access may be rate-limited or restricted per current OpenSky API policy — this is an optional fallback only, not the recommended or future-proof mechanism for homelab use.
- `self._bearer_token` remains `None`; no token fetch is attempted.

Both `OPENSKY_CLIENT_ID` and `OPENSKY_CLIENT_SECRET` must be declared in:
- `.env.example` (empty by default — correct homelab posture before credentials are registered)
- `IngestorSettings` in `main.py` (as `Optional[str] = None`)

Credentials are passed via `getattr(settings, ..., None)` so that a `settings=None` call (e.g., in tests) does not raise `AttributeError`.

### Poll Interval vs. Rate Limit — Clarification

These are two distinct concepts that must not be conflated:

| Concept | Value | Source | Used in |
|---|---|---|---|
| `OPENSKY_POLL_INTERVAL_SECONDS` | 120s | Operator directive | `run()` main loop sleep |
| `ComplianceGuard.rate_limit_interval()` | 0.2s (= 1/5.0 rps) | `PROVIDER_DEFAULTS["opensky"]["rate_limit_rps"]` | Burst protection for intra-cycle sub-requests (not applicable to single-request loop) |

The `run()` loop makes **one** HTTP request per cycle and sleeps 120 seconds. It naturally stays well within the 5 rps burst limit. `rate_limit_interval()` is not used in this story's loop — it is available for future adapters that make multiple sub-requests per cycle (e.g., per-ICAO queries).

### Retry / Reconnect Pattern

```python
backoff = OPENSKY_INITIAL_BACKOFF_SECONDS  # 30.0s

while True:
    try:
        received_at = datetime.now(UTC)
        raw_states = await self.fetch()
        # ... normalize loop ...
        backoff = OPENSKY_INITIAL_BACKOFF_SECONDS  # reset on success
        await asyncio.sleep(OPENSKY_POLL_INTERVAL_SECONDS)
    except httpx.HTTPStatusError as exc:
        self._log.warning("HTTP error %s — backing off %.0fs", exc.response.status_code, backoff)
        await asyncio.sleep(backoff)
        backoff = min(backoff * 2, OPENSKY_MAX_BACKOFF_SECONDS)
    except httpx.RequestError as exc:
        self._log.warning("Network error: %s — backing off %.0fs", exc, backoff)
        await asyncio.sleep(backoff)
        backoff = min(backoff * 2, OPENSKY_MAX_BACKOFF_SECONDS)
    except Exception as exc:
        self._log.error("Unexpected error: %s — backing off %.0fs", exc, backoff, exc_info=True)
        await asyncio.sleep(backoff)
        backoff = min(backoff * 2, OPENSKY_MAX_BACKOFF_SECONDS)
```

- Backoff sequence: 30 → 60 → 120 → 240 → 300 → 300 → ... (capped)
- Backoff resets to 30s after any successful cycle.
- `exc_info=True` on unexpected errors preserves full traceback in logs.
- Loop never exits — reconnect is automatic.

### Compliance Posture Preserved

OpenSky `PROVIDER_DEFAULTS` in `main.py` (Story 3.1, unchanged):

```python
"opensky": {
    "rate_limit_rps": 5.0,
    "storage_allowed": True,        # homelab use — terms under review
    "retention_hours": 48,
    "commercial_use_allowed": False, # non-commercial personal/homelab only
    "terms_status": "under_review",  # LIMITED MODE: no raw writes until confirmed compliant
}
```

At startup:
- `validate_startup()` passes (`storage_allowed=True`).
- `is_limited_mode()` returns `True` (`terms_status="under_review"`).
- `should_write_raw()` returns `False` (limited mode — raw storage gated).

This means the adapter fetches and normalizes, but Story 3.4 will see `should_write_raw()=False` and skip `raw_adsb` table writes. Normalized `entity_states` writes will proceed. This is the correct behavior for the under-review posture.

`run()` in this story logs the normalized states but does not write anywhere — this is by design. Stories 3.3 and 3.4 add those write paths.

### Import Chain (No Circular Imports)

```
schema.enums
    ↑
ingestor.compliance  ←  ingestor.base  ←  ingestor.main
                                                ↑
ingestor.normalizer  ←  ingestor.adapters.opensky  ↑
```

`adapters/opensky.py` imports from:
- `ingestor.base` (BaseIngestor)
- `ingestor.compliance` (ComplianceGuard, ProviderConfig)
- `ingestor.normalizer` (normalize_adsb)
- `schema.enums` (FeedTypeEnum)

`main.py` imports from `adapters.opensky` — this is one-way. No circular dependency.

### `pytest-asyncio` Configuration

Add to `services/ingestor/pyproject.toml`:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

This sets `asyncio` mode globally so all async test functions are automatically treated as async tests without per-function `@pytest.mark.asyncio` decorators.

### Testing `run()` Without Real I/O

Use `unittest.mock.AsyncMock` to patch `fetch()` and `asyncio.sleep`. Run the loop for a bounded number of iterations by having the mock raise `StopAsyncIteration` or `asyncio.CancelledError` after N calls:

```python
async def test_run_single_cycle(monkeypatch):
    adapter = make_adapter()
    call_count = 0

    async def mock_fetch():
        nonlocal call_count
        call_count += 1
        if call_count >= 2:
            raise asyncio.CancelledError
        return [VALID_STATE_DICT]

    monkeypatch.setattr(adapter, "fetch", mock_fetch)
    monkeypatch.setattr(asyncio, "sleep", AsyncMock())

    with pytest.raises(asyncio.CancelledError):
        await adapter.run()

    assert call_count == 2
    asyncio.sleep.assert_awaited_with(OPENSKY_POLL_INTERVAL_SECONDS)
```

`respx` mocks are used for Group C (`fetch()`) tests. `monkeypatch` / `AsyncMock` are used for Group D (`run()`) tests.

### Windows-Specific Packaging Note

Story 3.1 confirmed: `pip install -e shared/ -e services/ingestor[test]` works correctly on Windows when `panoptes-shared` is listed as a bare name (not `file://../../shared`). This story adds new packages (`httpx`, `pytest-asyncio`, `respx`) to the same `pyproject.toml` — same install pattern applies. No path reference changes needed.

---

## Scope Guardrails

### IN SCOPE — Story 3.2

- `services/ingestor/ingestor/adapters/opensky.py` — full implementation
- OpenSky state-vector HTTP fetch (`/api/states/all`)
- Array → named dict conversion (`_array_to_dict`)
- Continuous fetch loop (`run()`) with poll interval, retry, and backoff
- Normalization via Story 3.1 `normalize_adsb()` — called in `run()`, output logged only
- OAuth2 client credentials auth (client_id/client_secret → bearer token from env via settings; unauthenticated fallback when credentials absent)
- `main.py` — two targeted changes: registry entry + settings pass-through
- `pyproject.toml` — httpx + pytest deps added
- Unit tests for all adapter behavior (no live network required)

### DEFERRED — Do NOT implement in this story

| Item | Deferred to |
|---|---|
| `feed:health:{provider}` heartbeat publish to Valkey (15s interval, 30s TTL) | Story 3.3 |
| `redis.asyncio` import or Valkey connection of any kind | Story 3.3 |
| `health.py` implementation | Story 3.3 |
| Entity-state pub/sub to Valkey (`entity:adsb:{icao24}` key) | Story 3.3 |
| `entity_states` DB writes | Story 3.4 |
| `raw_adsb` DB writes | Story 3.4 |
| SQLAlchemy engine / async session setup | Story 3.4 |
| `should_write_raw()` enforcement (raw vs. normalized-only gate) | Story 3.4 |
| Feed health API (`/api/v1/health`) | Story 3.5 |
| Signal Desk feed health indicator | Story 3.6 |
| ADS-B Exchange adapter | Story 9.1 |
| AIS NOAA adapter | Story 9.2 |
| Dockerfile or Docker Compose changes | Story 1.6 (production deployment) |
| Processor logic, baseline computation, signal detection | Epics 4–7 |
| `geography_entity_id` assignment (always `None` from ingestor) | Processor responsibility (Epic 4) |
| Watch-matching or geospatial queries | Processor responsibility |

---

## Dependencies / Prerequisites

| Dependency | Status | Notes |
|---|---|---|
| Story 3.1 — Ingestor service base | ✅ Done (2026-03-11) | `BaseIngestor`, `ProviderConfig`, `ComplianceGuard`, `normalize_adsb`, `NormalizedAdsbState`, `IngestorSettings`, `PROVIDER_REGISTRY`, `PROVIDER_DEFAULTS` all implemented and tested |
| Story 1.3 — Shared schema package | ✅ Done | `FeedTypeEnum` available via `from schema.enums import FeedTypeEnum` |
| `httpx>=0.27` | Added in this story | `pip install -e services/ingestor[test]` after `pyproject.toml` update |
| `pytest-asyncio>=0.23` | Added in this story | Required for async test support |
| `respx>=0.20` | Added in this story | httpx mock library for `fetch()` tests |
| No DB / Valkey required | — | All tests are pure Python; no external services required for AC 9 |
| OpenSky credentials | Optional | `OPENSKY_CLIENT_ID` + `OPENSKY_CLIENT_SECRET` — empty = unauthenticated fallback (homelab default before credentials registered) |
| `.env` with `PROVIDER=opensky` | For smoke test only | Not required for `pytest` |

---

## Config / Env Expectations Before Implementation

The following env vars are **already declared** in `.env.example` and `IngestorSettings`. No new vars are needed for this story.

| Env Var | Current state | Story 3.2 use |
|---|---|---|
| `PROVIDER` | `PROVIDER=opensky` | Selects OpenSkyAdapter via registry |
| `OPENSKY_CLIENT_ID` | Must be added, empty by default | OAuth2 client ID for bearer token acquisition (empty → unauthenticated fallback) |
| `OPENSKY_CLIENT_SECRET` | Must be added, empty by default | OAuth2 client secret for bearer token acquisition (empty → unauthenticated fallback) |
| `LOG_LEVEL` | `info` | Adapter uses `logging.getLogger("ingestor.opensky")` |
| `DATABASE_URL` | Present but unused in this story | Not read in Story 3.2; Story 3.4 wires it |
| `VALKEY_HOST` | Present but unused in this story | Not read in Story 3.2; Story 3.3 wires it |
| `VALKEY_PORT` | Present but unused in this story | Not read in Story 3.2; Story 3.3 wires it |

`.env.example` must be updated to add `OPENSKY_CLIENT_ID=` and `OPENSKY_CLIENT_SECRET=` (empty by default). `IngestorSettings` in `main.py` must declare both as `Optional[str] = None`.

---

## References

- [Story 3.1 — Ingestor Service Base](3-1-ingestor-service-base.md) — `BaseIngestor`, `normalize_adsb`, `ComplianceGuard`, `IngestorSettings`, `PROVIDER_REGISTRY`
- [Architecture — Ingestor service](../planning-artifacts/architecture.md) (Collect stage, provider adapter pattern)
- [Architecture — Decision 9: Provider compliance](../planning-artifacts/architecture.md)
- [Project Context — Provider Terms Compliance](project-context.md#provider-terms-compliance)
- [Project Context — Pipeline Stage Separation](project-context.md#pipeline-stage-separation)
- [Project Context — Redis Key Namespaces](project-context.md#redis-key-namespaces-applies-to-valkey)
- [epics.md — Epic 3 story list](../planning-artifacts/epics.md)
- [sprint-status.yaml](sprint-status.yaml) — `3-2-opensky-adapter: backlog`
- [services/ingestor/ingestor/base.py](../../services/ingestor/ingestor/base.py) — `BaseIngestor` (do not modify)
- [services/ingestor/ingestor/normalizer.py](../../services/ingestor/ingestor/normalizer.py) — `normalize_adsb`, `NormalizedAdsbState` (do not modify)
- [services/ingestor/ingestor/compliance.py](../../services/ingestor/ingestor/compliance.py) — `ComplianceGuard`, `ProviderConfig` (do not modify)
- [services/ingestor/ingestor/main.py](../../services/ingestor/ingestor/main.py) — `IngestorSettings`, `PROVIDER_REGISTRY`, `PROVIDER_DEFAULTS` (targeted update only)
- [.env.example](../../.env.example) — `PROVIDER` var confirmed present; `OPENSKY_CLIENT_ID`, `OPENSKY_CLIENT_SECRET` must be added

---

## Dev Agent Record

### Completion Notes

- Initial implementation completed: `adapters/opensky.py` (OAuth2 bearer token flow, `_array_to_dict`, `fetch()`, `run()` with exponential backoff), `main.py` registry + settings pass-through, `pyproject.toml` deps, `adapters/__init__.py` comment, `tests/test_opensky_adapter.py` (Groups A–D).
- Code-review fixes applied: 401 retry-path test coverage added (C7-equivalent); unused imports cleaned up.
- All unit tests passing (Groups A–D + 43 prior Story 3.1 tests).
- Manual smoke test passed locally on Windows (Nikku, 2026-03-11):
  - `PROVIDER=opensky python -m ingestor.main` from repo root in active venv — startup clean, no RuntimeError.
  - Compliance log confirmed: `limited_mode=True, should_write_raw=False, interval=0.20s`.
  - OAuth token request succeeded (HTTP 200).
  - OpenSky states request succeeded (HTTP 200).
  - Adapter logged: `Cycle complete: fetched=10625, normalized=10625, skipped=0`.
  - Process stopped manually (Ctrl+C).
  - **Non-blocking:** `CancelledError` during asyncio sleep + `KeyboardInterrupt` on exit observed on Ctrl+C — expected shutdown behavior, not a story failure.
  - **Non-blocking:** Pydantic deprecation warning about class-based config observed at startup — technical debt, not a Story 3.2 blocker.
- Story 3.2 closed by Nikku after local validation (2026-03-11).

### Files Changed

```
services/ingestor/
├── pyproject.toml                    ← UPDATED: httpx>=0.27; pytest-asyncio, respx to [test]; asyncio_mode=auto
└── ingestor/
    ├── main.py                       ← UPDATED: OpenSkyAdapter registered; settings passed to adapter
    └── adapters/
        ├── __init__.py               ← UPDATED: comment reflects implementation
        └── opensky.py                ← CREATED

services/ingestor/tests/
└── test_opensky_adapter.py           ← CREATED
```
