# Story 3.6: Signal Desk Feed Health Visibility

**Status:** ready-for-dev
**Epic:** 3 — Live Feed Ingestion & Feed Health Visibility
**Sprint sequence:** Sixth story of Epic 3; completes Epic 3

---

## Story

As the Panoptes operator,
I need the Signal Desk to surface live feed health for all active providers — with colored status indicators in the Domain Pulse sidebar panel and chrome bar strip, and an explicit inline warning banner when any feed is stale —
so that I can immediately know whether the system is actively ingesting data, and never mistake a stale feed for a healthy one.

---

## Acceptance Criteria

### AC 1 — TypeScript types: `src/types/feeds.ts`

- `services/cockpit/src/types/feeds.ts` is created.
- Mirrors the `FeedHealth` and `FeedHealthResponse` Pydantic schemas from Story 3.5 exactly:

```typescript
export type FeedStatus = 'live' | 'stale'

export interface FeedHealth {
  provider: string
  status: FeedStatus
  last_heartbeat_at: string | null  // always null from API (Story 3.5 design)
  ttl_seconds: number | null        // remaining Valkey TTL when live; null when stale
}

export interface FeedHealthResponse {
  feeds: FeedHealth[]
  checked_at: string  // ISO-8601 UTC
}
```

- No additional fields, no transformation at the type layer. Types mirror the API wire format.

---

### AC 2 — React Query hook: `src/hooks/useFeedHealth.ts`

- `services/cockpit/src/hooks/useFeedHealth.ts` is created.
- Uses `useQuery` from `@tanstack/react-query` to poll `GET /api/v1/feeds/health` via `apiClient`.
- Extract `r.data.data` from the Panoptes response envelope (standard `{ data: ..., meta: ..., error: null }` shape).
- Configuration:
  - `queryKey: ['feed-health']`
  - `refetchInterval: 15_000` — 15s, aligned with ingestor heartbeat publish interval
  - `staleTime: 10_000` — data is considered fresh for 10s; stale after that but still displayed
  - `retry: 2` — retry twice on network error before surfacing failure
- Returns the raw `useQuery` result — no transformation in the hook. The consumer reads `.data`, `.isLoading`, `.isError`.
- **Error contract:** If the query is in error state (network failure, 5xx), consumers MUST treat all providers as unknown/stale. They must NOT assume any provider is healthy when the query fails or is loading.

**Implementation:**

```typescript
import { useQuery } from '@tanstack/react-query'
import { apiClient } from '@/lib/api-client'
import type { FeedHealthResponse } from '@/types/feeds'

export function useFeedHealth() {
  return useQuery<FeedHealthResponse, Error>({
    queryKey: ['feed-health'],
    queryFn: () =>
      apiClient.get('/api/v1/feeds/health').then((r) => r.data.data as FeedHealthResponse),
    refetchInterval: 15_000,
    staleTime: 10_000,
    retry: 2,
  })
}
```

---

### AC 3 — Chrome bar Domain Pulse strip: live status dots

- `services/cockpit/src/components/shell/ChromeBar.tsx` is updated.
- The internal `DomainPulseStrip` function is updated to consume `useFeedHealth()` directly.
- The hardcoded `FEED_LABELS` constant and its static dot array are removed.
- Dot and label color is derived from `feed.status`:
  - `status === 'live'` → dot color `var(--feed-healthy)` (`#22c55e`)
  - `status === 'stale'` → dot color `var(--feed-stale)` (`#ef4444`)
- **Loading state** (before first response): show no provider dots, or show a single neutral dot with label `Checking…` in `--text-tertiary`. Do NOT show any provider dot as healthy while status is unknown.
- **Error state** (query failed after retries): all provider labels, if shown, render in `var(--feed-stale)` — same as stale treatment. Do NOT imply healthy.
- Strip renders one dot per provider returned by the API. In Epic 3 scope this is `opensky` only. ADS-B Exch and NOAA AIS dots are added when those providers are added to `KNOWN_PROVIDERS` in Epic 9.
- Clicking dots: no action — read-only as per UX spec §4.1.
- Provider label format in chrome bar strip: capitalize `provider` field for display — `opensky` → `OpenSky`.

