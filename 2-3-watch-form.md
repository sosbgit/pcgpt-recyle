# Story 2.3: Watch Form

Status: ready-for-dev

## Dependency Note — Local-First Route

Stories 1.4 (VPS provisioning), 1.5 (production nginx), and 1.6 (production Docker Compose) remain **intentionally deferred** — production deployment, no bearing on local development.

**Prerequisites satisfied:**
- 1.1 (monorepo scaffold) — done ✓
- 1.2 (Alembic initial migration) — done ✓ — `watches`, `geography_entities` tables and 11 seeded entities
- 1.3 (shared schema package) — done ✓ — `WatchCreate`, `WatchUpdate`, `WatchResponse`, `SignalFamilyEnum` all live
- 1.7 (FastAPI app skeleton) — done ✓
- 1.8 (cockpit shell) — done ✓ — CSS token system, axios client, TanStack Query provider established
- 2.1 (Watches API CRUD) — done ✓ — `POST /api/v1/watches`, `PATCH /api/v1/watches/{id}` live and tested
- 2.2 (Watch Configuration surface) — done ✓ — Two-column surface, `WatchFormPanel.tsx` (stub body), `useWatches`, `useGeographyEntities`, `WatchListColumn`, `WatchListRow` all established

---

## Story

As an operator,
I want to open the Watch Configuration surface, click an existing watch or press `[+ New]`, and interact with a fully populated watch form — with a name field, geography entity selector, signal family toggles, escalation threshold slider, and baseline window input — so that I can create and edit complete watch configurations without leaving the `/watches` surface.

---

## Precise Scope

### This Story IS:
- Rendering the real form fields inside the right panel body (replacing the 2.2 placeholder text)
- Watch name text input (required)
- Geography entity dropdown — **editable for new watches only**; displayed as read-only in edit mode (see SM Ambiguity §1)
- Entity boundary thumbnail placeholder (geometry not available in this story — layout slot reserved)
- Signal family checkboxes (all 8, all enabled by default for new watches)
- Escalation threshold — range slider (1–100) + synced numeric input, default 65
- Baseline window — numeric input, min 7, suggested max 90, default 30 days
- Save/Cancel button row — right-aligned at the bottom of the form body
- Wiring Save to `POST /api/v1/watches` (create) and `PATCH /api/v1/watches/{id}` (update)
- Loading state on the Save button while the mutation is in-flight
- Inline `✓ Saved` confirmation replacing the button for 1.5s after successful update
- Inline error message below Save/Cancel when the API returns an error
- Cancel behavior: new-mode cancel notifies parent (exits `[+ New]` state); edit-mode cancel resets form fields to last fetched values
- After successful create: invalidate `['watches']` cache, select the newly created watch in the list
- Two new mutation hooks: `useCreateWatch` and `useUpdateWatch`
- Delete watch action hierarchy placeholder layout slot (dashed divider + reserved area below Save/Cancel) — static, no behavior — Story 2.5 populates it

### This Story is NOT:
- Story 2.4: compatible vs. incompatible change detection — no warning banners, no `[Save & Reset Baseline]` label, no per-field change tracking against incompatible-change rules
- Story 2.4: `[Save & Reset Baseline]` flow, baseline-reset confirmation UX
- Story 2.5: `[Disable]` button, inline disable confirmation, `[Enable]` toggle
- Story 2.5: `delete watch` text-link, delete confirmation block
- Story 2.6: status badges, colored left stripes, baseline progress bars, signal counts on list rows
- Entity boundary thumbnail rendering (geometry excluded from geography API; thumbnail is a reserved slot)
- Geography entity editing in edit mode (WatchUpdate schema excludes geography_entity_id; editing geography is gated on 2.4 incompatible-change flow)
- Backend changes — no schema migrations, no API changes, no new endpoints
- react-hook-form, zod, or any new npm package installations
- Modal-based UX — all interactions are inline within the existing right panel
- Shell or router changes

---

## Acceptance Criteria

### AC 1 — Form renders in panel body (new mode)
When `[+ New]` is clicked, the right column displays the sticky "NEW WATCH" header (established in 2.2) and a scrollable form body with the following fields in order:
1. **Watch Name** — full-width text input, placeholder "Enter watch name", initially empty
2. **Geography Entity** — labeled `"Geography Entity"`, a `<select>` dropdown showing all active entities from `useGeographyEntities()` (ordered by `name` asc, matching API sort), initially showing a placeholder option "Select entity…", with a read-only thumbnail placeholder below it
3. **Signal Families** — labeled `"Signal Families"`, 8 checkboxes, all checked by default (see signal family list in Dev Notes)
4. **Escalation Threshold** — labeled `"Escalation Threshold"`, range slider (1–100, step 1) + adjacent numeric input (same value), default value 65; helper text below: `"Signals at or above [N] escalate to Attention Required"`
5. **Baseline Window** — labeled `"Baseline Window"`, numeric input (integer, min 7, max 90), default value 30; label suffix `"days"`; helper text below: `"Minimum collection window before NOMINAL state is possible"`
6. **Save/Cancel row** — right-aligned; `[Save Changes]` primary button + `[Cancel]` secondary button

### AC 2 — Form populates from selected watch (edit mode)
When a watch row is clicked, the right column displays the sticky header with the watch's uppercase name (established in 2.2), and the form body is populated with:
- Watch Name: `watch.name`
- Geography Entity: displayed as the resolved entity name (from `entityMap`, same lookup used in `WatchListColumn`), **non-editable in this story** — rendered as static text with a `[locked]` visual cue (see Dev Notes §Form Field Styling)
- Signal Families: checkboxes pre-set to match `watch.enabled_signal_families` (checked if the enum value is present in the array, unchecked otherwise)
- Escalation Threshold: slider and input both show `watch.escalation_threshold`, rounded to the nearest integer for display
- Baseline Window: input shows `watch.baseline_window_days`
- Save/Cancel row: always visible

