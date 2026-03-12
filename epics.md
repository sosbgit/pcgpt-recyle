---
stepsCompleted: [step-01-validate-prerequisites, step-02-design-epics]
inputDocuments:
  - _bmad-output/planning-artifacts/prd.md
  - _bmad-output/planning-artifacts/architecture.md
  - _bmad-output/planning-artifacts/ux-design-specification.md
  - _bmad-output/planning-artifacts/product-brief-panoptes-2026-03-07.md
  - _bmad-output/project-context.md
implementationConstraints:
  - self-hosted, single-operator, solo-builder-feasible
  - signal-first not map-first
  - open-source / low-recurring-cost default
  - Valkey 7 (not Redis) unless explicitly changed
  - no paid SaaS additions
  - no multi-user, mobile, collaboration, or generic dashboard in v1
  - v1 scope bounded to ADS-B + AIS movement intelligence cockpit only
  - preserve platform extensibility hooks but do not activate Census, satellite, or globe-first modules in v1
v1ScopeNote: FR37 (dismissal history review) deferred to v2
approvedBy: Nikku
approvedDate: 2026-03-08
---

# panoptes - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for Panoptes v1, decomposing the requirements from the PRD, UX Design, and Architecture into implementable stories.

Panoptes v1 is a self-hosted, browser-based SPA cockpit for single-operator movement intelligence (ADS-B + AIS). The platform architecture is broader; v1 scope is strictly bounded to this cockpit. All Census, satellite, globe-first, multi-user, and collaboration scope is explicitly out of v1.

**Build sequence rationale:** Epics 1–4 establish the deployable foundation and data pipeline. Epic 5 delivers the first usable intelligence output — signals on the Signal Desk. Epics 6–8 complete the operator trust path (evidence → spatial). Epic 9 hardens for production and adds full multi-provider coverage.

---

## Requirements Inventory

### Functional Requirements

FR1: The system can ingest ADS-B position and flight data continuously from multiple providers without requiring operator presence
FR2: The system can ingest AIS vessel position and maritime data continuously from multiple providers without requiring operator presence
FR3: The system can handle individual provider outages or rate-limit events without cascading failure to other ingestion streams
FR4: The system can normalize raw feed data from multiple providers into a unified entity-state representation anchored to named geographic entities
FR5: The operator can create a watch condition targeting a defined geographic area, chokepoint, or corridor
FR6: The operator can enable or disable specific signal families per watch condition
FR7: The operator can configure escalation thresholds per watch condition
FR8: The operator can configure baseline window parameters per watch condition
FR9: The operator can modify watch configuration parameters; compatible changes may preserve existing baseline state, while materially incompatible changes (geography, corridor scope, baseline window) must trigger an explicit baseline reset and re-establishment
FR10: The operator can view all active watch conditions and their current status from a single surface
FR11: The system can establish a rolling baseline per active watch condition from accumulated feed data
FR12: The system can detect vessel dwell-time anomalies at monitored ports or anchorage zones against the established baseline
FR13: The system can detect cargo flight route deviations from established corridor baselines
FR14: The system can detect chokepoint throughput changes against rolling baselines
FR15: The system can detect vessel traffic density anomalies (clustering or thinning) in monitored maritime zones
FR16: The system can detect the absence or disappearance of expected entities or traffic patterns (absence detection) as an affirmative finding
FR17: The system can detect AIS vessel dark events at or near monitored strategic chokepoints
FR18: The system can detect GPS jamming zone formation from ADS-B position and velocity incoherence near monitored corridors
FR19: The system can detect commercial route mass deviations from established corridor baselines
FR20: The system can detect convergence of multiple independent signal families on the same geography within a time window and generate a compound finding
FR21: The system can assign a confidence score to each finding reflecting the strength and completeness of contributing evidence
FR22: The system can adjust compound signal confidence to account for degraded or partial source availability at the time of detection
FR23: The operator can view all active findings ranked by priority and confidence on a single primary surface
FR24: The operator can view the current status of all active watches (Attention Required / Monitoring / NOMINAL) from the Signal Desk
FR25: The operator can view a session summary showing signals generated since last viewed, escalated findings, and watch failures
FR26: The operator can view real-time feed health and ingestion status for all active data providers
FR27: The operator can distinguish between a confirmed NOMINAL state (active monitoring, no anomalies) and a degraded or stale state
FR28: The system communicates watch baseline-in-progress status as a named, explicit state
FR29: The Signal Desk updates to reflect new findings and feed health changes without requiring manual page refresh
FR30: The operator can drill into any finding and view its contributing data layers, source readings, confidence components, and feed state at time of detection
FR31: The operator can access the retained underlying source records and source-derived evidence for a finding, subject to applicable provider terms and retention policy
FR32: The operator can view the full provenance chain for any finding: finding → contributing sources → timestamps → feed state at detection time
FR33: The operator can view compound findings with constituent signals and their individual confidence scores alongside the compound score
FR34: The operator can trigger a spatial view of signal-relevant geographic and movement data from the evidence view
FR35: The spatial view displays only signal-relevant layers for the finding in context — it is not a default globe or general map browser
FR36: The operator can dismiss a finding with an associated reason recorded
FR37: [v2 — DEFERRED] The operator can review the history of dismissed findings and their recorded reasons
FR38: The operator can authenticate to access the self-hosted cockpit as a single authorized user
FR39: The system surfaces degraded or stale feed states explicitly — no silent failures
FR40: The system notifies the operator when any active watch stops receiving data, rather than displaying last-known status as current
FR41: The system persists all baseline state, signal history, evidence artifacts, and watch configurations across process restarts
FR42: The operator can assess overall system and feed health without accessing server logs or backend infrastructure directly
FR43: The system associates all findings, evidence artifacts, and watch conditions with named geographic entities at the data model level
FR44: The system maintains structurally distinct retention handling for raw feed data, normalized entity state, signal and evidence artifacts, and operator work product
FR45: The operator can access the full cockpit via a web browser on the self-hosted instance without installing additional client software

