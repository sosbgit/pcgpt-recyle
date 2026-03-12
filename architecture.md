---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
inputDocuments:
  - _bmad-output/planning-artifacts/product-brief-panoptes-2026-03-07.md
  - _bmad-output/planning-artifacts/prd.md
  - _bmad-output/planning-artifacts/brainstorming-report.md
workflowType: 'architecture'
project_name: 'panoptes'
user_name: 'Nikku'
date: '2026-03-08'
status: 'complete'
completedAt: '2026-03-08'
---

# Architecture Decision Document — Panoptes v1

**Author:** Nikku
**Date:** 2026-03-08
**Status:** Complete

---

## Project Context Analysis

### Requirements Overview

**Functional Requirements (45 items, 7 categories):**

| Category | Count | Architectural Implication |
|---|---|---|
| Feed Ingestion & Data Collection | FR1–FR4 | Multiple isolated ingestor processes; normalized entity model; provider-specific adapters |
| Watch Management & Configuration | FR5–FR10 | Persistent watch store; baseline lifecycle FSM; compatible vs. incompatible change detection |
| Intelligence Processing & Baseline Management | FR11–FR22 | Always-on processor; rolling statistical baselines; 8 signal detection algorithms; compound convergence engine |
| Signal Desk & Situational Awareness | FR23–FR29 | Real-time WebSocket push; NOMINAL state machine; session-delta computation; Domain Pulse |
| Evidence, Investigation, Provenance | FR30–FR35 | Append-only evidence records; provenance hash chain; signal-triggered spatial layer scoping |
| Operator Workflow | FR36–FR38 | Dismiss-with-reason write path; single-operator authentication |
| System Health & Data Model | FR39–FR45 | 4-tier retention; named geographic entity anchoring; no-silent-failure instrumentation; restart-safe persistence |

**Non-Functional Requirements (21 items):**

- **Performance:** Continuous feed throughput within provider rate limits; incremental (not full-scan) baseline recomputation; near-real-time signal surfacing; sub-second Evidence Surface drill-down
- **Reliability:** All durable state persists across restarts; automatic provider reconnection; no silent operational failures; sustained weeks-long operation without daily operator intervention
- **Security:** Required authentication; all data stored locally; intelligence outputs never transmitted externally; credentials managed via env vars, never in DB or logs
- **Data Integrity:** Evidence records are append-only; baseline state protected during concurrent writes; incompatible watch changes trigger explicit baseline reset; confidence scores accurately reflect degraded-source state
- **Operability:** Declarative containerized deployment; health assessable from cockpit without SSH; container-level fault isolation

**Scale & Complexity:** High. This is a continuous stream-processing platform with a statistical intelligence layer, geospatial analytics, real-time push, and compound signal detection — all in a single self-hosted deployment operated by one person.

### Technical Constraints & Dependencies

- **Deployment is fixed:** Contabo VPS, Linux, Docker Compose, single-server
- **No managed cloud services:** Everything self-hosted
- **Provider terms compliance:** Rate limits, storage limits, commercial-use restrictions — first-class design constraints, not post-build checks
- **Solo builder:** Architecture complexity must be achievable by one person to operational readiness; monolith-with-service-boundaries preferred over microservices
- **Persistence is non-negotiable:** In-memory-only state is a failure mode; restart recovery is a hard requirement
- **Single operator:** Eliminates multi-user session handling, permission matrix, tenant isolation, mobile breakpoints

### Cross-Cutting Concerns

1. **Data retention and cleanup** — four structurally distinct tiers; requires automated cleanup per tier
2. **Feed health propagation** — ingestor health must reach Signal Desk reliably in near-real-time
3. **Baseline integrity** — concurrent ingestion writes must not corrupt rolling baseline state
4. **Confidence degradation accounting** — compound confidence must accurately reflect partial source availability at detection time
5. **Provider terms compliance** — rate limits and storage constraints embedded in ingestion service design
6. **Geographic entity anchoring** — every signal, evidence artifact, and watch attaches to named entities; coordinates are supplementary, not primary
7. **Extensibility hooks** — data model preserves typed extension points (nullable FKs, tagged columns) for future context layers without implementing them in v1
8. **Real-time push** — Signal Desk live updates require server-push mechanism; DB polling is not acceptable for feed health and new signals

---

## Starter Template & Technology Stack

### Primary Technology Domain

Panoptes v1 is a **full-stack self-hosted processing platform** composed of:
- Python backend services (data ingestion, intelligence processing, API)
- React/TypeScript SPA (cockpit frontend)
- PostgreSQL + PostGIS (primary data store)
- Valkey 7 (real-time event bus and entity state cache)
- Docker Compose (deployment)

### Technology Stack Decisions

No single starter template fits this architecture. The project is initialized as a **monorepo** with service-level structure established per-service.

#### Backend: Python with FastAPI

**Decision:** Python 3.12 for all backend services.

