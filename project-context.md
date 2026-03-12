---
project_name: 'panoptes'
user_name: 'Nikku'
date: '2026-03-08'
sections_completed:
  - technology_stack
  - philosophy_and_constraints
  - architecture_principles
  - service_boundaries
  - naming_conventions
  - data_and_format_rules
  - state_management
  - cockpit_surfaces
  - extensibility_hooks
  - security_rules
  - provider_compliance
  - critical_anti_patterns
source_documents:
  - _bmad-output/planning-artifacts/architecture.md (primary)
  - _bmad-output/planning-artifacts/prd.md (primary)
  - _bmad-output/planning-artifacts/product-brief-panoptes-2026-03-07.md (primary)
  - docs/core/00-brainstorming-brief.md (background only — superseded)
status: 'complete'
---

# Project Context for AI Agents — Panoptes

_Critical rules and patterns AI agents must follow when implementing code for this project. Focuses on unobvious constraints, Panoptes-specific decisions, and things agents commonly get wrong. Architecture document at `_bmad-output/planning-artifacts/architecture.md` remains the authoritative technical reference._

---

## Source of Truth Hierarchy

When any conflict or ambiguity arises between documents:

1. **Latest chat-agreed conclusions** (user explicitly stated in session) — highest priority
2. **`_bmad-output/planning-artifacts/`** — architecture.md, prd.md, product-brief — authoritative
3. **`docs/core/00-brainstorming-brief.md`** — background context only; treat as superseded by the above
4. Never backfill from brainstorming-brief.md if planning-artifacts covers the topic

---

## Technology Philosophy

These rules apply to every technology choice, library selection, and infrastructure decision:

- **Default to open-source, self-hosted**: If an open-source self-hosted option exists that meets the requirement, use it. Do not default to SaaS.
- **No paid SaaS or subscriptions without explicit user approval**: Do not introduce Cloudflare, AWS, Datadog, Auth0, SendGrid, or any subscription-based external service unless Nikku explicitly approves it in session.
- **No vendor lock-in**: Avoid proprietary APIs, SDKs, or data formats as a default implementation choice.
- **Hobby-project affordability first**: Optimize for zero or near-zero recurring cost. Contabo VPS + open-source stack is the target operating cost profile.
- **Preserve future commercial viability**: Affordability constraints do not justify shortcuts that would block commercial use later (e.g., don't use GPL-licensed libraries in production services if MIT alternatives exist; don't hard-code free-tier limits into core logic).

---

## Technology Stack & Versions

| Layer | Technology | Version |
|---|---|---|
| Backend language | Python | 3.12 |
| API framework | FastAPI + Uvicorn | FastAPI 0.115+ |
| ORM | SQLAlchemy (async mode) | 2.x |
| Migrations | Alembic | latest stable |
| DB driver | asyncpg | latest stable |
| Database | PostgreSQL + PostGIS | 16 + 3.4 |
| Geospatial (Python) | Shapely + GeoAlchemy2 | Shapely 2.x |
| Statistics | NumPy + Pandas | latest stable |
| HTTP client (Python) | httpx | 0.27+ |
| Cache / Event bus | **Valkey 7** (Redis-compatible) | 7.x |
| Frontend framework | React + TypeScript | 18 + 5 |
| Frontend build | Vite | 5.x |
| Server state (frontend) | React Query (@tanstack/query) | 5.x |
| UI state (frontend) | Zustand | 4.x |
| Map rendering | MapLibre GL JS | 4.x |
| CSS | Tailwind CSS | 3.x |
| Component primitives | shadcn/ui (Radix UI) | latest |
| Containerization | Docker Compose v2 | latest stable |

### Cache / Event Bus: Valkey (not Redis)

- **Use Valkey 7** as the default cache and pub/sub event bus. Valkey is the open-source Redis fork maintained by the Linux Foundation — fully API-compatible with Redis 7.
- The Docker image is `valkey/valkey:7`. Do not use `redis:7` unless Nikku explicitly approves switching back to Redis.
- All `redis.asyncio` Python client code is compatible with Valkey without changes. The connection string changes only in the Docker Compose service name and image.
- This decision reflects the technology philosophy: Valkey is open-source and avoids Redis Inc. licensing risk.
- If a future architecture decision explicitly approves Redis (e.g., for a specific Redis-only feature), update this section.

