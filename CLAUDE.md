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
areas still excluded), then surfaces one when it is running clearly above the same point last
year:

- Group every row by `cleanLabel(label)` (folds footnote markers like `†`/`*` so state rows
  for one condition don't split into duplicate cards).
- **Baseline: last year's own table at the same MMWR week, not `m4`.** `fetchCDC()` pulls a
  second, trimmed table (`year = Y-1`, same `week`, `$select=states,label,m3`) and
  `processCDC()` uses its summed `m3` as `ytdPrevious` for every comparison. Reason, measured on
  the live feed: NNDSS counts keep growing for about a year as late reports arrive, so `m4` (last
  year, fully revised) is about **20% higher** than the same period looked when it was first
  published (median backfill 1.20 for 2025 data, 1.26 for 2024, IQR roughly 1.13-1.35).
  Comparing this year's still-incomplete `m3` against that revised `m4` pushed every condition
  about 18% toward "declining". It is not a small effect: in October 2025 it showed West Nile as
  *down* 7% in a season that finished *up* 54%.
  - Cross-year matching uses `labelKey()` (lowercase letters and digits only) because labels
    drift in spacing between annual tables: 2025 has `SalmonellaParatyphi infection`, 2026 has
    `Salmonella Paratyphi infection`.
  - Fallbacks, per condition, to the revised `m4` (`baselineSource = 'revised'`): the label is
    absent from last year's table (a new reporting category), or last year's early figure is
    **0 while the revised one is above 0**. That second case is a reporting gap, not a real
    zero: without it a 1-case anthrax year read as "0 then, 1 now", counted as emergent, and was
    pinned above a 33,000-case outbreak. If last year's whole table fails or comes back empty
    (e.g. a week 53 with no counterpart), every condition falls back and the `.cdc-note` says so
    in words, since that panel understates rises.
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
    a 2-case novel-influenza or anthrax cluster cannot pass the chance-variation test, and even if
    it did it would rank last by excess and fall off the cap. Missing a small high-consequence signal is the
    worst failure mode for an early-warning monitor, and no amount of loosening the ratio/excess
    knobs fixes it (that only pulls in endemic noise). **This is NOT a return of the banned
    `DISEASE_KEYWORDS` allowlist.** That allowlist was the *only* path to surface, so any unlisted
    condition (a new outbreak) was permanently invisible. This tier is purely **additive**: it can
    only *raise* a dangerous pathogen's visibility, never suppress anything — every condition is
    still independently scored by the data-driven signal below, so new/unlisted outbreaks still
    appear automatically.
- Otherwise (not high-consequence), `isSurfaced(d)` keeps a condition when **both**:
  1. *Beyond chance:* `riseZ(d) = (now - last - 1) / sqrt(now + last) >= SURFACE_MIN_Z`
     (1.645, one-sided 95%). This is the normal approximation to the exact conditional test for
     two Poisson counts; checked against the exact binomial test over 204,000 count pairs, it
     disagreed on 4, all sitting at p = 0.05. It replaces both the old fixed 5-case floor and the
     old "sustained rise" path: a 9 → 14 jump is +56% but well inside chance and stays out, a
     condition with no cases last year needs 5 (the same floor as before), and large conditions
     clear it easily.
  2. *Big enough to matter:* emergent (`ytdPrevious == 0`), or `ytdCurrent / ytdPrevious >=
     SURFACE_SURGE_RATIO` (1.2, at least 20% above last year at the same week). This is what
     keeps the high-volume endemic giants out: chlamydia at +2% is statistically "significant"
     at a million cases but nowhere near the 20% bar.
- **Evidence for this rule (backtest, re-run it before changing it).** Standing at weeks 12, 25
  and 41 of 2024 and 2025, each rule was scored against how the year actually turned out once
  both years had a full year of late reports (truth = matured ratio >= 1.10 and beyond chance):

  | Rule | Precision | Recall |
  |---|---|---|
  | Old: `m4` baseline, 1.2 surge or 1.10 + 50 excess, 5-case floor | 85% | 51% |
  | **Current: as-published baseline, 1.2 and z >= 1.645** | **85%** | **71%** |
  | As-published baseline, 1.10 and z >= 1.645 | 77% | 77% |
  | Current plus a guard that `now >= 0.9 x m4` | 91% | 67% |

  Same precision as before, a fifth more of the real rises caught. With a fair baseline the
  `[1.10, 1.20)` band is mostly reporting-timing noise (6 real rises against 11 false alarms), so
  **do not re-add a 1.10 "sustained rise" path**: it existed to compensate for the biased `m4`
  baseline, which made "+10%" really mean about +30%. The `0.9 x m4` guard was also rejected: it
  removed 7 false alarms but also 6 real rises, including *Candida auris* and CPO, the
  antimicrobial-resistance signals the panel most needs. Known residual weakness: when last
  year's early figure was depressed by a one-off reporting backlog (e.g. *C. auris* clinical in
  2024), this year looks inflated against it. No method here fixes that without losing more.
