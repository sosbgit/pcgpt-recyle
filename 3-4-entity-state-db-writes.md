# Story 3.4: Entity State DB Writes

**Status:** done
**Epic:** 3 — Live Feed Ingestion & Feed Health Visibility
**Sprint sequence:** Fourth story of Epic 3; unblocks 3.5 (feed health API), 4.1 (Processor — reads `entity_states`)

---

## Story

As the Panoptes ingestor service,
I need to persist each successfully normalized ADS-B entity state to the `entity_states` table after every normalization, and persist the raw provider payload to `raw_adsb` only when the compliance posture explicitly permits it,
so that the processor service can query accumulated entity observations for baseline computation, and raw data is retained subject to provider terms — with the current OpenSky posture (`limited_mode=True`, `should_write_raw=False`) correctly blocking raw writes without affecting normalized state persistence.

---

## Acceptance Criteria

### AC 1 — `IngestorSettings`: replace `database_url` field with POSTGRES_* properties

- `services/ingestor/ingestor/main.py` `IngestorSettings` is updated:
  - Remove the existing `database_url: Optional[str] = None` field.
  - Add individual PostgreSQL fields (all Optional to avoid breaking the no-DB code path):
    ```python
    postgres_host: Optional[str] = None      # POSTGRES_HOST
    postgres_port: int = 5432                # POSTGRES_PORT
    postgres_db: str = "panoptes"            # POSTGRES_DB
    postgres_user: str = "panoptes"          # POSTGRES_USER
    postgres_password: Optional[str] = None  # POSTGRES_PASSWORD
    ```
  - Add a `database_url` property that constructs the asyncpg URL when both `postgres_host` and `postgres_password` are set; returns `None` otherwise:
    ```python
    @property
    def database_url(self) -> Optional[str]:
        if self.postgres_host and self.postgres_password:
            return (
                f"postgresql+asyncpg://{self.postgres_user}:{self.postgres_password}"
                f"@{self.postgres_host}:{self.postgres_port}/{self.postgres_db}"
            )
        return None
    ```
  - This mirrors the pattern in `services/api/api/config.py` and is compatible with the existing `.env.example` `POSTGRES_*` variables. No new env vars needed.
- All other `IngestorSettings` fields (`provider`, `log_level`, `opensky_client_id`, etc.) are unchanged.

### AC 2 — `models.py`: SQLAlchemy ORM mapped classes for `entity_states` and `raw_adsb`

- `services/ingestor/ingestor/models.py` created.
- Defines `Base = DeclarativeBase()` scoped to the ingestor service (separate from any future API ORM base — different process).
- Defines `EntityState` mapped class targeting `entity_states` table:
  - `id: Mapped[uuid.UUID]` — primary key, `server_default=sa.text("gen_random_uuid()")`
  - `geography_entity_id: Mapped[Optional[uuid.UUID]]` — nullable; always `None` from ingestor (processor assigns)
  - `provider: Mapped[str]` — `Text`, not nullable
  - `entity_type: Mapped[str]` — `PgEnum(name="feed_type_enum", create_type=False)`; pass string value `"adsb"`
  - `icao24: Mapped[Optional[str]]` — `Text`, nullable
  - `mmsi: Mapped[Optional[str]]` — `Text`, nullable; always `None` for ADS-B
  - `call_sign: Mapped[Optional[str]]` — `Text`, nullable
  - `flight_number: Mapped[Optional[str]]` — `Text`, nullable; always `None` (not in `NormalizedAdsbState`)
  - `vessel_name: Mapped[Optional[str]]` — `Text`, nullable; always `None` for ADS-B
  - `position` — `Geometry("POINT", srid=4326)`, nullable; `None` when lat/lon absent
  - `altitude_ft: Mapped[Optional[float]]` — `Float`, nullable
  - `speed_knots: Mapped[Optional[float]]` — `Float`, nullable
  - `heading: Mapped[Optional[float]]` — `Float`, nullable
  - `observed_at: Mapped[datetime]` — `TIMESTAMP(timezone=True)`, not nullable
  - `created_at: Mapped[datetime]` — `TIMESTAMP(timezone=True)`, `server_default=sa.text("now()")`
- Defines `RawAdsb` mapped class targeting `raw_adsb` table:
  - `id: Mapped[uuid.UUID]` — primary key, `server_default=sa.text("gen_random_uuid()")`
  - `provider: Mapped[str]` — `Text`, not nullable
  - `icao24: Mapped[Optional[str]]` — `Text`, nullable
  - `received_at: Mapped[datetime]` — `TIMESTAMP(timezone=True)`, not nullable
  - `raw_payload: Mapped[dict]` — `JSONB`, not nullable
  - `created_at: Mapped[datetime]` — `TIMESTAMP(timezone=True)`, `server_default=sa.text("now()")`
