# Panoptes UI Decision Memo — Palantir Inspire Analysis
**Date:** 2026-03-10
**Author:** Sally (UX Designer)
**Status:** Approved — applies to Story 1.8 and forward
**Source references:** `references/screenshots/UX-UI/Palantir_Inspire/`
**Baseline spec:** `_bmad-output/planning-artifacts/ux-design-specification.md`

> **Scope:** This is a delta document. It does not replace the UX Design Specification.
> All decisions here either amend specific sections of that spec or confirm patterns already
> implied by it. Where a conflict exists, this memo takes precedence.
>
> **Signal-first doctrine is unchanged.** Signal Desk → Evidence Surface → Spatial View.
> Signal Desk remains the default home surface. Spatial View remains contextual drill-down only.

---

## 1. Borrow

### B1 — Narrower left rail, stronger icon presence

The Palantir left taskbar is an icon-forward strip (~70px, icon-only). Panoptes's 160px labeled
rail is functional but wider than necessary — it leans SaaS-app rather than cockpit instrument.

**Decision:** Reduce left rail from **160px → 120px**. Increase nav icon size to **20px** (from
implied ~16px). Keep text labels — they matter for operator surface orientation — but reduce
horizontal padding from 14px to 10px. The 40px freed reallocates to the signal list column.

Cascading layout adjustments:
- Signal Desk: signal list grows from 896px → **936px**
- Evidence Surface: 65% panel width unchanged; compressed Signal Desk grows proportionally
- Spatial View: `[120px rail | 320px dossier | ~1000px map]` at 1440px viewport
- Watch Configuration: `[120px rail | 320px watch list | ~1000px form area]`

---

### B2 — Deeper bg-base color step

The Palantir references use background blacks in the `#08–0d` hex range throughout. Panoptes's
`--bg-base: #0e1117` reads as dark but not quite cockpit-grade. The surface step feels
insufficiently elevated against it.

**Decision:** Shift one token only:
- `--bg-base: #0e1117` → **`#0a0e17`**

`--bg-surface` stays at `#161c27`. The contrast step between base and surface increases from
~8 luminance units to ~14 — panels read as more clearly elevated, the signal list lifts from
the background. All other color tokens unchanged.

---

### B3 — Left rail active state: icon + label color lift

Palantir's left rail communicates active state through icon color lift (muted → bright) in
addition to background treatment. Panoptes's current spec (`§4.2`) specifies only a 3px left
edge bar for active state. Icon and label color in the active case are unspecified, defaulting
to the same as inactive — which makes the active surface ambiguous at peripheral glance.