- **Rejected: a "last 8 weeks" window.** Differencing published cumulative counts
  (`m3(week N) - m3(week N-8)`) to get a more current signal was tried against the live feed and
  is unusable: states upload late cases in batches, so 8-week differences swing wildly
  (chlamydia +27%, gonorrhea +42%) and some go negative (chronic hepatitis B, chikungunya) when
  cases are reclassified. The year-to-date window lags, but it is the stable one. The card's
  "This week" figure is where current activity shows.
- Rank by `outbreakExcess(d)` = `ytdCurrent - ytdPrevious` (excess cases over last year at this
  week, an interpretable "how far above normal" measure that balances ratio and volume), cap at
  `SURFACE_MAX_CARDS` (24) so the panel stays focused.
- Each card prints **one sentence saying why it is on the panel** (`.disease-reason`), e.g.
  "29,929 more cases than at this week last year, beyond normal year-to-year variation", or for
  the severity tier "High-consequence pathogen, 153 new cases reported this week". The third stat
  cell is labelled `<prev year> same wk`, and the chip reads `+N% vs <prev year>`.
- `m3` sums correctly across states; the 52-week max (`m2`) does not, so it is not used for the
  surfacing decision. The card footer shows it as **"Peak state-week"** (`maxStateWeek`, the
  busiest single state-week in the past 52 weeks). It was previously labelled "52-wk max", which
  read as a national figure it never was. Emergent conditions have no prior-year baseline, so
  their card shows a "🆕 New this year" badge and a `NEW` chip instead of a misleading
  `N/A · Stable`.

The thresholds (`SURFACE_SURGE_RATIO`, `SURFACE_MIN_Z`, `SURFACE_MAX_CARDS`,
`HIGH_CONSEQUENCE_MIN_YTD`) are the tuning knobs for the *trend* signal; adjust those rather than
adding disease names to the trend logic, and backtest any change against matured outcomes first
(the method above needs only four extra `x9gk-5huc` tables per snapshot). The one list you *do*
curate by name is `HIGH_CONSEQUENCE_PATTERNS`, and only to widen/narrow the additive severity
floor, never to gate what the trend signal can surface.

**Surfacing diagnostic (tune from real data, not guesses):** `data.cdc.gov` **is reachable
from a session with full network access.** An earlier version of this file claimed a hard egress
block on it; that was a property of one restricted environment, not of the host, and it is not
true in general. Run a `curl` to find out which kind of environment you are in rather than
assuming either way, and where CDC is reachable, tune these thresholds against a live query.
The on-page diagnostic stays useful regardless, because it reports what the viewer's browser
actually received rather than what a session saw. `processCDC()` returns `evaluated` (distinct
conditions seen, aggregates excluded) and `surfacedCount` (how many passed `isSurfaced` before
the card cap), plus `highConsequenceCount` (how many surfaced via the severity tier), and
`renderCDC()` prints them into the `.cdc-note` line under the US section header ("Evaluated N
notifiable conditions · M surfaced · P high-consequence flagged · showing top K", followed by
the rule in one sentence and which baseline was used). `baselineAsPublished` says whether last
year's same-week table loaded; when it did not, the note says rises look about 20% smaller than
they are. This turns "why is the panel almost empty?" from a guess into something readable off
the live page: if `M` is small the data is genuinely quiet, if `M` is large but `K` is capped
raise `SURFACE_MAX_CARDS`. Do not tune the knobs blind: read this line on the live site and
backtest first.