- `models.py` does NOT call `Base.metadata.create_all()` — Alembic owns the schema.
- No relationships, no backref, no lazy loading — write-only ORM use.

### AC 3 — `writer.py`: `EntityStateWriter` class

- `services/ingestor/ingestor/writer.py` created.
- Defines `EntityStateWriter` class:
  - `__init__(self, session_factory, guard: ComplianceGuard) -> None`
    - Stores `self._session_factory = session_factory`
    - Stores `self._guard = guard`
    - Creates `self._log = logging.getLogger("ingestor.writer")`
    - At init time, logs once at WARNING if `guard.should_write_raw()` is False:
      ```
      "raw_adsb writes disabled: limited_mode=True — terms_status=under_review for this provider"
      ```
      This is the single one-time notice; no per-record warning.
  - `async def write(self, state: NormalizedAdsbState) -> bool`:
    - Opens an async session: `async with self._session_factory() as session:`
    - Within a transaction `async with session.begin():`
      1. Always: `await self._write_entity_state(session, state)`
      2. Conditionally: `await self._write_raw_adsb(session, state)` only when `self._guard.should_write_raw()` returns True
    - Both writes share one transaction — either both commit or both roll back.
    - Returns `True` on success.
    - On exception: logs WARNING with icao24 and error message, does NOT re-raise, returns `False`.
    - DB errors are non-fatal — write failures must not stop the ingestion loop.
  - `async def _write_entity_state(self, session, state: NormalizedAdsbState) -> None`:
    - Constructs position geometry: `WKTElement(f"POINT({state.longitude} {state.latitude})", srid=4326)` when both `state.latitude` and `state.longitude` are not `None`; else `None`.
    - **Note the coordinate order**: `POINT(longitude latitude)` — PostGIS uses (x, y) = (lon, lat). Do NOT swap.
    - Constructs and adds `EntityState` instance per field mapping in Dev Notes.
    - Calls `session.add(entity_state)`.
  - `async def _write_raw_adsb(self, session, state: NormalizedAdsbState) -> None`:
    - Constructs and adds `RawAdsb` instance from `state.raw_payload`, `state.provider`, `state.icao24`, `state.received_at`.
    - Calls `session.add(raw_adsb)`.

### AC 4 — `main.py`: engine, session factory, and writer wired into startup

- `services/ingestor/ingestor/main.py` updated with targeted changes only.
- Adds imports:
  ```python
  from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
  from ingestor.writer import EntityStateWriter
  ```
- In `async def main()`, after the compliance gate log and before Valkey client creation:
  ```python
  db_writer: Optional[EntityStateWriter] = None
  db_url = settings.database_url
  if db_url:
      engine = create_async_engine(db_url, echo=False, pool_pre_ping=True)
      session_factory = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
      db_writer = EntityStateWriter(session_factory, guard)
      log.info(
          "DB writer initialized: entity_states=enabled, raw_adsb=%s",
          guard.should_write_raw(),
      )
  else:
      log.warning(
          "POSTGRES_HOST or POSTGRES_PASSWORD not set — DB writes disabled. "
          "Set POSTGRES_* env vars to enable persistence."
      )
  ```
- Adapter instantiation updated to pass `db_writer`:
  ```python
  adapter = adapter_class(config, settings, redis_client, db_writer)
  ```
- All other `main()` logic (settings, registry, compliance gate, heartbeat, gather) is unchanged.
- `echo=False` — do not log every SQL statement; ingestor writes are high-frequency.

### AC 5 — `opensky.py`: writer param, per-record write call, `written_count` tracking

- `services/ingestor/ingestor/adapters/opensky.py` updated.
- `__init__` signature extended to accept an optional writer:
  ```python
  def __init__(self, config: ProviderConfig, settings=None, redis_client=None, db_writer=None) -> None:
  ```
  - Stores `self._db_writer = db_writer`.
  - All other `__init__` logic (token, httpx client, guard, logger, `_redis`) unchanged.
- `run()` loop updated:
  - Add `written_count = 0` alongside the other per-cycle counters.
  - After `await self._publish_state(state)`, add:
    ```python
    if self._db_writer is not None:
        if await self._db_writer.write(state):
            written_count += 1
    ```
  - Cycle log updated to include `written=N`:
    ```
    Cycle complete: fetched={N}, normalized={N}, skipped={N}, published={N}, written={N}
    ```
  - If `self._db_writer` is `None`: `written_count` stays 0, log shows `written=0`.
  - No structural change to the fetch/normalize/backoff loop from Story 3.3.
- All prior Stories 3.2/3.3 `__init__` call patterns (`OpenSkyAdapter(config)`, `OpenSkyAdapter(config, settings)`, `OpenSkyAdapter(config, settings, redis_client)`) remain valid — `db_writer` defaults to `None`.

