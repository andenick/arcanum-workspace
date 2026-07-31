# Methodex — Economic-Methodology MCP

**What:** a "Context7 for economic methodology" — collect, transcribe, structure, and serve (via an MCP server) the *construction-and-revision history* of every economic statistic. Killer query: *"How was U.S. GDP measured in 1985 vs 2015, and what changed?"* — answered with citations to the governing manual edition, cross-linked to the data vintage.

**Killer tool:** `get_methodology(statistic, as_of_date)`. Prior-art research (2026-06-01) confirmed nothing like it exists — genuine white space.

**Status:** a standalone, installable MCP server (`pip install -e .` → `methodex-mcp` console script). Name **Methodex** is final (domain via `methodex.io` / `.ai`). It began as a corner of the shared PDF corpus and became a first-class research project of its own on 2026-06-02.

## Relationship to the shared PDF corpus
Methodex is a methodology-PDF library + Knowledge Base, so it *consumes* the existing document holdings as an **input**: `phase0_dedup.py` reads the shared methodology-PDF holdings and the Robert `UNIFIED_PDF_CATALOG.csv` to dedup against the corpus already on hand.

## Layout
- `src/methodex_mcp.py` — the MCP server (**13 tools**: `resolve_statistic`, `get_methodology`, `get_revision_history`, `diff_methodology`, `search_methodology` (exact keyword), `semantic_search` (TF-IDF meaning-based, no external API), `get_document`, `get_concept_history`, `get_table_history`, `list_measures`, `get_vintage_data` (ALFRED/FRED real-time data cross-link), `get_methodology_timeline` (markdown/Mermaid export), `methodex_status` (coverage snapshot + per-statistic data card)). `get_revision_history`/`diff_methodology` take `verified_only`. Uses FastMCP if `mcp` is installed, else a CLI demo. All paths are `__file__`-relative → move-safe.
- `src/break_flags.py` — analysis tool: `break_flags(statistic_id, from_year, to_year)` + `annotate_series` + `break_table_csv` + `break_flags_json` + `annotate_series_csv` (R/Stata panel long-format with 0/1 break dummies) + CLI (`--csv`/`--json`/`--panel`).
- `tests/test_tools.py` — 10/10 checks (loads all 9 tools, resolves from disk).
- `pyproject.toml` — package `methodex-mcp`, `py-modules = ["methodex_mcp"]`, console entry `methodex-mcp`.
- `campaign_state.json` — **canonical resumable state** (source of truth for the loop).
- `*.py` — pipeline (dedup, classify, acquire, harvest_html, extract_text_all, merge_events, build_documents, merge_discovery, build_measures).
- `schema` — JSON Schemas for the 5-entity frozen ontology + Series/Measure.
- `*.md` — plans (`CAMPAIGN_PLAN.md`, `CORPUS_EXPANSION_PLAN.md`, `TIER5_SECONDARY_SOURCES.md`, `DISCOVERY_PLAN.md`).
- `DISCOVERY_WISHLIST.csv` — 467 net-new sources / 247 producers (gap-finding output for the acquisition agent).
- `library/` — 472 acquired source PDFs · `Knowledge_Base` — extraction text dumps · `logs/`
- `Inputs` · `Technical` · `Outputs` — the standard 3-folder project layout.

## State (as of 2026-06-02)
Stage D **COMPLETE** + source-discovery **COMPLETE**: **755 cited revision events / 135 statistics** (24 adversarially verified, completion rating 90/100), **1,758 schema-valid docs**, **106 NIPA Measure/Table entities**, 9 MCP tools (10/10 tests). Remaining work is user-gated: manual fetches (`MANUAL_FETCH_WORKLIST.md` — cbo.gov / bls.gov-online-HOM / FRASER all 403 to Python + WebFetch), copyright sign-off for a T3.3 public release, optional Hopper-OCR of pre-1993 SNA scans.

## Public-domain serving mode (M2)
The corpus mixes **public-domain US-federal-government** material (servable verbatim) with **in-copyright intl/academic** material (digest-only). `classify_license.py` sets `provenance.license` on every doc — `"US-Gov public domain"` iff the producer is a US-federal agency **and** geography is US (legal basis 17 U.S.C. §105), else `"copyright/unknown"` (report: `PUBLIC_DOMAIN_CARVE.md`; 957 of 2,072 docs are public domain). The hosted public instance runs with **`METHODEX_PUBLIC_ONLY=1`** (or CLI `--public-only`): the in-memory stores are filtered once at load to public-domain docs + their events/measures/statistics, so **every** tool serves ONLY public-domain content — no intl/academic/UNKNOWN material can appear. Default (env unset) = full corpus for the internal/research instance. Public-mode acceptance suite: `tests/test_public_only.py` (8/8); default suite `tests/test_tools.py` unchanged (16/16).

## Run mode
Continuousness-first: many roadmap rounds inline per agent turn, with a scheduled resume used only to cross a forced session boundary. Canonical state lives on disk (not in the conversation) so the run survives context compaction.

## Quick start
```
cd Methodex
python tests/test_tools.py                       # 10/10
python src/methodex_mcp.py demo                  # killer query demo
python src/break_flags.py US.BLS.CPI_U 1990 2005 # methodology-break timeline
pip install -e .                                 # registers `methodex-mcp` console script
```
