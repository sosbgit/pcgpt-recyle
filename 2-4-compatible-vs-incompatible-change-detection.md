# Story 2.4: Compatible vs. Incompatible Change Detection

Status: ready-for-dev

## Dependency Note — Local-First Route

Stories 1.4 (VPS provisioning), 1.5 (production nginx), and 1.6 (production Docker Compose) remain **intentionally deferred** — production deployment, no bearing on local development.

**Prerequisites satisfied:**
- 1.1 (monorepo scaffold) — done ✓
- 1.2 (Alembic initial migration) — done ✓ — `watches`, `geography_entities` tables and 11 seeded entities
- 1.3 (shared schema package) — done ✓ — `WatchCreate`, `WatchUpdate`, `WatchResponse`, `SignalFamilyEnum` all live
- 1.7 (FastAPI app skeleton) — done ✓
- 1.8 (cockpit shell) — done ✓ — CSS token system, axios client, TanStack Query provider established
- 2.1 (Watches API CRUD) — done ✓ — `PATCH /api/v1/watches/{id}` live and tested
- 2.2 (Watch Configuration surface) — done ✓ — Two-column surface, `WatchFormPanel.tsx`, hooks, columns all established
- 2.3 (Watch form) — done ✓ — All real form fields live; `saveLabel` comment in `WatchFormPanel.tsx` explicitly marks the 2.4 integration point

---

## Story

As an operator,
I want the Watch Configuration form to immediately highlight when I make a change that would reset the baseline — and require a distinct acknowledgment action to save that change — so that I never accidentally discard collected baseline data through an ordinary edit.

---

## Precise Scope

### This Story IS:
- Detecting incompatible field changes in **edit mode** (`mode === 'selected'`) only
- Defining exactly which fields are compatible and which are incompatible (see Field Classification below)
- Rendering an inline warning banner directly below any field whose current value diverges from the saved watch value in an incompatible way
- Changing the Save button label from `[Save Changes]` to `[Save & Reset Baseline]` when any incompatible change is detected
- Ensuring the warning banner and button label revert automatically when the operator reverts the incompatible field to its saved value
- Ensuring `[Cancel]` behavior remains correct and clears all warning state
- Modifying **only** `WatchFormPanel.tsx`

### This Story IS NOT:
- Anything in new watch mode (`mode === 'new'`) — no prior state exists to compare against; no warning applies
- Any changes to `WatchListColumn.tsx`, `WatchListRow.tsx`, `WatchConfigPage.tsx`, or any hook files
- Backend changes, schema changes, or migration changes
- Baseline reset backend behavior (state machine: baseline → `in_progress`, old baseline → `superseded`) — that is **Epic 4 / Story 4.3**
- A new API endpoint or new request contract — the same `PATCH /api/v1/watches/{id}` call is used regardless of compatible or incompatible save
- Story 2.5 (Disable / Delete hierarchy) — `data-slot="delete-zone"` placeholder is left untouched
- Story 2.6 (Watch list visual richness) — `WatchListRow` and its `badge` prop are untouched
- Any new npm packages

---

## Field Classification

This table is the authoritative definition for this story. Memorize it.

| Field | Compatible or Incompatible | In Edit Mode? | Notes |
|---|---|---|---|
| `name` | **Compatible** | Yes (editable) | Name has no effect on baseline. No warning. |
| `geography_entity_id` | N/A — **not in edit mode** | No (read-only, locked per 2.3) | Geography is locked after creation. Cannot be changed via the form. No detection needed here. See SM Clarification §1 below. |
| `enabled_signal_families` | **Incompatible** | Yes (checkboxes) | Any add or remove triggers the incompatible path. |
| `escalation_threshold` | **Compatible** | Yes (slider + input) | Explicitly per UX spec §8.4. No warning. |
| `baseline_window_days` | **Incompatible** | Yes (numeric input) | Any change from saved value triggers the incompatible path. |

---

## SM Ambiguity Clarifications

### §1 — Geography Change Is Not Detectable in Edit Mode (and Should Not Be)

The UX spec (§8.4) lists geography entity as an incompatible field. However, Story 2.3 intentionally locked geography to **read-only in edit mode** — the field renders as a plain `<div>` with `(locked)` label, not a `<select>`. The `geography_entity_id` state variable is populated from `watch.geography_entity_id` on form reset and is never modified by the operator in edit mode.

