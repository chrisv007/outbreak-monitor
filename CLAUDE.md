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
- **High-consequence severity tier (surfaces before the trend logic even runs).**
  `isHighConsequence(d)` matches a condition's cleaned label against `HIGH_CONSEQUENCE_PATTERNS`
  — a *deliberately narrow* set of pathogens whose expected US baseline is essentially **zero**,
  so a single case is genuinely anomalous: epidemic-prone, high-lethality, bioterror, and
  elimination/eradication-target diseases (measles, rubella, polio, diphtheria, plague, anthrax,
  smallpox, cholera, yellow fever, the viral hemorrhagic fevers, novel/variant influenza A,
  SARS/MERS/novel coronavirus). A high-consequence condition surfaces on **any confirmed case**
  (`ytdCurrent >= HIGH_CONSEQUENCE_MIN_YTD`, 1) **and** only when there is something to see — it
  is emergent, active this week (`currentWeek > 0`), or not declining (`ytdCurrent >=
  ytdPrevious`). A condition that is merely quietly *declining* high-consequence background does
  not surface. **Urgent** high-consequence signals (`isUrgentHighConsequence` — emergent, active,
  or rising) are **pinned to the top** so a small severe cluster is never buried; a present-but-
  flat/declining one (e.g. a single endemic plague case) is still shown but ranks by excess like
  everything else, so it does not outrank a major climbing outbreak. High-consequence cards carry
  a distinct red `⚠ High-consequence` badge; the YoY chip still shows the real trend.
  - **Two hard lessons, both learned from the live panel (the only place the CDC feed is
    reachable):** (1) *Scope it narrowly.* An earlier version included conditions with real
    endemic US baselines — botulism (~140 cases/yr), tularemia (~200/yr) — which flooded the top
    of the panel with declining endemic background wearing an alarm badge and buried the actual
    risers. For those, a single case is normal, not a signal, and a genuine cluster still surfaces
    on its own through the trend paths. They were removed. Do **not** re-add endemically-present
    conditions here. (2) *Match whole words, not substrings.* `HIGH_CONSEQUENCE_REGEXES` wraps
    each pattern in `\b...\b`. A bare-substring version let `cholera` match the bacterium name
    `Vibrio cholerae` inside the common, declining *Vibriosis* label and mis-flagged it as
    high-consequence. Word boundaries flag real `Cholera` while ignoring `cholerae`, and still
    cover NNDSS sub-labels (`Measles, Indigenous`, `Poliomyelitis, paralytic`) because the
    boundary falls on the comma/space.
  - Why the tier exists at all: the trend signal is magnitude- and change-driven, so it
    *structurally* suppresses exactly the pathogens where a couple of cases is a five-alarm fire —
    a 2-case novel-influenza or anthrax cluster sits under `SURFACE_MIN_YTD`, and even above it
    ranks last by excess and falls off the cap. Missing a small high-consequence signal is the
    worst failure mode for an early-warning monitor, and no amount of loosening the ratio/excess
    knobs fixes it (that only pulls in endemic noise). **This is NOT a return of the banned
    `DISEASE_KEYWORDS` allowlist.** That allowlist was the *only* path to surface, so any unlisted
    condition (a new outbreak) was permanently invisible. This tier is purely **additive**: it can
    only *raise* a dangerous pathogen's visibility, never suppress anything — every condition is
    still independently scored by the data-driven signal below, so new/unlisted outbreaks still
    appear automatically.
