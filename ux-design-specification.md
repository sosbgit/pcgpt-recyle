---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
status: complete
inputDocuments:
  - _bmad-output/planning-artifacts/product-brief-panoptes-2026-03-07.md
  - _bmad-output/planning-artifacts/prd.md
  - _bmad-output/planning-artifacts/architecture.md
  - _bmad-output/planning-artifacts/brainstorming-report.md
  - _bmad-output/project-context.md
workflowType: ux-design
completedAt: 2026-03-08
---

# UX Design Specification — Panoptes v1

**Author:** Nikku
**Date:** 2026-03-08
**Status:** Complete — approved through iterative wireframe review

---

## Table of Contents

1. [UX Vision & Principles](#1-ux-vision--principles)
2. [Platform Context](#2-platform-context)
3. [Visual Language](#3-visual-language)
4. [Navigation Model](#4-navigation-model)
5. [Signal Desk — Horizon View](#5-signal-desk--horizon-view)
6. [Evidence Surface](#6-evidence-surface)
7. [Spatial View](#7-spatial-view)
8. [Watch Configuration](#8-watch-configuration)
9. [System States](#9-system-states)
10. [Component Reference](#10-component-reference)
11. [Global Interaction Rules](#11-global-interaction-rules)

---

## 1. UX Vision & Principles

### 1.1 Design Vision

Panoptes is a single-operator movement intelligence cockpit. The interface must communicate confidence, not chaos. The operator opens the cockpit and immediately knows: *what did the system find, how certain is it, and what needs my attention.* Everything else is drill-down.

The visual language is operator-grade: dense, credible, calm. Not a consumer dashboard, not a SaaS product, not a news aggregator. Closer to a financial terminal or a mission operations display — except the primary data is signals about geographic entities, not prices or assets.

### 1.2 Core Principles

**Signal-first, not map-first.**
The default experience is a ranked list of findings. The map is reached by drilling into evidence. The operator never opens the cockpit to an empty globe.

**Conclusion before evidence.**
The signal card states what the system found — family, entity, confidence, age. Evidence is behind a click. The operator decides whether to investigate, not the UI.

**Explicit state, never silence.**
NOMINAL is a computed affirmative result, not the absence of alerts. Baseline-in-progress is a visible state, not a blank row. Feed degradation is surfaced immediately, not discovered by inference. No state in the system is represented by emptiness or absence of UI.

**Density over decoration.**
Whitespace is earned by content hierarchy, not added for visual comfort. A compact row communicates as much as a large card when the hierarchy is clear. Every pixel of vertical space in the signal list is operator working memory.

**Ambient liveness, without noise.**
The interface communicates that the system is running — a live clock, a heartbeat dot, a session delta badge — without animation that competes for attention. Updates arrive without fanfare. Urgency is communicated by zone placement and confidence score, not motion or color saturation.

**Drill-down as the primary interaction.**
The main flow is always: Signal Desk → Evidence Surface → (optional) Spatial View. Every surface deepens context on a specific signal. No surface is a general browser.

**Operator trust through provenance.**
Every finding is traceable to raw sources. The operator can always ask "why does the system think this?" and get a machine-readable, human-inspectable answer. Confidence is not a black box.

### 1.3 What This Interface Is Not

- A map application with signals as overlays
- A news aggregator or activity feed
- A multi-user collaboration tool
- A mobile or responsive web application
- A general-purpose geospatial explorer
- A BI dashboard with charts and metrics

---

## 2. Platform Context

**Target viewport:** 1440 × 900px minimum. No responsive breakpoints. Desktop browser only (Chrome, Firefox, Edge — modern evergreen).

**Input model:** Mouse and keyboard. No touch optimization required in v1.

**Authentication:** Single operator, JWT session. Login screen on expired token. No user management surface.

**Deployment:** Self-hosted, Contabo VPS, served via nginx. TLS may be self-signed. No offline mode.

**Real-time delivery:** WebSocket push from API. React Query cache invalidation on WebSocket events. No manual refresh required or encouraged.

**Rendering stack:** React 18 + TypeScript 5 + Vite 5. Map: MapLibre GL JS. State: React Query (server) + Zustand (UI). CSS: Tailwind + shadcn/ui.

---

## 3. Visual Language

### 3.1 Color System

All values are CSS custom properties. Dark theme only — no light mode in v1.

| Token | Hex | Use |
|---|---|---|
| `--bg-base` | `#0e1117` | Application background. Near-black with slight blue tint. |
| `--bg-surface` | `#161c27` | Panel backgrounds, card backgrounds. |
| `--bg-raised` | `#1c2333` | Elevated rows, hover states, open dropdowns. |
| `--border-subtle` | `#263047` | Panel borders, row dividers, horizontal rules. |
| `--border-muted` | `#1e2a3a` | Interior component borders (inputs, thumbnails). |
| `--text-primary` | `#e2e8f0` | Main readable text. Labels, signal names, values. |
| `--text-secondary` | `#8896aa` | Supporting text. Metadata, timestamps, labels. |
| `--text-tertiary` | `#4d5e73` | De-emphasized. Hash values, disabled states, ghost text. |
| `--accent-attention` | `#f59e0b` | Attention Required zone, amber. Left border on AR cards. Session badge when N > 0. |
| `--accent-critical` | `#ef4444` | Feed stale (red). High-confidence compound. |
| `--accent-monitoring` | `#64748b` | Monitoring zone label, steel. Left border on monitoring cards. |
| `--accent-nominal` | `#16a34a` | NOMINAL badge. Muted green — not bright. |
| `--accent-baseline` | `#3b82f6` | Baseline-in-progress bar fill. Muted blue. |
| `--accent-degraded` | `#f59e0b` | Feed degraded dot. Same as attention amber — degraded feed warrants same urgency. |
| `--source-ais` | `#0891b2` | AIS source badge background. Muted cyan. |
| `--source-adsb` | `#6366f1` | ADS-B source badge background. Muted indigo. |
| `--feed-healthy` | `#22c55e` | Domain Pulse dot: healthy. |
| `--feed-degraded` | `#f59e0b` | Domain Pulse dot: degraded (>30s, <TTL). |
| `--feed-stale` | `#ef4444` | Domain Pulse dot: stale (TTL expired). |

**Color usage rules:**
- Accent colors are applied sparingly. Only functional use — never decorative.
- Red (`--accent-critical`) is reserved for feed stale and the highest-confidence compound signals (>90). Do not use red for normal escalated signals.
- Do not use color to distinguish information that must also be distinguishable in greyscale. All status distinctions have both color and symbol/label.

### 3.2 Typography

**UI font:** `Inter` (primary). Fallback: system-ui, sans-serif.

**Monospace font:** `IBM Plex Mono`. Used for: confidence scores, UTC timestamps, coordinate values, SHA-256 hashes, provenance data, numeric badge values.

**Type scale:**

| Use | Size | Weight | Font |
|---|---|---|---|
| Panel section header | 11px | 600 | Inter, uppercase, tracked 0.08em |
| Signal card title | 14px | 500 | Inter |
| Signal card metadata | 12px | 400 | Inter |
| NOMINAL row label | 12px | 400 | Inter |
| Confidence score | 14px | 600 | IBM Plex Mono |
| Timestamp | 11px | 400 | IBM Plex Mono |
| Evidence layer key | 12px | 400 | Inter |
| Evidence layer value | 12px | 400 | IBM Plex Mono |
| Provenance hash | 11px | 400 | IBM Plex Mono, `--text-tertiary` |
| Button | 13px | 500 | Inter |
| Form label | 12px | 500 | Inter |
| Chrome / nav label | 12px | 500 | Inter |
| UTC clock | 12px | 400 | IBM Plex Mono |

**Line height:** 1.5 for body text. 1.2 for compact rows and metadata.

### 3.3 Density & Spacing

Panoptes uses a compact density preset. Spacing is systematic, not generous.

| Component | Vertical padding | Horizontal padding |
|---|---|---|
| Signal card (Attention Required) | 10px top + 10px bottom | 16px |
| Signal card (Monitoring) | 8px top + 8px bottom | 16px |
| NOMINAL row | 6px top + 6px bottom | 16px |
| Baseline row | 6px top + 6px bottom | 16px |
| Zone header bar | 8px top + 8px bottom | 16px |
| Evidence layer row (collapsed) | 8px top + 8px bottom | 16px |
| Evidence layer row (expanded) | 8px top + 12px bottom | 16px |
| Watch list row | 10px top + 10px bottom | 16px |
| Form field group | 16px top gap between groups | — |
| Chrome bar | 0 (flex center) height: 36px | 20px |
| Nav rail item | 10px top + 10px bottom | 14px |

**Gap between signal cards in the same zone:** 6px.
**Gap between zones:** 16px (zone header + 8px margin above it).

### 3.4 Panel System

All panels follow a consistent structural pattern:

- **Background:** `--bg-surface`
- **Border:** `1px solid --border-subtle`
- **Border radius:** `6px`
- **Panel header:** `11px uppercase tracked` label + optional count badge + optional icon button. Full-width, bottom border `--border-subtle`.

Elevation is communicated by background color step, not drop shadows. `--bg-base` → `--bg-surface` → `--bg-raised`. No box-shadows in the UI.

**Card types:**

| Type | Border style | Left accent |
|---|---|---|
| Attention Required card | `border: 1px solid --border-subtle; border-left: 3px solid --accent-attention` | 3px amber |
| Monitoring card | `border: 1px solid --border-subtle; border-left: 3px solid --accent-monitoring` | 3px steel |
| NOMINAL row | No card frame. Single-line row with `--bg-surface` background. | None |
| Baseline row | No card frame. Single-line row with `--bg-surface` background. | None |

### 3.5 Motion

Panoptes uses motion only for functional communication — never for decoration.

| Event | Animation |
|---|---|
| New signal arrives (WebSocket) | Card expands into position from zero height, 350ms ease-out. |
| `● LIVE` indicator (WebSocket event) | Single 800ms opacity pulse (1.0 → 0.4 → 1.0), then settles. Does not repeat until next event. |
| Evidence panel opens | Slides in from right, 250ms ease-out. Signal Desk dims simultaneously. |
| Evidence panel closes | Slides out right, 200ms ease-in. Signal Desk un-dims. |
| Signal dismissed | Card collapses to zero height, 300ms ease-in. Fade out simultaneously. |
| New signal incoming pulse on `[N↑]` badge | Badge increments; brief background flash (200ms). |
| No other animations. | Hover states: immediate (0ms). Transitions on background/border: 100ms max. |

---

## 4. Navigation Model

### 4.1 Chrome Bar

Full-width, 36px height, fixed to viewport top. `--bg-surface` background with `1px solid --border-subtle` bottom border.

**Left:** `◈ PANOPTES` — platform mark. Small-caps, 13px, `--text-secondary`. Not a link.

**Center:** UTC clock — `HH:MM:SS UTC`, IBM Plex Mono, 12px. Live, updating every second.

**Right (left to right):**
Domain Pulse strip — one dot per active provider. Inline, ambient.
Format: `[● OpenSky] [● ADS-B Exch] [● NOAA AIS]`
Dot color uses `--feed-healthy` / `--feed-degraded` / `--feed-stale`.
Degraded or stale state changes the dot and adds provider label in `--accent-degraded` or `--accent-critical` color.
Clicking any provider dot has no action — read-only status display.

Session badge — `[N↑]` when N > 0 (amber background), `[0]` when zero (monochrome). See §4.3 for full badge behavior. Clicking scrolls the Signal Desk to the first unviewed signal.

### 4.2 Left Navigation Rail

160px wide, full viewport height, fixed. `--bg-surface` background with `1px solid --border-subtle` right border.

**Structure:**
```
◈ icon    (platform mark, top, not interactive)
─────────
⊞  Desk         [N]    ← badge only when N > 0
⊞  Evidence
⊞  Spatial
─────────
⊞  Watches
```

**Active surface:** Indicated by a `3px solid --accent-attention` left edge bar on the active nav item row. Background of active row: `--bg-raised`.

**Desk badge `[N]`:** Same count as the chrome session badge. Amber background when N > 0, not rendered when 0.

**Evidence nav item:** Always accessible. When clicked without an active signal context, loads the last-viewed signal's Evidence Surface. If no prior signal exists (fresh session), navigates to Signal Desk instead.

**Spatial nav item:** Always accessible. When clicked without an active signal context, loads the last-viewed signal's Spatial View. If that signal had no spatial context (or no prior signal), navigates to Signal Desk.

**Surface switching:** No transition animation between surfaces. Immediate swap.

### 4.3 Session Badge — Behavior Specification

The `[N↑]` badge tracks signals generated since the operator's last session that have not yet been viewed.

**Increment:** Badge increments when a new signal is pushed via WebSocket while the session is active, if that signal has not yet been seen in this session.

**What counts as "viewed":**
- The operator clicked the signal card (opening Evidence for it).
- The signal card was scrolled into the viewport of the Signal Desk AND the operator's scroll position paused at or past that card for ≥ 500ms.
- The signal was dismissed (dismiss action counts as definitively seen).

**What does NOT clear the badge:**
- Navigating to the Signal Desk surface.
- The Signal Desk being the active surface while signals are above the fold but the operator has not scrolled to them.
- Opening any other surface.

**Decrement:** Badge decrements by one each time a signal is viewed per the criteria above. When the count reaches zero, the badge changes to monochrome `[0]` style (no amber, no `↑`). At zero it is still rendered — disappearing entirely would create a confusing gap.

**Click action:** Clicking the badge scrolls the Signal Desk to the first signal that has not yet been viewed (the oldest unviewed signal in the Attention Required zone, or Monitoring zone if no unviewed AR signals remain).

**Session boundary:** On fresh session start (after JWT refresh or re-login), the badge count is loaded from the API: signals generated since the last session's `ended_at` timestamp, with status `active`.

---

## 5. Signal Desk — Horizon View

### 5.1 Layout

**Viewport:** Full height minus chrome (864px usable).

```
[Chrome — fixed 36px]
[Nav rail — 160px fixed] | [Signal list — 896px scrollable] | [Sidebar — 320px fixed]
```

**Nav rail:** Fixed. Always visible.
**Signal list:** Vertically scrollable. All zones stack in a single scroll container.
**Right sidebar:** Fixed. Contains Session Summary and Domain Pulse only. No additional content ever appended. No widgets.

### 5.2 Signal List — Zone Structure

Zones render in this fixed order from top to bottom:

1. **Attention Required zone** — always rendered, even when empty.
2. **Monitoring zone** — always rendered, always labeled.
3. **NOMINAL rows** — rendered inline within Monitoring zone, below any active monitoring cards.
4. **Baseline rows** — rendered inline within Monitoring zone, below NOMINAL rows.

Scroll loads at the top of Attention Required. The operator always sees the highest-priority content first without any scroll action.

#### Attention Required Zone Header

```
─── ATTENTION REQUIRED  [N] ────────────────────────── ● LIVE
```

- `ATTENTION REQUIRED` — 11px uppercase tracked, `--text-secondary`
- `[N]` — count badge, amber background when N > 0
- `● LIVE` — right-aligned, 8px dot + `LIVE` label, `--feed-healthy` color. Pulses once per WebSocket event. Read-only.
- When zone is empty: header still renders. Below the header, a single subdued line reads: `No signals above escalation threshold. All monitored regions within normal parameters.` Color: `--text-tertiary`. This is explicit confirmation, not an absent state.

#### Monitoring Zone Header

```
─── MONITORING  [N signals · M watches] ────────────────────
```

- Count shows active monitoring signals (N) and total active watches (M).
- When no monitoring signals exist and all watches are NOMINAL: header still renders.

### 5.3 Signal Card — Attention Required

```
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃  Signal Family Name                       84 [███████ ] ┃
┃  Geography Entity · [AIS] [ADS-B] · ×2 · 4h ago         ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
```

- **Border:** `1px solid --border-subtle`, `border-left: 3px solid --accent-attention`
- **Background:** `--bg-surface`
- **Line 1:** Signal family name (`--text-primary`, 14px medium) + confidence score + fill bar (right-aligned)
- **Line 2:** Geography entity name · source badges · compound indicator (if compound) · signal age
- **Hover:** Background transitions to `--bg-raised` (100ms). Cursor: pointer.
- **Click:** Opens Evidence Surface panel from right.

**Confidence fill bar:** 8-character representation, proportional to score. `84` → `[███████ ]`. Fill color: `--accent-attention` for Attention Required zone. Bar is read-only.

**Compound indicator:** `×N` where N is the constituent count. Rendered in `--text-secondary`. Not a badge — inline text only. Same card shape; no dramatically different treatment.

**Signal age:** Relative time. `4h ago`, `2d ago`, `31m ago`. Renders in `--text-tertiary`.

**Source badges:** `[AIS]` — `--source-ais` background (muted cyan). `[ADS-B]` — `--source-adsb` background (muted indigo). `12px`, `--text-primary`. Read-only.

### 5.4 Signal Card — Monitoring Zone

Same structure as Attention Required card with these differences:
- **Border left:** `3px solid --accent-monitoring` (steel)
- **Confidence fill bar:** Fill color: `--accent-monitoring`
- **Border style:** Thin (`┌─┐` style, not thick `┏━┓`)

### 5.5 NOMINAL Row

No card frame. Single-line, compact row.

```
  ✓  Suez Canal Watch          NOMINAL   ·  checked 2m ago
```

- `✓` — `--accent-nominal` (muted green), 13px
- Watch name — `--text-secondary`, 12px
- `NOMINAL` badge — `--accent-nominal` background, 10px uppercase, horizontal padding 6px
- `checked Xm ago` — `--text-tertiary`, 11px IBM Plex Mono

**Interaction:** Read-only in v1. No click action. No drill-down (there is no signal to investigate). The NOMINAL state communicates "nothing to investigate here."

### 5.6 Baseline-in-Progress Row

No card frame. Single-line, compact row.

```
  ⧖  Strait of Malacca Watch     BASELINE  ·  ████████░░░  18/30d
```

- `⧖` — `--accent-baseline` (muted blue), 13px
- Watch name — `--text-secondary`, 12px
- `BASELINE` label — `--accent-baseline`, 11px uppercase
- Progress bar — inline, 80px wide, proportionally filled. Fill: `--accent-baseline`. Background: `--border-subtle`.
- `18/30d` — `--text-tertiary`, 11px IBM Plex Mono

**Interaction:** Clickable. Click navigates to Watch Configuration, pre-selecting this watch. Secondary action — not primary tap target but responsive.

### 5.7 Right Sidebar

**Fixed, 320px, full content height.** No scroll in the sidebar itself.

Contains exactly two sections, in this order:

#### Session Summary

```
SESSION
─────────────────
Mar 07  22:41
2 signals · 1 escalated
4 watches active
100% coverage
```

- Section header: 11px uppercase, `--text-secondary`
- Last session timestamp: IBM Plex Mono, 11px
- Counts: 12px, `--text-primary`
- Read-only. No interactive elements.

#### Domain Pulse

```
DOMAIN PULSE
─────────────────
OpenSky Network    [●]  OK · 8s
ADS-B Exchange     [●]  OK · 12s
NOAA AIS           [●]  OK · 5s
```

- Section header: 11px uppercase, `--text-secondary`
- Per provider row: provider name (12px `--text-secondary`) + status dot + status label + last heartbeat age (IBM Plex Mono 11px `--text-tertiary`)
- Dot states: `--feed-healthy` (green), `--feed-degraded` (amber), `--feed-stale` (red)
- Degraded state row: provider name and dot color change; adds label `Degraded` in amber; heartbeat age highlighted
- Stale state row: all text in `--accent-critical`, adds label `Stale — no signal`
- Read-only. No click action on any row.

**Degraded feed contextual note** (appears below provider row, only when degraded or stale):
```
  ADS-B signals may carry reduced confidence
  while this feed recovers.
```
`--text-tertiary`, 11px. Disappears when feed recovers.

**Sidebar constraint:** No additional content is ever added below Domain Pulse in v1. No watch summaries, no quick action panels, no charts. The sidebar is the ambient health context. It must never become a dashboard.

---

## 6. Evidence Surface

### 6.1 Trigger and Layout

**Trigger:** Operator clicks any signal card on Signal Desk.

**Layout:** Right-side panel occupying **65% of viewport width** (≈936px at 1440px). Signal Desk compresses to remaining 35% (≈504px), dims with `rgba(0,0,0,0.45)` overlay, and becomes non-interactive.

The Evidence panel is a layout slot, not a modal. It slides in from the right (250ms ease-out). The Signal Desk remains rendered behind it — the operator retains list context.

**Closing:** `← Signal Desk` breadcrumb link in panel header, OR `[✕]` button, OR `ESC` key. Panel slides out right (200ms). Signal Desk returns to full width and becomes interactive.

### 6.2 Panel Structure

The Evidence panel has three fixed regions:

```
┌───────────────────────────────────────────────────────┐
│  PANEL HEADER                              ▲ STICKY   │
│  ← Signal Desk  |  Signal Name      [✕]              │
│  Geography · Date · UTC Time · Age                    │
│  Watch: Watch Name                                    │
├───────────────────────────────────────────────────────┤
│                                                       │
│  EVIDENCE BODY                             ↕ SCROLL   │
│  (all content below header, above dismiss)            │
│                                                       │
│  ...confidence, constituents, evidence layers...      │
│                                                       │
├───────────────────────────────────────────────────────┤
│  DISMISS SECTION                           ▼ STICKY   │
│  [ reason for dismissal — required         ]          │
│                              [ Dismiss Signal ]       │
└───────────────────────────────────────────────────────┘
```

**Header:** Fixed at top. Does not scroll. Contains breadcrumb, signal name, close button, and signal identity line (geography, timestamp, age, watch name).

**Evidence body:** Independently scrollable. Fills available height between header and dismiss section.

**Dismiss section:** Fixed (sticky) at bottom. Always visible regardless of scroll depth in the body. Operator can dismiss without scrolling to the end of evidence.

### 6.3 Confidence Breakdown

Rendered as a structured inline tree. No charts or bar visualizations.

**Simple signal:**
```
CONFIDENCE   84
├─ Deviation from baseline (z-score normalized)   0.91
├─ Source completeness                            1.00
├─ Baseline maturity                              0.95
└─ Recency factor                                 1.00
```

**Compound signal:**
```
CONFIDENCE   84
├─ Harmonic mean of 2 constituents
├─ Convergence factor               × 0.95
└─ Source independence (AIS+ADS-B)  × 1.00

  (68 ⊕ 73) × 0.95 × 1.00 = 84
```

- Section header: 11px uppercase, `--text-secondary`
- Tree rows: 12px, `--text-secondary` label + IBM Plex Mono `--text-primary` value (right-aligned or inline)
- Formula line for compound: 13px IBM Plex Mono, `--text-primary`, centered or left-aligned
- All read-only.

**Degraded-feed confidence note** (shown when feed was degraded at detection time):
```
⚠  Source completeness reduced: ADS-B was degraded at detection.
   Confidence reflects partial feed availability.
```
Amber `⚠`, `--text-secondary` 12px. Read-only.

### 6.4 Constituent Signals (Compound Only)

Rendered as an expandable section:

```
CONSTITUENT SIGNALS
────────────────────────────────────────────────────
▶  Chokepoint Throughput Change    68   [AIS]
▶  Cargo Route Deviation           73   [ADS-B]
```

- Each row: `▶` chevron + signal family name + confidence score + source badge
- **Click (DRILL):** The Evidence body re-renders showing that constituent's evidence. Breadcrumb in the header extends: `← Signal Desk  /  Compound  /  Chokepoint Throughput Change`
- Navigating breadcrumb crumbs goes back up the chain without closing the panel.
- The constituent's own confidence breakdown and evidence layers are shown in the same Evidence body structure.

### 6.5 Contributing Evidence Layers

```
CONTRIBUTING EVIDENCE
────────────────────────────────────────────────────
▶  Feed State Snapshot    opensky      03:10 UTC
▶  Entity Observation     ais_noaa     03:08 UTC
▶  Baseline Snapshot      ais_noaa     window: 30d
▶  Constituent Signal Ref  ×2           —
```

- All collapsed by default. Collapsed row shows: type tag + source feed + timestamp (or descriptor).
- **Click `▶` (EXPAND):** Row expands inline. Payload rendered as key-value list. No navigation.

**Expanded evidence layer:**
```
▼  Entity Observation     ais_noaa     03:08 UTC
   ─────────────────────────────────────────────
   MMSI                   477123456
   Vessel class           Container
   Position               26.48°N  56.31°E
   Heading                278°
   Speed                  2.1 kts
   Dwell delta            +314%  (baseline: 18h · observed: 74h)
   ─────────────────────────────────────────────
   sha256: a3b4c5d6e7f8...              [READ]
```

- Key-value rows: label in `--text-secondary` 12px, value in IBM Plex Mono `--text-primary` 12px
- Provenance hash: `sha256: [first 12 chars]...` — IBM Plex Mono 11px `--text-tertiary`, right-aligned. Read-only. Full hash copyable on click (copy-to-clipboard, no navigation).
- Evidence type `feed_state_snapshot`: shows feed provider, status at detection time, heartbeat age at detection.
- Evidence type `baseline_snapshot`: shows mean, stddev, sample count, window used.

### 6.6 View Spatially Button

```
[ View Spatially → ]
```

- Rendered in the evidence body, below evidence layers.
- 13px, secondary button style (outline, not filled).
- **Click:** Navigates to Spatial View surface, passing full signal context. Evidence Surface state is preserved in browser history — back navigation returns to same Evidence state.
- Only shown if the signal's geography entity has a defined geometry (all seeded entities will; future extensibility noted).

### 6.7 Dismiss Workflow

The Dismiss section is sticky at the bottom of the Evidence panel.

```
DISMISS
──────────────────────────────────────────────────────
[ reason for dismissal — required                    ]
                                  [ Dismiss Signal ]
```

- Text input: required. Placeholder: `reason for dismissal — required`.
- `[ Dismiss Signal ]` button: disabled (greyed, not clickable) until input field is non-empty. No minimum character count, but the field must have text.
- On click: signal status updated to `dismissed` on server. Signal card in Signal Desk collapses (300ms fade + height collapse). Evidence panel closes automatically. Signal Desk returns to full width.
- Dismissed signals are removed from Signal Desk zones. They are accessible in evidence history (v2 feature) but not in v1 Signal Desk view.
- Dismissal reason + timestamp stored server-side. Contributes to per-watch calibration metrics.

---

## 7. Spatial View

### 7.1 Trigger and Layout

**Trigger:** Click `[ View Spatially → ]` from Evidence Surface body.

**Layout:**
```
[Chrome — fixed 36px]
[Nav rail — 160px fixed] | [Left dossier — 320px fixed] | [Map area — 960px]
```

The Spatial View is a full surface replace — Evidence Surface is no longer the active surface; Spatial View takes its place. Browser history entry is added so Back works.

### 7.2 Left Dossier Panel

320px wide, fixed. `--bg-surface` background, `1px solid --border-subtle` right border.

**Dossier header (sticky at top):**
```
← Evidence
─────────────────────────────
COMPOUND DISRUPTION PATTERN
Strait of Hormuz
Conf: 84  ·  03:14 UTC
```

- `← Evidence` — text link, `--text-secondary` 12px. Click returns to Evidence Surface with full state preserved.
- Signal name: 14px medium, `--text-primary`
- Entity + confidence + timestamp: 12px, `--text-secondary`

**Dossier body (scrollable below header if content overflows):**

```
Data as-of:
03:10 UTC  (4h ago)
⚠  Historical · not live
─────────────────────────
LAYERS
[✓] Strait Boundary     [TOGGLE]
[✓] AIS Vessels         [TOGGLE]
[✓] ADS-B Tracks        [TOGGLE]
[ ] Baseline Corridor   [TOGGLE]
─────────────────────────
SCOPE
AIS:    47 vessels
ADS-B:  12 tracks
─────────────────────────
ENTITY
Strait of Hormuz
Type: Chokepoint
26.5°N  56.2°E
```

**"Historical · not live" treatment:** Shown as a subdued `⚠ Historical · not live` line in `--text-tertiary`. The map is not desaturated or visually degraded — the data is accurate and credible. The indicator is informational, not a warning about data quality.

**Layer toggles:** Checkboxes. Each layer is signal-family-specific — only the layers relevant to the current signal type are listed. For a compound signal: all constituent signal types' layers are available. Layer list is not extensible by the operator; it is determined by the signal family.

**Scope counts:** Read-only display showing how many entities (vessels, tracks) are rendered for this signal context.

**Entity block:** Named entity metadata. Read-only.

### 7.3 Map Area

960px wide, fills remaining viewport.

**Base tiles:** Dark OSM tiles. Muted landmass (`#1a2030`-equivalent), minimal coastline detail, subdued labels. The map is an instrument backdrop, not a reference map. Signal overlays are the information.

**Signal-scoped overlays (rendered on map load for the active signal):**

| Layer | Visual | Color token |
|---|---|---|
| Entity boundary | Polygon or line outline, 2px | `--source-ais` (teal) for maritime; `--source-adsb` (indigo) for aviation. Filled with 10% opacity fill. |
| AIS vessel positions | Dots, 6px radius | `--source-ais` |
| ADS-B flight tracks | Dashed lines, 1.5px | `--source-adsb` |
| Baseline corridor | Dashed line, 1px | `--text-tertiary` (muted, background reference) |
| Dwell zones (for Port Dwell signals) | Polygon fill, 20% opacity | `--accent-attention` |

**Map controls (minimal):**

```
[+]  [-]  [⊞ fit to entity]
```

- `[+]` / `[-]`: zoom in / out. Bottom-right corner of map area.
- `[⊞ fit to entity]`: resets viewport to show the full entity boundary with comfortable padding. The map loads to this view on entry.
- No draw tools. No layer panel. No search. No basemap switcher. No scale bar (not required for v1).

**Entity interaction:**
- Click a vessel dot: tooltip overlay showing MMSI, vessel class, dwell deviation. Read-only. No action buttons.
- Click a flight track: tooltip overlay showing ICAO24, route deviation delta. Read-only.
- Tooltips dismiss on click-away.

**Map boundary rule:** The Spatial View is not a free-browse map. The map viewport is initialized to `[⊞ fit to entity]` on load. The operator can pan and zoom freely within the session, but there is no "explore nearby areas" mode, no global extent, no address search. It is a signal-context viewer, not a GIS workspace.

---

## 8. Watch Configuration

### 8.1 Layout

```
[Chrome — fixed 36px]
[Nav rail — 160px fixed] | [Watch list — 320px] | [Watch form — 960px]
```

**Watch list:** Fixed header (Domain Pulse strip) + scrollable watch list below.
**Watch form:** Fixed form header (watch name + action buttons) + scrollable form body.

### 8.2 Watch List Column

**Header (fixed at top of column):**
```
WATCHES                    [+ New]
─────────────────────────────────
Domain Pulse
[●] OpenSky  [●] ADS-B  [●] AIS
─────────────────────────────────
```

- Domain Pulse shown at the top of Watch Configuration so the operator knows which feeds are healthy before creating or modifying watches.
- `[+ New]` button: creates blank form in right column.

**Watch list rows (scrollable):**
```
▐ Strait of Hormuz
  ACTIVE  ·  2 signals
──────────────────────────────────
  Suez Canal
  NOMINAL  ·  0 signals
──────────────────────────────────
  Gulf-Europe Corridor
  NOMINAL  ·  0 signals
──────────────────────────────────
  Strait of Malacca
  BASELINE
  ████████░░░░  18/30d
```

- Active row (selected watch): `▐` left accent bar, `--bg-raised` background.
- Watch status badges:

| Status | Left stripe | Badge style |
|---|---|---|
| `ACTIVE` | `--accent-monitoring` | Steel label |
| `NOMINAL` | `--accent-nominal` | Muted green label |
| `BASELINE` | `--accent-baseline` | Blue-grey label + inline progress bar |
| `DEGRADED` | `--accent-degraded` | Amber label |
| `INACTIVE` | None | Row dimmed, `--text-tertiary` |

### 8.3 Watch Form

**Form header (fixed at top, sticky):**
```
STRAIT OF HORMUZ WATCH                     [Disable]
─────────────────────────────────────────────────────
```

- Watch name: 14px medium, `--text-primary`
- `[Disable]`: secondary button style (outline). Immediately below watch name, right-aligned. Visible at all times when a watch is loaded.
- **Delete is NOT in the form header.** See §8.7.

**Form body (scrollable):**

#### Geography Entity

```
Geography Entity
[▾ Strait of Hormuz                              ]
┌─────────────────────────────────────────────────┐
│  [entity boundary thumbnail — teal on dark bg]  │
└─────────────────────────────────────────────────┘
```

- Searchable dropdown: operator types entity name; list filters. All named entities from the seeded `geography_entities` table.
- Thumbnail: static map rendering of entity boundary (GeoJSON rendered to a small static image or Canvas element). Updates when selection changes. Read-only visual confirmation.
- No interactive map in Watch Configuration.

#### Signal Families

```
Signal Families
[✓]  Port Dwell-Time Anomaly
[✓]  Chokepoint Throughput Change
[✓]  Vessel Traffic Density Anomaly
[✓]  AIS Vessel Dark Event
[✓]  Cargo Route Deviation
[✓]  GPS Jamming Zone Formation
[✓]  Commercial Route Mass Deviation
[✓]  Compound Disruption Pattern
```

- All enabled by default for new watches.
- Each is a checkbox. Unchecking a family is an incompatible change (see §8.5).

#### Escalation Threshold

```
Escalation Threshold
[65]  ──────────────────────────●──── 100
Signals at or above 65 → Attention Required zone
```

- Range slider: 1–100. Default: 65.
- Numeric input adjacent to slider; both are editable and stay in sync.
- Helper text below reads back the operator's setting in plain language.
- Changing this value is a **compatible change** — does not reset baseline.

#### Baseline Window

```
Baseline Window
[30]  days
Minimum data collection before NOMINAL state is possible
```

- Numeric input. Default: 30. Min: 7. Suggested max: 90.
- Changing this value is an **incompatible change** if a baseline is established.

#### Save / Cancel

```
                         [Save Changes]     [Cancel]
```

- `[Save Changes]`: primary button style. On success: brief inline `✓ Saved` text replaces button for 1.5s, then reverts to `[Save Changes]`.
- `[Cancel]`: secondary button. Reverts all unsaved changes to last saved state.

### 8.4 Compatible vs. Incompatible Change Handling

**Compatible changes** (do not affect baseline): escalation threshold adjustment.
- No warning. Save silently. Watch continues.

**Incompatible changes** (would require baseline reset): geography entity, baseline window, signal family add/remove.
- Warning banner appears **immediately** on field change, before Save is attempted.
- The watch's Save button changes label to `Save & Reset Baseline`.

**Incompatible change warning:**
```
┌────────────────────────────────────────────────────────────────┐
│  ⚠  Changing this setting will reset the baseline for this     │
│     watch. 18 days of collected data will be discarded.        │
└────────────────────────────────────────────────────────────────┘
```

Amber `⚠`, 12px, `--bg-raised` background, amber left border. Appears inline directly below the changed field.

On `[Save & Reset Baseline]` click: baseline transitions to `in_progress`, old baseline marked `superseded`, event logged. Watch continues collecting.

On `[Cancel]`: warning disappears, field reverts, no change saved.

### 8.5 Disable Action

`[Disable]` is in the form header (always visible when a watch is loaded).

Click triggers inline confirmation that replaces the button in place:

```
Disable this watch? Signals will stop generating.
[Cancel]    [Confirm Disable]
```

No modal. No navigation. Inline in the form header area. Confirmation persists until the operator acts. On `[Confirm Disable]`: watch status → `inactive`. Watch list row dims. Form header updates to show `[Enable]` in place of `[Disable]`.

### 8.6 Disable vs. Inactive vs. Delete

- **Disable:** Watch stops generating signals. Configuration and baseline are preserved. Can be re-enabled instantly.
- **Re-enable:** Click `[Enable]` (same position as `[Disable]`). Watch resumes. If baseline was established before disable, it is still valid; watch transitions back to `active`.
- **Delete:** Permanent. Baseline is discarded. Configuration is removed. Cannot be undone.

### 8.7 Delete — Placement and Hierarchy

Delete is **not in the form header**. It is at the bottom of the form body, below the Save/Cancel buttons, separated by a dashed divider. It is visually recessed: text-link style, not a button, low contrast.

```
[Save Changes]     [Cancel]

─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─

Delete watch
Permanently removes this watch and all baseline data.
This action cannot be undone.                              delete watch
```

`delete watch` text-link: `--text-tertiary`, 12px, underline on hover. On click, expands an inline confirmation block:

```
┌──────────────────────────────────────────────────────────────┐
│  Delete "Strait of Hormuz Watch"?                            │
│  This will discard 18 days of baseline data.                 │
│                                                              │
│                        [Cancel]    [Confirm Delete]          │
└──────────────────────────────────────────────────────────────┘
```

- `[Confirm Delete]`: destructive button style (red border, red text). One click confirms. No second confirmation.
- `[Cancel]`: collapses confirmation block. No action taken.
- On confirmed delete: watch removed from list, form area shows "Select or create a watch."

**The Disable → Delete hierarchy:** Operator must scroll to the bottom of the form, past Save/Cancel, past the dashed divider, to reach Delete. This is intentional friction. Delete requires deliberate action, not accidental adjacency.

---

## 9. System States

### 9.1 NOMINAL State

**Definition:** A watch is NOMINAL when:
1. Watch status = `active`
2. No signals with status = `active` exist for this watch
3. Baseline status = `established`

**Visual treatment in Signal Desk:**
Single-line row in the Monitoring zone. `✓` badge (muted green), watch name, `NOMINAL` badge, `checked Xm ago` timestamp.

**The NOMINAL state must feel alive, not empty:**
- The `● LIVE` indicator in the Signal Desk header pulses on each WebSocket cycle — even when all watches are NOMINAL. This confirms the system is running.
- The `checked Xm ago` timestamp is the most recent time the processor evaluated this watch against current data and confirmed no anomalies. It must always be recent (< 5 minutes under normal operation) to be credible.
- If a NOMINAL watch's last-checked timestamp exceeds 10 minutes, the row should flag this as potentially stale: `⚠ checked 14m ago` in amber. This indicates a processor delay, not a NOMINAL confirmation.

**What NOMINAL communicates to the operator:** "I am actively monitoring this geography. I collected data, compared it against the established baseline, and found nothing that meets the signal criteria. You do not need to check the raw feed."

### 9.2 Baseline-in-Progress State

**Definition:** A watch has transitioned past `CREATED` but has not yet collected enough data to establish a baseline. Default window: 30 days.

**Visual treatment:**
- In Signal Desk: `⧖` row with inline progress bar showing `18/30d`.
- In Watch Configuration: same treatment in watch list.
- Signal Desk empty state for the watch: no signal cards will fire for this watch until baseline is established.

**First-run communication:** When a fresh watch is created, it appears immediately in the Signal Desk Monitoring zone as a baseline row. The operator understands: the system is collecting, not broken.

**Baseline maturity scaling:** Confidence scores for signals from watches with `established` but young baselines (established at minimum, not at 2×minimum) carry a lower baseline maturity factor in the confidence formula. This is surfaced in the Evidence confidence breakdown, not in the Signal Desk directly.

### 9.3 Feed Degraded / Stale State

Two feed health states exist below healthy:

**Degraded:** Heartbeat is arriving but inconsistently. Last heartbeat age > 30s but the Valkey key has not yet expired.
- Domain Pulse dot: `--feed-degraded` (amber)
- Chrome strip: `[▲ ADS-B Exch]` amber
- Sidebar contextual note: "Signals detected with ADS-B data may carry reduced confidence while this feed recovers."

**Stale:** Valkey TTL has expired. No heartbeat received within the TTL window (30s default).
- Domain Pulse dot: `--feed-stale` (red)
- Chrome strip: `[✕ ADS-B Exch]` red
- Sidebar note: "ADS-B feed is stale. Signals requiring ADS-B data will not fire."

**Confidence impact:** Confidence scores computed when a feed was degraded or stale at detection time carry a `source_completeness` factor below 1.0. This is surfaced in Evidence confidence breakdown and noted with `⚠` on the confidence line.

### 9.4 First-Run / Empty State

**Trigger:** No watches are configured.

**Signal Desk layout:** Signal list area is replaced by a single centered guidance panel. No zones rendered.

**Panel content:**
```
Panoptes is ready.

No watches configured. Data collection begins the moment a
watch is created. Feeds are healthy and ingesting.

Baselines require approximately 30 days before signals are
generated. Configure your watches and plan accordingly.

─────────────────────────────────────────────────

✓  Step 1  Verify feed health
           All three feeds healthy.

→  Step 2  Configure your first watch

           [ → Go to Watch Configuration ]
```

**Tone:** Direct, serious, procedural. No illustration, no marketing language, no exclamation marks. Two-step structure with Step 1 auto-evaluated from Domain Pulse state.

**Domain Pulse degraded at first run:** Step 1 renders `⚠` instead of `✓`, with text: "One or more feeds are degraded. Signals from those domains will not fire until they recover." The CTA still appears — operator can configure watches; the system will use whatever feeds are available.

### 9.5 Alert-Heavy State

No structural changes to the Signal Desk — it handles many signals gracefully because the zone structure is a list.

**Key behaviors at high signal volume:**
- Scroll position on load: top of Attention Required. Always.
- Compound signal rule: a compound signal and its constituent signals are NOT both listed in the Signal Desk. Only the compound is shown. Constituents are visible inside Evidence only. This prevents double-counting and list inflation.
- If more than 8 signals exist in Attention Required: a subtle scroll affordance indicator appears at the bottom of the visible zone: `↓ N more signals below`.
- Domain Pulse degraded state in the alert-heavy scenario: the sidebar degraded note is especially important. Operator must be able to correlate "lots of signals firing" with "feed health is fine" or "feed was degraded and now I need to re-evaluate confidence."

### 9.6 Compound Signal State

A compound signal is a cross-domain convergence finding. It represents the system's highest-confidence output.

**Signal Desk treatment:** Same card shape as simple signals. Compound indicator: `×N` inline in the metadata line. No separate visual framing. Confidence bar may be higher than constituent signals.

**Evidence Surface treatment:** Compound view shows confidence breakdown with harmonic mean formula. Constituent signals listed as drill-down rows. Each constituent can be drilled into independently — Evidence body re-renders for that constituent, with breadcrumb navigation.

**Spatial View treatment:** For compound signals, all constituent source layers are available for toggle. If ADS-B and AIS are both constituents, both vessel and flight track layers are available simultaneously on the same map. The map shows cross-domain correlation in space.

---

## 10. Component Reference

### 10.1 Confidence Bar Component

An 8-character proportional fill indicator. Used inline in signal cards.

```
84 [███████ ]    (84% fill, 7 of 8 chars filled)
71 [██████  ]    (71% fill, ~6 chars)
52 [████    ]    (52% fill, ~4 chars)
```

**Implementation:** A flex container: score (IBM Plex Mono 14px 600) + bar element (8 character-widths of fixed width; inner fill proportional to score). Fill color determined by zone context.

Read-only. No tooltip. The full confidence breakdown is in Evidence.

### 10.2 Source Badge Component

Small inline pill badge identifying the data source domain.

| Badge | Background | Label |
|---|---|---|
| `[AIS]` | `--source-ais` | `AIS`, 11px uppercase, `--text-primary` |
| `[ADS-B]` | `--source-adsb` | `ADS-B`, 11px uppercase, `--text-primary` |

Padding: 3px vertical, 6px horizontal. Border-radius: 3px. Rendered inline in signal card metadata line. Read-only.

### 10.3 Domain Pulse Dot Component

An 8px circle with fill color from `--feed-healthy`, `--feed-degraded`, or `--feed-stale`.

Used in:
- Chrome bar (ambient strip, small format, 6px)
- Signal Desk right sidebar Domain Pulse section (8px)
- Watch Configuration column header (6px)

No animation on the dot itself. State changes are immediate color updates.

### 10.4 Count Badge Component

A small pill containing a numeric count.

| Context | Color | Style |
|---|---|---|
| Session badge, N > 0 | `--accent-attention` background | `[N↑]` with `↑` glyph |
| Session badge, N = 0 | `--border-subtle` background, `--text-tertiary` | `[0]`, no `↑` |
| Zone header count | `--bg-raised` background | `[N]` |
| Nav Desk badge, N > 0 | `--accent-attention` background | `[N]` |

### 10.5 Inline Warning Banner Component

Used for incompatible change warnings in Watch Configuration.

```
┌──────────────────────────────────────────────────────────────┐
│  ⚠  Warning message text.                                    │
│     Optional second line.                                     │
└──────────────────────────────────────────────────────────────┘
```

- Background: `--bg-raised`
- Left border: 3px `--accent-degraded` (amber)
- `⚠` glyph: `--accent-degraded`
- Text: `--text-secondary` 12px
- Border: `1px solid --border-subtle`
- Border-radius: 4px
- Appears and disappears without animation (immediate).

### 10.6 Sticky Footer Pattern (Evidence Dismiss)

The dismiss section uses CSS `position: sticky; bottom: 0` within the Evidence panel's scroll container. The Evidence body uses `overflow-y: auto`. The panel's outer container is a flex column with the body taking `flex: 1` and the sticky footer anchored below.

This ensures the dismiss section is always visible at the bottom of the panel, regardless of evidence body scroll position, without requiring JavaScript scroll listeners.

---

## 11. Global Interaction Rules

These rules apply across all surfaces.

### 11.1 Navigation

| Rule | Detail |
|---|---|
| Nav rail is always visible and always interactive | On all surfaces, at all times. |
| Top chrome is always fixed | UTC clock, Domain Pulse strip, session badge never scroll away. |
| Evidence panel is a layout slot, not a modal | Signal Desk remains rendered behind it. Never a full viewport block. |
| Spatial View requires signal context | No free-browse map mode. Nav to Spatial without context loads last-viewed signal or redirects to Signal Desk. |
| Evidence breadcrumb navigation is not destructive | Navigating breadcrumb items does not close the Evidence panel; it re-renders the body in place. |
| Browser Back works as expected | `[ View Spatially → ]` pushes a history entry. Back returns to Evidence with full state. |

### 11.2 Signal Desk

| Rule | Detail |
|---|---|
| Attention Required zone always renders | Even when empty — with explicit empty text, not a blank area. |
| Monitoring zone always renders | With header and count, even if all watches are NOMINAL. |
| Compound constituents not listed separately | Only the compound card appears in Signal Desk. Constituents are Evidence-only. |
| New signal arrival animation | Card expands from zero height, 350ms. No glow, no sound, no modal interrupt. |
| Scroll position on load | Top of page. Operator always sees Attention Required first. |
| `[N↑]` badge clear rule | Clears only when unseen signals have been scrolled into viewport (≥500ms pause) or clicked. Opening Signal Desk alone does not clear the badge. |

### 11.3 Evidence Surface

| Rule | Detail |
|---|---|
| Dismiss requires reason text | Button is disabled until input is non-empty. |
| Dismiss footer is always visible | Sticky at bottom of panel. Never hidden by body scroll. |
| Dismiss is irreversible in-session | Dismissed signal leaves Signal Desk immediately. History view is a v2 feature. |
| Constituent drill is in-panel | Never opens a second panel. Re-renders Evidence body in place with breadcrumb. |
| Evidence layers collapsed by default | Collapsed state shows type + source + timestamp. Expanded on click. |
| Provenance hash is copy-on-click | Click copies to clipboard. No navigation, no modal. |

### 11.4 Spatial View

| Rule | Detail |
|---|---|
| Map loads to entity bounds | `[⊞ fit to entity]` behavior on every entry. |
| No layer browser | Layer list is determined by signal family. Operator cannot add arbitrary layers. |
| Data is historical | Spatial View shows data as-of detection time. Labeled clearly. Map is not desaturated. |
| Map interactions are local | Pan and zoom do not trigger data fetches or navigation. |
| Tooltip on entity click | Read-only. MMSI/ICAO24 + key deviation value. Dismiss on click-away. |

### 11.5 Watch Configuration

| Rule | Detail |
|---|---|
| Incompatible change warning appears on field change | Not on save. Operator sees consequence at point of decision. |
| Compatible changes save silently | Threshold changes do not trigger any warning. |
| Delete requires deliberate scroll + click + confirm | Placed below Save/Cancel, below a dashed divider, as a text-link. Not a button. |
| Disable is reversible, Delete is not | Both have inline confirmations. Only Delete has destructive styling on confirm button. |
| Domain Pulse visible in Watch Config header | Operator knows feed health before creating or modifying watches. |

### 11.6 Performance Expectations

| Expectation | Target |
|---|---|
| Signal Desk initial load (signals + watches) | < 1.5s to interactive |
| Evidence panel open (slide-in) | < 300ms total (animation 250ms + data fetch) |
| Spatial View map load (tiles + overlays) | < 2s to rendered entity boundary |
| WebSocket reconnect (if dropped) | Automatic, transparent. Signal Desk shows brief `● Reconnecting` state on `● LIVE` indicator. |
| Watch form save | Optimistic update. UI updates immediately; reverts on server error with inline error message. |

---

*End of UX Design Specification — Panoptes v1*