**Label capitalization helper** (keep inline in ChromeBar.tsx, not a shared utility):
```typescript
function formatProviderLabel(provider: string): string {
  const labels: Record<string, string> = { opensky: 'OpenSky' }
  return labels[provider] ?? provider
}
```

---

### AC 4 — Sidebar Domain Pulse panel: detailed per-provider rows

- `services/cockpit/src/pages/SignalDeskPage.tsx` is updated.
- The static Domain Pulse placeholder (all dots in `--text-tertiary`, static labels) is replaced with live data from `useFeedHealth()`.
- The section header `DOMAIN PULSE` is unchanged (11px uppercase, `--text-secondary`).
- Each feed returned by the API renders one row with this structure:
  ```
  [●]  OpenSky    OK · 4s
  ```
  For stale:
  ```
  [●]  OpenSky    Stale — no signal
  ```
- **Per-row layout** (per UX spec §5.7):
  - Dot: `●`, 10px, color from status
  - Provider name: 12px, `--text-secondary` when live, `var(--accent-critical)` when stale
  - Status label + heartbeat info (right-aligned or inline):
    - When live: `OK · {age}s` where `age = 30 - ttl_seconds` (approximate heartbeat age). If `ttl_seconds` is null but status is live (anomalous key with no TTL), display `OK` with no age.
    - When stale: `Stale — no signal`
  - Status text: 11px IBM Plex Mono, `--text-tertiary` when live, `var(--accent-critical)` when stale

- **Contextual note** — shown below any stale or degraded provider row (per UX spec §5.7):
  ```
  ADS-B signals may carry reduced confidence
  while this feed recovers.
  ```
  Style: 11px, `--text-tertiary`. Disappears automatically when feed recovers (hook re-poll re-renders).

- **Loading state**: rows display with `--text-tertiary` dots and label `Checking…` for each known provider. Do NOT render any dot as healthy.
- **Error state**: If the query fails, render a single row: `⚠ Feed status unavailable` in `--accent-critical`, 11px. Do NOT imply any provider is healthy.
- **No providers returned**: If `feeds` array is empty (should not happen in v1 but must be safe), render: `No feed data.` in `--text-tertiary`, 11px.

---

### AC 5 — Inline Warning Banner: stale feed notice in Signal Desk content area

- `services/cockpit/src/pages/SignalDeskPage.tsx` is updated.
- When one or more feeds have `status === 'stale'` (or the query is in error state), a non-blocking Inline Warning Banner renders **at the top of the signal list area**, above the Attention Required zone header.
- The banner does NOT render when all feeds are live and the query has succeeded.
- **Banner structure** (per UX spec "explicit state, never silence" and Epic 3 description reference to "non-blocking, no animation"):
  ```
  ⚠  OpenSky feed is stale — signals may carry reduced confidence while feed recovers.
  ```
  - For multiple stale providers: list each provider name in the message.
  - For query error state: `⚠ Feed health unavailable — signal confidence may be affected.`
- **Visual spec:**
  - Left border: `3px solid var(--accent-attention)` (amber — same urgency as degraded per UX spec §3.1)
  - Background: `var(--bg-raised)`
  - Padding: 8px horizontal, 6px vertical
  - Icon: `⚠` in `var(--accent-attention)`, 12px
  - Text: 12px, `--text-secondary`
  - No animation — immediate render/removal on state change (0ms)
  - No close/dismiss button — this is an ambient system state indicator, not an alert
  - No modal — entirely inline within the scroll container, before zone headers
- Banner disappears automatically when all feeds return to live state (React Query re-poll).

---

### AC 6 — No implicit health assumption at any render path

- At no point in the UI should any feed be implied to be healthy when:
  - `useFeedHealth()` is loading (initial load, not yet resolved)
  - `useFeedHealth()` is in error state (query failed)
  - A specific `feed.status === 'stale'`