**Minnesota focus:** This dashboard gives Minnesota special prominence. In every disease
card's state breakdown, the Minnesota row is pinned to the top and highlighted (`.mn-row`),
shown even when MN is outside the top 20 states or has no reported row (rendered as 0). The
summary bar also carries a "Minnesota YTD" tile rolling up MN year-to-date cases across all
currently flagged (surfaced) diseases. The `isMinnesota()` helper matches the reporting area
case-insensitively as either `MINNESOTA` or `MN`, since the live field could use either.

**Critical two-step query pattern:**

Step 1: find the latest available week for the current year:
```
GET https://data.cdc.gov/resource/x9gk-5huc.json?$select=max(week)%20as%20latest_week&$where=year='YYYY'
```

Step 2: fetch all rows for that week (only the columns used):
```
GET https://data.cdc.gov/resource/x9gk-5huc.json?$select=states,label,m1,m2,m3,m4&$where=year='YYYY'%20AND%20week=<N>&$limit=50000
```

Step 3: in parallel, last year's table for the same week, the fair baseline (see above):
```
GET https://data.cdc.gov/resource/x9gk-5huc.json?$select=states,label,m3&$where=year='<YYYY-1>'%20AND%20week=<N>&$limit=50000
```

Trimming with `$select` keeps the two tables together (about 108 KB gzipped) slightly under the
single untrimmed table they replaced (about 117 KB). All three queries are fetched with
`cache: 'no-cache'` for the same heuristic-caching reason as the respiratory feeds. The dataset
holds 2022 onward, so the baseline week is always available except a week 53 with no
counterpart.

Never assume a fixed week number. The dataset lags publication by one to two weeks, so the max-week query is required every session.

---

### 2. Respiratory Virus Surveillance (COVID-19, influenza, RSV)

**There are no US COVID-19 case counts to source, and there will not be.** Aggregate case
reporting ended in May 2023 and COVID-19 was removed from the nationally notifiable condition
list in July 2024, so the NNDSS feed in Layer 1 does not and will not carry it. Do not go
looking for a case-count endpoint, and do not add a tile or heading anywhere in this section
that implies case volume. The section carries the surveillance streams that replaced case
counting, on three deliberately different axes:

| Axis | Dataset | What it is |
|---|---|---|
| **Burden** (observed) | `rdmq-nq56` NSSP ED visit trajectories | Share of all emergency department visits attributable to each pathogen, weekly, by state and sub-state region, with the feed's own trend direction |
| **Severity** (observed) | `ua7e-t2fy` NHSN weekly hospital respiratory data | New confirmed admissions per 100k, percent of inpatient beds occupied, and week-over-week change, weekly, by jurisdiction |
| **Direction** (modelled) | `5dqz-y4ea` CDC Epidemic Trends and Rt | Modelled Rt with uncertainty, trend category and probability of growth, nowcast so it leads the observed series |

`rdmq-nq56` and `5dqz-y4ea` are both derived from the **same upstream NSSP emergency-department
data**, so they are **not independent confirmations of each other**. The page states this
explicitly. Read them as burden versus direction, never as two sources agreeing. NHSN is a
genuinely separate collection (hospitals reporting their own admissions), so it is the one real
cross-check in the section.

**Deliberately not used:** `2ew6-ywp6` NWSS wastewater. Two reasons, both checked against the
live API rather than assumed. It is SARS-CoV-2 only, so it cannot carry the three-pathogen
frame. More decisively, **it is stale**: `max(date_end)` is `2025-09-07`, roughly a year behind.
Descriptions of this dataset as "updated Fridays" no longer match what the ID actually serves.
Re-check `max(date_end)` before ever reintroducing it.

