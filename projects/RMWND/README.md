# Measuring the Wealth of Nations — Replication

Replication and extension program for the empirical claims in Shaikh & Tonak's *Measuring the Wealth of Nations* (Cambridge UP, 1994), plus follow-up studies, with **64 series** built on the Anu Framework. Four current V03 failures remain published as honest, registered divergences rather than being presented as replications.

![doctor](https://img.shields.io/badge/anu--doctor-39%20PASS%20%2F%200%20WARN%20%2F%200%20FAIL-brightgreen)
![release](https://img.shields.io/badge/release-v2.1.1-blue)
![license](https://img.shields.io/badge/license-MIT%20%2B%20CC--BY--4.0-blue)

---

## Living status

**Verified 2026-07-17: active maintenance; public release v2.1.1.** GitHub
`andenick/measuring-the-wealth-of-nations` published v2.1 on 2026-07-09 and the
source-verified v2.1.1 erratum on 2026-07-10. The live site and Drive package remain on the
v2.1 data payload because v2.1.1 changes validation metadata, not served values. The retired
`Outputs/Publish` mirror was archived on 2026-07-07 to
`{archive}/RMWND_publish_mirror_retired_20260707/`; it is not the live publish source.
The earlier v0.4 pre-release bundle also remains preserved in the audit archive.

Robert integration was completed in 2026-06: the v2 extraction is canonical in all five
Robert `_UNIFIED` indexes and v1 is marked superseded. This is unified-index integration,
not a separate project-local RobertDB. Current state authority is
`Technical/PIPELINE_STATE.json`; use `Technical/PROGRESS_LOG.md` for the append-only work log.

---

## Quickstart

```bash
git clone <repo-url> rmwnd && cd rmwnd
pip install -r Technical/requirements.txt
python Technical/build.py status      # current pipeline stage + per-stage completion
python Technical/build.py advance     # see (and run) the next action
cd Technical && make doctor           # current gate: 42 PASS / 1 WARN (P40) / 0 FAIL (anu-doctor project mode)
python Technical/build.py test        # current gate: 95 PASS / 2 skipped / 2 justified XFAIL
```

Container reproduction:

```bash
docker build -f Technical/Dockerfile -t rmwnd .
docker run --rm rmwnd python build.py status
```

Make targets (from `Technical/`):

```bash
make help                            # list targets
make build extenbooks chopped ledger # refresh derived artifacts
make doctor test review              # full validation triple
```

---

## Current release highlights (v2.1.1, 2026-07-10)

- **v2.1 data release**: gross-K* profit-rate variants, reconstructed time-varying-kIO
  exploitation arm and uncertainty band, honest lambda precision bands, and the Tier-A truth fixes.
- **v2.1.1 erratum**: source verification restored the XS1202 1964 reference value to `-0.009`;
  GitHub tag/release and citation metadata were patched, with no site/Drive data recut required.
- **Validation**: 60 V03 PASS / 4 honest registered FAIL; 39 doctor checks PASS; pytest
  95 passed / 2 skipped / 2 justified xfailed.
- **DIVERGENCE_REGISTER**: 73 entries; the known v2.1 relative review-path leak is documented
  for the next whole-register scrub.

Full current evidence is in `Technical/PROGRESS_LOG.md` (2026-07-09/10 entries) and
`Technical/PIPELINE_STATE.json`. Historical v1.x notes remain in `CHANGELOG.md`.

---

## Project structure

```
RMWND/
├── README.md                   # this file
├── CHANGELOG.md                # Keep-a-Changelog format
├── CONTRIBUTING.md             # how to add series / external studies / variants
├── PROJECT_INDEX.md            # navigation index
├── CLAUDE.md                   # per-project agent instructions
├── Inputs/                     # read-only source material (frozen)
│   ├── Shaikh Tonak/           # original book data + HDARP extractions
│   ├── ST2/                    # prior best implementation (frozen)
│   └── Salvaged/               # curated artifacts from predecessors
├── Technical/                  # active build directory
│   ├── series_registry.json   # 64 series definitions (single source of truth)
│   ├── ANU_LEDGER.json        # per-series artifact inventory
│   ├── PIPELINE_STATE.json    # pipeline stage progress (Stages 0-8)
│   ├── DIVERGENCE_REGISTER.json # 12 named divergences as of v1.2
│   ├── PROVENANCE_INDEX.json   # cross-references across DPR / EPR / VPR
│   ├── build.py               # one-shot orchestrator (status / advance / doctor / test / review)
│   ├── Dockerfile + Makefile  # hermetic + scripted reproduction
│   ├── requirements.txt       # pinned dependency set
│   ├── tests/                 # pytest (90 PASS + 1 honest XFAIL)
│   ├── Build/                 # documentation cascade
│   │   ├── ANU_BUILD_MANIFEST.json
│   │   ├── SUBSERIES_PLAN.json
│   │   ├── STEP_LOG.jsonl     # append-only per-action audit log
│   │   └── BUILD_NARRATIVE.md
│   ├── code/                  # L01 loaders / P02 processors / V03 validators / M0x adjusters / O01 outputters
│   ├── research/              # per-series research JSONs (verbatim quotes per Decision 0007)
│   ├── docs/                  # DPRs, EPRs, VPRs, methodology, GLOSSARY, decisions
│   │   ├── GLOSSARY.md        # Marxian + framework + data-source terminology
│   │   ├── series/            # per-series DPR + EPR
│   │   ├── variants/          # VPR variant records
│   │   └── decisions/         # (deprecated location; canon is the workspace decision record)
│   ├── chopped/               # machine-readable CSVs (wide format per Decision 0005)
│   ├── extenbooks/            # human-readable Excel workbooks (4-sheet canonical)
│   ├── viz/                   # Plotly Dash interactive visualization
│   └── Handoffs/              # session handoffs, plans, BACKLOG, doctor / review runs
└── Outputs/                   # distribution packages (3 channels)
    ├── Publish/               # retired pointer; former mirror archived 2026-07-07
    ├── Drive/                 # Google Drive consumer package (69 files at v1.0)
    ├── Archive/               # pointers to audit-grade archive releases
    └── Figures/               # exported visualizations
```

---

## Series ID scheme

| Prefix | Meaning | Count |
|--------|---------|-------|
| `S` | Primary series from the book (chapters 2–9, incl. S401/S402 I-O summaries) | 35 |
| `XS` | External-study replications (Tonak 1984, ST 1987/2002, Mohun 2005/2013, Moos 2017, Karabacak/Tonak 2022, Cronin 2001) + analytical/appendix series (XS001–XS004) | 29 |
| **Total** | | **64** |

*Prefix note (Wave-3A 2026-07-19): the registry's current scheme is 35 `S` + 29 `XS` under the AS/ES→XS migration (Series ID Spec v2.2); the historical 33 S / 27 ES / 4 AS split counts the same 64 series under the old prefixes. The `doctor` badge above reflects the v2.1.1 check set (39 checks); the current framework script runs 43 checks (42 PASS / 1 WARN P40 / 0 FAIL).*

Subseries suffixes (`-A`, `-EXT`, `-COMBINED`, `-INTERP`, `-FLOW`) are defined in `Technical/docs/GLOSSARY.md` §2 (subseries-suffix convention).

---

## 5-minute tour

A Jupyter notebook walkthrough (`notebooks/quickstart.ipynb`) is planned for the v1.2 Outputs/Drive bundle. It will load `master_data_long.csv`, plot S506 (rate of exploitation), S513 (Marxian profit rate, stock-form), and compare to Mohun 2005 (ES1401) on the same axes. Track this deliverable in `Technical/Handoffs/V1.2_OUTSTANDING_STEPS_PLAN.md` Track C.4.

Until then, the interactive Plotly Dash app gives the same tour live:

```bash
python Technical/viz/app.py    # opens at http://localhost:8050
```

---

## Documentation map

| Doc | Purpose |
|---|---|
| `README.md` | This file — orientation and quickstart. |
| `CHANGELOG.md` | Release notes, Keep-a-Changelog format. |
| `CONTRIBUTING.md` | Workflows for adding series / external studies / variants; PR template; review process. |
| `PROJECT_INDEX.md` | Navigation index to every important artifact. |
| `Technical/docs/GLOSSARY.md` | Marxian terms (TP\*, C\*, V\*, S\*, e, r\*, K\*, λ\*, …), framework terms (DPR, EPR, VPR, DIV, decisions), data-source abbreviations. |
| `Technical/docs/ROADMAP.md` | Release roadmap, milestone definitions. |
| `Technical/docs/IMPLEMENTATION_PLAN.md` | Multi-phase implementation plan. |
| `Technical/docs/API_REFERENCE.md` | Auto-generated API reference for `Technical/code/utils/` (regenerate via `python Technical/tools/gen_api_reference.py`). |
| `Technical/viz/exports/citation_graph.dot` / `.svg` / `.html` | Directed citation graph across the 64 series + external papers (Graphviz DOT, rendered SVG, interactive D3). Regenerate via `python Technical/tools/gen_citation_graph.py`. |
| `Technical/Handoffs/BACKLOG.md` | Items deferred from v1.2 to v1.3+. |
| `Technical/Handoffs/V1.2_OUTSTANDING_STEPS_PLAN.md` | The v1.2 plan, with Tracks A–E and per-iteration sequencing. |
| `Technical/INSTALL.md` | Installation details beyond the quickstart. |
| `Technical/CITATION.cff` | Citation metadata (cffconvert-validated). |
| Decision 0007 | Verbatim-quote canonical schema (workspace-level decision record). |
| Decision 0008 | reference_values year-keyed-scalars policy (workspace-level decision record). |

---

## Built with

- **Anu Framework v12.2** — current workspace framework; project history includes earlier v12.x builds
- **Python 3.11+** — data processing pipeline (3.11 / 3.12 / 3.13 supported)
- **pandas, numpy, scipy** — numerical core
- **pytest** — test suite (95 PASS + 2 skipped + 2 justified XFAIL at v2.1.1)
- **Plotly Dash** — interactive visualization
- **ruff + black** — code quality

---

## Citation

```bibtex
@book{shaikh_tonak_1994,
  author    = {Shaikh, Anwar and Tonak, E. Ahmet},
  title     = {Measuring the Wealth of Nations:
               The Political Economy of National Accounts},
  publisher = {Cambridge University Press},
  year      = {1994},
  address   = {Cambridge}
}
```

For citing this replication (DOI assignment pending Track C.1), use the metadata in `Technical/CITATION.cff`.

---

## License

- **Code**: MIT — see `Technical/LICENSE`.
- **Data**: CC-BY-4.0 (planned for the `Outputs/Publish/` bundle).
- **Original Shaikh & Tonak (1994) source material**: copyright Cambridge University Press; reproduced under fair-use academic-replication standards. Knowledge-base extractions live in `Inputs/Shaikh Tonak/Knowledge_Base/` and are not redistributed in the distribution channels.
