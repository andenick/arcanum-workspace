# Erwin — Military & Defense Economics

**Named after**: Erwin Rommel (1891–1944), the "Desert Fox," and Erwin Smith from *Attack on Titan* — both commanders who understood that strategy is resource allocation under constraint
**Created**: 2026-05-10
**Status**: Scaffolding
**Architecture**: AnuData v1.0 (planned)
**Domain**: Military expenditure, defense procurement, arms trade, veterans economics, military Keynesianism

---

## Purpose

Erwin constructs a comprehensive dataset on military economics — the single largest category of discretionary federal spending (~$886B official / ~$1.4T including related programs). Defense spending shapes industrial structure, regional economies, technological innovation, labor markets, and fiscal policy. It is the most direct expression of state power in the economy.

This project treats military spending not as an exogenous policy variable but as an endogenous feature of capitalism: military Keynesianism, imperial political economy, the permanent arms economy thesis (Kidron), and the military-industrial complex (Galbraith, Melman). Every analysis includes institutional and political dimensions — who benefits, what power structures are reproduced, and how defense spending interacts with accumulation dynamics.

## Research Questions

1. How has the composition of military spending evolved (personnel vs. procurement vs. R&D vs. operations)?
2. What is the geographic distribution of defense contracts and military installations?
3. How does military spending relate to profit rates in defense-adjacent industries (CD2)?
4. What does the SIPRI data reveal about the global arms economy and center-periphery dynamics?
5. How do military spending cycles correlate with business cycles and employment (Rosa)?
6. What is the fiscal opportunity cost of defense spending relative to social provisioning (Nightingale, Piketty)?
7. How does military R&D feed into civilian innovation (patents, technology transfer)?

## Data Sources

### Primary (API / Bulk Download)

| Source | Coverage | Records (est.) | Access Method |
|--------|----------|----------------|---------------|
| **SIPRI Military Expenditure Database** | 170+ countries, 1949-2025, constant/current USD, % GDP | 500K+ | sipri.org bulk download |
| **SIPRI Arms Transfers Database** | International arms trade, 1950-present | 200K+ | sipri.org download |
| **SIPRI Arms Industry Database** | Top 100 arms companies, revenue, profit | 50K+ | sipri.org download |
| **USAspending.gov** | Federal contracts, grants — DOD is largest agency | 5-10M | USAspending API |
| **FPDS (Federal Procurement Data System)** | Defense contracts by vendor, product, geography | 5M+ | SAM.gov bulk/API |
| **DOD Budget Materials** | Green Book, comptroller data, future years defense program | 200K+ | comptroller.defense.gov download |
| **VA Budget & Spending** | Veterans benefits, healthcare, disability | 500K+ | va.gov / USAspending |
| **FRED Military Series** | Defense spending in GDP, employment | 5K+ | FRED API (already in the shared data-checkout service) |
| **BEA Defense Contribution** | Defense share of GDP by component | 10K+ | BEA API |
| **BLS Defense Employment** | Employment in defense industries by NAICS | 50K+ | BLS API (via Rosa expansion) |
| **World Bank Military Data** | Military expenditure % GDP, armed forces personnel | 100K+ | World Bank API |

### Secondary (HDARP / Literature)

- Melman (1970) *Pentagon Capitalism* — military-industrial complex as economic system
- Kidron (1967) *A Permanent Arms Economy* — Marxian analysis of military spending
- Galbraith (1967) *The New Industrial State* — military procurement and corporate power
- Stiglitz & Bilmes (2008) *The Three Trillion Dollar War* — true cost accounting
- DOD annual reports and inspector general audits
- Congressional Research Service defense budget reports
- SIPRI Yearbooks

## Analytical Lenses (Cross-Cutting)

Per the Arcanum principle that all projects should be institutional and political:

- **Institutional (Polanyi lens)**: Defense procurement as state-constructed market. Regulatory capture by defense contractors. Revolving door between Pentagon and industry.
- **Historical (Maddison lens)**: Long-run military spending as share of GDP since 1790. Wartime mobilization and demobilization cycles. Cold War permanent arms economy.
- **Concentration (Baran lens)**: Defense contractor consolidation (from 51 to 5 prime contractors since 1990). Monopsony power of DOD. Profit rates in defense vs. civilian sectors.

## Cross-Project Links

| Project | Connection |
|---------|------------|
| **CD2** | Defense profits in sectoral profit rate analysis |
| **Rosa** | Military employment, defense industry wages, veteran labor market |
| **Gerhard** | Defense as largest discretionary budget category |
| **Piketty** | Military spending as fiscal opportunity cost — guns vs. butter |
| **Nightingale** | VA healthcare system, military health spending |
| **Rachel** | Military emissions, DOD as largest institutional polluter |
| **Michelle** | Military-to-prison pipeline, veteran incarceration |
| **Volcker** | War finance, defense-driven interest rate dynamics |
| **Lewis** | Arms trade in international political economy |
| **Leontief** | Military sector in I-O tables (Leontief himself did this work) |

## Structure

```
Erwin/
├── Inputs           # SIPRI downloads, DOD budget docs, USAspending data
├── Technical        # Collection scripts, AnuData pipeline
│   ├── AnuData/      # AnuData v1.0 pipeline (planned)
│   ├── DataService/  # shared data-checkout service integration (SIPRI/USAspending)
│   └── Handoffs     # Session documentation
└── Outputs          # Analysis results, defense spending reports
```

## Data-Service Integration

Priority data for the shared data-checkout service to ingest:
- SIPRI military expenditure (all countries, full time span, constant USD + % GDP)
- SIPRI arms transfers (major weapons, suppliers/recipients)
- USAspending DOD contract obligations by vendor, product service code, geography
- DOD comptroller budget authority by appropriation title
- BEA national defense contribution to GDP

## Knowledge Base Links

- Connects to: Marxian political economy (Campaign F — surplus extraction via state)
- Extends: Post-Keynesian macro (military Keynesianism, effective demand)
- World-Systems (Campaign E — military power in center-periphery structure)
- Recommended HDARP: Melman (1970), Stiglitz & Bilmes (2008), CRS defense primers