- Loading and error states must render neutral or stale visual treatment — never green dots, never `OK` labels.
- This is a hard requirement per architecture: "No feed is assumed healthy without a recent heartbeat."

---

### AC 7 — Build passes: `tsc -b && vite build`

- `tsc --noEmit` (or `tsc -b`) produces zero type errors.
- `vite build` completes successfully with no errors.
- All existing components (Watch Config, auth flow, navigation) remain unaffected.
- The Signal Desk is still the default home. Spatial View is not touched.

---

### AC 8 — Smoke test: live feed health visible in Signal Desk

Manual verification only — not in CI.

**Prerequisites:** `docker compose up valkey postgres -d`, API running, ingestor running (PROVIDER=opensky).

**Steps:**
1. Login at `http://localhost:5173` (Vite dev server running).
2. Signal Desk loads as default home.
3. In the chrome bar right section: one `● OpenSky` dot visible in green (`--feed-healthy`).
4. In the Signal Desk right sidebar, Domain Pulse section: `OpenSky` row visible with green dot, `OK · {N}s` label (N is 1–29).
5. Stop the ingestor. Wait >30 seconds for the Valkey `feed:health:opensky` key to expire.
6. Without page refresh: within ~15s (next React Query poll), the chrome bar dot turns red, the sidebar row turns red with `Stale — no signal`, and the Inline Warning Banner appears at the top of the signal list.
7. Restart ingestor. Within ~15s (next poll), all indicators return to green and the banner disappears.
8. No page refreshes performed during steps 5–7 — all changes are driven by React Query polling.

---

## Tasks / Subtasks

- [ ] **Task 1: Create `src/types/feeds.ts`** (AC 1)
  - [ ] 1.1 Create `services/cockpit/src/types/feeds.ts`
  - [ ] 1.2 Define `FeedStatus`, `FeedHealth`, `FeedHealthResponse` interfaces exactly matching AC 1 spec
  - [ ] 1.3 Verify: no extra fields, no transformation — wire-format types only

- [ ] **Task 2: Create `src/hooks/useFeedHealth.ts`** (AC 2)
  - [ ] 2.1 Create `services/cockpit/src/hooks/useFeedHealth.ts`
  - [ ] 2.2 Import `useQuery` from `@tanstack/react-query`, `apiClient` from `@/lib/api-client`, `FeedHealthResponse` from `@/types/feeds`
  - [ ] 2.3 Implement `useFeedHealth()` with `queryKey: ['feed-health']`, `refetchInterval: 15_000`, `staleTime: 10_000`, `retry: 2`
  - [ ] 2.4 Extract `r.data.data` in queryFn — unwrap the standard response envelope
  - [ ] 2.5 Verify: no transformation, no error-handling inside the hook — consumers handle error/loading states

- [ ] **Task 3: Update `ChromeBar.tsx` — live Domain Pulse strip** (AC 3)
  - [ ] 3.1 Add import for `useFeedHealth` and `FeedHealth` type
  - [ ] 3.2 Remove `FEED_LABELS` constant and static mapping
  - [ ] 3.3 Add `formatProviderLabel()` helper (inline, not exported)
  - [ ] 3.4 Update `DomainPulseStrip` to call `useFeedHealth()` and render dots from `data.feeds`
  - [ ] 3.5 Implement loading state: neutral `--text-tertiary` dot(s) or `Checking…` label — no green implied
  - [ ] 3.6 Implement error state: dots render in `var(--feed-stale)` — no healthy implied
  - [ ] 3.7 Derive dot color from `feed.status` using CSS tokens
  - [ ] 3.8 Verify: no click handler on dots — read-only

