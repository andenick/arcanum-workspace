# Graham — Securities Analysis & Political Economy

## Overview
Institutional political economy analysis of corporations, public markets, and the economy using SEC EDGAR filing data. Named for Benjamin Graham's pioneering work in systematic securities analysis — the project uses his analytical rigor but applies it to understanding the structure and dynamics of American capitalism, not stock selection.

The core asset is an XBRL fundamentals panel built from SEC EDGAR company facts: 14,734 companies, 119,919 firm-years, 80 financial variables, spanning 1983-2025. It is assembled entirely from public sources (see `data/MANIFEST.md`).

## Setup

1. **Install dependencies** (Python 3.10+):
   ```
   pip install -r requirements.txt
   ```
2. **Point the code at your data** by setting `DATA_ROOT` to the directory that holds
   the source data tree (see `data/MANIFEST.md` for the expected layout and public
   download links). All loaders read from `$DATA_ROOT`; the default is `./data`.
   ```
   export DATA_ROOT=/path/to/your/data      # Windows: set DATA_ROOT=C:\path\to\data
   ```
3. **Bring your own API keys** (both free) for the steps that fetch live series:
   - **FRED** — get a free key at <https://fred.stlouisfed.org/docs/api/api_key.html>, then set `FRED_API_KEY`.
   - **BEA** — get a free key at <https://apps.bea.gov/API/signup/>, then set `BEA_API_KEY`.

   Copy `.env.example` to `.env` and fill in your values, or export them in your shell.
4. **Run** any pipeline step, e.g.:
   ```
   python L01_load_xbrl.py
   ```
   Steps under `S01` build the core `graham_base.parquet` panel; later studies read it.

## Research Questions
- How do profit rates vary across sectors and over time? Is there equalization?
- What do concentration patterns reveal about market structure?
- How have capital accumulation, leverage, and financialization evolved?
- What is the relationship between firm size, profitability, and survival?
- How do cash flow patterns differ between productive and financial sectors?
- What can balance sheet dynamics tell us about business cycle mechanics?

## Data
- **Primary**: XBRL fundamentals export (~175 MB financial_metrics.csv, ~16,548 companies)
- **Balance sheet**: companyfacts.zip (2M+ facts from 19,132 SEC EDGAR filers, 2008+)
- **Prices**: yfinance annual close prices (50,725 price-years, 5,540 tickers, 2014-2025)
- **Sector**: 3-source classification (sector_groupings + industry_groupings + S&P 500 GICS), 92.1% coverage

