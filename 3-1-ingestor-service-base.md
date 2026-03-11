# Story 3.1: Ingestor Service Base

**Status:** ready-for-dev
**Epic:** 3 — Live Feed Ingestion & Feed Health Visibility
**Sprint sequence:** First story of Epic 3; unblocks 3.2, 3.3, 3.4

---

## Story

As the Panoptes ingestor service,
I need a well-defined base class, a normalization seam, and a compliance enforcement layer,
so that provider adapters (OpenSky — Story 3.2, ADS-B Exchange — Story 9.1, AIS NOAA — Story 9.2) can be plugged in without re-architecting the service, and provider terms are enforced from day one.

---

## Acceptance Criteria

**AC 1 — `base.py`: `BaseIngestor` abstract class**
- `services/ingestor/ingestor/base.py` is created.
- Defines `BaseIngestor(ABC)` with:
  - `provider_name: ClassVar[str]` — set by each concrete subclass (e.g., `"opensky"`).
  - `feed_type: ClassVar[FeedTypeEnum]` — set by each concrete subclass.
  - `__init__(self, config: ProviderConfig) -> None` — stores `self.config`.
  - `@abstractmethod async def fetch(self) -> list[dict]` — concrete adapter implements one polling cycle; returns list of raw provider dicts.
  - `@abstractmethod async def run(self) -> None` — concrete adapter (or base class in Story 3.2) implements the continuous service loop.
- No DB, Valkey, or HTTP client is instantiated in `BaseIngestor.__init__` in this story.

**AC 2 — `compliance.py`: `ProviderConfig` and `ComplianceGuard`**
- `services/ingestor/ingestor/compliance.py` is fully implemented (replacing the empty stub).
- Defines `ProviderConfig` (Pydantic `BaseModel`) with fields:
  - `rate_limit_rps: float` — maximum requests per second allowed by provider terms.
  - `storage_allowed: bool` — if `False`, ingestor must refuse to start.
  - `retention_hours: int` — maximum raw retention allowed by provider (default 48).
  - `commercial_use_allowed: bool` — for reference; ingestor does not block on this in v1.
  - `terms_status: Literal["under_review", "compliant", "restricted"]`
- Defines `ComplianceGuard` with:
  - `__init__(self, config: ProviderConfig) -> None`
  - `validate_startup(self) -> None` — raises `RuntimeError` with a clear message if `config.storage_allowed is False`. This is the hard gate before any I/O begins.
  - `is_limited_mode(self) -> bool` — returns `True` if `terms_status` is `"under_review"` or `"restricted"`. In limited mode the ingestor stores normalized entity state only (raw records withheld — enforced in Story 3.4).
  - `should_write_raw(self) -> bool` — returns `True` only when `storage_allowed is True` AND `is_limited_mode() is False`.
  - `rate_limit_interval(self) -> float` — returns `1.0 / config.rate_limit_rps` (seconds between polls).
- `validate_startup()` must be called before any provider connection is attempted.

**AC 3 — `normalizer.py`: ADS-B raw → `NormalizedAdsbState`**
- `services/ingestor/ingestor/normalizer.py` is created.
- Defines `NormalizedAdsbState` (Pydantic `BaseModel`) with fields:
  - `provider: str`
  - `entity_type: FeedTypeEnum` — always `FeedTypeEnum.adsb`.
  - `icao24: str`
  - `call_sign: Optional[str]`
  - `latitude: Optional[float]`
  - `longitude: Optional[float]`
  - `altitude_ft: Optional[float]` — converted from meters (OpenSky reports geo_altitude in meters).
  - `speed_knots: Optional[float]` — converted from m/s (OpenSky reports velocity in m/s).
  - `heading: Optional[float]` — true track in degrees.
  - `observed_at: datetime` — UTC; converted from Unix timestamp int.
  - `geography_entity_id: None` — always `None` from the ingestor; geospatial watch-matching is the processor's responsibility.
  - `raw_payload: dict` — the original unmodified dict from the provider (for `raw_adsb` write in Story 3.4).
  - `received_at: datetime` — UTC; the wall-clock time the ingestor received the record.