- [ ] **Task 4: Update `SignalDeskPage.tsx` — live sidebar Domain Pulse + Inline Warning Banner** (AC 4, AC 5, AC 6)
  - [ ] 4.1 Add imports: `useFeedHealth`, `FeedHealth`
  - [ ] 4.2 Call `useFeedHealth()` at the top of `SignalDeskPage` function (single call — React Query deduplicates across ChromeBar and SignalDeskPage)
  - [ ] 4.3 **Sidebar Domain Pulse section:**
    - [ ] 4.3a Remove static placeholder rows (all 3 hardcoded labels + tertiary dots)
    - [ ] 4.3b Implement loading state row(s) with neutral treatment
    - [ ] 4.3c Implement error state row
    - [ ] 4.3d Implement per-provider live row: dot color, provider label, `OK · {age}s` or `Stale — no signal`
    - [ ] 4.3e Implement heartbeat age derivation: `age = 30 - ttl_seconds` when `ttl_seconds` is not null
    - [ ] 4.3f Implement contextual note below any stale provider row
  - [ ] 4.4 **Inline Warning Banner:**
    - [ ] 4.4a Compute `hasStaleFeeds = isError || (data?.feeds.some(f => f.status === 'stale') ?? false)`
    - [ ] 4.4b Derive stale provider names from `data.feeds` for message text
    - [ ] 4.4c Render banner at top of signal list area (above Attention Required zone header) when `hasStaleFeeds`
    - [ ] 4.4d Apply visual spec: amber left border, `--bg-raised` background, no animation, no dismiss button
    - [ ] 4.4e Verify: banner does NOT render when all feeds are live and query succeeded

- [ ] **Task 5: Build verification** (AC 7)
  - [ ] 5.1 Run `tsc -b` from repo root or `services/cockpit` — zero type errors
  - [ ] 5.2 Run `vite build` — zero errors, module count increases by 2 (new types file + new hook file)
  - [ ] 5.3 Verify no regressions in Watch Config, auth, navigation surfaces

- [ ] **Task 6: Update sprint-status.yaml**
  - [ ] 6.1 Change `3-6-feed-health-signal-desk: backlog` → `ready-for-dev`

- [ ] **Task 7: Local smoke test** (AC 8)
  - [ ] 7.1 Start infra: `docker compose -f docker-compose.yml -f docker-compose.dev.yml up postgres valkey -d`
  - [ ] 7.2 Start API: `uvicorn services.api.api.main:app --reload --host 0.0.0.0 --port 8000 --env-file .env`
  - [ ] 7.3 Start cockpit dev server: `cd services/cockpit && npm run dev`
  - [ ] 7.4 Start ingestor: `PROVIDER=opensky python -m ingestor.main` (repo root, active venv)
  - [ ] 7.5 Login, verify chrome bar dot green, sidebar row green with age label
  - [ ] 7.6 Stop ingestor, wait >30s, verify all indicators flip stale without page refresh
  - [ ] 7.7 Restart ingestor, verify all indicators recover within ~15s (next poll)

---

## Dev Notes

### File Change Map

```
services/cockpit/src/
├── types/
│   └── feeds.ts                          ← CREATE: FeedStatus, FeedHealth, FeedHealthResponse
├── hooks/
│   └── useFeedHealth.ts                  ← CREATE: React Query hook, 15s poll
├── components/shell/
│   └── ChromeBar.tsx                     ← UPDATE: live DomainPulseStrip (remove static FEED_LABELS)
└── pages/
    └── SignalDeskPage.tsx                ← UPDATE: live sidebar Domain Pulse + Inline Warning Banner

_bmad-output/implementation-artifacts/
└── sprint-status.yaml                    ← UPDATE: 3-6-feed-health-signal-desk: ready-for-dev
```

**DO NOT TOUCH:**
- `src/layouts/CockpitShell.tsx` — layout is correct; no feed health logic goes here
- `src/components/shell/NavRail.tsx` — unchanged
- `src/router.tsx` — no new routes
- Any API service files — backend is complete (Story 3.5)
- Any ingestor service files — ingestor is complete (Stories 3.1–3.4)
- `src/pages/SpatialViewPage.tsx`, `EvidencePlaceholderPage.tsx` — not part of this story
- `src/components/watch-config/*` — unrelated to feed health

---

### API Contract (from Story 3.5 — stable)

