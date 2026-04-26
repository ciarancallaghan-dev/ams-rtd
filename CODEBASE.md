# AMS-RTD Train Times — Codebase Documentation

> **IMPORTANT: This file must be updated every time a change is made to the codebase.**
> It is the single source of truth for how the app works, why decisions were made, and what has changed.
> When working with Claude Code, paste this file into the conversation at the start of each session to restore full context.

---

## Project Overview

A Progressive Web App (PWA) that shows real-time train journey options between Amsterdam Centraal and Rotterdam Centraal, installable on iPhone via "Add to Home Screen". Built because the NS app assumes a 4-minute Schiphol transfer time; this app lets the user set their own minimum (default: 2 minutes).

**Live URL:** https://ciarancallaghan-dev.github.io/ams-rtd/
**GitHub repo:** https://github.com/ciarancallaghan-dev/ams-rtd

---

## Files

| File | Purpose |
|------|---------|
| `index.html` | Entire app — all HTML, CSS, and JavaScript inline (single file) |
| `manifest.json` | PWA manifest: name, icons, theme colour (#00587a teal) |
| `sw.js` | Service worker — caches index.html and manifest.json for offline/install support |
| `make_icons.py` | One-time script (no PIL needed) that generated icon-192.png and icon-512.png — can be deleted |
| `icon-192.png` | PWA home screen icon (192×192) |
| `icon-512.png` | PWA home screen icon (512×512) |
| `CODEBASE.md` | This file |

---

## Data Source

**rijdendetreinen.nl AJAX API** — no API key required, publicly accessible.

| Endpoint | Purpose |
|----------|---------|
| `GET /en/ajax/departures?station=CODE&offset=N` | Live departures from a station. `offset` shifts the window forward by N minutes (works for departures). Returns ~2 hours of data. |
| `GET /en/ajax/arrivals?station=CODE` | Live arrivals at a station. Returns ~2 hours from current server time. **Offset parameter does NOT shift the arrivals window** — always returns from "now". |

### Station codes
| Code | Station |
|------|---------|
| `ASD` | Amsterdam Centraal |
| `ASDZ` | Amsterdam Zuid |
| `SHL` | Schiphol Airport |
| `RTD` | Rotterdam Centraal |

### Key API fields (departures)
- `serviceNumber` — train service number (string, e.g. `"1840"`)
- `destination` — final destination of the train (e.g. `"Breda"`)
- `via` — comma-separated intermediate stops (e.g. `"Schiphol Airport, Rotterdam C."`)
- `transportType` — e.g. `"Intercity direct"`, `"Intercity"`, `"Sprinter"`
- `platform` — departure platform
- `departureTime` — ISO timestamp of scheduled departure
- `delaySeconds` — current delay in seconds
- `realDepartureTime` — actual departure time (scheduled + delay)

### Key API fields (arrivals)
- `serviceNumber` — same number as in departures for the same physical train
- `arrivalTime` — ISO timestamp of scheduled arrival
- `delaySeconds` / `delay` — delay in seconds
- `platform` — arrival platform

### Important API behaviour
- **Arrivals window is always ~2 hours from now**, regardless of offset. This means:
  - IC Direct (43 min journey): arrivals available for trains departing ASD up to ~77 min ahead
  - IC (65+ min journey): up to ~55 min ahead
  - Sprinter (75+ min): up to ~44 min ahead
- Trains outside that window are simply absent from the arrivals map. When this happens, the journey is **not shown** (no fallback estimates).
- The app auto-refreshes every 60 seconds in live mode, so trains enter the window naturally.

### CORS handling
Direct fetch is attempted first. If it fails (CORS), the request is retried via `https://corsproxy.io/?url=ENCODED_URL`.

---

## Journey Types

### AMS → RTD

**1. Direct**
- Filter: `asd.filter(d => goesVia(d, 'rotterdam'))`
- Uses `goesVia` (not `goesTo`) because many trains (e.g. IC Direct to Breda, IC to Roosendaal) have Rotterdam as a via stop, not the final destination. `goesVia` checks both `destination` and `via` fields.
- Arrival at RTD: looked up exactly by `serviceNumber` in `arrRTD`. If not found (outside 2h window), journey is skipped.

**2. Via Schiphol** (user changes trains at Schiphol)
- First leg filter: `asd.filter(d => goesVia(d, 'schiphol') && !goesVia(d, 'rotterdam'))` — trains that stop at Schiphol but do NOT go via Rotterdam (those are already direct)
- Second leg filter: `shl.filter(d => goesVia(d, 'rotterdam'))` — trains from Schiphol toward Rotterdam
- SHL arrival: looked up exactly in `arrSHL`. If not found, journey is skipped.
- RTD arrival: looked up exactly in `arrRTD` for the second leg. If not found, journey is skipped.
- Transfer time: `diffMin(shlArr, dep2)` — shown with colour coding (green/amber/red vs user's minimum)

**3. Via Amsterdam Zuid + Metro**
- Second leg (train from ASDZ to RTD) filter: `asdz.filter(d => goesVia(d, 'rotterdam'))`
- RTD arrival: looked up exactly in `arrRTD`. If not found, journey is skipped.
- ASD departure time: estimated backwards from ASDZ train departure minus metro time (GVB timetable)
- Metro timing: **timetable-based estimate** (see Metro section below) — clearly labelled "Timetable estimate" in the UI

### RTD → AMS

**1. Direct**
- Filter: `rtd.filter(d => goesTo(d, 'amsterdam centraal'))` — IC/ICD trains to Amsterdam Centraal
- Arrival at ASD: looked up exactly in `arrASD`. If not found, journey is skipped.

**2. Via Schiphol**
- First leg filter: `rtd.filter(d => goesVia(d, 'schiphol') && !goesTo(d, 'amsterdam centraal'))`
- Second leg filter: `shl.filter(d => goesVia(d, 'amsterdam'))` — trains from Schiphol toward Amsterdam
- SHL and ASD arrivals: both exact from API. If either not found, journey is skipped.

**3. Via Amsterdam Zuid + Metro**
- First leg (train RTD → ASDZ) filter: `rtd.filter(d => goesVia(d, 'amsterdam zuid') || goesTo(d, 'amsterdam zuid'))`
- ASDZ arrival: looked up exactly in `arrASDZ`. If not found, journey is skipped.
- ASD arrival: ASDZ arrival + metro time (GVB timetable estimate) — shown in grey, labelled "Timetable estimate"

---

## Metro Line 52 (Amsterdam Centraal ↔ Amsterdam Zuid)

Metro timing cannot be exact — there is no free, unauthenticated GVB real-time API. Instead, timing is calculated from the **published GVB timetable frequencies**:

| Period | Frequency |
|--------|-----------|
| Weekday peak (06:30–09:00, 16:00–18:30) | Every 5 min |
| Weekday off-peak | Every 7.5 min |
| Weekend | Every 10 min |

Journey time: 14 minutes. Average wait: half the frequency. Total = `round(freq/2 + 14)` minutes.

Metro times are shown with a grey "Timetable estimate" label and the [Metro 52] badge to distinguish them from live train data.

---

## No Fallback Policy

**There are no estimated or fallback train arrival times.** If a train's arrival cannot be confirmed from the rijdendetreinen arrivals API (i.e. the service number is not in the arrivals map), the journey is not shown. This guarantees all displayed train times are exact from the live data.

The only exception is metro timing (see above), which is accepted as timetable-based by the user.

---

## Key Functions (JavaScript)

### `goesTo(dep, word)`
Returns true if `dep.destination` contains `word` (case-insensitive).

### `goesVia(dep, word)`
Returns true if `dep.via` OR `dep.destination` contains `word` (case-insensitive). Used instead of `goesTo` for routes where Rotterdam/Amsterdam can appear as an intermediate stop (not final destination).

### `exactArrival(serviceNumber, arrMap)`
Looks up a service number in an arrivals Map. Returns `{ arr: Date, pf: platform }` if found, or `{ arr: null, pf: null }` if not. No fallback — callers must check for null and skip the journey if missing.

### `actualDepart(dep)`
Returns scheduled departure + delay as a Date object.

### `actualArrive(arr)`
Returns scheduled arrival + delay as a Date object.

### `getMetroTime(refDate)`
Returns `{ total, freq }` for metro line 52 based on GVB timetable for the given time/day.

### `fetchDeps(station, offset)`
Fetches departures from rijdendetreinen, with CORS proxy fallback.

### `fetchArrivals(station, offset)`
Fetches arrivals from rijdendetreinen and returns a `Map<serviceNumber, arrivalObject>`.

### `calcAMStoRTD(deps, arrRTD, arrSHL, refTime)`
Calculates all AMS→RTD journey options. `deps = { asd, asdz, shl, rtd }`.

### `calcRTDtoAMS(deps, arrASD, arrSHL, arrASDZ, refTime)`
Calculates all RTD→AMS journey options.

### `dedup(journeys)`
Removes duplicate journeys (same type + departure time) and sorts by departure time.

### `refresh()`
Main data fetch and render loop. Fetches 8 API endpoints in parallel (ASD, ASDZ, SHL, RTD departures + RTD, ASD, SHL, ASDZ arrivals). Auto-repeats every 60 seconds in live mode.

---

## Settings

| Setting | Default | Stored in |
|---------|---------|-----------|
| Schiphol transfer minimum | 2 min | `localStorage('shl')` |
| Departure time | Now (live) | Date/time picker (not persisted) |

Maximum offset supported: 240 minutes (~4 hours). Beyond that, a message directs the user to NS.nl.

---

## PWA / Installation

- `manifest.json` enables "Add to Home Screen" on iPhone (Safari)
- `sw.js` service worker caches `index.html` and `manifest.json`
- Cache is versioned (`'ams-rtd-vN'`) — **bump the version in `sw.js` every time `index.html` is updated**, otherwise installed PWAs will keep serving stale cached code
- Current cache version: **v3**
- Apple touch icon is generated dynamically at runtime via Canvas API (train emoji on teal background)

---

## Changelog

### Initial build
- Created PWA with manifest, service worker, and icons
- Data source: rijdendetreinen.nl (no API key required)
- Three journey types: Direct, Via Schiphol, Via Amsterdam Zuid (metro)
- Schiphol transfer minimum configurable (default 2 min)
- Direction toggle (AMS→RTD / RTD→AMS)
- Date/time picker for future departures (up to ~4 hours ahead)
- Metro timing from GVB timetable frequency
- CORS proxy fallback (corsproxy.io)

### Arrival time accuracy
- Added `fetchArrivals()` for RTD, ASD, SHL, ASDZ to get exact arrival times and platforms
- Matched departures to arrivals by `serviceNumber`
- Captured arrival platform (`pf`) from arrivals API for all legs including intermediate stops (Schiphol, Amsterdam Zuid)

### Fixed missing IC Direct trains
- Root cause: IC Direct trains (e.g. service 1842 to Breda) have `destination = "Breda"` not Rotterdam, so `goesTo(d, 'rotterdam')` missed them entirely
- Fix: changed direct route filter to `goesVia(d, 'rotterdam')` which checks both `via` and `destination` fields
- Same fix applied to ASDZ departure filter and Via Schiphol exclusion filter (`!goesVia` instead of `!goesTo`)
- Also fixed `shlToRtd` filter to use `goesVia` to include IC Direct trains from Schiphol

### Removed all fallback estimates
- Removed all hardcoded travel time constants (T object)
- `exactArrival()` now returns `null` when service number not in arrivals map
- Any journey where arrival is null is skipped entirely — never shown
- "Estimated arrival" / "Scheduled" labels removed; all shown times are exact
- Via Amsterdam Zuid temporarily removed (metro timing is inherently approximate)

### Restored Via Amsterdam Zuid with timetable metro
- User confirmed timetable-based metro timing is acceptable
- Re-added Via Amsterdam Zuid for both directions
- Train parts (ASDZ departure/arrival) require exact API data — if not available, journey skipped
- Metro portion labelled "Timetable estimate" in grey to distinguish from live data
- Re-added ASDZ departures and arrivals fetches (back to 8 parallel API calls)

### Via Amsterdam Zuid: show Amsterdam Zuid as explicit waypoint
- Restructured from 2-leg to 3-leg layout matching Via Schiphol style
- AMS→RTD: metro leg (ASD→ASDZ) → transfer indicator → train leg (ASDZ→RTD)
- RTD→AMS: train leg (RTD→ASDZ) → transfer indicator → metro leg (ASDZ→ASD)
- Transfer indicator shows "Change here" (AMS→RTD, 0 min by timetable calc) or "~N min wait" (RTD→AMS, = freq/2)
- Metro leg renders both departure and arrival stops with "Timetable estimate" label
- `renderDots` rewritten to a cleaner model: xfer leg emits mid-dot + dashed-line; each non-xfer leg emits its journey line (dashed if metro, solid if train); first leg emits start dot; last leg appends end dot

### Service worker cache bumps
- v1 → v2: after IC Direct fix and platform improvements
- v2 → v3: after metro route restoration
- Always bump when deploying index.html changes

---

## Known Limitations

1. **Metro timing** — GVB line 52 timing is from the published timetable, not real-time tracking. No free unauthenticated GVB real-time API exists.
2. **Arrivals window** — The rijdendetreinen arrivals API only covers ~2 hours from now. Trains arriving further out are simply not shown until they enter the window (auto-refresh brings them in).
3. **Offset for arrivals** — The `offset` parameter on the arrivals endpoint does not shift the window. Only the departures endpoint respects offset. This means future departure times (date picker) may show fewer journeys.
4. **No trip detail endpoint** — `/en/ajax/trip/{id}/{date}` returns 404. There is no way to get a full stop-by-stop schedule for an individual train from this API.