**Resolution:** Geography change detection is a non-issue in edit mode. The UX spec's incompatibility rule for geography applies architecturally (if geography were ever changeable), but since 2.3 made it immutable after creation, Story 2.4 has nothing to detect here. Do not add any geography change logic.

### §2 — Name Edits Are Always Compatible

Changing the watch name does not affect the baseline algorithm, signal detection, or any computed state. It is not listed as incompatible in the UX spec.

**Resolution:** Name edits are always compatible. No warning. No label change. The save proceeds silently.

### §3 — Incompatible Fields in Edit Mode Are Exactly Two

Given the field classification above, the **only fields that can trigger incompatible change detection in edit mode** are:
- `enabled_signal_families` (any checkbox toggle that results in a different set)
- `baseline_window_days` (any value that differs from `watch.baseline_window_days`)

Everything else is either compatible or immutable.

### §4 — Frontend-Only Detection; No API Contract Change

Change detection is **purely client-side**: compare current React state against the `watch` prop values. No API call is needed to determine if a change is incompatible.

**The save action** for incompatible changes uses the **identical `PATCH /api/v1/watches/{id}` call** as for compatible changes. No new fields, no new headers, no new endpoint. The request body difference is simply that the incompatible fields (`enabled_signal_families` or `baseline_window_days`) have changed values in the payload — which is already fully supported by `WatchUpdate` schema.

**Resolution:** Story 2.4 requires zero API contract changes. The backend already accepts `enabled_signal_families` and `baseline_window_days` in PATCH. The `useUpdateWatch.ts` hook is unchanged.

### §5 — Baseline Reset Backend Behavior Is NOT in This Story

The UX spec says: "On `[Save & Reset Baseline]` click: baseline transitions to `in_progress`, old baseline marked `superseded`, event logged."

This backend state machine behavior (baseline FSM transitions, baseline row `superseded` marking, event logging) belongs to **Epic 4 / Story 4.3** (Watch status state machine), when the processor service is built. No processor service exists yet.

**Resolution:** In Story 2.4, `[Save & Reset Baseline]` executes the same PATCH call as `[Save Changes]`. The label and warning communicate intent to the operator. The backend will enforce the actual baseline transition when Epic 4 is implemented. This is correct for the build sequence and creates no technical debt — the PATCH call is valid and the watch fields will update correctly.

### §6 — Warning Appears Per-Field, Not as a Single Global Banner

The UX spec says the warning "appears inline directly below the changed field." This means:
- If only `enabled_signal_families` changed: banner appears below the Signal Families section
- If only `baseline_window_days` changed: banner appears below the Baseline Window section
- If both changed: both banners appear (each below their respective section)

Both warnings use the same text and visual style. The button changes label once regardless of how many incompatible fields are dirty.

### §7 — Warning Clears Automatically; No Manual Dismissal

When the operator reverts an incompatible field back to its saved value, `hasIncompatibleChange` recomputes to `false` and the banner disappears immediately. There is no `[Dismiss]` button on the warning. It is a derived computed state, not user-controlled.

### §8 — Cancel Clears All Warning State Without API Call

`[Cancel]` in edit mode (existing behavior from 2.3) resets all form fields to `watch.*` values. Since this reverts `enabledSignalFamilies` and `baselineWindowDays` to their saved values, `hasIncompatibleChange` recomputes to `false` automatically. No additional cancel logic is needed for 2.4.

### §9 — `[Save & Reset Baseline]` in Relation to `justSaved` Flash

After a successful incompatible save:
1. `onSuccess` is called → `setJustSaved(true)` → label shows `✓ Saved` for 1.5s
2. `queryClient.invalidateQueries({ queryKey: ['watches'] })` runs → `useWatches` refetches
3. The `watch` prop updates with server-confirmed values (which now match the form values)
4. After 1.5s, `setJustSaved(false)` → label logic re-evaluates: since `watch` now reflects the saved values, `hasIncompatibleChange === false` → label returns to `[Save Changes]`

This sequence is correct and requires no special handling. The existing flow works correctly.

### §10 — No Change Detection for New Watch Mode

In `mode === 'new'`, there is no saved watch to compare against. All field values are defaults. The incompatible change concept does not apply because no baseline exists yet for a watch that hasn't been created.