Population must occur correctly when switching between watches. Clicking a different row must reset all form fields to the new watch's values.

### AC 3 — Slider + numeric input stay in sync
For Escalation Threshold: editing the numeric input updates the slider position. Moving the slider updates the numeric input. Both clamp at 1 (min) and 100 (max). Non-numeric input in the number field reverts to the last valid value on blur.

### AC 4 — Baseline window range enforcement
The baseline window numeric input enforces: min 7, max 90. Values outside the range are clamped on blur (not rejected mid-typing). Typing `5` → on blur → corrects to `7`. Typing `120` → on blur → corrects to `90`.

### AC 5 — Save (create) wires to POST
In new-watch mode, clicking `[Save Changes]`:
1. Validates: name is non-empty (after trim) and a geography entity is selected. If either fails, the button remains enabled and an inline validation message appears below the invalid field. **No Save attempt is made when name is empty or no entity is selected.**
2. Fires `POST /api/v1/watches` via `useCreateWatch()` with `{ name, geography_entity_id, enabled_signal_families, escalation_threshold, baseline_window_days }`.
3. During the in-flight request: the Save button shows `"Saving…"` and is disabled; Cancel is also disabled.
4. On success (HTTP 201): invalidates the `['watches']` TanStack Query cache; calls `onSaved(newWatch.id)` so `WatchConfigPage` transitions from "creating new" to "selected" mode with the newly created watch highlighted.
5. On API error: an inline error message appears below the Save/Cancel row in `--accent-degraded` / `--text-secondary` style — `"Save failed: [error message from API response]"`. Save and Cancel both re-enable.

### AC 6 — Save (update) wires to PATCH
In selected-watch mode, clicking `[Save Changes]`:
1. Validates: name is non-empty after trim. If empty, inline validation message appears.
2. Fires `PATCH /api/v1/watches/{watch.id}` via `useUpdateWatch()` with only the editable fields (`name`, `enabled_signal_families`, `escalation_threshold`, `baseline_window_days`). **`geography_entity_id` is never included in the PATCH body.**
3. During in-flight: same loading state as AC 5.
4. On success: invalidates `['watches']` cache; replaces `[Save Changes]` with inline `"✓ Saved"` (same font size, `--accent-nominal` color, `--text-secondary`) for 1.5 seconds, then reverts to `[Save Changes]`. `onSaved` is called with the current `watch.id`.
5. On API error: same inline error message pattern as AC 5.

### AC 7 — Cancel behavior
**New-watch mode cancel:** Clicking `[Cancel]` calls the `onCancel()` prop callback. `WatchConfigPage` handles this by clearing `isCreatingNew`, setting `selectedWatchId` to `null`, returning the surface to "no selection" state (mode = `'none'`).

**Edit-watch mode cancel:** Clicking `[Cancel]` resets all form fields back to the values from `watch` prop (the last successfully fetched watch data). No API call is made. No parent notification required. The form re-populates to the saved state as if the watch was just re-selected.

### AC 8 — Delete zone placeholder
Below the Save/Cancel row, a dashed divider line and a reserved layout block are rendered:
```
──── ──── ──── ──── ──── ──── ──── ──── ──── ──── ──── ───

[delete zone — Story 2.5]
```
The placeholder renders at 11px `--text-tertiary`: `"Watch actions — coming in Story 2.5"`. The `<div>` container has a data attribute `data-slot="delete-zone"` so Story 2.5 can target and replace it. This block renders only in `mode === 'selected'` — not in new-watch mode.

### AC 9 — Entity boundary thumbnail placeholder
Below the Geography Entity selector (in both new and edit modes), a fixed-height container (`height: 96px`) renders with `--bg-raised` background, `1px solid --border-subtle` border, `border-radius: 4px`. Inside: centered placeholder text at 11px `--text-tertiary`: `"Entity boundary — available in a future story"`. A `data-slot="entity-thumbnail"` attribute is on the outer container. This is the reserved slot for when entity geometry is exposed by the API.

### AC 10 — Geography entity display in edit mode
In edit-watch mode (mode = `'selected'`), the Geography Entity field renders as static text (the resolved entity name from `entityMap`), not an editable `<select>`. It is visually distinct from an active input — rendered at 13px `--text-secondary`, inside a container styled like a disabled field (`--bg-surface` background, `1px solid --border-subtle`, `border-radius: 4px`, `padding: 8px 12px`). A small lock icon or `(locked)` text at 11px `--text-tertiary` appears to its right or below, clarifying the field is read-only. No tooltip or popover is needed.

### AC 11 — Signal families — at least one required
If the operator unchecks all signal family checkboxes, the Save button becomes disabled and a small validation message appears: `"At least one signal family must be enabled."` Re-checking any checkbox re-enables Save. This prevents saving a watch with no detection families.

### AC 12 — Form body scrolls; sticky header is preserved
The form body (`<div>` below the sticky header) must scroll independently when content exceeds the column height. The sticky header (`position: sticky; top: 0`) remains visible during body scroll — this behavior was established in 2.2 and must not regress. No page-level scroll.

### AC 13 — `npm run build` passes
`tsc -b && vite build` completes without TypeScript errors after all new files are added and modified files are updated.

