# Measuring the Wealth of Nations — Replication

Complete replication and extension of every empirical claim in Shaikh & Tonak (1994), *Measuring the Wealth of Nations: The Political Economy of National Accounts* (Cambridge UP), plus eight follow-up studies — **64 data series (35 `S` primary book series + 29 `XS` extra series), spanning 1947–2025** (book-period replication 1948–1989 plus extension where applicable), reproducible from the included loaders, processors, and validators.

**Current release: v2.1.1 (2026-07-10, metadata erratum) — the served data payload is v2.1** (2026-07-09). Repo: `andenick/measuring-the-wealth-of-nations` (verified 2026-10-02). CI installs the bundle into a clean Python 3.12 venv on every push and runs the package-integrity check.

![CI](https://github.com/andenick/measuring-the-wealth-of-nations/actions/workflows/replicate.yml/badge.svg)

---

## Series inventory

| Prefix | Meaning | Count |
|--------|---------|:-----:|
| `S`  | Primary book series (Ch. 2, 4, 5, 6, 7, 8, 9) | 35 |
| `XS` | Extra series — analytical aggregates + external follow-up studies | 29 |

The `XS` prefix supersedes the retired `AS`/`ES` prefixes; legacy IDs map via `MIGRATION/crosswalk.csv`. Follow-up studies covered: Tonak (1984); Shaikh & Tonak (1987, 2002); Moos (2017); Mohun (2005, 2013); Karabacak & Tonak (2022, Turkey); Cronin (2001, New Zealand).

## Quick start

```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python build.py status          # 9-stage pipeline state + gate marks
python tests/ci_smoke.py        # package-integrity check (registry + every shipped CSV)
```

Makefile and Docker variants are provided (`INSTALL.md`); the orchestrator runs scripts in canonical phase order **S → L → P → V → M → A → O → E**.

## What's in the release (v2.1)

- **64 chopped CSVs** (`data/`) in canonical wide format + **64 Excel workbooks** (`extenbooks/`, 4 sheets per series: Data / Provenance / Research / Construction)
- **149 per-series docs** (`docs/`): a DPR for every series, an EPR where extension applies
- **185 pipeline scripts** (`code/`): L01 loaders, P02 processors, V03 validators, M04/A05/O06 phases
- **`DIVERGENCE_REGISTER.json` — 73 entries** documenting every intentional deviation from upstream sources and predecessor methodology
- **Validation:** per-series validators **60 PASS / 4 honest registered FAIL** (published as registered divergences, not presented as replications); regression + identity suite **95 pass / 2 skip / 2 justified xfail**; the framework self-audit reports 0 errors / 0 warnings
- **v2.1 analytical additions:** book-faithful gross-capital profit-rate variants (`S513/S514/S517-GROSS-A`, book period 1948–1989 — the printed book `r*` reproduced at MAE 0.0025, 42/42 years exact at 2 dp); a reconstructed time-varying I-O-uplift exploitation arm (`S506-EXT-MARX-KIO`, officially sourced and uncertainty-banded); λ/p* labor-value series published at 3 significant figures; per-series author quotes restored (110 → 348, zero invented or dropped); an answers-first `ANSWERS.md` at the repo root
- **`v2.1.1`** source-verified the XS1202 1964 reference value (restored to `-0.009`) and patched citation metadata — no served data recut required

## What this bundle can and cannot reproduce

A **published-outputs + provenance bundle**, not a from-raw build tree. Offline, with only `pip install -r requirements.txt`, you can install cleanly (CI does), inspect pipeline state, run the package-integrity check, and read every value's lineage (`series_registry.json`, `data/PROVENANCE_DICTIONARY.csv`, `COMPONENT_CHAINS`, per-series DPRs/EPRs, the divergence register). You cannot regenerate the numbers from raw sources — intermediate build inputs live in the maintainer tree; fresh BEA/BLS/FRED fetches need your own free API keys (`INSTALL.md`). **The bundle lets you audit and trust the published data; it does not let you rebuild it from scratch.**

## Known limitations (documented, not hidden)

- Chapter 7 labor-value series (S701–S703) are first-class implementations built from BLS CES, BEA Benchmark I-O matrices, and the Appendix F productive-share filter; benchmark years without I-O coverage are `nan`, never interpolated
- Some series cover only the book's original window; extensions are documented per-series in EPRs
- S517 (productive capital stock K*): the book's "gross" label matches BEA **Net** stock exactly at all five benchmark years — the net column is the headline; a book-faithful gross variant ships for the book period only

## License & citation

Code MIT; data outputs CC BY 4.0 (the file itself — upstream sources retain their own terms). Cite both the package (`CITATION.cff`) and the book: Shaikh, A., & Tonak, E. A. (1994). *Measuring the Wealth of Nations.* Cambridge University Press.
