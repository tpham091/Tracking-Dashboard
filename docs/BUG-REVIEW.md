# NFI Tracking — bug review punch list

Reviewed against `index.html` as uploaded (3,147 lines). Ranked by how likely each item is to
put visibly wrong data in front of a dispatcher. Line numbers refer to the uploaded file.

Findings marked **[verified]** were reproduced by running the actual functions from the file
against synthetic TMW/C3-shaped input; the rest are read from the code path.

---

## P0 — the NLT chain produces confidently wrong output

### 1. The NLT alert compares the ETA to the *next* stop against the NLT of the stop the driver is *at*
`getNltMiss` (2635) → `findRouteStopForLocation` (2615) picks the first route stop whose city
matches the driver's last reported location. But the ETA it compares against was produced by
`findNextEtaStop` (2330), which deliberately *skips past* the same-city stop to the one after it.
So the comparison is `ETA(stop N+1)` vs `NLT(stop N)` — a different store, a different deadline.

Three separate consequences:
- Wrong deadline used → both false "NLT Miss" alerts and missed real ones.
- `findRouteStopForLocation` does not filter `completed` stops, so a route with two stores in the
  same city (routine in Houston/San Antonio) matches the already-delivered one.
- If the driver reports a highway position instead of a city with a stop — and the Check Call tab's
  own placeholder is `I-35 near New Braunfels` — no stop matches, `getNltMiss` returns `null`, and
  NLT checking is silently switched off for that shipment. Nothing on screen says so.

### 2. An arrival after midnight defeats the NLT comparison entirely **[verified]**
`isNltAtRisk` (2625) takes `arrivalMin` as minutes-past-midnight with the date discarded, while
`projectRemainingSchedule` (2372-2390) legitimately runs its clock across midnight.

    isNltAtRisk('2300', 30)    // arrival 00:30, NLT 23:00  →  false   (should be true)
    isNltAtRisk('2300', 1410)  // arrival 23:30, NLT 23:00  →  true

The rollover guard at 2628 only corrects the *NLT* side (early-morning NLT + afternoon ETA), never
the arrival side. Overnight 227 Super Regional runs are precisely the loads carrying 2300/2400
NLTs, so this hits the highest-stakes lane and always fails in the "we're fine" direction.

### 3. Warehouse # extraction misses or grabs the wrong number
`extractWarehouseNumber` (2562) pulls the first bare 3-digit token out of `commodityShipper`:
- `SA - Frozen` — a value in the app's own commodity datalist (line 294) — has no 3-digit token, so
  `warehouse` is `''` and **every** NLT lookup for that shipment returns nothing, silently.
- `0227` or `227A` fail the `\b(\d{3})\b` boundary the same way.
- Any commodity string where another 3-digit number comes first yields a wrong warehouse, which
  then indexes a *real but wrong* deadline out of `NLT_LOOKUP[corp]` and displays it with full
  confidence.

This is a free-text field (`d-commodity`, 285) with a datalist but no validation, and it is also
overwritten by the `Warehouse Code` column on every outbound import.

### 4. A missing NLT pairing is indistinguishable from "no NLT applies"
At 2393, 2503 and 2645 the pattern is `corp && warehouse && NLT_LOOKUP[corp] ? NLT_LOOKUP[corp][warehouse] : ''`.
A corp that exists with no row for that warehouse yields `undefined` → `''` → the NLT cell renders
`—` and no alert fires — same output as a stop that genuinely has no deadline.

Measured against the shipped tables: `NLT_LOOKUP` has 393 corps across 17 warehouses, corps carry
between 5 and 12 warehouse entries each, 144 NLT corps have no `STORE_LOOKUP` row, and stores
`81`, `817`, `818` have no NLT row at all. Missed pairings are not a rare edge here.

### 5. The projected schedule silently re-bases onto "now" from an old location
`projectRemainingSchedule` starts its clock at `new Date()` (2372) but takes the location from the
last check call, whenever that was. `refreshProjectedSchedule` (2439) re-runs it on every checkbox
click, every carrier import and every close-out — hours later, with the same stale location. Every
projected arrival slides forward by the staleness, and the downstream NLT-risk flags slide with it.

Meanwhile `updates[].eta` is written once (2427-2429) and never recomputed, so the ETA column and
the "Projected remaining stops" table show different numbers for the same shipment, and
`computeNltAlerts` reads the frozen one while `computeNltRiskAlerts` reads the recomputed one.