### Redis Key Namespaces (applies to Valkey)

- `entity:adsb:{icao24}` — last normalized ADS-B state per aircraft (TTL: 2h)
- `entity:ais:{mmsi}` — last normalized AIS state per vessel (TTL: 6h)
- `feed:health:{provider}` — current feed health state (TTL: 30s, renewed by ingestor heartbeat every 15s)
- `signals:new` — pub/sub channel for newly generated signals
- `watch:status:{watch_id}` — current watch status cache

---

## Core Architectural Philosophy

### Signal-First, Not Map-First

- **Panoptes is signal-first**: The primary output of the platform is signals — anomalies detected against baselines. The map is a contextual tool, not the interface home.
- The pipeline enforces this in code: `collect → fuse → interpret → emit`. Raw data display is not a feature.
- **Spatial View is always triggered from evidence**: it opens in context of an active signal. It is never the default view, never a navigation home, and never rendered speculatively.
- Signal Desk is the operator's primary workspace. All UX and implementation decisions must subordinate the map to the signal.
- Do not add "show all entities on map" features, "live tracking views", or unprompted map layers. These are anti-patterns for this product.

### NOMINAL is Explicit, Not Implicit

- A watch is NOMINAL when: `status = active` AND no signals with `status = active` AND baseline is established.
- NOMINAL is a **computed state** — it must be calculated and displayed explicitly per-watch.
- Signal Desk must never show an empty cell, a spinner, or nothing for a watch that is active and has no signals. It must show NOMINAL with a last-checked timestamp.
- Never treat the absence of data as NOMINAL. Absence of data is either `baseline_in_progress` or a feed health issue.

### Pipeline Stage Separation

The pipeline has four hard stages. Each stage is owned by exactly one service class:

| Stage | Service | What it does |
|---|---|---|
| Collect | `ingestor-*` | Pull from provider, normalize, write raw + entity_state, publish to Valkey |
| Fuse | `processor` | Subscribe to entity events, intersect with watches, update baselines |
| Interpret | `processor` | Detect anomalies, compute confidence, generate signals + evidence |
| Emit | `processor` → `api` | Publish signal to Valkey; API pushes to WebSocket clients |

Do not collapse stages. Do not add detection logic to the API. Do not add DB reads to ingestors for watch config. Pipeline boundary violations are architectural bugs.

---

## Service Boundary Write Rules

These are hard constraints — violating them corrupts data integrity:

| Service | May Write | May NOT Write |
|---|---|---|
| `ingestor-*` | `raw_adsb`, `raw_ais`, `entity_states` | `signals`, `signal_evidence`, `watches`, `geography_entities` |
| `processor` | `signals`, `signal_evidence`, `compound_signal_links`, `watch_baselines` | `watches` (read-only for watch config), `dismissals` |
| `api` | `watches` (CRUD), `signals` (status + dismiss fields only), `dismissals` | `signal_evidence` (never — append-only, processor-only) |
| `cockpit` | Nothing directly | Has no DB or Valkey access |

**Evidence integrity rule**: `signal_evidence` records are append-only. The `SignalEvidence` SQLAlchemy model must not define an `update()` method. The API service must not have code paths that write to `signal_evidence` under any circumstances.

---

## Naming Conventions

### Database (PostgreSQL)

- Tables: `snake_case` plural nouns — `signals`, `signal_evidence`, `watches`, `geography_entities`, `entity_states`, `raw_adsb`, `raw_ais`, `compound_signal_links`, `dismissals`, `watch_config_history`
- Columns: `snake_case` — `geography_entity_id`, `detected_at`, `confidence_breakdown`, `source_completeness`
- Foreign keys: `{referenced_table_singular}_id` — `signal_id`, `watch_id`, `geography_entity_id`
- Indexes: `idx_{table}_{columns}` — `idx_signals_watch_id`, `idx_entity_states_entity_id_observed_at`
- Enum values: `snake_case` — `baseline_in_progress`, `attention_required`, `port_dwell_anomaly`, `compound_disruption`

### Python Backend

