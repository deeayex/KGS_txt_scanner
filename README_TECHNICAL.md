# Lease Production Tools — Technical README

A single-file, client-side web app for filtering and mapping Kansas Geological
Survey (KGS) oil-lease production exports. No build step, no backend, no framework.

---

## 1. Overview

`lease_scanner.html` is one self-contained HTML file (markup + CSS + vanilla JS in
an IIFE). The user loads a KGS CSV/TXT export locally; all parsing, filtering,
sorting, and mapping happen in the browser. Nothing is uploaded. The only network
dependencies are CDN libraries/fonts and the map tile server.

There is also a companion CLI, `kgs_downloader.py` (stdlib-only Python 3), that
downloads and unzips the official KGS archive files that this app consumes.

### Input data shape
21 columns, comma-delimited, quoted fields. The app keys on these:
- `MONTH-YEAR` — format `M-YYYY` (1- or 2-digit month, e.g. `3-2025`, `11-2025`).
- `PRODUCTION` — numeric, barrels for that lease-month. (Note: `PRODUCT` is the
  product *type* and is `"O"` for every row — it is **not** a quantity.)
- `LEASE_KID` — stable lease identifier (used for dedup).
- `OPERATOR`, `COUNTY` — filter fields.
- `LATITUDE`, `LONGITUDE` — used for map placement.
- `URL` — link to the KGS lease page.

Reference file: ~176 MB, ~653k rows, ~20,212 leases, Jan 2020–present.

---

## 2. Dependencies (all via CDN)

| Library | Version | Source |
|---|---|---|
| PapaParse | 5.4.1 | cdnjs |
| Leaflet | 1.9.4 | cdnjs |
| Leaflet.markercluster | 1.5.3 | cdnjs |
| Google Fonts | — | Oswald, IBM Plex Sans, IBM Plex Mono |
| Map tiles | — | OpenStreetMap (`tile.openstreetmap.org`) |

If deploying offline/air-gapped, these must be vendored locally and the tile layer
pointed at an internal tile server (see §7).

---

## 3. Data flow

```
load file ──► indexFile()  [PASS 1: one streaming pass]
                 ├─ Set: distinct months
                 ├─ Set: counties, operators        (autocomplete option lists)
                 ├─ Map: bestLease  (LEASE_KID → most-recent row, by monthKey)
                 └─ row count
              complete():
                 ├─ fill month dropdowns (range + exact)
                 ├─ build operatorList / countyList (sorted)
                 ├─ build leaseSites[]  (per-lease most recent w/ valid coords)
                 ├─ build leaseUnmapped[] (per-lease most recent w/o coords)
                 └─ build awSites[]  (American Warrior, active last 12 mo, w/ coords)

press Scan ──► runScan()   [PASS 2: re-stream whole file]
                 └─ filter each row: minProd, operator-contains, county-contains,
                    then date (exact == targetKey | range from..to)
                 complete(): renderResults(matches, info)
                                ├─ table (sortable, capped)
                                ├─ recap line
                                └─ computeScanSites(matches) → scanSites/scanUnmapped
```

### Why two passes / re-streaming
The reference file is ~176 MB. To keep memory bounded, the app does **not** retain
all 653k parsed rows. `indexFile` keeps only aggregates plus one row per lease
(`bestLease`, ~20k). Each scan re-streams the file from the retained `File` object
with `Papa.parse(file, {worker:true, step:...})`, accumulating only matches. This
trades a few seconds of re-parse per scan for a low, stable memory footprint.

---

## 4. Key state and constants (top of the IIFE)

- `COLS` — all 21 columns in canonical order (used for CSV export).
- `HIDDEN` — Set of 8 columns suppressed from the on-screen table
  (`API_NUMBER, FIELD, PRODUCING_ZONE, TWN_DIR, RANGE_DIR, LATITUDE, LONGITUDE, PRODUCT`).
- `DISPLAY_COLS = COLS \ HIDDEN` — on-screen table columns.
- `POPUP_COLS = DISPLAY_COLS \ {URL}` — map popup fields (URL rendered as a button).
- `NUMERIC` — Set of columns that sort numerically.
- `DISPLAY_CAP = 1000` — max table rows rendered (CSV export is uncapped).
- Filtering/map state: `lastMatches`, `lastInfo`, `sortCol/sortDir`, `leaseSites`,
  `leaseUnmapped`, `scanSites`, `scanUnmapped`, `awSites`, `map`, `clusterLayer`,
  `awLayer`, `mapMode` (`'all'|'scan'`), `lastRenderedMode`, `operatorList`,
  `countyList`.

### Date handling
- `monthKey("M-YYYY") → YYYY*100+MM` — sortable, used for equality/range compares.
- `monthAbs`/`mAbs("M-YYYY") → YYYY*12+(MM-1)` — linear month index, used for the
  AW "past 12 months" window (`cutoff = newestAbs - 11`).
- `daysInMonth("M-YYYY")` — `new Date(y, m, 0).getDate()`, leap-year correct; used for
  prod/day in popups.

### Coordinate validity
A point is plotted only if lat/lng parse and fall in `lat∈(30,42)`, `lng∈(-104,-94)`
(Kansas bounding box, with margin). Failing rows go to the `*Unmapped` lists.

---

## 5. Scanner internals

