---
name: robert-db-harvest
version: "1.0"
description: "Mechanically harvest KB table CSVs into robertdb.sqlite — sniff conventions, mint immutable table_uids, parse rows/cols, record mechanical field provenance, then chain quality scoring and view regeneration."
when-to-use: '"User wants to load (or re-load) a project''s extracted tables into the Robert database, after init; or to harvest a specific category/document/doc-glob."'
search-hints: "robert db harvest tables csv adapters convention sniff parse table_uid mechanical provenance quality views rebuild-doc"
argument-hint: "[project]"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit, Agent
requires: robert-db-init
part-of: Robert Database Framework v1.0
---

# robert-db-harvest — Stage 2 (HARVEST)

Walk the project's `Knowledge_Base`, discover every table artifact, parse it with
a sniffed convention adapter, and record one `xtables` row per table with an
immutable `<PJ>-T-NNNNNN` id. Harvest **invents no metadata** of its own: everything
it parses from a CSV is provenance `mechanical`. It additionally **ingests any HDARP
v6.3 native-enrichment sidecar** (`RDB_METADATA*.jsonl`) the extraction emitted,
feeding those honest `from_source`/`agent_inferred` lines through the unchanged
`rdb_merge_patches.py` — so on a v6.3 KB the harvested DB already carries real
titles/pages/units and `robert-db-enrich` becomes a thin verification pass. On a
legacy KB (no sidecar) honest recovery is still the job of `robert-db-enrich`.

## Purpose

Populate `documents`, `xtables`, `columns`, and mechanical `field_provenance` from
the KB, import catalog-derived document attributes, then chain quality scoring and
view regeneration so the DB and `views/` are consistent after every run.

## Preconditions

- `robert-db-init` has run; `robertdb.sqlite` exists with the `meta` stamp.
- The config's `kb.kb_root` resolves to the per-doc KB folders, and `kb.layout`
  (`volcker` | `ussr` | `auto`) is correct. The harvester still **sniffs per file**;
  layout is only a hint.
- Catalogs named in `config.catalogs.*` exist where declared (absent → skipped with
  a logged notice, not an error).

## Procedure

1. **Full harvest** (typical first run):
   ```bash
   PYTHONIOENCODING=utf-8 python rdb_harvest.py \
       --config <P>/robertdb_config.json
   ```
   This discovers docs, sniffs each CSV against `adapters.enabled` (in order),
   parses, mints ids, writes mechanical provenance + columns, raises mechanical
   quality flags (e.g. `RAGGED_ROWS`, `EMPTY_TABLE`, `MD5_UNRESOLVED`,
   `COUNT_MISMATCH_VS_AUDIT`), **ingests native-enrichment sidecars** (see below),
   then **chains** `rdb_merge_patches.py` (native, if any) → `rdb_quality.py` →
   `rdb_views.py`.

## Native enrichment ingestion (HDARP v6.3)

Per `docs/NATIVE_ENRICHMENT_CONTRACT.md`: when a doc folder contains
`RDB_METADATA.jsonl` / `RDB_METADATA_chunks_*.jsonl`, harvest resolves each line's
`source_relpath` to the minted `table_uid`, writes a patch dir
`enrichment/patches/NATIVE_<harvest_ts>/<doc_id>.jsonl`, and chains
`rdb_merge_patches.py --run-id NATIVE_<harvest_ts>` **before** quality (so
`obs_status`/`transcription_status` land before `derive_transcription` fills NULLs).
Controlled by `config.kb.native_enrichment` (`auto` default | `off` | `require`).
Matched lines are reported as `native_sidecars_ingested=N` in the run note; unmatched
lines are a logged notice (never applied to the wrong table). With `--no-chain`
(bulk mode) the patch dir is still written and the merge command is printed for you to
run once at the end.

2. **Scoped harvest** (large projects / iteration):
   - `--category Foreign_Trade` — only docs whose `category_raw` matches.
   - `--doc-glob '017_*'` — only docs whose folder name matches the glob.
   - `--limit N` — cap the number of docs (smoke test).
   - `--rebuild-doc <doc_id>` — re-harvest one document. Re-harvest matches the
     natural key `(doc_id, source_relpath)` + content hash and **never renumbers**
     an existing `table_uid`; new files get new ids, changed files update in place.

3. **Inspect.** Confirm counts and parse health:
   ```bash
   PYTHONIOENCODING=utf-8 python -c "import sqlite3,sys; c=sqlite3.connect(sys.argv[1]); \
print('docs',c.execute('select count(*) from documents').fetchone()[0]); \
print('tables',c.execute('select count(*) from xtables').fetchone()[0]); \
print('parse',dict(c.execute('select parse_status,count(*) from xtables group by parse_status').fetchall()))" \
       <P>/robertdb.sqlite
   ```
   Also review `views/DB_MANIFEST.json` and the harvest run row in `runs`.

## Convention adapters (sniffed, never assumed)

`marker_no_tables`, `meta_block_then_data` (USSR), `meta_cols_embedded` (Volcker
c2), `meta_stub_only`, `multirow_header` (ru/translit/en, ru/fr), `source_page_col`,
`plain_data`. The matched adapter is recorded as `xtables.convention_code`. A
`plain_data` CSV legitimately yields `not_captured` metadata — that is honest, not a
defect.

## Outputs

- `documents`, `xtables`, `columns`, `field_provenance` (basis `mechanical`),
  `quality_flags` (mechanical) populated in the DB.
- `views/*.csv` (+ Parquet where configured) and `views/DB_MANIFEST.json`
  regenerated.
- A `harvest` run-ledger row and `state/runs/RUN_HARVEST_*.json`.

## Failure handling

- **Unparseable CSV** → row written with `parse_status='unparseable'` +
  `UNPARSEABLE_CONVENTION` flag; harvest continues. Do not hand-fix the CSV in the
  KB — fix the adapter or leave the flag for audit.
- **Marker file** (`NO TABLES`) → `parse_status='marker_no_tables'`,
  `is_marker_file=1`; this is a valid "no tables present" confirmation, not an error.
- **`pdf_md5` unresolved** against the `_UNIFIED` spine → stored literal
  `'NOT_CAPTURED'` (never NULL-silent) + `MD5_UNRESOLVED` flag.
- **Count mismatch vs `DOCUMENT_AUDIT`** → `COUNT_MISMATCH_VS_AUDIT` flag for audit
  to adjudicate; harvest does not block.
- Harvest is **re-runnable**: re-running over the same KB is idempotent on
  `(doc_id, source_relpath)`; it never duplicates or renumbers ids.
