---
name: robert-db
version: "1.0"
description: "Framework index for the Robert Database Framework — turn per-project HDARP/Hopper Knowledge_Base extractions into research-grade per-project databases (SQLite canonical + regenerated views) with honest per-field provenance, a two-axis quality model, taxonomy + concordances, and publishable packages."
when-to-use: '"User asks what the Robert Database Framework is, how to turn a project''s Knowledge_Base into a database, which robert-db-* sub-skill fits a task, or wants the database build pipeline for a project after HDARP integration."'
search-hints: "robert database framework robertdb sqlite per-project table extraction provenance two-axis quality obs_status transcription_status taxonomy concordance enrich harvest audit publish data package"
argument-hint: "[project]"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit, Agent
requires: none
part-of: Robert Database Framework v1.0
---

# Robert Database Framework v1.0

The framework index. Read this first to learn what the framework does, which
sub-skill to invoke, and how the build pipeline fits the broader Arcanum pipeline.

## What it is

The Robert Database Framework turns the mechanical output of HDARP extraction
(per-project `Knowledge_Base/` folders of `FULL_TEXT*.md` + `CSV_Tables/` +
`equations/` + `figures/`) into a **research-grade per-project database**:

- **One SQLite database per project** (`robertdb.sqlite`) — the canonical store.
  No cross-project content ever lands in a project DB.
- **Regenerated views** (CSV/Parquet under `views/`) derived from the DB — never
  hand-edited; the DB is the single source of truth.
- **Honest per-field provenance** — every metadata field of every table records
  *how we know it* (`mechanical` | `from_source` | `agent_inferred` |
  `not_captured` | `unknown`). **NOT CAPTURED is a first-class value.**
- **A two-axis quality model** — `obs_status` (SDMX CL_OBS_STATUS, the *source*
  data quality) and `transcription_status` (V/H/L/R/X, the *agent-read* fidelity).
  The transcription axis is the framework's novel methods contribution.
- **Taxonomy + concordances** — a ratified 2-level per-project topic hierarchy and
  researcher-facing families of related tables across documents.
- **Publishable packages** — Data Package v2 + codebook + Croissant + CITATION.cff
  + llms.txt, leak-scrubbed and mirrored.

The unit of record is the **extracted table** (`xtables`), keyed by an immutable
`<PJ>-T-NNNNNN` id. Documents, columns, taxonomy, concordances, quality flags,
and a run ledger hang off it. The provenance spine is
`pdf_md5 → doc_id → table_uid`.

## When to use which sub-skill

| Want to… | Skill | Stage |
|---|---|---|
| Understand the framework / pick a sub-skill | **robert-db** (this) | index |
| Scaffold the project DB from its config | **robert-db-init** | 1 — INIT |
| Mechanically harvest tables from the KB into the DB | **robert-db-harvest** | 2 — HARVEST |
| Recover honest metadata (titles, pages, units, footnotes) via agents | **robert-db-enrich** | 3 — ENRICH |
| Seed/ratify a taxonomy and build concordances | **robert-db-organize** | 4 — ORGANIZE |
| Run the QA gate, honesty checks, and spot-checks | **robert-db-audit** | 5 — AUDIT |
| Package + mirror a publishable release | **robert-db-publish** | 6 — PUBLISH |
| Run the whole pipeline end-to-end (orchestrator) | **robert-db-build** | all |

## Pipeline diagram

```
/kb-integrate-pipeline  (KB + catalogs land in the project; alias: /hdarp-integrate-pipeline)
            |
            v
  robert-db-build  (orchestrator; --resume from BUILD_STATE.json)
            |
  +---------+---------+---------+---------+---------+
  |         |         |         |         |         |
 INIT --> HARVEST --> ENRICH --> ORGANIZE --> AUDIT --> PUBLISH
 (1)      (2)         (3)        (4)          (5)       (6)
  |         |          |          |            |          |
 DB        xtables   patches   taxonomy +   AUDIT_      Data Package v2
 scaffold  + quality + merge    concordances REPORT     + mirror
           + views   (JSONL)    (ratified)   (gate)
```

ENRICH and ORGANIZE both depend only on HARVEST and may overlap; AUDIT gates
PUBLISH. The orchestrator runs them in the order above and is resumable.

## Position in the broader Arcanum pipeline

Robert DB runs **after** `/kb-integrate-pipeline`. It consumes the integrated
Knowledge_Base + catalogs (`DOCUMENT_AUDIT.csv`, `TABLE_CATALOG.csv`,
`ENTITY_CATALOG.csv`, `CLASSIFICATION_MASTER.csv`) and the cross-project
`_UNIFIED/PDF_REGISTRY.csv` + `KB_CATALOG.csv` (read-only) to resolve the
`pdf_md5` provenance spine. It consumes **future Hopper KB output identically** —
the per-doc folder shape is the same; only `documents.process_type` differs
(`HDARP` | `SPHDARP` | `Hopper`). Robert DB never re-extracts a PDF; it builds a
database from what the extraction engine already produced.

It is **distinct from**:

- **The API data-checkout service** — serves *API* economic data (FRED/BEA/BLS…)
  on checkout; it is not a document-table database.
- **Anu** — Anu builds *constructed analytical series* (`series_registry.json`,
  D/XS ids) from KB reads. Anu is a downstream consumer, not a peer pipeline.

### Anu interop

Robert DB ids are **collision-proof against Anu**: Robert mints `<PJ>-T-NNNNNN`
(tables), `<PJ>-K-NNNN` (concordances), `<PJ>-RS-NNNN` reserved (row-set); Anu
mints `D###` / `XS###`. The `-T-`/`-K-`/`-RS-` infixes never collide with Anu's
prefix scheme. An Anu project that draws a fact from a Robert table **cites the
Robert id** (e.g. `PRJ-T-004217`) in its DPR and **mints its own** `D`/`XS` id for
the constructed series. Robert is the cited source-of-record; Anu is the
constructed-series layer on top of it.

## Pointers

- **Framework overview (shipped in this repo)** — [`docs/frameworks/robert-db.md`](../../../frameworks/robert-db.md)
- **Schema** — `robertdb_schema.sql` *(engine artifact, not shipped)*
- **Config spec** — `CONFIG_SPEC.md`
- **Patch contract** — `PATCH_CONTRACT.md`
- **Engine scripts** — `rdb_*.py` (single canonical
  copy; per-project state lives in `<P>/Technical/RobertDB/`)
- **Canonical standard** —
  `ROBERT_DATABASE_FRAMEWORK_OVERVIEW.md`

## Honesty principles (in force across every sub-skill)

> **NOT CAPTURED is a first-class value.** A field we did not recover is recorded
> as `not_captured`, never silently invented.
>
> **The source page is the authority.** Titles, footnotes, page refs, and units
> are `from_source` only when visible verbatim in the CSV / `Tables/*.md` /
> `FULL_TEXT` context. Anything else is `agent_inferred` — and `from_source` is
> never downgraded by `agent_inferred`.

## Version

v1.0 (2026-06-11). See `ROBERT_DATABASE_FRAMEWORK_OVERVIEW.md`
for the full standard and version history.
