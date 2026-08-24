# Code review pass — correctness, performance, visualization

Reviewed `index.html` as a whole. Every item below is in the tree; the verification
column says how each was checked rather than asserted.

## Correctness

| # | Problem | Fix | Verified by |
|---|---------|-----|-------------|
| 1 | `sanitizeKey` output was interpolated straight into `id="..."`. It strips only the characters Firebase rejects in a path, so a CS# containing `"` closed the attribute and everything after it parsed as more attributes — an import-supplied `onerror=` became a live handler. | New `domKey()` produces `[A-Za-z0-9_-]` only, with a short hash appended when characters were replaced so two CS#s can't collapse onto one id. Every element id now goes through it; `sanitizeKey` keeps its Firebase-path job. | Rendered a row with `CS"1<img src=x onerror=alert(1)>`: before, the `<img>` was a real element in the DOM; after, zero injected nodes and the expander works (it didn't before — the id mismatch broke it). |
| 2 | Report-derived text went unescaped into inline handlers (`onclick="…('${s.csNumber}')"`) and into cell text. An apostrophe from an import broke every control on the row. | New `jsArg()` escapes for the JS string literal then the HTML attribute; applied to all 18 handler sites. Cell values escaped in the Tracking Log and History rows. | Handler attribute for a CS# containing `'`, `"` and markup parses and round-trips the exact string. |
| 3 | `cascadeRouteStopEvents` anchored on `new Date()` when the clicked event had no timestamp. `applyCloseoutRows` writes `departureDate` and *then* ticks Arrived, so the whole back-fill was anchored to import time — the last stop got an arrival later than its own departure. | The anchor now comes from a time on the clicked stop itself (arrival ± dwell) before it falls back to now. | Close-out sequence on a 3-stop route: no arrival-after-departure anywhere. |
| 4 | The backward walk only ever subtracted synthetic drive/dwell estimates, so a back-fill could stamp an arrival earlier than a departure a dispatcher had typed. | Re-anchors onto recorded times during the walk, then a forward pass clamps every back-filled value against the times already on the record. | Route with a known 08:00 stop-0 departure: the whole event sequence comes out monotonic. |
| 5 | The same guard changed from "is empty" to "does it parse", which would have overwritten timestamps `formatCarrierDateTime` stores verbatim (date-only, ISO). | A value that is present but unparseable is never re-anchored to and never written over. | `2026-03-14 14:30` survives a cascade untouched while the empty stops fill in. |
| 6 | `gateOutDateTimeForShipment` placed a time-only gate out on the release date with no rollover, so `01:15` against a 22:00 release stamped ~21 hours *before* release and back-dated the route. | Rolls forward a day when the stamp lands more than 12 hours before the anchor; a modest early gate out is left alone. | `01:15` → next day; `21:40` → same day. |
| 7 | `applyGateOutToFirstRouteStop` marked `routeStops[0]` arrived + completed unconditionally, reporting a delivery that never happened on routes whose first stop is a store. It also set arrival equal to departure (zero dwell at the shipper). | Refuses a store-first route; derives arrival from departure minus dwell. | Store-first route untouched; shipper-first gets departure and a derived arrival. |

## Performance

Measured in Chromium on a 121-shipment board, before vs after:

| | before | after |
|---|---|---|
| Tracking Log render | 110 ms | 43 ms |
| re-render with 3 rows expanded | 103 ms | 22 ms |
| detail-panel cells carrying markup | 2057 | 51 |
| filter options in the DOM with every panel closed | 252 | 0 |

- **Detail panels render on first open, not always.** Route stops, the projected
  schedule, the stop log and every location ping were built for every shipment on every
  re-render and then hidden. Same change applied to History and Backhaul rows.
- **Filter dropdowns build only the open panel.** Five Tracking Log columns and twelve
  backhaul columns each rendered their full option list into a `display:none` panel, on
  every keystroke and every 30-second poll.
- **Empty-trailer assignment is indexed once per render** instead of rescanning every
  shipment for every trailer on every row (rows × trailers × shipments). The index is
  released in a `finally` so it can never outlive the render.
- **Sort keys computed once per row.** The "NLT Leave By" key walks the whole route
  through the store lookup; it was being recomputed inside the comparator.
- **Search is debounced** by one 120 ms frame instead of rebuilding the board per keystroke.

## UI and data visualization

- **Delivery Trend chart** (Admin/Data) — one column per operating day, split into store
  stops that beat their NLT and stops that missed it, with the on-time share direct-labelled
  on top. One axis: height is the day's recorded volume, so a bad day and a light day don't
  look alike. Hover gives the full count including stops still awaiting an arrival. Labels
  thin automatically so a month-long range doesn't overprint.
  - Colors are the reserved status pair, not a categorical ramp. Light steps
    (`#3E9E6E` / `#8A2A22`) pass all six checks against the light surface. The dark skin
    reuses the status colors the pills already use (`#31D17F` / `#FF4E5F`), which sit at
    deutan ΔE 7.5 — inside the floor band, legal only with a second encoding — so the late
    segment is hatched and every column carries a direct label and a legend.
