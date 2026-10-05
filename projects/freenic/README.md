# FreeNIC — Free National Information Center

**A research-grade, openly published US banking data warehouse.** FreeNIC harmonizes the major public regulatory data sources — FFIEC Call Reports & UBPR, FDIC (BankFind, SDI, Summary of Deposits, historical), the Federal Reserve (FR Y-9C/BHCF, H.8, FRED), OCC, NCUA, SEC EDGAR, and derived academic panels — into a single, provenance-tracked warehouse with a harmonized variable dictionary, then exports a clean, citable public release.

- **Warehouse:** 62 base tables (49 `main` + 6 `catalog` + 7 `dict`) · 52 shaped views · **4.97 billion rows** (4,968,889,667) · coverage span **1782–2026** across 21 source families (data vintage 2026Q1)
- **Public release (v1.0.0):** **61 files / 13.2 GiB** — 60 Parquet base-table exports plus the 163-year (1863–2026) bank-aggregate spine `long_bank_aggregates_1863_2026.parquet`; every served Parquet's row count equals its warehouse source table's (row-parity gate: 61/61)
- **Explorer site:** [freenic.org](https://freenic.org)
> **Hosting status (2026-10-04):** the self-hosted origin box behind this site has been offline
> since late September 2026 (boot failure; physical repair in progress). The site returns when the
> box is back. The repository and data packages below remain fully available. · **Data host:** [data.freenic.org](https://data.freenic.org) · **Current release: v1.1.0** (2026-07-16 — verified Luck/finhist reconstruction, below)

Repo: `andenick/FreeNIC` (verified 2026-10-02). Code MIT; the data compilation CC-BY-4.0.

---

## Query the data over HTTP (no download)

Every released Parquet is served with HTTP byte-range support, so DuckDB's `httpfs` queries it in place — you fetch only the bytes your query touches:

```python
import duckdb
con = duckdb.connect()
con.execute("INSTALL httpfs; LOAD httpfs;")
con.execute("""
    SELECT failure_year, COUNT(*) AS n_failures, SUM(total_assets) AS assets
    FROM 'https://data.freenic.org/bank_failures.parquet'
    WHERE failure_year IS NOT NULL
    GROUP BY failure_year ORDER BY failure_year DESC LIMIT 10
""").fetchdf()
```

Works identically from R (`duckdb`/`arrow`), the `duckdb` CLI, and browser DuckDB-Wasm. The per-file catalog (bytes, sha256, rows, provenance, URL) is `release-tools/release_v1.0.0/release_manifest.json`.

## Repository layout

| Directory | What it holds |
|---|---|
| ``pipeline/`` | ingestion + validation: a 74-script phase pipeline, 21 read-only test suites, the verified ``reconstruction/`` module, quarterly refresh protocol (`REFRESH.md`) |
| ``site/`` | the freenic.org explorer (FastAPI + Jinja): data + variable-dictionary explorers, curated-slice serving, self-hosting guide (`DATA_SERVING.md`) |
| ``release-tools/`` | release packaging + v1.0.0 metadata (manifest, changelog, citation, license, codebook, croissant, checksums) |

Large artifacts (warehouse DuckDB, full Parquet release, curated slice) are not committed — served from data.freenic.org and cataloged in the release manifest.

## Reconstructing Luck / finhist from raw (v1.1.0)

v1.1.0 rebuilds the Correia–Luck–Verner ("Failing Banks", *QJE* 2026) call-report panel and the OCC historical panel ("finhist") from raw FreeNIC data, then proves the result against the published datasets **cell by cell**. The derivability boundary is stated and enforced per era — cells outside it are classed NOT-DERIVABLE and never imputed; the genuine 1942–1958 gap is kept absent, never synthetically filled. Pre-registered cell-match gates, reported exactly as they came out — **including the failure**:

| Era | Matched share | Gate | Verdict |
|---|---|---|---|
| 1959Q4–1975Q4 (CLV `.dta` derivation layer) | 99.9753% | ≥ 99.9% | **PASS** |
| 1863–1941 finhist (derivation layer) | 99.7061% | ≥ 99.5% | **PASS** |
| 1976–2026 (TRUE independent re-derivation from Fed-direct raw MDRM) | 96.4338% | ≥ 99.5% | **FAIL** |

The 1976–2026 independent tier **fails its gate and is reported plainly**; supplementary metrics (value fidelity 99.90% where both panels report; two-sided divergence 0.0972% of derivable) are stated as non-gate numbers. A tri-engine anchor re-runs the QJE AUC horse race with independently reconstructed regressors — ANCHORED-with-explained-deltas — and an independent adversarial re-review reproduced every headline number bit-for-bit.

## Refresh, replication, citation

FFIEC publishes each reporting quarter ~75 days after quarter-end; the refresh protocol (acquire → ingest → validate → dictionary → views/coverage → republish) is documented in `pipeline/REFRESH.md`. The ``anu/`` directory is a complete data-replication package (`series_registry.json`, fetch/process/validate scripts, Data Provenance Records — see `anu/README.md`). Cite: Anderson, Nicholas. *FreeNIC: Free National Information Center* (v1.0.0), 2026. Upstream sources carry their own terms (most US-government public-domain; the "Failing Banks" deposit CC0 1.0, NY-Fed slice under NY-Fed Terms of Use — full posture in `LICENSE_POSTURE.md`).
