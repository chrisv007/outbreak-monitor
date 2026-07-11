# Outbreak Monitor

A real-time disease outbreak surveillance dashboard that aggregates data from three public health sources into a single, self-contained web page. It displays US weekly notifiable-disease counts from CDC NNDSS, a detailed breakdown of one current priority outbreak (with a live data-freshness indicator), and the 20 most recent Disease Outbreak News alerts from WHO — all rendered client-side with no backend or build step required.

**Live site:** [https://chrisv007.github.io/outbreak-monitor/](https://chrisv007.github.io/outbreak-monitor/)

---

## Data Sources

| Source | Attribution | What it provides |
|---|---|---|
| **CDC NNDSS** | Centers for Disease Control and Prevention, National Notifiable Diseases Surveillance System | Weekly case counts by disease and state, served via the Socrata JSON API. National and regional rollup rows (e.g. `U.S. Residents`, `U.S. Total`, census-division groups) are filtered out client-side so headline totals are not double-counted against the per-state rows. Minnesota is pinned and highlighted in every state breakdown, with a dedicated Minnesota year-to-date tile in the summary bar. |
| **Global Priority Outbreaks** | Source varies by featured outbreak (currently INRB-UMIE / INSP SitRep pipeline for Ebola BDBV 2026) | Detailed case/death breakdown of one current high-priority outbreak, with a live/stale data-freshness banner that signals when the feed needs replacing |
| **WHO DON** | World Health Organization, Disease Outbreak News | Official outbreak alerts and situation reports |

---

## Architecture

The entire application is a single `index.html` file. It uses vanilla JavaScript to fetch data from the three sources above at page load, then builds and inserts DOM elements directly. There is no framework, no bundler, no npm install, and no server component. Deploying the dashboard means serving the file via GitHub Pages or any static host.

---

## Contributing

Before making any changes, read **[CLAUDE.md](./CLAUDE.md)**. It contains the persistent project context that Claude Code loads at the start of every session, including confirmed API field names, critical query patterns for each data source, and lessons learned from past bugs. Any pull request that touches an external API endpoint should include evidence (a live curl response) that the new query returns real data.
