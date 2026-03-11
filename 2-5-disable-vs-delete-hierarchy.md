# Story 2.5: Disable / Enable / Delete Hierarchy

Status: ready-for-dev

## Dependency Note — Local-First Route

Stories 1.4 (VPS provisioning), 1.5 (production nginx), and 1.6 (production Docker Compose) remain **intentionally deferred** — production deployment, no bearing on local development.

**Prerequisites satisfied:**
- 1.1 (monorepo scaffold) — done ✓
- 1.2 (Alembic initial migration) — done ✓ — `watches`, `geography_entities` tables, 11 seeded entities
- 1.3 (shared schema package) — done ✓ — `WatchCreate`, `WatchUpdate`, `WatchResponse`, `SignalFamilyEnum` all live
- 1.7 (FastAPI app skeleton) — done ✓
- 1.8 (cockpit shell) — done ✓ — CSS token system, axios client, TanStack Query provider established
- 2.1 (Watches API CRUD) — done ✓ — `PATCH /api/v1/watches/{id}` and `DELETE /api/v1/watches/{id}` live and tested
- 2.2 (Watch Configuration surface) — done ✓ — Two-column surface; `rightSlot` prop and `data-slot="delete-zone"` seams established
- 2.3 (Watch form) — done ✓ — All real form fields live; `rightSlot` and `data-slot="delete-zone"` untouched
- 2.4 (Compatible vs. incompatible change detection) — done ✓ — Incompatible banners live; `data-slot="delete-zone"` placeholder confirmed intact: "Watch actions — coming in Story 2.5"

---

## Story

As an operator,
I want to disable an active watch, re-enable a disabled watch, or permanently delete a watch — all from within the Watch Configuration surface — so that I can control the lifecycle of each watch condition without leaving the `/watches` route, with each action clearly differentiated by its reversibility and consequence.

---

## Precise Scope

### This Story IS:
- Adding `[Disable]` / `[Enable]` button to the form panel header (right side), visible only in edit mode (`mode === 'selected'`)
- Inline two-step confirmation for the Disable action (no modal)
- Single-click Enable with no confirmation (reversible, no destructive consequence)
- Adding the Delete control at the bottom of the form body, below Save/Cancel, below the existing dashed divider, inside `data-slot="delete-zone"`
- Text-link style for the delete trigger (low visual weight, deliberate friction)
- Inline expansion confirmation block for Delete (no modal)
- Post-action local UI state: watch remains selected after Disable/Enable; selection clears after Delete
- A small, contained backend change: add `is_active: Optional[bool]` to `WatchUpdate` + update PATCH handler status logic
- Repurposing the existing `DELETE /api/v1/watches/{id}` endpoint to be a true permanent delete (row removal) — superseding Story 2.1's soft-disable behavior
- One new frontend hook: `useDeleteWatch.ts`
- Inline error/status feedback for each lifecycle action

### This Story IS NOT:
- Story 2.6 status badges, status badge pills, colored left stripes, baseline progress bars on list rows
- Epic 4 processor-side consequences of disable (stopping signal detection) or re-enable (restarting baseline computation)
- The baseline FSM state machine — status transitions driven by the processor are Epic 4 / Story 4.3
- Shell redesign or router changes
- Modal dialogs — all confirmations are inline
- New npm packages
- Changes to `WatchListRow.tsx` or `WatchListColumn.tsx`
- Changes to `useWatches.ts`, `useGeographyEntities.ts`, or `useCreateWatch.ts`

---

## SM Ambiguity Clarifications

### §1 — API Contract Gap: The Existing DELETE Is a Soft-Disable, Not a Permanent Delete

**Discovery:** Reading `services/api/api/routers/watches.py` reveals that the existing `DELETE /api/v1/watches/{id}` endpoint (implemented in Story 2.1) does **not** remove the row from the database. It performs a soft-disable: sets `watch.is_active = False` and `watch.status = WatchStatusEnum.inactive`, then returns the updated `WatchResponse`. Story 2.1's carry-forward notes confirm: "Soft-delete confirmed."

This was the correct interim behavior at Story 2.1 time — the full lifecycle design was not yet defined. Story 2.5 now defines it.

**Resolution:** Story 2.5 **repurposes** the `DELETE /api/v1/watches/{id}` endpoint to be a **true permanent delete** — it removes the watch row from the database. The `WatchBaseline` FK has `ondelete="CASCADE"`, so all baseline rows for the deleted watch are automatically removed. This changes the behavior of an existing endpoint. The change is intentional and documented.

Immediately after a confirmed delete:
- The watch row is gone from the DB
- `GET /api/v1/watches` no longer returns it
- The frontend clears selection → form area shows "Select or create a watch"

### §2 — API Contract Gap: No Enable Endpoint Exists; WatchUpdate Has No is_active Field

**Discovery:** The `WatchUpdate` Pydantic schema (`shared/schema/watches.py`, line 96–101) contains:
```python
class WatchUpdate(BaseModel):
    name: Optional[str] = None
    enabled_signal_families: Optional[list[SignalFamilyEnum]] = None
    escalation_threshold: Optional[float] = None
    baseline_window_days: Optional[int] = None
```
There is no `is_active` field. The PATCH handler cannot currently disable or enable a watch. There is no separate enable endpoint.

**Resolution:** Story 2.5 adds `is_active: Optional[bool] = None` to `WatchUpdate`. The PATCH handler already uses `model_dump(exclude_none=True)` + generic `setattr` loop, so `is_active` will be applied automatically. However, the PATCH handler must also apply the correct `status` transition when `is_active` changes:
- When PATCH receives `is_active=False`: also set `watch.status = WatchStatusEnum.inactive`
- When PATCH receives `is_active=True`: also set `watch.status = WatchStatusEnum.created` (Epic 4 FSM will compute and advance the status from there based on baseline state)

This logic is added directly in the PATCH handler's for-loop (2–3 lines). No new endpoint is needed. Disable and Enable both use the existing `PATCH /api/v1/watches/{id}` via the existing `useUpdateWatch` hook.

### §3 — Disable Uses PATCH, Not DELETE

**Resolution of naming confusion:** Given §1 and §2 above, the frontend action mapping is:
- **Disable** → `PATCH /api/v1/watches/{id}` with body `{ is_active: false }` → sets `is_active=False, status=inactive`
- **Enable** → `PATCH /api/v1/watches/{id}` with body `{ is_active: true }` → sets `is_active=True, status=created`
- **Delete** → `DELETE /api/v1/watches/{id}` → permanently removes row from DB

