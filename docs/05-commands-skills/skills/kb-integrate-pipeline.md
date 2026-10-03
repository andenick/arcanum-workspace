---
name: kb-integrate-pipeline
description: "Method-agnostic pipeline running all 7 KB integration phases in sequence for a project's Knowledge Base, keyed by each doc's read_method."
when-to-use: "User wants to run full KB integration automation after completing PDF processing (any read engine — HDARP or Hopper) on a project"
search-hints: "kb integrate pipeline hdarp hopper knowledge base catalog sync automated read_method"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit
argument-hint: "[project] [--engine {hdarp|hopper}]"
requires: hdarp-extract
part-of: KB Integration Pipeline (KBIP) v1.0
aliases: [hdarp-integrate-pipeline]
---

# KB Integration Pipeline (KBIP) v1.0

> **Command:** `/kb-integrate-pipeline` — alias **`/hdarp-integrate-pipeline`** retained for back-compat
> (the old name still works and does the same thing).
> **What changed from "HDARP Integration Pipeline":** this is the same 7-phase pipeline, generalized to
> *any* read method. It now reads each document's **`read_method`** (from `READ_METHOD.json` or
> `manifest.json.read_method`, defaulting to `HDARP` when absent) and emits the **KBIP catalog schema v1.0**
> (a method-tagged SUPERSET of the old catalogs). **Existing HDARP projects are unaffected:** every change is
> strictly additive (new columns/catalogs blank or `read_method=HDARP`), never a change to existing data.
> Canonical standard: `KB_INTEGRATION_PIPELINE_BUILD_PLAN.md`.

## Description

Method-agnostic, automated pipeline that runs all 7 KB integration phases in sequence for a project's
Knowledge_Base. This skill orchestrates backup, audit, cataloging, classification, cross-referencing,
integration, and Robert sync without manual intervention — for documents read by **any** engine (cloud-Claude
HDARP/SPHDARP/… or local-GPU Hopper Line v2), keying every document by its `read_method` and joining the whole
chain by `source_md5` (the read-method spine).

## Usage

```bash
/kb-integrate-pipeline [project] [--engine {hdarp|hopper}]
/kb-integrate-pipeline <P>                      # Full pipeline on <P> (defaults --engine hdarp)
/kb-integrate-pipeline <P> --skip-backup
/kb-integrate-pipeline <P> --engine hopper      # a KB read by Hopper Line v2
/kb-integrate-pipeline ALL                      # All projects

# Alias (identical behavior, back-compat):
/hdarp-integrate-pipeline <P>
```

### `--engine {hdarp|hopper}` (default `hdarp`)

The documented seam for selecting the **expected** read engine of the project's KB. It does NOT change the
per-document logic — the pipeline ALWAYS reads each doc's own `read_method` (see below) and tags rows
accordingly. `--engine` sets the default/expected method for docs that lack a `READ_METHOD.json` and the
`manifest.json.read_method` field:

- `--engine hdarp` (default): docs without a self-declared method default to `read_method=HDARP`. **This is the
  legacy behavior** — every existing HDARP project re-integrates identically.
- `--engine hopper`: docs without a self-declared method default to `read_method=Hopper` (for a freshly
  Hopper-extracted KB whose `/hopper-integrate` on-ramp already tagged the docs).

Mixed-engine KBs are fully supported: the per-doc `read_method` always wins over the `--engine` default.

## Read-method awareness (KBIP v1.0)

Before cataloging, the pipeline resolves each document's `read_method` (and `read_version` / `models_used`)
from the **read-method spine** (one key, three names, same value, joined by `source_md5` —
see KBIP build plan §2):

1. **`READ_METHOD.json`** in the doc folder (preferred; schema v1.0 — see build plan §3.3). Fields read:
   `read_method`, `read_version`, `models_used`, `source_md5`, `native_enrichment`, `confidence_sidecar`.
2. else **`manifest.json.read_method`** (top-level string) + a `<!-- read_method: … -->` header comment in
   `FULL_TEXT_chunks_*.md`.