**Resolution:** All detection logic is conditional on `mode === 'selected' && watch !== undefined`. No changes to new-watch behavior.

---

## Acceptance Criteria

1. **No warning in new watch mode:** With `[+ New]` selected, filling in signal families or baseline window shows no warning banner and save button stays labeled `[Save Changes]`.

2. **No warning for compatible edits in edit mode:** Editing watch name only → no banner. Adjusting escalation threshold only → no banner. Save button stays labeled `[Save Changes]`.

3. **Signal family incompatible change detected immediately:** In edit mode, unchecking any signal family checkbox that was previously checked (or re-checking one that was unchecked) triggers immediate display of the incompatible change warning banner below the Signal Families section. No save attempt is required.

4. **Baseline window incompatible change detected immediately:** In edit mode, changing the baseline window numeric input to any value that differs from `watch.baseline_window_days` triggers immediate display of the incompatible change warning banner below the Baseline Window section. Warning appears on field blur (when value is committed) or on every onChange, per the developer's judgment on which feels more responsive — as long as it is visible before the operator can click save.

5. **Warning banner visual spec:**
   ```
   ┌────────────────────────────────────────────────────────────────────────┐
   │  ⚠  Changing this setting will reset the baseline for this watch.      │
   │     Collected data will be discarded.                                  │
   └────────────────────────────────────────────────────────────────────────┘
   ```
   - Amber `⚠` icon (using `--accent-attention` color)
   - 12px body text in `--text-secondary`
   - `--bg-raised` background
   - `3px solid --accent-attention` left border
   - `4px` border radius
   - `8px` vertical padding, `12px` horizontal padding
   - Appears inline directly below the changed field's group (signal families section or baseline window section)

6. **Save button label change:** When `hasIncompatibleChange === true`, the save button label changes from `[Save Changes]` to `[Save & Reset Baseline]`. The button remains primary button style (amber fill). The label change is the only visual difference.

7. **Both warnings when both fields are dirty:** If both `enabled_signal_families` and `baseline_window_days` are changed from saved values, both banners render (each below their respective field group). The button shows `[Save & Reset Baseline]`.

8. **Warning clears on revert:** If the operator reverts a changed field back to its saved value:
   - The banner below that field disappears immediately
   - If no other incompatible field is still dirty, the button reverts to `[Save Changes]`
   - If another incompatible field is still dirty, its banner persists and button label stays `[Save & Reset Baseline]`

9. **Cancel clears all warning state:** Clicking `[Cancel]` in edit mode resets all fields to saved `watch.*` values. All banners disappear. Button reverts to `[Save Changes]`. No API call. Existing cancel behavior is unchanged; 2.4 adds no new cancel logic.

10. **`[Save & Reset Baseline]` submits the same PATCH call:** Clicking `[Save & Reset Baseline]` calls `updateMutation.mutate()` with the same payload structure as `[Save Changes]`. No new endpoint, no new fields, no behavioral difference in the mutation.

11. **`✓ Saved` flash works after incompatible save:** After a successful `[Save & Reset Baseline]`, the `✓ Saved` confirmation appears for 1.5s exactly as it does for compatible saves. After 1.5s, the button returns to `[Save Changes]` (because the refetched `watch` prop now matches form values, so `hasIncompatibleChange === false`).

12. **Geography field is unaffected:** The read-only geography display in edit mode shows no warning, no banner, no label change. No detection logic touches geography.

13. **`isSignalFamiliesEmpty` error still works:** The existing validation (at least one signal family must be enabled) remains active. If all families are unchecked, the save button is disabled and the existing `"At least one signal family must be enabled."` error message renders. This is not a 2.4 concern but must not regress.

14. **`npm run build` passes:** `tsc -b && vite build` completes without TypeScript errors after all changes.