- Otherwise (not high-consequence), `isSurfaced(d)`: keep a condition if
  `ytdCurrent (m3) >= SURFACE_MIN_YTD` (noise floor, 5) **and** it clears one of three paths:
  - *emergent* — `ytdPrevious`/m4 `== 0` with current cases (a new/re-emergent condition); or
  - *strong surge* (path 1) — `ytdCurrent / ytdPrevious >= SURFACE_SURGE_RATIO` (1.2 = at
    least 20% above last year), shown regardless of volume so genuine low-base surges appear; or
  - *sustained rise* (path 2) — `ratio >= SURFACE_SUSTAINED_RATIO` (1.10 = at least 10% above
    last year) **and** `outbreakExcess(d) >= SURFACE_MIN_EXCESS` (50 excess cases).
  Path 2 exists because a single ratio bar is magnitude-blind: without it a condition climbing
  10-19% while carrying dozens-to-hundreds of excess cases (a real-world burden) would be
  dropped, while a tiny 12→19 jump surfaces at +58%. The `>= 1.10` guard on path 2 keeps the
  high-volume endemic giants out — e.g. chlamydia up 1% is ~5,000 raw excess cases but ratio
  ≈ 1.01, so it fails both paths. Flat endemic conditions (ratio < 1.10) fall away on their own.
  **Keep this floor a real distance below the 1.2 surge bar.** It was originally 1.15 — only a
  sliver under 1.2 — so path 2's *effective* reach was just the `[1.15, 1.20)` window that path 1
  did not already cover. That window is rarely populated, so path 2 changed almost nothing on the
  live panel (the "it didn't work" report), and any high-burden condition rising 10-15% was still
  dropped. Widening to 1.10 gives the absolute-burden signal a band it can actually act in.
- Rank by `outbreakExcess(d)` = `ytdCurrent - ytdPrevious` (excess cases over last year — an
  interpretable "how far above normal" measure that balances ratio and volume), cap at
  `SURFACE_MAX_CARDS` (24) so the panel stays focused.
- `m3`/`m4` are plain counts that sum correctly across states; the 52-week max (`m2`) does
  not, so it is not used for the surfacing decision. Emergent conditions have no prior-year
  baseline, so their card shows a "🆕 New this year" badge and a `NEW` YoY chip instead of a
  misleading `N/A · Stable`.

The thresholds (`SURFACE_MIN_YTD`, `SURFACE_SURGE_RATIO`, `SURFACE_SUSTAINED_RATIO`,
`SURFACE_MIN_EXCESS`, `SURFACE_MAX_CARDS`, `HIGH_CONSEQUENCE_MIN_YTD`) are the tuning knobs for
the *trend* signal; adjust those rather than adding disease names to the trend logic. The one
list you *do* curate by name is `HIGH_CONSEQUENCE_PATTERNS`, and only to widen/narrow the
additive severity floor — never to gate what the trend signal can surface.

