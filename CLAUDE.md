# CLAUDE.md — Outbreak Monitor Persistent Context

## Project Overview

**Outbreak Monitor** is a single-page dashboard that surfaces infectious-disease surveillance data from three public sources in near-real time. A visitor opens `index.html` (served as a GitHub Pages site) and sees:

- **US NNDSS panel** — weekly case counts for nationally notifiable conditions, pulled from CDC's Socrata API. Conditions are surfaced by an outbreak *signal* computed from the data (year-over-year elevation), not from a hardcoded disease list.
- **Global linelists panel** — individual-level outbreak records from Global.health.
- **WHO DON panel** — the 20 most recent Disease Outbreak News alerts from the WHO API.

**Three-layer architecture:**

| Layer | Source | Granularity |
|---|---|---|
| US national surveillance | CDC NNDSS (Socrata) | Weekly counts by disease and state |
| Global linelists | Global.health | Line-level records |
| WHO alerts | WHO DON REST API | Alert-level documents |

Everything runs client-side. There is no backend, no build step, and no dependencies beyond vanilla JavaScript.

---

## Data Sources

### 1. CDC NNDSS (Socrata)

**Base dataset:** `https://data.cdc.gov/resource/x9gk-5huc`

**Confirmed live JSON field names:** `year`, `week`, `states`, `label`, `m1`, `m2`, `m3`, `m4`

| Field | Meaning |
|---|---|
| `year` | Surveillance year (string, e.g. `"2025"`) |
| `week` | MMWR week number (string, e.g. `"18"`) |
| `states` | State abbreviation or geographic label |
| `label` | Disease/condition name |
| `m1` | Current-week case count |
| `m2` | 52-week maximum |
| `m3` | Year-to-date current year |
| `m4` | Year-to-date previous year |

**Quirks:**
- The dataset is exposed as both CSV and JSON on Socrata; the two formats use different field names. The JSON field names above are the ones confirmed to work.
- Aggregate rows (e.g. `TOTAL`, `U.S. Total`, `U.S. Residents`, `Non-U.S. Residents`, `U.S. Territories`, and US Census regional labels like `NEW ENGLAND`, `MIDDLE ATLANTIC`, `EAST NORTH CENTRAL`, `WEST NORTH CENTRAL`, `SOUTH ATLANTIC`, `EAST SOUTH CENTRAL`, `WEST SOUTH CENTRAL`, `MOUNTAIN`, `PACIFIC`) appear in the dataset alongside state-level rows and will produce double-counting if included. Filter them out client-side. **Match them punctuation-insensitively.** The live labels carry periods (`U.S. Residents`), so a bare `area.toUpperCase()` compare against a period-free constant like `US RESIDENTS` silently fails to match, letting the national rollup through — it then inflates the headline YTD *and* renders as a phantom "state" row (this was the source of the Hantavirus 20-vs-10 double count). Normalize by deleting periods and folding hyphens/whitespace before comparing (see `normalizeArea()` / `isAggregateArea()` in `index.html`).

**Which conditions get surfaced (signal-based, no hardcoded list):** The `x9gk-5huc` feed
carries *every* nationally notifiable condition (100+), most of them steady endemic
background (chlamydia, gonorrhea, salmonellosis, ...) whose counts dwarf any emerging
outbreak. The panel must show what is *anomalously active*, not the endemic baseline and
not a fixed set of names.

This used to be done with a static `DISEASE_KEYWORDS` allowlist + a `matchDisease()`
substring check — anything not pre-listed was dropped. That is exactly wrong for an outbreak
monitor: a genuinely new outbreak (cyclosporiasis was the case that exposed it) could never
appear no matter how large it grew, and every new outbreak required a code edit. **Do not
reintroduce a name allowlist.**

The replacement lets the data decide. `processCDC()` aggregates *all* conditions (aggregate
areas still excluded), then surfaces one when it is running above its own prior-year
baseline, using fields the feed already provides:

- Group every row by `cleanLabel(label)` (folds footnote markers like `†`/`*` so state rows
  for one condition don't split into duplicate cards).
- `isSurfaced(d)`: keep a condition if `ytdCurrent (m3) >= SURFACE_MIN_YTD` (noise floor, 5)
  **and** it clears one of three paths:
  - *emergent* — `ytdPrevious`/m4 `== 0` with current cases (a new/re-emergent condition); or
  - *strong surge* (path 1) — `ytdCurrent / ytdPrevious >= SURFACE_SURGE_RATIO` (1.2 = at
    least 20% above last year), shown regardless of volume so genuine low-base surges appear; or
  - *sustained rise* (path 2) — `ratio >= SURFACE_SUSTAINED_RATIO` (1.15 = at least 15% above
    last year) **and** `outbreakExcess(d) >= SURFACE_MIN_EXCESS` (50 excess cases).
  Path 2 exists because a single ratio bar is magnitude-blind: without it a condition climbing
  15-19% while carrying hundreds of excess cases (a large real-world burden) would be dropped,
  while a tiny 12→19 jump surfaces at +58%. The `>= 1.15` guard on path 2 keeps the high-volume
  endemic giants out — e.g. chlamydia up 1% is ~5,000 raw excess cases but ratio ≈ 1.01, so it
  fails both paths. Flat endemic conditions (ratio < 1.15) fall away on their own.
- Rank by `outbreakExcess(d)` = `ytdCurrent - ytdPrevious` (excess cases over last year — an
  interpretable "how far above normal" measure that balances ratio and volume), cap at
  `SURFACE_MAX_CARDS` (24) so the panel stays focused.
- `m3`/`m4` are plain counts that sum correctly across states; the 52-week max (`m2`) does
  not, so it is not used for the surfacing decision. Emergent conditions have no prior-year
  baseline, so their card shows a "🆕 New this year" badge and a `NEW` YoY chip instead of a
  misleading `N/A · Stable`.

The thresholds (`SURFACE_MIN_YTD`, `SURFACE_SURGE_RATIO`, `SURFACE_SUSTAINED_RATIO`,
`SURFACE_MIN_EXCESS`, `SURFACE_MAX_CARDS`) are the only tuning knobs; adjust those rather
than adding disease names.

**Minnesota focus:** This dashboard gives Minnesota special prominence. In every disease
card's state breakdown, the Minnesota row is pinned to the top and highlighted (`.mn-row`),
shown even when MN is outside the top 20 states or has no reported row (rendered as 0). The
summary bar also carries a "Minnesota YTD" tile rolling up MN year-to-date cases across all
currently flagged (surfaced) diseases. The `isMinnesota()` helper matches the reporting area
case-insensitively as either `MINNESOTA` or `MN`, since the live field could use either.

**Critical two-step query pattern:**

Step 1 — find the latest available week for the current year:
```
GET https://data.cdc.gov/resource/x9gk-5huc.json?$select=max(week)%20as%20latest_week&$where=year='YYYY'
```

Step 2 — fetch all rows for that week:
```
GET https://data.cdc.gov/resource/x9gk-5huc.json?$where=year='YYYY'%20AND%20week=<N>&$limit=50000
```

Never assume a fixed week number. The dataset lags publication by one to two weeks, so the max-week query is required every session.

---

### 2. Global Priority Outbreaks (featured-outbreak slot)

This is a swappable slot that gives one current high-priority outbreak a detailed
breakdown. The outbreak it tracks is defined entirely by the `FEATURED_OUTBREAKS`
config object in `index.html`. When the tracked outbreak winds down, swap the config
for a new feed (see the data-freshness rule below for the signal to do so).

**Currently tracking:** Ebola, Bundibugyo virus (BDBV), DRC & Uganda 2026. WHO declared
a PHEIC on 2026-05-17; it is the largest BDBV outbreak on record.

**Data source:** `INRB-UMIE/Ebola_DRC_2026` (INSP daily SitRep pipeline), raw CSVs under
`data/insp_sitrep/processed/`. Confirmed live field names:

| File | Fields |
|---|---|
| `insp_sitrep__cumulative_confirmed_cases__daily.csv` | `nom`, `date`, `cumulative_confirmed_cases` |
| `insp_sitrep__cumulative_confirmed_deaths__daily.csv` | `nom`, `date`, `cumulative_confirmed_deaths` |
| `insp_sitrep__national_cumulative_confirmed_cases__daily.csv` | `nom` (=`DRC`), `date`, `national_cumulative_confirmed_cases` |
| `insp_sitrep__national_cumulative_confirmed_deaths__daily.csv` | `nom` (=`DRC`), `date`, `national_cumulative_confirmed_deaths` |

**Quirks:**
- Per-zone files use **different value-column names** than the national files
  (`cumulative_*` vs `national_cumulative_*`). Mixing them up returns zeros.
- The data is a cumulative daily series: take each zone's row at the **latest date**, do
  not sum across dates.
- Zone-name spelling variants exist within the same feed (for example `Mongbwalu` vs
  `Mongbalu`). Fold them with `data/aliases.csv` (`observed_name` -> `canonical_nom`)
  before aggregating, or the per-zone table double-lists the same zone.
- An `NA` zone row holds cases not yet assigned to a health zone. Keep it, labelled
  "Unassigned"; drop the `DRC` aggregate row from the per-zone table.
- Deaths cells may be `ND` (no data); treat as 0.

**Non-negotiable data-freshness indicator:** the section must always show whether the feed
is still live. `dataFreshness()` compares the feed's most recent date against the viewer's
current date and the config's `freshnessWindowDays` (currently 14). Inside the window the
section shows a green LIVE banner; past it the banner flips to a red STALE / replace-this-
section warning. This is how the user knows when to have the featured outbreak swapped out.

---

### 3. WHO Disease Outbreak News (DON)

**Endpoint for 20 most recent alerts:**
```
GET https://www.who.int/api/news/diseaseoutbreaknews?$top=20&$orderby=PublicationDateAndTime%20desc
```

**Critical pattern:** Always sort descending and take `$top=20`. Do NOT paginate. The API returns oldest records first by default, so without `$orderby=PublicationDateAndTime desc` you get the oldest 20 records, not the newest. Pagination accumulates old records that `$orderby` alone cannot fix once you have started iterating.

---

## Working Style

- Deliver complete files. Do not emit partial diffs or "add the following lines" instructions when a full file replacement is safer.
- No em dashes in output or code comments.
- No redirects to Amazon, Google, or other external stores for tooling.
- Plain acknowledgment of mistakes; no hedging or blame-shifting.
- Verify facts against live data before suggesting or committing a fix. See the next section.

---

## Critical Lesson: API Schema Verification Is Non-Negotiable

The CDC NNDSS dataset is available in at least two formats on the Socrata platform (CSV download and JSON API). The field names differ between formats. Code written against the CSV schema will silently return empty results or `undefined` values when run against the JSON endpoint, with no error thrown.

**Rule:** Any code change that touches an external API endpoint must be validated with a live `curl` call that returns HTTP 200 and contains real, non-empty data before the change is committed. Example:

```bash
# Confirm JSON field names for the NNDSS dataset
curl -s "https://data.cdc.gov/resource/x9gk-5huc.json?\$where=year='2025'%20AND%20week=18&\$limit=2" | python3 -m json.tool
```

If the response is an empty array `[]`, the query is wrong. Do not commit until you see rows.

---

## Development Log

| Entry | Description |
|---|---|
| **PR #6** | Codex-assisted fix. Implemented two-step max(week) query to find the latest available MMWR week before fetching rows. Corrected field names to the confirmed JSON schema: `year`, `week`, `states`, `label`, `m1`, `m2`, `m3`, `m4`. Added client-side exclusion of aggregate geographic rows (TOTAL, U.S. TOTAL, regional census group names) to prevent double-counting. Fixed WHO DON query to use `$orderby=PublicationDateAndTime desc` so the 20 most recent alerts are returned instead of the 20 oldest. |
| **Hantavirus DOM fix** | Corrected DOM append order for the hantavirus section; elements were being inserted out of sequence, causing the panel to render incorrectly. |
| **Featured-outbreak swap (Ebola BDBV 2026)** | Re-evaluated the Global Priority Outbreaks slot. The MV Hondius hantavirus outbreak concluded ~2026-05-11 (13 cases, source feed static), so it was no longer the most relevant outbreak. Replaced it with the Ebola Bundibugyo virus outbreak in DRC & Uganda (WHO PHEIC 2026-05-17; 782 confirmed / 181 deaths as of 2026-06-13; largest BDBV outbreak on record). New source: `INRB-UMIE/Ebola_DRC_2026` INSP SitRep CSVs, aggregated by health zone with `aliases.csv` canonicalization. Added a `dataFreshness()` live/stale indicator (green LIVE banner vs red STALE/replace banner, `freshnessWindowDays=14`) so it is always obvious when the feed has gone stale and the slot needs a new outbreak. Verified all feeds return HTTP 200 with real rows; national totals match WHO. |
| **Minnesota highlight** | Gave Minnesota special prominence in the US NNDSS section. Each disease card's state breakdown now pins Minnesota to the top and highlights it (always shown, even at 0 or outside the top 20). Added a "Minnesota YTD (tracked)" summary-bar tile that rolls up MN year-to-date cases across all tracked diseases. New `isMinnesota()` helper matches `MINNESOTA`/`MN` case-insensitively. Note: `data.cdc.gov` was not reachable from the build environment (network egress policy), so this display-only change was not live-curl verified; it does not alter the API query and the matcher tolerates either area representation. |
| **Aggregate double-counting fix** | Fixed a double-counting bug in the US NNDSS headline totals. The `U.S. Residents` national rollup row was passing through the aggregate filter because the filter compared `area.toUpperCase()` (which yields the dotted `U.S. RESIDENTS`) against period-free constants like `US RESIDENTS`, so it never matched. The rollup was then summed into the headline YTD *and* shown as a phantom state row — e.g. Hantavirus reported YTD 20 when the true per-state sum was 10 (10 states + 10 rollup). Replaced the ad-hoc uppercase compare with `normalizeArea()` (deletes periods, folds hyphens/whitespace) + a shared `isAggregateArea()` helper used both in `processCDC` (excludes the rows from all sums) and as a defensive second guard in `renderCDC` (keeps them out of the state table). Also caught the same latent mismatch for `U.S. Territories` and `Non-U.S. Residents`. Verified with a Node simulation (headline now equals the per-state sum; all real states/territories incl. `U.S. Virgin Islands` retained). `data.cdc.gov` remains blocked by the egress policy (403 CONNECT), so no live curl; the change is client-side filtering only and does not touch the API query. Reviewed the Ebola and WHO sections for the same bug class: the Ebola headline reads the independent national feed (not summed zones) and per-zone rows are deduped via `aliases.csv`, and WHO is a plain alert count, so neither double-counts. |
| **Signal-based disease surfacing (remove hardcoded allowlist)** | Replaced the static `DISEASE_KEYWORDS` allowlist + `matchDisease()` substring filter with a data-driven outbreak signal. Root cause of the reported issue: the US NNDSS panel only rendered conditions whose label matched a pre-listed name, so the active cyclosporiasis outbreak (never in the list) could never surface — counter to the dashboard's purpose. The old allowlist did serve a real need (the feed carries 100+ conditions dominated by high-volume endemic disease, so showing everything by raw count would bury outbreak signal), but an identity filter is the wrong tool: it can never surface a *new* outbreak and needs a code edit per disease. New design: `processCDC()` aggregates every condition (aggregate areas still excluded), then `isSurfaced()` keeps those with `m3` YTD `>= SURFACE_MIN_YTD` (5) that are either emergent (`m4` prior-year YTD `== 0`) or surging (`m3/m4 >= SURFACE_SURGE_RATIO`, 1.2); results rank by excess-over-last-year (`m3 - m4`) and cap at `SURFACE_MAX_CARDS` (24). Flat endemic conditions drop out on their own; new outbreaks appear automatically with a "🆕 New this year" badge. Labels grouped via `cleanLabel()` to fold footnote markers. No API/query change — `fetchCDC` already pulls all rows for the latest week, so this is purely client-side selection using fields already in use. `data.cdc.gov` remains egress-blocked (403 CONNECT), so no live curl; verified with a Node simulation (cyclosporiasis + emergent measles + surging pertussis surface; flat chlamydia, sub-threshold salmonellosis, and below-floor noise are excluded; rollup rows stay out of per-condition sums) and a JS syntax check of the inline script. |
| **Surfacing refinement (add absolute-burden path)** | The live panel was surfacing only ~4 conditions and felt a touch strict. Root cause: the sole inclusion gate was a *relative* ratio (`>= 1.2`), which is magnitude-blind — it over-shows big proportional jumps off a tiny base (Haemophilus type b, 12→19 = +58%, only 7 excess cases) while dropping a condition climbing 15-19% that carries hundreds of excess cases, the burden surveillance actually cares about. `outbreakExcess` was used only for ranking, never for inclusion. Fix: added a second inclusion path in `isSurfaced()`. Path 1 (strong surge) is unchanged — `ratio >= SURFACE_SURGE_RATIO` (1.2), any volume. Path 2 (sustained rise) surfaces a condition with `ratio >= SURFACE_SUSTAINED_RATIO` (1.15) **and** `outbreakExcess >= SURFACE_MIN_EXCESS` (50). The `>= 1.15` guard on path 2 is what keeps the high-volume endemic giants out: chlamydia up ~1% is ~5,000 raw excess cases but ratio ≈ 1.01, so it still fails both paths. Net effect: a few more genuinely high-burden climbers surface, the list stays focused, and trivial small-base blips (e.g. 40→46, excess 6) stay out. Purely client-side selection; no API/query change. `data.cdc.gov` remains egress-blocked (403 CONNECT), so verified with an expanded Node simulation (strong-surge, low-base surge, high-burden moderate climber, small-base blip, just-under-threshold climber, flat mega-STI, declining, and emergent cases all classify as expected) plus a JS syntax check. |