### AC 6 — `pyproject.toml`: add DB dependencies

- `services/ingestor/pyproject.toml` updated:
  ```toml
  [project]
  dependencies = [
      "panoptes-shared",
      "pydantic-settings>=2.0",
      "httpx>=0.27",
      "redis>=5.0",
      "sqlalchemy[asyncio]>=2.0",
      "asyncpg>=0.29",
      "geoalchemy2>=0.14",
  ]
  ```

### AC 7 — `tests/test_writer.py`: unit test suite for `EntityStateWriter`

- `services/ingestor/tests/test_writer.py` created.
- All tests use `AsyncMock` session and a mock session factory — no live DB required.
- Required test coverage:

  **Group W — `EntityStateWriter.write()`**
  - W1: Successful write with `should_write_raw=False` (default OpenSky posture) → `session.add()` called exactly once (entity_state only); transaction committed; `write()` returns `True`.
  - W2: `should_write_raw=True` → `session.add()` called exactly twice (entity_state + raw_adsb); both within one `begin()` context.
  - W3: `should_write_raw=False` → `session.add()` called exactly once; a `RawAdsb` instance is never passed to `session.add()`.
  - W4: `session.begin()` context raises `SQLAlchemyError` → `write()` returns `False`; WARNING logged; no exception propagated to caller.
  - W5: `EntityState` fields populated correctly from `NormalizedAdsbState`: `provider`, `icao24`, `call_sign`, `altitude_ft`, `speed_knots`, `heading`, `observed_at` all match input state; `mmsi`, `flight_number`, `vessel_name` are all `None`.
  - W6: `state.latitude=None` → `EntityState.position` is `None`; no `WKTElement` constructed; no crash.
  - W7: `state.longitude=None` (with latitude set) → `EntityState.position` is `None`.
  - W8: Both `state.latitude` and `state.longitude` are floats → `EntityState.position` is a `WKTElement`; the WKT string is `f"POINT({longitude} {latitude})"` (lon first).
  - W9: `entity_type` field on `EntityState` is the string `"adsb"` (`.value` extracted from `FeedTypeEnum.adsb`), not the enum object.

- `pytest services/ingestor/tests/ -v` passes — all prior Groups A–D (Story 3.2) and Groups H, V (Story 3.3) still green alongside new Group W.

### AC 8 — Smoke test: DB writes end-to-end

- With PostgreSQL+PostGIS running, schema migrated (`alembic upgrade head`), and `POSTGRES_*` vars set in `.env`:
  ```bash
  python -m ingestor.main
  ```
  - Service starts without error; logs confirm "DB writer initialized".
  - After first poll cycle (≤120s): rows present in `entity_states` with `provider='opensky'`, `entity_type='adsb'`.
  - `raw_adsb` table remains empty (OpenSky `limited_mode=True` → `should_write_raw=False`).
  - Rows with a valid lat/lon have a non-null PostGIS `position` value; rows without position fix have `position=NULL`.
  - Cycle log includes `written=N` where N ≈ normalized count.
  - `geography_entity_id` is `NULL` on all rows (processor assigns — Epic 4).
- With `POSTGRES_HOST` or `POSTGRES_PASSWORD` absent:
  - Service starts without crashing; WARNING logged about missing config; Valkey publish continues unaffected.
- **Do NOT run the live smoke test in CI.** This AC is local manual verification only.

---

## Tasks / Subtasks

- [x] **Task 1: Update `IngestorSettings` in `main.py`** (AC 1)
  - [x] 1.1 Remove `database_url: Optional[str] = None` field
  - [x] 1.2 Add `postgres_host: Optional[str] = None`, `postgres_port: int = 5432`, `postgres_db: str = "panoptes"`, `postgres_user: str = "panoptes"`, `postgres_password: Optional[str] = None`
  - [x] 1.3 Add `@property def database_url(self) -> Optional[str]` — construct asyncpg URL when host+password present; return `None` otherwise
  - [x] 1.4 Verify: all other `IngestorSettings` fields unchanged; `class Config` block unchanged

- [x] **Task 2: Create `services/ingestor/ingestor/models.py`** (AC 2)
  - [x] 2.1 Import: `uuid`, `datetime`, `typing.Optional`, `typing.Any`, `sqlalchemy as sa`, `sqlalchemy.orm.DeclarativeBase`, `sqlalchemy.orm.Mapped`, `sqlalchemy.orm.mapped_column`, `sqlalchemy.dialects.postgresql.JSONB`, `sqlalchemy.dialects.postgresql.ENUM as PgEnum`, `geoalchemy2.Geometry`
  - [x] 2.2 Define `Base = DeclarativeBase()`
  - [x] 2.3 Define `EntityState(Base)` with all columns per AC 2 spec — pay attention to nullable/not-nullable per migration
  - [x] 2.4 Define `RawAdsb(Base)` with all columns per AC 2 spec
  - [x] 2.5 Verify: no `create_all()`, no relationships, no `__repr__` needed