The existing `useUpdateWatch` hook is reused for Disable and Enable (it sends the provided body to PATCH). A new `useDeleteWatch` hook is created for Delete. No new hook is needed for Disable/Enable.

### §4 — Action Visibility Rules: Edit Mode Only, Never New Mode

The Disable/Enable button and the Delete zone are **only visible in edit mode** (`mode === 'selected'`). They are never shown in new-watch mode (`mode === 'new'`). This is already partially enforced by the existing `{mode === 'selected' && ...}` guard around `data-slot="delete-zone"` at line 567 of `WatchFormPanel.tsx`.

The form header's `{rightSlot}` renders in both 'new' and 'selected' modes per the current header DOM. Story 2.5 implements the Disable/Enable button **directly inside WatchFormPanel** (not via the `rightSlot` prop from WatchConfigPage), guarded by `mode === 'selected' && watch != null`. The `rightSlot` prop remains on the component interface for future extensibility but is unused in this story.

### §5 — Disable vs. Enable: Which Shows When

**Action hierarchy:**
- When `watch.is_active === true`: show `[Disable]` in form header
- When `watch.is_active === false`: show `[Enable]` in form header
- Never both simultaneously

This is derived from the `watch` prop: `watch?.is_active`. The button renders its current state based on the live `watch` prop. After a successful Disable or Enable, TanStack Query invalidates the `watches` cache → re-fetch → `watch` prop updates via `WatchConfigPage`'s `selectedWatch` derivation → button state updates automatically. No local state needed for which button is showing.

### §6 — Disable Confirmation: Two-Click Inline, No Modal

**Resolution:** Click `[Disable]` → the button area in the form header transitions to an inline confirmation row:

```
Disable this watch? Signals will stop generating.
                    [Cancel]    [Confirm Disable]
```

No modal. No navigation. The confirmation replaces the `[Disable]` button within the form header `rightSlot` area. Clicking `[Cancel]` reverts to `[Disable]`. Clicking `[Confirm Disable]` fires the PATCH.

Local state: `[showDisableConfirm, setShowDisableConfirm] = useState(false)` inside `WatchFormPanel`.

After successful Confirm Disable:
- `setShowDisableConfirm(false)` — confirmation collapses
- `queryClient.invalidateQueries({ queryKey: ['watches'] })` — cache invalidated by mutation hook
- Re-fetched watch has `is_active=false` → button shows `[Enable]`

If disable PATCH fails:
- `setShowDisableConfirm(false)` — confirmation collapses
- Show inline error near the lifecycle button (a `saveError`-style message in the header area, 12px, `--accent-degraded`)

### §7 — Enable: Single Click, No Confirmation

**Resolution:** `[Enable]` is a single-click action with no confirmation step. Enable is reversible (Disable can be re-applied at any time) and has no data loss. Showing a confirmation would add friction to a low-consequence action.

After successful `[Enable]` click:
- PATCH fires immediately
- During mutation (`isPending`): button shows "Enabling…" (disabled, reduced opacity)
- On success: cache invalidated → re-fetch → `watch.is_active=true` → button shows `[Disable]`
- On error: show inline error near the button

### §8 — Delete Confirmation: Inline Expand Block, No Modal

**Resolution:** The delete trigger is a text-link at the bottom of the form body, inside `data-slot="delete-zone"`, below the existing dashed divider. On click, an inline confirmation block expands below the link:

```
Delete watch
Permanently removes this watch and all baseline data.
This action cannot be undone.

───────────────────────────────────────
Delete "{watch.name}"?
This will permanently discard all baseline data for this watch.

                    [Cancel]    [Confirm Delete]
───────────────────────────────────────
```

`[Confirm Delete]`: destructive button style — `1px solid var(--accent-degraded)`, `--accent-degraded` text color, transparent background. Single click confirms — no second confirmation.

`[Cancel]`: collapses the confirmation block, returns to the text-link view. No action taken.

Local state: `[showDeleteConfirm, setShowDeleteConfirm] = useState(false)` inside `WatchFormPanel`.

During DELETE mutation (`isDeletePending`): `[Confirm Delete]` shows "Deleting…" and is disabled.

### §9 — Post-Delete State: Selection Clears, Form Shows None State

After a confirmed delete:
- `onWatchDeleted?.()` callback fires (new prop on `WatchFormPanel`)
- `WatchConfigPage.handleWatchDeleted` → `setSelectedWatchId(null)`; `setIsCreatingNew(false)`
- Form mode transitions to `'none'` → center placeholder: "Select a watch from the list, or click [+ New] to create one."
- The deleted watch no longer appears in the list (GET re-fetch returns without it)

The `onWatchDeleted` prop is required for this handoff. Add it to `WatchFormPanelProps`:
```typescript
onWatchDeleted?: () => void
```

### §10 — Post-Disable/Enable State: Watch Remains Selected

After a successful Disable or Enable:
- `selectedWatchId` in WatchConfigPage does not change
- The watch remains selected in the list (same row highlighted)
- The watch row's visual treatment updates automatically: Story 2.2 already dims inactive watches (`is_active=false → --text-tertiary`). After disable, the row dims. After enable, it un-dims. No new list-row code is needed.
- The form remains open with the same watch loaded
- The 2.4 incompatible change banners (if any were showing) remain visible — they are derived from form field state, not from is_active status. The developer must NOT add any logic to clear banners on Disable/Enable.

### §11 — Unsaved Form Edits Do Not Block Lifecycle Actions

**Resolution:** Lifecycle actions (Disable/Enable/Delete) fire independently of form dirty state. If the operator has unsaved edits and clicks `[Disable]`, the PATCH fires with `{ is_active: false }` only — form field edits are not included in the payload and are not lost. The operator can continue editing after the disable.

This is consistent with the UX spec's separation of concerns: form edits are data configuration; Disable/Enable/Delete are lifecycle state transitions.

One edge case: After Delete, the form's unsaved edits are lost — but this is intentional, since the watch no longer exists. The form resets to 'none' state after delete, discarding any unsaved state.

### §12 — 2.4 Warning Banners Are Unaffected by 2.5

Story 2.4's incompatible change banners are derived from form field state. Story 2.5 does not touch that logic. The banners remain visible during and after any lifecycle action. The developer must not add any code that clears or interacts with `isSignalFamiliesIncompatible`, `isBaselineWindowIncompatible`, or `hasIncompatibleChange` from Story 2.5 code.

### §13 — status Re-Assignment on Enable: Created, Not Restored

When enabling a watch (`is_active=True` PATCH), the backend sets `status = 'created'`. It does NOT attempt to restore the pre-disable status. The processor (Epic 4 / Story 4.3) will:
- Detect the watch is active again
- Check if a valid baseline exists in `watch_baselines`
- Advance the status appropriately (`created → baseline_in_progress` or `created → active` if baseline established)

