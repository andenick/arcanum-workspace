# MacroModeling

Comprehensive macroeconomic model library spanning all major traditions — heterodox and mainstream. Implements, replicates, and compares models from Stock-Flow Consistent (SFC), DSGE, New Keynesian, Real Business Cycle, VAR, heterogeneous-agent, input-output, Kaleckian, Sraffian, overlapping-generations, and agent-based frameworks.

**53 native model implementations across 11 traditions**, version 7.0.0. Repo: `andenick/macromodeling` (verified 2026-10-02).

---

## Vision

Build the most thorough open collection of macroeconomic models, each implemented from primary sources with full documentation of equations, calibration, and empirical validation. Every model is traceable to its source paper via a structured Knowledge Base of extracted equations and parameters. Cross-model comparisons reveal how different traditions answer the same policy questions differently.

## Model Traditions

| Tradition | Key References | Status |
|-----------|---------------|--------|
| **Stock-Flow Consistent (SFC)** | Godley & Lavoie (2007, 2012), Zezza WP 494/919/958 | 16 models (12 GL + 3 Levy + endogenous money) |
| **DSGE** | Smets-Wouters, Gali, CEE, NK-Capital | 4 native + 3 external |
| **New Keynesian** | Woodford, Gali textbook, Hicks cycle | 2 native + 3 external |
| **Real Business Cycle** | Kydland-Prescott, KPR, Solow | 3 native + 1 external |
| **VAR / SVAR / BVAR** | Sims, Blanchard-Quah, sign restrictions, narrative | 4 native + 1 external |
| **Heterogeneous Agent** | Aiyagari, HANK, Krusell-Smith, buffer-stock | 4 native + 1 external |
| **Input-Output** | Pasinetti, Lewis dual economy, empirical I-O | 4 native + 2 external |
| **Kaleckian** | Bhaduri-Marglin, Goodwin, Kaldor, conflict inflation | 7 native |
| **Sraffian** | Sraffa prices, supermultiplier, joint production, Shaikh | 6 native |
| **Overlapping Generations** | Diamond (1965) OLG | 1 native |
| **Agent-Based** | Dosi K+S, SFC-ABM | 2 native |

## Installation

Requires Python 3.11+ (developed on 3.13).

```bash
git clone https://github.com/andenick/macromodeling.git
cd macromodeling
python -m venv .venv && source .venv/bin/activate    # Windows: .venv\Scripts\activate
pip install -r Technical/requirements.txt

pytest Technical/tests/test_all_models.py    # fresh clone: 159 passed, 2 skipped
```

The two skips are studies depending on a local-only data tree not part of the repository; every model implementation is exercised. `Technical/requirements.txt` lists only what the repository actually imports — external packages surveyed in `Technical/External/README.md` (econpizza, gEconpy, econ-ark, quantecon, …) are optional. **Local-only directories** (`Inputs/`, `Knowledge_Base/`, `empirical_studies/`, `Outputs/`) are gitignored — large PDFs, vendored repos, generated data, run outputs; they will not appear on a fresh clone.

## API keys — bring your own (optional)

No key is required to run the models, the test suite, or any simulation; keys are only needed to *fetch* live series. Copy `.env.example` to `.env` and set `FRED_API_KEY` (free), optionally `DATA_ROOT` (a local ALFRED / Z.1 / FRED CSV tree) and `DATA_KEYS` (an external key store). Without a key or local tree, calibration scripts fall back to **synthetic placeholder series** — fabricated, not observed; every artifact they produce is labelled `synthetic`, and `V01_validate_targets.py` reports those rows as `NOT_VALIDATED` rather than passing them against published benchmarks.

## Quick start & data

```bash
cd Technical/Models/SFC/godley_lavoie && python model_sim.py       # run an existing model
cd tests && python verify_model_sim.py                              # verify replication
cd calibration && python calibrate_model_pc.py                      # calibrate to US data
```

Data sources: Z.1 Financial Accounts (SFC sectoral balance sheets) · FRED macro series · BEA NIPA calibration targets · ALFRED historical vintages (real-time VAR analysis).

## License & history

MIT (code) + CC BY 4.0 (outputs/docs); `CITATION.cff` carries machine-readable citation metadata; authoritative per-model status lives in `Technical/MODEL_CATALOG.md`. Began as "Levy Macro Model" (October 2025), implementing the 12 Godley-Lavoie SFC textbook models; renamed **MacroModeling** (April 2026) and expanded to all major macroeconomic modeling traditions.