#### Confirmed live JSON field names

All three schemas below were read off live `data.cdc.gov` responses, not inferred from dataset
descriptions or CSV headers.

**`rdmq-nq56` (NSSP ED visits).** Layout is **wide**: one column per pathogen, not one row per
pathogen.

| Field | Meaning |
|---|---|
| `week_end` | Week ending date, `calendar_date` (`"2026-08-22T00:00:00.000"`) |
| `geography` | Full state name, or `United States` on the national row |
| `county` | `All` on state and national rows; a county name on sub-state rows |
| `trend_source` | `State` / `HSA` / `United States`, the row-granularity discriminator |
| `percent_visits_covid`, `percent_visits_influenza`, `percent_visits_rsv` | Percent of all ED visits |
| `percent_visits_combined` | All three combined |
| `percent_visits_smoothed_covid`, `percent_visits_smoothed_1`, `percent_visits_smoothed_rsv` | Smoothed variants. **Note `_1` is influenza**, a Socrata name collision, not a typo. `percent_visits_smoothed` (unsuffixed) is *combined*. |
| `ed_trends_covid`, `ed_trends_influenza`, `ed_trends_rsv` | Trend text |
| `hsa`, `hsa_counties`, `hsa_nci_id`, `fips`, `buildnumber` | Geography and build metadata |

`ed_trends_*` vocabulary, confirmed by grouping the whole table: `Increasing`, `Decreasing`,
`No Change`, `Data Unavailable`, `Limited Data`, `Sparse`, `No Data`. Only the first three are
directions. The rest are reporting states and must not be rendered as if they were trends.

**`ua7e-t2fy` (NHSN hospital).** Very wide (300+ columns). Only these are used:

| Field | Meaning |
|---|---|
| `weekendingdate` | Week ending date, `calendar_date` |
| `jurisdiction` | **Postal abbreviation** (`MN`), plus `USA` and `Region 1`..`Region 10` |
| `totalconfc19newadmper100k`, `totalconfflunewadmper100k`, `totalconfrsvnewadmper100k` | New confirmed admissions per 100k |
| `pctconfc19inptbeds`, `pctconffluinptbeds`, `pctconfrsvinptbeds` | Percent of inpatient beds occupied |
| `totalconfc19newadmpctchg`, `totalconfflunewadmpctchg`, `totalconfrsvnewadmpctchg` | Week-over-week percent change |

**`5dqz-y4ea` (Epidemic Trends and Rt).**

| Field | Meaning |
|---|---|
| `as_of` | Modelling-run vintage, `calendar_date`. New vintage weekly. |
| `date` | Estimate date. **Daily, not weekly.** One vintage carries a 29-day series ending on `as_of`. |
| `disease` | `COVID-19`, `Influenza`, `RSV` |
| `state` | **Full state name** (`Minnesota`), 51 jurisdictions, **no national row** |
| `median`, `lower`, `upper` | Rt point estimate and interval bounds |
| `interval_width` | `0.95`, or absent when not estimated |
| `p_growing` | Probability of epidemic growth, 0 to 1 |
| `category` | `Growing`, `Likely Growing`, `Not Changing`, `Likely Declining`, `Declining`, `Not Estimated` |

**Geography vocabularies differ and must be reconciled.** NSSP and Rt both use full state
names; NHSN uses postal abbreviations. `STATE_ABBR` in `index.html` is what joins them. Without
it the Minnesota row silently loses its admissions figures.

#### Query patterns

Same never-assume-a-fixed-week discipline as NNDSS.

```
# NSSP: latest published week, then that week's state + national rows
GET rdmq-nq56.json?$select=max(week_end) as latest
GET rdmq-nq56.json?$where=week_end='<latest>T00:00:00.000' AND trend_source in('State','United States')

# NHSN: same shape
GET ua7e-t2fy.json?$select=max(weekendingdate) as latest
GET ua7e-t2fy.json?$where=weekendingdate='<latest>T00:00:00.000'

# Rt: latest vintage, its newest estimate date, and the per-disease last real estimate
GET 5dqz-y4ea.json?$select=max(as_of) as latest
GET 5dqz-y4ea.json?$select=max(date) as latest_date&$where=as_of='<latest>T00:00:00.000'
GET 5dqz-y4ea.json?$select=disease, max(date) as last_estimated&$where=median IS NOT NULL&$group=disease
GET 5dqz-y4ea.json?$where=as_of='<latest>T00:00:00.000' AND date='<latest_date>T00:00:00.000'
```

