# Westchester — County Data Platform

**A full-stack web application for Westchester County, NY open data — demographic analysis, sidewalk infrastructure planning, transit accessibility, and interactive mapping. React frontend + FastAPI backend.**

Repo: `andenick/westchester` (verified 2026-10-02).

---

## Why This Exists

Westchester County publishes open data across dozens of portals (county GIS, census, MTA, municipal budgets) but there's no unified platform for analysis. This project combines demographic, infrastructure, transit, and property data into interactive dashboards designed for municipal planners, researchers, and residents.

## Quick Start

```bash
git clone https://github.com/andenick/westchester.git
cd westchester

# Backend
cd Technical/src/backend
pip install -r requirements.txt
python main.py                     # Starts on http://localhost:8000

# Frontend (new terminal)
cd Technical/src/frontend
npm install
npm run dev                        # Starts on http://localhost:5173
```

## Features

- **Demographic Dashboard**: population, income, housing, and education data by municipality
- **Sidewalk Planning Dashboard**: infrastructure coverage analysis with interactive mapping
- **Transit Accessibility**: Metro-North station analysis and commute patterns
- **Property Tax Explorer**: assessment and tax-rate comparisons across municipalities
- **Interactive Maps**: Leaflet-based geospatial visualization with layer controls

## Data Sources

| Source | Content | Access |
|--------|---------|--------|
| Westchester County GIS | Tax parcels, sidewalks, municipal boundaries | [GIS Portal](https://gis.westchestergov.com/) |
| US Census / ACS | Population, income, housing, education (2020 + 5-year ACS) | [data.census.gov](https://data.census.gov/) |
| MTA / Metro-North | Station locations, ridership, schedules | [MTA Open Data](https://new.mta.info/open-data) |
| Westchester County Budget | Municipal budget documents | County website |
| USDA SNAP | Food access indicators | [USDA ERS](https://www.ers.usda.gov/) |

## Structure & Stack

`Inputs/` (raw shapefiles, CSVs) · `Technical/src/` — `backend/` (FastAPI server), `frontend/` (React + Vite + Tailwind dashboard pages), `data_importers/`, `data_pipeline/` · `Output/` (deliverables). Stack: React · TypeScript · Vite · Tailwind CSS · Leaflet · Recharts · FastAPI · Python · SQLite · Leaflet + GeoJSON maps · Netlify (frontend) + Render (backend) deployment.

**Requirements**: Python 3.11+ (backend/data), Node.js 18+ (frontend). **No API keys required to run** — all data is from public open-data portals; an optional free Socrata app token (`SOCRATA_APP_TOKEN`) raises the anonymous rate limit for `data.ny.gov` downloads (see `Technical/.env.template`).

## Citation & License

Suggested citation: Anderson, Nicholas. *Westchester County Data Platform* (2026). https://github.com/andenick/westchester — the repository README carries the full BibTeX entry. License: MIT.