**Surfacing diagnostic (tune from real data, not guesses):** `data.cdc.gov` is unreachable
from every build/session environment here (egress policy, 403 CONNECT), so the thresholds can
never be validated against the live feed from inside a session — only in the viewer's browser,
which *can* reach CDC. To make that possible, `processCDC()` returns `evaluated` (distinct
conditions seen, aggregates excluded) and `surfacedCount` (how many passed `isSurfaced` before
the card cap), plus `highConsequenceCount` (how many surfaced via the severity tier), and
`renderCDC()` prints them into the `.cdc-note` line under the US section header ("Evaluated N
notifiable conditions · M running above prior-year baseline · P high-consequence flagged ·
showing top K"). This turns "why is the panel almost empty?" from a guess into something readable off the
live page: if `M` is small the data is genuinely quiet, if `M` is large but `K` is capped raise
`SURFACE_MAX_CARDS`. Do not tune the ratio/excess knobs blind — read this line on the live site
first.

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

**Currently tracking:** Ebola, Bundibugyo virus (BDBV), DRC, Uganda & international imports
2026. WHO declared a PHEIC on 2026-05-17; it is the largest BDBV outbreak on record.

**Data source:** Global.health `globaldothealth/outbreak-data`, the **Ebola BVD 2026 line
list (PUBLIC VIEW)** — one CORS-open CSV covering *every* affected country, so Uganda and
international imports appear alongside DRC. **CC BY 4.0** — Global.health is a data
*aggregator*, not itself a health authority, so the section carries an explicit attribution
and license note. Raw URL (URL-encoded spaces):
`https://raw.githubusercontent.com/globaldothealth/outbreak-data/main/Ebola%20BVD/Data/Ebola%20BVD%202026%20linelist%20-%20PUBLIC%20VIEW.csv`

**Why this replaced INRB-UMIE:** the former source (`INRB-UMIE/Ebola_DRC_2026`, INSP daily
SitRep) is **DRC-only by construction** — country-level cumulative CSVs for the DRC national
total and DRC health zones. Uganda and imports could *never* appear no matter how fresh the
feed. The whole INRB config and its cumulative-series aggregation logic were removed.

**Shape — this is a LINE LIST, not a cumulative series.** One record per case. Totals are
built by **counting and grouping records**, not by reading a cumulative value column. Key
columns (`parseCSV` matches header names *after trimming*, since the live file ships some with
trailing spaces and a lowercase `Health zone`):

| Column | Use |
|---|---|
| `Case_status` | `confirmed` / `suspected` / `probable` / `discarded` / `contact` |
| `Location_Admin0` | **case location** = country attribution key |
| `Location Admin1` | province (note: space, not underscore) |
| `Health zone` | sub-country drill-down granularity (mostly DRC) |
| `Outcome` | `Death` marks a confirmed death (else empty / `Recovered`) |
| `Nationality`, `Travel_history*` | **NOT used for attribution** (see traps) |
| `Date_confirmation` | drives the "recently confirmed (last 14 days)" signal |
| `Date_last_modified` | drives the live/stale indicator |

**Aggregation rules (in `fetchOutbreak`):**
- **Confirmed only in headline totals.** Filter to `Case_status == 'confirmed'`. The file also
  carries suspected, probable, discarded and contact records; `suspected` alone is ~900 of
  ~3,200 rows, so folding it into the total would badly distort it. Totals count confirmed
  records; deaths count confirmed records with `Outcome == 'Death'`.
- **Country totals are grouped DYNAMICALLY from the data**, keyed on `Location_Admin0`. There
  is no hardcoded country list, so a new country (e.g. the single confirmed France import)
  surfaces on its own with no code change. Sorted by case count.
- **Health-zone drill-down is preserved**, now derived from the line list: pick the largest
  affected country that actually carries zone detail (DRC in practice) and break its confirmed
  cases down by `Health zone` (empty -> `Unassigned`), rendered as a collapsible `<details>`
  under the country table.
- **Recently-confirmed** count over the last `recentWindowDays` (14) from `Date_confirmation`,
  shown as a live activity line.

**Two verified traps (design around them, do not regress):**
1. **Attribution must follow case location, never nationality or travel history.** Some
   confirmed *DRC* cases carry foreign nationality/travel history — e.g. ID 542, an *American*
   national with travel history to *Germany*, is correctly a **DRC** case (`Location_Admin0 =
   Democratic Republic of the Congo`, `Health zone = Nyankunde`). Grouping strictly on
   `Location_Admin0` handles this. Do not group on `Nationality` or `Travel_history_location`.
2. **The file is not clean RFC 4180.** It has CRLF endings, quoted fields with embedded commas,
   a **blank trailing unnamed column** (holds a stray source URL for exactly one row), and at
   least one **malformed date** (`20260715`, no dashes, in `Date_last_modified`). `parseCSV` is
   a full character state machine (handles quotes, `""` escapes, embedded newlines, CRLF) rather
   than a naive line/`,` split. Dates are normalized through `normISODate()` before any
   comparison: left raw, `20260715` would **beat** a real `2026-07-16` in a lexical max (because
   `'0' > '-'`) and then parse to `Invalid Date`, silently breaking the freshness banner.
   `normISODate` normalizes ISO, compact `YYYYMMDD`, and `M/D/YYYY`, rejecting out-of-range
   month/day, so the max stays honest.

**Non-negotiable data-freshness indicator:** the section must always show whether the feed is
still live. `dataFreshness()` compares the feed's own most-recent `Date_last_modified` (across
all records — any edit signals activity) against the viewer's current date and the config's
`freshnessWindowDays` (currently 14). Inside the window the section shows a green LIVE banner;
past it the banner flips to a red STALE / replace-this-section warning. This is how the user
knows when to have the featured outbreak swapped out. It is driven off the file's own dates, not
an assumption that the feed updates forever.