- **Filters** (applied per row in `runScan` step): production `>= minProd`;
  `OPERATOR.toLowerCase().includes(opQuery)`; `COUNTY.toLowerCase().includes(countyQuery)`;
  date is exact (`k === targetKey`) or range (`fromKey <= k <= toKey`, auto-swapped if
  reversed).
- **Sorting** (`sortBy`): sorts the full `lastMatches` array (not just visible rows),
  so the displayed top-N reflects global order. First click on a `NUMERIC` column is
  descending, text is ascending; re-click flips. `localeCompare(...,{numeric:true})`
  for text. `renderTable` re-renders the capped slice and the header arrows.
- **CSV export**: `Papa.unparse` over `COLS` (full record), reflects the current sort
  order; filename encodes the active filter (`lease_matches_<tag>.csv`).
- **Autocomplete** (`attachAutocomplete(inputId, listId, getOptions)`): substring
  filter over the option list, capped at 60 shown, keyboard nav (↑/↓/Enter/Esc),
  `mousedown` selection (fires before input blur). Operator/county are free-text, so
  partial entries still filter at scan time even without an explicit pick.

---

## 6. Map internals

- `ensureMap()` — initializes the Leaflet map once (Kansas view), adds the OSM tile
  layer, the production `clusterLayer` (markercluster), the legend control, and the
  AW layer. Idempotent.
- `renderMarkers()` — clears and repopulates `clusterLayer` from `activeSites()`
  (`leaseSites` in `all` mode, `scanSites` in `scan` mode). Each point is an
  `L.circleMarker` styled by `prodStyle()`. Updates count/note/toggle, fits bounds,
  calls `invalidateSize()` (needed because the tab is `display:none` until shown).
  Guarded by `lastRenderedMode` to avoid redundant re-renders on tab switches.
- `prodStyle(r)` — bucketed radius+fill by `PRODUCTION` (5 bands: <10 / <100 / <500 /
  <2000 / ≥2000). Bands and colors are mirrored in the legend control.
- `popupHtml(r)` — `POPUP_COLS` as a key/value grid; inserts a computed `PROD/DAY`
  row immediately after `PRODUCTION`; appends the KGS link button. Bound lazily
  (`bindPopup(() => popupHtml(s.r))`).
- **Scan↔map link**: `renderResults` calls `computeScanSites(matches)` (dedup matched
  rows by `LEASE_KID`, keep most-recent matched row w/ valid coords), enables the
  *Scan results* toggle, sets `mapMode='scan'`, and refreshes if the map is visible.
- **American Warrior layer** (`buildAwLayer`): its own `markerClusterGroup` with a
  custom `iconCreateFunction` (blue themed count bubbles) and `L.divIcon` "A" leaf
  markers (`zIndexOffset:1000`). It is **independent** of `mapMode` and the scan
  filter — built from `awSites` (operator contains `"american warrior"`, active in the
  last 12 months) and toggled by the `#awToggle` checkbox. Persists across All/Scan
  switches and re-renders.

---

## 7. Common customizations

| Want to… | Change |
|---|---|
| Show more/fewer table rows | `DISPLAY_CAP` |
| Show/hide table columns | `HIDDEN` set |
| Change which columns sort as numbers | `NUMERIC` set |
| Re-bucket marker size/color | `prodStyle()` thresholds **and** the legend `rows` array in `ensureMap` |
| Change the AW "recent" window | `awCutoff = newestAbs - 11` (months back) |
| Change AW operator match | the `"american warrior"` substring test |
| Count AW "past year" from today instead of newest data month | replace `mAbs(newestFirst[0])` with `mAbs` of the current date |
| Different basemap / offline tiles | the `L.tileLayer(...)` URL + attribution |
| Loosen/tighten plotted-coord bounds | the `lat>30 && lat<42 && lng>-104 && lng<-94` checks (3 places: leaseSites, computeScanSites, awSites) |

---

## 8. Constraints & gotchas

- **No browser storage** (localStorage/sessionStorage) is used or supported in this
  context; all state is in-memory for the session.
- **Map tiles require internet**; the rest works offline. A blank/colored map with
  visible markers means tiles are blocked (firewall) — verify `tile.openstreetmap.org`
  is reachable.
- **Tab sizing**: Leaflet must `invalidateSize()` after the map tab becomes visible,
  since it initializes/refreshes while `display:none` would yield a 0×0 viewport.
- **circleMarker + markercluster** is used deliberately (vector markers, `preferCanvas:true`)
  to render ~20k points without bitmap-icon overhead.
- **PRODUCT vs PRODUCTION**: never filter quantities on `PRODUCT`.
- Coordinate filter silently drops out-of-range rows into the unmapped note; widen the
  bounds if you load non-Kansas data.

---

## 9. Companion: `kgs_downloader.py`

Interactive stdlib-only CLI. Presents the KGS archive catalog (oil/gas × decade +
2020-present + all-leases listing), downloads the selected `.zip`(s) from
`https://www.kgs.ku.edu/PRS/Ora_Archive/`, and extracts the `.txt`. Confirms before
downloading pre-1987 (1980–1989) archives, which carry an IHS Energy redistribution
restriction. Run: `python kgs_downloader.py`. KGS refreshes monthly from the Kansas
Dept. of Revenue, so a monthly run + reload keeps the app current.