- Files: `snake_case` — `signal_processor.py`, `watch_manager.py`, `adsb_ingestor.py`
- Classes: `PascalCase` — `SignalProcessor`, `WatchManager`, `WatchBaseline`, `GeographyEntity`
- Pydantic models (schema layer): `PascalCase` with suffix — `SignalCreate`, `WatchResponse`, `EvidenceRecord`
- SQLAlchemy ORM models: `PascalCase` no suffix — `Signal`, `Watch`, `GeographyEntity`
- Functions/methods: `snake_case` — `detect_dwell_anomaly()`, `update_baseline_incremental()`
- Async functions: always `async def` for any DB or Valkey operation — no exceptions
- Constants: `UPPER_SNAKE_CASE` — `DEFAULT_BASELINE_WINDOW_DAYS = 30`, `HEARTBEAT_INTERVAL_SECONDS = 15`

### TypeScript / React Frontend

- Component files: `PascalCase` — `SignalDesk.tsx`, `EvidenceSurface.tsx`, `WatchConfiguration.tsx`
- Utility / hook files: `camelCase` — `useSignals.ts`, `formatConfidence.ts`, `ws.ts`
- Components: `PascalCase` JSX — `<SignalCard />`, `<DomainPulse />`, `<NominalBadge />`
- Hooks: `use` prefix — `useSignalDesk()`, `useWatchConfig()`, `useFeedHealth()`
- API response types: mirror Pydantic schema names exactly — `Signal`, `Watch`, `GeographyEntity`, `FeedHealth`
- Zustand stores: `use{Name}Store` pattern — `useSignalStore()`, `useSessionStore()`
- Constants: `UPPER_SNAKE_CASE` in `src/constants.ts`

### API Endpoints

- All endpoints prefixed `/api/v1/` — maintain from day one, even in early development
- Resource names: plural nouns — `/signals`, `/watches`, `/geography/entities`
- No trailing slashes
- Query params: `snake_case` — `?signal_family=port_dwell_anomaly&min_confidence=65`

---

## Data & Format Rules

### Timestamps

- All timestamps stored as `timestamptz` in PostgreSQL — always UTC
- All API timestamps: ISO-8601 strings in UTC format — `2026-03-08T14:32:00Z`
- Frontend renders in operator's local timezone for display only; stores and compares in UTC
- **Python**: always `datetime.now(UTC)` — never `datetime.now()` (naive datetime is a bug)
- **TypeScript**: parse ISO strings with `date-fns` parseISO; always treat as UTC on arrival

### Geometry

- All geometry stored in PostGIS as WGS84 (EPSG:4326)
- API responses return GeoJSON geometry objects: `{"type": "Point", "coordinates": [lon, lat]}`
- **Never** return raw WKT or WKB strings in API responses
- MapLibre GL JS renders GeoJSON exclusively — no coordinate system conversion needed at the client

### Confidence Scores

- Always `float` in range `[0.0, 100.0]`
- Never `null` — if confidence cannot be computed, the signal is not emitted
- Stored as `float` in DB; serialized as float in API responses (not integer, not string)
- `confidence_breakdown` is always a JSONB object alongside the score — agents must include both fields when writing a Signal record

### Single-Signal Confidence Formula

```
confidence = (
  deviation_component    # z-score from baseline, capped and normalized
  * source_completeness  # 1.0 = fully available; degrades toward 0.5 for partial
  * baseline_maturity    # 0.5 at minimum sample threshold → 1.0 at 2x threshold
  * recency_factor       # 1.0 if within last polling cycle; decays for stale data
) * 100
```

All four components must be stored in `confidence_breakdown: JSONB` on the Signal record.

### Compound Signal Confidence Formula

```
compound_confidence = (
  harmonic_mean(constituent_confidences)     # harmonic mean penalizes weak constituents
  * convergence_factor                       # temporal + spatial coincidence score (0–1)
  * source_independence_factor               # ADS-B + AIS = 1.0; same-source = 0.75
  * min(constituent_source_completeness)     # worst-feed floor applied
) * 100
```

### API Response Envelope

All REST responses use this exact envelope — no exceptions:

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

### Pagination

All list endpoints: `{ "data": [...], "meta": { "total": N, "page": 1, "page_size": 50 } }`. Default page size: 50. Max: 200.