- **NLT Alerts panel is tiered.** It listed every unverified load unbounded; with 121
  in-transit loads that pushed the entire Tracking Log below the fold. NLT Miss and At Risk
  are the alerts and get the headline count; Unverified collapses to one line naming the
  commonest reason, and the body is height-capped.
- **Store Service misses get an inline bar** scaled to the worst store in range, so the
  column ranks at a glance instead of being read numeral by numeral.
- The commodity KPI bar used a hardcoded `#31D17F` outside the token set; it now reads
  `--chart-ok` and is unchanged on the dark skin.

## Simplification

- `emptyDeliveryTally` / `tallyStoreStopAgainstNlt` — the definition of a measured store
  stop lived inline in the commodity KPI; it is now in one place and shared with the trend.
- `cascadeRouteStopEvents` lost a redundant special case (the departure branch was already
  covered by the previous event in the same walk).

---

# Follow-up: automatic pickup stamping, and elapsed time on the status pill

## Automatic stamping never overwrites a time

The rule now: **an automatic path fills a blank time; only a person replaces one that is
already there.** Six paths wrote times without checking whether something was already
recorded, so a stop checkbox, a re-import, or a close-out file could silently replace a
departure a dispatcher had typed.

| Path | Was | Now |
|---|---|---|
| Stop checkbox → `deriveShipperDepartureTime` (`updateRouteStopField`) | overwrote `departureTime` | fills it only if blank |
| Same, in History (`updateHistoryRouteStopField`) | overwrote `departureTime` | fills it only if blank |
| Outbound import gate out (`importBulk`) | overwrote `departureTime` | fills it only if blank |
| Carrier import shipper departure (`importCarrierRows`) | overwrote `departureTime` | fills it only if blank |
| Carrier re-import stop merge (`mergeRouteStopData`) | incoming report replaced `arrivalDate` / `departureDate` | existing value wins; incoming fills blanks |
| Close-out import (`applyCloseoutRows`) | wrote the completion date over the last stop's `departureDate` | fills it only if blank |

Deliberately unchanged:

- **`updateDepartureTime`** — the Departed field's own editor. That is a person correcting
  the value on purpose, so it still replaces, and still clears when emptied.
- **The Departed *state* still flips.** `markDepartedIfActive` runs whether or not the time
  was taken; only the timestamp is protected.
- **Arrived / Completed flags still OR together** on re-import, so a checkbox ticked by hand
  and one set by a report both survive.
- **Planned fields** (`earliestDate`, `carrierEta`, `latestCheckCallTime`) still take the
  newer report's value — those are the carrier's to revise. The *actuals* are not.

Lateness is now judged against the time actually on the record rather than against the
value the auto path offered, so a load isn't cleared of a late departure by a gate out that
was never written.

## The status pill carries the overage

`past_scheduled` rendered as "Past Scheduled Pickup", which read identically at one minute
over and at six hours over. It now reads `Past 8m`, `Past 47m`, `Past 2h 15m`, `Past 6h 10m`
— the elapsed time since the Pick Up Start date and time, refreshed on every render (the
board already re-renders on a 30-second poll). It also fits on one line, where the old label
wrapped to two.

`getStatus` and the pill now share `scheduledDepartureDateTime()`, so the status decision
and the number reported for it can't disagree about when the deadline was. `statusMeta` was
already being *called* as `statusMeta(status, s)` in both the log and the CSV export while
declaring one parameter; it now uses the second, so the export carries the same label.

Unchanged: the "Past Scheduled Pickup" stat tile (a count) and the status filter option (a
filter key), both of which name the category rather than one load's overage.

### Verified

- Status labels at −90 / +8 / +47 / +135 / +370 minutes, plus a departed load, render as
  `Pending Pickup`, `Past 8m`, `Past 47m`, `Past 2h 15m`, `Past 6h 10m`, `In Transit` — on
  the board and through `statusMeta` as the CSV export calls it.
- A typed `07:45` survives a stop-checkbox cascade that derived `09:30`; a blank one gets
  `09:30`; a manual edit to `11:22` still wins.
- A re-import carrying `23:59` leaves an existing `21:40` / `22:10` alone, still OR-s the
  arrived/completed flags, and still fills a stop that had no times.
- Gate-out stamping leaves an existing first-stop departure alone.


---

# Auto-pickup: a Delivered status no longer invents a stop time

Geofence entry/exit are used as the stop's Arrival and Departure exactly as before — when
the carrier reports an actual time, that is a real recorded event and it counts.

