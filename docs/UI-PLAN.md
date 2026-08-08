# NFI Tracking — visual direction

Reskin only. No changes to the data model, Firebase structure, function signatures, element IDs, or
any class name that JavaScript reads or writes.

## Principles

1. **The three things that matter get the most visual weight.** Last Location, ETA, and the NLT
   Alerts panel are the reason this tool exists. Everything else recedes.
2. **Glanceable beats pretty.** Target is a dispatcher reading this at arm's length for two seconds
   at a time, on a 1440px laptop, many times an hour. Nothing gets smaller or lower-contrast than
   it is today.
3. **Color carries one meaning at a time.** Right now NFI brand red, "past scheduled" red, and
   "NLT miss" red are the same red; role chips reuse the status blue and amber. Each hue gets one
   job.

## Color system

Semantic tokens layered over a fixed ramp. Existing token names stay defined (`--red`, `--green`,
`--steel`, `--brand`, `--amber`, `--blue` and the `-bg` pairs) because `showToast` and the CSS both
reference them — new tokens are added alongside rather than replacing them.

**Neutrals** — 8 steps from page background to ink, replacing the current ad-hoc `#F4F6F8` /
`#FBFCFD` / `#B7C0C9` / `#9CAAB8` greys:

    --n-0  #FFFFFF   surface
    --n-25 #FAFBFC   inset / input rest
    --n-50 #F2F5F8   table header, hover
    --n-100 #E5EAEF  hairline borders
    --n-200 #CBD5DF  strong borders, disabled
    --n-400 #8494A4  tertiary text
    --n-600 #52616F  secondary text
    --n-900 #101821  primary ink

**Status** — four states, each with fg / bg / border, used *only* for shipment and NLT status:

    critical  #B0322A on #FCEDEB, border #E9BDB7   NLT Miss, Past Scheduled, Trailer Pending
    warning   #9A5B12 on #FDF3E4, border #EFD7B0   At Risk, Ready to Depart, Moving, Errand
    ok        #1E6B44 on #E6F4EB, border #B7DCC6   In Transit, Departed, on-time NLT
    neutral   #52616F on #EEF1F4, border #DCE2E8   unknown / no data

**Brand** — `--brand #EE3124` is retired from status duty and reserved for the NFI wordmark, the
active tab, and the primary action button. This is the single biggest legibility win: red in a data
cell will then always mean "something is wrong".

**Role** — its own axis, not borrowed from status:

    DCM   steel  #233C57  (existing --steel)
    NFI   indigo #3D4E9C

Both roles appear as a filled badge in the header *and* as a 3px bar across the top of the app
shell, so the active passcode is unmistakable without reading anything.

## Typography

Three families kept, each with one job, plus system fallbacks so a slow font load doesn't reflow
the board:

- **Barlow Condensed 700/800** — the header wordmark, stat numerals, and panel titles only.
- **Inter 400/500/600/700** — all UI text, labels, table body.
- **IBM Plex Mono 500/600** — every identifier and every time value: CS#, trailer #, truck #,
  release, departed, ETA, NLT, projected arrival/departure. Times currently drift between mono and
  Inter; making that consistent is most of the density improvement on its own.

Scale (replacing the current 9.5/10/10.5/11/11.5/12/12.5/13/13.5/14/16/22/24/28 spread):

    11  micro    column headers, pills, badges
    12  small    table body, hints
    13  base     form inputs, filter controls
    15  emphasis Last Location input, ETA value
    18  section  card titles, NLT panel header
    24  display  stat numerals
    32  hero     ETA/NLT count where it earns it

`font-variant-numeric: tabular-nums` on every numeric and time cell so columns of times align
vertically — currently they ragged in proportional Inter.

## Spacing

One 4px scale: 4 / 8 / 12 / 16 / 24 / 32, replacing the current 3, 5, 6, 7, 9, 11, 14, 18, 20, 22,
26, 34 mix. Table cells move from `7px 6px` to `8px 10px` — more horizontal breathing room without
adding rows' height, so the same number of shipments fit on screen.

Radii: 6px controls, 10px cards, 999px pills. Two shadow levels only (dropdown, toast).

## Layout and hierarchy changes

**Last Location + ETA — promoted.** Today they are columns 14 and 15 of 17, at 11.5px, visually
identical to Consignee. Proposed: group them as a "Live Position" pair at the right edge with
`position: sticky` so they never scroll out horizontally, a tinted background band distinguishing
them from the rest of the row, a 15px input for Last Location, and the ETA rendered as a 15px mono
value with its NLT context beneath it. The stale treatment stays exactly as it behaves now
(cleared value, `Needs Update` pill, red-bordered input) but reads at a distance instead of as a
small pink box.

**NLT Alerts — promoted.** Moves to the top of the Tracking Log as a sticky panel with a hard
left rule, a large count, and two clearly separated tiers: NLT Miss as solid critical, At Risk as
outlined warning. One deliberate behavior note: today the card is `display:none` when there are no
alerts, which makes "all clear" and "the panel is broken" look identical. I'd like to render a
quiet green "No NLT exposure — N shipments checked" line instead. That is a one-line change in
`renderNltAlerts`'s empty branch and touches no data — say the word if you'd rather it stay hidden.

**Tracking Log rows.** Full-row red/amber washes are replaced by a 3px left accent bar plus a 4%
tint, so status pills stay legible against the row instead of fighting it. Sticky table header.
Hairline borders at `--n-100`. Column-group separators between the DCM block (CS# → Confirmed Loc.)
and the NFI block (Empty Trailer → ETA), which also reinforces who owns which fields.

**Expanded stop details.** The nested tables become an inset panel on `--n-25` with its own small
caption, aligned time inputs on a consistent 96px width, checkboxes at a 24px hit target (up from
the browser default ~13px — real gain for fast clicking), and the NLT column set in mono and
weighted, since that column is the reason the panel gets opened.

**Forms and import zones.** Cards get a titled header row; the 3-column form grid stays. Drop zones
get a clearer resting state and a stronger `.dragging` state. All existing markup and handlers are
untouched.

## Explicitly preserved

Every one of these keeps its current behavior and its current DOM contract:

- Checkbox cascade — `updateRouteStopField` handlers, `stopcell-${domCs}-${i}-${col}` IDs.
- Keyboard grid navigation — `handleStopGridNav`, same IDs, same focus order.
- Drag-and-drop import — `setupDropZone`, the `.dragging` class, all four zone IDs.
- Inline time/text editing — `.inline-edit`, `onblur`/`onkeydown` handlers, `.stale-prompt`.
- Class names read from JS: `status-pill` + `green|amber|red|grey`, `row-past`, `row-soon`,
  `row-ready`, `row-terminal`, `commodity-pill`, `armed`, `active`, `cf-*`, `stat` + color.
- Every element ID, every `datalist`, the expander/history ID scheme, `sanitizeKey` output.
- Uppercase-as-you-type on text inputs, focus/caret restoration across re-render.

## Sequence

1. Token block + typography + spacing primitives (`:root`, base elements). Visually neutral pass.
2. Header, role treatment, tabs, gate.
3. Tracking Log: table chrome, row accents, sticky header, column groups.
4. Last Location / ETA promotion, NLT panel.
5. Expanded stop details, projected schedule table, forms, import zones, history.

Reviewable at any step — each is CSS plus, at most, additive wrapper markup.