**Endpoint:** `GET /api/v1/feeds/health`
**Auth:** Bearer token (required — uses existing `apiClient` interceptors)
**Response envelope:**

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
  "meta": { "timestamp": "...", "request_id": "..." },
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
  ...
}
```

**Key contracts this story depends on:**
- `status` is always either `"live"` or `"stale"` — never null, never absent
- `ttl_seconds` is a positive integer when live (1–30), `null` when stale
- `last_heartbeat_at` is always `null` (by design — Story 3.5 decision, Valkey key stores `"alive"` only)
- `feeds` array contains exactly one entry per provider in `KNOWN_PROVIDERS` — currently `["opensky"]`
- Auth is required — `apiClient` handles token attachment and refresh automatically (Story 1.8)

**Envelope unwrap pattern** (consistent with existing hooks `useWatches`, `useGeographyEntities`):
```typescript
queryFn: () =>
  apiClient.get('/api/v1/feeds/health').then((r) => r.data.data as FeedHealthResponse)
```

---

### Heartbeat Age Derivation

The API returns `ttl_seconds` — the remaining Valkey TTL (set at 30s by the ingestor heartbeat). The sidebar displays approximate heartbeat age as:

```typescript
const heartbeatAge = feed.ttl_seconds != null ? 30 - feed.ttl_seconds : null
// e.g. ttl_seconds=26 → age=4s ("OK · 4s")
// e.g. ttl_seconds=null (stale) → age=null ("Stale — no signal")
```

Display: `OK · {heartbeatAge}s` — IBM Plex Mono, 11px, `--text-tertiary`.
Clamp: If `heartbeatAge < 0` (shouldn't happen but defensive), display `OK · <1s`.
If `status === 'live'` but `ttl_seconds === null` (anomalous — no-TTL key): display `OK` with no age suffix.

---

### React Query Deduplication

Both `ChromeBar.tsx` and `SignalDeskPage.tsx` call `useFeedHealth()` independently. Because they share the same `queryKey: ['feed-health']`, React Query deduplicates these into a single network request per poll cycle. No prop-drilling or context needed.

**Important:** Do NOT lift `useFeedHealth()` into `CockpitShell.tsx` or any layout wrapper and pass data as props. Each consumer calls the hook directly — this is the React Query pattern for this codebase (see `useWatches` and `useGeographyEntities` — each consuming component calls its own hook).

---

### Design Token Reference

All tokens are defined in `src/styles/tokens.css` (established in Story 1.8):

| Token | Hex | Use in this story |
|---|---|---|
| `--feed-healthy` | `#22c55e` | Live provider dot in chrome bar and sidebar |
| `--feed-stale` | `#ef4444` | Stale provider dot + text; error state |
| `--feed-degraded` | `#f59e0b` | Reserved for future degraded state (not used in Epic 3) |
| `--accent-attention` | `#f59e0b` | Inline Warning Banner left border |
| `--bg-raised` | `#1c2333` | Inline Warning Banner background |
| `--text-secondary` | `#8896aa` | Banner text, provider label (live) |
| `--text-tertiary` | `#4d5e73` | Heartbeat age, loading state, contextual note |
| `--accent-critical` | `#ef4444` | Stale provider label text, error state text |
| `--border-subtle` | `#263047` | No new borders added; existing zone dividers unchanged |

---

### Provider Label Map

In Epic 3, only `"opensky"` is returned by the API. The label map handles display formatting:

```typescript
const PROVIDER_LABELS: Record<string, string> = {
  opensky: 'OpenSky',
}

function formatProviderLabel(provider: string): string {
  return PROVIDER_LABELS[provider] ?? provider
}
```

This map is intentionally minimal. ADS-B Exchange and NOAA AIS entries are added when those providers are added to `KNOWN_PROVIDERS` in Stories 9.1 and 9.2.

---

### Inline Warning Banner — Full Spec

```typescript
// Stale provider message construction
const staleProviders = data?.feeds.filter(f => f.status === 'stale').map(f => formatProviderLabel(f.provider)) ?? []
const bannerMessage = isError
  ? 'Feed health unavailable — signal confidence may be affected.'
  : staleProviders.length === 1
    ? `${staleProviders[0]} feed is stale — signals may carry reduced confidence while feed recovers.`
    : `${staleProviders.join(', ')} feeds are stale — signals may carry reduced confidence while feeds recover.`

const hasStaleFeeds = isError || (data?.feeds.some(f => f.status === 'stale') ?? false)
```

