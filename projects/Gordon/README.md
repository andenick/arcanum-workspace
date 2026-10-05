# Gordon — Employment & Inflation Toolkit

**Gordon** is a macroeconomic employment-and-inflation intelligence toolkit. It aggregates
employment, inflation, and financial-market data from free public sources — FRED, BLS, BEA,
Census, FDIC, NBER, OECD, World Bank, JST, Maddison, Bank of England, ILO — into one unified
analytical dataset, and ships **two interactive dashboards** on top of it: a Plotly Dash app
(Python) and an R Shiny app. The dashboards and dataset publish under the name **StarCruiser**;
the project name is Gordon. Repo: [`andenick/gordon-web`](https://github.com/andenick/gordon-web),
version **v9.0** (verified 2026-10-04; the repository's naming note confirms Gordon is the
current project name).

**What's in the repository:**

- `Technical/` — per-source import and analysis scripts (FRED, BLS, BEA, Census CBP, JST,
  Maddison, Bank of England, WDI, OECD, NBER, FDIC), catalog builders, geographic clustering,
  shift-share, Beveridge-curve and inflation-decomposition analyses, plus a data dictionary
  with series definitions and provenance
- `Dashboard/` — the Plotly Dash app (`app.py`) and the R Shiny app (`app.R` + modules)
- `Technical/data_layer/` — the published data layer (~3 MB): five prepared chopped CSVs, a
  `series_registry.json` of **611 series across 32 categories and 16 analyses**, and per-series
  provenance notes — enough for the Dash app to run on a fresh clone
- `anu/` — a complete data-replication package (registry, fetch/process/validate scripts, and
  Data Provenance Records) that reproduces the published layer from the original public sources

The full pipeline registers **1,135 series across 32 families** (repository description); the
repository itself ships the compact derived layer above rather than the source corpus.

**Honest limits:** no source data is redistributed — everything upstream is public and
re-downloaded per `data/MANIFEST.md`, with only free API keys required (FRED, Census); the
frozen `PortableDashboard/` snapshot ships without its `Data/` tree.

**License:** MIT (code) + CC BY 4.0 (data & documentation); `CITATION.cff` carries citation
metadata.
