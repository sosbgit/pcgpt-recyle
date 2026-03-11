# Story 2.6: Watch List Visual Richness

Status: ready-for-dev

## Dependency Note — Local-First Route

Stories 1.4 (VPS provisioning), 1.5 (production nginx), and 1.6 (production Docker Compose) remain **intentionally deferred** — production deployment, no bearing on local development.

**Prerequisites satisfied:**
- 1.1 (monorepo scaffold) — done ✓
- 1.2 (Alembic initial migration) — done ✓ — `watches`, `geography_entities` tables, 11 seeded entities
- 1.3 (shared schema package) — done ✓ — `WatchResponse`, `SignalFamilyEnum` live
- 1.7 (FastAPI app skeleton) — done ✓
- 1.8 (cockpit shell) — done ✓ — CSS token system, axios client, TanStack Query provider established
- 2.1 (Watches API CRUD) — done ✓ — watch status field live and returned on all list/detail responses
- 2.2 (Watch Configuration surface) — done ✓ — `WatchListRow.tsx` with `badge` seam; `WatchListColumn.tsx` established
- 2.3 (Watch form) — done ✓
- 2.4 (Compatible vs. incompatible change detection) — done ✓
- 2.5 (Disable vs. Delete hierarchy) — done ✓ — `is_active` transitions live; inactive row dimming confirmed

---

## Story

As an operator,
I want each watch row in the Watch Configuration list to communicate its current state at a glance — through a colored left edge stripe, a status badge, and a baseline progress indicator — so that I can immediately assess the health of all my watch conditions without opening each one.

---

## Precise Scope

### This Story IS:
- Replacing the plain status text fallback in `WatchListRow.tsx` with a **colored status badge** (`ACTIVE`, `NOMINAL`, `BASELINE`, `DEGRADED`, `INACTIVE`) per the UX spec §8.2 badge table
- Replacing the selection-driven amber left stripe with a **status-driven left edge stripe** — the stripe color communicates watch state, not selection state
- Adding selection feedback via **background change only** (`--bg-raised` when selected) — the amber stripe is retired
- Adding a **baseline progress bar** (80px wide, 4px high, proportionally filled) for watches in `baseline_in_progress` or `created` status — shown beneath the badge row, filled at `0 / baseline_window_days` until Epic 4 delivers real elapsed data
- Rendering **richer secondary metadata**: entity name + status badge on the secondary line; progress bar on a third line for baseline-state watches
- **Frontend only** — no backend changes, no API changes, no new npm packages
- **One file only**: `WatchListRow.tsx`