- Defines `normalize_adsb(raw: dict, provider: str, received_at: datetime) -> NormalizedAdsbState`:
  - Maps OpenSky state-vector fields (see Dev Notes) to `NormalizedAdsbState`.
  - `icao24` is required; if absent or falsy, raises `ValueError("icao24 missing in raw record")`.
  - `call_sign`: strip whitespace; `None` if empty.
  - `altitude_ft`: `None` if `geo_altitude` is `None` or absent; otherwise `round(geo_altitude * 3.28084, 1)`.
  - `speed_knots`: `None` if `velocity` is `None`; otherwise `round(velocity * 1.94384, 1)`.
  - `heading`: directly from `true_track`; `None` if absent.
  - `observed_at`: `datetime.fromtimestamp(time_position, tz=UTC)` if `time_position` is not None; else fall back to `received_at`.
  - `raw_payload` and `received_at` are passed through as-is.
- `geography_entity_id` is hardcoded `None` — no geospatial lookup in the ingestor.

**AC 4 — `main.py`: startup with PROVIDER selection and compliance gate**
- `services/ingestor/ingestor/main.py` is replaced (the stub is removed).
- Defines `IngestorSettings` (Pydantic `BaseSettings`) loading from environment:
  - `provider: str` — from `PROVIDER` env var; required.
  - `log_level: str` — from `LOG_LEVEL`; default `"info"`.
  - OpenSky-specific (optional, needed only when `provider="opensky"`): `opensky_username: Optional[str]`, `opensky_password: Optional[str]`.
  - Infrastructure vars (present in env but used in later stories): `database_url: Optional[str]` (from `DATABASE_URL`), `valkey_host: str` (default `"valkey"`), `valkey_port: int` (default `6379`).
- Defines `PROVIDER_REGISTRY: dict[str, type[BaseIngestor]] = {}` — empty in this story; Story 3.2 adds the OpenSky entry.
- `main()` async function:
  1. Load `IngestorSettings`.
  2. Look up provider in `PROVIDER_REGISTRY`; if not found, raise `RuntimeError(f"No adapter registered for provider: {settings.provider!r}. Add it to PROVIDER_REGISTRY in main.py.")`.
  3. Build `ProviderConfig` for the selected provider (hard-coded defaults per provider name — see Dev Notes).
  4. Instantiate `ComplianceGuard(config)` and call `guard.validate_startup()`.
  5. Instantiate the adapter class with the config.
  6. `await adapter.run()`.
- `if __name__ == "__main__": asyncio.run(main())` entry point present.
- Service starts and exits with a clear `RuntimeError` when `PROVIDER=opensky` (since the registry is empty). This is correct and expected for Story 3.1 — it proves the startup path is wired.

**AC 5 — `adapters/__init__.py`: adapter package placeholder**
- `services/ingestor/ingestor/adapters/__init__.py` documents the import pattern future adapters will follow:
  ```python
  # Adapter modules register themselves in PROVIDER_REGISTRY via main.py.
  # Example (added in Story 3.2):
  #   from ingestor.adapters.opensky import OpenSkyAdapter
  #   PROVIDER_REGISTRY["opensky"] = OpenSkyAdapter
  ```
- No concrete adapter classes in this story.

**AC 6 — `pyproject.toml`: dependencies**
- `services/ingestor/pyproject.toml` updated with:
  - `pydantic-settings>=2.0` added (required for `IngestorSettings` in AC 4).
  - `panoptes-shared @ file://../../shared` remains (existing).
- No other new runtime dependencies added in this story (httpx, redis, sqlalchemy, asyncpg are added in their respective stories).

**AC 7 — `tests/` directory and test suite**
- `services/ingestor/tests/` directory created with `__init__.py`.
- `services/ingestor/tests/test_normalizer.py`: unit tests for `normalize_adsb()`:
  - Full valid OpenSky state vector → all fields populated correctly (altitude converted, speed converted, timestamp converted, call_sign stripped).
  - Missing `icao24` → `ValueError` raised.
  - `time_position=None` → `observed_at` falls back to `received_at`.
  - `geo_altitude=None` → `altitude_ft` is `None`.
  - `velocity=None` → `speed_knots` is `None`.
  - `geography_entity_id` is always `None` in output.
- `services/ingestor/tests/test_compliance.py`: unit tests for `ProviderConfig` + `ComplianceGuard`:
  - `storage_allowed=False` → `validate_startup()` raises `RuntimeError`.
  - `terms_status="under_review"` → `is_limited_mode()` returns `True`.
  - `terms_status="compliant"` → `is_limited_mode()` returns `False`.
  - `is_limited_mode()=True` → `should_write_raw()` returns `False`.
  - `is_limited_mode()=False`, `storage_allowed=True` → `should_write_raw()` returns `True`.
  - `rate_limit_rps=5.0` → `rate_limit_interval()` returns `0.2`.