### NonFunctional Requirements

NFR1: Feed ingestion must sustain continuous throughput for both ADS-B and AIS streams within applicable provider rate limits, without data gaps under normal operating conditions
NFR2: Signal processing must not introduce delays that render findings materially stale by the time they surface on the Signal Desk; the system must not batch-process findings only on operator session open
NFR3: Baseline computation must be incremental as new data arrives; full baseline recomputation on every processing cycle is not acceptable at production data volumes
NFR4: The Signal Desk must load and present current findings within an acceptable time on initial open during normal system operation
NFR5: Evidence Surface drill-down from a finding must complete without perceptible delay under normal data volumes
NFR6: Backend services must recover from unexpected process restarts and restore full operational state without operator intervention
NFR7: Feed ingestion services must reconnect automatically after provider outages or connection failures without requiring operator action
NFR8: The system must not enter a silent operational failure state; any condition that prevents normal watch monitoring must surface to the operator via the Signal Desk
NFR9: The system must sustain continuous operation on self-hosted infrastructure over weeks without requiring active daily maintenance by the operator
NFR10: Operator authentication is required to access the cockpit; the interface must not be accessible without valid credentials
NFR11: All persisted data must be stored locally on the self-hosted instance
NFR12: Intelligence outputs, operator work product, and watchlist configurations must not be transmitted to external services
NFR13: Provider API credentials must be stored and accessed securely; credentials must not be exposed in logs, responses, or cockpit surfaces
NFR14: Traffic between cockpit and backend services must be protected through local-only deployment boundaries and/or encrypted authenticated transport
NFR15: Evidence and provenance records must be append-only or versioned; they must not be silently or retroactively altered in any way that breaks the traceability chain
NFR16: Baseline state must be protected against corruption during concurrent ingestion and processing operations
NFR17: Watch configuration changes that invalidate existing baseline data must trigger an explicit baseline reset
NFR18: Confidence scores must accurately reflect the actual source data available at time of detection
NFR19: The system must be deployable and reproducible from configuration without manual state reconstruction
NFR20: Routine system health assessment must be possible through the cockpit without requiring operator access to server logs
NFR21: Containerized deployment must ensure service isolation such that a failing component does not require full system restart to recover

