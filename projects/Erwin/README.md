# Erwin — Defense Economics

**A research site on the economics of military spending**: US federal defense spending and its composition, cross-country military expenditure, NATO burden-sharing, the arms industry, defense employment, and defense-adjacent manufacturing profit rates.

The site treats military spending not as an exogenous policy variable but as an **endogenous feature of the economy** — the military-Keynesianism and permanent-arms-economy literature. **328 published data series in 17 categories**, every one tracing to a named published source; nothing on the site is synthetic, estimated, or interpolated by the project. Repo: `andenick/erwin-web` (verified 2026-10-02).

---

## Where the data comes from

All sources are public and named on the site's methodology page: **SIPRI** (military expenditure, arms industry, arms transfers), **NATO** defence expenditure tables, **BEA** NIPA and GDP-by-industry, **BLS** defense employment, **FRED**, **USAspending**, **US Department of Defense** budget documents, the **World Bank WDI**, and **Historical Statistics of the United States**.

The site does not embed its data. At runtime it reads a **published data package** (`Erwin_Web_v1.0.0/`, 82.9 MB / 663 files) mounted read-only into the container, containing:

| Piece | What it is |
|---|---|
| `chopped/` + `chopped_parquet/` | one CSV + one Parquet per series (327 files each) |
| `series_registry.json`, `data_dictionary.csv`, `PROVENANCE.csv` | per-series provenance, construction, units, coverage |
| `WEB_MANIFEST.json` | counts, category breakdown, per-series shape/row-count/chartability |
| `bundles/data/erwin_data_v1.zip` | the whole dataset in one 15,537,057-byte zip, served at `/downloads/data.zip` |
| `BUNDLE_MANIFEST.csv` | every bundle member's bytes + SHA-256 |
| `CITATION.cff`, `llms.txt`, `TERMS.md` | citation record, agent-readable site map, terms |

The package is not distributed in the repository — but it can be **rebuilt from public sources** via the ``anu/`` replication package (registry for all 331 series, per-source fetchers, provenance records, validator).

## Serving it honestly

- **Downloads are first-class**: every series as CSV and Parquet at `/downloads/series/<id>.csv|.parquet`, the bulk zip, dictionary, provenance, terms, and citation — the `/data` page lists all 327 per-series links without JavaScript. JSON is offered only as *metadata* under `/metadata/`, not as a data format.
- **Six column layouts, one honest charting layer**: `app/shapes.py` reads each file and says what it actually is — plain time series, country panel, BEA API wire format, wide, cross-section, or shifted-header. Cross-sections draw as ranked horizontal bar charts; a series that cannot be charted answers HTTP 200 with a stated reason, never a 5xx.
- **Count-asserting readiness**: `/readyz` fails if the package is not mounted or holds no series; the deploy gate asserts counts and bytes, never a bare HTTP 200. `ERWIN_DATA_DIR` has no default — an unset value fails loudly rather than mounting nothing.
- No CDN dependency; Plotly and every stylesheet are vendored.

## Run it

```bash
pip install -r app/requirements.txt
ERWIN_WEB_PKG=/path/to/Erwin_Web_v1.0.0 \
  python -m uvicorn app.main:app --port 8000
# or: docker compose up --build   (set ERWIN_DATA_DIR in .env)
```

Python 3.12 is what the image pins and tests against. One first-party telemetry package is deliberately not bundled (see the repository README's dependency note for the minimal workaround).

## License

- **Code**: MIT
- **Data**: research and educational use; cite per the `CITATION.cff` shipped in the package. Underlying sources (SIPRI, NATO, BEA, BLS, the Federal Reserve, DoD) retain their own terms.

Suggested citation: Anderson, N. (2026). *Erwin — Defense Economics.*
