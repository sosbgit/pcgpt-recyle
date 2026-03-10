# Story 2.2: Watch Configuration Surface

Status: done

## Dependency Note — Local-First Route

Stories 1.4 (VPS provisioning), 1.5 (production nginx), and 1.6 (production Docker Compose) remain **intentionally deferred** — production deployment, no bearing on local development.

**Prerequisites satisfied:**
- 1.1 (monorepo scaffold) — done ✓ — `services/cockpit/src/components/watch-config/.gitkeep` stub in place
- 1.7 (FastAPI app skeleton) — done ✓ — auth, JWT, session restore all live
- 1.8 (cockpit shell) — done ✓ — CockpitShell layout locked; `/watches` route renders `WatchConfigPage.tsx` placeholder; CSS token system, axios client, Zustand auth store, TanStack Query provider all established
- 2.1 (Watches API CRUD) — done ✓ — `GET /api/v1/watches`, `GET /api/v1/geography/entities` both live, tested, and manually verified; 8 endpoint shapes confirmed

---

## Story

As an operator,
I want to open the Watch Configuration surface and immediately see a usable two-column layout — a watch list on the left and a form panel on the right — where I can scan my existing watches, click one to bring it into focus, and create new watches with a single action,
so that I have a working surface scaffold to manage my watch conditions and the developer can build the complete form, change detection, and disable/delete logic on top of it in subsequent stories.

---

## Acceptance Criteria

1. **Two-column layout renders:** Navigating to `/watches` renders a two-column surface inside the CockpitShell `<main>` slot — no page-level scroll; the surface fills the slot exactly (`height: 100%`, `overflow: hidden`).
   - Left column: **320px** fixed width, full height, `--bg-surface` background, `1px solid --border-subtle` right border.
   - Right column: **flex-1** (fills remaining width), full height, `--bg-base` background.
   - Each column scrolls independently.