### This Story IS NOT:
- Any changes to `WatchListColumn.tsx`, `WatchConfigPage.tsx`, `WatchFormPanel.tsx`, or any hook files
- Backend changes, schema changes, or migration changes
- Returning signal counts per watch from the API (signals don't exist yet; this is Epic 5 / Story 5.7)
- Displaying real baseline elapsed days (the progress bar renders at `0/baseline_window_days` until Epic 4 / Story 4.3 wires up baseline computation and the API returns elapsed days)
- Any new npm packages
- Any changes to the UX token system (`tokens.css`) — all required tokens (`--accent-monitoring`, `--accent-nominal`, `--accent-baseline`, `--accent-degraded`) are already defined from Story 1.8
- Spatial View, Signal Desk, Evidence Surface, or any other cockpit surface
- Any routing changes

---

## SM Ambiguity Clarifications

### §1 — Status-Driven Stripe Replaces Selection-Amber Stripe

Story 2.2 used `3px solid --accent-attention` (amber) as the selected-row indicator, and `3px solid transparent` for unselected. This was a placeholder — the UX spec (§8.2) defines a richer system where the left stripe communicates **watch status**, not selection.

**Resolution:** The stripe color is now derived from watch status, always applied (or transparent for INACTIVE). Selection is communicated via **background only**: `--bg-raised` when selected, transparent when not.

The amber stripe was the 2.2 scaffold. Story 2.6 retires it. Do not preserve the amber stripe in any state.

### §2 — Baseline Progress Bar: Placeholder at 0/n Until Epic 4

The UX spec (§8.2, §9.2) shows `████████░░░  18/30d` — a filled progress bar with elapsed/target days. Rendering this accurately requires `baseline_days_elapsed` from the API, which in turn requires the processor service (Epic 4 / Story 4.3) to track when baseline collection started.

Neither the processor service nor a `baseline_days_elapsed` field on `WatchResponse` exists yet.

**Resolution:** Implement the progress bar component at full visual fidelity, using `0 / watch.baseline_window_days` as the fraction. The bar renders as an empty track (0% fill). The `{elapsed}/{total}d` label renders as `0/{baseline_window_days}d`. When Epic 4 adds `baseline_days_elapsed` to the API response and the dev updates `WatchRow` interface, the fraction will auto-populate with real data — no additional visual changes needed.

This is not a hack — it correctly expresses the system state: a freshly created watch has zero days of baseline collected. The visual slot is correct and ready.

### §3 — Signal Count Deferred to Epic 5

The UX spec wireframe shows `ACTIVE  ·  2 signals` in the secondary row. This requires a per-watch active signal count from the signals API, which does not exist until Epic 5 / Story 5.6.

**Resolution:** Do not add signal count to the secondary row in Story 2.6. The secondary row renders: `{entityName}  ·  <StatusBadge>`. When Epic 5 introduces the signals API, the watch list will be updated to include signal count at that time.

Do not add a "0 signals" placeholder. A zero signal count is ambiguous before the signals system exists — it could mean "no signals yet generated" or "truly NOMINAL." It is not surfaced until it carries real meaning.

### §4 — Badge Prop Seam: Retired

`WatchListRow` has a `badge?: ReactNode` prop reserved in Story 2.2 with this comment: "Reserved for Story 2.6: pass `<StatusBadge status={watch.status} />` to replace plain status text."

**Resolution:** Story 2.6 renders the status badge **internally** within `WatchListRow`, not externally via the `badge` prop. The prop is retired. Remove it from the interface. `WatchListColumn.tsx` never used it, so no call site needs updating.

### §5 — WatchListColumn.tsx: No Changes Required

`WatchListColumn.tsx` renders `WatchListRow` components but never passed the `badge` prop. All visual richness in Story 2.6 is internal to `WatchListRow`. `WatchListColumn.tsx` is unchanged.

### §6 — CREATED Status Treatment

`WatchRow.status` can be `"created"` — the initial state after a watch is first saved, before the processor has run. The UX spec does not define a separate visual for CREATED; it collapses into the baseline-in-progress pattern.

**Resolution:** Treat `"created"` the same as `"baseline_in_progress"` for all visual purposes: blue left stripe (`--accent-baseline`), `BASELINE` badge, empty progress bar (`0/{baseline_window_days}d`). The operator reads this as: the system is preparing to collect, not yet monitoring.

### §7 — Selection Indicator: Background Change Only

With status-driven stripes, the selection feedback mechanism changes. The amber left border (`--accent-attention`) was the 2.2 selection signal. It is retired.

**Resolution:** Selection = `--bg-raised` background. No stripe color change on selection. The status stripe remains the same whether the row is selected or not. This is consistent with UX spec §8.2 which describes "Active row (selected watch): `▐` left accent bar, `--bg-raised` background" — the accent bar is the status bar, not a selection-specific amber bar.

### §8 — Padding Compensation: Always 13px

Current code: `paddingLeft: isSelected ? 13 : 16` compensates for the 3px left border only when selected.

With 2.6, the 3px left border is always present (even if `transparent` for INACTIVE). Use `paddingLeft: 13` always for consistent left-edge alignment. The `padding` shorthand `'10px 12px 10px 13px'` can replace the current multi-property approach.

---

## Acceptance Criteria

1. **Left stripe — ACTIVE watch:** A watch with `status === "active"` shows `3px solid var(--accent-monitoring)` left stripe whether selected or not. No amber stripe.

2. **Left stripe — NOMINAL watch:** A watch with `status === "nominal"` shows `3px solid var(--accent-nominal)` left stripe.

3. **Left stripe — BASELINE watch (baseline_in_progress or created):** A watch with `status === "baseline_in_progress"` or `"created"` shows `3px solid var(--accent-baseline)` left stripe.

4. **Left stripe — DEGRADED watch:** A watch with `status === "degraded"` shows `3px solid var(--accent-degraded)` left stripe.

5. **Left stripe — INACTIVE watch:** A watch with `is_active === false` shows `3px solid transparent` left stripe. Row text is dimmed to `--text-tertiary` per existing Story 2.5 behavior. No stripe visible.

6. **Selection feedback — background only:** Selecting any row applies `--bg-raised` background. The left stripe does not change color on selection. No amber stripe anywhere.

7. **Status badge — ACTIVE:** Secondary row shows `{entityName}  ·  ` followed by `ACTIVE` in `--accent-monitoring` color, 10px uppercase. No background fill on the badge — colored text only.

8. **Status badge — NOMINAL:** Secondary row shows `NOMINAL` in `--accent-nominal` color, 10px uppercase.

9. **Status badge — BASELINE:** Secondary row shows `BASELINE` in `--accent-baseline` color, 10px uppercase. A baseline progress bar and fraction are rendered on a third line (see AC #10–11).

10. **Baseline progress bar — visual spec:**
    ```
    [░░░░░░░░░░░░]  0/30d
    ```
    - Width: 80px
    - Height: 4px
    - Border-radius: 2px
    - Track background: `var(--border-subtle)`
    - Fill: `var(--accent-baseline)`
    - Fill width: `clamp(0, (baseline_days_elapsed / baseline_window_days), 1) * 80px`
    - For Story 2.6: `baseline_days_elapsed` is `0` (no field yet from API) — bar renders empty
    - Displayed inline with the fraction label; the bar and label are rendered on a new line below the badge row

11. **Baseline progress fraction:**
    - Adjacent to the bar: `{baseline_days_elapsed}/{watch.baseline_window_days}d`
    - For Story 2.6: always `0/{watch.baseline_window_days}d`
    - Font: 10px, `--text-tertiary`
    - Monospace preferred (IBM Plex Mono if available via existing font stack, else inherit)

12. **Status badge — DEGRADED:** Secondary row shows `DEGRADED` in `--accent-degraded` color, 10px uppercase.

13. **Status badge — INACTIVE:** No status badge. Row is fully dimmed (`--text-tertiary`). Existing 2.5 dimming behavior is preserved and unchanged.

14. **Status badge — CREATED:** Treated identically to `baseline_in_progress`: `BASELINE` badge in `--accent-baseline`, with progress bar at `0/{baseline_window_days}d`.

15. **Row height — non-baseline rows:** Visual height is consistent with Story 2.2 compact treatment (10px top + 10px bottom padding). Two-line content (name + secondary) renders at existing height.

16. **Row height — baseline/created rows:** Three-line content (name + badge line + progress line). Taller row is acceptable and expected — the progress bar row adds height naturally. No fixed-height override.

17. **Inactive dimming preserved:** The Story 2.5 dimming behavior (`is_active === false` → `--text-tertiary` for all text) is unchanged. The only interaction between 2.5 and 2.6 is: INACTIVE rows get no left stripe and no status badge (per AC #5 and #13).

18. **No regression — existing watch list behaviors:**
    - `[+ New]` button still works
    - Row click still triggers `onSelectWatch`
    - Loading state still renders correctly
    - Error state still renders correctly
    - Empty state still renders correctly
    - Domain Pulse strip in column header is unchanged

19. **`npm run build` passes:** `tsc -b && vite build` completes without TypeScript errors after all changes.

20. **Smoke test passes locally:**
    ```bash
    # Terminal 1: API + infra (from repo root)
    docker compose -f docker-compose.yml -f docker-compose.dev.yml up postgres valkey -d
    uvicorn services.api.api.main:app --reload --host 0.0.0.0 --port 8000 --env-file .env

    # Terminal 2: Cockpit (from repo root)
    cd services/cockpit
    npm run dev
    # → Login → navigate to /watches
    # → Confirm all existing watches show their status badge (colored text label)
    # → Confirm no amber left stripe on any row
    # → Confirm selected row shows --bg-raised background, status stripe unchanged on selection
    # → Disable a watch (from form) → confirm INACTIVE row: no stripe, all text dimmed, no badge
    # → Re-enable the watch → confirm stripe and badge return
    # → If any watch is in "created" status → confirm BASELINE badge + empty progress bar
    # → npm run build → no TypeScript or Vite build errors
    ```

---

## Non-Goals (Explicit Scope Boundaries)

| Item | Deferred To |
|---|---|
| Real baseline elapsed days in progress bar | Epic 4 / Story 4.3 — processor baseline FSM provides elapsed data |
| Signal count in watch row secondary text | Epic 5 / Story 5.7 — signals API required |
| Any API or backend changes | Not in this story |
| Per-watch signal count badge or alert count | Epic 5 |
| Domain Pulse dot color changes (still grey) | Epic 3 |
| New watch status states beyond current schema | Epic 4 / Epic 5 |
| Hover animation, transition effects beyond 100ms | Not needed; hover state is immediate |
| `WatchListColumn.tsx` changes | Not required |
| `WatchConfigPage.tsx` changes | Not required |
| `WatchFormPanel.tsx` changes | Not required |
| New npm packages | Not required |
| Entity thumbnail in watch row | Not in scope for any Epic 2 story |

---

## Implementation Notes

### File Structure — What to Modify

```
services/cockpit/src/
└── components/
    └── watch-config/
        └── WatchListRow.tsx   ← MODIFY ONLY — this is the only file that changes
```

**Files NOT touched:**
- `WatchListColumn.tsx` — no changes
- `WatchConfigPage.tsx` — no changes
- `WatchFormPanel.tsx` — no changes (2.4 banners, 2.5 lifecycle controls: all untouched)
- `useWatches.ts`, `useGeographyEntities.ts`, `useCreateWatch.ts`, `useUpdateWatch.ts`, `useDeleteWatch.ts` — no changes
- `src/layouts/CockpitShell.tsx` — never modify
- `src/styles/tokens.css` — never modify
- `src/lib/api-client.ts` — never modify
- `services/api/` — no backend changes
- `migrations/` — no schema changes

---

### Status Color Mapping

Define a helper inside the module (above the component function):

```typescript
type WatchStatus = 'created' | 'baseline_in_progress' | 'active' | 'nominal' | 'degraded' | 'inactive'

function getStatusStripeColor(status: string, isActive: boolean): string {
  if (!isActive) return 'transparent'
  switch (status as WatchStatus) {
    case 'active':               return 'var(--accent-monitoring)'
    case 'nominal':              return 'var(--accent-nominal)'
    case 'baseline_in_progress': return 'var(--accent-baseline)'
    case 'created':              return 'var(--accent-baseline)'
    case 'degraded':             return 'var(--accent-degraded)'
    default:                     return 'transparent'
  }
}

function getStatusBadgeColor(status: string): string {
  switch (status as WatchStatus) {
    case 'active':               return 'var(--accent-monitoring)'
    case 'nominal':              return 'var(--accent-nominal)'
    case 'baseline_in_progress': return 'var(--accent-baseline)'
    case 'created':              return 'var(--accent-baseline)'
    case 'degraded':             return 'var(--accent-degraded)'
    default:                     return 'var(--text-tertiary)'
  }
}

function getStatusLabel(status: string): string {
  switch (status as WatchStatus) {
    case 'active':               return 'ACTIVE'
    case 'nominal':              return 'NOMINAL'
    case 'baseline_in_progress': return 'BASELINE'
    case 'created':              return 'BASELINE'
    case 'degraded':             return 'DEGRADED'
    case 'inactive':             return ''
    default:                     return status.replace(/_/g, ' ').toUpperCase()
  }
}
```

These are pure functions — no React hooks, no side effects. Define them at module scope above the component.

---

### Baseline Progress Bar

The progress bar renders only for `baseline_in_progress` and `created` status watches.

```typescript
// Baseline elapsed days — Story 2.6 placeholder: 0 until Epic 4 provides API field
const BASELINE_DAYS_ELAPSED = 0

function BaselineProgressBar({ windowDays }: { windowDays: number }) {
  const fraction = Math.min(BASELINE_DAYS_ELAPSED / Math.max(windowDays, 1), 1)
  return (
    <div style={{ display: 'flex', alignItems: 'center', gap: 6, marginTop: 4 }}>
      <div
        style={{
          width: 80,
          height: 4,
          borderRadius: 2,
          background: 'var(--border-subtle)',
          overflow: 'hidden',
        }}
      >
        <div
          style={{
            width: `${fraction * 100}%`,
            height: '100%',
            background: 'var(--accent-baseline)',
          }}
        />
      </div>
      <span
        style={{
          fontSize: 10,
          color: 'var(--text-tertiary)',
          fontVariantNumeric: 'tabular-nums',
        }}
      >
        {BASELINE_DAYS_ELAPSED}/{windowDays}d
      </span>
    </div>
  )
}
```

`BASELINE_DAYS_ELAPSED` is a module-level constant set to `0`. When Epic 4 wires up `baseline_days_elapsed` on the API response:
1. Add `baseline_days_elapsed?: number` to the `WatchRow` interface in `useWatches.ts`
2. Replace `BASELINE_DAYS_ELAPSED` constant with `watch.baseline_days_elapsed ?? 0` in the component
3. No visual changes needed — the component already handles the full 0–100% range

---

### Updated WatchListRow Component — Full Structure

```typescript
import type { ReactNode } from 'react'
import { WatchRow } from '@/hooks/useWatches'

interface WatchListRowProps {
  watch: WatchRow
  entityName: string
  isSelected: boolean
  onClick: () => void
}
```

Note: `badge?: ReactNode` prop removed. It was the 2.2 seam; Story 2.6 renders status internally.

Row layout:
- `padding: '10px 12px 10px 13px'` — always 13px left (compensates for the always-present 3px border)
- `borderLeft: '3px solid {stripeColor}'` — always present, transparent for INACTIVE
- `background: isSelected ? 'var(--bg-raised)' : 'transparent'` — selection signal only

Secondary row structure:
```tsx
{/* Secondary row: entity name · status badge */}
<div style={{ display: 'flex', alignItems: 'center', gap: 4, flexWrap: 'wrap' }}>
  <span style={{ fontSize: 11, color: subColor }}>{entityName}</span>
  {statusLabel && (
    <>
      <span style={{ fontSize: 11, color: 'var(--text-tertiary)' }}>·</span>
      <span
        style={{
          fontSize: 10,
          color: badgeColor,
          textTransform: 'uppercase',
          letterSpacing: '0.06em',
          fontWeight: 500,
        }}
      >
        {statusLabel}
      </span>
    </>
  )}
</div>

{/* Baseline progress bar — only for baseline/created status */}
{isBaseline && <BaselineProgressBar windowDays={watch.baseline_window_days} />}
```

Where:
- `isBaseline = watch.status === 'baseline_in_progress' || watch.status === 'created'`
- `statusLabel = getStatusLabel(watch.status)` — empty string for INACTIVE (renders nothing)
- `badgeColor = getStatusBadgeColor(watch.status)`
- `stripeColor = getStatusStripeColor(watch.status, watch.is_active)`
- `isInactive = !watch.is_active` (unchanged from 2.5)
- `textColor = isInactive ? 'var(--text-tertiary)' : 'var(--text-primary)'` (unchanged)
- `subColor = isInactive ? 'var(--text-tertiary)' : 'var(--text-secondary)'` (unchanged)

---

### Architecture Guardrails

1. **Signal Desk is default home.** This story does not touch routing. Do not modify `src/router.tsx`.
2. **No new npm packages.** All layout and color is inline styles using existing CSS tokens.
3. **CSS tokens only.** All colors via `var(--*)`. No hardcoded hex values.
4. **No new files.** One file modified: `WatchListRow.tsx`.
5. **No backend changes.** `WatchRow` interface in `useWatches.ts` is unchanged.
6. **No modals.** Status badges and progress bars are inline elements in the row body.
7. **Amber stripe is retired.** No use of `--accent-attention` in `WatchListRow.tsx` for left border.
8. **Inactive behavior preserved from 2.5.** `is_active === false` → dimmed text, no stripe, no badge.

---

## Testing Expectations

This story is frontend-only. No backend test changes.

**Manual smoke test:** AC #20 defines the full scenario. Verified by Nikku per the established pattern.

**Unit tests:** Not required for this story (no React component test suite established). Optional.

**Build verification:** `tsc -b && vite build` must pass clean (0 errors). Same standard as prior stories.

**Regression checks:**
- Existing lifecycle controls (Disable/Enable/Delete from Story 2.5) remain visible and functional in WatchFormPanel — these are in a different file and are untouched
- Incompatible change banners from Story 2.4 remain functional — different file, untouched
- Watch form still opens on row click
- `[+ New]` button still opens new-watch form
- Loading/error/empty states in WatchListColumn render correctly (unchanged)

---

## Dependencies and Prerequisites

| Dependency | Status | Notes |
|---|---|---|
| Story 2.5 (Disable/Delete hierarchy) | Done ✓ | `is_active` field on WatchRow; inactive dimming behavior established |
| Story 2.2 (Watch Configuration surface) | Done ✓ | `WatchListRow.tsx` with `badge` seam; this story replaces that seam with internal richness |
| `watch.status` field on WatchRow | Done ✓ (Story 2.1) | Returns `"created"`, `"baseline_in_progress"`, `"active"`, `"nominal"`, `"degraded"`, `"inactive"` |
| CSS token `--accent-monitoring` | Done ✓ (Story 1.8) | Steel color for ACTIVE stripe and badge |
| CSS token `--accent-nominal` | Done ✓ (Story 1.8) | Green color for NOMINAL |
| CSS token `--accent-baseline` | Done ✓ (Story 1.8) | Blue color for BASELINE |
| CSS token `--accent-degraded` | Done ✓ (Story 1.8) | Amber color for DEGRADED (same value as `--accent-attention`) |
| CSS token `--border-subtle` | Done ✓ (Story 1.8) | Progress bar track background |

---

## Carry-Forward Intelligence from Stories 2.2–2.5

| Item | Impact on Story 2.6 |
|---|---|
| `WatchListRow` uses plain inline styles (no Tailwind, no CSS modules) | All new styles follow the same pattern — `style={{}}` objects on JSX elements |
| `isInactive` dimming logic established in 2.5 | Preserved unchanged. `is_active === false` → `--text-tertiary`. No stripe. No badge. |
| `badge?: ReactNode` seam from 2.2 | Retired. Badge rendered internally. Prop removed. No call site change needed (WatchListColumn never passed it). |
| `borderLeft` selection-amber from 2.2 | Replaced entirely. The amber stripe belongs to the signal-desk era UX, not watch-config state. |
| `paddingLeft` compensation pattern from 2.2 | Keep compensation. Change from conditional (`isSelected ? 13 : 16`) to always `13px` — 3px border always present. |
| `WatchRow.status` field comment in `useWatches.ts` | Status values are: `"created" | "baseline_in_progress" | "active" | "nominal" | "degraded" | "inactive"` — all handled in switch statements |
| Story 2.4 introduced `useMemo` in `WatchFormPanel.tsx` | Irrelevant to 2.6 — different file |
| Story 2.5 `data-slot="delete-zone"` and `rightSlot` in WatchFormPanel | Irrelevant to 2.6 — different file |

---

## References

- [UX spec §8.2 — Watch List Column](../planning-artifacts/ux-design-specification.md) — Status badge table, left stripe per status, baseline row wireframe, Domain Pulse strip
- [UX spec §9.2 — Baseline-in-Progress State](../planning-artifacts/ux-design-specification.md) — `⧖` row pattern, progress bar fill, `18/30d` fraction format
- [UX spec §3.1 — Color System](../planning-artifacts/ux-design-specification.md) — `--accent-monitoring`, `--accent-nominal`, `--accent-baseline`, `--accent-degraded` token definitions
- [UX spec §3.3 — Density & Spacing](../planning-artifacts/ux-design-specification.md) — Watch list row: 10px top + 10px bottom, 16px horizontal
- [Epics §Epic 2 — Watch Configuration](../planning-artifacts/epics.md) — "Watch list: status badges, empty state, active/disabled visual distinction"
- [Epics FR10](../planning-artifacts/epics.md) — "Operator can view all active watch conditions and their current status from a single surface"
- [Story 2.2 Completion Notes](./2-2-watch-configuration-surface.md) — `badge` prop seam, inactive dimming, `borderLeft` selection indicator
- [Story 2.5 Completion Notes](./2-5-disable-vs-delete-hierarchy.md) — `is_active` field, INACTIVE dimming confirmed via smoke test; WatchListRow/Column explicitly noted as untouched
- [Current WatchListRow.tsx](../../../../services/cockpit/src/components/watch-config/WatchListRow.tsx) — existing structure, `badge` prop, `isInactive` dimming, `borderLeft` logic to be replaced
- [Current tokens.css](../../../../services/cockpit/src/styles/tokens.css) — all accent tokens confirmed present

---

## Tasks / Subtasks

- [ ] Task 1: Define status helper functions at module scope (AC: #1–#6, #7–#14)
  - [ ] 1.1 Add `getStatusStripeColor(status, isActive)` — returns CSS var string; transparent for INACTIVE and unknown
  - [ ] 1.2 Add `getStatusBadgeColor(status)` — returns CSS var string for badge text color
  - [ ] 1.3 Add `getStatusLabel(status)` — returns uppercase display string; empty string for INACTIVE

- [ ] Task 2: Add `BaselineProgressBar` component inline (AC: #10, #11)
  - [ ] 2.1 Implement `BASELINE_DAYS_ELAPSED = 0` constant (documented as placeholder for Epic 4)
  - [ ] 2.2 Implement `BaselineProgressBar({ windowDays })` — 80px track, 4px height, `--accent-baseline` fill, `--border-subtle` track, `0/{windowDays}d` label at 10px monospace `--text-tertiary`

- [ ] Task 3: Update `WatchListRowProps` interface (AC: #19 — TypeScript build)
  - [ ] 3.1 Remove `badge?: ReactNode` prop (2.2 seam retired)
  - [ ] 3.2 Remove `ReactNode` import (no longer needed after badge removal)

- [ ] Task 4: Update `WatchListRow` component body (AC: #1–#18)
  - [ ] 4.1 Compute derived values: `stripeColor`, `statusLabel`, `badgeColor`, `isBaseline`, `isInactive`, `textColor`, `subColor`
  - [ ] 4.2 Replace `borderLeft` logic: `'3px solid {stripeColor}'` always (no selection conditional on stripe)
  - [ ] 4.3 Replace `paddingLeft` logic: always `13px` (drop the `isSelected ? 13 : 16` conditional)
  - [ ] 4.4 Add `background: isSelected ? 'var(--bg-raised)' : 'transparent'` — selection via background only (unchanged semantically, same as before)
  - [ ] 4.5 Replace badge/status fallback block: render entity name + optional `·` separator + colored `statusLabel` span; skip badge block entirely when `statusLabel` is empty (INACTIVE)
  - [ ] 4.6 Add `{isBaseline && <BaselineProgressBar windowDays={watch.baseline_window_days} />}` below secondary row

- [ ] Task 5: TypeScript compile + smoke test (AC: #19, #20)
  - [ ] 5.1 `npm run build` — 0 TypeScript errors ✓
  - [ ] 5.2 Start infra + API + cockpit dev server; navigate to `/watches`
  - [ ] 5.3 Verify all watch rows show status badges (colored text)
  - [ ] 5.4 Verify no amber left stripe anywhere; stripe color matches watch status
  - [ ] 5.5 Verify selected row: stripe unchanged, background shifts to `--bg-raised`
  - [ ] 5.6 Verify INACTIVE row: no stripe, no badge, all text dimmed to `--text-tertiary`
  - [ ] 5.7 Verify BASELINE/CREATED row (if any watch in that status): `BASELINE` badge + empty `0/Nd` progress bar
  - [ ] 5.8 Verify existing behaviors unchanged: `[+ New]`, row click, loading/error/empty states

---

## Dev Agent Record

*(to be filled in by dev agent after implementation)*

---

## Completion Notes

*(to be filled in after smoke test)*

---

## File List

| File | Action |
|---|---|
| `services/cockpit/src/components/watch-config/WatchListRow.tsx` | Modified |

---

## Change Log

- 2026-03-11: Story 2.6 created by Bob (SM, claude-sonnet-4-6). Prerequisites confirmed: 1.1, 1.2, 1.3, 1.7, 1.8, 2.1, 2.2, 2.3, 2.4, 2.5 all done. SM ambiguity resolution documented (§1–§8). Status badge table, stripe-vs-selection clarification, baseline progress placeholder, signal count deferral, badge prop retirement all resolved. Status: ready-for-dev.