Story 2.5 does not implement this processor logic. `status='created'` on re-enable is correct, safe, and non-destructive — baseline data is preserved in `watch_baselines` rows (they are not deleted by disable/enable).

### §14 — Baseline Data on Delete Is Cascade-Deleted by DB

Permanently deleting a watch (DELETE endpoint) causes:
- The `Watch` row to be removed from the `watches` table
- All `WatchBaseline` rows for that watch to be automatically removed by `ondelete="CASCADE"` (confirmed FK in `schema/watches.py` line 54)

No explicit cleanup code is needed in the DELETE handler. The cascade handles it.

### §15 — useDeleteWatch Hook Returns Updated Watch List State via Cache Invalidation

The new `useDeleteWatch` hook fires `DELETE /api/v1/watches/{id}`. On success:
- `queryClient.invalidateQueries({ queryKey: ['watches'] })` triggers a re-fetch
- Since the watch is now removed from the DB, the re-fetched list won't contain it
- The `onWatchDeleted` callback fires (from mutation's `onSuccess`) → WatchConfigPage clears selection → mode → 'none'

The timing: `invalidateQueries` and `onWatchDeleted()` can fire in the same `onSuccess` callback. The selection clear will cause a re-render; the re-fetch will update the list. No race condition.

### §16 — The rightSlot Prop: Unused by WatchConfigPage in This Story

Story 2.2 reserved `rightSlot?: ReactNode` on `WatchFormPanel` for Story 2.5's Disable button. Story 2.5 implements the Disable/Enable button logic **directly inside WatchFormPanel's render** (not as a passed node). The `rightSlot` prop remains on the interface (do not remove it — it may be useful for future surfaces), but `WatchConfigPage` does not pass it in this story.

The `{rightSlot}` render in the form header (line 321 of `WatchFormPanel.tsx`) becomes redundant once the lifecycle button is implemented directly. The developer should keep the `{rightSlot}` render where it is (it renders `null` if not passed) and place the lifecycle button render adjacent to it or replace it. The `rightSlot` line must remain in the JSX as `{rightSlot ?? null}` to preserve the prop contract.

**Recommended header structure after 2.5:**
```tsx
<div style={{ ...headerStyle }}>
  <span style={{ ...labelStyle }}>{headerLabel}</span>
  <div style={{ display: 'flex', alignItems: 'center', gap: 8 }}>
    {/* Story 2.5: Lifecycle button — edit mode only */}
    {mode === 'selected' && watch && <LifecycleButton ... />}
    {/* rightSlot: reserved for future surfaces */}
    {rightSlot ?? null}
  </div>
</div>
```

Or: implement lifecycle button inline (no sub-component) using local state. Either approach is acceptable.

---

## Acceptance Criteria

### Lifecycle Button — Disable

1. **[Disable] visible only in edit mode with active watch:** When `mode === 'selected'` and `watch.is_active === true`, the form header shows a `[Disable]` button, right-aligned. In `mode === 'new'`, no button is visible. When `watch.is_active === false`, no `[Disable]` button is visible.

2. **[Disable] button style:** Secondary outline style — `border: 1px solid var(--border-subtle)`, `background: transparent`, `color: var(--text-secondary)`, `fontSize: 12`, `padding: 4px 12px`, `borderRadius: 4`. Same visual weight as the form's `[Cancel]` button but smaller.

3. **[Disable] first click → inline confirmation row:** Clicking `[Disable]` replaces the button in the header area with an inline confirmation row:
   ```
   Disable this watch?    [Cancel]    [Confirm Disable]
   ```
   No modal. No full-page overlay. The confirmation is within the form header area.

4. **[Cancel] in disable confirmation:** Collapses confirmation row and restores the `[Disable]` button. No API call. No state change.

5. **[Confirm Disable] fires PATCH:** Clicking `[Confirm Disable]` calls `PATCH /api/v1/watches/{id}` with body `{ is_active: false }`. During mutation, the button shows a pending state (e.g., "Disabling…", disabled, reduced opacity).

6. **After successful disable:** Confirmation row disappears. Header now shows `[Enable]` (because `watch.is_active === false` after re-fetch). The watch row in the list dims (`--text-tertiary`) per Story 2.2's existing inactive row treatment. Form remains open with the same watch selected. No incompatible change banners are touched.

7. **Disable error:** If the PATCH fails, the confirmation row disappears and an inline error message appears near the lifecycle button area: 12px, `var(--accent-degraded)`. `[Disable]` button is restored.

### Lifecycle Button — Enable

8. **[Enable] visible only in edit mode with inactive watch:** When `mode === 'selected'` and `watch.is_active === false`, the form header shows `[Enable]`. In new mode, never visible.

9. **[Enable] button style:** Secondary outline style — same dimensions as `[Disable]`. Text: "Enable". The label communicates the action (what will happen), not the current state.

10. **[Enable] single click fires PATCH immediately:** No confirmation step. Click → PATCH `{ is_active: true }` fires. During mutation: button shows "Enabling…", disabled, reduced opacity.

11. **After successful enable:** Button returns to `[Disable]` (watch is now active). The watch row un-dims in the list. Form remains open. No form fields reset.

12. **Enable error:** Inline error message near the lifecycle button area. `[Enable]` button restored.

### Delete Zone

13. **Delete zone visible only in edit mode:** `data-slot="delete-zone"` content is only rendered when `mode === 'selected'`. In `mode === 'new'` or `mode === 'none'`, delete zone is not rendered. (This is already enforced by the existing `{mode === 'selected' && ...}` guard at line 567 of `WatchFormPanel.tsx`.)

14. **Delete zone structure — collapsed state:**
    ```
    ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ (dashed divider — already exists)

    Delete watch
    Permanently removes this watch and all baseline data.
    This action cannot be undone.
    ```
    - "Delete watch" is a text-link: `color: var(--text-tertiary)`, `fontSize: 12`, `cursor: pointer`, `textDecoration: underline on hover`. NOT a button.
    - Descriptive lines: 12px, `var(--text-tertiary)`.
    - No border, no background, no button chrome.

15. **Delete zone — click "delete watch" → confirmation block expands inline:**
    ```
    Delete "{watch.name}"?
    This will permanently discard all baseline data for this watch.

                        [Cancel]    [Confirm Delete]
    ```
    - Confirmation block expands inline within the delete zone area. No modal.
    - Watch name is `watch.name` (not uppercase — use the stored name).
    - `[Cancel]`: secondary outline style (`border: 1px solid var(--border-subtle)`, transparent bg). Collapses block, no action.
    - `[Confirm Delete]`: destructive style — `border: 1px solid var(--accent-degraded)`, `color: var(--accent-degraded)`, transparent background. No red fill — outline only.

16. **[Confirm Delete] fires DELETE:** Calls `DELETE /api/v1/watches/{id}`. During mutation: `[Confirm Delete]` shows "Deleting…", disabled, reduced opacity.

17. **After successful delete:** `onWatchDeleted()` callback fires → `WatchConfigPage` clears `selectedWatchId` → form mode becomes `'none'` → center placeholder: "Select a watch from the list, or click [+ New] to create one." The deleted watch no longer appears in the list.

18. **Delete error:** Confirmation block remains open. Inline error message below the buttons: 12px, `var(--accent-degraded)`. Operator can retry or `[Cancel]`.

### Backend Changes

19. **`WatchUpdate` accepts `is_active`:** `shared/schema/watches.py` — `WatchUpdate` includes `is_active: Optional[bool] = None`. PATCH calls with `{ is_active: false }` or `{ is_active: true }` are accepted and applied.

20. **PATCH handler applies status on is_active change:** `services/api/api/routers/watches.py` PATCH handler — when `is_active=False` is in the update, also sets `watch.status = WatchStatusEnum.inactive`. When `is_active=True` is in the update, also sets `watch.status = WatchStatusEnum.created`. This happens after the generic `setattr` loop (or as a special-case before commit).

21. **DELETE handler permanently removes the row:** The DELETE handler now removes the `Watch` row from the database. The response is a `204 No Content` (no body) or a `200` success envelope with a confirmation message. `WatchBaseline` rows for the deleted watch are cascade-deleted by the DB constraint. The handler no longer performs a soft-disable.

22. **No schema migrations needed:** `is_active` already exists as a column in the `watches` table (it was added in the 1.2 migration). Adding it to `WatchUpdate` is a Pydantic schema change only — no Alembic migration required.

### Holistic / Integration

23. **2.4 warning banners unaffected:** If incompatible change banners are visible (from Story 2.4), they remain visible after any lifecycle action. Story 2.5 does not modify, clear, or interact with `isSignalFamiliesIncompatible`, `isBaselineWindowIncompatible`, or `hasIncompatibleChange`.

24. **New watch mode is pristine:** In `mode === 'new'`, the form shows no lifecycle controls — no [Disable] / [Enable] button, no delete zone. Existing behavior unchanged.

25. **`npm run build` passes:** `tsc -b && vite build` completes without TypeScript errors after all changes. Expect ~155+ modules, 0 errors.

26. **Smoke test passes locally:** (See Testing Expectations section for the full scenario script.)

---

## Non-Goals (Explicit Scope Boundaries)

| Item | Deferred To |
|---|---|
| Status badge visual treatment on list rows (colored left stripe, pill badge) | Story 2.6 |
| `BASELINE` progress bar row treatment | Story 2.6 |
| Baseline computation restart when watch is re-enabled | Epic 4 / Story 4.3 |
| Processor stopping signal detection for inactive watches | Epic 4 |
| Watch status FSM transitions (beyond `created`/`inactive` set by 2.5) | Epic 4 / Story 4.3 |
| Entity boundary thumbnail in form | Future story |
| Domain Pulse live dots (green/amber/red) | Epic 3 |
| VPS / nginx / production Docker scope | Stories 1.4–1.6 (deferred) |
| Any new npm packages | Not needed |
| Modal-based confirmations | Never in v1 (UX global rule A6) |
| Changes to `WatchListRow.tsx` or `WatchListColumn.tsx` | Story 2.6 only |

---

## Story Boundary Callouts

### 2.5 → 2.6 (Watch List Visual Richness)

- 2.5 does NOT add status badge pill components, colored left stripe treatment, or inline signal counts to `WatchListRow`.
- After a Disable, the watch row dims — this uses Story 2.2's **existing** inactive row dimming (`is_active=false → --text-tertiary on both text lines`). No new row code is needed.
- Story 2.6 will add the full badge treatment. That story modifies `WatchListRow.tsx` only.
- **Risk to avoid:** Do not add any `status`, `badge`, or `is_active` visual logic to `WatchListRow.tsx` in this story.

### 2.5 → Epic 4 (Watch Lifecycle State Machine)

- 2.5 sets `is_active` and `status` via PATCH (minimal). This is a direct DB write.
- The processor (Epic 4) will implement the full FSM: monitoring feeds, advancing `created → baseline_in_progress → active → nominal`, detecting degraded state, etc.
- The `status='created'` value written on Enable is the correct reentry point. Epic 4 will advance it without any special signal from Story 2.5.
- **Risk to avoid:** Do not implement any baseline re-computation, signal detection, or status polling in Story 2.5.

### 2.5 → 2.4 (Incompatible Change Detection)

- 2.5 does NOT touch `isSignalFamiliesIncompatible`, `isBaselineWindowIncompatible`, `hasIncompatibleChange`, `saveLabel`, or the amber warning banner renders.
- Disable/Enable/Delete fire independently of any incompatible-change state. The banners may remain visible during and after lifecycle actions. This is correct.
- **Risk to avoid:** Do not add any logic that calls `setSaveError`, modifies the incompatible-change detection, or references the `saveLabel` variable from Story 2.5 code.

### 2.5 Backend: DELETE Semantics Change vs. Story 2.1

Story 2.1 implemented `DELETE /api/v1/watches/{id}` as soft-delete (is_active=False). Story 2.5 changes this to permanent delete. **This is a deliberate, documented supersession.** The Story 2.1 carry-forward notes are not a constraint — they describe what was built at that time. Story 2.5 intentionally corrects the semantic.

The developer should update the router's `deactivate_watch` function: rename it `delete_watch`, replace the soft-disable logic with `await session.delete(watch); await session.commit()`, and return a `204` or success envelope.

---

## Implementation Notes

### Backend Changes — Minimal, Contained

**File 1: `shared/schema/watches.py`**

Add `is_active` to `WatchUpdate`:

```python
class WatchUpdate(BaseModel):
    name: Optional[str] = None
    enabled_signal_families: Optional[list[SignalFamilyEnum]] = None
    escalation_threshold: Optional[float] = None
    baseline_window_days: Optional[int] = None  # incompatible — triggers baseline reset in processor
    is_active: Optional[bool] = None             # Story 2.5: disable (False) / enable (True)
```

**File 2: `services/api/api/routers/watches.py`**

PATCH handler — add status handling after the generic `setattr` loop:

```python
@router.patch("/{watch_id}")
async def update_watch(
    watch_id: uuid.UUID,
    body: WatchUpdate,
    _: str = Depends(verify_token),
    session: AsyncSession = Depends(get_db),
) -> JSONResponse:
    watch = await session.get(Watch, watch_id)
    if watch is None:
        raise WatchNotFoundError(f"Watch {watch_id} not found.")

    update_data = body.model_dump(exclude_none=True)
    if "enabled_signal_families" in update_data:
        update_data["enabled_signal_families"] = [
            f.value for f in (body.enabled_signal_families or [])
        ]
    for field, value in update_data.items():
        setattr(watch, field, value)

    # Story 2.5: Apply status transitions when is_active changes.
    # Epic 4 FSM will advance status further based on processor state.
    if body.is_active is False:
        watch.status = WatchStatusEnum.inactive
    elif body.is_active is True:
        watch.status = WatchStatusEnum.created

    watch.updated_at = datetime.now(UTC)
    await session.commit()
    await session.refresh(watch)
    return JSONResponse(
        content=success_envelope(WatchResponse.model_validate(watch).model_dump(mode="json"))
    )
```

DELETE handler — repurpose to true permanent delete:

```python
@router.delete("/{watch_id}", status_code=204)
async def delete_watch(
    watch_id: uuid.UUID,
    _: str = Depends(verify_token),
    session: AsyncSession = Depends(get_db),
) -> None:
    watch = await session.get(Watch, watch_id)
    if watch is None:
        raise WatchNotFoundError(f"Watch {watch_id} not found.")
    await session.delete(watch)
    await session.commit()
    # WatchBaseline rows cascade-deleted by FK ondelete=CASCADE
```

**Note on 204:** The frontend `useDeleteWatch` hook should handle a 204 response (no body). Axios returns the response with `data: ''` or `data: null` on 204 — the hook just checks `response.status === 204` or ignores the body.

### Frontend — New Hook

**File: `services/cockpit/src/hooks/useDeleteWatch.ts`** (CREATE)

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { apiClient } from '@/lib/api-client'

async function deleteWatch(id: string): Promise<void> {
  await apiClient.delete(`/api/v1/watches/${id}`)
}

export function useDeleteWatch() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: deleteWatch,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['watches'] })
    },
  })
}
```

The `onSuccess` invalidation in the hook handles cache update for the list. The `onWatchDeleted` callback (fired from the component's `onSuccess`) handles the UI state transition (clearing selection).

### Frontend — WatchFormPanel.tsx Changes

**Add to props:**
```typescript
onWatchDeleted?: () => void  // called after successful delete → WatchConfigPage clears selection
```

**Add local state** (alongside existing `[justSaved, setJustSaved]`, etc.):
```typescript
const [showDisableConfirm, setShowDisableConfirm] = useState(false)
const [showDeleteConfirm, setShowDeleteConfirm] = useState(false)
const [lifecycleError, setLifecycleError] = useState<string | null>(null)
```

**Add mutation:**
```typescript
const deleteMutation = useDeleteWatch()
```
Import `useDeleteWatch` from `@/hooks/useDeleteWatch`.

The existing `updateMutation` from `useUpdateWatch` handles Disable and Enable.

**Derived state for lifecycle:**
```typescript
const isDisablePending = updateMutation.isPending   // shared with save — isPending is already used
const isDeletePending = deleteMutation.isPending
```
Note: `updateMutation.isPending` is shared between Save and Disable/Enable. This is acceptable — only one mutation runs at a time. When disable/enable fires, the form's Save button will also show disabled/pending. This is correct behavior: don't allow concurrent mutations.

**Handlers:**
```typescript
const handleDisableConfirm = () => {
  setLifecycleError(null)
  updateMutation.mutate(
    { id: watch!.id, body: { is_active: false } },
    {
      onSuccess: () => { setShowDisableConfirm(false) },
      onError: (error) => {
        setShowDisableConfirm(false)
        setLifecycleError(extractErrorMessage(error))
      },
    },
  )
}