2. **Watch list column header renders correctly:**
   ```
   WATCHES                              [+ New]
   ─────────────────────────────────────────────
   [●] OpenSky  [●] ADS-B Exch  [●] AIS
   ─────────────────────────────────────────────
   ```
   - "WATCHES" label: 11px uppercase tracked, `--text-secondary`.
   - `[+ New]` button: right-aligned, secondary button style (outline, `--border-subtle`, `--text-secondary` text, 11px). Clicking sets the surface to "creating new" mode (see AC #6).
   - Domain Pulse strip: 3 read-only grey dots (`--text-tertiary`, 6px), labeled "OpenSky", "ADS-B Exch", "AIS" at 11px `--text-secondary`. All dots grey (`--text-tertiary`) — feeds are not live until Epic 3. No click behavior.
   - Two `1px solid --border-subtle` horizontal dividers: one below the WATCHES / [+ New] row, one below the Domain Pulse row.
   - Header area does not scroll — it is fixed at the top of the left column.

3. **Watch list loads from API:** On mount, the surface calls `GET /api/v1/watches` and `GET /api/v1/geography/entities` via TanStack Query. Both calls use `apiClient` from `@/lib/api-client` (which attaches the Bearer token automatically via the existing request interceptor). If either call is in-flight, the list body shows a loading indicator. If either call fails, an inline error message appears in the list body area.

4. **Watch list rows render:** Each watch returned by the API renders as a clickable row in the list body:
   ```
   ─────────────────────────────────────────
     Strait of Hormuz Watch
     Strait of Hormuz  ·  active
   ─────────────────────────────────────────
   ```
   - Row layout: 16px left padding, 12px top/bottom padding, full column width.
   - Line 1 (watch name): 13px, `--text-primary` for active watches, `--text-tertiary` for inactive watches.
   - Line 2 (entity name · status): 11px, `--text-secondary` for active watches, `--text-tertiary` for inactive watches. Entity name is resolved from the geography entities response via `geography_entity_id`. Status text is the raw `status` value from the API (e.g., `active`, `created`, `inactive`, `baseline_in_progress`).
   - Inactive watches (`is_active: false`): entire row uses `--text-tertiary` on both lines; no left accent bar; no background highlight.
   - `1px solid --border-subtle` top divider on every row.
   - Rows are ordered as returned by the API (`created_at` descending).

5. **Watch list empty state:** If the API returns an empty array, the list body shows:
   ```
   No watches configured.
   Click [+ New] to create your first watch.
   ```
   12px, `--text-tertiary`, centered vertically in the list body. "[+ New]" in this text is not a link — it refers to the button above.

6. **Row selection — single source of truth:** Clicking a watch row selects it. The selected row receives:
   - `3px solid --accent-attention` left border.
   - `--bg-raised` background.
   - No other visual treatment.
   Only one row can be selected at a time. Clicking `[+ New]` deselects any selected watch and enters "creating new" mode. Clicking a row while in "creating new" mode exits that mode and selects the clicked watch instead.

7. **Form panel — no selection state:** When no watch is selected and `[+ New]` has not been clicked, the right column shows a centered placeholder:
   ```
   Select a watch from the list, or click [+ New] to create one.
   ```
   12px, `--text-tertiary`, centered vertically and horizontally in the column.

8. **Form panel — creating new state:** When `[+ New]` is clicked, the right column shows a sticky panel header:
   ```
   NEW WATCH
   ─────────────────────────────────────────────
   ```
   - Header: 16px top/bottom padding, 20px left padding, `--bg-surface` background, `1px solid --border-subtle` bottom border. Fixed at top of right column (does not scroll).
   - "NEW WATCH" label: 14px, weight 500, `--text-secondary` (placeholder styling — form not yet implemented).
   - Panel body (scrollable, below header): placeholder text "Watch form — implementation pending (Story 2.3)" at 12px `--text-tertiary`, padded 20px, top 16px.

9. **Form panel — watch selected state:** When a watch row is clicked, the right column shows a sticky panel header:
   ```
   STRAIT OF HORMUZ WATCH
   ─────────────────────────────────────────────
   ```
   - Header: same dimensions and background as AC #8.
   - Watch name: 14px, weight 500, `--text-primary`, uppercase. Uses the watch's `name` field from the API.
   - Panel body (scrollable, below header): placeholder text "Watch form — implementation pending (Story 2.3)" at 12px `--text-tertiary`, padded 20px, top 16px.
   - **The form header does NOT include a `[Disable]` button in this story.** The disable/delete action hierarchy is Story 2.5.

10. **Nav rail active state:** The "Watches" nav item shows the active state (3px amber left border + `--bg-raised` + `--text-primary` icon/label) when on `/watches`. This is handled by the existing `NavLink` component from Story 1.8 and requires no new code.

11. **Token and auth:** All API calls succeed when the operator is authenticated. A 401 response triggers the existing axios refresh interceptor (Story 1.8). The watch list does not implement any independent auth handling — it relies entirely on the established interceptor chain.

12. **`npm run build` passes:** `tsc -b && vite build` completes without TypeScript errors after all new files are added.

13. **Smoke test passes locally:**
    ```bash
    # Terminal 1: API + infra (from repo root)
    docker compose -f docker-compose.yml -f docker-compose.dev.yml up postgres valkey -d
    uvicorn services.api.api.main:app --reload --host 0.0.0.0 --port 8000 --env-file .env

    # Terminal 2: Cockpit (from repo root)
    cd services/cockpit
    npm run dev
    # → Login → navigate to Watches via nav rail
    # → Two-column layout renders: 320px left column, form column fills remaining
    # → Watch list header visible: WATCHES [+ New], Domain Pulse strip, dividers
    # → If watches exist in DB: rows render with name + entity name + status text
    # → If no watches in DB: empty state message renders
    # → Click watch row → row highlights (amber left bar, bg-raised), right panel shows watch name header
    # → Click [+ New] → row deselects, right panel shows NEW WATCH header
    # → Click different row → previous selection clears, new row highlights
    # → npm run build → no TypeScript or Vite build errors
    ```

---

## Non-Goals (Explicit Scope Boundaries)

This story establishes the surface structure only. The following are explicitly deferred:

| Item | Deferred To |
|---|---|
| Watch form fields (geography dropdown, signal families, threshold slider, baseline window) | Story 2.3 |
| Geography entity thumbnail / entity boundary preview | Story 2.3 |
| Optimistic save, inline save confirmation | Story 2.3 |
| Compatible vs. incompatible change detection and warning banners | Story 2.4 |
| `[Disable]` / `[Enable]` button and inline confirmation | Story 2.5 |
| Delete link, delete confirmation block | Story 2.5 |
| Status badge visual treatment (colored left stripes per status, `--accent-monitoring` for ACTIVE, `--accent-nominal` for NOMINAL, etc.) | Story 2.6 |
| Baseline-in-progress progress bar (`████████░░  18/30d`) | Story 2.6 |
| Signal count on list rows (`ACTIVE · 2 signals`) | Story 2.6 (no signals exist until Epic 5) |
| Live Domain Pulse dots (green/amber/red) | Epic 3 |
| Backend/API CRUD work | Story 2.1 (done) |
| Schema / migrations | Story 1.2 (done) |
| VPS / nginx / production Docker scope | Stories 1.4–1.6 (deferred) |

---

## Story Boundary Callouts — Ambiguity with 2.3 / 2.4 / 2.5 / 2.6

The following boundaries are intentionally explicit to prevent scope drift:

### 2.2 → 2.3 (Watch Form)
- 2.2 shows a **stub right panel** with watch name in the header and placeholder text in the body.
- 2.3 **replaces the panel body** with real form fields. The sticky header pattern, `--bg-surface` header background, and `1px solid --border-subtle` bottom border established in 2.2 are the exact structure 2.3 builds on — do not change them.
- Geography entities are fetched in 2.2 (`GET /api/v1/geography/entities`) and cached by TanStack Query. Story 2.3 consumes the same cached data via `useGeographyEntities()` — no duplicate fetch.

### 2.2 → 2.4 (Change Detection)
- 2.2 introduces **zero** form field state. There is nothing to track for changes.
- 2.4 adds its change-detection logic entirely within the form component built by 2.3. Story 2.2 has no dependency on or coupling to 2.4.

### 2.2 → 2.5 (Disable / Delete Hierarchy)
- 2.2's form panel header shows **only the watch name** — no action buttons.
- 2.5 adds `[Disable]` to the right side of the form header and `delete watch` link below the form body. These are additive; 2.5 modifies the `WatchFormPanel` component created here.
- **Risk:** If Story 2.3 or 2.5 hardcodes a full form header markup, the Disable button placement from the UX spec (`§8.3`) may not align. Mitigation: 2.2 uses a `right-slot` prop pattern in the form header (`rightSlot?: React.ReactNode`), defaulting to `null`, so 2.5 can pass `<DisableButton />` without modifying the header component structure.

### 2.2 → 2.6 (Watch List Visual Richness)
**This is the sharpest boundary to enforce.**
- 2.2 shows status as **plain text** (`active`, `nominal`, `inactive`, `baseline_in_progress`) — no colored left stripe on the row, no styled badge pill.
- 2.6 replaces the status text with **status badge components**: per-status colored left stripe, pill badge, and for `baseline_in_progress` a progress bar.
- 2.2 does apply **inactive row dimming** (`--text-tertiary` on both text lines, no accent treatment). This is a trivial, stable visual rule that prevents inactive watches from appearing active — safe to include here.
- 2.6 also handles the full signal count on rows (`2 signals`) — not relevant until Epic 5 data exists.
- **Risk:** 2.6 will need to modify `WatchListRow` (or replace it). Make `WatchListRow` accept a `badge?: React.ReactNode` prop (defaulting to a plain `<span>` with status text) so 2.6 can pass in the full badge component. Document this in the component's inline comment.

---

## Tasks / Subtasks

- [x] Task 1: Create TanStack Query hooks (AC: #3)
  - [x] 1.1 Create `services/cockpit/src/hooks/useWatches.ts`: exports `useWatches()` — `useQuery({ queryKey: ['watches'], queryFn: fetchWatches })` where `fetchWatches` calls `GET /api/v1/watches` via `apiClient`; returns `WatchRow[]` (typed interface, see Dev Notes)
  - [x] 1.2 Create `services/cockpit/src/hooks/useGeographyEntities.ts`: exports `useGeographyEntities()` — `useQuery({ queryKey: ['geography-entities'], queryFn: fetchGeographyEntities })` where `fetchGeographyEntities` calls `GET /api/v1/geography/entities` via `apiClient`; returns `GeographyEntityRow[]` (typed interface, see Dev Notes); staleTime: 5 minutes (entity list changes rarely)

- [x] Task 2: Create watch list column component (AC: #2, #4, #5, #6)
  - [x] 2.1 Create `src/components/watch-config/WatchListColumn.tsx`: accepts `selectedWatchId: string | null`, `isCreatingNew: boolean`, `onSelectWatch: (id: string) => void`, `onNewWatch: () => void` props
  - [x] 2.2 Implement fixed column header: "WATCHES" label + `[+ New]` button, Domain Pulse placeholder strip, two dividers (see Dev Notes for exact layout)
  - [x] 2.3 Create `src/components/watch-config/WatchListRow.tsx`: accepts `watch: WatchRow`, `entityName: string`, `isSelected: boolean`, `onClick: () => void`, `badge?: React.ReactNode` (defaults to plain status text span). Implements selected state (amber left border + bg-raised) and inactive dimming.
  - [x] 2.4 Implement scrollable list body in `WatchListColumn`: loading state, error state, empty state (AC #5), list of `WatchListRow` when data available
  - [x] 2.5 Build entity name lookup map: `entityMap = useMemo(() => new Map(entities.map(e => [e.id, e.name])), [entities])` — resolves geography entity name from `watch.geography_entity_id`

- [x] Task 3: Create watch form panel component (AC: #7, #8, #9)
  - [x] 3.1 Create `src/components/watch-config/WatchFormPanel.tsx`: accepts `mode: 'none' | 'new' | 'selected'`, `watchName?: string`, `rightSlot?: React.ReactNode` (reserved for Story 2.5 disable button, default null)
  - [x] 3.2 Implement `mode === 'none'` state: centered placeholder text (AC #7)
  - [x] 3.3 Implement `mode === 'new'` state: sticky header "NEW WATCH" + placeholder body (AC #8)
  - [x] 3.4 Implement `mode === 'selected'` state: sticky header with `watchName` in uppercase + placeholder body (AC #9)
  - [x] 3.5 Sticky header implementation: `position: sticky; top: 0; z-index: 1` within the column's scroll container (see Dev Notes — important pattern for 2.3)

- [x] Task 4: Replace `WatchConfigPage.tsx` with full two-column layout (AC: #1, #6, #10, #11)
  - [x] 4.1 Replace stub `src/pages/WatchConfigPage.tsx` with full implementation: `height: 100%; display: flex; overflow: hidden` at page level
  - [x] 4.2 Left column: `width: 320px; minWidth: 320px; height: 100%; background: --bg-surface; borderRight: 1px solid --border-subtle; display: flex; flexDirection: column`
  - [x] 4.3 Right column: `flex: 1; height: 100%; background: --bg-base; overflow-y: auto`
  - [x] 4.4 Wire selection state: `useState<string | null>(null)` for `selectedWatchId` and `useState<boolean>(false)` for `isCreatingNew`
  - [x] 4.5 Wire `onSelectWatch`: sets `selectedWatchId`, clears `isCreatingNew`
  - [x] 4.6 Wire `onNewWatch`: clears `selectedWatchId`, sets `isCreatingNew = true`
  - [x] 4.7 Determine form panel mode: `mode = isCreatingNew ? 'new' : selectedWatchId ? 'selected' : 'none'`
  - [x] 4.8 Pass selected watch name to `WatchFormPanel`: look up in watches data by `selectedWatchId`

- [x] Task 5: TypeScript compile + smoke test (AC: #12, #13)
  - [x] 5.1 `npm run build` — confirm no TypeScript errors — PASSED (153 modules, 0 errors)
  - [x] 5.2 Start infra + API + cockpit dev server; navigate to `/watches` — manual smoke test (Nikku)
  - [x] 5.3 Verify layout dimensions: 320px left, form fills remaining; no page scroll; each column independently scrollable
  - [x] 5.4 Verify watch list: rows load from API, entity names resolved, selection state correct
  - [x] 5.5 Verify `[+ New]` / row click / deselection behavior
  - [x] 5.6 Verify empty state when no watches in DB

---

## Dev Notes

### File Structure — What to Create / Replace

```
services/cockpit/src/
├── hooks/
│   ├── useWatches.ts              ← CREATE
│   └── useGeographyEntities.ts   ← CREATE
├── components/
│   └── watch-config/
│       ├── WatchListColumn.tsx   ← CREATE (replaces .gitkeep)
│       ├── WatchListRow.tsx      ← CREATE
│       └── WatchFormPanel.tsx    ← CREATE
└── pages/
    └── WatchConfigPage.tsx       ← REPLACE stub
```

**Files NOT touched:**
- `src/layouts/CockpitShell.tsx` — shell layout is locked; no modifications
- `src/styles/tokens.css` — color tokens are locked; reference existing variables only
- `src/lib/api-client.ts` — interceptor chain established; do not modify
- `src/components/shell/NavRail.tsx` — no changes; active state on Watches item already works via `NavLink`
- `services/api/` — no backend changes in this story
- `migrations/` — no schema changes
- `docker-compose.yml` / `docker-compose.dev.yml` — no changes

---

### TypeScript Interfaces

Define in the hook files. Keep these minimal — they match the API response shape exactly.

```typescript
// src/hooks/useWatches.ts
export interface WatchRow {
  id: string
  name: string
  geography_entity_id: string
  status: string           // "created" | "baseline_in_progress" | "active" | "nominal" | "degraded" | "inactive"
  is_active: boolean
  escalation_threshold: number
  baseline_window_days: number
  enabled_signal_families: string[]
  created_at: string
  updated_at: string
}

// src/hooks/useGeographyEntities.ts
export interface GeographyEntityRow {
  id: string
  name: string
  entity_type: string
  is_active: boolean
}
```

Do NOT import from `shared/schema/` — that package is Python only. Define thin TypeScript interfaces that match the JSON API response shape.

---

### Hook Implementation Pattern

```typescript
// src/hooks/useWatches.ts
import { useQuery } from '@tanstack/react-query'
import { apiClient } from '@/lib/api-client'

export interface WatchRow { /* see above */ }

async function fetchWatches(): Promise<WatchRow[]> {
  const res = await apiClient.get<{ data: WatchRow[] }>('/api/v1/watches')
  return res.data.data
}

export function useWatches() {
  return useQuery({
    queryKey: ['watches'],
    queryFn: fetchWatches,
  })
}
```

```typescript
// src/hooks/useGeographyEntities.ts
import { useQuery } from '@tanstack/react-query'
import { apiClient } from '@/lib/api-client'

export interface GeographyEntityRow { /* see above */ }

async function fetchGeographyEntities(): Promise<GeographyEntityRow[]> {
  const res = await apiClient.get<{ data: GeographyEntityRow[] }>('/api/v1/geography/entities')
  return res.data.data
}

export function useGeographyEntities() {
  return useQuery({
    queryKey: ['geography-entities'],
    queryFn: fetchGeographyEntities,
    staleTime: 5 * 60 * 1000,  // 5 minutes — entity list changes rarely
  })
}
```

**`apiClient` has no `baseURL`** (patched in Story 1.8 M4). All calls use relative `/api/v1/...` paths — Vite proxies to `:8000` in dev. Do NOT add `http://localhost:8000` prefix anywhere.

---

### WatchConfigPage Layout Implementation

```typescript
// src/pages/WatchConfigPage.tsx (simplified structure)
export function WatchConfigPage() {
  const [selectedWatchId, setSelectedWatchId] = useState<string | null>(null)
  const [isCreatingNew, setIsCreatingNew] = useState(false)

  const { data: watches, isLoading: watchesLoading, error: watchesError } = useWatches()
  const { data: entities } = useGeographyEntities()

  const entityMap = useMemo(
    () => new Map((entities ?? []).map(e => [e.id, e.name])),
    [entities]
  )

  const selectedWatch = watches?.find(w => w.id === selectedWatchId) ?? null

  const formMode = isCreatingNew ? 'new' : selectedWatchId ? 'selected' : 'none'

  const handleSelectWatch = (id: string) => {
    setSelectedWatchId(id)
    setIsCreatingNew(false)
  }

  const handleNewWatch = () => {
    setSelectedWatchId(null)
    setIsCreatingNew(true)
  }

  return (
    <div style={{ height: '100%', display: 'flex', overflow: 'hidden' }}>
      {/* Left column */}
      <div style={{
        width: 320, minWidth: 320, height: '100%',
        background: 'var(--bg-surface)',
        borderRight: '1px solid var(--border-subtle)',
        display: 'flex', flexDirection: 'column',
      }}>
        <WatchListColumn
          selectedWatchId={selectedWatchId}
          isCreatingNew={isCreatingNew}
          watches={watches ?? []}
          entityMap={entityMap}
          isLoading={watchesLoading}
          error={watchesError}
          onSelectWatch={handleSelectWatch}
          onNewWatch={handleNewWatch}
        />
      </div>

      {/* Right column */}
      <div style={{ flex: 1, height: '100%', overflowY: 'auto', background: 'var(--bg-base)' }}>
        <WatchFormPanel
          mode={formMode}
          watchName={selectedWatch?.name}
        />
      </div>
    </div>
  )
}
```

**Critical layout note:** The `<main>` slot in `CockpitShell` is `position: fixed; top: 36; left: 120; right: 0; bottom: 0; overflowY: auto`. By setting `height: 100%; overflow: hidden` on `WatchConfigPage`, the page fills the slot exactly without triggering the slot's outer scroll. Each column's internal scroll is independent. This is the correct pattern for multi-column surfaces with independent scroll regions.

---

### WatchListColumn Header — Exact DOM Structure

```typescript
{/* Fixed header — does not scroll */}
<div style={{ flexShrink: 0 }}>
  {/* Row 1: WATCHES + [+ New] */}
  <div style={{
    display: 'flex', alignItems: 'center', justifyContent: 'space-between',
    padding: '10px 12px',
  }}>
    <span style={{ fontSize: 11, textTransform: 'uppercase', letterSpacing: '0.08em', color: 'var(--text-secondary)' }}>
      Watches
    </span>
    <button
      onClick={onNewWatch}
      style={{
        fontSize: 11, color: 'var(--text-secondary)',
        border: '1px solid var(--border-subtle)',
        background: 'transparent', borderRadius: 3,
        padding: '2px 8px', cursor: 'pointer',
      }}
    >
      + New
    </button>
  </div>

  {/* Divider */}
  <div style={{ height: 1, background: 'var(--border-subtle)' }} />

  {/* Row 2: Domain Pulse strip */}
  <div style={{ padding: '8px 12px', display: 'flex', gap: 12, alignItems: 'center' }}>
    {['OpenSky', 'ADS-B Exch', 'AIS'].map(label => (
      <div key={label} style={{ display: 'flex', alignItems: 'center', gap: 4 }}>
        <div style={{ width: 6, height: 6, borderRadius: '50%', background: 'var(--text-tertiary)' }} />
        <span style={{ fontSize: 11, color: 'var(--text-secondary)' }}>{label}</span>
      </div>
    ))}
  </div>

  {/* Divider */}
  <div style={{ height: 1, background: 'var(--border-subtle)' }} />
</div>

{/* Scrollable list body */}
<div style={{ flex: 1, overflowY: 'auto' }}>
  {/* rows here */}
</div>
```

---

### WatchListRow — Exact DOM Pattern

```typescript
// WatchListRow.tsx
interface WatchListRowProps {
  watch: WatchRow
  entityName: string
  isSelected: boolean
  onClick: () => void
  badge?: React.ReactNode  // reserved for Story 2.6 status badges; defaults to plain text
}

export function WatchListRow({ watch, entityName, isSelected, onClick, badge }: WatchListRowProps) {
  const isInactive = !watch.is_active
  const textColor = isInactive ? 'var(--text-tertiary)' : 'var(--text-primary)'
  const subColor = 'var(--text-tertiary)'  // always tertiary for secondary line
  const subActiveColor = isInactive ? 'var(--text-tertiary)' : 'var(--text-secondary)'

  return (
    <div
      onClick={onClick}
      style={{
        padding: '12px 12px 12px 16px',
        cursor: 'pointer',
        borderTop: '1px solid var(--border-subtle)',
        borderLeft: isSelected ? '3px solid var(--accent-attention)' : '3px solid transparent',
        background: isSelected ? 'var(--bg-raised)' : 'transparent',
        paddingLeft: isSelected ? 13 : 16,  // compensate for 3px border
      }}
    >
      <div style={{ fontSize: 13, color: textColor, marginBottom: 4 }}>
        {watch.name}
      </div>
      <div style={{ fontSize: 11, color: subActiveColor }}>
        {entityName}
        {badge ?? (
          <>
            {' · '}
            <span>{watch.status.replace(/_/g, ' ')}</span>
          </>
        )}
      </div>
    </div>
  )
}
```

**Note for Story 2.6:** The `badge` prop exists as a slot. In 2.2, it is always `undefined`, rendering the plain status text fallback. Story 2.6 passes in a `<StatusBadge status={watch.status} />` component and removes the status text fallback from the secondary line. This requires no change to `WatchListRow` itself.

---

### WatchFormPanel — Sticky Header Pattern

The form panel header must be `position: sticky; top: 0` within the right column's scroll container, not `position: fixed`. This is important — 2.3 will add a scrollable form body below the header, and `sticky` keeps the header visible during body scroll without breaking the layout.

```typescript
// WatchFormPanel.tsx (structural pattern for 2.3 to build on)
interface WatchFormPanelProps {
  mode: 'none' | 'new' | 'selected'
  watchName?: string
  rightSlot?: React.ReactNode  // reserved for Story 2.5: <DisableButton /> — default null
}

export function WatchFormPanel({ mode, watchName, rightSlot }: WatchFormPanelProps) {
  if (mode === 'none') {
    return (
      <div style={{
        height: '100%', display: 'flex',
        alignItems: 'center', justifyContent: 'center',
      }}>
        <p style={{ fontSize: 12, color: 'var(--text-tertiary)', textAlign: 'center' }}>
          Select a watch from the list, or click [+ New] to create one.
        </p>
      </div>
    )
  }

  const headerLabel = mode === 'new'
    ? 'NEW WATCH'
    : (watchName ?? '').toUpperCase()

  return (
    <div style={{ display: 'flex', flexDirection: 'column' }}>
      {/* Sticky header */}
      <div style={{
        position: 'sticky', top: 0, zIndex: 1,
        background: 'var(--bg-surface)',
        borderBottom: '1px solid var(--border-subtle)',
        padding: '16px 20px',
        display: 'flex', alignItems: 'center', justifyContent: 'space-between',
      }}>
        <span style={{ fontSize: 14, fontWeight: 500, color: mode === 'new' ? 'var(--text-secondary)' : 'var(--text-primary)' }}>
          {headerLabel}
        </span>
        {rightSlot ?? null}  {/* Story 2.5 will pass <DisableButton /> here */}
      </div>

      {/* Scrollable body — Story 2.3 replaces this placeholder */}
      <div style={{ padding: '16px 20px' }}>
        <p style={{ fontSize: 12, color: 'var(--text-tertiary)', margin: 0 }}>
          Watch form — implementation pending (Story 2.3)
        </p>
      </div>
    </div>
  )
}
```

**`rightSlot` note for Story 2.5:** When 2.5 adds the Disable button, it passes `rightSlot={<DisableButton watch={selectedWatch} onDisabled={handleWatchUpdated} />}` from `WatchConfigPage`. This prop flows through to the form header without requiring modifications to `WatchFormPanel`'s header DOM structure.

---

### API Response Shape — Confirmed

From Story 2.1 implementation (manually verified by Nikku):

```typescript
// GET /api/v1/watches — response.data.data
[
  {
    id: "uuid",
    name: "Strait of Hormuz Watch",
    geography_entity_id: "uuid",
    status: "created" | "baseline_in_progress" | "active" | "nominal" | "degraded" | "inactive",
    is_active: true,
    escalation_threshold: 65.0,
    baseline_window_days: 30,
    enabled_signal_families: ["port_dwell_anomaly", "ais_dark_event"],
    created_at: "2026-03-10T12:00:00+00:00",
    updated_at: "2026-03-10T12:00:00+00:00",
  }
]

// GET /api/v1/geography/entities — response.data.data
[
  {
    id: "uuid",
    name: "Strait of Hormuz",
    entity_type: "chokepoint",
    source: "seeded",
    metadata: { ... },
    tags: [],
    is_active: true,
    created_at: "...",
    updated_at: "...",
  }
]
```

Both responses use the standard Panoptes envelope: `{ data: [...], meta: { timestamp, request_id }, error: null }`.

---

### No New npm Packages

All required dependencies are already installed from Story 1.8:
- `@tanstack/react-query` — hooks
- `react` / `react-dom` — components
- `zustand` — auth store (consumed via axios interceptor; not directly used in this story)
- `axios` — via `apiClient`
- `react-router-dom` — routing already established

**Do NOT install:**
- `react-hook-form`, `zod`, or any form validation library — form fields are Story 2.3
- Any modal library — no modals in v1 (UX rule A6)
- `maplibre-gl` — Epic 8 only
- Any charting library — no charts in v1

---

### Architecture Compliance Guardrails

1. **Signal Desk is default home.** This story does not touch routing. Route `/` still redirects to `/desk`. Do not modify `src/router.tsx` beyond what is necessary.

2. **`WatchConfigPage` renders inside CockpitShell `<Outlet>`.** Never render a full-viewport `position: fixed` wrapper inside `WatchConfigPage` — the shell's `<main>` is already fixed. Use `height: 100%` relative to the shell's fixed bounds.

3. **`apiClient` — no baseURL.** All fetch calls use relative `/api/v1/...` paths. Do NOT hardcode `http://localhost:8000`.

4. **`withCredentials: true` is already set** on `apiClient`. The watches and geography calls will include the httpOnly refresh cookie automatically. Do not re-configure this.

5. **No localStorage or sessionStorage.** Watch selection state lives in React `useState` only — it does not persist across page refreshes (intentional for this story's scope).

6. **No modals.** The `[+ New]` button transitions the form panel inline — no modal overlay, no Dialog component.

7. **INACTIVE row dimming is an approved inclusion** in this story (trivial, stable). All other status badge treatments (colored left stripes, badge pills, baseline progress bar) are Story 2.6.

8. **`rightSlot` in `WatchFormPanel` must default to `null`** — not undefined, not an empty Fragment. This ensures Story 2.5 can pass a button node cleanly without TypeScript issues.

9. **`badge` prop in `WatchListRow` must default to `undefined`** (not passed from `WatchListColumn` in this story). The fallback plain-text status rendering is the 2.2 behavior. Story 2.6 passes the badge node.

---

### Carry-Forward Intelligence from Stories 1.8 and 2.1

| Item | Impact on Story 2.2 |
|---|---|
| CockpitShell `<main>` is `position: fixed; top: 36; left: 120; right: 0; bottom: 0; overflowY: auto` | `WatchConfigPage` must use `height: 100%; overflow: hidden` at top level to prevent double-scroll |
| `apiClient` has no `baseURL`; uses relative `/api/v1/...` paths via Vite proxy | All watch/geography API calls use `/api/v1/watches` and `/api/v1/geography/entities` — never `http://localhost:8000/...` |
| `withCredentials: true` on `apiClient` + 401 refresh interceptor established | Watch list API calls automatically get auth headers + refresh on 401; no custom auth logic needed |
| `QueryClientProvider` already wraps the app via `App.tsx` | `useWatches()` and `useGeographyEntities()` hooks work immediately; no provider setup needed |
| `WatchConfigPage.tsx` exists as a placeholder — replace (not create) | Use Edit tool; file exists at `src/pages/WatchConfigPage.tsx` |
| `src/components/watch-config/.gitkeep` exists | New components go in this directory; do NOT delete the `.gitkeep` (or replace it with the new files, leaving the `.gitkeep` as a sibling is fine) |
| CSS token system locked (`src/styles/tokens.css`) | Reference all colors via `var(--token-name)` — `--bg-surface`, `--bg-base`, `--bg-raised`, `--border-subtle`, `--text-primary`, `--text-secondary`, `--text-tertiary`, `--accent-attention` |
| Startup: repo root → `uvicorn services.api.api.main:app --reload --host 0.0.0.0 --port 8000 --env-file .env` | Smoke test uses this path (confirmed by Nikku 2026-03-10 for Stories 2.1+) |
| `GET /api/v1/watches` and `GET /api/v1/geography/entities` are live and tested | No API changes needed in this story |

---

### Project Structure Notes

**What Story 2.2 establishes for Stories 2.3–2.6:**
- Two-column layout and column dimensions — locked; Stories 2.3–2.6 add content to the columns, never change their structure
- `WatchFormPanel` component + `rightSlot` prop pattern — 2.5 adds Disable button here
- `WatchListRow` component + `badge` prop pattern — 2.6 adds status badge here
- `useWatches()` / `useGeographyEntities()` query hooks — 2.3 uses the same cached data for the form dropdowns
- Column scroll architecture (each column independent) — 2.3 relies on this for sticky form header + scrollable body

**Files stories MUST NOT modify after 2.2 establishes them:**
- `src/layouts/CockpitShell.tsx` — never modify in feature stories
- `src/styles/tokens.css` — color tokens locked; changes require explicit UX approval
- `src/lib/api-client.ts` — interceptor chain is established; add capabilities only, never remove or reorder

---

### References

- [Source: _bmad-output/planning-artifacts/ux-design-specification.md#§8.1] — Watch Configuration layout: `[120px rail | 320px watch list | ~1000px form area]`
- [Source: _bmad-output/planning-artifacts/ux-design-specification.md#§8.2] — Watch list column: header structure, Domain Pulse position, row format
- [Source: _bmad-output/planning-artifacts/ux-design-specification.md#§8.3] — Watch form: sticky header, watch name, rightSlot for Disable (Story 2.5)
- [Source: _bmad-output/planning-artifacts/ux-design-specification.md#§10.3] — Domain Pulse dot: 6px in Watch Config header, grey in this story
- [Source: _bmad-output/planning-artifacts/ux-design-specification.md#§11.5] — Global Watch Config interaction rules
- [Source: _bmad-output/planning-artifacts/ux-design-delta-palantir-inspire-2026-03-10.md#B1] — Watch Config: `[120px rail | 320px watch list | ~1000px form area]` (120px rail confirmed)
- [Source: _bmad-output/planning-artifacts/ux-design-delta-palantir-inspire-2026-03-10.md#A6] — No modals anywhere in v1
- [Source: _bmad-output/planning-artifacts/epics.md#Epic 2] — Story 2.2 scope: "Watch Configuration surface: two-column layout (watch list left, form right per UX spec §8)"
- [Source: _bmad-output/implementation-artifacts/1-8-cockpit-shell.md#CockpitShell layout] — `<main>` slot dimensions; CSS token set; `NavLink` active state
- [Source: _bmad-output/implementation-artifacts/1-8-cockpit-shell.md#package.json] — `@tanstack/react-query ^5.56.2` already installed; no new dependencies
- [Source: _bmad-output/implementation-artifacts/2-1-watches-api-crud.md#AC#1] — `GET /api/v1/watches` response envelope: `{ data: WatchRow[], meta, error }` ordered by `created_at` desc
- [Source: _bmad-output/implementation-artifacts/2-1-watches-api-crud.md#AC#7] — `GET /api/v1/geography/entities` response: active only, ordered by `name` asc, no geometry

---

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6 (Amelia — Dev Agent)

### Debug Log References

- `npm run build` passed clean: 153 modules transformed, 0 TypeScript errors, 0 Vite build errors. Build time: 1.09s.

### Completion Notes List

1. All 6 files created/replaced as specified. No files outside the story boundary were touched.
2. `useWatches` and `useGeographyEntities` hooks follow the exact patterns from Dev Notes including `staleTime: 5 * 60 * 1000` on entities.
3. `WatchListRow` includes the `badge?: React.ReactNode` extension seam (default `undefined`) with inline comment directing Story 2.6 on how to use it.
4. `WatchFormPanel` includes the `rightSlot?: React.ReactNode` extension seam (default `null`) with inline comment directing Story 2.5.
5. Entity name lookup includes a safe fallback: if the entity ID is not found in the map (entity not returned by API), the raw `geography_entity_id` UUID is displayed rather than crashing.
6. `watchesError` is cast as `Error | null` when passed to `WatchListColumn` — TanStack Query v5 types `error` as `Error | null` which is compatible.
7. AC #10 (nav rail active state) requires no new code — existing `NavLink` in `NavRail.tsx` handles it automatically.
8. AC #11 (auth) requires no new code — `apiClient` interceptor chain from Story 1.8 handles it automatically.
9. No new npm packages installed. All dependencies from Story 1.8 used as-is.
10. Story 5.2–5.6 subtasks left unchecked pending Nikku manual smoke test.

### File List

- `services/cockpit/src/hooks/useWatches.ts` — CREATED
- `services/cockpit/src/hooks/useGeographyEntities.ts` — CREATED
- `services/cockpit/src/components/watch-config/WatchListRow.tsx` — CREATED
- `services/cockpit/src/components/watch-config/WatchListColumn.tsx` — CREATED
- `services/cockpit/src/components/watch-config/WatchFormPanel.tsx` — CREATED
- `services/cockpit/src/pages/WatchConfigPage.tsx` — REPLACED (stub → full implementation)

## Change Log

- 2026-03-10: Story 2.2 created by Bob (SM, claude-sonnet-4-6). Prerequisites confirmed: 1.1, 1.7, 1.8, 2.1 all done. Story boundary reviewed against 2.3/2.4/2.5/2.6 with explicit callouts. Status: ready-for-dev.
- 2026-03-10: Story 2.2 implemented by Amelia (Dev, claude-sonnet-4-6). All tasks complete. `npm run build` passes clean. 6 files created/replaced. Pending: Nikku manual smoke test (AC #13, tasks 5.2–5.6). Status: review.
- 2026-03-10: Story 2.2 code review remediation applied (Amelia, claude-sonnet-4-6). M1/L1/L2 patched. `npm run build` passes clean (153 modules, 0 errors). Remaining open item: Nikku manual smoke test (AC #13, tasks 5.2–5.6). All code-review findings closed.
- 2026-03-10: Story 2.2 closed (Nikku). Manual smoke test passed locally. Login, /watches route, two-column layout, watch row selection (amber left border + bg-raised), [+ New] / deselect / reselect cycle, inactive row dimming — all verified clean. No layout or overflow issues. Status: done.