**Decision:** Keep the `3px solid --accent-attention` left edge bar. Additionally:
- Active nav item: icon color **`--text-primary`** (#e2e8f0), label color **`--text-primary`**
- Inactive nav items: icon color `--text-secondary` (#8896aa), label color `--text-secondary`
- Active row background: `--bg-raised` (unchanged from existing spec)

Active surface is now unambiguous without requiring focus on the left edge bar alone.

---

### B4 — Consistent collapsible section headers throughout Evidence panel

The Palantir side card uses `▼ SECTION NAME` as a universal pattern — every named block in the
detail panel is collapsible and follows the same header treatment. In the Panoptes Evidence
Surface (`§§6.3–6.5`), Contributing Evidence Layers already use `▶`/`▼` chevrons, but
Confidence Breakdown and Constituent Signals are rendered as flat, always-open sections.

**Decision:** All named section blocks in the Evidence panel body follow a single collapsible
header pattern:

```
▼  CONFIDENCE              (default: open)
▼  CONSTITUENT SIGNALS     (default: open — compound only)
▼  CONTRIBUTING EVIDENCE   (default: open)
```

Header format: `▶/▼` chevron + 11px uppercase tracked `--text-secondary` label. Click toggles
collapse state. Section content area animates height 200ms ease-out on open, 150ms ease-in on
close. The information within each section is unchanged from the original spec — this is visual
consistency only.

---

### B5 — Map entity tooltip dimensions (confirmed and codified)

The Palantir map popup cards are compact (~240px wide), anchored directly to the map feature,
show entity name + type + 3–4 key-value rows, and dismiss on click-away with no action buttons.
The Panoptes Spatial View spec (`§7.3`) describes tooltip behavior but does not specify
dimensions or visual treatment. The references confirm the right scale.

**Decision:** Codify Spatial View entity tooltip spec:
- **Width:** 240px
- **Background:** `--bg-raised` (`#1c2333`)
- **Border:** `1px solid --border-subtle`
- **Border-radius:** `4px` (tighter than panel 6px — callout, not panel)
- **Padding:** 10px top/bottom, 12px left/right
- **Content order:** entity name (14px/500/`--text-primary`) → type label
  (11px/400/`--text-secondary`) → key-value rows (12px `--text-secondary` label /
  12px IBM Plex Mono `--text-primary` value)
- **Anchor:** 8px offset above or beside the feature dot; auto-flip to stay within viewport
- **Dismiss:** click-away anywhere on map canvas
- **No action buttons.** No "Explore" CTA. Read-only.

---

### B6 — Bottom timeline zone reservation in Spatial View

The Palantir timeline is a collapsible bottom strip (~120–140px expanded, ~32px collapsed
header) docked to the bottom of the map area — not full-width, not modal. Panoptes does not
plan playback in v1 but the Spatial View layout will need surgery later if space is not
reserved now.

**Decision:** Add a **32px reserved zone** at the bottom of the Spatial View map area. In v1
this zone renders as background `--bg-base` only — no content, no label, no border, no
component. The map canvas height is reduced by 32px accordingly. When Timeline (v1.5+) is
implemented, it expands from this reserved zone upward. The zone is invisible in v1 — it is
structural layout space only.

---

## 2. Avoid

### A1 — Globe-first or map-first default surface

Every Palantir reference opens to a 3D globe or flat map as the primary content area with
a list sidebar. This is the geospatial-explorer pattern.

**Rule:** Panoptes default surface is **Signal Desk** (ranked signal list). Signal Desk is
always the first surface rendered on session start. No globe. No map. The operator's first
question is answered before any spatial context is needed.

---

### A2 — General map explore behavior

The Palantir paradigm allows clicking anywhere on the map to inspect arbitrary features. There
is no signal context required. This is surveillance-browse, not signal investigation.

**Rule:** Panoptes Spatial View is **signal-scoped only**. The map loads to `[⊞ fit to entity]`
for the active signal's geography entity. No free-browse mode. No click-anywhere-to-explore.
No address or coordinate search bar on the map. The Spatial View is a drill-down instrument,
not a GIS workspace.

---

### A3 — Classification banner

The vivid classification strip ("UNCLASSIFIED // NOTIONAL DATA") across the top chrome of
every Palantir reference is a US government/IC operational compliance requirement.

**Rule:** No classification marking UI in Panoptes. Not as a feature, not as an aesthetic
reference. Do not replicate this pattern even as a visual nod.

---

### A4 — OS-style top menu bar

The File / Edit / View / Support dropdown menu bar is a thick-client desktop application
pattern that also occupies a second horizontal chrome band above the map.

**Rule:** No menu bar. Chrome bar (`§4.1`) is the only top chrome element — 36px, single row.

---

### A5 — Multi-situation tab strip

The breadcrumb tab strip for switching between parallel investigation "situations" is a
multi-object workflow pattern for teams managing concurrent active operations.

**Rule:** No tab strip. Panoptes is a single-operator, single-context cockpit. The Chrome bar
contains exactly: platform mark · UTC clock · Domain Pulse strip · session badge.
Nothing else is added to the Chrome bar in v1.

---

### A6 — Centered modal interruption for signal review

The Detection Alert modal (fullscreen dimmed overlay, centered card with Confirm/Reject
action buttons) blocks operator context and forces a binary decision interrupt.

**Rule:** No modals for signal review or dismissal. The dismiss workflow is inline in the
Evidence panel sticky bottom section — requires reason text, non-blocking until explicitly
submitted. No modals anywhere in the v1 cockpit.

---

### A7 — Media thumbnails and satellite imagery in detail panels

The Palantir side card shows harbor photographs and satellite imagery crops as evidence
supporting detection findings. This is appropriate for a satellite tasking platform with
imagery as primary evidence.

**Rule:** No imagery thumbnails or media in Panoptes panels. Evidence is expressed as
structured key-value rows: signals, baseline snapshots, provenance records. No visual media.

---

### A8 — Activity trend bar charts embedded in dossier sections

The "Detected Vessels" and "Average days underway per month" bar charts embedded within
the collapsible ACTIVITY section of the Palantir dossier are BI-dashboard elements.

**Rule:** Panoptes communicates statistical deviation through the confidence formula tree
and raw evidence field values (dwell delta, z-score normalized). No embedded bar charts,
sparklines, or trend graphs in v1.

---

### A9 — Multiple floating panels over the map

The Palantir example showing "EB8 Feed" and "Capture angle" as independent floating panels
simultaneously overlaid on the map is a multi-panel chaos pattern.

**Rule:** Panoptes Spatial View shows **one entity tooltip at a time**. No floating panels
stacked over the map canvas.

---

### A10 — Action CTA buttons in dossier panel headers

The "Explore plans" and "Project ship location" primary action buttons appearing in the
dossier header zone of the Palantir detail card.

**Rule:** The only interactive navigation element in the Spatial View left dossier header
is `← Evidence` (back nav link). No action CTAs in panel headers. Actions — if needed in
future surfaces — belong in designated action zones at the bottom of the panel, not the header.

---

## 3. Story 1.8 Changes

Story 1.8 implements the Panoptes cockpit shell: Chrome bar, left nav rail, surface routing,
and layout zone scaffolding for all four surfaces. The following are concrete spec changes
that apply to Story 1.8.

---

### C1 — Left rail width and icon sizing

**Amends §4.2**

| Attribute | Previous spec | Updated spec |
|---|---|---|
| Rail width | 160px | **120px** |
| Nav item icon size | (unspecified, ~16px) | **20px** |
| Horizontal padding | 14px | **10px** |
| Label | 12px/500, `--text-secondary` | Unchanged |

No other nav rail behavior changes.

---

### C2 — Color token: --bg-base

**Amends §3.1**

| Token | Previous value | Updated value |
|---|---|---|
| `--bg-base` | `#0e1117` | **`#0a0e17`** |

All other color tokens in §3.1 are unchanged.

---

### C3 — Left rail active state: icon and label color

**Amends §4.2**

Active nav item state: three simultaneous signals applied together:
1. `3px solid --accent-attention` left edge bar (unchanged)
2. Row background: `--bg-raised` (unchanged)
3. **Icon color: `--text-primary`** (new)
4. **Label color: `--text-primary`** (new)

Inactive nav item: icon color `--text-secondary`, label color `--text-secondary`.

---

### C4 — Bottom timeline zone reservation in Spatial View layout

**Amends §7.1**

Updated Spatial View layout:

```
[Chrome — fixed 36px]
[Nav rail — 120px fixed] | [Left dossier — 320px fixed] | [Map canvas — fills remaining × (viewport_h − 36px − 32px)]
                                                          [Timeline zone — 32px — reserved, v1: empty, bg-base]
```

The timeline zone is a layout slot only in v1. It renders as `background: var(--bg-base)`,
no content, no border, no text. Map canvas height is reduced by 32px. The zone is not
interactive and has no hover state. It must not suggest to the operator that it is a
forthcoming feature — it is invisible.

---

### C5 — Map entity tooltip dimensions

**Adds to §7.3**

Spatial View entity tooltip (vessel dot click, track click):
- Width: `240px`
- Background: `--bg-raised`
- Border: `1px solid --border-subtle`
- Border-radius: `4px`
- Padding: `10px 12px`
- Content: entity name (14px/500/`--text-primary`) → type (11px/400/`--text-secondary`)
  → key-value rows (12px `--text-secondary` / 12px IBM Plex Mono `--text-primary`)
- Anchor: 8px offset from feature; auto-flip if near viewport edge
- Dismiss: click-away on map canvas

---

### C6 — Evidence panel: universal collapsible section headers

**Amends §§6.3, 6.4, 6.5**

All named content sections in the Evidence panel body use the same collapsible header:

```
▼  SECTION NAME        ← open state
▶  SECTION NAME        ← collapsed state
```

- Chevron: 12px, `--text-tertiary`
- Label: 11px uppercase tracked, `--text-secondary`
- Click target: full-width header row, cursor pointer, hover background `--bg-raised` (100ms)
- Animate: height transition 200ms ease-out (open), 150ms ease-in (close)
- Default states: CONFIDENCE open · CONSTITUENT SIGNALS open · CONTRIBUTING EVIDENCE open

Section content within each block is unchanged from the original spec.

---

*End of memo.*