### AC 14 — Smoke test passes locally
```bash
# Terminal 1: API + infra
docker compose -f docker-compose.yml -f docker-compose.dev.yml up postgres valkey -d
uvicorn services.api.api.main:app --reload --host 0.0.0.0 --port 8000 --env-file .env

# Terminal 2: Cockpit
cd services/cockpit
npm run dev

# Smoke test checklist:
# → Navigate to /watches
# → Click [+ New] → form renders with all fields blank/default
# → Fill in name, select entity, verify all 8 signal families checked, verify threshold=65 / window=30
# → Click [Save Changes] → loading state → 201 response → new watch appears in list, is selected
# → Correct header shows new watch name in uppercase
# → Click the new watch in the list → form populates with its values
# → Change escalation threshold → slider + input stay in sync
# → Click [Cancel] → form reverts to saved values
# → Change a field → Click [Save Changes] → ✓ Saved confirmation → list row updated
# → Verify geography entity field is read-only (static text) in edit mode
# → Uncheck all signal families → Save button disables → re-check one → Save re-enables
# → npm run build → no TypeScript or Vite errors
```

---

## Non-Goals (Explicit Scope Boundaries)

| Item | Deferred To |
|---|---|
| Compatible vs. incompatible change detection | Story 2.4 |
| `[Save & Reset Baseline]` button label variant | Story 2.4 |
| Per-field incompatible-change warning banners | Story 2.4 |
| Baseline-reset confirmation flow | Story 2.4 |
| `[Disable]` button in form header | Story 2.5 |
| Inline disable confirmation | Story 2.5 |
| `[Enable]` toggle | Story 2.5 |
| `delete watch` text-link and confirmation block | Story 2.5 |
| Status badges (colored left stripes, pill labels) on watch list rows | Story 2.6 |
| Baseline progress bars on watch list rows | Story 2.6 |
| Signal count on watch list rows | Story 2.6 (no signals until Epic 5) |
| Entity boundary thumbnail (requires geometry in geography API response) | Future story |
| Geography entity editing in edit mode (requires WatchUpdate schema change + 2.4 incompatible-change flow) | Story 2.4 |
| Live Domain Pulse dots | Epic 3 |
| Any backend / API / schema / migration changes | Stories 1.2, 2.1 (done) |
| VPS / nginx / production Docker scope | Stories 1.4–1.6 (deferred) |
| react-hook-form, zod, or any form library | Not in scope for v1 |
| Modal-based UX | Never in v1 (UX rule A6) |

---

## Story Boundary Callouts — 2.3 vs. 2.4 / 2.5 / 2.6

### 2.3 → 2.4 (Compatible vs. Incompatible Change Detection)

This is the sharpest and most consequential boundary to enforce.

**What 2.3 does:** Simple save — one save path, always labeled `[Save Changes]`, no field change categorization.

**What 2.4 adds:** A change-detection layer that intercepts the save flow and branches based on whether changed fields are "compatible" (threshold — no warning) or "incompatible" (geography, signal families, baseline window — triggers warning banner + `[Save & Reset Baseline]` label change).

**How to keep 2.4 from leaking into 2.3:**
- The save handler in 2.3 is a single `handleSave()` function inside `WatchFormPanel`. **Do not add any field-comparison logic** — no "was this field changed?", no "is this change incompatible?", no `isIncompatibleChange` boolean. None of that belongs in 2.3.
- The save button label in 2.3 is **always** the string literal `"Save Changes"`. Do not make it a state variable or derive it from field state. Story 2.4 adds that label-switching logic.
- 2.4 adds its incompatible change detection by adding a call to a `useIncompatibleChangeDetection(formState, originalWatch)` hook inside `WatchFormPanel`. This hook does not exist in 2.3 — 2.4 creates and wires it. The seam is: 2.4 modifies `WatchFormPanel` to add this hook and use its output to conditionally set the button label and show warning banners. Zero prop changes from `WatchConfigPage`.
- **Risk to watch for:** If the dev adds any comment like `// TODO: check if incompatible` or any `if (watch?.baseline_window_days !== formState.baselineWindowDays)` logic in 2.3, that is scope creep. Remove it.

### 2.3 → 2.5 (Disable / Delete Hierarchy)

**What 2.3 does:** Reserves the `rightSlot` prop in `WatchFormPanel` header (established in 2.2, maintained here). Reserves the "delete zone" layout slot at the bottom of the form body (AC 8 above).

**What 2.5 adds:** Populates `rightSlot` with `<DisableButton />` from `WatchConfigPage`. Replaces the `data-slot="delete-zone"` placeholder with the real delete text-link and confirmation block.

**How to keep 2.5 out of 2.3:**
- The `rightSlot` prop already exists and defaults to `null`. Do not add a `[Disable]` button, `[Enable]` toggle, or any inline confirmation UX — those are 2.5's entire story.
- The delete zone placeholder (AC 8) is a visual stub only. No click handlers, no mutation, no `window.confirm`. Just the placeholder text and the reserved container.
- `WatchFormPanel` must NOT contain `onDisable`, `onDelete`, or any disable/delete-related props or callbacks in 2.3.

### 2.3 → 2.6 (Watch List Visual Richness)

**What 2.3 does:** No changes to `WatchListRow` or `WatchListColumn`. The watch list remains exactly as established in 2.2.

**What 2.6 does:** Adds status badge components, colored left stripes, baseline progress bars to `WatchListRow`. Uses the existing `badge?: React.ReactNode` prop slot.

**Risk:** After a watch save in 2.3, `['watches']` is invalidated and the list re-renders. The re-render must still use the plain-text status fallback from 2.2 (not a badge). Do not add any status styling or badge rendering in 2.3 — just verify the list re-fetches and re-renders correctly with existing 2.2 row components.

---

## SM Ambiguity Flags

### Ambiguity 1: Geography entity — editable in edit mode?

**The conflict:** The UX spec (§8.3) shows a searchable geography entity dropdown in the form body. The UX spec (§8.4) classifies geography change as an incompatible change. However, the backend `WatchUpdate` Pydantic model (confirmed in Story 2.1) **does not include `geography_entity_id`** — it is absent from the PATCH endpoint body.

**Resolution for 2.3:** Geography entity is **read-only in edit mode**. Render it as static text showing the current entity name. Do not wrap it in a `<select>` for edit mode. For **new-watch mode**, geography entity IS a `<select>` dropdown (required field for `POST`).