- All tests pass with `pytest services/ingestor/tests/ -v` from repo root (no external services required).

**AC 8 — Import smoke test**
- From `services/ingestor`, with required dependencies installed:
  ```bash
  python -c "from ingestor.base import BaseIngestor; from ingestor.normalizer import normalize_adsb, NormalizedAdsbState; from ingestor.compliance import ProviderConfig, ComplianceGuard; print('ok')"
  ```
  exits 0.

---

## Tasks / Subtasks

- [ ] **Task 1: Create `services/ingestor/ingestor/base.py`** (AC 1)
  - [ ] 1.1 Import `ABC`, `abstractmethod` from `abc`; import `ClassVar` from `typing`
  - [ ] 1.2 Import `ProviderConfig` from `ingestor.compliance` (forward-safe: compliance.py has no import from base.py)
  - [ ] 1.3 Import `FeedTypeEnum` from `schema.enums`
  - [ ] 1.4 Define `BaseIngestor(ABC)` with `provider_name: ClassVar[str]` and `feed_type: ClassVar[FeedTypeEnum]`
  - [ ] 1.5 Define `__init__(self, config: ProviderConfig) -> None` storing `self.config`
  - [ ] 1.6 Define `@abstractmethod async def fetch(self) -> list[dict]` with docstring
  - [ ] 1.7 Define `@abstractmethod async def run(self) -> None` with docstring noting "Story 3.2 implements the continuous loop"

- [ ] **Task 2: Implement `services/ingestor/ingestor/compliance.py`** (AC 2)
  - [ ] 2.1 Import `Literal`, `Optional` from `typing`; import `BaseModel` from `pydantic`
  - [ ] 2.2 Define `ProviderConfig(BaseModel)` with all 5 fields (rate_limit_rps, storage_allowed, retention_hours, commercial_use_allowed, terms_status)
  - [ ] 2.3 Define `ComplianceGuard.__init__(self, config: ProviderConfig)`
  - [ ] 2.4 Define `validate_startup()` — raises `RuntimeError` if `storage_allowed is False`
  - [ ] 2.5 Define `is_limited_mode()` — True when `terms_status in ("under_review", "restricted")`
  - [ ] 2.6 Define `should_write_raw()` — True when `storage_allowed and not is_limited_mode()`
  - [ ] 2.7 Define `rate_limit_interval()` — returns `1.0 / self.config.rate_limit_rps`
  - [ ] 2.8 Add module-level docstring documenting the first-class compliance design intent

- [ ] **Task 3: Create `services/ingestor/ingestor/normalizer.py`** (AC 3)
  - [ ] 3.1 Import `datetime`, `UTC` from `datetime`; `Optional` from `typing`; `BaseModel` from `pydantic`
  - [ ] 3.2 Import `FeedTypeEnum` from `schema.enums`
  - [ ] 3.3 Define `NormalizedAdsbState(BaseModel)` with all fields per AC 3
  - [ ] 3.4 Implement `normalize_adsb(raw: dict, provider: str, received_at: datetime) -> NormalizedAdsbState`
  - [ ] 3.5 Altitude conversion: `geo_altitude * 3.28084` rounded to 1 decimal
  - [ ] 3.6 Speed conversion: `velocity * 1.94384` rounded to 1 decimal
  - [ ] 3.7 Timestamp: `datetime.fromtimestamp(time_position, tz=UTC)` with fallback to `received_at`
  - [ ] 3.8 call_sign: strip whitespace; None if empty string
  - [ ] 3.9 Raise `ValueError` if `icao24` absent or falsy
  - [ ] 3.10 `geography_entity_id` always `None` — add inline comment explaining the processor handles geospatial matching

- [ ] **Task 4: Replace `services/ingestor/ingestor/main.py`** (AC 4)
  - [ ] 4.1 Import `asyncio`, `logging`; import `BaseSettings` from `pydantic_settings`
  - [ ] 4.2 Import `BaseIngestor` from `ingestor.base`; import `ProviderConfig`, `ComplianceGuard` from `ingestor.compliance`
  - [ ] 4.3 Define `IngestorSettings(BaseSettings)` with all env vars per AC 4
  - [ ] 4.4 Define `PROVIDER_DEFAULTS: dict[str, dict]` with OpenSky rate-limit and compliance defaults (see Dev Notes) — used to build `ProviderConfig` per provider
  - [ ] 4.5 Define `PROVIDER_REGISTRY: dict[str, type[BaseIngestor]] = {}`
  - [ ] 4.6 Implement `async def main()` with steps: load settings → lookup registry → build config → validate startup → instantiate → run
  - [ ] 4.7 Add clear `RuntimeError` message when provider not in registry (not a bare KeyError)
  - [ ] 4.8 Set up stdlib `logging` at the configured level before the startup sequence