### Additional Requirements

#### From Architecture

- **Monorepo structure**: Project is a monorepo with service-level directory boundaries; no starter template — scaffolded from scratch following the specified project tree
- **Docker Compose v2**: All services run as Docker containers; named volumes for all persistent state (postgres-data, valkey-data); restart policies ensure service-level recovery
- **Alembic migrations**: Initial schema covers `geography_entities`, `watches`, `watch_baselines`, `signals`, `signal_evidence`, `compound_signal_links`, `entity_states`, `raw_adsb`, `raw_ais`, `dismissals`
- **Geography entity seeding**: 11 named geographic entities (chokepoints, corridors, ports) seeded in initial migration
- **Shared schema package**: `shared/schema/` Python package containing Pydantic models and enums shared across services (signals.py, watches.py, geography.py, feeds.py, enums.py)
- **Ingestor service**: Single containerized service; PROVIDER env var selects adapter at startup; BaseIngestor abstract class; compliance.py enforces provider rate limits; health.py publishes 15s heartbeat with 30s TTL in Valkey 7
- **Processor service**: Subscribes to Valkey 7 pub/sub; Welford's algorithm for incremental baseline; BaseSignalDetector abstract class; append-only evidence (no ORM update() on SignalEvidence)
- **API service**: FastAPI + SQLAlchemy async; JWT + bcrypt single-operator auth; API versioned at `/api/v1/`; WebSocket endpoint for real-time push
- **Cockpit service**: Browser-based SPA — React 18 + TypeScript 5 + Vite 5; built to static files; served via nginx; MapLibre GL JS for Spatial View; Tailwind CSS + shadcn/ui; React Query + Zustand
- **Valkey 7** (`valkey/valkey:7`): cache and event bus; `redis.asyncio` client is compatible; do not substitute Redis
- **PostgreSQL 16 + PostGIS 3.4**: Primary data store; JSONB for flexible evidence payloads
- **Extensibility hooks remain intact but unused in v1**: `geography_entities.people_place_context_id`, `economic_context_id`, `infrastructure_context_id` stay nullable; `SignalFamily` enum extensible; `/api/v1/` prefix from day one
- **4-tier retention**: raw feed 48h / normalized entity state 37d / signals+evidence indefinite / operator work product indefinite
- **All timestamps UTC**: `datetime.now(UTC)` in Python, ISO-8601 in API, UTC-parsed in TypeScript
- **Production deployment target**: Contabo VPS (Linux), Namecheap domain, nginx reverse proxy, TLS via Let's Encrypt

#### From UX Design

- **Browser SPA, desktop layout only**: No responsive breakpoints; target viewport 1440×900px minimum; desktop browser only (Chrome, Firefox, Edge modern evergreen); no touch optimization in v1; accessed via web browser — no native app packaging
- **Dark theme only**: Single dark theme; no theme toggle
- **Signal-first navigation**: Signal Desk is default landing surface after login; Spatial View is never a default home or navigation destination
- **CockpitShell layout**: Fixed chrome bar (top), left navigation rail (48px), main content area; 4 nav items: Signal Desk, Evidence (contextual), Spatial View (contextual), Watch Configuration
- **Session badge**: Count badge on nav rail showing signals since last **viewed** — clears when signals are actually viewed, not on login/session open
- **Evidence Surface is a layout slot**: Signal Desk remains rendered behind it; Evidence is not a full-surface takeover and not a modal
- **Evidence Surface pattern**: Sticky header (signal context) / scrollable body (evidence layers, confidence breakdown, source badges, timestamps, feed state) / sticky footer (inline dismiss — required reason text field + confirm, no modal)
- **Spatial View is triggered from Evidence only**: Receives full signal context; browser history entry added so Back returns to Evidence; never rendered as standalone navigation destination or default home
- **NOMINAL is a computed explicit state**: Never a placeholder or default text
- **Baseline-in-progress is a named state**: Distinct row type on Signal Desk, not an error or empty state
- **Minimal motion**: Card expand 350ms on new signal arrival; surface switches immediate; hover transitions 100ms max; no gratuitous animation
- **Watch Configuration**: Two-column layout; compatible vs. incompatible change detection with explicit UX branching; Disable vs. Delete hierarchy
- **First-run/empty state**: Signal Desk shows onboarding prompt if no watches configured