### 6. Stale locations still drive NLT alerts
The Tracking Log blanks the ETA and demands a fresh check call after `LAST_LOCATION_STALE_MIN`
(2046, 2816) — but `computeNltAlerts`/`computeNltRiskAlerts` (2660, 2672) apply no such cutoff. The
NLT panel keeps making definite claims about a truck nobody has heard from in six hours, while the
row two inches below it says "Needs Update".

---

## P1 — imports produce wrong or missing data

### 7. Delimiter sniffing looks at one line only **[verified]**
`parseDelimitedRows` (885) picks tab-vs-comma from the first non-empty line. A tab-separated export
with a plain title line above the header is parsed as CSV — every row becomes a single cell and the
whole import yields nothing. The mirror case (a CSV whose header contains a stray tab) mis-parses
in the other direction.

### 8. The header row is only recognized on row 0 **[verified]**
`mapRows` (921) tests `rows[0]` and nothing else. Given a preamble line that happens to contain a
tab, it falls back to `DEFAULT_INDEX_MAP` positional mapping *and* treats the real header row as
data — producing a live shipment record whose CS# is literally `Shipment ID`:

    { commodityShipper: 'Warehouse Code', csNumber: 'Shipment ID', storeNumber: 'Store #', ... }

Data rows survive only as long as C3's column order still matches the hard-coded positions
(1324). Reorder a column in the export and every field shifts silently.

### 9. Header-name variants silently drop columns **[verified]**
`HEADER_MAP` (1312) has `'store #'` and `'trailer #'` but not `'store#'`/`'trailer#'`. With a header
written without the space:

    { commodityShipper: '227', csNumber: 'CS1', scheduleRelease: '08/08/2026 09:30 AM' }
    // storeNumber and trailerNumber gone → Stops blank, Consignee blank, Trailer# blank

Because two other headers still matched, `headerHits >= 2` passes and no warning is shown.
`normalizeHeader` (914) collapses runs of whitespace but doesn't handle underscores, non-breaking
spaces, or trailing `:`/`*`. Same gap in `HEADER_MAP_CARRIER_IMPORT`, `HEADER_MAP_CARRIER_STOPS`
and `HEADER_MAP_EMPTY`.

### 10. The block-bleed fix only holds for `CS`-prefixed refs **[verified]**
The guard at 1618 is `/^CS\d+$/i.test(probeCs)`. With a numeric `Ref #` and one header for two
back-to-back blocks, the original bug returns in full:

    [{ csNumber: '8842137', truckNumber: '101', driverName: 'JOHN DOE',
       routeStops: [ {companyName:'HEB CORP# 0006', ...},
                     {companyName:'Enroute', city:'T2', earliestDate:'202', ...},  // block 2's main row
                     {companyName:'HEB CORP# 0014', ...} ] }]                      // block 2's stop

The second shipment disappears entirely and its stops attach to the first CS#. Any ref format other
than `CS12345` — `CS-12345`, a bare number, a `REF` prefix — triggers this.

### 11. Stop-block detection needs 4 recognized headers
`stopHits >= 4` (1600). A stop header exporting only Company/City/Arrival/Departure without
Earliest/Arrived/Completed never registers, and every stop row falls through to main-row parsing.
Once `stopMap` *is* set, any row with a value in any of the five stop columns becomes a stop —
including footer/total rows.

### 12. Carrier re-import wipes manual stop progress
`importCarrierRows` (1825) does `existing.routeStops = r.routeStops` wholesale. Every Arrived and
Completed checkbox and every hand-typed arrival/departure entered since the last import is
discarded, and `refreshProjectedSchedule` then rebuilds the projection from the blank route.

### 13. A blank cell can never clear a field
The `Object.keys(patch).forEach(k => { if(patch[k] !== '') ... })` idiom (1857, 1894, 1935) means a
corrected report can add data but can never remove wrong data — a bad trailer #, state, driver or
truck # sticks until someone deletes the whole shipment.

### 14. Inline stop-time editing reuses the previous date part
`updateRouteStopTime` (1521-1528) keeps the existing date and swaps in the `HH:MM` from the time
input. Typing `00:15` for a stop that happened after midnight backdates it a full day, which then
feeds the back-fill cascade at 2457-2472 and every downstream date comparison.

### 15. Unrecognized datetime formats degrade to a 120-minute guess
`formatCarrierDateTime` (1498) only accepts `M/D/YY[YY] H:MM[:SS] [AM/PM]`. An ISO or
`DD/MM/YYYY` export returns the raw text, `parseCarrierDateTime` returns `null`, and
`projectRemainingSchedule` falls to `ETA_DEFAULT_MIN` (2385) with no indication the timestamps were
never understood.