- [ ] **Task 5: Update `services/ingestor/ingestor/adapters/__init__.py`** (AC 5)
  - [ ] 5.1 Replace empty file with comment block documenting the expected import pattern for Story 3.2

- [ ] **Task 6: Update `services/ingestor/pyproject.toml`** (AC 6)
  - [ ] 6.1 Add `pydantic-settings>=2.0` to `[project] dependencies`

- [ ] **Task 7: Create `services/ingestor/tests/` and write tests** (AC 7)
  - [ ] 7.1 Create `services/ingestor/tests/__init__.py` (empty)
  - [ ] 7.2 Create `services/ingestor/tests/test_normalizer.py`:
    - [ ] 7.2a Full valid record test (all conversions verified)
    - [ ] 7.2b Missing icao24 → ValueError
    - [ ] 7.2c `time_position=None` fallback to `received_at`
    - [ ] 7.2d `geo_altitude=None` → `altitude_ft=None`
    - [ ] 7.2e `velocity=None` → `speed_knots=None`
    - [ ] 7.2f `geography_entity_id` is always `None`
  - [ ] 7.3 Create `services/ingestor/tests/test_compliance.py`:
    - [ ] 7.3a `storage_allowed=False` → `validate_startup()` raises `RuntimeError`
    - [ ] 7.3b `terms_status="under_review"` → `is_limited_mode()` True
    - [ ] 7.3c `terms_status="compliant"` → `is_limited_mode()` False
    - [ ] 7.3d limited mode → `should_write_raw()` False
    - [ ] 7.3e compliant + storage_allowed → `should_write_raw()` True
    - [ ] 7.3f `rate_limit_rps=5.0` → `rate_limit_interval()` == 0.2

- [ ] **Task 8: Smoke test** (AC 8)
  - [ ] 8.1 `pip install -e services/ingestor/` in repo venv with shared already installed
  - [ ] 8.2 Import smoke test passes (see AC 8)
  - [ ] 8.3 `pytest services/ingestor/tests/ -v` passes (all tests green, no DB or Valkey needed)

---

## Dev Notes

### File Change Map

```
services/ingestor/
├── pyproject.toml                   ← UPDATE: add pydantic-settings>=2.0
├── Dockerfile                       ← DO NOT TOUCH (Story 1.6 / production scope)
└── ingestor/
    ├── __init__.py                  ← DO NOT TOUCH
    ├── main.py                      ← REPLACE stub with full startup sequence
    ├── base.py                      ← CREATE (does not yet exist in repo)
    ├── normalizer.py                ← CREATE (does not yet exist in repo)
    ├── compliance.py                ← POPULATE (was empty stub)
    ├── health.py                    ← DO NOT TOUCH (implemented in Story 3.3)
    └── adapters/
        └── __init__.py              ← UPDATE: add comment block only

tests/                               ← CREATE (directory does not yet exist)
├── __init__.py
├── test_normalizer.py
└── test_compliance.py
```

### Circular Import Prevention

`base.py` imports from `compliance.py`. `compliance.py` must NOT import from `base.py`. Import direction is one-way:
```
compliance.py ← base.py ← main.py
normalizer.py ← main.py (and later: adapters/opensky.py)
```

`main.py` imports from `base`, `compliance`, and (in Story 3.2) from `adapters.opensky`. No circular dependencies at any layer.

### OpenSky State Vector Field Map

OpenSky REST API `/api/states/all` response: `states` is a list of arrays. Each state vector array:

| Index | Field name | Type | Maps to |
|---|---|---|---|
| 0 | `icao24` | str | `icao24` (required) |
| 1 | `callsign` | str\|None | `call_sign` (strip whitespace) |
| 2 | `origin_country` | str | not stored in EntityState |
| 3 | `time_position` | int\|None | `observed_at` (Unix ts → UTC datetime) |
| 4 | `last_contact` | int | not stored separately |
| 5 | `longitude` | float\|None | `longitude` |
| 6 | `latitude` | float\|None | `latitude` |
| 7 | `baro_altitude` | float\|None | not used (use geo_altitude) |
| 8 | `on_ground` | bool | not stored in EntityState |
| 9 | `velocity` | float\|None | `speed_knots` (× 1.94384) |
| 10 | `true_track` | float\|None | `heading` |
| 11 | `vertical_rate` | float\|None | not stored in v1 |
| 12 | `sensors` | list\|None | not stored |
| 13 | `geo_altitude` | float\|None | `altitude_ft` (× 3.28084) |
| 14 | `squawk` | str\|None | not stored in v1 |
| 15 | `spi` | bool | not stored |
| 16 | `position_source` | int | not stored in v1 |

