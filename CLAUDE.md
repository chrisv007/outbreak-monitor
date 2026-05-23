# CLAUDE.md — Outbreak Monitor Persistent Context

## Project Overview

**Outbreak Monitor** is a single-page dashboard that surfaces infectious-disease surveillance data from three public sources in near-real time. A visitor opens `index.html` (served as a GitHub Pages site) and sees:

- **US NNDSS panel** — weekly case counts for nationally notifiable conditions, pulled from CDC's Socrata API.
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
- Aggregate rows (e.g. `TOTAL`, `U.S. TOTAL`, and US Census regional labels like `NEW ENGLAND`, `MIDDLE ATLANTIC`, `EAST NORTH CENTRAL`, `WEST NORTH CENTRAL`, `SOUTH ATLANTIC`, `EAST SOUTH CENTRAL`, `WEST SOUTH CENTRAL`, `MOUNTAIN`, `PACIFIC`) appear in the dataset alongside state-level rows and will produce double-counting if included. Filter them out client-side.

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

### 2. Global.health Linelists

**Base URL:** `https://api.global.health/` (confirm exact endpoint in current `index.html`)

**Quirks:** Field names and available filters vary by outbreak dataset. Verify schema against a live response before changing query parameters.

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