**Why:** Enabling geography change in edit mode requires two things that are not in 2.3 scope: (1) a `WatchUpdate` schema change to add `geography_entity_id` (backend work), and (2) the incompatible-change UX flow from Story 2.4. Both gates must be open before geography entity editing is meaningful in the form.

**Downstream:** Story 2.4 should address this explicitly when implementing incompatible change detection. If geography entity editing is desired (matching the UX spec), Story 2.4's backend scope may need to add `geography_entity_id` to `WatchUpdate`, or a separate `PATCH` body variant for incompatible changes.

### Ambiguity 2: Does submit (create/update) fully belong in 2.3?

**The question:** Should 2.3 wire form save to the actual `POST`/`PATCH` API, or just implement the form UI and leave API wiring for a later story?

**Resolution for 2.3:** Yes — full API wiring belongs in this story. The API endpoints are live (Story 2.1 done), the form fields map cleanly to the request shapes, and the watch list needs to update after save. Deferring the API call would leave the form non-functional and create an additional story that only wires two REST calls — not worth the overhead.

**Scope boundary on the wiring:** The 2.3 save path is a simple `POST` / `PATCH` with no pre-save intercept logic. Story 2.4 adds a pre-save incompatible-change gate. This is clearly additive: 2.4 wraps the existing save handler with additional checks. The 2.3 handler remains in place; 2.4 calls it after confirmation.

### Ambiguity 3: Where do form validation boundaries stop in 2.3?

**What validation is in 2.3:**
- `name`: required (non-empty after trim) — client-side, prevent submit
- `geography_entity_id`: required for new watches — client-side, prevent submit
- `escalation_threshold`: 1–100 range — clamp on blur; no hard block on Save (clamping handles it)
- `baseline_window_days`: 7–90 range — clamp on blur
- `enabled_signal_families`: at least 1 selected — client-side, disable Save button

**What validation is NOT in 2.3:**
- "Is this change incompatible?" — 2.4 only
- Cross-field validation (e.g., baseline window only meaningful for certain entity types) — not in v1
- Server-side error codes for duplicate watch names — surface the API's error message inline; no special client-side handling

**Rule:** If a validation check requires comparing the new form value to the previously saved watch value, it does not belong in 2.3. That comparison pattern belongs to 2.4's change-detection system.

### Ambiguity 4: "Optimistic update" — full or partial?

The UX spec §11.6 says "Watch form save: Optimistic update. UI updates immediately; reverts on server error with inline error message."

True optimistic update would mean: update the `['watches']` TanStack Query cache entry immediately (before the API call resolves), then roll back if the API returns an error.

**Resolution for 2.3:** Implement a **simplified success-path pattern** instead of full optimistic update:
- Disable Save button and show `"Saving…"` during the request
- On success: invalidate `['watches']` (triggers re-fetch, list updates from server) and show `✓ Saved`
- On error: show inline error, re-enable Save

True optimistic cache mutation adds complexity (need to snapshot and rollback the cache) that is not necessary at this stage — there is no real-time concurrent update risk with a single operator. The "optimistic" UX goal (fast feedback) is achieved by the `✓ Saved` confirmation and the immediate list re-fetch.

---

## Tasks / Subtasks