**Verification note:** unlike CDC (egress-blocked here), the Global.health raw CSV **is**
reachable from the session environment (CORS-open, confirmed HTTP 200). Every change to this
section was validated against the live file: a Node aggregation simulation (2,094 confirmed;
DRC 2,073 / Uganda 20 / France 1; 798 deaths; 46 DRC zones; latest 2026-07-16 with the
malformed value correctly *not* poisoning the max; 572 confirmed in 14d) plus a headless
Chromium render of `index.html` (country table + zone drill-down + LIVE banner, no JS errors).

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
- **Footer "last modified" timestamp is mandatory maintenance on every change.** The page
  footer carries a static `Dashboard page last modified: <date> <time> <CST|CDT>` line
  (`.build-timestamp` in `index.html`). It exists so the user can tell at a glance whether
  the live GitHub Pages deploy has actually picked up the latest commit — it is unrelated
  to the in-page "Refresh" control, which reflects client-side data fetch time, not the
  page build. **Any commit that changes `index.html` must update this string** to the
  current date/time in US Central time (CST or CDT, whichever is currently in effect —
  Central observes DST). Get the real current time before writing it, e.g.:
  `TZ=America/Chicago date "+%Y-%m-%d %I:%M %p %Z"`. Do not guess or reuse a stale value.

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
| **Footer "last modified" build timestamp** | The user could not tell whether the live GitHub Pages deploy actually reflected the latest committed `index.html`, since prior sessions had made several changes and page-cache/deploy-lag made it ambiguous. Added a small `.build-timestamp` line at the bottom of the footer reading "Dashboard page last modified: `<date> <time> <CST|CDT>`" — a static string, not computed client-side, deliberately independent of the existing "Refresh" control (which reflects data-fetch time, not page build time). Documented as a mandatory-maintenance rule in the Working Style section: every commit touching `index.html` must update this string to the real current US Central time. No API/data logic touched. |
| **High-consequence severity tier (additive early-warning floor)** | Prompted by a full external evaluation of the US Outbreak Surveillance panel. The evaluation's key correct insight: the panel is a *change-detector, not an outbreak-detector*, and its most dangerous failure is structural, not a mis-set threshold — a small high-consequence signal (novel influenza A, a viral hemorrhagic fever, plague, anthrax, low-count measles) is killed by `SURFACE_MIN_YTD = 5` and/or ranked off the 24-card cap because it carries few "excess" cases. Loosening the ratio/excess knobs cannot fix this; it only pulls in endemic noise. Fix: added `isHighConsequence()` + `HIGH_CONSEQUENCE_PATTERNS` (epidemic-prone / high-lethality / bioterror Category A / elimination-target pathogens) and an any-case floor `HIGH_CONSEQUENCE_MIN_YTD = 1`. In `isSurfaced()` a high-consequence condition surfaces on any confirmed case, bypassing the noise floor and both ratio gates; in `processCDC()` the sort pins high-consequence conditions to the top (by raw count) so they can't be capped out; the card shows a distinct red `⚠ High-consequence` badge while the YoY chip still reports the real (possibly flat/declining) trend; `processCDC()` returns `highConsequenceCount`, added to the `.cdc-note` diagnostic. **This is explicitly additive, not the banned `DISEASE_KEYWORDS` allowlist** — the allowlist was the *only* surfacing path (new outbreaks invisible); this tier can only *add* a dangerous pathogen, never suppress, and the data-driven signal still scores every condition independently. Deliberately did NOT act on the evaluation's other suggestions: a secondary "largest ongoing by volume" readout was declined (it would resurface the endemic giants — chlamydia/gonorrhea — the panel exists to exclude), pertussis staying out is *correct* (down ~69% YoY, genuinely not "running above last year"), and the foodborne/PulseNet blind spot is a data-source limit (aggregate NNDSS can't see WGS clusters), out of scope. Purely client-side selection; no API/query change. `data.cdc.gov` remains egress-blocked here (403 CONNECT), so verified with a 19-case Node simulation (low-count novel flu/anthrax/VHF/polio now surface; declining plague/diphtheria/botulism still surface via severity; anthrax at 0 cases does not; cyclosporiasis/measles/Hib still surface via trend; chlamydia, gonorrhea +8%, pertussis −69%, small blips, below-floor all still excluded; severity tier pins above trend cards) plus a JS syntax check. |
| **Severity-tier refinement: narrow the scope + whole-word matching + activity gate** | Follow-up after the high-consequence tier shipped and was seen on the live panel. The screenshot exposed two concrete failures. (1) *False match:* `HIGH_CONSEQUENCE_PATTERNS` matched with a bare substring test, so `cholera` matched the bacterium name `Vibrio cholerae` inside the common **Vibriosis** label — flagging Vibriosis (Probable 988, Confirmed 488, both *declining*) as high-consequence. Fixed with `HIGH_CONSEQUENCE_REGEXES` (each pattern wrapped in `\b...\b`); real `Cholera` still matches, `cholerae` no longer does, and comma/space-delimited NNDSS sub-labels still match. (2) *Endemic flooding:* the tier surfaced every listed pathogen on any case regardless of trend, pinned to the top by raw count. In practice that filled the top of the panel with declining endemic background — botulism (Infant, Foodborne, wound), tularemia, an already-covered declining Measles-Imported, a 1-case declining Cholera — all wearing a red alarm badge and burying the genuine risers (Cyclosporiasis +152%, Chikungunya +260%, Mpox). Two fixes: **scope** — removed conditions with real endemic US baselines (`botulism`, `tularemia`; also dropped bare `polio` in favor of `poliomyelitis`/`poliovirus`), leaving only pathogens that are abnormal to see at all (a genuine cluster of the endemic ones still surfaces via the trend signal); **activity gate** — a high-consequence condition now surfaces only when emergent, active this week (`currentWeek > 0`), or not declining (`ytdCurrent >= ytdPrevious`), and the sort pins only *urgent* high-consequence (`isUrgentHighConsequence` — emergent/active/rising) to the top, letting a present-but-flat rare case (e.g. a single endemic plague case) still show but rank by excess so it never outranks a major climber. Net effect on the screenshot data: 15 cards → 7, led by Measles Indigenous (+61%), then the real risers by magnitude, with Plague flagged at the bottom; Vibriosis/botulism/tularemia/declining-cholera all gone. The tier's core value is preserved — a Node simulation confirms emergent/rising novel-influenza A, anthrax, VHF, polio, and diphtheria clusters still surface and pin urgent. Purely client-side selection; no API/query change. `data.cdc.gov` remains egress-blocked here (403 CONNECT), so verified with a Node simulation over the exact live screenshot rows plus edge cases, and a JS syntax check. |
| **Surfacing fix: PR #11 sustained-rise path was a near no-op** | Follow-up to PR #11, reported as "the change doesn't appear to have worked." PR #11 added path 2 (sustained rise) to `isSurfaced()` to catch high-burden conditions climbing under the 1.2 surge bar, but set its floor at `SURFACE_SUSTAINED_RATIO = 1.15` — only a sliver below 1.2. Path 1 already admits everything `>= 1.2`, so path 2's *effective* reach was just the razor-thin `[1.15, 1.20)` ratio window; that band is rarely populated in the live feed, so the surfaced-condition count barely moved and any high-burden condition rising 10-15% was still dropped. It behaved exactly as coded, but the code didn't do what the goal needed. Fix: lowered `SURFACE_SUSTAINED_RATIO` to `1.10`, giving the absolute-burden path a real 10-point band `[1.10, 1.20)` to act in while the floor still rejects flat endemic giants (chlamydia +1% → ratio ≈ 1.01, excluded; gonorrhea +8% → 1.08, excluded). Also added a surfacing diagnostic so this class of "it didn't visibly change anything" is debuggable *in the browser* (the only place the live CDC feed is reachable — the session env is egress-blocked, 403, so no prior session could ever validate thresholds against real data, the root reason repeated blind threshold nudges "didn't work"): `processCDC()` now returns `evaluated` + `surfacedCount`, rendered as the `.cdc-note` line under the US section header. Verified with an expanded Node simulation (the previously-dropped +12%/67-excess case now surfaces; chlamydia, gonorrhea +8%, small-base blips, declining, and below-floor all still excluded; 11/11 as expected) plus a JS syntax check. `data.cdc.gov` remains egress-blocked here (403 CONNECT), so no live curl was possible; the new diagnostic line is what confirms the effect on the live site. |
| **Featured-outbreak source migration: INRB-UMIE → Global.health line list** | The Ebola featured slot pointed entirely at `INRB-UMIE/Ebola_DRC_2026` (INSP SitRep), which is **DRC-only by construction** — DRC national totals and DRC health-zone cumulative CSVs. Uganda and international imports could never appear regardless of feed freshness. Replaced it with the Global.health **Ebola BVD 2026 line list (PUBLIC VIEW)**, one CORS-open CSV covering all affected countries. Dropped INRB entirely: config (four cumulative URLs + `aliasUrl` + value-field names) and the cumulative-series/`aliases.csv` aggregation logic both removed. New shape is a **record-per-case line list**, so `fetchOutbreak` now counts and groups records instead of reading cumulative columns: **confirmed cases only** in headline totals (suspected/probable/discarded excluded — suspected alone is ~900/3,200 rows and would distort the total); **country totals grouped dynamically** from `Location_Admin0` (no hardcoded country list, so the lone France import surfaces on its own); **health-zone drill-down preserved** as a collapsible breakdown of the largest zone-bearing country (DRC); plus a "recently confirmed (14d)" activity line from `Date_confirmation`. Designed around two verified traps: (1) **attribution follows case location, not nationality/travel history** — ID 542 (American national, travel history to Germany) is correctly a DRC case, handled by grouping strictly on `Location_Admin0`; (2) **not clean RFC 4180** — CRLF, quoted embedded commas, a blank trailing unnamed column, and a malformed `Date_last_modified` (`20260715`) that, left raw, would beat `2026-07-16` in a lexical max (`'0' > '-'`) then parse to Invalid Date and break the freshness banner. `parseCSV` rewritten as a full character state machine; new `normISODate()` normalizes/validates dates before any max. Freshness now driven off the file's own most-recent `Date_last_modified`, not an assumption of perpetual updates. Updated the summary tile ("all countries"), footer attribution (Global.health, CC BY 4.0), and this doc; noted CC BY 4.0 with attribution since Global.health is an aggregator, not a health authority. **Unlike CDC, the Global.health raw CSV IS reachable here** — verified against the live file with a Node aggregation simulation (2,094 confirmed; DRC 2,073 / Uganda 20 / France 1; 798 deaths, CFR 38.1%; 46 DRC zones; latest 2026-07-16, malformed value correctly not poisoning the max; 572 confirmed in 14d) and a headless Chromium render of `index.html` (country table + zone drill-down + LIVE banner, zero JS errors) plus a JS syntax check. |