What did not count as one, and was being written anyway: when a load's **Load Status** came
back Delivered, `markShipmentDeliveredFromCarrierImport` auto-picked-up the final stop by
stamping its arrival **and** departure with `Terminated At`, or failing that
`Last Updated Timestamp` — the moment the report was generated, which is identical across
every row in the file — and falling back to the import time if neither parsed.

In the shipped `Loads_20260824095216_549737.xls` that value is `08/24/2026 03:03` on all 110
rows, a full day before the 08/25 appointments. A store stop with no real time was therefore
recorded as arriving at 03:03 the previous day, with zero dwell, and scored against its NLT
on that basis.

A Delivered status is the carrier asserting the stops happened, not a record of when. The
function now sets the arrived/completed flags and the shipment state, and writes no times.

## What still supplies a stop time

| Source | Counts? |
|---|---|
| Geofence Entry / Exit with an actual value | **Yes** — unchanged |
| Somebody typing in the Arrival / Departure cell | **Yes** — unchanged |
| Gate Out Time on an outbound import | **Yes**, into a blank first stop, unchanged |
| The back-fill cascade behind the stop checkboxes | **Yes**, into blanks, unchanged |
| A Load Status of Delivered, with no time anywhere | **No** — this is the change |

A delivered stop with no recorded time now simply has none. It shows as a store stop in the
delivery KPI but not a measured one, instead of being measured against a timestamp that was
never an arrival.

### Verified

Against the real report's column layout, load CS10191934 with `Load Status = Delivered` on
both stops, geofence times on the shipper stop and none on the store stop:

- The shipper stop keeps its real geofence values as Arrival `08/25/2026 09:05` and
  Departure `08/25/2026 09:48`, and `deriveShipperDepartureTime` returns `09:48`.
- The store stop is flagged arrived and completed with no arrival or departure, where it
  would previously have been stamped `08/24/2026 03:03`.
- Neither stop is scored against an NLT off a fabricated time.


---

# "Auto-pickup" is a word, not a time

The Loads export does not leave the geofence columns blank when the platform infers a stop.
It writes the literal text **`Auto-pickup`** or **`Auto-delivery`** into `Geofence Entry Time`
and `Geofence Exit Time`.

`formatCarrierDateTime` passes text it doesn't recognise through verbatim — deliberately, so a
carrier value is never silently destroyed — so those words were stored as the stop's arrival
and departure. Downstream, a non-empty string is a time:

- `arrived: !!arrivalDate` flagged the stop arrived off the word.
- The stop grid's date and time inputs rendered blank, because the value won't parse — so the
  record held `Auto-pickup` while the cell looked empty.
- The back-fill cascade then refused to fill that stop, because the guard is "is something
  already recorded here" and something was.
- On the Backhaul Log, the **Pickup Actual** column displayed the word `Auto-pickup` where a
  timestamp belongs.

`geofenceDateTime()` now gates both geofence columns: a value that parses as a datetime is
used exactly as before, and anything else is treated as no reading. Applied to both
`extractLoadsCarrierImportRows` (shipments) and `extractBackhaulLoadsRows` (backhauls).

The backhaul pickup also had `formatCarrierDateTime(row.departureDate || row.pickedUpTimestamp)`.
That fallback is dropped: when the geofence reads `Auto-pickup`, `Picked Up Timestamp` carries
the platform's inferred pickup time — the same auto stamp under another name. Where the geofence
is real, the two columns agree anyway (36 of 46 pickup rows in the sample export are identical;
the other 10 are exactly the `Auto-pickup` rows).

## Measured on the shipped export

`Loads_20260824094854_666281.xls`, 125 stop rows across 48 loads, every load Delivered:

| | before | after |
|---|---|---|
| Backhaul time fields holding `Auto-pickup` / `Auto-delivery` | 22 (across 11 loads) | 0 |
| Shipment stop time fields holding them | 28 | 0 |
| Backhaul loads with a pickup departure | 46 | 36 |
| Shipment stops carrying a time | 119 | 105 |
| Shipment stops flagged arrived | 125 | 124 |

The arrived count barely moves because `Stop Status` still counts: `Completed` is the carrier
asserting the stop happened, and that assertion is kept. The one stop that loses the flag had
`Stop Status = Delayed`, which asserts nothing, and was previously flagged arrived purely
because the word `Auto-pickup` was sitting in its arrival field.

Load CS10190639 is the whole change in one row — Pickup Actual goes from `Auto-pickup` to `—`,
while its real Delivery Actual `08/24/2026 04:00` and its On Time status are untouched.

## Note on the previous commit

The earlier fix to `markShipmentDeliveredFromCarrierImport` — a Delivered load status stamping
the report's own generation timestamp as the final stop's arrival and departure — was a
different bug found on the way here, and it stands on its own. It is unrelated to the
`Auto-pickup` text handled above.