**Rationale:**
- Python is the strongest fit for the intelligence processing core: NumPy/Pandas for rolling statistics, Shapely/GeoAlchemy2 for geospatial, asyncio for concurrent feed ingestion
- FastAPI provides async HTTP + native WebSocket in one framework — exactly what the API service needs
- Single backend language eliminates inter-service protocol friction for a solo builder
- Pydantic (FastAPI's native schema layer) gives shared data models between services with validation

**Key Python libraries:**
- `fastapi` + `uvicorn` — API server
- `sqlalchemy[asyncio]` + `alembic` — async ORM + migrations
- `asyncpg` — high-performance async PostgreSQL driver
- `geoalchemy2` + `shapely` — PostGIS integration and geospatial geometry
- `numpy` + `pandas` — baseline statistics computation
- `redis.asyncio` — async Valkey-compatible client (redis.asyncio is wire-compatible with Valkey 7)
- `pydantic-settings` — environment-based configuration
- `httpx` — async HTTP for provider APIs
- `websockets` — WebSocket feed connections (ADS-B Exchange)

#### Frontend: React + TypeScript + Vite

**Decision:** React 18 + TypeScript 5 + Vite 5.

**Rationale:**
- React has the broadest ecosystem for the geospatial and data visualization components Panoptes needs
- Vite provides fast HMR during development on a self-hosted machine
- TypeScript ensures shared interface contracts between cockpit and API responses
- AI coding agents generate more consistent React/TS code than alternatives

**Key frontend libraries:**
- `react-query` (`@tanstack/query`) — server state management and cache; handles polling intervals and stale-state
- `zustand` — lightweight client UI state (active signal selection, surface navigation, session state)
- `maplibre-gl` — WebGL map rendering for Spatial View (open-source Mapbox GL fork)
- `tailwindcss` — utility-first CSS; cockpit-appropriate dense information design
- `shadcn/ui` — unstyled composable component primitives built on Radix UI; operator cockpit styling applied on top
- `recharts` — sparklines and confidence visualization on Signal Desk
- `date-fns` — date handling; ISO-8601 throughout

#### Database: PostgreSQL 16 + PostGIS 3.4

**Decision:** PostgreSQL 16 with PostGIS 3.4.

**Rationale:**
- PostGIS is the industry standard for storing and querying geographic geometries; no alternative competes at this use case
- Geographic entity anchoring (named chokepoints, corridors, ports) requires spatial indexing and spatial joins
- PostgreSQL's JSONB handles the flexible evidence payload schema without schema migration on every signal type addition
- Table partitioning (range on `recorded_at`) supports the raw feed retention cleanup requirement
- Strong asyncio support via asyncpg; Alembic handles migrations including PostGIS geometry columns

**Extensions required:**
- `postgis` — geometry types and spatial functions
- `pg_stat_statements` — query performance monitoring
- `btree_gist` — temporal range overlap indexing for baseline windows

#### Cache and Message Bus: Valkey 7

**Decision:** Valkey 7 (standalone, single-node).

**Rationale:**
- Valkey pub/sub provides the real-time event channel between backend services and the WebSocket server without DB polling
- Valkey hashes store the last-known entity state per vessel/aircraft (reduces DB reads during processor hot path)
- Valkey TTL management handles automatic expiry of stale entity cache entries
- Lightweight enough to run comfortably alongside all other services on a single VPS

**Valkey key namespaces:**
- `entity:adsb:{icao24}` — last normalized ADS-B state per aircraft (TTL: 2h)
- `entity:ais:{mmsi}` — last normalized AIS state per vessel (TTL: 6h)
- `feed:health:{provider}` — current feed health state (TTL: 30s, renewed by ingestor heartbeat)
- `signals:new` — pub/sub channel for newly generated signals
- `watch:status:{watch_id}` — current watch status cache

#### Deployment: Docker Compose

**Decision:** Docker Compose v2 with named volumes for all persistent state.

**Rationale:**
- Reproducible environment on Contabo VPS without environment-specific configuration
- Named volumes ensure Postgres data, Valkey data, and any operator work product survive container restarts
- Service-level isolation: a failing ingestor does not restart the processor or API
- Simple to operate: `docker compose up -d` to start, `docker compose logs -f <service>` to inspect

---

## Core Architectural Decisions

### Decision 1: Monorepo with Service-Level Boundaries

**Decision:** Single Git repository, multiple service directories, shared schema package.

**Rationale:** Solo builder benefits from unified repo management, shared type definitions, and simplified CI. Services remain independently restartable via Docker Compose.

**Service boundaries:**

| Service | Language | Responsibility |
|---|---|---|
| `ingestor-opensky` | Python | OpenSky Network ADS-B feed; normalize; write raw to DB; publish to Valkey |
| `ingestor-adsbexchange` | Python | ADS-B Exchange feed; same pipeline |
| `ingestor-ais` | Python | NOAA AIS + additional AIS providers; normalize; write; publish |
| `processor` | Python | Consumes normalized entity state; maintains baselines; detects anomalies; generates signals + evidence artifacts |
| `api` | Python/FastAPI | REST + WebSocket; serves cockpit; subscribes to Valkey signal channel; pushes to WebSocket clients |
| `cockpit` | React/TS | SPA; builds to static; served via `api` or nginx |
| `postgres` | PostgreSQL+PostGIS | All durable state |
| `valkey` | Valkey 7 | Entity state cache; pub/sub event bus |

### Decision 2: Data Flow Architecture

**Core philosophy implemented as pipeline stages:**

```
Feed Provider
    ↓ [raw polling or WebSocket]
Ingestor Service (per-provider)
    ↓ [normalize to entity-state schema]
    → Write raw record to postgres raw_adsb / raw_ais (48h retention)
    → Write normalized entity state to postgres entity_states
    → Update Valkey entity cache
    → Publish to Valkey normalized:adsb / normalized:ais channel
    ↓
Processor Service (subscribes to normalized channels)
    ↓ [per active watch]
    → Retrieve relevant entity states from DB (geographic intersection)
    → Compare against watch baseline (from WatchBaseline record)
    → Update baseline incrementally (rolling statistics)
    → If anomaly detected → create Signal + SignalEvidence records
    → Publish to Valkey signals:new channel
    ↓
API Service (subscribes to signals:new)
    → Push to connected WebSocket clients (Signal Desk live update)
    → Expose REST endpoints for all cockpit data
    ↓
Cockpit SPA
    → Signal Desk Horizon View (WebSocket-driven live panel)
    → Evidence Surface (REST on demand)
    → Spatial View (REST on demand, signal-scoped layers)
    → Watch Configuration (REST CRUD)
```

This enforces **collect → fuse → interpret → emit** as an explicit pipeline stage sequence, not a monolithic processing loop.

### Decision 3: Data Retention Architecture (Four-Tier)

**Tier 1 — Raw Feed Records**
- Tables: `raw_adsb`, `raw_ais`
- Retention: 48 hours (rolling, partitioned by hour)
- Cleanup: automated partition drop job (cron or Postgres pg_cron)
- Purpose: compliance buffer, short-window replay if needed
- Note: exact retention duration subject to provider terms review; 48h is the conservative default

**Tier 2 — Normalized Entity State**
- Table: `entity_states`
- Retention: `max(active_baseline_window_days) + 7 days` (configurable; default 37 days)
- Cleanup: automated sweep on rows older than retention window
- Purpose: baseline computation; anomaly detection lookback
- PostGIS geometry column for position; entity FK for named entity anchoring

**Tier 3 — Signal and Evidence Artifacts**
- Tables: `signals`, `signal_evidence`, `compound_signals`
- Retention: indefinite (until operator-defined policy)
- Evidence records are **append-only** — no update or delete except by explicit operator action
- Provenance hash stored on each evidence record for integrity verification

**Tier 4 — Operator Work Product**
- Tables: `watches`, `watch_baselines`, `dismissals`, `watch_config_history`
- Retention: indefinite; survive all restarts; deleted only by explicit operator action
- Watch configuration changes are versioned — incompatible changes create a new baseline record and mark the old one superseded

### Decision 4: Baseline Architecture

**Baseline approach:** Rolling statistical window per watch condition per signal family.

**What is stored per baseline:**
```
WatchBaseline:
  watch_id, signal_family
  window_start, window_end (the covered time range)
  status: {in_progress, established, stale, reset}
  parameters: JSONB {
    mean, stddev, median,
    p10, p25, p75, p90,
    sample_count, coverage_hours,
    last_updated_at
  }
  established_at (null until sufficient data)
  minimum_sample_threshold (per signal family defaults)
```

**Incremental update:** Processor updates baseline parameters as new entity state records arrive. No full-window recomputation on each cycle. Running statistics (Welford's algorithm for online mean/variance) used for the rolling mean and stddev.

**Baseline establishment gate:** A watch does not generate escalated signals until the baseline has `sample_count >= minimum_sample_threshold` AND `coverage_hours >= minimum_coverage_hours`. Until established, status = `baseline_in_progress` and Signal Desk shows this explicitly (not NOMINAL, not silent).

**Incompatible change rule:** If the operator changes geography scope, baseline window, or signal family for an existing watch, the processor resets the baseline (marks old as `superseded`, creates new `in_progress` record, logs the event). Compatible changes (threshold only) preserve existing baseline.

### Decision 5: Confidence Architecture

**Single-signal confidence score (0–100):**

```
confidence = (
  deviation_component   # how far from baseline (z-score capped, normalized)
  * source_completeness  # 1.0 = feed fully available; degrades toward 0.5 for partial
  * baseline_maturity    # 0.5 at minimum threshold → 1.0 at 2x threshold
  * recency_factor       # 1.0 if within last polling cycle; decays for stale data
) * 100
```

All components are stored in `confidence_breakdown: JSONB` on each Signal record so the operator can inspect why confidence is what it is.

**Compound signal confidence:**

```
compound_confidence = (
  harmonic_mean(constituent_confidences)  # harmonic mean penalizes weak constituents
  * convergence_factor  # temporal + spatial coincidence score (0–1)
  * source_independence_factor  # ADS-B + AIS = 1.0; same-source = 0.6
  * min(constituent_source_completeness)  # worst-feed floor applied
) * 100
```

**Source independence:** ADS-B and AIS are structurally independent feed types. When both contribute to a compound signal, `source_independence_factor = 1.0`. If both contributors are from ADS-B only, `source_independence_factor = 0.75` (some correlation assumed). This is a named config parameter, not a magic number.

**Degradation rule (FR22, NFR18):** If any contributing feed was in a `degraded` or `stale` state at detection time, this is captured in `source_completeness` in the Signal record AND flagged in the UI. The compound confidence is lowered accordingly. The Signal Desk will never display a compound signal as fully-sourced when a feed was partially unavailable.

**Signal Desk thresholds:**

| Range | Zone | Default behavior |
|---|---|---|
| ≥ operator's escalation threshold (default 65) | Attention Required | Escalated to top zone |
| 40–64 | Monitoring | Sub-threshold, visible in Monitoring zone |
| < 40 | Sub-threshold | Suppressed (not shown unless operator enables) |

Operator configures escalation threshold per watch. Signal Desk zones reflect this.

### Decision 6: Watch Model and Status Machine

**Watch status FSM:**

```
CREATED → BASELINE_IN_PROGRESS → ACTIVE → DEGRADED → ACTIVE
                                         ↘ INACTIVE (operator disabled)
         ← ← ← ← ← RESET ← ← ← ← ← ← (incompatible config change)
```

- `BASELINE_IN_PROGRESS`: Data collecting; minimum threshold not yet met; no signals emitted
- `ACTIVE`: Baseline established; signals being evaluated and emitted
- `DEGRADED`: One or more feeds for this watch type are partially unavailable; signals continue with degraded confidence
- `INACTIVE`: Operator-disabled; no processing

**Watch-to-geography-entity binding:** Each watch binds to exactly one `geography_entity`. The entity provides the spatial boundary for all spatial intersection queries. Multiple watches can target the same entity (e.g., two different watches on Strait of Hormuz with different signal families or thresholds).

### Decision 7: Signal Model

```
Signal:
  id: UUID (PK)
  signal_family: enum (PORT_DWELL_ANOMALY | CARGO_ROUTE_DEVIATION |
                        CHOKEPOINT_THROUGHPUT_CHANGE | VESSEL_TRAFFIC_DENSITY |
                        COMPOUND_DISRUPTION | COMMERCIAL_MASS_DEVIATION |
                        AIS_DARK_EVENT | GPS_JAMMING_EFFECT)
  watch_id: FK → watches
  geography_entity_id: FK → geography_entities
  status: enum (active | dismissed | expired | superseded)
  confidence: float (0–100)
  confidence_breakdown: JSONB
  source_completeness: JSONB  (per-feed availability at detection time)
  detected_at: timestamptz
  first_seen_at: timestamptz
  last_confirmed_at: timestamptz
  dismissed_at: timestamptz | NULL
  dismiss_reason: text | NULL
  dismissed_by: text (always "operator" in v1; extensibility for future)

SignalEvidence:
  id: UUID (PK)
  signal_id: FK → signals
  evidence_type: enum (entity_observation | feed_state_snapshot |
                        baseline_snapshot | constituent_signal_ref)
  source_feed: text  (e.g., "opensky", "ais_noaa")
  observed_at: timestamptz
  geometry: geometry(Point, 4326) | NULL  (entity position if applicable)
  payload: JSONB  (entity readings, deviation values, baseline params at detection, etc.)
  provenance_hash: text  (SHA-256 of payload, for evidence integrity)
  -- append-only: no UPDATE/DELETE in application layer

CompoundSignalLink:
  compound_signal_id: FK → signals (where signal_family = COMPOUND_DISRUPTION)
  constituent_signal_id: FK → signals
  convergence_weight: float  (this constituent's contribution to compound confidence)
```

### Decision 8: Geography Entity Model

```
GeographyEntity:
  id: UUID (PK)
  name: text  (e.g., "Strait of Hormuz", "Suez Canal", "Port of Jebel Ali")
  entity_type: enum (chokepoint | corridor | port | anchorage_zone | region)
  geometry: geometry(Geometry, 4326)  (polygon for areas, linestring for corridors)
  bounding_box: geometry(Polygon, 4326)  (spatial index optimization)
  source: enum (seeded | operator_defined)
  metadata: JSONB  (aliases, country_codes, description, UN/LOCODE, etc.)
  tags: text[]
  is_active: boolean

  -- Extensibility hooks (v1: always NULL; future layers attach here)
  people_place_context_id: UUID | NULL  → (future: Census/demographic context table)
  economic_context_id: UUID | NULL       → (future: economic baseline table)
  infrastructure_context_id: UUID | NULL → (future: infrastructure intelligence table)

  created_at: timestamptz
  updated_at: timestamptz
```

**v1 seeded entities (minimum set — operator configures additional via Watch Configuration):**
- Strait of Hormuz (chokepoint)
- Strait of Malacca (chokepoint)
- Suez Canal (chokepoint)
- Bab-el-Mandeb (chokepoint)
- Dover Strait (chokepoint)
- Panama Canal (chokepoint)
- Singapore Strait (chokepoint)
- Port of Jebel Ali (port)
- Port of Singapore (port)
- Gulf-Europe Cargo Corridor (corridor)
- Trans-Pacific Cargo Corridor (corridor)

### Decision 9: Provider Terms Compliance Architecture

**Design-time rule:** Provider terms compliance is a first-class architectural constraint. No production-scale ingestion begins before terms are reviewed and architecture confirmed compliant. This is an MVP success criterion per the PRD.

**Implementation approach:**
- Each ingestor has a named `provider_config` including: `rate_limit_rps`, `storage_allowed`, `retention_hours`, `commercial_use_allowed`
- These config values are read at startup; ingestor refuses to start if `storage_allowed = false`
- Raw record retention window is set per-provider from this config, not hardcoded
- Credentials are read from environment variables only — never stored in DB, never logged, never returned in API responses

**Per-provider compliance config (v1 defaults, pending terms review):**

| Provider | Rate Limit | Raw Retention | Notes |
|---|---|---|---|
| OpenSky Network | 5 req/sec (free tier) | 48h | Free tier allows non-commercial use; review for this use case |
| ADS-B Exchange | Per-plan limit | 48h | Commercial terms require explicit review |
| NOAA AIS | No stated limit | 48h | Public data; storage terms to be confirmed |

**Compliance gate:** The `terms_status` field in provider config can be `under_review`, `compliant`, or `restricted`. If `under_review` or `restricted`, the ingestor runs in a **limited mode** (only stores normalized entity state, not raw records) to avoid committing to a storage pattern before terms are clear.

### Decision 10: Authentication

**Decision:** Single-operator session authentication using JWT with a long-lived refresh token stored in an httpOnly cookie.

**Rationale:** No OAuth, no SSO, no user management. One hardcoded operator credential (username + bcrypt-hashed password in environment config). JWT access token (15-minute expiry) + refresh token (30-day expiry, httpOnly cookie). Password and token config loaded from environment at API startup.

This is the minimum-viable authentication for a self-hosted single-operator tool. It protects the cockpit from accidental exposure on an open port without adding identity-management complexity.

### Decision 11: API Design

**Decision:** REST for cockpit data access + WebSocket for real-time Signal Desk push.

**REST API structure (internal only, not externally exposed):**

```
GET  /api/v1/signals                    → Signal Desk list (ranked, paginated)
GET  /api/v1/signals/{id}               → Single signal detail
GET  /api/v1/signals/{id}/evidence      → Evidence records for a signal
GET  /api/v1/signals/{id}/spatial       → Signal-scoped spatial layer data (GeoJSON)
POST /api/v1/signals/{id}/dismiss       → Dismiss with reason
GET  /api/v1/signals/dismissed          → Dismissed signal history

GET  /api/v1/watches                    → All watches with current status
POST /api/v1/watches                    → Create watch
GET  /api/v1/watches/{id}               → Single watch detail
PATCH /api/v1/watches/{id}              → Update watch config
DELETE /api/v1/watches/{id}             → Deactivate watch
GET  /api/v1/watches/{id}/baseline      → Baseline status and parameters

GET  /api/v1/geography/entities         → All named geographic entities
GET  /api/v1/geography/entities/{id}    → Single entity with geometry

GET  /api/v1/system/health              → System + feed health summary
GET  /api/v1/system/session-summary     → Session summary (signals since last login)

POST /api/v1/auth/login                 → Authenticate
POST /api/v1/auth/refresh               → Refresh access token
POST /api/v1/auth/logout                → Invalidate session

WS   /ws/signals                        → Real-time Signal Desk push channel
```

**WebSocket message types (server → client):**
```json
{ "type": "signal:new",    "payload": { ...signal_summary } }
{ "type": "signal:update", "payload": { "id": "...", "status": "...", "confidence": ... } }
{ "type": "feed:health",   "payload": { "provider": "opensky", "status": "degraded", "since": "..." } }
{ "type": "watch:status",  "payload": { "watch_id": "...", "status": "active" } }
{ "type": "ping",          "payload": null }
```

**API response envelope:**
```json
{
  "data": { ... },
  "meta": { "timestamp": "ISO-8601", "request_id": "uuid" },
  "error": null
}
```

Error responses:
```json
{
  "data": null,
  "meta": { "timestamp": "...", "request_id": "..." },
  "error": { "code": "SIGNAL_NOT_FOUND", "message": "Signal not found", "detail": null }
}
```

---

## Implementation Patterns & Consistency Rules

### Naming Patterns

**Database (PostgreSQL):**
- Tables: `snake_case` plural nouns — `signals`, `signal_evidence`, `watches`, `geography_entities`, `entity_states`, `raw_adsb`, `raw_ais`
- Columns: `snake_case` — `geography_entity_id`, `detected_at`, `confidence_breakdown`
- Foreign keys: `{referenced_table_singular}_id` — `signal_id`, `watch_id`, `geography_entity_id`
- Indexes: `idx_{table}_{columns}` — `idx_signals_watch_id`, `idx_entity_states_entity_id_observed_at`
- Enums: `snake_case` values — `baseline_in_progress`, `attention_required`, `port_dwell_anomaly`

**Python backend:**
- Files: `snake_case` — `signal_processor.py`, `watch_manager.py`, `adsb_ingestor.py`
- Classes: `PascalCase` — `SignalProcessor`, `WatchManager`, `WatchBaseline`
- Functions/methods: `snake_case` — `detect_dwell_anomaly()`, `update_baseline_incremental()`
- Async functions: always `async def` for DB and Valkey operations; no blocking calls in async context
- Constants: `UPPER_SNAKE_CASE` — `DEFAULT_BASELINE_WINDOW_DAYS = 30`
- Pydantic models (schema): `PascalCase` — `SignalCreate`, `WatchResponse`, `EvidenceRecord`
- SQLAlchemy models: `PascalCase` — `Signal`, `Watch`, `GeographyEntity`

**TypeScript/React frontend:**
- Files: `PascalCase` for components (`SignalDesk.tsx`, `EvidenceSurface.tsx`); `camelCase` for utils/hooks (`useSignals.ts`, `formatConfidence.ts`)
- Components: `PascalCase` — `<SignalCard />`, `<DomainPulse />`, `<WatchStatusBadge />`
- Hooks: `use` prefix — `useSignalDesk()`, `useWatchConfig()`, `useFeedHealth()`
- API response types: mirror Pydantic schema names — `Signal`, `Watch`, `GeographyEntity`
- Zustand stores: `use{Name}Store` — `useSignalStore()`, `useSessionStore()`
- Constants: `UPPER_SNAKE_CASE` in `constants.ts`

**API endpoints:**
- All endpoints prefixed `/api/v1/`
- Resource names: plural nouns — `/signals`, `/watches`, `/geography/entities`
- No trailing slashes
- Query params: `snake_case` — `?signal_family=port_dwell_anomaly&min_confidence=65`

### Structure Patterns

**Error handling — backend:**
```python
# All service errors raise typed exceptions
class PanoptesError(Exception): ...
class SignalNotFoundError(PanoptesError): ...
class BaselineNotEstablishedError(PanoptesError): ...
class FeedUnavailableError(PanoptesError): ...

# FastAPI exception handlers map to standard error envelope
@app.exception_handler(SignalNotFoundError)
async def signal_not_found_handler(request, exc):
    return JSONResponse(status_code=404, content={"data": None, "error": {"code": "SIGNAL_NOT_FOUND", ...}})
```

**Error handling — frontend:**
- React Query handles API error state; no try/catch in components
- Error states displayed in-place (not full-page error screens) — cockpit remains navigable
- Feed health errors surfaced via Domain Pulse, not modal interrupts

**Async patterns — Python:**
- All DB operations use `async with session:` pattern (SQLAlchemy async sessions)
- All Valkey operations are async (`await redis.publish(...)`) — `redis.asyncio` client is wire-compatible with Valkey 7
- Ingestors use `asyncio.gather()` for concurrent polling where applicable
- Never use `time.sleep()` in async context; always `await asyncio.sleep()`

**State management — frontend:**
- Server state (signals, watches, feed health): React Query with appropriate `staleTime` and `refetchInterval`
- UI state (selected signal, active surface, session timestamps): Zustand
- Never store server data in Zustand; never store UI state in React Query

**Date/time:**
- All timestamps stored as `timestamptz` in PostgreSQL (UTC)
- All API timestamps are ISO-8601 strings in UTC (`2026-03-08T14:32:00Z`)
- Frontend renders timestamps in operator's local timezone for display, but stores/compares in UTC
- Never use `datetime.now()` in Python — always `datetime.now(UTC)`

**Evidence integrity:**
- No `UPDATE` or `DELETE` on `signal_evidence` records in application layer
- SQLAlchemy model has no `update()` method defined — enforced at ORM layer
- Provenance hash computed on `payload` dict before serialization; stored alongside
- Hash algorithm: SHA-256, hex-encoded

**NOMINAL state:**
- A watch is NOMINAL when: `status = active` AND no `signals` with `status = active` AND baseline established
- NOMINAL is an explicit computed state, not an absence of data
- Signal Desk displays NOMINAL per-watch with last-checked timestamp — never an empty cell

**Feed health:**
- Each ingestor publishes a heartbeat to `feed:health:{provider}` every 15 seconds (Valkey key with 30s TTL)
- If TTL expires without renewal, API reports feed as `stale` for that provider
- Processor explicitly checks `source_completeness` per provider before computing confidence
- No feed state is silently assumed healthy without a recent heartbeat

### Format Patterns

**Confidence score:** Always `float` in range `[0.0, 100.0]`. Never `null`. If computation is impossible (no baseline), the signal is not emitted — confidence is always a valid number when a signal exists.

**Geometry on API responses:** GeoJSON geometry objects (`{"type": "Point", "coordinates": [lon, lat]}`). Always WGS84 (EPSG:4326). Never raw WKT or WKB in API responses.

**Signal summary (for Signal Desk list):**
```json
{
  "id": "uuid",
  "signal_family": "port_dwell_anomaly",
  "signal_family_label": "Port Dwell-Time Anomaly",
  "geography_entity": { "id": "uuid", "name": "Strait of Hormuz", "entity_type": "chokepoint" },
  "confidence": 71.4,
  "status": "active",
  "detected_at": "2026-03-08T03:14:00Z",
  "age_seconds": 4320,
  "is_compound": false,
  "source_completeness": { "opensky": 1.0, "ais_noaa": 0.85 },
  "watch_id": "uuid"
}
```

**Pagination:** All list endpoints return `{ "data": [...], "meta": { "total": N, "page": 1, "page_size": 50 } }`. Default page size: 50. Max: 200.

---

## Project Structure & Boundaries

### Complete Project Directory Structure

```
panoptes/
├── README.md
├── .gitignore
├── .env.example                    # Template for all environment variables
├── docker-compose.yml              # Production compose file
├── docker-compose.dev.yml          # Development overrides
│
├── infra/
│   ├── postgres/
│   │   └── init.sql                # PostGIS extension enable; initial seed data
│   ├── valkey/
│   │   └── valkey.conf
│   └── nginx/
│       └── nginx.conf              # Cockpit static serving + API proxy
│
├── shared/
│   └── schema/
│       ├── __init__.py
│       ├── signals.py              # Shared Pydantic models (Signal, SignalEvidence, etc.)
│       ├── watches.py              # Watch, WatchBaseline models
│       ├── geography.py            # GeographyEntity models
│       ├── feeds.py                # FeedHealth, RawAdsb, RawAis models
│       └── enums.py                # SignalFamily, WatchStatus, EntityType enums
│
├── migrations/
│   ├── env.py                      # Alembic configuration
│   ├── script.py.mako
│   └── versions/
│       ├── 001_initial_schema.py
│       ├── 002_add_postextensibility_hooks.py
│       └── ...
│
├── services/
│
│   ├── ingestor/
│   │   ├── Dockerfile
│   │   ├── pyproject.toml
│   │   ├── ingestor/
│   │   │   ├── __init__.py
│   │   │   ├── main.py             # Entry point; reads PROVIDER env var; loads correct adapter
│   │   │   ├── base.py             # BaseIngestor abstract class
│   │   │   ├── normalizer.py       # Raw → EntityState normalization (shared logic)
│   │   │   ├── adapters/
│   │   │   │   ├── opensky.py      # OpenSky Network API adapter
│   │   │   │   ├── adsbexchange.py # ADS-B Exchange adapter
│   │   │   │   └── ais_noaa.py     # NOAA AIS adapter
│   │   │   ├── compliance.py       # Provider terms enforcement (rate limits, storage rules)
│   │   │   └── health.py           # Feed heartbeat publisher
│   │   └── tests/
│   │       ├── test_normalizer.py
│   │       └── test_adapters.py
│   │
│   ├── processor/
│   │   ├── Dockerfile
│   │   ├── pyproject.toml
│   │   ├── processor/
│   │   │   ├── __init__.py
│   │   │   ├── main.py             # Entry point; event loop; subscribes to Valkey channels
│   │   │   ├── watch_manager.py    # Watch lifecycle; baseline FSM
│   │   │   ├── baseline.py         # Rolling baseline computation (Welford's algorithm)
│   │   │   ├── confidence.py       # Confidence scoring (single + compound)
│   │   │   ├── signals/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── base.py         # BaseSignalDetector abstract class
│   │   │   │   ├── port_dwell.py   # Port Dwell-Time Anomaly detector
│   │   │   │   ├── route_deviation.py  # Cargo Route Deviation detector
│   │   │   │   ├── chokepoint_throughput.py
│   │   │   │   ├── vessel_density.py
│   │   │   │   ├── compound.py     # Compound Disruption Pattern assembler
│   │   │   │   ├── dark_event.py   # AIS vessel dark event detector
│   │   │   │   └── gps_jamming.py  # GPS jamming incoherence detector
│   │   │   ├── spatial.py          # GeoPandas/Shapely spatial intersection helpers
│   │   │   └── evidence.py         # Evidence artifact builder; provenance hash
│   │   └── tests/
│   │       ├── test_baseline.py
│   │       ├── test_confidence.py
│   │       └── signals/
│   │           ├── test_port_dwell.py
│   │           └── ...
│   │
│   ├── api/
│   │   ├── Dockerfile
│   │   ├── pyproject.toml
│   │   ├── api/
│   │   │   ├── __init__.py
│   │   │   ├── main.py             # FastAPI app; lifespan; router registration
│   │   │   ├── config.py           # Pydantic Settings; env var loading
│   │   │   ├── database.py         # Async SQLAlchemy engine + session factory
│   │   │   ├── valkey_client.py    # Valkey connection; pub/sub subscriber
│   │   │   ├── auth.py             # JWT logic; single-operator credential check
│   │   │   ├── dependencies.py     # FastAPI Depends() factories (auth, db, valkey)
│   │   │   ├── routers/
│   │   │   │   ├── signals.py      # /api/v1/signals routes
│   │   │   │   ├── watches.py      # /api/v1/watches routes
│   │   │   │   ├── geography.py    # /api/v1/geography routes
│   │   │   │   ├── system.py       # /api/v1/system/health, session-summary
│   │   │   │   └── auth.py         # /api/v1/auth routes
│   │   │   ├── websocket.py        # /ws/signals WebSocket handler; Valkey → WS bridge
│   │   │   ├── models/             # SQLAlchemy ORM models
│   │   │   │   ├── signal.py
│   │   │   │   ├── watch.py
│   │   │   │   ├── geography.py
│   │   │   │   └── entity_state.py
│   │   │   └── exceptions.py       # Custom exception classes + handlers
│   │   └── tests/
│   │       ├── test_auth.py
│   │       ├── test_signals.py
│   │       └── test_websocket.py
│   │
│   └── cockpit/
│       ├── Dockerfile              # Build stage only; output served by nginx
│       ├── package.json
│       ├── vite.config.ts
│       ├── tsconfig.json
│       ├── tailwind.config.ts
│       ├── .env.example
│       ├── public/
│       └── src/
│           ├── main.tsx            # React entry point
│           ├── App.tsx             # Router + auth guard
│           ├── constants.ts        # SIGNAL_FAMILIES, ENTITY_TYPES, thresholds
│           ├── types/              # TypeScript interfaces (mirrors Pydantic schemas)
│           │   ├── signal.ts
│           │   ├── watch.ts
│           │   ├── geography.ts
│           │   └── feed.ts
│           ├── api/                # API client layer
│           │   ├── client.ts       # Axios instance; auth token injection; error handling
│           │   ├── signals.ts      # Signal API functions
│           │   ├── watches.ts
│           │   ├── geography.ts
│           │   └── system.ts
│           ├── stores/             # Zustand stores (UI state only)
│           │   ├── sessionStore.ts # Last-seen timestamp; active surface
│           │   └── selectionStore.ts # Currently selected signal, watch
│           ├── hooks/              # React Query hooks (server state)
│           │   ├── useSignals.ts
│           │   ├── useWatches.ts
│           │   ├── useFeedHealth.ts
│           │   └── useSessionSummary.ts
│           ├── components/
│           │   ├── ui/             # shadcn/ui primitives (auto-generated)
│           │   ├── layout/
│           │   │   ├── CockpitShell.tsx      # Permanent outer shell (nav, auth)
│           │   │   └── SurfaceContainer.tsx  # Active surface wrapper
│           │   ├── signal-desk/
│           │   │   ├── SignalDesk.tsx         # Primary surface
│           │   │   ├── HorizonView.tsx        # 3-zone layout (Attention/Monitoring/Pulse)
│           │   │   ├── AttentionRequired.tsx
│           │   │   ├── MonitoringZone.tsx
│           │   │   ├── DomainPulse.tsx        # Feed health status panel
│           │   │   ├── SessionSummary.tsx
│           │   │   ├── SignalCard.tsx         # Individual signal in list
│           │   │   └── NominalBadge.tsx       # Explicit NOMINAL display
│           │   ├── evidence-surface/
│           │   │   ├── EvidenceSurface.tsx    # Slide-over or panel
│           │   │   ├── ConfidenceBreakdown.tsx
│           │   │   ├── EvidenceLayer.tsx      # One contributing evidence record
│           │   │   ├── CompoundConstituents.tsx
│           │   │   ├── ProvenanceChain.tsx
│           │   │   └── DismissPanel.tsx
│           │   ├── spatial-view/
│           │   │   ├── SpatialView.tsx        # MapLibre GL container
│           │   │   ├── SignalLayers.tsx        # Signal-scoped GeoJSON layers
│           │   │   └── EntityBoundary.tsx     # Named geographic entity geometry
│           │   └── watch-config/
│           │       ├── WatchConfiguration.tsx # Surface
│           │       ├── WatchList.tsx
│           │       ├── WatchEditor.tsx        # Create/edit form
│           │       ├── GeographyPicker.tsx    # Named entity selector
│           │       ├── SignalFamilyToggles.tsx
│           │       ├── ThresholdSlider.tsx
│           │       └── BaselineStatus.tsx
│           └── lib/
│               ├── confidence.ts   # Confidence formatting, tier classification
│               ├── dates.ts        # UTC/local conversion, relative time display
│               ├── geojson.ts      # GeoJSON utilities for MapLibre
│               └── ws.ts           # WebSocket connection manager (reconnect logic)
```

### Architectural Boundaries

**Ingestor → Database boundary:**
- Ingestors write to `raw_adsb`/`raw_ais` (Tier 1) and `entity_states` (Tier 2)
- Ingestors never read from `signals`, `watches`, or `geography_entities`
- Ingestors publish normalized events to Valkey but do not consume Valkey

**Processor → Database boundary:**
- Processor reads `entity_states`, `watches`, `watch_baselines`, `geography_entities`
- Processor writes `signals`, `signal_evidence`, `compound_signal_links`, updates `watch_baselines`
- Processor reads from Valkey entity cache; subscribes to normalized event channels
- Processor publishes to `signals:new` Valkey channel

**API → Database boundary:**
- API reads all tables for serving cockpit requests
- API writes only: `signals` (status + dismiss fields), `watches` (CRUD), `dismissals`
- API never writes to `signal_evidence` (append-only, processor-only write path)
- API subscribes to `signals:new` Valkey channel for WebSocket push

**Cockpit → API boundary:**
- Cockpit has no direct DB or Valkey access
- All data via REST or WebSocket
- No polling of individual signal endpoints for live updates — WebSocket push only

**Data flow integrity rule:** Signals and evidence are created only by the Processor. The API can change signal `status` (dismiss) but cannot create or modify evidence records. This boundary enforces evidence integrity.

### Integration Points

**Internal communication:**

| From | To | Mechanism | Data |
|---|---|---|---|
| Ingestor | PostgreSQL | asyncpg/SQLAlchemy | Raw records, entity states |
| Ingestor | Valkey | redis.asyncio pub | Normalized entity events, heartbeats |
| Processor | Valkey | redis.asyncio sub | Normalized entity events |
| Processor | PostgreSQL | asyncpg/SQLAlchemy | Baseline reads/writes, signal writes |
| Processor | Valkey | redis.asyncio pub | New signal events |
| API | PostgreSQL | asyncpg/SQLAlchemy | All reads, signal dismiss writes |
| API | Valkey | redis.asyncio sub | New signal events for WS push |
| Cockpit | API | HTTP REST | Data requests |
| Cockpit | API | WebSocket | Live Signal Desk updates |

**External integrations:**

| Provider | Protocol | Auth | Notes |
|---|---|---|---|
| OpenSky Network | REST polling | API key (env) | 5 req/sec free tier limit |
| ADS-B Exchange | REST or WebSocket | API key (env) | Terms review required |
| NOAA AIS | REST or bulk download | None (public) | Confirm storage terms |

**Data flow (end-to-end for a compound signal):**

```
1. Ingestor-OpenSky polls OpenSky API → receives aircraft state vectors
2. Ingestor normalizes → writes EntityState records (Tier 2) to DB
3. Ingestor publishes normalized:adsb:batch event to Valkey

4. Ingestor-AIS polls NOAA → receives vessel reports
5. Ingestor normalizes → writes EntityState records to DB
6. Ingestor publishes normalized:ais:batch event to Valkey

7. Processor subscribes → receives both batch events
8. Processor loads active watches with geography intersection
9. For Strait of Hormuz watch:
   a. Route Deviation detector: queries entity_states for cargo flights
      → deviation from corridor baseline detected (confidence 73)
      → writes Signal (CARGO_ROUTE_DEVIATION) + SignalEvidence records
   b. Chokepoint Throughput detector: queries entity_states for vessels
      → throughput below seasonal baseline (confidence 68)
      → writes Signal (CHOKEPOINT_THROUGHPUT_CHANGE) + SignalEvidence records
   c. Compound assembler: sees two constituent signals on same geography/window
      → computes compound confidence (78 = harmonic_mean(73,68) * convergence * independence)
      → writes Signal (COMPOUND_DISRUPTION) + CompoundSignalLink records

10. Processor publishes {type: "signal:new", signal_id: "...", confidence: 78} to Valkey

11. API WebSocket handler receives Valkey message
12. API pushes { type: "signal:new", payload: {...signal_summary} } to connected client

13. Cockpit React: WebSocket event received → React Query invalidates signals cache
14. Signal Desk re-renders → COMPOUND_DISRUPTION appears in Attention Required zone
    with confidence 78, geography "Strait of Hormuz", constituents listed

15. Operator clicks → Evidence Surface opens
16. API GET /signals/{id}/evidence → returns all SignalEvidence records
17. Operator sees: constituent signals, individual evidence layers, confidence breakdown,
    feed state at detection time, provenance hashes

18. Operator clicks "View Spatially" → Spatial View opens
19. API GET /signals/{id}/spatial → returns GeoJSON (vessel positions, corridor geometry,
    flight tracks, Strait geometry)
20. MapLibre GL renders signal-scoped layers only
```

---

## Architecture Validation

### Coherence Validation

**Decision Compatibility:**
- Python FastAPI + asyncio + asyncpg + Valkey-compatible async client (redis.asyncio) are all asyncio-native — no event loop conflicts
- PostGIS geometry columns handled by GeoAlchemy2 with native SQLAlchemy 2 async support
- React Query + Zustand are designed to be used together; no state management conflicts
- MapLibre GL JS is standalone; no React integration conflict (render to a div ref)
- All date handling is UTC-first in both Python (datetime with tzinfo=UTC) and TypeScript (date-fns)

**Pattern Consistency:**
- Evidence append-only pattern enforced at both ORM model layer (no update method) and service boundary (Processor writes, API cannot write evidence)
- NOMINAL is a computed property derived from Watch status + Signal status query — no inconsistency between services
- Feed health via Valkey TTL is consistent: ingestor owns publishing, API owns consuming, cockpit owns displaying

**Structure Alignment:**
- All signal detection logic lives in `processor/signals/` — no detection logic in API or cockpit
- All geospatial computation lives in `processor/spatial.py` — no raw geometry computation in API routes
- Cockpit has no business logic — only presentation and user interaction

### Requirements Coverage Validation

**Functional Requirements — all 45 covered:**

| FR Category | Coverage |
|---|---|
| FR1–FR4 (Feed Ingestion) | Ingestor services with per-provider adapters, normalization, geographic entity anchoring on entity_states |
| FR5–FR10 (Watch Management) | `watches` + `watch_baselines` schema; Watch Configuration surface; baseline FSM; incompatible change → reset |
| FR11–FR22 (Intelligence Processing) | Processor service; 7 signal detectors + compound assembler; confidence + degradation architecture; absence detection in dark_event.py and route_deviation.py |
| FR23–FR29 (Signal Desk) | SignalDesk surface; WebSocket push; HorizonView 3-zone; NOMINAL explicit state; DomainPulse; SessionSummary |
| FR30–FR35 (Evidence + Spatial) | EvidenceSurface; full provenance chain; compound constituents display; SpatialView triggered from evidence, signal-scoped layers only |
| FR36–FR38 (Operator Workflow) | DismissPanel; single-operator JWT auth |
| FR39–FR45 (System Health + Data Model) | No-silent-failure via Valkey TTL + heartbeat; 4-tier retention via partitioned tables + cleanup; geographic entity FK on all signal records; append-only evidence; browser-only cockpit |

**Non-Functional Requirements — all 21 covered:**

| NFR | Coverage |
|---|---|
| NFR1–NFR5 (Performance) | Incremental baseline (Welford's); asyncio non-blocking ingestion; Valkey pub/sub for near-real-time; indexed spatial queries via PostGIS GIST |
| NFR6–NFR9 (Reliability) | Named Docker volumes; service-level restart isolation in Compose; automatic Valkey reconnect; heartbeat-based staleness detection |
| NFR10–NFR14 (Security) | JWT auth on all API routes; all data in named Postgres volumes; no external transmission; credentials in env vars; local-network-only by default |
| NFR15–NFR18 (Data Integrity) | Append-only evidence (ORM + service boundary); provenance hash; Postgres transactions for concurrent baseline writes; explicit baseline reset on incompatible changes |
| NFR19–NFR21 (Operability) | Docker Compose declarative deployment; health via cockpit Domain Pulse; container-level fault isolation |

**Geographic entity anchoring requirement (PRD explicit constraint):** Every `Signal`, `SignalEvidence`, and `Watch` has a `geography_entity_id` FK. The `GeographyEntity` schema has nullable `people_place_context_id` and `economic_context_id` FKs as structural extensibility hooks. This satisfies both the v1 requirement and the platform hook requirement.

### Gap Analysis

**No critical gaps identified.** Architecture covers all 45 FRs and 21 NFRs.

**Implementation considerations to surface early:**
- Provider terms review must happen before production ingestion. The compliance gate (`terms_status` field) structurally enforces this but the review itself is a human-process dependency outside the codebase.
- Baseline establishment period (default 30 days) means signals won't fire meaningfully for 30 days after initial setup. This is expected behavior, but the operator experience during this window must be carefully implemented (explicit NOMINAL / baseline-in-progress states, not silence).
- MapLibre GL JS requires a map tile source for the Spatial View base layer. Recommend: OSM tiles via a self-hosted [MapTiler] or public tile source for v1. Not a blocking gap but must be decided before Spatial View is implemented.

**Decisions pushed to UX Design (not architecture):**

- Exact visual styling of NOMINAL state (color, icon, text) — architecture only requires it be explicit and distinct from silence
- Layout proportions of the 3-zone Horizon View — architecture only requires the three zones exist
- Confidence score display format (number, color band, label) — architecture delivers the score; UX decides presentation
- Dismiss workflow UX (modal, inline form, slide-over) — architecture delivers the endpoint; UX decides the interaction
- Evidence Surface layout (how many layers visible at once, scroll behavior) — architecture delivers all evidence records; UX decides display
- Empty state design for new installations (the "first run" experience beyond NOMINAL state communication)
- Signal aging and staleness visual treatment on Signal Desk
- Watch Configuration geography picker interaction (map-based selection vs dropdown — this is a UX decision; architecture seeds the entity list)

### Architecture Readiness Assessment

**Overall Status: READY FOR IMPLEMENTATION**

**Confidence Level: High**

**Key Strengths:**
- Signal-first philosophy is enforced in the architecture, not just as a UI preference: the data pipeline ensures signals are the primary output type, not raw data display
- Compound signal confidence accounting for source degradation satisfies the hardest integrity requirement in the PRD
- The four-tier retention model with structurally distinct tables eliminates ambiguity in cleanup policies
- Geographic entity anchoring with nullable extensibility FKs satisfies both v1 constraint and platform hook requirement without broadening v1 scope
- Valkey heartbeat-based feed health with TTL expiry means staleness detection is passive — no polling required from the API
- Solo-builder appropriate: Python monolanguage backend, Docker Compose deployment, no managed cloud dependencies

**Platform Extensibility Hooks Preserved:**
- `geography_entities.people_place_context_id` — Census/demographic context attachment point
- `geography_entities.economic_context_id` — economic baseline attachment point
- `geography_entities.infrastructure_context_id` — infrastructure context attachment point
- `signal_evidence.evidence_type` enum is extensible to new evidence types without schema change
- `SignalFamily` enum is extensible — new signal families require a new detector class, no schema migration
- Provider adapter pattern in ingestor means new data sources add an adapter file, not a new service architecture
- `WatchBaseline.parameters` is JSONB — baseline parameter schema evolves without migrations
- Watch `enabled_signal_families` is a list — future signal families are opt-in per watch, no schema changes

**What is deliberately NOT hooked (v1 boundary maintained):**
- No Census API client code
- No demographic data tables
- No multi-tenant schema (no `operator_id` FKs)
- No API output endpoints for external consumers
- No collaboration features
- No Timeline View data structures (evidence exists but temporal reconstruction tooling is v2)

---

## Recommended Technical Architecture Summary

### Stack at a Glance

| Layer | Technology | Version |
|---|---|---|
| Backend language | Python | 3.12 |
| API framework | FastAPI + Uvicorn | FastAPI 0.115+ |
| ORM | SQLAlchemy (async) + Alembic | 2.x |
| Database | PostgreSQL + PostGIS | 16 + 3.4 |
| Cache / Event bus | Valkey | 7.x |
| Geospatial (Python) | Shapely + GeoAlchemy2 | Shapely 2.x |
| Statistics (Python) | NumPy + Pandas | Latest stable |
| HTTP client (Python) | httpx | 0.27+ |
| Frontend framework | React + TypeScript | 18 + 5 |
| Frontend build | Vite | 5.x |
| Frontend state (server) | React Query (@tanstack/query) | 5.x |
| Frontend state (UI) | Zustand | 4.x |
| Map rendering | MapLibre GL JS | 4.x |
| CSS | Tailwind CSS | 3.x |
| Component primitives | shadcn/ui | Latest |
| Containerization | Docker Compose v2 | Latest stable |

### Key Architectural Decisions — Summary

| Decision | Choice | Rationale |
|---|---|---|
| Backend language | Python (all services) | Best fit for stats, geospatial, async ingestion; single language for solo builder |
| Service topology | 3 ingestors + 1 processor + 1 API + 1 cockpit | Isolated per-provider ingestion; clear stage separation; manageable for solo builder |
| Data pipeline | Staged (ingest → normalize → process → emit) | Enforces collect→fuse→interpret→emit philosophy in code, not just in words |
| Primary store | PostgreSQL + PostGIS | Named entity anchoring + spatial queries require PostGIS; no alternative |
| Event bus | Valkey pub/sub | Lightweight; enables real-time push without DB polling; entity state caching |
| Real-time push | WebSocket (FastAPI native) | Signal Desk live updates without client polling |
| Baseline computation | Welford's online algorithm | Incremental update; no full-window recompute per cycle; O(1) per observation |
| Confidence model | Multi-factor (deviation × source × maturity × recency) | Accounts for baseline immaturity, feed degradation, and temporal decay |
| Compound confidence | Harmonic mean × convergence × independence × degradation floor | Penalizes weak constituents; rewards source independence; accurately reflects degraded-source state |
| Evidence integrity | Append-only at ORM + service boundary levels | Enforces provenance chain without a separate immutable log system |
| Retention | 4-tier with distinct tables and policies | Structurally distinct per PRD requirement; clear cleanup automation per tier |
| Authentication | JWT + bcrypt single-operator | Minimum viable auth; no complexity overhead for single-user deployment |
| API design | REST + WebSocket | REST for all cockpit data; WebSocket for Signal Desk live push only |
| Deployment | Docker Compose, named volumes | Reproducible; restart-safe; service-level fault isolation; appropriate for Contabo VPS |
| Spatial View | MapLibre GL JS, signal-scoped only | Open source; WebGL performance; triggered from evidence, not default home |

### Major Tradeoffs and Risks

| Tradeoff / Risk | Description | Mitigation |
|---|---|---|
| **Python GIL in processor** | CPU-intensive baseline computation could block asyncio event loop | Run computationally heavy stats in a thread pool executor; Welford's algorithm is lightweight per observation |
| **30-day baseline establishment period** | Operator sees no meaningful signals for ~30 days after new watch creation | Explicit NOMINAL / baseline-in-progress UX states; clear expectation-setting; system is not "empty", it is "collecting" |
| **Provider terms risk** | ADS-B Exchange or other providers may restrict commercial or storage use | Compliance gate prevents production ingestion before terms review; conservative 48h raw retention default; terms_status config field |
| **Single-server resource contention** | All services on one Contabo VPS; processor CPU spike could affect API responsiveness | Separate containers with resource limits in Compose; processor and API do not share event loop |
| **Compound signal false positives** | Two weak signals in the same geography could fire a compound with misleading confidence | Harmonic mean penalizes weak constituents; convergence factor requires temporal + spatial coincidence; operator dismiss-with-reason feeds learning |
| **MapLibre tile source dependency** | Spatial View requires a map tile source; self-hosted tile server adds complexity | Use public OSM tile endpoints for v1; document as known dependency; tile server is v1.5+ concern |
| **Baseline state corruption on concurrent writes** | Multiple ingestion events arriving simultaneously could race on baseline parameter update | SQLAlchemy async sessions with `SELECT FOR UPDATE` on WatchBaseline rows during update; single processor service eliminates multi-process race |
| **Valkey as single point of failure** | If Valkey goes down, real-time push stops and feed health becomes invisible | API falls back to DB polling for feed health if Valkey unreachable; Docker Compose restart policy ensures quick recovery; named volume persists no critical durable state in Valkey |

### What Must Stay Flexible for Future Platform Expansion

1. **GeographyEntity extensibility FKs** — `people_place_context_id`, `economic_context_id`, `infrastructure_context_id` must remain nullable and unused in v1. Do not add placeholder data to these columns.
2. **SignalFamily enum** — must be extensible in code. New signal families add a detector class and an enum value; no architectural change required.
3. **Provider adapter pattern** — ingestor adapter interface must remain stable. New providers (satellite, weather, etc.) implement the same `BaseIngestor` interface.
4. **Watch model's `enabled_signal_families` list** — new signal families are addable to existing watches without schema migration.
5. **Evidence payload as JSONB** — different domains will produce different evidence structures. JSONB flexibility must be preserved; do not normalize the evidence payload into columns.
6. **`dismissed_by` field on Signal** — currently always "operator" in v1. In v2+ (multi-operator or automated suppression), this field carries meaning without schema change.
7. **API versioning prefix `/api/v1/`** — maintain from day one so v2 endpoints can be added alongside without breaking cockpit.
8. **CockpitShell.tsx surface registration** — the cockpit surface navigation must be driven by a surface registry pattern, not hardcoded if-else chains. New surfaces (Timeline View, Briefing View) must be registerable without modifying the shell.

---

## Implementation Handoff

### First Implementation Story

The first implementation action is environment and infrastructure scaffolding:
1. Initialize monorepo directory structure per project tree above
2. Create `docker-compose.yml` with all services defined (Postgres + PostGIS, Valkey, placeholder containers for services)
3. Create `.env.example` with all required environment variables documented
4. Create `migrations/` with Alembic setup and first migration (initial schema: `geography_entities`, `watches`, `watch_baselines`, `signals`, `signal_evidence`, `compound_signal_links`, `entity_states`, `raw_adsb`, `raw_ais`, `dismissals`)
5. Seed `geography_entities` with the 11 named entities listed in this architecture document

**Subsequent story sequence:**
1. Ingestor service (OpenSky adapter first; baseline pattern established)
2. Entity state normalization and DB write
3. Processor baseline computation (no signal detection yet; just establish baseline FSM)
4. First signal detector (Port Dwell-Time Anomaly — simplest spatially bounded query)
5. API service (signals endpoint, watches CRUD, system health)
6. Cockpit scaffolding (CockpitShell, auth, Signal Desk empty state)
7. Signal Desk live signals display (REST polling, then WebSocket upgrade)
8. Evidence Surface
9. Remaining signal detectors
10. Compound assembler
11. Spatial View (MapLibre integration)
12. Watch Configuration surface
13. Second and third ingestor providers
14. Feed health, Domain Pulse, session summary

**AI Agent Guidelines:**
- All architectural decisions in this document take precedence over framework defaults
- Use the patterns in the Implementation Patterns section for all naming, error handling, and data format decisions
- Do not add features, tables, or endpoints not described here — extensibility hooks are structural, not feature scope
- Evidence records are **never** modified after creation — enforce at ORM model layer (no `update()` method on `SignalEvidence`)
- All timestamps are UTC; `datetime.now(UTC)` in Python, ISO-8601 strings in API, UTC-parsed in TypeScript
- Signal Desk NOMINAL state is a computed output — never a default or placeholder text
- Spatial View is always triggered from evidence; never rendered as a default home or navigation destination

---

*Architecture document complete. All workflow steps 1–8 executed. Ready for epic and story creation.*