- [x] **Task 3: Create `services/ingestor/ingestor/writer.py`** (AC 3)
  - [x] 3.1 Import: `logging`, `geoalchemy2.WKTElement`, `ingestor.compliance.ComplianceGuard`, `ingestor.normalizer.NormalizedAdsbState`, `ingestor.models.EntityState`, `ingestor.models.RawAdsb`
  - [x] 3.2 Implement `EntityStateWriter.__init__` — session_factory, guard, log; one-time WARNING log if `should_write_raw()` is False
  - [x] 3.3 Implement `_write_entity_state()` — WKTElement position construction (lon, lat order), entity_type value extraction, None for mmsi/flight_number/vessel_name, session.add()
  - [x] 3.4 Implement `_write_raw_adsb()` — raw_payload dict, provider, icao24, received_at, session.add()
  - [x] 3.5 Implement `write()` — async session context, begin() transaction, compliance gate for raw write, try/except with WARNING log, return bool
  - [x] 3.6 Verify: no per-record warning log for raw skip — one-time only at init

- [x] **Task 4: Update `services/ingestor/ingestor/main.py`** (AC 4)
  - [x] 4.1 Add SQLAlchemy async imports: `create_async_engine`, `async_sessionmaker`, `AsyncSession`
  - [x] 4.2 Add `from ingestor.writer import EntityStateWriter`
  - [x] 4.3 After compliance gate log: conditionally create `engine` → `session_factory` → `EntityStateWriter` when `settings.database_url` is not None
  - [x] 4.4 Log INFO when writer initialized (with raw_adsb status); log WARNING when DB vars absent
  - [x] 4.5 Pass `db_writer` as 4th arg to `adapter_class(config, settings, redis_client, db_writer)`
  - [x] 4.6 Verify: heartbeat creation, `asyncio.gather`, all other main() logic unchanged

- [x] **Task 5: Update `services/ingestor/ingestor/adapters/opensky.py`** (AC 5)
  - [x] 5.1 Add `db_writer=None` as 4th parameter to `__init__`; store `self._db_writer = db_writer`
  - [x] 5.2 Add `written_count = 0` to `run()` cycle initializers (alongside `success_count`, `skip_count`, `published_count`)
  - [x] 5.3 After `_publish_state()` call in normalize loop: if `self._db_writer` is not None, call `await self._db_writer.write(state)`; increment `written_count` on `True` return
  - [x] 5.4 Update cycle log format string: add `written={written_count}`
  - [x] 5.5 Verify: Groups A–D and V tests all pass with updated `__init__` signature (db_writer defaults to None — backward compat)

- [x] **Task 6: Update `services/ingestor/pyproject.toml`** (AC 6)
  - [x] 6.1 Add `sqlalchemy[asyncio]>=2.0`, `asyncpg>=0.29`, `geoalchemy2>=0.14` to `[project] dependencies`

- [x] **Task 7: Create `services/ingestor/tests/test_writer.py`** (AC 7)
  - [x] 7.1 Import `pytest`, `AsyncMock`, `MagicMock`, `EntityStateWriter`, `ComplianceGuard`, `ProviderConfig`, `NormalizedAdsbState`, `WKTElement`
  - [x] 7.2 Create helper fixtures: `make_state(icao24="abc123", latitude=37.0, longitude=-122.0, ...)`, `make_guard(write_raw=False)`, `make_session_factory()`
  - [x] 7.3 Write W1: one session.add call (entity_state only), returns True
  - [x] 7.4 Write W2: should_write_raw=True → two session.add calls
  - [x] 7.5 Write W3: should_write_raw=False → one session.add call, no RawAdsb added
  - [x] 7.6 Write W4: begin() raises → returns False, WARNING logged, no exception
  - [x] 7.7 Write W5: EntityState field assertions for provider, icao24, call_sign, altitude_ft, speed_knots, heading, observed_at; mmsi/flight_number/vessel_name all None
  - [x] 7.8 Write W6: latitude=None → position=None
  - [x] 7.9 Write W7: longitude=None → position=None
  - [x] 7.10 Write W8: both lat and lon present → position is WKTElement with `POINT(lon lat)` string
  - [x] 7.11 Write W9: entity_type is string `"adsb"`, not the enum object
  - [x] 7.12 Run full test suite: `pytest services/ingestor/tests/ -v` — all Groups A–D, H, V, W pass