15. **Smoke test passes locally:**
    ```bash
    # Terminal 1: API + infra (from repo root)
    docker compose -f docker-compose.yml -f docker-compose.dev.yml up postgres valkey -d
    uvicorn services.api.api.main:app --reload --host 0.0.0.0 --port 8000 --env-file .env

    # Terminal 2: Cockpit (from repo root)
    cd services/cockpit
    npm run dev
    # → Login → navigate to /watches
    # → Select an existing watch
    # → Uncheck one signal family checkbox
    #   → Warning banner appears below Signal Families section
    #   → Save button label changes to [Save & Reset Baseline]
    # → Re-check the family (revert)
    #   → Warning disappears
    #   → Save button returns to [Save Changes]
    # → Change baseline window value (e.g., 30 → 45)
    #   → Warning banner appears below Baseline Window section
    #   → Save button label changes to [Save & Reset Baseline]
    # → Click [Save & Reset Baseline]
    #   → PATCH call fires (verify in Network tab: /api/v1/watches/{id})
    #   → ✓ Saved appears for ~1.5s
    #   → Warning disappears (server values now match form)
    #   → Button returns to [Save Changes]
    # → Select same watch again (confirm updated values are now the saved defaults)
    # → Edit name only → no warning, save button stays [Save Changes]
    # → Edit threshold only → no warning, save button stays [Save Changes]
    # → Create new watch via [+ New] → no warnings at any point
    # → npm run build → no TypeScript or Vite build errors
    ```

---

## Non-Goals (Explicit Scope Boundaries)

| Item | Deferred To |
|---|---|
| Baseline reset backend state machine (baseline → `in_progress`, superseded) | Epic 4 / Story 4.3 |
| `[Disable]` / `[Enable]` button in form header | Story 2.5 |
| `delete watch` link + inline confirmation | Story 2.5 |
| `data-slot="delete-zone"` content (currently shows placeholder text) | Story 2.5 |
| Status badge visual treatment on list rows | Story 2.6 |
| Baseline progress bar on list rows | Story 2.6 |
| Any new API endpoints or request contract changes | Not needed in any Epic 2 story |
| react-hook-form or any other new library | Not needed; plain useState is sufficient |
| Geography change detection in edit mode | Not applicable — geography is read-only in edit mode per 2.3 |

---

## Story Boundary Callouts — 2.4 vs 2.5 / 2.6

### 2.4 → 2.5 (Disable / Delete)
- 2.4 does NOT touch `data-slot="delete-zone"`. It still reads "Watch actions — coming in Story 2.5."
- 2.4 does NOT touch `rightSlot`. The `rightSlot` prop in `WatchFormPanel` is still `null` by default.
- 2.5 adds `[Disable]` to `rightSlot` and replaces the `data-slot="delete-zone"` content. These are purely additive changes to 2.4's output.

### 2.4 → 2.6 (Watch List Visual Richness)
- 2.4 does NOT touch `WatchListRow.tsx` or `WatchListColumn.tsx`. List rows are 2.6's domain.
- 2.4 does NOT touch the `badge` prop seam on `WatchListRow`.

### 2.4 → Epic 4 (Baseline State Machine)
- 2.4's `[Save & Reset Baseline]` fires a normal PATCH. Epic 4 / Story 4.3 will implement the processor-side baseline FSM that responds to `enabled_signal_families` or `baseline_window_days` changes by resetting baseline state.
- The frontend correctly communicates intent. The backend correctly acts on it — but in a later epic.
- **This is not an incomplete implementation.** The watch fields update correctly. The baseline state machine simply does not exist yet.

---

## Implementation Notes

### The 2.3 Integration Seam

`WatchFormPanel.tsx` line 103–106 already contains this comment and code:
```typescript
// Note for Story 2.4: 2.4 adds incompatible change detection and may override this label
// to 'Save & Reset Baseline' when isIncompatible === true. In 2.3 the label is always
// one of these three values.
const saveLabel = isPending ? 'Saving…' : justSaved ? '✓ Saved' : 'Save Changes'
```
Story 2.4 modifies this `saveLabel` derivation to incorporate `hasIncompatibleChange`.

### File Structure — What to Modify

```
services/cockpit/src/
└── components/
    └── watch-config/
        └── WatchFormPanel.tsx   ← MODIFY ONLY — this is the only file that changes
```

**Files NOT touched:**
- `WatchListColumn.tsx` — no changes
- `WatchListRow.tsx` — no changes
- `WatchConfigPage.tsx` — no changes
- `useWatches.ts`, `useGeographyEntities.ts`, `useCreateWatch.ts`, `useUpdateWatch.ts` — no changes
- `src/layouts/CockpitShell.tsx` — never modify
- `src/styles/tokens.css` — never modify
- `src/lib/api-client.ts` — never modify
- `services/api/` — no backend changes
- `migrations/` — no schema changes