- [ ] Task 1: Create mutation hooks (AC: #5, #6)
  - [ ] 1.1 Create `services/cockpit/src/hooks/useCreateWatch.ts`: exports `useCreateWatch()` — `useMutation({ mutationFn: createWatch })` where `createWatch` calls `POST /api/v1/watches` via `apiClient`. Returns the created `WatchRow`. `onSuccess`: call `queryClient.invalidateQueries({ queryKey: ['watches'] })`.
  - [ ] 1.2 Create `services/cockpit/src/hooks/useUpdateWatch.ts`: exports `useUpdateWatch()` — `useMutation({ mutationFn: updateWatch })` where `updateWatch` calls `PATCH /api/v1/watches/{id}` via `apiClient`. Returns the updated `WatchRow`. `onSuccess`: call `queryClient.invalidateQueries({ queryKey: ['watches'] })`.

- [ ] Task 2: Expand `WatchFormPanel.tsx` — props and structure (AC: #1, #2, #10, #12)
  - [ ] 2.1 Add/update props interface: drop `watchName?: string`; add `watch?: WatchRow`, `entities: GeographyEntityRow[]`, `onSaved?: (watchId: string) => void`, `onCancel?: () => void`. Preserve `mode: 'none' | 'new' | 'selected'` and `rightSlot?: ReactNode`.
  - [ ] 2.2 Add form state: `name`, `geographyEntityId`, `enabledSignalFamilies`, `escalationThreshold`, `baselineWindowDays`, `justSaved`, `saveError`. Use `useState` only — no form library.
  - [ ] 2.3 Add `useEffect` to reset form state when `watch` or `mode` changes: if `mode === 'new'` → reset to blank defaults; if `mode === 'selected'` and `watch` changes → populate from `watch` fields.
  - [ ] 2.4 Derive sticky header label from `watch?.name ?? ''` when `mode === 'selected'`; `'NEW WATCH'` when `mode === 'new'`. Remove previous `watchName` string prop dependency.
  - [ ] 2.5 Ensure `rightSlot ?? null` still renders correctly in the header — no regression on the 2.5 extension seam.

- [ ] Task 3: Implement form fields in `WatchFormPanel.tsx` body (AC: #1, #2, #3, #4, #9, #10, #11)
  - [ ] 3.1 Watch Name field: `<label>` + full-width `<input type="text">` with placeholder "Enter watch name". On change: update `name` state. Required for both new and edit.
  - [ ] 3.2 Geography Entity field — new mode: `<label>` + `<select>` dropdown populated from `entities` prop. First option: `<option value="">Select entity…</option>`, then one `<option>` per entity (value = `entity.id`, label = `entity.name`). Controlled by `geographyEntityId` state.
  - [ ] 3.3 Geography Entity field — edit mode: `<label>` + static text container (disabled-style div) showing resolved entity name from `entities` prop lookup by `watch.geography_entity_id`. Show `(locked)` label at 11px `--text-tertiary` (AC 10).
  - [ ] 3.4 Entity boundary thumbnail placeholder (AC 9): `<div data-slot="entity-thumbnail">` with height 96px, `--bg-raised` bg, `1px solid --border-subtle` border, border-radius 4px, centered placeholder text.
  - [ ] 3.5 Signal Families section: `<label>` heading + 8 checkbox rows using `SIGNAL_FAMILIES` constant (see Dev Notes). Each row: `<input type="checkbox">` + `<label>`. Controlled by `enabledSignalFamilies` state (array of string values).
  - [ ] 3.6 Escalation Threshold: `<label>` + flex row with `<input type="range" min={1} max={100} step={1}>` + `<input type="number" min={1} max={100}>`. Sync both onChange. Helper text below.
  - [ ] 3.7 Baseline Window: `<label>` + `<input type="number" min={7} max={90}>` + inline `"days"` label. Clamp to [7, 90] on blur (AC 4). Helper text below.
  - [ ] 3.8 Signal family at-least-one validation: compute `isSignalFamiliesEmpty = enabledSignalFamilies.length === 0`; pass to save button disabled logic (AC 11).

- [ ] Task 4: Implement Save / Cancel row and delete zone placeholder (AC: #5, #6, #7, #8)
  - [ ] 4.1 Save/Cancel row: right-aligned flex container. `[Save Changes]` primary button (disabled when `isPending || name.trim() === '' || (mode === 'new' && !geographyEntityId) || isSignalFamiliesEmpty`). Shows `"Saving…"` label when `isPending`.
  - [ ] 4.2 Cancel button: disabled when `isPending`. Calls form-level cancel handler (see 4.4).
  - [ ] 4.3 `justSaved` state: after successful update mutation, set `justSaved = true` → show `"✓ Saved"` (green/nominal color) in place of `[Save Changes]` → `setTimeout(() => setJustSaved(false), 1500)`.
  - [ ] 4.4 Cancel handler: if `mode === 'new'` → call `onCancel()`; if `mode === 'selected'` → reset all form state fields back to `watch` prop values (no API call, no parent callback).
  - [ ] 4.5 Inline error display: `{saveError && <p style={{ color: 'var(--accent-degraded)', fontSize: 12 }}>{saveError}</p>}` below Save/Cancel row.
  - [ ] 4.6 Delete zone placeholder (AC 8): render only in `mode === 'selected'`, below Save/Cancel row. Dashed divider + `<div data-slot="delete-zone">` with placeholder text.

- [ ] Task 5: Implement save mutation calls in `WatchFormPanel.tsx` (AC: #5, #6)
  - [ ] 5.1 Call `useCreateWatch()` and `useUpdateWatch()` inside `WatchFormPanel`.
  - [ ] 5.2 Implement `handleSave()`: if `mode === 'new'` → call `createMutation.mutate(createPayload)` where payload is `{ name: name.trim(), geography_entity_id: geographyEntityId, enabled_signal_families: enabledSignalFamilies, escalation_threshold: escalationThreshold, baseline_window_days: baselineWindowDays }`.
  - [ ] 5.3 Create mutation `onSuccess`: `onSaved?.(data.id)` with the returned watch ID.
  - [ ] 5.4 Create mutation `onError`: set `saveError` state from the error message.
  - [ ] 5.5 Implement `handleSave()` update path: if `mode === 'selected'` → call `updateMutation.mutate({ id: watch.id, body: { name: name.trim(), enabled_signal_families: enabledSignalFamilies, escalation_threshold: escalationThreshold, baseline_window_days: baselineWindowDays } })`. **Never include `geography_entity_id` in the update body.**
  - [ ] 5.6 Update mutation `onSuccess`: `setJustSaved(true)` + `setTimeout` to clear; `onSaved?.(watch.id)`.
  - [ ] 5.7 Update mutation `onError`: set `saveError` state.
  - [ ] 5.8 Clear `saveError` state whenever form fields change (so stale error doesn't show after user corrects input).

- [ ] Task 6: Update `WatchConfigPage.tsx` (AC: #2, #5, #6, #7)
  - [ ] 6.1 Pass `watch={selectedWatch ?? undefined}` and `entities={entities ?? []}` to `WatchFormPanel`.
  - [ ] 6.2 Add `handleWatchSaved(watchId: string)`: sets `selectedWatchId = watchId`, `setIsCreatingNew(false)`. Covers both create (select new watch) and update (keep selected watch, re-renders with fresh data after cache invalidation).
  - [ ] 6.3 Add `handleFormCancel()`: sets `isCreatingNew(false)`. (For edit mode cancel, the form handles it internally — `onCancel` only needs to handle the new-mode case at the page level, but passing it uniformly is fine.)
  - [ ] 6.4 Pass `onSaved={handleWatchSaved}` and `onCancel={handleFormCancel}` to `WatchFormPanel`.
  - [ ] 6.5 Remove `watchName` prop from `WatchFormPanel` call site (replaced by `watch` prop).

- [ ] Task 7: TypeScript compile + smoke test (AC: #13, #14)
  - [ ] 7.1 `npm run build` — confirm no TypeScript errors
  - [ ] 7.2 Start infra + API + cockpit dev server; run through the full smoke test sequence in AC 14 (Nikku)
  - [ ] 7.3 Verify new-watch create: form → save → new watch appears in list and is selected
  - [ ] 7.4 Verify edit-watch: click watch → form populates → edit a field → save → ✓ Saved confirmation → list updates
  - [ ] 7.5 Verify cancel from new mode: exits to no-selection state
  - [ ] 7.6 Verify cancel from edit mode: form reverts to saved values, watch stays selected
  - [ ] 7.7 Verify slider/input sync on escalation threshold
  - [ ] 7.8 Verify baseline window clamping on blur

---

## Dev Notes

### File Structure — What to Create / Modify

```
services/cockpit/src/
├── hooks/
│   ├── useCreateWatch.ts          ← CREATE
│   └── useUpdateWatch.ts          ← CREATE
├── components/
│   └── watch-config/
│       └── WatchFormPanel.tsx     ← MODIFY (replace placeholder body with real form)
└── pages/
    └── WatchConfigPage.tsx        ← MODIFY (pass watch data + callbacks to WatchFormPanel)
```

**Files NOT touched in this story:**
- `src/layouts/CockpitShell.tsx` — locked; no modifications
- `src/styles/tokens.css` — locked
- `src/lib/api-client.ts` — locked
- `src/components/shell/NavRail.tsx` — locked
- `src/components/watch-config/WatchListColumn.tsx` — no changes
- `src/components/watch-config/WatchListRow.tsx` — no changes
- `src/hooks/useWatches.ts` — no changes
- `src/hooks/useGeographyEntities.ts` — no changes
- `services/api/` — no backend changes
- `migrations/` — no schema changes

---

### Signal Families Constant

Define this constant at the top of `WatchFormPanel.tsx` (or in a separate `constants/watchForm.ts` if preferred). The order here is the canonical display order:

```typescript
const SIGNAL_FAMILIES: { value: string; label: string }[] = [
  { value: 'port_dwell_anomaly',           label: 'Port Dwell-Time Anomaly' },
  { value: 'cargo_route_deviation',        label: 'Cargo Route Deviation' },
  { value: 'chokepoint_throughput_change', label: 'Chokepoint Throughput Change' },
  { value: 'vessel_traffic_density',       label: 'Vessel Traffic Density Anomaly' },
  { value: 'compound_disruption',          label: 'Compound Disruption Pattern' },
  { value: 'commercial_mass_deviation',    label: 'Commercial Route Mass Deviation' },
  { value: 'ais_dark_event',               label: 'AIS Vessel Dark Event' },
  { value: 'gps_jamming_effect',           label: 'GPS Jamming Zone Formation' },
]
```

Default for new watches: `ALL_SIGNAL_FAMILIES = SIGNAL_FAMILIES.map(f => f.value)` — all 8 enabled.

These string values match `SignalFamilyEnum` values in `shared/schema/enums.py` and are stored as `list[str]` in the DB (`ARRAY(Text)` column). Do not import the Python enum — define as string constants in TypeScript.

---

### Mutation Hook Patterns

```typescript
// src/hooks/useCreateWatch.ts
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { apiClient } from '@/lib/api-client'
import type { WatchRow } from '@/hooks/useWatches'

interface CreateWatchPayload {
  name: string
  geography_entity_id: string
  enabled_signal_families: string[]
  escalation_threshold: number
  baseline_window_days: number
}

async function createWatch(payload: CreateWatchPayload): Promise<WatchRow> {
  const res = await apiClient.post<{ data: WatchRow }>('/api/v1/watches', payload)
  return res.data.data
}

export function useCreateWatch() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: createWatch,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['watches'] })
    },
  })
}
```

```typescript
// src/hooks/useUpdateWatch.ts
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { apiClient } from '@/lib/api-client'
import type { WatchRow } from '@/hooks/useWatches'

interface UpdateWatchPayload {
  id: string
  body: {
    name?: string
    enabled_signal_families?: string[]
    escalation_threshold?: number
    baseline_window_days?: number
    // NOTE: geography_entity_id intentionally excluded — WatchUpdate schema does not support it
  }
}

async function updateWatch({ id, body }: UpdateWatchPayload): Promise<WatchRow> {
  const res = await apiClient.patch<{ data: WatchRow }>(`/api/v1/watches/${id}`, body)
  return res.data.data
}

export function useUpdateWatch() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: updateWatch,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['watches'] })
    },
  })
}
```

---

### `WatchFormPanel.tsx` — Updated Props Interface

```typescript
import type { ReactNode } from 'react'
import type { WatchRow } from '@/hooks/useWatches'
import type { GeographyEntityRow } from '@/hooks/useGeographyEntities'

interface WatchFormPanelProps {
  mode: 'none' | 'new' | 'selected'
  watch?: WatchRow                        // populated when mode === 'selected'
  entities: GeographyEntityRow[]          // from useGeographyEntities in WatchConfigPage
  rightSlot?: ReactNode                   // Story 2.5: <DisableButton /> — default null
  onSaved?: (watchId: string) => void     // called after successful create or update
  onCancel?: () => void                   // called when Cancel is clicked in 'new' mode
}
```

**`watchName` prop is removed** — the header label is now derived from `watch?.name` in `mode === 'selected'`. Update the call site in `WatchConfigPage` accordingly.

---

### Form State Reset Pattern

```typescript
// Inside WatchFormPanel
const ALL_DEFAULTS = {
  name: '',
  geographyEntityId: '',
  enabledSignalFamilies: SIGNAL_FAMILIES.map(f => f.value),  // all enabled
  escalationThreshold: 65,
  baselineWindowDays: 30,
}

useEffect(() => {
  if (mode === 'new') {
    setName(ALL_DEFAULTS.name)
    setGeographyEntityId(ALL_DEFAULTS.geographyEntityId)
    setEnabledSignalFamilies(ALL_DEFAULTS.enabledSignalFamilies)
    setEscalationThreshold(ALL_DEFAULTS.escalationThreshold)
    setBaselineWindowDays(ALL_DEFAULTS.baselineWindowDays)
    setSaveError(null)
    setJustSaved(false)
  } else if (mode === 'selected' && watch) {
    setName(watch.name)
    setGeographyEntityId(watch.geography_entity_id)
    setEnabledSignalFamilies([...watch.enabled_signal_families])
    setEscalationThreshold(Math.round(watch.escalation_threshold))
    setBaselineWindowDays(watch.baseline_window_days)
    setSaveError(null)
    setJustSaved(false)
  }
}, [mode, watch?.id])  // key on watch.id — not the whole object — to avoid unnecessary resets
```

The `watch?.id` dependency is intentional. If the same watch is re-fetched (after a successful save invalidates the cache), the effect must NOT re-run and reset the form mid-edit. Using `watch.id` means the effect only re-triggers when a genuinely different watch is selected.

---

### Save Button States

```typescript
// Derived state for Save button
const isPending = createMutation.isPending || updateMutation.isPending
const isNameEmpty = name.trim() === ''
const isEntityMissing = mode === 'new' && !geographyEntityId
const isSignalFamiliesEmpty = enabledSignalFamilies.length === 0
const isSaveDisabled = isPending || isNameEmpty || isEntityMissing || isSignalFamiliesEmpty

const saveLabel = isPending ? 'Saving…' : justSaved ? '✓ Saved' : 'Save Changes'
// Note for Story 2.4: 2.4 adds incompatible change detection and overrides this label
// to 'Save & Reset Baseline' when isIncompatible === true. In 2.3, label is always one
// of the three states above.
```

---

### Form Field Styling (Reference)

All form fields use inline styles with CSS variables (no new CSS files). Follow these conventions:

| Element | Style |
|---|---|
| Section label | `fontSize: 11, textTransform: 'uppercase', letterSpacing: '0.08em', color: 'var(--text-secondary)', marginBottom: 6` |
| Text input / select (active) | `width: '100%', padding: '8px 12px', background: 'var(--bg-raised)', border: '1px solid var(--border-subtle)', borderRadius: 4, color: 'var(--text-primary)', fontSize: 13` |
| Geography entity (read-only display) | Same container dimensions but `background: 'var(--bg-surface)', color: 'var(--text-secondary)'`. No pointer cursor. Add `(locked)` text at `--text-tertiary` 11px below or inline. |
| Checkbox row | `display: flex, alignItems: 'center', gap: 8, padding: '4px 0'`. Label at 13px `--text-primary`. |
| Range slider | Native `<input type="range">`. Min/max styling via CSS variables if needed, but do not add a custom slider library. |
| Numeric input | `width: 72, textAlign: 'center'` for adjacent-to-slider usage. |
| Helper text | `fontSize: 11, color: 'var(--text-tertiary)', marginTop: 4` |
| Primary button (Save) | `padding: '8px 16px', background: 'var(--accent-attention)', border: 'none', borderRadius: 4, color: 'white', fontSize: 13, cursor: 'pointer'`. Disabled: `opacity: 0.45, cursor: 'not-allowed'`. |
| Secondary button (Cancel) | `padding: '8px 16px', background: 'transparent', border: '1px solid var(--border-subtle)', borderRadius: 4, color: 'var(--text-secondary)', fontSize: 13, cursor: 'pointer'`. |
| ✓ Saved text | `fontSize: 13, color: 'var(--accent-nominal)'` |
| Error message | `fontSize: 12, color: 'var(--accent-degraded)', marginTop: 8` |
| Form body padding | `padding: '20px'` overall |
| Field group spacing | `marginBottom: 20` between field groups |
| Dashed divider (delete zone) | `borderTop: '1px dashed var(--border-subtle)', marginTop: 24, marginBottom: 16` |

---

### `WatchConfigPage.tsx` — Updated Call Site

```typescript
// In WatchConfigPage — updated WatchFormPanel usage

const handleWatchSaved = (watchId: string) => {
  setSelectedWatchId(watchId)
  setIsCreatingNew(false)
}

const handleFormCancel = () => {
  setIsCreatingNew(false)
  // Note: if in edit mode, WatchFormPanel handles cancel internally (field reset).
  // This callback only meaningfully changes page state in 'new' mode.
}

// In the render:
<WatchFormPanel
  mode={formMode}
  watch={selectedWatch ?? undefined}
  entities={entities ?? []}
  onSaved={handleWatchSaved}
  onCancel={handleFormCancel}
/>
```

**Remove** the old `watchName={selectedWatch?.name}` prop — it no longer exists. `WatchFormPanel` derives the header label from `watch?.name` internally.

---

### Error Message Extraction from API Response

The API returns errors in the standard Panoptes envelope:
```json
{ "data": null, "error": { "code": "GEOGRAPHY_ENTITY_NOT_FOUND", "message": "..." }, "meta": {...} }
```

Axios will throw on non-2xx responses. Extract the message:
```typescript
const extractErrorMessage = (error: unknown): string => {
  if (error && typeof error === 'object' && 'response' in error) {
    const axiosError = error as { response?: { data?: { error?: { message?: string } } } }
    return axiosError.response?.data?.error?.message ?? 'Unknown error'
  }
  return 'Save failed. Please try again.'
}
```

Use in `onError` callbacks of both mutations. Set the result into `saveError` state.

---

### No New npm Packages

All dependencies already installed from Story 1.8:
- `@tanstack/react-query` — `useMutation`, `useQueryClient`
- `react` / `react-dom` — components, hooks
- `axios` — via `apiClient`

**Do NOT install:**
- `react-hook-form`, `formik`, or any form library
- `zod` or any validation library
- `@radix-ui/react-slider` or any custom slider component
- Any modal or popover library
- `maplibre-gl` — Epic 8 only

Use native HTML `<input type="range">`, `<input type="text">`, `<input type="number">`, `<select>`, `<input type="checkbox">`.

---

### Architecture Compliance Guardrails

1. **Signal Desk is default home.** This story does not touch routing or the Signal Desk surface.

2. **`apiClient` — no baseURL.** All fetch calls use relative `/api/v1/...` paths via Vite proxy.

3. **No localStorage.** Form state lives in React `useState` only.

4. **No modals.** The form is inline in the right panel. No `<dialog>`, no `z-index` overlay, no backdrop.

5. **`rightSlot` must remain functional.** After all `WatchFormPanel` modifications, verify the sticky header still renders `rightSlot ?? null` correctly in the right-aligned position. Story 2.5 depends on this.

6. **`WatchListRow` and `WatchListColumn` are untouched.** The list re-renders after cache invalidation — verify existing 2.2 row behavior (badge prop default, status text fallback) is not broken.

7. **Geography entity ID in PATCH body: never.** This cannot be stated strongly enough. The PATCH endpoint does not accept `geography_entity_id`. Including it would either be silently ignored (if the API uses `exclude_none`) or could cause a validation error. The 2.3 update mutation body is: `{ name, enabled_signal_families, escalation_threshold, baseline_window_days }` only.

8. **`watch?.id` as useEffect dependency, not `watch`.** Using `watch` as the effect dependency would cause an infinite re-render loop if `watches` data is re-fetched after save (new reference, same data). Depend on `watch?.id` only.

9. **Escalation threshold as integer in the UI, float in the API.** The slider uses `step={1}` and the state variable is a number. Send as a number to the API — no `.toFixed()` conversion needed. The API accepts `65` and `65.0` equivalently.

---

### Carry-Forward Intelligence from Stories 2.1 and 2.2

| Item | Impact on Story 2.3 |
|---|---|
| `WatchUpdate` schema has no `geography_entity_id` | Geography entity is READ-ONLY in edit mode — NEVER include it in the PATCH body |
| `POST /api/v1/watches` returns HTTP 201 | `apiClient.post(...)` will resolve normally for 201 — axios treats 2xx as success by default |
| `enabled_signal_families` in API response is `string[]` (not `SignalFamilyEnum[]`) | Use `string[]` in form state; no enum conversion needed in TypeScript |
| `useGeographyEntities()` already called in `WatchConfigPage` with `staleTime: 5min` | Pass `entities` as a prop to `WatchFormPanel` — do NOT call `useGeographyEntities()` again inside `WatchFormPanel` |
| `WatchFormPanel` sticky header uses `position: sticky; top: 0; zIndex: 1` | Form body renders below the header; no sticky positioning on the body div itself — just `overflowY: auto` on the outer right column |
| `rightSlot` defaults to `null` in `WatchFormPanel` | Maintain this default. Pass nothing from `WatchConfigPage` in 2.3. |
| `WatchListRow` has `badge?: ReactNode` slot | Not touched in this story. After cache invalidation, rows re-render with existing 2.2 plain-text status fallback. |
| `apiClient` has `withCredentials: true` | The POST and PATCH calls will include the httpOnly refresh cookie automatically. |

---

### References

- [Source: _bmad-output/planning-artifacts/ux-design-specification.md#§8.3] — Watch form body: geography dropdown, signal family checkboxes, escalation threshold slider, baseline window input, Save/Cancel placement
- [Source: _bmad-output/planning-artifacts/ux-design-specification.md#§8.4] — Compatible vs. incompatible change handling (NOT in 2.3 — defines the 2.4 boundary)
- [Source: _bmad-output/planning-artifacts/ux-design-specification.md#§8.5] — Disable action (NOT in 2.3 — defines the 2.5 boundary)
- [Source: _bmad-output/planning-artifacts/ux-design-specification.md#§8.7] — Delete hierarchy (NOT in 2.3 — defines the 2.5 boundary)
- [Source: _bmad-output/planning-artifacts/ux-design-specification.md#§11.5] — Watch Configuration global rules: incompatible change warning on field change (2.4), domain pulse visible (2.2 done)
- [Source: _bmad-output/planning-artifacts/ux-design-specification.md#§11.6] — Performance: "Watch form save: Optimistic update" (simplified as success-path pattern per SM Ambiguity §4)
- [Source: _bmad-output/planning-artifacts/epics.md#Epic 2] — Story 2.3 scope: "Watch form: geographic entity selection, signal family toggles, escalation thresholds, baseline window parameter"
- [Source: _bmad-output/implementation-artifacts/2-1-watches-api-crud.md#AC#2] — `POST /api/v1/watches` body: `WatchCreate` — name, geography_entity_id, enabled_signal_families, escalation_threshold, baseline_window_days
- [Source: _bmad-output/implementation-artifacts/2-1-watches-api-crud.md#AC#4] — `PATCH /api/v1/watches/{id}` body: `WatchUpdate` — name, enabled_signal_families, escalation_threshold, baseline_window_days (NO geography_entity_id)
- [Source: _bmad-output/implementation-artifacts/2-2-watch-configuration-surface.md#WatchFormPanel] — Sticky header pattern, `rightSlot` prop, `mode` enum — all must be preserved
- [Source: shared/schema/enums.py#SignalFamilyEnum] — 8 signal family values used in `SIGNAL_FAMILIES` constant
- [Source: shared/schema/watches.py#WatchCreate/WatchUpdate] — Confirmed field sets for POST vs PATCH

---

## Dev Agent Record

*(To be filled in by the dev agent upon implementation)*

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

## Change Log

- 2026-03-10: Story 2.3 created by Bob (SM, claude-sonnet-4-6). Prerequisites confirmed: 1.1, 1.2, 1.3, 1.7, 1.8, 2.1, 2.2 all done. SM ambiguities called out: geography edit mode, submit scope, validation boundaries, optimistic update scope. Boundary callouts for 2.4 (incompatible change), 2.5 (disable/delete), 2.6 (list visuals) documented. Status: ready-for-dev.
