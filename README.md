# Outbreak Monitor

A real-time disease outbreak surveillance dashboard that aggregates data from three public health sources into a single, self-contained web page. It displays US weekly notifiable-disease counts from CDC NNDSS, individual-level outbreak records from Global.health linelists, and the 20 most recent Disease Outbreak News alerts from WHO — all rendered client-side with no backend or build step required.

**Live site:** [https://chrisv007.github.io/outbreak-monitor/](https://chrisv007.github.io/outbreak-monitor/)

---

## Data Sources

| Source | Attribution | What it provides |
|---|---|---|
| **CDC NNDSS** | Centers for Disease Control and Prevention, National Notifiable Diseases Surveillance System | Weekly case counts by disease and state, served via the Socrata JSON API |
| **Global.health** | Global.health initiative | Individual-level linelist records for selected outbreak events |
| **WHO DON** | World Health Organization, Disease Outbreak News | Official outbreak alerts and situation reports |

---

## Architecture

The entire application is a single `index.html` file. It uses vanilla JavaScript to fetch data from the three sources above at page load, then builds and inserts DOM elements directly. There is no framework, no bundler, no npm install, and no server component. Deploying the dashboard means serving the file via GitHub Pages or any static host.

---

## Contributing

Before making any changes, read **[CLAUDE.md](./CLAUDE.md)**. It contains the persistent project context that Claude Code loads at the start of every session, including confirmed API field names, critical query patterns for each data source, and lessons learned from past bugs. Any pull request that touches an external API endpoint should include evidence (a live curl response) that the new query returns real data.