Banner renders only when `hasStaleFeeds` is true. During loading (`isLoading && !data`): do NOT show the banner — wait for first data before surfacing the warning. After first data, if stale, show immediately.

---

### State Machine Summary

| Query state | `data` available | Chrome bar dots | Sidebar rows | Banner |
|---|---|---|---|---|
| Loading (first load) | No | Neutral/empty | `Checking…` | Not shown |
| Success, all live | Yes | Green per provider | Green rows, `OK · Ns` | Not shown |
| Success, any stale | Yes | Red for stale providers | Red row, `Stale — no signal` | Shown |
| Error (after retries) | No (or stale) | Red / error | Error row | Shown |

---

### No New Dependencies Required

All required packages are already present in `services/cockpit/package.json`:
- `@tanstack/react-query` — present since Story 1.8 (used by `useWatches`, `useGeographyEntities`)
- `axios` — present since Story 1.8 (`apiClient`)
- No new npm packages needed.

---

### No New Env Vars Required

No environment variables are added. The cockpit uses `apiClient` which routes via Vite proxy (`/api → :8000`) in dev and nginx in production. Both are established.

---

### Windows / Local Dev Notes

Cockpit dev server (from `services/cockpit/`):
```bash
npm run dev
```
Access at `http://localhost:5173`. Vite proxies `/api/*` to `localhost:8000`.

API must be running for `useFeedHealth()` to resolve. Without the API, the hook will fail after retries and render the error state — which is correct behavior.

---

## Scope Guardrails

### IN SCOPE — Story 3.6

- `services/cockpit/src/types/feeds.ts` — TypeScript types matching Story 3.5 API contract
- `services/cockpit/src/hooks/useFeedHealth.ts` — React Query hook, 15s polling
- `services/cockpit/src/components/shell/ChromeBar.tsx` — live Domain Pulse strip (dots only)
- `services/cockpit/src/pages/SignalDeskPage.tsx` — live sidebar Domain Pulse panel + Inline Warning Banner
- Passive staleness: UI renders whatever the API says; no client-side TTL tracking
- OpenSky only (one provider) — the API controls `KNOWN_PROVIDERS`
- Loading and error states that never imply healthy
- Build verification (`tsc -b && vite build`)
- Manual smoke test (local dev, no CI)

### DEFERRED — Do NOT implement in this story

| Item | Deferred to |
|---|---|
| Domain Pulse full component with degraded (amber) state | Story 9.5 — requires degraded state logic (processor-side) |
| ADS-B Exchange + NOAA AIS provider dots | Stories 9.1, 9.2 — extend `KNOWN_PROVIDERS` in API, add label entries in cockpit then |
| Session Summary panel (right sidebar) | Story 5.9 / Epic 9 — pending signal data |
| WebSocket real-time push of feed health changes | Story 5.8 — WebSocket endpoint (React Query polling is sufficient for Epic 3) |
| `● LIVE` dot animation on Attention Required zone header | Story 5.8 (WebSocket events needed) |
| Session delta badge live count | Story 5.9 |
| Watch state rows (NOMINAL, Baseline-in-Progress) | Stories 4.5 |
| Signal cards on Signal Desk | Story 5.7 |
| Evidence Surface | Epic 6 |
| Spatial View | Epic 8 |
| Processor service, baseline computation, signal detection | Epics 4–7 |
| Docker Compose production changes | Story 1.6 |
| Any ingestor or API service changes | Backend complete through Story 3.5 |
| `pg_stat_statements` / operational monitoring | Story 9.7 |

---

## Dependencies / Prerequisites