const handleEnableClick = () => {
  setLifecycleError(null)
  updateMutation.mutate(
    { id: watch!.id, body: { is_active: true } },
    {
      onError: (error) => { setLifecycleError(extractErrorMessage(error)) },
    },
  )
}

const handleDeleteConfirm = () => {
  setLifecycleError(null)
  deleteMutation.mutate(watch!.id, {
    onSuccess: () => {
      setShowDeleteConfirm(false)
      onWatchDeleted?.()
    },
    onError: (error) => { setLifecycleError(extractErrorMessage(error)) },
  })
}
```

**Note on TypeScript and `useUpdateWatch` body type:** The existing `useUpdateWatch` hook sends a typed body. When calling it with `{ is_active: false }` or `{ is_active: true }`, TypeScript must accept `is_active` in the mutation body type. Update the `WatchUpdateBody` or equivalent type in `useUpdateWatch.ts` to match the expanded `WatchUpdate` schema (add `is_active?: boolean`). This is a 1-line change to the TypeScript interface in that hook file.

**Reset confirm state on watch change:**
When the selected watch changes (mode or watch.id changes), reset lifecycle state:
```typescript
useEffect(() => {
  setShowDisableConfirm(false)
  setShowDeleteConfirm(false)
  setLifecycleError(null)
  // ... existing reset logic for form fields ...
}, [mode, watch?.id])
```
Add the new setters to the existing `useEffect` — do not create a second effect.

**Header render — lifecycle button:**
```tsx
{/* Lifecycle button — edit mode only */}
{mode === 'selected' && watch && (
  <div style={{ display: 'flex', alignItems: 'center', gap: 8 }}>
    {lifecycleError && (
      <span style={{ fontSize: 11, color: 'var(--accent-degraded)' }}>
        {lifecycleError}
      </span>
    )}
    {showDisableConfirm ? (
      <>
        <span style={{ fontSize: 12, color: 'var(--text-secondary)' }}>
          Disable this watch?
        </span>
        <button onClick={() => setShowDisableConfirm(false)} style={secondaryButtonStyle}>
          Cancel
        </button>
        <button
          onClick={handleDisableConfirm}
          disabled={isPending}
          style={{ ...secondaryButtonStyle, opacity: isPending ? 0.45 : 1 }}
        >
          {isPending ? 'Disabling…' : 'Confirm Disable'}
        </button>
      </>
    ) : watch.is_active ? (
      <button onClick={() => { setLifecycleError(null); setShowDisableConfirm(true) }} style={secondaryButtonStyle}>
        Disable
      </button>
    ) : (
      <button
        onClick={handleEnableClick}
        disabled={isPending}
        style={{ ...secondaryButtonStyle, opacity: isPending ? 0.45 : 1 }}
      >
        {isPending ? 'Enabling…' : 'Enable'}
      </button>
    )}
    {rightSlot ?? null}
  </div>
)}
{mode !== 'selected' && (rightSlot ?? null)}
```

Define `secondaryButtonStyle: React.CSSProperties`:
```typescript
const secondaryButtonStyle: React.CSSProperties = {
  padding: '4px 12px',
  background: 'transparent',
  border: '1px solid var(--border-subtle)',
  borderRadius: 4,
  color: 'var(--text-secondary)',
  fontSize: 12,
  cursor: 'pointer',
}
```

**Delete zone render — replace placeholder:**
```tsx
{/* ── Delete zone (mode === 'selected' only — guard already exists) ──────── */}
{mode === 'selected' && (
  <>
    <div style={{ borderTop: '1px dashed var(--border-subtle)', marginTop: 24, marginBottom: 16 }} />
    <div data-slot="delete-zone">
      {!showDeleteConfirm ? (
        <div>
          <button
            onClick={() => { setLifecycleError(null); setShowDeleteConfirm(true) }}
            style={{
              background: 'none',
              border: 'none',
              padding: 0,
              cursor: 'pointer',
              fontSize: 12,
              color: 'var(--text-tertiary)',
              textDecoration: 'underline',
              marginBottom: 4,
            }}
          >
            Delete watch
          </button>
          <p style={{ fontSize: 11, color: 'var(--text-tertiary)', margin: 0 }}>
            Permanently removes this watch and all baseline data.
            This action cannot be undone.
          </p>
        </div>
      ) : (
        <div style={{
          padding: '12px',
          background: 'var(--bg-raised)',
          border: '1px solid var(--border-subtle)',
          borderRadius: 4,
        }}>
          <p style={{ fontSize: 13, color: 'var(--text-primary)', marginBottom: 4 }}>
            Delete "{watch?.name}"?
          </p>
          <p style={{ fontSize: 12, color: 'var(--text-secondary)', marginBottom: 12 }}>
            This will permanently discard all baseline data for this watch.
          </p>
          {lifecycleError && (
            <p style={{ fontSize: 11, color: 'var(--accent-degraded)', marginBottom: 8 }}>
              {lifecycleError}
            </p>
          )}
          <div style={{ display: 'flex', justifyContent: 'flex-end', gap: 8 }}>
            <button
              onClick={() => setShowDeleteConfirm(false)}
              disabled={isDeletePending}
              style={{ ...secondaryButtonStyle, opacity: isDeletePending ? 0.45 : 1 }}
            >
              Cancel
            </button>
            <button
              onClick={handleDeleteConfirm}
              disabled={isDeletePending}
              style={{
                ...secondaryButtonStyle,
                border: '1px solid var(--accent-degraded)',
                color: 'var(--accent-degraded)',
                opacity: isDeletePending ? 0.45 : 1,
              }}
            >
              {isDeletePending ? 'Deleting…' : 'Confirm Delete'}
            </button>
          </div>
        </div>
      )}
    </div>
  </>
)}
```

### Frontend — WatchConfigPage.tsx Changes

Add `handleWatchDeleted` handler and wire to `WatchFormPanel`:

```typescript
const handleWatchDeleted = () => {
  setSelectedWatchId(null)
  setIsCreatingNew(false)
}
```

Pass to `WatchFormPanel`:
```tsx
<WatchFormPanel
  mode={formMode}
  watch={selectedWatch ?? undefined}
  entities={entities ?? []}
  onSaved={handleWatchSaved}
  onCancel={handleFormCancel}
  onWatchDeleted={handleWatchDeleted}   // ← new
