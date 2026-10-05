# Foodberg — Historical Food Price Explorer

**Status: 🟢 LIVE at [foodberg.org](https://foodberg.org)**
> **Hosting status (2026-10-04):** the self-hosted origin box behind this site has been offline
> since late September 2026 (boot failure; physical repair in progress). The site returns when the
> box is back. The repository and data packages below remain fully available. · State verified against the repository README: 2026-10-02

**A full-stack web application for exploring historical food commodity prices, built with React and FastAPI. A multi-source SQLite build covering USDA NASS history, USDA PSD, World Bank Pink Sheet, FAO producer prices, and BLS retail data — with honest coverage badges rather than fabricated trend lines.**

> **Project state:** Two tracks. (1) **The web app — LIVE at [foodberg.org](https://foodberg.org)**, deployed on a self-hosted Docker box behind a Cloudflare Tunnel (FastAPI + Caddy → SPA). Data is acquired offline by the maintainers' collectors, rebaked through `rebake_history.py`, and baked into the production Docker image — no runtime API calls in production. (2) The document track — **895 Hopper-read documents** landed in `Knowledge_Base/`, method-tagged, catalogued, and packaged on 2026-07-16; the canonical integration pass and downstream database build are still pending per the repository README.

---

## Why This Exists

Understanding food prices requires combining data from scattered government sources (USDA, BLS, FAO, World Bank) into a single queryable interface. Foodberg harmonizes these into a SQLite database with a React frontend for interactive exploration — designed for researchers, journalists, and historically minded chefs who want to see the data behind the food system.

---

## Quick Start

```bash
git clone https://github.com/andenick/Foodberg.git
cd Foodberg

# Backend
cd backend
python -m venv venv
venv\Scripts\activate            # Windows (or: source venv/bin/activate)
pip install -r requirements.txt
python -m database.import_all    # Populate database from public APIs
python main.py                   # Starts on http://localhost:8000

# Frontend (new terminal)
cd ../frontend
npm install
npm run dev                      # Starts on http://localhost:3000
```

---

## Features

- **Price Explorer**: Multi-source price browsing with source-picker tabs (NASS farm gate, global spot/Pink Sheet, BLS retail); coverage badges and stat cards for thin series; CSV download per chart
- **Geographic Comparison**: Three-mode toggle — FAOSTAT producer prices by country, US state NASS prices, World Bank development indicators
- **Historical Trends**: Multi-commodity line charts, eligibility restricted to commodities with real multi-year history
- **Food Price Index**: Composite indices for 6 food groups (meat, dairy, cereals, oils, sugar, produce) from FAO and BLS data
- **Data Sources**: Database overview with per-source record counts and status cards

---

## Data Sources

The production `foodberg.db` is a multi-million-row SQLite build whose exact counts vary by rebake. Per-source records as documented in the repository README:

| Source | Records | Coverage |
|----------------------|----------|----------|
| USDA NASS (history) | ~1.06M | 44 commodities, national + state prices, 1908–2026 |
| FAO FAOSTAT | ~167K | Producer prices by country + country food CPIs |
| World Bank Pink Sheet | ~49K | CMO monthly commodity prices |
| BLS AP (retail) | ~20K | Average price series, monthly 1980–2026 |
| World Bank WDI | ~3K | Development indicators |
| USDA PSD | 192 MB | Supply & demand quantities (acquired, not surfaced in UI) |
| Composite Indices | ~3K | Computed from FAO + BLS data |

---

## Repository Structure

```
Foodberg/
├── README.md
├── backend/                FastAPI server (project-local, lags deploy tree)
│   ├── main.py             API endpoints
│   ├── database/           SQLAlchemy models, importers
│   ├── indices/            Composite index computation
│   ├── data/               SQLite database (foodberg.db)
│   └── data_sources/       API clients (FRED, FAO, World Bank, USDA)
├── frontend/               React app (canonical source for frontend)
│   └── src/pages/          9 pages (Home, PriceIndex, Explorer, Detail, Geographic, Trends, Sources, Downloads, SupplyDemand)
├── Inputs/                 Raw data & acquired PDFs (gitignored)
├── Technical/              Processing scripts, docs, progress log
└── Outputs/                Deliverables (wishlists, PSD exports)
```

> **Deploy tree:** the canonical production backend is the maintainer's private deploy tree — it includes `main.py`, `worldbank_client.py`, `rebake_history.py`, and the baked `data/foodberg.db` (gitignored, in the Docker image only).

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 19, TypeScript, Vite, Tailwind CSS, Recharts |
| Backend | FastAPI, Python, SQLAlchemy |
| Database | SQLite, multi-million-row production build (exact counts vary by rebake) |
| Deployment | Docker + Caddy + Cloudflare Tunnel (self-hosted box); live at [foodberg.org](https://foodberg.org) |
| Data Pipeline | offline collectors → rebake_history.py → baked image |

---

## Requirements

- **Python 3.11+** — backend
- **Node.js 18+** — frontend
- **API keys** (optional) — `USDA_NASS_API_KEY` for the NASS collector, `FRED_API_KEY` for FRED data refresh

---

## Anu replication package

The ``anu/`` directory contains a complete data-replication package: `series_registry.json` (the canonical data contract), fetch/process/validate scripts, and Data Provenance Records. See ``anu/README.md`` to reproduce the data.

---

## Citation

```bibtex
@software{foodberg2026,
  title = {Foodberg: Historical Food Price Explorer},
  author = {Anderson, Nicholas},
  year = {2026},
  url = {https://foodberg.org}
}
```

---

## License

MIT