- [x] **Task 8: Local smoke test** (AC 8)
  - [x] 8.1 `pip install -e shared/ -e "services/ingestor[test]"` — installs sqlalchemy, asyncpg, geoalchemy2
  - [x] 8.2 Import smoke: `python -c "from ingestor.writer import EntityStateWriter; from ingestor.models import EntityState, RawAdsb; print('ok')"`
  - [x] 8.3 `pytest services/ingestor/tests/ -v` — all prior groups + new W group pass
  - [x] 8.4 With live PostgreSQL + Valkey, run `alembic upgrade head`, then `python -m ingestor.main`; verify `entity_states` rows after first cycle, `raw_adsb` empty, `written=N` in cycle log

---

## Dev Notes

### File Change Map

```
services/ingestor/
├── pyproject.toml                    ← UPDATE: sqlalchemy[asyncio]>=2.0, asyncpg>=0.29, geoalchemy2>=0.14
└── ingestor/
    ├── main.py                       ← UPDATE: IngestorSettings (postgres vars + property),
    │                                            engine + session factory + writer creation,
    │                                            db_writer passed to adapter
    ├── models.py                     ← CREATE: EntityState and RawAdsb ORM mapped classes
    ├── writer.py                     ← CREATE: EntityStateWriter
    ├── base.py                       ← DO NOT TOUCH (Story 3.1 seam)
    ├── normalizer.py                 ← DO NOT TOUCH (Story 3.1 seam)
    ├── compliance.py                 ← DO NOT TOUCH (Story 3.1 seam)
    ├── health.py                     ← DO NOT TOUCH (Story 3.3 seam)
    └── adapters/
        ├── __init__.py               ← DO NOT TOUCH
        └── opensky.py                ← UPDATE: db_writer param, write() call, written_count

services/ingestor/tests/
├── test_writer.py                    ← CREATE (Groups W1–W9)
└── test_opensky_adapter.py           ← VERIFY: Groups A–D and V still pass; no test changes required
```

### OpenSky Compliance Posture (must be preserved exactly)

Current `PROVIDER_DEFAULTS["opensky"]` in `main.py`:

```python
"opensky": {
    "rate_limit_rps": 5.0,
    "storage_allowed": True,
    "retention_hours": 48,
    "commercial_use_allowed": False,
    "terms_status": "under_review",   # ← limited_mode=True
}
```

`ComplianceGuard` evaluation:

| Method | Result | Reason |
|---|---|---|
| `is_limited_mode()` | `True` | `terms_status="under_review"` |
| `should_write_raw()` | `False` | `storage_allowed=True AND NOT limited_mode` → `True AND False` → **`False`** |

This means for OpenSky in Story 3.4:
- `entity_states` writes: **always enabled** — normalized state is the safe output
- `raw_adsb` writes: **blocked** — limited_mode prevents raw storage until terms are confirmed

Do NOT change `PROVIDER_DEFAULTS` for OpenSky. Do NOT add logic that bypasses `should_write_raw()`. The compliance gate lives exclusively in `writer.py`.

### `entity_states` Field Mapping

| `NormalizedAdsbState` field | `entity_states` column | Notes |
|---|---|---|
| `provider` | `provider` | direct |
| `entity_type.value` | `entity_type` | Python enum → string `"adsb"` |
| `icao24` | `icao24` | direct |
| — | `mmsi` | always `None` for ADS-B |
| `call_sign` | `call_sign` | direct |
| — | `flight_number` | always `None` — not in `NormalizedAdsbState` |
| — | `vessel_name` | always `None` for ADS-B |
| lat/lon → `WKTElement` | `position` | `None` when either lat or lon is `None` |
| `altitude_ft` | `altitude_ft` | direct |
| `speed_knots` | `speed_knots` | direct |
| `heading` | `heading` | direct |
| `observed_at` | `observed_at` | direct, UTC `datetime` |
| — | `geography_entity_id` | always `None` — processor assigns via geospatial watch-matching (Epic 4) |
| — | `created_at` | DB server default `now()` — do not set in ORM object |

### `raw_adsb` Field Mapping

| `NormalizedAdsbState` field | `raw_adsb` column | Notes |
|---|---|---|
| `provider` | `provider` | direct |
| `icao24` | `icao24` | direct |
| `received_at` | `received_at` | direct |
| `raw_payload` | `raw_payload` | original unmodified OpenSky state-vector dict |
| — | `created_at` | DB server default `now()` — do not set in ORM object |

### PostGIS Point Construction (critical: longitude first)

```python
from geoalchemy2 import WKTElement

# In _write_entity_state():
if state.latitude is not None and state.longitude is not None:
    position = WKTElement(f"POINT({state.longitude} {state.latitude})", srid=4326)
else:
    position = None
```

**POINT(longitude latitude)** — PostGIS WKT uses (x=longitude, y=latitude) order. This matches the migration's `Geometry("POINT", srid=4326, spatial_index=False)` column definition. Swapping lat/lon is a silent data corruption bug — positions would appear in the wrong hemisphere.

### ORM Model Pattern (write-only)