### Detection Logic — Recommended Implementation

The form already uses plain `useState` (not react-hook-form). Add `useMemo` to the existing `react` import.

```typescript
import { useEffect, useMemo, useState } from 'react'
```

Add two derived values in the "Derived state" section (below the existing `isPending`, `isNameEmpty`, etc.):

```typescript
// ── Incompatible change detection — edit mode only ──────────────────────────
// Geography is read-only in edit mode (locked per Story 2.3). Only signal families
// and baseline window can be changed by the operator in edit mode.

const isSignalFamiliesIncompatible = useMemo(() => {
  if (mode !== 'selected' || !watch) return false
  const saved = new Set(watch.enabled_signal_families)
  const current = new Set(enabledSignalFamilies)
  if (saved.size !== current.size) return true
  for (const f of current) if (!saved.has(f)) return true
  return false
}, [mode, watch, enabledSignalFamilies])

const isBaselineWindowIncompatible = useMemo(() => {
  if (mode !== 'selected' || !watch) return false
  return baselineWindowDays !== watch.baseline_window_days
}, [mode, watch, baselineWindowDays])

const hasIncompatibleChange = isSignalFamiliesIncompatible || isBaselineWindowIncompatible
```

Update `saveLabel` (replacing the current line 106):

```typescript
const saveLabel = isPending
  ? 'Saving…'
  : justSaved
    ? '✓ Saved'
    : hasIncompatibleChange
      ? 'Save & Reset Baseline'
      : 'Save Changes'
```

### Warning Banner Component — Inline Pattern

Define a small inline constant for the banner style (or use inline object literals — same result):

```typescript
const incompatibleBannerStyle: React.CSSProperties = {
  marginTop: 8,
  padding: '8px 12px',
  background: 'var(--bg-raised)',
  borderLeft: '3px solid var(--accent-attention)',
  borderRadius: 4,
  fontSize: 12,
  color: 'var(--text-secondary)',
  display: 'flex',
  alignItems: 'flex-start',
  gap: 8,
}
```

Place below Signal Families field group (after the `isSignalFamiliesEmpty` error block):
```tsx
{isSignalFamiliesIncompatible && (
  <div style={incompatibleBannerStyle}>
    <span style={{ color: 'var(--accent-attention)', flexShrink: 0 }}>⚠</span>
    <span>Changing this setting will reset the baseline for this watch. Collected data will be discarded.</span>
  </div>
)}
```

Place below Baseline Window field group (after the helper text):
```tsx
{isBaselineWindowIncompatible && (
  <div style={incompatibleBannerStyle}>
    <span style={{ color: 'var(--accent-attention)', flexShrink: 0 }}>⚠</span>
    <span>Changing this setting will reset the baseline for this watch. Collected data will be discarded.</span>
  </div>
)}
```

### `useMemo` Dependency Arrays — Important

The `isSignalFamiliesIncompatible` memo depends on `enabledSignalFamilies` (array). Since React compares arrays by reference, and `setEnabledSignalFamilies` always produces a new array (via filter/spread), this correctly re-evaluates on every checkbox toggle. Do not sort the arrays — use Set-based comparison to ensure order-independence.

### Signal Families Set Comparison — Order Independence

The API returns `enabled_signal_families` as an array in whatever order the DB stored them. The user may toggle checkboxes in any order. Always compare signal families as **sets** (order-independent), not as sorted or sorted-and-joined strings. The reference implementation above (Set construction + size + has() check) is correct.

### Baseline Window Timing

`baselineWindowDays` updates on `onChange` and clamps on `onBlur` (per 2.3 `handleBaselineBlur`). The detection should use the current committed value in state. Since `isBaselineWindowIncompatible` is a `useMemo` derived from `baselineWindowDays` state, it will re-evaluate on every `onChange` and every `onBlur`. This means the banner may flicker if the user types "4" then "4" → "45" (briefly at 4, which is invalid and clamps to 7). This is acceptable behavior — the clamped value is what matters, not the in-flight text.

### No Changes to Save Payload or Mutation