3. else **default** to the `--engine` value (`HDARP` unless `--engine hopper`).

`read_method` enum (frozen v1.0): `HDARP | SPHDARP | PHDARP | SPDARP | PDARP | Sraffa_OCR | Hopper | manual`.
`read_version` examples: `HDARP v6.3`, `Hopper Line v2 v1.0`, `Sraffa 4.0`.

**Back-compat guarantee:** a legacy HDARP project with no `READ_METHOD.json` and no `manifest.read_method`
resolves to `read_method=HDARP` — so its catalogs are identical to the old output plus the new (constant)
`read_method=HDARP` column. An agent must be able to answer "how was this doc read?" from the folder alone
(`READ_METHOD.json`) OR from `source_md5 → PROCESSING_LOG`.

## Lifecycle Position

This skill is the INTEGRATE step for any read engine.

**HDARP lane** (cloud Claude Read-tool — unchanged):
1. `/hdarp-campaign` — campaign setup
2. `/preparehdarp` — chunk and prepare PDFs
3. `/sphdarp` (or variants) — extract all 4 content types via agents
4. `/hdarp-wrapup` — validate, remediate, document, close out
5. **`/kb-integrate-pipeline`** (alias `/hdarp-integrate-pipeline`) — catalog, classify, crossref, Robert sync

**Hopper lane** (local-GPU Hopper Line v2):
1. `/hopper` — extract a PDF/folder offline on a local consumer-GPU machine (4-artifact output)
2. `/hopper-integrate` — on-ramp: LAND extraction into `<P>/Knowledge_Base/<doc>`, TAG
   `read_method=Hopper` (writes `READ_METHOD.json`, `manifest.read_method`, FULL_TEXT header), hand off
3. **`/kb-integrate-pipeline --engine hopper`** — same 7 phases, `read_method=Hopper` rows

**Prerequisites**: All batches/docs in scope must be VERIFIED (HDARP: via `/hdarp-wrapup`; Hopper: via the
HL2 S8 validate.py gate surfaced by `/hopper-integrate`) before running integration. Integration assumes
extraction quality has been validated, regardless of read method.

### Downstream

Once integration completes, the project's catalogs are ready for **`robert-db-build`** (Robert Database Framework v1.0), which consumes these catalogs to turn the Knowledge_Base into a research-grade per-project table database (SQLite canonical + honest per-field provenance + publishable packages). The KBIP v1.0 catalogs carry `read_method` through so `documents.process_type` lands correctly (HDARP docs → `HDARP`, Hopper docs → `Hopper`) — with **no schema change** between an HDARP project and a Hopper project. See `ROBERT_DATABASE_FRAMEWORK_OVERVIEW.md`.

## Pipeline Phases (Automatic)

| Phase | Name | Description | Output (KBIP v1.0) |
|-------|------|-------------|--------|
| 1 | BACKUP | Zip-based KB archive (KB_CAMPAIGN_{date}.zip) | BACKUP_MANIFEST.json |
| 2 | AUDIT | Document-level catalog (+ `read_method` resolution) | DOCUMENT_AUDIT.csv |
| 3 | CATALOG | Tables, equations, figures, entities, **chart data** | *_CATALOG.csv files (+ CHART_DATA_CATALOG.csv) |
| 4 | CLASSIFY | Source type, topic, temporal period (+ `read_method`) | CLASSIFICATION_MASTER.csv |
| 5 | CROSSREF | Document relationship mapping (+ `engines_present`) | CROSS_REFERENCE_INDEX.json |
| 6 | INTEGRATE | Project manifest and final catalogs | INTEGRATION_MANIFEST.json |
| 7 | ROBERT-SYNC | Sync to Robert unified KB (+ method-keyed ledger rows) | SYNC_REPORT.md |

Phases run identically for HDARP and Hopper; the only difference is which `read_method` value each row
carries and whether the Hopper-only columns are populated (blank for HDARP).

