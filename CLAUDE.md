# CLAUDE.md — Outbreak Monitor Persistent Context

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