/>
```

### Architecture Compliance Guardrails

1. **Signal Desk is default home.** This story does not touch routing. Do not modify `src/router.tsx`.
2. **No new npm packages.** All required dependencies are already installed.
3. **No modals.** All confirmations are inline — no `Dialog`, no overlay, no `position: fixed` wrapper.
4. **CSS tokens only.** `var(--accent-degraded)` for destructive styling; `var(--border-subtle)` for buttons; `var(--text-secondary/tertiary)` for text. No hardcoded hex colors.
5. **No changes to shell layout.** `CockpitShell.tsx`, `tokens.css`, `api-client.ts` are untouched.
6. **No changes to list components.** `WatchListRow.tsx` and `WatchListColumn.tsx` are untouched.
7. **Single mutation at a time.** The `updateMutation.isPending` guard on `isSaveDisabled` already prevents concurrent saves. Lifecycle actions share `isPending` with the save button — this is correct and prevents mutation conflicts.

---

## File Structure — What to Create / Modify

```
shared/
└── schema/
    └── watches.py                          ← MODIFY: add is_active to WatchUpdate

services/api/api/routers/
└── watches.py                              ← MODIFY: PATCH status logic; DELETE → permanent

services/cockpit/src/
├── hooks/
│   ├── useDeleteWatch.ts                   ← CREATE (new permanent-delete hook)
│   └── useUpdateWatch.ts                   ← MODIFY: add is_active?: boolean to body type
├── components/
│   └── watch-config/
│       └── WatchFormPanel.tsx              ← MODIFY (primary changes: header button, delete zone)
└── pages/
    └── WatchConfigPage.tsx                 ← MODIFY: add handleWatchDeleted, pass onWatchDeleted
