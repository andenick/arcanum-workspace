# DuBois — race.heterodata.org

**Race, stratification & economic disparities in the United States**, presented as a research website with a full data-replication package. Named for W.E.B. Du Bois, who pioneered the empirical study of race and economic stratification.

**Status: live site with all 15 routes serving real, source-traced data** (verified against the repository README, 2026-10-02). Repo: `andenick/race-web`.

---

## What it covers

Six measurable dimensions of racial economic disparity — **wealth, income, employment, poverty, housing, and criminal justice** — plus education, business ownership, geography, and long-run historical context, reconstructed from authoritative public sources:

- **Wealth** — Federal Reserve Survey of Consumer Finances (1989–2022, 12 waves); the Black–White median wealth gap is the north-star series
- **Income / Poverty / Housing / Education** — US Census ACS (B19013, B17001, B25003, C15002)
- **Employment** — BLS Current Population Survey via FRED (Black/White unemployment ratio, 1972–2025)
- **Criminal justice** — BJS *Prisoners* 2020 (imprisonment rates by race)
- **Business** — Census Annual Business Survey (employer firms by owner race)
- **Historical** — SlaveVoyages trans-Atlantic slave-trade database (TAST 2019), MeasuringWorth / HSUS demographics

Nothing on the site is fabricated or interpolated; where a source publishes imputed estimates, that is stated and the figure carried through as published.

## The numbers behind the site

- **15 routes**, every one backed by real data — no "coming soon" placeholders
- **20 data CSVs** in `app/data/` (22 published files total, including the data dictionary and citation record), each listed with download URL and SHA-256 in ``DATA_MANIFEST.md``
- **Replication package** (``anu/``): 27 series, loaders → processors → validators, rebuilding every published CSV from the original public sources (`make all`)

Known gaps are documented on the site's methodology page: no standard ACS 1-year estimates exist for 2020 (a one-year hole in five series families), the 2025 unemployment point is a six-month average, and the data dictionary covers 12 of the 20 data CSVs.

## Stack

- **FastAPI** + Jinja2 templates + **Plotly.js** (vendored, no CDN) — Python 3.12, gunicorn + uvicorn workers
- Shared Heterodata site chrome (header/footer, ecosystem switcher, theme toggle), all vendored
- First-party telemetry writes one row per request to local SQLite — no cookies, no third party, no raw IP stored

```bash
pip install -r app/requirements.txt
cd app && python -m uvicorn main:app --reload --port 8090     # → http://localhost:8090
```

Data files are not distributed in the repository; fetch them from the live site's `/data` page per `DATA_MANIFEST.md`.

## Verification

Count-asserting, not status-code-asserting: each page is checked for the number of records it actually renders against its source CSV, and each chart is confirmed to paint in a real browser. Pages are also checked for real data (no placeholders), offline operation with no CDN, legible charts at every viewport width, and no literal markdown.

**Live at [race.heterodata.org](https://race.heterodata.org).**
> **Hosting status (2026-10-04):** the self-hosted origin box behind this site has been offline
> since late September 2026 (boot failure; physical repair in progress). The site returns when the
> box is back. The repository and data packages below remain fully available.

## License

- **Code**: MIT
- **Data**: reconstructed from public-domain / open government data (CC-BY-4.0 for the harmonized dataset); original agencies remain authoritative