**In Story 3.1**, `normalize_adsb()` accepts a raw **dict** (Story 3.2 converts the array format to dict before passing). The normalizer does not handle index-based array parsing — that is the adapter's responsibility. This keeps the normalizer testable without tying it to OpenSky's wire format.

Example raw dict the normalizer expects:
```python
{
    "icao24": "abc123",
    "callsign": "UAL123  ",   # trailing spaces are normal
    "time_position": 1710000000,
    "longitude": -122.4,
    "latitude": 37.8,
    "geo_altitude": 10500.0,  # meters
    "velocity": 250.0,        # m/s
    "true_track": 270.0,
}
```

### `PROVIDER_DEFAULTS` in `main.py`

Hard-code the default `ProviderConfig` parameters per provider. These are read from architecture Decision 9 / project-context.md provider compliance table:

```python
PROVIDER_DEFAULTS: dict[str, dict] = {
    "opensky": {
        "rate_limit_rps": 5.0,
        "storage_allowed": True,       # subject to non-commercial terms review
        "retention_hours": 48,
        "commercial_use_allowed": False,
        "terms_status": "under_review",  # safe default until explicitly reviewed
    },
    "adsbexchange": {
        "rate_limit_rps": 1.0,         # conservative default; per-plan
        "storage_allowed": True,
        "retention_hours": 48,
        "commercial_use_allowed": True,
        "terms_status": "under_review",
    },
    "ais_noaa": {
        "rate_limit_rps": 10.0,
        "storage_allowed": True,
        "retention_hours": 48,
        "commercial_use_allowed": True,
        "terms_status": "under_review",
    },
}
```

Note: `terms_status="under_review"` is the safe default for all providers. This puts the service in **limited mode** (no raw writes) until a provider is explicitly marked `"compliant"` by the operator. This is the correct behavior per architecture — no production-scale raw storage before terms are confirmed.

### `NormalizedAdsbState` — Relationship to ORM Models

`NormalizedAdsbState` is an intermediate transport object, **not** an ORM model. Story 3.4 maps it to:
- `EntityState` ORM: uses `latitude`/`longitude` to construct a PostGIS `Point(longitude, latitude, srid=4326)` geometry
- `RawAdsb` ORM: uses `raw_payload` and `received_at`

`normalizer.py` must NOT import `EntityState`, `RawAdsb`, or any SQLAlchemy type. It is a pure transformation layer with no I/O.

### `pydantic-settings` Usage Pattern

```python
from pydantic_settings import BaseSettings

class IngestorSettings(BaseSettings):
    provider: str                         # PROVIDER env var (required)
    log_level: str = "info"               # LOG_LEVEL
    database_url: Optional[str] = None    # DATABASE_URL (used in Story 3.4)
    valkey_host: str = "valkey"           # VALKEY_HOST
    valkey_port: int = 6379               # VALKEY_PORT
    opensky_username: Optional[str] = None
    opensky_password: Optional[str] = None
    adsbx_api_key: Optional[str] = None

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"
        extra = "ignore"  # don't fail on unknown env vars from .env
```

### `health.py` — Do Not Touch

`health.py` remains an empty stub. It is fully implemented in Story 3.3 (Valkey pub/sub and feed heartbeat). Do not add placeholder code to it in this story.

### Architecture Compliance Guardrails

1. **No DB access in this story.** The ingestor in 3.1 has no `database_url` in active use, no SQLAlchemy engine, no async session. This constraint is absolute.

2. **No Valkey access in this story.** `redis.asyncio` is not imported or instantiated. Valkey wiring is Story 3.3.

3. **No HTTP client in this story.** `httpx` is not imported. The `fetch()` method is abstract — no HTTP calls are made.

4. **`geography_entity_id` is always `None` from the ingestor.** The processor handles geospatial watch matching. Do not add PostGIS intersection queries to the ingestor — this is an architectural boundary violation.