```python
# models.py — minimal excerpt showing key patterns
import uuid
from datetime import datetime
from typing import Optional, Any
import sqlalchemy as sa
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy.dialects.postgresql import JSONB, ENUM as PgEnum
from sqlalchemy import Text, Float, TIMESTAMP
from geoalchemy2 import Geometry

class Base(DeclarativeBase):
    pass

class EntityState(Base):
    __tablename__ = "entity_states"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, server_default=sa.text("gen_random_uuid()"))
    geography_entity_id: Mapped[Optional[uuid.UUID]] = mapped_column(nullable=True)
    provider: Mapped[str] = mapped_column(Text, nullable=False)
    entity_type: Mapped[str] = mapped_column(PgEnum(name="feed_type_enum", create_type=False), nullable=False)
    icao24: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    mmsi: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    call_sign: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    flight_number: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    vessel_name: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    position: Mapped[Optional[Any]] = mapped_column(Geometry("POINT", srid=4326), nullable=True)
    altitude_ft: Mapped[Optional[float]] = mapped_column(Float, nullable=True)
    speed_knots: Mapped[Optional[float]] = mapped_column(Float, nullable=True)
    heading: Mapped[Optional[float]] = mapped_column(Float, nullable=True)
    observed_at: Mapped[datetime] = mapped_column(TIMESTAMP(timezone=True), nullable=False)
    created_at: Mapped[datetime] = mapped_column(TIMESTAMP(timezone=True), server_default=sa.text("now()"))
```

### Write Path Order Within `run()` Normalize Loop

```
# Per-record order (Story layer annotations):
1. state = normalize_adsb(...)              # Story 3.1
2. success_count += 1
3. if await self._publish_state(state):     # Story 3.3
       published_count += 1
4. if self._db_writer is not None:          # Story 3.4
       if await self._db_writer.write(state):
           written_count += 1
```

The DB write is async, sequential within the per-record loop, and non-fatal: a write failure does not skip the next record.

### `write()` Transaction Semantics

Both `entity_states` and `raw_adsb` (when permitted) are written in a single transaction via `async with session.begin()`. If the `entity_states` INSERT succeeds but `raw_adsb` INSERT fails (or vice versa), both roll back. This is intentional — partial writes are worse than no write.

Since `should_write_raw()` is currently `False` for OpenSky, only the `entity_states` INSERT occurs in each transaction.

### Testing `EntityStateWriter` Without Live DB

```python
from unittest.mock import AsyncMock, MagicMock, call
import pytest

@pytest.fixture
def mock_session_factory():
    session = MagicMock()
    session.add = MagicMock()
    # begin() is an async context manager
    session.begin = MagicMock()
    session.begin.return_value.__aenter__ = AsyncMock(return_value=None)
    session.begin.return_value.__aexit__ = AsyncMock(return_value=False)
    # session factory is an async context manager
    factory = MagicMock()
    factory.return_value.__aenter__ = AsyncMock(return_value=session)
    factory.return_value.__aexit__ = AsyncMock(return_value=False)
    return factory, session

@pytest.fixture
def make_guard():
    def _make(write_raw: bool = False):
        config = ProviderConfig(
            rate_limit_rps=5.0,
            storage_allowed=True,
            retention_hours=48,
            commercial_use_allowed=False,
            terms_status="compliant" if write_raw else "under_review",
        )
        return ComplianceGuard(config)
    return _make
```

### `asyncpg` and `geoalchemy2` Compatibility

`geoalchemy2` handles the Python ↔ PostGIS geometry conversion automatically when using SQLAlchemy ORM with `asyncpg`. The `WKTElement(wkt_string, srid=4326)` object is the correct input for an ORM INSERT on a `Geometry("POINT", srid=4326)` column — geoalchemy2 converts it to the appropriate binary format for asyncpg.

If a geometry roundtrip is needed (e.g. in tests that check position values), use `geoalchemy2.shape.to_shape(EntityState.position)` to convert the stored geometry back to a Shapely object. This is not needed for the write-only ingestor story but useful context.

### Windows / Local Dev Install Sequence

```bash
# From repo root (active venv)
pip install -e shared/ -e "services/ingestor[test]"
```

New deps `sqlalchemy[asyncio]>=2.0`, `asyncpg>=0.29`, and `geoalchemy2>=0.14` install automatically.

For the smoke test, PostgreSQL+PostGIS must be reachable. Start with:
```bash
docker compose up postgres -d
```
Then run migrations:
```bash
alembic upgrade head
```
Then start the ingestor:
```bash
python -m ingestor.main
```

### Pydantic Class Config Deprecation (carry-forward from 3.2)

`IngestorSettings` uses `class Config:` (Pydantic v1 style) and emits a deprecation warning at startup. Pre-existing technical debt. Do NOT attempt to fix it in this story.

---

## Scope Guardrails

