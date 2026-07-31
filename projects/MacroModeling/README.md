# MacroModeling

Comprehensive macroeconomic model library spanning all major traditions -- heterodox and mainstream. Implements, replicates, and compares models from Stock-Flow Consistent (SFC), DSGE, New Keynesian, Real Business Cycle, VAR, Input-Output, Kaleckian, Sraffian, and Agent-Based frameworks.

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

## Architecture

```
MacroModeling/
  Inputs
    Papers/              Source PDFs organized by tradition
    Data/                Economic data (FRED, BEA, Z.1)
  Technical
    Knowledge_Base      Extracted equations, tables, parameters
    Models/              Implementations organized by tradition
      SFC/
        godley_lavoie/   Godley-Lavoie textbook models (3 implemented)
        levy_institute/  Levy Strategic Analysis models
      DSGE/  NK/  VAR/  IO/  Kaleckian/  ABM/  RBC/  OG/  HetAgent/
    shared/              Cross-model utilities
      framework/         Base model class, SFC framework, solvers
      data_loaders/      ALFRED, Z.1, FRED loaders
      visualization/     Plotting and comparison tools
      calibration/       Calibration infrastructure
      validation/        Cross-model validation
    AnuData/           Empirical model estimation and comparison
    ANU_REPLICATOR/      Replication of published data series
    series_registry.json Anu Suite single source of truth
    MODEL_TAXONOMY.md    Classification of all model families
    MODEL_CATALOG.md     Specific implementation targets
  Outputs
    Data/                Model simulation outputs
    Reports/             Analysis reports (LaTeX -> PDF)
    Comparisons/         Cross-tradition comparison results
```

## Quick Start

### Run an existing SFC model

```bash
cd godley_lavoie
python model_sim.py
```

### Verify model replication

```bash
cd tests
python verify_model_sim.py
```

### Calibrate to US data

```bash
cd calibration
python calibrate_model_pc.py
```

## How to Add a New Model

1. **Source the paper**: Place the PDF in `{tradition}`
2. **Extract the equations**: Transcribe the model's equations, tables, and parameters into `{tradition}`
3. **Implement**: Create model code in `{model_name}`
4. **Validate**: Write verification tests comparing to published results
5. **Calibrate**: Add calibration scripts using the shared data loaders from `shared/`
6. **Document**: Update `MODEL_CATALOG.md` with implementation status
7. **Compare**: Add cross-model comparison studies

## Data Sources

Models draw on standard public macroeconomic data sources:

- **Z.1 Financial Accounts** -- SFC models (sectoral balance sheets, flow of funds)
- **FRED macro series** -- All traditions (GDP, unemployment, inflation, interest rates)
- **BEA NIPA** -- Calibration targets across traditions
- **FAILING_BANKS** -- Financial stability models
- **Historical vintages** -- Real-time data analysis for VAR

## Pipelines

### Paper -> Knowledge Base
```
Papers -> transcribe equations/tables -> Knowledge_Base
```

### Empirical Comparison
```
data ingestion -> loading -> processing -> validation -> model estimation -> analysis -> output
```

## History

This project began as "Levy Macro Model" (October 2025), focused on implementing the 12 Godley-Lavoie SFC textbook models. Renamed to "MacroModeling" (April 2026) and expanded to cover all major macroeconomic modeling traditions.

---

**Version**: 7.0.0
**Last Updated**: 2026-05-05
**53 native models | 161 tests | 25 FRED series | 6 historical eras | 11 traditions | 27 repos | 53/53 equation cards | 53/53 mechanism tags | 33 PARAM_SPACES | 139x fast solver | ModelWorkbench | 10 reusable EquationBlocks | All gaps closed**