`calendar_date` filters need the full `T00:00:00.000` suffix. The
`$where=median IS NOT NULL` grouped query is the seasonality signal, read from the data rather
than assumed. See below.

#### Quirks and design rules

- **Socrata omits null fields entirely from JSON rows.** A state with no reported percentage
  does not return `"percent_visits_covid": null`; the key is simply absent. Iowa is the live
  example: on 2026-08-22 its state row carries no `percent_visits_*` keys at all and
  `ed_trends_* = "Data Unavailable"`. **Absent is never zero.** These are percentages and rates,
  so `0%` is a claim that activity is nil, which is a different statement from "not reported".
  `num()` coerces blanks to 0 and must not be used in this section; use `numOrNull()`.
- **Sub-state rows must be excluded or every state double-counts.** Filter on
  `trend_source='State'`, not on `county`. The table holds 640k HSA rows against 10k state rows.
  Same class of bug as the NNDSS regional rollups.
- **NHSN carries its own rollups**: `USA` plus `Region 1` through `Region 10`. Drop the regions,
  use `USA` only as the explicit national figure. NHSN also covers five territories that NSSP
  and Rt do not; they are excluded from the state table so no row has one column filled and two
  permanently blank.
- **Minnesota is pinned and highlighted** in every state table (`.mn-row`, same convention as
  the NNDSS breakdown), shown even when MN has no published row, and then as an explicit
  "no data".
- **No respiratory tile was added to the summary bar.** Every tile there is a case count; a
  percentage or a modelled Rt sitting alongside them would read as one, which is exactly the
  confusion this section exists to prevent.

#### Weekly ED-visit trend chart (one per virus card)