### FR Coverage Map

| FR | Epic | Notes |
|---|---|---|
| FR1 | 3 (partial: OpenSky), 9 (full) | ADS-B Exchange added in Epic 9 |
| FR2 | 9 | AIS NOAA adapter |
| FR3 | 3 (isolation design), 9 (full) | Multi-provider isolation complete in Epic 9 |
| FR4 | 3 | Normalization to entity-state |
| FR5 | 2 | Create watch |
| FR6 | 2 | Signal families per watch |
| FR7 | 2 | Escalation thresholds |
| FR8 | 2 | Baseline window config |
| FR9 | 2 | Compatible vs. incompatible change handling |
| FR10 | 2 | Watch list view |
| FR11 | 4 | Welford incremental baseline |
| FR12 | 5 | Port dwell anomaly (reference detector) |
| FR13 | 5 | Cargo route deviation |
| FR14 | 7 | Chokepoint throughput |
| FR15 | 7 | Vessel density (full fidelity after Epic 9 AIS) |
| FR16 | 7 | Absence detection |
| FR17 | 7 | AIS dark events (full fidelity after Epic 9 AIS) |
| FR18 | 7 | GPS jamming |
| FR19 | 7 | Mass route deviation |
| FR20 | 7 | Compound signal |
| FR21 | 5 | Single-signal confidence scoring |
| FR22 | 5 | Degraded confidence adjustment |
| FR23 | 5 | Signal Desk ranked findings |
| FR24 | 4 (watch states), 5 (full layout) | Watch status on Signal Desk |
| FR25 | 5 (badge), 9 (full summary) | Clears on view not on login |
| FR26 | 3 (indicators), 9 (Domain Pulse) | Feed health |
| FR27 | 4 | NOMINAL vs degraded distinction |
| FR28 | 4 | Baseline-in-progress named state |
| FR29 | 5 | WebSocket real-time push |
| FR30 | 6 | Evidence drill-down |
| FR31 | 6 | Source records access |
| FR32 | 6 | Provenance chain |
| FR33 | 7 | Compound finding Evidence view |
| FR34 | 8 | Trigger Spatial View from Evidence |
| FR35 | 8 | Signal-scoped spatial layers only |
| FR36 | 6 | Dismiss with inline required reason |
| FR37 | **v2 — DEFERRED** | Dismissal history review out of v1 scope |
| FR38 | 1 | Authentication |
| FR39 | 3 (no-silent-fail design), 9 (full) | No silent failures |
| FR40 | 4 | Data-stop notification via heartbeat staleness |
| FR41 | 1 | Named Docker volumes |
| FR42 | 9 | Domain Pulse + health surface |
| FR43 | 1 (schema + seed), 2 (watch FK) | Geographic entity anchoring |
| FR44 | 1 (schema structure), 9 (TTL enforcement) | Retention tiers |
| FR45 | 1 | Nginx serving browser SPA |

---

## Epic List

### Epic 1: Platform Foundation, Deployment Bootstrap & Authentication
**Operator outcome:** The operator has a fully deployable Panoptes instance — running in Docker Compose both locally and on their self-hosted Contabo VPS — with a real domain, TLS, nginx reverse proxy, and an authenticated cockpit accessible from a web browser showing the correct empty/first-run state.

Production-like deployment is established from day one. Local dev mirrors production via Docker Compose dev overrides. VPS, domain, and TLS are not deferred.