The `handleSave` function in `mode === 'selected'` already sends:
```typescript
{
  name: name.trim(),
  enabled_signal_families: enabledSignalFamilies,
  escalation_threshold: escalationThreshold,
  baseline_window_days: baselineWindowDays,
}
```
This payload is identical for compatible and incompatible saves. No modification to `handleSave` is needed for the payload. The only difference is the button label that triggers `handleSave`.

### Architecture Compliance Guardrails

1. **Signal Desk is default home.** This story does not touch routing. Do not modify `src/router.tsx`.
2. **No new npm packages.** `useMemo` is built into React; already installed.
3. **No modals.** The warning banner is an inline element in the form body. No `Dialog`, no overlay.
4. **CSS tokens only.** Use `var(--accent-attention)`, `var(--bg-raised)`, `var(--text-secondary)`. No hardcoded hex values.
5. **No new files.** One file modified: `WatchFormPanel.tsx`.

---

## Testing Expectations

This story is frontend-only. No backend test changes.

**Manual smoke test:** AC #15 defines the full scenario. Verified by Nikku per the established pattern.

**Unit tests:** Not required for this story (the project has not established a React component test suite). If the developer wishes to add tests, they would use React Testing Library to assert banner render/no-render based on props. This is optional and not a blocking requirement for story completion.

**Build verification:** `tsc -b && vite build` must pass clean (0 errors). Same standard as prior stories.

**Regression checks:**
- New watch creation flow: no banners, no label change
- Compatible edit (name, threshold): no banners, no label change
- Prior cancel behavior: form still reverts correctly in both new and edit modes
- `✓ Saved` flash: still works after incompatible save
- `isSignalFamiliesEmpty` validation: still shows error when all families unchecked

---

## Dependencies and Prerequisites

| Dependency | Status | Notes |
|---|---|---|
| Story 2.3 (Watch Form) | Done ✓ | `WatchFormPanel.tsx` with all form fields live. The `saveLabel` comment at line 103–106 is the explicit integration seam for this story. |
| `PATCH /api/v1/watches/{id}` | Done ✓ (Story 2.1) | Accepts `enabled_signal_families` and `baseline_window_days`. No changes needed. |
| `useUpdateWatch.ts` | Done ✓ (Story 2.3) | No changes needed. |
| TanStack Query cache invalidation | Done ✓ (Story 2.3) | After save, `watch` prop updates from re-fetch, auto-clearing `hasIncompatibleChange`. |
| CSS token `--accent-attention` | Done ✓ (Story 1.8) | Amber color for warning icons and banner borders. |
| CSS token `--bg-raised` | Done ✓ (Story 1.8) | Banner background. |

---

## Carry-Forward Intelligence from Stories 2.2 and 2.3

| Item | Impact on Story 2.4 |
|---|---|
| `WatchFormPanel` uses plain `useState` (no react-hook-form) | Detection uses `useMemo` comparing state values vs. `watch` prop. No form library abstractions. |
| `useEffect([mode, watch?.id])` resets form | After incompatible save, `watch.id` unchanged → `useEffect` does NOT re-run. `hasIncompatibleChange` auto-clears via prop update. This is correct behavior. |
| Geography is read-only in edit mode | Zero geography detection logic in 2.4. Do not add any. |
| `saveLabel` comment at line 103 of `WatchFormPanel.tsx` | This is the exact integration point. Replace the `saveLabel` derivation to incorporate `hasIncompatibleChange`. |
| `data-slot="delete-zone"` placeholder at bottom of form body | Do not touch. Story 2.5 fills this. |
| `rightSlot` prop defaults to `null` | Do not touch. Story 2.5 passes `<DisableButton />` here. |
| `--accent-attention` used for Save button fill color | Same token used for warning banner left border and ⚠ icon. Consistent amber language. |

---

## References