```

**Files NOT touched:**
- `WatchListColumn.tsx` — no changes
- `WatchListRow.tsx` — no changes
- `useWatches.ts` — no changes
- `useGeographyEntities.ts` — no changes
- `useCreateWatch.ts` — no changes
- `src/layouts/CockpitShell.tsx` — never modify
- `src/styles/tokens.css` — never modify
- `src/lib/api-client.ts` — never modify
- `migrations/` — no schema migrations required

---

## Testing Expectations

This story touches backend (Python) and frontend (React). Both require testing.

**Backend tests:**

The existing `tests/test_watches.py` file (from Story 2.1) includes tests for the DELETE endpoint as a soft-disable. Those tests now test the wrong behavior — they expect a 200 response with an updated watch body showing `is_active=false`. After 2.5, DELETE returns `204 No Content` and the row is removed.

The developer must:
- Update the existing soft-delete test to test permanent deletion: assert 204 status, assert the watch no longer appears in `GET /api/v1/watches`, assert `GET /api/v1/watches/{id}` returns 404.
- Add a test for `PATCH with is_active=False`: assert status becomes `inactive`, `is_active` becomes `False`.
- Add a test for `PATCH with is_active=True` on an inactive watch: assert `is_active` becomes `True`, `status` becomes `created`.

**Frontend tests:** No React component tests are required (project has not established a React test suite). Manual smoke test is the verification gate.

**Build verification:** `tsc -b && vite build` must pass clean (0 errors).

**Manual smoke test (AC #26):**

```bash
# Terminal 1: API + infra (from repo root)
docker compose -f docker-compose.yml -f docker-compose.dev.yml up postgres valkey -d
uvicorn services.api.api.main:app --reload --host 0.0.0.0 --port 8000 --env-file .env