**Stories:**
- Monorepo scaffold: directory structure per architecture spec, `.env.example` with all required variables, `.gitignore`, Docker Compose dev overrides
- Alembic initial migration: all 10 tables (`geography_entities`, `watches`, `watch_baselines`, `signals`, `signal_evidence`, `compound_signal_links`, `entity_states`, `raw_adsb`, `raw_ais`, `dismissals`) + extensibility FK columns nullable; seed 11 named geography entities
- Shared schema Python package: `shared/schema/` (signals.py, watches.py, geography.py, feeds.py, enums.py)
- Contabo VPS provisioning + Namecheap domain DNS (A record → VPS IP)
- Production nginx config: reverse proxy for API (`/api/`) and cockpit (static SPA), host routing, TLS via Let's Encrypt (Certbot or acme.sh)
- Production Docker Compose: postgres+PostGIS 3.4, Valkey 7 (`valkey/valkey:7`), nginx, named volumes (`postgres-data`, `valkey-data`), restart policies
- FastAPI app skeleton: app structure, Pydantic Settings env loading, database.py (async SQLAlchemy engine), redis_client.py (Valkey connection), `/api/v1/auth` JWT + bcrypt single-operator login
- Cockpit shell: React 18 + Vite 5 + TypeScript 5 scaffold, login screen, CockpitShell layout (chrome bar + left nav rail + main area), Signal Desk first-run/empty state (onboarding prompt when no watches exist)

**FRs:** FR38, FR41, FR43 (schema), FR44 (schema structure), FR45
**NFRs:** NFR10, NFR11, NFR13, NFR14, NFR19, NFR21

---

### Epic 2: Watch Configuration
**Operator outcome:** The operator can create, configure, enable/disable, and manage watch conditions targeting named geographic entities — fully usable before any data flows.

**Stories:**
- Watches API CRUD (`/api/v1/watches`), geography list API (`/api/v1/geography`)
- Watch Configuration surface: two-column layout (watch list left, form right per UX spec §8)
- Watch form: geographic entity selection, signal family toggles, escalation thresholds, baseline window parameter
- Compatible vs. incompatible change detection: compatible edits preserve baseline; incompatible changes (geography, corridor scope, baseline window) surface explicit `[Save & Reset Baseline]` branch with confirmation
- Disable vs. Delete hierarchy (UX spec §8.6–8.7): Disable suspends watch; Delete requires explicit secondary confirmation; re-enable restores prior valid baseline
- Watch list: status badges, empty state, active/disabled visual distinction

**FRs:** FR5, FR6, FR7, FR8, FR9, FR10
**NFRs:** NFR17

---

### Epic 3: Live Feed Ingestion & Feed Health Visibility
**Operator outcome:** The system continuously ingests ADS-B data from OpenSky Network, normalizes it to entity-state anchored to geography entities, and the operator sees live feed health on the Signal Desk — with explicit degraded/stale surfacing, never silent failure.

**Stories:**
- Ingestor service: `BaseIngestor` abstract class, `normalizer.py` (raw → EntityState normalization), `compliance.py` (provider rate limits and storage rules enforcement)
- OpenSky Network adapter (`adapters/opensky.py`): continuous ADS-B position + flight data fetch loop, automatic reconnect on outage
- Valkey 7 pub/sub: entity-state publish on ingest; feed heartbeat publish (15s interval, 30s TTL key in Valkey)
- Entity state writes to DB (`entity_states`, `raw_adsb` tables)
- Feed health API (`/api/v1/health`): derive feed status from Valkey heartbeat key staleness (absent key = stale)
- Signal Desk: per-provider feed health indicators (live / stale), degraded/stale Inline Warning Banner (non-blocking, no animation per UX spec §10.5)

**FRs:** FR1 (partial: OpenSky ADS-B), FR3 (isolation design), FR4, FR26, FR39, FR40
**NFRs:** NFR1, NFR7, NFR8, NFR12, NFR13

---

### Epic 4: Baseline Computation & Watch Lifecycle States
**Operator outcome:** The operator sees each watch transition through its named lifecycle states — Baseline-in-Progress → Monitoring → NOMINAL — and knows the system is actively learning, not silently idle.