- [UX spec §8.4 — Compatible vs. Incompatible Change Handling](_bmad-output/planning-artifacts/ux-design-specification.md) — field classification, banner visual spec, button label change
- [UX spec §10.5 — Inline Warning Banner Component](_bmad-output/planning-artifacts/ux-design-specification.md) — banner background, border, icon colors
- [UX spec §11 — Global Interaction Rules: "Incompatible change warning appears on field change, not on save"](_bmad-output/planning-artifacts/ux-design-specification.md)
- [Epics §FR9](_bmad-output/planning-artifacts/epics.md) — "compatible changes may preserve existing baseline; incompatible changes must trigger explicit baseline reset"
- [Epics §NFR17](_bmad-output/planning-artifacts/epics.md) — "Watch configuration changes that invalidate existing baseline data must trigger an explicit baseline reset"
- [Story 2.3 Completion Notes](_bmad-output/implementation-artifacts/2-3-watch-form.md) — "No incompatible-change detection (2.4)"; `saveLabel` integration seam documented
- [Story 2.3 WatchFormPanel.tsx — actual source](./) — plain useState form, exact `saveLabel` derivation at line 103–106, `handleCancel` edit mode revert behavior

---

## Tasks / Subtasks

- [ ] Task 1: Update `WatchFormPanel.tsx` — add `useMemo` to imports (AC: #2–#8)
  - [ ] 1.1 Add `useMemo` to the React import line (existing: `{ useEffect, useState }` → `{ useEffect, useMemo, useState }`)

- [ ] Task 2: Add incompatible change detection derived values (AC: #2–#8)
  - [ ] 2.1 Add `isSignalFamiliesIncompatible` useMemo — Set-based comparison of `enabledSignalFamilies` vs `watch.enabled_signal_families`; returns `false` when `mode !== 'selected' || !watch`
  - [ ] 2.2 Add `isBaselineWindowIncompatible` useMemo — strict equality comparison of `baselineWindowDays` vs `watch.baseline_window_days`; returns `false` when `mode !== 'selected' || !watch`
  - [ ] 2.3 Add `hasIncompatibleChange` derived boolean — `isSignalFamiliesIncompatible || isBaselineWindowIncompatible`

- [ ] Task 3: Update `saveLabel` derivation (AC: #6, #11)
  - [ ] 3.1 Replace the existing `saveLabel` line with the 4-branch ternary: `isPending → 'Saving…' | justSaved → '✓ Saved' | hasIncompatibleChange → 'Save & Reset Baseline' | 'Save Changes'`

- [ ] Task 4: Render warning banner below Signal Families section (AC: #3, #5, #7, #8)
  - [ ] 4.1 Define `incompatibleBannerStyle` constant (or inline style object) with `--bg-raised` background, `3px solid --accent-attention` left border, `8px 12px` padding, 12px text
  - [ ] 4.2 Place `{isSignalFamiliesIncompatible && <div ...>⚠ banner</div>}` below the `isSignalFamiliesEmpty` error block in the Signal Families field group

- [ ] Task 5: Render warning banner below Baseline Window section (AC: #4, #5, #7, #8)
  - [ ] 5.1 Place `{isBaselineWindowIncompatible && <div ...>⚠ banner</div>}` below the helper text in the Baseline Window field group

- [ ] Task 6: Verify no changes to save payload or cancel logic (AC: #9, #10)
  - [ ] 6.1 Confirm `handleSave` is unchanged — same PATCH payload for compatible and incompatible saves
  - [ ] 6.2 Confirm `handleCancel` is unchanged — existing revert behavior already clears `hasIncompatibleChange` as a side effect

- [ ] Task 7: TypeScript compile + smoke test (AC: #14, #15)
  - [ ] 7.1 `npm run build` — confirm no TypeScript errors
  - [ ] 7.2 Start infra + API + cockpit dev server; perform full smoke test (Nikku)
  - [ ] 7.3 Verify signal families warning: uncheck → banner appears; re-check → banner disappears; save → ✓ Saved → banner gone
  - [ ] 7.4 Verify baseline window warning: change value → banner appears; revert → banner gone; save → ✓ Saved → banner gone
  - [ ] 7.5 Verify compatible edits: name change, threshold change → no banners at any point
  - [ ] 7.6 Verify new watch mode: no banners at any point
  - [ ] 7.7 Verify `isSignalFamiliesEmpty` validation still works (all families unchecked → save disabled, error message shown)

---

## Change Log

- 2026-03-10: Story 2.4 created by Bob (SM, claude-sonnet-4-6). Prerequisites confirmed: 1.1, 1.2, 1.3, 1.7, 1.8, 2.1, 2.2, 2.3 all done. Full SM ambiguity resolution documented (§1–§10). Field classification table authoritative. Story boundary callouts against 2.5, 2.6, Epic 4. Status: ready-for-dev.