### IN SCOPE — Story 3.4

- `IngestorSettings`: replace `database_url` field with individual POSTGRES_* fields + `database_url` property
- `models.py`: `EntityState` and `RawAdsb` SQLAlchemy ORM mapped classes (write-only)
- `writer.py`: `EntityStateWriter` with compliance-gated raw write, non-fatal error handling
- `main.py`: engine creation, session factory, `EntityStateWriter` instantiation, passed to adapter
- `opensky.py`: `db_writer` parameter, `write()` call per normalized state, `written_count` in cycle log
- `pyproject.toml`: `sqlalchemy[asyncio]>=2.0`, `asyncpg>=0.29`, `geoalchemy2>=0.14`
- Unit tests for `EntityStateWriter` (Groups W1–W9)
- Compliance posture preserved: `entity_states` always written; `raw_adsb` gated by `should_write_raw()`

### DEFERRED — Do NOT implement in this story

| Item | Deferred to |
|---|---|
| Feed health API (`/api/v1/health`) | Story 3.5 |
| Signal Desk feed health indicators (UI) | Story 3.6 |
| `geography_entity_id` population on `entity_states` | Epic 4 — processor assigns via geospatial watch-matching |
| Heartbeat-based watch staleness | Story 4.4 |
| Processor service and Valkey subscriber | Story 4.1 |
| Baseline computation, watch FSM | Stories 4.2–4.3 |
| 4-tier retention TTL enforcement (48h raw purge, 37d entity-state purge) | Story 9.4 |
| ADS-B Exchange adapter | Story 9.1 |
| AIS NOAA adapter + `raw_ais` writes | Story 9.2 |
| Signal detection, evidence, signals table writes | Epics 5–7 |
| ORM models in API service (separate process) | API stories only |
| Dockerfile / Docker Compose changes | Story 1.6 (production) |
| Any frontend changes | Not in scope for any ingestor story |
| Connection pooling tuning or production pool sizes | Story 9.8 |
| Changing OpenSky compliance posture (`terms_status="compliant"`) | Operator decision — not a story |

---

## Dependencies / Prerequisites

| Dependency | Status | Notes |
|---|---|---|
| Story 3.1 — Ingestor service base | ✅ Done (2026-03-11) | `NormalizedAdsbState`, `ComplianceGuard.should_write_raw()`, `IngestorSettings` all present |
| Story 3.2 — OpenSky adapter | ✅ Done (2026-03-11) | `OpenSkyAdapter.run()` loop seam; `db_writer=None` extends signature backward-compatibly |
| Story 3.3 — Valkey pub/sub | ✅ Done (2026-03-11) | `redis_client` wired; `run()` loop already has publish step; this story adds DB write as the next step |
| Story 1.2 — Alembic initial migration | ✅ Done (2026-03-09) | `entity_states` and `raw_adsb` tables exist in schema; **no new migrations needed** |
| POSTGRES_* env vars in `.env.example` | ✅ Present | `POSTGRES_HOST`, `POSTGRES_PORT`, `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` all declared |
| `sqlalchemy[asyncio]>=2.0`, `asyncpg>=0.29`, `geoalchemy2>=0.14` | Added this story | Not yet in ingestor `pyproject.toml` |
| Running PostgreSQL+PostGIS | For smoke test only | `docker compose up postgres -d` + `alembic upgrade head` |

---

## Config / Env Expectations Before Implementation

No new env vars required. The ingestor reads the same `POSTGRES_*` vars already present in `.env.example`.

| Env Var | `.env.example` value | Story 3.4 use |
|---|---|---|
| `POSTGRES_HOST` | `postgres` | `IngestorSettings.postgres_host` → `database_url` property |
| `POSTGRES_PORT` | `5432` | `IngestorSettings.postgres_port` → `database_url` property |
| `POSTGRES_DB` | `panoptes` | `IngestorSettings.postgres_db` → `database_url` property |
| `POSTGRES_USER` | `panoptes` | `IngestorSettings.postgres_user` → `database_url` property |
| `POSTGRES_PASSWORD` | `changeme_in_production` | `IngestorSettings.postgres_password` → `database_url` property |
| `VALKEY_HOST` | `valkey` | Unchanged from Story 3.3 |
| `VALKEY_PORT` | `6379` | Unchanged from Story 3.3 |
| `PROVIDER` | `opensky` | Unchanged |
| `OPENSKY_CLIENT_ID` | — | Unchanged |
| `OPENSKY_CLIENT_SECRET` | — | Unchanged |

Constructed `database_url` property value (example):
```
postgresql+asyncpg://panoptes:changeme_in_production@postgres:5432/panoptes
```

For local dev (outside Docker Compose): set `POSTGRES_HOST=localhost` in `.env` to reach the forwarded port.

---

## References