## Structure
- **Inputs/** — Source materials. Read-only originals.
- **Technical/** — Processing, analysis, working files.
  - `AnuData/` — Pipeline scripts and project registry
  - `DATA_SOURCES.md` — SEC/macro data layout under `DATA_ROOT`
- **Outputs/** — Finished deliverables, reports, datasets.

## AnuData Architecture

| Study | Name | Analyses | Status |
|-------|------|----------|--------|
| S01 | Data Infrastructure | L01 panel loader, L01b prices, L01c FRED, V01 validation | COMPLETE |
| S02 | Profitability & Distribution | A01-A07 (profit rates, markups, taxes, surplus, fin vs non-fin) | COMPLETE |
| S03 | Capital & Investment | B01-B06 (investment drought, buybacks, cash, leverage, R&D) | COMPLETE |
| S04 | Structure & Dynamics | C01-C07 (concentration, zombies, financialization, size, turbulence) | COMPLETE |
| S05 | Deep Dives | D01-D20 (dispersion decomposition, zombie profiling, TCJA, fragility, accumulation regimes) | COMPLETE |
| S06 | Visualization | F01-F22 (22 publication-quality figures, PNG + PDF) | COMPLETE |
| S08 | Advanced Analytics | E01-E06 (DuPont, cash conversion, Altman Z, survival, transitions, regressions) | COMPLETE |
| S10 | Cash Flow & Cost of Capital | F01-F06 (FCF quality, capex anatomy, debt structure, WACC, credit spreads, MEC schedule) | COMPLETE |
| S11 | Synthesis | G01-G04 (hollow corporations, zombie survival paradox, Shaikh real competition, inequality decomposition) | COMPLETE |
| S12 | Structural & Institutional | H01-H04 (corporate genealogy, geography, SIC migration, auditor) | COMPLETE |
| S13 | Labor & Distribution | H06-H07 (labor share, equity compensation) | COMPLETE |
| S14 | Financial Fragility | H08-H10 (Minsky FII, rate stress test, sectoral contagion) | COMPLETE |
| S15 | Investment & Accumulation | H11-H14 (total investment, goodwill, self-financing, lease leverage) | COMPLETE |
| S16 | Market Structure | H15-H17 (monopsony, competition persistence, markup cyclicality) | COMPLETE |
| S17 | Data Acquisition | FFIEC banking + BIS/OECD international benchmarks | COMPLETE |

## Pipeline
```
L01 (load XBRL + balance sheet + prices + sectors) → analysis scripts → outputs
```

## Key Findings (first pass, 2026-05-10)

| Finding | Value | Implication |
|---------|-------|-------------|
| Profit rate dispersion widening | CV 0.58 to 1.19 (2000s to 2020s) | Challenges Shaikh equalization at firm level |
| Within-sector dominance | 96% of profit rate variance is within-sector | Firm heterogeneity, not sectoral divergence |
| TCJA tax cut | ETR 25.4% to 17.8% (-30%) | Largest structural break in panel |
| Buyback/capex ratio | 13% to 73% (2005 to 2020s) | Buybacks dominate corporate cash use |
| Investment drought | Replacement ratio 1.35 to 1.01 | Net investment near zero |
| Zombie firms | 53.5% can't cover interest (2020s) | 57% are chronic (>50% of years) |
| Financial Services CR4 | 0.65 to 0.94 | Extreme concentration |
| Net firm exit | -919/yr (2020s) | More firms leaving than entering |
| Firm size Gini | 0.94 | Extreme inequality among firms |
| Retained earnings erosion | 61.4% negative RE (2020s) | Up from 18.6% pre-GFC |
| Ponzi finance | 53.2% of firms (Minsky classification) | Leverage D/E rising from 0.44 to 1.35 |
| Superstar persistence | 37 firms in top-50 for 10+ years | But 49.9% 5-year overlap — partial churn |
| MEC below WACC | 66% of firms earn less than cost of capital (2024) | Keynesian investment drought quantified |
| MEC schedule shift | MEC > WACC: 74% (2008) → 34% (2024) | Demand curve for capital shifted left |
| Earnings obscurance | Technology CC=0.51 — 49% of earnings not backed by cash | Accrual-heavy sectors overstate profit |
| Growth capex collapse | Growth share of capex: ~85% (2008) → ~30% (2024) | Most capex is just maintenance |
| Profit quartile stickiness | Q1 (low profit) persistence: 77.7% | Unprofitable firms stay unprofitable |
| DuPont: leverage drives financials | Financial Services ROE from 7.76x leverage, not margins | Fragile profitability model |
| Altman Z: 65% in distress zone | Median Z-score = 0.59 (2024) | Most firms technically near bankruptcy |
| Survival by size | Q1 median = 7 years, Q4 = infinity | Small firms die fast |
| Profit rate convergence | β = 0.089 (91% mean reversion/year) | Strong support for Shaikh equalization |
| Investment drought is a transformation | Total inv (capex+R&D) rate rose 3.2%→3.6% | R&D compensates for falling capex |
| Minsky FII peaked 2023 | FII=0.77, 60% Ponzi firms, $43T Ponzi assets | Rate hikes pushed fragility to maximum |
| 9/13 sectors countercyclical markups | Prices rise in recessions | Strong evidence of market power |
| Labor share falling | 64.3% → 62.2% (-2pp, SGA-based) | Capital deepening / extraction |
| Info Tech: 25% goodwill | One quarter of assets from acquisitions | Growth via M&A, not organic |
| Financials most systemically connected | r=0.93 with Communication Services | Contagion risk concentrated in finance |
| Comm Services + Info Tech: monopsony signals | SGA/revenue falling + high CR4 | Wage suppression in concentrated sectors |
| SIC reclassification is 0% | Sectoral recomposition is genuine capital flow | Not an accounting artifact |
| US zombie share 3.5× Japan's | US 53.5% vs Japan 15% (2000s) | US zombification exceeds the original "zombie economy" |
| 6% of exits are acquisitions | 901 firms acquired, not killed | Reclassifying raises median survival 9→10 yr |
| Utilities highest acquisition rate | 29% of exits are M&A | Capital consolidation, not destruction |

## Status
- 2026-05-11: Full pipeline complete — 100+ scripts, 99 CSVs, 28 figures, 727K monthly prices, 39K monthly betas, 240 FFIEC bank records
  - Panel: 119,919 records, 14,734 companies, 80 columns, 1983-2025
  - L01d enhancement: +11 XBRL columns (AR, AP, inventory, ETR, etc.), derived ratios (ROA 82%, ROE 88%, D/E 70%), EPS backfill (+29,623 records)
  - Prices: 50,725 annual prices, 5,540 tickers (yfinance)
  - FRED macro: deflator, fed funds rate, BEA corporate profits
  - S01 infrastructure (6 scripts), S02 profitability (8), S03 capital (6), S04 structure (7), S05 deep dives (20), S06 visualization (8 figures)
  - V01 validation: PASS (no duplicates, good referential integrity, 13/80 cols >90% coverage)
  - Outputs packaged: 46 CSVs + 16 figure files + DATA_DICTIONARY.md