5. **`compliance.py` is first-class, not a nice-to-have.** `validate_startup()` must be called before any I/O in `main.py`. The architecture is explicit: "Provider terms compliance is a first-class architectural constraint. No production-scale ingestion begins before terms are reviewed."

6. **`PROVIDER_REGISTRY` starts empty.** Story 3.2 adds the OpenSky entry. The empty registry in 3.1 is correct — it proves the dispatch mechanism works before any real adapter exists.

7. **`Dockerfile` is not touched.** The stub Dockerfile has a known build-context limitation noted in its TODO comment (Story 1.6 / production deployment scope). Do not attempt to fix it in this story.

---

## Scope Guardrails

### IN SCOPE — Story 3.1
- `BaseIngestor` abstract class interface
- `ProviderConfig` + `ComplianceGuard` — full implementation
- `NormalizedAdsbState` Pydantic model + `normalize_adsb()` function — full implementation
- `IngestorSettings` Pydantic BaseSettings — full implementation
- `PROVIDER_REGISTRY` dict + startup dispatch in `main.py`
- `PROVIDER_DEFAULTS` hard-coded config per provider
- `pydantic-settings` added to `pyproject.toml`
- Unit tests for normalizer and compliance (pure Python, no I/O)

### DEFERRED — Do NOT implement in this story

| Item | Deferred to |
|---|---|
| OpenSky continuous fetch loop (`adapters/opensky.py`) | Story 3.2 |
| Automatic reconnect on outage | Story 3.2 |
| `health.py` Valkey heartbeat publishing (15s interval, 30s TTL) | Story 3.3 |
| `redis.asyncio` import or Valkey connection | Story 3.3 |
| `entity_states` DB writes | Story 3.4 |
| `raw_adsb` DB writes | Story 3.4 |
| SQLAlchemy engine / async session setup | Story 3.4 |
| Feed health API (`/api/v1/health`) | Story 3.5 |
| Signal Desk feed health indicators | Story 3.6 |
| ADS-B Exchange adapter | Story 9.1 |
| AIS NOAA adapter | Story 9.2 |
| Dockerfile production fix (build context shared dep) | Story 1.6 (VPS deployment) |
| Docker Compose `restart: unless-stopped` policy changes | Story 1.6 |
| Processor logic, baseline computation, signal detection | Epics 4–7 |
| Any schema migrations | None needed; existing schema covers EntityState + RawAdsb |

---

## Dependencies / Prerequisites

| Dependency | Status | Notes |
|---|---|---|
| Story 1.1 — Monorepo scaffold | ✅ Done | `services/ingestor/` directory and stub files exist |
| Story 1.3 — Shared schema package | ✅ Done | `FeedTypeEnum`, `EntityState`, `RawAdsb` available via `from schema import ...` |
| Story 2.x — Epic 2 complete | ✅ Done (2.6 done 2026-03-11) | No direct dependency; noted for sprint sequencing |
| `pydantic-settings` package | Needed | Added to `pyproject.toml` in Task 6 |
| No DB / Valkey / HTTP required | — | All tests are pure Python; no external services needed to pass AC 7 |

---

## References

- [Architecture Decision 9 — Provider terms compliance](../planning-artifacts/architecture.md)
- [Architecture — Ingestor service pipeline](../planning-artifacts/architecture.md) (Collect stage)
- [Project Context — Provider Terms Compliance](../project-context.md#provider-terms-compliance)
- [Project Context — Pipeline Stage Separation](../project-context.md#pipeline-stage-separation)
- [Project Context — Service Boundary Write Rules](../project-context.md#service-boundary-write-rules)
- [Project Context — Redis Key Namespaces](../project-context.md#redis-key-namespaces-applies-to-valkey)
- [epics.md — Epic 3 story list](../planning-artifacts/epics.md)
- [sprint-status.yaml — Epic 3 backlog](sprint-status.yaml)
- [shared/schema/feeds.py](../../shared/schema/feeds.py) — `EntityState`, `RawAdsb`, `FeedTypeEnum`
- [shared/schema/enums.py](../../shared/schema/enums.py) — `FeedTypeEnum`
- [services/ingestor/ingestor/compliance.py](../../services/ingestor/ingestor/compliance.py) — existing stub
- [services/ingestor/ingestor/main.py](../../services/ingestor/ingestor/main.py) — existing stub (replace)
- [.env.example](../../.env.example) — `PROVIDER`, `OPENSKY_USERNAME`, `OPENSKY_PASSWORD` env vars