| Dependency | Status | Notes |
|---|---|---|
| Story 3.5 — Feed Health API | ✅ Done (2026-03-11) | `GET /api/v1/feeds/health` live. Response shape confirmed in smoke test. |
| Story 1.8 — Cockpit shell | ✅ Done (2026-03-10) | `apiClient`, `@tanstack/react-query` QueryClientProvider, all CSS tokens in `tokens.css`, CockpitShell layout locked. |
| Story 3.3 — Valkey pub/sub | ✅ Done (2026-03-11) | Ingestor publishes `feed:health:opensky` with 30s TTL every 15s. Passive staleness confirmed working. |
| `ChromeBar.tsx` | ✅ Present | Has static `DomainPulseStrip` — this story replaces it with live version. |
| `SignalDeskPage.tsx` | ✅ Present | Has static Domain Pulse placeholder in right sidebar — this story replaces it. |
| CSS tokens (`--feed-healthy`, `--feed-stale`, `--accent-attention`, etc.) | ✅ Defined in `src/styles/tokens.css` | All required tokens already declared. No new tokens needed. |
| `@tanstack/react-query` | ✅ In `package.json` | Present since Story 1.8. |
| Vite proxy `/api → :8000` | ✅ In `vite.config.ts` | `apiClient` relative paths proxy correctly in dev. |

---

## Config / Env Expectations Before Implementation

No new environment variables. No new Docker Compose changes. All configuration is established.

| Item | Status |
|---|---|
| API running at `:8000` | Required for smoke test (not for build) |
| Valkey running | Required for smoke test (not for build) |
| Ingestor running (PROVIDER=opensky) | Required to see live feed state in smoke test |
| `JWT_SECRET_KEY`, `OPERATOR_USERNAME`, `OPERATOR_PASSWORD_HASH` | Required in `.env` for API auth (existing) |
| `VALKEY_HOST=localhost` | Set in `.env` if running Valkey outside Docker on Windows |

---

## References

- [Story 3.5 — Feed Health API](3-5-feed-health-api.md) — endpoint spec, response shape, passive staleness model, TTL semantics
- [Story 1.8 — Cockpit Shell](1-8-cockpit-shell.md) — `apiClient`, QueryClientProvider setup, CSS token declarations, ChromeBar scaffold
- [Architecture](../planning-artifacts/architecture.md) — Feed Health Architecture section; Valkey key namespaces
- [Project Context — Feed Health Architecture](../project-context.md#feed-health-architecture) — passive staleness rule; "no feed assumed healthy without recent heartbeat"
- [Project Context — Frontend State Management Rules](../project-context.md#frontend-state-management-rules) — React Query for server state; feed health errors via Domain Pulse only
- [UX Design Spec §4.1 — Chrome Bar Domain Pulse strip](../planning-artifacts/ux-design-specification.md) — one dot per provider, read-only
- [UX Design Spec §5.7 — Right Sidebar Domain Pulse](../planning-artifacts/ux-design-specification.md) — per-provider rows, dot colors, contextual note
- [UX Design Spec §3.1 — Color System](../planning-artifacts/ux-design-specification.md) — `--feed-healthy`, `--feed-degraded`, `--feed-stale` tokens
- [epics.md — Epic 3 story list](../planning-artifacts/epics.md) — "feed health indicators + degraded/stale Inline Warning Banner (non-blocking, no animation)"
- [sprint-status.yaml](sprint-status.yaml) — `3-6-feed-health-signal-desk: backlog → ready-for-dev`
- [services/cockpit/src/components/shell/ChromeBar.tsx](../../services/cockpit/src/components/shell/ChromeBar.tsx) — static placeholder to be replaced
- [services/cockpit/src/pages/SignalDeskPage.tsx](../../services/cockpit/src/pages/SignalDeskPage.tsx) — static sidebar placeholder to be replaced
- [services/cockpit/src/styles/tokens.css](../../services/cockpit/src/styles/tokens.css) — all CSS tokens
- [services/cockpit/src/hooks/useWatches.ts](../../services/cockpit/src/hooks/useWatches.ts) — reference hook pattern
- [shared/schema/feeds.py](../../shared/schema/feeds.py) — source Pydantic types for TypeScript mirror