---

## Python Async Rules

- All DB operations use `async with session:` pattern via SQLAlchemy async sessions
- All Valkey operations are `await redis.publish(...)` — `redis.asyncio` client throughout
- Ingestors use `asyncio.gather()` for concurrent polling where applicable
- **Never** use `time.sleep()` in an async context — always `await asyncio.sleep()`
- CPU-heavy baseline stats (Welford's algorithm updates) run in thread pool executor if they block the event loop

---

## Frontend State Management Rules

- **Server state** (signals, watches, feed health, geography entities): React Query with appropriate `staleTime` and `refetchInterval`
- **UI state** (selected signal ID, active surface, session timestamps): Zustand
- **Never** store server data (API responses) in Zustand
- **Never** store UI state (selected items, navigation) in React Query cache
- React Query handles API error state — no try/catch blocks inside components for data fetching
- Error states are displayed in-place in the relevant component — cockpit remains fully navigable on partial failure
- Feed health errors surface via Domain Pulse panel only — never as modal interrupts or page-blocking states

---

## Cockpit Surfaces

Four **permanent** cockpit surfaces. These are always registered; they cannot be removed or made optional:

| Surface | Component | Description |
|---|---|---|
| Signal Desk | `SignalDesk.tsx` | Primary operator workspace; HorizonView 3-zone (Attention / Monitoring / Pulse); WebSocket-driven |
| Evidence Surface | `EvidenceSurface.tsx` | Drill-down into a signal's evidence chain; opens from Signal Desk |
| Spatial View | `SpatialView.tsx` | MapLibre GL map; signal-scoped GeoJSON layers; opened from evidence only |
| Watch Configuration | `WatchConfiguration.tsx` | Watch CRUD; baseline status; geography entity picker |

**Surface registration pattern**: `CockpitShell.tsx` must drive surface navigation via a surface registry (array/map of surface descriptors), not hardcoded if/else or switch chains. New surfaces (Timeline View, Briefing View — v2+) must be registerable by adding an entry to the registry without modifying CockpitShell logic.

---

## Data Retention Architecture

Four structurally distinct tiers — automated cleanup runs per tier independently:

| Tier | Tables | Retention | Cleanup method |
|---|---|---|---|
| 1 — Raw Feed | `raw_adsb`, `raw_ais` | 48h (per-provider config; subject to terms review) | Partitioned by hour; partition drop job |
| 2 — Entity State | `entity_states` | `max(baseline_window_days) + 7d` (default 37d) | Automated sweep on rows older than window |
| 3 — Signals + Evidence | `signals`, `signal_evidence`, `compound_signal_links` | Indefinite | Operator-defined policy only |
| 4 — Operator Work Product | `watches`, `watch_baselines`, `dismissals`, `watch_config_history` | Indefinite | Explicit operator action only |

- Tier 1 raw retention duration is read from `provider_config` at startup — it is not hardcoded
- Tier 3 and 4 records survive all container restarts — they are in named Docker volumes

---

## Extensibility Hooks (v1: Must Stay NULL)

These fields exist in the schema as structural hooks for future platform expansion. In v1 they are **always NULL**. Do not populate them, do not add placeholder data, do not add code that reads them:

- `geography_entities.people_place_context_id` — future Census/demographic context FK
- `geography_entities.economic_context_id` — future economic baseline FK
- `geography_entities.infrastructure_context_id` — future infrastructure intelligence FK

These hooks acknowledge that geography/geospatial analytics and people-and-place context are foundational to the broader Panoptes platform vision, even though v1 does not implement them. Keeping the FKs nullable and unimplemented is the correct v1 behavior.

Additional extensibility points that must remain stable:

- `SignalFamily` enum: extensible by adding enum value + detector class — no schema migration required
- `signal_evidence.evidence_type` enum: extensible to new evidence types
- `WatchBaseline.parameters`: JSONB — baseline parameter schema evolves without migrations
- `Watch.enabled_signal_families`: list field — new signal families are opt-in per watch
- `Signal.dismissed_by`: always `"operator"` in v1; field exists for multi-operator or automated suppression in v2+
- Ingestor `BaseIngestor` interface: stable — new providers implement it without changing service architecture
- API versioning prefix `/api/v1/`: must be present from the first commit

---

## Watch Lifecycle State Machine

```
CREATED → BASELINE_IN_PROGRESS → ACTIVE → DEGRADED → ACTIVE
                                       ↘ INACTIVE (operator-disabled)
         ← ← ← ← ← RESET ← ← ← ← ← ← (incompatible config change)
```

- `BASELINE_IN_PROGRESS`: data collecting; minimum sample threshold not met; **no signals emitted**
- `ACTIVE`: baseline established; signals evaluated and emitted
- `DEGRADED`: one or more feeds partially unavailable; signals continue with degraded confidence scores
- `INACTIVE`: operator-disabled; processor skips this watch entirely

**Incompatible change rule**: if the operator changes geography scope, baseline window, or signal family for an existing watch, mark old `WatchBaseline` as `superseded`, create a new `in_progress` baseline record, and log the event. Do not silently reuse an existing baseline after a scope change.

**Compatible change rule**: threshold-only changes preserve the existing baseline.

---

## Feed Health Architecture

- Each ingestor publishes a heartbeat to `feed:health:{provider}` every **15 seconds** (Valkey key with **30s TTL**)
- If the TTL expires without renewal, the API reports that feed as `stale` — no active staleness polling is needed; TTL expiry is the mechanism
- Processor checks `source_completeness` per provider before computing any confidence score
- No feed is assumed healthy without a recent heartbeat — this is a hard rule, not a "nice to have"
- Feed health errors are surfaced via the Domain Pulse panel in Signal Desk — not via modal dialogs or full-page states

---

## Provider Terms Compliance

Provider compliance is a **first-class architectural constraint**, not a post-build check:

- Each ingestor reads a `provider_config` at startup containing: `rate_limit_rps`, `storage_allowed`, `retention_hours`, `commercial_use_allowed`, `terms_status`
- `terms_status` values: `under_review` | `compliant` | `restricted`
- If `terms_status` is `under_review` or `restricted`: ingestor runs in **limited mode** — stores normalized entity state only, does not write raw records to Tier 1
- If `storage_allowed = false`: ingestor refuses to start
- Provider API credentials are read from environment variables only — never stored in DB, never logged, never returned in API responses

| Provider | Default Rate Limit | Default Raw Retention | Notes |
|---|---|---|---|
| OpenSky Network | 5 req/sec | 48h | Free tier; non-commercial use; requires review |
| ADS-B Exchange | Per-plan | 48h | Commercial terms require explicit review |
| NOAA AIS | None stated | 48h | Public data; confirm storage terms |

---

## Security Rules

- JWT access token: 15-minute expiry
- JWT refresh token: 30-day expiry, stored as httpOnly cookie
- Single operator credential: bcrypt-hashed password + username, loaded from environment variables at API startup — never in DB
- All API routes require authentication except `/api/v1/auth/login`
- WebSocket connection at `/ws/signals` requires a valid JWT passed at connection time
- All data stored locally — intelligence outputs are never transmitted to external services
- No analytics SDKs, telemetry clients, or external logging services in any service

---

## Error Handling Patterns

### Python Backend

Custom exception hierarchy — always raise typed exceptions, never bare `Exception`:

```python
class PanoptesError(Exception): ...
class SignalNotFoundError(PanoptesError): ...
class BaselineNotEstablishedError(PanoptesError): ...
class FeedUnavailableError(PanoptesError): ...
class WatchNotFoundError(PanoptesError): ...
class EvidenceIntegrityError(PanoptesError): ...
```

FastAPI exception handlers map typed exceptions to the standard error envelope with appropriate HTTP status codes.

### TypeScript Frontend

- React Query handles all API error states — no try/catch in components
- Errors displayed in-place within the component that owns the data
- Never use full-page error screens that block cockpit navigation

---

## Critical Anti-Patterns

**Never do these things:**

- `datetime.now()` in Python — always `datetime.now(UTC)` (naive datetime bugs are silent and hard to find)
- `time.sleep()` in async context — always `await asyncio.sleep()`
- Write `signal_evidence` records from the API service — processor only
- Store server state (API data) in Zustand — React Query only
- Store UI selection state in React Query — Zustand only
- Render Spatial View as a default or home surface — it is always triggered from evidence
- Display an empty cell on Signal Desk for an active NOMINAL watch — NOMINAL must be explicit
- Return WKT or WKB geometry strings in API responses — always GeoJSON
- Hardcode provider credentials, rate limits, or retention windows in service logic — always read from config
- Assume a feed is healthy without a recent heartbeat — no implicit health assumptions
- Add new tables or endpoints not described in architecture.md — extensibility hooks are structural, not feature scope
- Populate `people_place_context_id`, `economic_context_id`, or `infrastructure_context_id` in v1
- Use `docker run` commands or manual container management — Docker Compose v2 only
- Add paid SaaS dependencies without explicit user approval

---

## Project Directory Structure (Key Paths)

```
panoptes/
├── docker-compose.yml               # Production compose (Valkey image: valkey/valkey:7)
├── docker-compose.dev.yml           # Dev overrides
├── .env.example                     # All env vars documented here
├── infra/
│   ├── postgres/init.sql            # PostGIS enable + seed geography_entities
│   ├── valkey/valkey.conf           # Valkey config
│   └── nginx/nginx.conf             # Cockpit static serving + API proxy
├── shared/schema/                   # Shared Pydantic models across services
│   ├── signals.py, watches.py, geography.py, feeds.py, enums.py
├── migrations/                      # Alembic migrations
│   └── versions/001_initial_schema.py  # First migration: all tables + PostGIS
├── services/
│   ├── ingestor/                    # Single service image; PROVIDER env var selects adapter
│   │   └── ingestor/adapters/       # opensky.py, adsbexchange.py, ais_noaa.py
│   ├── processor/                   # Baseline FSM + all signal detectors
│   │   └── processor/signals/       # One file per SignalFamily detector
│   ├── api/                         # FastAPI REST + WebSocket
│   │   └── api/routers/             # signals.py, watches.py, geography.py, system.py, auth.py
│   └── cockpit/                     # React SPA (builds to static)
│       └── src/
│           ├── components/signal-desk/
│           ├── components/evidence-surface/
│           ├── components/spatial-view/
│           └── components/watch-config/
```

---

## Implementation Handoff Sequence

For agents implementing stories, the established order is:

1. Monorepo scaffolding + Docker Compose (all services, named volumes, Valkey)
2. Alembic initial migration (all tables + PostGIS, seed geography_entities with 11 named entities)
3. Ingestor service (OpenSky adapter first; BaseIngestor pattern established)
4. Entity state normalization + DB write + Valkey publish
5. Processor baseline computation (baseline FSM only; no signal detection yet)
6. First signal detector: Port Dwell-Time Anomaly (simplest spatially bounded query)
7. API service (signals list, watches CRUD, system health)
8. Cockpit scaffolding (CockpitShell with surface registry, auth guard, Signal Desk empty state)
9. Signal Desk live signal display (React Query polling → WebSocket upgrade)
10. Evidence Surface
11. Remaining signal detectors (route_deviation, chokepoint_throughput, vessel_density, dark_event, gps_jamming)
12. Compound assembler
13. Spatial View (MapLibre GL JS integration; map tile source: public OSM for v1)
14. Watch Configuration surface
15. Second and third ingestor providers (ADS-B Exchange, NOAA AIS)
16. Feed health, Domain Pulse, session summary

---

---

## Usage Guidelines

**For AI Agents:**

- Read this file before implementing any code in the Panoptes project
- Follow ALL rules exactly as documented — architecture.md is the authoritative reference for full detail
- When in doubt on a technology choice, prefer open-source and self-hosted
- When in doubt on a rule, prefer the more restrictive option
- Do not add features, tables, endpoints, or services not described in architecture.md
- Update this file if new patterns are explicitly agreed with Nikku in session

**For Humans (Nikku):**

- Keep this file lean — focused on rules agents actually need, not comprehensive documentation
- Update when technology stack changes or new conventions are agreed
- Remove rules that have become obvious or are no longer relevant
- `docs/core/00-brainstorming-brief.md` is background only — do not update it; update `_bmad-output/planning-artifacts/` instead

Last Updated: 2026-03-08
