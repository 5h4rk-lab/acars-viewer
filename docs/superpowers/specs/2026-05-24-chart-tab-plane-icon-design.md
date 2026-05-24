# Design: Chart Tab + Rotating Plane Icon

**Date:** 2026-05-24  
**Project:** ACARS Viewer (`index.html` — single-file vanilla JS + Leaflet)

---

## Overview

Two independent visual enhancements to the ACARS flight tracker:

1. **Rotating airplane icon** — replaces the static circle at the last known position on the map with a top-down airplane silhouette that rotates to match the aircraft's heading.
2. **Chart tab** — adds a 5th "Chart" tab to the right detail panel showing altitude, speed, and fuel over time using the Canvas 2D API.

---

## Feature 1: Rotating Airplane Icon

### What changes

The last position marker (currently an orange `circleMarker` at `i === f.pos.length - 1`) is replaced with a Leaflet `divIcon` containing an SVG airplane silhouette.

### Icon design

Top-down airplane shape: fuselage (vertical rect, rounded), swept wings (path), tail fins (path). Matches the style chosen during brainstorming (Option C).

### Rotation

CSS `transform: rotate(${hdg}deg)` applied to the SVG wrapper div. `hdg` comes from `p.hdg` already parsed in `ingest()`. North = 0°, East = 90°, etc. — matches Leaflet's coordinate system.

### Color

Uses `altColor(p.alt)` — same function already applied to path segments — so the icon color is consistent with the altitude-colored trail.

### Fallback

If `p.hdg` is null (not reported in that message label), render the existing orange `circleMarker` unchanged. No silent heading assumed.

### Scope

Only the **last** position gets the plane icon. All intermediate dots remain `circleMarker`. The "Show All" map mode (`renderAllMap`) uses the same pattern — last position of each flight gets its plane icon.

---

## Feature 2: Chart Tab

### Tab structure

Add a 5th tab `<div class="dtab" data-tab="chart">Chart</div>` and corresponding `<div class="tab-pane" id="tab-chart">` to the existing detail panel. Tab click behavior handled by the existing `querySelectorAll('.dtab')` listener — no change needed there.

### Rendering

`renderDetail(key)` gains a **Chart tab** section that:

1. Creates (or reuses) a `<canvas id="chart-canvas">` inside `#tab-chart`
2. Calls `drawChart(f)` which renders to that canvas

### Chart layout

Single canvas. Three data series drawn on top of each other:

| Series | Color | Style | Y-axis |
|--------|-------|-------|--------|
| Altitude | `#7b68ee` | Filled area under line | Left — FL units (alt / 100) |
| Speed (TAS) | `#00d4ff` | Dashed line | Right — knots |
| Fuel (FOB) | `#ffcc00` | Dashed line | Right — lbs (shared axis with speed, independent scale) |

### Axes

- **X-axis:** time, mapped from first to last position fix timestamp. Labels: HH:MM at start, midpoint, end.
- **Left Y-axis:** FL0 to FL(maxAlt/100), with 4 grid lines. Labels: `FL360` style.
- **Right Y-axis:** 0 to max(speed, fuel/100) with shared scale per series. Labels only shown if data present.
- **Grid:** horizontal lines at 25%, 50%, 75% of canvas height, color `#1a2332`.

### Legend

Strip at top of canvas: colored line swatches + labels. Only shows legend entries for series that have at least one data point.

### Empty state

If `f.pos.length === 0`: render `<div class="empty">NO POSITION DATA</div>` instead of canvas.

If a series has no data (e.g., no speed reported): that line is silently omitted.

### Canvas sizing

Canvas width = container width (`#tab-chart` clientWidth). Height = 220px fixed. Re-renders on tab click when a flight is selected.

---

## Implementation notes

- No external libraries. Canvas 2D API only.
- All changes inside `index.html`. No new files.
- `renderDetail(key)` is the entry point — both features triggered from `select(key)`.
- `renderAllMap()` needs updating for the plane icon at last position per flight.
- Timestamp parsing: existing `m.ts` format is `YYYY/MM/DD HH:MM:SS.mmm` — parse with `new Date(ts.replace(/\//g,'-'))` for X-axis mapping.

---

## Out of scope

- Animation / playback
- Export
- Live mode
- Any other features discussed but not selected
