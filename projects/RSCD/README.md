# RSCD — Replication of Shaikh (2016)

Open replication and 1860–2025 extension of the empirical material in:

> Shaikh, Anwar (2016). *Capitalism: Competition, Conflict, Crises.* Oxford University Press.

**118 series** across 17 chapters, 5 external studies, and 9 analytical constructs. **116 producible chopped CSVs**; the remaining 2 (S703, S704) are formally classified `data_unavailable` — published as honest gaps, not fabricated.

**Current release: v1.6.1 (2026-07-19) — "Validation hardening + provenance completion", superseding v1.6.0 (2026-07-11).** Zero data-value changes in v1.6.1: every chopped CSV is value-identical to v1.6.0; all changes are validation infrastructure, metadata, and documentation. Certification counts (unchanged from v1.6): **106 book-period/extension PASS · 8 theoretical · 2 cross-sectional-unavailable · 2 extension-only · 2 data-unavailable — 118 total, zero FAIL.** Repo: `andenick/shaikh-capitalism-data` (verified 2026-10-02); CI runs the replicator check on every push.

---

## Quickstart

```bash
git clone https://github.com/andenick/shaikh-capitalism-data.git
cd shaikh-capitalism-data
python -m venv .venv && .venv/Scripts/activate
pip install -r requirements.txt

python anu/scripts/V01_validate.py        # key-free package gate: shipped data vs registry

cp replicator/config/api_keys.env.example replicator/config/api_keys.env
# edit FRED_API_KEY (and optionally BEA_API_KEY)

python replicator/scripts/replicate.py --series S201   # smoke test (single series)
python replicator/scripts/replicate.py --all           # full replication (~45 min)

python anu/scripts/P01_construct_series.py --all       # or via the anu package layer
python anu/scripts/P02_write_chopped.py
python anu/scripts/V01_validate.py --dir anu/data/final/chopped

python viz/app.py                          # Plotly Dash explorer → http://127.0.0.1:8050
```

## What's in the repository

- `series_registry.json` — canonical 118-series metadata (single source of truth), plus `SUBSOURCE_METADATA.json`, `SERIES_CORRESPONDENCE_MATRIX.json` (Shaikh → modern-source crosswalk), `PIPELINE_STATE.json`, `ANU_LEDGER.json`, `VALIDATION_REPORT.json` (per-series MAE / max_abs / n)
- `code/` — the S00 → L01 → P02 → V03 → M04 → A05 → O06 pipeline (118 per-series loaders, constructors, validators) with `run.py` orchestrator (`--series` / `--health` / `--report`)
- `chopped/` + `extenbooks/` — the deliverable: 116 machine-readable CSVs + 116 Excel workbooks
- `replicator/` — self-contained clean-venv reproduction package (`scripts/replicate.py`, bundled `lib/`, `inputs_bundled/`)
- `anu/` — the Anu replication-package layer: package registry, per-source fetchers, construct + V01 gate, per-source DPRs, `make check` (key-free)
- `research/` — 118 per-series research dossiers with verbatim Shaikh quotes (118/118)
- `docs/` — per-chapter research summaries + adequacy reports (17/17 chapters PASS the adequacy gate), 236 per-series DPR/EPR docs, 6 architectural decision records, methodology notes (NIPA T7.11 FISIM remap, IFS line→SDMX remap)
- `Build/` — `BUILD_NARRATIVE.md`, `STEP_LOG.jsonl` (1,376 timestamped pipeline events), phase validation and viz-quality reports
- `viz/` — Plotly Dash application

## Headline results

| Metric | Value |
|--------|-------|
| Series authored | 118 |
| Series producing chopped output | 116 |
| Series `data_unavailable` | 2 (S703, S704) |
| Verbatim Shaikh quotes in `research/` | 118/118 |
| Chapter adequacy gate PASS | 17/17 |
| Mean validation MAE (face-value match) | < 1.5% |
| Visualization QA | 11/11 PASS (+1 N/A) |

v1.6.1 additionally registered **218 independent validation anchors** from printed artifacts independent of the files their loaders read (Shaikh 2020; Shaikh & Jacobo 2020; Shaikh, Coronado & Nassif-Pires 2020; IRS SOI Pub 1304; Weber & Shaikh; BEA *Long Term Economic Growth* 1966), reducing the circular-validation warnings from 7 to 1 — the last deferral (S202, BEA 1977 *Fixed Reproducible Tangible Wealth*) is documented in the registry and requires academic-library access.

## License & predecessors

Code MIT; data CC-BY-4.0 (attribution to Shaikh + this repository). This is the v1.0+ rebuild; earlier prototypes — **Capitalism Data (CD)**, 105 series, frozen 2025, and **CD2**, 114 series, frozen 2026-04 — are superseded, with crosswalks in `MIGRATION/`.
