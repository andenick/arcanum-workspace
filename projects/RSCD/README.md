# RSCD — Shaikh 2016 Replication (Clean Rebuild, Anu Framework v12.2)

Replication and extension program for the empirical data series in Anwar Shaikh's
*Capitalism: Competition, Conflict, Crises* (Oxford University Press, 2016). Two source-dark
series remain honestly `data_unavailable`; five series are withheld from public output.

This is a **clean rebuild** that supersedes two predecessors (CD and CD2). Both predecessors live frozen under `Inputs/` as reference material. The active build lives in `Technical/`.

**Living status (verified 2026-07-17):** **v1.6.0 is published** on GitHub
(`andenick/shaikh-capitalism-data`), with CI green and the hub serving v1.6. The project itself
is healthy (doctor 0 FAIL / 1 honest WARN). A separate workspace-level follow-on program was
paused by the user on 2026-07-11 before any writes; that pause does not roll back or qualify
the published release. The v1.6 audit archive is complete; the older v1.1
pre-release archive remains preserved. RSCD has no project-local RobertDB. Current evidence is
`Technical/PROGRESS_LOG.md` and the latest handoff.

## Quick Start

```bash
cd Technical
python code/run.py --list            # enumerate all S00/L01/P02/V03/M04/A05/O06 scripts
python code/run.py --health          # check imports, API keys, registry parse, paths
python code/run.py --series S201     # run one series end-to-end (L01 -> P02 -> V03)
python code/run.py --validate-only            # FULL 118-series V03 batch (regenerates VALIDATION_REPORT.json;
                                              #   continue-on-fail; non-zero exit if any series FAILs)
python code/run.py --validate-only --series S201   # validate a single series
python code/run.py --gate            # CI gate: anu-doctor + anchor suite + full V03 batch
                                     #   (non-zero exit on any FAIL or anchor RED)
python code/run.py --report          # print the summary table from VALIDATION_REPORT.json
```

> `run.py` is flag-driven — there are no `status`/`advance` sub-commands. Pipeline
> stage lives in `Technical/PIPELINE_STATE.json`; the validation summary is
> `run.py --report`.

## Project Structure

```
RSCD/
├── README.md                  # this file
├── REBUILD_PLAN.md            # 10-phase rebuild plan
├── INPUTS_README.md           # Inputs/ tree documentation (Inputs/ itself is write-protected)
├── Inputs/                    # READ-ONLY, frozen legacy
│   ├── Capitalism Data/      # CD legacy v1 (Shaikh 2016, Anu Framework v4.x era)
│   └── CD2/                   # CD2 prior best (Anu Framework v6.0 + cd2-replicator)
├── SalvagedInputs/            # Curated benchmarks pulled forward from CD/CD2
│   ├── book_data/             # Shaikh's published values (ground truth for V03)
│   ├── extension_benchmarks/  # CD/CD2 final CSVs as validation truth
│   ├── methodology_decisions/ # Decision logs worth keeping
│   ├── figures_reference/     # HDARP figure metadata (209 figures, A- quality)
│   └── MANIFEST.md
├── Technical/                 # Active Anu Framework v12.0 build
│   ├── series_registry.json   # Single source of truth (Phase 2)
│   ├── PIPELINE_STATE.json    # 9-stage tracker
│   ├── ANU_LEDGER.json        # Per-series artifact inventory
│   ├── SERIES_CORRESPONDENCE_MATRIX.json
│   ├── SUBSOURCE_METADATA.json
│   ├── VALIDATION_REPORT.json
│   ├── PROVENANCE_INDEX.json
│   ├── PROGRESS_LOG.md
│   ├── Build/                 # anu-build cascade
│   ├── code/                  # S00/L01/P02/V03/M04/A05/O06 + utils + run.py
│   ├── research/              # Per-series research dossiers (Phase 3)
│   ├── docs/                  # Chapters, DPRs, EPRs, decisions, methodology
│   ├── chopped/               # Machine-readable CSVs (Phase 8)
│   ├── extenbooks/            # Human-readable Excel workbooks (Phase 8)
│   ├── viz/                   # Plotly Dash app (Phase 9)
│   ├── Handoffs/              # Session handoff docs
│   ├── MIGRATION/             # Crosswalks CD/CD2 → RSCD
│   ├── data/                  # raw/ + processed/ Parquet
│   └── config/                # api_keys.env (gitignored)
└── Outputs/                   # Distribution packages (Phase 10)
    ├── Publish/               # GitHub replication repo
    ├── Drive/                 # Consumer Google Drive package
    ├── Archive/               # Audit-grade transparency package
    ├── Figures/               # Exported visualizations
    └── Reports/               # PDFs, executive summaries
```

## Frozen Legacy

| Path | What | Why frozen |
|---|---|---|
| `Inputs/Capitalism Data/` | CD project (S001–S105, Shiny v10.1, 182 Extenbooks, 209 figures) | Superseded by clean rebuild; methodology drift, ad-hoc layout |
| `Inputs/CD2/` | CD2 (S001–S113, Anu Framework v6.0, `cd2-replicator`) | Prior best — re-author logic against S/ES/AS scheme, don't copy |

Both legacy trees are write-protected — no build step may modify them. `SalvagedInputs/` holds the curated subset the new build inherits.

## Series ID Scheme

Series ID Spec v2.2 (Anu v12.2). Canonical prefixes are `S` and `XS`; the legacy
`AS`/`ES` prefixes were retired in the 2026-06-10 AS/ES → XS migration (anu-doctor
P12 now rejects them). 118 series total (101 `S` + 17 `XS`).

| Prefix | Meaning | Pattern | Example |
|--------|---------|---------|---------|
| `S` | Book series (Shaikh 2016), chapters 2–17 | `S{chapter}{seq}` | `S201` (Ch2 series 01) |
| `XS` | "Extra Series" — carries `xs_class` + `xs_attribution` | `XS###` / `XS####` | `XS001`, `XS2001` |
| &nbsp;&nbsp;↳ `xs_class: appendix` | GPIM construction internals (former `AS001`–`AS009`, chapter 6) | `XS00#` | `XS003` |
| &nbsp;&nbsp;↳ `xs_class: external_study` | Other-study series (former `ES2001`–`ES2305`, chapter 0) | `XS2###` | `XS2201` |

Full reset from CD/CD2: the old `S001–S113` flat IDs are **not** preserved. The
old→new (incl. AS/ES → XS) mapping is `Technical/MIGRATION/crosswalk.csv` (also
shipped in the publish bundle); CD/CD2 traceability is in
`Technical/MIGRATION/CD_to_RSCD_crosswalk.csv` and `CD2_to_RSCD_crosswalk.csv`.

## Built With

- **Anu Framework v12.2** — current framework; earlier release records retain their historical stamps
- **Python 3.11+** — data processing pipeline
- **Plotly Dash** — interactive visualization

## Citation

Shaikh, A. (2016). *Capitalism: Competition, Conflict, Crises*. Oxford University Press.

## License

MIT (code) + CC-BY-4.0 (data). See `Technical/LICENSE`.