The first version was a 26px sparkline with no axis, no values and a one-line caption ("MN
solid, US grey, scale 0 to X% shared by all three"). The user reported not being able to tell
what it showed. It was rebuilt so it reads without decoding:

- **Title in words** ("Weekly share of ER visits for Influenza"), a **legend** with line keys,
  a **y-axis** with three clean gridlines (`0%`, midpoint, max via `respNiceMax()`), and **x-axis
  dates** (first, middle, last week ending).
- **A plain-language summary under it** (`respChartSummary()`): this week's Minnesota and US
  share, versus the week before and versus the start of the window, plus CDC's own trend call.
  Directions are read off the two-decimal values shown, so the sentence never contradicts the
  numbers. The week-before comparison matters: COVID in MN fell 0.30% to 0.21% in the latest
  week while still being up on 12 weeks ago, and comparing only to the window start hid that.
- **Hover/tap crosshair + tooltip** listing both series for that week, the same readout on
  keyboard focus (arrow keys step through weeks), and a collapsed **"Weekly numbers" table**
  so no value is hover-only. Tooltip text goes in through `textContent`.
- **Shared y-scale across all three viruses is kept on purpose** (same honesty reason as before:
  a near-zero RSV series must look flat, not like a mountain). The note under each chart says so,
  and the summary sentence carries the relative change a flat line cannot show.
- **Colours:** Minnesota `#3A6E2E` (accent, the series the card is about), US `#968D81` (grey
  context line). Checked with the dataviz palette validator against the card surface `#FAF7F0`:
  both clear 3:1 contrast and the pair separates under colour-blind simulation (protan
  ΔE 15.2). The grey failing the chroma floor is intended for an emphasis chart (one hue + grey).
  The old US grey `#B9AF9D` was only 2.0:1 against the card.
- **Series are aligned on `week_end`**, not array position. The old code paired MN and US
  points by index, so a week missing for one geography would shift that whole line sideways
  onto the wrong dates. A missing week now breaks the line instead of being drawn as zero.
- The chart is **drawn after insertion at its real pixel width** (`drawRespChart()`), and
  redrawn on resize, so text and dots stay crisp rather than stretching with
  `preserveAspectRatio="none"`. Handlers are assigned (`el.onpointermove = ...`), not added,
  so redraws never stack listeners.
- The card-header chip is CDC's own trend call for **Minnesota ED visits** and now says so
  (`MN ▲ Increasing`, with a title tooltip). Unlabelled, a "No change" beside a falling line
  read as a verdict on the whole card.

#### Rt constraints (all load-bearing, do not regress them)

1. **Rt measures direction only, never burden.** An Rt below 1 means transmission is shrinking,
   *not* that activity is low. Rt is never presented alone: it always sits under the observed
   burden and severity blocks in the same card, and the card text says so in as many words.
2. **Seasonality is expressed in the data, not by rows disappearing. This was the key
   correction.** Flu and RSV rows **keep publishing all summer**. They simply carry
   `category = "Not Estimated"` with no `median`. Confirmed on the 2026-08-25 vintage: 1,479
   Influenza rows and 1,479 RSV rows, every one `Not Estimated`, while COVID-19 has 1,363
   `Growing`. So **"is this feed still publishing" and "is this pathogen currently estimated"
   are two independent questions**, and reading them separately is what keeps a normal summer
   from firing a stale-feed alarm. Do not implement the off-season as "rows are missing".
   - The last real flu and RSV estimate is **2026-05-26** (`max(date)` where `median IS NOT
     NULL`), not the 29 May date quoted in some briefs.
   - **The estimation window was read off the feed, not assumed.** Grouping non-null medians by
     month returns **September through May** in both the 2024-25 and 2025-26 seasons, with June,
     July and August empty. `RT_SEASON_MONTHS` encodes that.
   - `RT_CORE_SEASON_MONTHS` (Nov-Apr) is deliberately narrower. A seasonal pathogen going
     unestimated in deep winter is genuinely odd and gets flagged; September, October and May
     are shoulder months where a pause is unremarkable and would otherwise fire a false alarm at
     each season edge.
3. **"Not estimated" is not "no data".** Rt is withheld for a state when ED visit volume is too
   low, an anomaly is detected, or the model fails reliability checks. A row present with a null
   median renders as "not estimated"; a location absent from the feed renders as "no data".
   Iowa is the live example on the current vintage. Keep the two distinct in card and table.
4. **Methods break on 2026-06-01** (EpiNow2 replaced by a hierarchical GAM). The section stays
   strictly inside one `as_of` vintage, so no displayed series spans the seam. Because the last
   flu and RSV estimate (2026-05-26) predates it, the off-season card annotates that those
   values came from the previous method and are not comparable to current-method numbers.

#### Freshness

Three independent indicators, one per feed, because a seasonal pause in one is not a problem in
the others. Each compares the feed's own latest date against the viewer's clock over
`RESP_FRESHNESS_WINDOW_DAYS` (14).

`respRtStatus()` then resolves the Rt card into exactly one of five states, in this precedence:

| State | Condition | Rendering |
|---|---|---|
| `error` | Feed could not be read | Neutral note; burden blocks unaffected |
| `stale` | No new `as_of` vintage inside the window | Red. A feed fault, explicitly *not* a seasonal pause |
| `active` | Pathogen has real estimates in the current vintage | Rt value, interval, category, growth probability |
| `paused-seasonal` | Feed fresh, pathogen unestimated, seasonal pathogen outside core season | Calm amber "off-season, not currently estimated" |
| `paused-unexpected` | Feed fresh, pathogen unestimated, but in core season or year-round (COVID-19) | Red, flagged as not the usual pause |

Feed staleness outranks everything, including an active estimate. A year-round pathogen never
receives the calm seasonal treatment.

**Publication cadence (checked 2026-10-16):** the Rt `as_of` is a Monday and the vintage lands
on `data.cdc.gov` the following Friday around 16:00 UTC (the 2026-10-06 run appeared
2026-10-09). A healthy feed therefore reads 3 to 11 days old, which the 14-day window covers
with slack; a red Rt chip means a week was genuinely missed or the browser got an old answer.

**Observed 2026-10-26: the Rt dataset was empty.** `5dqz-y4ea` returned `count(*) = 0` and
`max(as_of)` returned `[{}]` after a republish stamped 2026-10-23 16:04 UTC (the column schema
was intact). The page correctly fell into the `error` state ("Modelled direction unavailable")
with the burden and severity blocks unaffected. This was a CDC-side outage, not a code fault;
check the row count before changing anything if the Rt chip is red.

**Two display traps fixed after a false "Stale. Run of Sep 28, 17 days ago" report:**
- `data.cdc.gov` sends `Last-Modified` but no `Cache-Control`, so browsers may heuristically
  cache query responses. The user's page received the Sep 29 vintage a week after the Oct 6 one
  was live. `socrataJSON()` now fetches with `cache: 'no-cache'`. Do not remove it.
- `new Date('2026-09-29')` is midnight UTC, which US time zones render as the previous day.
  `fmtDate()` reads bare `YYYY-MM-DD` values as local midnight. Every feed date shown on the
  page is a bare date, so this matters everywhere, not just in this section.

**Verification:** every field name and every query above returned live HTTP 200 rows from
`data.cdc.gov` before any code was written. Rendering and decision logic were then checked in
headless Chromium at 320, 390, 430 and 1200px against live payloads, plus forced scenarios:
stale Rt feed, stale ED feed, NHSN down, all three down, Minnesota absent from the Rt feed,
Minnesota present but not estimated, and a January clock with a fresh feed still reporting flu
unestimated. A 19-assertion unit sweep over `respRtStatus()` covers all twelve months and the
stale-versus-seasonal precedence. No console errors and no horizontal page overflow at any
width. The rebuilt trend chart was re-checked the same way (320, 390, 430, 1200px against the
live 2026-10-17 week): chart widths 258 to 368px, no card or page overflow, hover tooltip and
arrow-key readout both returning the right week's MN and US values, no console errors.

---

### 3. Global Priority Outbreaks (featured-outbreak slot)

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

**Verification note:** the Global.health raw CSV is CORS-open and reachable (confirmed HTTP
200). Every change to this
section was validated against the live file: a Node aggregation simulation (2,094 confirmed;
DRC 2,073 / Uganda 20 / France 1; 798 deaths; 46 DRC zones; latest 2026-07-16 with the
malformed value correctly *not* poisoning the max; 572 confirmed in 14d) plus a headless
Chromium render of `index.html` (country table + zone drill-down + LIVE banner, no JS errors).

---

### 4. WHO Disease Outbreak News (DON)

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

**This rule was once waived, and the waiver was a mistake worth remembering.** The respiratory
section was first built in a session whose egress policy blocked `data.cdc.gov`, so its column
names were never read off a live response. Rather than stop, that version compensated with
machinery: every field became a candidate list, the code probed each dataset at page load and
bound each logical field to whichever column happened to exist, and a diagnostic line printed
the resolved schema onto the page so someone with network access could read it back. It was a
careful design for a problem that should not have been accepted in the first place.

When the same datasets were finally curled, the guesses turned out to be wrong in ways the
probing could not have recovered from. The ED feed is wide rather than long, its influenza
smoothed column is named `percent_visits_smoothed_1`, the Rt feed is daily rather than weekly
and has no national row, and, most importantly, the seasonal pause is expressed as rows that
keep publishing with `category = "Not Estimated"` rather than as rows going missing, which is
the opposite of what the code was built to detect. No amount of runtime binding finds a
behaviour the design never anticipated.

The lesson is not "write better fallbacks". It is: **if the schema cannot be verified, the
honest move is to say so and stop, not to build an elaborate mechanism that makes an unverified
guess look rigorous.** A candidate-list probe is a reasonable tactic for absorbing future schema
drift on a feed you have already confirmed. It is not a substitute for confirming it once.