- [Story 3.1 — Ingestor Service Base](3-1-ingestor-service-base.md) — `NormalizedAdsbState`, `ComplianceGuard.should_write_raw()`, `IngestorSettings`
- [Story 3.2 — OpenSky Adapter](3-2-opensky-adapter.md) — `OpenSkyAdapter.run()` loop; `__init__` extension pattern
- [Story 3.3 — Valkey Pub/Sub](3-3-valkey-pubsub.md) — `run()` publish seam; `db_writer` is placed immediately after publish in the loop
- [Story 1.2 — Alembic Initial Migration](1-2-alembic-initial-migration.md) — `entity_states` and `raw_adsb` schema; column definitions
- [Architecture](../planning-artifacts/architecture.md) — 4-tier retention; Decision 9 (compliance)
- [Project Context — Pipeline Stage Separation](../project-context.md) — Collect stage owns raw + entity_state writes
- [Project Context — Provider Terms Compliance](../project-context.md) — `should_write_raw()` gate rationale
- [epics.md — Epic 3: Entity state DB writes](../planning-artifacts/epics.md)
- [sprint-status.yaml](sprint-status.yaml) — `3-4-entity-state-db-writes: backlog → ready-for-dev`
- [services/ingestor/ingestor/main.py](../../services/ingestor/ingestor/main.py) — `IngestorSettings`, `main()` entry point
- [services/ingestor/ingestor/adapters/opensky.py](../../services/ingestor/ingestor/adapters/opensky.py) — `__init__` and `run()` to extend
- [services/ingestor/pyproject.toml](../../services/ingestor/pyproject.toml) — dependencies to update
- [migrations/versions/001_initial_schema.py](../../migrations/versions/001_initial_schema.py) — authoritative column definitions for `entity_states` and `raw_adsb`
- [.env.example](../../.env.example) — POSTGRES_* vars confirmed present

---

## Dev Agent Record

### Completion Notes

**Initial implementation** completed 2026-03-11:
- `models.py` created: `EntityState` and `RawAdsb` ORM mapped classes per AC 2 spec. No `create_all()`, no relationships, write-only use.
- `writer.py` created: `EntityStateWriter` with one-time init WARNING for raw skip, `write()` atomic transaction, `_write_entity_state()` (WKT lon-lat order), `_write_raw_adsb()`.
- `main.py` updated: `IngestorSettings` field replaced with individual POSTGRES_* fields + `database_url` property; engine + `async_sessionmaker` + `EntityStateWriter` wired conditionally; `db_writer` passed to adapter.
- `opensky.py` updated: `db_writer=None` param added to `__init__`; `written_count` initialized in `run()`; write-after-publish call added; cycle log extended with `written={N}`.
- `pyproject.toml` updated: `sqlalchemy[asyncio]>=2.0`, `asyncpg>=0.29`, `geoalchemy2>=0.14` added.
- `tests/test_writer.py` created: Groups W1–W9 covering all `EntityStateWriter` paths using `AsyncMock` session — no live DB required.

**Code-review fixes** applied 2026-03-11:
- F1: Removed duplicate broad `except Exception` block from `writer.py` — only `except SQLAlchemyError` DB-failure handler retained.
- F2: Corrected stale `run()` docstring in `opensky.py` — now correctly documents `self._db_writer` DB writes as a Story 3.4 responsibility.

**Final DEV code review passed** 2026-03-11: F1 and F2 both confirmed resolved; no new issues introduced.

### Manual Smoke Test — 2026-03-11 (Nikku, Windows local)

Command: `PROVIDER=opensky python -m ingestor.main` from repo root in active venv, with PostgreSQL+PostGIS and Valkey running locally.

Results:
1. Startup succeeded with no `RuntimeError`.
2. Log: `Compliance OK: limited_mode=True, should_write_raw=False, interval=0.20s`
3. Log: `DB writer initialized: entity_states=enabled, raw_adsb=False`
4. One-time writer warning: `raw_adsb writes disabled: limited_mode=True — terms_status=under_review for this provider`
5. OpenSky token request succeeded.
6. OpenSky states request succeeded.
7. Cycle log: `Cycle complete: fetched=8036, normalized=8036, skipped=0, published=8036, written=8036`
8. Manual DB check: `entity_states` count = 8036 ✅
9. Manual DB check: `raw_adsb` count = 0 ✅ (compliance gate working)
10. Manual DB check: `geography_entity_id` NULL on sampled rows ✅ (processor assigns — Epic 4)
11. Process stopped with Ctrl+C. `CancelledError` during asyncio sleep + `KeyboardInterrupt` on exit — non-blocking expected asyncio.gather teardown behavior.
12. Pydantic class-based config deprecation warning observed at startup — non-blocking pre-existing technical debt (carry-forward from Story 3.2).

**Story 3.4 closed by Nikku after local validation (2026-03-11).**
