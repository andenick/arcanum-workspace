# Foodberg — Historical Food Price Explorer

**Status: 🟢 LIVE at [foodberg.org](https://foodberg.org)** · State verified: 2026-07-25

**A full-stack web application for exploring historical food commodity prices, built with React and FastAPI. Over 4 million records — 6 commodity-price datasets plus economic indicators and derived composite indices — from public US government and international sources, covering 85 browsable commodities.**

> **Project state:** Two tracks. (1) **The web app — LIVE at [foodberg.org](https://foodberg.org)**, self-hosted on an HP EliteDesk 800 G5 (Docker + Caddy + Cloudflare Tunnel); the `foodberg.db` SQLite database is **~2.94 GB** (see Data Sources below). Authoritative deployment record: `DEPLOYMENT_TRUTH.md`. (2) The KB wishlist track — v4 wishlist current (1,985 entries / 105 categories, `2026.06.20 KB Wishlist v4 Global`); ~802 source PDFs acquired into `Inputs` (2026-05-10) but **not yet extracted**. No HDARP campaign has run — no `Knowledge_Base`, no `BATCH_STATE.json`. See `PROGRESS_LOG.md`.

---

## Why This Exists

Understanding food prices requires combining data from scattered government sources (USDA, BLS, FAO, World Bank, FRED) into a single queryable interface. Foodberg harmonizes these into a SQLite database with a React frontend for interactive exploration — designed for researchers, journalists, and historically minded chefs who want to see the data behind the food system.

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

- **Food Price Index**: Composite indices for 6 food groups (meat, dairy, cereals, oils, sugar, produce) from FAO and BLS data, 1990–present
- **Price Explorer**: Browse 85 agricultural commodities from USDA WASDE, AMS, and NASS data
- **Geographic Comparison**: Compare prices across US states and regions
- **Historical Trends**: Multi-commodity comparison with correlation analysis
- **Live Terminal Prices**: USDA Market News API integration (requires `USDA_API_KEY`)

---

## Data Sources

The live `foodberg.db` is **~2.94 GB**. Row counts by dataset:

| Dataset (`table`) | Records | Source |
|-------------------|--------:|--------|
| `ams_wholesale_prices` | 1,671,751 | USDA AMS Market News (terminal/wholesale prices) |
| `wasde` | 1,459,734 | USDA World Agricultural Supply & Demand Estimates |
| `nass` | ~1,060,000 | USDA National Agricultural Statistics Service |
| `faostat` | ~167,000 | FAO FAOSTAT (global food & agriculture statistics) |
| World Bank Pink Sheet | ~49,000 | World Bank global commodity prices |
| `retail` | 22,398 | Retail food prices |
| `economic_indicators` | 16,246 | CPI / PPI / macro indicators |
| `composite_indices` | 3,146 | Derived food-price indices |

---

## Repository Structure

```
Foodberg/
├── README.md
├── backend/                FastAPI server (24 GET endpoints)
│   ├── main.py             API endpoints
│   ├── database/           SQLAlchemy models, importers
│   ├── indices/            Composite index computation
│   ├── data/               SQLite database (foodberg.db)
│   └── data_sources/       API clients (FRED, FAO, World Bank, USDA)
├── frontend/               React app
│   └── src/pages/          7 pages (Home, Index, Explorer, Detail, Geographic, Trends, Sources)
├── Inputs                 Raw data (gitignored — re-downloadable from public APIs)
├── Technical              Processing scripts, docs, deployment configs
└── Outputs                Deliverables
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 19, TypeScript, Vite, Tailwind CSS, Recharts |
| Backend | FastAPI, Python, SQLAlchemy |
| Database | SQLite (`foodberg.db`, ~2.94 GB) |
| Deployment | Self-hosted on an HP EliteDesk 800 G5 — Docker + Caddy + Cloudflare Tunnel; live at [foodberg.org](https://foodberg.org) |

---

## Requirements

- **Python 3.11+** — backend
- **Node.js 18+** — frontend
- **API keys** (optional) — `USDA_API_KEY` for live terminal prices, `FRED_API_KEY` for FRED data refresh

---

## Citation

```bibtex
@software{foodberg2026,
  title = {Foodberg: Historical Food Price Explorer},
  author = {Anderson, Nicholas},
  year = {2026},
  url = {https://github.com/andenick/Foodberg}
}
```

---

## License

MIT