### 16. Time-only Excel serials are rejected
`excelSerialParts` (1481) requires `20000 < n < 70000`, so a cell holding a time-only serial (`0.5`)
is dropped.

### 17. Close-out assumes the last route stop is the delivery
`applyCloseoutRows` (1796) always writes `ord_completiondate` to `routeStops.length - 1`. If the
export's last stop is a yard return or a drop, the wrong stop is closed and the auto-Delivered flip
at 2482 fires off it.

---

## P2 — validation, staleness, and concurrency

### 18. Two entry points for Last Location, only one validated
The inline field in the Tracking Log (2843 → `updateLastLocationFromLog`) enforces `City, ST` and
recomputes the projection. The Check Call tab (`submitUpdate`, 1962) accepts arbitrary free text,
recomputes nothing, and writes an update whose `eta` is whatever was typed. `timeStringToMinutes`
then fails on anything that isn't `HH:MM` and `getNltMiss` bails out silently. Given NFI's core job
here is keeping this field current, the unvalidated path is the one to worry about.

### 19. The 30-second poll erases in-progress typing
`initApp`'s interval (3076) calls `loadShipments()` then `renderLog()`, which rebuilds `innerHTML`
(2863). Focus and caret position are restored (2869-2877) but the *value* comes back from state —
so a half-typed "SAN ANTON" in Last Location, Confirmed Location, or a departure time vanishes
mid-keystroke every 30 seconds.

### 20. Whole-record writes, no transactions
`saveShipmentRemote` (732) `set()`s the entire record. DCM editing Confirmed Location and NFI
editing Truck # on the same CS# at the same time means last-write-wins across *all* fields, not
just the edited one. `updateLastLocationFromLog` (2413) and `updateConfirmedLocation` (2194) also
write without an intervening `loadShipments()`, unlike `submitCarrier`/`submitUpdate`.

### 21. `stopLog` and `routeStops` are unreconciled parallel records
Check Call writes `stopLog[stopNum]` (1997-2004); TMW imports write `routeStops[]`. The Log renders
both (2822, 2854) and nothing keeps them consistent — and `stopLog` keys are positional, so a
re-import that changes the stop count leaves them pointing at different stops.

### 22. Report-derived text interpolated unescaped into markup and handlers
`fillDatalist` (807), `rebuildEmptyDropdownOptions` (1183), and the row handlers in `renderLog`
(2830, 2838, 2842-2847) inject `s.csNumber` / trailer IDs straight into `onclick="...('${...}')"`
and attribute values. A quote or apostrophe from an import breaks the row's controls. `escapeAttr`
already exists and is used elsewhere in the same function.

### 23. Small drift worth cleaning while nearby
- History tab copy (553) says loads move over "~45 minutes after being marked Delivered", but
  `HISTORY_GRACE_MIN = 0` (625) archives immediately.
- `statusMeta(status, s)` is called with two arguments (2810, 2909); the function takes one (2036).
- `formatNltDisplay('2400')` renders `24:00` (2631).
- `deleteShipment` (2066) removes the live record without archiving it to `history/`.

### 24. Everything runs on browser-local time
`new Date()`, `getHours()`, `todayStr()`, `getStatus`'s release comparison (2030) and the NLT
minute math are all local-clock; Excel serials are read as UTC (1486-1489). Correct while everyone
is in Central Time — worth stating explicitly, because a laptop on the wrong TZ shifts every ETA
and NLT verdict without any visible symptom.

### 25. Access model (noting, not proposing)
Passcodes are compared client-side (3092-3114) and role gating is `style.display` only; the
Firebase config and Mapbox token ship in the file. That's a speed bump, as the comment says. The
one thing worth checking outside this file is that the Realtime Database rules aren't
world-writable, since the database URL is public in the page.

---

## Suggested order of work

1. **Items 1, 2, 3, 4, 6** — the NLT chain. These are what "confidently wrong NLT status" looks
   like, and 1 and 2 are wrong in the direction of *not* alerting.
2. **Items 5, 18, 19** — location/ETA freshness and the unvalidated second entry point.
3. **Items 7, 8, 9, 10** — parsing robustness, all four verified reproductions.
4. **Items 12, 13, 14, 17** — import overwrite/clearing semantics.
5. Everything else.
