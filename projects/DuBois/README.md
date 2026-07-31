# DuBois — Housing, Rents & Eviction Economics

**Named after**: W.E.B. Du Bois (1868–1963), sociologist, historian, and pioneering data visualization innovator whose *The Philadelphia Negro* (1899) was among the first rigorous empirical studies of urban housing, labor, and racial inequality in America
**Created**: 2026-05-10
**Status**: Scaffolding
**Architecture**: AnuData v1.0 (planned)
**Domain**: Housing prices, rents, affordability, eviction, homelessness, residential segregation

---

## Purpose

DuBois extends Arcanum's existing HMDA mortgage lending data (59.7M records) into the broader housing economy: prices, rents, affordability, evictions, and residential segregation. HMDA captures who gets loans — DuBois captures what happens to housing markets, tenants, and communities.

Du Bois's 1900 Paris Exposition data portraits — visualizing Black American economic life through innovative charts — pioneered the kind of data-driven social analysis this project continues.

## Research Questions

1. How do housing cost burdens vary by income, race, and geography?
2. What is the relationship between eviction rates and labor market conditions?
3. How do housing price cycles relate to financial instability (Minsky)?
4. What are the spatial patterns of residential segregation and how have they evolved?
5. How does housing wealth concentration compare to financial wealth concentration?

## Data Sources

### Primary (API / Bulk Download)

| Source | Coverage | Records (est.) | Access Method |
|--------|----------|----------------|---------------|
| **Zillow Research Data** | Home values (ZHVI), rents (ZORI), inventory — zip/metro/county | 2-5M | Zillow Econ Data API / CSV |
| **Census ACS** | Housing characteristics, costs, tenure, crowding | 5-10M | Census API |
| **American Housing Survey** | Detailed housing conditions, biennial | 500K+ | Census bulk download |
| **FHFA House Price Index** | Repeat-sales HPI by MSA, state, zip | 1M+ | FHFA bulk download |
| **Eviction Lab (Princeton)** | Eviction filing rates by county/city, 2000-present | 500K+ | evictionlab.org download |
| **HUD Fair Market Rents** | FMR by county/MSA, annual | 200K+ | HUD download |
| **HUD PIT Count** | Point-in-time homeless counts, annual | 50K+ | HUD exchange |
| **Census Building Permits** | New residential construction by geography | 500K+ | Census API |
| **FHFA Mortgage Performance** | Loan-level performance data | 2M+ | FHFA download |

### Secondary (HDARP / Literature)

- Desmond (2016) *Evicted* — ethnography of eviction
- Rothstein (2017) *The Color of Law* — government-sponsored segregation
- Case & Shiller housing price research
- Mian & Sufi (2014) *House of Debt*

## Cross-Project Links

| Project | Connection |
|---------|------------|
| **HMDA** | Mortgage lending ↔ housing prices and outcomes |
| **Volcker** | Bank exposure to housing sector |
| **Piketty** (new) | Housing wealth in wealth distribution |
| **Westchester2_Final** | GIS infrastructure analysis ↔ housing patterns |
| **Davis** (new) | Housing instability ↔ criminal justice contact |
| **Gerhard** | Public housing, Section 8, HUD spending |

## Structure

```
DuBois/
├── Inputs           # Zillow, Census, FHFA, Eviction Lab downloads
├── Technical        # Collection scripts, AnuData pipeline
│   ├── AnuData/      # AnuData v1.0 pipeline (planned)
│   ├── DataService/  # shared data-checkout service integration
│   └── Handoffs     # Session documentation
└── Outputs          # Analysis results, housing maps, affordability reports
```

## Data-Service Integration

Priority data for the shared data-checkout service to ingest:
- Zillow ZHVI and ZORI time series (metro/county level)
- FHFA HPI by MSA and state
- Census ACS housing cost burden ratios
- HUD Fair Market Rents
- Eviction Lab filing rates

## Knowledge Base Links

- Connects to: Financial instability (Minsky/Campaign B), accumulation theory
- Data visualization: Du Bois data portraits tradition
- Recommended HDARP: Desmond (2016), Rothstein (2017) for institutional context