**Stories:**
- Processor service: Valkey 7 pub/sub subscriber event loop (`main.py`), `watch_manager.py` baseline FSM
- `baseline.py`: Welford's incremental rolling baseline computation; never full recompute per cycle
- Watch status state machine writes: `CREATED → BASELINE_IN_PROGRESS → ACTIVE → NOMINAL`; transition logged with timestamps
- Heartbeat-based staleness: watch transitions to degraded state if Valkey feed TTL expires; surfaces to Signal Desk (not silent)
- Signal Desk watch state rows: Baseline-in-Progress row (per UX spec §5.6), Monitoring row, NOMINAL row (per UX spec §5.5) — all explicit named states; NOMINAL only when system confirms active monitoring with zero anomalies
- Data-stop notification surfaced on Signal Desk (FR40)

**FRs:** FR11, FR24, FR27, FR28, FR40
**NFRs:** NFR2, NFR3, NFR6, NFR16

---

### Epic 5: Signal Detection & Signal Desk *(First Usable Intelligence Slice)*
**Operator outcome:** The operator opens the browser-based cockpit and sees ranked anomaly signals on the Signal Desk with confidence scores, updating in real-time. This is the first moment the system produces actionable intelligence — the start of the operator trust path. Full confidence in a finding builds through Evidence (Epic 6) and optionally Spatial View (Epic 8).

**Stories:**
- `BaseSignalDetector` abstract class (`signals/base.py`)
- Port Dwell-Time Anomaly detector (`signals/port_dwell.py`): reference implementation, simplest spatially-bounded query
- Cargo Route Deviation detector (`signals/route_deviation.py`): ADS-B signal, usable immediately with Epic 3 data
- Single-signal confidence scoring (`confidence.py`): multi-factor formula per architecture Decision 5
- Degraded confidence adjustment: confidence reflects actual source availability at detection time
- Signals API (`/api/v1/signals`): ranked findings with confidence scores, watch status, feed state metadata
- Signal Desk: full layout — Attention Required zone, Monitoring zone, NOMINAL zone; signal cards (UX spec §5.3–5.4), confidence bar component (UX spec §10.1)
- WebSocket real-time push (`/api/v1/ws`): new signal cards expand from zero height (350ms, no glow/sound/modal interrupt per UX spec §11.2)
- Session delta badge on nav rail: count of signals since last **viewed** — clears when operator views signals, not on login

**FRs:** FR12, FR13, FR21, FR22, FR23, FR24 (full), FR25 (badge), FR29
**NFRs:** NFR2, NFR4, NFR18

---

### Epic 6: Evidence Surface & Dismissal
**Operator outcome:** The operator can drill into any signal and inspect the full evidence chain, provenance, and confidence breakdown — deepening trust in a finding. They can dismiss a finding with a required reason recorded inline.

**Key UX constraints:** Evidence Surface occupies a layout slot; Signal Desk remains rendered behind it — Evidence is not a full-surface takeover and not a modal. Dismiss is inline in the sticky footer with a required reason text field; no separate modal.

**Stories:**
- Evidence Surface layout slot: sticky header (signal title, type, confidence, watch context), scrollable body (evidence layers, confidence breakdown component, source badges, timestamps, feed state at detection time), sticky footer with inline dismiss
- Evidence API (`/api/v1/evidence/{signal_id}`): contributing layers, source readings, confidence components, feed state at detection
- Provenance hash chain: finding → contributing sources → timestamps → feed state; displayed in scrollable body (UX spec §6.5)
- Confidence breakdown component: visual bar + numeric per-factor breakdown (UX spec §6.3)
- "View Spatially" button in scrollable body: navigates to Spatial View surface with full signal context (UX spec §6.6); browser history entry added
- Inline dismiss workflow: sticky footer shows required reason text field + `[Confirm Dismiss]`; reason stored; dismissed signal leaves Signal Desk; no modal (UX spec §6.7)
- Append-only enforcement at ORM layer: no `update()` method on `SignalEvidence` model