# Terminal 2: Cockpit (from repo root)
cd services/cockpit
npm run dev
# → Login → navigate to /watches

# === Disable flow ===
# → Select an active watch (is_active=true)
# → Form header shows [Disable] button, right-aligned
# → Click [Disable]
#   → Button area shows: "Disable this watch?" [Cancel] [Confirm Disable]
# → Click [Cancel]
#   → Confirmation disappears, [Disable] button restored
# → Click [Disable] again → [Confirm Disable]
#   → PATCH fires (Network tab: PATCH /api/v1/watches/{id} with {is_active: false})
#   → Button area shows "Disabling…" briefly
#   → On success: header now shows [Enable]; list row dims to --text-tertiary
# → Verify watch is still selected (same row highlighted in list)
# → 2.4 warning banners (if any) remain unaffected

# === Enable flow ===
# → With the just-disabled watch still selected, header shows [Enable]
# → Click [Enable]
#   → PATCH fires: PATCH /api/v1/watches/{id} with {is_active: true}
#   → Button shows "Enabling…" briefly
#   → On success: header shows [Disable]; list row un-dims
# → Verify watch is still selected

# === Delete flow ===
# → Select a watch (can be the same or different)
# → Scroll to bottom of form body, below Save/Cancel, below dashed divider
# → See "Delete watch" text-link + descriptive text
# → Click "delete watch"
#   → Inline confirmation block expands: watch name, warning, [Cancel] [Confirm Delete]
# → Click [Cancel]
#   → Confirmation block collapses, text-link restored
# → Click "delete watch" again → confirmation expands
# → Click [Confirm Delete]
#   → DELETE fires (Network tab: DELETE /api/v1/watches/{id})
#   → Shows "Deleting…" briefly
#   → On success: form shows "Select a watch from the list, or click [+ New] to create one."
#   → Watch no longer appears in list
# → Verify by navigating to /watches in Network tab: GET /api/v1/watches confirms watch is gone

# === New watch mode — no lifecycle controls ===
# → Click [+ New]
# → Form header shows no [Disable] or [Enable] button
# → No delete zone appears below Save/Cancel