## Execution Model

- **Batch Size**: 100 documents per batch
- **Parallel Agents**: 5 Sonnet agents per batch for catalog phase
- **Checkpointing**: State saved after each phase
- **Resumable**: Can resume from last completed phase if interrupted
- **State File**: `Technical/HDARP_Integration/PIPELINE_STATE.json`

## Output Structure

```
{Project}/Technical/HDARP_Integration/
|-- PIPELINE_STATE.json          # Pipeline execution state
|-- BACKUP_MANIFEST.json         # Phase 1: File inventory
|-- DOCUMENT_AUDIT.csv           # Phase 2: Document audit
|-- TABLE_CATALOG.csv            # Phase 3a: All tables (+ read_method)
|-- EQUATION_CATALOG.csv         # Phase 3b: All equations (+ read_method)
|-- FIGURE_CATALOG.csv           # Phase 3c: All figures (+ read_method)
|-- ENTITY_CATALOG.csv           # Phase 3d: Extracted entities
|-- CHART_DATA_CATALOG.csv       # Phase 3e: Chart->data extractions (KBIP v1.0; blank for HDARP)
|-- CLASSIFICATION_MASTER.csv    # Phase 4: Document classifications (+ read_method)
|-- CROSS_REFERENCE_INDEX.json   # Phase 5: Document relationships (+ engines_present)
|-- INTEGRATION_MANIFEST.json    # Phase 6: Final statistics
`-- PIPELINE_COMPLETE.md         # Summary report
```

> The per-doc `READ_METHOD.json` lives in each `Knowledge_Base/<doc>/` folder (written by the extraction
> on-ramp — `/hopper-integrate` for Hopper, the W7 back-tag pass for legacy HDARP), NOT in
> `HDARP_Integration/`. The pipeline READS it; it does not author it. When absent, `read_method` defaults
> per `--engine` (HDARP by default).

## Phase Details

### Phase 1: BACKUP (Zip-Based KB Archival)

- Creates zip archive of finalized Knowledge_Base: `Archives/KB_CAMPAIGN_{YYYYMMDD}.zip`
- Includes all KB files, BATCH_STATE.json, HDARP_MASTER_CATALOG.csv, and HDARP_Processing/ contents
- Generates BACKUP_MANIFEST.json alongside the zip file
- Verifies zip integrity before proceeding
- Each campaign creates a separate zip (never modifies existing archives)

### Phase 2: AUDIT

- Enumerates all document directories — **EXCLUDING any child of `Knowledge_Base/` whose name begins
  with `_`.** These are sibling *layers* and control trees, not documents: `_OCR_Only/` (the HDARP v6.4
  Hybrid verbatim body-text layer), `_repair_backups/`, `_quarantined_partials/`,
  `_superseded_duplicates/`. Counting one writes a phantom row into `DOCUMENT_AUDIT.csv` and inflates
  the document count into every downstream catalog. The Robert DB harvest script already enforces this
  via its `_excluded_dir`; the asymmetry between the two sides was the hazard.
  **Verify after the audit: no `document_id` in `DOCUMENT_AUDIT.csv` starts with `_`.**
- Checks for: FULL_TEXT.md (primary, Sraffa 4.0), Text/ (legacy), CSV_Tables/, completion markers
- **Resolves `read_method` / `read_version` / `models_used` / `source_md5`** per the read-method spine
  (`READ_METHOD.json` → `manifest.read_method` → `--engine` default). Resolution is per-document.
- Records completeness status for each document
- Outputs: DOCUMENT_AUDIT.csv, COMPLETION_GAPS.md

**Schema v1.1 — the three Hybrid columns (added 2026-08-27, HDARP v6.4).** Append-only; v1.0 readers
are unaffected because the columns are added at the end and `schema_version` distinguishes them.

| Column | Values | Why it must be in the catalog |
|---|---|---|
| `extraction_method` | `verbatim` \| `analytical_digest` | **The single most consequential fact about a document's body text, and it was not being recorded anywhere a consumer could see it.** In one production corpus, 353 of 384 documents are `analytical_digest` — a paraphrase, not the text. A downstream user quoting from a digest as if it were the source would be quoting words the author never wrote. The HDARP processing rule ("Hybrid body text = two layers") already requires this be recorded per document; this is where it becomes visible. |
| `ocr_layer_path` | relpath to `_OCR_Only/<short_id>/` or empty | Links the document to its verbatim layer **without a second catalog row** — a second row keyed on the same `source_md5` would break the idempotent md5 key KBIP relies on. |
| `ocr_layer_complete` | `true` \| `false` \| empty | `false` = pages remain in `_OCR_Only/_GPU_OCR_QUEUE.md` awaiting the Sraffa 4.0 GPU pass. Records the gap rather than implying coverage the layer does not have. |

**`DOCUMENT_AUDIT.csv` — KBIP catalog schema v1.0 (frozen columns, build plan §3.4):**

```
document_id, document_name, source_md5, read_method, read_version, models_used,
has_full_text, full_text_chars, chunk_markers, csv_count, equation_count, figure_count,
chart_extracted_count, entity_count, page_count, born_digital_fraction, mean_confidence,
quality_score, status, batch_id, processing_date, schema_version, notes,
extraction_method, ocr_layer_path, ocr_layer_complete
```

This is the **union** of the HDARP and Hopper dialects. It is a SUPERSET of the legacy HDARP audit:
- **Always present:** `read_method` (defaults to `HDARP` for legacy docs).
- **Hopper-only columns** (`models_used`, `chart_extracted_count`, `born_digital_fraction`, `mean_confidence`)
  are **left blank** for HDARP docs — never fabricated.
- `read_version` for HDARP docs = `PROCESSING_LOG.process_version` (e.g. `HDARP v6.3`) when available, else blank.
- `schema_version` = `1.0` (KBIP). Every existing HDARP field keeps its meaning and value.

**HDARP behavior unchanged, added:** `read_method, read_version, models_used, source_md5,
chart_extracted_count, mean_confidence, schema_version` columns (plus `born_digital_fraction`); HDARP rows
get `read_method=HDARP` and blanks in the Hopper-only columns. No existing column is renamed, dropped, or
re-valued.

### Phase 3: CATALOG (Parallel Processing)

**3a. TABLE_CATALOG.csv**
- Scans all CSV_Tables/ directories
- Records: table_id, document_id, title, rows, columns, file_path, **read_method**

**3b. EQUATION_CATALOG.csv**
- Scans all Equations/ files (.tex, .md)
- Records: equation_id, document_id, latex, description, variables, **read_method**

**3c. FIGURE_CATALOG.csv**
- Scans all Figures/ and Images/ files
- Records: figure_id, document_id, title, chart_type, file_path, **read_method**

**3d. ENTITY_CATALOG.csv**
- Extracts entities from FULL_TEXT.md files (Sraffa 4.0 assembled, with provenance headers)
- Entity types: PERSON, ORGANIZATION, REGULATION, DATE, LOCATION, ACRONYM
- Records: entity_id, entity_type, entity_name, document_ids, frequency

**3e. CHART_DATA_CATALOG.csv** (KBIP v1.0 — canonical chart→data catalog)
- Scans for chart-extraction artifacts (Hopper's `chart_extracted` content type — chart→data series).
- Records (frozen, build plan §3.4):
  `document_id, chart_id, content_type, n_points, method=chart_estimated, estimated=TRUE, read_method, path`
- **For HDARP projects this catalog is created but has no rows** (HDARP has no chart→data extraction) — it
  exists so the HDARP and Hopper catalog sets are the same shape (a SUPERSET, never different data).

**HDARP behavior unchanged, added:** a `read_method` column on TABLE/EQUATION/FIGURE catalogs (constant
`HDARP` for legacy docs), and a new (empty-for-HDARP) `CHART_DATA_CATALOG.csv`. ENTITY_CATALOG is unchanged.

### Phase 4: CLASSIFY

- Applies taxonomy based on document naming patterns
- Parses Year_Source_Topic directory name format
- Classification dimensions:
  - Source Type (Regulatory_Fed, Academic_NBER, etc.)
  - Topic (Capital_Regulation, Stress_Testing, etc.)
  - Temporal Period (Pre_Basel, Basel_I_Era, etc.)
  - Content Type (Annual_Report, Working_Paper, etc.)
- Outputs: `CLASSIFICATION_MASTER.csv` += `read_method`.

**CRITICAL — `source_type` stays a REAL classification.** `CLASSIFICATION_MASTER.source_type` must hold the
document's actual provenance classification (e.g. `Regulatory_Fed`, `Academic_NBER`) — it is **never** set to
a literal engine string like `"hopper_extraction"` or `"HDARP"`. The read engine is recorded ONLY in the
separate `read_method` column. (This guards against the hopper_to_anu dialect's hard-coded `source_type`.)

**HDARP behavior unchanged, added:** a `read_method` column (constant `HDARP` for legacy docs). `source_type`
and all other classification dimensions are produced exactly as before.

### Phase 5: CROSSREF

- Maps document relationships using:
  - Citation patterns in text
  - Entity co-occurrence
  - Source organization links
- Relationship types: cites, supersedes, related_to, same_source
- Outputs: `CROSS_REFERENCE_INDEX.json` += a top-level `"engines_present": [...]` array listing the distinct
  `read_method` values across the project's docs (e.g. `["HDARP"]`, or `["HDARP","Hopper"]` for a mixed KB).

**HDARP behavior unchanged, added:** the `engines_present` array (`["HDARP"]` for a pure HDARP project). All
relationship mapping is produced exactly as before.

### Phase 6: INTEGRATE

- Generates INTEGRATION_MANIFEST.json with statistics (now includes a per-`read_method` doc/table breakdown)
- Creates INTEGRATION_COMPLETE.md summary
- Validates all catalog completeness

**HDARP behavior unchanged, added:** a `read_method` breakdown in the manifest stats (all-`HDARP` for a pure
HDARP project).

### Phase 7: ROBERT-SYNC

> **Ledger-writer code is W5 (out of scope here).** This section SPECIFIES the behavior; the actual
> `PROCESSING_LOG`/`KB_CATALOG`/`PROVENANCE_LEDGER` row-writer is implemented in build-plan item W5.

- Registers project in PROJECT_REGISTRY.json (v2.0 schema — see `ROBERT_PDF_LIBRARY_V2_STANDARD.md`)
- Copies catalogs to `Robert/HDARP_Integration/`
- **Writes method-keyed ledger rows, keyed by `source_md5`** (the read-method spine; build plan §3.5):
  - `PROCESSING_LOG`: `process_type` / `processing_protocol` = the doc's resolved `read_method`
    (HDARP docs → `HDARP`; Hopper docs → `Hopper`), with `process_version` = the doc's `read_version`.
  - `KB_CATALOG`: per-KB-doc counts + `kb_path`, carrying `read_method` and `source_md5`.
  - `PROVENANCE_LEDGER`: `processing_protocol` = the doc's `read_version` (e.g. `Hopper Line v2 v1.0`),
    `meta_sidecar_path` = the doc's confidence sidecar when present.
  - Rows are written **idempotently** (keyed by `source_md5`; replay yields no dupes).
- Updates UNIFIED_CATALOG.csv, UNIFIED_ENTITY_INDEX.csv
- **PDF sync (v2.0 convention)**: source PDFs are migrated into `_PDF_LIBRARY/<Project>/` (flat) via `migrate_project_to_robert.py` (or the `/robert-pdf-sync` skill wrapper). Content-type symlinks land in `_PDF_LIBRARY/_BY_CONTENT_TYPE/<type>/`. Naming: `<wl_id>__<title>__<source>.pdf` for wishlist-tracked acquisitions; `[YYYY]__Author__Title__hash.pdf` for bulk-library entries — both md5-anchored via UNIFIED_PDF_CATALOG.csv.
- Generates SYNC_REPORT.md

## State File Schema

```json
{
  "project": "<P>",
  "version": "1.0",
  "kbip_version": "1.0",
  "engine_default": "hdarp",
  "started": "2026-02-17T00:00:00Z",
  "last_updated": "2026-02-17T12:00:00Z",
  "phases_complete": ["backup", "audit"],
  "current_phase": "catalog",
  "documents_total": 2551,
  "documents_processed": 450,
  "errors": [],
  "warnings": []
}
```

The `kbip_version`/`engine_default` keys are additive; a legacy state file without them resumes unchanged
(treated as `kbip_version: "1.0"`, `engine_default: "hdarp"`).

## Integration with Other Skills

- **After (HDARP)**: /phdarp, /sphdarp, /pdarp, /spdarp (HDARP processing complete)
- **After (Hopper)**: /hopper → /hopper-integrate (on-ramp tags `read_method=Hopper`)
- **Uses**: /hdarp-integrate phases internally
- **Updates**: Robert's unified Knowledge_Base (method-keyed ledger rows by `source_md5`)
- **Downstream**: `robert-db-build` (consumes the KBIP v1.0 catalogs; `read_method` → `documents.process_type`)

## Success Criteria

- [ ] All documents audited (DOCUMENT_AUDIT.csv)
- [ ] All tables cataloged (TABLE_CATALOG.csv)
- [ ] All equations cataloged (EQUATION_CATALOG.csv)
- [ ] All figures cataloged (FIGURE_CATALOG.csv)
- [ ] Entities extracted (ENTITY_CATALOG.csv)
- [ ] Chart→data cataloged (CHART_DATA_CATALOG.csv; empty for HDARP)
- [ ] `read_method` resolved + present on every audit/classify/table/equation/figure row
- [ ] Classifications applied (CLASSIFICATION_MASTER.csv; `source_type` is a REAL class, not an engine string)
- [ ] Cross-references mapped (CROSS_REFERENCE_INDEX.json; `engines_present` set)
- [ ] Synced to Robert unified KB (method-keyed ledger rows by `source_md5`)
- [ ] PIPELINE_COMPLETE.md generated

---

**Skill Version**: 2.0 (KBIP v1.0)
**Command**: `/kb-integrate-pipeline` (alias `/hdarp-integrate-pipeline`)
**Status**: PRODUCTION READY
**Created**: 2026-02-17
**Updated**: 2026-06-22 (W4 — method-agnostic KBIP v1.0; prior: 2026-04-30)
**Dependencies**: hdarp-integrate.md, Sraffa 4.0 Protocol, Robert Knowledge_Base, READ_METHOD.json schema v1.0
**Note**: FULL_TEXT.md is now the primary text source (Sraffa 4.0 assembled with provenance). Text/ directory is legacy fallback.

> **Back-compat note (D3 rename):** the file name `hdarp-integrate-pipeline.md` is retained so the old
> `/hdarp-integrate-pipeline` invocation keeps working; the canonical command is now `/kb-integrate-pipeline`
> (see `aliases:` in the frontmatter). "HDARP integration" survives only as this doc note. No deployed-vs-master
> sync step applies — this skill ships as a single deployed file under `skills` (no copy exists in
> `master`).

<!-- KBIP v1.0 (2026-06-22, W4): renamed HDARP Integration Pipeline -> KB Integration Pipeline; method-agnostic
     (read_method spine), KBIP catalog schema v1.0 superset, --engine seam, alias preserved. Strictly additive /
     back-compatible per KB_INTEGRATION_PIPELINE_BUILD_PLAN.md. -->
<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
<!-- HDARP Framework v6.3 (2026-06-13): native RDB enrichment capture added. -->
