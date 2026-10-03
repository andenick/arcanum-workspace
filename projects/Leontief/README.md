# Leontief — U.S. Input-Output Tables Analysis

**Leontief** is an open analysis toolkit and website for U.S. input-output (I-O) accounts. It collects every annual U.S. Bureau of Economic Analysis (BEA) I-O table from **1997 through 2024** at the BEA Summary 71-sector level, computes the standard derived matrices, and publishes them as machine-readable downloads alongside tutorials and reproducible empirical studies. Named after **Wassily Leontief** (1906–1999), Nobel laureate and pioneer of input-output analysis. Repo: `andenick/leontief` (verified 2026-10-02).

> **This is a code-only repository.** The raw BEA inputs and generated outputs are not committed. The collectors below rebuild the source data from the public BEA API; the website's `site_data/` cache is built from those outputs.

---

## What it does

- **Collects** all 28 annual U.S. BEA I-O tables (1997–2024) plus GDP-by-industry and satellite accounts (trade, capital, energy) via the BEA API.
- **Derives**, for each year, seven matrices: the raw **Use** and **Supply** tables, the direct-requirements matrix **A**, a squared variant **A_square**, the **Leontief inverse** L = (I − A)⁻¹, the **value-added** rows, and the **final-demand** columns.
- **Validates** derived matrices against BEA's own published benchmark figures.
- **Publishes** everything through a FastAPI website (`webapp/`) with downloads in CSV / XLSX / JSON / Parquet, a 10-tutorial Learn track, and 10 reproducible studies.

## Repository layout

```
.
├── Technical/          # Data construction pipeline (run to rebuild source data)
│   ├── src/            # BEA collectors, parsers, I-O analysis, integration modules
│   ├── scripts/        # One-off analysis / comparison / download scripts
│   ├── apps/           # Streamlit exploration platform
│   ├── docs/           # LaTeX report templates
│   └── research/       # Research notes
├── webapp/             # FastAPI website (see webapp/README.md for the full guide)
├── anu/                # Data-replication package (see anu/README.md)
└── requirements.txt
```

## Quick start

```bash
git clone https://github.com/andenick/leontief.git
cd leontief
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\Activate
pip install -r requirements.txt

# Configure (bring your own BEA key — free at https://apps.bea.gov/API/signup/)
cp .env.example .env

# Rebuild the source data (optional)
export BEA_API_KEY=your-free-key-here
python "Technical/src/bea_api_collector.py"

# Serve the website from a pre-built cache (no key needed to serve)
cd webapp
pip install -r requirements.txt
python data_pipeline/build_sectors.py
python data_pipeline/build_cache.py
python -m uvicorn app.main:app --app-dir . --port 8080
```

`BEA_API_KEY` is required to rebuild data; `DATA_ROOT` optionally relocates the data tree outside the clone.

## About the matrix dimensions

The canonical Use matrix is 71 × 71 (commodity-by-industry). The derived technical-coefficient matrix **A** is non-square (one commodity row is dropped during BEA's reconciliation); **A_square** restores a full 71 × 71 form so the Leontief inverse **L** = (I − A)⁻¹ can be computed. This is an intentional methodological choice, documented on the website's `/methodology` page.

## License & replication

Dual-licensed (see `LICENSE` / `LICENSES.md`): **code** (Technical/, webapp/, study bundles) under **MIT**; **derived matrices, documentation, and site content** under **CC BY 4.0**. BEA source data is U.S. government public domain. The [`anu/`](anu/) directory contains a complete data-replication package: `series_registry.json` (the canonical data contract), fetch/process/validate scripts, and Data Provenance Records — see [`anu/README.md`](anu/README.md) to reproduce the data.