# === Build check ===
# → npm run build → 0 TypeScript errors, 0 Vite errors
```

---

## Dependencies and Prerequisites

| Dependency | Status | Notes |
|---|---|---|
| Story 2.2 (Watch Configuration surface) | Done ✓ | `rightSlot` prop and `data-slot="delete-zone"` seams established |
| Story 2.3 (Watch form) | Done ✓ | Full form live; `data-slot="delete-zone"` placeholder: "Watch actions — coming in Story 2.5" |
| Story 2.4 (Incompatible change detection) | Done ✓ | Banners live; seams intact; `WatchFormPanel.tsx` is the file to modify |
| `PATCH /api/v1/watches/{id}` | Done ✓ (Story 2.1) | Accepts partial updates; will be extended with `is_active` |
| `DELETE /api/v1/watches/{id}` | Done ✓ (Story 2.1) | Will be repurposed from soft-disable to permanent delete |
| `useUpdateWatch.ts` | Done ✓ (Story 2.3) | Reused for Disable and Enable (body type extended) |
| CSS token `--accent-degraded` | Done ✓ (Story 1.8) | Red/destructive color for Delete button outline |
| CSS token `--text-tertiary` | Done ✓ (Story 1.8) | Low-contrast text for delete link |
| `WatchRow.is_active` field in TypeScript | Done ✓ (Story 2.2) | `WatchRow` interface includes `is_active: boolean` |

---

## Carry-Forward Intelligence from Prior Stories

| Item | Impact on Story 2.5 |
|---|---|
| `rightSlot` seam in `WatchFormPanel` header | Story 2.5 implements lifecycle button **directly** in the header (alongside `rightSlot`). `rightSlot` stays in JSX but WatchConfigPage does not pass a value. |
| `data-slot="delete-zone"` placeholder at bottom of form body | Story 2.5 **replaces** this placeholder with the real delete zone content (text-link + confirm block). The existing `{mode === 'selected' && ...}` guard remains. |
| `useEffect([mode, watch?.id])` resets form | Story 2.5 adds `setShowDisableConfirm`, `setShowDeleteConfirm`, `setLifecycleError` to the **existing** useEffect reset logic. Do not create a second effect. |
| `updateMutation.isPending` shared state | Disable/Enable reuse `updateMutation`. The `isPending` flag already gates `isSaveDisabled`. During disable/enable mutations, the Save button is also disabled — this is correct and acceptable. |
| `extractErrorMessage` helper | Already defined in `WatchFormPanel.tsx`. Reuse for lifecycle error handling. |
| `WatchBaseline` FK has `ondelete="CASCADE"` | Permanent watch delete automatically cleans up all baseline rows. No explicit cleanup needed. |
| `GET /api/v1/watches` returns ALL watches including inactive | After disable, watch stays in list (dimmed). After permanent delete, watch disappears. This distinction is correct and intentional. |
| Story 2.2 inactive row dimming | `WatchListRow` already applies `--text-tertiary` when `watch.is_active === false`. No 2.5 changes needed to the list for disable visual feedback. |
| `WatchStatusEnum.inactive` and `WatchStatusEnum.created` | Already defined in `schema/enums.py` (from Story 1.3). Both values are valid for PATCH handler use. |

---

## References

- [UX spec §8.3 — Watch Form: form header Disable button, rightSlot](_bmad-output/planning-artifacts/ux-design-specification.md)
- [UX spec §8.5 — Disable Action: inline confirmation, no modal, status→inactive, button→Enable](_bmad-output/planning-artifacts/ux-design-specification.md)
- [UX spec §8.6 — Disable vs. Inactive vs. Delete: reversibility rules, baseline preservation](_bmad-output/planning-artifacts/ux-design-specification.md)
- [UX spec §8.7 — Delete Placement and Hierarchy: text-link, dashed divider, inline expand, destructive confirm](_bmad-output/planning-artifacts/ux-design-specification.md)
- [UX spec §11.5 — Watch Configuration global rules: "Delete requires deliberate scroll + click + confirm"](_bmad-output/planning-artifacts/ux-design-specification.md)
- [Epics §Epic 2 — "Disable vs. Delete hierarchy (UX spec §8.6–8.7): Disable suspends watch; Delete requires explicit secondary confirmation; re-enable restores prior valid baseline"](_bmad-output/planning-artifacts/epics.md)
- [Story 2.4 §Scope — data-slot="delete-zone" and rightSlot seams untouched](_bmad-output/implementation-artifacts/2-4-compatible-vs-incompatible-change-detection.md)
- [Story 2.3 carry-forward notes — rightSlot seam preserved, data-slot=delete-zone and data-slot=entity-thumbnail reserved](_bmad-output/implementation-artifacts/sprint-status.yaml)
- [shared/schema/watches.py — WatchUpdate (no is_active yet), Watch ORM (is_active column present), WatchBaseline FK ondelete=CASCADE](shared/schema/watches.py)
- [services/api/api/routers/watches.py — DELETE as soft-disable (Story 2.1 behavior, superseded by this story)](services/api/api/routers/watches.py)
- [Architecture §4-tier retention — watches table, baseline retention rules](_bmad-output/planning-artifacts/architecture.md)

---

## Tasks / Subtasks

- [ ] Task 1: Backend — extend WatchUpdate schema (AC: #19)
  - [ ] 1.1 Add `is_active: Optional[bool] = None` to `WatchUpdate` in `shared/schema/watches.py`

- [ ] Task 2: Backend — update PATCH handler (AC: #20)
  - [ ] 2.1 After the generic `setattr` loop in `update_watch`, add: `if body.is_active is False: watch.status = WatchStatusEnum.inactive` / `elif body.is_active is True: watch.status = WatchStatusEnum.created`

- [ ] Task 3: Backend — repurpose DELETE to permanent delete (AC: #21)
  - [ ] 3.1 Replace soft-disable logic in `deactivate_watch` with `await session.delete(watch); await session.commit()`
  - [ ] 3.2 Change handler return to `204 No Content` (add `status_code=204` to decorator, return `None`)
  - [ ] 3.3 Rename function from `deactivate_watch` to `delete_watch`

- [ ] Task 4: Backend — update tests (Testing Expectations section)
  - [ ] 4.1 Update existing soft-delete test: assert 204 response, assert watch absent from `GET /watches`, assert `GET /watches/{id}` returns 404
  - [ ] 4.2 Add test: `PATCH {is_active: false}` → `status=inactive, is_active=False`
  - [ ] 4.3 Add test: `PATCH {is_active: true}` on inactive watch → `status=created, is_active=True`

- [ ] Task 5: Frontend — create useDeleteWatch hook (AC: #17)
  - [ ] 5.1 Create `services/cockpit/src/hooks/useDeleteWatch.ts` with `useMutation` calling `DELETE /api/v1/watches/{id}`, `onSuccess` invalidates `['watches']` cache

- [ ] Task 6: Frontend — extend useUpdateWatch body type (AC: #5, #10)
  - [ ] 6.1 Add `is_active?: boolean` to the body type in `useUpdateWatch.ts`

- [ ] Task 7: Frontend — WatchFormPanel props + state + effect (AC: #6, #11, #23)
  - [ ] 7.1 Add `onWatchDeleted?: () => void` to `WatchFormPanelProps` interface
  - [ ] 7.2 Add local state: `showDisableConfirm`, `showDeleteConfirm`, `lifecycleError`
  - [ ] 7.3 Import and instantiate `useDeleteWatch` hook
  - [ ] 7.4 Add `handleDisableConfirm`, `handleEnableClick`, `handleDeleteConfirm` handlers
  - [ ] 7.5 Add reset of lifecycle state to existing `useEffect([mode, watch?.id])`: `setShowDisableConfirm(false)`, `setShowDeleteConfirm(false)`, `setLifecycleError(null)`

- [ ] Task 8: Frontend — implement header lifecycle button (AC: #1–#7, #8–#12)
  - [ ] 8.1 Define `secondaryButtonStyle` const (or inline)
  - [ ] 8.2 Implement Disable/Confirm/Enable conditional render in form header, guarded by `mode === 'selected' && watch`
  - [ ] 8.3 Inline error display near lifecycle button for `lifecycleError`

- [ ] Task 9: Frontend — implement delete zone (AC: #13–#18)
  - [ ] 9.1 Replace `data-slot="delete-zone"` placeholder content with text-link (collapsed state)
  - [ ] 9.2 Implement inline expand confirmation block (watch name, warning, [Cancel] [Confirm Delete])
  - [ ] 9.3 Destructive button style for [Confirm Delete]: `--accent-degraded` border + text, no fill

- [ ] Task 10: Frontend — WatchConfigPage (AC: #17)
  - [ ] 10.1 Add `handleWatchDeleted` handler: `setSelectedWatchId(null); setIsCreatingNew(false)`
  - [ ] 10.2 Pass `onWatchDeleted={handleWatchDeleted}` to `WatchFormPanel`

- [ ] Task 11: Build verification + smoke test (AC: #25, #26)
  - [ ] 11.1 `npm run build` (from repo root: `cd services/cockpit && npm run build`) — confirm 0 TypeScript errors
  - [ ] 11.2 Backend tests: `pytest services/api/tests/test_watches.py` — all passing
  - [ ] 11.3 Start infra + API + cockpit dev server; perform full smoke test (Nikku)
  - [ ] 11.4 Verify Disable flow: [Disable] → confirm → [Enable] visible; row dims; watch stays selected
  - [ ] 11.5 Verify Enable flow: [Enable] → [Disable] visible; row un-dims; watch stays selected
  - [ ] 11.6 Verify Delete flow: text-link → confirm → watch gone from list; form shows none state
  - [ ] 11.7 Verify new-watch mode: no lifecycle controls visible
  - [ ] 11.8 Verify 2.4 banners unaffected by disable action (trigger an incompatible change, then disable — banner must remain visible)

---

## Change Log

- 2026-03-10: Story 2.5 created by Bob (SM, claude-sonnet-4-6). Prerequisites confirmed: 1.1, 1.2, 1.3, 1.7, 1.8, 2.1, 2.2, 2.3, 2.4 all done. SM ambiguity resolution documented (§1–§16). Key findings: existing DELETE is soft-disable (repurposed to permanent delete), WatchUpdate has no is_active field (added), no enable endpoint (handled via PATCH). Backend changes are minimal and contained. Status: ready-for-dev.