**FRs:** FR30, FR31, FR32, FR36
**NFRs:** NFR5, NFR15, NFR18

*FR37 (dismissal history review) — deferred to v2.*

---

### Epic 7: Advanced Signal Detectors & Compound Signals
**Operator outcome:** The operator sees the full spectrum of movement intelligence — all 8 signal families, including compound convergence findings that flag multi-signal patterns on the same geography.

**Stories:**
- Chokepoint throughput detector (`signals/chokepoint_throughput.py`)
- Vessel density anomaly detector (`signals/vessel_density.py`)
- Absence detection detector (`signals/absence.py`): absence is an affirmative finding, not empty state
- AIS dark event detector (`signals/dark_event.py`)
- GPS jamming incoherence detector (`signals/gps_jamming.py`)
- Commercial mass route deviation detector (`signals/mass_deviation.py`) — distinct from single-aircraft route deviation
- Compound assembler (`signals/compound.py`): harmonic mean confidence + convergence bonus + source independence factor + degradation floor (architecture Decision 5); generates compound finding when ≥2 signal families converge on same geography within time window
- Compound signal card on Signal Desk: constituent count badge
- Compound signal Evidence Surface view: constituent signals list with individual scores + compound score (UX spec §6.4, FR33)

**FRs:** FR14, FR15, FR16, FR17, FR18, FR19, FR20, FR33
**NFRs:** NFR18

*FR15 (vessel density) and FR17 (AIS dark events) reach full fidelity after AIS ingestion is live in Epic 9.*

---

### Epic 8: Spatial View
**Operator outcome:** The operator can trigger a geospatial view of signal-relevant movement data directly from the Evidence Surface, with only context-appropriate layers loaded — never a default globe or general map browser.

**Stories:**
- MapLibre GL JS integration (open-source renderer; self-hosted or free tile source — no paid mapping SaaS)
- Spatial View surface: full surface replace (Evidence Surface removed from layout, Spatial View takes its place); browser history entry added — Back returns to Evidence Surface with full state restored
- Left dossier panel: signal context, geography entity metadata, source summary (UX spec §7.2)
- Signal-scoped layer loading: only layers relevant to the triggering signal type are loaded and displayed (UX spec §7.3)
- "View Spatially" button in Evidence Surface body activates this surface with full signal context passed as route state
- Spatial View is never reachable as a default navigation destination or home surface

**FRs:** FR34, FR35
**NFRs:** *(UX spec governs all Spatial View behavior)*

---

### Epic 9: Multi-Provider Ingestion, Full Operational Hardening & Session Intelligence
**Operator outcome:** The system ingests from all three providers (OpenSky, ADS-B Exchange, AIS NOAA), the operator has full session intelligence via Domain Pulse and session summary panel, and the platform is validated for weeks-long unattended operation on the self-hosted VPS.

**Stories:**
- ADS-B Exchange adapter (`adapters/adsbexchange.py`): second ADS-B provider, isolation from OpenSky
- AIS NOAA adapter (`adapters/ais_noaa.py`): vessel position and maritime data; activates FR15/FR17 at full fidelity
- Multi-provider feed isolation validation: single provider outage does not affect other streams (FR3 full)
- 4-tier retention TTL enforcement: raw feed 48h purge, entity-state 37d purge, signals/evidence indefinite, operator work product indefinite
- Domain Pulse component: per-provider health dots, color-coded (green/amber/red), no animation on dot itself — state changes are immediate color updates (UX spec §10.3)
- Session summary panel (UX spec §5.7 right sidebar): full signal count since last viewed, escalated findings count, watch failures — clears correctly on view
- `pg_stat_statements` query monitoring wired; slow query visibility in cockpit health surface
- Operational readiness: end-to-end smoke test on VPS, TLS renewal verification, Docker named volume backup documentation

**FRs:** FR1 (full multi-provider), FR2 (AIS), FR3 (full), FR25 (full session summary), FR26 (Domain Pulse full), FR39 (full), FR42
**NFRs:** NFR1, NFR6, NFR7, NFR8, NFR9, NFR20
